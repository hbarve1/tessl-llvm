---
name: tessl-llvm
description: Add or extend a New Pass Manager LLVM function pass on LLVM 22.1.x (in-tree or out-of-tree), aligned with LangRef and tree conventions.
---

# LLVM 22 — New PM function pass workflow

Use this skill when the user wants a **new or updated optimization/analysis-style function pass** targeting **LLVM 22.1.x**. Consult tile docs on demand: [IR](../../docs/ir-and-types.md), [passes](../../docs/new-pm-passes.md), [out-of-tree](../../docs/out-of-tree.md).

## Before writing code

1. Confirm **LLVM version** — Require 22.1.x sources or install; if the tree is older/newer, say so and either align the answer to their tree or recommend checking out `llvmorg-22.1.2`.
2. Locate **similar passes** in the same codebase (`llvm/lib/Transforms/*` in-tree, or neighbor files out-of-tree). Match naming, analysis usage, and registration style.
3. Decide **analysis needs** — `DominatorTree`, `LoopInfo`, scalar evolution, etc.; plan `AnalysisManager` getters in `run()`.

## Implementation steps

1. **Define the pass** as `llvm::PassInfoMixin<YourPass>` with `PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM)`.
2. **Use analyses** via `AM.getResult<SomeAnalysis>(F)`; preserve analyses you do not invalidate in the returned `PreservedAnalyses`.
3. **IR changes** — Prefer LLVM helpers (`IRBuilder`, `replaceAllUsesWith`, `eraseFromParent`) over manual opcode surgery; respect SSA when inserting in mid-block (split blocks / phis as needed).
4. **Registration** — In-tree: wire through `PassBuilder` / `PassRegistry` the same way sibling passes do. Out-of-tree: follow plugin or static registration used by the project’s `opt` driver or custom tool.

## Verification

1. **Build** the target that includes the pass; fix compile errors against **actual** 22.x headers.
2. **Run** `opt` (or the project’s driver) on a small `.ll` fixture; dump IR before/after if useful (`-print-after-all` sparingly).
3. **Tests** — In-tree, add `llvm/test/Transforms/...` FileCheck tests; out-of-tree, add the project’s usual regression test style.

## Stop conditions

- If the request mixes **legacy PM** only APIs, flag it and outline NPM migration or match the repo if it is intentionally legacy.
- If the user needs **target/backend** changes, pivot to [TableGen](../../docs/tablegen.md) and backend code paths—do not fake instruction patterns.
