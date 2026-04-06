# Version notes — tile vs LLVM

## Tile `0.2.0`

- Targets **LLVM 22.1.x**, aligned with tag **`llvmorg-22.1.2`** as the reference patch level.
- Docs emphasize **New Pass Manager**, **opaque `ptr`**, and **TableGen-first** target work.

## When LLVM bumps (e.g. 23.x)

- Refresh [index.md](index.md) version strings and upstream links.
- Re-read **release notes** for pass pipeline, IR, or CMake changes; update [new-pm-passes.md](new-pm-passes.md) and [ir-and-types.md](ir-and-types.md) if LangRef or defaults shifted.
- Bump **tile semver** per Tessl guidelines and republish.

## Older LLVM branches

Not covered in this tile revision; a future release may add `docs/llvm-21.md` or separate tiles—see [index.md](index.md).
