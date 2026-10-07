# Backend — Java, PF4J extension

Implement `PluginBackend.register(PluginContext ctx)` on an `@Extension` class in package
`dev.mosaicast.plugin.<name>.*`. Compile against the SDK only — never core — and stay Spring-free.

## `PluginContext` — twelve accessors, exactly

```java
DocStore store();                                // the generic doc store
SchemaStore schema();                            // null unless the manifest declares schema entities
PluginBlobs blobs();                             // null unless the manifest declares a blobs block (0.8.0)
Tags tags();                                     // null unless the manifest declares a tags block (0.9.0)
Users users();                                    // null unless the manifest declares identity (0.13.0)
Notifier notifier();                              // null unless the manifest declares notifications (0.14.0)
CrossUserStore allUsers();                        // null unless data.readsAllUsers: true (0.16.0)
PluginConfig config();
FeedAccess feeds();
Locales locales();                                // never null — the site's languages (0.10.0)
Translation translation();                        // null unless external.kinds declares "translation" AND
                                                   // an admin configured a provider (0.10.0; gated 0.11.0)
org.slf4j.Logger logger();                       // already named "plugin.<pluginId>"
void onSchedule(Duration every, Runnable task);          // fixed period, captured once at register()
void onSchedule(Supplier<Duration> every, Runnable task); // period RE-READ before every tick (0.15.0)
default void onEpisodeReleased(Consumer<String> listener); // a planned episode just became RELEASED (0.18.0)
default void onEpisodePhaseChanged(BiConsumer<String, EpisodePhase> listener); // any write-driven phase change, incl. becoming null on cancel (0.19.0)
```

**It's `notifier()`, not `notify()`.** ARCHITECTURE §7.4 specifies the latter, and it cannot compile:
`Object.notify()` is `final`, so no interface may declare that name. The TypeScript half is `ctx.notify`,
exactly as specified — the two differ because one of them has to.

There is **no `ctx.log(...)` in Java** (that is the TypeScript context) and no route-registration API — a
plugin does not author HTTP endpoints.

**`translation()`'s gate is Java-only-half of 0.11.0's change.** Declaring `external.kinds` is the whole
story on the backend — there is no role floor here, because `register()` and `onSchedule` have no visitor.
`external.usedBy` governs only the browser client (`ctx.translation` in TypeScript); it is ignored for this
accessor.

`logger()` is the one to use. A logger you build yourself with `LoggerFactory.getLogger(...)` falls outside
the `plugin.` prefix, so the host cannot attribute it to you or show it in the admin log viewer. The host
persists `info`+ and surfaces `warn`+ there; a tight loop gets rate-limited, not stored.

Both `onSchedule` overloads are fixed-rate on a small shared pool, wrap each tick in a try/catch, and are
**skipped entirely while the plugin is disabled**. `onSchedule(Duration, Runnable)` is now a **`default`
method** delegating to the `Supplier` form — implementing only the old one no longer compiles for a
hand-rolled `PluginContext` (not that you should be hand-rolling one; see `testing.md`).

**Use the `Supplier` overload whenever the period comes from `ctx.config()`.** The `Duration` form reads
the value once, during `register()`, and holds it for the life of the process — an operator saves a new
interval, the admin form reports success, and the plugin runs at the old cadence until the host restarts:

```diff
- ctx.onSchedule(Duration.ofSeconds(ctx.config().get("ingestIntervalSeconds", Integer.class, 60)), this::ingest);
+ ctx.onSchedule(() -> Duration.ofSeconds(ctx.config().get("ingestIntervalSeconds", Integer.class, 60)), this::ingest);
```

The host calls the supplier once at registration (validated strictly — a non-positive value here is
rejected outright) and again **before every tick**, rescheduling when the answer differs — a config change
takes effect within one old period, not at the next restart. The supplier runs on the scheduler thread, so
it must be **cheap and side-effect-free**: reading a field or `ctx.config()` is right, querying or blocking
is not. A supplier returning `null` or a non-positive `Duration`, or one that throws, leaves the task on
its last valid period rather than dropping it — logged, never silent. The host may clamp a very short
period to a floor it owns; treat the value you supply as a request, the same as the manifest's other
numbers. Keep the plain `Duration` overload wherever the cadence is genuinely fixed — it is not deprecated.

### `onEpisodeReleased` — hearing about a planned episode going live (0.18.0)

A `default` no-op the host overrides, called with the episode's public slug once a **planned** episode binds
to its feed item (or a confirmed suggestion, or a manual match) and its status flips `PLANNED → PUBLISHED`:

```java
ctx.onEpisodeReleased(this::open);
ctx.onSchedule(Duration.ofMinutes(15), () ->
    ctx.feeds().episodesIn(Scope.site()).stream()
        .filter(slug -> ctx.feeds().display(slug).phase() == EpisodePhase.RELEASED)
        .filter(this::stillClosed)
        .forEach(this::open));
```

