# Frontend — Web Component via the SDK

```ts
import { defineMosaicastElement } from '@mosaicast/plugin-sdk';

defineMosaicastElement({
  tag: 'sample-highlight',
  render: ({ ctx, root }) => {
    // …build DOM into `root` (a shadow root)…
    return () => { /* cleanup: call every unsubscribe you took */ };
  },
});
```

Re-assigning `ctx` re-renders, running the cleanup your previous render returned. Defining a tag that is
already defined is a no-op. Bundle as an ES library with **no externals** — the SDK and your framework must
be bundled so you never collide with the host's own copies.

## `ctx` — the typed surface

```
scope: { type: 'site'|'feed'|'season'|'episode'; id: string }   // slot scope; never `user`; site id is 'main'
episodes: string[]                                              // resolved, access-filtered public slugs
episodeLabels?: Record<string, string>                          // slug → label; may be absent or partial
episode?: { status: 'PLANNED'|'PUBLISHED'|'WITHDRAWN' }         // see "what the host actually supplies"
user: { id: string; role: 'admin'|'podcaster'|'fan' } | null    // null = anonymous
api: { get/post/put/delete<T>(path, body?): Promise<T> }        // /api/plugins/<id>/*, auth attached
log(level: 'debug'|'info'|'warn'|'error', message: string): void
consent: { has(cat), granted(), request(cat): Promise<boolean>, onChange(cb): Unsubscribe }
filter:  { current(): FilterState; onChange(cb): Unsubscribe }  // read-only — plugins consume, never define axes
player:  { currentTime(): number; seekTo(s): void; on(ev, cb): Unsubscribe }
route:   { path: string; onChange(cb): Unsubscribe }            // subpath under /p/<id>/
locale:  { current(): string; onChange(cb): Unsubscribe }
progress:{ get(episodeId): Promise<number | null> }             // core listening progress, seconds
theme: ThemeTokens
```

Also exported: `PLATFORM_API_VERSION`, `SELF_SCOPE_ID` (`'me'`), `DataScopeType`, `DocEntry<T>`,
`PagedDocs<T>`, `resolveArtwork(snapshot)`, `createPluginI18n`, and the documentation-only manifest types.

## What the host actually supplies today

The type is ahead of the shell. Verified against core's `buildCtx.ts` — assume this until core says otherwise:

- **`ctx.episode` is not populated.** Never branch on `episode?.status`; if you need publication state, it is
  not available client-side.
- **`filter.current()` always returns `{}`**, and **`filter.onChange`, `player.on`, `route.onChange` and
  `locale.onChange` return no-op unsubscribes that never fire.** Only `consent.onChange` is live.
- `route.path` is set once per mount (re-mounted on navigation), `locale.current()` is fixed for the mount,
  and `progress.get` reads `localStorage["mc.progress.<slug>"]`.

Practical rule: **read state at render time**; treat the event APIs as forward compatibility. Keep taking
their return values and returning them from cleanup — they will start firing, and a leaked subscription into
a detached shadow root is the bug you cannot see.

## `ctx.api` — the host's fixed doc-store surface

You do not author backend routes. The host exposes exactly:

```
GET    data/{scopeType}/{scopeId}/{key}
PUT    data/{scopeType}/{scopeId}/{key}          body = raw JSON  → 204
DELETE data/{scopeType}/{scopeId}/{key}                           → 204, idempotent
GET    data/{scopeType}/{scopeId}?prefix=&page=&size=             → PagedDocs<T> { items, page, size, totalElements, totalPages }
```

`scopeType` ∈ `site | feed | season | episode | user`. Paging: `page` default 0, `size` default 50,
**max 200**. Keys must match `^[A-Za-z0-9._:-]{1,200}$`. Scope ids are slugs — `encodeURIComponent` every
path segment you interpolate.

Status codes worth handling: **404** unknown or **disabled** plugin, unknown scope, or a missing document ·
**400** a `user` id other than `me`, or an illegal key · **401** `user/me` while anonymous · **403** below the
manifest's read/write floor · **403 `problems/backend-owned-key`** on a key the manifest reserves for the
backend · **429** on the log endpoint only.

