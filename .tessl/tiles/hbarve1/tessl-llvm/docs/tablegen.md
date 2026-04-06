# TableGen (22.1.x)

TableGen is LLVM’s DSL for **declarative description** of instructions, registers, calling conventions, attributes, and many backend details. Backends and parts of Clang rely heavily on `.td` files.

## Where it shows up

- **Target backends** — `*.td` under `lib/Target/<Arch>/` describe instructions, schedules, register classes.
- **Clang** — Some attributes and builtins are TableGn-driven.
- **Misc** — DAG ISel patterns, calling conventions, intrinsics wiring (target-dependent).

## Workflow (high level)

1. Edit **records** in `.td` (classes, multiclasses, `def`, `defm`).
2. Run **tablegen** (often via CMake `llvm-tablegen` / custom targets) to emit C++ inc files or diagnostics.
3. Rebuild affected targets; fix pattern/operand mismatches from TableGen or MC errors.

## References

- [TableGen overview](https://llvm.org/docs/TableGen/index.html)
- Backend-specific docs under `llvm/lib/Target/<Arch>/` (*README*, *BuildLog*) where present.

## Agent hint

When editing patterns, **look at neighboring defs** in the same `.td` file for arity, operand types, and predicate style—TableGen is terse and consistency with local style matters.
