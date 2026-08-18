---
name: scaffolding-a-mosaicast-plugin
description: Use when starting a new Mosaicast plugin repo from scratch, bootstrapping an empty mosaicast-plugin-* repo, copying the plugin-sample as a template, or refreshing a plugin repo's shared boilerplate (CI, DCO and license-header workflows, dependabot, pre-commit, SPDX, CLAUDE.md). Covers the rename ritual, the canonical folder layout, gradle and Vite setup, build.sh/install.sh, and the AGPL-vs-Apache license split. Use writing-a-mosaicast-plugin for the plugin contract itself.
---

# Scaffolding a Mosaicast plugin repo

Start from `mosaicast-plugin-sample` — it is maintained as a copy template and is the only plugin repo with
working code. Do **not** start from `mosaicast-plugin-bingo`, `-stats` or `-wiki`: they are bootstrap-only
repos whose docs are pre-0.4.0 and whose BRIEFs specify contracts that no longer exist (see "Bootstrapping an
existing empty repo" below).

For the plugin contract itself — manifest fields, `ctx`, storage, tests — use **writing-a-mosaicast-plugin**.

## Canonical layout

```
<repo>/
├── plugin.json                 # the manifest
├── build.sh                    # → dist/ only; never touches core
├── install.sh                  # optional, MOSAICAST_PLUGINS_DIR shortcut
├── CLAUDE.md  README.md  LICENSE  CONTRIBUTING.md  SECURITY.md  CODE_OF_CONDUCT.md
├── .editorconfig  .gitignore  .pre-commit-config.yaml
├── docs/ARCHITECTURE.md        # read-only spec, copied from core
├── docs/BRIEF.md               # what THIS repo builds — write it first, it is the DoD
├── .github/{dependabot.yml, pull_request_template.md, ISSUE_TEMPLATE/*,
│            workflows/{ci.yml, dco.yml, license-header.yml}}
├── backend/                    # Gradle, Java 21, PF4J
│   ├── build.gradle.kts settings.gradle.kts gradlew gradle/wrapper/
│   └── src/{main,test}/java/dev/mosaicast/plugin/<name>/
└── frontend/                   # Vite lib build → one ES bundle
    ├── package.json tsconfig.json vite.config.ts vitest.config.ts vitest.setup.ts
    ├── locales/en.json         # source language and fallback
    └── src/
```

Nothing forces this layout — core only reads `dist/` — but every tool, CI job and script below assumes it.

## Write `docs/BRIEF.md` first

It states what this plugin is, its scope, its public contract and its definition of done. Along with
`docs/ARCHITECTURE.md` (copied from core, **read-only**, it wins on conflict) it is what a fresh agent reads
before touching code. Do not start with the manifest.

## The rename ritual

Copying the sample means renaming it in nine places. Missing one produces a plugin that builds and does not load.

| What | Where |
|---|---|
| `id`, `name`, `version` | `plugin.json` (the `id` **must equal the installed folder name**) |
| element tags | `plugin.json` `frontend.elements` **and** each `defineMosaicastElement({ tag })` |
| bundle file name | `plugin.json` `frontend.entry` **and** `vite.config.ts` `lib.fileName` |
| jar name | `backend/build.gradle.kts` `tasks.jar { archiveFileName }` |
| Java package | `backend/src/**/dev/mosaicast/plugin/<name>/` + the `@Extension` class |
| gradle project name | `backend/settings.gradle.kts` |
| npm package name | `frontend/package.json` |
| install destination | `install.sh` `dest=` |
| dist assertions | `.github/workflows/ci.yml` package job (`dist/<id>.jar`, `dist/assets/<id>.es.js`) |
| license | `LICENSE`, every SPDX header, `CONTRIBUTING.md`, `README.md` |

Then strip the sample's demo content: its four fake `consent.services[]` entries exist to demonstrate all
four categories — a real plugin with no third parties **omits `consent` entirely** and the site stays
banner-free.

## `build.sh` and `install.sh`

`build.sh` writes only `dist/` (jar + `assets/` + `plugin.json`) and **never touches core**. Distribution is
a separate, manual step. `install.sh` is optional and must fail loudly if `MOSAICAST_PLUGINS_DIR` is unset —
never make the build depend on it.

```bash
#!/usr/bin/env bash
# SPDX-License-Identifier: <license>
# SPDX-FileCopyrightText: 2026 The Mosaicast Authors
set -euo pipefail
( cd backend  && ./gradlew --quiet clean jar )
( cd frontend && npm ci && npm run build )
rm -rf dist && mkdir -p dist/assets
cp backend/build/libs/*.jar dist/
cp frontend/build/*.es.js   dist/assets/
cp plugin.json              dist/
echo "✓ dist/ ready — copy to \$MOSAICAST_PLUGINS_DIR and restart core"
```

```bash
: "${MOSAICAST_PLUGINS_DIR:?Set MOSAICAST_PLUGINS_DIR or copy dist/ manually}"
dest="$MOSAICAST_PLUGINS_DIR/<id>"
rm -rf "$dest" && mkdir -p "$dest"
cp -r dist/* "$dest/"
```

## Backend setup

```kotlin
group = "dev.mosaicast.plugin"
version = "<plugin version>"
java { toolchain { languageVersion.set(JavaLanguageVersion.of(21)) } }

repositories {
    mavenLocal()          // dev-time: an SDK checkout, or includeBuild("../mosaicast-plugin-sdk")
    mavenCentral()
    maven {
        url = uri("https://maven.pkg.github.com/Mosaicast/mosaicast-plugin-sdk")
        credentials {     // GitHub Packages needs auth even for public reads
            username = providers.gradleProperty("gpr.user").orElse(providers.environmentVariable("GITHUB_ACTOR")).orNull
            password = providers.gradleProperty("gpr.token").orElse(providers.environmentVariable("GITHUB_TOKEN")).orNull
        }
    }
}

dependencies {
    compileOnly("dev.mosaicast:plugin-api:0.8.0")
    compileOnly("org.pf4j:pf4j:3.12.0")
    annotationProcessor("org.pf4j:pf4j:3.12.0")   // generates the extension index — without it nothing loads
    testImplementation(platform("org.junit:junit-bom:5.11.0"))
    testImplementation("org.junit.jupiter:junit-jupiter")
    testImplementation("dev.mosaicast:plugin-testkit:0.8.0")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
```

`compileOnly`, not `implementation` — the host provides the API. Never add Spring; PF4J classloading inside
core's fat JAR is fiddly and plain extensions are the mitigation.

## Frontend setup

Vite library build, ES format only, one file:

```ts
export default defineConfig({
  // The bundle is loaded straight into the browser (no consumer bundler), so bake NODE_ENV in — otherwise
  // a bundled framework references `process.env.NODE_ENV` and throws `process is not defined` at load.
  define: { 'process.env.NODE_ENV': JSON.stringify('production') },
  build: {
    outDir: 'build', emptyOutDir: true,
    lib: { entry: resolve(__dirname, 'src/<id>-element.tsx'), formats: ['es'], fileName: () => '<id>.es.js' },
    // No `rollupOptions.external` — the framework and the SDK must be bundled, so this plugin never
    // collides with the host's own React instance or SDK version.
  },
});
```

The sample uses React, deliberately confined to `frontend/` so another framework is a folder swap. `tsconfig`
is `noEmit`, so **`npm run typecheck` (`tsc --noEmit`) is the only thing that type-checks** — Vite does not.
Scripts: `build` / `test` / `typecheck`.

## Shared boilerplate — copy the sample's fixed versions

The three official plugin repos still carry the *broken originals* of these files. If you are copying from
anywhere but the sample, fix them:

- **`.pre-commit-config.yaml`**: the SPDX hook's `entry:` must be a **block scalar** (`>-`). The command
  contains `: `, which terminates an unquoted YAML value and makes the whole file unparseable — the hook
  then never runs and nobody notices.
- **`.github/dependabot.yml`**: gradle at `/backend`, npm at `/frontend`. Pointed at `/` it matches nothing.
- **Workflow actions**: `actions/checkout@v7`, `actions/setup-java@v5.x`, `actions/setup-node@v7`.
- **Default branch**: `master`, not `main`, in every workflow trigger.
- **`.gitignore`**: include `dist/`, `build/`, `node_modules/`, and `.playwright-mcp/`.
- Strip leftover template markers like `# Copy to: .github/workflows/dco.yml (in EVERY repo)`.

## CI — three jobs

1. **backend** — Temurin 21, gradle cache, `./gradlew --no-daemon clean build`, with
   `GITHUB_ACTOR`/`GITHUB_TOKEN` for the Packages read.
2. **frontend** — Node 20, `npm ci`, **typecheck**, build, test.
3. **package** — run `./build.sh`, then assert `dist/plugin.json`, `dist/<id>.jar` and
   `dist/assets/<id>.es.js` are non-empty, **and that `plugin.json`'s `platformApi` matches the
   `plugin-api:<version>` the backend compiled against**, then upload `dist/` as an artifact. Jobs 1 and 2
   stay green even if `build.sh` breaks, and the version lives in more than one file — this job is what
   catches both.

Plus the two org-wide workflows every repo carries: `dco.yml` (per-commit `Signed-off-by`, merges skipped)
and `license-header.yml` (SPDX header on changed source files).

## Licensing, SPDX, DCO

- **AGPL-3.0-or-later** for official feature plugins (bingo, stats, wiki) and core. **Apache-2.0** for the
  sample, the SDK and the skills repo — anything meant to be copied must not be copyleft.
- SPDX header in **every** source file, with the **fixed** holder — never inferred from git config:
  ```
  // SPDX-License-Identifier: <license>
  // SPDX-FileCopyrightText: 2026 The Mosaicast Authors
  ```
- `git commit -s` on every commit (DCO).
- README convention: title → one-line pitch → "Part of Mosaicast… Status" → What is this (pointing at
  ARCHITECTURE + BRIEF) → Build & test → Build & install → Contributing → License with the SPDX two-liner →
  **"Name & trademark: … please rename forks."**

## Seed `CLAUDE.md`

Keep it under ~200 lines. It must state: the mandatory reads (ARCHITECTURE wins on conflict; BRIEF defines
scope), the stack, the commands, and the binding conventions — including the current `platformApi` pin
(**0.8.0**, exact `major.minor` match, rejected at load on mismatch), SDK-only imports, per-user data in the
`USER` scope, the manifest `data` floors vs slot `visibleTo`, SPDX/DCO, and that tests are part of the work.
End with a pointer telling Claude to check for the **writing-a-mosaicast-plugin** skill before building, and
to pause and recommend installing it if absent.

## Bootstrapping an existing empty repo

`mosaicast-plugin-bingo`, `-stats` and `-wiki` are "Initial Bootstrap" repos: docs and community files only,
no code. Before implementing one:

- **Re-copy `docs/ARCHITECTURE.md` from core.** Theirs is a pre-0.4.0 snapshot that still shows the removed
  `consent: { categories, externalSources }` shape, a `Scope` without `USER`, and a `query` returning
  `List<JsonNode>`.
- **Re-read their `docs/BRIEF.md` critically.** Bingo's specifies `card:host:{userId}` / `card:fan:{userId}`
  doc keys under an episode scope — that is exactly the IDOR the `USER` scope exists to eliminate. Wiki's
  mandates schema storage, which core rejected when the BRIEF was written and now supports. Flag the
  deviation and rewrite the BRIEF rather than implementing it as written.
- Refresh the shared boilerplate per the list above before adding code.
