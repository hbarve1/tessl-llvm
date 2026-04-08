---
name: add-vectorization-hint
description: Guide LLVM 22's auto-vectorizer and SLP vectorizer from a frontend. Covers loop vectorization metadata, interleaving, loop distribution, marking parallel accesses, controlling SLP, and how to check whether vectorization happened.
---

# Skill: Add Vectorization Hints to LLVM 22 IR

Use this skill when your language has array/vector operations or counted loops that should run as SIMD code.

---

## Step 0 — How LLVM vectorizes

LLVM has two vectorizers:

| Vectorizer | What it does |
|------------|-------------|
| **Loop Vectorizer** | Converts scalar counted loops into SIMD loops; most impactful |
| **SLP Vectorizer** | Combines independent scalar instructions with the same pattern into one SIMD op |

Both run automatically in `-O2` / `-O3` pipelines. Your job is to:
1. Emit IR that the vectorizers can analyze (simple induction variables, no aliasing loads/stores).
2. Attach metadata hints to guide or force specific behavior.

---

## Step 1 — Basic loop structure for vectorizability

For the loop vectorizer to fire, the loop must have:
- A simple induction variable (`i = 0; i < N; i++`).
- No loop-carried dependencies (or provably no aliasing between loop-body accesses).
- A back-edge branch in the loop latch block.

Emit the canonical form:

```cpp
// for (int i = 0; i < N; i++) A[i] = B[i] + C[i];

llvm::BasicBlock *PreheaderBB = B.GetInsertBlock();
llvm::BasicBlock *HeaderBB  = llvm::BasicBlock::Create(Ctx, "loop.header", F);
llvm::BasicBlock *BodyBB    = llvm::BasicBlock::Create(Ctx, "loop.body",   F);
llvm::BasicBlock *ExitBB    = llvm::BasicBlock::Create(Ctx, "loop.exit",   F);

B.CreateBr(HeaderBB);

// Header: PHI for induction variable
B.SetInsertPoint(HeaderBB);
llvm::PHINode *I = B.CreatePHI(B.getInt32Ty(), 2, "i");
I->addIncoming(B.getInt32(0), PreheaderBB);
llvm::Value *Cond = B.CreateICmpSLT(I, N, "i.lt.n");
B.CreateCondBr(Cond, BodyBB, ExitBB);

// Body: A[i] = B[i] + C[i]
B.SetInsertPoint(BodyBB);
llvm::Value *BPtr = B.CreateGEP(B.getInt32Ty(), BBase, {I});
llvm::Value *CPtr = B.CreateGEP(B.getInt32Ty(), CBase, {I});
llvm::Value *APtr = B.CreateGEP(B.getInt32Ty(), ABase, {I});
llvm::Value *BVal = B.CreateLoad(B.getInt32Ty(), BPtr);
llvm::Value *CVal = B.CreateLoad(B.getInt32Ty(), CPtr);
B.CreateStore(B.CreateAdd(BVal, CVal), APtr);

// Latch: increment and jump back
llvm::Value *Next = B.CreateAdd(I, B.getInt32(1), "i.next");
I->addIncoming(Next, BodyBB);
// ← attach loop metadata to THIS branch (back-edge)
llvm::BranchInst *BackBr = B.CreateBr(HeaderBB);

B.SetInsertPoint(ExitBB);
```

---

## Step 2 — Attach loop vectorization metadata

Loop metadata attaches to the **back-edge branch** (the `br` that jumps back to the header):

```cpp
llvm::LLVMContext &Ctx = M.getContext();

// Force vectorization on
llvm::MDNode *VecEnable = llvm::MDNode::get(Ctx, {
    llvm::MDString::get(Ctx, "llvm.loop.vectorize.enable"),
    llvm::ConstantAsMetadata::get(
        llvm::ConstantInt::get(llvm::Type::getInt1Ty(Ctx), 1))
});

// Request width=8 (hint — actual width depends on target)
llvm::MDNode *VecWidth = llvm::MDNode::get(Ctx, {
    llvm::MDString::get(Ctx, "llvm.loop.vectorize.width"),
    llvm::ConstantAsMetadata::get(
        llvm::ConstantInt::get(llvm::Type::getInt32Ty(Ctx), 8))
});

// Interleave 2 iterations (software pipelining)
llvm::MDNode *Interleave = llvm::MDNode::get(Ctx, {
    llvm::MDString::get(Ctx, "llvm.loop.interleave.count"),
    llvm::ConstantAsMetadata::get(
        llvm::ConstantInt::get(llvm::Type::getInt32Ty(Ctx), 2))
});

// The loop MDNode — first operand MUST be the node itself
llvm::MDNode *LoopMD = llvm::MDNode::get(Ctx,
    {nullptr, VecEnable, VecWidth, Interleave});
LoopMD->replaceOperandWith(0, LoopMD); // self-reference

BackBr->setMetadata(llvm::LLVMContext::MD_loop, LoopMD);
```

---

## Step 3 — Disable vectorization for a loop