**Best effort, and only a shortcut — never the only path.** The host calls it once per release, after the
binding transaction commits, on a host thread; a listener that throws is caught and logged against this
plugin, and the host moves on to the next one. The event is **not durable and not replayed**: a plugin that
was stopped, restarting or being upgraded at that exact moment never hears about that release. Always also
reconcile on an `onSchedule` task, checking `phase()` rather than trusting the event alone — the listener
should be idempotent, since both paths may act on the same release.

**Not fired for every new episode** — only a release that was *planned first*. An episode arriving from the
feed already released, with no prior plan, never triggers it; neither does a planned episode becoming
`UPCOMING` (that's the clock passing `announceAt`, and nothing is written). There is no frontend equivalent
event: the shell hands a mounted component a new `ctx` when the phase changes, so a component renders from
`ctx.episode.phase` and lets `defineMosaicastElement` re-render it.

Test it with `FakePluginContext.fireEpisodeReleased(slug)`, paired with
`FakeFeedAccess.withPhase(slug, EpisodePhase.RELEASED)` so a lookup inside the listener sees the released
state — see `testing.md`.

### `onEpisodePhaseChanged` — the direction `onEpisodeReleased` can't cover (0.19.0)

`onEpisodeReleased` only ever fires on one direction: an episode becoming visible. `onEpisodePhaseChanged`
fires on **every** write-driven phase change, which is what you need if your backend publishes anything on
a schedule rather than computing it per request:

```java
ctx.onEpisodePhaseChanged((slug, phase) -> {
    if (phase == null || phase == EpisodePhase.PLANNED || phase == EpisodePhase.WITHDRAWN) {
        republishIndex();          // drop it from what anonymous readers see, right now
    }
});
```

**Why this exists, specifically:** a podcaster can push an announced episode's `announceAt` back into the
future, and it becomes `PLANNED` again — hidden from everyone below podcaster **at once**. Anything you
compute **per request** (a sitemap, `ShareMetadataProvider`, `PageRouteProvider`, `SearchProvider`) is
already correct, because each of those calls `FeedAccess.display(slug)` fresh every time. Anything you
**stored** on a schedule — a site-wide index, a count, a teaser card — keeps naming that episode to
anonymous readers until your next tick, otherwise. This hook closes that gap.

**When it fires:** the host compares the phase just before a write with the phase just after, at the same
instant, and calls this only when they differ — an announce, an `announceAt` edit in *either* direction, a
release, a withdrawal, a withdrawn episode coming back, and a cancellation. On cancellation the listener
receives `phase == null`: the episode no longer exists, its episode-scoped documents are already gone with
it, but anything your *other* scopes say about it (that site-wide index again) is yours to drop.
**Deleting a feed fires it too, once per episode, with `phase == null` (core 0.8.1)** — the same "no longer
exists" signal a cancelled plan already gets, so one `null`-handling branch covers both.


**Expect concurrent calls — this is not "one at a time," and deleting a feed is exactly where it bites.**
Each event is its own task on its own thread, so a write touching many episodes reaches your listener as
many calls *at once*, in no guaranteed order, every one of them already seeing every episode gone. A
listener that republishes something expensive on each call (a site-wide index, say) should **coalesce**:
let one pass run and have the rest mark it dirty, rather than recomputing once per slug. **Guard that
coalescing with a `java.util.concurrent.locks.ReentrantLock`, not `synchronized`** — the host's threads are
virtual, and on Java 21 a virtual thread blocked on a `synchronized` monitor pins its carrier thread, which
is the exact platform-thread-starvation failure virtual threads exist to avoid. This isn't a style
preference; `synchronized` here is a real production incident waiting on enough concurrent episodes.
**`mosaicast-plugin-sample` 2.20.1 itself still coalesces its recompute with `synchronized (RECOMPUTE_LOCK)`**
(`recomputeHighlightStats`) — written the same day as the SDK 0.19.0 upgrade, before this javadoc landed at
0.19.1. Don't copy that part of it; it's exactly the pattern this rule exists to stop.

**When it does not fire:** the clock passing `announceAt` on its own involves no write, so nothing calls
this for a `PLANNED → UPCOMING` transition — that direction only makes an episode *more* visible, and being
briefly late about it is harmless. Reconcile that direction on your schedule, same as always.

A release is also a phase change, so if you register both, **`onEpisodeReleased` listeners run first**, then
`onEpisodePhaseChanged` ones. Delivery is otherwise identical: once per change, after the transaction
commits, on a host thread, best effort — not durable, not replayed, keep the listener idempotent.

Test it with `FakePluginContext.fireEpisodePhaseChanged(slug, phase)` (`phase == null` for the cancellation
case) and `episodePhaseListenerCount()` to assert a plugin subscribed at all — see `testing.md`.

## `Scope`

```java
public record Scope(ScopeType type, String id)
ScopeType = SITE | FEED | SEASON | EPISODE | USER
Scope.site()                 // id pinned to SITE_ID = "main"
Scope.user()                 // id pinned to SELF_ID = "me"
Scope.feed(id) / season(id) / episode(id)
Scope.season(feedSlug, season)   // builds "<feedSlug>:<season>" for you — since 0.17.0
```

Ids are **public slugs**, not UUIDs (a season is `<feedSlug>:<n>`). The canonical constructor normalises
SITE and USER ids whatever you pass. There is deliberately **no `Scope.user(String)`** — a plugin has no
business naming a user, so the IDOR cannot be written. `Scope.site(String)` was removed in 0.5.0. A `switch`
over `ScopeType` must handle `USER` or have a `default`.

**Prefer `Scope.season(feedSlug, season)` over hand-building the id with `Scope.season(id)`** (0.17.0) — it
validates (`IllegalArgumentException` on a blank `feedSlug` or a negative `season`) and matches what
`DisplaySnapshot.seasonScope()` returns, so a per-season aggregate never risks a separator typo. The two
parts come from a `DisplaySnapshot`'s own `feed()`/`season()` accessors (see below).

`Role` = `ADMIN | PODCASTER | FAN`; anonymous is the absence of a user, not a `Role` value.

## `DocStore`

```java
String KEY_PATTERN          = "^[A-Za-z0-9._:-]{1,200}$";
String BACKEND_OWNED_PATTERN = "^(\\*|[A-Za-z0-9._:-]{1,200}\\*?)$";   // since 0.6.0

<T> Optional<T> get(Scope scope, String key, Class<T> type);
void            put(Scope scope, String key, Object value);      // last-write-wins
boolean         delete(Scope scope, String key);                 // idempotent
List<DocEntry>      query(Scope scope, String keyPrefix);
// queryAcrossUsers(prefix) was here until 0.16.0 — it moved to ctx.allUsers(), see below
```

- **Every scoped method throws `UnsupportedOperationException` on a `USER` scope — reads included.** A
  backend thread has no calling user, so resolving `me` would have to pick someone. Per-user data is written
  by the frontend against `data/user/me/…` and read back by the backend only in aggregate.
- `put` throws `IllegalArgumentException` on a key violating `KEY_PATTERN`.
- `DocEntry(String key, JsonNode value)` — `tools.jackson.databind.JsonNode`.
- **`DocStore` lost `queryAcrossUsers` in 0.16.0** and has otherwise not changed since 0.5.0.
  `backendOwned` is enforced by the host on the HTTP surface only; your backend keeps writing those keys.
- **`data.keyFloors` (0.19.0, manifest-only) is the same story.** Like `backendOwned`, it is an HTTP-surface
  rule — `ctx.store()` reads and writes every key regardless of any floor declared on it. See `manifest.md`
  for the selector grammar, the raise-only rule, and the three places a hidden key behaves differently
  (listing, batch read, single read/write).

### Aggregates across users — `ctx.allUsers()`, declared (0.16.0)

```json
"data": { "writableBy": "fan", "readableBy": "anonymous", "readsAllUsers": true }
```
```java
CrossUserStore everyone = ctx.allUsers();                    // null without data.readsAllUsers
List<OwnedDocEntry> marks = everyone.query("mark:");         // record(UUID userId, String key, JsonNode value)
ctx.store().put(Scope.episode(slug), "leaderboard", board);  // publish for the frontend to read
```

**It is a declared capability now.** Reading every user's partition — every account's documents, owner UUIDs
included — is the one read that crosses an ownership boundary, so since 0.16.0 it is `null` unless the
manifest says `data.readsAllUsers: true`, the same null-means-not-declared shape as `blobs()`/`users()`/
`notifier()`, and an operator sees it on the admin plugin page. Until 0.16.0 it was
`ctx.store().queryAcrossUsers(prefix)` and every plugin had it by existing. If your code depends on it, treat
`null` as a manifest bug and **throw** — an empty aggregate would publish "nobody did anything" as fact.

`allUsers().query(...)` is backend-only, read-only and has **no HTTP surface**, so no visitor's request can reach
another's data through it. The `userId` is host-resolved from the partition the doc lives in, never
client-supplied — that is what makes the aggregate true; a client-reported summary is a summary of whatever
users typed. `userId` is an identifier, not a display name, and the contract offers no way to turn it into one.

Whatever you publish this way is a **shared-scope key your backend authors** — list it in
`data.backendOwned`, and write it in `register()` too, not only on the schedule (see `manifest.md`).

## `SchemaStore` (only when the manifest declares entities)

`ctx.schema()` is `null` for a doc-only plugin. Provisioning has already run by the time `register()` is
called.

```java
String        namespace();                    // "plugin_<id>_"
Set<String>   entities();
<T> Optional<T> find(String entity, long id, Class<T> type);
<T> List<T>   select(String entity, Criteria c, Class<T> type);
<T> List<T>   search(String entity, String field, String text, Criteria c, Class<T> type);
long          count(String entity, Criteria c);
long          insert(String entity, Map<String,Object> values);     // returns the assigned id
int           update(String entity, long id, Map<String,Object> values);
int           delete(String entity, Criteria c);
```

Address entities and fields **by declared name only** — never SQL, never a table name. That is the whole
scoping guarantee. An undeclared entity or field throws `IllegalArgumentException`, as does supplying `id`
on insert/update, and `search` on a field not declared `:fulltext`; empty search text returns an empty list.

`Criteria` is an immutable builder, **AND-only** (there is no `or` in this version):

```java
Criteria.where("published", Op.EQ, true).and("views", Op.GT, 100)
        .orderBy("updatedAt", Direction.DESC).limit(20).offset(0);
Criteria.all();
Op = EQ NE LT LTE GT GTE IN LIKE IS_NULL IS_NOT_NULL
```

Since 0.7.0 the frontend **can read** these tables through `ctx.schema` (`select`/`search`/`find`/`count`,
same vocabulary, `page`/`size` paging) — projecting the corpus into doc keys for the UI is obsolete. It
stays **read-only** over HTTP: your backend is the only writer of relational truth. A frontend that must
write puts a document in the doc store and this backend ingests it in `onSchedule(...)`.

**That HTTP read surface has its own floor since 0.19.0 — `storage.schemaReadableBy`**, defaulting to
`data.readableBy` when the manifest leaves it out. It is a manifest-level, HTTP-only rule exactly like
`keyFloors` above: `SchemaStore` here, in your backend, is **unaffected** by it either way — this table is
yours, unconditionally, from the moment provisioning finishes.

Note the two shapes that differ from the Java side, so a test written against one does not mislead you on
the other: the frontend's `find` resolves `null` for a missing row (Java returns `Optional.empty()`), and an
undeclared entity is a 404 from the host rather than an `IllegalArgumentException`.

## `PluginBlobs` (only when the manifest declares a `blobs` block)

`ctx.blobs()` is `null` otherwise — same shape and same reasoning as `ctx.schema()`. The Java half exists
largely for what only a backend can do: fetching on a schedule, and **collecting the orphans nothing else
collects**.

```java
BlobInfo        put(String filename, String mime, InputStream data);   // stream read to end, NOT closed
Optional<BlobInfo> stat(String ref);
InputStream     open(String ref);                                      // streaming; you close it
boolean         delete(String ref);                                    // idempotent
List<BlobInfo>  list(int page, int size);                              // newest first; host caps size
String          urlFor(String ref);                                    // root-relative, host-served
BlobQuota       quota();                                               // effective, not what you declared
```

```java
record BlobInfo(String ref, String filename, String mime, long size, Instant updatedAt) {}
record BlobQuota(long usedBytes, long quotaBytes, long maxFileBytes) { long remainingBytes(); }
```

```java
PluginBlobs blobs = ctx.blobs();
try (InputStream in = Files.newInputStream(path)) {
    BlobInfo stored = blobs.put("architecture.png", "image/png", in);
    ctx.store().put(Scope.site(), "diagram", Map.of("ref", stored.ref()));   // store the ref
}
```

- **`put` throws `IllegalArgumentException`** for all four refusals: type not allowed, bytes contradicting
  the declared type, over the per-file ceiling, or over the quota. `quota()` is how you see them coming.
- **`mime` on the way back is what the host sniffed**, not what you claimed. SVG is never storable.
- `stat`/`open` see only your own namespace — another plugin's ref is indistinguishable from a missing one.
- **The ref is the identity**; `urlFor(ref)` is derived and unchecked (it does not verify the file exists).
  Never store a URL, never build a ref.
- A file outlives the document that named it. A scheduled sweep — list what you have, drop what nothing
  points at — is the plugin's job, because only the plugin knows which those are.

## `FeedAccess` and `PluginConfig`

```java
List<String>    episodesIn(Scope scope);   // access-filtered slugs; Scope.user() returns nothing — includes `planned` ones (0.18.0)
DisplaySnapshot display(String refId);     // not authoritative — the host overwrites it on every feed refetch

<T> Optional<T> get(String key, Class<T> type);
<T> T           get(String key, Class<T> type, T fallback);
```

**`FeedAccess` sees `planned` episodes regardless of who is looking — this is the one place access filtering
does not apply (0.18.0).** Preparing content before an announcement is the entire point of a planned
episode, so your backend gets it unfiltered. That means **you** are now the access boundary for anything you
republish: a leaderboard, a computed card, a cached aggregate built from `episodesIn`/`display` must check
`display(slug).phase()` and withhold a `PLANNED` episode's content from anyone who is not its preparer,
exactly the way the host withholds the episode page itself. The frontend has no equivalent gap — `ctx.feeds`
and `ctx.episodes` stay access-filtered there (see `frontend.md`).

`DisplaySnapshot(title, description, audioUrl, publishedAt, duration, imageUrl, feedImageUrl, author,
subtitle, descriptionText, feed, season, episodeNo, phase, announceAt)` — the **15-component** canonical
constructor since 0.18.0 — plus `artwork()`, which falls back from episode image to feed image, and
`seasonScope()`.

- **`description` is the feed's show notes verbatim — untrusted third-party HTML** (documented since 0.16.0).
  Anything that writes show notes into output — an `OgMeta` description, a `SearchHit` excerpt, a
  notification — uses **`descriptionText()`** (0.16.0), which the host reduced to plain text. Never `null`.
- The 9-arg constructor (without `descriptionText`) still compiles, **deprecated for removal**, and leaves
  it `""`; the 10-arg one (without `feed`/`season`/`episodeNo`) and the 13-arg one (without `phase`/
  `announceAt`) both stay for a fixture that never looks at that part — a fixture exercising release phase
  should use all fifteen.
- **`phase` and `announceAt` are identity too, like `feed`/`season`/`episodeNo` (0.18.0)** — derived by the
  host on read from the stored status, the announcement instant and the clock, never written into the
  snapshot; a feed refetch cannot move an episode between phases any more than it can move it between
  seasons. `phase()` is an `EpisodePhase` (`PLANNED | UPCOMING | RELEASED | WITHDRAWN`) — **branch on it,
  never on the episode's stored `status`**, since an `UPCOMING` episode is still internally `PLANNED` but is
  fully public. `announceAt()` is set only while `PLANNED`/`UPCOMING` with a scheduled announcement; absent
  once released or withdrawn, and cleared automatically by `FakeFeedAccess.withPhase(...)` in tests when you
  move an episode out of those two phases.
- **`feed`, `season`, `episodeNo` are the one authoritative trio (0.17.0)** — resolved from `EpisodeRef`
  identity (§4.4) when the host hands the snapshot over, **never** part of what a feed refetch overwrites
  like every other field here. All three optional: absent against a feed with no season data, and
  `episodeNo` absent for an unnumbered episode inside a numbered season (never infer one from the other).
  `seasonScope()` returns `Scope.season(feed, season)`, or `null` when either is missing — prefer it over
  reading the two fields and building the scope yourself.
- **`season`/`episodeNo` are "as the site places the episode," not purely `itunes:*` any more (core 0.7.7,
  documented as of 0.19.0).** A podcaster can set either by hand in the admin — a prologue a show calls
  "episode 0," which `itunes:episode` structurally cannot carry, is the case this exists for. The hand-set
  value wins, survives every later feed poll, and **may be `0`**. Checking `if (episodeNo != null)` is
  right; checking `if (episodeNo != 0)` or (TS) `if (episodeNo)` is a bug that only shows up on exactly the
  episodes this feature exists to support.
  ```java
  var snaps = episodeSlugs.stream().map(ctx.feeds()::display).filter(Objects::nonNull).toList();
  for (var snap : snaps) {
      Scope season = snap.seasonScope();                 // null-safe; never hand-build "<feed>:<n>"
      if (season != null) totals.merge(season, minutesOf(snap), Double::sum);
  }
  ```

## `Tags` (only when the manifest declares a `tags` block)

`ctx.tags()` is `null` for a plugin declaring no `tags` block — same shape as `schema()` and `blobs()`.

```java
List<TagInfo> all();                                       // the whole site vocabulary, most-used first
List<String>  episodesWith(String tag);                    // canonicalised; unknown tag → empty, not error
List<String>  tagsOn(String episodeSlug);
List<TagInfo> similarTo(String tag, int limit);             // co-occurrence, best first — advice, not a contract
List<String>  subjectsWith(String tag);                     // your own plugin's subjects only
List<String>  tagsOnSubject(String subjectKey);
void          tagSubject(String subjectKey, String tag);    // idempotent; adds a new tag to the vocabulary
void          untagSubject(String subjectKey, String tag);  // idempotent; removes an assignment, never the tag
void          tagEpisode(String episodeSlug, String tag);          // needs tags.writesEpisodes
void          untagEpisode(String episodeSlug, String tag);        // needs tags.writesEpisodes; yours only
```

```java
record TagInfo(String tag, String label, int episodes, int subjects) {}
```

- `tagSubject`/`untagSubject` need only `tags.readsVocabulary`; `tagEpisode`/`untagEpisode` additionally need
  `tags.writesEpisodes` and throw `UnsupportedOperationException` without it — the Java mirror of the 403 the
  HTTP surface returns.
- `tag` in every method accepts **any spelling**; the host canonicalises (trim, collapse whitespace,
  casefold) and keeps the first spelling seen as the display `label`.
- `untagEpisode` removes only **your plugin's own** assignment — one recorded with `source = plugin:<id>`. If
  the feed or a podcaster also tagged the same episode with the same tag, it stays tagged after your call.
- There is no `delete`/`rename` on the vocabulary itself — a plugin may never remove a shared word or another
  writer's row; that is admin's job in the UI, not an API a plugin backend can reach.

## `Users` (only when the manifest declares an `identity` block, 0.13.0)

`ctx.users()` is `null` for a plugin declaring no `identity` block — same shape and reasoning as `schema()`
and `blobs()`. The fix for an `allUsers()` aggregate: it hands you `OwnedDocEntry(userId, …)`, UUIDs and
nothing else, so a leaderboard built from it had no way to draw a person.

```java
List<UserRef> resolve(Collection<UUID> ids);
```

```java
record UserRef(UUID id, String displayName, String avatarUrl, Role role) {}
```

- **Absent, not redacted, and therefore not index-aligned.** An unknown, erased or pseudonymised id is
  simply missing from the result — no `null` element. `resolve(List.of(a, b))` may come back with one entry
  or zero; match on `UserRef.id()`, never on position.
- **Duplicate ids resolve once; an empty input returns an empty list rather than throwing.**
- **It resolves, it does not enumerate.** There is no list/search method and never will be — you may ask
  only about ids you already came by through your own scope (a doc you wrote, a row `allUsers().query(...)`
  handed you).
- **`avatarUrl` is finished and never null.** Always `/api/users/{id}/avatar` (ARCHITECTURE §8.7),
  host-relative — every user has one, generated from the UUID when there's no provider picture, so there is
  no fallback for a plugin to write.
- **Store the UUID, resolve at render — never persist `displayName`.** A name copied into your own storage
  survives the rename meant to shed it and the erasure meant to end it, and core cannot reach inside your
  schema tables to fix either — it provisioned them without ever learning which column is a person. **The
  host cannot enforce this; this sentence is the enforcement.**
- **No role floor.** The HTTP surface behind this (`GET /api/plugins/<id>/users?ids=…`) is not gated by
  `data.readableBy` — there is nothing here a visitor couldn't already see wherever the ids came from.

## `Notifier` (only when the manifest declares a `notifications` block, 0.14.0)

`ctx.notifier()` is `null` for a plugin declaring no `notifications` block. This is the **one surface that
writes into another user's experience** — everything else a plugin touches is its own scope or the current
visitor's — so expect to call it from `onSchedule(...)`, where the thing worth announcing usually finishes.

```java
List<UUID> send(Collection<UUID> userIds, NotifyMessage message) throws NotificationException;
```

```java
record NotifyMessage(Map<String, String> text, String link) {
    NotifyMessage(Map<String, String> text)  // no link
    NotifyMessage(String english)            // English-only — the honest shape if you ship no other language
    String textFor(String locale)            // falls back to English; what the shell does when it draws the bell
    NotifyMessage withLink(String target)    // returns a new instance — this record is immutable
}
```

- **`text` must contain `en`** — the canonical constructor throws `IllegalArgumentException` if it's
  missing, blank, or if `text` is empty. Codes are trimmed and lower-cased on the way in.
- **`send` tells you who actually got it — read the return value.** Ineligible or erased recipients are
  left out, not rejected: one stale participant must not cost the other forty-nine theirs. A plugin that
  ignores the return and works from a stale list notifies nobody while looking perfectly healthy.
- **Two host-enforced bounds, neither yours to lift:**
  - **Eligibility** — you may only reach a user id your plugin already holds `USER`-scope data for, checked
    against the same partitions `allUsers().query(...)` spans — whether or not you declared `readsAllUsers`.
  - **Rate limits are the host's.** `NotificationException.Reason.RATE_LIMITED` (`retryable()` true) covers
    both the per-recipient daily allowance (`perUserPerDay`: what your manifest *asks*, capped by
    `mosaicast.plugin-notifications.hard-per-user-per-day`, default **20**; the operator's default when you
    ask for nothing is `mosaicast.plugin-notifications.default-per-user-per-day`, default **5**) and a
    per-call batch ceiling (`mosaicast.plugin-notifications.max-batch`, default **200** — checked against
    the *eligible* count, so a call naming 1000 ids with 150 eligible ones passes). A scheduled sender
    should **hold the batch for the next tick** on `RATE_LIMITED`, not drop it.
- **`link` is validated and internal-only.** `INVALID_LINK` (not retryable) for a scheme, a leading `//`, a
  backslash, a colon, a `..` segment, or a path under another plugin's `/p/<otherId>/`. A bare core absolute
  path (`/episodes/kraken`) or your own `/p/<pluginId>/…` subpath (with or without the prefix) is accepted
  and normalised to an absolute path.
- **There is no read side.** No list, no count, no mark-as-read, no "was this opened" — and nothing here
  reaches email, ever.

```java
enum NotificationException.Reason { RATE_LIMITED /* the only retryable() one */, INVALID_LINK, INVALID_MESSAGE }
```

## `Locales` and `Translation` (0.10.0)

`ctx.locales()` is **never `null`** — every install has at least English. `ctx.translation()` is `null`
unless the manifest declares `external.kinds: ["translation"]` **and** an admin configured a provider
(0.11.0 added the first half of that gate; the operator half existed since 0.10.0).

```java
List<LocaleInfo> available();              // languages the shell can render in
List<LocaleInfo> contentLocales();         // languages content may be authored in — build editor tabs from this
String           defaultLocale();
boolean          isContentLocale(String code);   // the write-time check; the browser's list is only a hint
```

```java
record LocaleInfo(String code, String nativeName, boolean isDefault) {}
```

`available()` and `contentLocales()` are genuinely different lists — a site can require content in a
language its UI does not offer. Build a per-locale editor from `contentLocales()`, never `available()`.

```java
TranslationResult translate(TranslationRequest request) throws TranslationException;
boolean            available();            // whether a call would even be attempted — advisory, re-check may lie
```

```java
record TranslationRequest(String text, String from, String to, Format format) {}   // Format: TEXT | HTML
record TranslationResult(String text, String detectedSourceLanguage, String providerId, boolean fromCache) {}
```

`TranslationException` is **checked** and carries `reason()` — `NO_PROVIDER`, `MISCONFIGURED`,
`RATE_LIMITED`, `BUSY`, `TIMEOUT`, `PROVIDER_FAILED` — plus `retryable()` for the three worth retrying.
Two things worth internalising:

- **Markdown is neither `TEXT` nor `HTML`.** Send it as `TEXT` and expect links and code fences to come back
  mangled — the provider does not know they are markup. Splitting markdown into translatable blocks is the
  caller's job.
- **Machine output is a draft.** Store it flagged and let a person confirm it before it is shown as fact —
  the same posture core's own legal-page prefill takes.

`Locales`/`Translation` never reach `USER`-partitioned data and carry no per-plugin quota; the only gate on
either is the manifest.

## Optional extension points

Implement zero, one or more alongside `PluginBackend` — **on the same class is fine and now correct**:

```java
Optional<OgMeta>     metaFor(String subpath);           // ShareMetadataProvider — link previews under /p/<id>/*
List<SitemapUrl>     urls();                            // SitemapProvider — entries for sitemap.xml
boolean               hasRoute(String subpath);          // PageRouteProvider — real 404s (0.9.1)
List<SearchHit>       search(String query, Role role, int limit);   // SearchProvider — site-wide search (0.9.0)
void                  eraseUser(String userId);          // UserDataHandler — account deletion reaches you (0.9.0)
Optional<Map<String,Object>> exportUser(String userId);  // UserDataHandler — defaulted to Optional.empty()
Optional<UserExport>  exportFiles(String userId);        // UserDataHandler — this plugin's part of a GDPR
                                                          // export, as files (0.19.0, wired by core 0.8.0)
```

Since core 0.6.7 the host uses PF4J's `SingletonExtensionFactory`, so **all your extension points run on the
same instance `register(ctx)` ran on**. The sample's old `static ctx` workaround (which existed because PF4J
built a fresh object per lookup, leaving providers with a null context) is obsolete — a plain instance field
is correct.