There are **no ETags, no `If-Match`, no 409 and no size cap** on the doc store — writes are last-write-wins.
Two tiles editing one key will clobber each other; if that matters, partition the keys. The doc API is not
rate-limited; `POST /api/plugins/<id>/log` is (per plugin, fixed one-minute window) and is gated by the
**write** floor.

### Per-user data

```ts
import { SELF_SCOPE_ID } from '@mosaicast/plugin-sdk';
await ctx.api.put(`data/user/${SELF_SCOPE_ID}/mark:${ctx.scope.id}:b3`, { marked: true });
```

`SELF_SCOPE_ID` is the literal `me`, resolved server-side from the session. Any other `user` id is a **400**
(never a silent substitution); anonymous is a **401**. The partition is **flat** — one per user, not per user
*and* entity — so the entity goes in the key. `DataScopeType` (`Scope['type'] | 'user'`) is a separate type
from `Scope` on purpose: `user` addresses storage, never a slot, and never appears in `ctx.scope`.

There is no TS counterpart to `queryAcrossUsers`, deliberately. Build leaderboards on the backend and read
the precomputed result from an entity scope.

**Migrating a plugin that stored per-user keys under entity scopes:** the backend can read legacy keys but
cannot write into anyone's partition, so the move is necessarily client-side and lazy. On mount, if the new
`data/user/me/…` key is absent, read the legacy one, `put` it to the new path, `delete` the old. Keep that
path until everyone has been back, then drop it and have the backend delete whatever legacy keys remain.

## Logging

`ctx.log(level, message)` is the **only** way to log from the frontend — never POST to
`/api/plugins/<id>/log` through `ctx.api` yourself. Failures are swallowed on purpose: a plugin reporting a
problem must not become a second, louder problem in the visitor's browser.

## Consent and CSP

`ctx.consent.request(category)` opens the host's consent settings and resolves with the visitor's answer —
this is what powers a click-to-load placeholder. **Call it from a click handler, never from render/mount.**
It is one host-wide surface: concurrent calls from several tiles join the same dialog, and every call
resolves exactly once. Resolving `true` loads nothing for you — load the gated resource afterwards yourself.
Re-check `ctx.consent.has(category)` on every load rather than caching a `request()` result; consent can be
withdrawn mid-session, which is what `onChange` is for.

What the CSP actually does with your manifest's `consent.services[].hosts`:

- Granted hosts widen **only** `script-src`, `frame-src` and `connect-src`.
- `img-src` / `media-src` get a blanket `https:` **unless** the operator enables
  `mosaicast.security.strict-media-sources`, which narrows them to feed-derived origins plus your consented
  hosts. **An image that works today can break the moment an operator flips that flag** — so declare the
  origins you load media from, even though it looks unnecessary in the default configuration.
- Refused or undecided ⇒ the origin is absent from the header entirely. `ctx.consent.has()` is advisory; the
  header is not. An undeclared host stays blocked even with consent granted.
- `style-src 'unsafe-inline'` is kept deliberately, because a runtime-constructed shadow root cannot use a
  nonce. Style your component inline as usual.

## Theme

The SDK injects `ctx.theme` into the shadow root as CSS custom properties — do not do this by hand. Just
reference `var(--mc-*)`:

```
bg→--mc-bg          surface→--mc-surface   text→--mc-text            textMuted→--mc-text-muted
accent→--mc-accent  accentContrast→--mc-accent-contrast              accent2→--mc-accent-2   border→--mc-border
```

## i18n and deep links

```ts
const i18n = createPluginI18n({ en, de }, ctx.locale);   // locales/en.json is the source and the fallback
// render → i18n.t('key', params)
// cleanup → i18n.dispose()
```

Resolution is active locale → `en` → the key itself. It subscribes to `ctx.locale.onChange` for its whole
life, so **call `dispose()` from your render's cleanup** or the subscription leaks. Feed and author content
is data, not UI — do not translate it.

Deep links live under `/p/<pluginId>/…` and require a `page` slot declared with `scope: "site"`; without one
the route is a real 404. The subpath arrives as `ctx.route.path`. For link previews, implement
`ShareMetadataProvider` on the backend (see `backend.md`) — the page itself is client-rendered, so OG tags
are the only thing a crawler sees.
