# Out-of-Tree LLVM 20 Projects — CMake Setup & Linking

Reference: [Building LLVM with CMake](https://llvm.org/docs/CMake.html) | [llvm-config](https://llvm.org/docs/CommandGuide/llvm-config.html)

---

## When to build out-of-tree

Build out-of-tree (against an installed LLVM) when you are:
- Writing a standalone compiler frontend or language runtime
- Developing a pass plugin loadable by `opt --load-pass-plugin`
- Building tooling (e.g., a static analyzer, refactoring tool) on top of LLVM

Build in-tree (inside the LLVM source tree) when you are:
- Adding a new target backend
- Adding in-tree passes, intrinsics, or IR changes
- Contributing to LLVM upstream

---

## Installing LLVM 20

```bash
# macOS (Homebrew)
brew install llvm@20
# Headers: /opt/homebrew/opt/llvm@20/include
# CMake:   /opt/homebrew/opt/llvm@20/lib/cmake/llvm

# Ubuntu / Debian
apt install llvm-20-dev libclang-20-dev
# Headers: /usr/lib/llvm-20/include
# CMake:   /usr/lib/llvm-20/lib/cmake/llvm

# Build from source
cmake -S llvm -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/usr/local/llvm-20 \
  -DLLVM_TARGETS_TO_BUILD="X86;AArch64;RISCV" \
  -DLLVM_ENABLE_PROJECTS="clang" \
  -G Ninja
ninja -C build install
```

Locate the cmake config directory:
```bash
llvm-config-20 --cmakedir
# /usr/lib/llvm-20/lib/cmake/llvm
```

---

## Minimal CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(MyCompiler CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# ── Locate LLVM 20 ────────────────────────────────────────────────────────────
find_package(LLVM 20 REQUIRED CONFIG)
message(STATUS "LLVM version: ${LLVM_PACKAGE_VERSION}")
message(STATUS "LLVM CMake dir: ${LLVM_DIR}")

include_directories(${LLVM_INCLUDE_DIRS})
separate_arguments(LLVM_DEFINITIONS_LIST NATIVE_COMMAND ${LLVM_DEFINITIONS})
add_definitions(${LLVM_DEFINITIONS_LIST})

# ── Map LLVM components to link targets ───────────────────────────────────────
llvm_map_components_to_libnames(LLVM_LIBS
  Core          # LLVMContext, Module, IRBuilder, basic types
  Support       # raw_ostream, StringRef, Error, CommandLine
  Analysis      # DominatorTree, ScalarEvolution, AliasAnalysis
  Passes        # PassBuilder, NPM infrastructure
  IRReader      # parseIRFile, parseAssemblyString
  BitWriter     # WriteBitcodeToFile
  Target        # TargetMachine, TargetRegistry
  X86CodeGen    # X86 backend (replace with your target)
  X86AsmParser
  X86Desc
  X86Info
)

# ── Targets ───────────────────────────────────────────────────────────────────
add_executable(mycompiler
  src/main.cpp
  src/Frontend.cpp
  src/CodeGen.cpp
)

target_include_directories(mycompiler PRIVATE include)
target_link_libraries(mycompiler PRIVATE ${LLVM_LIBS})
```

---

## Component reference

List all available components:
```bash
llvm-config-20 --components
```

### Commonly used components

| Component | What it provides |
|-----------|-----------------|
| `Core` | `LLVMContext`, `Module`, `Function`, `IRBuilder`, all IR types |
| `Support` | `raw_ostream`, `StringRef`, `Error`, `Expected`, `CommandLine`, `MemoryBuffer` |
| `Analysis` | `DominatorTree`, `LoopInfo`, `ScalarEvolution`, `AAResults`, `CallGraph` |
| `Passes` | `PassBuilder`, `ModulePassManager`, all standard analyses |
| `IRReader` | `parseIRFile()`, `parseAssemblyString()` |
| `BitReader` | `parseBitcodeFile()` |
| `BitWriter` | `WriteBitcodeToFile()` |
| `Linker` | `Linker::linkModules()` |
| `TransformUtils` | `CloneFunction`, `InlineFunction`, utility transforms |
| `Scalar` | Scalar optimization passes (instcombine, simplifycfg, etc.) |
| `IPO` | Interprocedural passes (inliner, argument promotion, etc.) |
| `Vectorize` | Loop and SLP vectorizers |
| `Target` | `TargetMachine`, `TargetRegistry`, `TargetOptions` |
| `MC` | `MCContext`, `MCInstrInfo`, `MCRegisterInfo` — needed for MC layer |
| `ExecutionEngine` | LLVM interpreter / JIT infrastructure |
| `MCJIT` | MCJIT execution engine |
| `OrcJIT` | ORC JIT v2 (preferred JIT in LLVM 20) |
| `X86CodeGen` | X86 target code generation |
| `AArch64CodeGen` | AArch64 target code generation |
| `RISCVCodeGen` | RISC-V target code generation |

---

## Build and configure

```bash
# Using LLVM_DIR explicitly
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_DIR=$(llvm-config-20 --cmakedir)

# Or via CMAKE_PREFIX_PATH
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_PREFIX_PATH=/usr/local/llvm-20

cmake --build build -j$(nproc)
```

---

## Pass plugin CMake setup

To build a pass loadable by `opt --load-pass-plugin`:

```cmake
add_library(MyPassPlugin MODULE
  src/MyPass.cpp
)

target_include_directories(MyPassPlugin PRIVATE
  ${LLVM_INCLUDE_DIRS}
  include
)

target_compile_definitions(MyPassPlugin PRIVATE ${LLVM_DEFINITIONS})

set_target_properties(MyPassPlugin PROPERTIES
  CXX_STANDARD 17
  POSITION_INDEPENDENT_CODE ON
  # No default lib prefix on macOS/Linux
  PREFIX ""
)

# Do NOT link LLVM libs into the plugin — use symbols from the host (opt)
# target_link_libraries(MyPassPlugin PRIVATE ${LLVM_LIBS})  ← DON'T DO THIS
```

Test the plugin:
```bash
opt --load-pass-plugin=./MyPassPlugin.so \
    -passes=my-pass -S input.ll
```

---

## LLVM_ENABLE_ASSERTIONS and build types

```bash
# Debug build with assertions (slow, catches bugs early)
cmake -DCMAKE_BUILD_TYPE=Debug \
      -DLLVM_ENABLE_ASSERTIONS=ON ...

# Release with assertions (recommended for development)
cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo \
      -DLLVM_ENABLE_ASSERTIONS=ON ...

# Release (production / benchmarking)
cmake -DCMAKE_BUILD_TYPE=Release \
      -DLLVM_ENABLE_ASSERTIONS=OFF ...
```

Match `LLVM_ENABLE_ASSERTIONS` to what your LLVM install was built with to avoid ABI mismatches.

---

## Multiple LLVM installs

If you have multiple LLVM versions installed, avoid mixing them:

```bash
# Ensure cmake finds LLVM 20, not LLVM 19:
export PATH=/usr/local/llvm-20/bin:$PATH
cmake -DLLVM_DIR=/usr/local/llvm-20/lib/cmake/llvm ...

# Check version after configure:
# The cmake output should show: "LLVM version: 20.x.x"
```

---

## Useful cmake variables set by LLVMConfig.cmake

| Variable | Description |
|----------|-------------|
| `LLVM_PACKAGE_VERSION` | Full version string, e.g. `20.0.0` |
| `LLVM_VERSION_MAJOR` | Major version integer, e.g. `20` |
| `LLVM_INCLUDE_DIRS` | Header search paths |
| `LLVM_LIBRARY_DIRS` | Library search paths |
| `LLVM_DEFINITIONS` | Compiler definitions (e.g. `-D_GNU_SOURCE`) |
| `LLVM_TOOLS_BINARY_DIR` | Path to `opt`, `llc`, `llvm-lit`, etc. |
| `LLVM_BUILD_MAIN_SRC_DIR` | Source tree (only set for build-tree installs) |
| `LLVM_ENABLE_ASSERTIONS` | Whether assertions were enabled |
| `LLVM_ENABLE_EH` | Whether exceptions are enabled in LLVM |
| `LLVM_ENABLE_RTTI` | Whether RTTI is enabled in LLVM |

> **Important:** If `LLVM_ENABLE_RTTI=OFF` (the default), your project must also disable RTTI:
> ```cmake
> if(NOT LLVM_ENABLE_RTTI)
>   target_compile_options(mycompiler PRIVATE -fno-rtti)
> endif()
> ```

---

## Common mistakes

- **Do NOT** hardcode `-lLLVMCore` etc. — use `llvm_map_components_to_libnames` so CMake handles ordering and transitive deps.
- **Do NOT** link LLVM libs into a pass plugin — the plugin is loaded into `opt` which already has them; double-linking causes symbol conflicts.
- **Do NOT** mix LLVM 20 headers with LLVM 19 libraries — always verify `LLVM_PACKAGE_VERSION` in the cmake output.
- **Do NOT** forget to match RTTI and exception settings between your project and the LLVM install.
- **ALWAYS** call `InitializeAll*()` target init functions before using `TargetRegistry` or creating `TargetMachine`.
