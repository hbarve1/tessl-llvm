---
name: out-of-tree-setup
description: Scaffold an out-of-tree LLVM 20 compiler/language project from scratch. Covers CMake configuration, finding LLVM, linking components, a minimal LLVMContext driver, and build instructions.
---

# Skill: Set Up an Out-of-Tree LLVM 20 Project

Use this skill when the user wants to start a new compiler, language runtime, or LLVM-based tool that builds **against** an installed LLVM 20 rather than inside the LLVM source tree.

---

## Step 0 — Prerequisites

Confirm the user has LLVM 20 installed. Common locations:

```bash
# Homebrew (macOS)
brew install llvm@20
# Path: /opt/homebrew/opt/llvm@20

# apt (Ubuntu/Debian)
apt install llvm-20-dev
# Path: /usr/lib/llvm-20

# From source
cmake --install build --prefix /usr/local/llvm-20
```

Find the cmake config dir:
```bash
llvm-config-20 --cmakedir
# or
find /usr -name "LLVMConfig.cmake" 2>/dev/null
```

---

## Step 1 — Project layout

Create this minimal layout:

```
<project>/
├── CMakeLists.txt
├── include/
│   └── <Project>.h
└── src/
    └── main.cpp
```

Scale up later: add `lib/`, `test/`, `tools/` as needed.

---

## Step 2 — Root `CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.20)
project(<ProjectName> CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# ── Find LLVM 20 ──────────────────────────────────────────────────────────────
find_package(LLVM 22 REQUIRED CONFIG)

message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")
message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")

include_directories(${LLVM_INCLUDE_DIRS})
add_definitions(${LLVM_DEFINITIONS})

# ── Map component names to library names ──────────────────────────────────────
# Common components — add/remove as needed:
#   Core         — LLVMContext, Module, IRBuilder, basic IR types
#   Support      — raw_ostream, StringRef, Error, CommandLine
#   Analysis     — pass analyses (DominatorTree, ScalarEvolution, etc.)
#   Passes       — New Pass Manager, PassBuilder, StandardInstrumentations
#   IRReader     — parseIRFile / parseAssemblyString
#   BitWriter    — WriteBitcodeToFile
#   Target       — TargetMachine, TargetRegistry
#   X86          — X86 target (or AArch64, RISCV, etc.)
llvm_map_components_to_libnames(LLVM_LIBS
  Core
  Support
  Analysis
  Passes
  IRReader
  BitWriter
  Target
  X86CodeGen      # replace/extend with your target
  X86AsmParser
  X86Desc
  X86Info
)

# ── Build the executable ──────────────────────────────────────────────────────
add_executable(<ProjectName> src/main.cpp)

target_include_directories(<ProjectName> PRIVATE
  ${CMAKE_SOURCE_DIR}/include
)

target_link_libraries(<ProjectName> PRIVATE ${LLVM_LIBS})
```

> **Note:** Use `llvm_map_components_to_libnames` — do NOT hardcode `-lLLVMCore` etc.
> Component names are case-sensitive and listed in `llvm-config --components`.

---

## Step 3 — Minimal `src/main.cpp`

This compiles, creates a module, prints IR, and JIT-compiles a trivial function:

```cpp
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Verifier.h"
#include "llvm/Support/InitLLVM.h"
#include "llvm/Support/TargetSelect.h"
#include "llvm/Support/raw_ostream.h"

using namespace llvm;

int main(int argc, char **argv) {
  // InitLLVM handles signal handlers, pretty-stack-trace, and ManagedStatic.
  InitLLVM X(argc, argv);

  // Initialize targets (required before any codegen or JIT).
  InitializeAllTargetInfos();
  InitializeAllTargets();
  InitializeAllTargetMCs();
  InitializeAllAsmParsers();
  InitializeAllAsmPrinters();

  LLVMContext Ctx;
  auto M = std::make_unique<Module>("hello", Ctx);
  M->setTargetTriple(sys::getDefaultTargetTriple());

  // Build: define i32 @add(i32 %a, i32 %b) { ret i32 add(a, b) }
  IRBuilder<> Builder(Ctx);
  FunctionType *FT = FunctionType::get(
      Builder.getInt32Ty(),
      {Builder.getInt32Ty(), Builder.getInt32Ty()},
      /*isVarArg=*/false);

  Function *F = Function::Create(FT, Function::ExternalLinkage, "add", *M);
  F->getArg(0)->setName("a");
  F->getArg(1)->setName("b");

  BasicBlock *Entry = BasicBlock::Create(Ctx, "entry", F);
  Builder.SetInsertPoint(Entry);
  Value *Sum = Builder.CreateAdd(F->getArg(0), F->getArg(1), "sum");
  Builder.CreateRet(Sum);

  // Verify the module.
  if (verifyModule(*M, &errs())) {
    errs() << "Module verification failed\n";
    return 1;
  }

  // Print IR to stdout.
  M->print(outs(), nullptr);
  return 0;
}
```

---

## Step 4 — Build

```bash
# Configure — point CMAKE_PREFIX_PATH at your LLVM 20 install prefix
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_PREFIX_PATH=/path/to/llvm-20/install
  # or: -DLLVM_DIR=$(llvm-config-20 --cmakedir)

# Build
cmake --build build -j$(nproc)

# Run
./build/<ProjectName>
```

Expected output: LLVM IR for the `add` function.

---

## Step 5 — Add an optimization pipeline (optional but recommended)

```cpp
#include "llvm/Passes/PassBuilder.h"

// After module is built and verified:
PassBuilder PB;
LoopAnalysisManager LAM;
FunctionAnalysisManager FAM;
CGSCCAnalysisManager CGAM;
ModuleAnalysisManager MAM;

// Register all the standard analyses.
PB.registerModuleAnalyses(MAM);
PB.registerCGSCCAnalyses(CGAM);
PB.registerFunctionAnalyses(FAM);
PB.registerLoopAnalyses(LAM);
PB.crossRegisterProxies(LAM, FAM, CGAM, MAM);

// Build a standard O2 pipeline.
ModulePassManager MPM = PB.buildPerModuleDefaultPipeline(OptimizationLevel::O2);
MPM.run(*M, MAM);
```

---

## Step 6 — Extend to a pass plugin (optional)

If you want to ship a pass loadable by `opt --load-pass-plugin`, see the
**add-npm-pass** skill. Add this to your `CMakeLists.txt` alongside the executable:

```cmake
add_library(MyPassPlugin MODULE src/MyPass.cpp)
target_include_directories(MyPassPlugin PRIVATE ${LLVM_INCLUDE_DIRS} include)
target_compile_definitions(MyPassPlugin PRIVATE ${LLVM_DEFINITIONS})
set_target_properties(MyPassPlugin PROPERTIES
  CXX_STANDARD 17
  POSITION_INDEPENDENT_CODE ON
)
```

---

## Common mistakes to avoid

- **Do NOT** use `llvm-config --libs` output directly in CMake — use `llvm_map_components_to_libnames` instead; the cmake integration handles RPATH and ordering correctly.
- **Do NOT** call `InitializeAll*()` functions after constructing a `TargetMachine` — initialize before.
- **Do NOT** mix LLVM 20 headers with a different LLVM runtime version — ensure cmake finds the correct `LLVMConfig.cmake`.
- **Do NOT** use `new Module(...)` — use `std::make_unique<Module>(...)`.
- **Do NOT** store raw `Value *` pointers after IR modifications that can RAUW (Replace All Uses With) them.
- **ALWAYS** call `verifyModule` during development; it catches IR invariant violations early.
