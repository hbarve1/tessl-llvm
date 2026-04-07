# LLVM IR — Types, Values, and Constants (LLVM 20)

Reference: [LLVM LangRef](https://llvm.org/docs/LangReference.html) | [IRBuilder Doxygen](https://llvm.org/doxygen/classllvm_1_1IRBuilderBase.html)

---

## Type hierarchy

All LLVM types inherit from `llvm::Type`. Types are uniqued per `LLVMContext` — two `i32` types in the same context are the same pointer.

```
Type
├── IntegerType         — i1, i8, i16, i32, i64, i128, arbitrary iN
├── FloatingPointType   — half, bfloat, float, double, x86_fp80, fp128
├── PointerType         — ptr (opaque, LLVM 20: no typed pointers)
├── ArrayType           — [N x T]
├── FixedVectorType     — <N x T>
├── ScalableVectorType  — <vscale x N x T>  (SVE / RISC-V V)
├── StructType          — { T1, T2, ... } or named/opaque structs
├── FunctionType        — ret (arg1, arg2, ...) [vararg]
└── VoidType / LabelType / MetadataType / TokenType
```

---

## Getting types (LLVM 20)

```cpp
LLVMContext &Ctx = M.getContext();
IRBuilder<> B(Ctx);

// Integer types
Type *I1  = B.getInt1Ty();    // i1
Type *I8  = B.getInt8Ty();    // i8
Type *I32 = B.getInt32Ty();   // i32
Type *I64 = B.getInt64Ty();   // i64
Type *IN  = IntegerType::get(Ctx, N);  // arbitrary iN

// Floating point
Type *F32 = B.getFloatTy();   // float
Type *F64 = B.getDoubleTy();  // double
Type *F16 = B.getHalfTy();    // half
Type *BF  = B.getBFloatTy();  // bfloat

// Pointer — LLVM 20: opaque only, no element type
Type *Ptr    = B.getPtrTy();          // ptr (address space 0)
Type *PtrAS1 = B.getPtrTy(1);         // ptr addrspace(1)
// Equivalent: PointerType::get(Ctx, 0)

// Void
Type *Void = B.getVoidTy();

// Struct (literal)
StructType *ST = StructType::get(Ctx, {I32, F64});
// Named struct (for recursive/opaque types)
StructType *Named = StructType::create(Ctx, "MyStruct");
Named->setBody({I32, Ptr});

// Array
ArrayType *Arr = ArrayType::get(I32, 16);   // [16 x i32]

// Fixed vector
VectorType *V4I32 = FixedVectorType::get(I32, 4);    // <4 x i32>
// Scalable vector
VectorType *SV    = ScalableVectorType::get(I32, 4); // <vscale x 4 x i32>

// Function type
FunctionType *FT = FunctionType::get(I32, {I32, I32}, /*vararg=*/false);
```

### Type predicates

```cpp
T->isIntegerTy()       T->isIntegerTy(32)   // is i32?
T->isFloatingPointTy()
T->isPointerTy()
T->isStructTy()
T->isArrayTy()
T->isVectorTy()
T->isVoidTy()
T->isSized()           // false for opaque structs, void, function types
```

---

## Value hierarchy

All LLVM IR entities are `llvm::Value` subclasses. A `Value` has a `Type *` and a list of `Use`s (users of this value).

```
Value
├── Constant
│   ├── ConstantInt          — integer constants
│   ├── ConstantFP           — floating-point constants
│   ├── ConstantPointerNull  — null pointer
│   ├── ConstantAggregateZero — zeroinitializer
│   ├── ConstantArray / ConstantStruct / ConstantVector
│   ├── ConstantExpr         — constant expressions (GEP, bitcast, etc.)
│   ├── UndefValue           — undef
│   ├── PoisonValue          — poison (LLVM 20 preferred over undef for UB)
│   └── GlobalValue
│       ├── GlobalVariable
│       └── Function
├── Instruction              — lives inside a BasicBlock
│   ├── BinaryOperator (add, mul, ...)
│   ├── CmpInst (icmp, fcmp)
│   ├── BranchInst, ReturnInst, SwitchInst
│   ├── LoadInst, StoreInst
│   ├── GetElementPtrInst
│   ├── CallInst, InvokeInst
│   ├── PHINode
│   ├── AllocaInst
│   └── ... (80+ instruction kinds)
├── BasicBlock
├── Argument                 — function parameters
└── InlineAsm
```

---

## Creating constants

```cpp
// Integer constants
Constant *Zero  = B.getInt32(0);
Constant *One   = ConstantInt::get(I32, 1);
Constant *Big   = ConstantInt::get(I64, 0xDEADBEEFULL);
Constant *True  = B.getTrue();   // i1 1
Constant *False = B.getFalse();  // i1 0

// Floating-point constants
Constant *Pi  = ConstantFP::get(F64, 3.14159265358979);
Constant *FZero = ConstantFP::get(F32, 0.0);

// Null pointer (opaque)
Constant *Null = ConstantPointerNull::get(B.getPtrTy());

// Aggregate zeroinitializer
Constant *ZeroArr = ConstantAggregateZero::get(ArrayType::get(I32, 4));

// Struct constant
Constant *SC = ConstantStruct::get(ST, {B.getInt32(1),
                                        ConstantFP::get(F64, 2.0)});

// Undef / Poison
Value *U = UndefValue::get(I32);
Value *P = PoisonValue::get(I32);  // prefer poison over undef for UB modeling
```

---

## IRBuilder — building instructions

`IRBuilder<>` is the primary API for emitting instructions. Always set the insertion point before emitting.

```cpp
// Set insertion point to end of a BasicBlock
B.SetInsertPoint(BB);
// Set to before a specific instruction
B.SetInsertPoint(SomeInst);

// Arithmetic
Value *Sum  = B.CreateAdd(A, B_val, "sum");     // NSW/NUW variants:
Value *SumN = B.CreateNSWAdd(A, B_val);         // add nsw
Value *Mul  = B.CreateMul(A, B_val);
Value *FAdd = B.CreateFAdd(A, B_val);           // float add

// Bitwise
Value *And = B.CreateAnd(A, Mask);
Value *Or  = B.CreateOr(A, Mask);
Value *Shl = B.CreateShl(A, Amt);
Value *LShr = B.CreateLShr(A, Amt);
Value *AShr = B.CreateAShr(A, Amt);

// Comparisons
Value *Eq  = B.CreateICmpEQ(A, B_val);
Value *Slt = B.CreateICmpSLT(A, B_val);        // signed less-than
Value *Ult = B.CreateICmpULT(A, B_val);        // unsigned less-than
Value *Flt = B.CreateFCmpOLT(A, B_val);        // float ordered less-than

// Memory (LLVM 20: must provide explicit element type for load/store)
AllocaInst *Alloca = B.CreateAlloca(I32, nullptr, "x");
B.CreateStore(Value42, Alloca);
Value *Loaded = B.CreateLoad(I32, Alloca, "x.val");

// GEP (opaque pointer — always provide explicit type)
Value *GEP = B.CreateGEP(I32, ArrayPtr, {B.getInt64(0), B.getInt32(3)});
// Inbounds GEP (UB if out of bounds, enables more optimization)
Value *IGEP = B.CreateInBoundsGEP(I32, ArrayPtr, {Idx});
// LLVM 22: GEPNoWrapFlags for richer no-wrap semantics
Value *NGEP = B.CreateGEP(I32, ArrayPtr, {Idx}, "gep",
                           GEPNoWrapFlags::noUnsignedWrap()); // no unsigned overflow

// Get index type for a given address space (new in LLVM 22)
IntegerType *IdxTy = B.getIndexTy(M->getDataLayout(), /*AddrSpace=*/0);

// Casts
Value *Ext  = B.CreateSExt(I8Val, I32);         // sign extend
Value *ZExt = B.CreateZExt(I8Val, I32);         // zero extend
Value *Trunc = B.CreateTrunc(I64Val, I32);
Value *BitC  = B.CreateBitCast(Val, DestTy);    // same-size reinterpret
Value *PtrI  = B.CreatePtrToInt(PtrVal, I64);
Value *IPtr  = B.CreateIntToPtr(IntVal, B.getPtrTy());
Value *FPExt = B.CreateFPExt(F32Val, F64);

// Control flow
B.CreateBr(TargetBB);                           // unconditional branch
B.CreateCondBr(Cond, TrueBB, FalseBB);          // conditional branch
B.CreateRet(RetVal);
B.CreateRetVoid();

// Function calls
CallInst *CI = B.CreateCall(FnType, FnPtr, {Arg0, Arg1}, "result");
// Intrinsic call
Value *R = B.CreateIntrinsic(Intrinsic::abs, {I32}, {Val, B.getFalse()});

// PHI node
PHINode *Phi = B.CreatePHI(I32, 2, "phi");
Phi->addIncoming(ValFromBB1, BB1);
Phi->addIncoming(ValFromBB2, BB2);

// Select
Value *Sel = B.CreateSelect(Cond, TrueVal, FalseVal);
```

---

## Metadata

Metadata attaches non-semantic information to instructions and functions.

```cpp
// Attach debug location (DILocation)
Inst->setDebugLoc(DebugLoc::get(Line, Col, Scope));

// Attach named metadata to instruction
MDNode *Node = MDNode::get(Ctx, {MDString::get(Ctx, "mykey")});
Inst->setMetadata("my.tag", Node);

// Read metadata
MDNode *M = Inst->getMetadata("my.tag");
if (M) { /* ... */ }

// !tbaa — type-based alias analysis
// !range — integer range hint
// !nonnull — pointer is non-null
// !align — alignment assertion
// These are set via IRBuilder helpers or manually via setMetadata.
```

---

## Use-def chains

```cpp
// Iterate users of a value (who uses this value):
for (User *U : V->users()) {
  if (auto *I = dyn_cast<Instruction>(U)) { /* ... */ }
}

// Iterate operands of an instruction (what this instruction uses):
for (Use &Op : I->operands()) {
  Value *Operand = Op.get();
}

// RAUW — replace all uses of OldVal with NewVal
OldVal->replaceAllUsesWith(NewVal);

// Check if value has no uses (safe to erase)
if (V->use_empty()) V->eraseFromParent();
```

---

## Useful casts and predicates

```cpp
// Safe downcast — returns nullptr if wrong type
auto *CI = dyn_cast<ConstantInt>(V);

// Assert downcast — UB if wrong type; use only when certain
auto *F  = cast<Function>(V);

// Type check
isa<PHINode>(V)
isa<CallInst>(V)

// Pattern matching (prefer over manual casting)
#include "llvm/IR/PatternMatch.h"
using namespace llvm::PatternMatch;
Value *X, *Y;
if (match(V, m_Add(m_Value(X), m_Value(Y)))) { /* V = X + Y */ }
if (match(V, m_ConstantInt(C))) { /* V is a constant integer C */ }
if (match(V, m_Zero())) { /* V is integer/FP zero */ }
```
