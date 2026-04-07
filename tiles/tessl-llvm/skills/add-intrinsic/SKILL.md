---
name: add-intrinsic
description: Add a new LLVM IR intrinsic in LLVM 22. Covers TableGen definition, header regeneration, IRBuilder call-site usage, verifier rules, and lit testing.
---

# Skill: Add a New LLVM IR Intrinsic (LLVM 22)

Use this skill when the user wants to add a new first-class LLVM IR intrinsic — a built-in function with a well-known name (`llvm.*`) that the optimizer and backends can reason about specially.

> If the user only needs to call an existing intrinsic from a frontend, skip to Step 4.

---

## Step 0 — Decide intrinsic category

| Category | TableGen file | Example |
|----------|--------------|---------|
| Generic (target-independent) | `include/llvm/IR/Intrinsics.td` | `llvm.memcpy`, `llvm.expect` |
| Target-specific | `include/llvm/IR/Intrinsics<Target>.td` | `llvm.x86.*`, `llvm.aarch64.*` |
| In-tree experimental | `include/llvm/IR/IntrinsicsExperimental.td` | Prefix: `llvm.experimental.*` |

For a new language or target, use **target-specific** (`IntrinsicsMyTarget.td`) or prefix with `llvm.experimental.` during development.

---

## Step 1 — Define the intrinsic in TableGen

Open the appropriate `.td` file and add:

```tablegen
// In include/llvm/IR/Intrinsics.td  (or Intrinsics<Target>.td)

def int_my_intrinsic : DefaultAttrsIntrinsic<
  [llvm_i32_ty],              // Return types (list; use [] for void)
  [llvm_i32_ty, llvm_i32_ty], // Parameter types
  [IntrNoMem, IntrWillReturn] // Attributes / properties
>;
```

The function name is derived from the `def` name:
- `int_my_intrinsic` → `llvm.my.intrinsic`
- `int_x86_my_op`   → `llvm.x86.my.op`

### Common intrinsic properties (LLVM 22)

| Property | Meaning |
|----------|---------|
| `IntrNoMem` | Does not access memory |
| `IntrReadMem` | Only reads memory |
| `IntrWriteMem` | Only writes memory |
| `IntrArgMemOnly` | Only accesses memory via pointer args |
| `IntrWillReturn` | Always returns (no infinite loops/throws) |
| `IntrSpeculatable` | Safe to speculate (no side effects) |
| `IntrHasSideEffects` | Has side effects; cannot be removed |
| `Commutative` | First two args commute |
| `ImmArg<ArgIndex<N>>` | Arg N must be an immediate constant |
| `ReadOnly<ArgIndex<N>>` | Pointer arg N is read-only |

### Common type tokens

```tablegen
llvm_i1_ty    llvm_i8_ty    llvm_i16_ty   llvm_i32_ty   llvm_i64_ty
llvm_half_ty  llvm_float_ty llvm_double_ty
llvm_ptr_ty                               // opaque pointer (LLVM 22 default)
llvm_anyint_ty  llvm_anyfloat_ty          // overloaded
llvm_anyvector_ty
llvm_void_ty
```

For an **overloaded intrinsic** (type varies per call):

```tablegen
def int_my_abs : DefaultAttrsIntrinsic<
  [llvm_anyint_ty],
  [LLVMMatchType<0>],   // param type must match return type
  [IntrNoMem, IntrWillReturn]
>;
```

---

## Step 2 — Regenerate TableGen headers

LLVM generates `IntrinsicEnums.inc` and `IntrinsicImpl.inc` from the `.td` files.

**During development (rebuild LLVM):**
```bash
cmake --build build --target intrinsics_gen
# or just rebuild:
cmake --build build -j$(nproc)
```

The generated files end up in:
```
build/include/llvm/IR/IntrinsicEnums.inc
build/include/llvm/IR/IntrinsicImpl.inc
```

You do **not** hand-edit these files — they are always regenerated.

After regeneration, the new intrinsic is accessible via:
```cpp
llvm::Intrinsic::my_intrinsic   // enum value
```

---

## Step 3 — Add a lowering / selection rule (if target-specific)

For a target-specific intrinsic to generate real machine code:

### GlobalISel path (preferred in LLVM 22)

Add a `GINodeEquiv` entry in the target's `.td` file and a `GICombineRule` or
`GISelCombinerPass` entry, or handle it in `<Target>LegalizerInfo.cpp`:

```cpp
// In <Target>LegalizerInfo.cpp, legalizeIntrinsic():
case Intrinsic::my_intrinsic: {
  MIRBuilder.buildInstr(MyTarget::MY_OPCODE)
      .addDef(MI.getOperand(0).getReg())
      .addUse(MI.getOperand(2).getReg());
  MI.eraseFromParent();
  return true;
}
```

