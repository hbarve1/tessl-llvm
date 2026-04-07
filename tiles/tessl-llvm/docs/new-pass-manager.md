# New Pass Manager (NPM) — Architecture & Usage (LLVM 22)

Reference: [WritingAnLLVMNewPMPass](https://llvm.org/docs/WritingAnLLVMNewPMPass.html) | [NewPassManager](https://llvm.org/docs/NewPassManager.html)

> In LLVM 22, the legacy `PassManager` types still exist for compatibility (e.g. `TargetMachine::addPassesToEmitFile`), but **all new passes must use NPM**. Never build new work on `FunctionPass` / `ModulePass` / `getAnalysis<>()`.

---

## Architecture overview

```
PassManager<Unit>          — ordered list of passes over IR unit
AnalysisManager<Unit>      — lazy cache of analysis results for IR unit
PassInstrumentation        — hooks for debugging / profiling passes
PassBuilder                — constructs standard pipelines from strings
```

**Four IR units, four manager pairs:**

| IR Unit | PassManager | AnalysisManager | Alias |
|---------|------------|----------------|-------|
| `Module` | `ModulePassManager` | `ModuleAnalysisManager` | MAM |
| `Function` | `FunctionPassManager` | `FunctionAnalysisManager` | FAM |
| `Loop` | `LoopPassManager` | `LoopAnalysisManager` | LAM |
| `LazyCallGraph::SCC` | `CGSCCPassManager` | `CGSCCAnalysisManager` | CGAM |

Passes at a finer granularity nest inside coarser ones via **adaptors**:
- `createModuleToFunctionPassAdaptor(FPM)` — runs FPM over every Function in a Module
- `createFunctionToLoopPassAdaptor(LPM)` — runs LPM over every Loop in a Function
- `createModuleToPostOrderCGSCCPassAdaptor(CGPM)` — runs CGPM over call graph SCCs

---

## Pass skeleton

### Transform pass (FunctionPass example)

**Header:**
```cpp
#include "llvm/IR/PassManager.h"

namespace llvm {
class MyTransformPass : public PassInfoMixin<MyTransformPass> {
public:
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &FAM);
  static bool isRequired() { return false; } // skip on optnone
};
} // namespace llvm
```

**Implementation:**
```cpp
PreservedAnalyses MyTransformPass::run(Function &F,
                                        FunctionAnalysisManager &FAM) {
  bool Changed = false;
  // ... modify F ...
  if (!Changed) return PreservedAnalyses::all();
  PreservedAnalyses PA;
  PA.preserveSet<CFGAnalyses>(); // CFG unchanged
  return PA;
}
```

### Analysis pass

```cpp
// Result struct — what the analysis computes
struct MyAnalysisResult {
  unsigned InstructionCount;
};

class MyAnalysis : public AnalysisInfoMixin<MyAnalysis> {
  friend AnalysisInfoMixin<MyAnalysis>;
  static AnalysisKey Key; // defined in .cpp: AnalysisKey MyAnalysis::Key;
public:
  using Result = MyAnalysisResult;
  Result run(Function &F, FunctionAnalysisManager &);
};
```

```cpp
// .cpp
AnalysisKey MyAnalysis::Key;

MyAnalysis::Result MyAnalysis::run(Function &F, FunctionAnalysisManager &) {
  unsigned Count = 0;
  for (auto &BB : F)
    Count += BB.size();
  return {Count};
}
```

**Consuming an analysis in a transform pass:**
```cpp
auto &Result = FAM.getResult<MyAnalysis>(F);
// Result.InstructionCount is now available
```

---

## PreservedAnalyses — what to return

```cpp
// Nothing modified
return PreservedAnalyses::all();

// Everything may have changed
return PreservedAnalyses::none();

// Only CFG structure preserved (dominators, loop info still valid)
PreservedAnalyses PA;
PA.preserveSet<CFGAnalyses>();
return PA;

// Specific analyses preserved
PA.preserve<DominatorTreeAnalysis>();
PA.preserve<LoopAnalysis>();
return PA;
```

**Rule:** Return the most conservative (most preserving) accurate answer.
Over-preserving causes stale analysis results; under-preserving causes
unnecessary recomputation.

---

## AnalysisManager API

```cpp
// Get a result — compute if not cached
auto &DT = FAM.getResult<DominatorTreeAnalysis>(F);

// Get a result only if already cached (returns nullptr if not)
auto *DTPtr = FAM.getCachedResult<DominatorTreeAnalysis>(F);

// Invalidate a function's analyses after modification
FAM.invalidate(F, PA);

// Register a custom analysis (outside of standard registration)
FAM.registerPass([&] { return MyAnalysis(); });
```

---

## PassBuilder — standard pipelines

```cpp
#include "llvm/Passes/PassBuilder.h"

PassBuilder PB;

// Create and cross-register all four analysis managers
LoopAnalysisManager     LAM;
FunctionAnalysisManager FAM;
CGSCCAnalysisManager    CGAM;
ModuleAnalysisManager   MAM;

PB.registerModuleAnalyses(MAM);
PB.registerCGSCCAnalyses(CGAM);
PB.registerFunctionAnalyses(FAM);
PB.registerLoopAnalyses(LAM);
PB.crossRegisterProxies(LAM, FAM, CGAM, MAM);

// Pre-built pipelines
ModulePassManager MPM;
MPM = PB.buildPerModuleDefaultPipeline(OptimizationLevel::O0); // -O0
MPM = PB.buildPerModuleDefaultPipeline(OptimizationLevel::O2); // -O2
MPM = PB.buildPerModuleDefaultPipeline(OptimizationLevel::O3); // -O3
MPM = PB.buildPerModuleDefaultPipeline(OptimizationLevel::Os); // -Os
MPM = PB.buildPerModuleDefaultPipeline(OptimizationLevel::Oz); // -Oz

// Parse a pipeline string (same as opt -passes=...)
auto MaybeErr = PB.parsePassPipeline(MPM, "function(instcombine,simplifycfg)");
if (MaybeErr) { /* handle llvm::Error */ }

// Run the pipeline
MPM.run(*M, MAM);
```

### Pipeline string syntax

```
module(pass1,pass2)           — module-level passes
function(pass1,pass2)         — run over all functions
cgscc(pass1)                  — run over call graph SCCs
loop(pass1)                   — run over loops

instcombine                   — instruction combining
simplifycfg                   — CFG simplification
mem2reg                       — promote allocas to SSA
gvn                           — global value numbering
sroa                          — scalar replacement of aggregates
inline                        — function inlining
loop-unroll<>                 — loop unrolling
loop-vectorize                — auto-vectorization
early-cse                     — early CSE
dce                           — dead code elimination
```

Verify available pass names:
```bash
opt --print-passes 2>&1 | less
```

---

## Adding a pass to a pipeline extension point

```cpp
// Register callback — called when PassBuilder builds O2+ pipelines
PB.registerPipelineEarlySimplificationEPCallback(
    [](ModulePassManager &MPM, OptimizationLevel Level) {
      if (Level != OptimizationLevel::O0)
        MPM.addPass(MyModulePass());
    });

// Other extension points:
// registerScalarOptimizerLateEPCallback  — after scalar opts
// registerVectorizerStartEPCallback      — before vectorizer
// registerOptimizerLastEPCallback        — very end of pipeline
// registerPipelineParsingCallback        — for custom pipeline string names
```

---

## Building pipelines manually

```cpp
// Function-level pipeline
FunctionPassManager FPM;
FPM.addPass(InstCombinePass());
FPM.addPass(SimplifyCFGPass());
FPM.addPass(MyTransformPass());

// Wrap in module pipeline
ModulePassManager MPM;
MPM.addPass(createModuleToFunctionPassAdaptor(std::move(FPM)));
MPM.run(*M, MAM);
```

---

## Loop passes

Loop passes receive additional context vs function passes:

```cpp
class MyLoopPass : public PassInfoMixin<MyLoopPass> {
public:
  PreservedAnalyses run(Loop &L, LoopAnalysisManager &LAM,
                        LoopStandardAnalysisResults &AR, LPMUpdater &U);
};
```

`LoopStandardAnalysisResults` provides cheap access to:
```cpp
AR.DT   // DominatorTree
AR.LI   // LoopInfo
AR.SE   // ScalarEvolution
AR.AC   // AssumptionCache
AR.TLI  // TargetLibraryInfo
```

Signal loop deletion to the updater:
```cpp
U.markLoopAsDeleted(L, L.getName());
return PreservedAnalyses::all(); // or appropriate PA
```

---

## CGSCC passes

```cpp
class MyCGSCCPass : public PassInfoMixin<MyCGSCCPass> {
public:
  PreservedAnalyses run(LazyCallGraph::SCC &C, CGSCCAnalysisManager &AM,
                        LazyCallGraph &CG, CGSCCUpdateResult &UR);
};
```

Useful for interprocedural transforms like inlining, argument promotion.

---

## Registering passes for `opt` (in-tree)

In `llvm/lib/Passes/PassRegistry.def`:
```
FUNCTION_PASS("my-pass", MyTransformPass())
MODULE_PASS("my-module-pass", MyModulePass())
LOOP_PASS("my-loop-pass", MyLoopPass())
CGSCC_PASS("my-cgscc-pass", MyCGSCCPass())
FUNCTION_ANALYSIS("my-analysis", MyAnalysis())
```

Add the header include to `llvm/lib/Passes/PassBuilder.cpp`.

For **out-of-tree plugins**, see the `add-npm-pass` skill.

---

## Instrumentation and debugging

```cpp
// Print IR before/after each pass (like -print-after-all)
PB.registerAfterPassCallback([](StringRef PassName, Any IR,
                                const PreservedAnalyses &PA) {
  errs() << "After " << PassName << ":\n";
});

// Verify IR after every pass (expensive — debug builds only)
// Add VerifierPass to the pipeline:
FPM.addPass(VerifierPass());
```

---

## Common mistakes

- **Do NOT** call `PM.add(createLegacyPass())` — legacy PM is gone.
- **Do NOT** hold references to analysis results across calls that may invalidate them — re-query with `FAM.getResult<>()`.
- **Do NOT** forget `crossRegisterProxies` — without it, analyses can't cross IR-unit boundaries (e.g., a function pass can't query module analyses).
- **Do NOT** run `MPM.run()` before `crossRegisterProxies()`.
- **ALWAYS** return `PreservedAnalyses::all()` for no-op passes — returning `none()` forces expensive re-computation.
