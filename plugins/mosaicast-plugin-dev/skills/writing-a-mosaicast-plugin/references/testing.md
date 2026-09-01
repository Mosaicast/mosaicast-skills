# Tests

Required by every repo's `docs/BRIEF.md` DoD (ARCHITECTURE §13.5). No core, no database, no browser.

## Backend — `dev.mosaicast.plugin.testkit.*`

`testImplementation("dev.mosaicast:plugin-testkit:0.11.0")`

| Fake | Notes |
|---|---|
| `FakePluginContext` | `store()` narrows to `InMemoryDocStore` and `logger()` to `RecordingLogger`, so no casts. `onSchedule` runs the task **synchronously and immediately**; `scheduledCount()` counts registrations. `withTags(Tags)` / `withLocales(Locales)` / `withTranslation(Translation)` are **chaining mutators**, not constructor parameters — the constructor list stopped growing at five arguments (`store, config, feeds, schema[, blobs]`) on purpose (0.9.0). |
| `InMemoryDocStore` | `asUser(uuid)`, `docsOf(uuid)`, `withBackendOwned(...)`. Optional `ObjectMapper` ctor — Jackson 3, so `JsonMapper.builder().build()`. |
| `FakeSchemaStore` | `new FakeSchemaStore(ns).withEntity("page", "slug", "title").withFulltext("page", "markdown")` — enforces the same declaration the host does. |
| `InMemoryPluginBlobs` | `withLimits(maxFile, quota)`, `withMimeTypes(Set.of(…))`, `rejectContent("bad.png")`, plus `usedBytes()` / `size()` / `bytesOf(ref)`. Refuses what the host refuses. |
| `FakeFeedAccess` | `withDisplay(refId, snapshot)`; `display(unknownId)` throws; `episodesIn(Scope.user())` is empty. |
| `FakeTags` (0.9.0) | `withEpisodeWrites()` stands in for `tags.writesEpisodes`, **off by default** so the refused branch gets exercised; `withFeedTag(slug, tag)` seeds a row this plugin may read but must not remove. Canonicalises tags exactly as the host does (`FakeTags.canonical(tag)` is exposed to assert against). |
| `FakeLocales` (0.10.0) | `FakeLocales.englishOnly()`, `.withUi(codes...)` (also adds to content), `.withContent(codes...)`, `.withDefault(code)`. |
| `FakeTranslation` (0.10.0) | `FakeTranslation.marking()` succeeds and marks output `"[to] text"`; `FakeTranslation.failing(reason)` always throws `TranslationException`; `.unavailable()` makes `available()` lie `false` while `translate` still works — the race the real host has between an admin's removal and your next call. |
| `MapPluginConfig` | `new MapPluginConfig().with("refreshIntervalMinutes", 5)` |
| `RecordingLogger` | `events()` / `events(Level)` / `clear()`; formatted messages, throwable captured separately. |
| `SearchProviderHarness` (0.9.0) | `new SearchProviderHarness(provider).search(query)` calls the provider **once per `Role`, anonymous included**, and returns a `SearchResults` with `.forRole(role)`, `.titles(role)` and `.leakedToAnonymous(subpath)` — the one assertion this extension point's unusual access rule exists for. |
| `UserDataHandlerHarness` (0.9.0) | `.eraseTwice(userId)` calls `eraseUser` twice as a retry would and fails with a clear message if the second call throws where the first succeeded; `.export(userId)` calls `exportUser` (call before `eraseTwice`, not after). |
| `PageRouteProviderHarness` (0.9.1) | `.check(subpaths...)` always probes the **root** (`""`) whether you list it or not, records a throw as the `200` the host would still serve, and returns a `RouteAnswers` with `.serves(subpath)`, `.servesRoot()`, `.notFound()`, `.served()`, `.threw(subpath)`. |

### Testing an aggregate

Seed what a frontend would have written, then assert on the aggregate:

