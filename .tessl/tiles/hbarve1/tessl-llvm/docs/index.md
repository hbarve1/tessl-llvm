# tessl-llvm — LLVM 20.x Tile

**Audience:** Compiler engineers, language implementers, and tooling authors building on LLVM 20.

**Version contract:** All content in this tile is pinned to **LLVM 20.x** (specifically the LLVM 20.0 release line). API signatures, pass names, header paths, and CMake variables reflect LLVM 20 — not LLVM 17/18/19. When in doubt, verify against the [LLVM 20 source](https://github.com/llvm/llvm-project/tree/llvmorg-20.0.0) or [LLVM 20 release notes](https://releases.llvm.org/20.0.0/docs/ReleaseNotes.html).

---

## How to use this tile

This tile provides **three layers** of LLVM-specific assistance:

| Layer | Purpose | Files |
|-------|---------|-------|
| **Docs** | On-demand reference — consult these pages for API details, architecture, and examples | `docs/*.md` |
| **Steering rules** | Always-loaded agent guidance — concise rules that shape every LLVM code generation decision | `rules/new-programming-language.md` |
| **Skills** | Step-by-step workflows — invoke a skill to execute a specific LLVM task end-to-end | `skills/*/SKILL.md` |

**When to use this tile vs upstream docs:**
- Use this tile first for LLVM 20 API signatures, CMake patterns, pass registration, and common workflows.
- Use [official LLVM docs](https://llvm.org/docs/) and [Doxygen](https://llvm.org/doxygen/) for exhaustive API coverage, rarely-used APIs, and language reference grammar.
- Use [LLVM release notes](https://releases.llvm.org/20.0.0/docs/ReleaseNotes.html) for the authoritative list of LLVM 20 breaking changes.

---

## Documentation pages

| Page | What it covers |
|------|---------------|
| [IR Types & Values](ir-types.md) | Types, constants, metadata, IRBuilder patterns |
| [New Pass Manager](new-pass-manager.md) | NPM architecture, PassBuilder, AnalysisManager, pass pipeline |
| [TableGen](tablegen.md) | TableGen syntax, backends, adding records for registers/instructions/intrinsics |
| [Out-of-Tree Projects](out-of-tree.md) | CMake setup, find_package(LLVM 20), component linking |
| [Code Generation](codegen.md) | SelectionDAG, GlobalISel, MachineFunction, TargetMachine |
| [LLVM 20 Version Notes](version-notes.md) | Breaking changes from LLVM 17/18/19 → 20 |

---

## Available skills

| Skill | Invoke when... |
|-------|---------------|
| `add-npm-pass` | Adding a new optimization or analysis pass |
| `out-of-tree-setup` | Starting a new compiler/tool project against LLVM 20 |
| `add-intrinsic` | Adding a new `llvm.*` IR intrinsic |
| `version-sync` | Migrating an existing project to LLVM 20 |
| `new-target-backend` | Adding a new ISA target backend |

---

## LLVM 20 at a glance — things that changed

- **Legacy PassManager removed.** Only the New Pass Manager (NPM) exists.
- **Opaque pointers enforced.** `i8*`, `i32*` etc. are gone — only `ptr`.
- **`llvm::Optional` removed.** Use `std::optional` / `std::nullopt`.
- **`Triple.h` moved** to `llvm/TargetParser/Triple.h`.
- **`Intrinsic::getDeclaration()` deprecated** — use `getOrInsertDeclaration()`.
- **C++17 required** for all downstream projects.

See [version-notes.md](version-notes.md) for the full migration reference.

---

## Scope of this tile (v0.1.0)

**In scope:**
- IR construction and manipulation (IRBuilder, Module, Function, BasicBlock)
- New Pass Manager passes and analyses
- Out-of-tree project setup (CMake, linking)
- TableGen for registers, instructions, and intrinsics
- SelectionDAG and GlobalISel codegen basics
- LLVM 20 migration guidance

**Out of scope (v0.1.0):**
- Clang/Flang frontend internals
- MLIR (separate tile)
- LLD linker internals
- Polly / BOLT
- LLVM 19/18/17 — see `version-notes.md` for diff; prior version support planned for a later tile release
