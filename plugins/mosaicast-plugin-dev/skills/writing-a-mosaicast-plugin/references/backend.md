# Backend — Java, PF4J extension

Implement `PluginBackend.register(PluginContext ctx)` on an `@Extension` class in package
`dev.mosaicast.plugin.<name>.*`. Compile against the SDK only — never core — and stay Spring-free.

## `PluginContext` — ten accessors, exactly

```java
DocStore store();                                // the generic doc store
SchemaStore schema();                            // null unless the manifest declares schema entities
PluginBlobs blobs();                             // null unless the manifest declares a blobs block (0.8.0)
Tags tags();                                     // null unless the manifest declares a tags block (0.9.0)
PluginConfig config();
FeedAccess feeds();
Locales locales();                                // never null — the site's languages (0.10.0)
Translation translation();                        // null unless external.kinds declares "translation" AND
                                                   // an admin configured a provider (0.10.0; gated 0.11.0)
org.slf4j.Logger logger();                       // already named "plugin.<pluginId>"
void onSchedule(Duration every, Runnable task);  // ShedLock-wrapped, at most once across instances
```

There is **no `ctx.log(...)` in Java** (that is the TypeScript context) and no route-registration API — a
plugin does not author HTTP endpoints.

**`translation()`'s gate is Java-only-half of 0.11.0's change.** Declaring `external.kinds` is the whole
story on the backend — there is no role floor here, because `register()` and `onSchedule` have no visitor.
`external.usedBy` governs only the browser client (`ctx.translation` in TypeScript); it is ignored for this
accessor.

`logger()` is the one to use. A logger you build yourself with `LoggerFactory.getLogger(...)` falls outside
the `plugin.` prefix, so the host cannot attribute it to you or show it in the admin log viewer. The host
persists `info`+ and surfaces `warn`+ there; a tight loop gets rate-limited, not stored.

`onSchedule` requires a positive period, is fixed-rate on a small shared pool, wraps each tick in a
try/catch, and is **skipped entirely while the plugin is disabled**.

## `Scope`

```java
public record Scope(ScopeType type, String id)
ScopeType = SITE | FEED | SEASON | EPISODE | USER
Scope.site()                 // id pinned to SITE_ID = "main"
Scope.user()                 // id pinned to SELF_ID = "me"
Scope.feed(id) / season(id) / episode(id)
```

Ids are **public slugs**, not UUIDs (a season is `<feedSlug>:<n>`). The canonical constructor normalises
SITE and USER ids whatever you pass. There is deliberately **no `Scope.user(String)`** — a plugin has no
business naming a user, so the IDOR cannot be written. `Scope.site(String)` was removed in 0.5.0. A `switch`
over `ScopeType` must handle `USER` or have a `default`.

`Role` = `ADMIN | PODCASTER | FAN`; anonymous is the absence of a user, not a `Role` value.

## `DocStore`

```java
String KEY_PATTERN          = "^[A-Za-z0-9._:-]{1,200}$";
String BACKEND_OWNED_PATTERN = "^(\\*|[A-Za-z0-9._:-]{1,200}\\*?)$";   // since 0.6.0

<T> Optional<T> get(Scope scope, String key, Class<T> type);
void            put(Scope scope, String key, Object value);      // last-write-wins
boolean         delete(Scope scope, String key);                 // idempotent
List<DocEntry>      query(Scope scope, String keyPrefix);
List<OwnedDocEntry> queryAcrossUsers(String keyPrefix);          // since 0.5.0
```

- **Every scoped method throws `UnsupportedOperationException` on a `USER` scope — reads included.** A
  backend thread has no calling user, so resolving `me` would have to pick someone. Per-user data is written
  by the frontend against `data/user/me/…` and read back by the backend only in aggregate.
- `put` throws `IllegalArgumentException` on a key violating `KEY_PATTERN`.
- `DocEntry(String key, JsonNode value)` — `tools.jackson.databind.JsonNode`.
- **`DocStore` itself has not changed since 0.5.0.** (0.8.0 moved the Java contract only by adding
  `PluginContext.blobs()`.) `backendOwned` is enforced by the host on the HTTP surface only; your backend
  keeps writing those keys.

### Aggregates across users

```java
List<OwnedDocEntry> marks = ctx.store().queryAcrossUsers("mark:");   // record(UUID userId, String key, JsonNode value)
ctx.store().put(Scope.episode(slug), "leaderboard", board);          // publish for the frontend to read
```

`queryAcrossUsers` is backend-only, read-only and has **no HTTP surface**, so no visitor's request can reach
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
List<String>    episodesIn(Scope scope);   // access-filtered slugs; Scope.user() returns nothing
DisplaySnapshot display(String refId);     // not authoritative — the host overwrites it on every feed refetch

<T> Optional<T> get(String key, Class<T> type);
<T> T           get(String key, Class<T> type, T fallback);
```

`DisplaySnapshot(title, description, audioUrl, publishedAt, duration, imageUrl, feedImageUrl, author,
subtitle)` plus `artwork()`, which falls back from episode image to feed image.

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
```

Since core 0.6.7 the host uses PF4J's `SingletonExtensionFactory`, so **all your extension points run on the
same instance `register(ctx)` ran on**. The sample's old `static ctx` workaround (which existed because PF4J
built a fresh object per lookup, leaving providers with a null context) is obsolete — a plain instance field
is correct.

- `OgMeta(title, description, imageUrl)`: title and description non-null, `imageUrl` nullable and falling
  back to the site default. The first provider with a non-empty answer wins; `subpath` is never null and is
  empty at the plugin root.
- `SitemapUrl(loc, lastModified)`: `lastModified` nullable. Entries are **filtered to your own namespace** —
  `loc` must equal `/p/<id>` or start with `/p/<id>/`; anything else is dropped.
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
- **`UserDataHandler.eraseUser`/`exportUser`** (core 0.9.0 host): asked before an account row is dropped.
  Erase or pseudonymise is **your call** — the host cannot make it; keep the identity link cut but the
  contribution intact where an aggregate (a leaderboard, a vote count) must stay correct, hard-delete where
  the content itself is the person's. **Must be idempotent** — a failed deletion is retried, and the host
  reports a receipt (complete, plus what has not finished) rather than a bare success. `exportUser` defaults
  to `Optional.empty()` and is called independently of erasure, for a data-export request in its own right.
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
and never hand you a `JsonNode` — **only `query(...)`/`queryAcrossUsers(...)` do.** A plugin that never
queries has nothing Jackson-shaped to change.

## Lifecycle

A broken or incompatible plugin is disabled with an admin warning at startup; core keeps booting. Disabling
a running plugin is immediate for every host-mediated surface — public manifest, data API, assets, deep
link, scheduler ticks, and backend writes (which start failing with an access-denied error) — but the
already-started backend object stays in the process until core restarts. Deleting the folder makes the
plugin dormant; stored data survives until an admin explicitly purges it.