```java
var ctx = new FakePluginContext();
ctx.store().asUser(alice).put(Scope.user(), "mark:ep-1:b3", Map.of("marked", true));
ctx.store().asUser(bob).put(Scope.user(), "mark:ep-1:b3", Map.of("marked", true));

plugin.register(ctx);                                  // onSchedule fires synchronously

assertEquals(2, ctx.store().queryAcrossUsers("mark:").size());
assertTrue(ctx.store().get(Scope.episode("ep-1"), "leaderboard", Board.class).isPresent());
```

`asUser` has **no production counterpart** — no real `DocStore` can write into a user's partition. It exists
so a test can stand in for the client.

### Testing file storage

`FakePluginContext` gained a **five-argument** overload taking a `PluginBlobs`; the four-argument form is
untouched and still means "no file storage" (`ctx.blobs()` → `null`).

```java
var blobs = new InMemoryPluginBlobs()
        .withLimits(5 * 1024 * 1024, 64 * 1024 * 1024)
        .withMimeTypes(Set.of("image/png"))
        .rejectContent("liar.png");                       // stands in for the host's content sniffing
var ctx = new FakePluginContext(new InMemoryDocStore(), new MapPluginConfig(),
        new FakeFeedAccess(Map.of()), null, blobs);
```

It **enforces the ceilings and the allow-list** — a plugin that only ever meets an accepting fake discovers
its 20 MB upload path on a real install. Defaults are deliberately small (1 MB per file, 8 MB total, common
raster images), so a test moving real quantities has to say so. All four refusals surface as
`IllegalArgumentException`, as they do from the host's `PluginBlobs`.

It does **not** sniff content — reimplementing that would be a second, diverging copy of a security rule —
so name the file that should be refused with `rejectContent(...)`. Refs are sequential (`blob-1`, `blob-2`,
…) so an assertion can name one; real refs are opaque UUIDs and a plugin must never build one.

### Testing `backendOwned`

`withBackendOwned` makes the test kit enforce what the host enforces: the `asUser(...)` view is the client
(refused with `IllegalStateException`), the base store is your backend (never refused). A malformed pattern
throws `IllegalArgumentException` **when you declare it**, so a typo in a security declaration fails in your
tests rather than at load.

```java
var store = new InMemoryDocStore().withBackendOwned("stats", "agg:*");
store.put(Scope.site(), "stats", computed);                          // backend: fine
assertThrows(IllegalStateException.class,
        () -> store.asUser(mallory).put(Scope.site(), "stats", forged));
store.asUser(mallory).put(Scope.user(), "mark:ep-1", ok);            // USER is exempt, even under "*"
```

A hand-rolled `DocStore` fake must now implement `queryAcrossUsers` — switching to `InMemoryDocStore` is the
cheaper fix.

### Testing `tags`, and the two extension points a request can reach

```java
var tags = new FakeTags().withFeedTag("kraken-ep", "maritime");
var ctx = new FakePluginContext(new InMemoryDocStore(), new MapPluginConfig(),
        new FakeFeedAccess(Map.of()), null).withTags(tags);

plugin.register(ctx);
assertThrows(UnsupportedOperationException.class, () -> tags.tagEpisode("kraken-ep", "lore")); // no writesEpisodes yet

tags.withEpisodeWrites();
tags.tagEpisode("kraken-ep", "lore");
tags.untagEpisode("kraken-ep", "maritime");                 // refuses to remove the feed's own row instead
assertEquals(List.of("maritime"), tags.tagsOn("kraken-ep")); // your untag of the feed's tag was a no-op
```

```java
var results = new SearchProviderHarness(new WikiSearch(pages)).search("kraken");
assertTrue(results.forRole(Role.ADMIN).size() >= results.forRole(null).size());
assertFalse(results.leakedToAnonymous("_admin/draft-page"));   // the one thing this harness exists to catch
```

```java
new UserDataHandlerHarness(new WikiUserData(pages)).export(userId);   // before erasure — an export in its own right
new UserDataHandlerHarness(new WikiUserData(pages)).eraseTwice(userId);   // must survive a retry
```

