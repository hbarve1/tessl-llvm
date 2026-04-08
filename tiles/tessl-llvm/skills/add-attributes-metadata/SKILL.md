---
name: add-attributes-metadata
description: Add function/parameter attributes and IR metadata to LLVM 22 IR to unlock optimizer opportunities. Covers NoUnwind, ReadNone, NoCapture, Noundef, loop vectorization hints, branch weights, TBAA, !range, and !nonnull.
---

# Skill: Add Attributes and Metadata to LLVM 22 IR

Use this skill to annotate your generated IR so the optimizer can make better decisions — inlining, alias analysis, vectorization, branch layout, and more.

---

## Step 0 — Why this matters

Correct attributes are one of the highest-ROI improvements you can make to a frontend:

- `nounwind` eliminates EH cleanup code on every call.
- `nocapture` on parameters enables escape analysis and alias analysis.
- `noundef` enables constant folding and value-range propagation.
- Loop metadata controls auto-vectorization and unrolling.

---

## Step 1 — Function attributes

Set once per function, before or after emitting its body:

```cpp
// Never throws — most important; enables call-site EH cleanup elision
F->addFnAttr(llvm::Attribute::NoUnwind);

// Reads no memory (pure function)
F->addFnAttr(llvm::Attribute::ReadNone);

// Reads memory but never writes
F->addFnAttr(llvm::Attribute::ReadOnly);

// Only accesses memory via its own arguments (no globals)
F->addFnAttr(llvm::Attribute::ArgMemOnly);

// Does not recurse (direct or indirect)
F->addFnAttr(llvm::Attribute::NoRecurse);

// Always returns (no infinite loops, no exit() calls)
F->addFnAttr(llvm::Attribute::WillReturn);

// Never returns (e.g., panic(), abort()) — always emit unreachable after call
F->addFnAttr(llvm::Attribute::NoReturn);

// Rarely called — deprioritize in layout
F->addFnAttr(llvm::Attribute::Cold);

// On a call site (not function definition):
CI->addFnAttr(llvm::Attribute::NoUnwind);
```

---

## Step 2 — Parameter and return attributes

```cpp
llvm::AttrBuilder AB(Ctx);

// Pointer not stored past the call (enables escape analysis)
AB.addAttribute(llvm::Attribute::NoCapture);

// Pointer does not alias any other pointer in the call (like C restrict)
AB.addAttribute(llvm::Attribute::NoAlias);

// Pointer is never null
AB.addAttribute(llvm::Attribute::NonNull);

// Safe to dereference N bytes through this pointer
AB.addDereferenceableAttr(8);

// Value is never undef or poison
AB.addAttribute(llvm::Attribute::NoUndef);

// Attach to parameter index 0 (0-indexed)
F->addParamAttrs(0, AB);

// Return attributes (e.g., returned pointer is never null)
llvm::AttrBuilder RetAB(Ctx);
RetAB.addAttribute(llvm::Attribute::NonNull);
RetAB.addDereferenceableAttr(16);
F->addRetAttrs(RetAB);
```

### Quick reference — most impactful parameter attributes

| Attribute | Impact |
|-----------|--------|
| `NoCapture` | Enables escape analysis; critical for alias analysis quality |
| `NoAlias` | Like `restrict` — unlocks reordering and vectorization |
| `NonNull` | Eliminates null checks on the parameter |
| `Noundef` | Enables more aggressive constant propagation |
| `Dereferenceable(N)` | Allows speculative loads without null checks |

---

## Step 3 — Loop vectorization and unroll metadata

Attach `!llvm.loop` to the **back-edge branch** (terminator of the loop latch block):

```cpp
#include "llvm/IR/MDBuilder.h"

llvm::LLVMContext &Ctx = M.getContext();

// Back-edge branch of the loop
llvm::BranchInst *BackBr =
    llvm::cast<llvm::BranchInst>(LoopLatchBB->getTerminator());

// Build hint nodes
llvm::MDNode *VecEnable = llvm::MDNode::get(Ctx, {
    llvm::MDString::get(Ctx, "llvm.loop.vectorize.enable"),
    llvm::ConstantAsMetadata::get(
        llvm::ConstantInt::get(llvm::Type::getInt1Ty(Ctx), 1))
});

llvm::MDNode *UnrollCount = llvm::MDNode::get(Ctx, {
    llvm::MDString::get(Ctx, "llvm.loop.unroll.count"),
    llvm::ConstantAsMetadata::get(
        llvm::ConstantInt::get(llvm::Type::getInt32Ty(Ctx), 4))
});

// Build the loop MDNode — first operand MUST be the node itself (self-reference)
llvm::MDNode *LoopMD = llvm::MDNode::get(Ctx, {nullptr, VecEnable, UnrollCount});
LoopMD->replaceOperandWith(0, LoopMD); // replace placeholder with self

BackBr->setMetadata(llvm::LLVMContext::MD_loop, LoopMD);
```

