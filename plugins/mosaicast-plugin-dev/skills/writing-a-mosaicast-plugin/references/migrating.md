# Migrating an existing plugin up to 0.19.0

The SDK's own `MIGRATION.md` (in the `mosaicast-plugin-sdk` checkout, or `v0.19.0/MIGRATION.md` on GitHub —
0.15.0 was once tagged only `0.15.0` without the `v`; both forms resolve now) is the authoritative checklist for the
**SDK** half of each step — read it, it is short and version-scoped. This file adds two things that doc
does not: the **core-side** changes each release shipped alongside it, and one file walking the **whole
chain** for a plugin that has not moved since 0.8.0.

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
| 0.11.x → 0.12.0 | Yes (`platformApi`) | No — binary break only, source-compatible via overloads | `OgMeta.locale`, `SitemapUrl.alternates` — hreflang for plugin pages |
| 0.12.x → 0.13.0 | Yes | Yes — a hand-built `ctx.user` literal in TS tests | `ctx.users`/`identity` — resolving UUIDs to a name + avatar |
| 0.13.x → 0.14.0 | Yes (`platformApi`) | No — new surface, nothing removed | `ctx.notify`/`ctx.notifier()`/`notifications` — an in-app inbox |
| 0.14.x → 0.15.0 | Yes (`platformApi`) | Java: only if you implement `PluginContext` yourself | `onSchedule(Supplier<Duration>, …)` — a schedule that follows config; `MosaicastHandle` — a component that survives a new `ctx` |
| 0.15.x → 0.16.0 | Yes (`platformApi`; `data.readsAllUsers` if you aggregate over users) | **Yes — `DocStore.queryAcrossUsers` is gone**; TS: a hand-written `DocClient`/`DisplaySnapshot` literal | `ctx.sanitize`, `descriptionText`, `--mc-accent-text`, config bounds, `consent.categoryLabels`, `ctx.docs.getMany` |
| 0.16.x → 0.17.0 | Yes (`platformApi`) | No — new, optional fields and a live `ctx.filter`; nothing removed | `DisplaySnapshot.feed`/`.season`/`.episodeNo`, `seasonScope()`, a live `ctx.filter`, uncapped `ctx.episodes`, ZIP uploads, per-`blobs` floors |
| 0.17.x → 0.18.0 | Yes (`platformApi`) | **Yes — a hand-built `ctx.episode` in TS tests** (`phase` now required) | `ctx.episode.phase`/`.announceAt`, `DisplaySnapshot.phase`/`.announceAt`, `onEpisodeReleased`, planned-episode visibility |
| 0.18.x → 0.19.0 | Yes (`platformApi`; `keyFloors`/`schemaReadableBy` only if you use them) | No — new, optional manifest fields and surfaces; nothing removed | `data.keyFloors`, `storage.schemaReadableBy`, `onEpisodePhaseChanged`, `UserDataHandler.exportFiles` (GDPR), uncapped `displayMany` |

Do them **in order**; do not skip to 0.19.0 and back-port the manifest fields, because 0.9.0's compile
break and 0.10.0's `ctx.translation` addition both have to land first for the later steps to make sense.

---

## 0.18.x → 0.19.0: key floors, a schema read floor, the other phase hook, and file exports

Needs **core 0.8.0** for the GDPR-export half; **core 0.7.8** is enough for everything else (`keyFloors`,
`schemaReadableBy`, `onEpisodePhaseChanged`, the `displayMany` split). All of it shipped in the SDK at
0.19.0 regardless — the two-core-release stagger is a core rollout detail, not something you choose per
feature. `mosaicast-plugin-sample` has **not yet** moved to 0.19.0 at the time of writing; check its tag
before treating it as a worked example for this step.

This release is **also security-shaped, like 0.16.0's audit**, for one specific reason: a host older than
0.19.0 does not know `keyFloors` or `schemaReadableBy` exist, so it would load a plugin declaring either and
quietly serve those keys/rows at the plugin's *wider* floor instead of the narrower one the manifest asks
for. That is why it is a `platformApi` minor and not a patch.

**Required.**

