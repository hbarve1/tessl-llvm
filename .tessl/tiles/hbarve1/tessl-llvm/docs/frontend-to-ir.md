# Frontend → LLVM IR Lowering Patterns (LLVM 22)

This page covers systematic patterns for lowering a language's AST to LLVM IR using `IRBuilder<>`. Each section maps a high-level language construct to the canonical IR sequence.

Reference: [ir-types.md](ir-types.md) | [IRBuilder Doxygen](https://llvm.org/doxygen/classllvm_1_1IRBuilderBase.html)

---

## Setup: LLVMContext, Module, IRBuilder

```cpp
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Verifier.h"

LLVMContext Ctx;
auto M = std::make_unique<Module>("my_lang", Ctx);
IRBuilder<> B(Ctx);

// Helper: get or declare an external function
Function *getOrDeclareFunc(Module &M, StringRef Name, FunctionType *FT) {
  if (auto *F = M.getFunction(Name)) return F;
  return Function::Create(FT, Function::ExternalLinkage, Name, M);
}
```

---

## Scalar expressions

### Arithmetic and comparison

```cpp
// a + b, a - b, a * b
Value *add(Value *A, Value *B) { return B.CreateAdd(A, B, "add"); }
Value *sub(Value *A, Value *B) { return B.CreateSub(A, B, "sub"); }
Value *mul(Value *A, Value *B) { return B.CreateMul(A, B, "mul"); }

// Integer division — signed vs unsigned matters
Value *sdiv(Value *A, Value *B) { return B.CreateSDiv(A, B, "sdiv"); }
Value *udiv(Value *A, Value *B) { return B.CreateUDiv(A, B, "udiv"); }

// Float arithmetic
Value *fadd(Value *A, Value *B) { return B.CreateFAdd(A, B, "fadd"); }

// Comparisons → i1
Value *eq (Value *A, Value *B) { return B.CreateICmpEQ(A, B); }
Value *lt (Value *A, Value *B) { return B.CreateICmpSLT(A, B); } // signed
Value *ult(Value *A, Value *B) { return B.CreateICmpULT(A, B); } // unsigned
Value *flt(Value *A, Value *B) { return B.CreateFCmpOLT(A, B); } // float ordered

// Boolean: i1 and/or/not
Value *land(Value *A, Value *B) { return B.CreateAnd(A, B); }
Value *lor (Value *A, Value *B) { return B.CreateOr(A, B);  }
Value *lnot(Value *A)           { return B.CreateNot(A);    }
```

### Type conversions

```cpp
// Widen integer
Value *sext(Value *V, Type *T) { return B.CreateSExt(V, T); }  // signed extend
Value *zext(Value *V, Type *T) { return B.CreateZExt(V, T); }  // zero extend
// Narrow integer
Value *trunc(Value *V, Type *T) { return B.CreateTrunc(V, T); }
// Int ↔ float
Value *sitofp(Value *V, Type *T) { return B.CreateSIToFP(V, T); }
Value *fptosi(Value *V, Type *T) { return B.CreateFPToSI(V, T); }
```

---

## Local variables (mutable)

The canonical pattern: alloca in the entry block, load/store everywhere else.
Run `mem2reg` / `sroa` passes afterwards to promote to SSA registers.

```cpp
// Allocate a local variable in the function entry block
AllocaInst *createEntryAlloca(Function *F, Type *Ty, StringRef Name) {
  IRBuilder<> EntryB(&F->getEntryBlock(), F->getEntryBlock().begin());
  return EntryB.CreateAlloca(Ty, nullptr, Name);
}

// Assign:  x = expr
void emitAssign(AllocaInst *Var, Value *Val) {
  B.CreateStore(Val, Var);
}

// Load:  use x
Value *emitLoad(AllocaInst *Var, Type *Ty) {
  return B.CreateLoad(Ty, Var, Var->getName());
}
```

> **Always alloca in the entry block**, not at the point of use. LLVM's `mem2reg` only promotes entry-block allocas. Allocas elsewhere become real stack slots.

---

## If / else

```cpp
// if (Cond) { Then } else { Else }
Value *emitIfElse(Function *F, Value *Cond,
                  std::function<Value*()> emitThen,
                  std::function<Value*()> emitElse,
                  Type *ResultTy) {
  BasicBlock *ThenBB  = BasicBlock::Create(Ctx, "if.then", F);
  BasicBlock *ElseBB  = BasicBlock::Create(Ctx, "if.else", F);
  BasicBlock *MergeBB = BasicBlock::Create(Ctx, "if.merge", F);

  B.CreateCondBr(Cond, ThenBB, ElseBB);

  // Then branch
  B.SetInsertPoint(ThenBB);
  Value *ThenVal = emitThen();
  B.CreateBr(MergeBB);
  ThenBB = B.GetInsertBlock(); // may have changed if emitThen added blocks

  // Else branch
  B.SetInsertPoint(ElseBB);
  Value *ElseVal = emitElse();
  B.CreateBr(MergeBB);
  ElseBB = B.GetInsertBlock();

  // Merge with PHI
  B.SetInsertPoint(MergeBB);
  if (ResultTy && !ResultTy->isVoidTy()) {
    PHINode *Phi = B.CreatePHI(ResultTy, 2, "if.result");
    Phi->addIncoming(ThenVal, ThenBB);
    Phi->addIncoming(ElseVal, ElseBB);
    return Phi;
  }
  return nullptr;
}

// if (Cond) { Then }  — no else, no result value
void emitIf(Function *F, Value *Cond, std::function<void()> emitThen) {
  BasicBlock *ThenBB  = BasicBlock::Create(Ctx, "if.then", F);
  BasicBlock *MergeBB = BasicBlock::Create(Ctx, "if.merge", F);
  B.CreateCondBr(Cond, ThenBB, MergeBB);
  B.SetInsertPoint(ThenBB);
  emitThen();
  if (!B.GetInsertBlock()->getTerminator())
    B.CreateBr(MergeBB);
  B.SetInsertPoint(MergeBB);
}
```

---

## While loop

```cpp
// while (Cond) { Body }
void emitWhile(Function *F, std::function<Value*()> emitCond,
               std::function<void()> emitBody) {
  BasicBlock *CondBB = BasicBlock::Create(Ctx, "while.cond", F);
  BasicBlock *BodyBB = BasicBlock::Create(Ctx, "while.body", F);
  BasicBlock *ExitBB = BasicBlock::Create(Ctx, "while.exit", F);

  B.CreateBr(CondBB);

  B.SetInsertPoint(CondBB);
  Value *Cond = emitCond();
  B.CreateCondBr(Cond, BodyBB, ExitBB);

  B.SetInsertPoint(BodyBB);
  emitBody();
  if (!B.GetInsertBlock()->getTerminator())
    B.CreateBr(CondBB); // back-edge

  B.SetInsertPoint(ExitBB);
}
```

## For loop (counted)

```cpp
// for (i = Start; i < End; i++) { Body(i) }
void emitFor(Function *F, Value *Start, Value *End,
             std::function<void(Value *)> emitBody) {
  Type *I32 = B.getInt32Ty();
  AllocaInst *IVar = createEntryAlloca(F, I32, "i");
  B.CreateStore(Start, IVar);

  BasicBlock *CondBB = BasicBlock::Create(Ctx, "for.cond", F);
  BasicBlock *BodyBB = BasicBlock::Create(Ctx, "for.body", F);
  BasicBlock *IncrBB = BasicBlock::Create(Ctx, "for.incr", F);
  BasicBlock *ExitBB = BasicBlock::Create(Ctx, "for.exit", F);

  B.CreateBr(CondBB);

  B.SetInsertPoint(CondBB);
  Value *IVal = B.CreateLoad(I32, IVar, "i");
  B.CreateCondBr(B.CreateICmpSLT(IVal, End), BodyBB, ExitBB);

  B.SetInsertPoint(BodyBB);
  emitBody(B.CreateLoad(I32, IVar, "i"));
  B.CreateBr(IncrBB);

  B.SetInsertPoint(IncrBB);
  Value *Next = B.CreateAdd(B.CreateLoad(I32, IVar, "i"), B.getInt32(1));
  B.CreateStore(Next, IVar);
  B.CreateBr(CondBB);

  B.SetInsertPoint(ExitBB);
}
```

---

## Functions

### Defining a function

```cpp
Function *emitFunctionDecl(Module &M, StringRef Name,
                            Type *RetTy, ArrayRef<Type *> ParamTys,
                            bool IsVarArg = false) {
  auto *FT = FunctionType::get(RetTy, ParamTys, IsVarArg);
  auto *F  = Function::Create(FT, Function::ExternalLinkage, Name, M);
  return F;
}

Function *emitFunctionDef(Module &M, StringRef Name,
                           Type *RetTy, ArrayRef<Type *> ParamTys,
                           ArrayRef<StringRef> ParamNames,
                           std::function<Value*(Function *, ArrayRef<Value*>)> emitBody) {
  Function *F = emitFunctionDecl(M, Name, RetTy, ParamTys);

  // Name parameters
  unsigned I = 0;
  SmallVector<Value *, 8> Args;
  for (Argument &Arg : F->args()) {
    if (I < ParamNames.size()) Arg.setName(ParamNames[I]);
    Args.push_back(&Arg);
    ++I;
  }

  BasicBlock *Entry = BasicBlock::Create(Ctx, "entry", F);
  B.SetInsertPoint(Entry);

  Value *Ret = emitBody(F, Args);
  if (Ret)
    B.CreateRet(Ret);
  else
    B.CreateRetVoid();

  verifyFunction(*F, &errs());
  return F;
}
```

### Calling a function

```cpp
// Direct call
Value *emitCall(Function *Callee, ArrayRef<Value *> Args, StringRef Name = "") {
  return B.CreateCall(Callee->getFunctionType(), Callee, Args, Name);
}

// Indirect call through a function pointer
Value *emitIndirectCall(FunctionType *FT, Value *FnPtr,
                        ArrayRef<Value *> Args, StringRef Name = "") {
  return B.CreateCall(FT, FnPtr, Args, Name);
}
```

### Recursive functions

Declare before defining — LLVM allows forward references at the IR level:

```cpp
// 1. Declare first (creates an external declaration)
Function *RecFn = emitFunctionDecl(M, "factorial", I64Ty, {I64Ty});

// 2. Add body — RecFn is already visible for recursive calls
BasicBlock *Entry = BasicBlock::Create(Ctx, "entry", RecFn);
B.SetInsertPoint(Entry);
Value *N = RecFn->getArg(0);
// if (n <= 1) return 1; else return n * factorial(n-1);
// ... emit using emitIfElse and emitCall(RecFn, {B.CreateSub(N, One)})
```

---

## Structs

```cpp
// Define a struct type: { i32, double, ptr }
StructType *PersonTy = StructType::create(Ctx, "Person");
PersonTy->setBody({B.getInt32Ty(), B.getDoubleTy(), B.getPtrTy()});

// Allocate a struct on the stack
AllocaInst *P = B.CreateAlloca(PersonTy, nullptr, "p");

// Store to field 0 (age): GEP then store
Value *AgePtr = B.CreateStructGEP(PersonTy, P, 0, "age.ptr");
B.CreateStore(B.getInt32(30), AgePtr);

// Load from field 1 (score)
Value *ScorePtr = B.CreateStructGEP(PersonTy, P, 1, "score.ptr");
Value *Score    = B.CreateLoad(B.getDoubleTy(), ScorePtr, "score");

// Allocate a struct on the heap (call malloc / language allocator)
// Then use the returned ptr with GEP the same way
```

> Use `CreateStructGEP(StructTy, Ptr, FieldIndex)` — not `CreateGEP` with two indices. It's clearer and type-safe.

---

## Arrays

```cpp
// Stack array: [10 x i32]
Type *ArrTy = ArrayType::get(B.getInt32Ty(), 10);
AllocaInst *Arr = B.CreateAlloca(ArrTy, nullptr, "arr");

// Access element i: GEP [10 x i32]* with {0, i}
Value *ElemPtr = B.CreateInBoundsGEP(ArrTy, Arr,
    {B.getInt64(0), Idx}, "elem.ptr");
B.CreateStore(Val, ElemPtr);
Value *Elem = B.CreateLoad(B.getInt32Ty(), ElemPtr, "elem");

// Heap array (pointer to first element, no array type in IR):
// AllocSize = sizeof(i32) * N
Value *AllocSize = B.CreateMul(B.getInt64(4), N, "alloc.sz");
// Call malloc, cast result to ptr (opaque) — element type tracked by you
Value *HeapArr = B.CreateCall(MallocFn, {AllocSize}, "heap.arr");
// Access element i: GEP i32 with {i}
Value *HElemPtr = B.CreateInBoundsGEP(B.getInt32Ty(), HeapArr, {Idx});
```

---

## Closures / first-class functions

LLVM IR has no native closure type. The standard approach:

1. **Function pointer + environment pointer** (two-word pair)
2. **Closure struct** = `{ ptr fnptr, ptr env }` where `env` holds captured variables

```cpp
// Closure struct type: { ptr, ptr }
StructType *ClosureTy = StructType::get(Ctx, {B.getPtrTy(), B.getPtrTy()});

// Environment struct for captures: { i32 captured_x, double captured_y }
StructType *EnvTy = StructType::get(Ctx, {B.getInt32Ty(), B.getDoubleTy()});

// Closure function signature: (ptr env, <original args...>) -> ret
FunctionType *ClosureFnTy = FunctionType::get(RetTy,
    {B.getPtrTy(), Arg1Ty, Arg2Ty}, false);

// Emit the closure body function
Function *ClosureFn = Function::Create(ClosureFnTy,
    Function::InternalLinkage, "my_closure", M);
{
  BasicBlock *Entry = BasicBlock::Create(Ctx, "entry", ClosureFn);
  B.SetInsertPoint(Entry);
  Value *EnvPtr = ClosureFn->getArg(0); // first arg is always env
  // Load captured x from env field 0
  Value *XPtr = B.CreateStructGEP(EnvTy, EnvPtr, 0);
  Value *X    = B.CreateLoad(B.getInt32Ty(), XPtr, "x");
  // ... use X and remaining args
}

// At call site: allocate env, fill captures, build closure struct
AllocaInst *Env = B.CreateAlloca(EnvTy, nullptr, "env");
B.CreateStore(CapturedX, B.CreateStructGEP(EnvTy, Env, 0));

AllocaInst *Closure = B.CreateAlloca(ClosureTy, nullptr, "closure");
B.CreateStore(ClosureFn, B.CreateStructGEP(ClosureTy, Closure, 0));
B.CreateStore(Env,       B.CreateStructGEP(ClosureTy, Closure, 1));

// Invoke closure:
Value *FnPtr  = B.CreateLoad(B.getPtrTy(),
    B.CreateStructGEP(ClosureTy, Closure, 0));
Value *EnvArg = B.CreateLoad(B.getPtrTy(),
    B.CreateStructGEP(ClosureTy, Closure, 1));
B.CreateCall(ClosureFnTy, FnPtr, {EnvArg, Arg1, Arg2});
```

---

## String literals

```cpp
// Emit a global string constant and get a ptr to it
Value *emitStringLiteral(Module &M, StringRef Str, StringRef Name = ".str") {
  // CreateGlobalStringPtr creates a [N x i8] global and returns ptr to first byte
  return B.CreateGlobalStringPtr(Str, Name, /*AddressSpace=*/0, &M);
}
// Result is a ptr (opaque) usable anywhere a C `const char*` is expected.
```

---

## Early return / break / continue

Model control flow exits as jumps to a pre-created exit block:

```cpp
// Pattern: single exit block per function, return value via alloca
AllocaInst *RetSlot = createEntryAlloca(F, RetTy, "retval");
BasicBlock *ExitBB  = BasicBlock::Create(Ctx, "exit", F);

// At each return site:
B.CreateStore(RetVal, RetSlot);
B.CreateBr(ExitBB);

// At ExitBB:
B.SetInsertPoint(ExitBB);
Value *FinalRet = B.CreateLoad(RetTy, RetSlot, "retval");
B.CreateRet(FinalRet);

// Break/continue: keep a stack of (CondBB, ExitBB) per loop, jump to them.
```

---

## Post-lowering: run mem2reg

After emitting IR with alloca/load/store, promote locals to SSA values:

```cpp
#include "llvm/Passes/PassBuilder.h"

PassBuilder PB;
FunctionAnalysisManager FAM;
// ... register analyses ...

FunctionPassManager FPM;
FPM.addPass(PromotePass());      // mem2reg
FPM.addPass(InstCombinePass());  // clean up trivial redundancy
FPM.addPass(SimplifyCFGPass());  // remove empty/unreachable blocks
FPM.run(*F, FAM);
```

---

## Common mistakes

- **Do NOT** build PHI nodes manually for every variable — use alloca + mem2reg instead. PHIs are notoriously hard to construct correctly during initial lowering.
- **Do NOT** alloca inside loop bodies — always alloca in the **entry block** so `mem2reg` can promote them.
- **Do NOT** forget to update `B.GetInsertBlock()` after emitting a nested construct — the insert point may have moved.
- **Do NOT** call `verifyModule` only at the end — call `verifyFunction` after each emitted function to catch mistakes early.
- **ALWAYS** use `Function::InternalLinkage` for helper/closure functions not visible outside the module; `ExternalLinkage` only for public symbols.
