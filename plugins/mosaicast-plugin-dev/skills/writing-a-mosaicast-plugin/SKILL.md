---
name: writing-a-mosaicast-plugin
description: Use when creating, scaffolding, or modifying a Mosaicast plugin (any mosaicast-plugin-* repo or the plugin-sample). Covers the plugin.json manifest (slots, scopes, doc vs schema storage, consent), the backend PluginBackend/PluginContext contract, the frontend Web Component via the SDK ctx and theme tokens, build.sh packaging to dist/, installing into MOSAICAST_PLUGINS_DIR, and testing against the SDK test kit. Trigger whenever writing the manifest, adding a slot, wiring ctx, building a plugin's backend or frontend, or setting up build.sh.
---

# Writing a Mosaicast plugin

Follow the house pattern for every Mosaicast plugin. This skill orchestrates using the repo's own docs — it does not replace them.

## Before you start
1. Read `docs/ARCHITECTURE.md` §7 (plugin system, also §6 for scope/theme/i18n) and `docs/BRIEF.md` in this repo. **ARCHITECTURE.md is stale against the released SDK on six points** (Jackson package, `SchemaStore` surface, `ctx.logger()`/`ctx.log()`, consent shape, manifest `consent.services[]`, `episodeLabels`) — for those, the SDK's own `README.md`/`CHANGELOG.md`/`MIGRATION.md` (checked out at `mosaicast-plugin-sdk/`) are the accurate source. Everything else in ARCHITECTURE still wins on conflict.
2. A plugin depends **only** on `mosaicast-plugin-sdk`. Never import core code, and stay **Spring-free** (PF4J classloading inside core's Spring Boot fat JAR is fiddly — plain PF4J extensions are the mitigation). Take exact signatures from the SDK source/Javadoc/TSDoc — don't guess.
   - Current SDK version: **0.4.0**. `platformApi` in your manifest must match it exactly on `major.minor` — don't copy the illustrative `"1.x"` from the architecture doc's example. **Core rejects a mismatch at load** (visible with its reason in the admin log viewer), so a manifest copied from an old sample plugin (`0.2.0`/`0.3.x`) fails to load, it doesn't just warn.
   - Java: `compileOnly("dev.mosaicast:plugin-api:0.4.0")`, `testImplementation("dev.mosaicast:plugin-testkit:0.4.0")` (groupId `dev.mosaicast`). Published to GitHub Packages (`https://maven.pkg.github.com/Mosaicast/mosaicast-plugin-sdk`, needs a PAT with `read:packages`) or use `mavenLocal()`/a composite build (`includeBuild("../mosaicast-plugin-sdk")`) against a local checkout.
   - TS: `npm install @mosaicast/plugin-sdk@0.4.0` (+ `/testing` subpath for tests).
3. Data lives in the generic **doc store** unless you specifically need relational features (full-text search, revisions, backlinks) — see Storage below. Any plugin may declare a schema in its manifest, but **core still rejects `storage: "schema"` at load** — the `SchemaStore` query surface shipped in the SDK in 0.4.0, but core hasn't dropped its rejection yet. Until it does, ship schema-declaring plugins as doc-store-only, or expect them to be disabled at startup. Check core's current release notes before relying on schema storage.

## A plugin is one folder that builds to `dist/`
Backend JAR (PF4J extension) + frontend Web Component bundle + `plugin.json`. No forced internal layout — core only cares about what lands in `dist/`.

### 1. Manifest — `plugin.json`
Key fields: `id`, `version`, `platformApi`, `backend` (`basePath`, `extensions`), `frontend` (`entry`, `elements`), `slots[]`, `storage`, `config`, `consent`.
A slot is `{ scope: site|feed|season|episode, element, placement, visibleTo: anonymous|fan|podcaster, order? }`. `placement` is a named region the **host shell** defines per view — the full current set is `top, card, main, sidebar, player, feed, site, admin`. Multiple plugins in one region stack vertically by `order` (ties: plugin ID alphabetical). Target only regions the host defines; an unknown region is rejected at startup. `card` is the compact one-liner shown on episode feed cards — declare it or omit it; full rendering goes in `main`.

`consent` is a **service-level** declaration, not a category list — the legacy `{ categories, externalSources }` shape is rejected from `0.4.0`:
```json
"consent": { "services": [{
  "id": "plausible", "name": "Plausible Analytics", "provider": "Plausible Insights OÜ",
  "category": "analytics", "privacyUrl": "https://plausible.io/privacy",
  "hosts": ["https://plausible.example"], "thirdCountryTransfer": false,
  "storage": [{ "name": "plausible_ignore", "type": "localStorage", "purpose": "opt-out marker", "duration": "persistent" }]
}] }
```
`hosts` needs the scheme and **is also the CSP allow-list** — an origin you didn't declare stays blocked even after consent is granted (the failure looks like "the embed just doesn't load", not a permissions error). `provider` is the operating company, not your plugin. The visitor decides per **category**, not per service — two services sharing a category (yours and another plugin's) are granted or refused together; use `necessary` only for what genuinely can't be refused. No third-party services → omit `consent` entirely, site stays banner-free. The SDK exports `ConsentServiceDeclaration`/`ConsentStorageDeclaration` (TS) as documentation-only types to catch a typo in your editor — the SDK doesn't read `plugin.json`, core is still authoritative.

Endpoints span three distinct URL namespaces — don't conflate them: backend API `/api/plugins/<id>/*`, static frontend assets `/plugins/{id}/assets/*`, and deep-linkable plugin pages `/p/<pluginId>/*` (mapped to `ctx.route` in the frontend).

### 2. Backend — Java, PF4J extension
Implement `PluginBackend.register(PluginContext ctx)`. From `ctx`:
- `store()` → `DocStore`: `get(Scope, key, Class<T>)`, `put(Scope, key, Object)` (last-write-wins — model concurrency yourself, e.g. per-user keys, if that matters), `delete(Scope, key) → boolean` (idempotent), `query(Scope, keyPrefix)` → `List<DocEntry>`.
- `schema()` → `SchemaStore`, only non-null if the manifest declares a schema (and — see above — core doesn't load those yet regardless). Since 0.4.0 it's a real query surface, not just `namespace()`: `entities()`, `find(entity, id, Class<T>)`, `select(entity, Criteria, Class<T>)`, `search(entity, field, text, Criteria, Class<T>)` (the `text:fulltext` path — needs the field declared `:fulltext`), `count(entity, Criteria)`, `insert(entity, Map<String,Object>) → long id`, `update(entity, id, Map<String,Object>)`, `delete(entity, Criteria)`. Address entities/fields **by declared name only**, never SQL, never a table name — that's the whole scoping guarantee. `Criteria` is an immutable builder (`Criteria.where(field, Op, value)`/`.and(...)`/`.orderBy(...)`/`.limit(...)`/`.offset(...)`, `Criteria.all()` for everything) over `EQ NE LT LTE GT GTE IN LIKE IS_NULL IS_NOT_NULL` — **AND-only**, no `or` in this version. An undeclared entity or field throws `IllegalArgumentException`, not a runtime failure.
- `feeds()` → `FeedAccess`: `episodesIn(scope)`, `display(refId)` → `DisplaySnapshot` (not authoritative, host overwrites on every feed refetch).
- `config()` → `PluginConfig`: `get(key, Class<T>)` / `get(key, Class<T>, fallback)`.
- `logger()` → a plain SLF4J `Logger`, already named `plugin.<pluginId>` by the host. Use it, don't build your own with `LoggerFactory.getLogger(...)` — a self-named logger falls outside the `plugin.` prefix and the host can't attribute it to you or show it in the admin log viewer. Host persists `info`+ and surfaces `warn`+ there; a tight loop gets rate-limited, not stored.
- `onSchedule(Duration every, Runnable task)` — ShedLock-wrapped, runs at most once across instances.
- `Scope` = record `(ScopeType: SITE|FEED|SEASON|EPISODE, id)`, factories `Scope.site(id)` etc. `Role` enum = `ADMIN|PODCASTER|FAN` (anonymous = `user == null`, not a Role value).

Optional extension points (implement zero, one, or both): `ShareMetadataProvider.metaFor(subpath) → Optional<OgMeta>` (title/description non-null, imageUrl nullable — powers link previews under `/p/<id>/*`), `SitemapProvider.urls() → List<SitemapUrl>` (loc non-null, lastModified nullable).

Package `dev.mosaicast.plugin.<name>.*`.

**Jackson 3, since 0.4.0.** `DocEntry.value()` (from `store().query(...)`) is `tools.jackson.databind.JsonNode`, not `com.fasterxml.jackson...` — `plugin-api` now depends on `tools.jackson.core:jackson-databind:3.1.4`. If you build an `ObjectMapper` yourself (e.g. for `InMemoryDocStore` in tests), use `JsonMapper.builder().build()` — mappers are immutable in Jackson 3, no bare `new ObjectMapper()`. `JacksonException` is unchecked now, so drop any `throws JsonProcessingException`/catch-only-to-satisfy-the-compiler blocks. **`store().get(scope, key, Class<T>)`, `config().get(...)` and `schema()` all deserialize straight into your own type and never hand you a `JsonNode`** — only `store().query(...)` does. A plugin that never calls `query()` has nothing Jackson-shaped to change.

### 3. Frontend — Web Component via the SDK
Use `defineMosaicastElement({ tag, render })` from `@mosaicast/plugin-sdk`. Re-assigning `ctx` re-renders (running any cleanup your previous render returned); calling with a tag already defined is a no-op. `ctx` fields, exactly:
```
scope: { type: site|feed|season|episode, id }
episodes: string[]                                      // resolved, access-filtered by host — public slugs
episodeLabels?: Record<string, string>                   // slug → display label, may be absent/partial — fall back to the slug
episode?: { status: PLANNED|PUBLISHED|WITHDRAWN }        // present on episode scope
user: { id, role: admin|podcaster|fan } | null           // null = anonymous
api: { get/post/put/delete<T>(path, body?) }             // calls /api/plugins/<id>/*, auth attached
log(level: debug|info|warn|error, message: string): void // the ONLY way to log from frontend — never POST /api/plugins/{id}/log through ctx.api
consent: {
  has(category): boolean
  granted(): string[]
  request(category): Promise<boolean>                    // call from a user gesture only, never on mount
  onChange(cb): Unsubscribe
}
filter: { current(), onChange(cb): Unsubscribe }         // read-only — plugins consume, never define axes
player: { currentTime(), seekTo(s), on(ev, cb): Unsubscribe }
route: { path, onChange(cb): Unsubscribe }                // subpath under /p/<id>/
locale: { current(), onChange(cb): Unsubscribe }
progress: { get(episodeId) → Promise<number|null> }       // core listening progress, seconds
theme: ThemeTokens
```
**Every `onChange` (`consent`, `filter`, `route`, `locale`, `player.on`) returns an `Unsubscribe` since 0.4.0** — return it (or wrap it) from your render's cleanup callback so the SDK detaches it on unmount/re-render; a dropped subscription keeps firing into a detached shadow root. If you hand-roll a `PluginContext` for a test instead of using `makeMockCtx` (see Tests), every `onChange` must return a function, and `consent` must be a full `ConsentApi` (`has`/`granted`/`request`/`onChange`) — the old two-method shape no longer compiles.

`ctx.consent.request(category)` opens the host's consent settings and resolves with the visitor's answer — this is what powers a click-to-load placeholder; call it from the click handler, never from `render`/mount. Resolving `true` doesn't load anything for you — load the gated resource yourself afterwards. It's one host-wide surface: concurrent calls from multiple plugin tiles join the same dialog rather than opening several, and every call resolves exactly once. Re-check `ctx.consent.has(category)` on every load rather than caching a `request()` result — consent can be withdrawn mid-session from the settings page, which is exactly what `onChange` is for.

The SDK auto-injects `theme` into the shadow root as CSS custom properties — you don't do this by hand: `bg→--mc-bg, surface→--mc-surface, text→--mc-text, textMuted→--mc-text-muted, accent→--mc-accent, accentContrast→--mc-accent-contrast, accent2→--mc-accent-2, border→--mc-border`. Just reference `var(--mc-*)` in your styles.

## i18n & deep links
- UI strings via `createPluginI18n(catalogs, ctx.locale)` with `locales/en.json` (source, also the fallback) + others; resolution is active locale → `en` → the key itself. Feed/author content is data, not UI — don't translate it. It subscribes to `ctx.locale.onChange` for its whole life — call the returned `.dispose()` from your render's cleanup callback, or the subscription leaks (harmless against a `0.3.x` host, whose `onChange` returns nothing — `dispose()` no-ops safely there too).
- Design components to **fail gracefully**: the host wraps every slot mount in an error boundary — a thrown error blanks only your tile, never the page.

## Storage: doc vs schema
Default = doc store (`plugin_data`, scope+key JSONB, GIN-indexed). For relational needs, declare a schema instead: `"storage": { "schema": { "page": { "slug": "string:indexed:unique", "title": "string", "markdown": "text:fulltext", "updatedAt": "timestamp:indexed" } } }`. The platform provisions namespaced tables (`plugin_<id>_*`) via its own migration runner — **the plugin never writes DDL**. As of the SDK's `0.4.0`, `SchemaStore` has a real query surface (see Backend above) — but **core still rejects `storage: "schema"` at load** until it ships the matching release, so a schema-declaring plugin won't load against current core regardless of SDK version. Default to doc-store-only until that lands.

## Build & install
`build.sh` builds backend + frontend and writes `dist/` (JAR + `assets/` + `plugin.json`). **`build.sh` writes only to `dist/` and never touches core.** Distribution is a separate, manual step: copy `dist/` into `$MOSAICAST_PLUGINS_DIR` and restart core. An optional `install.sh` may shortcut the copy only if the variable is set — never require it. `mosaicast-plugin-sample/docs/BRIEF.md` has the canonical `build.sh`/`install.sh` template to copy from.

## Lifecycle & failure isolation
A broken or `platformApi`-incompatible plugin is **disabled with an admin warning** at startup — core keeps booting, it never crashes the host over one bad plugin. Deleting a plugin's folder makes it dormant; its stored data is retained until an admin explicitly purges it.

## Tests (required — ARCHITECTURE §13.5)
- Backend: unit-test with the Java `plugin-testkit` (`dev.mosaicast.plugin.testkit.*`) — `FakePluginContext`, `InMemoryDocStore`, `FakeFeedAccess`, `MapPluginConfig`, `FakeSchemaStore` (enforces the same declaration the host does), `RecordingLogger` (formatted messages, throwable captured separately — wired as `FakePluginContext.logger()`). No core, no DB; `onSchedule` runs synchronously.
- Frontend: mount the Web Component with `makeMockCtx(overrides)` from `@mosaicast/plugin-sdk/testing` (returns `MockPluginContext`, adds `.logs`; defaults include `DEFAULT_THEME`, a mock `api` that records calls, and a consent double from `makeMockConsent()` — `grant`/`revoke`/`requests`/`autoGrantOnRequest`, **deny-everything by default**), assert the DOM. Prefer this over hand-rolling a `PluginContext` fake — it stays in sync with `onChange`'s `Unsubscribe` return and `ConsentApi`'s shape across SDK bumps; a hand-rolled fake breaks on both every time the contract does.

## Conventions
- SPDX header in every source file: `SPDX-License-Identifier: <license>` + `SPDX-FileCopyrightText: 2026 The Mosaicast Authors`. License per repo: **AGPL-3.0-or-later** for official plugins (bingo/stats/wiki), **Apache-2.0** for the sample.
- Sign commits (`git commit -s`, DCO).
- Keep README/CLAUDE.md current; ARCHITECTURE.md and BRIEF.md are read-only specs.

## Definition of done
Comes from `docs/BRIEF.md` — check it. Generally: valid manifest, at least one slot renders through `ctx` with theme tokens applied, data in the store, tests green, and a `dist/` that loads in core and shows in its slots.
