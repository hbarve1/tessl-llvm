---
name: frontend-to-ir
description: Lower a toy language AST to LLVM 20 IR using IRBuilder. Covers project setup, AST node types, expression lowering, control flow, functions, structs, and applying mem2reg.
---

# Skill: Lower a Language Frontend to LLVM IR (LLVM 20)

Use this skill when the user wants to emit LLVM IR from a language AST, expression tree, or bytecode interpreter. This produces a working, verifiable LLVM Module from source-language constructs.

---

## Step 0 — Establish context and module

```cpp
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Verifier.h"
#include "llvm/Support/InitLLVM.h"

// Typically held in a CodeGen context struct:
struct CodeGenCtx {
  llvm::LLVMContext Ctx;
  std::unique_ptr<llvm::Module> M;
  llvm::IRBuilder<> B;
  // Symbol table: name → alloca (for local variables)
  std::map<std::string, llvm::AllocaInst *> Locals;

  CodeGenCtx(llvm::StringRef ModName)
      : M(std::make_unique<llvm::Module>(ModName, Ctx)), B(Ctx) {}
};
```

---

## Step 1 — Map source types to LLVM types

Create a helper that converts your AST type to an LLVM `Type *`:

```cpp
llvm::Type *toLLVMType(CodeGenCtx &CG, ASTType T) {
  switch (T.kind) {
  case ASTType::Bool:    return CG.B.getInt1Ty();
  case ASTType::Int32:   return CG.B.getInt32Ty();
  case ASTType::Int64:   return CG.B.getInt64Ty();
  case ASTType::Float64: return CG.B.getDoubleTy();
  case ASTType::Void:    return CG.B.getVoidTy();
  case ASTType::Ptr:
    return llvm::PointerType::get(CG.Ctx, 0); // opaque ptr
  case ASTType::Struct:
    return lookupOrCreateStructType(CG, T.name);
  }
}
```

---

## Step 2 — Emit a function definition

```cpp
llvm::Function *emitFunction(CodeGenCtx &CG, const FuncDecl &Decl) {
  // Build LLVM function type
  llvm::SmallVector<llvm::Type *, 8> ParamTys;
  for (auto &P : Decl.params)
    ParamTys.push_back(toLLVMType(CG, P.type));
  auto *FT = llvm::FunctionType::get(toLLVMType(CG, Decl.retType),
                                      ParamTys, /*vararg=*/false);

  // Create function (ExternalLinkage for public, InternalLinkage for private)
  auto Linkage = Decl.isPublic ? llvm::Function::ExternalLinkage
                               : llvm::Function::InternalLinkage;
  auto *F = llvm::Function::Create(FT, Linkage, Decl.name, *CG.M);

  // Name parameters
  unsigned I = 0;
  for (auto &Arg : F->args())
    Arg.setName(Decl.params[I++].name);

  // Entry block
  auto *EntryBB = llvm::BasicBlock::Create(CG.Ctx, "entry", F);
  CG.B.SetInsertPoint(EntryBB);
  CG.Locals.clear();

  // Alloca for each parameter (enables mem2reg and mutable params)
  for (auto &Arg : F->args()) {
    auto *Slot = CG.B.CreateAlloca(Arg.getType(), nullptr, Arg.getName());
    CG.B.CreateStore(&Arg, Slot);
    CG.Locals[std::string(Arg.getName())] = Slot;
  }

  // Emit body
  emitBlock(CG, F, Decl.body);

  // Ensure terminator on entry block if body didn't add one
  if (!CG.B.GetInsertBlock()->getTerminator()) {
    if (Decl.retType.kind == ASTType::Void)
      CG.B.CreateRetVoid();
    else
      CG.B.CreateUnreachable(); // missing return — caught by verifier
  }

  llvm::verifyFunction(*F, &llvm::errs());
  return F;
}
```

---

## Step 3 — Emit expressions