```java
var routes = new PageRouteProviderHarness(new WikiRoutes(pages))
        .check("glossary/kraken", "glossary/tpyo", "_search/kraken");
assertTrue(routes.servesRoot());                 // probed even though it was never listed above
assertEquals(List.of("glossary/tpyo"), routes.notFound());
assertTrue(routes.failures().isEmpty());
```

## Frontend — `@mosaicast/plugin-sdk/testing`

Mount the element with `makeMockCtx(overrides)` and assert on the DOM.

```ts
import {
  makeMockCtx, makeMockConsent, makeMockSchema, makeMockBlobs, makeMockTags, makeMockDocs, makeMockFeeds,
  makeMockTranslation, apiError, flushMockApi, DEFAULT_THEME,
} from '@mosaicast/plugin-sdk/testing';

const ctx = makeMockCtx({
  scope: { type: 'episode', id: 'ep-1' },
  user: { id: 'u1', role: 'fan' },
  apiResponses: { 'data/episode/ep-1/leaderboard': { rows: [] } },
});
// …mount, then:
expect(ctx.api.calls).toContainEqual({ method: 'get', path: 'data/episode/ep-1/leaderboard' });
expect(ctx.logs).toEqual([]);
```

`MockPluginContext` = `PluginContext` + `api.calls` / `api.responses` + `logs` + `navigations`. Defaults:
site scope with id `main`, no episodes, anonymous user, **real** `docs`/`feeds` doubles (every plugin has a
doc store and can read snapshots — there is no "declared it or not" case for these two), **`tags: null`**,
**`schema: null`**, **`blobs: null`**, **`translation: null`**, `locale.available()`/`.content()` →
English-only, empty filter, player at 0s, empty route, `en`, `progress → null`, `DEFAULT_THEME`, and a
consent double that **denies everything except `necessary`**. `episodeLabels` is absent; `tags`, `schema`,
`blobs` and `translation` are all `null` on the same argument: a component written against a value that is
always there never handles the case where it is not, and most plugins declare none of the three manifest
blocks that turn them non-`null`.

### `apiError(status, problem?)` and `flushMockApi(client)` (0.9.0)

```ts
const ctx = makeMockCtx({ apiResponses: { 'data/site/main/stats': apiError(403, { detail: 'backendOwned' }) } });
// …mount, click the button that writes computed stats…
await flushMockApi(ctx.api);            // waits for calls AND the microtask hops a component adds after
expect(ctx.logs).toContainEqual({ level: 'warn', message: expect.stringContaining('backendOwned') });
```

`apiError` is a canned rejection you register in `apiResponses` in place of a success value — it is what a
test for the 403/404/500 branch of a component reaches for. `flushMockApi` is what removes the classic
footgun: the mock resolves before a component's own `.then(setState)` runs, so a bare
`await Promise.resolve()` covers one microtask hop and not two, and the symptom is an assertion that fails
*only sometimes* depending on how many hops the component happens to take.

`ctx.links` is the exception: it is a **real implementation**, not a stub, because these are pure string
builders that can simply be right. It mirrors the host's URL shapes and core has a test pinning the two
together, so `expect(href).toBe(ctx.links.episode('ep-1', { t: 724 }))` is a fair assertion.

`makeMockConsent(initial)` gives `grant` / `revoke` / `requests` / `autoGrantOnRequest` for testing the
click-to-load path.

### `route` is the one override that merges (0.7.1)

```ts
const ctx = makeMockCtx({ route: { path: 'kraken' } });   // navigate still records, onChange still works
// …mount, click a link…
expect(ctx.navigations).toEqual([{ subpath: 'glossary/kraken', replace: false }]);
```

Every other member is all-or-nothing; `route` merges over the default because pinning a subpath is the common
case. On **0.7.0** a hand-built override had to spell out `navigate` too (`PluginRoute` gained it as a
required member) — that is the release's only compile break, and only `tsc --noEmit` catches it. Supplying
your own `navigate` still opts out of the recorder, which is the one case where `navigations` stays empty.

