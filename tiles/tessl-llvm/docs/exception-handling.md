# Exception Handling in LLVM IR (LLVM 20)

Reference: [ExceptionHandling](https://llvm.org/docs/ExceptionHandling.html) | [LangRef: invoke](https://llvm.org/docs/LangReference.html#invoke-instruction)

---

## Overview

LLVM represents exception handling using three instruction types:

| Instruction | Purpose |
|-------------|---------|
| `invoke` | Like `call`, but branches to a landing pad on exception |
| `landingpad` | Entry point of exception handling code in the callee's frame |
| `resume` | Re-throws a caught exception (like C++ `throw;`) |

Two ABI models are supported:

| Model | Use when |
|-------|---------|
| **Itanium C++ ABI** (`personality = __gxx_personality_v0`) | C++, Rust, Swift, most languages on Linux/macOS |
| **SJLJ** (`personality = __gxx_personality_sj0`) | Targets without native unwind tables (embedded, WASM) |
| **Windows SEH** (`personality = __C_specific_handler`) | Windows x64 |

---

## Personality function

Every function that participates in exception handling must declare a **personality function**. This is the runtime routine that inspects the exception and decides whether a landing pad handles it.

```cpp
// Declare the personality function (usually provided by the C++ runtime)
llvm::FunctionType *PersonalityFT =
    llvm::FunctionType::get(B.getInt32Ty(), /*vararg=*/true);
llvm::Function *Personality =
    llvm::Function::Create(PersonalityFT, llvm::Function::ExternalLinkage,
                            "__gxx_personality_v0", *M);

// Attach to the function that contains landing pads
F->setPersonalityFn(Personality);
```

---

## invoke — call that can unwind

Replace `call` with `invoke` for any call that may throw:

```cpp
// call:   %r = call i32 @foo(i32 %x)
// invoke: %r = invoke i32 @foo(i32 %x)
//                to label %normal unwind label %lpad

llvm::BasicBlock *NormalBB = llvm::BasicBlock::Create(Ctx, "normal", F);
llvm::BasicBlock *LpadBB   = llvm::BasicBlock::Create(Ctx, "lpad",   F);

llvm::InvokeInst *II = B.CreateInvoke(
    CalleeFT,       // function type
    Callee,         // function pointer
    NormalBB,       // destination on normal return
    LpadBB,         // destination on unwind (landing pad)
    {Arg0, Arg1},   // call arguments
    "result"        // result name (if non-void)
);
```

The result of `invoke` is available in `NormalBB`:
```cpp
B.SetInsertPoint(NormalBB);
// II->use the return value here, then continue normally
```

---

## landingpad — catch / cleanup

A **landing pad** is the first instruction in the unwind destination block.
It extracts exception info from the runtime:

```cpp
B.SetInsertPoint(LpadBB);

// The landingpad returns { ptr, i32 }:
//   ptr  — pointer to the thrown exception object
//   i32  — selector value (which clause matched)
llvm::Type *LPadTy = llvm::StructType::get(Ctx, {B.getPtrTy(), B.getInt32Ty()});
llvm::LandingPadInst *LPad = B.CreateLandingPad(LPadTy, /*numClauses=*/1);

// Clause options:
// 1. catch <type>  — catch a specific exception type
llvm::Constant *ExnType = M->getOrInsertGlobal("_ZTIi",    // typeinfo for int
                                                 B.getPtrTy());
LPad->addClause(ExnType);

// 2. catch null ptr — catch-all (like catch(...))
LPad->addClause(llvm::ConstantPointerNull::get(B.getPtrTy()));

// 3. filter [types] — C++ exception specifications (rare)
// LPad->addClause(llvm::ConstantArray::get(...));

// 4. cleanup — always run (like C++ destructors); no type check
LPad->setCleanup(true);
```

After the landingpad, extract the components:

```cpp
// Extract exception pointer (i.e., the thrown object)
Value *ExnPtr  = B.CreateExtractValue(LPad, 0, "exn.ptr");
// Extract selector (which catch clause matched)
Value *SelVal  = B.CreateExtractValue(LPad, 1, "exn.sel");
```

---

## Dispatch: which catch clause matched?

Use `llvm.eh.typeid.for` to get the selector value for a type:

```cpp
// Get the expected selector value for catching type T
llvm::Function *TypeIDFn = llvm::Intrinsic::getOrInsertDeclaration(
    M, llvm::Intrinsic::eh_typeid_for, {B.getPtrTy()});
Value *TypeID = B.CreateCall(TypeIDFn, {ExnType});

// Compare selector to dispatch to the right handler
Value *Matches = B.CreateICmpEQ(SelVal, TypeID);
B.CreateCondBr(Matches, CatchIntBB, RethrowBB);
```

---

## __cxa_begin_catch / __cxa_end_catch

The Itanium ABI requires wrapping catch bodies with C++ runtime calls:

```cpp
// Declare the ABI helpers
llvm::FunctionType *BeginCatchTy =
    llvm::FunctionType::get(B.getPtrTy(), {B.getPtrTy()}, false);
llvm::Function *BeginCatch =
    llvm::Function::Create(BeginCatchTy, llvm::Function::ExternalLinkage,
                            "__cxa_begin_catch", *M);

llvm::FunctionType *EndCatchTy =
    llvm::FunctionType::get(B.getVoidTy(), {}, false);
llvm::Function *EndCatch =
    llvm::Function::Create(EndCatchTy, llvm::Function::ExternalLinkage,
                            "__cxa_end_catch", *M);

// In the catch block:
B.SetInsertPoint(CatchIntBB);
Value *ExnObj = B.CreateCall(BeginCatch, {ExnPtr}, "exn.obj");
// ... handle exception using ExnObj ...
B.CreateCall(EndCatch, {});
B.CreateBr(AfterCatchBB);
```

---

## resume — re-throw

To re-throw a caught exception (propagate it up the call stack):

```cpp
B.SetInsertPoint(RethrowBB);
B.CreateResume(LPad); // LPad is the landingpad value { ptr, i32 }
```

---

## cleanup — destructors / finally

A cleanup landing pad runs regardless of exception type (like `finally` or C++ destructors):

```cpp
B.SetInsertPoint(CleanupBB);
llvm::LandingPadInst *CleanupLP = B.CreateLandingPad(LPadTy, 0);
CleanupLP->setCleanup(true);

// Run cleanup code (call destructors, release resources, etc.)
B.CreateCall(DestructorFn, {ObjPtr});

// Then resume unwinding
B.CreateResume(CleanupLP);
```

---

## Full try/catch pattern

```cpp
// try { result = foo(x); }
// catch (int e) { result = -1; }

llvm::BasicBlock *TryBB     = llvm::BasicBlock::Create(Ctx, "try",     F);
llvm::BasicBlock *NormalBB  = llvm::BasicBlock::Create(Ctx, "normal",  F);
llvm::BasicBlock *LpadBB    = llvm::BasicBlock::Create(Ctx, "lpad",    F);
llvm::BasicBlock *CatchBB   = llvm::BasicBlock::Create(Ctx, "catch",   F);
llvm::BasicBlock *AfterBB   = llvm::BasicBlock::Create(Ctx, "after",   F);

// try block
B.SetInsertPoint(TryBB);
llvm::InvokeInst *II = B.CreateInvoke(FooFT, FooFn, NormalBB, LpadBB, {X});

// normal path — use return value
B.SetInsertPoint(NormalBB);
Value *ResultNormal = II;
B.CreateBr(AfterBB);

// landing pad
B.SetInsertPoint(LpadBB);
F->setPersonalityFn(Personality);
llvm::LandingPadInst *LP = B.CreateLandingPad(LPadTy, 1);
LP->addClause(IntTypeInfo); // catch int
Value *ExnPtr = B.CreateExtractValue(LP, 0);
Value *SelVal = B.CreateExtractValue(LP, 1);
Value *IntTypeID = B.CreateCall(TypeIDFn, {IntTypeInfo});
B.CreateCondBr(B.CreateICmpEQ(SelVal, IntTypeID), CatchBB, RethrowBB);

// catch (int e)
B.SetInsertPoint(CatchBB);
B.CreateCall(BeginCatch, {ExnPtr});
// ... read e from ExnPtr ...
Value *ResultCatch = B.getInt32(-1);
B.CreateCall(EndCatch, {});
B.CreateBr(AfterBB);

// after — merge results
B.SetInsertPoint(AfterBB);
llvm::PHINode *Result = B.CreatePHI(B.getInt32Ty(), 2, "result");
Result->addIncoming(ResultNormal, NormalBB);
Result->addIncoming(ResultCatch, CatchBB);
```

---

## Non-C++ languages: custom personality

For a language with different exception semantics, implement a custom personality function in C:

```c
// Personality function signature (Itanium ABI)
_Unwind_Reason_Code my_personality(
    int version, _Unwind_Action actions,
    uint64_t exceptionClass,
    struct _Unwind_Exception *exceptionObject,
    struct _Unwind_Context *context);
```

Then declare it in LLVM IR and attach via `F->setPersonalityFn()`.

---

## Windows SEH (structured exception handling)

For Windows x64 targets, use `__C_specific_handler` as personality and `catchpad` / `cleanuppad` instructions (WinEH model):

```llvm
; Windows SEH uses a different set of instructions:
; catchswitch, catchpad, catchret, cleanuppad, cleanupret
; — different from the Itanium landingpad model
```

See [LLVM WinEH docs](https://llvm.org/docs/ExceptionHandling.html#wineh) for the full WinEH model.

---

## Common mistakes

- **Do NOT** use `call` for functions that may throw in a function that uses exception handling — use `invoke` instead, or the unwind tables will be incorrect.
- **Do NOT** forget `F->setPersonalityFn()` — landing pads without a personality function will fail to compile.
- **Do NOT** use `landingpad` in a function without `invoke` — the verifier rejects this.
- **Do NOT** mix Itanium and Windows EH models in the same module.
- **ALWAYS** call `__cxa_begin_catch` / `__cxa_end_catch` around catch bodies when using the Itanium ABI — skipping these causes memory leaks and incorrect unwinding.
- **ALWAYS** add a `resume` or `unreachable` at the end of landing pad chains — missing terminators fail verification.