- **`OgMeta(title, description, imageUrl, locale)`** (0.12.0 — `locale` is new; the old 3-arg constructor
  still compiles and means `locale = null`): title and description non-null, `imageUrl` nullable and falling
  back to the site default. The first provider with a non-empty answer wins; `subpath` is never null and is
  empty at the plugin root.
  - **`locale`** — the language *this title and description are written in*, or `null` for "whatever the
    host resolved for this request" (most plugin pages — leave it out). Say it only when your page's text is
    fixed in one language regardless of who asks — `og:locale` on that response is this value, not the
    site's request-resolved one, and it is what the host now documents as *the language of that URL*
    (ARCHITECTURE §6.4/§6.6), not an install-wide constant. **It is a claim about the text in this record**:
    if you fell back to a default language because you had no translation for the requested locale, the
    honest value is your default's code, never the code that was asked for.
- **`SitemapUrl(loc, lastModified, alternates)`** (0.12.0 — `alternates` is new; the old 2-arg constructor
  still compiles and means "no translation group", `Map.of()`). Entries are **filtered to your own
  namespace** — `loc` must equal `/p/<id>` or start with `/p/<id>/`; anything else is dropped, and every path
  inside `alternates` is confined the same way.
  - **`alternates`** — `Map<String, String>` of locale code → path, feeding the sitemap's `hreflang`
    alternates (ARCHITECTURE §6.6). Empty means no translation group, which is what the host already
    assumed pre-0.12.0. **The map must contain an entry pointing at `loc` itself** — that entry is you
    naming the language *this* page is written in, which the host has no way to know and will not guess;
    the canonical constructor throws `IllegalArgumentException` without it. Rendering one path per language:
    map every locale to that same `loc` (`Map.of("en", loc, "de", loc)`). A wiki with a German article at a
    *different* path from its English one lists both paths under one shared map, on both entries.
  - **List a language only if that page is really written in it.** Serving your default-language content to
    a reader who asked for German is a kindness to a visitor and a lie to a crawler that is told a
    translation exists.
  - **The host still owns URL shape.** Alternates are bare paths, never `?lang=` appended by you — the host
    adds the parameter, leaves the site default on the *bare* URL, points `x-default` there, and makes the
    group reciprocal. An alternate aimed outside your own `/p/<id>/` namespace is **dropped silently**
    (logged server-side, not rejected) rather than failing the whole entry — losing a legitimate page from
    the sitemap over one bad alternate would be the larger punishment. If dropping leaves nothing naming
    `loc`'s own language, the **whole group is discarded**, since a set that no longer says what its own
    page is written in says nothing the host can honestly emit. `SitemapProviderHarness` (see `testing.md`)
    catches this before it ships.
