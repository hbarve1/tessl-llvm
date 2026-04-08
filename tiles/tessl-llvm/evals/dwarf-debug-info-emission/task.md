# Add Debugger Support to a Toy Language Compiler

## Problem/Feature Description

A small team has built a toy compiler for a language called "Pico" that lowers a simple AST to LLVM IR. The compiler works — it emits correct code — but users are frustrated that they can't use `gdb` or `lldb` to step through their Pico programs at the source level. Every crash shows disassembly with no file names or line numbers.

Your task is to add DWARF debug information to the existing Pico compiler so that:
1. The compiler emits proper DWARF sections in the output
2. Variables declared in Pico source code are visible in the debugger
3. Stepping through code in `lldb` shows the correct Pico source line

The existing compiler has an `LLVMContext`, a `Module`, and an `IRBuilder<>`. Functions are already being emitted with allocas for local variables and parameters. What's missing is the entire debug metadata layer.

## Output Specification

Produce a C++ source file `pico_debug.cpp` that implements the debug info layer for the Pico compiler. It should demonstrate:

1. A `DebugInfoEmitter` class (or equivalent struct) that integrates with the existing CodeGen context
2. Initialization code that sets up the debug info infrastructure before any IR is emitted
3. A `beginFunction()` method that integrates debug metadata with each `llvm::Function`
4. Methods for declaring function parameters and local variables as debuggable entities
5. A method for associating source locations with IR instructions
6. Proper teardown that ensures debug metadata is complete before the module is output

The file should be self-contained and readable without needing to run the compiler — someone reviewing the code should be able to verify the debug API is being used correctly. Include a short `main()` or standalone `demo()` function that exercises the above methods on a trivial example (one function with one parameter and one local variable).
