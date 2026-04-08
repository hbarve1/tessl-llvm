---
name: add-lto
description: Enable Link-Time Optimization (LTO and ThinLTO) for an LLVM 22-based compiler. Covers full LTO vs ThinLTO trade-offs, bitcode emission from a frontend, LLD integration, out-of-tree whole-program optimization via ModulePassManager, and common debugging tips.
---

# Skill: Enable LTO / ThinLTO (LLVM 22)

Use this skill when you want cross-module optimization — inlining, devirtualization, dead-code elimination, or whole-program alias analysis — across separately compiled translation units.

---

## Step 0 — Understand the two models

| | Full LTO | ThinLTO |
|---|---|---|
| How it works | All bitcode merged into one giant module, then optimized | Per-module summaries at compile time; parallel backends at link time |
| Memory use | Very high (all IR in RAM at once) | Much lower (only imported pieces) |
| Link time | Slow for large programs | Fast, parallelism-friendly |
| Optimization quality | Maximum (whole-program view) | Close to full LTO for most workloads |
| When to use | Small-to-medium programs, maximum perf | Large codebases (Chromium-scale) |

---

## Model A: Using Clang as the driver (most frontends)

If your compiler emits bitcode files (`.bc`) or hands off to Clang for linking:

### Step A1 — Emit bitcode from your frontend

```cpp
#include "llvm/Bitcode/BitcodeWriter.h"

// After all IR is generated and verified:
std::error_code EC;
llvm::raw_fd_ostream OS("output.bc", EC, llvm::sys::fs::OF_None);
if (EC) { llvm::errs() << "Error: " << EC.message() << "\n"; return 1; }

llvm::WriteBitcodeToFile(*M, OS);
```

### Step A2 — Full LTO via Clang/LLD

```bash
# Compile step: emit bitcode-containing objects
clang -flto -O2 -c a.ll -o a.o
clang -flto -O2 -c b.ll -o b.o

# Link step: LLD does LTO internally
clang -flto -O2 -fuse-ld=lld a.o b.o -o prog
```

### Step A3 — ThinLTO via Clang/LLD

```bash
clang -flto=thin -O2 -c a.ll -o a.o
clang -flto=thin -O2 -c b.ll -o b.o
clang -flto=thin -O2 -fuse-ld=lld a.o b.o -o prog
```

**Rules:**
- Use the **same** `-flto`/`-flto=thin` flag at both compile and link.
- Use the **same** target triple and ABI options throughout.
- LTO requires an LTO-aware linker: LLD (recommended) or GNU `ld` + `LLVMgold.so` plugin.

---

## Model B: Whole-program optimization in a single process

If your compiler processes multiple modules in one process (e.g., a JIT or AOT driver), you can merge and optimize without file-based LTO:

### Step B1 — Link modules together

```cpp
#include "llvm/Linker/Linker.h"

// Start with an empty "combined" module
auto Combined = std::make_unique<llvm::Module>("combined", Ctx);

// Link each translation unit into it
llvm::Linker L(*Combined);
for (auto &TU : TranslationUnits) {
  if (L.linkInModule(std::move(TU)))
    llvm::report_fatal_error("Linking failed");
}
```

### Step B2 — Run a module-level optimization pipeline

```cpp
#include "llvm/Passes/PassBuilder.h"

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

// Build an O2-equivalent pipeline for the combined module
llvm::ModulePassManager MPM =
    PB.buildPerModuleDefaultPipeline(llvm::OptimizationLevel::O2);
MPM.run(*Combined, MAM);
```

### Step B3 — Emit native code from the combined module

```cpp
#include "llvm/Target/TargetMachine.h"
#include "llvm/Support/TargetSelect.h"

// ... initialize target, create TargetMachine, emit object file ...
// (same as single-module compilation — see out-of-tree.md)
```

---

## CMake: ensure BitWriter component is linked

For bitcode emission, add `BitWriter` to your `llvm_map_components_to_libnames` call:

```cmake
llvm_map_components_to_libnames(LLVM_LIBS
  core support bitwriter linker passes
  # add your target(s) here
)
target_link_libraries(MyCompiler PRIVATE ${LLVM_LIBS})
```

---

## Inspecting bitcode artifacts

```bash
# Disassemble bitcode to readable IR
llvm-dis output.bc -o output.ll

# Assemble IR back to bitcode
llvm-as output.ll -o output.bc

# Inspect LTO sections in an object
llvm-readobj --llvm-bitcode-section output.o

# Check ThinLTO summary
llvm-readobj --thinlto-summary output.o
```

---

## Debugging LTO issues

| Symptom | Likely cause |
|---|---|
| Symbols undefined at link | Hidden visibility blocking import; check `__attribute__((visibility("default")))` |
| ODR violations / wrong behavior | Two definitions of the same symbol; check for `inline` mismatches |
| LTO binary differs from non-LTO | Optimization exposing UB — check with `-fsanitize=undefined` |
| Linker ignores bitcode | Using a non-LTO-aware linker; switch to `lld` |
| Bitcode mismatch error | Mixing bitcode from different LLVM versions — pin to LLVM 22 |

---

## Common mistakes

- **Never** mix `-flto` objects with a non-LTO-aware linker without understanding the consequences.
- **Never** assume bitcode is portable across LLVM major versions — always pin producer and consumer to the same version.
- **Never** forget to pass optimization level flags consistently at link time.
- **Always** use `llvm-dis` to inspect failing modules when diagnosing IR-level LTO bugs.
- **Always** verify your combined module with `verifyModule(*M, &errs())` before running the optimization pipeline.

---

## See also

- [out-of-tree.md](../../docs/out-of-tree.md) — CMake components and TargetMachine setup
- [new-pass-manager.md](../../docs/new-pass-manager.md) — pass pipelines used at link time
