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

Defining a tag that is already defined is a no-op. Bundle as an ES library with **no externals** — the SDK
and your framework must be bundled so you never collide with the host's own copies.

**Re-assigning `ctx` no longer means "destroy and rebuild" unless your render returns a bare cleanup
callback (0.15.0).** That default still holds — cleanup runs, `root` is cleared, `render` is called again —
and a static component needs to change nothing. But a host that rebuilds its own context object on every
one of its renders reassigns `ctx` roughly **four times a second during playback**, and every reassignment
under the old rule tore a stateful component down: lost input, lost scroll position, a dialog slammed shut,
every in-flight request re-fired. Return a `MosaicastHandle` instead and you decide what a new `ctx` costs:

```ts
import { defineMosaicastElement, type MosaicastHandle } from '@mosaicast/plugin-sdk';

defineMosaicastElement({
  tag: 'bingo-card',
  render: ({ ctx, root }): MosaicastHandle => {
    const app = mountMyFramework(root, ctx);
    return { update: (next) => app.setCtx(next), destroy: () => app.unmount() };
  },
});
```

```ts
interface MosaicastHandle {
  update?(ctx: PluginContext): void;   // called in place of a re-render; root and its DOM are untouched
  destroy?(): void;                    // real teardown — a disconnect, or a re-render with no `update`
}
```

- **`update` runs with the SDK's own bookkeeping already done** — the `--mc-*` theme variables are
  refreshed before it's called, so you don't have to.
- **An identical `ctx` object is ignored either way**, `update` present or not — you never have to diff a
  context yourself to protect a component from a churning host.
- **A DOM move (disconnect immediately followed by reconnect) no longer leaves the element dead.** Before,
  a move tore the render down and left it waiting for a *different* context object that might never come;
  now a reconnect renders again.
- **Returning a cleanup callback still means exactly what it always did** — no behavior changes for a
  component that ignores this entirely, so upgrading costs nothing unless you opt in.
- **If you built a module-level cache to survive the remount storm, this is the release to delete it —
  measure first**, since the storm itself is what `update` exists to stop.

## `ctx` — the typed surface

```
scope: { type: 'site'|'feed'|'season'|'episode'; id: string }   // slot scope; never `user`; site id is 'main'
episodes: string[]                                              // resolved, access-filtered public slugs
episodeLabels?: Record<string, string>                          // slug → label; may be absent or partial
episode?: { status: 'PLANNED'|'PUBLISHED'|'WITHDRAWN' }         // see "what the host actually supplies"
user: { id: string; role: 'admin'|'podcaster'|'fan';                // null = anonymous
         displayName: string; avatarUrl: string } | null            // presentation, added 0.13.0
api: PluginApiClient                                            // /api/plugins/<id>/*, auth attached; typed errors
docs: DocClient                                                  // never null — typed doc-store client (0.9.0)
feeds: FeedsClient                                               // never null — episode display snapshots (0.9.0)
sanitize(html: string | null | undefined): string                // never null — the host's HTML policy (0.16.0)
tags: TagsClient | null                                          // null unless the manifest declares `tags` (0.9.0)
users: UserDirectory | null                                      // null unless the manifest declares `identity` (0.13.0)
notify: NotifyClient | null                                      // null unless the manifest declares `notifications` (0.14.0)
schema: SchemaClient | null                                     // null unless the manifest declares storage.schema
blobs: BlobClient | null                                        // null unless the manifest declares a blobs block
log(level: 'debug'|'info'|'warn'|'error', message: string): void
consent: { has(cat), granted(), request(cat): Promise<boolean>, onChange(cb): Unsubscribe }
filter:  { current(): FilterState; onChange(cb): Unsubscribe }  // read-only — plugins consume, never define axes
player:  { currentTime(): number; seekTo(s): void; on(ev, cb): Unsubscribe }
route:   { path: string; query: URLSearchParams; hash: string;  // subpath, query, hash under /p/<id>/ (0.9.0)
           onChange(cb): Unsubscribe;
           navigate(subpath, { replace? }): void }              // SPA move inside your own subtree
links:   { episode(slug, { t? }): string;                       // host URL shapes — strings, not navigation
           feed(slug, { season?, tag?, order? }): string }
locale:  { current(): string; onChange(cb): Unsubscribe;
           available(): LocaleInfo[]; content(): LocaleInfo[] }  // UI vs. authoring languages (0.10.0)
translation: TranslationClient | null                            // null unless declared AND configured (0.10.0/0.11.0)
progress:{ get(episodeId): Promise<number | null> }             // core listening progress, seconds
theme: ThemeTokens
```

