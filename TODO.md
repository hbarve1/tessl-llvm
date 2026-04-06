# TODO — tessl-llvm tile

Checklist for getting the LLVM Tessl tile from stub to a useful public registry package.

## Tile manifest

- [x] Align [`tiles/tessl-llvm/tile.json`](tiles/tessl-llvm/tile.json) with current Tessl schema (`steering`; tile at **0.2.0**).
- [x] Set a clear `summary` that names the audience and **pinned LLVM version** (**22.1.x**).
- [ ] Decide `describes` (package URL) — only if Tessl/registry supports a suitable identifier for LLVM; otherwise omit.
- [ ] Set `"private": false` when ready for irreversible public listing; keep private while iterating.
- [ ] Optionally set `entrypoint` to a registry landing file if not using default.

## Documentation (`tiles/tessl-llvm/docs/`)

- [x] Rewrite [`docs/index.md`](tiles/tessl-llvm/docs/index.md): scope, LLVM version contract, how agents should use this tile vs upstream docs.
- [x] Add focused pages: IR/types, New PM passes, TableGen, out-of-tree, version notes.
- [x] Link to official [LLVM release notes](https://releases.llvm.org/) for the pinned version; avoid stale inline API dumps.

## Steering (`tiles/tessl-llvm/rules/`)

- [x] Replace placeholder steering with [`llvm-language-implementer.md`](tiles/tessl-llvm/rules/llvm-language-implementer.md) (NPM, coding standards, verify APIs for 22.x).
- [x] Wire filenames in `tile.json` under `steering`.

## Skills (`tiles/tessl-llvm/skills/tessl-llvm/`)

- [x] Implement [`SKILL.md`](tiles/tessl-llvm/skills/tessl-llvm/SKILL.md) — New PM function pass workflow for LLVM 22.1.x.
- [x] Valid Agent Skills front matter (`name`, `description`).

## Quality and publish

- [x] `tessl install file:./tiles/tessl-llvm` — smoke test locally.
- [x] `tessl tile lint ./tiles/tessl-llvm` — fix all issues.
- [ ] Optional: `tessl tile pack --output ./dist ./tiles/tessl-llvm`.
- [ ] `tessl login` and `tessl tile publish` when content is ready.

## Repository hygiene

- [ ] Optionally add published tile to root [`tessl.json`](tessl.json) for dogfooding.
- [ ] Bump tile semver when LLVM or content meaningfully changes; document policy in README if needed.

## Maintenance

- [ ] On new LLVM releases: refresh version notes, adjust skills if APIs moved, bump tile version, republish.

## Older LLVM versions

- [ ] Add docs or separate version scope for LLVM 21.x and earlier (user follow-up).