The double has no router and no URL, so `navigate` does **not** move `route.path`. Assert on `navigations`;
drive a route change by rendering again with a different `path`.

### `makeMockDocs(initial)` / `makeMockFeeds(snapshots)` / `makeMockTags(opts)` (0.9.0)

```ts
const docs = makeMockDocs({ 'data/user/me/marks': { b3: true } });
const ctx = makeMockCtx({ docs });
await mount(ctx);
expect(docs.stored['data/user/me/marks']).toEqual({ b3: true, b4: true });
```

`makeMockDocs` **validates keys the way the real client does** — a key with a `/` or over 200 characters
throws here instead of first surfacing as a production 400. `stored` is keyed `"<partition>/<key>"`
(`docPath('self')` → `data/user/me`, `'site'` → `data/site/main`), directly inspectable.

```ts
const feeds = makeMockFeeds().withDisplay('kraken', { title: 'The Kraken', description: '' });
const ctx = makeMockCtx({ feeds, episodes: ['kraken', 'gated'] });
await mount(ctx);
expect(root.textContent).toContain('The Kraken');   // and renders nothing for 'gated' — unregistered ⇒ absent
```

`makeMockFeeds` resolves an unregistered slug to `null` (or drops it from a `displayMany` batch) rather than
throwing — exactly what the host does for an episode this visitor may not see. `requested` records every
slug asked for, batched calls included, and `displayMany` clamps at `DISPLAY_BATCH_LIMIT` the same way the
host does.

```ts
const tags = makeMockTags({ writesEpisodes: false }).withFeedTag('kraken', 'maritime');
const ctx = makeMockCtx({ tags });
await mount(ctx);
await expect(tags.tagEpisode('kraken', 'lore')).rejects.toThrow();   // writesEpisodes off by default
```

`makeMockTags` **refuses what the host refuses**: episode writes throw unless `writesEpisodes: true` is
passed, and `untagEpisode` only ever removes this plugin's own row, so a tag seeded with `withFeedTag`
survives your call — same as production. Canonicalisation (trim, collapse whitespace, casefold) is applied
the way the host applies it, so a test writing `'Maritime '` and reading `'maritime'` passes here for the
same reason it passes against core.

### `makeMockTranslation(opts)` (0.10.0)

```ts
const ctx = makeMockCtx({ translation: makeMockTranslation() });
// or, for a refusal path:
const refused = makeMockTranslation({ fail: apiError(403, { detail: 'below external.usedBy' }) });
```

The default `translate` **marks** the text rather than faking a real translation — `"[nl] Hello"` — so an
assertion pins *that the component asked for Dutch*, not a plausible-looking string that could hide the
wrong target language. `requests` records every call. **`ctx.translation` defaults to `null`** in
`makeMockCtx` — test that path too, since it is the one more plugins get wrong than the happy path.

### `makeMockBlobs(opts)` (0.8.0)

```ts
const blobs = makeMockBlobs({ mimeTypes: ['image/png'], rejectContent: ['liar.png'] });
const ctx = makeMockCtx({ blobs });

await mount(ctx);
expect(blobs.uploads[0]).toMatchObject({ filename: 'diagram.png', mime: 'image/png' });
expect(blobs.stored).toHaveLength(1);
expect(blobs.removals).toEqual([]);
```

A `BlobClient` double storing in memory, recording `uploads` / `removals` / `stored`, and **refusing what
the host refuses**: the type allow-list, the per-file ceiling and the quota, in that order. Defaults are
small on purpose (1 MB per file, 8 MB total, common raster images). Refusals **reject the promise** — write
the test for that path, since a component that never handles one shows its first refusal to a podcaster.

Like the Java double it does not read file formats; `rejectContent: ['liar.png']` stands in for the host's
sniffing. `urlFor` returns the host's URL shape, `list` is newest-first, and `remove` is idempotent.