Also exported: `PLATFORM_API_VERSION`, `SELF_SCOPE_ID` (`'me'`), `DataScopeType`, `DocEntry<T>`,
`PagedDocs<T>`, `PluginRoute`, `PluginLinks`, `SchemaClient`/`SchemaQuery`/`SchemaPredicate`/`SchemaOp`/
`SchemaPage<T>`, `BlobClient`/`BlobInfo`/`BlobPage`/`BlobQuota`, `DocClient`/`DocTarget`/`DOC_KEY_PATTERN`,
`FeedsClient`/`DISPLAY_BATCH_LIMIT`, `TagsClient`/`TagInfo`, `UserDirectory`/`UserRef`,
`NotifyClient`/`NotifyMessage`/`notifyText`, `TranslationClient`/`TranslationRequest`/
`TranslationResult`, `LocaleInfo`, `PluginApiError`/`isPluginApiError`/`ProblemDetail`,
`resolveArtwork(snapshot)`, `matchRoute(path, patterns)`, `iconCss`/`iconMask`, `createPluginI18n`, and the
documentation-only manifest types (`PluginManifest`, `PluginTagsDeclaration`, `PluginExternalDeclaration`,
`ExternalServiceKind`, `PluginNavDeclaration`, `PluginIdentityDeclaration`, `PluginNotificationsDeclaration`,
…). Note there is **no** TS type for the manifest's `blobs` block — core validates it, and nothing in the
SDK reads `plugin.json`.

## What the host actually supplies today

Verified against core's `frontend/src/plugins/buildCtx.ts` and `PluginMount.tsx` at 0.7.2 — assume this
until core says otherwise. **`docs`, `feeds`, `tags`, `users`, `notify`, `schema`, `blobs`, `translation`,
`locale.available/content` and `consent.has/granted/request` are real, wired implementations** — the
"contract ahead of implementation" gap 0.9.0 shipped with has closed for all of them.

**The 4×/s `ctx` churn `MosaicastHandle` (0.15.0) exists to survive is itself already fixed on the host
side, as of 0.7.2.** `PluginMount.tsx` wraps `buildCtx(...)` in `useMemo`, and the player-time reader in its
dependency array (`playerCurrentTime`) is itself a `useCallback(() => playerRef.current.currentTime, [])` —
a stable reference reading a ref, not a value that changes every render. The SDK's own migration guide
hedges this as "independent of this release, can land before it"; it has landed. That doesn't make
`MosaicastHandle` optional — a slower host tick, another plugin's re-render, or a future core change can
still reassign `ctx` more than "never" — but a component you're debugging today for state loss on this host
is not hitting the four-times-a-second case the SDK's example was written against.

- **`ctx.episode` is not populated.** Never branch on `episode?.status`; if you need publication state, it is
  not available client-side. Still true at 0.7.2 — this is the one field the shell has never wired.
- **`filter.current()` always returns `{}`**, and **`filter.onChange`, `player.on`, `route.onChange` and
  `locale.onChange` return no-op unsubscribes that never fire.** `consent.onChange` and `route.navigate` are
  the live ones. This is unchanged since the 0.8.0-era skill and is worth re-checking on every bump, since
  it is exactly the kind of thing that quietly starts working.
- `route.path` is **not** subscribed to — the host rebuilds `ctx` when the subpath changes and re-assigns it,
  which re-runs your render (with your previous cleanup first). So read `ctx.route.path` at render time and
  a route change reaches you; `onChange` still never fires. `route.query`/`route.hash` are populated **only
  on a page mount** (`/p/<id>/…`); a slot mounted in a core region has none, because its query string is the
  shell's filter state, which is `ctx.filter`'s to expose.
- `route.navigate` is a real router call and works (see below), except in a mount with no router above it,
  where it degrades to a no-op rather than throwing.
