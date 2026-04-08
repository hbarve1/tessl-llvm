---
name: add-sanitizer
description: Instrument LLVM 22 IR for AddressSanitizer (ASan), UndefinedBehaviorSanitizer (UBSan), or ThreadSanitizer (TSan). Covers enabling sanitizers via TargetMachine/PassBuilder, adding manual shadow-memory checks, inserting UBSan runtime calls, and CMake setup.
---

# Skill: Add Sanitizer Instrumentation (LLVM 22)

Use this skill when you want to add memory-safety, undefined-behavior, or data-race detection to your language runtime or compiler output.

---

## Step 0 — Approach options

| Approach | Use when |
|----------|---------|
| **Pass pipeline sanitizers** (ASan/TSan/MSan/UBSan) | Your compiler drives the full pipeline; easiest — just enable the pass |
| **Manual IR instrumentation** | You only want to check specific operations (custom UBSan-style checks) |
| **Link against sanitizer runtime** | Your output must be compatible with Clang's sanitizer runtimes |

---

## Part 1: Enable ASan/TSan/UBSan via PassBuilder (recommended)

The cleanest approach for a compiler that owns its optimization pipeline:

```cpp
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Transforms/Instrumentation/AddressSanitizer.h"
#include "llvm/Transforms/Instrumentation/ThreadSanitizer.h"

llvm::PassBuilder PB;
llvm::ModuleAnalysisManager MAM;
llvm::CGSCCAnalysisManager CGAM;
llvm::FunctionAnalysisManager FAM;
llvm::LoopAnalysisManager LAM;
PB.registerModuleAnalyses(MAM);
PB.registerCGSCCAnalyses(CGAM);
PB.registerFunctionAnalyses(FAM);
PB.registerLoopAnalyses(LAM);
PB.crossRegisterProxies(LAM, FAM, CGAM, MAM);

llvm::ModulePassManager MPM;

// --- AddressSanitizer ---
llvm::AddressSanitizerOptions AsanOpts;
// Instrument module-level globals
MPM.addPass(llvm::ModuleAddressSanitizerPass(AsanOpts,
    /*UseGlobalGC=*/false, /*UseOdrIndicator=*/true,
    llvm::AsanDtorKind::Global));
// Per-function instrumentation
llvm::FunctionPassManager FPM;
FPM.addPass(llvm::AddressSanitizerPass(AsanOpts));
MPM.addPass(llvm::createModuleToFunctionPassAdaptor(std::move(FPM)));

// --- ThreadSanitizer ---
// MPM.addPass(llvm::ModuleThreadSanitizerPass());
// FPM.addPass(llvm::ThreadSanitizerPass());

MPM.run(*M, MAM);
```

Then link the sanitizer runtime at link time:
```bash
clang output.o -fsanitize=address -o prog
# or link manually: -lasan
```

---

## Part 2: Mark functions for sanitizer control

```cpp
// Disable sanitizer for a specific function (e.g., runtime internals):
F->addFnAttr(llvm::Attribute::SanitizeAddress);    // opt-in to ASan
F->addFnAttr(llvm::Attribute::DisableSanitizerInstrumentation); // opt-out

// Equivalently, the LLVM IR attributes:
// define void @foo() sanitize_address { ... }
// define void @bar() no_sanitize_address { ... }
```

---

## Part 3: Manual UBSan-style runtime checks

Insert explicit checks for undefined behavior your language defines (e.g., integer overflow, null dereference, out-of-bounds):

```cpp
// --- Null pointer check before a load ---
llvm::BasicBlock *OkBB     = llvm::BasicBlock::Create(Ctx, "ptr.ok", F);
llvm::BasicBlock *TrapBB   = llvm::BasicBlock::Create(Ctx, "ptr.null", F);

llvm::Value *IsNull = B.CreateIsNull(Ptr, "is.null");
B.CreateCondBr(IsNull, TrapBB, OkBB);

// Trap block: call the runtime handler or llvm.trap
B.SetInsertPoint(TrapBB);
llvm::Function *TrapFn = llvm::Intrinsic::getOrInsertDeclaration(
    M, llvm::Intrinsic::trap);
B.CreateCall(TrapFn);
B.CreateUnreachable(); // terminates the block

B.SetInsertPoint(OkBB);
// ... safe to load here ...
```

```cpp
// --- Signed integer overflow check (add) ---
// Use llvm.sadd.with.overflow.*
llvm::Function *SAddOvf = llvm::Intrinsic::getOrInsertDeclaration(
    M, llvm::Intrinsic::sadd_with_overflow, {B.getInt32Ty()});

llvm::Value *Result = B.CreateCall(SAddOvf, {LHS, RHS});
llvm::Value *Sum     = B.CreateExtractValue(Result, 0, "sum");
llvm::Value *Overflowed = B.CreateExtractValue(Result, 1, "ovf");

B.CreateCondBr(Overflowed, TrapBB, OkBB);
```

---

## Part 4: Use `llvm.ubsantrap` for UBSan-style aborts

LLVM 22 provides a dedicated UBSan trap intrinsic that carries an error kind:

```cpp
// llvm.ubsantrap(i8 kind) — kind maps to UBSan error codes
// Common kinds: 0=add-overflow, 1=sub-overflow, 2=mul-overflow,
//               11=null-ptr-deref, 23=bounds
llvm::Function *UBTrap = llvm::Intrinsic::getOrInsertDeclaration(
    M, llvm::Intrinsic::ubsantrap);

B.SetInsertPoint(TrapBB);
B.CreateCall(UBTrap, {B.getInt8(11)}); // 11 = null-pointer-deref
B.CreateUnreachable();
```

---

## CMake: link against sanitizer runtimes

```cmake
# If using Clang as the final linker, just pass sanitizer flags:
target_link_options(MyCompiler PRIVATE -fsanitize=address)
target_compile_options(MyCompiler PRIVATE -fsanitize=address)

# To compile the sanitized output with Clang:
# clang -fsanitize=address -O1 output.ll -o prog
```

---

## Verifying instrumentation

```bash
# Run ASan-instrumented binary
ASAN_OPTIONS=detect_leaks=1 ./prog

# Check what instrumentation was inserted (dump IR after passes)
opt -passes="asan-module,asan" -S output.ll -o output.asan.ll
cat output.asan.ll | grep "__asan"
```

---

## Common mistakes

- **Never** apply sanitizer passes before inlining — inline first so sanitizer sees the combined code.
- **Never** mix sanitized and non-sanitized objects without understanding which runtime is linked.
- **Always** add `unreachable` after `llvm.trap` / `llvm.ubsantrap` calls — two terminators in a block is a verifier error.
- **Always** disable sanitizer instrumentation on your GC runtime internals and shadow-stack code — sanitizing the sanitizer causes false positives.
- **Always** verify the module after inserting checks (`verifyModule(*M, &errs())`).
