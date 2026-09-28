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

1. **A disposable dev stack.** If the user has a `mosaicast-core` checkout, `dev/instance.sh --name <plugin>
   up --plugin-dir "$PWD/dist"` stands up an isolated, seeded Postgres + feed server + dev-profile app under
   `/tmp/mosaicast-dev/<name>/`, and `dev/instance.sh --name <plugin> down` throws it all away. Nothing real
   is at risk, so you can seed freely — seeded with only the fictional sample feed, never real dev data.

   ```bash
   dev/instance.sh --name <plugin> up --plugin-dir "$PWD/dist"   # your own build; --plugins copies ./plugins instead
   dev/instance.sh --name <plugin> status                        # is it up, and which plugin ids actually loaded
   dev/instance.sh --name <plugin> logs [-f]                     # the app log — last 200 lines, or follow
   dev/instance.sh --name <plugin> psql                          # interactive shell on the fleeting database
   dev/instance.sh --name <plugin> psql -At -c "select 1"        # scriptable; -t only added when there's no TTY
   dev/instance.sh --name <plugin> restart                       # rebuilt plugin, same data — see §5
   dev/instance.sh --name <plugin> down                          # tear down only this name
   dev/instance.sh ls                                            # every instance: state, URL, core SHA, behind master?
   ```

   `--app-arg --some.property=value` (repeatable, on `up` or `restart`) passes an extra Spring property to
   the app — e.g. pointing `mosaicast.external.allowed-private-origins` at a local LibreTranslate for an
   `external` block. Properties the script already sets are refused. `restart` replays the app args (and
   plugin dirs) the name was started with unless you pass new ones.

   **Always pass `--name`, and pick one that's yours** (the plugin's own name is a good default). Named
   instances are fully isolated — own Postgres container, ports, feed server, process and plugins dir, all
   under `/tmp/mosaicast-dev/<name>/` — so several sessions (core, SDK, sample, wiki, bingo, stats, …) run
   side by side without colliding. **A session only ever `up`s/`down`s its own name**: `down` stops only the
   processes that name started (by recorded pid and lineage) and removes only its own container — there is
   no pattern-matched `pkill` reaching anyone else's instance, and `up` on a name still running is refused
   rather than torn down. Bare `dev/instance.sh up` (no `--name`) is the `default` instance, fixed at
   `:5433`/`:8081`/`:8099` — fine solo, a collision waiting to happen once anyone else is also running one.

   **Ports are allocated, not fixed** for any name other than `default` — read them from `env` rather than
   assuming a number:

   ```bash
   source <(dev/instance.sh --name <plugin> env)   # MC_APP_URL, MC_APP_PORT, MC_PG_PORT, …
   ```

   **`--plugins` and `--plugin-dir` are both opt-in and do different things.** `--plugins` copies the
   checkout's own `./plugins` folder (what the sample-screenshot flow uses; loads the sample plugin's demo
   card, which has no business in a real plugin's dev loop); `--plugin-dir PATH` (repeatable) copies one
   built plugin in as `<id>/` — the right flag for your own `dist/`, and it copies rather than links, so a
   rebuild needs a fresh `up` (or re-copy) to take effect. `status` reports which ids actually loaded by
   asking the running app, not by echoing the flag back, so it also catches a plugin that failed validation.
   `--core REF` pins a specific core commit instead of the working tree (default `origin/master` for a named
   instance) — resolved to a SHA and built once in a cached worktree, so your uncommitted core edits never
   leak into someone else's instance.

   **`--admin` is accepted but currently does nothing.** It is parsed into a flag the script never reads —
   `up` always logs the seeded feed in as `podcaster`, never as `admin`, whatever you pass. Don't rely on it
   for an `/admin/*` or `/account` screenshot; log in yourself against the running app instead:

   ```bash
   source <(dev/instance.sh --name <plugin> env)
   curl -s -c jar.txt "$MC_APP_URL/api/meta" >/dev/null
   xsrf=$(awk '/XSRF-TOKEN/{print $7}' jar.txt)
   curl -s -b jar.txt -c jar.txt -H "X-XSRF-TOKEN: $xsrf" \
     -X POST "$MC_APP_URL/api/auth/dev-login?role=admin"
   ```

   (`dev/instance.sh` defines a reusable `login <role>` shell function doing exactly this and echoing the
   cookie jar path — `source` the script and call it directly rather than retyping the curl pair, if you are
   scripting more than one request. Browsers scope cookies by host, not port, so two instances open in one
   browser profile on `localhost` log each other out — use one profile per instance.)
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