- `locale.current()` is fixed for the mount; `locale.available()`/`locale.content()` come from
  `GET /api/i18n/locales` and the admin's content-languages setting respectively — real lists, not stubs, and
  they may legitimately disagree with your own `locales/*.json` catalogs (§12.7).
- `ctx.translation` is non-`null` **only** when your manifest declares `external.kinds: ["translation"]`
  **and** the site admin has selected a provider — every current install unless one was configured. The
  shell resolves both halves into a single `hasTranslation` flag before deciding, so the two reasons for
  `null` really are indistinguishable from inside a component, exactly as the SDK docs say.
- `progress.get` reads `localStorage["mc.progress.<slug>"]`.

Practical rule: **read state at render time**; treat the event APIs as forward compatibility. Keep taking
their return values and returning them from cleanup — they will start firing, and a leaked subscription into
a detached shadow root is the bug you cannot see.

## `ctx.api` — the host's fixed doc-store surface

You do not author backend routes. `ctx.api` is the **doc store's** surface — the schema store and the blob
store have their own, below — and the host exposes exactly:

```
GET    data/{scopeType}/{scopeId}/{key}                           → the doc, or 204 when not set (core 0.7.4)
GET    data/{scopeType}?ids=a,b&keys=x,y                          → { a: { x: … }, b: {} } — ≤100 ids, ≤100 keys
PUT    data/{scopeType}/{scopeId}/{key}          body = raw JSON  → 204
DELETE data/{scopeType}/{scopeId}/{key}                           → 204, idempotent
GET    data/{scopeType}/{scopeId}?prefix=&page=&size=             → PagedDocs<T> { items, page, size, totalElements, totalPages }
```

`scopeType` ∈ `site | feed | season | episode | user`. Paging: `page` default 0, `size` default 50,
**max 200**. Keys must match `^[A-Za-z0-9._:-]{1,200}$`. Scope ids are slugs — `encodeURIComponent` every
path segment you interpolate.

Status codes worth handling: **204** the key is simply not set (it was a 404 before core 0.7.4 / SDK 0.16.0) ·
**404** unknown or **disabled** plugin, or unknown scope — a *wrong address* now, never "not set" ·
**400** a `user` id other than `me`, or an illegal key · **401** `user/me` while anonymous · **403** below the
manifest's read/write floor · **403 `problems/backend-owned-key`** on a key the manifest reserves for the
backend · **429** on the log endpoint only.

There are **no ETags, no `If-Match`, no 409 and no size cap** on the doc store — writes are last-write-wins.
Two tiles editing one key will clobber each other; if that matters, partition the keys. The doc API is not
rate-limited; `POST /api/plugins/<id>/log` is (per plugin, fixed one-minute window) and is gated by the
**write** floor.

### Typed rejections and `getOrNull` (0.9.0)

Every `PluginApiClient` method rejects with a `PluginApiError` on a non-2xx response — `status` plus the
RFC 7807 body (`problem?: { type?, title?, detail?, instance? }`), so you can finally tell *the read floor
refused you* apart from *this key is `backendOwned`*, both 403s the contract deliberately words differently:

```ts
import { isPluginApiError } from '@mosaicast/plugin-sdk';

try {
  await ctx.api.put(`data/site/main/stats`, computed);
} catch (e) {
  if (isPluginApiError(e) && e.status === 403) {
    ctx.log('warn', e.problem?.detail ?? 'refused');
    return;
  }
  throw e;                                            // a real failure — do not swallow it
}
```

Use `isPluginApiError`, never `instanceof` — the error crosses a bundle boundary from the host, so it is not
guaranteed to share a constructor with anything in your bundle.

For a document nobody has written yet the host now answers **204**, and raw `ctx.api.get` resolves
`undefined` for it (before 0.16.0 it rejected with a 404). **`ctx.api.getOrNull<T>(path)`** resolves `null`
for both the 204 and a 404, so the reflexive
`.catch(() => undefined)` — which also swallows the 500, the 403 and a network failure — stops being
necessary:

```diff
-ctx.api.get<Stats>(path).then(setStats).catch(() => setStats(undefined));
+ctx.api.getOrNull<Stats>(path).then(setStats);   // null when absent; a real failure still rejects
```

### `ctx.docs` — a typed doc-store client (0.9.0)

