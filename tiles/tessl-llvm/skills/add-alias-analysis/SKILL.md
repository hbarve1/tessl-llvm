---
name: add-alias-analysis
description: Use and extend alias analysis in LLVM 22. Covers querying AAResults from a function pass, using IR-level hints (noalias, TBAA) as the preferred approach, writing a custom AA analysis with the New Pass Manager, and common ModRef patterns.
---

# Skill: Use and Extend Alias Analysis (LLVM 22)

Use this skill when you need to query whether two memory references may alias, push language-specific aliasing facts into the optimizer, or write a custom alias analysis pass.

---

## Step 0 — Choose the right approach

| Goal | Recommended approach |
|------|---------------------|
| Tell the optimizer your language's aliasing rules | IR-level hints: `noalias`, `nocapture`, TBAA metadata (no C++ needed) |
| Query alias info in an optimization pass | `AAResults` via the function analysis manager |
| Language-specific AA that IR hints can't express | Custom `AAResultBase` subclass (advanced) |

**Start with IR-level hints** — they are the highest-ROI approach and compose with all existing LLVM alias analyses automatically.

---

## Part 1: IR-level hints (preferred)

### 1a — `noalias` on parameters

Tell the optimizer that a pointer argument does not alias any other pointer visible to the function (like C `restrict`):

```cpp
llvm::AttrBuilder AB(Ctx);
AB.addAttribute(llvm::Attribute::NoAlias);
F->addParamAttrs(0, AB); // parameter 0 is noalias
```

In IR: `define void @foo(ptr noalias %p, ptr %q)`

### 1b — `nocapture` on parameters

Tell the optimizer that the pointer is not stored anywhere that outlives the call:

```cpp
llvm::AttrBuilder AB(Ctx);
AB.addAttribute(llvm::Attribute::NoCapture);
F->addParamAttrs(0, AB);
```

### 1c — TBAA metadata

Tag loads and stores with type information so the optimizer knows accesses to different language types can't alias:

```cpp
#include "llvm/IR/MDBuilder.h"

llvm::MDBuilder MDB(Ctx);
llvm::MDNode *Root    = MDB.createTBAARoot("MyLang TBAA");
llvm::MDNode *IntTy   = MDB.createTBAANode("int",    Root);
llvm::MDNode *PtrTy   = MDB.createTBAANode("object", Root);

llvm::MDNode *IntTag = MDB.createTBAAStructTagNode(IntTy, IntTy, 0);
llvm::MDNode *PtrTag = MDB.createTBAAStructTagNode(PtrTy, PtrTy, 0);

// Tag every load and store
LI->setMetadata(llvm::LLVMContext::MD_tbaa, IntTag);
SI->setMetadata(llvm::LLVMContext::MD_tbaa, PtrTag);
```

Cache the root/type nodes — recreating them each time creates distinct trees that don't interact.

---

## Part 2: Querying `AAResults` in a function pass

In an NPM function pass, obtain `AAResults` through the analysis manager:

```cpp
#include "llvm/Analysis/AliasAnalysis.h"
#include "llvm/Analysis/MemoryLocation.h"
#include "llvm/IR/PassManager.h"

struct MyOptPass : llvm::PassInfoMixin<MyOptPass> {
  llvm::PreservedAnalyses run(llvm::Function &F,
                              llvm::FunctionAnalysisManager &FAM) {
    auto &AA = FAM.getResult<llvm::AAManager>(F);

    // Compare two memory locations
    for (auto &I : llvm::instructions(F)) {
      if (auto *LI = llvm::dyn_cast<llvm::LoadInst>(&I)) {
        for (auto &J : llvm::instructions(F)) {
          if (auto *SI = llvm::dyn_cast<llvm::StoreInst>(&J)) {
            llvm::MemoryLocation LocL = llvm::MemoryLocation::get(LI);
            llvm::MemoryLocation LocS = llvm::MemoryLocation::get(SI);
            llvm::AliasResult AR = AA.alias(LocL, LocS);
            if (AR == llvm::AliasResult::NoAlias) {
              // Safe to reorder this load past this store
            }
          }
        }
      }
    }
    return llvm::PreservedAnalyses::all();
  }
};
```

