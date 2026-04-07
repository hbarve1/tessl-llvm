---
name: jit-setup
description: Set up an ORC JIT v2 execution engine for a language runtime using LLVM 22. Covers LLJIT, LLLazyJIT, ThreadSafeModule, symbol exposure, optimization pipeline, and calling JIT'd functions.
---

# Skill: Set Up an ORC JIT v2 Engine (LLVM 22)

Use this skill when the user wants to JIT-compile and execute LLVM IR at runtime — for a REPL, scripting engine, or language runtime.

---

## Step 0 — CMake setup

```cmake
find_package(LLVM 22 REQUIRED CONFIG)

llvm_map_components_to_libnames(LLVM_LIBS
  Core Support OrcJIT ExecutionEngine
  X86CodeGen X86AsmParser X86Desc X86Info   # replace with your native target
)

add_executable(my_jit src/main.cpp)
target_include_directories(my_jit PRIVATE ${LLVM_INCLUDE_DIRS})
target_compile_definitions(my_jit PRIVATE ${LLVM_DEFINITIONS})
target_link_libraries(my_jit PRIVATE ${LLVM_LIBS})
```

---

## Step 1 — Initialize targets (main entry point)

```cpp
#include "llvm/Support/InitLLVM.h"
#include "llvm/Support/TargetSelect.h"

int main(int argc, char **argv) {
  llvm::InitLLVM X(argc, argv);  // signal handlers, stack traces

  // Must be called before any JIT is constructed
  llvm::InitializeNativeTarget();
  llvm::InitializeNativeTargetAsmPrinter();
  llvm::InitializeNativeTargetAsmParser();

  // ... create JIT, run code ...
}
```

---

## Step 2 — Create LLJIT (eager compilation)

```cpp
#include "llvm/ExecutionEngine/Orc/LLJIT.h"

llvm::ExitOnError ExitOnErr;

auto JIT = ExitOnErr(llvm::orc::LLJITBuilder().create());
```

For more control:

```cpp
auto JIT = ExitOnErr(
    llvm::orc::LLJITBuilder()
        .setNumCompileThreads(4)          // parallel compilation
        .create()
);
```

---

## Step 3 — Expose host symbols (stdlib, runtime)

JIT'd code cannot see `printf`, `malloc`, etc. by default:

```cpp
#include "llvm/ExecutionEngine/Orc/DynamicLibrarySearchGenerator.h"

auto &MainJD = JIT->getMainJITDylib();
MainJD.addGenerator(
    ExitOnErr(llvm::orc::DynamicLibrarySearchGenerator::GetForCurrentProcess(
        JIT->getDataLayout().getGlobalPrefix()))
);
```

To expose a specific host function by name:

```cpp
#include "llvm/ExecutionEngine/Orc/AbsoluteSymbols.h"

double myRuntimeSqrt(double x) { return std::sqrt(x); }

auto &ES = JIT->getExecutionSession();
ExitOnErr(MainJD.define(
    llvm::orc::absoluteSymbols({
        {ES.intern("my_sqrt"),
         {llvm::orc::ExecutorAddr::fromPtr(&myRuntimeSqrt),
          llvm::JITSymbolFlags::Exported | llvm::JITSymbolFlags::Callable}}
    })
));
```

---

## Step 4 — Add an IR module

Wrap your module in `ThreadSafeModule` and add it to the JIT:

```cpp
#include "llvm/ExecutionEngine/Orc/ThreadSafeModule.h"

// Build or receive the module
auto M = buildMyModule();  // std::unique_ptr<llvm::Module>

// Each module needs its own context for thread-safety
auto Ctx = std::make_unique<llvm::LLVMContext>();
// Note: M must have been built with *Ctx, OR re-parse into Ctx

llvm::orc::ThreadSafeModule TSM(std::move(M), std::move(Ctx));
ExitOnErr(JIT->addIRModule(std::move(TSM)));
```

---

## Step 5 — (Optional) Run optimizations before compilation

