---
name: add-debug-info
description: Add DWARF debug info to an existing LLVM 20 IR frontend. Covers DIBuilder setup, compile unit, function subprograms, local variable declarations, source locations on instructions, and module flags.
---

# Skill: Add Debug Info to an LLVM Frontend (LLVM 20)

Use this skill when the user wants to add debuggable output (source-line mapping, variable inspection in gdb/lldb) to an existing LLVM IR code generator.

---

## Step 0 — Prerequisites

- You have an existing `Module`, `IRBuilder<>`, and functions already emitting IR.
- LLVM must be built with debug info support (default in all standard installs).
- No extra CMake components needed — `DIBuilder` is in `LLVMCore`.

---

## Step 1 — Add DIBuilder and compile unit

In your CodeGen context, add:

```cpp
#include "llvm/IR/DIBuilder.h"
#include "llvm/IR/DebugInfoMetadata.h"

// Add to CodeGenCtx or equivalent:
std::unique_ptr<llvm::DIBuilder> DBuilder;
llvm::DICompileUnit *CU = nullptr;
llvm::DIFile *MainFile  = nullptr;
```

Initialize before emitting any IR:

```cpp
DBuilder = std::make_unique<llvm::DIBuilder>(*M);

MainFile = DBuilder->createFile(
    "myfile.my",          // filename (basename)
    "/path/to/source"     // directory
);

CU = DBuilder->createCompileUnit(
    llvm::dwarf::DW_LANG_C,   // closest standard language code
    MainFile,
    "MyLang Compiler 1.0",    // producer
    /*isOptimized=*/false,
    /*Flags=*/"",
    /*RuntimeVersion=*/0
);

// Required module flags — add immediately after creating CU
M->addModuleFlag(llvm::Module::Warning, "Debug Info Version",
                 llvm::DEBUG_METADATA_VERSION);
M->addModuleFlag(llvm::Module::Warning, "Dwarf Version", 5);
```

---

## Step 2 — Create debug types

Create a `DIType *` for each source-language type. Do this once and cache:

```cpp
std::map<std::string, llvm::DIType *> DITypeCache;

llvm::DIType *getOrCreateDIType(const std::string &Name) {
  auto It = DITypeCache.find(Name);
  if (It != DITypeCache.end()) return It->second;

  llvm::DIType *T = nullptr;
  if      (Name == "int")    T = DBuilder->createBasicType("int",  32, llvm::dwarf::DW_ATE_signed);
  else if (Name == "bool")   T = DBuilder->createBasicType("bool",  1, llvm::dwarf::DW_ATE_boolean);
  else if (Name == "float")  T = DBuilder->createBasicType("float",64, llvm::dwarf::DW_ATE_float);
  else if (Name == "byte")   T = DBuilder->createBasicType("byte",  8, llvm::dwarf::DW_ATE_unsigned);
  // Add struct/array/pointer types similarly

  DITypeCache[Name] = T;
  return T;
}
```

---

## Step 3 — Attach subprogram to each function

Do this at the start of emitting each function, before the entry block:

```cpp
llvm::DISubprogram *beginFunction(llvm::Function *F,
                                   const FuncDecl &Decl,
                                   unsigned Line) {
  // Build subroutine type: [retType, param1Type, param2Type, ...]
  llvm::SmallVector<llvm::Metadata *, 8> EltTys;
  EltTys.push_back(getOrCreateDIType(Decl.retType));
  for (auto &P : Decl.params)
    EltTys.push_back(getOrCreateDIType(P.typeName));
  auto *FnDITy = DBuilder->createSubroutineType(
      DBuilder->getOrCreateTypeArray(EltTys));

  auto *SP = DBuilder->createFunction(
      CU,                                          // scope
      Decl.name,                                   // name
      F->getName(),                                // linkage name
      MainFile,
      Line,                                        // declaration line
      FnDITy,
      Line,                                        // scope line (opening brace)
      llvm::DINode::FlagPrototyped,
      llvm::DISubprogram::SPFlagDefinition
  );

  F->setSubprogram(SP);
  return SP;
}
```

---

## Step 4 — Declare parameters as debug variables

After creating allocas for parameters in the entry block:

