---
name: releasing-a-mosaicast-plugin
description: Use when cutting a release of a Mosaicast plugin, bumping its version or its platformApi/SDK pin, running build.sh and verifying dist/, tagging, or installing a built plugin into MOSAICAST_PLUGINS_DIR and confirming core loaded it. Covers the version anchors that must stay in sync, the GitHub Packages PAT prerequisite, dist/ verification, and the load-failure modes to check in the admin log viewer.
---

# Releasing a Mosaicast plugin

Two versions are in play and they are not the same thing:

- **the plugin's own version** — SemVer, yours to choose, lives in three files
- **`platformApi`** — the host contract the backend compiled against, currently **0.19.0** (core **0.8.0**
  hosts it, matched by core on exact `major.minor`; core's own minor moved 0.7.8 → 0.8.0 for the GDPR export
  wiring **without** another `platformApi` bump — patch and even a core minor are free to the host, but keep
  one `platformApi` string across all four anchors, because the contract test and the CI drift guard compare
  them literally)

Getting either out of sync produces a plugin that builds cleanly and is rejected at load, quietly, with the
reason only in the admin log viewer.

## The version anchors

**Plugin version** — must be identical in all three:

| File | Field |
|---|---|
| `plugin.json` | `"version"` |
| `backend/build.gradle.kts` | `version = "…"` |
| `frontend/package.json` | `"version"` |

**Contract version** — must be identical in all of:

| File | Field |
|---|---|
| `plugin.json` | `"platformApi"` |
| `backend/build.gradle.kts` | `dev.mosaicast:plugin-api:<v>` and `dev.mosaicast:plugin-testkit:<v>` |
| `frontend/package.json` | `"@mosaicast/plugin-sdk"` |
| the contract test | `expect(manifest.platformApi).toBe(PLATFORM_API_VERSION)` |
| `.github/workflows/ci.yml` | the package job's `platformApi` vs `plugin-api:<v>` assertion |

Nothing automates this for a plugin repo. The SDK solved the same problem with
`scripts/set-version.sh` (one script bumping every anchor) plus a release workflow asserting the tag equals
all of them before publishing — **port that pattern** if you release this plugin more than occasionally. A
`sed` across the three files plus a green CI package job is the minimum.

## Release order

```bash
# 1. bump — all three plugin-version anchors together
# 2. build from clean
./build.sh                       # gradle clean jar + npm ci + npm run build → dist/

# 3. verify what actually shipped
for f in dist/plugin.json dist/<id>.jar dist/assets/<id>.es.js; do [ -s "$f" ] || echo "MISSING $f"; done
grep -o '"platformApi": *"[^"]*"' dist/plugin.json
grep -o 'plugin-api:[0-9][^"]*'   backend/build.gradle.kts     # these two must agree

# 4. tests
( cd backend && ./gradlew test )
( cd frontend && npm test && npm run typecheck )               # Vite does not type-check; tsc --noEmit does

# 5. changelog entry in README, signed commit, tag
git commit -s -m "release <version>"
git tag v<version>
```

`build.sh` writes **only** `dist/` and never touches core. Run it *after* the version bump — a `dist/` built
before the bump ships the old manifest and is the most common way a release goes out wrong.

## Prerequisite that trips every first-timer

The SDK artifacts live on GitHub Packages, which **requires authentication even for public reads**. Provide
`gpr.user`/`gpr.token` gradle properties locally (a PAT with `read:packages`) or `GITHUB_ACTOR`/`GITHUB_TOKEN`
in CI. Without it the backend build fails to resolve `dev.mosaicast:plugin-api` with a 401 that reads like
the artifact does not exist.

**0.19.0 is published** on both npm and GitHub Packages (tagged `v0.19.0`), so no workaround is needed today. If a
future SDK version you need is on master but **untagged**, resolve it from a local checkout instead — `mavenLocal()` after `./gradlew publishToMavenLocal` in the SDK repo, or
`includeBuild("../mosaicast-plugin-sdk")`. Do not ship a release built that way without confirming the
artifact is public first, or nobody else can rebuild it.

**The SDK's 0.15.0 release originally shipped tagged only `0.15.0`, breaking its own `v<version>`
convention** (every other release, `v0.19.0` and `v0.18.0` down to `v0.1.0`, has the prefix). Never affected npm/GitHub
Packages, which publish off `package.json`/`build.gradle.kts` rather than the tag. `v0.15.0` was since added
as a second tag on the same commit, so both `git checkout v0.15.0` and `git checkout 0.15.0` resolve —
but don't assume every future release gets this treatment; check the tag actually exists before a script
assumes the `v` prefix.

