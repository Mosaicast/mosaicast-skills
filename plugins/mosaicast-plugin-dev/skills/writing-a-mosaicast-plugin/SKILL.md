---
name: writing-a-mosaicast-plugin
description: Use when creating or modifying a Mosaicast plugin (any mosaicast-plugin-* repo or the plugin-sample). Covers the plugin.json manifest (slots and placements, the page slot behind /p/<id>/*, data access floors, backendOwned keys, consent services, doc vs schema storage, the blobs file-storage block, the tags vocabulary block, the external-services block, nav entrances into the host menu, the license/author/homepage/attribution credit fields), the backend PluginBackend/PluginContext contract and its optional extension points (ShareMetadataProvider and its per-URL OgMeta.locale, SitemapProvider and its hreflang-alternates SitemapUrl.alternates, PageRouteProvider, SearchProvider, UserDataHandler), per-user data in the USER scope, the frontend Web Component via the SDK ctx (ctx.docs, ctx.feeds, ctx.tags, ctx.schema reads, ctx.blobs uploads, ctx.links, ctx.locale.available/content, ctx.translation, ctx.route.navigate, typed ctx.api errors) and theme tokens including the --mc-icon-* icon set, testing against the SDK test kit, and installing a plugin by spec (owner/repo@tag#sha256). Trigger whenever writing the manifest, adding a slot, wiring ctx, storing per-user data, uploading or serving files, linking to core pages, querying schema tables from the frontend, reading or writing the shared tag vocabulary, contributing to site search, handling account deletion/export, real 404s for a page plugin's unknown subpaths, declaring external-service (translation) use, declaring what language a plugin page or sitemap translation group is written in, adding a nav entrance, navigating inside a page plugin, declaring backend-owned or schema storage, styling a plugin tile's icons, crediting a plugin's license or data source, bumping platformApi, installing or releasing a plugin, or building a plugin's backend or frontend.
---

# Writing a Mosaicast plugin

Everything here was checked against the SDK and core working trees, not against the changelogs. Where a
doc and this skill disagree, see "Which docs to trust" below.

## Version pins — get these wrong and the plugin does not load

Contract version is **0.12.0** (`PlatformApi.VERSION`, `PLATFORM_API_VERSION`), and core **0.6.24** hosts it.
The host demands an **exact `major.minor`** match; patch is free. A `0.11.x` manifest is *rejected at load*,
not warned about.

```json5
"platformApi": "0.12.0"                                 // plugin.json
```
```kotlin
compileOnly("dev.mosaicast:plugin-api:0.12.0")          // backend/build.gradle.kts
compileOnly("org.pf4j:pf4j:3.15.1")
annotationProcessor("org.pf4j:pf4j:3.15.1")             // mandatory: generates the @Extension index
testImplementation("dev.mosaicast:plugin-testkit:0.12.0")
```
```json5
"@mosaicast/plugin-sdk": "0.12.0"                       // frontend/package.json
```

Both halves of `0.12.0` are published and tagged `v0.12.0` (npm, and GitHub Packages for the Java artifacts)
— no `mavenLocal()` workaround needed. Maven is GitHub Packages
(`https://maven.pkg.github.com/Mosaicast/mosaicast-plugin-sdk`), which needs a PAT with `read:packages`
**even for public reads**. Jackson is **3.2.1** (`tools.jackson.*`), not `com.fasterxml`. **Pin the same
string in all four places** — the CI drift guard and the manifest contract test both compare them.

Coming from an older pin? `references/migrating.md` walks 0.8.0 → 0.9.0 → 0.9.1 → 0.10.0 → 0.11.0 → 0.12.0 in
order — read it top to bottom rather than jumping straight to 0.12.0, since 0.9.0's compile break and 0.10.0's
silent `ctx.translation` gate both still apply on the way up. 0.12.0 itself is the easy step: a manifest bump
and a rebuild, no code change unless you deconstruct `OgMeta`/`SitemapUrl` as record patterns.

To re-check the pin yourself:
`grep -n 'VERSION = ' mosaicast-plugin-sdk/plugin-api/src/main/java/dev/mosaicast/plugin/api/PlatformApi.java`

## The rules you cannot get wrong

1. **Per-user data goes in the `USER` scope, never in a key.** `data/user/me/<key>` from the frontend;
   `Scope.user()` exists but every `DocStore` method throws `UnsupportedOperationException` on it —
   *reads included*. There is deliberately no `Scope.user(String)`. A per-user key under an entity scope
   (`mark:<userId>:cell`) is an IDOR: keys are client input, scope ids are public slugs.
2. **A shared-scope document has no owner.** Anything above your `writableBy` floor can `PUT` any key in any
   SITE/FEED/SEASON/EPISODE scope — including a value your backend computed. If your backend authors a key,
   **declare it in `data.backendOwned`**, and write it in `register()` as well as on the schedule.
3. **`data` is the only thing that governs data access.** A slot's `visibleTo` is rendering only. Omitting
   `readableBy` falls back to the *write* floor (closed), not to anonymous.
4. **Depend only on the SDK.** Never import core. Stay Spring-free — plain PF4J extensions.
5. **A deep link needs a `page` slot.** `/p/<id>/*` is a real 404 unless the plugin is active *and* declares
   `{ "scope": "site", "placement": "page" }`. Inside that subtree, move with **`ctx.route.navigate(subpath)`**
   — never `history.pushState` + a synthetic `popstate`, which works today and is not the contract.
6. **Every `onChange` returns an `Unsubscribe`** — return it from your render's cleanup, or a detached shadow
   root keeps receiving callbacks.
7. **`ctx.schema` is read-only and `null` unless the manifest declares `storage.schema`.** There are no
   schema writes over HTTP: the backend stays the only writer of relational truth, and a frontend that must
   write puts a doc in the doc store for the backend to ingest on its schedule (eventually consistent — design
   the UI for it).
8. **File storage is opt-in and `null` without a manifest `blobs` block** — `ctx.blobs` (TS), `ctx.blobs()`
   (Java), and every `…/blob` path answers 404. Unlike schema, **writes are the point**: `data.writableBy`
   plus the quota is the whole authorization story.
9. **Store the `ref`, never the URL.** A ref is the file's identity; `urlFor(ref)` is derived at render time
   and the host may reshape it. And **nothing collects orphans** — delete what you stop pointing at.
10. **Icons are a mask, never a `background-image`.** `mask-image: var(--mc-icon-x); background: currentColor`
    takes your colour and re-themes with everything else; a background image bakes in a colour and ignores
    the theme.
11. **`license`/`author`/`homepage`/`attribution` need no `platformApi` bump — and bumping it for them is
    actively harmful.** They are unvalidated manifest fields; `platformApi` compatibility is an exact
    `major.minor` match, so a bump for a non-breaking field change would reject every installed plugin.
12. **Declaring `tags` is two separate capabilities, not one.** `readsVocabulary` lets you read the site's
    shared vocabulary and tag your own subjects; `writesEpisodes` additionally lets you tag *episodes*,
    which changes the shell's filter options and what core recommends beside that episode. A block asking
    for neither is rejected at load — omit the block instead. You may never delete a tag, rename one, or
    remove another writer's assignment (the feed's included) — only your own.
13. **`ctx.translation` has two independent reasons to be `null`, and they are deliberately
    indistinguishable.** Your manifest did not declare `external.kinds: ["translation"]`, *or* the operator
    configured no provider (every site, by default). Check the manifest before the admin panel, and never
    cache the handle — read `ctx.translation` at the point of use, since the operator half can change under
    a running plugin. A non-`null` handle is still not permission: `translate()` 403s below
    `external.usedBy` (default `podcaster`).
14. **A `page` plugin's unknown subpaths answer `200` until you implement `PageRouteProvider`.** Absent
    means today's behaviour — every subpath under your plugin renders your not-found view inside a `200`,
    which a crawler indexes as real content. `hasRoute("")` (the empty string) is your own root; a lookup
    written over your own slugs alone answers `false` there and 404s your landing page.
15. **`SearchProvider` is the one place the host does not filter for you.** It has no model of your
    objects, so returning a draft page to an anonymous visitor (`role == null`) is a leak nothing else
    catches — the access check is yours to write, and `SearchProviderHarness` calls you once per role,
    anonymous included, so the test is one line.
16. **`nav[]`'s JSON key is `visibleTo`, not `role`.** The SDK's `PluginNavDeclaration` TS type (documentation
    only) names it `role`; core's actual manifest field — the one the host parses — is `visibleTo`, same as
    a slot. Write `visibleTo` in `plugin.json`; core wins over the SDK type on any disagreement.
17. **Prefer `ctx.docs`/`ctx.feeds`/`getOrNull` over hand-built paths and swallowed 404s.** `ctx.docs.get`
    resolves to `null` on 404 instead of rejecting; `ctx.api.getOrNull` does the same for the raw client.
    `ctx.feeds.display`/`displayMany` replace a scheduled ingest that copies episode snapshots into your own
    doc store — read them live, they are not authoritative and the host overwrites them on every refetch.

## Before you build: ask for a browser and an instance

Do this **once, at the start** of plugin work. Both are recommended, **neither is a blocker** — if the user
has neither, build the plugin in full anyway and say at the end what could not be verified.

1. **Check whether you have browser tooling** (Playwright MCP or equivalent). If not, tell the user once
   that you can still build and unit-test but cannot see the tile render, catch a CSP refusal or check a
   phone width — and that enabling a browser tool would fix that. Then carry on; do not ask again.
2. **Ask for a Mosaicast instance** you may install into and **restart** (a restart is how core picks up a
   rebuilt plugin), and **ask whether it holds production or test data**. If the user has a `mosaicast-core`
   checkout, `dev/instance.sh up --plugins` stands up a disposable seeded stack — best option, nothing real
   at risk. **`--plugins` is not optional here** — plugins are opt-in and the flag defaults *off* (the
   stack exists for README screenshots first, where the sample plugin's demo card is noise), so without it
   your plugin never loads and the tile you are testing is silently absent.

Then the data rule, which is absolute:

- **Production data → never insert, update or delete anything.** Read, render, screenshot. If a path can
  only be exercised by writing, say so and ask for a test instance instead of working around it.
- **Test or dummy data → you may seed, but ask first**, naming the scope and keys you intend to write.

**With both a browser and an instance, test across viewports and both themes while you develop** — 375×667,
667×375, 768×1024, 1280×800, 1920×1080, light and dark. Your tile renders in regions of wildly different
widths and you do not control any of them. Details, the full loop and what to look for: `references/dev-environment.md`.

## A plugin is one folder that builds to `dist/`

Backend JAR (PF4J extension) + frontend Web Component bundle + `plugin.json`. No forced internal layout —
core only reads what lands in `dist/`. `build.sh` writes **only** `dist/` and never touches core; copying
into `$MOSAICAST_PLUGINS_DIR` and restarting is a separate manual step (an optional `install.sh` may
shortcut it, never require it). Since core 0.6.15 a *released* plugin can also be installed by spec —
`owner/repo@tag#sha256:…`, resolved by `scripts/install-plugin.sh` or the container's `MOSAICAST_PLUGINS` —
without touching the filesystem by hand; see `references/dev-environment.md`.

Five distinct URL namespaces, don't conflate them:
- `/api/plugins/<id>/data/*` — the host's fixed generic doc-store API (you do not author routes)
- `/api/plugins/<id>/schema/*` — the host's read-only schema API, behind `ctx.schema` (schema plugins only)
- `/api/plugins/<id>/blob/*` — the host's file API, behind `ctx.blobs` (plugins declaring `blobs` only)
- `/plugins/<id>/assets/*` — static frontend bundle
- `/p/<id>/*` — deep-linkable plugin pages, surfaced as `ctx.route`

## Where to read next

| You are doing | Read |
|---|---|
| `plugin.json`: fields, slots, placements, floors, `backendOwned`, consent, config, schema, `blobs` and credit (`license`/`author`/`homepage`/`attribution`) declarations | `references/manifest.md` |
| Java backend: `PluginContext`, `DocStore`, `SchemaStore`, `PluginBlobs`, aggregates, scheduling, extension points | `references/backend.md` |
| Web Component: `ctx` surface, `ctx.schema` queries, `ctx.blobs` uploads, `ctx.links`, `route.navigate`, doc-store HTTP calls, CSP, theme, `--mc-icon-*`, i18n | `references/frontend.md` |
| Tests (required by the BRIEF's DoD) | `references/testing.md` |
| Browser + live-instance setup, viewport matrix, data-safety rules, the build→install→restart loop, installing a released plugin by spec | `references/dev-environment.md` |
| Moving an existing plugin from 0.8.x up to 0.12.0 | `references/migrating.md` |

Live reference implementation: **`mosaicast-plugin-sample` v2.10.0**, still on **SDK 0.8.0** as of this
writing — it predates `tags`/`feeds`/`docs`/`external`/`nav` and is not a source for those. It declares a
`blobs` block and exercises `ctx.blobs` (upload, `urlFor`, the `null` degrade path) and
`ctx.links.episode/feed`; its `README.md` carries the worked `curl` forgery that motivates `backendOwned`,
its page slot uses `ctx.route.navigate`, and its `docs/BRIEF.md` pattern is the definition of done for every
plugin repo. Re-check its tag before relying on a version number quoted here — it moves independently of the
SDK and has lagged it before.

## Which docs to trust

- **SDK `README.md` / `CHANGELOG.md` / `MIGRATION.md` and the Javadoc/TSDoc** — accurate, take signatures from here.
- **`docs/ARCHITECTURE.md`** — current through the `?lang=` URL scheme and `hreflang` alternates (§6.4, §6.6,
  §12.7, 0.12.0). Correct on the `data` block and `backendOwned` (§7.2), the `tags` and `external` blocks and
  their refusal shapes (§7.2), `nav[]` and the default entry a page plugin gets without one (§7.2 — but see
  the `role`/`visibleTo` drift below), the `USER` scope, `queryAcrossUsers` and every extension point
  including `SearchProvider`, `PageRouteProvider` and `UserDataHandler` (§7.4), the schema provider **and its
  read HTTP surface** (§7.6), the `blobs` manifest block and file storage (§7.2, §11, §11.1), site-wide
  search (§6.7), account deletion reaching plugin data (§12.8), languages as a runtime registry and
  host-mediated translation (§12.7, §16), and purge (§7.8). §7.5's `ctx` block lists `docs`, `feeds`, `tags`,
  `schema`, `blobs`, `links`, `locale.available/content`, `translation` and `route.navigate`. §6.6 now says
  `og:locale` is the language of *that URL*, not an install-wide constant — read this before implementing
  `ShareMetadataProvider`, since it is the reason `OgMeta.locale` exists (see `backend.md`).
  Still stale on: the `"platformApi": "1.x"` example (that string does not even parse), the example's
  `placement: "admin"` slot (accepted by validation, rendered nowhere), §7.3's region list (missing `page`),
  §7.5's `ctx` block (still missing `episodeLabels`, `log`, `consent.granted/request` and the `Unsubscribe`
  returns), and a doc-store "indexable fields" escape hatch that was never implemented.
- **The SDK's `PluginNavDeclaration` TS type disagrees with core on one field name.** The TS type
  (documentation-only, like the rest of `PluginManifest`) calls it `role`; core's `NavEntry` record — what
  actually parses `plugin.json` — calls it `visibleTo`. Write `visibleTo`; core is authoritative on the
  manifest, full stop.
- **`mosaicast-plugin-bingo` / `-stats` / `-wiki`** — bootstrap-only repos carrying a **pre-0.4.0**
  `ARCHITECTURE.md` copy and BRIEFs that specify the legacy consent shape and the per-user-key IDOR
  (`card:fan:{userId}`). Treat their docs as untrusted; the SDK and the sample win.

## Conventions

- SPDX header in every source file: `SPDX-License-Identifier: <license>` +
  `SPDX-FileCopyrightText: 2026 The Mosaicast Authors` (fixed holder — never from git config).
  **AGPL-3.0-or-later** for official plugins (bingo/stats/wiki), **Apache-2.0** for the sample and SDK.
- Sign commits (`git commit -s`, DCO).
- Java packages `dev.mosaicast.plugin.<name>.*`; npm scope `@mosaicast`.
- Keep README/CLAUDE.md current; `ARCHITECTURE.md` and `BRIEF.md` are read-only specs.
- Design every component to **fail gracefully**: the host wraps each slot mount in an error boundary, so a
  thrown error blanks your tile only — but that tile is all the user sees.

## Definition of done

From the repo's `docs/BRIEF.md`. Generally: a manifest that loads, at least one slot rendering through `ctx`
with theme tokens applied, data in the store, tests green, and a `dist/` that core loads and shows.