### SelectionDAG path (legacy but still used)

Override `LowerINTRINSIC_WO_CHAIN` or `LowerINTRINSIC_W_CHAIN` in
`<Target>ISelLowering.cpp`:

```cpp
SDValue <Target>TargetLowering::LowerINTRINSIC_WO_CHAIN(
    SDValue Op, SelectionDAG &DAG) const {
  unsigned IntNo = Op.getConstantOperandVal(0);
  switch (IntNo) {
  case Intrinsic::my_intrinsic:
    return DAG.getNode(MyTargetISD::MY_NODE, SDLoc(Op), Op.getValueType(),
                       Op.getOperand(1), Op.getOperand(2));
  }
  return SDValue();
}
```

---

## Step 4 — Call the intrinsic from IRBuilder

```cpp
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/IntrinsicInst.h"
#include "llvm/IR/Intrinsics.h"

// Inside a function body:
IRBuilder<> Builder(InsertBB, InsertBB->end());

// Option A: IRBuilder::CreateIntrinsic (LLVM 22 preferred API)
Value *Result = Builder.CreateIntrinsic(
    Intrinsic::my_intrinsic,
    {Builder.getInt32Ty()},   // type arguments for overloaded intrinsics; {} if not overloaded
    {ArgA, ArgB}              // call arguments
);

// Option B: Manual — getOrInsertFunction then CreateCall
Function *F = Intrinsic::getOrInsertDeclaration(M, Intrinsic::my_intrinsic,
                                                 {Builder.getInt32Ty()});
Value *Result2 = Builder.CreateCall(F, {ArgA, ArgB});
```

> In LLVM 22, `Intrinsic::getDeclaration()` is **removed** — use
> `Intrinsic::getOrInsertDeclaration` instead.

---

## Step 5 — Add verifier rules (if needed)

If the intrinsic has non-trivial constraints (e.g., arg 2 must be a constant,
pointer alignment requirements), add a check in `lib/IR/Verifier.cpp`:

```cpp
case Intrinsic::my_intrinsic: {
  Assert(isa<ConstantInt>(Call.getArgOperand(1)),
         "llvm.my.intrinsic: second argument must be a constant integer", &Call);
  break;
}
```

---

## Step 6 — Write a lit test

**Path:** `test/CodeGen/<Target>/my-intrinsic.ll` (for codegen tests)
or `test/Transforms/<Category>/my-intrinsic.ll` (for IR-level tests)

```llvm
; RUN: opt -passes=verify -S %s | FileCheck %s
; RUN: opt -passes=instcombine -S %s | FileCheck %s --check-prefix=OPT

declare i32 @llvm.my.intrinsic(i32, i32)

define i32 @test(i32 %a, i32 %b) {
; CHECK-LABEL: @test
  %r = call i32 @llvm.my.intrinsic(i32 %a, i32 %b)
; CHECK: call i32 @llvm.my.intrinsic
  ret i32 %r
}

; Test constant folding (if applicable)
define i32 @test_const() {
; OPT-LABEL: @test_const
  %r = call i32 @llvm.my.intrinsic(i32 3, i32 4)
; OPT: ret i32 7
  ret i32 %r
}
```

Run:
```bash
llvm-lit test/CodeGen/<Target>/my-intrinsic.ll -v
```

---

## Step 7 — Add InstCombine fold (optional)

If the intrinsic has constant-fold / simplification rules, add them in
`lib/Transforms/InstCombine/InstCombineCalls.cpp`:

```cpp
case Intrinsic::my_intrinsic: {
  // Fold llvm.my.intrinsic(c1, c2) to constant when both args are constants.
  if (auto *C1 = dyn_cast<ConstantInt>(II->getArgOperand(0)))
    if (auto *C2 = dyn_cast<ConstantInt>(II->getArgOperand(1)))
      return IC.replaceInstUsesWith(
          *II, ConstantInt::get(II->getType(),
                                C1->getValue() + C2->getValue()));
  break;
}
```

---

## Common mistakes to avoid

- **Do NOT** use `Intrinsic::getDeclaration()` — it is removed in LLVM 22; use `getOrInsertDeclaration()` or `getDeclarationIfExists()`.
- **Do NOT** forget to rebuild `intrinsics_gen` after editing `.td` files.
- **Do NOT** use named pointer types (`i8*`) in intrinsic signatures — LLVM 22 uses **opaque pointers** (`ptr`) everywhere.
- **Do NOT** declare the intrinsic with `declare` in your `.td` and also hand-write a `declare` in IR — the verifier will reject duplicate declarations with mismatched types.
- **ALWAYS** set appropriate memory attributes (`IntrNoMem`, `IntrReadMem`, etc.) — wrong attributes block valid optimizations or allow incorrect ones.
- **ALWAYS** test with `-passes=verify` before testing with optimization passes.
