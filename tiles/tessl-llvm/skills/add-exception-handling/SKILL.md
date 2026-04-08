---
name: add-exception-handling
description: Add exception handling (try/catch/finally) to an LLVM 22 IR frontend using invoke, landingpad, and the Itanium C++ ABI. Covers personality function, invoke vs call, catch dispatch, __cxa_begin/end_catch, resume, cleanup blocks, and Windows SEH notes.
---

# Skill: Add Exception Handling to an LLVM Frontend (LLVM 22)

Use this skill when your language needs try/catch/finally semantics and you need to emit correct LLVM IR unwind tables.

---

## Step 0 — Prerequisites

- Existing `Module`, `IRBuilder<>`, and functions already emitting IR.
- Target must support native unwind tables (x86-64 Linux/macOS = Itanium ABI; Windows = SEH model).
- No extra CMake components needed — EH support is part of `LLVMCore`.

---

## Step 1 — Declare the personality function

Every function that contains a landing pad must have a personality function attached.

```cpp
#include "llvm/IR/Function.h"
#include "llvm/IR/IntrinsicsX86.h"

// Declare __gxx_personality_v0 (Itanium ABI — Linux/macOS)
llvm::FunctionType *PersonalityFT =
    llvm::FunctionType::get(B.getInt32Ty(), /*vararg=*/true);
llvm::Function *Personality =
    llvm::Function::Create(PersonalityFT,
                           llvm::Function::ExternalLinkage,
                           "__gxx_personality_v0", *M);

// Attach to every function that has landing pads
F->setPersonalityFn(Personality);
```

---

## Step 2 — Replace `call` with `invoke` inside try blocks

Any call that may throw must use `invoke`, not `call`:

```cpp
llvm::BasicBlock *NormalBB = llvm::BasicBlock::Create(Ctx, "normal", F);
llvm::BasicBlock *LpadBB   = llvm::BasicBlock::Create(Ctx, "lpad",   F);

llvm::InvokeInst *II = B.CreateInvoke(
    CalleeFT,       // function type (FunctionType*)
    Callee,         // callee Value*
    NormalBB,       // branch on normal return
    LpadBB,         // branch on unwind
    {Arg0, Arg1},   // call arguments
    "result"        // result name (omit for void)
);

// Normal path: result is available here
B.SetInsertPoint(NormalBB);
// ... use II as the return value ...
```

---

## Step 3 — Emit the landing pad

The landing pad must be the **first instruction** in the unwind destination block:

```cpp
B.SetInsertPoint(LpadBB);

// landingpad returns { ptr, i32 }
//   ptr — exception object pointer
//   i32 — selector (which clause matched)
llvm::Type *LPadTy = llvm::StructType::get(Ctx,
    {B.getPtrTy(), B.getInt32Ty()});
llvm::LandingPadInst *LP = B.CreateLandingPad(LPadTy, /*numClauses=*/1);

// Add a catch clause for a specific type (typeinfo global pointer)
llvm::Constant *IntTypeInfo =
    M->getOrInsertGlobal("_ZTIi", B.getPtrTy()); // typeinfo for int
LP->addClause(IntTypeInfo);

// OR: catch-all (like catch(...))
// LP->addClause(llvm::ConstantPointerNull::get(B.getPtrTy()));

// OR: cleanup — always run (destructors, finally)
// LP->setCleanup(true);

// Extract the two components
llvm::Value *ExnPtr = B.CreateExtractValue(LP, 0, "exn.ptr");
llvm::Value *SelVal = B.CreateExtractValue(LP, 1, "exn.sel");
```

---

## Step 4 — Dispatch to the correct catch handler

Use `llvm.eh.typeid.for` to get the expected selector value for a type:

```cpp
#include "llvm/IR/Intrinsics.h"

llvm::Function *TypeIDFn = llvm::Intrinsic::getOrInsertDeclaration(
    M, llvm::Intrinsic::eh_typeid_for, {B.getPtrTy()});

llvm::Value *ExpectedID = B.CreateCall(TypeIDFn, {IntTypeInfo});
llvm::Value *Matches    = B.CreateICmpEQ(SelVal, ExpectedID);

llvm::BasicBlock *CatchBB   = llvm::BasicBlock::Create(Ctx, "catch",   F);
llvm::BasicBlock *RethrowBB = llvm::BasicBlock::Create(Ctx, "rethrow", F);
B.CreateCondBr(Matches, CatchBB, RethrowBB);
```

---

## Step 5 — Wrap the catch body with `__cxa_begin_catch` / `__cxa_end_catch`

The Itanium ABI requires these around every catch handler:

