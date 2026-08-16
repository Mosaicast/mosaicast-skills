# Migrating an existing plugin to 0.6.0

The SDK's own `MIGRATION.md` (in the `mosaicast-plugin-sdk` checkout) is the authoritative checklist for the
**SDK** half — read it, it is short. This file adds the **core-side** changes it does not cover, because
core moved too.

Coming from `0.4.x`? Do the 0.5.0 migration first (the `USER` scope and the `data` block), then this.

## The SDK half, in brief

Nothing written against 0.5.0 stops compiling. This is a version bump plus one manifest line if your backend
computes anything.

1. `plugin.json` → `"platformApi": "0.6.0"`; `build.gradle.kts` → `plugin-api` / `plugin-testkit` `0.6.0`;
   `package.json` → `@mosaicast/plugin-sdk` `0.6.0`. You have no choice about timing: the moment the host
   runs 0.6.x, every 0.5.x plugin is rejected at load.
2. Declare every shared-scope key your backend writes in `data.backendOwned` — anything in `register()` or
   `onSchedule(...)` that calls `ctx.store().put(Scope.site()/feed()/season()/episode(), …)`.
3. **(Re)write those keys in `register()`**, not only on a schedule: the declaration refuses *new* client
   writes but does not remove a value forged before it existed.
4. Never list a client-written key (vote, mark, answer). If it belongs to a person it belongs in the `USER`
   scope.
5. Prove it: `new InMemoryDocStore().withBackendOwned("stats")` and assert `asUser(...)` writes throw.

Quick checklist: `platformApi` at 0.6.0 in all four places · every backend-authored shared key declared ·
those keys written in `register()` · no client key in the list · a test asserting the forged write is refused
· tests green against testkit/`makeMockCtx` 0.6.0.

## The core half — what changed under you

Core is at 0.6.7. None of this breaks a 0.5.x-era plugin's code, but it changes what is possible and what
was silently broken.

**Schema storage now works** (core 0.6.6). If your plugin was parked on the doc store because
`storage: {schema: …}` was rejected at load, that blocker is gone. See `manifest.md` for the declaration
grammar and the additive-only migration rules. The bare string `"storage": "schema"` is still rejected — you
must declare at least one entity. Note there is still **no HTTP surface for the schema store**, so anything
your UI reads has to be projected into a doc key.

**Extension points now run on the instance `register(ctx)` ran on** (core 0.6.7, `SingletonExtensionFactory`).
If you implemented `ShareMetadataProvider` or `SitemapProvider` and worked around a null context by making
your context field `static`, **delete that workaround** — a plain instance field is now correct. If you
skipped those extension points because they "did nothing", they work now: symptoms of the old bug were
missing sitemap entries, `/p/<id>/…` falling back to site-level OG tags, or an NPE from a call you never made.

**The `page` placement.** If your plugin has a deep link, it needs a slot with
`{ "scope": "site", "placement": "page" }` — without one, `/p/<id>/*` is a real 404. Check any older manifest.

**A `placement: "admin"` slot renders nowhere.** It has always validated; the shell has never mounted it.
If you shipped one expecting an admin surface, move the UI to a `sidebar` slot with
`visibleTo: "podcaster"` (that is what the sample does for its settings tile).

**Read floors are no longer inferred.** Since core 0.6.0 the `data` block is the only thing that governs
data access; the old derivation from the minimum slot `visibleTo` is gone and nothing replaced it. A plugin
with an anonymous display slot and no `data` block now 403s on reads that used to work — declare
`"readableBy": "anonymous"` if the data really is public.

**Strict media sources** (core 0.6.1, off by default). If an operator enables
`mosaicast.security.strict-media-sources`, `img-src`/`media-src` narrow to feed-derived origins plus your
*consented* hosts. Images and audio your component loads from anywhere else stop rendering. Declare those
origins in `consent.services[].hosts` now, even though it looks unnecessary today.

## After the bump

```bash
./gradlew build && npm ci && npm run build && ./build.sh
```

Copy `dist/` into `$MOSAICAST_PLUGINS_DIR/<id>`, restart core, and **check the admin log viewer at startup** —
a rejected manifest disables only your plugin, quietly, with its reason there.
