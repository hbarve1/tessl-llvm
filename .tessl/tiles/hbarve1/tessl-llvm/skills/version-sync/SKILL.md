---
name: version-sync
description: Migrate an out-of-tree LLVM project to a new LLVM version (e.g., LLVM 19 → 20). Covers CMake bump, common LLVM 20 API breakages, header moves, pass renames, and opaque pointer migration.
---

# Skill: Sync Out-of-Tree Project to a New LLVM Version (→ LLVM 20)

Use this skill when the user wants to upgrade an existing out-of-tree LLVM project to LLVM 20 from an older version (17, 18, or 19). This is a systematic, checklist-driven process.

---

## Step 0 — Snapshot and branch

```bash
git checkout -b llvm-20-migration
```

Never migrate on `main` — the process produces many intermediate build failures.

---

## Step 1 — Bump CMake version requirement

In `CMakeLists.txt`:

```cmake
# Before:
find_package(LLVM 19 REQUIRED CONFIG)

# After:
find_package(LLVM 20 REQUIRED CONFIG)
```

Point `LLVM_DIR` or `CMAKE_PREFIX_PATH` at your LLVM 20 install:

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Debug \
  -DLLVM_DIR=$(llvm-config-20 --cmakedir)
```

---

## Step 2 — Collect all build errors

```bash
cmake --build build -j$(nproc) 2>&1 | tee build-errors.txt
```

Work through errors category by category using the reference below.

---

## Step 3 — Fix: Opaque pointers (LLVM 15+, enforced in 17+, fully removed in 20)

LLVM 20 has **no typed pointer types**. All pointers are `ptr`.

| Old (typed) | New (opaque) |
|-------------|--------------|
| `Type::getInt8PtrTy(Ctx)` | `PointerType::get(Ctx, 0)` or `Builder.getPtrTy()` |
| `PointerType::get(ElemTy, AS)` | `PointerType::get(Ctx, AS)` |
| `GEP->getPointerElementType()` | Requires explicit element type at construction site |
| `CallInst::getFunctionType()` | Still works — CallInst stores its own FunctionType |
| `LoadInst(PtrTy, Ptr, ...)` | Must pass element type explicitly: `LoadInst(ElemTy, Ptr, ...)` |

**Migration rule:** Track the pointee type alongside your `Value *` pointers.
Wherever your code inferred type from a pointer, store the type separately.

---

## Step 4 — Fix: Removed legacy Pass Manager APIs

The legacy `PassManager` is **removed** in LLVM 20.

| Removed (legacy PM) | Replacement (NPM) |
|--------------------|-------------------|
| `#include "llvm/IR/LegacyPassManager.h"` | `#include "llvm/Passes/PassBuilder.h"` |
| `legacy::PassManager PM` | `ModulePassManager MPM` |
| `PM.add(createMyPass())` | `MPM.addPass(MyPass())` |
| `PM.run(M)` | `MPM.run(M, MAM)` |
| `FunctionPass`, `ModulePass` base classes | `PassInfoMixin<T>` + `run()` method |
| `AU.setPreservesAll()` | Return `PreservedAnalyses::all()` |
| `getAnalysis<X>()` | `FAM.getResult<X>(F)` |
| `PassManagerBuilder` | `PassBuilder::buildPerModuleDefaultPipeline()` |

See the **add-npm-pass** skill for full NPM pass structure.

---

## Step 5 — Fix: Renamed and moved APIs (LLVM 17 → 20 cumulative)

### Headers moved

| Old path | New path |
|----------|----------|
| `llvm/ADT/Optional.h` | Removed — use `std::optional` |
| `llvm/ADT/None.h` | Removed — use `std::nullopt` |
| `llvm/ADT/Triple.h` | `llvm/TargetParser/Triple.h` |
| `llvm/Support/Host.h` | `llvm/TargetParser/Host.h` |
| `llvm/Support/TypeSize.h` | `llvm/Support/TypeSize.h` (unchanged) |