The same endpoints `ctx.api` reaches, with path building, key validation and the `'self'`/`'site'`
shorthands done for you. Never `null` — every plugin has a doc store.

```ts
get<T>(target: DocTarget, key: string): Promise<T | null>       // null on absence, never a rejection
getMany<T>(type: Scope['type'], ids: string[], keys: string[])  // 0.16.0 — one request for a page of cards
  : Promise<Record<string, Record<string, T>>>                   //   misses absent; split over 100 for you
put<T>(target: DocTarget, key: string, value: T): Promise<void>
list<T>(target: DocTarget, { prefix?, page?, size? }): Promise<PagedDocs<T>>
remove(target: DocTarget, key: string): Promise<void>
```

`DocTarget` is a `Scope`, or one of two shorthands whose id is fixed: **`'self'`** → `data/user/me` and
**`'site'`** → `data/site/main`. That makes the most security-relevant convention in the whole contract —
per-user data goes in the `USER` scope, never in a key — the shortest thing to write:

```ts
await ctx.docs.put('self', `mark:${ctx.scope.id}`, marks);   // instead of building data/user/me/… by hand
```

A key failing `DOC_KEY_PATTERN` **throws synchronously at the call site**, with the pattern in the message,
instead of costing a round trip you then read a 400 body to explain. Everything else — the floors,
`backendOwned`, the 400 on an unknown scope, the 401 on an anonymous `user` request — is unchanged and still
the host's to enforce; `ctx.api` remains the escape hatch for anything `ctx.docs` does not cover.

**What the client remembers for you — guaranteed since 0.16.0**, per plugin and signed-in identity, for the
life of the page: identical `get`s in flight share one request; a miss (the 204) is remembered, `getMany`
misses included; your own `put`/`remove` forget the address they touched; **hits are never cached**; errors
are never remembered. So **delete any miss cache you wrote around `ctx.docs`** — it is redundant — and keep a
hit cache, if at all, no longer than a render: it hides writes made in other sessions. Before this, 98% of one
real session's plugin requests were "not set", one key asked 178 times.

```ts
const docs = await ctx.docs.getMany<Highlight>('episode', ctx.episodes.slice(0, 20), ['highlight']);
for (const slug of ctx.episodes) render(slug, docs[slug]?.highlight ?? null);
```

### `ctx.feeds` — episode display snapshots (0.9.0)

The frontend half of the Java `FeedAccess`. Never `null`, and **the one surface with no `readableBy` gate
of its own** — it returns host data the same visitor can already read from `/api/episodes/*`; it exists so
a plugin need not know that URL shape, the same argument `ctx.links` makes.

```ts
const cards = await ctx.feeds.displayMany(ctx.episodes.slice(0, 20));   // one request, not N
for (const slug of ctx.episodes) {
  const snap = cards[slug];
  if (!snap) continue;                       // filtered out for this visitor — normal, not an error
  render(slug, snap.title, resolveArtwork(snap));
}
```

```ts
display(slug: string): Promise<DisplaySnapshot | null>          // null: no snapshot, or not visible — indistinguishable
displayMany(slugs: string[]): Promise<Record<string, DisplaySnapshot>>   // clamped at DISPLAY_BATCH_LIMIT (200), not rejected
```

**Not authoritative and worth reading live rather than copying.** The host overwrites the snapshot on every
feed refetch — that propagation is the reason to call `ctx.feeds` per render instead of projecting it into
your own doc store on a schedule, which is the pattern this surface exists to retire. A missing key in
`displayMany`'s answer must be treated as "not shown to this visitor," never as a failure — it cannot be
used to enumerate episodes `ctx.episodes` did not already hand you.

**`description` is untrusted third-party HTML** — the feed's show notes verbatim, unsanitized by the host.
Never assign it to `innerHTML`. Show **`descriptionText`** (0.16.0, plain text, never absent) on a card or a
teaser; if you really need the markup, `ctx.sanitize(snap.description)`. Rendering `description` as React
text is the other bug: it prints the tags.

### `ctx.sanitize` — HTML you did not write (0.16.0)

```ts
notes.innerHTML = ctx.sanitize(snap.description);          // show notes
page.innerHTML  = ctx.sanitize(marked.parse(markdown));    // a podcaster's Markdown — AFTER rendering
```