```diff
- "platformApi": "0.18.0",
+ "platformApi": "0.19.0",
- compileOnly("dev.mosaicast:plugin-api:0.18.0")      testImplementation("…plugin-testkit:0.18.0")
+ compileOnly("dev.mosaicast:plugin-api:0.19.0")      testImplementation("…plugin-testkit:0.19.0")
- "@mosaicast/plugin-sdk": "0.18.0"
+ "@mosaicast/plugin-sdk": "0.19.0"
```

**That's the whole required migration if you use none of the new manifest fields.** Nothing was removed;
everything below is opt-in.

**1. Delete any hand-rolled chunking in front of `displayMany`.** It no longer clamps at 200 — past that it
splits into several requests and merges them, the same guarantee `ctx.docs.getMany` already had:

```diff
- const slugs = ctx.episodes;
- const chunks = []; for (let i = 0; i < slugs.length; i += 200) chunks.push(slugs.slice(i, i + 200));
- const snaps = Object.assign({}, ...(await Promise.all(chunks.map(c => ctx.feeds.displayMany(c)))));
+ const snaps = await ctx.feeds.displayMany(ctx.episodes);
```

**2. If a schema plugin's tile must be anonymous but its rows should not be, add `storage.schemaReadableBy`.**
Before, `storage.schemaReadableBy` didn't exist and `data.readableBy` silently doubled as the schema read
floor — a plugin like bingo, anonymous by design, served every row (player entries keyed by user id,
quiet-episode rows) to anyone:

```diff
  "data": { "readableBy": "anonymous", "writableBy": "fan" },
- "storage": { "schema": { … } }
+ "storage": { "schema": { … }, "schemaReadableBy": "podcaster" }
```

**3. If you have private bookkeeping beside public numbers in the doc store, give those keys their own
floor instead of splitting the plugin:**

```diff
  "data": { "readableBy": "anonymous", "writableBy": "podcaster" },
+ "keyFloors": [{ "keys": ["import:*", "staged:*"], "readableBy": "podcaster" }]
```

Raise-only, strictest-match-wins, checked after `backendOwned` on writes — see `manifest.md` for the full
rejection list. `ctx.store()` on your backend is unaffected either way.

**4. If your backend publishes anything about an episode on a schedule, register
`onEpisodePhaseChanged` alongside `onEpisodeReleased`.** The release hook only covers an episode becoming
*more* visible; this one covers the leak — an episode becoming *less* visible while something you cached
still names it:

```diff
  ctx.onEpisodeReleased(this::open);
+ ctx.onEpisodePhaseChanged((slug, phase) -> {
+     if (phase == null || phase == EpisodePhase.PLANNED || phase == EpisodePhase.WITHDRAWN) {
+         republishIndex();        // drop it from what anonymous readers see, right now
+     }
+ });
```

Anything you compute **per request** (a sitemap, `ShareMetadataProvider`, `PageRouteProvider`,
`SearchProvider`) was already correct and needs none of this.

**5. If you implement `UserDataHandler`, add `exportFiles` for a real GDPR export — `exportUser` is now
the fallback, not the feature.** The host asks `exportFiles` first; an empty `Optional` (the default) falls
back to `exportUser` exactly as before, so a `Map`-only plugin keeps exporting with zero code changes.
Implement `exportFiles` when your data has a format worth keeping as a format:

```diff
  @Override
  public void eraseUser(String userId) { … }
+
+ @Override
+ public Optional<UserExport> exportFiles(String userId) {
+     List<Card> cards = cardsOf(userId);
+     if (cards.isEmpty()) return Optional.empty();
+     return Optional.of(UserExport.of(
+             ExportFile.text("cards.json", "application/json", toBingoV1(cards))));
+ }
```

Bounded by `UserExport.MAX_BYTES` (32 MiB) within `UserExport.TIMEOUT` (60 s) — exceed either and your part
is recorded `failed`, never silently truncated. No `UserDataHandler` at all is fine and shows as
`not-supported` rather than `empty` in the person's archive — core can tell "never implemented this" apart
from "implemented it, had nothing."

**TS compile fallout: none.** Every new field is optional or has a `default` method; `PluginBlobsDeclaration`
already existed and just gained two optional fields. **Java compile fallout: none either** —
`onEpisodePhaseChanged` is a `default void` the same way `onEpisodeReleased` is, so a hand-rolled
`PluginContext` test double keeps compiling.