## Install and verify

```bash
rm -rf "$MOSAICAST_PLUGINS_DIR/<id>" && mkdir -p "$MOSAICAST_PLUGINS_DIR/<id>"
cp -r dist/* "$MOSAICAST_PLUGINS_DIR/<id>/"
# restart core
```

The folder name **must equal the manifest `id`**. Then verify in this order — each step fails differently:

1. **Admin → plugins**: is it listed and **active**? A rejected manifest disables that plugin only; core
   boots fine and says nothing else.
2. **Admin log viewer**: the rejection reason lands there at startup. Also where your `ctx.logger()` output
   (`info`+ persisted, `warn`+ surfaced) and frontend `ctx.log` entries appear.
3. **The slots**: does each declared slot actually render? Remember `placement: "admin"` renders nowhere and
   `scope: "season"` matches no region — both validate happily.
4. **`/p/<id>/`**: resolves only if the plugin is active *and* declares a `{ scope: "site", placement: "page" }`
   slot; otherwise a real 404. If you declared `nav[]`, check the host's menu shows your entries (or, with no
   `nav[]` at all, one default entry named after the manifest's `name`).
5. **Config**: values appear in the generated admin form; a field the caller may not edit is redacted from
   the read-back too.
6. **Narrow widths and dark theme**: check the tile at a phone width (375px) and in both themes before
   calling it released — regions like `card`, `player` and `top` are much tighter than a laptop makes them
   look. If you have no browser access, say so in the release notes rather than implying it was checked.

## Load-failure modes, ranked by how often they happen

| Symptom | Cause |
|---|---|
| plugin absent from admin, reason in the log viewer | `platformApi` minor drift — core matches `major.minor` exactly |
| same | installed folder name ≠ manifest `id` |
| same | `data.backendOwned` entry that is not an exact key, a single-trailing-`*` prefix, or the bare `*` (entries are **not trimmed** — `" stats"` fails) |
| same | a schema field whose **type changed** since the last load — provisioning is additive-only and refuses type changes |
| same | `storage: "schema"` as a bare string, or a schema block with no entities |
| same | unknown slot placement, `data.writableBy: "anonymous"`, a consent category containing a dot, a host without a scheme |
| same | a `blobs` block with a non-positive limit, an empty `mimeTypes` list, `image/svg+xml` in it, an unknown `readableBy`/`writableBy` floor, or `blobs.writableBy: "anonymous"` |
| same | a `data.keyFloors` entry naming no keys, raising neither floor, below the plugin's own floor, with a bad selector, or `writableBy: "anonymous"` |
| same | `storage.schemaReadableBy` outside the four known roles |
| same | a `tags` block with both `readsVocabulary` and `writesEpisodes` false, or an `external` block with empty `kinds` — both "asks for nothing" refusals |
| same | an `external.kinds` entry that is not `"translation"` (the only kind today), or `nav[]` entries with no `page` slot, no `label`, a `../`-climbing `path`, or two entries normalising to the same `path` |
| `nav[]` loads but core ignores your entry's role floor | you wrote `"role"` — core's field is `"visibleTo"`, same as a slot; the SDK's TS type disagrees with core here |
| uploads all fail with 415 | the install's `allowed-mime-types` does not include what you declared — the lists are intersected |
| uploads fail with 413 at an unexpected size | an admin's per-plugin grant replaced your manifest's ask; read `…/blob/quota` |
| loads, but the extension never runs | missing `annotationProcessor("org.pf4j:pf4j:…")`, so no extension index was generated |
| loads, but shows stale/wrong data | old `dist/` — `build.sh` ran before the version bump |
| a tile is blank | the slot's error boundary caught a thrown render; check the browser console and `ctx.log` |

## Disabling, purging, uninstalling

Toggling a plugin off is immediate for every host-mediated surface — public manifest, data API, assets, deep
link, scheduler ticks and backend writes — but the already-started backend stays in the process until core
restarts. Deleting the folder makes the plugin dormant and **keeps its data**. Purge (admin) deletes its
documents, drops its schema tables **and removes its stored files**; the config and the enabled flag
survive. Plan a purge before
reinstalling a plugin whose schema field types changed, since provisioning will otherwise refuse the load.