The host's own feed-HTML policy (exported as `FEED_HTML_POLICY`): prose, links, lists, tables, images; drops
`<style>`, `<script>`, `<iframe>`, forms, every `style`/`srcset`/event-handler attribute and `javascript:`/
`data:` links; external links get `target="_blank" rel="noopener noreferrer nofollow ugc"`. Synchronous,
never `null`, no declaration.

**Never `DOMPurify.sanitize(html)` with defaults.** They stop scripts but allow `<style>` and `style=`, and
the contract requires `style-src 'unsafe-inline'` — so a stylesheet in someone else's HTML becomes a
full-viewport click-jacking overlay or attribute-selector CSS that leaks form values. Both the wiki and the
sample shipped exactly this. It also strips `class` and `data-*`: if **your own** generated markup needs them
(the wiki's link tokens do), replace it with nonce placeholder words before parsing, sanitize, then restore
your elements **into text nodes only** — see `mosaicast-plugin-wiki`'s `markdown.ts`. Test with
`sanitizeLikeHost` from `/testing` (needs jsdom).

### `ctx.tags` — the site's shared vocabulary (0.9.0)

`null` unless the manifest declares a `tags` block, mirroring `ctx.schema`/`ctx.blobs`.

```ts
const tags = ctx.tags;
if (!tags) return;                                                // no `tags` block declared

for (const t of await tags.all()) suggest(t.label, t.tag);        // the site's real vocabulary, not a free-text box
const href = ctx.links.feed('the-sample-cast', { tag: 'kraken' }); // and it links straight to the feed view
await tags.tagSubject(`page:${slug}`, 'Maritime Lore');            // your namespace, your key
```

```ts
all(): Promise<TagInfo[]>
episodesWith(tag): Promise<string[]>                 // filtered to what the caller may see
tagsOn(episodeSlug): Promise<string[]>
similarTo(tag, limit): Promise<TagInfo[]>             // co-occurrence, best first — advice, not a number to compare
subjectsWith(tag): Promise<string[]>                  // your own subjects only
tagsOnSubject(subjectKey): Promise<string[]>
tagSubject(subjectKey, tag): Promise<void>            // idempotent; needs only tags.readsVocabulary
untagSubject(subjectKey, tag): Promise<void>          // idempotent; removes an assignment, never the tag itself
tagEpisode(episodeSlug, tag): Promise<void>           // needs tags.writesEpisodes; rejects (403) without it
untagEpisode(episodeSlug, tag): Promise<void>         // needs tags.writesEpisodes; removes only YOUR assignment
```

`TagInfo = { tag, label, episodes, subjects }` — `tag` is the host's canonical key (send any spelling, it
converges), `label` is presentation kept from first use, `episodes` counts site-wide and `subjects` counts
only your own plugin's — you cannot see the size of a store you cannot read.

**Two writes that look alike and are not.** `tagSubject`/`untagSubject` touch only keys you invented in
your own namespace — `data.writableBy` is the whole story. `tagEpisode`/`untagEpisode` change the shell's
filter options *and* what core recommends beside that episode, so they need the second manifest flag and
reject with a `PluginApiError` (403) without it. What you may never do, and the host enforces rather than
merely discourages it: delete a tag from the vocabulary, rename one, or remove another writer's assignment
— `untagEpisode` only ever takes back your own row, even if the feed tagged the same episode the same way.

### `ctx.users` — turning UUIDs into people (0.13.0)

`null` unless the manifest declares an `identity` block. Fixes what `allUsers()`-built aggregates
couldn't do: a leaderboard has ids and documents, and no way to draw a person.

```ts
const dir = ctx.users;
if (!dir) return;                                    // no `identity` block declared

const board = await ctx.docs.get<{ userId: string; score: number }[]>('site', 'agg:leaderboard');
const people = await dir.resolve((board ?? []).map((row) => row.userId));
const byId = new Map(people.map((u) => [u.id, u]));  // match on id — the array is not index-aligned

for (const row of board ?? []) {
  const who = byId.get(row.userId);
  render(who?.displayName ?? 'Former listener', who?.avatarUrl);  // the row outlives its author
}
```

