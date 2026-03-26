# TODO — tessl-llvm tile

Checklist for getting the LLVM Tessl tile from stub to a useful public registry package.
**Target: LLVM 20.x** (previous version support planned for later)

---

## Phase 1 — Tile manifest

- [ ] Align [`tiles/tessl-llvm/tile.json`](tiles/tessl-llvm/tile.json) with current Tessl schema (use `steering` instead of legacy `rules` key if required by `tessl tile lint`).
- [ ] Update `summary` to name the audience and pinned LLVM version: "LLVM 20.x tile for building compilers and language runtimes".
- [ ] Set `"private": false` when ready for public listing; keep private while iterating.
- [ ] Optionally set `describes` (package URL) if Tessl registry supports a suitable identifier for LLVM.

---

## Phase 2 — Documentation (`tiles/tessl-llvm/docs/`)

- [ ] Rewrite [`docs/index.md`](tiles/tessl-llvm/docs/index.md): scope, LLVM 20 version contract, how agents should use this tile vs upstream docs.
- [ ] Add `docs/ir-types.md` — LLVM IR types, values, constants, metadata in LLVM 20.
- [ ] Add `docs/new-pass-manager.md` — NPM architecture, PassManager/AnalysisManager, pass registration in LLVM 20.
- [ ] Add `docs/tablegen.md` — TableGen syntax, backends (LLVM, Clang), how to add records in LLVM 20.
- [ ] Add `docs/out-of-tree.md` — CMake setup, find_package(LLVM), linking libs, llvm-config usage for LLVM 20.
- [ ] Add `docs/codegen.md` — SelectionDAG / GlobalISel basics, MachineFunction, TargetMachine in LLVM 20.
- [ ] Add `docs/version-notes.md` — Key LLVM 20 API changes vs LLVM 19/18 (breaking changes agents must know).
- [ ] Link all pages to official [LLVM 20 release notes](https://releases.llvm.org/20.0.0/docs/ReleaseNotes.html).

---

## Phase 3 — Steering rules (`tiles/tessl-llvm/rules/`)

- [ ] Replace placeholder `rules/new-programming-language.md` with LLVM 20-oriented agent steering rules:
  - Always use New Pass Manager (NPM) — never legacy PM.
  - Verify API signatures against LLVM 20 headers before emitting code.
  - Use `llvm::Expected<T>` / `llvm::Error` for fallible operations, not raw pointers.
  - Follow LLVM coding standards (no `using namespace llvm` in headers, etc.).
  - Always check `LLVM_VERSION_MAJOR == 20` guards when referencing version-specific APIs.
  - Prefer `PatternMatch` / `IRBuilder` over manual IR construction.
  - Link to detailed doc pages for on-demand reference.

---

## Phase 4 — Skills (`tiles/tessl-llvm/skills/tessl-llvm/`)

### Skill 1: Add a New Pass Manager (NPM) pass
- [x] Write `skills/tessl-llvm/SKILL.md` — pass kinds, header, impl, PassRegistry.def, out-of-tree plugin, CMake, lit test, NPM cheat sheet, common mistakes.

### Skill 2: Set up an out-of-tree LLVM 20 project
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
- [ ] `tessl install file:./tiles/tessl-llvm` — smoke test locally.
- [ ] `tessl tile lint ./tiles/tessl-llvm` — fix all issues.
- [ ] Optional: `tessl tile pack --output ./dist ./tiles/tessl-llvm`.
- [ ] `tessl login` and `tessl tile publish` when content is ready.

---

## Phase 6 — Repository hygiene and maintenance

- [ ] Add published tile to root [`tessl.json`](tessl.json) for dogfooding.
- [ ] Bump tile semver when LLVM or content meaningfully changes.
- [ ] Document version policy in README.
- [ ] On new LLVM releases: refresh version notes, adjust skills if APIs moved, bump tile version, republish.
- [ ] Future: add LLVM 19/18 support as separate tile versions or variant docs.
