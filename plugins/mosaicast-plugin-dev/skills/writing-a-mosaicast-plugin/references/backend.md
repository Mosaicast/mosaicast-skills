# Backend — Java, PF4J extension

Implement `PluginBackend.register(PluginContext ctx)` on an `@Extension` class in package
`dev.mosaicast.plugin.<name>.*`. Compile against the SDK only — never core — and stay Spring-free.

## `PluginContext` — six accessors, exactly

```java
DocStore store();                                // the generic doc store
SchemaStore schema();                            // null unless the manifest declares schema entities
PluginConfig config();
FeedAccess feeds();
org.slf4j.Logger logger();                       // already named "plugin.<pluginId>"
void onSchedule(Duration every, Runnable task);  // ShedLock-wrapped, at most once across instances
```

There is **no `ctx.log(...)` in Java** (that is the TypeScript context) and no route-registration API — a
plugin does not author HTTP endpoints.

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
- The interface **did not change in 0.6.0**. `backendOwned` is enforced by the host on the HTTP surface
  only; your backend keeps writing those keys.

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

Remember the frontend cannot reach the schema store. Project anything the UI needs into a doc key.

## `FeedAccess` and `PluginConfig`

```java
List<String>    episodesIn(Scope scope);   // access-filtered slugs; Scope.user() returns nothing
DisplaySnapshot display(String refId);     // not authoritative — the host overwrites it on every feed refetch

<T> Optional<T> get(String key, Class<T> type);
<T> T           get(String key, Class<T> type, T fallback);
```

`DisplaySnapshot(title, description, audioUrl, publishedAt, duration, imageUrl, feedImageUrl, author,
subtitle)` plus `artwork()`, which falls back from episode image to feed image.

## Optional extension points

Implement zero, one or both alongside `PluginBackend` — **on the same class is fine and now correct**:

```java
Optional<OgMeta>     metaFor(String subpath);   // ShareMetadataProvider — link previews under /p/<id>/*
List<SitemapUrl>     urls();                    // SitemapProvider — entries for sitemap.xml
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
- A provider that throws is logged and skipped; it can never break a render or the sitemap. Disabling the
  plugin removes its sitemap URLs and OG tags immediately.

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
