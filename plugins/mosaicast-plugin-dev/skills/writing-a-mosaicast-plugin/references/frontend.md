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
schema: SchemaClient | null                                     // null unless the manifest declares storage.schema
blobs: BlobClient | null                                        // null unless the manifest declares a blobs block
log(level: 'debug'|'info'|'warn'|'error', message: string): void
consent: { has(cat), granted(), request(cat): Promise<boolean>, onChange(cb): Unsubscribe }
filter:  { current(): FilterState; onChange(cb): Unsubscribe }  // read-only — plugins consume, never define axes
player:  { currentTime(): number; seekTo(s): void; on(ev, cb): Unsubscribe }
route:   { path: string; onChange(cb): Unsubscribe;             // subpath under /p/<id>/
           navigate(subpath, { replace? }): void }              // SPA move inside your own subtree
links:   { episode(slug, { t? }): string;                       // host URL shapes — strings, not navigation
           feed(slug, { season?, tag?, order? }): string }
locale:  { current(): string; onChange(cb): Unsubscribe }
progress:{ get(episodeId): Promise<number | null> }             // core listening progress, seconds
theme: ThemeTokens
```

Also exported: `PLATFORM_API_VERSION`, `SELF_SCOPE_ID` (`'me'`), `DataScopeType`, `DocEntry<T>`,
`PagedDocs<T>`, `PluginRoute`, `PluginLinks`, `SchemaClient`/`SchemaQuery`/`SchemaPredicate`/`SchemaOp`/
`SchemaPage<T>`, `BlobClient`/`BlobInfo`/`BlobPage`/`BlobQuota`, `resolveArtwork(snapshot)`,
`createPluginI18n`, and the documentation-only manifest types. Note there is **no** TS type for the
manifest's `blobs` block — core validates it, and nothing in the SDK reads `plugin.json`.

## What the host actually supplies today

The type is ahead of the shell. Verified against core's `buildCtx.ts` — assume this until core says otherwise:

- **`ctx.episode` is not populated.** Never branch on `episode?.status`; if you need publication state, it is
  not available client-side.
- **`filter.current()` always returns `{}`**, and **`filter.onChange`, `player.on`, `route.onChange` and
  `locale.onChange` return no-op unsubscribes that never fire.** `consent.onChange` and `route.navigate` are
  the live ones.
- `route.path` is **not** subscribed to — the host rebuilds `ctx` when the subpath changes and re-assigns it,
  which re-runs your render (with your previous cleanup first). So read `ctx.route.path` at render time and
  a route change reaches you; `onChange` still never fires.
- `route.navigate` is a real router call and works (see below), except in a mount with no router above it,
  where it degrades to a no-op rather than throwing.
- `locale.current()` is fixed for the mount, and `progress.get` reads `localStorage["mc.progress.<slug>"]`.

Practical rule: **read state at render time**; treat the event APIs as forward compatibility. Keep taking
their return values and returning them from cleanup — they will start firing, and a leaked subscription into
a detached shadow root is the bug you cannot see.

## `ctx.api` — the host's fixed doc-store surface

You do not author backend routes. `ctx.api` is the **doc store's** surface — the schema store and the blob
store have their own, below — and the host exposes exactly:

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

## `ctx.schema` — the host's read-only schema surface (0.7.0)

The frontend counterpart of the Java `SchemaStore`, and the second of the host's three surfaces — separate
paths, separate rules. `ctx.schema` is `null` unless the manifest declares `storage.schema`, mirroring
`ctx.schema()` on the backend, so TypeScript makes you handle the doc-store case:

```ts
if (!ctx.schema) return;                       // doc-store plugin: nothing to query

interface Page { id: number; slug: string; title: string; markdown: string; updatedAt: string }

