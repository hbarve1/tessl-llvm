# Code Generation — SelectionDAG, GlobalISel, MachineFunction (LLVM 22)

Reference: [CodeGenerator](https://llvm.org/docs/CodeGenerator.html) | [GlobalISel](https://llvm.org/docs/GlobalISel/index.html)

---

## Code generation pipeline overview

```
LLVM IR (Module/Function)
    │
    ▼ IR-level optimizations (NPM passes)
    │
    ▼ SelectionDAGISel or IRTranslator  ← IR → Machine IR
    │
    ▼ Legalization                      ← expand illegal types/ops
    │
    ▼ Register allocation               ← virtual → physical regs
    │
    ▼ Post-RA passes                    ← peephole, scheduling
    │
    ▼ AsmPrinter / MCStreamer           ← emit .s or .o
```

Two ISel paths exist in LLVM 22:

| Path | Status | Use when |
|------|--------|---------|
| **SelectionDAG** | Mature, widely used | Most existing targets |
| **GlobalISel** | Production-ready for major targets | New targets, AArch64, X86, RISC-V |

---

## TargetMachine

`TargetMachine` is the entry point for all codegen for a target.

```cpp
#include "llvm/MC/TargetRegistry.h"
#include "llvm/Target/TargetMachine.h"
#include "llvm/TargetParser/Host.h"   // sys::getDefaultTargetTriple()

// Look up target by triple
std::string Triple = sys::getDefaultTargetTriple();
std::string Error;
const Target *TheTarget = TargetRegistry::lookupTarget(Triple, Error);
if (!TheTarget) { errs() << Error; return 1; }

// Create TargetMachine
TargetOptions Options;
auto RM = std::optional<Reloc::Model>(Reloc::PIC_);
TargetMachine *TM = TheTarget->createTargetMachine(
    Triple,
    /*CPU=*/"generic",       // or "native", or specific CPU like "cortex-a53"
    /*Features=*/"",         // feature string, e.g. "+avx2,-avx512f"
    Options,
    RM,
    /*CM=*/std::nullopt,
    CodeGenOptLevel::Default
);

// Set data layout on module
M->setDataLayout(TM->createDataLayout());
M->setTargetTriple(Triple);
```

### Emit object code or assembly

```cpp
#include "llvm/CodeGen/TargetPassConfig.h"
#include "llvm/IR/LegacyPassManager.h"  // only for addPassesToEmitFile in LLVM 22

// Emit to file (uses legacy PM internally — this API remains)
SmallVector<char, 0> ObjBuffer;
raw_svector_ostream OS(ObjBuffer);

legacy::PassManager PM;
if (TM->addPassesToEmitFile(PM, OS, nullptr,
                             CodeGenFileType::ObjectFile)) {
  errs() << "Target does not support object emission\n";
  return 1;
}
PM.run(*M);
// ObjBuffer now contains the ELF/Mach-O/COFF object
```

---

## MachineFunction and MachineIR

After instruction selection, IR is lowered to `MachineFunction` / `MachineInstr` / `MachineBasicBlock`.

```cpp
// Access MachineFunction from a MachineFunctionPass
MachineFunction &MF = /* provided by pass */;

// Iterate machine basic blocks
for (MachineBasicBlock &MBB : MF) {
  // Iterate machine instructions
  for (MachineInstr &MI : MBB) {
    unsigned Opcode = MI.getOpcode();
    // Iterate operands
    for (MachineOperand &MO : MI.operands()) {
      if (MO.isReg())  { Register Reg = MO.getReg(); }
      if (MO.isImm())  { int64_t  Val = MO.getImm(); }
      if (MO.isMBB())  { MachineBasicBlock *TBB = MO.getMBB(); }
    }
  }
}

// MachineRegisterInfo — virtual register info
MachineRegisterInfo &MRI = MF.getRegInfo();
Register VReg = MRI.createVirtualRegister(&MyISA::GPRRegClass);
Register PReg = MyISA::R0;  // physical register

// MachineFrameInfo — stack frame
MachineFrameInfo &MFI = MF.getFrameInfo();
int FI = MFI.CreateStackObject(Size, Alignment, /*isSS=*/false);
```

---

## SelectionDAG ISel

SelectionDAG lowers IR to a DAG of `SDNode`s, applies patterns from TableGen, and emits `MachineInstr`s.

### Key classes

| Class | Description |
|-------|-------------|
| `SelectionDAG` | The DAG; owns all SDNodes |
| `SDValue` | A (node, result index) pair — the primary currency |
| `SDNode` | A DAG node (operation + operands + types) |
| `SDLoc` | Source location for an SDNode |

### ISelLowering — custom lowering

Override in `<Target>ISelLowering.cpp`:

```cpp
// Constructor — set up legal types and operations
MyISATargetLowering::MyISATargetLowering(const TargetMachine &TM,
                                          const MyISASubtarget &STI)
    : TargetLowering(TM) {
  addRegisterClass(MVT::i32, &MyISA::GPRRegClass);
  computeRegisterProperties(STI.getRegisterInfo());

  // Mark operations as legal / custom / expand / promote
  setOperationAction(ISD::ADD,      MVT::i32, Legal);
  setOperationAction(ISD::SDIV,     MVT::i32, Expand);   // no hardware div
  setOperationAction(ISD::ROTL,     MVT::i32, Custom);   // handle specially
  setOperationAction(ISD::CTPOP,    MVT::i32, Expand);

  setLoadExtAction(ISD::EXTLOAD,  MVT::i32, MVT::i8,  Expand);
  setTruncStoreAction(MVT::i32, MVT::i8, Expand);
}

// Lower a Custom operation
SDValue MyISATargetLowering::LowerOperation(SDValue Op,
                                             SelectionDAG &DAG) const {
  switch (Op.getOpcode()) {
  case ISD::ROTL: return LowerROTL(Op, DAG);
  default: llvm_unreachable("unimplemented custom lowering");
  }
}

SDValue MyISATargetLowering::LowerROTL(SDValue Op, SelectionDAG &DAG) const {
  SDLoc DL(Op);
  SDValue LHS = Op.getOperand(0);
  SDValue RHS = Op.getOperand(1);
  // Build custom SDNode sequence or MyISAISD node
  return DAG.getNode(MyISAISD::ROTL, DL, Op.getValueType(), LHS, RHS);
}
```

### Calling convention lowering

```cpp
SDValue MyISATargetLowering::LowerFormalArguments(
    SDValue Chain, CallingConv::ID CallConv, bool IsVarArg,
    const SmallVectorImpl<ISD::InputArg> &Ins, const SDLoc &DL,
    SelectionDAG &DAG, SmallVectorImpl<SDValue> &InVals) const {
  // Use TableGen-generated CC_MyISA to assign locations
  SmallVector<CCValAssign, 16> ArgLocs;
  CCState CCInfo(CallConv, IsVarArg, DAG.getMachineFunction(),
                 ArgLocs, *DAG.getContext());
  CCInfo.AnalyzeFormalArguments(Ins, CC_MyISA);

  for (CCValAssign &VA : ArgLocs) {
    if (VA.isRegLoc()) {
      Register VReg = MF.getRegInfo().createVirtualRegister(&MyISA::GPRRegClass);
      MF.getRegInfo().addLiveIn(VA.getLocReg(), VReg);
      SDValue ArgVal = DAG.getCopyFromReg(Chain, DL, VReg, VA.getValVT());
      InVals.push_back(ArgVal);
    } else {
      // Stack argument — load from frame
      // ...
    }
  }
  return Chain;
}
```

---

## GlobalISel

GlobalISel operates on SSA Machine IR (`MachineInstr` with virtual registers) throughout, avoiding the SDNode intermediate representation.

### Pipeline stages

```
IRTranslator        → IR  → generic MachineInstrs (G_ADD, G_LOAD, ...)
Legalizer           → expand/narrow/widen illegal types and ops
RegBankSelect       → assign register banks (GPR vs FPR vs vector)
InstructionSelect   → generic MI → target MI
```

### Key generic opcodes (G_ prefix)

```
G_ADD, G_SUB, G_MUL, G_SDIV, G_UDIV
G_AND, G_OR, G_XOR, G_SHL, G_ASHR, G_LSHR
G_LOAD, G_STORE
G_GEP (→ G_PTR_ADD in LLVM 22)
G_ICMP, G_FCMP
G_BR, G_BRCOND
G_PHI
G_CONSTANT, G_FCONSTANT
G_ZEXT, G_SEXT, G_TRUNC, G_BITCAST
G_CALL, G_RETURN_VALUE
```

### Legalizer

```cpp
// In MyISALegalizerInfo.cpp
MyISALegalizerInfo::MyISALegalizerInfo(const MyISASubtarget &ST) {
  using namespace TargetOpcode;
  const LLT S32 = LLT::scalar(32);
  const LLT S64 = LLT::scalar(64);

  // i32 add is natively legal
  getActionDefinitionsBuilder(G_ADD).legalFor({S32}).clampScalar(0, S32, S32);

  // i64 add: widen to i64 only if supported, otherwise narrow to two i32
  getActionDefinitionsBuilder(G_MUL)
      .legalFor({S32})
      .clampScalar(0, S32, S32);

  // Division not supported — expand to library call
  getActionDefinitionsBuilder(G_SDIV).libcall();

  getLegacyLegalizerInfo().computeTables();
}
```

### InstructionSelect (TableGen-driven)

```cpp
// Implement selectImpl() in MyISAInstructionSelector.cpp
bool MyISAInstructionSelector::select(MachineInstr &I) {
  if (selectImpl(I, *CoverageInfo))  // tries TableGen patterns
    return true;

  // Manual selection for ops without patterns
  switch (I.getOpcode()) {
  case TargetOpcode::G_CONSTANT: {
    int64_t CVal = I.getOperand(1).getCImm()->getSExtValue();
    MachineInstr *NewMI = BuildMI(*I.getParent(), I, I.getDebugLoc(),
                                   TII.get(MyISA::MOVI))
        .addDef(I.getOperand(0).getReg())
        .addImm(CVal);
    I.eraseFromParent();
    return constrainSelectedInstRegOperands(*NewMI, TII, TRI, RBI);
  }
  }
  return false;
}
```

---

## MachineFunctionPass (post-ISel passes)

```cpp
class MyMachinePass : public MachineFunctionPass {
public:
  static char ID;
  MyMachinePass() : MachineFunctionPass(ID) {}

  bool runOnMachineFunction(MachineFunction &MF) override {
    bool Changed = false;
    for (MachineBasicBlock &MBB : MF) {
      for (MachineInstr &MI : MBB) {
        // inspect / modify MI
      }
    }
    return Changed;
  }

  StringRef getPassName() const override { return "My Machine Pass"; }
};
char MyMachinePass::ID = 0;
```

Register in `TargetPassConfig::addPreRegAlloc()` or `addPostRegAlloc()`.

---

## Common mistakes

- **Do NOT** use `SDValue` after calling `DAG.DeleteNode()` or after DAG legalization — nodes can be replaced.
- **Do NOT** forget `computeRegisterProperties()` in `TargetLowering` constructor — without it, legal type tables are incomplete.
- **Do NOT** skip `getLegacyLegalizerInfo().computeTables()` at the end of `LegalizerInfo` constructor.
- **Do NOT** mix GlobalISel and SelectionDAG selection in the same function without care — the two paths are mutually exclusive per function.
- **ALWAYS** use `BuildMI(MBB, InsertPt, DL, TII.get(Opc))` to build new MachineInstrs — never construct them directly.
- **ALWAYS** call `constrainSelectedInstRegOperands` after manual instruction selection in GlobalISel.
