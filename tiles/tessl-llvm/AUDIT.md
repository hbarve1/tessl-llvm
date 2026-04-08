# Skills Audit Report — himank-test/tessl-llvm

**Date:** 2026-04-08  
**Tile version at time of audit:** 0.4.0  

---

## Methodology

Compared every file in `docs/` against the skills registered in `tile.json`. Any doc without a corresponding skill was flagged as a gap. Additionally, common LLVM tasks not yet covered by any doc or skill were noted.

---

## Gaps found at audit time

### 1. Doc-backed gaps (doc existed, skill did not)

| Doc file | Missing skill ID | Added? |
|---|---|---|
| `docs/exception-handling.md` | `add-exception-handling` | ✅ |
| `docs/gc-statepoints.md` | `add-gc-statepoints` | ✅ |
| `docs/attributes-metadata.md` | `add-attributes-metadata` | ✅ |
| `docs/lto.md` | `add-lto` | ✅ |
| `docs/alias-analysis.md` | `add-alias-analysis` | ✅ |

### 2. Additional gaps (no doc or skill existed)

| Topic | Missing skill ID | Added? |
|---|---|---|
| Sanitizer instrumentation (ASan, UBSan, TSan) | `add-sanitizer` | ✅ |
| Auto-vectorization and loop hints | `add-vectorization-hint` | ✅ |
| Struct / union / tuple type lowering | `lower-struct-types` | ✅ |

---

## Skill inventory after audit

| Skill ID | Skill file | Backed by doc? |
|---|---|---|
| `add-npm-pass` | `skills/tessl-llvm/SKILL.md` | `docs/new-pass-manager.md` |
| `out-of-tree-setup` | `skills/out-of-tree-setup/SKILL.md` | `docs/out-of-tree.md` |
| `add-intrinsic` | `skills/add-intrinsic/SKILL.md` | `docs/ir-types.md` |
| `version-sync` | `skills/version-sync/SKILL.md` | `docs/version-notes.md` |
| `new-target-backend` | `skills/new-target/SKILL.md` | `docs/codegen.md` |
| `frontend-to-ir` | `skills/frontend-to-ir/SKILL.md` | `docs/frontend-to-ir.md` |
| `add-debug-info` | `skills/add-debug-info/SKILL.md` | `docs/debug-info.md` |
| `jit-setup` | `skills/jit-setup/SKILL.md` | `docs/jit.md` |
| `lit-filecheck` | `skills/lit-filecheck/SKILL.md` | _(standalone skill)_ |
| `add-calling-convention` | `skills/add-calling-convention/SKILL.md` | `docs/calling-conventions.md` |
| `add-exception-handling` | `skills/add-exception-handling/SKILL.md` | `docs/exception-handling.md` |
| `add-gc-statepoints` | `skills/add-gc-statepoints/SKILL.md` | `docs/gc-statepoints.md` |
| `add-attributes-metadata` | `skills/add-attributes-metadata/SKILL.md` | `docs/attributes-metadata.md` |
| `add-lto` | `skills/add-lto/SKILL.md` | `docs/lto.md` |
| `add-alias-analysis` | `skills/add-alias-analysis/SKILL.md` | `docs/alias-analysis.md` |
| `add-sanitizer` | `skills/add-sanitizer/SKILL.md` | _(no doc yet)_ |
| `add-vectorization-hint` | `skills/add-vectorization-hint/SKILL.md` | _(no doc yet)_ |
| `lower-struct-types` | `skills/lower-struct-types/SKILL.md` | _(no doc yet)_ |

**Total skills: 18**

---

## Remaining doc-only content (no skill yet)

These docs exist but cover concepts that are typically referenced rather than acted on step-by-step, so a procedural skill was not added. Revisit if users request guided workflows for these topics.

| Doc file | Notes |
|---|---|
| `docs/tablegen.md` | Covered implicitly by `new-target-backend`; could become a standalone `write-tablegen-defs` skill |
| `docs/ir-types.md` | General reference; covered by `frontend-to-ir` and `lower-struct-types` |
| `docs/codegen.md` | Deep-dive reference; covered by `new-target-backend` |
| `docs/index.md` | Tile landing page — not a skill target |

---

## Suggested future additions

| Skill ID | Description |
|---|---|
| `write-tablegen-defs` | Step-by-step: define registers, instructions, and calling conventions in TableGen for a new target |
| `add-sanitizer-runtime` | Write a custom sanitizer runtime (shadow memory, callbacks) rather than using Clang's |
| `emit-wasm-target` | LLVM 22 WebAssembly target specifics: emscripten triple, exception model, wasm-specific intrinsics |
| `add-profile-guided-optimization` | Emit PGO instrumentation, collect profiles, and feed them back into the optimization pipeline |
