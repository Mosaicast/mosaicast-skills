# mosaicast-skills

Claude Code **plugin marketplace** for Mosaicast development. One source of truth for the shared skills used across the plugin repos — versioned and updatable, so changes propagate to everyone (incl. external plugin authors) without copying files into each repo.

Ships one plugin, **mosaicast-plugin-dev**, with three skills:

| Skill | Use for |
|---|---|
| `writing-a-mosaicast-plugin` | The plugin contract: `plugin.json`, slots and placements, data floors and `backendOwned`, consent, doc vs schema storage, the `blobs` file-storage block, the backend `PluginContext`, the frontend `ctx`, tests. Deep detail lives in `references/`. |
| `scaffolding-a-mosaicast-plugin` | Starting a new plugin repo from the sample: layout, rename ritual, gradle/Vite setup, `build.sh`, shared boilerplate, CI, SPDX/DCO. |
| `releasing-a-mosaicast-plugin` | Version anchors, `dist/` verification, installing into `MOSAICAST_PLUGINS_DIR`, and the load-failure modes to check. |

Current contract: **`platformApi` 0.8.0** (SDK 0.8.0, core 0.6.14 hosting platformApi 0.8.x).

## Use it
```bash
/plugin marketplace add mosaicast/mosaicast-skills   # once
/plugin install mosaicast-plugin-dev@mosaicast       # once
/plugin marketplace update                           # later, to pull new versions
```
After install, the skills trigger automatically when Claude works on a plugin (they're model-invoked).

## Versioning / update model
This behaves like a package:
- **Stable mode (default here):** `version` is set in `plugin.json` + the marketplace entry. Consumers get updates only when you **bump the version** — predictable, easy rollback.
- **Auto mode:** if you *omit* `version` and host in git, **every commit** counts as a new version — maximum propagation, less control.
- **Pinning/rollback:** the marketplace entry can pin a plugin to a commit SHA; revert the entry to roll everyone back.

Pick stable mode unless you want every commit to ship. Bump `version` in both `plugins/mosaicast-plugin-dev/.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` on each release — nothing enforces that they match.

## Maintenance
Keep the skills in sync with the real SDK, core and `mosaicast-plugin-sample`. Every SDK minor is a breaking change pre-1.0, so a release there means: re-check the pinned versions and signatures against the SDK and core sources (not their changelogs), update the skills, bump the version, push. Consumers run `/plugin marketplace update`.

## License
Apache-2.0 (permissive dev tooling) — see `LICENSE`.
