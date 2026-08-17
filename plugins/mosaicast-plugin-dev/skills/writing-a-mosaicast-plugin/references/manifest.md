# `plugin.json` — the manifest

Validated at load by core's `PluginManifest.validate()`. A rejection **disables that plugin only**, with its
reason in the admin log viewer; core keeps booting. Unknown JSON fields are tolerated everywhere, so a typo
in a field name is silently ignored rather than rejected — spell them exactly.

The manifest `id` **must equal the plugin's folder name** under `$MOSAICAST_PLUGINS_DIR`, or the plugin is
rejected.

## Full shape

```json
{
  "id": "sample",
  "version": "2.8.0",
  "platformApi": "0.7.1",
  "name": "Sample",
  "backend":  { "basePath": "/api/plugins/sample", "extensions": ["dev.mosaicast.plugin.sample.SamplePlugin"] },
  "frontend": { "entry": "sample.es.js", "elements": ["sample-highlight", "sample-highlight-card"] },
  "slots": [
    { "scope": "episode", "element": "sample-highlight-card", "placement": "card", "visibleTo": "anonymous", "order": 100 }
  ],
  "storage": "doc",
  "data":   { "readableBy": "anonymous", "writableBy": "podcaster", "backendOwned": ["stats", "agg:*"] },
  "config": { "refreshIntervalMinutes": { "type": "number", "default": 30, "editableBy": "podcaster" } },
  "consent": { "services": [] }
}
```

`backend.basePath` and `backend.extensions` are **decorative** — nothing reads them. PF4J's generated
`extensions.idx` decides what loads, which is why `annotationProcessor("org.pf4j:pf4j:…")` is mandatory.

## `platformApi`

Exact `major.minor` match against the host's `PlatformApi.VERSION`; patch is free. Pre-1.0 the *minor*
carries breaking changes, so `0.6.0` against a 0.7.x host is rejected, and `"1.x"` fails to parse at all.
`"0.7"`, `"0.7.0"` and `"0.7.1"` all pass against a 0.7.x host — but keep the string identical to the SDK
version your code builds against, because the contract test and the CI drift guard compare them literally.

## `slots[]`

`{ scope, element, placement, visibleTo?, order? }`. Only `placement` is validated.

Valid placements (`KNOWN_PLACEMENTS`): `top`, `card`, `main`, `sidebar`, `player`, `feed`, `site`, `admin`,
`page`. An unknown one is rejected at load.

What the shell actually renders, and the scope your element receives:

| placement | scope handed to the element | rendered in |
|---|---|---|
| `top` | site (`site/main`) | the top bar, every page |
| `site` | site | the site panel |
| `sidebar` | site **or** feed **or** episode | site panel, feed panel, episode page |
| `feed` | feed | the feed panel |
| `card` | episode | the compact episode feed card |
| `main` | episode | the episode page body |
| `player` | episode | the player bar |
| `page` | **site** | `/p/<pluginId>/…` |

Three traps:

