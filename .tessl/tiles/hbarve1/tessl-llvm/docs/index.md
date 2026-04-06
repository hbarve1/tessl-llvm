# LLVM 22.1.x — Tessl tile

This tile orients coding agents for work on **LLVM 22.1.x** (latest stable at tile authoring: **22.1.2**). Use it when building compilers, backends, Clang tooling, or out-of-tree LLVM consumers.

## What this tile is for

- **Scoped context** — Reduces answers rooted in outdated LLVM releases.
- **On-demand docs** — Split by topic so agents load [IR](ir-and-types.md), [passes](new-pm-passes.md), [TableGen](tablegen.md), or [out-of-tree](out-of-tree.md) as needed.
- **Steering + skill** — Always-on conventions plus a procedural workflow in the bundled skill.

## Authoritative upstream sources

Always treat these as canonical; this tile summarizes and routes.

| Resource | URL |
|----------|-----|
| Release notes (22.1.0) | https://releases.llvm.org/22.1.0/docs/ReleaseNotes.html |
| Language Reference | https://llvm.org/docs/LangRef.html |
| Programmer’s Manual | https://llvm.org/docs/ProgrammersManual.html |
| Coding Standards | https://llvm.org/docs/CodingStandards.html |
| TableGen | https://llvm.org/docs/TableGen/index.html |

Patch releases (22.1.1, 22.1.2, …) follow the same major/minor API surface unless a release note says otherwise.

## Document map

- [IR and types](ir-and-types.md)
- [New Pass Manager and passes](new-pm-passes.md)
- [TableGen](tablegen.md)
- [Out-of-tree LLVM](out-of-tree.md)
- [Version notes for this tile](version-notes.md)

## Older LLVM versions

Future tile updates may add parallel docs or versions for LLVM 21.x and earlier. For now, **only 22.1.x** is in scope.