### Common loop hints

| Key | Value | Effect |
|-----|-------|--------|
| `llvm.loop.vectorize.enable` | `i1 1` | Force vectorization on |
| `llvm.loop.vectorize.enable` | `i1 0` | Disable vectorization |
| `llvm.loop.vectorize.width` | `i32 N` | Hint SIMD width |
| `llvm.loop.unroll.count` | `i32 N` | Unroll N times |
| `llvm.loop.unroll.disable` | `i1 1` | Disable unrolling |
| `llvm.loop.interleave.count` | `i32 N` | Interleave N iterations |

---

## Step 4 — Branch weights

Guide block layout and branch prediction:

```cpp
#include "llvm/IR/MDBuilder.h"

llvm::MDBuilder MDB(Ctx);

// CondBr weights: TrueWeight : FalseWeight
llvm::MDNode *Weights = MDB.createBranchWeights(/*True=*/100, /*False=*/1);
llvm::BranchInst *Br = B.CreateCondBr(Cond, ThenBB, ElseBB);
Br->setMetadata(llvm::LLVMContext::MD_prof, Weights);

// SwitchInst: one weight per case including default (order: default first)
llvm::MDNode *SwitchW = MDB.createBranchWeights({10, 90, 5});
SI->setMetadata(llvm::LLVMContext::MD_prof, SwitchW);
```

---

## Step 5 — TBAA (type-based alias analysis)

TBAA lets the optimizer prove that accesses to different types cannot alias:

```cpp
#include "llvm/IR/MDBuilder.h"

llvm::MDBuilder MDB(Ctx);

// Create a type tree (do once, cache the nodes)
llvm::MDNode *Root    = MDB.createTBAARoot("MyLang TBAA");
llvm::MDNode *IntTy   = MDB.createTBAANode("int",   Root);
llvm::MDNode *FloatTy = MDB.createTBAANode("float", Root);

// Create access tags (type, type, offset)
llvm::MDNode *IntTag   = MDB.createTBAAStructTagNode(IntTy,   IntTy,   0);
llvm::MDNode *FloatTag = MDB.createTBAAStructTagNode(FloatTy, FloatTy, 0);

// Tag loads/stores
llvm::LoadInst  *LI = B.CreateLoad(B.getInt32Ty(), IntPtr);
LI->setMetadata(llvm::LLVMContext::MD_tbaa, IntTag);

llvm::StoreInst *SI = B.CreateStore(FVal, FloatPtr);
SI->setMetadata(llvm::LLVMContext::MD_tbaa, FloatTag);
// Optimizer now knows: int stores can never alias float stores
```

---

## Step 6 — `!range`, `!nonnull`, `!align`

```cpp
// !range — loaded integer is in [lo, hi)
llvm::Type *I32 = B.getInt32Ty();
llvm::MDNode *Range = llvm::MDNode::get(Ctx, {
    llvm::ConstantAsMetadata::get(llvm::ConstantInt::get(I32, 0)),
    llvm::ConstantAsMetadata::get(llvm::ConstantInt::get(I32, 256))
});
LI->setMetadata(llvm::LLVMContext::MD_range, Range);

// !nonnull — loaded pointer is never null
LI->setMetadata(llvm::LLVMContext::MD_nonnull,
                llvm::MDNode::get(Ctx, {}));

// !align / setAlignment — pointer alignment (prefer setAlignment)
LI->setAlignment(llvm::Align(8));
```

---

## Recommended attribute checklist per function type

| Function kind | Attributes to set |
|---|---|
| Pure math function (no I/O) | `nounwind`, `readnone`, `willreturn` |
| Read-only accessor | `nounwind`, `readonly` |
| Allocator returning new memory | `nounwind`, `noalias` on return |
| Panic / error handler | `nounwind`, `noreturn` |
| Pointer parameter not stored | `nocapture` on param |
| Pointer param not null | `nonnull` on param |

---

## Common mistakes

- **Never** set `readnone` or `readonly` on functions that actually write memory — the optimizer will incorrectly eliminate writes.
- **Never** set `nounwind` on functions that call `longjmp` or throw — EH tables will be silently omitted.
- **Never** forget the self-referential first operand in loop metadata — the MDNode must point to itself.
- **Always** set `noundef` on values you know are fully initialized — it enables value-range propagation.
- **Always** use `nocapture` on pointer parameters your language guarantees don't escape — it is the most impactful single attribute for alias analysis.