**Test kit:** `InMemoryDocStore.withKeyFloor(pattern, readableBy, writableBy)`, `asUser(UUID, Role)` and
`asAnonymous()`; `FakePluginContext.onEpisodePhaseChanged`/`fireEpisodePhaseChanged(slug, phase)`/
`episodePhaseListenerCount()`; `UserDataHandlerHarness.exportFiles(userId)`. TS: `makeMockDocs(initial, {
data, viewer })` for floor enforcement, `MockFeedsClient.batches` for asserting the `displayMany` split.

---

## 0.17.x → 0.18.0: planned episodes, release phase, and hearing about a release

Needs **core 0.7.7** — a 0.18 plugin calling `snapshot.phase()`/`ctx.episode.phase` against an older host's
`plugin-api` jar (loaded parent-first) fails with `NoSuchMethodError`, before the exact-`major.minor` check
even runs. Core 0.7.7 ships the planned-episode feature this release's SDK half depends on; `0.7.6` alone is
not enough even though it is newer than `0.7.4`. `mosaicast-plugin-sample` 2.19.0 has already moved to
0.18.0 and exercises `onEpisodeReleased` plus a phase check in its preview logic — a real worked example,
check its tag is still current before copying from it.

**Required.**

```diff
- "platformApi": "0.17.0",
+ "platformApi": "0.18.0",
- compileOnly("dev.mosaicast:plugin-api:0.17.0")      testImplementation("…plugin-testkit:0.17.0")
+ compileOnly("dev.mosaicast:plugin-api:0.18.0")      testImplementation("…plugin-testkit:0.18.0")
- "@mosaicast/plugin-sdk": "0.17.0"
+ "@mosaicast/plugin-sdk": "0.18.0"
```

**The one compile break: a hand-built `ctx.episode` in a TS test.** `phase` is now a required field —
`episode: { status: 'PLANNED' }` stops type-checking:

```diff
+ import { makeMockEpisode } from '@mosaicast/plugin-sdk/testing';
  const ctx = makeMockCtx({
    scope: { type: 'episode', id: 'kraken' },
-   episode: { status: 'PLANNED' },
+   episode: makeMockEpisode('planned'),
  });
```

Java: `FakePluginContext`'s `PluginContext` keeps compiling unchanged — `onEpisodeReleased` is a `default`
method, so a double of your own that never overrides it is unaffected. A `DisplaySnapshot` fixture on the
13-argument constructor (without `phase`/`announceAt`) also keeps compiling.

**If your plugin prepares content for planned episodes — a bingo before the episode airs, a wiki page ahead
of a release — three things to actually change:**

**1. Stop branching on `status`; branch on `phase`.** They diverge the moment an episode is announced: an
`UPCOMING` episode is still internally `status == PLANNED`, but it is fully public.

```diff
- if (snapshot.status() == EpisodeStatus.PLANNED) { /* treat as hidden */ }
+ if (snapshot.phase() == EpisodePhase.PLANNED) { /* actually hidden — podcasters/admins only */ }
```

**2. Your backend's `FeedAccess` now hands you `planned` episodes — that is new, and it makes you the access
boundary for anything you republish.** Before 0.18.0 a planned episode did not exist as far as plugins were
concerned; now `episodesIn`/`display` return it unfiltered, because preparing content before an announcement
is the point. If your backend aggregates and republishes (a leaderboard, a cached card), check `phase()`
before handing a `PLANNED` episode's content to anyone who isn't its preparer — the host no longer does this
filtering for you on this one surface.

**3. React to a release with `onEpisodeReleased`, but never rely on it alone:**

```diff
+ ctx.onEpisodeReleased(this::open);
  ctx.onSchedule(Duration.ofMinutes(15), () ->
      ctx.feeds().episodesIn(Scope.site()).stream()
+         .filter(slug -> ctx.feeds().display(slug).phase() == EpisodePhase.RELEASED)
          .filter(this::stillClosed)
          .forEach(this::open));
```

It is **best effort**: fired once, after the binding commits, not durable, not replayed — a plugin that was
not running at that moment never hears about that release. The reconciliation loop you probably already have
is still required; make the listener idempotent since both may act on the same release.

**Recommended, if you store uploads that should not be as public as your data.** Give the `blobs` block its
own floors: `"blobs": { …, "readableBy": "podcaster" }`. Leave them out and nothing changes — they default to
the `data` floors (this landed core-side in 0.7.6; the SDK's `PluginBlobsDeclaration` TS type caught up with
the `readableBy`/`writableBy` fields in 0.18.0 — the type itself has existed since 0.9.0).

