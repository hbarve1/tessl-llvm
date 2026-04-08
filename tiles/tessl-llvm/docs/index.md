# tessl-llvm — LLVM 22.x Tile

**Audience:** Compiler engineers, language implementers, and tooling authors building on LLVM 22.

**Version contract:** All content in this tile is pinned to **LLVM 22.x** (specifically the LLVM 22.1.2 release). API signatures, pass names, header paths, and CMake variables reflect LLVM 22 — not LLVM 19/20/21. When in doubt, verify against the [LLVM 22 source](https://github.com/llvm/llvm-project/tree/llvmorg-22.1.2) or [LLVM 22 release notes](https://releases.llvm.org/22.1.0/docs/ReleaseNotes.html).

---

## How to use this tile

This tile provides **three layers** of LLVM-specific assistance:

| Layer | Purpose | Files |
|-------|---------|-------|
| **Docs** | On-demand reference — consult these pages for API details, architecture, and examples | `docs/*.md` |
| **Steering rules** | Always-loaded agent guidance — concise rules that shape every LLVM code generation decision | `rules/new-programming-language.md` |
| **Skills** | Step-by-step workflows — invoke a skill to execute a specific LLVM task end-to-end | `skills/*/SKILL.md` |

**When to use this tile vs upstream docs:**
- Use this tile first for LLVM 22 API signatures, CMake patterns, pass registration, and common workflows.
- Use [official LLVM docs](https://llvm.org/docs/) and [Doxygen](https://llvm.org/doxygen/) for exhaustive API coverage, rarely-used APIs, and language reference grammar.
- Use [LLVM release notes](https://releases.llvm.org/22.1.0/docs/ReleaseNotes.html) for the authoritative list of LLVM 22 breaking changes.

---

## Documentation pages

| Page | What it covers |
|------|---------------|
| [IR Types & Values](ir-types.md) | Types, constants, metadata, IRBuilder patterns |
| [New Pass Manager](new-pass-manager.md) | NPM architecture, PassBuilder, AnalysisManager, pass pipeline |
| [TableGen](tablegen.md) | TableGen syntax, backends, adding records for registers/instructions/intrinsics |
| [Out-of-Tree Projects](out-of-tree.md) | CMake setup, find_package(LLVM 22), component linking |
| [Code Generation](codegen.md) | SelectionDAG, GlobalISel, MachineFunction, TargetMachine |
| [Frontend → IR Lowering](frontend-to-ir.md) | AST lowering: expressions, control flow, functions, closures, structs |
| [Debug Info (DWARF)](debug-info.md) | DIBuilder, DISubprogram, DILocalVariable, source locations |
| [ORC JIT v2](jit.md) | LLJIT, LLLazyJIT, ThreadSafeModule, symbol resolution, REPL pattern |
| [Exception Handling](exception-handling.md) | invoke, landingpad, personality functions, cleanup/catch |
| [GC & Statepoints](gc-statepoints.md) | gcroot (legacy), gc.statepoint / gc.relocate, StackMap |
| [Attributes & Metadata](attributes-metadata.md) | Function/param attributes, loop hints, branch weights, TBAA, !range |
| [Calling Conventions](calling-conventions.md) | `CallingConv`, TableGen CC, `CCState` / `CCValAssign`, ABI lowering |
| [LTO & ThinLTO](lto.md) | Bitcode, full vs thin link, Clang/LLD flags, summaries |
| [Alias Analysis](alias-analysis.md) | `AliasResult`, `MemoryLocation`, ModRef, TBAA / `noalias`, custom AA |
| [LLVM 22 Version Notes](version-notes.md) | Breaking changes from LLVM 19/20/21 → 22 |

---

## Available skills

| Skill | Invoke when... |
|-------|---------------|
| `add-npm-pass` | Adding a new optimization or analysis pass |
| `out-of-tree-setup` | Starting a new compiler/tool project against LLVM 22 |
| `add-intrinsic` | Adding a new `llvm.*` IR intrinsic |
| `version-sync` | Migrating an existing project to LLVM 22 |
| `new-target-backend` | Adding a new ISA target backend |
| `frontend-to-ir` | Lowering an AST to LLVM IR with IRBuilder |
| `add-debug-info` | Adding DWARF debug info to a frontend |
| `jit-setup` | Setting up ORC JIT v2 for a language runtime or REPL |
| `lit-filecheck` | Writing lit/FileCheck tests for passes, transforms, or codegen |
| `add-calling-convention` | Defining or wiring a calling convention (in-tree target or IR-level ABI) |
| `add-exception-handling` | Adding try/catch/finally with `invoke`/`landingpad` and the Itanium ABI |
| `add-gc-statepoints` | Adding GC support: shadow-stack (gcroot) or relocating collector (statepoints) |
| `add-attributes-metadata` | Annotating IR with `nounwind`/`nocapture`, loop hints, TBAA, branch weights |
| `add-lto` | Enabling full LTO or ThinLTO for cross-module optimization |
| `add-alias-analysis` | Querying `AAResults`, emitting `noalias`/TBAA hints, or writing a custom AA pass |
| `add-sanitizer` | Instrumenting output with ASan, UBSan, or custom runtime checks |
| `add-vectorization-hint` | Guiding the loop vectorizer and SLP vectorizer via loop metadata |
| `lower-struct-types` | Lowering structs, unions, tuples, and nested composites to LLVM IR |

---

## LLVM 22 at a glance — things that changed

- **Legacy PassManager still present but never use it for new code.** Use NPM (`PassInfoMixin`, `FunctionAnalysisManager`, etc.) for all passes.
- **Opaque pointers enforced.** `i8*`, `i32*` etc. are gone — only `ptr`.
- **`llvm::Optional` removed.** Use `std::optional` / `std::nullopt`.
- **`Triple.h` moved** to `llvm/TargetParser/Triple.h` (old `llvm/ADT/Triple.h` removed).
- **`Intrinsic::getDeclaration()` removed** — use `getOrInsertDeclaration()` or `getDeclarationIfExists()`.
- **`DbgInstPtr` return type** on `DIBuilder::insertDeclare` and `insertDbgValueIntrinsic` — can be `Instruction*` or `DbgRecord*`.
- **C++17 required** for all downstream projects.
- **MCJIT removed** — use ORC JIT v2 (`LLJIT` / `LLLazyJIT`).

See [version-notes.md](version-notes.md) for the full migration reference.

---

## Tile audit

[AUDIT.md](../AUDIT.md) — gap analysis, full skill inventory, and suggested future additions.

---

## Scope of this tile (v0.5.0)

**In scope:**
- IR construction and manipulation (IRBuilder, Module, Function, BasicBlock)
- New Pass Manager passes and analyses
- Out-of-tree project setup (CMake, linking)
- TableGen for registers, instructions, and intrinsics
- SelectionDAG and GlobalISel codegen basics
- Calling conventions, LTO/ThinLTO, and alias analysis concepts
- LLVM 22 migration guidance

**Out of scope (v0.1.0):**
- Clang/Flang frontend internals
- MLIR (separate tile)
- LLD linker internals
- Polly / BOLT
- LLVM 19/18/17 — see `version-notes.md` for diff; prior version support planned for a later tile release
