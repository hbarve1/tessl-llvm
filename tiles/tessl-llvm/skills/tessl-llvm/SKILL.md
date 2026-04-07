---
name: add-npm-pass
description: Add a New Pass Manager (NPM) pass to an LLVM 20 in-tree or out-of-tree project. Covers FunctionPass, ModulePass, LoopPass, and CGSCCPass kinds, registration, pipeline wiring, CMake, and lit testing.
---

# Skill: Add a New Pass Manager (NPM) Pass (LLVM 20)

Use this skill whenever the user wants to create a new optimization or analysis pass using the LLVM New Pass Manager (NPM). All passes in LLVM 20 use NPM — the legacy PassManager is removed.

---

## Step 0 — Decide pass kind

Ask or infer the pass kind from the user's description:

| Kind | Unit of IR | Base class | Use when |
|------|-----------|------------|----------|
| Function | `llvm::Function` | Inherits nothing; `run(Function &, FunctionAnalysisManager &)` | Most transforms and analyses |
| Module | `llvm::Module` | `run(Module &, ModuleAnalysisManager &)` | Whole-program / cross-function |
| Loop | `llvm::Loop` | `run(Loop &, LoopAnalysisManager &, LoopStandardAnalysisResults &, LPMUpdater &)` | Loop transforms |
| CGSCC | `llvm::LazyCallGraph::SCC` | `run(LazyCallGraph::SCC &, CGSCCAnalysisManager &, LazyCallGraph &, CGSCCUpdateResult &)` | Interprocedural over SCCs |

For a **transform pass** use `PreservedAnalyses` as the return type.
For an **analysis pass** inherit from `AnalysisInfoMixin<YourPass>` and return a result struct.

---

## Step 1 — Create the header

**Path (in-tree):** `include/llvm/Transforms/<Category>/<PassName>.h`
**Path (out-of-tree):** `include/<PassName>.h`

```cpp
//===- <PassName>.h - <One-line description> --------------------*- C++ -*-===//
// LLVM 20 New Pass Manager pass
//===----------------------------------------------------------------------===//
#ifndef LLVM_TRANSFORMS_<CATEGORY>_<PASSNAME>_H
#define LLVM_TRANSFORMS_<CATEGORY>_<PASSNAME>_H

#include "llvm/IR/PassManager.h"

namespace llvm {

class <PassName> : public PassInfoMixin<<PassName>> {
public:
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &FAM);

  // Required for opt to skip this pass on optnone functions.
  static bool isRequired() { return false; }
};

} // namespace llvm

#endif // LLVM_TRANSFORMS_<CATEGORY>_<PASSNAME>_H
```

Replace `Function` / `FunctionAnalysisManager` with the appropriate IR unit from Step 0.

---

## Step 2 — Implement the pass

**Path (in-tree):** `lib/Transforms/<Category>/<PassName>.cpp`
**Path (out-of-tree):** `lib/<PassName>.cpp` or `src/<PassName>.cpp`

```cpp
//===- <PassName>.cpp - <One-line description> ----------------------------===//
// LLVM 20 New Pass Manager pass implementation
//===----------------------------------------------------------------------===//
#include "llvm/Transforms/<Category>/<PassName>.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/Instructions.h"
#include "llvm/Support/raw_ostream.h"

using namespace llvm;

PreservedAnalyses <PassName>::run(Function &F, FunctionAnalysisManager &FAM) {
  // TODO: implement pass logic
  // Return PA indicating which analyses are preserved.
  // If nothing is modified:
  return PreservedAnalyses::all();
  // If everything may be modified:
  // return PreservedAnalyses::none();
  // If only the CFG is preserved:
  // PA.preserveSet<CFGAnalyses>(); return PA;
}
```

Key LLVM 20 APIs to know inside `run()`:
- `FAM.getResult<DominatorTreeAnalysis>(F)` — get a cached analysis result.
- `FAM.invalidate(F, PA)` — explicitly invalidate analyses after modification.
- `llvm::make_range(F.begin(), F.end())` — iterate basic blocks.
- `llvm::PatternMatch` — match IR patterns without manual casting.

---

## Step 3 — Register the pass for `opt` pipeline strings (in-tree only)

### 3a. `PassRegistry.def`

Add one line in the appropriate section of `llvm/lib/Passes/PassRegistry.def`:

```
FUNCTION_PASS("<pass-name>", <PassName>())
```

Use `MODULE_PASS`, `LOOP_PASS`, or `CGSCC_PASS` for other kinds.

### 3b. `PassBuilder.cpp` includes

Add the header include near the other `Transforms/<Category>/` includes in
`llvm/lib/Passes/PassBuilder.cpp`:

```cpp
#include "llvm/Transforms/<Category>/<PassName>.h"
```

No other changes needed — `PassRegistry.def` drives the registration.

### Out-of-tree registration

If out-of-tree, register the pass with a `PassPluginLibraryInfo` entry point so
`opt --load-pass-plugin` can find it:

```cpp
#include "llvm/Passes/PassPlugin.h"

llvm::PassPluginLibraryInfo get<PassName>PluginInfo() {
  return {LLVM_PLUGIN_API_VERSION, "<pass-name>", LLVM_VERSION_STRING,
          [](PassBuilder &PB) {
            PB.registerPipelineParsingCallback(
                [](StringRef Name, FunctionPassManager &FPM,
                   ArrayRef<PassBuilder::PipelineElement>) {
                  if (Name == "<pass-name>") {
                    FPM.addPass(<PassName>());
                    return true;
                  }
                  return false;
                });
          }};
}

extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
  return get<PassName>PluginInfo();
}
```

---

## Step 4 — CMake

### In-tree

In `lib/Transforms/<Category>/CMakeLists.txt`, add the source file:

```cmake
add_llvm_component_library(LLVM<Category>
  ...
  <PassName>.cpp
  ...
)
```

### Out-of-tree (plugin)

```cmake
find_package(LLVM 22 REQUIRED CONFIG)

add_llvm_pass_plugin(<PassName>
  <PassName>.cpp
  DEPENDS
  intrinsics_gen
)
```

If `add_llvm_pass_plugin` is not available in your LLVM install, use:

```cmake
add_library(<PassName> MODULE <PassName>.cpp)
target_include_directories(<PassName> PRIVATE ${LLVM_INCLUDE_DIRS})
target_compile_definitions(<PassName> PRIVATE ${LLVM_DEFINITIONS})
set_target_properties(<PassName> PROPERTIES
  CXX_STANDARD 17
  POSITION_INDEPENDENT_CODE ON
)
```

---

## Step 5 — Write a lit test

**Path (in-tree):** `test/Transforms/<Category>/<pass-name>.ll`
**Path (out-of-tree):** `test/<pass-name>.ll`

```llvm
; RUN: opt -passes=<pass-name> -S %s | FileCheck %s
; RUN: opt -passes=<pass-name> -disable-output %s   ; smoke test — must not crash

define i32 @test_basic(i32 %x) {
entry:
  ; CHECK-LABEL: @test_basic
  ; CHECK: <expected IR pattern after pass>
  ret i32 %x
}
```

For an out-of-tree plugin:

```llvm
; RUN: opt --load-pass-plugin=%llvmshlibdir/<PassName>%pluginext \
; RUN:     -passes=<pass-name> -S %s | FileCheck %s
```

Run tests:
```bash
# In-tree
llvm-lit test/Transforms/<Category>/<pass-name>.ll

# Out-of-tree (after cmake --build)
llvm-lit test/<pass-name>.ll
```

---

## Step 6 — Smoke test manually

```bash
# In-tree
echo 'define void @f() { ret void }' | \
  opt -passes=<pass-name> -S -

# Out-of-tree plugin
echo 'define void @f() { ret void }' | \
  opt --load-pass-plugin=./<PassName>.so -passes=<pass-name> -S -
```

Expected: IR printed without crash, with any expected transforms applied.

---

## Quick reference — LLVM 20 NPM cheat sheet

```
PreservedAnalyses::all()         — no IR changed
PreservedAnalyses::none()        — anything may have changed
PA.preserve<DominatorTreeAnalysis>() — CFG unchanged, DT still valid
PA.preserveSet<CFGAnalyses>()    — all CFG-only analyses preserved

FAM.getResult<X>(F)              — get (cached) analysis X for function F
FAM.getCachedResult<X>(F)        — get only if already computed (nullable)
FAM.invalidate(F, PA)            — tell manager what F's pass invalidated

FunctionPassManager FPM;
FPM.addPass(MyPass());           — build a pass pipeline manually
ModulePassManager MPM;
MPM.addPass(createModuleToFunctionPassAdaptor(std::move(FPM)));
```

---

## Common mistakes to avoid

- **Do NOT** use `legacy::PassManager`, `FunctionPass`, `ModulePass` base classes — these are removed in LLVM 20.
- **Do NOT** call `AU.setPreservesAll()` or `getAnalysis<>()` — NPM uses `FAM.getResult<>()`.
- **Do NOT** forget `PassInfoMixin<YourPass>` — required for pass identification.
- **Do NOT** hold raw pointers to analysis results across IR mutations — re-query after any modification.
- **Do NOT** use `using namespace llvm;` in headers.