```cpp
llvm::Value *emitExpr(CodeGenCtx &CG, const Expr &E) {
  switch (E.kind) {

  // Integer literal
  case Expr::IntLit:
    return CG.B.getInt32(E.intVal);

  // Float literal
  case Expr::FloatLit:
    return llvm::ConstantFP::get(CG.B.getDoubleTy(), E.floatVal);

  // Bool literal
  case Expr::BoolLit:
    return E.boolVal ? CG.B.getTrue() : CG.B.getFalse();

  // Variable read — load from alloca
  case Expr::Var: {
    auto *Slot = CG.Locals.at(E.name);
    return CG.B.CreateLoad(Slot->getAllocatedType(), Slot, E.name);
  }

  // Binary operation
  case Expr::BinOp:
    return emitBinOp(CG, E);

  // Unary negation
  case Expr::Neg:
    return CG.B.CreateNeg(emitExpr(CG, *E.operand), "neg");

  // Function call
  case Expr::Call: {
    auto *Callee = CG.M->getFunction(E.callee);
    llvm::SmallVector<llvm::Value *, 8> Args;
    for (auto &A : E.args) Args.push_back(emitExpr(CG, A));
    return CG.B.CreateCall(Callee->getFunctionType(), Callee, Args, "call");
  }

  // Struct field access: obj.field
  case Expr::Field: {
    auto *Obj  = CG.Locals.at(E.objName);
    auto *STy  = lookupStructType(CG, E.structName);
    auto *FPtr = CG.B.CreateStructGEP(STy, Obj, E.fieldIndex, "field");
    return CG.B.CreateLoad(toLLVMType(CG, E.fieldType), FPtr, E.fieldName);
  }

  // Array subscript: arr[i]
  case Expr::Index: {
    auto *Arr    = emitExpr(CG, *E.array);
    auto *Idx    = emitExpr(CG, *E.index);
    auto *ElemTy = toLLVMType(CG, E.elemType);
    auto *Ptr    = CG.B.CreateInBoundsGEP(ElemTy, Arr, {Idx}, "elem");
    return CG.B.CreateLoad(ElemTy, Ptr, "elem.val");
  }

  default:
    llvm_unreachable("unhandled expression kind");
  }
}

llvm::Value *emitBinOp(CodeGenCtx &CG, const Expr &E) {
  llvm::Value *L = emitExpr(CG, *E.lhs);
  llvm::Value *R = emitExpr(CG, *E.rhs);
  switch (E.op) {
  case Op::Add:  return CG.B.CreateAdd(L, R, "add");
  case Op::Sub:  return CG.B.CreateSub(L, R, "sub");
  case Op::Mul:  return CG.B.CreateMul(L, R, "mul");
  case Op::Div:  return CG.B.CreateSDiv(L, R, "div");
  case Op::Eq:   return CG.B.CreateICmpEQ(L, R, "eq");
  case Op::Lt:   return CG.B.CreateICmpSLT(L, R, "lt");
  case Op::And:  return CG.B.CreateAnd(L, R, "and");
  case Op::Or:   return CG.B.CreateOr(L, R, "or");
  default:       llvm_unreachable("unhandled binary operator");
  }
}
```

---

## Step 4 — Emit statements

```cpp
void emitStmt(CodeGenCtx &CG, llvm::Function *F, const Stmt &S) {
  switch (S.kind) {

  // Variable declaration: let x: T = expr
  case Stmt::VarDecl: {
    auto *Ty   = toLLVMType(CG, S.varType);
    auto *Slot = createEntryAlloca(F, Ty, S.varName);
    CG.Locals[S.varName] = Slot;
    if (S.init)
      CG.B.CreateStore(emitExpr(CG, *S.init), Slot);
    break;
  }

  // Assignment: x = expr
  case Stmt::Assign: {
    auto *Val  = emitExpr(CG, *S.value);
    auto *Slot = CG.Locals.at(S.varName);
    CG.B.CreateStore(Val, Slot);
    break;
  }

  // Return statement
  case Stmt::Return:
    if (S.value)
      CG.B.CreateRet(emitExpr(CG, *S.value));
    else
      CG.B.CreateRetVoid();
    // Create an unreachable block so subsequent code has somewhere to go
    CG.B.SetInsertPoint(
        llvm::BasicBlock::Create(CG.Ctx, "after.ret", F));
    break;

  // Expression statement (e.g., function call for side effects)
  case Stmt::ExprStmt:
    emitExpr(CG, *S.expr);
    break;

  // if / if-else
  case Stmt::If: {
    llvm::Value *Cond   = emitExpr(CG, *S.cond);
    auto *ThenBB  = llvm::BasicBlock::Create(CG.Ctx, "if.then", F);
    auto *MergeBB = llvm::BasicBlock::Create(CG.Ctx, "if.merge", F);
    auto *ElseBB  = S.elseBody ? llvm::BasicBlock::Create(CG.Ctx, "if.else", F)
                               : MergeBB;
    CG.B.CreateCondBr(Cond, ThenBB, ElseBB);

    CG.B.SetInsertPoint(ThenBB);
    emitBlock(CG, F, S.thenBody);
    if (!CG.B.GetInsertBlock()->getTerminator()) CG.B.CreateBr(MergeBB);

    if (S.elseBody) {
      CG.B.SetInsertPoint(ElseBB);
      emitBlock(CG, F, *S.elseBody);
      if (!CG.B.GetInsertBlock()->getTerminator()) CG.B.CreateBr(MergeBB);
    }
    CG.B.SetInsertPoint(MergeBB);
    break;
  }

  // while loop
  case Stmt::While: {
    auto *CondBB = llvm::BasicBlock::Create(CG.Ctx, "while.cond", F);
    auto *BodyBB = llvm::BasicBlock::Create(CG.Ctx, "while.body", F);
    auto *ExitBB = llvm::BasicBlock::Create(CG.Ctx, "while.exit", F);
    CG.B.CreateBr(CondBB);

    CG.B.SetInsertPoint(CondBB);
    CG.B.CreateCondBr(emitExpr(CG, *S.cond), BodyBB, ExitBB);

    CG.B.SetInsertPoint(BodyBB);
    emitBlock(CG, F, S.loopBody);
    if (!CG.B.GetInsertBlock()->getTerminator()) CG.B.CreateBr(CondBB);

    CG.B.SetInsertPoint(ExitBB);
    break;
  }

  default:
    llvm_unreachable("unhandled statement kind");
  }
}

void emitBlock(CodeGenCtx &CG, llvm::Function *F,
               const std::vector<Stmt> &Stmts) {
  for (auto &S : Stmts)
    emitStmt(CG, F, S);
}

// Always alloca in entry block
llvm::AllocaInst *createEntryAlloca(llvm::Function *F, llvm::Type *Ty,
                                     llvm::StringRef Name) {
  llvm::IRBuilder<> EB(&F->getEntryBlock(), F->getEntryBlock().begin());
  return EB.CreateAlloca(Ty, nullptr, Name);
}
```

