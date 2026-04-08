---
name: add-gc-statepoints
description: Add garbage-collection support to an LLVM 22 IR frontend. Covers the shadow-stack (gcroot) model for simple GCs and the statepoint/gc.relocate model for moving/relocating collectors, including the RewriteStatepointsForGC pass and StackMap section.
---

# Skill: Add GC Support to an LLVM Frontend (LLVM 22)

Use this skill when your language runtime needs a garbage collector and you must tell LLVM where GC-safe points are and which pointers are live.

---

## Step 0 — Choose a model

| Model | Use when |
|-------|---------|
| **gcroot / shadow stack** | Simple stop-the-world GC, prototype, or interpreter. Easy to implement. |
| **Statepoints** | Moving/relocating collector (objects can be copied). Production use. |

For new language runtimes, prefer **statepoints** — gcroot is legacy and does not support moving objects.

---

## Model A: gcroot (shadow stack)

### Step A1 — Set the GC strategy on functions

```cpp
// Mark each function that manages GC pointers
F->setGC("shadow-stack");
```

### Step A2 — Mark GC root allocas

Every heap-pointer local variable needs a root alloca in the entry block:

```cpp
#include "llvm/IR/Intrinsics.h"

// Alloca for the GC root (always in entry block)
llvm::AllocaInst *Root = B.CreateAlloca(B.getPtrTy(), nullptr, "obj.root");
// Initialize to null before first use — GC may scan it before you store
B.CreateStore(llvm::ConstantPointerNull::get(B.getPtrTy()), Root);

// Declare llvm.gcroot
llvm::Function *GCRootFn = llvm::Intrinsic::getOrInsertDeclaration(
    M, llvm::Intrinsic::gcroot, {});

// Mark the alloca as a GC root (must call immediately after alloca)
B.CreateCall(GCRootFn,
    {Root, llvm::ConstantPointerNull::get(B.getPtrTy())});
```

### Step A3 — Use the root across GC points

```cpp
// Store the object pointer into the root before any call that may GC
B.CreateStore(ObjPtr, Root);

// ... calls that may trigger GC ...

// Reload after the GC point — the shadow-stack GC does NOT move objects,
// but reading from Root keeps the pattern consistent.
llvm::Value *LiveObj = B.CreateLoad(B.getPtrTy(), Root, "obj");
```

### Step A4 — Link the shadow-stack runtime

You must supply a shadow-stack runtime (or link against one). Minimal C structure:

```c
struct FrameMap  { int NumRoots; int NumMeta; const void *Meta[]; };
struct StackEntry { struct StackEntry *Next; const struct FrameMap *Map; void *Roots[]; };
extern struct StackEntry *llvm_gc_root_chain; // thread-local in production
```

---

## Model B: Statepoints (relocating GC)

### Step B1 — Set the GC strategy on functions

```cpp
F->setGC("statepoint-example"); // built-in test strategy
// Or register a custom strategy — see Step B5
```

### Step B2 — Use `RewriteStatepointsForGC` pass (recommended)

Let the pass automatically insert `gc.statepoint` + `gc.relocate` sequences around all calls:

```cpp
#include "llvm/Transforms/Scalar/RewriteStatepointsForGC.h"

// Add to your function pass pipeline before codegen:
FPM.addPass(llvm::RewriteStatepointsForGCPass());
```

The pass:
1. Finds all calls in functions with a GC strategy.
2. Computes live GC pointer sets at each call.
3. Rewrites calls to `gc.statepoint` + `gc.relocate` sequences.
4. Emits StackMap metadata.

After this pass, every GC pointer use after a call site is replaced with a `gc.relocate` result.

### Step B3 — Manual statepoint emission (advanced / understanding only)

If you emit statepoints manually instead of using the pass:

```llvm
; Statepoint wraps a call to @foo(i32 %x) with GC pointer %obj1 live:
%sp = call token @llvm.experimental.gc.statepoint.p0(
    i64 0,      ; call ID
    i32 0,      ; patch bytes
    ptr @foo,   ; callee
    i32 1,      ; # call args
    i32 0,      ; flags
    i32 %x,     ; call arg 0
    i32 0,      ; # transition args
    i32 0,      ; # deopt args
    ptr %obj1   ; GC live value (will appear in stack map)
)
%result = call i32  @llvm.experimental.gc.result.i32(token %sp)
%obj1.r = call ptr  @llvm.experimental.gc.relocate(token %sp, i32 7, i32 7)
; After the statepoint, always use %obj1.r — NOT %obj1
```

The C++ API equivalent:
```cpp
#include "llvm/IR/Statepoint.h"

// Use IRBuilder helpers or construct via StatepointFlags / makeStatepointCall
// In practice, always prefer RewriteStatepointsForGCPass
```

### Step B4 — Read the StackMap section

After compiling, the object file contains a `__llvm_stackmaps` section:

```bash
# Inspect the stack map
llvm-readobj --llvm-stackmap output.o
```

Your GC runtime reads this section at startup to discover the locations of all live pointers at each GC-safe point.

### Step B5 — Custom GC strategy (optional)

Register a custom strategy for code generation hooks:

```cpp
#include "llvm/CodeGen/GCStrategy.h"

class MyGCStrategy : public llvm::GCStrategy {
public:
  MyGCStrategy() {
    UseStatepoints = true;
    UsesMetadata   = false;
    NeededSafePoints = 0;
  }
};

static llvm::GCRegistry::Add<MyGCStrategy>
    X("my-gc", "My language GC strategy");
```

Functions with `gc "my-gc"` will use it during codegen.

---

## Verifying the output

```bash
# Compile the IR
llc -filetype=obj output.ll -o output.o

# Check the stack map section exists
llvm-readobj --llvm-stackmap output.o

# Disassemble to verify relocate sequences are present
llvm-dis output.bc
```

---

## Common mistakes

- **Never** use a pointer variable after a statepoint — always use the `gc.relocate` result. The original pointer is invalid if the GC moved the object.
- **Never** apply `RewriteStatepointsForGC` without first calling `F->setGC(...)` — the pass silently skips functions without a GC strategy.
- **Never** mix gcroot and statepoints in the same module — they are incompatible.
- **Always** initialize root allocas to `null` before the first `gcroot` call.
- **Always** reload GC roots after any call that may trigger collection (gcroot model).
- **Always** use `gc.relocate` for **every** live GC pointer across a statepoint — not just the ones you think are "interesting" to the optimizer.
