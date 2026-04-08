# Interactive Arithmetic REPL with JIT Compilation

## Problem/Feature Description

A startup is building a domain-specific scripting language for financial analysts. The language supports simple arithmetic expressions with named variables. To give users instant feedback, the team wants a REPL (Read-Eval-Print Loop) that compiles each expression to native code on-the-fly and executes it immediately — giving the performance of compiled code without a separate compile step.

The team has a partially-written codebase that still uses MCJIT from an older LLVM version. Their senior engineer left notes saying MCJIT is gone in the new LLVM and the whole JIT layer needs to be rewritten. You've been brought in to write the new JIT infrastructure from scratch using the current LLVM 22 APIs.

The REPL should handle expressions like:
- `2 + 3 * 4` → 14
- `let x = 10` (store a variable)
- `x + 5` → 15

Since this will run in a headless CI environment without interactive input, implement the REPL logic as a `runExpressions` function that takes a vector of expression strings (or pre-built IR modules) and evaluates them in sequence, printing results to stdout.

## Output Specification

Produce a single self-contained C++ source file `jit_repl.cpp` that:

1. Implements the JIT infrastructure using LLVM 22 APIs
2. Contains a `main()` function that demonstrates the REPL by evaluating at least 3 hardcoded expressions and printing results
3. Shows how to add a module to the JIT, look up a symbol, and call it

You do not need to implement a full parser — you may hardcode the expressions as LLVM IR construction code (using IRBuilder) to demonstrate the JIT pipeline. The focus is on the JIT infrastructure, not the parser.

Also produce `CMakeLists.txt` for building the project against LLVM 22.
