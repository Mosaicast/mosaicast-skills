---
name: writing-a-mosaicast-plugin
description: Use when creating or modifying a Mosaicast plugin (any mosaicast-plugin-* repo or the plugin-sample). Covers the plugin.json manifest (slots and placements, the page slot behind /p/<id>/*, data access floors, backendOwned keys, consent services, doc vs schema storage, the blobs file-storage block (its own optional readableBy/writableBy floors, separate from data's, and ZIP as a storable type), the tags vocabulary block, the external-services block, the identity block (ctx.users, resolving user UUIDs to a name/avatar), the notifications block (ctx.notify/ctx.notifier, putting a message in a user's inbox), nav entrances into the host menu, config fields with a localized label/description and a closed options set, the license/author/homepage/attribution credit fields), the backend PluginBackend/PluginContext contract (including onSchedule's Supplier<Duration> overload for a schedule that actually follows config) and its optional extension points (ShareMetadataProvider and its per-URL OgMeta.locale, SitemapProvider and its hreflang-alternates SitemapUrl.alternates, PageRouteProvider, SearchProvider, UserDataHandler and its exportFiles/UserExport/ExportFile GDPR file export), per-user data in the USER scope, data.keyFloors for raising a doc-store key's floor above the plugin's own, storage.schemaReadableBy for the schema surface's own read floor, the frontend Web Component via the SDK ctx (ctx.docs, ctx.feeds, ctx.tags, ctx.users, ctx.notify, ctx.schema reads, ctx.blobs uploads, ctx.links, ctx.locale.available/content, ctx.translation, ctx.route.navigate, the live ctx.filter, the uncapped and now-unclamped ctx.episodes/displayMany, and ctx.episode's status/phase/announceAt, typed ctx.api errors including the keyFloor problem type) plus DisplaySnapshot's feed/season/episodeNo/phase/announceAt identity fields (season/episodeNo now also podcaster-hand-settable and possibly 0), seasonScope()/resolveSeasonScope(), onEpisodeReleased and onEpisodePhaseChanged for reacting to an episode's release or phase change, defineMosaicastElement's MosaicastHandle (a render surviving a reassigned ctx instead of being torn down) and theme tokens including the --mc-icon-* icon set, testing against the SDK test kit, and installing a plugin by spec (owner/repo@tag#sha256). Trigger whenever writing the manifest, adding a slot, wiring ctx, storing per-user data, uploading or serving files, linking to core pages, querying schema tables from the frontend, reading or writing the shared tag vocabulary, contributing to site search, handling account deletion/export, real 404s for a page plugin's unknown subpaths, declaring external-service (translation) use, declaring what language a plugin page or sitemap translation group is written in, resolving user ids to a display name/avatar, sending an in-app notification to a user, adding a nav entrance, labeling or restricting a config field, making a scheduled task follow a configurable interval, keeping a component's state across a re-rendered ctx, sanitizing HTML a plugin did not write (ctx.sanitize), aggregating over every user's data (data.readsAllUsers / ctx.allUsers), bounding a config value, labelling a consent category, navigating inside a page plugin, declaring backend-owned or schema storage, styling a plugin tile's icons, crediting a plugin's license or data source, bumping platformApi, installing or releasing a plugin, placing an episode in its season or feed for a per-season aggregate, reading the visitor's active season/tag/sort filter, storing a ZIP archive, declaring an upload floor stricter than the plugin's data floor, preparing content for a planned/quiet episode before it is announced, reacting to an episode's release, branching on an episode's release phase, reacting to an episode becoming less visible, raising a specific doc-store key's access floor above the plugin's own, gating schema reads separately from data reads, handling an episode's hand-set season or episode number, exporting a plugin's data for a GDPR request, or building a plugin's backend or frontend.
---

# Writing a Mosaicast plugin

Everything here was checked against the SDK and core working trees, not against the changelogs. Where a
doc and this skill disagree, see "Which docs to trust" below.

## Version pins — get these wrong and the plugin does not load

Contract version is **0.19.0** (`PlatformApi.VERSION`), and core **0.8.2** hosts it — but
`PLATFORM_API_VERSION` in the SDK itself reads **`'0.19.1'`** (the current patch), and the host matches on
exact `major.minor` only, so `"0.19.0"` and `"0.19.1"` both load. If your own manifest contract test
compares `platformApi` against `PLATFORM_API_VERSION` literally, write `"0.19.1"` — same reasoning as the
0.16.1 precedent below. A `0.18.x` manifest is still *rejected at load*, not warned about, and a 0.19 plugin
is rejected by core 0.7.x. Core's own minor moved 0.7.8 → 0.8.0 → 0.8.1 → 0.8.2 **without another
`platformApi` bump** — the GDPR file-export wiring, the feed-deletion phase event and the streamed blob
reads all landed as core patches/minors with no contract change; core's minor tracks its own build
milestones independently of the plugin contract (see `backend.md`'s `UserDataHandler` section).

```json5
"platformApi": "0.19.1"                                 // plugin.json — "0.19.0" also loads; keep one string
```
```kotlin
compileOnly("dev.mosaicast:plugin-api:0.19.1")          // backend/build.gradle.kts
compileOnly("org.pf4j:pf4j:3.16.0")
annotationProcessor("org.pf4j:pf4j:3.16.0")             // mandatory: generates the @Extension index
testImplementation("dev.mosaicast:plugin-testkit:0.19.1")
```
```json5
"@mosaicast/plugin-sdk": "0.19.1"                       // frontend/package.json
```

Both halves of `0.19.1` are published on npm and GitHub Packages (tag `v0.19.1`) — no `mavenLocal()`
workaround needed. Maven is GitHub Packages
(`https://maven.pkg.github.com/Mosaicast/mosaicast-plugin-sdk`), which needs a PAT with `read:packages`
**even for public reads**. Jackson is **3.2.2** (`tools.jackson.*`), not `com.fasterxml`. **Pin the same
string in all four places** — the CI drift guard and the manifest contract test both compare them.

**0.19.0 → 0.19.1 is a patch, nothing to re-declare.** Two things worth knowing anyway: `i18n.bytes` (from
`createPluginI18n`) now formats in **binary units** (`KiB`/`MiB`/`GiB`/`TiB`, matching core's admin) instead
of decimal — a component test pinning the old `'268.4 MB'`-style output needs the binary one; and
`PluginContext.onEpisodePhaseChanged`'s javadoc now documents that calls arrive **concurrently** (see rule
40 below) — that was true since 0.19.0, just undocumented here until now.

Coming from an older pin? `references/migrating.md` walks 0.8.0 → 0.9.0 → 0.9.1 → 0.10.0 → 0.11.0 → 0.12.0 →
0.13.0 → 0.14.0 → 0.15.0 → 0.16.0 → 0.17.0 → 0.18.0 → 0.19.0 in order — read it top to bottom rather than
jumping straight to 0.19.0, since 0.9.0's compile break and 0.10.0's silent `ctx.translation` gate both still
apply on the way up. 0.12.0 was the easy step (manifest bump only); 0.13.0 and 0.14.0 each added one more
optional, `null`-unless-declared capability (`ctx.users`, `ctx.notify`); 0.15.0 fixes two real bugs — a
configurable schedule that used to ignore its own config, and a component destroyed several times a second
during playback; 0.16.0 is the audit release — a real compile break (`queryAcrossUsers` → declared
`ctx.allUsers()`) and a security fix (`ctx.sanitize`) that both shipped plugins needed; 0.17.0 is additive
only (`DisplaySnapshot.feed`/`.season`/`.episodeNo`, `seasonScope()`); 0.18.0 adds planned-episode awareness
(`ctx.episode.phase`, `DisplaySnapshot.phase`/`.announceAt`, `onEpisodeReleased`) and a TS test-fixture
compile break (`ctx.episode` now requires `phase`); 0.19.0 is **also security-shaped, like 0.16.0** —
`data.keyFloors` and `storage.schemaReadableBy` only mean anything on a host new enough to enforce them, so
an older host loading a plugin that declares either would silently serve those keys/rows at the plugin's
(wider) floor. It also drops a trap you may have worked around: `ctx.feeds.displayMany` no longer clamps at
200, so delete any hand-rolled chunking.

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
17. **Prefer `ctx.docs`/`ctx.feeds`/`getOrNull` over hand-built paths and swallowed 404s.** An unset key is a
    **204** since core 0.7.4 (404 now means a wrong address); `ctx.docs.get` resolves `null` for it, and the
    client dedupes in-flight reads and remembers misses **briefly** (~30 s, never across a navigation, core
    0.7.5) — delete your own miss cache, and re-read on your own cadence a key your backend or another session
    writes later. Many scopes at once: `ctx.docs.getMany(type, ids, keys)`.
    `ctx.feeds.display`/`displayMany` replace a scheduled ingest that copies episode snapshots into your own
    doc store — read them live, they are not authoritative and the host overwrites them on every refetch.
18. **`ctx.users` resolves, it does not enumerate, and its answer is absent-not-redacted.** An unknown,
    erased or pseudonymised id is simply *missing* from the result — no `null` element, no tombstone — so
    the array is not index-aligned with what you asked for and may be shorter. Match on `id`, never on
    position. **Store the UUID, resolve at render, never persist a `displayName`** — the host cannot enforce
    that one; a copied name outlives the rename and the erasure both meant to end it.
19. **`ctx.notify`/`ctx.notifier()` write into *another* user's experience — the one plugin surface that
    does.** Two bounds you cannot lift: you may only notify a user your plugin already holds `USER`-scope
    data for (the same partitions `allUsers().query(...)` reads), and the rate limits are the host's, not
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
23. **HTML you did not write goes through `ctx.sanitize`, never `DOMPurify` defaults** (0.16.0). The
    defaults allow `<style>` and `style=`; under the contract's `style-src 'unsafe-inline'` that is a
    page-wide overlay (the wiki shipped it and was defaced). Sanitize **after** Markdown rendering.
    `DisplaySnapshot.description` is untrusted feed HTML — show `descriptionText` unless you need markup.
24. **Reading every user's partition is declared** (`data.readsAllUsers`, 0.16.0) and goes through
    `ctx.allUsers()`, `null` without it. `DocStore.queryAcrossUsers` no longer exists. Treat `null` as a
    manifest bug and throw — an empty aggregate publishes "nobody did anything" as fact.
25. **Colour text, links and focus rings with `--mc-accent-text`**, never `--mc-accent` (0.16.0). The latter
    is the admin's unchecked seed and is for fills only.
26. **Bound numeric config** (`min`/`max`/`step`, 0.16.0) — the host refuses an out-of-range save and treats
    a stored one as unset. Without `"min": 1`, `0` is a legal interval that turns your schedule off.
27. **A consent category you introduce needs `consent.categoryLabels`** (0.16.0), or visitors are asked to
    consent to a bare id wrapped in a generic phrase.
28. **An episode's `feed`/`season`/`episodeNo` are the one authoritative part of `DisplaySnapshot`** (0.17.0,
    needs core 0.7.6+) — resolved from identity on read, not overwritten by a feed refetch like every other
    field. Build a season scope with `seasonScope(feed, n)` / `resolveSeasonScope(snapshot)`, never
    `` `${feed}:${season}` `` by hand, and never parse `ctx.episodeLabels` for this — it's a display string
    that drops the season of an unnumbered episode.
29. **`ctx.filter.current()` is live since core 0.7.6, `{}` on 0.7.5 and older, and always `{}` on a page
    mount regardless of core version.** Treat an absent axis (`season`/`tags`/`sort`) as "unfiltered" and one
    build runs on every core version; a page mount's query string is `ctx.route.query`, never `ctx.filter`.
30. **`ctx.episodes` covers the whole scope since core 0.7.6** — older hosts silently stopped at 200 for a
    `feed`/`site` scope, which under-counted a long-running show's aggregate with no error. **`displayMany`
    itself stopped clamping at 0.19.0 (core 0.8.0)** — past `DISPLAY_BATCH_LIMIT` (200) it now splits into
    several requests and merges them, the same guarantee `ctx.docs.getMany` already had. Delete any
    hand-rolled chunking of `ctx.episodes` before calling `displayMany` — on 0.19.0+ it is dead code, and on
    an older host it was silently dropping episodes past 200, not protecting you from anything.
31. **`blobs` may declare its own `readableBy`/`writableBy`, separate from `data`'s** (core 0.7.6) — absent
    still falls back to the `data` floors, so nothing changes for an existing manifest. `application/zip` is
    now a storable, default-allowed type; declare it by its real name, not a browser's alias
    (`application/x-zip-compressed` etc. are canonicalised, not separately permitted).
32. **A `planned` episode is invisible below podcaster — on both sides, but not symmetrically.** The
    frontend never sees one at all: it is absent from `ctx.episodes`, and `ctx.feeds.display`/`displayMany`
    answer `null`/drop it for that visitor, same as any other access-filtered episode. Your **backend**'s
    `FeedAccess` sees it regardless of who is looking, because preparing content before an announcement is
    the point — so a backend that republishes what it reads (a leaderboard, a computed card) must check
    `DisplaySnapshot.phase()` itself before handing a `planned` episode's content to anyone who is not its
    preparer. Branch on **`phase`** (`EpisodePhase`/`DisplaySnapshot.phase`), never on the stored `status` —
    an `UPCOMING` episode is still `status == PLANNED` but is public.
33. **`ctx.onEpisodeReleased` is a best-effort shortcut, never the only path.** It fires once, after the
    binding commits, on a host thread — not durable, not replayed, and never called for an episode that
    arrives already released. A plugin that was stopped, restarting or mid-upgrade at the moment of release
    simply never hears about it. Always pair it with a periodic `onSchedule` reconciliation that checks
    `phase() == RELEASED` for anything your plugin still treats as pending, and make the listener idempotent
    — the event and the reconciliation may both act on the same release.
34. **`ctx.episode` now requires `phase`, and `status` is actually filled (0.18.0, needs core 0.7.7).**
    Before, core left `status` empty despite the SDK declaring it — a hand-built test fixture
    (`episode: { status: 'PLANNED' }`) silently asserted nothing. `makeMockEpisode(phase, announceAt?)`
    builds a correct one; a literal missing `phase` now fails `tsc --noEmit`, the same trap every `ctx`
    member gain springs. There is still no frontend release *event* — a phase change arrives as a new `ctx`,
    same mechanism as a route or filter change.
35. **`data.keyFloors` raises the floor of named keys, and it is raise-only** (0.19.0). `[{ keys,
    readableBy?, writableBy? }]`, same selector grammar as `backendOwned`. Rejected at load: a floor *below*
    the plugin's own `readableBy`/`writableBy`, `writableBy: "anonymous"`, an entry with neither floor set,
    or an entry naming no keys. Several matching entries combine to the **strictest** per direction. Write
    order is plugin floor → `backendOwned` → key floor, so a `backendOwned` key stays unwritable regardless
    of its key floor. A listing drops a hidden key before paging (so totals count only what the reader may
    see); `getMany` leaves it absent, like a miss; a single `get`/`put`/`remove` is a 403 —
    `PROBLEM_TYPES.keyFloor`, distinct from the plugin-floor `forbidden` and the `backendOwnedKey` 403. Not
    for the `USER` scope, and your own `ctx.store()` reads/writes every key regardless — this is a
    *client*-facing floor only.
36. **`storage.schemaReadableBy` is the schema surface's own read floor, separate from `data.readableBy`**
    (0.19.0) — gates `ctx.schema`'s four read endpoints (`select`/`search`/`count`/one row), any of the four
    roles, defaulting to `data.readableBy` when absent. Matters most for a plugin whose **tile** must be
    anonymous (bingo) but whose **rows** should not be — without it, every schema row (player entries keyed
    by user id, rows for a quiet planned episode) was served to anonymous visitors right alongside the
    public ones. The backend's `SchemaStore` is unaffected; this is a read-API floor only.
37. **`ctx.onEpisodePhaseChanged((slug, phase) -> …)` exists for the direction `onEpisodeReleased` can't
    cover: an episode becoming *less* visible** (0.19.0). A podcaster pushing `announceAt` back into the
    future makes a released-looking episode `PLANNED` again, instantly, for every visitor below podcaster —
    but whatever your backend *published on a schedule* (a site index, a count, a teaser) keeps naming it
    until the next tick. This hook fires on that write: announce, an `announceAt` edit either direction, a
    release, a withdrawal, a withdrawn episode returning, and cancellation (`phase` arrives `null` — the
    episode no longer exists). **The clock passing `announceAt` fires nothing** — becoming visible late is
    harmless, so that direction is still yours to reconcile on a schedule. Anything computed **per request**
    (a sitemap, OG tags, `PageRouteProvider`, search) was already correct and needs none of this; this hook
    is only for what you cached. Delivery is the same as `onEpisodeReleased`: best effort, not durable, and
    on a release the release listeners run first. **Deleting a whole feed fires it too, once per episode,
    `phase == null`** (core 0.8.1) — the same "gone" signal cancellation already sends.
38. **`DisplaySnapshot.season`/`.episodeNo` are no longer purely feed-derived, and both may be `0`** (core
    0.7.7, documented as of SDK 0.19.0). A podcaster can set either by hand in the admin — the hand-set value
    wins, survives feed polls, and is how a prologue episode gets to be "episode 0" at all, since
    `itunes:episode` can't carry a zero. Never write `if (snapshot.episodeNo())` / `if (episodeNo)` — zero is
    a real, meaningful value here, not "unset"; check for `null`/`undefined` explicitly instead.
39. **`UserDataHandler.exportFiles(userId)` is now real, wired by core 0.8.0** — your plugin's part of a
    person's GDPR export, as files in your own format under `plugins/<id>/` in one ZIP, not a JSON map you
    hope core can render. Returning `Optional.empty()` (the default) falls back to `exportUser` the way it
    always has, so a `Map`-only plugin keeps exporting unchanged with zero code to write. Bounded by
    `UserExport.MAX_BYTES` (32 MiB total) within `UserExport.TIMEOUT` (60 s) — go over either and the host
    records your plugin's part as **`failed`**, never silently truncated. Export is **read-only** and
    **this person only**: a leaderboard row that merely mentions them is not theirs to hand over. No
    `UserDataHandler` at all shows as **`not-supported`** in the person's archive, not `empty` — core can
    tell "said nothing" apart from "nothing to ask."
40. **`onEpisodePhaseChanged` calls arrive concurrently, not one at a time — guard shared state with
    `ReentrantLock`, never `synchronized`** (SDK 0.19.1 javadoc). Each event is its own task on its own
    thread; deleting a feed with many episodes is the case that makes this obvious, since every one of its
    episodes fires at once. If your listener coalesces (one pass handles several calls rather than
    recomputing per slug), the lock guarding that must be `java.util.concurrent.locks.ReentrantLock` — the
    host's threads are virtual, and on Java 21 a virtual thread blocked on a `synchronized` monitor **pins
    its carrier thread**, which starves every other virtual thread sharing it. Not a style nit; a real
    incident waiting for enough concurrent episodes.

## Before you build: ask for a browser and an instance

Do this **once, at the start** of plugin work. Both are recommended, **neither is a blocker** — if the user
has neither, build the plugin in full anyway and say at the end what could not be verified.

1. **Check whether you have browser tooling** (Playwright MCP or equivalent). If not, tell the user once
   that you can still build and unit-test but cannot see the tile render, catch a CSP refusal or check a
   phone width — and that enabling a browser tool would fix that. Then carry on; do not ask again.
2. **Ask for a Mosaicast instance** you may install into and **restart** (a restart is how core picks up a
   rebuilt plugin), and **ask whether it holds production or test data**. If the user has a `mosaicast-core`
   checkout, `dev/instance.sh --name <plugin> up --plugin-dir "$PWD/dist"` stands up a disposable, **named**
   seeded stack — best option, nothing real at risk. **Always pass `--name`**: named instances get their own
   Postgres, ports and plugins dir under `/tmp/mosaicast-dev/<name>/`, so several sessions run side by side
   without colliding, and a session only ever `up`s/`down`s its own name. Read ports from
   `source <(dev/instance.sh --name <plugin> env)` rather than assuming a number — only the bare, un-named
   `default` instance keeps fixed `:5433`/`:8081`/`:8099`. **`--plugin-dir` is what loads your own build** —
   `--plugins` instead copies the checkout's own `./plugins` folder (the sample plugin's demo card, noise for
   real plugin work) — so without one of the two your plugin never loads and the tile you are testing is
   silently absent.

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
| Moving an existing plugin from 0.8.x up to 0.19.0 | `references/migrating.md` |

Live reference implementation: **`mosaicast-plugin-sample` 2.20.1 on SDK 0.19.1** (check it merged and
tagged before trusting it — it caught up from 2.19.0/0.18.0 since the last pass). It exercises `keyFloors`
(`plugin.json`: `drafts`/`announced` raised to `podcaster`) and `onEpisodePhaseChanged`, but **not**
`storage.schemaReadableBy` or `UserDataHandler.exportFiles` — don't expect either in it. What it does
exercise, still accurately: `feed`/`season`/`episodeNo`, the live `ctx.filter`, `identity`, `notifications`,
`tags`, `external`, `blobs`, `data.readsAllUsers`, bounded config, a labelled consent category and a
four-entry `nav[]` (with `visibleTo`, not the SDK type's `role`), Markdown through `ctx.sanitize`, and
`ctx.blobs`/`ctx.links`/`ctx.route.navigate`. Its `README.md` carries the worked `curl` forgery that
motivates `backendOwned`, and its `docs/BRIEF.md` pattern is the definition of done for every plugin repo.
For foreign HTML that must keep the plugin's own `class`/`data-*` markup, the pattern to copy is
`mosaicast-plugin-wiki` 0.5.0's `markdown.ts`.

## Which docs to trust

- **SDK `README.md` / `CHANGELOG.md` / `MIGRATION.md` and the Javadoc/TSDoc** — accurate, take signatures from here.
- **`docs/ARCHITECTURE.md`** — current through §8.8 (`ctx.users`, the `identity` block) and mostly current
  on §17 (notifications) — **but see the flat-out wrong code block below**, which is worth reading before
  the rest of this bullet. Correct on the `data` block and `backendOwned` (§7.2), the `tags` and `external`
  blocks and their refusal shapes (§7.2), `nav[]` and the default entry a page plugin gets without one
  (§7.2 — but see the `role`/`visibleTo` drift below), the `USER` scope, the cross-user read (now correctly
  named `ctx.allUsers()`, the pre-0.16.0 `DocStore.queryAcrossUsers` kept only as a parenthetical "was") and
  every extension point including `SearchProvider`, `PageRouteProvider` and `UserDataHandler` (§7.4), the
  schema provider **and its read HTTP surface** (§7.6), the `blobs` manifest block and file storage (§7.2,
  §11, §11.1), the `?lang=` URL scheme and `hreflang` alternates (§6.4, §6.6, §12.7), site-wide search
  (§6.7), account deletion reaching plugin data (§12.8), languages as a runtime registry and host-mediated
  translation (§12.7, §16), purge (§7.8), the live `ctx.filter` and uncapped `ctx.episodes` (§6.1, core
  0.7.6), `DisplaySnapshot.feed`/`.season`/`.episodeNo` as identity rather than feed presentation (§4.2,
  §4.4, `platformApi` 0.17.0), planned episodes — quiet-by-default creation, the derived `EpisodePhase`,
  `onEpisodeReleased` and the backend-sees-it/frontend-doesn't visibility split (§4.3, §5.3, §7.4, §7.5,
  `platformApi` 0.18.0), and **now a new §12.8.1** covering the GDPR file export end to end: the
  `exportFiles`-then-`exportUser` order, the ZIP layout, the per-account rate limit, and the
  `complete|empty|failed|outstanding|not-supported` outcome vocabulary (`platformApi` 0.19.0, core 0.8.0).
  §7.5's `ctx` block lists `docs`, `feeds`, `tags`, `users`, `notify`, `schema`, `blobs`, `links`,
  `locale.available/content`, `translation`, `filter`, `episode` (`status`/`phase`/`announceAt`) and
  `route.navigate`.
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
- **This skill's own prior claim that `ctx.episode` is never populated is now wrong — core 0.7.7 wired it.**
  Earlier skill revisions (correctly, at the time) said `status` was declared but always empty and told you
  never to branch on it. As of `platformApi` 0.18.0 / core 0.7.7, `ctx.episode = { status, phase,
  announceAt? }` is real on the `episode` scope, `phase` is what to branch on, and a phase change arrives as
  a freshly-assigned `ctx` (no separate event). If you find a copy of this skill still saying the field is
  dead, re-verify against the current core rather than trusting either claim blindly.
- **The SDK's `PluginBlobsDeclaration` TS type has existed since 0.9.0 — a prior claim in `frontend.md` that
  there is "no TS type for the manifest's `blobs` block" was simply wrong, not just stale.** It documents
  `maxFileBytes`/`quotaBytes`/`mimeTypes` and, since 0.18.0, the optional `readableBy`/`writableBy` pair —
  core still owns validation and is still authoritative on rejection, but the type exists for editor
  autocomplete and was never absent.
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
