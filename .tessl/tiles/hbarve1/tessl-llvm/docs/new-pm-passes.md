# New Pass Manager — passes (22.1.x)

LLVM’s **New Pass Manager (NPM)** is the default path for `opt`, many Clang pipelines, and modern in-tree code.

## Mental model

- **Analysis passes** — Read IR and expose results via `AnalysisManager` (e.g. `DominatorTree`, `LoopInfo`).
- **Transform passes** — Mutate IR; declare `PreservedAnalyses` to skip invalidating analyses when safe.
- **Adaptor** — Pipelines connect module/function/adaptor passes (`PassBuilder`).

## Adding a function pass (typical pattern)

1. Subclass **`llvm::PassInfoMixin<YourPass>`** (or legacy `Pass` only if matching surrounding codebase).
2. Implement `PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM)`.
3. Register with **`PassInstrumentationCallbacks`** / plugin APIs if out-of-tree; in-tree, use `PassRegistry.def` / `PassBuilder` bindings consistent with neighbors in `llvm/lib`.

## Pipeline construction

- **`PassBuilder`** builds default optimization pipelines (`buildPerModuleDefaultPipeline`, `buildFunctionSimplificationPipeline`, etc.).
- Custom pipelines compose **function** and **module** adaptors explicitly.

## References

- [Writing an LLVM Pass](https://llvm.org/docs/WritingAnLLVMPass.html) — verify examples against 22.x (NPM sections).
- [New Pass Manager](https://llvm.org/docs/NewPassManager.html).

## Pitfall

- Copying **legacy** `FunctionPass` / `getAnalysis<>()` examples without converting to **`FunctionAnalysisManager`** patterns will diverge from current LLVM tree practice.
