# Out-of-tree LLVM (22.1.x)

Projects that **link against LLVM libraries** but live outside `llvm-project` need a stable CMake setup and compatible C++ ABI against the **same** LLVM build the user installs.

## CMake essentials

- Use **`find_package(LLVM REQUIRED CONFIG)`** pointing at an LLVM build that exported `LLVMConfig.cmake` (install or build-tree).
- **`llvm_map_components_to_libnames`** + `target_link_libraries` for needed components (`Core`, `Support`, `IR`, `PassBuilder`, etc.).
- Match **`CMAKE_CXX_STANDARD`** to LLVM (LLVM 22 requires C++17 or newer per project policy—follow `LLVM_REQUIRED_CXX_STANDARD` or docs in `LLVMConfig.cmake`).

## Pass / plugin builds

- **LLVM plugins** — `PassPlugins` loading shared objects; must export registration entry points expected by the pass plugin loader for your LLVM version.
- **Static linking** tools — Link the same component set `opt`/`llc` would use for similar features; missing symbols usually mean a forgotten component.

## Version skew

- **LLVM_MAJOR** mismatch (headers vs libraries) causes subtle crashes. Pin **exact** minor/patch when debugging “impossible” failures.

## References

- [Building LLVM with CMake](https://llvm.org/docs/CMake.html)
- [How to set up LLVM-style RTTI](https://llvm.org/docs/HowToSetUpLLVMStyleRTTI.html) if adding class hierarchies.