- **`admin` validates but renders nowhere.** No admin route mounts a slot region. A slot declaring it loads
  clean and silently does nothing. (ARCHITECTURE §7.2's example manifest shows exactly such a slot.)
- **`scope: "season"` never matches any region.** Season scope exists in storage, not in the shell.
- **`page` must be declared with `scope: "site"`**, and without it `/p/<id>/*` returns a real 404 — the host
  reserves the route but refuses to soft-404 a page nobody opted into.

`visibleTo` ∈ `anonymous | fan | podcaster | admin`, and governs **rendering only** — it has had no effect on
data access since core 0.6.0. Absent ⇒ anonymous (everyone). An *unrecognised* value silently falls back to
`podcaster`, so a typo hides your tile from fans rather than failing loudly.

`order` ascending; absent sorts last; ties broken by plugin id. Multiple plugins in one region stack.

## `data` — the access floors and the reserved keys

```json
"data": { "readableBy": "anonymous", "writableBy": "podcaster", "backendOwned": ["stats", "agg:*"] }
```

Floors are `anonymous | fan | podcaster | admin`. **`writableBy` may not be `anonymous`** (rejected: a write
needs a signed-in user to belong to). Defaults when the block or a field is absent: `writableBy` =
`podcaster`, and **`readableBy` = the write floor**, not anonymous — saying nothing gets the closed answer.
So a plugin with an anonymous display slot and no `data` block **403s on reads**; if the data really is
public, say `"readableBy": "anonymous"` explicitly.

`readableBy` also governs the **schema** read surface (`/api/plugins/<id>/schema/*`) — one floor for both
stores, so a plugin that already declares one is covered. `writableBy` has no schema counterpart: there are
no schema writes over HTTP.

Neither floor applies to the `USER` scope in either direction: no floor makes someone else's partition
readable, none stands between a caller and their own, and `writableBy` does not gate it (it protects the
*shared* surface). A fan writes `data/user/me/…` under a plugin declaring `writableBy: "podcaster"`.
Anonymous on a `user` path is a 401 regardless.

### `backendOwned` (0.6.0)

Authorization on the data API is **per plugin, not per document**. Any caller above `writableBy` can
overwrite or delete any key in any shared scope, including a value your backend computed on a schedule — the
host cannot tell your scheduled write from a `curl`:

```bash
curl -b cookie.podcaster -X PUT .../api/plugins/sample/data/site/main/stats \
     -H "X-XSRF-TOKEN: $XT" -d '{"totalEpisodes":9999}'      # 204, and now everyone reads 9999
```

Declaring the key closes it: clients may still **read** it, but a client `PUT`/`DELETE` is refused with a
403 whose problem type is `problems/backend-owned-key` — deliberately distinct from the role-floor 403, so
you can tell which rule refused you. `ctx.store()` is unaffected; that is the entire point.

Grammar (`DocStore.BACKEND_OWNED_PATTERN`, shared verbatim between SDK and core so they cannot drift):

```
^(\*|[A-Za-z0-9._:-]{1,200}\*?)$
```

- an exact key (`stats`), a key-legal prefix with **one trailing** `*` (`agg:*`), or the bare `*`
  ("the whole store is computed, clients read only")
- no `*` in the middle, no empty entry, case-sensitive
- **not trimmed** — `" stats"` is rejected at load, on purpose. A malformed entry fails the plugin loudly
  rather than being dropped, because a dropped security declaration is the worst outcome available.

Three consequences worth internalising:

1. **It does not clean up.** Declaring a key does not remove a value a client forged before the declaration
   existed. The row survives until your backend overwrites it — so **write every computed key in
   `register()` as well as in `onSchedule(...)`**.
2. **It is ignored for `USER` scopes.** Even a bare `*` leaves `data/user/me/…` writable by its owner,
   because your backend cannot write a user partition at all.
3. **Never list a client-written key** (votes, marks, answers). Those belong in the `USER` scope; reserving
   them just breaks your own frontend. The sample asserts exactly this in a test.

A bare `*` makes `writableBy` vestigial for shared scopes, but it is still required and still may not be
`anonymous`.

## `storage` — doc (default) or schema

Absent ⇒ `"doc"`. The doc store is `plugin_data`: scope + key → JSONB, GIN-indexed, addressed by scope and
key. Anything belonging to one person goes in the `USER` scope; everything else in an entity scope.

Relational needs (full-text search, revisions, backlinks) declare a schema instead — provisioning works as of
core 0.6.6 and the **frontend read API since platformApi 0.7.0**; earlier guidance that core rejects schema
storage, or that only the backend can reach it, is obsolete:

```json
"storage": { "schema": {
  "page": {
    "slug":      "string:indexed:unique",
    "title":     "string",
    "markdown":  "text:fulltext",
    "views":     "integer",
    "updatedAt": "timestamp:indexed"
  }
} }
```

- The **bare string `"schema"` is still rejected**: *"storage declares \"schema\" but no entities; use
  \"doc\" or declare at least one"*. Any other unrecognised string silently becomes `doc`.
- Types → SQL: `string`/`text` → `text`, `integer` → `bigint`, `number` → `double precision`, `boolean`,
  `timestamp` → `timestamptz`.
- Modifiers: `indexed`, `unique` (implies a unique index), `fulltext` (GIN over
  `to_tsvector('simple', …)`). `:fulltext` is only legal on `string`/`text`.
- Entity, field and **plugin id** must match `[A-Za-z][A-Za-z0-9_]{0,39}` — the id becomes part of the table
  name. Two fields differing only in case are rejected.
- Declaring `id` is refused: it is assigned by the platform.
- Tables are `plugin_<pluginId>_<entity>`, capped at 47 characters (Postgres truncates at 63 and the runner
  reserves room for index suffixes; a collision after truncation would silently merge two entities).
- Provisioning runs **before `register()`**, transactionally, and is **additive only**: missing columns are
  added; a **changed field type refuses the plugin at load**. The plugin never writes DDL.
- A schema plugin **also keeps its doc store**. Declaring a schema is also what makes `ctx.schema` non-`null`
  in the frontend: the host serves `GET /api/plugins/<id>/schema/*` (**reads only**, gated by
  `data.readableBy`) and a doc-store plugin 404s there. Writes stay with the backend — a frontend that must
  write goes through the doc store and the backend ingests it. See `frontend.md`.
- Purge drops the plugin's documents *and* its schema tables (config and the enabled flag survive).

## `config`

```json
"config": { "refreshIntervalMinutes": { "type": "number", "default": 30, "editableBy": "podcaster" } }
```

Types `string | number | boolean` only; `default` must match the declared type; `editableBy` ∈
`admin` (default) | `podcaster`. The admin form is **generated** — plugins never ship config UI. A value the
caller may not edit is redacted in the read-back too. A JSON `null` clears an override.

## `consent`

A **service-level** declaration. The legacy `{ categories, externalSources }` shape is rejected since 0.4.0.

```json
"consent": { "services": [{
  "id": "plausible", "name": "Plausible Analytics", "provider": "Plausible Insights OÜ",
  "category": "analytics", "privacyUrl": "https://plausible.io/privacy",
  "hosts": ["https://plausible.example"], "thirdCountryTransfer": false,
  "storage": [{ "name": "plausible_ignore", "type": "localStorage", "purpose": "opt-out marker", "duration": "persistent" }]
}] }
```

- `hosts` needs a scheme and **is also the CSP allow-list** — an undeclared origin stays blocked even after
  consent is granted (it looks like "the embed just doesn't load", not a permissions error).
  `https://*.foo.example` is legal as a leading-label wildcard; a bare `*` is not; a blank host is rejected.
- `category` must match `[a-z0-9_-]{1,40}` — **no dots**, that is the consent cookie's separator. The host
  labels `necessary`, `functional`, `analytics`; any other slug is allowed and shown raw.
- A `necessary` claim needs **admin approval**; unapproved, it is demoted to the pseudo-category
  `unreviewed` and prompted like anything else. Use `necessary` only for what genuinely cannot be refused.
- The visitor decides per **category**, not per service — two services sharing a category (yours and another
  plugin's) are granted or refused together.
- `provider` is the operating company, not your plugin. `storage[].name` may not be `*` and may not contain
  a mid-string `*`.
- No third-party services → **omit `consent` entirely** and the site stays banner-free.
- TS exports `ConsentServiceDeclaration` / `ConsentStorageDeclaration` / `PluginDataDeclaration` as
  documentation-only types to catch typos in your editor; the SDK never reads `plugin.json` — core is
  authoritative.

## Every rejection reason

`manifest has no id` · `platformApi %s is incompatible with host %s` · `unknown slot placement: …` ·
`data floor '%s' is not one of […]` · `data.writableBy may not be 'anonymous'` ·
`data.backendOwned entry '%s' is not usable` · `config field '%s' has unknown type/unknown editableBy/default
does not match declared type` · `consent must declare services[]` (plus missing `name`, missing `category`,
bad category token, blank or scheme-less host, bad wildcard, unparsable origin, storage item without a name,
`*` as a storage name) · every schema resolution failure above · folder name ≠ manifest `id` ·
`cannot read plugin.json`.
