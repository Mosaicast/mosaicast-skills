# Migrating an existing plugin to 0.8.0

The SDK's own `MIGRATION.md` (in the `mosaicast-plugin-sdk` checkout) is the authoritative checklist for the
**SDK** half — read it, it is short. This file adds the **core-side** changes it does not cover, because
core moved too.

Coming from `0.6.x`? Do the 0.7.0 migration first (the frontend schema client and `route.navigate`), then
this.

## The SDK half, in brief

**A version bump, and nothing else** unless you implement `PluginContext` yourself. Everything in 0.8.0 is
new surface: no plugin code written against 0.7.1 changes behaviour, and **no existing test double breaks**
— `FakePluginContext`'s four-argument form still compiles and still means "no file storage".

1. `plugin.json` → `"platformApi": "0.8.0"`; `build.gradle.kts` → `plugin-api` / `plugin-testkit` `0.8.0`;
   `package.json` → `@mosaicast/plugin-sdk` `0.8.0`. Both halves are published and tagged `v0.8.0`. You have
   no choice about timing: the moment the host runs 0.8.x, every 0.7.x plugin is rejected at load.
2. **If you hand-roll a `PluginContext`** (Java) rather than using `FakePluginContext`, add `blobs()` —
   returning `null` is correct for a plugin that declares no `blobs` block. On the TS side a hand-rolled
   `ctx` needs `blobs` and `links` too; `makeMockCtx` is the cheaper answer.
3. **Optional: store files.** Declare a `blobs` block, then `ctx.blobs` (TS) / `ctx.blobs()` (Java) stops
   being `null`. Store the **ref**, never the URL; surface refusals to the person who picked the file; and
   delete what you stop pointing at, because nothing collects orphans.
4. **Optional: replace hardcoded core URLs with `ctx.links`.** If you wrote `` `/episodes/${slug}` ``
   anywhere, that is `ctx.links.episode(slug)` now — same string today, and the host's problem when a route
   changes. `ctx.links.episode(slug, { t })` is the timestamp deep link.

Quick checklist: `platformApi` 0.8.0 in all four places · `tsc --noEmit` clean · every `ctx.blobs` use behind
a `null` check · refs stored, never URLs · a refusal path the UI actually shows · no hardcoded `/episodes/…`
or `/feeds/…` left · tests green against testkit / `makeMockCtx` 0.8.0.

## The core half — what changed under you

Core is at **0.6.14** and compiles against `plugin-api` **0.8.0** (core's app version and the SDK's are
independent schemes). Since the 0.7.x-era skills, these are the changes a plugin author can see:

**File storage exists** (core 0.6.11, `POST /api/plugins/{id}/blob` + four more paths). Opt-in through the
manifest `blobs` block; without it every path is a 404. The operator caps both ceilings and owns the MIME
allow-list, and **an admin can grant per-plugin limits in the admin panel** (core 0.6.13) — a grant
*replaces* the manifest's ask rather than being minimised with it, so `…/blob/quota` is the only honest
source for what you actually got. Refusals are **413** (size/quota) and **415** (declared or actual type),
with distinct problem types. SVG is never storable, and a manifest asking for it is rejected at load.

**`ctx.links.episode/feed`** (core 0.6.11) — the host's own URL shapes as plain string builders.

**An episode link can point at a moment** (core 0.6.9): `/episodes/{slug}?t=754`, which is what
`ctx.links.episode(slug, { t })` produces. An explicit `t` beats the listener's stored progress **without
overwriting it** — the position is only written back once playback advances five seconds past the shared
one. Unparsable values are dropped rather than rejected, so a mangled timestamp still opens the episode.

**The shell has its own share dialog** on episodes, feed tabs and the site panel (core 0.6.12) — prepared
destinations plus a start-at row. It makes **no consent decision** and loads no third-party script. Do not
build a second one into a plugin tile.

**Global upload size rose to 12 MB** (`spring.servlet.multipart.max-file-size`) so it clears the plugin
ceiling, and **plugin blob writes join the upload rate-limit bucket**, matched by path shape
(`/api/plugins/*/blob`), so ordinary doc-store writes stay out of a bucket sized for files.

Still true from the 0.6.x era, worth re-checking on an older manifest:

- **The schema store has an HTTP read surface** (core 0.6.8), gated by `data.readableBy`; `ctx.schema` is
  `null` for a doc-store plugin. Projecting a corpus into doc keys for the UI is obsolete.
- **`ctx.route.navigate`** is wired to the shell's router and namespace-confined host-side; a route change
  rebuilds `ctx` and re-runs your render, while `route.onChange` stays an inert no-op.
- **Extension points run on the instance `register(ctx)` ran on** (core 0.6.7). Delete any `static ctx`
  workaround in a `ShareMetadataProvider` / `SitemapProvider`.
- **A deep link needs a `page` slot**; **`placement: "admin"` renders nowhere**; **read floors are not
  inferred from slots** — an anonymous display slot plus no `data` block means 403 on reads, on all three
  surfaces now.
- **Strict media sources** (core 0.6.1, off by default) narrow `img-src`/`media-src` to feed-derived origins
  plus your *consented* hosts. Note this is one more reason to serve images you own through `ctx.blobs`:
  host-served blobs are same-origin, so they need no CSP host and make no consent decision.

## After the bump

```bash
./gradlew build && npm ci && npm run build && npm run typecheck && ./build.sh
```

Copy `dist/` into `$MOSAICAST_PLUGINS_DIR/<id>`, restart core, and **check the admin log viewer at startup** —
a rejected manifest disables only your plugin, quietly, with its reason there.