**Test the `null` path too.** `ctx.blobs` is `null` for a plugin whose manifest declares no `blobs` block,
which is what `makeMockCtx()` gives you by default — a component should degrade, not throw.

### `makeMockSchema(rows)` (0.7.0)

```ts
const schema = makeMockSchema({
  page: [{ id: 1, slug: 'kraken', title: 'The Kraken', markdown: 'a big squid' }],
});
const ctx = makeMockCtx({ schema, route: { path: 'kraken' } });
await mount(ctx);
expect(schema.queries[0]).toMatchObject({ method: 'search', entity: 'page' });
```

A `SchemaClient` answering from plain arrays, with `queries` recorded and `rows` mutable. An entity absent
from `rows` **rejects** the way the host 404s an undeclared one (every method is `async`, so it reaches your
component as a rejection, not a synchronous throw).

Its `search` is a **case-insensitive substring match, not Postgres full-text search** — no stemming, no
`ts_rank` ordering. Use it to prove your component renders hits and handles none; prove the searching itself
against a live host. It does share the host's one rule that matters — **empty text matches nothing** — and
applies `where`/`orderBy`/`page`/`size` faithfully (`like` anchors `%` the way the host does).

Prefer this over a hand-rolled `PluginContext`: it stays in sync with `Unsubscribe` returns, the `ConsentApi`
shape and new members like `schema`, `blobs`, `docs`, `feeds`, `tags`, `translation`, `links`,
`locale.available/content` and `route.navigate`/`route.query`/`route.hash` across SDK bumps, and a
hand-rolled fake breaks on all of them every time the contract moves. If you do hand-roll one, every
`onChange` must return a function, `consent` must implement all four methods, `route` must carry `navigate`
plus `query`/`hash`, and `links`/`docs`/`feeds` must be present (they are non-nullable — unlike `schema`,
`blobs`, `tags` and `translation`).

## The manifest ↔ bundle contract test

Copy this from the sample — it is the cheapest guard against the failure that actually happens (a version
bumped in one file, or a `backendOwned` pattern that swallows a key your own UI writes):

```ts
import { PLATFORM_API_VERSION } from '@mosaicast/plugin-sdk';
import manifest from '../../plugin.json';

it('declares the SDK it was built against', () => {
  expect(manifest.platformApi).toBe(PLATFORM_API_VERSION);
});

it('reserves no key the client writes', () => {
  const covered = (k: string) => (manifest.data.backendOwned ?? []).some(p =>
    p === '*' || (p.endsWith('*') ? k.startsWith(p.slice(0, -1)) : k === p));
  for (const key of ['highlight', 'settings', 'fav:ep-1']) expect(covered(key)).toBe(false);
});
```

If you declare `blobs`, add the same cheap guard there — a manifest that asks for a type the platform never
stores loads fine and refuses every upload:

```ts
it('asks for no unstorable type', () => {
  expect(manifest.blobs?.mimeTypes ?? []).not.toContain('image/svg+xml');
});
```

If you declare `tags`, pin that it asks for at least one of the two things it can:

```ts
it('tags block asks for something', () => {
  expect(manifest.tags?.readsVocabulary || manifest.tags?.writesEpisodes).toBe(true);
});
```

If you declare `external`, the cheap guard is on `usedBy` rather than on `kinds` (an empty `kinds` already
fails to load, loudly) — catch an accidental `anonymous` floor on a metered call before it ships:

```ts
it('does not open translation to anonymous visitors', () => {
  expect(manifest.external?.usedBy ?? 'podcaster').not.toBe('anonymous');
});
```

Pair all of it with the CI `package` job's drift guard, which asserts `dist/plugin.json`'s `platformApi`
equals the `plugin-api:<version>` the backend actually compiled against.

## Running

```
cd backend  && ./gradlew test
cd frontend && npm test && npm run typecheck    # Vite does not type-check; tsc --noEmit is the only thing that does
```