### Type/API renames

| Old | New | Version |
|-----|-----|---------|
| `llvm::Optional<T>` | `std::optional<T>` | 17+ |
| `llvm::None` | `std::nullopt` | 17+ |
| `llvm::MaybeAlign` | `MaybeAlign` (still in llvm ns) | — |
| `AttributeList::get(Ctx, AS, Attrs)` | signature changed; check headers | 18+ |
| `Intrinsic::getDeclaration()` | `Intrinsic::getOrInsertDeclaration()` | 20 |
| `DIBuilder::finalize()` | Still works | — |
| `TargetLibraryInfo::has()` | `TargetLibraryInfo::getLibFunc()` | 18+ |

### Pass name changes (pipeline strings)

| Old pipeline string | New (LLVM 20) |
|--------------------|--------------|
| `loop-unroll` | `loop-unroll<>` (parameterized) |
| `scalar-evolution` | `scalar-evolution` (unchanged) |
| `instcombine` | `instcombine<>` |
| `simplifycfg` | `simplifycfg<>` |

Always verify pass names with:
```bash
opt --print-passes 2>&1 | grep <name>
```

---

## Step 6 — Fix: TableGen changes (if applicable)

- `Intrinsics*.td`: `IntrinsicProperty` list format unchanged in LLVM 20; verify your custom records compile with `llvm-tblgen`.
- Register class names: check `<Target>RegisterInfo.td` for any upstream renames.
- `add_tablegen()` CMake macro: still present in LLVM 20.

---

## Step 7 — Fix: Target / CodeGen API changes

| Old | New / Notes |
|-----|-------------|
| `TargetMachine::addPassesToEmitFile(PM, ...)` with legacy PM | Use `TargetMachine::addPassesToEmitFile` with `legacy::PassManager` OR switch to `LLVMTargetMachineEmitToMemoryBuffer` C API |
| `MachineFunction::getProperties()` | API stable |
| `GlobalISel::IRTranslator` | Stable; confirm pass name in NPM pipeline |

---

## Step 8 — Fix: LLVM 20 C++ standard bump

LLVM 20 requires **C++17** at minimum. Ensure your CMake has:

```cmake
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
```

Remove any C++14 compatibility shims.

---

## Step 9 — Run tests

```bash
# Run your project's lit tests
llvm-lit test/ -v

# Run LLVM's own tests for any in-tree patches
llvm-lit llvm/test/ -v --filter <YourArea>
```

Fix any test failures caused by changed IR output (e.g., new pass pipeline default changes, attribute printing changes).

---

## Step 10 — Update version guards

Search for `LLVM_VERSION_MAJOR` guards and update:

```bash
grep -r "LLVM_VERSION_MAJOR" src/ include/
```

```cpp
// Before (supporting LLVM 19 and 20):
#if LLVM_VERSION_MAJOR >= 19
  // new API
#else
  // old API
#endif

// After (LLVM 20 only — remove guards):
// new API  // no guard needed
```

---

## Step 11 — Update `README` / docs

Note the new minimum LLVM version and any user-visible behavior changes.

---

## Checklist summary

```
[ ] CMakeLists.txt: bump find_package(LLVM 20 ...)
[ ] Opaque pointers: remove all typed pointer construction
[ ] Legacy PM: migrate to NPM (PassInfoMixin, FAM, PreservedAnalyses)
[ ] llvm::Optional → std::optional, llvm::None → std::nullopt
[ ] Triple.h / Host.h: update include paths
[ ] Intrinsic::getDeclaration → getOrInsertDeclaration
[ ] C++ standard: ensure CMAKE_CXX_STANDARD 17
[ ] Pass pipeline strings: verify with opt --print-passes
[ ] Version guards: remove LLVM_VERSION_MAJOR < 20 branches
[ ] Rebuild and run all tests
[ ] Commit on migration branch, open PR
```
