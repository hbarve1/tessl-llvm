# tessl-llvm

This repository maintains a **[Tessl](https://tessl.io/) tile** aimed at people building compilers, language runtimes, and other tooling on **LLVM 22.x**. The tile packages documentation, steering (always-on guidance for agents procedural skills so coding assistants get **version-aware context** instead of guessing against outdated LLVM APIs and workflows.

## Why this exists

LLVM moves quickly: the New Pass Manager, IR details, TableGen, and out-of-tree layouts change between releases. Generic model training lags behind the tree developers actually use. A Tessl tile gives agents **curated, explicit** context tied to a **pinned LLVM release**, and ships through the same install and update path as other Tessl registry content.

## Goals

- **Context for language implementers** — IR, passes, codegen interfaces, TableGen, and practical workflows (in-tree vs out-of-tree) explained in agent-friendly chunks.
- **Explicit LLVM version alignment** — Each published tile version should clearly state which upstream LLVM release it targets; refresh content and semver when LLVM moves.
- **Public registry tile** — Publish a discoverable package (e.g. `hbarve1/tessl-llvm`) so anyone can `tessl install` it into their project.
- **Skills + docs + steering** — Combine on-demand docs, mandatory conventions, and step-by-step skills so agents both *search* and *execute* correctly.

## Repository layout

| Path | Purpose |
|------|---------|
| [`tiles/tessl-llvm/`](tiles/tessl-llvm/) | Tile root: `tile.json`, `docs/`, `rules/`, `skills/` |
| [`tessl.json`](tessl.json) | Project manifest for Tessl dependencies (optional once the tile is published) |

Tile authoring and publishing follow the [Tessl docs](https://docs.tessl.io/) (creating tiles, developing locally, distributing via registry).

## Non-goals

- Replacing the official LLVM manuals or release notes — this tile **summarizes and routes**; upstream remains canonical for full detail.
- Exhaustive API coverage for every LLVM subsystem in v1 — start narrow, expand based on real language-implementer needs.

## Contributing

Work is tracked in [`TODO.md`](TODO.md). After changes, validate with `tessl tile lint` on the tile directory and publish per Tessl’s registry workflow when ready.
