# Migrate Legacy LLVM Tool to LLVM 22

## Problem/Feature Description

An open source tool called `ir-stats` analyzes LLVM IR files and prints statistics about their contents. It was last updated for LLVM 18 and has accumulated several API usages that no longer compile against LLVM 22. The maintainer wants to cut a new release supporting LLVM 22 and has asked for a complete migration of the source files.

Your task is to produce corrected versions of the files below so they compile cleanly against LLVM 22. You should also produce a `MIGRATION.md` file documenting the changes you made and why, so other contributors can learn from them.

## Input Files

The following files are provided as inputs. Extract them before beginning.

=============== FILE: inputs/CMakeLists.txt ===============
cmake_minimum_required(VERSION 3.16)
project(ir-stats CXX)

find_package(LLVM 18 REQUIRED CONFIG)

include_directories(${LLVM_INCLUDE_DIRS})
add_definitions(${LLVM_DEFINITIONS})

target_link_libraries(ir-stats PRIVATE
  -lLLVMCore
  -lLLVMSupport
  -lLLVMIRReader
  -lLLVMAnalysis
)

add_executable(ir-stats src/main.cpp src/StatsPass.cpp)
=============== END FILE ===============

=============== FILE: inputs/src/StatsPass.h ===============
#pragma once
#include "llvm/IR/PassManager.h"
#include "llvm/ADT/Triple.h"
#include "llvm/Support/Host.h"
#include <string>
using namespace llvm;

struct StatsResult {
  unsigned NumFunctions;
  unsigned NumInstructions;
};

class StatsPass : public FunctionPass {
public:
  static char ID;
  StatsPass() : FunctionPass(ID) {}
  bool runOnFunction(Function &F) override;
  void getAnalysisUsage(AnalysisUsage &AU) const override {
    AU.setPreservesAll();
  }
  StatsResult Result;
};
=============== END FILE ===============

=============== FILE: inputs/src/StatsPass.cpp ===============
#include "StatsPass.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/Instructions.h"

char StatsPass::ID = 0;

bool StatsPass::runOnFunction(Function &F) {
  Result.NumFunctions++;
  for (auto &BB : F)
    for (auto &I : BB)
      Result.NumInstructions++;
  return false;
}
=============== END FILE ===============

=============== FILE: inputs/src/main.cpp ===============
#include "StatsPass.h"
#include "llvm/ADT/Optional.h"
#include "llvm/IR/LegacyPassManager.h"
#include "llvm/IR/Module.h"
#include "llvm/IRReader/IRReader.h"
#include "llvm/Support/SourceMgr.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/Support/TargetSelect.h"
#include "llvm/IR/Intrinsics.h"
#include "llvm/IR/IRBuilder.h"

using namespace llvm;

// Helper: get a declaration for the llvm.sqrt intrinsic
Function *getSqrtDecl(Module &M) {
  return Intrinsic::getDeclaration(&M, Intrinsic::sqrt, {Type::getFloatTy(M.getContext())});
}

// Helper: create a typed pointer to float
Type *getFloatPtrTy(LLVMContext &Ctx) {
  return Type::getFloatTy(Ctx)->getPointerTo();
}

int main(int argc, char **argv) {
  if (argc < 2) {
    errs() << "Usage: ir-stats <file.ll>\n";
    return 1;
  }

  LLVMContext Ctx;
  SMDiagnostic Err;
  auto M = parseIRFile(argv[1], Err, Ctx);
  if (!M) {
    Err.print(argv[0], errs());
    return 1;
  }

  // Get host triple
  Triple HostTriple(sys::getDefaultTargetTriple());
  M->setTargetTriple(HostTriple.getTriple());

  Optional<std::string> OptComment;
  if (argc > 2) OptComment = std::string(argv[2]);

  // Run the stats pass using the legacy pass manager
  legacy::PassManager PM;
  StatsPass *SP = new StatsPass();
  PM.add(SP);
  PM.run(*M);

  outs() << "Functions:    " << SP->Result.NumFunctions    << "\n";
  outs() << "Instructions: " << SP->Result.NumInstructions << "\n";
  if (OptComment.hasValue())
    outs() << "Comment: " << OptComment.getValue() << "\n";

  return 0;
}
=============== END FILE ===============

## Output Specification

Produce corrected versions of all four files:
- `CMakeLists.txt`
- `src/StatsPass.h`
- `src/StatsPass.cpp`
- `src/main.cpp`

Also produce:
- `MIGRATION.md` — a brief document listing each change made and the reason

The corrected files must compile against LLVM 22. The stats functionality (counting functions and instructions) should be preserved, including accepting an optional comment argument.
