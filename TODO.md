# TODO — tessl-llvm tile

Checklist for getting the LLVM Tessl tile from stub to a useful public registry package.
**Target: LLVM 22.x** (older LLVM releases may get separate tile variants later)

---

## Phase 1 — Tile manifest

- [ ] Align [`tiles/tessl-llvm/tile.json`](tiles/tessl-llvm/tile.json) with current Tessl schema (use `steering` instead of legacy `rules` key if required by `tessl tile lint`).
- [ ] Update `summary` to name the audience and pinned LLVM version: "LLVM 22.x tile for building compilers and language runtimes".
- [ ] Set `"private": false` when ready for public listing; keep private while iterating.
- [ ] Optionally set `describes` (package URL) if Tessl registry supports a suitable identifier for LLVM.

---

## Phase 2 — Documentation (`tiles/tessl-llvm/docs/`)

- [x] Rewrite [`docs/index.md`](tiles/tessl-llvm/docs/index.md): scope, LLVM 22 version contract, how agents should use this tile vs upstream docs.
- [x] Add `docs/ir-types.md` — LLVM IR types, values, constants, metadata, IRBuilder patterns in LLVM 22.
- [x] Add `docs/new-pass-manager.md` — NPM architecture, PassBuilder, AnalysisManager, PreservedAnalyses, pass registration in LLVM 22.
- [x] Add `docs/tablegen.md` — TableGen syntax, multiclass, foreach, register/instr/intrinsic/CC defs, cmake integration.
- [x] Add `docs/out-of-tree.md` — CMake find_package(LLVM 22), llvm_map_components_to_libnames, component reference, pass plugin setup.
- [x] Add `docs/codegen.md` — TargetMachine, SelectionDAG ISelLowering, GlobalISel pipeline, MachineFunction/MachineInstr, MachineFunctionPass.
- [x] Add `docs/version-notes.md` — LLVM 22 breaking changes and migration notes (NPM, opaque pointers, `std::optional`, header moves, deprecations, etc.).
- [x] Link all pages to official [LLVM 22 release notes](https://releases.llvm.org/22.1.0/docs/ReleaseNotes.html).

---

## Phase 3 — Steering rules (`tiles/tessl-llvm/rules/`)

- [x] Replace placeholder `rules/new-programming-language.md` with LLVM 22-oriented agent steering rules covering: NPM only, opaque pointers, std::optional, API correctness, coding standards, IR construction, TableGen, out-of-tree CMake, GlobalISel preference — all linking to detailed doc pages.

---

## Phase 4 — Skills (`tiles/tessl-llvm/skills/tessl-llvm/`)

### Skill 1: Add a New Pass Manager (NPM) pass
- [x] Write `skills/tessl-llvm/SKILL.md` — pass kinds, header, impl, PassRegistry.def, out-of-tree plugin, CMake, lit test, NPM cheat sheet, common mistakes.

### Skill 2: Set up an out-of-tree LLVM 22 project
- [x] Write `skills/out-of-tree-setup/SKILL.md` — CMake find_package, llvm_map_components_to_libnames, minimal main.cpp with LLVMContext/IRBuilder, build instructions, optimization pipeline setup, pass plugin variant.

### Skill 3: Add a new IR intrinsic
- [x] Write `skills/add-intrinsic/SKILL.md` — TableGen definition, property table, type tokens, overloaded intrinsics, intrinsics_gen rebuild, IRBuilder::CreateIntrinsic, verifier rules, lit test, InstCombine fold example.

### Skill 4: Sync out-of-tree project to a new LLVM version
- [x] Write `skills/version-sync/SKILL.md` — git branch, CMake bump, opaque pointer migration, legacy PM removal, header renames, API renames, TableGen changes, C++17 requirement, version guards, checklist.

### Skill 5: Add a new target backend (skeleton)
- [x] Write `skills/new-target/SKILL.md` — Triple registration, directory structure, TableGen register/instr stubs, MCTargetDesc, TargetMachine, TargetRegistry wiring, CMake with tablegen targets, build verification.

---

## Phase 5 — Quality and publish

- [x] Update `tile.json` to reference all new skill paths.
- [x] `tessl install file:./tiles/tessl-llvm` — ✔ Installed hbarve1/tessl-llvm@0.1.0
- [x] `tessl tile lint ./tiles/tessl-llvm` — ✔ Tile is valid (1.8k front-loaded, 9.9k on-demand, 13.6k content tokens)
- [ ] Optional: `tessl tile pack --output ./dist ./tiles/tessl-llvm`.
- [ ] `tessl login` and `tessl tile publish` when content is ready.

---

## Phase 7 — Language implementation layer (next up)

### Critical — Every real language needs these

#### Task 1: Frontend → IR lowering
- [x] Add `docs/frontend-to-ir.md` — AST lowering patterns: expressions, if/else, loops, functions, closures, structs, arrays
- [x] Add `skills/frontend-to-ir/SKILL.md` — full CodeGenCtx, type mapping, expression/statement/block emit, mem2reg cleanup

#### Task 2: Debug info (DWARF)
- [x] Add `docs/debug-info.md` — DIBuilder, DICompileUnit, DISubprogram, DILocalVariable, DILocation, module flags
- [x] Add `skills/add-debug-info/SKILL.md` — step-by-step: DIBuilder init, types, subprograms, param/local declares, source locations, finalize

#### Task 3: ORC JIT v2
- [x] Add `docs/jit.md` — LLJIT, LLLazyJIT, ThreadSafeModule, DynamicLibrarySearchGenerator, optimization transform, REPL pattern, multiple JITDylibs
- [x] Add `skills/jit-setup/SKILL.md` — step-by-step: CMake, target init, LLJIT, symbol exposure, IR module add, optimization, lookup/call, lazy JIT, REPL

#### Task 4: Exception handling
- [x] Add `docs/exception-handling.md` — invoke, landingpad, personality functions, __cxa_begin/end_catch, resume, cleanup, Itanium/Windows SEH, full try/catch pattern

### High — Production quality

#### Task 5: Lit / FileCheck testing
- [x] Add `skills/lit-filecheck/SKILL.md` — RUN lines, CHECK directives, captures, negative checks, IR/codegen/plugin tests, lit.cfg.py, debugging failures

#### Task 6: Garbage collection / statepoints
- [x] Add `docs/gc-statepoints.md` — gcroot shadow stack, gc.statepoint/gc.relocate/gc.result, RewriteStatepointsForGC pass, StackMap section, custom GC strategy

#### Task 7: Attributes & loop metadata
- [x] Add `docs/attributes-metadata.md` — function/param/return attributes table, loop hints (!llvm.loop), branch weights (!prof), TBAA, !range, !nonnull, !align

### Medium — Advanced use cases

#### Task 8: Custom calling conventions
- [ ] Add `docs/calling-conventions.md` — CallingConv enum, TableGen CallingConv.td, CCState, CCValAssign, full custom CC walkthrough
- [ ] Add `skills/add-calling-convention/SKILL.md` — skill: define and wire a custom calling convention end-to-end

#### Task 9: LTO / ThinLTO
- [ ] Add `docs/lto.md` — LTO vs ThinLTO, plugin interface, `lld` integration, LLVM IR bitcode, summary index

#### Task 10: Custom alias analysis
- [ ] Add `docs/alias-analysis.md` — AAResult, AliasAnalysis NPM pass, ModRef, writing language-specific AA (e.g., no-alias for immutable values)

---

## Phase 6 — Repository hygiene and maintenance

- [x] Add published tile to root [`tessl.json`](tessl.json) for dogfooding — already present as `file:./tiles/tessl-llvm`.
- [ ] Bump tile semver when LLVM or content meaningfully changes.
- [ ] Document version policy in README.
- [ ] On new LLVM releases: refresh version notes, adjust skills if APIs moved, bump tile version, republish.
- [ ] Future: add LLVM 21 and older support as separate tile versions or variant docs.