```ts
resolve(ids: string[]): Promise<UserRef[]>   // UserRef = { id, displayName, avatarUrl, role }
```

- **Absent, not redacted.** An unknown, erased or pseudonymised id is simply missing from the resolved
  array — there is no `null` entry and no tombstone, so **the result is not index-aligned with `ids`** and
  may be shorter. Never read `found[i]` and assume it answers `ids[i]`.
- **Resolves, does not enumerate.** No list call, and none is coming — you may only ask about ids you
  already hold through your own scope.
- **`avatarUrl` is always `/api/users/{id}/avatar`, always populated.** Every user has one (host-generated
  from the UUID when there's no provider picture). Put it straight in an `src`.
- **Store the UUID, resolve at render — never persist `displayName`.** The host cannot enforce this one:
  it provisioned your schema tables without ever learning which column holds a person, so §12.8's erasure
  guarantee cannot reach inside them. A name you copied outlives both the rename meant to shed it and the
  erasure meant to end it.
- `ctx.user` (the signed-in caller, not `ctx.users`) also gained `displayName` and `avatarUrl` this
  release — same fields, same rule: presentation, read at render, never kept.

### `ctx.notify` — putting a message in a user's inbox (0.14.0)

`null` unless the manifest declares a `notifications` block. **Most real use is on the backend**
(`ctx.notifier()` — see `backend.md`); reach for this one when a visitor's own action is what *other*
participants need to hear about.

```ts
const notify = ctx.notify;
if (!notify) return;                                  // no `notifications` block declared

const told = await notify.send(participants, {
  text: notifyText(catalogs, 'bingo.resolved', { episode: 'S02E04' }),  // your createPluginI18n catalogs
  link: `board/${boardId}`,                            // your own subtree; host-validated
});
if (told.length < participants.length) prunePartnerList(participants, told);  // read the return value
```

```ts
send(userIds: string[], msg: NotifyMessage): Promise<string[]>   // NotifyMessage = { text, link? }
```

`notifyText(catalogs, key, params?)` builds the per-locale `text` map from the same catalogs you already
pass `createPluginI18n`, interpolating each language's template. A locale whose catalog lacks the key is
**left out** rather than filled with the literal key; if `en` itself is missing, it falls back to
`catalogs.en?.[key] ?? key` rather than throwing (unlike a hand-built map missing `en`, which the client
refuses before sending — see below).

**This is the one plugin surface that writes into somebody else's experience.** Two bounds the host holds,
not you:

- **Eligibility** — only a user id your plugin already holds `USER`-scope data for. An ineligible or erased
  recipient is silently absent from the resolved array, never a rejection — **the return value is the only
  way to see a partial send**, and a component that ignores it notifies nobody while looking healthy.
- **Rate limits are the host's**, not a counter you keep. `send` rejects with a `PluginApiError`: **429**
  when the send cap is exhausted (hold the batch for a later render/tick, do not drop it), **400** when
  `msg`/`link` is one the host will not draw (a code bug — will fail identically next time).

**`text` must contain `en`** or the call is refused before it reaches the network — the one language a site
can never switch off, and the only fallback a reader is guaranteed to understand. **`link` is internal
only**: a bare core path, or a subpath of your own `/p/<pluginId>/` (with or without the prefix); anything
with a scheme, `//`, a backslash, `:`, `..`, or another plugin's namespace is refused.

There is **no read side** — you cannot list, count or mark an inbox, or learn whether anyone opened what
you sent. Nothing here reaches email.

### `ctx.locale.available()` / `.content()` and `ctx.translation` (0.10.0 / 0.11.0)

Two lists, deliberately different: `available()` is what the shell can **render** in — mirror it if you
build a language switcher. `content()` is what the admin permits text to be **authored** in — build an
editor's tabs from this one, never from `available()`, since a site can require content in a language its
UI does not offer.

```ts
for (const l of ctx.locale.content()) addTab(l.code, l.nativeName);   // an editor's language tabs
```

`ctx.translation` is `null` for **either of two indistinguishable reasons**: your manifest does not declare
`"external": { "kinds": ["translation"] }`, or the operator configured no provider (every site, by default).
Check the manifest before the admin panel, and **never cache the handle** — the operator half can change
under a running plugin, so read `ctx.translation` at the point of use:

```ts
if (ctx.translation?.available()) {
  try {
    const { text } = await ctx.translation.translate({ text: body, to: targetLocale });
    render(text, { draft: true });                     // machine output is a draft — show it as one
  } catch (e) {
    if (isPluginApiError(e) && e.status === 403) { /* below external.usedBy */ }
    else { /* 409 no-provider/misconfigured, 429 rate-limited, 503 busy, 504 timeout */ }
  }
}
```

`format` is `'text'` (default) or `'html'` — **markdown is neither**; send it as text and expect links and
code fences to come back mangled, since a translator does not know they are markup. A non-`null` handle is
still not permission: `translate()` 403s a visitor below `external.usedBy`, so disable your button on
`available()` rather than letting a click fail.

### `matchRoute` — the prefix matcher for page plugins (0.9.0)

```ts
import { matchRoute } from '@mosaicast/plugin-sdk';

const match = matchRoute(ctx.route.path, ['', 'article/:slug', '_search/:term']);
// whole-path match with `:param` capture; null when nothing matches — never a `startsWith` that also
// matches "article-archive"
```

### Per-user data

```ts
import { SELF_SCOPE_ID } from '@mosaicast/plugin-sdk';
await ctx.api.put(`data/user/${SELF_SCOPE_ID}/mark:${ctx.scope.id}:b3`, { marked: true });
```

`SELF_SCOPE_ID` is the literal `me`, resolved server-side from the session. Any other `user` id is a **400**
(never a silent substitution); anonymous is a **401**. The partition is **flat** — one per user, not per user
*and* entity — so the entity goes in the key. `DataScopeType` (`Scope['type'] | 'user'`) is a separate type
from `Scope` on purpose: `user` addresses storage, never a slot, and never appears in `ctx.scope`.

There is no TS counterpart to the backend's `ctx.allUsers()` (declared as `data.readsAllUsers`), deliberately. Build leaderboards on the backend and read
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
accentText→--mc-accent-text   (0.16.0; optional in ctx.theme, always on :root from core)
```

**Text, links and focus rings use `--mc-accent-text`; `--mc-accent` is for fills only**, paired with
`--mc-accent-contrast`. The accent is the admin's unchecked seed — a pale one measured 1.12:1 as link text;
`--mc-accent-text` is the same colour clamped to WCAG AA against both `--mc-bg` and `--mc-surface`. A quick
audit: every `color: var(--mc-accent)` and `outline: … var(--mc-accent)` in your CSS is a bug.

### `--mc-icon-*` — the shell's icon set (core 0.6.15)

A second published token family, same channel as the colour tokens above: custom properties that inherit
across the shadow boundary, so a Web Component uses the shell's icons with **no SDK import, no
`platformApi` bump and no version skew** — a plugin built against an older SDK picks up an icon added to
core the day it lands.

**Consume as a mask, never as a background image**, or the icon ignores your theme:

```css
/* Right — the icon takes YOUR colour and re-themes with everything else */
.icon {
  mask-image: var(--mc-icon-close);
  mask-size: contain;
  mask-repeat: no-repeat;
  background: currentColor;
  width: 1em;
  height: 1em;
}

/* Wrong — background-image renders the icon in its own baked colour, ignoring light/dark and your accent */
.icon { background-image: var(--mc-icon-close); }
```

The published set is **larger than what core itself draws**, on purpose: a plugin builds against a
*released* core, so an icon that is not already published is one its author cannot add without waiting for
a core release. Coverage includes charts/percentages, documents/journal/history, board/trophy/dice, and
`choice-single`/`choice-multi` for poll-style UIs. Published names are a contract like the colour tokens —
**add freely, rename never**.

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

**Every subpath under a `page` slot answers `200` until you implement `PageRouteProvider`** (backend,
0.9.1) — a mistyped slug and a page you deleted last year both render your not-found view inside a real
`200`. Nothing here to do on the frontend beyond keeping your not-found view honest; the backend reference
covers the provider itself.

**A `page` plugin with no `nav[]` entry gets one default menu item at its root**, labelled with the
manifest's `name` — declare `nav` (see `manifest.md`) only once you have more than one entrance worth
naming (a wiki's front page, a random article, a podcaster-only "new page" form).