---

## 0.16.x → 0.17.0: placing an episode in its season and feed

Needs **core 0.7.6** — a 0.17 plugin calling `snapshot.season()`/`snapshot.feed()` against an older host's
`plugin-api` jar (loaded parent-first) fails with `NoSuchMethodError`, before the exact-`major.minor` check
even runs. `mosaicast-plugin-sample` has **not yet** moved to 0.17.0 at the time of writing — check its tag
before treating it as a worked example for this step.

**Required.**

```diff
- "platformApi": "0.16.0",
+ "platformApi": "0.17.0",
- compileOnly("dev.mosaicast:plugin-api:0.16.0")      testImplementation("…plugin-testkit:0.16.0")
+ compileOnly("dev.mosaicast:plugin-api:0.17.0")      testImplementation("…plugin-testkit:0.17.0")
- "@mosaicast/plugin-sdk": "0.16.0"
+ "@mosaicast/plugin-sdk": "0.17.0"
```

**That's the whole required migration — every addition is optional to use**, and nothing was removed. Three
things worth adopting anyway:

**1. If you aggregate per season, stop parsing `ctx.episodeLabels`.** It is a display string
(`"S01E06 · Title"`) that also drops the season entirely for an unnumbered episode — two bugs for the price
of one regex. `DisplaySnapshot.feed`/`.season`/`.episodeNo` are the authoritative identity fields instead,
resolved by the host on read and never overwritten by a feed refetch like the rest of the snapshot is:

```diff
- const season = /S(\d+)/.exec(ctx.episodeLabels?.[slug] ?? '')?.[1];
+ const snap = await ctx.feeds.display(slug);
+ const season = snap && resolveSeasonScope(snap);   // Scope | undefined — never hand-build "<feed>:<n>"
```
```java
- Scope season = Scope.season(feedSlugFromSomewhereElse + ":" + guessedSeasonNumber);
+ Scope season = snapshot.seasonScope();              // null-safe; needs feed() and season() both present
```

