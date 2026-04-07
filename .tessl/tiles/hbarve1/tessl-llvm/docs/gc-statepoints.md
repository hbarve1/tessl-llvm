# Garbage Collection & Statepoints (LLVM 22)

Reference: [GarbageCollection](https://llvm.org/docs/GarbageCollection.html) | [Statepoints](https://llvm.org/docs/Statepoints.html) | [StackMaps](https://llvm.org/docs/StackMaps.html)

---

## Overview: two GC models

| Model | Mechanism | Use when |
|-------|----------|---------|
| **gcroot (legacy)** | `llvm.gcroot` intrinsic, shadow stack | Simple stop-the-world collectors; easy to implement |
| **Statepoints (modern)** | `gc.statepoint` / `gc.relocate` / `gc.result` | Precise, relocating GCs (moving collectors); production use |

For new language runtimes, use **statepoints** — they support moving collectors and are what major GC'd runtimes (e.g., Java via Graal, Julia) use.

---

## Model 1: gcroot (legacy shadow stack)

`llvm.gcroot` marks a local variable as a GC root. The compiler inserts it into a shadow stack that the GC can walk.

### Declare the GC strategy

```cpp
// Attach GC strategy name to the function
F->setGC("shadow-stack");  // built-in shadow-stack GC
// or a custom strategy registered with llvm::GCRegistry
```

### Mark GC roots

```cpp
// For each heap pointer local variable, create an alloca and mark as gcroot
// The alloca type must be a pointer (ptr in LLVM 22)
AllocaInst *Root = B.CreateAlloca(B.getPtrTy(), nullptr, "obj.root");
B.CreateStore(llvm::ConstantPointerNull::get(B.getPtrTy()), Root); // init to null

// Declare llvm.gcroot intrinsic
llvm::Function *GCRoot = llvm::Intrinsic::getOrInsertDeclaration(
    M, llvm::Intrinsic::gcroot, {});

// Mark the alloca as a GC root
// Arg 0: ptr to the alloca (must be a ptr-to-ptr in shadow-stack model)
// Arg 1: metadata ptr (type descriptor or null)
B.CreateCall(GCRoot,
    {Root, llvm::ConstantPointerNull::get(B.getPtrTy())});

// Store object pointer into root before any call that may trigger GC
B.CreateStore(ObjPtr, Root);

// Load from root after any GC point (object may have moved — shadow-stack GC)
Value *LiveObj = B.CreateLoad(B.getPtrTy(), Root, "obj");
```

### Shadow stack runtime support

You must link against (or implement) the shadow stack runtime:
```c
// Minimal C implementation of shadow stack GC support
struct FrameMap { int NumRoots; int NumMeta; const void *Meta[]; };
struct StackEntry { struct StackEntry *Next; const struct FrameMap *Map;
                    void *Roots[]; };
struct StackEntry *llvm_gc_root_chain; // thread-local in practice
```

---

## Model 2: Statepoints (relocating GC)

Statepoints allow a **moving GC** — the collector can relocate objects, and LLVM ensures live pointers are updated through `gc.relocate`.

### Concepts

```
gc.statepoint   — a call site annotated as a GC-safe point
gc.result       — extracts the call return value from a statepoint
gc.relocate     — provides the post-GC address of a pointer that may have moved
```

At each `gc.statepoint`, the compiler generates a **stack map** entry recording:
- The live GC-managed pointers at that call site
- Their locations (register or stack slot)

The GC runtime reads the stack map to find and update all live pointers.

### Emitting statepoints

In IR, a call through a statepoint looks like:

```llvm
; Normal call:
%r = call i32 @foo(i32 %x)

; Statepoint equivalent:
%sp = call token @llvm.experimental.gc.statepoint.p0(
    i64 0,          ; ID (arbitrary, for correlation with runtime)
    i32 0,          ; # patch bytes (0 for normal calls)
    ptr @foo,       ; callee
    i32 1,          ; # call args
    i32 0,          ; flags
    i32 %x,         ; call arg 0
    i32 0,          ; # transition args
    i32 0,          ; # deopt args
    ptr %obj1,      ; GC live value 0 (will be in stack map)
    ptr %obj2       ; GC live value 1
)
%r      = call i32  @llvm.experimental.gc.result.i32(token %sp)
%obj1.r = call ptr  @llvm.experimental.gc.relocate(token %sp, i32 7, i32 7)
          ; args: statepoint token, base index, derived index
          ; obj1 is at position 7 (after the fixed statepoint args)
%obj2.r = call ptr  @llvm.experimental.gc.relocate(token %sp, i32 8, i32 8)
```

After the statepoint, use `%obj1.r` and `%obj2.r` — not the original `%obj1` / `%obj2` — as the GC may have moved them.

### Using RewriteStatepointsForGC pass

Manually inserting statepoints is tedious. Use the `RewriteStatepointsForGC` pass to transform ordinary calls into statepoints automatically:

```cpp
#include "llvm/Transforms/Scalar/RewriteStatepointsForGC.h"

// Add to your pass pipeline (must run before codegen):
FPM.addPass(llvm::RewriteStatepointsForGCPass());
```

This pass:
1. Finds all calls in functions with a GC strategy set.
2. Computes the live GC pointer set at each call.
3. Rewrites calls to `gc.statepoint` + `gc.relocate` sequences.

**Prerequisite:** Mark your GC-managed pointer type with `gc` in the IR:

```cpp
// Set GC strategy on each function that manages GC pointers
F->setGC("statepoint-example"); // or your custom strategy name
```

### StackMap section

After compilation, the object file contains a `__llvm_stackmaps` section with a binary encoding of all statepoint locations and live pointer offsets. Your GC reads this at runtime.

```bash
# Inspect the stack map
llvm-readobj --llvm-stackmap output.o
```

---

## Custom GC strategy (plugin)

Register a custom GC strategy for code generation hooks:

```cpp
#include "llvm/CodeGen/GCStrategy.h"
#include "llvm/CodeGen/GCMetadata.h"

class MyGCStrategy : public llvm::GCStrategy {
public:
  MyGCStrategy() {
    UseStatepoints = true;       // use statepoint model
    UsesMetadata   = false;
    NeededSafePoints = 0;
  }
};

static llvm::GCRegistry::Add<MyGCStrategy> X("my-gc",
    "My language GC strategy");
```

Link the plugin into your compiler; functions with `gc "my-gc"` will use it.

---

## Choosing between gcroot and statepoints

| Criteria | gcroot | Statepoints |
|----------|--------|-------------|
| GC can move objects? | No (shadow stack copies, doesn't relocate) | Yes |
| Complexity of IR | Simple | Complex (gc.relocate for every live pointer) |
| Handled by optimizer? | Limited | Full (RewriteStatepointsForGC automates it) |
| Production use | Rare (simple interpreters) | Yes (Julia, Graal, etc.) |
| LLVM 22 status | Legacy, not recommended for new code | Recommended |

---

## Common mistakes

- **Do NOT** use `%obj` after a statepoint — always use the `gc.relocate` result `%obj.r`. Using the pre-statepoint pointer is undefined behavior in a moving GC.
- **Do NOT** apply `RewriteStatepointsForGC` without first setting `F->setGC(...)` on functions that manage GC pointers — it silently ignores functions without a GC strategy.
- **Do NOT** mix gcroot and statepoints in the same module — they are incompatible models.
- **ALWAYS** initialize root allocas to null before the first `llvm.gcroot` call — the GC may read them before you store an object.
- **ALWAYS** use `gc.relocate` for every live GC pointer across a statepoint, including pointers that appear dead to the optimizer but alive to the GC.
