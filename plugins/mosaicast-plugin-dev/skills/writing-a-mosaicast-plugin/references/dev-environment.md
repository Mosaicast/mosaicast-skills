# Development environment — browser and a live instance

A plugin is a **rendered tile inside someone else's page**, at a size you do not control, in a theme you do
not own. Unit tests prove the logic; only a browser against a running host proves the thing the user
actually gets. Ask for both at the **start** of the work, not after the code is written.

**Neither is a blocker.** If the user cannot provide them, build the plugin anyway, in full, and say plainly
in your final message what could not be verified.

## 1. Browser access

Check whether you have browser tooling available (Playwright MCP or equivalent) before you start.

- **Available** → use it: load the host, mount the slot, screenshot, read the console. The sample repo
  already gitignores `.playwright-mcp/`, so this is the house tool.
- **Not available** → tell the user once, early:

  > I don't have browser access in this session. I can still build and unit-test the plugin, but I won't be
  > able to see the tile render, catch a CSP refusal, or check it on a phone width. If you can enable a
  > browser tool (e.g. the Playwright MCP server), I'll verify visually as I go.

  Then continue. Do not stall, do not ask again every turn, and do not degrade the deliverable.

Without a browser, be explicit about the gap at the end: *"unit-tested only; the tile has not been rendered
in a host."*

## 2. A Mosaicast instance to develop against

Ask the user for an instance you can install into and **restart** — a restart is how core picks up a
rebuilt plugin, so without permission to restart it, an instance is barely useful.

Three shapes, best first:

1. **A disposable dev stack.** If the user has a `mosaicast-core` checkout, `dev/screenshots.sh up` stands up
   an isolated, seeded Postgres + feed server + dev-profile app (on `:8081`), and `dev/screenshots.sh down`
   throws it away. Nothing real is at risk, so you can seed freely.
2. **Their instance with test/dummy data.** Fine to restart, fine to seed — after asking (see below).
3. **Their instance with production data.** Useful for rendering against real feeds. **Read-only** (see below).

Questions worth asking together, once:

- Is there an instance I can install this plugin into, and may I restart it?
- Does it hold production data or test data?
- What is `MOSAICAST_PLUGINS_DIR`, and how do you restart it?

If the answer is no instance: build, unit-test, and hand over `dist/` with the install steps. Say that it has
not been loaded by a host.

## 3. Data safety

**Production data: never insert, update or delete anything.** No seeded fixtures, no "just one test
episode", no writes through the plugin's own doc-store API to see if a write path works. Read, render,
screenshot — that is all. If a code path can only be exercised by writing, say so and ask for a test
instance rather than working around it.

**Test or dummy data: you may seed, but ask first.** Say what you intend to write and where — which scope,
which keys — and wait for a yes. Then prefer writing through the plugin's own surface (`ctx.api` from the
frontend, `ctx.store()` from the backend) over touching the database, so what you exercise is the real path.
Clean up what you seeded when you are done, or say what you left behind.

Three rules that hold in every case:

- Per-user data is written **as the signed-in user** against `data/user/me/…`. You cannot seed another
  person's partition — no API allows it, and the fact that you cannot is the security property. Test
  multi-user behaviour with the test kit's `InMemoryDocStore.asUser(...)`, not against a live host.
- Do not seed a key your manifest lists in `data.backendOwned` — the host will refuse the client write with
  a 403, and that refusal is correct.
- **Uploads are data too.** A file you upload while testing occupies the plugin's quota until something
  deletes it, and nothing collects orphans — so on a shared or production instance, ask first and remove
  what you stored (`ctx.blobs.remove(ref)` or the backend's `delete(ref)`) when you are done.

## 4. When you have both: test across viewports

A slot renders in regions with very different widths — `card` in a feed list, `player` in the player bar,
`sidebar` beside the episode body, `top` in the header — and core has recently been fixing phone-width
header and admin layout, so narrow widths are where breakage lives.

Test the tile at, at minimum:

| Viewport | Why |
|---|---|
| 375 × 667 (phone portrait) | the tightest column your tile will ever get |
| 667 × 375 (phone landscape) | short viewport — anything vertically greedy shows here |
| 768 × 1024 (tablet portrait) | the single/two-column breakpoint |
| 1280 × 800 (laptop) | the default reading width, and what README screenshots use |
| 1920 × 1080 (wide) | catches a tile that stretches instead of capping its measure |

Also check both **light and dark**: you consume `var(--mc-*)` tokens, so a hard-coded colour only shows up
when the host's theme flips.

What to look for, beyond "it renders": horizontal overflow inside a shadow root, text clipped in the `card`
one-liner, a control smaller than a touch target on a phone, and the browser console — a CSP refusal for an
undeclared consent host looks like "the embed just didn't load", not like an error.

Three things only a live host can prove, because the test kit deliberately cannot:

- **`ctx.route.navigate`** — click an internal link and watch the network panel: no bundle re-fetch, one new
  history entry, a working back button, and the URL under `/p/<id>/`. `replace: true` should add no entry.
  The mock records calls but has no router, so `route.path` never moves there.
- **Full-text search through `ctx.schema`** — `makeMockSchema`'s `search` is a case-insensitive substring
  match with no stemming and no `ts_rank` ordering. Ranking and stemming are only real against Postgres.
- **Uploads through `ctx.blobs`** — no double reads file formats, so the host's content sniffing (a `.png`
  that is not one) and its **effective** limits (operator caps and any admin grant, visible in
  Admin → the plugin's storage panel) can only be exercised live. Check both the happy path and a refusal:
  a file over the ceiling → 413, a type off the allow-list → 415.

## 5. The loop

```bash
./build.sh
rm -rf "$MOSAICAST_PLUGINS_DIR/<id>" && mkdir -p "$MOSAICAST_PLUGINS_DIR/<id>"
cp -r dist/* "$MOSAICAST_PLUGINS_DIR/<id>/"
# restart core, then reload the page
```

After every restart, check the **admin log viewer** first. A rejected manifest disables only your plugin,
quietly — the page just will not have your tile on it, which looks exactly like a render bug and is not one.