```cpp
// Declare once, reuse:
llvm::FunctionType *BeginTy =
    llvm::FunctionType::get(B.getPtrTy(), {B.getPtrTy()}, false);
llvm::Function *BeginCatch =
    llvm::Function::Create(BeginTy, llvm::Function::ExternalLinkage,
                           "__cxa_begin_catch", *M);

llvm::FunctionType *EndTy =
    llvm::FunctionType::get(B.getVoidTy(), {}, false);
llvm::Function *EndCatch =
    llvm::Function::Create(EndTy, llvm::Function::ExternalLinkage,
                           "__cxa_end_catch", *M);

// In the catch block:
B.SetInsertPoint(CatchBB);
llvm::Value *ExnObj = B.CreateCall(BeginCatch, {ExnPtr}, "exn.obj");
// ... handle the exception (load from ExnObj, etc.) ...
B.CreateCall(EndCatch, {});
B.CreateBr(AfterBB);
```

---

## Step 6 — Re-throw with `resume`

If no handler matched, propagate the exception up:

```cpp
B.SetInsertPoint(RethrowBB);
B.CreateResume(LP); // terminates the block
```

---

## Step 7 — Cleanup blocks (finally / destructors)

A cleanup landing pad runs regardless of exception type:

```cpp
llvm::BasicBlock *CleanupBB = llvm::BasicBlock::Create(Ctx, "cleanup", F);
B.SetInsertPoint(CleanupBB);
llvm::LandingPadInst *CleanupLP = B.CreateLandingPad(LPadTy, 0);
CleanupLP->setCleanup(true);

// Run cleanup (call destructor, close file, etc.)
B.CreateCall(DestructorFn, {ObjPtr});

// Resume unwinding after cleanup
B.CreateResume(CleanupLP);
```

---

## Full try/catch pattern (consolidated)

```cpp
// try { result = foo(x); }
// catch (int e) { result = -1; }

// Set personality once at function start
F->setPersonalityFn(Personality);

llvm::BasicBlock *TryBB    = llvm::BasicBlock::Create(Ctx, "try",    F);
llvm::BasicBlock *NormalBB = llvm::BasicBlock::Create(Ctx, "normal", F);
llvm::BasicBlock *LpadBB   = llvm::BasicBlock::Create(Ctx, "lpad",   F);
llvm::BasicBlock *CatchBB  = llvm::BasicBlock::Create(Ctx, "catch",  F);
llvm::BasicBlock *AfterBB  = llvm::BasicBlock::Create(Ctx, "after",  F);

// try block
B.SetInsertPoint(TryBB);
llvm::InvokeInst *II = B.CreateInvoke(FooFT, FooFn, NormalBB, LpadBB, {X});

// normal path
B.SetInsertPoint(NormalBB);
B.CreateBr(AfterBB);

// landing pad
B.SetInsertPoint(LpadBB);
llvm::LandingPadInst *LP = B.CreateLandingPad(LPadTy, 1);
LP->addClause(IntTypeInfo);
llvm::Value *ExnPtr = B.CreateExtractValue(LP, 0);
llvm::Value *SelVal = B.CreateExtractValue(LP, 1);
llvm::Value *IntTypeID = B.CreateCall(TypeIDFn, {IntTypeInfo});
B.CreateCondBr(B.CreateICmpEQ(SelVal, IntTypeID), CatchBB, RethrowBB);

// catch (int e)
B.SetInsertPoint(CatchBB);
B.CreateCall(BeginCatch, {ExnPtr});
llvm::Value *ResultCatch = B.getInt32(-1);
B.CreateCall(EndCatch, {});
B.CreateBr(AfterBB);

// rethrow
B.SetInsertPoint(RethrowBB);
B.CreateResume(LP);

// after — PHI for result
B.SetInsertPoint(AfterBB);
llvm::PHINode *Result = B.CreatePHI(B.getInt32Ty(), 2, "result");
Result->addIncoming(II,          NormalBB);
Result->addIncoming(ResultCatch, CatchBB);
```

---

## Windows SEH note

For Windows x64, use `__C_specific_handler` as personality and the WinEH instruction set (`catchswitch`, `catchpad`, `catchret`, `cleanuppad`, `cleanupret`) — these are a different model from the Itanium `landingpad`. See [LLVM WinEH docs](https://llvm.org/docs/ExceptionHandling.html#wineh).

---

## Common mistakes

- **Never** use `call` for throwable calls in functions with landing pads — use `invoke`.
- **Never** forget `F->setPersonalityFn()` — landing pads without a personality crash the backend.
- **Never** add a `landingpad` in a function that has no `invoke` — the verifier rejects it.
- **Always** call `__cxa_begin_catch` / `__cxa_end_catch` around catch bodies (Itanium ABI).
- **Always** terminate landing pad chains with `resume` or `unreachable`.
- **Always** call `verifyModule(*M, &errs())` after wiring EH — terminator invariants are easy to violate.