const hits = await ctx.schema.search<Page>('page', 'markdown', term, {
  where: [{ field: 'published', op: 'eq', value: true }],
  orderBy: [{ field: 'updatedAt', direction: 'desc' }],
  size: 20,
});                                            // SchemaPage<Page>: items, page, size, totalElements, totalPages
```

```
select<T>(entity, query?)                  → SchemaPage<T>
search<T>(entity, field, text, query?)     → SchemaPage<T>   // field must be declared :fulltext
find<T>(entity, id)                        → T | null        // null on a missing row, does NOT reject
count(entity, { where }?)                  → number
```

`SchemaQuery` = `{ where?: SchemaPredicate[]; orderBy?: { field, direction }[]; page?; size? }`.
`SchemaOp` = `eq ne lt lte gt gte like in isNull isNotNull` (the Java `Criteria.Op` lower-cased).
**Predicates AND only** — no `or`, same as `Criteria`; model it as two queries and merge. Paging is
`page`/`size` (from 0, default 50, host caps at 200), not `limit`/`offset`.

What the host enforces:

- **Names come from your manifest.** Entity and field names are resolved server-side against your own
  declaration and values are bound, never interpolated — another plugin's tables are unnameable, not merely
  blocked.
- Access is the same **`data.readableBy`** floor as the doc store. One rule for both surfaces.
- **404** an entity you never declared (or a doc-store plugin hitting `schema/…` at all) · **400** an
  undeclared field, a value that will not coerce to the declared type, or `search` on a field that is not
  `:fulltext` · **403** below `readableBy`.
- Empty `search` text matches **nothing**, not everything. `like` is case-sensitive with `%` as the wildcard;
  for text inside a field use `search`, which reads the GIN index a leading-wildcard `like` cannot.
- Results are ranked best-match first unless your `orderBy` replaces that ordering.

Values go in the field's declared type — a `Date` or an ISO-8601 string for `timestamp`. One wire-format
trap: an `in` list is comma-joined with **no escaping**, so a value containing a comma cannot be expressed
in an `in` predicate.

**Reads only, and that is the contract, not a gap.** A v1 plugin authors no HTTP routes, so no plugin code
runs at request time to enforce slug uniqueness, append a revision atomically or reject malformed input.
The backend stays the only writer: a frontend that must write puts a document in the doc store and the
backend ingests it in `onSchedule(...)` — eventually consistent, so render optimistically or show a
"saving…" state that resolves on the next read.

The raw endpoints, if you ever bypass the client:

```
GET /api/plugins/<id>/schema/{entity}?where=&orderBy=&page=&size=
GET /api/plugins/<id>/schema/{entity}/search?field=&q=&where=&orderBy=&page=&size=
GET /api/plugins/<id>/schema/{entity}/count?where=          → { count }
GET /api/plugins/<id>/schema/{entity}/{rowId}               → the row, or 404
```

## `ctx.route.navigate` — moving inside your own subtree (0.7.0)

Only meaningful for a plugin with a `page` slot.

```ts
link.addEventListener('click', (e) => {
  e.preventDefault();
  ctx.route.navigate('glossary/kraken');           // → /p/<pluginId>/glossary/kraken
});

ctx.route.navigate('index', { replace: true });    // swaps the entry: no back-button step
```

- `subpath` is the same coordinate `route.path` hands you, relative to `/p/<pluginId>/`. The host prefixes
  the namespace, strips a leading `/` and drops `.`/`..` segments, so **you cannot name another plugin's
  route or a core one**; `?query` and `#hash` survive.
- Real SPA navigation — a history entry, a working back button, and no re-fetch of the shell, the registry
  or any plugin bundle. That is the whole point: an `<a href>` to your own page is a full document load.
- **Keep the real `href` on the anchor.** Middle-click, "open in new tab", crawlers and shareability all
  need it; `navigate` only takes over the plain-click path.
- **Never `history.pushState` + a synthetic `popstate`.** It happens to work against the host's current
  router and is not part of the contract.

## `ctx.blobs` — file storage (0.8.0)

`null` unless the manifest declares a `blobs` block, exactly like `ctx.schema`. Unlike the schema surface,
**writes are the point**: a file carries no relational invariant for plugin code to enforce, so
`data.writableBy` plus the quota is the whole authorization story and your own editing UI uploads directly.