```cpp
void declareParam(llvm::AllocaInst *Alloca, const std::string &Name,
                  unsigned ArgNo, unsigned Line,
                  llvm::DISubprogram *SP) {
  auto *DIVar = DBuilder->createParameterVariable(
      SP, Name, ArgNo, MainFile, Line,
      getOrCreateDIType(lookupParamTypeName(Name)),
      /*AlwaysPreserve=*/true  // keep even if optimized away
  );
  // insertDeclare returns DbgInstPtr in LLVM 22 — usually ignored
  DBuilder->insertDeclare(
      Alloca, DIVar, DBuilder->createExpression(),
      llvm::DILocation::get(M->getContext(), Line, /*Col=*/1, SP),
      &*F->getEntryBlock().getFirstInsertionPt()
  );
}
```

---

## Step 5 — Declare local variables

When emitting a `let x = ...` declaration:

```cpp
void declareLocal(llvm::AllocaInst *Alloca, const std::string &Name,
                  unsigned Line, llvm::DISubprogram *SP) {
  auto *DIVar = DBuilder->createAutoVariable(
      SP, Name, MainFile, Line,
      getOrCreateDIType(lookupLocalTypeName(Name)),
      /*AlwaysPreserve=*/false
  );
  DBuilder->insertDeclare(
      Alloca, DIVar, DBuilder->createExpression(),
      llvm::DILocation::get(M->getContext(), Line, 1, SP),
      B.GetInsertBlock()
  );
}
```

---

## Step 6 — Attach source locations to instructions

The most impactful step for debugger stepping — every statement should carry a `!dbg` location.

```cpp
// Set location on IRBuilder — all subsequent instructions inherit it
void setLocation(unsigned Line, unsigned Col, llvm::DISubprogram *SP) {
  B.SetCurrentDebugLocation(
      llvm::DILocation::get(M->getContext(), Line, Col, SP));
}

// Clear for compiler-synthesized code (alloca prologue, mem2reg artifacts)
void clearLocation() {
  B.SetCurrentDebugLocation(llvm::DebugLoc());
}
```

**Pattern — surround each statement emission:**

```cpp
// Before emitting each statement:
setLocation(Stmt.line, Stmt.col, SP);

// Emit alloca prologue without location (avoid mis-stepping):
clearLocation();
AllocaInst *Slot = createEntryAlloca(F, Ty, Name);
// ... store initial value ...

// Restore location for the actual statement:
setLocation(Stmt.line, Stmt.col, SP);
```

---

## Step 7 — Finalize

**Critical:** call `DBuilder->finalize()` after all functions are emitted, before printing/compiling the module:

```cpp
// After emitting all functions and global variables:
DBuilder->finalize();

// Then verify
if (llvm::verifyModule(*M, &llvm::errs()))
  llvm::report_fatal_error("Module verification failed after debug info");
```

---

## Step 8 — Verify debug info with llvm-dwarfdump

```bash
# Compile to object
clang -c output.ll -o output.o

# Inspect DWARF
llvm-dwarfdump output.o

# Check for errors
llvm-dwarfdump --verify output.o

# Check line table
llvm-dwarfdump --debug-line output.o
```

Expected: each source line you marked with `setLocation` appears in the `.debug_line` section.

---

## Step 9 — Test with a debugger

```bash
# Compile with debug info preserved
clang -g output.ll -o my_program

# Debug
lldb my_program
# (lldb) b main
# (lldb) run
# (lldb) n          ← step over (should show source lines)
# (lldb) p x        ← print variable x (requires Step 4/5)
```

---

## Common mistakes

- **Do NOT** forget `DBuilder->finalize()` — most common mistake; causes silent truncation of debug metadata.
- **Do NOT** attach locations to alloca instructions in the entry block — this confuses debuggers by making the frame setup appear as source code.
- **Do NOT** reuse the same `DILocation` object across all instructions — create a new one per source location.
- **Do NOT** set `AlwaysPreserve=false` for parameters — optimizers may eliminate them from debug info; use `true` for all parameters.
- **ALWAYS** add `Debug Info Version` and `Dwarf Version` module flags before `finalize()`.
- **ALWAYS** clear the debug location (`clearLocation()`) for compiler-synthesized instructions not present in source.
