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
  "version": "2.17.0",
  "platformApi": "0.19.1",
  "name": "Sample",
  "license": "Apache-2.0",
  "author": "The Mosaicast Authors",
  "homepage": "https://github.com/Mosaicast/mosaicast-plugin-sample",
  "attribution": "https://example.org/data-source",
  "backend":  { "basePath": "/api/plugins/sample", "extensions": ["dev.mosaicast.plugin.sample.SamplePlugin"] },
  "frontend": { "entry": "sample.es.js", "elements": ["sample-highlight", "sample-highlight-card"] },
  "slots": [
    { "scope": "episode", "element": "sample-highlight-card", "placement": "card", "visibleTo": "anonymous", "order": 100 },
    { "scope": "site", "element": "sample-highlight", "placement": "page", "visibleTo": "anonymous" }
  ],
  "nav": [{ "path": "", "label": "Sample", "icon": "star" }],
  "storage": "doc",
  "data":   { "readableBy": "anonymous", "writableBy": "podcaster", "backendOwned": ["stats", "agg:*"], "readsAllUsers": true },
  "blobs":  { "maxFileBytes": 5242880, "quotaBytes": 268435456, "mimeTypes": ["image/png", "image/jpeg"] },
  "tags":   { "readsVocabulary": true, "writesEpisodes": false },
  "external": { "kinds": ["translation"], "usedBy": "podcaster" },
  "identity": { "resolvesUsers": true },
  "notifications": { "sends": true, "perUserPerDay": 5 },
  "config": { "refreshIntervalMinutes": { "type": "number", "default": 30, "editableBy": "podcaster", "min": 1, "max": 1440, "step": 1 } },
  "consent": { "services": [], "categoryLabels": {} }
}
```

**`frontend.entry` has a grammar (enforced by core 0.7.4, documented in SDK 0.16.0):** a relative path under
the plugin's own `assets/` — `[A-Za-z0-9._-]` segments joined by `/`, no leading `/`, no `.`/`..` segment, no
`?` or `#`. Anything else is **rejected at load**, not rewritten. `FRONTEND_ENTRY_PATTERN` is the same rule.

`backend.basePath` and `backend.extensions` are **decorative** — nothing reads them. PF4J's generated
`extensions.idx` decides what loads, which is why `annotationProcessor("org.pf4j:pf4j:…")` is mandatory.

## `platformApi`

Exact `major.minor` match against the host's `PlatformApi.VERSION`; patch is free. Pre-1.0 the *minor*
carries breaking changes, so `0.15.x` against a 0.16.x host is rejected, and `"1.x"` fails to parse at all.
`"0.16"` and `"0.16.0"` both pass against a 0.16.x host — but keep the string identical to the SDK version
your code builds against, because the contract test and the CI drift guard compare them literally.

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

One floor pair, several surfaces: `readableBy` also governs blob reads (`…/blob`, `…/blob/{ref}`,
`…/blob/quota`, unless `blobs` declares its own — see below) and every call through `ctx.tags` — reads and
writes alike, layered under the tags block's own `readsVocabulary`/`writesEpisodes` gate; `writableBy`
governs blob uploads/deletes and the tags writes alongside that same gate. `backendOwned` does not reach the
blob or tags surfaces at all — neither lets a caller name an existing key or row belonging to someone else
(a blob ref is a host-minted UUID; a tags write is scoped to your own subjects, or recorded with your
plugin as its source).

**The schema read API is its own floor since 0.19.0 — `storage.schemaReadableBy`, not `data.readableBy`.**
Before, `readableBy` governed `/api/plugins/<id>/schema/*` the same way it governed everything else; a
plugin whose tile had to be anonymous (bingo) therefore also served every schema row to anonymous visitors
— player entries keyed by user id, rows for a quiet planned episode. `storage.schemaReadableBy` defaults to
`data.readableBy` when absent, so nothing changes for an existing manifest that never needed the split; see
"`storage`" below. There are no schema writes over HTTP either way, so `writableBy` still has no schema
counterpart.

