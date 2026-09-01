# Migrating an existing plugin up to 0.11.0

The SDK's own `MIGRATION.md` (in the `mosaicast-plugin-sdk` checkout, or `v0.11.0/MIGRATION.md` on GitHub)
is the authoritative checklist for the **SDK** half of each step — read it, it is short and version-scoped.
This file adds two things that doc does not: the **core-side** changes each release shipped alongside it,
and one file walking the **whole chain** for a plugin that has not moved since 0.8.0.

**You have no choice about timing on every step below.** `platformApi` matches `major.minor` **exactly** —
the moment core runs a new minor, every plugin declaring the old one is rejected at load, reason in the
admin log viewer. Patch releases (0.9.0 → 0.9.1) are the one exception: the manifest may stay put.

Coming from `0.7.x` or earlier? Do the [0.8.0 migration](https://github.com/Mosaicast/mosaicast-plugin-sdk/blob/v0.8.0/MIGRATION.md)
first (file storage, `ctx.links`), then start here.

## The chain, in order

| Step | Manifest change required? | Compile break? | What it's for |
|---|---|---|---|
| 0.8.x → 0.9.0 | Yes (`platformApi`) | Yes — a hand-built `ctx` in tests | tags, `ctx.feeds`, `ctx.docs`, typed API errors, `SearchProvider`, `UserDataHandler`, `nav[]` |
| 0.9.0 → 0.9.1 | **No** (patch) | No | `PageRouteProvider` — real 404s for unknown subpaths |
| 0.9.x → 0.10.0 | Yes | Yes — a hand-built `ctx` in tests | `ctx.locale.available/content`, `ctx.translation` |
| 0.10.x → 0.11.0 | Yes | **No — the trap is silent** | `external` manifest block gates `ctx.translation` |

Do them **in order**; do not skip to 0.11.0 and back-port the manifest fields, because 0.9.0's compile
break and 0.10.0's `ctx.translation` addition both have to land first for the 0.11.0 step to make sense.

---

## 0.10.x → 0.11.0: declaring what external services you use

```diff
  // plugin.json
- "platformApi": "0.10.0",
+ "platformApi": "0.11.0",
```

```diff
- implementation("dev.mosaicast:plugin-api:0.10.0")
+ implementation("dev.mosaicast:plugin-api:0.11.0")
- "@mosaicast/plugin-sdk": "^0.10.0"
+ "@mosaicast/plugin-sdk": "^0.11.0"
```

**The one that will catch you: `ctx.translation` goes `null` until you declare, and nothing warns you.**
`translation` was already `TranslationClient | null`, so the type system is silent — the handle simply
becomes `null` at runtime and you find out because a translate button stopped working.

If your plugin used `ctx.translation`/`ctx.translation()` on 0.10.0, add:

```diff
  // plugin.json
+ "external": { "kinds": ["translation"], "usedBy": "podcaster" }
```

Nothing else changes — your existing null-check already covers the right code, now for two indistinguishable
reasons instead of one: your manifest did not ask, or the operator configured no provider. **Unexpected
`null`? Check the manifest before the admin panel.** `usedBy` defaults to `podcaster`; leave it unless you
have a specific reason to widen it, and never to `anonymous` without one — it is legal but opens a metered
API to anyone who loads the page.

**Core-side:** core 0.6.23 built the validator that enforces this — `external.kinds` naming an unknown kind,
or an empty `kinds`, is **rejected at load**; `usedBy: anonymous` loads but logs an admin-visible warning.
`ctx.translation()` on the Java side is gated by the declared kind alone; `usedBy` is browser-only and
ignored server-side.

---

## 0.9.x → 0.10.0: the site's languages, and translation

```diff
  // plugin.json
- "platformApi": "0.9.1",
+ "platformApi": "0.10.0",
```

**The one compile break:** `PluginContext` gained `translation`, and `ctx.locale` gained `available()` and
`content()`. A hand-built `ctx` literal in a test stops compiling — `tsc --noEmit` catches it, not your test
runner. Use `makeMockCtx()` instead of hand-building.

**What you may now want:** `ctx.locale.content()` if your plugin authors anything per language — languages
are a runtime registry an admin edits, so build editor tabs from `content()`, never `available()` (a site
can require content in a language its UI does not offer). `ctx.translation` if you have text worth
translating — remember **markdown is neither `'text'` nor `'html'`**, and machine output is a draft a human
must confirm before it is shown as fact.

**Core-side:** core built the locale registry (`GET /api/i18n/locales`, `MOSAICAST_LOCALES_DIR` drop-in
catalogs merging key-by-key) and a LibreTranslate-backed translation provider behind the admin's external-
services settings — the operator half of the 0.11.0 gate above. **Core did not implement `ctx.translation`
before this step** — on any host older than the one hosting `platformApi 0.10.0`, the handle was always
`null`.

---

## 0.9.0 → 0.9.1: real 404s for your unknown subpaths

**Nothing mandatory.** A patch: `platformApi` matches `major.minor`, so a manifest declaring `0.9.0` keeps
loading against a `0.9.1`+ host unchanged — bump the *dependency* only when you want the new interface:

```diff
- implementation("dev.mosaicast:plugin-api:0.9.0")
+ implementation("dev.mosaicast:plugin-api:0.9.1")
```

**What it adds:** every subpath under a `page` slot answers `200` today — a typo'd slug and a page deleted
last year both render your not-found view inside a real `200 OK`, which a crawler indexes as content.
Implement `PageRouteProvider.hasRoute(subpath)` (add the class to `backend.extensions`, no manifest
declaration needed) and the host turns a `false` into a real `404`.

```java
public final class WikiRoutes implements PageRouteProvider {
    @Override
    public boolean hasRoute(String subpath) {
        return subpath.isEmpty()                        // your own root — do not forget it
                || subpath.startsWith("_search/")
                || pages.exists(slugOf(subpath));
    }
}
```

Two traps: **the root is a route** (`subpath` is empty at `/p/<id>/`) — a lookup written purely over your
own known slugs answers `false` there and 404s your own landing page, which is why
`PageRouteProviderHarness` always probes it. And **do not reuse `ShareMetadataProvider`** — your subtree can
legitimately hold views with nothing to describe (a search-result page) that still exist; `metaFor` says
how to describe a page, `hasRoute` says whether it exists, and conflating them 404s working routes.

**Core-side:** implemented in core's `PluginPageController` — absent means today's `200`-for-everything
behaviour, so this step is purely additive and safe to skip until you want it.

---

## 0.8.x → 0.9.0: the release that came out of using the contract

```diff
  // plugin.json
- "platformApi": "0.8.0",
+ "platformApi": "0.9.0",
```

```diff
- implementation("dev.mosaicast:plugin-api:0.8.0")
+ implementation("dev.mosaicast:plugin-api:0.9.0")
- "@mosaicast/plugin-sdk": "^0.8.0"
+ "@mosaicast/plugin-sdk": "^0.9.0"
```

Take `0.9.1` as your dependency once you are through this section (the manifest still says `0.9.0` — the
patch floats) — see the step above.

**The one compile break:** `PluginContext` gained three members (`docs`, `feeds`, `tags`) and `PluginRoute`
gained two (`query`, `hash`). A hand-built `ctx` literal in a test stops compiling; the fix is not to add the
members but to stop hand-building the context:

```diff
-const ctx = { scope: { type: 'site', id: 'main' }, episodes: [], user: null, /* … */ } as PluginContext;
+const ctx = makeMockCtx({ scope: { type: 'site', id: 'main' } });
```

On the Java side there is **no** break: `FakePluginContext` gained `withTags(...)` as a chaining mutator, so
every existing constructor call still compiles.

**Delete the code the SDK now owns, if you have it:**

- A `declaredType(file)` helper — `blobs.upload` normalises the declared MIME type by default now.
- A `formatTime`/`formatBytes` pair that hardcodes `.` as the decimal separator — use
  `i18n.duration(seconds)` / `i18n.bytes(quota.usedBytes)` from `createPluginI18n`.
- An English-shaped `n === 1 ? … : …` plural — use `i18n.plural('moments', n)` with catalog keys
  `moments.one`/`moments.other`.
- A hand-rolled icon-CSS file — `iconCss(['star', 'clock'])` replaces it, and fixes a real bug if your
  fallback was `mask-image: none` (renders **solid squares** on any host missing an icon you used).
- A `flush()` helper counting microtask hops — `flushMockApi(ctx.api)` does it for you.
- A private doc-store path builder — `ctx.docs.put('self', key, value)` replaces
  `` ctx.api.put(`data/user/me/${encodeURIComponent(key)}`, value) ``.
- A projection of episode titles/artwork into your own doc store on a schedule — `ctx.feeds.displayMany(...)`
  reads the host's snapshot live; deleting the projection usually deletes a `backendOwned` key with it.
- A private search box — implement `SearchProvider` instead. **Read the access note first**: it is the one
  extension point where the host does not filter for you, so a leak here is yours to prevent.
- Personal data held in **schema columns or blobs** with no story for account deletion — implement
  `UserDataHandler`. Core drops `USER`-scope documents itself; it has no idea which of your own columns is a
  person.

**What did not change:** `ctx.api`, `ctx.store()`, `ctx.schema`, `ctx.blobs`, `ctx.links`, `ctx.consent`,
`ctx.route.navigate`, `ctx.player`, `ctx.progress`, `ctx.theme`, both access floors, `backendOwned`, the
`USER`-scope exemptions, `FakePluginContext`'s existing constructors.

**Core-side:** core shipped the host half of all of this in the same release wave — `ctx.feeds`/`ctx.docs`/
`ctx.tags` are real (not stubs), site-wide search groups plugin hits by source at `/api/search?q=`, and
account deletion now calls every installed `UserDataHandler` before dropping the row, with an open-record
receipt rather than a bare success for the plugins that have not finished.

---

## After every bump

```bash
./gradlew build && npm ci && npm run build && npm run typecheck && ./build.sh
```

Copy `dist/` into `$MOSAICAST_PLUGINS_DIR/<id>`, restart core, and **check the admin log viewer at
startup** — a rejected manifest disables only your plugin, quietly, with its reason there. A `platformApi`
minor mismatch is the single most common reason a plugin is silently absent from the admin list.
