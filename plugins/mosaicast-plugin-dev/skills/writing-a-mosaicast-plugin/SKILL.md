---
name: writing-a-mosaicast-plugin
description: Use when creating or modifying a Mosaicast plugin (any mosaicast-plugin-* repo or the plugin-sample). Covers the plugin.json manifest (slots and placements, the page slot behind /p/<id>/*, data access floors, backendOwned keys, consent services, doc vs schema storage), the backend PluginBackend/PluginContext contract, per-user data in the USER scope, the frontend Web Component via the SDK ctx and theme tokens, and testing against the SDK test kit. Trigger whenever writing the manifest, adding a slot, wiring ctx, storing per-user data, declaring backend-owned or schema storage, bumping platformApi, or building a plugin's backend or frontend.
---

# Writing a Mosaicast plugin

Everything here was checked against the SDK and core working trees, not against the changelogs. Where a
doc and this skill disagree, see "Which docs to trust" below.

## Version pins — get these wrong and the plugin does not load

Contract version is **0.6.0** (`PlatformApi.VERSION`, `PLATFORM_API_VERSION`). Core 0.6.x demands an
**exact `major.minor`** match; patch is free. A `0.5.x` manifest is *rejected at load*, not warned about.

```json5
"platformApi": "0.6.0"                                  // plugin.json
```
```kotlin
compileOnly("dev.mosaicast:plugin-api:0.6.0")           // backend/build.gradle.kts
compileOnly("org.pf4j:pf4j:3.12.0")
annotationProcessor("org.pf4j:pf4j:3.12.0")             // mandatory: generates the @Extension index
testImplementation("dev.mosaicast:plugin-testkit:0.6.0")
```
```json5
"@mosaicast/plugin-sdk": "0.6.0"                        // frontend/package.json
```

Maven is GitHub Packages (`https://maven.pkg.github.com/Mosaicast/mosaicast-plugin-sdk`), which needs a PAT
with `read:packages` **even for public reads**. SDK master has **no `v0.6.0` tag yet** — until it is
published, resolve via `mavenLocal()` or `includeBuild("../mosaicast-plugin-sdk")` against a local checkout.
Jackson is **3.2.1** (`tools.jackson.*`), not `com.fasterxml`.

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
   `{ "scope": "site", "placement": "page" }`.
6. **Every `onChange` returns an `Unsubscribe`** — return it from your render's cleanup, or a detached shadow
   root keeps receiving callbacks.

## Before you build: ask for a browser and an instance

Do this **once, at the start** of plugin work. Both are recommended, **neither is a blocker** — if the user
has neither, build the plugin in full anyway and say at the end what could not be verified.

1. **Check whether you have browser tooling** (Playwright MCP or equivalent). If not, tell the user once
   that you can still build and unit-test but cannot see the tile render, catch a CSP refusal or check a
   phone width — and that enabling a browser tool would fix that. Then carry on; do not ask again.
2. **Ask for a Mosaicast instance** you may install into and **restart** (a restart is how core picks up a
   rebuilt plugin), and **ask whether it holds production or test data**. If the user has a `mosaicast-core`
   checkout, `dev/screenshots.sh up` stands up a disposable seeded stack — best option, nothing real at risk.

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
shortcut it, never require it).

Three distinct URL namespaces, don't conflate them:
- `/api/plugins/<id>/data/*` — the host's fixed generic doc-store API (you do not author routes)
- `/plugins/<id>/assets/*` — static frontend bundle
- `/p/<id>/*` — deep-linkable plugin pages, surfaced as `ctx.route`

## Where to read next

| You are doing | Read |
|---|---|
| `plugin.json`: fields, slots, placements, floors, `backendOwned`, consent, config, schema declaration | `references/manifest.md` |
| Java backend: `PluginContext`, `DocStore`, `SchemaStore`, aggregates, scheduling, extension points | `references/backend.md` |
| Web Component: `ctx` surface, what the host actually implements, doc-store HTTP calls, CSP, theme, i18n | `references/frontend.md` |
| Tests (required by the BRIEF's DoD) | `references/testing.md` |
| Browser + live-instance setup, viewport matrix, data-safety rules, the build→install→restart loop | `references/dev-environment.md` |
| Moving an existing plugin from 0.5.0 to 0.6.0 | `references/migrating.md` |

Live reference implementation: **`mosaicast-plugin-sample` v2.7.0** (on SDK 0.6.0). Its `README.md` carries
the worked `curl` forgery that motivates `backendOwned`, and its `docs/BRIEF.md` pattern is the definition
of done for every plugin repo.

## Which docs to trust

- **SDK `README.md` / `CHANGELOG.md` / `MIGRATION.md` and the Javadoc/TSDoc** — accurate, take signatures from here.
- **`docs/ARCHITECTURE.md`** — now correct on the `data` block and `backendOwned` (§7.2), the `USER` scope,
  `queryAcrossUsers` and the extension points (§7.4), backend-owned keys and the schema provider (§7.6), and
  purge (§7.8). Still stale on: the `"platformApi": "1.x"` example (that string does not even parse), the
  example's `placement: "admin"` slot (accepted by validation, rendered nowhere), §7.3's region list (missing
  `page`), §7.5's `ctx` block (missing `episodeLabels`, `log`, `consent.granted/request`, and the
  `Unsubscribe` returns), and a doc-store "indexable fields" escape hatch that was never implemented.
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
