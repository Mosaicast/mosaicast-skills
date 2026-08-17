# Migrating an existing plugin to 0.7.x

The SDK's own `MIGRATION.md` (in the `mosaicast-plugin-sdk` checkout) is the authoritative checklist for the
**SDK** half — read it, it is short. This file adds the **core-side** changes it does not cover, because
core moved too.

Coming from `0.5.x`? Do the 0.6.0 migration first (`data.backendOwned`, the keys only your backend may
write), then this.

## The SDK half, in brief

**No plugin code written against 0.6.0 changes behaviour.** Both additions are new surface. The one thing
that stops compiling is a *test* — and only on 0.7.0.

1. `plugin.json` → `"platformApi": "0.7.1"`; `build.gradle.kts` → `plugin-api` / `plugin-testkit` `0.7.1`;
   `package.json` → `@mosaicast/plugin-sdk` `0.7.1`. Both halves are published (npm + GitHub Packages), so no
   `mavenLocal()` detour. You have no choice about timing: the moment the host runs 0.7.x, every 0.6.x plugin
   is rejected at load.
2. **Take 0.7.1, not 0.7.0.** It is test-kit-only — same contract, `platformApi` still matches on
   `major.minor` — and it makes step 3 a no-op.
3. **A hand-built `route` override needs `navigate` on 0.7.0.** `PluginRoute` gained it as a required member,
   so `route: { path: 'kraken', onChange: () => () => {} }` no longer satisfies the type. On **0.7.1** the
   override *merges* over the default instead: drop the stubs and write `route: { path: 'kraken' }`, keeping
   the `navigations` recorder. Your test runner will not catch this either way — **only `tsc --noEmit`
   will**, which is the argument for having a `typecheck` script at all.
4. **If your frontend reaches the schema store through doc-key projections, stop.** `ctx.schema` reads the
   provisioned tables directly (`select`/`search`/`find`/`count`), including the full-text index. It is
   `null` for a doc-store plugin, so every use goes behind a `null` check. Reads only — the backend is still
   the only writer of relational truth.
5. **If you have a `page` slot, replace internal `<a href>` full loads and any `history.pushState` +
   synthetic `popstate` with `ctx.route.navigate(subpath)`.** Keep the `href` on the anchor; `navigate` takes
   over the plain-click path only.

Quick checklist: `platformApi` 0.7.1 in all four places · `tsc --noEmit` clean · every `ctx.schema` use
behind a `null` check · no frontend code expecting a schema **write** · internal page links call `navigate`
and keep their `href` · no `pushState`/`popstate` left · tests green against testkit / `makeMockCtx` 0.7.1.

## The core half — what changed under you

Core is at **0.6.8** and compiles against `plugin-api` **0.7.0** (so it accepts `0.7`, `0.7.0`, `0.7.1`).
Note core's own app version and the SDK's are independent schemes.

**The schema store has an HTTP read surface** (`GET /api/plugins/<id>/schema/*`), gated by the same
`data.readableBy` floor as the doc store, and served only to a plugin that declares `storage.schema` — a
doc-store plugin 404s there. That is what `ctx.schema` talks to; see `frontend.md` for the query grammar and
the status codes.

**`ctx.route.navigate` is wired to the shell's router.** The host confines the target to `/p/<pluginId>/`:
a leading `/` is stripped and `.`/`..` segments are dropped, so another plugin's route is unnameable rather
than merely refused. In a mount with no router above it, `navigate` degrades to a no-op.

**A route change no longer needs a subscription.** The shell rebuilds `ctx` when the subpath changes and
re-assigns it, which re-runs your render after your previous cleanup. `route.onChange` is still an inert
no-op — read `ctx.route.path` at render time.

Still true from the 0.6.x era, worth re-checking on an older manifest:

- **Extension points run on the instance `register(ctx)` ran on** (core 0.6.7). Delete any `static ctx`
  workaround in a `ShareMetadataProvider` / `SitemapProvider`.
- **A deep link needs a `page` slot** (`{ "scope": "site", "placement": "page" }`), or `/p/<id>/*` is a real
  404 — and without one `navigate` has nowhere to go.
- **`placement: "admin"` renders nowhere.** Move that UI to a `sidebar` slot with `visibleTo: "podcaster"`.
- **Read floors are not inferred from slots.** An anonymous display slot plus no `data` block means 403 on
  reads — declare `"readableBy": "anonymous"` if the data really is public. This now governs the schema
  surface too.
- **Strict media sources** (core 0.6.1, off by default): if an operator enables
  `mosaicast.security.strict-media-sources`, `img-src`/`media-src` narrow to feed-derived origins plus your
  *consented* hosts. Declare the origins you load media from in `consent.services[].hosts` now.

## After the bump

```bash
./gradlew build && npm ci && npm run build && npm run typecheck && ./build.sh
```

Copy `dist/` into `$MOSAICAST_PLUGINS_DIR/<id>`, restart core, and **check the admin log viewer at startup** —
a rejected manifest disables only your plugin, quietly, with its reason there.
