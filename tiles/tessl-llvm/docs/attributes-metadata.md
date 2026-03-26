# Function Attributes, Parameter Attributes & IR Metadata (LLVM 20)

Reference: [LangRef Attributes](https://llvm.org/docs/LangReference.html#function-attributes) | [LangRef Metadata](https://llvm.org/docs/LangReference.html#metadata)

---

## Function attributes

Function attributes tell the optimizer what a function is guaranteed to do or not do.
They live on `Function` and on `CallInst`/`InvokeInst`.

### Setting attributes in C++

```cpp
// On a Function definition:
F->addFnAttr(llvm::Attribute::NoUnwind);     // function never throws
F->addFnAttr(llvm::Attribute::ReadNone);     // reads no memory (pure)
F->addFnAttr(llvm::Attribute::ReadOnly);     // reads memory but doesn't write
F->addFnAttr(llvm::Attribute::NoRecurse);    // does not recurse
F->addFnAttr(llvm::Attribute::AlwaysInline); // always inline at call sites
F->addFnAttr(llvm::Attribute::NoInline);     // never inline
F->addFnAttr(llvm::Attribute::Cold);         // rarely called

// On a call site:
CI->addFnAttr(llvm::Attribute::NoUnwind);

// Parameterized attributes:
F->addFnAttr(llvm::Attribute::getWithAlignmentValue(Ctx, 16)); // alignstack=16
```

### Key function attributes for language implementers

| Attribute | LLVM IR spelling | Meaning |
|-----------|-----------------|---------|
| `NoUnwind` | `nounwind` | Never throws / unwinds. Required for optimizer to elide EH cleanup. |
| `ReadNone` | `readnone` | Reads and writes no memory. Enables CSE and code motion. |
| `ReadOnly` | `readonly` | Reads memory but never writes. Enables CSE. |
| `WriteOnly` | `writeonly` | Writes memory but never reads it. |
| `ArgMemOnly` | `argmemonly` | Only accesses memory through its arguments. |
| `InaccessibleMemOnly` | `inaccessiblememonly` | Only accesses memory not accessible by the caller. |
| `NoRecurse` | `norecurse` | Does not call itself directly or indirectly. |
| `WillReturn` | `willreturn` | Always returns (no infinite loops, no calls to `exit`). |
| `NoReturn` | `noreturn` | Never returns (e.g., `exit()`, `throw`). Emit `unreachable` after the call. |
| `Speculatable` | `speculatable` | Safe to speculate (no side effects observable if not executed). |
| `AlwaysInline` | `alwaysinline` | Always inline. Overrides size/cost heuristics. |
| `NoInline` | `noinline` | Never inline. |
| `OptimizeNone` | `optnone` | Skip all optimization passes for this function. |
| `Cold` | `cold` | Rarely executed; deprioritize in layout. |
| `Hot` | `hot` | Frequently executed; prioritize in layout. |
| `NoFree` | `nofree` | Does not call `free` (or equivalent). |
| `NoSync` | `nosync` | Does not synchronize with other threads. |

---

## Parameter and return attributes

```cpp
// Parameter attributes — on Argument or call operand
llvm::AttrBuilder ParamAB(Ctx);
ParamAB.addAttribute(llvm::Attribute::NoCapture);   // ptr not stored past call
ParamAB.addAttribute(llvm::Attribute::NoAlias);     // ptr doesn't alias others
ParamAB.addAttribute(llvm::Attribute::NonNull);     // ptr is never null
ParamAB.addDereferenceableAttr(8);                  // safe to deref 8 bytes
ParamAB.addAttribute(llvm::Attribute::ReadOnly);    // param ptr only read

// Attach to function parameter (0-indexed)
F->addParamAttrs(0, ParamAB);

// Return attributes
llvm::AttrBuilder RetAB(Ctx);
RetAB.addAttribute(llvm::Attribute::NonNull);       // returned ptr never null
RetAB.addDereferenceableAttr(16);
F->addRetAttrs(RetAB);
```

### Key parameter attributes

| Attribute | Meaning |
|-----------|---------|
| `NoCapture` | The pointer is not stored anywhere that outlives the call. Enables escape analysis. |
| `NoAlias` | The pointer does not alias any other pointer accessible by the function. Like C `restrict`. |
| `NonNull` | Pointer argument is guaranteed non-null. |
| `Dereferenceable(N)` | Pointer is safe to dereference for N bytes. |
| `DereferenceableOrNull(N)` | Either null, or dereferenceable for N bytes. |
| `Align(N)` | Pointer is aligned to N bytes. |
| `ReadOnly` | Pointer argument is only read through, not written. |
| `WriteOnly` | Pointer argument is only written to. |
| `Noundef` | Value is never `undef` or `poison`. Enables more aggressive opts. |
| `Returned` | Function returns this argument unchanged (e.g., `self` in fluent APIs). |

---

## Loop metadata (`!llvm.loop`)

Attach loop metadata to the branch that closes the loop (the back-edge terminator):

```cpp
// Get the back-edge branch instruction (the `br` at the end of the loop body)
BranchInst *BackBr = cast<BranchInst>(LoopBB->getTerminator());

// Build metadata node
llvm::LLVMContext &Ctx = M.getContext();

// Self-referential loop ID (required by LLVM loop metadata format)
llvm::MDNode *LoopID = llvm::MDNode::getTemporary(Ctx, {}).release();
llvm::Metadata *LoopIDRef = llvm::MDNode::get(Ctx, {LoopID});

// Vectorize hint
llvm::MDNode *VecEnable = llvm::MDNode::get(Ctx, {
    llvm::MDString::get(Ctx, "llvm.loop.vectorize.enable"),
    llvm::ConstantAsMetadata::get(llvm::ConstantInt::get(
        llvm::Type::getInt1Ty(Ctx), 1))
});

// Unroll count
llvm::MDNode *UnrollCount = llvm::MDNode::get(Ctx, {
    llvm::MDString::get(Ctx, "llvm.loop.unroll.count"),
    llvm::ConstantAsMetadata::get(llvm::ConstantInt::get(
        llvm::Type::getInt32Ty(Ctx), 4))
});

// Combine: first element must be the loop ID itself
llvm::MDNode *LoopMD = llvm::MDNode::get(Ctx, {LoopIDRef, VecEnable, UnrollCount});
// Replace the self-reference
LoopMD->replaceOperandWith(0, LoopMD);

BackBr->setMetadata(llvm::LLVMContext::MD_loop, LoopMD);
```

### Common loop hints

| Metadata key | Values | Effect |
|-------------|--------|--------|
| `llvm.loop.vectorize.enable` | `i1 1` / `i1 0` | Force enable/disable auto-vectorization |
| `llvm.loop.vectorize.width` | `i32 N` | Hint vector width |
| `llvm.loop.unroll.count` | `i32 N` | Unroll N times |
| `llvm.loop.unroll.enable` | `i1 1` | Force unrolling |
| `llvm.loop.unroll.disable` | `i1 1` | Disable unrolling |
| `llvm.loop.interleave.count` | `i32 N` | Interleave N iterations |
| `llvm.loop.distribute.enable` | `i1 1` | Enable loop distribution |
| `llvm.loop.parallel_accesses` | `MDNode *` | Mark accesses as parallel (no loop-carried deps) |

---

## Branch weights (`!prof`)

Guide branch prediction and block layout:

```cpp
// CreateCondBr with branch weights (taken : not-taken)
llvm::MDBuilder MDB(Ctx);
llvm::MDNode *Weights = MDB.createBranchWeights(/*TrueWeight=*/100,
                                                 /*FalseWeight=*/1);
BranchInst *Br = B.CreateCondBr(Cond, ThenBB, ElseBB);
Br->setMetadata(llvm::LLVMContext::MD_prof, Weights);
```

```cpp
// SwitchInst branch weights (one weight per case + default)
llvm::MDNode *SwitchWeights = MDB.createBranchWeights({10, 90, 5});
SI->setMetadata(llvm::LLVMContext::MD_prof, SwitchWeights);
```

---

## TBAA (type-based alias analysis)

TBAA metadata lets the optimizer distinguish accesses to different types:

```cpp
llvm::MDBuilder MDB(Ctx);

// Create a TBAA type tree
llvm::MDNode *Root   = MDB.createTBAARoot("MyLang TBAA");
llvm::MDNode *IntTy  = MDB.createTBAANode("int",  Root);
llvm::MDNode *FloatTy= MDB.createTBAANode("float",Root);

// Tag a load/store with TBAA type
llvm::MDNode *IntTag  = MDB.createTBAAStructTagNode(IntTy,  IntTy,  0);
llvm::MDNode *FloatTag= MDB.createTBAAStructTagNode(FloatTy,FloatTy,0);

LoadInst  *LI = B.CreateLoad(B.getInt32Ty(), IntPtr);
LI->setMetadata(llvm::LLVMContext::MD_tbaa, IntTag);

StoreInst *SI = B.CreateStore(FVal, FloatPtr);
SI->setMetadata(llvm::LLVMContext::MD_tbaa, FloatTag);
// Now the optimizer knows int and float stores can't alias
```

---

## `!range` — integer value range hint

```cpp
// Tell optimizer that a loaded value is in [0, 256)
llvm::MDNode *Range = llvm::MDNode::get(Ctx, {
    llvm::ConstantAsMetadata::get(llvm::ConstantInt::get(I32, 0)),
    llvm::ConstantAsMetadata::get(llvm::ConstantInt::get(I32, 256))
});
LI->setMetadata(llvm::LLVMContext::MD_range, Range);
```

---

## `!nonnull` — pointer non-null hint

```cpp
// Assert loaded pointer is never null
LI->setMetadata(llvm::LLVMContext::MD_nonnull,
                llvm::MDNode::get(Ctx, {}));
```

---

## `!align` — load/store alignment hint

```cpp
// Assert pointer alignment of 8 bytes on a load
LI->setMetadata(llvm::LLVMContext::MD_align,
    llvm::MDNode::get(Ctx, {
        llvm::ConstantAsMetadata::get(
            llvm::ConstantInt::get(I64, 8))}));
// Alternatively use setAlignment() on the instruction:
LI->setAlignment(llvm::Align(8));
```

---

## Common mistakes

- **Do NOT** set `ReadNone` or `ReadOnly` on functions that actually write memory — this causes the optimizer to incorrectly eliminate stores or hoist loads.
- **Do NOT** set `NoUnwind` on functions that may throw or call `longjmp` — exception tables will be missing.
- **Do NOT** forget to make loop metadata self-referential — the first operand of the loop MDNode must be the node itself.
- **ALWAYS** set `Noundef` on values you know are fully initialized — it enables more aggressive constant propagation and dead code elimination.
- **ALWAYS** use `NoCapture` on pointer parameters that your language guarantees don't escape — it is one of the most impactful attributes for alias analysis.