**2. If you read `ctx.filter`, handle `{}` as "unfiltered" on every core version, not just as a default.**
It is **live since core 0.7.6** — `current()` reflects the shell's `season`/`tag`/`order` URL filters and
`onChange` fires on a real change — but still `{}` on core 0.7.5 and older, and still `{}` on a page mount
on every core version (there the query string is `ctx.route.query`, not the shell's). One build runs
everywhere only if an absent axis is always treated as unfiltered, never as "not loaded yet."

**3. If you aggregate over a whole `feed`/`site` scope, re-check your numbers.** `ctx.episodes` used to stop
at 200 for those two scope types; a show past that length under-counted silently, no error, no truncation
flag. Core 0.7.6 pages the resolution to the end — nothing to change in your code, but a plugin that worked
around the 200 cap with its own pagination can delete that workaround.

**TS compile fallout: none.** `feed`/`season`/`episodeNo` are new optional fields — a `DisplaySnapshot`
literal that omits them still compiles. Java fixtures on the 10-argument `DisplaySnapshot` constructor
(without the trio) keep compiling too; only the 9-argument one (without `descriptionText`, deprecated since
0.16.0) is scheduled for removal.

**Recommended, if you store files.**
- `application/zip` is now a storable, default-allowed `blobs.mimeTypes` entry — declare it by name, not a
  browser alias (`application/x-zip-compressed` etc. are canonicalised to it, not separately permitted).
- `blobs` can declare its own `readableBy`/`writableBy`, separate from `data`'s — use it to keep raw uploads
  more private than the computed numbers your plugin publishes from them.

---

## 0.15.x → 0.16.0: what three test passes found in the contract

Needs **core 0.7.4** (`mosaicast-core` #229) — a 0.16 plugin is rejected by 0.7.2/0.7.3, a 0.15 one by 0.7.4.
Worked examples: `mosaicast-plugin-sample` 2.17.0 and `mosaicast-plugin-wiki` 0.5.0.

**Required.**

```diff
- "platformApi": "0.15.0",
+ "platformApi": "0.16.0",
- compileOnly("dev.mosaicast:plugin-api:0.15.0")      testImplementation("…plugin-testkit:0.15.0")
+ compileOnly("dev.mosaicast:plugin-api:0.16.0")      testImplementation("…plugin-testkit:0.16.0")
- "@mosaicast/plugin-sdk": "0.15.0"
+ "@mosaicast/plugin-sdk": "0.16.0"
```

**Required if your backend aggregates over users** — the compiler finds every site:

```diff
  "data": { "writableBy": "fan", "readableBy": "anonymous",
+           "readsAllUsers": true }
- for (OwnedDocEntry e : ctx.store().queryAcrossUsers("fav:")) { … }
+ for (OwnedDocEntry e : ctx.allUsers().query("fav:")) { … }     // null without the declaration
```
Tests: append `.withReadsAllUsers()` to each `FakePluginContext`, or `allUsers()` is `null` (the sample's
suite went from 70 green to 67 red on exactly this). Assertions on `store.queryAcrossUsers(p)` become
`store.acrossUsers().query(p)`.

**Required if you render HTML you did not write — this is a security fix.** Grep for `DOMPurify`,
`innerHTML`, `dangerouslySetInnerHTML` and `.description`:

```diff
- el.innerHTML = DOMPurify.sanitize(marked.parse(md));      // defaults allow <style> and style=
+ el.innerHTML = ctx.sanitize(marked.parse(md));             // the host's policy, after rendering
- <span>{snap.description}</span>                             // prints the feed's tags as text
+ <span>{snap.descriptionText}</span>
```
If your own generated markup needs `class`/`data-*`, see "`ctx.sanitize`" in `frontend.md` for the
placeholder pattern. Drop the direct `dompurify` dependency afterwards.

**TS compile fallout**: a hand-written `DocClient` wrapper needs `getMany`; a `DisplaySnapshot` literal needs
`descriptionText`; Java fixtures on the 9-arg `DisplaySnapshot` constructor get a removal warning.

**Recommended.**
- `color: var(--mc-accent)` / `outline: … var(--mc-accent)` → `--mc-accent-text`. Fills stay.
- `"min"`/`"max"`/`"step"` on numeric config (at least `"min": 1` on an interval); your clamp becomes a
  fallback.
- A label for any consent category you introduced (`consent.categoryLabels`).
- `ctx.docs.getMany` where you read keys for many scopes; delete any miss cache around `ctx.docs`.
- Check `frontend.entry` against `FRONTEND_ENTRY_PATTERN`.

---

## 0.14.x → 0.15.0: a schedule that follows config, and a component that survives a new `ctx`

```diff
  // plugin.json
- "platformApi": "0.14.0",
+ "platformApi": "0.15.0",
```

```diff
- implementation("dev.mosaicast:plugin-api:0.14.0")
+ implementation("dev.mosaicast:plugin-api:0.15.0")
- "@mosaicast/plugin-sdk": "^0.14.0"
+ "@mosaicast/plugin-sdk": "^0.15.0"
```

**That is the whole *required* migration.** Nothing was removed or reshaped — `onSchedule` gained an
overload and a render may return more than it used to, and both old forms mean exactly what they always
did. Two of the three things this release adds fix bugs your plugin may already have, though, and are
worth doing even though nothing forces them.

### Fix 1: your configurable interval is lying, if you have one

If your tick rate comes from `ctx.config()`, this is a real bug, not a style preference. The period was
captured once, during `register()`, and held for the process's life — an operator saves a new value, the
admin form says it worked, and the plugin runs at the old cadence until the host restarts.

```diff
- ctx.onSchedule(
-         Duration.ofSeconds(ctx.config().get("ingestIntervalSeconds", Integer.class, 60)),
-         this::ingest);
+ ctx.onSchedule(
+         () -> Duration.ofSeconds(ctx.config().get("ingestIntervalSeconds", Integer.class, 60)),
+         this::ingest);
```

One character of real change. The host re-reads the supplier before every tick and reschedules when it
differs — a saved config change now takes effect within one old period, not at the next restart. Keep the
`Duration` overload wherever the cadence is genuinely constant; it is not deprecated. The supplier runs on
a scheduler thread, so keep it cheap: reading `ctx.config()` or a field is fine, querying or blocking is
not. A `null`, a non-positive `Duration`, or a throw leaves the task on its last valid period — logged,
never dropped.

Test it with `FakePluginContext.scheduledPeriods()`, which re-reads every supplier on demand:

```java
var config = new MapPluginConfig(Map.of("ingestIntervalSeconds", 60));
var ctx = new FakePluginContext(new InMemoryDocStore(), config, new FakeFeedAccess(Map.of()), null);
plugin.register(ctx);

config.with("ingestIntervalSeconds", 10);
assertEquals(List.of(Duration.ofSeconds(10)), ctx.scheduledPeriods());   // fails if you captured a Duration
```

### Fix 2: your component may be getting destroyed several times a second

Every `ctx` assignment used to run your cleanup, clear `root`, and re-render — which reads as a rare event
(a consent choice, a language switch) and is not: a host that rebuilds its own context object on every one
of its renders reassigns `ctx` roughly four times a second during playback, tearing down every plugin
element on that page at the same rate. Component state, in-flight requests, scroll position and open
dialogs were lost each time, and every effect behind them re-ran.

```diff
  defineMosaicastElement({
    tag: 'bingo-card',
    render: ({ ctx, root }) => {
      const app = mountMyFramework(root, ctx);
-     return () => app.unmount();
+     return { update: (next) => app.setCtx(next), destroy: () => app.unmount() };
    },
  });
```

Return a `MosaicastHandle` and you decide what a new `ctx` means: `update` runs in place (theme variables
already refreshed, `root` untouched), `destroy` fires only on a real disconnect. Returning a bare cleanup
callback still behaves exactly as before — a static card needs no change. An **identical** context object
is ignored either way, and an element **moved** in the DOM now renders again on reconnect instead of
staying dead. Delete a module-level cache you built to survive the remount storm — measure first.

**Worth checking before you assume this bug still bites**: `mosaicast-core` 0.7.2 already memoizes the
context object it hands plugins (`PluginMount.tsx`, `useMemo`), and the player-time reader inside it is a
stable `useCallback(..., [])`. The four-times-a-second churn the SDK's own example is written against is
fixed on this host as of this release. `MosaicastHandle` is still worth adopting — a slower host tick or a
future core change can still reassign `ctx` — but don't spend time chasing this specific symptom against a
current core if your component still loses state; look elsewhere first.

### Also new: your config fields can say what they are (optional, typing only)

```diff
  config: {
    ingestIntervalSeconds: {
      type: 'number', default: 60, editableBy: 'podcaster',
+     label: { en: 'Ingest interval', de: 'Abrufintervall' },
+     description: { en: 'Seconds between two ingest runs.', de: 'Sekunden zwischen zwei Läufen.' },
    },
+   matchMode: {
+     type: 'string', default: 'fuzzy',
+     options: [{ value: 'fuzzy', label: 'Fuzzy' }, { value: 'exact', label: 'Exact' }],
+   },
  },
```

Plugins may not build their own config UI, so without `label` the generic admin form shows an operator the
raw key and nothing else. **`options` is not new host behavior** — core has validated a closed set since it
shipped; the SDK's own TS type was simply behind, so a plugin already declaring an older `platformApi` and
using `options` was already checked this way.

---

## 0.13.x → 0.14.0: telling a user something happened

```diff
  // plugin.json
- "platformApi": "0.13.0",
+ "platformApi": "0.14.0",
```

```diff
- implementation("dev.mosaicast:plugin-api:0.13.0")
+ implementation("dev.mosaicast:plugin-api:0.14.0")
- "@mosaicast/plugin-sdk": "^0.13.0"
+ "@mosaicast/plugin-sdk": "^0.14.0"
```

**That is the whole migration if you send nothing.** Nothing was removed or reshaped; `PluginContext`
gained a method, but plugins *consume* that interface rather than implement it, and `FakePluginContext`
implements the new one for you. No compile break in either language.

## What the release adds, and why you might want it

A plugin that finishes something a user took part in — a bingo resolving — could until now only hope they
came back and looked:

```diff
  // plugin.json
+ "notifications": { "sends": true, "perUserPerDay": 5 },
```

```java
// The backend, where nearly all real use lives — the thing worth announcing usually finishes on a timer.
ctx.onSchedule(Duration.ofMinutes(15), () -> {
    var participants = ctx.store().queryAcrossUsers("mark:").stream().map(OwnedDocEntry::userId).toList();
    try {
        List<UUID> told = ctx.notifier().send(participants,
                new NotifyMessage(Map.of("en", "Bingo resolved for S02E04",
                                          "de", "Bingo für S02E04 aufgelöst")).withLink("board/42"));
        if (told.size() < participants.size()) ctx.logger().info("notified {}/{}", told.size(), participants.size());
    } catch (NotificationException e) {
        if (e.retryable()) return;   // over the cap — hold it, the next tick will do
        throw new IllegalStateException("bad notification", e);
    }
});
```

```ts
const notify = ctx.notify;
if (!notify) return;                   // no `notifications` block in this plugin's manifest
const told = await notify.send(participants, {
  text: notifyText(catalogs, 'bingo.resolved', { episode: 'S02E04' }),   // your createPluginI18n catalogs
  link: `board/${id}`,
});
```

### The four rules that will catch you

1. **You may only notify users you already hold `USER`-scope data for.** Host-enforced against the same
   partitions `queryAcrossUsers` reads. There is no way to ask for more.
2. **`send` tells you who actually got it — read the return value.** An ineligible or erased recipient is
   left out rather than failing the call, so a partial send is normal and the resolved list is the only way
   to see one. A plugin ignoring this and working from a stale list notifies nobody while looking healthy.
3. **Send every language at once.** `NotifyMessage.text` must contain `en` — §12.7 makes it the one
   language a site cannot switch off. Build the map with `notifyText(catalogs, key, params)` rather than by
   hand, which is where a plugin quietly ships one locale short. The set is fixed at **send** time; a
   language added next month shows English on messages already written.
4. **The cap is a real branch.** Java throws checked `NotificationException` (`Reason.RATE_LIMITED`,
   `retryable()` true); TypeScript rejects `PluginApiError` 429. A scheduled sender should **hold the batch
   for the next tick**, not drop it. An invalid `link` (off-site, or outside your own subtree) is
   `INVALID_LINK`/400 and will fail identically next time.

There is **no read side** — you cannot list, count or mark an inbox, or learn whether anyone opened what
you sent. Nothing here reaches email.

**⚠ Do not copy ARCHITECTURE §17.1's own TS snippet.** It still reads
`send(...): Promise<void>` / `{ key, params?, link? }` — the proposal shape core could never implement (no
plugin catalog exists to resolve a `key` against, and `void` cannot express a partial send). What actually
ships is what this section shows. See `SKILL.md`'s "Which docs to trust" for the full trace.

### Test it against the refusals

```java
var ctx = new FakePluginContext();
ctx.store().asUser(ana).put(Scope.user(), "mark:s2e04:b3", true);   // this is what makes Ana notifiable
ctx.withNotifier(new FakeNotifier(ctx.store()).withPerUserPerDay(2));
```

```ts
const notify = makeMockNotify({ notifiable: ['u-1'], perUserPerDay: 2 });
const ctx = makeMockCtx({ notify });
```

`FakeNotifier` reads eligibility from the doc store rather than a list you seed — give a user a row and
they become notifiable, exactly as they became a participant. The cap is off until you arm it. `ctx.notify`
/ `ctx.notifier()` default to `null` in both, so a test that never passes one keeps checking your plugin
survives a manifest with no `notifications` block.

---

## 0.12.x → 0.13.0: rendering the people behind the UUIDs

```diff
  // plugin.json
- "platformApi": "0.12.0",
+ "platformApi": "0.13.0",
```

```diff
- implementation("dev.mosaicast:plugin-api:0.12.0")
+ implementation("dev.mosaicast:plugin-api:0.13.0")
- "@mosaicast/plugin-sdk": "^0.12.0"
+ "@mosaicast/plugin-sdk": "^0.13.0"
```

**Java plugins have nothing else to do** — `PluginContext` gained a method, plugins consume rather than
implement it, `FakePluginContext` implements the new one for you.

### The one compile break: a hand-built `ctx.user` in a TS test

`ctx.user` gained `displayName` and `avatarUrl`:

```diff
  const ctx = makeMockCtx({
-   user: { id: 'u1', role: 'podcaster' },
+   user: { id: 'u1', role: 'podcaster', displayName: 'Ana', avatarUrl: '/api/users/u1/avatar' },
  });
```

Anonymous is still `user: null` — nothing to change there. This is the compile break that would otherwise
silently break every `testing.md` example in this skill that predates 0.13.0; check any hand-built `ctx.user`
literal in your own tests for it.

### What the release adds

A backend calling `queryAcrossUsers` gets `OwnedDocEntry(userId, …)` — UUIDs and nothing else, so a
leaderboard built from it had ids and no way to draw a person:

```ts
const dir = ctx.users;
if (!dir) return;                      // no `identity` block in this plugin's manifest

const board = await ctx.docs.get<{ userId: string; score: number }[]>('site', 'agg:leaderboard');
const people = await dir.resolve((board ?? []).map((row) => row.userId));
const byId = new Map(people.map((u) => [u.id, u]));
```

```diff
  // plugin.json
+ "identity": { "resolvesUsers": true },
```

### The three rules that will catch you

1. **Absent, not redacted — and therefore not index-aligned.** An unknown, erased or pseudonymised id is
   simply missing; match on `id`, never on position.
2. **Store UUIDs, resolve at render. Never persist a display name.** The host cannot enforce this one —
   your storage's own tables are opaque to it.
3. **`avatarUrl` is finished.** Always `/api/users/{id}/avatar`, always populated — put it in an `src` and
   stop thinking about it.

It also **resolves rather than enumerates**: no list call, and none is coming.

### Test it against the absent case

```ts
const users = makeMockUsers({ 'u-1': 'Ana' });
const ctx = makeMockCtx({ users });
users.forget('u-1');                   // the erased-author case
```

```java
var users = new FakeUsers().withUser(ana, "Ana", Role.FAN);
var ctx = new FakePluginContext().withUsers(users);
```

---

## 0.11.x → 0.12.0: saying what language your pages are in

```diff
  // plugin.json
- "platformApi": "0.11.0",
+ "platformApi": "0.12.0",
```

```diff
- implementation("dev.mosaicast:plugin-api:0.11.0")
+ implementation("dev.mosaicast:plugin-api:0.12.0")
- "@mosaicast/plugin-sdk": "^0.11.0"
+ "@mosaicast/plugin-sdk": "^0.12.0"
```

**That is the whole migration for most plugins.** `OgMeta` and `SitemapUrl` each gained a component
(`locale`, `alternates`), which breaks *binary* compatibility — so you must rebuild — but the old shapes
survive as real constructors, so there is nothing to edit unless you deconstruct one of these records in a
pattern (`case OgMeta(var t, var d, var i)`) or call a canonical constructor reflectively:

```java
new OgMeta(title, description, imageUrl);       // still compiles: "whatever language the host resolved"
new SitemapUrl(loc, lastModified);               // still compiles: no translation group, as before
```

**Nothing on the TypeScript side changed at all** — `@mosaicast/plugin-sdk` moves to `0.12.0` only because
the two packages share one version anchor. No frontend code, hand-built `ctx`, or `makeMockCtx` call needs
touching for this step.

**What you may now want, if you implement `ShareMetadataProvider` or `SitemapProvider`:** core just shipped
per-locale URLs (`?lang=<code>`, `hreflang` alternates in `sitemap.xml`) and had nowhere to ask a plugin
what language its own pages are in — it deliberately emitted **no** alternates for plugin sitemap entries
rather than assume the site's UI languages apply to content it cannot read. `OgMeta.locale` and
`SitemapUrl.alternates` are that ask. Full detail and the honesty rules (list a language only if the page
is really written in it) are in `references/backend.md`'s "Optional extension points" section — worth
reading in full before you set either, since a wrong value announces a translation that is not really
there. Verify what you built with `SitemapProviderHarness` (`references/testing.md`) before shipping —
the host's failure mode for a bad translation group is silent (it drops the entry, or the whole group).

**Core-side:** core 0.6.24 built the `?lang=` URL scheme, confines every plugin alternate to that plugin's
own `/p/<pluginId>/` namespace exactly as it already did `loc`, drops an out-of-namespace alternate rather
than rejecting the whole entry, and drops the whole translation group if nothing is left naming `loc`'s own
language. `og:locale` on a plugin page is now documented as *the language of that URL*, not an install-wide
constant (ARCHITECTURE §6.4/§6.6) — read this before assuming a fixed `og:locale` if your plugin's pages
render in the site's active locale, since that is no longer a safe assumption to hardcode around.

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