**Per-key floors since 0.19.0: `data.keyFloors`.** The two floors above cover every key uniformly;
`keyFloors` raises *specific* keys above them — see below.

**`identity` and `notifications` sit outside this floor pair entirely** — neither `readableBy` nor
`writableBy` governs `ctx.users` or `ctx.notify`. Resolving users has no role floor at all (see `identity`
below); sending a notification is bounded by eligibility and the host's rate limits, not by `data`.

Neither floor applies to the `USER` scope in either direction: no floor makes someone else's partition
readable, none stands between a caller and their own, and `writableBy` does not gate it (it protects the
*shared* surface). A fan writes `data/user/me/…` under a plugin declaring `writableBy: "podcaster"`.
Anonymous on a `user` path is a 401 regardless.

### `readsAllUsers` (0.16.0)

`"readsAllUsers": true` is what makes the backend's `ctx.allUsers()` non-`null` — the read of **every**
user's `USER` partition, owner UUIDs included. Absent means no, like every other capability. It is the one
read that crosses an ownership boundary, so an operator should see it on the manifest (core shows it on the
admin plugin page) — "can enumerate everyone who ever used me" is not what the `USER` scope alone implies.
Until 0.16.0 this was `DocStore.queryAcrossUsers` and every plugin had it. Declare it only for a real
aggregate (leaderboard, rollup, moderation); the frontend is unaffected either way.

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

### `keyFloors` (0.19.0)

```json
"keyFloors": [
  { "keys": ["import:*", "staged:*"], "readableBy": "podcaster" },
  { "keys": ["bundles"], "writableBy": "admin" }
]
```

Raises the read or write floor of specific keys above the plugin's own `readableBy`/`writableBy` — private
bookkeeping beside public numbers, or an admin-only setting beside podcaster-writable ones, without
splitting the plugin into two manifests. Same selector grammar as `backendOwned`
(`DOC_KEY_SELECTOR_PATTERN`, an alias of `BACKEND_OWNED_PATTERN` — identical pattern, both names load): an
exact key, a `prefix*`, or the bare `*`.

**Rejected at load, every one of them:**
- a floor **below** the plugin's own `readableBy`/`writableBy` — raise-only, always
- `writableBy: "anonymous"` — same rule as the plugin-level floor
- an entry with no `keys`
- an entry raising neither `readableBy` nor `writableBy`
- a selector that isn't an exact key, a `prefix*`, or `*`

**At runtime:**
- Several entries matching one key combine to the **strictest** floor, per direction, independently.
- **Write order is plugin floor → `backendOwned` → key floor** — a `backendOwned` key stays client-unwritable
  no matter what its key floor says; a key floor cannot loosen what `backendOwned` closed.
- A **listing** (`GET .../data/{scopeType}/{scopeId}?prefix=…`) drops a key below the caller's floor
  *before* paging, so `totalElements` counts only what that caller may see — not "everything, minus a few
  redactions after the fact."
- A **batch read** (`getMany`) leaves a hidden key **absent**, indistinguishable from a miss.
- A **single** `get`/`put`/`remove` on a key below its floor is a **403** with its own problem type,
  `PROBLEM_TYPES.keyFloor` (`…/problems/key-floor`) — distinct from the plugin-floor `forbidden` and the
  `backendOwned` `backend-owned-key`, so a client can tell the three 403s apart without matching English.
- **Ignored for the `USER` scope**, same as `backendOwned`. **Your own `ctx.store()` is unaffected** — a key
  floor is a client-facing rule only; the backend reads and writes every key regardless.