---

## Step 5 — Emit the whole module

```cpp
std::unique_ptr<llvm::Module> emitModule(const Program &Prog) {
  CodeGenCtx CG("my_lang_module");

  // First pass: declare all functions (allows mutual recursion)
  for (auto &FD : Prog.functions) {
    llvm::SmallVector<llvm::Type *, 8> PTys;
    for (auto &P : FD.params) PTys.push_back(toLLVMType(CG, P.type));
    auto *FT = llvm::FunctionType::get(toLLVMType(CG, FD.retType), PTys, false);
    llvm::Function::Create(FT, llvm::Function::ExternalLinkage, FD.name, *CG.M);
  }

  // Second pass: emit bodies
  for (auto &FD : Prog.functions)
    emitFunction(CG, FD);

  // Verify entire module
  if (llvm::verifyModule(*CG.M, &llvm::errs()))
    llvm::report_fatal_error("IR verification failed");

  return std::move(CG.M);
}
```

---

## Step 6 — Apply mem2reg and basic cleanup

After emitting the full module, run `PromotePass` (mem2reg) to convert
alloca/load/store chains to SSA `phi` nodes:

```cpp
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Transforms/Scalar/Mem2Reg.h"
#include "llvm/Transforms/InstCombine/InstCombine.h"
#include "llvm/Transforms/Scalar/SimplifyCFG.h"

void runCleanupPasses(llvm::Module &M) {
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

  llvm::FunctionPassManager FPM;
  FPM.addPass(llvm::PromotePass());        // alloca → SSA phi
  FPM.addPass(llvm::InstCombinePass());    // constant folding, trivial simplification
  FPM.addPass(llvm::SimplifyCFGPass());    // remove empty blocks, merge redundant branches

  llvm::ModulePassManager MPM;
  MPM.addPass(llvm::createModuleToFunctionPassAdaptor(std::move(FPM)));
  MPM.run(M, MAM);
}
```

---

## Step 7 — Print or compile

```cpp
// Print IR to stdout
M->print(llvm::outs(), nullptr);

// Write bitcode
#include "llvm/Bitcode/BitcodeWriter.h"
std::error_code EC;
llvm::raw_fd_ostream Out("output.bc", EC);
llvm::WriteBitcodeToFile(*M, Out);

// Compile to object file — see out-of-tree-setup skill for TargetMachine setup
```

---

## Common mistakes

- **Do NOT** create allocas anywhere except the entry block — use `createEntryAlloca`. `mem2reg` only promotes entry-block allocas.
- **Do NOT** emit `phi` nodes by hand during initial lowering — use alloca + `mem2reg` instead. Phi construction requires knowing all predecessors upfront.
- **Do NOT** call `verifyModule` only at the end — call `verifyFunction` after each function to localize bugs.
- **Do NOT** forget to declare all functions before emitting bodies — mutual recursion requires forward declarations.
- **ALWAYS** check `getTerminator()` before adding a branch — adding two terminators to a block is an IR invariant violation.
- **ALWAYS** call `B.GetInsertBlock()` (not save a stale pointer) after nested emission — the insert block can change.