- **`PageRouteProvider.hasRoute(subpath)`** (core 0.9.1 host): the host turns `false` into a real `404` for
  `/p/<id>/<subpath>`; `true` (the default when unimplemented) keeps serving `200` for everything, which is
  today's soft-404 behaviour. `subpath` is received exactly as `ShareMetadataProvider.metaFor` receives it —
  **empty at your own root**, so a lookup written purely over your own known slugs must answer `false` there
  or 404s your landing page. Deliberately **not** a second reading of `ShareMetadataProvider`: your subtree
  can legitimately hold views with nothing to describe (a search-result page) that still exist. It runs on a
  request, like `SearchProvider` — keep it cheap; a throw is logged and skipped, serving `200`.
- **`SearchProvider.search(query, role, limit)`** (core 0.9.0 host): your content contributed to
  `/api/search?q=`, grouped by source rather than merged into one ranking — your `score` and Postgres
  `ts_rank` are not on one scale. `role` is `null` for an anonymous caller. **This is the one extension point
  where the host does not filter for you** — it has no model of your objects, so returning a draft page to
  an anonymous visitor is a leak nothing else catches. Results name a `subpath` under `/p/<id>/`; the host
  resolves the URL and drops `.`/`..` segments the same way `ctx.route.navigate` does.
- **`UserDataHandler.eraseUser`/`exportUser`/`exportFiles`** (core 0.9.0 host; `exportFiles` since SDK
  0.19.0, wired by core 0.8.0): asked before an account row is dropped. Erase or pseudonymise is **your
  call** — the host cannot make it; keep the identity link cut but the contribution intact where an
  aggregate (a leaderboard, a vote count) must stay correct, hard-delete where the content itself is the
  person's. **Must be idempotent** — a failed deletion is retried, and the host reports a receipt (complete,
  plus what has not finished) rather than a bare success.
  - **`exportFiles` is now the real GDPR export path (ARCHITECTURE §12.8.1); `exportUser` is its fallback,
    not a separate feature.** The host asks `exportFiles` first; an empty `Optional` (its default) makes it
    ask `exportUser` instead and write a non-empty map as `plugins/<id>/data.json` — so a plugin written
    against the old `Map` form keeps exporting with zero changes, and a plugin with a real export just
    implements `exportFiles` and leaves `exportUser` at its default:
    ```java
    @Override
    public Optional<UserExport> exportFiles(String userId) {
        List<Card> cards = cardsOf(userId);
        if (cards.isEmpty()) return Optional.empty();
        return Optional.of(UserExport.of(
                ExportFile.text("cards.json", "application/json", toBingoV1(cards))));
    }
    ```
  - **Bounded, and never truncated.** At most `UserExport.MAX_BYTES` (32 MiB) across every `ExportFile`,
    answered within `UserExport.TIMEOUT` (60 s); go over either and the host records this plugin's part as
    **`failed`** rather than handing over a partial export that reads as a complete one. `ExportFile(path,
    mediaType, bytes)` paths must match `ExportFile.PATH_PATTERN` (relative, no `.`/`..` segment) — the host
    packs every file under `plugins/<id>/` in one ZIP alongside what core holds directly.
  - **Only this person's data, same discipline as erasure.** A leaderboard row that merely *mentions* the
    requester is not theirs to hand over — the host has no way to check this, so it is entirely on you.
    **Read-only**: the export may be retried, and a person may ask again tomorrow; a call must change
    nothing it reads.
  - **Every outcome is recorded, never silent.** `complete` (handed over data), `empty` (asked, had
    nothing), `failed` (threw, timed out, or went over the byte cap), `outstanding` (plugin switched off or
    rejected — asked again once it's back), and **`not-supported`** for a plugin with no `UserDataHandler`
    at all — core cannot tell "holds nothing" from "never implemented this," so it says which.
  - Called independently of `eraseUser` either way — an export is a request in its own right, and one
    missing the plugin half is an incomplete answer to a legal one.
- A provider that throws is logged and skipped; it can never break a render, the sitemap, a page route or a
  search page. Disabling the plugin removes its sitemap URLs and OG tags immediately, and — because
  `UserDataHandler` is asked whenever a plugin has ever stored anything, not only while it is active — a
  switched-off plugin still owes any outstanding erasure once it is switched back on.

## Jackson 3

`plugin-api` depends on `tools.jackson.core:jackson-databind:3.2.1`. `DocEntry.value()` and
`OwnedDocEntry.value()` are `tools.jackson.databind.JsonNode`, not `com.fasterxml.jackson...`. Mappers are
immutable: build one with `JsonMapper.builder().build()`, never `new ObjectMapper()`. `JacksonException` is
unchecked, so drop `throws JsonProcessingException` and catch-to-satisfy-the-compiler blocks.

`store().get(...)`, `config().get(...)` and every `SchemaStore` read deserialize straight into your own type
and never hand you a `JsonNode` — **only `query(...)`/`allUsers().query(...)` do.** A plugin that never
queries has nothing Jackson-shaped to change.

## Lifecycle

A broken or incompatible plugin is disabled with an admin warning at startup; core keeps booting. Disabling
a running plugin is immediate for every host-mediated surface — public manifest, data API, assets, deep
link, scheduler ticks, and backend writes (which start failing with an access-denied error) — but the
already-started backend object stays in the process until core restarts. Deleting the folder makes the
plugin dormant; stored data survives until an admin explicitly purges it.