A plugin that under-declares a floor here is not quietly fine on an older host: a host older than 0.19.0
doesn't know the field at all, ignores it, and serves those keys at the plugin's *wider* floor — which is
exactly why this shipped in a `platformApi` minor rather than as a patch.

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
}, "schemaReadableBy": "podcaster" }
```

- The **bare string `"schema"` is still rejected**: *"storage declares \"schema\" but no entities; use
  \"doc\" or declare at least one"*. Any other unrecognised string silently becomes `doc`.
- **`schemaReadableBy` (0.19.0) is the schema read API's own floor** — `select`/`search`/`count`/one row,
  all four gated the same way — defaulting to `data.readableBy` when absent, so an existing manifest is
  unaffected. Any of the four roles is legal (same vocabulary as `blobs.readableBy`); a value outside them
  is *"storage.schemaReadableBy '%s' is not one of […]"*, rejected at load exactly like a bad `blobs` floor.
  Declare it when your **tile** needs to be anonymous but your **rows** should not be — the backend's
  `SchemaStore` is unaffected either way; this is an HTTP read-API floor only.
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

## `blobs` — file storage (0.8.0)

```json
"blobs": { "maxFileBytes": 5242880, "quotaBytes": 268435456,
           "mimeTypes": ["image/png", "image/jpeg", "image/webp", "application/zip"],
           "readableBy": "podcaster", "writableBy": "podcaster" }
