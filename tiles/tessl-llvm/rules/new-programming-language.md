# LLVM 20 — Agent Steering Rules

These rules apply whenever you are writing, reviewing, or modifying code that uses LLVM 20 APIs. They are always loaded into context — keep them in mind for every LLVM code generation decision.

---

## Pass Manager

- **Always use the New Pass Manager (NPM).** The legacy `PassManager` is removed in LLVM 20. Never emit `legacy::PassManager`, `FunctionPass`, `ModulePass`, `getAnalysis<>()`, `AU.setPreservesAll()`, or `createXYZPass()` factory functions.
- Pass classes must inherit from `PassInfoMixin<T>` and implement `PreservedAnalyses run(IRUnit &, AnalysisManager &)`.
- Return `PreservedAnalyses::all()` when nothing is modified. Return `PreservedAnalyses::none()` only when the full IR may have changed.
- Obtain analysis results via `FAM.getResult<AnalysisType>(F)`, not via `getAnalysis<>()`.
- For more: [new-pass-manager.md](../docs/new-pass-manager.md)

---

## Pointers and types

- **All pointers are opaque (`ptr`).** Never use typed pointer types (`i8*`, `i32*`) or `getPointerElementType()`.
- Construct pointers with `Builder.getPtrTy()` or `PointerType::get(Ctx, AddrSpace)`.
- **Always provide an explicit element type** for `LoadInst`, `StoreInst`, and `GetElementPtrInst`:
  - `Builder.CreateLoad(ElemTy, Ptr, "name")`
  - `Builder.CreateGEP(ElemTy, Ptr, Indices)`
- Track pointee types in your own data structures — do not re-derive them from pointer types.
- For more: [ir-types.md](../docs/ir-types.md)

---

## Standard library types

- **Use `std::optional<T>` and `std::nullopt`** — `llvm::Optional` and `llvm::None` are removed.
- Use `llvm::Expected<T>` and `llvm::Error` for fallible operations, not raw null pointer returns.
- Use `llvm::SmallVector`, `llvm::StringRef`, `llvm::ArrayRef` for LLVM-internal collections.

---

## API correctness

- **Verify API signatures against LLVM 20 headers** before emitting any LLVM API call. When uncertain, consult the relevant doc page or note that verification is needed.
- Use `Intrinsic::getOrInsertDeclaration()` — `Intrinsic::getDeclaration()` is deprecated.
- Include headers from `llvm/TargetParser/Triple.h` and `llvm/TargetParser/Host.h` — the old `llvm/ADT/Triple.h` and `llvm/Support/Host.h` paths are moved.
- For the full list of LLVM 20 breaking changes: [version-notes.md](../docs/version-notes.md)

---

## Coding standards

- **Do NOT use `using namespace llvm;` in headers.** Use it in `.cpp` files only.
- Prefer `auto *` for downcasts where the type is obvious from context. Use `cast<T>()` only when you are certain of the type; use `dyn_cast<T>()` for safe checked casts.
- Use `llvm::PatternMatch` (`m_Add`, `m_Value`, `m_ConstantInt`, etc.) for IR pattern matching instead of manual `dyn_cast` chains.
- Prefer `llvm::errs()` / `llvm::outs()` over `std::cerr` / `std::cout`.
- Use `LLVM_DEBUG(dbgs() << ...)` for debug output — controlled by `-debug` / `DEBUG_TYPE`.

---

## IR construction

- Use `IRBuilder<>` for all instruction emission — never construct `Instruction` subclasses directly via `new`.
- Use `PoisonValue::get(T)` instead of `UndefValue::get(T)` when modeling undefined behavior — poison is the correct LLVM 20 model for UB.
- Always call `verifyModule(*M, &errs())` during development to catch IR invariant violations early.
- Use `IRBuilder<>::CreateNSWAdd` / `CreateNUWAdd` / `CreateFAddFMF` when you know flags apply — these enable more optimization.

---

## TableGen

- Always include `llvm/Target/Target.td` at the top of target `.td` files.
- Always set `let Namespace = "MyTarget"` on all registers, register classes, and instructions.
- Never hand-edit `*Gen*.inc` files — always regenerate via `cmake --build --target XYZCommonTableGen`.
- For more: [tablegen.md](../docs/tablegen.md)

---

## Out-of-tree projects

- Use `find_package(LLVM 20 REQUIRED CONFIG)` and `llvm_map_components_to_libnames` — never hardcode `-lLLVMCore` etc.
- Set `CMAKE_CXX_STANDARD 17` — LLVM 20 requires C++17.
- If `LLVM_ENABLE_RTTI=OFF` (default), add `-fno-rtti` to your project's compile options.
- Never link LLVM libs into a pass plugin — use only symbols provided by the `opt` host process.
- For more: [out-of-tree.md](../docs/out-of-tree.md)

---

## Code generation

- Prefer GlobalISel for new targets — it operates on Machine IR throughout and avoids the SelectionDAG intermediate representation.
- Use `BuildMI(MBB, InsertPt, DL, TII.get(Opc))` to build `MachineInstr`s — never construct them directly.
- Always call `constrainSelectedInstRegOperands` after manual instruction selection in GlobalISel.
- For more: [codegen.md](../docs/codegen.md)

---

## Frontend IR lowering

- Use the alloca + mem2reg pattern for local variables — never emit PHI nodes manually during initial lowering.
- Always alloca in the entry block, not at the point of use. `PromotePass` (mem2reg) only promotes entry-block allocas.
- Always check `getTerminator()` before adding a branch — two terminators on one block is a verifier error.
- Forward-declare all functions before emitting bodies to support mutual recursion.
- For more: [frontend-to-ir.md](../docs/frontend-to-ir.md)

---

## Debug info

- Always call `DBuilder.finalize()` after all functions are emitted — never skip it.
- Clear the debug location (`B.SetCurrentDebugLocation({})`) for compiler-synthesized instructions with no source correspondence.
- Set `AlwaysPreserve=true` for parameter variables so they survive optimization.
- Add `Debug Info Version` and `Dwarf Version` module flags.
- For more: [debug-info.md](../docs/debug-info.md)

---

## JIT (ORC v2)

- Use `LLJIT` or `LLLazyJIT` — never MCJIT (deprecated/removed in LLVM 20).
- Call `InitializeNativeTarget()`, `InitializeNativeTargetAsmPrinter()` at program start, before constructing any JIT.
- Use one `LLVMContext` per `ThreadSafeModule` for correct thread-safety.
- Add `DynamicLibrarySearchGenerator::GetForCurrentProcess` if JIT'd code calls libc.
- For more: [jit.md](../docs/jit.md)

---

## Exception handling

- Use `invoke` (not `call`) for any call that may unwind in a function that has landing pads.
- Always set `F->setPersonalityFn()` on functions with landing pads.
- Always call `__cxa_begin_catch` / `__cxa_end_catch` around catch bodies (Itanium ABI).
- For more: [exception-handling.md](../docs/exception-handling.md)

---

## Attributes and metadata

- Set `NoUnwind` on all functions that never throw — it unlocks significant optimizer opportunities.
- Set `NoCapture` on pointer parameters that don't escape the call — critical for alias analysis.
- Set `Noundef` on values guaranteed to be fully initialized.
- Use `!llvm.loop` metadata to communicate vectorization/unroll hints; don't rely on the optimizer to infer them.
- For more: [attributes-metadata.md](../docs/attributes-metadata.md)