### `AliasResult` values

| Value | Meaning |
|-------|---------|
| `NoAlias` | Cannot overlap for the accessed sizes |
| `MustAlias` | Always the same location |
| `MayAlias` | Conservative unknown (default) |
| `PartialAlias` | Overlapping but not identical |

---

## Part 3: ModRef — can a call read/write a location?

```cpp
// Does this call read or write through ptr %p?
llvm::MemoryLocation Loc = llvm::MemoryLocation::getBeforeOrAfter(Ptr);
llvm::ModRefInfo MRI = AA.getModRefInfo(CallSite, Loc);

if (llvm::isNoModRef(MRI)) {
  // The call neither reads nor writes through Ptr
  // Safe to hoist a load from Ptr above the call
}
if (llvm::isModSet(MRI)) {
  // The call may write through Ptr
}
if (llvm::isRefSet(MRI)) {
  // The call may read through Ptr
}
```

---

## Part 4: Custom AA analysis (advanced)

Only needed when IR hints cannot express your language's aliasing rules. The pattern:

```cpp
#include "llvm/Analysis/AliasAnalysis.h"
#include "llvm/IR/PassManager.h"

// Result type — implements the alias query
class MyAAResult : public llvm::AAResultBase {
public:
  explicit MyAAResult() : AAResultBase() {}

  llvm::AliasResult alias(const llvm::MemoryLocation &LocA,
                          const llvm::MemoryLocation &LocB,
                          llvm::AAQueryInfo &AAQI,
                          const llvm::Instruction *CtxI) {
    // If both pointers are known immutable globals, they don't alias heap
    if (isImmutableGlobal(LocA.Ptr) && isHeapPtr(LocB.Ptr))
      return llvm::AliasResult::NoAlias;
    // Delegate everything else to the chain
    return AAResultBase::alias(LocA, LocB, AAQI, CtxI);
  }

private:
  bool isImmutableGlobal(const llvm::Value *V) { /* ... */ return false; }
  bool isHeapPtr(const llvm::Value *V)         { /* ... */ return false; }
};

// Analysis wrapper (NPM pattern)
class MyAA : public llvm::AnalysisInfoMixin<MyAA> {
  friend llvm::AnalysisInfoMixin<MyAA>;
  static llvm::AnalysisKey Key;
public:
  using Result = MyAAResult;
  MyAAResult run(llvm::Function &, llvm::FunctionAnalysisManager &) {
    return MyAAResult();
  }
};
llvm::AnalysisKey MyAA::Key;
```

Register it in your `PassBuilder` callback:

```cpp
PB.registerFunctionAnalyses([](llvm::FunctionAnalysisManager &FAM) {
  FAM.registerPass([&] { return MyAA(); });
});

// Also register with AAManager so it participates in alias queries:
PB.registerParseAACallback([](llvm::StringRef Name,
                               llvm::AAManager &AAM) -> bool {
  if (Name == "my-aa") { AAM.registerFunctionAnalysis<MyAA>(); return true; }
  return false;
});
```

---

## Invalidation

When your pass modifies the IR in a way that affects aliasing, invalidate the AA results:

```cpp
// Return from run():
llvm::PreservedAnalyses PA;
PA.preserve<llvm::DominatorTreeAnalysis>(); // keep what you didn't break
// Not preserving AAManager causes it to be recomputed next time
return PA;
```

---

## Common mistakes

- **Never** cache `AAResults` across IR modifications — always re-query after changes.
- **Never** confuse pointer equality (`Value* ==`) with aliasing — different `GEP` values can `MustAlias`.
- **Never** ignore `PartialAlias` in range-sensitive reasoning.
- **Always** verify `ModRef` before hoisting instructions past calls.
- **Always** prefer IR-level hints (`noalias`, TBAA) over a custom AA pass — they are simpler and compose better.

---

## See also

- [attributes-metadata.md](../../docs/attributes-metadata.md) — TBAA, `noalias`, `nocapture` in detail
- [new-pass-manager.md](../../docs/new-pass-manager.md) — analysis manager, `FAM.getResult`, invalidation
