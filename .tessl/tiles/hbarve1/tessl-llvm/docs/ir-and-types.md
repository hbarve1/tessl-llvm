# LLVM IR and types (22.1.x orientation)

## Module, function, basic blocks

- **Module** — Top container: functions, globals, aliases, comdats, metadata.
- **Function** — SSA IR in **basic blocks**; terminators end each block (`br`, `ret`, `switch`, `invoke`, etc.).
- **Instructions** — Most live in a block and produce **SSA values**; `phi` merges values at block joins.

## Type system highlights

- **Integer** — `i1`, `i8`, … arbitrary bit width.
- **Floating** — `half`, `bfloat`, `float`, `double`, `fp128`, ppc_fp128, `x86_fp80`, etc.
- **Pointers** — Opaque pointers: `ptr` (legacy typed pointers are gone in modern LLVM; **do not** use `i8*` as a stand-in for “byte pointer” in new discussions—use `ptr` with address spaces as needed).
- **Aggregates** — `array`, `struct`, `vector`; packed layout matters for some targets.
- **Labels / tokens** — For exception handling and tokens (`token` type).

## Fast references

- Full opcode and type rules: [LangRef](https://llvm.org/docs/LangRef.html).
- IR assembly printed/read by `llvm::parseIR`, `llc`, `opt`, etc.

## Common agent mistakes to avoid

- Assuming **typed pointers** or old intrinsic names without checking LangRef for 22.x.
- Confusing **LLVM IR types** with Clang/C++ types—frontend lowering is target- and language-specific.
