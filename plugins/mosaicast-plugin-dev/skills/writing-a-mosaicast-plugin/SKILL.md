---
name: writing-a-mosaicast-plugin
description: Use when creating or modifying a Mosaicast plugin (any mosaicast-plugin-* repo or the plugin-sample). Covers the plugin.json manifest (slots and placements, the page slot behind /p/<id>/*, data access floors, backendOwned keys, consent services, doc vs schema storage, the blobs file-storage block, the tags vocabulary block, the external-services block, the identity block (ctx.users, resolving user UUIDs to a name/avatar), the notifications block (ctx.notify/ctx.notifier, putting a message in a user's inbox), nav entrances into the host menu, config fields with a localized label/description and a closed options set, the license/author/homepage/attribution credit fields), the backend PluginBackend/PluginContext contract (including onSchedule's Supplier<Duration> overload for a schedule that actually follows config) and its optional extension points (ShareMetadataProvider and its per-URL OgMeta.locale, SitemapProvider and its hreflang-alternates SitemapUrl.alternates, PageRouteProvider, SearchProvider, UserDataHandler), per-user data in the USER scope, the frontend Web Component via the SDK ctx (ctx.docs, ctx.feeds, ctx.tags, ctx.users, ctx.notify, ctx.schema reads, ctx.blobs uploads, ctx.links, ctx.locale.available/content, ctx.translation, ctx.route.navigate, typed ctx.api errors) plus defineMosaicastElement's MosaicastHandle (a render surviving a reassigned ctx instead of being torn down) and theme tokens including the --mc-icon-* icon set, testing against the SDK test kit, and installing a plugin by spec (owner/repo@tag#sha256). Trigger whenever writing the manifest, adding a slot, wiring ctx, storing per-user data, uploading or serving files, linking to core pages, querying schema tables from the frontend, reading or writing the shared tag vocabulary, contributing to site search, handling account deletion/export, real 404s for a page plugin's unknown subpaths, declaring external-service (translation) use, declaring what language a plugin page or sitemap translation group is written in, resolving user ids to a display name/avatar, sending an in-app notification to a user, adding a nav entrance, labeling or restricting a config field, making a scheduled task follow a configurable interval, keeping a component's state across a re-rendered ctx, navigating inside a page plugin, declaring backend-owned or schema storage, styling a plugin tile's icons, crediting a plugin's license or data source, bumping platformApi, installing or releasing a plugin, or building a plugin's backend or frontend.
---

# Writing a Mosaicast plugin

Everything here was checked against the SDK and core working trees, not against the changelogs. Where a
doc and this skill disagree, see "Which docs to trust" below.

## Version pins — get these wrong and the plugin does not load

Contract version is **0.15.0** (`PlatformApi.VERSION`, `PLATFORM_API_VERSION`), and core **0.7.2** hosts it.
The host demands an **exact `major.minor`** match; patch is free. A `0.14.x` manifest is *rejected at load*,
not warned about.

```json5
"platformApi": "0.15.0"                                 // plugin.json
```
```kotlin
compileOnly("dev.mosaicast:plugin-api:0.15.0")          // backend/build.gradle.kts
compileOnly("org.pf4j:pf4j:3.15.1")
annotationProcessor("org.pf4j:pf4j:3.15.1")             // mandatory: generates the @Extension index
testImplementation("dev.mosaicast:plugin-testkit:0.15.0")
```
```json5
"@mosaicast/plugin-sdk": "0.15.0"                       // frontend/package.json
```

Both halves of `0.15.0` are published on npm and GitHub Packages — no `mavenLocal()` workaround needed.
**The git tag is `0.15.0`, not `v0.15.0`** — every other SDK release follows `v<version>`, and this one
release doesn't; `git checkout v0.15.0` 404s. Maven is GitHub Packages
(`https://maven.pkg.github.com/Mosaicast/mosaicast-plugin-sdk`), which needs a PAT with `read:packages`
**even for public reads**. Jackson is **3.2.2** (`tools.jackson.*`), not `com.fasterxml`. **Pin the same
string in all four places** — the CI drift guard and the manifest contract test both compare them.

Coming from an older pin? `references/migrating.md` walks 0.8.0 → 0.9.0 → 0.9.1 → 0.10.0 → 0.11.0 → 0.12.0 →
0.13.0 → 0.14.0 → 0.15.0 in order — read it top to bottom rather than jumping straight to 0.15.0, since
0.9.0's compile break and 0.10.0's silent `ctx.translation` gate both still apply on the way up. 0.12.0 was
the easy step (manifest bump only); 0.13.0 and 0.14.0 each added one more optional, `null`-unless-declared
capability (`ctx.users`, `ctx.notify`); 0.15.0 fixes two real bugs — a configurable schedule that used to
ignore its own config, and a component destroyed several times a second during playback.

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
18. **`ctx.users` resolves, it does not enumerate, and its answer is absent-not-redacted.** An unknown,
    erased or pseudonymised id is simply *missing* from the result — no `null` element, no tombstone — so
    the array is not index-aligned with what you asked for and may be shorter. Match on `id`, never on
    position. **Store the UUID, resolve at render, never persist a `displayName`** — the host cannot enforce
    that one; a copied name outlives the rename and the erasure both meant to end it.
19. **`ctx.notify`/`ctx.notifier()` write into *another* user's experience — the one plugin surface that
    does.** Two bounds you cannot lift: you may only notify a user your plugin already holds `USER`-scope
    data for (the same partitions `queryAcrossUsers` reads), and the rate limits are the host's, not
    yours. `send()` returns who was **actually** notified — a partial send is normal, not a bug, so read the
    return value or a stale participant list notifies nobody while looking healthy. `NotifyMessage.text`
    must carry every language up front (`en` required) — there is no read side, and nothing here reaches
    email.
20. **Neither `identity` nor `notifications` is validated at load.** Unlike `tags`/`blobs`/`external`,
    core has no `validate*()` for either block — any shape loads, and an absent flag (`resolvesUsers`,
    `sends`) **defaults to `true`** the moment you declare the block at all, the opposite default from what
    the SDK's own TS types (which mark both required) imply. `"identity": {}` is `"identity": {
    "resolvesUsers": true }`.
21. **If your schedule's period comes from `ctx.config()`, use `onSchedule(Supplier<Duration>, Runnable)`,
    not the `Duration` overload.** The `Duration` form captures the value once, during `register()`, and
    holds it for the process's life — an operator saves a new interval, the admin form says it worked, and
    your plugin keeps running at the old one until the host restarts. The fix is a one-character diff:
    wrap the read in a lambda (`() -> Duration.ofSeconds(ctx.config().get(...))`); the host re-reads it
    before every tick. Keep the `Duration` overload only where the cadence is genuinely fixed.
22. **A render returning a bare cleanup callback is destroyed and rebuilt on *every* `ctx` reassignment —
    which can be several times a second during playback.** Return a `MosaicastHandle` (`{ update?, destroy?
    }`) instead and you decide what a new `ctx` costs: `update(next)` runs in place, `root` stays untouched,
    and `destroy` fires only on a real disconnect. A static component with no state needs no change; one
    holding component state, in-flight requests, or scroll/dialog state should switch or it is silently
    losing all of that several times a second. An **identical** `ctx` object is ignored either way — you
    never have to diff it yourself.

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
| Moving an existing plugin from 0.8.x up to 0.15.0 | `references/migrating.md` |

Live reference implementation: **`mosaicast-plugin-sample` v2.15.0**, now on **SDK 0.14.0** — a real jump
from the long 0.8.0 stretch this skill used to warn about, and worth a full re-read rather than trusting
the old warning. It now declares (and exercises) `identity`, `notifications`, `tags`, `external`, `blobs`
and a four-entry `nav[]` (including a `visibleTo: "podcaster"` entry — using the correct field name, not
the SDK type's `role`) alongside the things it always had: `ctx.blobs` (upload, `urlFor`, the `null` degrade
path), `ctx.links.episode/feed`, and a page slot using `ctx.route.navigate`. Its `README.md` carries the
worked `curl` forgery that motivates `backendOwned`, and its `docs/BRIEF.md` pattern is the definition of
done for every plugin repo. One minor behind current SDK (0.14.0, not yet 0.15.0) as of this writing — check
its tag before trusting a version number quoted here; it moves independently of the SDK and has lagged it
before.

## Which docs to trust

- **SDK `README.md` / `CHANGELOG.md` / `MIGRATION.md` and the Javadoc/TSDoc** — accurate, take signatures from here.
- **`docs/ARCHITECTURE.md`** — current through §8.8 (`ctx.users`, the `identity` block) and mostly current
  on §17 (notifications) — **but see the flat-out wrong code block below**, which is worth reading before
  the rest of this bullet. Correct on the `data` block and `backendOwned` (§7.2), the `tags` and `external`
  blocks and their refusal shapes (§7.2), `nav[]` and the default entry a page plugin gets without one
  (§7.2 — but see the `role`/`visibleTo` drift below), the `USER` scope, `queryAcrossUsers` and every
  extension point including `SearchProvider`, `PageRouteProvider` and `UserDataHandler` (§7.4), the schema
  provider **and its read HTTP surface** (§7.6), the `blobs` manifest block and file storage (§7.2, §11,
  §11.1), the `?lang=` URL scheme and `hreflang` alternates (§6.4, §6.6, §12.7), site-wide search (§6.7),
  account deletion reaching plugin data (§12.8), languages as a runtime registry and host-mediated
  translation (§12.7, §16), and purge (§7.8). §7.5's `ctx` block lists `docs`, `feeds`, `tags`, `users`,
  `notify`, `schema`, `blobs`, `links`, `locale.available/content`, `translation` and `route.navigate`.
  §6.6 says `og:locale` is the language of *that URL*, not an install-wide constant — read this before
  implementing `ShareMetadataProvider`, since it is the reason `OgMeta.locale` exists (see `backend.md`).
  Still stale on: the `"platformApi": "1.x"` example (that string does not even parse), the example's
  `placement: "admin"` slot (accepted by validation, rendered nowhere), §7.3's region list (missing `page`),
  §7.5's `ctx` block (still missing `episodeLabels`, `log`, `consent.granted/request` and the `Unsubscribe`
  returns), and a doc-store "indexable fields" escape hatch that was never implemented.
- **§17.1's own TS snippet does not compile against anything real — do not copy it.** It still shows
  `NotifyClient.send(...): Promise<void>` and `NotifyMessage = { key, params?, link? }`, which is the
  *proposal* the SDK's own CHANGELOG explains it had to abandon: nothing in core can resolve a plugin's
  translation `key` (a plugin's catalogs live in its frontend bundle, and the bell renders on pages that
  bundle never loads), and `void` cannot express the partial send the host's eligibility rule guarantees.
  What actually ships — and what `backend.md`/`frontend.md` document — is `send(...): Promise<string[]>`
  (TS) / `List<UUID> send(...)` (Java) and `NotifyMessage.text` as a `locale → sentence` map built with
  `notifyText(catalogs, key, params)`. The prose in §17 (the eligibility rule, the rate-limit shape,
  in-app-only) is accurate; only this one code block is not.
- **The SDK's `PluginNavDeclaration` TS type disagrees with core on one field name.** The TS type
  (documentation-only, like the rest of `PluginManifest`) calls it `role`; core's `NavEntry` record — what
  actually parses `plugin.json` — calls it `visibleTo`. Write `visibleTo`; core is authoritative on the
  manifest, full stop.
- **Neither `identity` nor `notifications` is validated at load, unlike every other opt-in block.**
  `tags`/`blobs`/`external` reject a block that asks for nothing; `identity`/`notifications` have no
  `validate*()` method at all — `"identity": {}` and `"notifications": {}` load silently and *default their
  one flag to true* (`resolvesUsers`/`sends` default true when the field is absent, not false), unlike the
  SDK's TS types, which mark both fields required. A typo here is not a load-time error; it is a plugin
  that quietly got the capability it asked for.
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
