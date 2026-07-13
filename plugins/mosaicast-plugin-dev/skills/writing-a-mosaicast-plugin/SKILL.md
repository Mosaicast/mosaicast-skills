---
name: writing-a-mosaicast-plugin
description: Use when creating, scaffolding, or modifying a Mosaicast plugin (any mosaicast-plugin-* repo or the plugin-sample). Covers the plugin.json manifest (slots, scopes, doc vs schema storage, consent), the backend PluginBackend/PluginContext contract, the frontend Web Component via the SDK ctx and theme tokens, build.sh packaging to dist/, installing into MOSAICAST_PLUGINS_DIR, and testing against the SDK test kit. Trigger whenever writing the manifest, adding a slot, wiring ctx, building a plugin's backend or frontend, or setting up build.sh.
---

# Writing a Mosaicast plugin

Follow the house pattern for every Mosaicast plugin. This skill orchestrates using the repo's own docs — it does not replace them.

## Before you start
1. Read `docs/ARCHITECTURE.md` §7 (plugin system, also §6 for scope/theme/i18n) and `docs/BRIEF.md` in this repo. `ARCHITECTURE.md` is identical across all plugin repos (single source of truth); the brief adds only repo-specific detail. On conflict, ARCHITECTURE wins.
2. A plugin depends **only** on `mosaicast-plugin-sdk`. Never import core code, and stay **Spring-free** (PF4J classloading inside core's Spring Boot fat JAR is fiddly — plain PF4J extensions are the mitigation). Take exact signatures from the SDK source/Javadoc/TSDoc — don't guess.
   - Current SDK version: **0.2.0**. `platformApi` in your manifest must match it — don't copy the illustrative `"1.x"` from the architecture doc's example, use the real current version (or a range that includes it).
   - Java: `compileOnly("dev.mosaicast:plugin-api:0.2.0")`, `testImplementation("dev.mosaicast:plugin-testkit:0.2.0")` (groupId `dev.mosaicast`). Published to GitHub Packages (`https://maven.pkg.github.com/Mosaicast/mosaicast-plugin-sdk`, needs a PAT with `read:packages`) or use `mavenLocal()`/a composite build (`includeBuild("../mosaicast-plugin-sdk")`) against a local checkout.
   - TS: `npm install @mosaicast/plugin-sdk@0.2.0` (+ `/testing` subpath for tests).
3. Data lives in the generic **doc store** unless you specifically need relational features (full-text search, revisions, backlinks) — see Storage below. Any plugin may declare a schema; it's not wiki-exclusive, wiki is just the current adopter.

## A plugin is one folder that builds to `dist/`
Backend JAR (PF4J extension) + frontend Web Component bundle + `plugin.json`. No forced internal layout — core only cares about what lands in `dist/`.

### 1. Manifest — `plugin.json`
Key fields: `id`, `version`, `platformApi`, `backend` (`basePath`, `extensions`), `frontend` (`entry`, `elements`), `slots[]`, `storage`, `config`, `consent`.
A slot is `{ scope: site|feed|season|episode, element, placement, visibleTo: anonymous|fan|podcaster, order? }`. `placement` is a named region the **host shell** defines per view — the full current set is `top, card, main, sidebar, player, feed, site, admin`. Multiple plugins in one region stack vertically by `order` (ties: plugin ID alphabetical). Target only regions the host defines; an unknown region is rejected at startup. `card` is the compact one-liner shown on episode feed cards — declare it or omit it; full rendering goes in `main`.

Endpoints span three distinct URL namespaces — don't conflate them: backend API `/api/plugins/<id>/*`, static frontend assets `/plugins/{id}/assets/*`, and deep-linkable plugin pages `/p/<pluginId>/*` (mapped to `ctx.route` in the frontend).

### 2. Backend — Java, PF4J extension
Implement `PluginBackend.register(PluginContext ctx)`. From `ctx`:
- `store()` → `DocStore`: `get(Scope, key, Class<T>)`, `put(Scope, key, Object)` (last-write-wins — model concurrency yourself, e.g. per-user keys, if that matters), `query(Scope, keyPrefix)`.
- `schema()` → `SchemaStore` (only non-null if the manifest declares a schema): `namespace()`.
- `feeds()` → `FeedAccess`: `episodesIn(scope)`, `display(refId)` → `DisplaySnapshot` (not authoritative, host overwrites on every feed refetch).
- `config()` → `PluginConfig`: `get(key, Class<T>)` / `get(key, Class<T>, fallback)`.
- `onSchedule(Duration every, Runnable task)` — ShedLock-wrapped, runs at most once across instances.
- `Scope` = record `(ScopeType: SITE|FEED|SEASON|EPISODE, id)`, factories `Scope.site(id)` etc. `Role` enum = `ADMIN|PODCASTER|FAN` (anonymous = `user == null`, not a Role value).

Optional extension points (implement zero, one, or both): `ShareMetadataProvider.metaFor(subpath) → Optional<OgMeta>` (title/description non-null, imageUrl nullable — powers link previews under `/p/<id>/*`), `SitemapProvider.urls() → List<SitemapUrl>` (loc non-null, lastModified nullable).

Package `dev.mosaicast.plugin.<name>.*`.

### 3. Frontend — Web Component via the SDK
Use `defineMosaicastElement({ tag, render })` from `@mosaicast/plugin-sdk`. Re-assigning `ctx` re-renders (running any cleanup your previous render returned); calling with a tag already defined is a no-op. `ctx` fields, exactly:
```
scope: { type: site|feed|season|episode, id }
episodes: string[]                                    // resolved, access-filtered by host
episode?: { status: PLANNED|PUBLISHED|WITHDRAWN }      // present on episode scope
user: { id, role: admin|podcaster|fan } | null         // null = anonymous
api: { get/post/put/delete<T>(path, body?) }           // calls /api/plugins/<id>/*, auth attached
consent: { has(cat), onChange(cb) }
filter: { current(), onChange(cb) }                    // read-only — plugins consume, never define axes
player: { currentTime(), seekTo(s), on(ev, cb) }
route: { path, onChange(cb) }                           // subpath under /p/<id>/
locale: { current(), onChange(cb) }
progress: { get(episodeId) → Promise<number|null> }     // core listening progress, seconds
theme: ThemeTokens
```
The SDK auto-injects `theme` into the shadow root as CSS custom properties — you don't do this by hand: `bg→--mc-bg, surface→--mc-surface, text→--mc-text, textMuted→--mc-text-muted, accent→--mc-accent, accentContrast→--mc-accent-contrast, accent2→--mc-accent-2, border→--mc-border`. Just reference `var(--mc-*)` in your styles.

## i18n & deep links
- UI strings via `createPluginI18n(catalogs, ctx.locale)` with `locales/en.json` (source, also the fallback) + others; resolution is active locale → `en` → the key itself. Feed/author content is data, not UI — don't translate it.
- Design components to **fail gracefully**: the host wraps every slot mount in an error boundary — a thrown error blanks only your tile, never the page.

## Storage: doc vs schema
Default = doc store (`plugin_data`, scope+key JSONB, GIN-indexed). For relational needs, declare a schema instead: `"storage": { "schema": { "page": { "slug": "string:indexed:unique", "title": "string", "markdown": "text:fulltext", "updatedAt": "timestamp:indexed" } } }`. The platform provisions namespaced tables (`plugin_<id>_*`) via its own migration runner — **the plugin never writes DDL**.

## Build & install
`build.sh` builds backend + frontend and writes `dist/` (JAR + `assets/` + `plugin.json`). **`build.sh` writes only to `dist/` and never touches core.** Distribution is a separate, manual step: copy `dist/` into `$MOSAICAST_PLUGINS_DIR` and restart core. An optional `install.sh` may shortcut the copy only if the variable is set — never require it. `mosaicast-plugin-sample/docs/BRIEF.md` has the canonical `build.sh`/`install.sh` template to copy from.

## Lifecycle & failure isolation
A broken or `platformApi`-incompatible plugin is **disabled with an admin warning** at startup — core keeps booting, it never crashes the host over one bad plugin. Deleting a plugin's folder makes it dormant; its stored data is retained until an admin explicitly purges it.

## Tests (required — ARCHITECTURE §13.5)
- Backend: unit-test with the Java `plugin-testkit` (`dev.mosaicast.plugin.testkit.*`) — `FakePluginContext`, `InMemoryDocStore`, `FakeFeedAccess`, `MapPluginConfig`. No core, no DB; `onSchedule` runs synchronously.
- Frontend: mount the Web Component with `makeMockCtx(overrides)` from `@mosaicast/plugin-sdk/testing` (defaults include `DEFAULT_THEME`, a mock `api` that records calls), assert the DOM.

## Conventions
- SPDX header in every source file: `SPDX-License-Identifier: <license>` + `SPDX-FileCopyrightText: 2026 The Mosaicast Authors`. License per repo: **AGPL-3.0-or-later** for official plugins (bingo/stats/wiki), **Apache-2.0** for the sample.
- Sign commits (`git commit -s`, DCO).
- Keep README/CLAUDE.md current; ARCHITECTURE.md and BRIEF.md are read-only specs.

## Definition of done
Comes from `docs/BRIEF.md` — check it. Generally: valid manifest, at least one slot renders through `ctx` with theme tokens applied, data in the store, tests green, and a `dist/` that loads in core and shows in its slots.
