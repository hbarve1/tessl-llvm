# Lit / FileCheck Test Suite for a Constant Folding Pass

## Problem/Feature Description

A compiler team just landed an in-tree LLVM pass called `constfold-demo` that performs several simple optimizations on integer arithmetic: it folds `x + 0` to `x`, `x * 1` to `x`, and `x - x` to `0`. The pass also eliminates branches whose condition is a constant `true` or `false`.

Before merging to the main branch, the team lead requires a proper lit test file that validates each transformation using FileCheck. The test file will live at `test/Transforms/ConstFoldDemo/constfold-demo.ll` and will be run as part of CI with `llvm-lit`.

The team has seen CI failures caused by tests that appeared to pass but produced incorrect results silently, and by tests that broke whenever someone renamed an internal IR value. The lead wants a robust test file that follows LLVM testing best practices to avoid these recurring issues.

## Output Specification

Produce a single file `constfold-demo.ll` that:

1. Tests all four transformations (add-zero folding, mul-one folding, sub-self-to-zero, constant-branch elimination) — at least one test function per transformation
2. Includes a negative test that verifies a volatile load is NOT eliminated
3. Uses multiple RUN lines (at minimum: one correctness run and one smoke test)
4. Uses FileCheck patterns that won't break on minor IR changes

The test file should be ready to run with:
```
llvm-lit constfold-demo.ll
```
(assuming `opt` is on PATH and the pass is registered as `constfold-demo`)
