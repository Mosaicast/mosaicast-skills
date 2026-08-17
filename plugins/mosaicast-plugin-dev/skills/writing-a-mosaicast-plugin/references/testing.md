# Tests

Required by every repo's `docs/BRIEF.md` DoD (ARCHITECTURE §13.5). No core, no database, no browser.

## Backend — `dev.mosaicast.plugin.testkit.*`

`testImplementation("dev.mosaicast:plugin-testkit:0.7.1")` — the Java fakes are unchanged since 0.6.0.

| Fake | Notes |
|---|---|
| `FakePluginContext` | `store()` narrows to `InMemoryDocStore` and `logger()` to `RecordingLogger`, so no casts. `onSchedule` runs the task **synchronously and immediately**; `scheduledCount()` counts registrations. |
| `InMemoryDocStore` | `asUser(uuid)`, `docsOf(uuid)`, `withBackendOwned(...)`. Optional `ObjectMapper` ctor — Jackson 3, so `JsonMapper.builder().build()`. |
| `FakeSchemaStore` | `new FakeSchemaStore(ns).withEntity("page", "slug", "title").withFulltext("page", "markdown")` — enforces the same declaration the host does. |
| `FakeFeedAccess` | `withDisplay(refId, snapshot)`; `display(unknownId)` throws; `episodesIn(Scope.user())` is empty. |
| `MapPluginConfig` | `new MapPluginConfig().with("refreshIntervalMinutes", 5)` |
| `RecordingLogger` | `events()` / `events(Level)` / `clear()`; formatted messages, throwable captured separately. |

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

## Frontend — `@mosaicast/plugin-sdk/testing`

Mount the element with `makeMockCtx(overrides)` and assert on the DOM.

```ts
import { makeMockCtx, makeMockConsent, makeMockSchema, DEFAULT_THEME } from '@mosaicast/plugin-sdk/testing';

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
site scope with id `main`, no episodes, anonymous user, **`schema: null`**, empty filter, player at 0s, empty
route, `en`, `progress → null`, `DEFAULT_THEME`, and a consent double that **denies everything except
`necessary`**. `episodeLabels` is absent by default and `schema` is `null` by default on the same argument: a
component written against a value that is always there never handles the case where it is not.

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
shape and new members like `schema` / `route.navigate` across SDK bumps, and a hand-rolled fake breaks on all
of them every time the contract moves. If you do hand-roll one, every `onChange` must return a function,
`consent` must implement all four methods, and `route` must carry `navigate`.

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

Pair it with the CI `package` job's drift guard, which asserts `dist/plugin.json`'s `platformApi` equals the
`plugin-api:<version>` the backend actually compiled against.

## Running

```
cd backend  && ./gradlew test
cd frontend && npm test && npm run typecheck    # Vite does not type-check; tsc --noEmit is the only thing that does
```
