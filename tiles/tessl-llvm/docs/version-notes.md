# LLVM 22 Version Notes — Breaking Changes & Migration Guide

Reference: [LLVM 22 Release Notes](https://releases.llvm.org/22.1.0/docs/ReleaseNotes.html) | Source: `llvmorg-22.1.2`

This page documents the most impactful API and behavioral changes when migrating to LLVM 22 from earlier versions. Read this before starting any migration — see the `version-sync` skill for the step-by-step workflow.

---

## LLVM 22 — High-impact changes

### 1. `Intrinsic::getDeclaration()` removed

`getDeclaration()` is gone. Two replacements:

```cpp
// Create or find an intrinsic declaration (most common — use this)
Function *F = Intrinsic::getOrInsertDeclaration(M, Intrinsic::memcpy,
                                                  {PtrTy, I64Ty});

// Look up only if it already exists — returns nullptr if not
Function *F = Intrinsic::getDeclarationIfExists(M, Intrinsic::memcpy);
```

---

### 2. DIBuilder: `DbgInstPtr` return type

`insertDeclare`, `insertDbgValueIntrinsic`, `insertDbgAssign`, and `insertLabel`
now return `DbgInstPtr` instead of `Instruction *`:

```cpp
// LLVM 22: DbgInstPtr = PointerUnion<Instruction *, DbgRecord *>
DbgInstPtr Result = DBuilder.insertDeclare(Alloca, DIVar, Expr, Loc, InsertPt);

// If you need to branch on which kind was returned:
if (auto *I = Result.dyn_cast<Instruction *>()) { /* ... */ }
if (auto *R = Result.dyn_cast<DbgRecord *>())   { /* ... */ }

// Most of the time you don't need to inspect the return — just ignore it
DBuilder.insertDeclare(Alloca, DIVar, DBuilder.createExpression(), Loc, InsertPt);
```

**Why DbgInstPtr?** LLVM 22 completes the **DbgRecord** migration: debug intrinsics
(`llvm.dbg.declare`, `llvm.dbg.value`) can now be represented as non-instruction
`DbgRecord` nodes attached to instructions, separate from the instruction stream.
This improves optimization accuracy for debug info.

---

### 3. New DIBuilder APIs

```cpp
// insertDeclareValue — separate from insertDeclare
DbgInstPtr DBuilder.insertDeclareValue(Value *Val, DILocalVariable *Var,
                                       DIExpression *Expr,
                                       const DILocation *DL,
                                       InsertPosition InsertPt);

// insertDbgAssign — assignment tracking (new in LLVM 22)
// Links a store instruction to its debug variable assignment
DbgInstPtr DBuilder.insertDbgAssign(Instruction *LinkedInstr, Value *Val,
                                    DILocalVariable *SrcVar,
                                    DIExpression *ValExpr, Value *Addr,
                                    DIExpression *AddrExpr,
                                    const DILocation *DL);

// createFunction: new UseKeyInstructions parameter (last, defaults false)
DISubprogram *DBuilder.createFunction(
    DIScope *Scope, StringRef Name, StringRef LinkageName, DIFile *File,
    unsigned LineNo, DISubroutineType *Ty, unsigned ScopeLine,
    DINode::DIFlags Flags = DINode::FlagZero,
    DISubprogram::DISPFlags SPFlags = DISubprogram::SPFlagZero,
    // ... existing params ...
    bool UseKeyInstructions = false   // NEW in LLVM 22
);
```

---

### 4. CreateGEP — GEPNoWrapFlags parameter

`CreateGEP` now accepts an optional `GEPNoWrapFlags` parameter:

```cpp
// LLVM 22 signature:
Value *CreateGEP(Type *Ty, Value *Ptr, ArrayRef<Value *> IdxList,
                 const Twine &Name = "",
                 GEPNoWrapFlags NW = GEPNoWrapFlags::none()); // NEW

// Flags:
GEPNoWrapFlags::none()     // default — no assumptions
GEPNoWrapFlags::inBounds() // same as CreateInBoundsGEP; UB if out of bounds
GEPNoWrapFlags::noUnsignedWrap()  // pointer arithmetic doesn't unsigned-overflow
GEPNoWrapFlags::noSignedWrap()    // pointer arithmetic doesn't signed-overflow

// CreateInBoundsGEP is still a convenience wrapper:
Value *CreateInBoundsGEP(Type *Ty, Value *Ptr, ArrayRef<Value *> IdxList,
                         const Twine &Name = "");
```

---

### 5. MCJIT removed — ORC JIT v2 only

`llvm/ExecutionEngine/MCJIT.h` and related classes are removed.
Use `LLJIT` or `LLLazyJIT` from `llvm/ExecutionEngine/Orc/LLJIT.h`.

Note: `LLLazyJIT` is declared in `LLJIT.h` — there is no separate `LLLazyJIT.h`.

```cpp
// LLVM 22:
#include "llvm/ExecutionEngine/Orc/LLJIT.h"  // contains both LLJIT and LLLazyJIT

auto JIT     = ExitOnErr(LLJITBuilder().create());
auto LazyJIT = ExitOnErr(LLLazyJITBuilder().create());
```

---

### 6. Legacy PassManager — still present, but do not use for new code

`llvm/IR/LegacyPassManager.h`, `legacy::PassManager`, `legacy::FunctionPassManager`,
`FunctionPass`, `ModulePass` base classes in `llvm/Pass.h` are **still present** in
LLVM 22 for backward compatibility and for `TargetMachine::addPassesToEmitFile`.

**Do not use them for new pass development.** All new passes must use NPM
(`PassInfoMixin<T>`, `run(IRUnit &, AnalysisManager &)`, `PreservedAnalyses`).

```cpp
// Still works in LLVM 22 (for codegen emission only):
legacy::PassManager PM;
TM->addPassesToEmitFile(PM, OS, nullptr, CodeGenFileType::ObjectFile);
PM.run(*M);

// New pass development — always use NPM:
class MyPass : public PassInfoMixin<MyPass> {
public:
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &FAM);
};
```

---

### 7. `getIndexTy()` — new IRBuilder method

```cpp
// Get the integer type appropriate for pointer indexing in a given address space
IntegerType *IdxTy = B.getIndexTy(M->getDataLayout(), /*AddrSpace=*/0);
// Equivalent to DL.getIndexType(B.getPtrTy(AddrSpace))
```

---

## LLVM 20/21 → 22 cumulative changes (still relevant)

### Opaque pointers (enforced since LLVM 17, now the only option)

```cpp
// All pointer construction:
Type *Ptr = B.getPtrTy();                    // ptr (addrspace 0)
Type *Ptr = PointerType::get(Ctx, AS);       // ptr addrspaceN

// LoadInst / StoreInst — explicit element type required:
B.CreateLoad(ElemTy, Ptr, "val");
B.CreateStore(Val, Ptr);

// GEP — explicit element type required:
B.CreateGEP(ElemTy, Ptr, Indices);
B.CreateInBoundsGEP(ElemTy, Ptr, Indices);
```

### `llvm::Optional` removed (since LLVM 17)

```cpp
// Before:
llvm::Optional<int> X = llvm::None;
// After:
std::optional<int> X = std::nullopt;
```

### Header moves (completed by LLVM 18, still required)

| Old path | New path |
|----------|----------|
| `llvm/ADT/Triple.h` | `llvm/TargetParser/Triple.h` |
| `llvm/Support/Host.h` | `llvm/TargetParser/Host.h` |
| `llvm/Support/TargetRegistry.h` | `llvm/MC/TargetRegistry.h` |

---

## Migration checklist (→ LLVM 22)

```
[ ] Intrinsic::getDeclaration()    → getOrInsertDeclaration() or getDeclarationIfExists()
[ ] DIBuilder insertDeclare()      → return type is now DbgInstPtr — update any Instruction* captures
[ ] DIBuilder createFunction()     → new UseKeyInstructions param (last, default false — safe to ignore)
[ ] MCJIT                          → LLJIT / LLLazyJIT from llvm/ExecutionEngine/Orc/LLJIT.h
[ ] LLLazyJIT.h include            → use LLJIT.h (LLLazyJIT is declared there)
[ ] Legacy PM for new passes       → replace with PassInfoMixin + NPM
[ ] Opaque pointers                → remove any getPointerElementType() calls
[ ] llvm::Optional                 → std::optional
[ ] llvm/ADT/Triple.h              → llvm/TargetParser/Triple.h
[ ] llvm/Support/Host.h            → llvm/TargetParser/Host.h
[ ] C++ standard                   → CMAKE_CXX_STANDARD 17 minimum
[ ] GEP inbounds                   → optionally adopt GEPNoWrapFlags for richer semantics
[ ] Rebuild and run all tests
```