```ts
const blobs = ctx.blobs;
if (!blobs) return;                                   // this plugin declared no `blobs` block

const stored = await blobs.upload(file);              // a File from <input type="file">
img.src = blobs.urlFor(stored.ref);                   // derive at render time
await ctx.api.put('data/site/main/logo', { ref: stored.ref });   // store the ref, never the URL
```

```
upload(file: File | Blob, { filename? })  → BlobInfo   // multipart POST; filename overrides the File's own
list({ page?, size? })                    → BlobPage   // newest first; { items, page, size, total }
remove(ref)                               → void       // idempotent
urlFor(ref)                               → string     // /api/plugins/<id>/blob/<ref>, root-relative
quota()                                   → BlobQuota  // { usedBytes, quotaBytes, maxFileBytes }
```

`BlobInfo` = `{ ref, filename: string | null, mime, size, updatedAt }`. **`mime` is what the host determined
from the bytes**, not what the browser claimed — the two differ exactly when someone lied, which is why it is
the value worth keeping.

Rules that matter:

- **The `ref` is the identity; the URL is derived.** Store the ref in your doc/row and call `urlFor` at
  render time. A stored URL is a copy of a decision the host is entitled to change.
- **Nothing collects orphans.** A file outlives the document that named it and only your plugin knows which
  those are — delete what you stop pointing at, or sweep from the backend on a schedule.
- **Surface refusals.** The person who picked the file is the only one who can pick a different one. The
  host checks, in order: size against your effective ceiling, the declared type against the effective
  allow-list, the *actual* type sniffed from the leading bytes, then the quota. **SVG is never accepted.**
- **Read the quota first.** `quota()` reports the *effective* numbers (operator caps and any admin grant),
  not what your manifest asked for. Telling someone the ceiling beats refusing them after an upload.

Status codes worth handling: **404** no `blobs` block, unknown/disabled plugin, or an unknown ref · **413
`problems/blob-quota-exceeded`** the file is over the per-file ceiling or would exceed the quota · **415
`problems/blob-type-not-allowed`** the declared type is not permitted, or the bytes contradict it (worded
apart on purpose — the fixes differ: send a smaller file versus delete something first) · **403** below the
relevant `data` floor. Blob writes also sit in the upload rate-limit bucket, matched by path shape, so
ordinary doc-store writes are unaffected.

Serving is **same-origin under `/api/`**, so rendering your own upload needs **no CSP host and no consent
decision** — which an external image URL cannot say. Downloads support `Range` (206) and are cached
immutably, since a ref is a fresh UUID per upload and never reused.

## `ctx.links` — the host's own URL shapes (0.8.0)

```ts
ctx.links.episode('kraken')               // /episodes/kraken
ctx.links.episode('kraken', { t: 724 })   // /episodes/kraken?t=724   (12:04)
ctx.links.feed('main', { season: '2' })   // /feeds/main?season=2
ctx.links.feed('main', { order: 'oldest', tag: 'interview' })
```

**Strings, not navigation** — put the result in a real `href` and let the visitor click. It grants no new
capability (you could always write any `href`); it moves knowledge of core's URL shapes back to the host, so
a plugin that linked to an episode stops being a thing that breaks when a route changes. Deliberately *not*
part of `ctx.route`, which is namespace-confined by construction.

Two details the builders handle for you: `?t=0` and a non-finite or negative `t` are dropped (one moment,
one URL), and `order: 'newest'` is left out because it is the host's default and what `SiteUrls`
canonicalizes to. `?t=` seeks the player to that second and **beats the listener's stored position without
overwriting it** — the position is only written back once playback advances five seconds past the shared
one.

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
the route is a real 404. The subpath arrives as `ctx.route.path`, and you move between them with
`ctx.route.navigate` (above). For link previews, implement
`ShareMetadataProvider` on the backend (see `backend.md`) — the page itself is client-rendered, so OG tags
are the only thing a crawler sees.