When your language semantics forbid reordering (e.g., loops with intentional side-effect ordering):

```cpp
llvm::MDNode *VecDisable = llvm::MDNode::get(Ctx, {
    llvm::MDString::get(Ctx, "llvm.loop.vectorize.enable"),
    llvm::ConstantAsMetadata::get(
        llvm::ConstantInt::get(llvm::Type::getInt1Ty(Ctx), 0))
});

llvm::MDNode *UnrollDisable = llvm::MDNode::get(Ctx, {
    llvm::MDString::get(Ctx, "llvm.loop.unroll.disable"),
    llvm::ConstantAsMetadata::get(
        llvm::ConstantInt::get(llvm::Type::getInt1Ty(Ctx), 1))
});

llvm::MDNode *LoopMD = llvm::MDNode::get(Ctx, {nullptr, VecDisable, UnrollDisable});
LoopMD->replaceOperandWith(0, LoopMD);
BackBr->setMetadata(llvm::LLVMContext::MD_loop, LoopMD);
```

---

## Step 4 — Mark accesses as parallel (no loop-carried deps)

If your language guarantees no aliasing between the loop's loads and stores (e.g., immutable source arrays):

```cpp
// Assign an access group to all loop memory accesses
llvm::MDNode *AccessGroup = llvm::MDNode::getDistinct(Ctx, {});
LI->setMetadata(llvm::LLVMContext::MD_access_group, AccessGroup);
SI->setMetadata(llvm::LLVMContext::MD_access_group, AccessGroup);

// Tell the loop that accesses in this group are parallel
llvm::MDNode *ParAccesses = llvm::MDNode::get(Ctx, {
    llvm::MDString::get(Ctx, "llvm.loop.parallel_accesses"),
    AccessGroup
});
// Include in the loop's MDNode alongside other hints
```

Alternatively, pass pointer parameters with `noalias` attribute — the vectorizer understands that too.

---

## Step 5 — Loop unrolling

```cpp
// Unroll exactly 4 times
llvm::MDNode *Unroll4 = llvm::MDNode::get(Ctx, {
    llvm::MDString::get(Ctx, "llvm.loop.unroll.count"),
    llvm::ConstantAsMetadata::get(
        llvm::ConstantInt::get(llvm::Type::getInt32Ty(Ctx), 4))
});

// Or: unroll fully (small loops)
llvm::MDNode *UnrollFull = llvm::MDNode::get(Ctx, {
    llvm::MDString::get(Ctx, "llvm.loop.unroll.full"),
});
```

---

## Step 6 — SLP vectorization hints

The SLP vectorizer runs automatically on straight-line code. Help it by:

1. Emitting a sequence of identical operations on adjacent memory:
```cpp
// v[0] = a[0] + b[0];
// v[1] = a[1] + b[1];
// v[2] = a[2] + b[2];
// v[3] = a[3] + b[3];
// SLP will combine these into one <4 x i32> add
```

2. Ensuring the pointers are based on a common base with constant GEP offsets.

3. Disabling SLP for a function if your language semantics require scalar execution:
```cpp
F->addFnAttr("no-vectorize-slp");
```

---

## Step 7 — Check if vectorization happened

```bash
# Run the vectorizer with remarks enabled
opt -passes="loop-vectorize" -pass-remarks=loop-vectorize \
    -pass-remarks-missed=loop-vectorize \
    -pass-remarks-analysis=loop-vectorize \
    input.ll -S -o out.ll 2>&1 | grep -E "vectorized|missed"

# Example output:
# remark: input.ll:10:3: vectorized loop (vectorization width: 8, ...)
# remark: input.ll:20:3: loop not vectorized: value cannot be identified ...
```

---

## Common loop metadata reference

| Key | Value | Effect |
|-----|-------|--------|
| `llvm.loop.vectorize.enable` | `i1 1` / `i1 0` | Force on / off |
| `llvm.loop.vectorize.width` | `i32 N` | Hint SIMD width |
| `llvm.loop.interleave.count` | `i32 N` | Software pipeline N iterations |
| `llvm.loop.unroll.count` | `i32 N` | Unroll N times |
| `llvm.loop.unroll.disable` | `i1 1` | Disable unrolling |
| `llvm.loop.unroll.full` | (empty) | Fully unroll (small loops) |
| `llvm.loop.distribute.enable` | `i1 1` | Split loop into vectorizable parts |
| `llvm.loop.parallel_accesses` | `MDNode*` | Mark accesses as independent |

---

## Common mistakes

- **Never** forget the self-referential first operand in loop MDNode — the verifier will silently drop the metadata.
- **Never** attach loop metadata to a non-back-edge branch — it has no effect.
- **Never** use `vectorize.enable=1` on loops with actual loop-carried dependencies — miscompilation.
- **Always** emit canonical induction variables — non-standard induction patterns block vectorization.
- **Always** use `noalias` on pointer parameters when your language can guarantee non-aliasing — this is often all that's needed to unlock vectorization.
