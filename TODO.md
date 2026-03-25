# TODO — tessl-llvm tile

Checklist for getting the LLVM Tessl tile from stub to a useful public registry package.

## Tile manifest

- [ ] Align [`tiles/tessl-llvm/tile.json`](tiles/tessl-llvm/tile.json) with current Tessl schema (e.g. use `steering` instead of legacy `rules` key if required by `tessl tile lint`).
- [ ] Set a clear `summary` that names the audience and **pinned LLVM version** (e.g. LLVM 19.x).
- [ ] Decide `describes` (package URL) — only if Tessl/registry supports a suitable identifier for LLVM; otherwise omit.
- [ ] Set `"private": false` when ready for irreversible public listing; keep private while iterating.
- [ ] Optionally set `entrypoint` to a registry landing file if not using default.

## Documentation (`tiles/tessl-llvm/docs/`)

- [ ] Rewrite [`docs/index.md`](tiles/tessl-llvm/docs/index.md): scope, LLVM version contract, how agents should use this tile vs upstream docs.
- [ ] Add focused pages (split for MCP): IR/types, New PM passes, TableGen, out-of-tree workflow, version/changelog notes vs previous tile release.
- [ ] Link to official [LLVM release notes](https://releases.llvm.org/) for the pinned version; avoid stale inline API dumps.

## Steering (`tiles/tessl-llvm/rules/`)

- [ ] Replace placeholder steering markdown with concise LLVM-oriented rules (NPM framing, coding standards, verify APIs against pinned version).
- [ ] Wire filenames in `tile.json` under `steering`.

## Skills (`tiles/tessl-llvm/skills/tessl-llvm/`)

- [ ] Implement [`SKILL.md`](tiles/tessl-llvm/skills/tessl-llvm/SKILL.md) with a real workflow (e.g. add/update a New PM pass, or sync out-of-tree tooling to a new LLVM).
- [ ] Valid Agent Skills front matter (`name`, `description`).

## Quality and publish

- [ ] `tessl install file:./tiles/tessl-llvm` — smoke test locally.
- [ ] `tessl tile lint ./tiles/tessl-llvm` — fix all issues.
- [ ] Optional: `tessl tile pack --output ./dist ./tiles/tessl-llvm`.
- [ ] `tessl login` and `tessl tile publish` when content is ready.

## Repository hygiene

- [ ] Optionally add published tile to root [`tessl.json`](tessl.json) for dogfooding.
- [ ] Bump tile semver when LLVM or content meaningfully changes; document policy in README if needed.

## Maintenance

- [ ] On new LLVM releases: refresh version notes, adjust skills if APIs moved, bump tile version, republish.