```

**Opt-in and declared, never derived** — the same rule as the data floors. Absent ⇒ no file storage at all:
`ctx.blobs` is `null`, `ctx.blobs()` is `null`, and every `/api/plugins/<id>/blob*` path answers **404**,
indistinguishably from an unknown or switched-off plugin.

Every field is optional; omitting one takes the install's own value. What you declare is a *request*:

- **The operator's numbers win.** `mosaicast.plugin-blobs.*` holds defaults (`default-quota-bytes` 256 MB,
  `default-max-file-bytes` 10 MB out of the box) and the install-wide `allowed-mime-types`. Your declared
  types are **intersected** with that list, and no grant widens it — what a file may *be* is a security
  question, not a capacity one. Asking for more than an install allows is granted less, never rejected: a
  plugin's portability should not depend on the most restrictive install it might meet.
- **An admin can raise (or lower) the limits per plugin** in the admin panel since core 0.6.13, and an
  admin's grant **replaces** the manifest's ask rather than being minimised with it. A limit set below what
  is already stored is allowed with a warning — nothing is deleted, further uploads just fail.
- So `quota()` / `GET …/blob/quota` is **the only honest source** for what was actually granted. Read it
  before letting someone pick a file; telling them up front beats a refusal after the upload.

What is rejected at load (all three are silent failures otherwise — a plugin that loads, declares storage
and refuses every upload):

- a non-positive `maxFileBytes` or `quotaBytes` — *"blobs limits must be positive; got …"*
- `mimeTypes` present but naming nothing usable — *"blobs.mimeTypes is present but names no type — omit it
  to take the operator's list"*
- `image/svg+xml` anywhere in `mimeTypes` — **SVG is never storable** (a script container wearing an
  image's extension); an operator cannot re-enable it either, since it is filtered out of the install's
  allow-list too
- a `blobs.readableBy`/`writableBy` that isn't one of the known floors — *"blobs floor '%s' is not one of
  […]"* — or a `writableBy` of `anonymous` — *"blobs.writableBy may not be 'anonymous' — an upload needs a
  signed-in user to belong to"* (same rule as `data.writableBy`)

**ZIP is storable (core 0.7.6).** `application/zip` is in the default allow-list; the sniffer recognises the
bytes and canonicalises the browser aliases (`application/x-zip-compressed`, `application/x-zip`) to it, so
declare `application/zip` in `mimeTypes`, never an alias. Anything that isn't an image or audio is served
`Content-Disposition: attachment`, not inline.

**`blobs` may set its own access floors (`readableBy`/`writableBy`, core 0.7.6), in the same vocabulary and
on the same rule as `data` (`writableBy` may not be `anonymous`).** Absent means the `data` floors — nothing
changes for an existing manifest. Use this to keep uploaded files more private than the plugin's computed
data: a stats plugin can publish its aggregate numbers under `data.readableBy: "anonymous"` while keeping the
raw archives it computed them from behind `blobs.readableBy: "podcaster"`. Without an explicit `blobs` floor,
reads take `data.readableBy` and writes take `data.writableBy`, same as before. `backendOwned` does not
apply to blobs either way. Purge takes a plugin's files with it, matched on the namespace exactly.

## `tags` — the shared vocabulary (0.9.0)

```json
"tags": { "readsVocabulary": true, "writesEpisodes": false }
```

**Opt-in and declared, never derived** — the same rule `data`, `blobs` and `external` follow. Absent ⇒ no
tag surface at all: `ctx.tags` is `null`, `PluginContext.tags()` is `null`, both endpoints 404.

Two flags because the two acts are not alike:

- **`readsVocabulary`** — read the site's whole tag vocabulary and tag your own subjects (`tagSubject` /
  `untagSubject`, keyed by a subject key you invent in your own namespace). `data.writableBy` is the whole
  authorization story for these — nothing else to declare.
- **`writesEpisodes`** — additionally tag/untag *episodes* (`tagEpisode` / `untagEpisode`). This is a real
  capability, not a convenience: it changes the shell's filter options and what `RelatedProvider`
  recommends beside that episode, so an operator should be able to read it off the manifest before
  installing, separately from the first flag.

**A block declaring neither is rejected at load** — *"tags block asks for nothing (readsVocabulary and
writesEpisodes are both false) — omit the block to declare no tag surface"* — since it would produce a
surface that exists (`ctx.tags` non-null) and refuses every call through it.

What no plugin may do, enforced rather than merely asked: delete a tag from the vocabulary (it is shared
and outlives its last assignment), rename one (a vocabulary-wide edit, admin's job), or remove another
writer's assignment on an episode — the feed's included. Every plugin write is recorded with
`source = plugin:<id>`, which is what makes the last rule enforceable: `untagEpisode` only ever removes
*your own* row.

The host owns the canonical key (trim, collapse internal whitespace, casefold) and keeps the display
`label` from first use, so `Maritime` and `maritime ` converge on one tag without lower-casing what a
visitor reads. Send any spelling; store and compare on the canonical key the host hands back.

## `external` — admin-configured third-party services (0.11.0)

```json
"external": { "kinds": ["translation"], "usedBy": "podcaster" }
```

**Opt-in and declared, never derived**, on the same terms as `blobs` and `tags`. Absent ⇒ no external
surface at all — `ctx.translation` / `ctx.translation()` are **`null` regardless of what the operator
configured**, which is the point: whether you declared is a static fact about your own manifest, and the
gate is checked before the operator's provider choice is even looked at.

- **`kinds`** — a list of `ExternalServiceKind` values; `"translation"` is the only member today
  (transcription, text-to-speech and embeddings are the shapes the surface is built to take next). An
  unknown kind is **rejected at load**, not dropped — *"external kind '%s' is not one of […]"* — because
  within one `platformApi` the vocabulary is closed and a name nothing answers to is a typo, the same
  reasoning `data.backendOwned` is refused under.
- **`usedBy`** — the lowest role that may trigger a call **from this plugin's UI**. Defaults to
  `podcaster`, matching `data.writableBy`'s floor. **Browser-side only** — `register()` and `onSchedule`
  have no visitor and no role, so declaring the kind is the whole gate on the backend. Unlike
  `data.writableBy`, **`anonymous` is legal here** (a self-hosted, free provider is a real case an operator
  might want public) — but it is logged as a warning at load, because behind a metered provider it is an
  open spending endpoint reachable by anyone who loads the page.
- **Empty `kinds` is rejected at load** — *"external block declares no kinds — omit the block to declare no
  external surface"* — the same "asks for nothing" refusal `tags` gets, for the same reason.
- One floor for the **whole plugin**, not per kind — with one kind today the two are the same thing spelled
  differently; a later per-kind floor can only *narrow* this one.

A non-`null` `ctx.translation` is still not permission: the host enforces `usedBy` **at the call**, so a
visitor below the floor holds a working-looking handle whose `translate()` rejects with **403**.

## `nav` — entrances into the host's menu (0.9.0)

```json
"nav": [
  { "path": "", "label": "Wiki" },
  { "path": "random", "label": "Random article", "icon": "shuffle" },
  { "path": "_new", "label": "New page", "visibleTo": "podcaster" }
]
```

A `page` plugin owns `/p/<id>/*`, but nothing links to it — without `nav` a visitor has to already know the
URL. A plugin declares *what* its entrances are; the host decides where and how they render in its own
navigation chrome.

- **Requires a `page` slot.** `nav` entries on a plugin with no `{ "scope": "site", "placement": "page" }`
  slot are rejected at load — *"nav entries declared without a `page` slot — /p/\<id\> would render
  nothing"*.
- **Declaring none is normal and back-compatible.** A `page` plugin with no `nav` block gets **one default
  entry at its root**, labelled with the manifest's `name` — every page plugin that predates this field
  keeps loading and keeps a menu entry, with no re-release required.
- **`path`** — the subpath below `/p/<id>/`; empty or absent means the plugin's root. Normalised and
  range-checked: a leading `/` or any `.`/`..` segment is **rejected at load**, not silently rewritten —
  *"nav path must be a plain subpath of the plugin, without a leading `/` or `..`"*. Two entries
  normalising to the same path are a **duplicate nav path** rejection.
- **`label`** — required; a blank one is rejected. Taken **verbatim, and never translated** — core has no
  plugin catalogs to translate it against. Your own in-page tab bar can translate its copy of the same
  entrance; pin `path`/`icon`/`role` between the two lists and let the label differ.
- **`icon`** — an `--mc-icon-*` name, without the prefix. Optional and **never validated** — an unknown name
  simply fails to render at the icon step; the host holds no icon list to check against (the palette lives
  in generated CSS), and rejecting a whole plugin over a mistyped decoration would be disproportionate.
- **The field is `visibleTo`, not `role`** — see "Which docs to trust" in `SKILL.md`. The SDK's
  `PluginNavDeclaration` TS type (documentation only) calls it `role`; core's manifest parser expects
  `visibleTo`, the same key a slot uses. Absent means anonymous.

## `identity` — resolving user UUIDs to people (0.13.0)

```json
"identity": { "resolvesUsers": true }
```

**Opt-in and declared, never derived** — the same rule `blobs`, `tags` and `external` follow, even though
your plugin already *has* the ids (`allUsers().query(...)` hands them over, if you declared `readsAllUsers`). What's being granted here is
not access to the UUIDs but **turning them into a name and a picture**, and that's the part an operator
should read off the manifest before installing. Absent ⇒ `ctx.users` / `ctx.users()` are `null` and the
`GET /api/plugins/<id>/users` endpoint 404s — same shape as `schema`, `blobs`, `tags`.

- **`resolvesUsers`** — the only field. **`true` is the default when the block is present and the field is
  absent or `null`** (`!Boolean.FALSE.equals(resolvesUsers)` — anything but an explicit `false` reads as
  yes). `"identity": {}` therefore grants the capability exactly as `"identity": { "resolvesUsers": true }`
  does. This is the *opposite* default from `data.readableBy` (which closes on silence) and from `tags`
  (which rejects a block asking for nothing) — read `"identity": {}` as "yes" here, not as "declared but
  inert."
- **No `validate()` method exists for this block at all** — unlike `tags`/`blobs`/`external`, nothing here
  is ever a load-time rejection. A malformed or empty block loads clean and defaults to granting the
  capability; there is no "asks for nothing" refusal to catch a mistake.
- **No role floor on the read.** `GET /api/plugins/<id>/users?ids=…` is not gated by `data.readableBy` —
  there is nothing here a visitor couldn't already see if they can see whatever rendered the ids (a public
  leaderboard, say), so an anonymous caller may resolve too.
- **It resolves, it never enumerates.** Comma-separated `ids=` only, capped at 500 per call (extra ids are
  silently dropped, not rejected); there is no list/search endpoint and never will be — you may only ask
  about ids you already came by through your own scope.
- **Unresolvable is absent, not an error or a redaction.** An unknown, malformed, erased or pseudonymised id
  is simply missing from the response array — one shape for all four cases, so the answer can never be used
  to tell them apart. A malformed UUID in `ids=` is skipped, not a 400.

## `notifications` — an in-app inbox for your plugin's users (0.14.0)

```json
"notifications": { "sends": true, "perUserPerDay": 5 }
```

**Opt-in and declared, never derived**, on the same terms as `identity`. Absent ⇒ `ctx.notify` /
`ctx.notifier()` are `null` and `POST /api/plugins/<id>/notify` 404s.

- **`sends`** — defaults to `true` when the block is present and the field is absent, **exactly like
  `identity.resolvesUsers`** — `"notifications": {}` already grants the capability. Same "opposite of
  `tags`" caveat applies, and the same absence of any `validate()` method: a malformed block loads clean.
- **`perUserPerDay`** — what your plugin *asks* for; the operator's host-wide cap
  (`mosaicast.plugin-notifications.hard-per-user-per-day`, default **20**) is what you actually get, taken
  as the smaller of the two — exactly how `blobs` quotas work. Omitting it takes the operator's *default*
  (`mosaicast.plugin-notifications.default-per-user-per-day`, default **5**), not zero: an absent number
  reads as "no opinion," never as "never send." There is **no per-plugin admin override** the way `blobs`
  has one — the caps are host-wide config only, as of this writing.
- **This is the one plugin surface that writes into *another* user's experience.** Everything else a plugin
  touches is its own scope or the current visitor's. Two host-enforced bounds, neither yours to widen:
  - **Eligibility**: you may only notify a user id your plugin already holds `USER`-scope data for — the
    same partitions `allUsers().query(...)` reads (whether or not you declared `readsAllUsers`). An ineligible id is silently dropped from `send()`'s return
    value, never an error; your call never learns *why* one recipient didn't get it.
  - **A batch is capped** at `mosaicast.plugin-notifications.max-batch` (default **200**) — checked against
    the *eligible* count, not the raw list you passed, so asking for 1000 with 150 eligible passes. Over the
    cap is `NotificationException.Reason.RATE_LIMITED` / a **429**, and it is `retryable()` — hold the batch
    for the next scheduler tick rather than dropping it.
- **`NotifyMessage.text` must contain `en`.** The one language a site can never switch off, so it is the
  only fallback a reader is guaranteed to understand; a message missing it is rejected before anything is
  sent (Java: `IllegalArgumentException` at construction; TS: the mock and the real client both refuse it).
  Use `notifyText(catalogs, key, params)` — built from the same catalogs you already pass
  `createPluginI18n` — rather than filling the map by hand, which is exactly where a plugin quietly ships
  one locale short.
- **`link` is host-validated and internal-only.** Accepts a bare core absolute path (`/episodes/kraken`) or
  a subpath of your own `/p/<pluginId>/` — with or without the `/p/<pluginId>/` prefix. Refused
  (`INVALID_LINK` / **400**, not retryable) for anything with a scheme, a leading `//`, a backslash, a
  colon, or a `..` segment — including protocol-relative `//evil.example`, which reads like a path and
  isn't one — and for a path under someone *else's* `/p/<otherPluginId>/`.
- **There is no read side, ever.** You cannot list an inbox, count it, mark it read, or learn whether a
  message was opened. And nothing here reaches email — in-app only.

## `license` / `author` / `homepage` / `attribution` — credit (core 0.6.15)

```json
"license": "AGPL-3.0-or-later",
"author": "The Mosaicast Authors",
"homepage": "https://github.com/Mosaicast/mosaicast-plugin-bingo",
"attribution": "https://example.org/data-source"
```

Four optional strings, surfaced through the anonymous `GET /api/plugins/manifest` and shown on the host's
public **`/about`** page — what an install runs, and under what terms, is not privileged information.

**`attribution` is not a duplicate of `homepage`.** "Where this lives" and "who deserves credit for it" are
different links: a plugin that borrows data, artwork or an upstream library can credit the source without
giving up its own page.

**Never validated, and never a rejection reason.** `license` is not checked against the SPDX list, none of
the four is checked for shape, and a plugin written before they existed keeps loading exactly as before —
credit is not a correctness concern. Add them freely; there is nothing here to get wrong.

**Adding these does not need a `platformApi` bump**, and bumping it *for* this change would be actively
harmful. Unknown manifest fields are ignored by the host and there is no manifest type in the SDK at all, so
this is additive in both directions — a manifest declaring these four loads on an older host (it just ignores
them), and a manifest without them loads on a newer one (the About page's plugin card renders without that
plugin's credit). `platformApi` compatibility, by contrast, is an **exact `major.minor`** match — bumping it
here would reject every already-installed plugin until each one re-released, for a change that needed no
contract move at all.

## `config`

```json
"config": {
  "refreshIntervalMinutes": {
    "type": "number", "default": 30, "editableBy": "podcaster",
    "label": { "en": "Ingest interval", "de": "Abrufintervall" },
    "description": "Minutes between two ingest runs."
  },
  "matchMode": {
    "type": "string", "default": "fuzzy",
    "options": [{ "value": "fuzzy", "label": "Fuzzy" }, { "value": "exact", "label": "Exact" }]
  }
}
```

Types `string | number | boolean` only; `default` must match the declared type; `editableBy` ∈
`admin` (default) | `podcaster`. The admin form is **generated** — plugins never ship config UI. A value the
caller may not edit is redacted in the read-back too. A JSON `null` clears an override.

- **`label` / `description`** — a plain string (one language) or a locale-code map (`{"en": "…", "de":
  "…"}`), resolved by the host down the chain the shell always uses: exact locale → base language (`de-AT`
  finds `de`) → `en` → any entry present. **Without `label` the generic form shows the operator the raw
  key** (`refreshIntervalMinutes`) and nothing else — plugins never get their own config UI, so this is the
  only chance to say what a setting is. Rejected at load only for *shape* — neither a string nor an object —
  never for content: a label the host cannot read is cosmetic, and refusing to load a whole plugin over one
  would be worse than showing the key.
- **`options`** — `[{ "value": …, "label"? }]`, declaring a closed set; the field renders as a select and
  refuses anything outside it, both for the manifest's own `default` (checked at load) and for an operator's
  override (checked at write time). `value` must match the field's declared `type` — a `string` field with
  a numeric option value is rejected at load, as is a `default` that isn't one of the declared options. This
  is not a new host behavior tied to a `platformApi` bump — **core has validated `options` since it
  shipped**; the SDK's own TS type (`PluginConfigOption`, `LocalizedText`) was simply behind until 0.15.0,
  so a plugin declaring an older `platformApi` and using `options` was already validated this way without
  the SDK telling its author the shape.
- **Bounds (0.16.0, enforced by core 0.7.4)** — `min` / `max` / `step` on a `number` field, `minLength` /
  `maxLength` on a `string` one. An out-of-range write is **refused with a 400 naming the bound** ("must be at
  least 1"), never clamped; the admin form renders them as input constraints. At load, a bound on the wrong
  type, `min > max`, a non-positive `step` or a `default` outside its own bounds is rejected. A value stored
  *before* a bound existed that now breaks it counts as **unset**, so the default applies — which is what
  lets you trust `config().get(...)` and drop your own clamp. Declare `"min": 1` on any interval: without it
  `0` was a legal save that switched scheduled work off. No `pattern` yet (Java vs JS regex dialects).

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
  labels `necessary`, `functional`, `analytics`; any other slug is allowed — and since 0.16.0 **you owe it a
  label** in `consent.categoryLabels` (below). Unlabelled, core 0.7.4 shows it as "Other services: social"
  with a generic hint; before, the visitor was asked to consent to the bare id.
- **`categoryLabels` (0.16.0)** — keyed by category id, block-level (not per service, so two services in one
  category cannot disagree): `{ "social": { "label": { "en": "Sharing buttons", "de": "…" }, "hint": "…" } }`,
  each a string or a locale map. Rejected at load: a label for a **host** category (`necessary`,
  `functional`, `analytics`, `unreviewed`), a label for a category none of your services declares, and one
  with no text. Two plugins labelling the same category: the lowest plugin id wins. Not the pre-0.4
  `categories` array, which stays rejected.
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
`data.backendOwned entry '%s' is not usable` · `blobs limits must be positive; got %s` ·
`blobs.mimeTypes is present but names no type` · `blobs.mimeTypes may not include image/svg+xml` ·
`blobs floor '%s' is not one of […]` · `blobs.writableBy may not be 'anonymous'` ·
`data.keyFloors entry names no keys` · `data.keyFloors selector '%s' is not usable: it must be an exact key,
a prefix ending in *, or the bare *` · `data.keyFloors entry for %s raises neither readableBy nor writableBy`
· `data.keyFloors floor '%s' is not one of […]` · `data.keyFloors writableBy may not be 'anonymous' — a
write needs a signed-in user` · `data.keyFloors readableBy '%s' for %s is below the plugin's read floor '%s'`
· `data.keyFloors writableBy '%s' for %s is below the plugin's write floor '%s'` (0.19.0) ·
`storage.schemaReadableBy '%s' is not one of […]` (0.19.0) ·
`tags block asks for nothing (readsVocabulary and writesEpisodes are both false)` ·
`external block declares no kinds` · `external kind '%s' is not one of […]` ·
`external.usedBy '%s' is not one of […]` (`external.usedBy: anonymous` loads fine but logs a warning) ·
`nav entries declared without a page slot` · `nav entry has no label` ·
`nav path must be a plain subpath of the plugin, without a leading / or ..` · `duplicate nav path: …` ·
`license`/`author`/`homepage`/`attribution` are **never** a rejection reason — see above — · `icon` on a
`nav` entry is **never** a rejection reason, it just fails to render · **nothing under `identity` or
`notifications` is ever a rejection reason** — neither block has a `validate*()` method, so any shape
(including one that inverts the field to `false`, or one that is simply `{}`) loads without error ·
`config field '%s' has unknown type/unknown editableBy/has a label or description that is neither a string
nor a locale object/has an option with no value/has an option that is not a <type>/default does not match
declared type/default is not one of its options/declares min/max/step, which only a number field takes/
declares minLength/maxLength, which only a string field takes/has min above max/step must be positive/
default must be at least …` (0.16.0) · `frontend.entry '…' is not usable` ·
`consent.categoryLabels cannot relabel the host's own category/labels '…', which none of this plugin's
consent services declares/'…' has no label text` (0.16.0) · `consent must declare services[]` (plus missing `name`, missing `category`,
bad category token, blank or scheme-less host, bad wildcard, unparsable origin, storage item without a name,
`*` as a storage name) · every schema resolution failure above · folder name ≠ manifest `id` ·
`cannot read plugin.json`.
