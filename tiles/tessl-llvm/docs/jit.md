# ORC JIT v2 — Architecture & Usage (LLVM 20)

Reference: [ORCv2 design](https://llvm.org/docs/ORCv2.html) | [BuildingAJIT tutorial](https://llvm.org/docs/tutorial/BuildingAJIT1.html)

> ORC JIT v2 is the only supported JIT in LLVM 20. MCJIT is deprecated. Use `LLJIT` or `LLLazyJIT` for new projects.

---

## Architecture overview

```
JITDylib              — a dynamic library of JIT'd symbols; like a shared object
  │
  ├── IRCompileLayer   — compiles LLVM IR → object code
  ├── ObjectLayer      — links object files; resolves relocations
  └── DefinitionGenerator — searches external libs for missing symbols

ExecutionSession      — the root; owns all JITDylibs; manages threading
MaterializationUnit   — lazily or eagerly provides symbols
SymbolStringPool      — interned symbol names
```

**Two ready-made JIT types:**

| Type | Use when |
|------|---------|
| `LLJIT` | Eager compilation — compile all IR up front when added |
| `LLLazyJIT` | Lazy compilation — compile a function only on first call |

---

## LLJIT — eager JIT

```cpp
#include "llvm/ExecutionEngine/Orc/LLJIT.h"
#include "llvm/ExecutionEngine/Orc/ThreadSafeModule.h"
#include "llvm/Support/InitLLVM.h"
#include "llvm/Support/TargetSelect.h"

using namespace llvm;
using namespace llvm::orc;

int main(int argc, char **argv) {
  InitLLVM X(argc, argv);

  // Initialize native target (must be called before creating JIT)
  InitializeNativeTarget();
  InitializeNativeTargetAsmPrinter();
  InitializeNativeTargetAsmParser();

  // Create LLJIT instance
  auto JIT = LLJITBuilder().create();
  if (!JIT) {
    errs() << toString(JIT.takeError()) << "\n";
    return 1;
  }

  // Add an LLVM module
  LLVMContext Ctx;
  auto M = buildMyModule(Ctx);       // returns std::unique_ptr<Module>
  // Wrap in ThreadSafeModule (required by ORC — owns context + module)
  ThreadSafeModule TSM(std::move(M), std::make_unique<LLVMContext>());

  if (auto Err = (*JIT)->addIRModule(std::move(TSM))) {
    errs() << toString(std::move(Err)) << "\n";
    return 1;
  }

  // Look up and call a function
  auto Sym = (*JIT)->lookup("my_function");
  if (!Sym) {
    errs() << toString(Sym.takeError()) << "\n";
    return 1;
  }

  // Cast to function pointer and call
  auto *FnPtr = Sym->toPtr<int(int)>();
  int Result = FnPtr(42);
  return 0;
}
```

---

## LLLazyJIT — lazy (on-demand) compilation

```cpp
#include "llvm/ExecutionEngine/Orc/LLLazyJIT.h"

auto LazyJIT = LLLazyJITBuilder().create();
if (!LazyJIT) { /* handle error */ }

// Add module — functions compiled only on first call
ThreadSafeModule TSM(std::move(M), std::make_unique<LLVMContext>());
if (auto Err = (*LazyJIT)->addLazyIRModule(std::move(TSM))) { /* handle */ }

// Lookup triggers compilation of that function only
auto Sym = (*LazyJIT)->lookup("expensive_function");
```

---

## ThreadSafeModule and ThreadSafeContext

ORC is thread-safe. `ThreadSafeModule` wraps a `Module` + `LLVMContext` with a mutex so multiple threads can compile different modules concurrently.

```cpp
// Option A: dedicated context per module (recommended for concurrency)
auto TSM = ThreadSafeModule(
    std::move(M),
    std::make_unique<LLVMContext>()  // owns a fresh context
);

// Option B: shared context (simpler, but serializes compilation)
auto SharedCtx = std::make_shared<ThreadSafeContext>(std::make_unique<LLVMContext>());
auto TSM2 = ThreadSafeModule(std::move(M2), SharedCtx);

// Operate on the module with the lock held:
SharedCtx->withModuleDo(*TSM2.getModule(), [](Module &M) {
  // safe to access M here
});
```

---

## JITDylibs and symbol visibility

```cpp
ExecutionSession &ES = (*JIT)->getExecutionSession();

// Get the main JITDylib (created by LLJIT automatically)
JITDylib &MainJD = (*JIT)->getMainJITDylib();

// Create a new JITDylib (like loading a shared library)
JITDylib &LibJD = ES.createBareJITDylib("mylib");

// Add module to a specific dylib
(*JIT)->addIRModule(LibJD, std::move(TSM));

// Search order: look in MainJD first, then LibJD
MainJD.addToLinkOrder(LibJD);
```

---

## Exposing host process symbols (stdlib, runtime)

By default, JIT'd code cannot call `printf`, `malloc`, etc. Add a generator:

```cpp
#include "llvm/ExecutionEngine/Orc/DynamicLibrarySearchGenerator.h"

// Expose all symbols from the current process
auto &MainJD = (*JIT)->getMainJITDylib();
MainJD.addGenerator(
    cantFail(DynamicLibrarySearchGenerator::GetForCurrentProcess(
        (*JIT)->getDataLayout().getGlobalPrefix()))
);
// Now JIT'd code can call malloc, printf, etc.
```

To expose only a specific shared library:

```cpp
MainJD.addGenerator(
    cantFail(DynamicLibrarySearchGenerator::Load(
        "/path/to/myruntime.so",
        (*JIT)->getDataLayout().getGlobalPrefix()))
);
```

---

## Adding custom symbols (host → JIT)

Inject a host C++ function as a JIT-visible symbol:

```cpp
#include "llvm/ExecutionEngine/Orc/AbsoluteSymbols.h"

// Expose a host function to JIT'd code
int myRuntimeFunc(int x) { return x * 2; }

auto &ES = (*JIT)->getExecutionSession();
auto &MainJD = (*JIT)->getMainJITDylib();

cantFail(MainJD.define(
    absoluteSymbols({
        {ES.intern("my_runtime_func"),
         {ExecutorAddr::fromPtr(&myRuntimeFunc),
          JITSymbolFlags::Exported | JITSymbolFlags::Callable}}
    })
));
```

---

## Running optimization passes before JIT

```cpp
#include "llvm/Passes/PassBuilder.h"

// Wrap LLJIT's IR transform layer with an optimization pipeline
(*JIT)->getIRTransformLayer().setTransform(
    [](ThreadSafeModule TSM, const MaterializationResponsibility &R)
        -> Expected<ThreadSafeModule> {
      TSM.withModuleDo([](Module &M) {
        PassBuilder PB;
        LoopAnalysisManager LAM;
        FunctionAnalysisManager FAM;
        CGSCCAnalysisManager CGAM;
        ModuleAnalysisManager MAM;
        PB.registerModuleAnalyses(MAM);
        PB.registerCGSCCAnalyses(CGAM);
        PB.registerFunctionAnalyses(FAM);
        PB.registerLoopAnalyses(LAM);
        PB.crossRegisterProxies(LAM, FAM, CGAM, MAM);
        ModulePassManager MPM =
            PB.buildPerModuleDefaultPipeline(OptimizationLevel::O2);
        MPM.run(M, MAM);
      });
      return std::move(TSM);
    });
```

---

## Error handling pattern

ORC uses `llvm::Expected<T>` and `llvm::Error` everywhere:

```cpp
// Helper to exit on error (for simple tools)
template <typename T>
T exitOnErr(Expected<T> E) {
  if (!E) {
    errs() << toString(E.takeError()) << "\n";
    exit(1);
  }
  return std::move(*E);
}

auto JIT  = exitOnErr(LLJITBuilder().create());
auto Sym  = exitOnErr(JIT->lookup("foo"));
auto *Fn  = Sym.toPtr<void()>();
// LLJIT provides its own ExitOnError:
llvm::ExitOnError ExitOnErr;
auto JIT2 = ExitOnErr(LLJITBuilder().create());
```

---

## CMake components

```cmake
llvm_map_components_to_libnames(LLVM_LIBS
  Core Support OrcJIT ExecutionEngine
  X86CodeGen X86AsmParser X86Desc X86Info  # native target
)
```

---

## Common mistakes

- **Do NOT** use MCJIT (`llvm/ExecutionEngine/MCJIT.h`) — it is deprecated; use ORC v2.
- **Do NOT** share `LLVMContext` across threads without `ThreadSafeContext` — data races will corrupt IR.
- **Do NOT** call `InitializeNativeTarget()` after constructing a JIT — call it first in `main()`.
- **Do NOT** forget `InitializeNativeTargetAsmPrinter()` — without it, the JIT can't emit code.
- **Do NOT** hold raw pointers to JIT'd memory after the `JITDylib` or `ExecutionSession` is destroyed — they become dangling.
- **ALWAYS** add a `DynamicLibrarySearchGenerator` if JIT'd code calls any C stdlib functions.
- **ALWAYS** use `ExitOnError` or properly handle `llvm::Error` — ORC never silently fails.
