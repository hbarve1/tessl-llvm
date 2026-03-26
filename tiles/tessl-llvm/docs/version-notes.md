# LLVM 20 Version Notes — Breaking Changes & Migration Guide

Reference: [LLVM 20 Release Notes](https://releases.llvm.org/20.0.0/docs/ReleaseNotes.html)

This page documents the most impactful API and behavioral changes when migrating to LLVM 20 from earlier versions (17, 18, 19). Read this before starting any migration — see the `version-sync` skill for the step-by-step workflow.

---

## LLVM 20 — High-impact changes

### 1. Legacy PassManager removed

The `llvm/IR/LegacyPassManager.h` infrastructure (`legacy::PassManager`, `FunctionPass`, `ModulePass`, `getAnalysis<>()`, `PassManagerBuilder`) is **completely removed**.

**Migrate to New Pass Manager (NPM):**

| Legacy | NPM |
|--------|-----|
| `#include "llvm/IR/LegacyPassManager.h"` | `#include "llvm/Passes/PassBuilder.h"` |
| `legacy::PassManager PM` | `ModulePassManager MPM` |
| `PM.add(createMyPass())` | `MPM.addPass(MyPass())` |
| `PM.run(M)` | `MPM.run(M, MAM)` |
| `class MyPass : public FunctionPass` | `class MyPass : public PassInfoMixin<MyPass>` |
| `void runOnFunction(Function &F)` | `PreservedAnalyses run(Function &, FunctionAnalysisManager &)` |
| `getAnalysis<DominatorTreeWrapperPass>().getDomTree()` | `FAM.getResult<DominatorTreeAnalysis>(F)` |
| `AU.setPreservesAll()` | `return PreservedAnalyses::all()` |

> Exception: `TargetMachine::addPassesToEmitFile` still uses an internal legacy PM for MC emission. This API remains in LLVM 20.

---

### 2. Opaque pointers fully enforced

Typed pointers (`i8*`, `i32**`, etc.) are removed. All pointers are `ptr`.

**Source changes required:**

```cpp
// LLVM 19 and earlier:
Type *I8Ptr = Type::getInt8PtrTy(Ctx);
PointerType *PT = PointerType::get(ElemTy, AddrSpace);
auto *ElemTy = GEP->getPointerElementType();  // removed

// LLVM 20:
Type *Ptr = PointerType::get(Ctx, 0);         // ptr addrspace(0)
Type *Ptr = Builder.getPtrTy();               // same, via IRBuilder
// No getPointerElementType() — track element type in your own data structures
```

**LoadInst / StoreInst construction:**
```cpp
// LLVM 19: type inferred from pointer
new LoadInst(Ptr, "val", InsertBefore);       // REMOVED

// LLVM 20: explicit element type required
new LoadInst(I32Ty, Ptr, "val", InsertBefore);
Builder.CreateLoad(I32Ty, Ptr, "val");
```

**GEP construction:**
```cpp
// LLVM 20: always provide explicit element type
Builder.CreateGEP(ElemTy, Ptr, Indices);
Builder.CreateInBoundsGEP(ElemTy, Ptr, Indices);
```

---

### 3. `llvm::Optional` removed

Replaced by `std::optional` (C++17 standard library).

```cpp
// LLVM 17-19 (deprecated from 17, removed in 20):
#include "llvm/ADT/Optional.h"
llvm::Optional<int> X = llvm::None;
if (X.hasValue()) { ... X.getValue() ... }

// LLVM 20:
std::optional<int> X = std::nullopt;
if (X.has_value()) { ... X.value() ... }
// or: if (X) { ... *X ... }
```

Also removed: `llvm::None` → `std::nullopt`.

---

### 4. Header path changes

| Old path | New path (LLVM 20) |
|----------|-------------------|
| `llvm/ADT/Optional.h` | Removed — use `<optional>` |
| `llvm/ADT/None.h` | Removed — use `<optional>` |
| `llvm/ADT/Triple.h` | `llvm/TargetParser/Triple.h` |
| `llvm/Support/Host.h` | `llvm/TargetParser/Host.h` |
| `llvm/Support/TargetRegistry.h` | `llvm/MC/TargetRegistry.h` |
| `llvm/Support/TargetSelect.h` | Unchanged |
| `llvm/IR/LegacyPassManager.h` | Removed |
| `llvm/Transforms/IPO/PassManagerBuilder.h` | Removed |

---

### 5. `Intrinsic::getDeclaration()` deprecated → `getOrInsertDeclaration()`

```cpp
// Deprecated (LLVM 20 warns, future versions will remove):
Function *F = Intrinsic::getDeclaration(M, Intrinsic::memcpy, {PtrTy, I64Ty});

// LLVM 20 preferred:
Function *F = Intrinsic::getOrInsertDeclaration(M, Intrinsic::memcpy,
                                                  {PtrTy, I64Ty});
```

---

### 6. C++17 required

LLVM 20 itself requires C++17. Your out-of-tree project must also build with C++17:

```cmake
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
```

C++14 compatibility shims in your code (e.g., manual `std::make_unique` reimplementations, `llvm::make_unique`) should be replaced with standard C++17 equivalents.

---

## LLVM 19 → 20 specific changes

| Area | Change |
|------|--------|
| Legacy PM | Fully removed (was deprecated since LLVM 17) |
| Typed pointers | Fully removed (opaque pointers enforced since LLVM 17) |
| `llvm::Optional` | Fully removed (deprecated since LLVM 16) |
| `TargetRegistry` header | Moved to `llvm/MC/TargetRegistry.h` (done in LLVM 14, enforced now) |
| `PassManagerBuilder` | Removed — use `PassBuilder` |
| `createXYZPass()` factory functions | Most removed — use pass constructors directly: `XYZPass()` |

---

## LLVM 18 → 19 changes (still relevant if migrating from 18)

| Area | Change |
|------|--------|
| `AttributeList::get()` | Signature changed — `AttrBuilder` no longer takes `LLVMContext` in constructor |
| `TargetLibraryInfo::has()` | Replaced by `getLibFunc()` pattern |
| `MemorySSA` | Some analysis invalidation rules changed |
| `DIBuilder` | Minor API cleanups |
| `SelectionDAG::getSetCC()` | Signature adjusted |

---

## LLVM 17 → 18 changes (still relevant if migrating from 17)

| Area | Change |
|------|--------|
| `Triple.h` | Moved to `llvm/TargetParser/Triple.h` |
| `Host.h` | Moved to `llvm/TargetParser/Host.h` |
| `TargetParser/` | New directory — many target parser headers moved here |
| Opaque pointers | `getPointerElementType()` removed in 18 |
| `llvm::Optional` | Deprecated — migration to `std::optional` expected |

---

## Checking version at compile time

```cpp
#include "llvm/Config/llvm-config.h"

#if LLVM_VERSION_MAJOR >= 20
  // LLVM 20+ API
  auto *F = Intrinsic::getOrInsertDeclaration(M, ID, Types);
#elif LLVM_VERSION_MAJOR >= 17
  // LLVM 17-19 API
  auto *F = Intrinsic::getDeclaration(M, ID, Types);
#endif
```

For LLVM 20-only codebases: **remove all version guards** for < 20 and use the current API directly. Guards add maintenance cost and are only warranted for libraries that must support multiple LLVM versions simultaneously.

---

## Quick migration checklist

```
[ ] Legacy PM: replace with NPM (PassInfoMixin, FunctionAnalysisManager, etc.)
[ ] Typed pointers: audit all PointerType::get(ElemTy, ...) calls
[ ] LoadInst/StoreInst: add explicit element type argument
[ ] GEP: add explicit element type argument
[ ] llvm::Optional → std::optional
[ ] llvm::None → std::nullopt
[ ] #include "llvm/ADT/Optional.h" → remove (use <optional>)
[ ] #include "llvm/ADT/Triple.h" → "llvm/TargetParser/Triple.h"
[ ] #include "llvm/Support/Host.h" → "llvm/TargetParser/Host.h"
[ ] Intrinsic::getDeclaration → getOrInsertDeclaration
[ ] createXYZPass() → XYZPass() constructors
[ ] PassManagerBuilder → PassBuilder
[ ] C++17: set CMAKE_CXX_STANDARD 17
[ ] LLVM_VERSION_MAJOR guards: remove < 20 branches
[ ] Rebuild intrinsics_gen if any .td files changed
[ ] Run full test suite after migration
```
