# Debug Info (DWARF) — DIBuilder (LLVM 22)

Reference: [SourceLevelDebugging](https://llvm.org/docs/SourceLevelDebugging.html) | [DIBuilder Doxygen](https://llvm.org/doxygen/classllvm_1_1DIBuilder.html)

---

## Overview

LLVM debug info is represented as metadata nodes attached to IR. `DIBuilder` is the API to create them. The generated metadata follows the **DWARF** standard and is consumed by debuggers (gdb, lldb).

Key concepts:

| Class | DWARF equivalent | Purpose |
|-------|-----------------|---------|
| `DICompileUnit` | `DW_TAG_compile_unit` | Top-level unit; one per source file |
| `DIFile` | `DW_TAG_file_type` | A source file reference |
| `DISubprogram` | `DW_TAG_subprogram` | A function / method |
| `DILexicalBlock` | `DW_TAG_lexical_block` | A `{}` scope |
| `DILocalVariable` | `DW_TAG_variable` | A local variable or parameter |
| `DIGlobalVariable` | `DW_TAG_variable` (global) | A global variable |
| `DIType` (and subtypes) | `DW_TAG_base_type` etc. | Type descriptors |
| `DILocation` | `DW_AT_location` | Source location attached to an instruction |

---

## Setup: create DIBuilder and compile unit

```cpp
#include "llvm/IR/DIBuilder.h"
#include "llvm/IR/DebugInfoMetadata.h"

// One DIBuilder per module
llvm::DIBuilder DBuilder(*M);

// One DIFile per source file
llvm::DIFile *File = DBuilder.createFile("myfile.my", "/path/to/src");

// One DICompileUnit per compilation unit
llvm::DICompileUnit *CU = DBuilder.createCompileUnit(
    llvm::dwarf::DW_LANG_C,   // use closest standard language, or DW_LANG_lo_user
    File,
    "MyLang Compiler 1.0",    // producer string
    /*isOptimized=*/false,
    /*Flags=*/"",
    /*RuntimeVersion=*/0
);

// After all debug info is created — MUST be called before module emission
DBuilder.finalize();
```

---

## DITypes

### Basic types

```cpp
// Primitive types: name, size in bits, DWARF encoding
llvm::DIType *Int32Ty = DBuilder.createBasicType(
    "int", 32, llvm::dwarf::DW_ATE_signed);
llvm::DIType *UInt8Ty = DBuilder.createBasicType(
    "byte", 8, llvm::dwarf::DW_ATE_unsigned);
llvm::DIType *BoolTy  = DBuilder.createBasicType(
    "bool", 1, llvm::dwarf::DW_ATE_boolean);
llvm::DIType *F64Ty   = DBuilder.createBasicType(
    "float64", 64, llvm::dwarf::DW_ATE_float);
```

### Pointer types

```cpp
// ptr to Int32Ty, 64-bit pointer size
llvm::DIType *PtrTy = DBuilder.createPointerType(Int32Ty, /*SizeInBits=*/64);
```

### Struct types

```cpp
llvm::DICompositeType *StructTy = DBuilder.createStructType(
    CU,                              // scope
    "Point",                         // name
    File, /*Line=*/5,                // declaration location
    /*SizeInBits=*/64,
    /*AlignInBits=*/32,
    llvm::DINode::FlagZero,
    /*DerivedFrom=*/nullptr,
    DBuilder.getOrCreateArray({})    // members filled in below
);

// Create member fields
auto *XMember = DBuilder.createMemberType(
    StructTy, "x", File, /*Line=*/6,
    /*SizeInBits=*/32, /*AlignInBits=*/32, /*OffsetInBits=*/0,
    llvm::DINode::FlagZero, Int32Ty);
auto *YMember = DBuilder.createMemberType(
    StructTy, "y", File, 7, 32, 32, /*OffsetInBits=*/32,
    llvm::DINode::FlagZero, Int32Ty);

// Replace empty member list with actual members
DBuilder.replaceArrays(StructTy,
    DBuilder.getOrCreateArray({XMember, YMember}));
```

### Array types

```cpp
// int[10]: element type Int32Ty, count 10
llvm::DICompositeType *ArrTy = DBuilder.createArrayType(
    /*SizeInBits=*/320,
    /*AlignInBits=*/32,
    Int32Ty,
    DBuilder.getOrCreateArray({DBuilder.getOrCreateSubrange(0, 10)})
);
```

---

## Subprograms (functions)

```cpp
// Build subroutine type: (int, int) -> int
llvm::DISubroutineType *FnDITy = DBuilder.createSubroutineType(
    DBuilder.getOrCreateTypeArray({Int32Ty, Int32Ty, Int32Ty})
    // First element is return type, rest are parameter types
);

// Create the subprogram metadata
llvm::DISubprogram *SP = DBuilder.createFunction(
    CU,                          // scope (CU or parent DIType for methods)
    "add",                       // function name
    "add",                       // linkage name (mangled name if applicable)
    File,
    /*Line=*/1,                  // declaration line
    FnDITy,
    /*ScopeLine=*/1,             // line of opening brace
    llvm::DINode::FlagPrototyped,
    llvm::DISubprogram::SPFlagDefinition
    // New in LLVM 22: optional UseKeyInstructions=false (last param, default safe)
);

// Attach to LLVM Function
F->setSubprogram(SP);

// Push scope for lexical block tracking
DBuilder.finalizeSubprogram(SP); // call after emitting the function body
```

---

## Local variables and parameters

```cpp
// Parameter (ArgNo >= 1; 0 means local variable)
llvm::DILocalVariable *ParamX = DBuilder.createParameterVariable(
    SP,           // scope: the subprogram
    "x",          // name
    /*ArgNo=*/1,  // 1-based parameter index
    File,
    /*Line=*/1,
    Int32Ty,
    /*AlwaysPreserve=*/true
);

// Local variable
llvm::DILocalVariable *LocalVar = DBuilder.createAutoVariable(
    SP,           // scope
    "result",
    File,
    /*Line=*/3,
    Int32Ty,
    /*AlwaysPreserve=*/false
);

// Insert llvm.dbg.declare — binds debug variable to an alloca address
// LLVM 22: returns DbgInstPtr (PointerUnion<Instruction*, DbgRecord*>) — usually ignored
DBuilder.insertDeclare(
    Alloca,       // the alloca holding the variable
    ParamX,       // DILocalVariable
    DBuilder.createExpression(),  // empty = no indirection
    llvm::DILocation::get(Ctx, /*Line=*/1, /*Col=*/1, SP),
    &*F->getEntryBlock().getFirstInsertionPt()
);

// Insert llvm.dbg.value — binds debug variable to an SSA value (after mem2reg)
// LLVM 22: returns DbgInstPtr — usually ignored
DBuilder.insertDbgValueIntrinsic(
    Val,          // the SSA value
    LocalVar,
    DBuilder.createExpression(),
    llvm::DILocation::get(Ctx, 3, 5, SP),
    InsertBefore
);

// NEW in LLVM 22: insertDbgAssign — assignment tracking
// Links a store instruction to the debug variable it assigns
DBuilder.insertDbgAssign(
    StoreInst,       // the store instruction that performs the assignment
    Val,             // the value being stored
    LocalVar,        // DILocalVariable
    DBuilder.createExpression(),   // value expression
    Ptr,             // address operand (alloca or heap ptr)
    DBuilder.createExpression(),   // address expression
    llvm::DILocation::get(Ctx, Line, Col, SP)
);
```

---

## Source locations on instructions

Every instruction that should map to a source line needs a `!dbg` attachment:

```cpp
// Create a DILocation
llvm::DILocation *Loc = llvm::DILocation::get(Ctx,
    /*Line=*/10, /*Column=*/5, SP);

// Attach to builder — all subsequent instructions get this location
B.SetCurrentDebugLocation(Loc);

// Or attach to a specific instruction after creation:
Inst->setDebugLoc(Loc);

// Clear location (for compiler-synthesized code with no source line)
B.SetCurrentDebugLocation(llvm::DebugLoc());
```

---

## Lexical scopes

```cpp
// { ... } block inside a function
llvm::DILexicalBlock *Block = DBuilder.createLexicalBlock(
    SP,       // parent scope
    File,
    /*Line=*/5, /*Col=*/1
);

// Use Block as the scope for DILocalVariables inside the { }
// and as the scope argument to DILocation::get()
```

---

## Global variables

```cpp
llvm::DIGlobalVariableExpression *DGVE =
    DBuilder.createGlobalVariableExpression(
        CU,
        "globalCounter",
        "globalCounter",          // linkage name
        File, /*Line=*/1,
        Int32Ty,
        /*IsLocalToUnit=*/false,
        /*IsDefined=*/true
    );

// Attach to GlobalVariable
GV->addDebugInfo(DGVE);
```

---

## Module flags

Set module-level debug info flags — required for most debuggers:

```cpp
// Emit dwarf version and debug info format flags
M->addModuleFlag(llvm::Module::Warning, "Dwarf Version", 5);      // DWARF 5
M->addModuleFlag(llvm::Module::Warning, "Debug Info Version",
                 llvm::DEBUG_METADATA_VERSION);                    // must match
M->addModuleFlag(llvm::Module::Max,     "frame-pointer",
                 llvm::ConstantInt::get(llvm::Type::getInt32Ty(Ctx), 2));
```

---

## Full example: function with one parameter and local variable

```cpp
llvm::DIBuilder DBuilder(*M);
llvm::DIFile *File  = DBuilder.createFile("hello.my", ".");
llvm::DICompileUnit *CU = DBuilder.createCompileUnit(
    llvm::dwarf::DW_LANG_C, File, "MyLang", false, "", 0);

llvm::DIType *I32DI = DBuilder.createBasicType("int", 32,
                                                llvm::dwarf::DW_ATE_signed);
llvm::DISubroutineType *FnTy = DBuilder.createSubroutineType(
    DBuilder.getOrCreateTypeArray({I32DI, I32DI})); // ret=i32, arg=i32

llvm::DISubprogram *SP = DBuilder.createFunction(
    CU, "square", "square", File, 1, FnTy, 1,
    llvm::DINode::FlagPrototyped,
    llvm::DISubprogram::SPFlagDefinition);
F->setSubprogram(SP);

// Declare parameter
llvm::DILocalVariable *ParamN = DBuilder.createParameterVariable(
    SP, "n", 1, File, 1, I32DI, true);
DBuilder.insertDeclare(NSlot, ParamN, DBuilder.createExpression(),
    llvm::DILocation::get(Ctx, 1, 1, SP),
    &*F->getEntryBlock().getFirstInsertionPt());

// Set location on the multiply instruction
B.SetCurrentDebugLocation(llvm::DILocation::get(Ctx, 2, 3, SP));
Value *Result = B.CreateMul(N, N, "sq");

M->addModuleFlag(llvm::Module::Warning, "Debug Info Version",
                 llvm::DEBUG_METADATA_VERSION);
M->addModuleFlag(llvm::Module::Warning, "Dwarf Version", 5);

DBuilder.finalize();
```

---

## Common mistakes

- **Do NOT** forget `DBuilder.finalize()` — without it, debug metadata is incomplete and debuggers may crash or show garbage.
- **Do NOT** set `DISubprogram::SPFlagDefinition` on declarations (function prototypes without body) — use `SPFlagZero` for declarations.
- **Do NOT** attach `DILocation` from an outer scope to instructions inside a nested `DILexicalBlock` — the scope chain must be consistent.
- **Do NOT** use `DW_LANG_C` for non-C languages if DWARF defines a closer language tag; use `DW_LANG_lo_user`–`DW_LANG_hi_user` range for custom languages.
- **ALWAYS** add module flags for `Debug Info Version` and `Dwarf Version` — missing flags cause `llvm-dwarfdump` and debuggers to reject the binary.
- **ALWAYS** clear the debug location (`B.SetCurrentDebugLocation({})`) for compiler-synthesized instructions to prevent incorrect line mappings.
