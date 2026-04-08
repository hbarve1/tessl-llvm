# Dead Store Eliminator — Out-of-Tree Pass Plugin

## Problem/Feature Description

A compiler research team at a university is building a custom static analysis toolkit on top of LLVM 22. They want to ship a standalone `opt` plugin that identifies and reports dead stores (stores to memory locations whose values are never subsequently loaded). Rather than patching LLVM itself, they want a loadable pass plugin distributed as a shared library that anyone with LLVM 22 installed can drop in and run with `opt --load-pass-plugin`.

The team has been burned before by out-of-tree projects that used the old LLVM pass infrastructure and broke every LLVM release. They want this new plugin built correctly from the start using the current pass infrastructure — structured so `opt` can find and register it, and with a lit test that validates basic behavior.

## Output Specification

Produce the following files that together form the complete pass plugin project:

- `CMakeLists.txt` — root CMake file for the project
- `DeadStoreReporter.h` — pass class declaration
- `DeadStoreReporter.cpp` — pass implementation (the pass should at minimum iterate over a function's instructions and report stores; the exact analysis logic is up to you)
- `test/dead-store-reporter.ll` — a lit test file for the plugin using FileCheck

The pass should be named `dead-store-reporter` in the opt pipeline (i.e., usable via `-passes=dead-store-reporter`).

Keep the implementation focused — the grader will check the structure and API choices, not the sophistication of the analysis.
