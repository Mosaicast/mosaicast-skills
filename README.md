# mosaicast-skills

Claude Code **plugin marketplace** for Mosaicast development. One source of truth for the shared skills used across the plugin repos — versioned and updatable, so changes propagate to everyone (incl. external plugin authors) without copying files into each repo.

Currently ships one plugin:
- **mosaicast-plugin-dev** → skill `writing-a-mosaicast-plugin` (manifest, slots, ctx, doc/schema storage, test kit, build.sh, SPDX).

## Use it
```bash
/plugin marketplace add mosaicast/mosaicast-skills   # once
/plugin install mosaicast-plugin-dev@mosaicast       # once
/plugin marketplace update                           # later, to pull new versions
```
After install, the skill triggers automatically when Claude works on a plugin (it's model-invoked).

## Versioning / update model
This behaves like a package:
- **Stable mode (default here):** `version` is set in `plugin.json` + the marketplace entry. Consumers get updates only when you **bump the version** — predictable, easy rollback.
- **Auto mode:** if you *omit* `version` and host in git, **every commit** counts as a new version — maximum propagation, less control.
- **Pinning/rollback:** the marketplace entry can pin a plugin to a commit SHA; revert the entry to roll everyone back.

Pick stable mode unless you want every commit to ship. Bump `version` in both `plugin.json` and `.claude-plugin/marketplace.json` on each release.

## Maintenance
Keep the skill in sync with the real SDK + sample. After the SDK and `mosaicast-plugin-sample` exist, refine `SKILL.md` to match the actual built signatures, bump the version, push. Consumers run `/plugin marketplace update`.

## License
Apache-2.0 (permissive dev tooling) — see `LICENSE`. Replace the stub with the full text before the first public commit.