On a **named `dev/instance.sh` stack** (§2 above), don't `down`/`up` to pick up a rebuild — that reseeds the
database and throws away whatever you wrote to exercise the plugin. Use `restart` instead: it re-copies the
plugin dirs recorded at `up` (or new ones you pass), restarts only the app, and keeps the database, the feed
and the ports:

```bash
./build.sh
dev/instance.sh --name <plugin> restart      # new app, same data, same core SHA unless --core is given
```

`restart` refuses a `--core` whose newest migration is older than the database's, rather than starting an app
that can't read its own schema.

On **any other instance** (manual install, no `dev/instance.sh`), the manual copy is still the right tool —
it is your own unreleased build, so there is no tag to install by:

```bash
./build.sh
rm -rf "$MOSAICAST_PLUGINS_DIR/<id>" && mkdir -p "$MOSAICAST_PLUGINS_DIR/<id>"
cp -r dist/* "$MOSAICAST_PLUGINS_DIR/<id>/"
# restart core, then reload the page
```

After every restart, check the **admin log viewer** first. A rejected manifest disables only your plugin,
quietly — the page just will not have your tile on it, which looks exactly like a render bug and is not one.

## 6. Installing a *released* plugin by spec (core 0.6.15)

Hand-copying `dist/` still works and always will — it is the only option for your own unreleased build (§5
above). For a plugin that already has a GitHub release, there is now a second path that needs no filesystem
access to the host at all:

```bash
scripts/install-plugin.sh Mosaicast/mosaicast-plugin-wiki@v1.0.0#sha256:abc123…   # on a host checkout
```
```yaml
MOSAICAST_PLUGINS: "Mosaicast/mosaicast-plugin-wiki@v1.0.0#sha256:abc123…"        # in a container, resolved
                                                                                    # before the JVM starts
```

Also accepts a bare `owner/repo` (latest release), a tarball URL, or a local `.tgz` — each with an optional
`#sha256:…`. **No registry**: GitHub Releases are the index, reached through the plain
`releases/…/download/plugin.tgz` redirect — no API call, no token, no JSON parsing.

Worth carrying into any conversation about installing or releasing a plugin:

- **Plugins are read once, at startup — this has not changed.** The container resolves `MOSAICAST_PLUGINS`
  *before* the JVM boots for exactly that reason; anything arriving after is invisible until the next
  restart. The restart requirement in §2 above is not superseded by any of this.
- **The installed folder name comes from the manifest's own `id`, never the repo name.** A folder that
  disagrees with its manifest is rejected at load regardless of how it got there.
- **The container path resolves prebuilt tarballs only** — no git, no JDK in the runtime image. Clone-and-
  build is a developer-machine path (`scripts/install-plugin.sh` supports it there; the container does not).
- **Pin a tag *and* a checksum.** A plugin is trusted, in-process, unsandboxed code — this doesn't change
  that trust model, but making installation one env var away means the *documented default* should be the
  auditable form, not the shortest one. An unpinned `owner/repo` runs whatever that repo publishes next.
- Restarts are **idempotent**: an already-installed spec is recognised without re-downloading, because each
  install records the spec that produced it.

## 7. Publishing a plugin so others can install it this way

`dev/templates/release-plugin.yml` in `mosaicast-core` is a workflow to copy into a plugin repo as
`.github/workflows/release.yml`. On a published GitHub Release it runs `build.sh`, attaches a `plugin.tgz`
asset, refuses a tag that disagrees with the manifest's `version`, and appends the tarball's SHA-256 to the
release notes — an operator installing it has a pinned spec to copy, not a checksum to compute themselves.

**The asset name `plugin.tgz` is load-bearing, not a style choice.** The installer resolves `owner/repo@tag`
straight to `releases/download/<tag>/plugin.tgz` — a fixed URL, no API call. Rename the asset and every
`owner/repo` install for that plugin breaks.