```cpp
#include "llvm/Passes/PassBuilder.h"

JIT->getIRTransformLayer().setTransform(
    [](llvm::orc::ThreadSafeModule TSM,
       const llvm::orc::MaterializationResponsibility &)
        -> llvm::Expected<llvm::orc::ThreadSafeModule> {
      TSM.withModuleDo([](llvm::Module &M) {
        llvm::PassBuilder PB;
        llvm::LoopAnalysisManager LAM;
        llvm::FunctionAnalysisManager FAM;
        llvm::CGSCCAnalysisManager CGAM;
        llvm::ModuleAnalysisManager MAM;
        PB.registerModuleAnalyses(MAM);
        PB.registerCGSCCAnalyses(CGAM);
        PB.registerFunctionAnalyses(FAM);
        PB.registerLoopAnalyses(LAM);
        PB.crossRegisterProxies(LAM, FAM, CGAM, MAM);
        PB.buildPerModuleDefaultPipeline(llvm::OptimizationLevel::O2)
          .run(M, MAM);
      });
      return std::move(TSM);
    });
```

---

## Step 6 — Look up and call a JIT'd function

```cpp
// Lookup triggers compilation (for LLJIT, always eager)
auto Sym = ExitOnErr(JIT->lookup("my_function"));

// Cast to function pointer
auto *Fn = Sym.toPtr<int(int, int)>();
int Result = Fn(3, 4);  // call the JIT'd function
```

---

## Step 7 — Lazy JIT (compile on first call)

```cpp
#include "llvm/ExecutionEngine/Orc/LLLazyJIT.h"

auto LazyJIT = ExitOnErr(llvm::orc::LLLazyJITBuilder().create());

// Add module — functions compiled only on first invocation
ExitOnErr(LazyJIT->addLazyIRModule(std::move(TSM)));

// Lookup installs a trampoline stub; actual compilation deferred
auto Sym = ExitOnErr(LazyJIT->lookup("my_function"));
auto *Fn = Sym.toPtr<int(int)>();
Fn(1);  // ← compilation happens here, on first call
```

---

## Step 8 — REPL pattern (add modules incrementally)

For a REPL, add a new module per expression/statement:

```cpp
// Each REPL iteration:
while (true) {
  std::string Input = readLine();

  // Parse input into an AST, lower to IR in a fresh module+context
  auto Ctx = std::make_unique<llvm::LLVMContext>();
  auto M   = lowerToIR(*Ctx, Input);   // your frontend

  llvm::orc::ThreadSafeModule TSM(std::move(M), std::move(Ctx));
  ExitOnErr(JIT->addIRModule(std::move(TSM)));

  // Each anonymous expression is lowered to "__anon_expr"
  auto Sym = ExitOnErr(JIT->lookup("__anon_expr"));
  auto *Fn = Sym.toPtr<double()>();
  llvm::outs() << "= " << Fn() << "\n";
}
```

For a REPL, function definitions accumulate across modules naturally because the `JITDylib` persists across iterations.

---

## Step 9 — Multiple JITDylibs (namespacing / sandboxing)

```cpp
auto &ES = JIT->getExecutionSession();
auto &RuntimeJD = ExitOnErr(ES.createJITDylib("runtime"));

// Load runtime module into RuntimeJD
ExitOnErr(JIT->addIRModule(RuntimeJD, std::move(RuntimeTSM)));

// User code in MainJD can see RuntimeJD symbols
auto &MainJD = JIT->getMainJITDylib();
MainJD.addToLinkOrder(RuntimeJD);

// Load user module into MainJD
ExitOnErr(JIT->addIRModule(MainJD, std::move(UserTSM)));
```

---

## Common mistakes

- **Do NOT** use MCJIT — it is deprecated; ORC v2 (`LLJIT`/`LLLazyJIT`) is the correct API in LLVM 22.
- **Do NOT** call `InitializeNativeTarget()` after creating a JIT instance — it must come first.
- **Do NOT** forget `InitializeNativeTargetAsmPrinter()` — the JIT cannot emit code without it.
- **Do NOT** share an `LLVMContext` between threads without `ThreadSafeContext` — use one context per `ThreadSafeModule`.
- **Do NOT** hold raw function pointers past the lifetime of the `JIT` object — they point into JIT'd memory that is freed on destruction.
- **ALWAYS** add `DynamicLibrarySearchGenerator` if JIT'd code calls any libc/runtime functions.
- **ALWAYS** use `ExitOnError` or handle `llvm::Expected<T>` / `llvm::Error` — ORC never silently discards errors.
