---
name: writing-a-mosaicast-plugin
description: Use when creating, scaffolding, or modifying a Mosaicast plugin (any mosaicast-plugin-* repo or the plugin-sample). Covers the plugin.json manifest (slots, scopes, doc vs schema storage, consent), the backend PluginBackend/PluginContext contract, the frontend Web Component via the SDK ctx and theme tokens, build.sh packaging to dist/, installing into MOSAICAST_PLUGINS_DIR, and testing against the SDK test kit. Trigger whenever writing the manifest, adding a slot, wiring ctx, building a plugin's backend or frontend, or setting up build.sh.
---

# Writing a Mosaicast plugin

Follow the house pattern for every Mosaicast plugin. This skill orchestrates using the repo's own docs — it does not replace them.

## Before you start
1. Read `docs/ARCHITECTURE.md` §7 (plugin system) and `docs/BRIEF.md` in this repo. On conflict, ARCHITECTURE wins.
2. A plugin depends **only** on `mosaicast-plugin-sdk`. Never import core code. Take exact signatures from the built SDK's Javadoc/TSDoc — don't guess (ARCHITECTURE §3.5).
3. Data lives in the generic **doc store** unless the brief says schema provider (wiki only).

## A plugin is one folder that builds to `dist/`
Backend JAR (PF4J extension) + frontend Web Component bundle + `plugin.json`.

### 1. Manifest — `plugin.json`
Key fields: `id`, `version`, `platformApi` (must match the built SDK version — the host rejects a mismatch at startup), `backend` (`basePath`, `extensions`), `frontend` (`entry`, `elements`), `slots[]`, `storage`, `config`, `consent`.
A slot is `{ scope: site|feed|season|episode, element, placement: main|sidebar|admin|card, visibleTo: anonymous|fan|podcaster, order? }`. Multiple plugins in one region stack by `order`. Target only regions the host defines; an unknown region is rejected at startup.

### 2. Backend — Java, PF4J extension
Implement `PluginBackend.register(PluginContext ctx)`. From `ctx`: `store()` (hard-scoped doc store: `get/put/query` by `Scope`), `feeds().episodesIn(scope)`, `config()`, `onSchedule(...)` (ShedLock-wrapped). Endpoints live under `/api/plugins/<id>/*`. Package `dev.mosaicast.plugin.<name>.*`.

### 3. Frontend — Web Component via the SDK
Use `defineMosaicastElement` from `@mosaicast/plugin-sdk`. Read `this.ctx`: `scope`, `episodes[]` (resolved by the host), `episode?.status`, `user`, `api`, `consent`, `filter` (read-only — plugins consume filters, never define them), `player`, `route` (subpath under `/p/<id>/` for deep links), `locale` (+ `onChange`), `progress` (core listening progress), `theme`. **Inject `ctx.theme` tokens as CSS custom properties into the shadow root** so the plugin matches the host's light/dark. Talk to the backend via `ctx.api`.

## i18n & deep links
- UI strings via the SDK's `createPluginI18n(catalogs)` with `locales/en.json` (source) + `de.json`; react to `ctx.locale.onChange`. Feed/author content is data, not UI — don't translate it.
- Plugin pages live under `/p/<pluginId>/…` via `ctx.route`. If your content is worth sharing (like wiki pages), implement the optional `ShareMetadataProvider` (`metaFor(subpath) → OgMeta`) so shared links get a preview.
- If your plugin has crawlable pages, also implement the optional `SitemapProvider` (`urls()`) so they land in the site's sitemap.
- Design components to **fail gracefully**: the host error-bounds each slot — a thrown error blanks only your tile, but a defensive component degrades nicer.

## Storage: doc vs schema
Default = doc store (`plugin_data`, scope+key JSONB). Only the wiki uses the **schema provider**: declare entities in the manifest, the platform provisions namespaced tables via Flyway, **the plugin never writes DDL**.

## Build & install
`build.sh` builds backend + frontend and writes `dist/` (JAR + `assets/` + `plugin.json`). **`build.sh` writes only to `dist/` and never touches core.** Distribution is a separate, manual step: copy `dist/` into `$MOSAICAST_PLUGINS_DIR` and restart core. An optional `install.sh` may shortcut the copy only if the variable is set. The folder layout is not enforced.

## Tests (required — ARCHITECTURE §13.5)
- Backend: unit-test with `FakePluginContext` + `InMemoryDocStore` from the Java `plugin-testkit` — no core, no DB.
- Frontend: mount the Web Component with `makeMockCtx` from `@mosaicast/plugin-sdk/testing`, assert the DOM.

## Conventions
- SPDX header in every source file. License per repo: **AGPL-3.0-or-later** for official plugins (bingo/stats/wiki), **Apache-2.0** for the sample.
- Sign commits (`git commit -s`, DCO).
- Keep README/CLAUDE.md current; ARCHITECTURE.md and BRIEF.md are read-only specs.

## Definition of done
Comes from `docs/BRIEF.md` — check it. Generally: valid manifest, at least one slot renders through `ctx` with theme tokens applied, data in the store, tests green, and a `dist/` that loads in core and shows in its slots.
