# TableGen — Syntax, Backends, and Usage (LLVM 22)

Reference: [TableGen Overview](https://llvm.org/docs/TableGen/index.html) | [TableGen Language Reference](https://llvm.org/docs/TableGen/ProgRef.html)

---

## What is TableGen?

TableGen is LLVM's domain-specific language for describing structured, repetitive data — primarily:
- **Target registers** (`RegisterInfo.td`)
- **Instructions** (`InstrInfo.td`)
- **Intrinsics** (`Intrinsics*.td`)
- **Calling conventions** (`CallingConv.td`)
- **Scheduling models** (`SchedModel.td`)
- **Clang attributes** (`Attr.td`)

TableGen files (`.td`) are compiled by `llvm-tblgen` into C++ `.inc` files that are `#include`d into the build. You never hand-edit the generated `.inc` files.

---

## Language basics

### Records and classes

```tablegen
// A class — like a template / abstract base
class Reg<string name, bits<5> enc> {
  string Name = name;
  bits<5> Encoding = enc;
  string Namespace = "MyISA";
}

// A def — a concrete record (instantiation)
def R0 : Reg<"r0", 0>;
def R1 : Reg<"r1", 1>;
def SP : Reg<"sp", 29>;
def LR : Reg<"lr", 30>;
def PC : Reg<"pc", 31>;
```

### Inheritance and mixins

```tablegen
class GPReg<string name, bits<5> enc> : Reg<name, enc> {
  let isCallerSaved = true;
}

def R2 : GPReg<"r2", 2>;
def R3 : GPReg<"r3", 3>;
```

### `let` — override a field

```tablegen
def MyInst : Instruction {
  let Namespace = "MyISA";
  let OutOperandList = (outs GPR:$dst);
  let InOperandList  = (ins  GPR:$src1, GPR:$src2);
  let AsmString = "add $dst, $src1, $src2";
  let Pattern   = [(set GPR:$dst, (add GPR:$src1, GPR:$src2))];
}
```

### `multiclass` — generate multiple records at once

```tablegen
multiclass BinOp<string name, SDNode Op> {
  def rr : MyISAInst<(outs GPR:$dst), (ins GPR:$src1, GPR:$src2),
                     !strconcat(name, " $dst, $src1, $src2"),
                     [(set GPR:$dst, (Op GPR:$src1, GPR:$src2))]>;
  def ri : MyISAInst<(outs GPR:$dst), (ins GPR:$src1, i32imm:$imm),
                     !strconcat(name, " $dst, $src1, $imm"),
                     [(set GPR:$dst, (Op GPR:$src1, imm:$imm))]>;
}

defm ADD : BinOp<"add", add>;
defm SUB : BinOp<"sub", sub>;
// Generates: ADDrr, ADDri, SUBrr, SUBri
```

### `foreach` — iterate over a list

```tablegen
foreach i = [0, 1, 2, 3, 4, 5, 6, 7] in {
  def R#i : Reg<"r"#i, i>;
}
```

### Useful built-in operators

| Operator | Description |
|----------|-------------|
| `!strconcat(a, b)` | Concatenate strings |
| `!cast<T>(x)` | Cast value `x` to type `T` |
| `!listsplat(val, n)` | List of `n` copies of `val` |
| `!foreach(x, list, expr)` | Map expression over list |
| `!foldl(init, list, acc, x, body)` | Fold over list |
| `!if(cond, a, b)` | Conditional value |
| `!not(b)` | Logical not |
| `!and(a, b)` / `!or(a, b)` | Logical operators |
| `!add(a, b)` / `!shl(a, n)` | Arithmetic on integers |
| `!size(list)` | Length of a list |
| `!empty(list)` | Is list empty? |

---

## Register definitions

```tablegen
// include/llvm/Target/Target.td is always included first
include "llvm/Target/Target.td"

class MyReg<bits<5> Enc, string AsmName, list<string> AltNames = []>
    : Register<AsmName, AltNames> {
  let HWEncoding{4-0} = Enc;
  let Namespace = "MyISA";
}

// Define registers
foreach i = 0-31 in
  def R#i : MyReg<i, "r"#i>;

// Aliases
def ZERO : MyReg<0,  "zero">;  // R0 alias

// Register classes — group registers by type and alignment
def GPR : RegisterClass<"MyISA", [i32], 32, (add
    R0, R1, R2, R3, R4, R5, R6, R7,
    R8, R9, R10, R11, R12, R13, R14, R15
)>;

def FPR : RegisterClass<"MyISA", [f32], 32, (add
    // float registers ...
)>;
```

---

## Instruction definitions

```tablegen
// Base instruction class
class MyISAInst<dag outs, dag ins, string asmstr,
                list<dag> pattern, InstrItinClass itin = NoItinerary>
    : Instruction {
  let Namespace     = "MyISA";
  let OutOperandList = outs;
  let InOperandList  = ins;
  let AsmString      = asmstr;
  let Pattern        = pattern;
  let Itinerary      = itin;

  // Encoding fields (customize to your ISA)
  bits<32> Inst;
  bits<5>  Rd;
  bits<5>  Rs1;
  bits<5>  Rs2;
}

// ADD instruction
def ADD : MyISAInst<
  (outs GPR:$rd),
  (ins  GPR:$rs1, GPR:$rs2),
  "add $rd, $rs1, $rs2",
  [(set i32:$rd, (add i32:$rs1, i32:$rs2))]
> {
  let Inst{31-26} = 0b000001; // opcode
  let Inst{25-21} = Rd;
  let Inst{20-16} = Rs1;
  let Inst{15-11} = Rs2;
}

// Immediate operand type
def i32imm : Operand<i32>;

def ADDI : MyISAInst<
  (outs GPR:$rd),
  (ins  GPR:$rs1, i32imm:$imm),
  "addi $rd, $rs1, $imm",
  [(set i32:$rd, (add i32:$rs1, imm:$imm))]
>;
```

---

## SelectionDAG patterns

Patterns (`list<dag>`) describe how to match LLVM IR SDNodes and emit instructions.

```tablegen
// Pattern: (set $dst, (SDNode $src1, $src2))
//           ^^^  ^^^   ^^^^^^ output   inputs
//           out  reg   DAG node

[(set GPR:$dst, (add GPR:$src1, GPR:$src2))]  // integer add
[(set FPR:$dst, (fadd FPR:$src1, FPR:$src2))] // float add
[(store GPR:$src, ADDRri:$addr)]               // store to address
[(set GPR:$dst, (load ADDRri:$addr))]          // load from address

// Immediate patterns
[(set GPR:$dst, (add GPR:$src1, imm:$imm))]
[(set GPR:$dst, (add GPR:$src1, (i32 simm16:$imm)))]  // with predicate

// No pattern (handled manually in ISelLowering)
[]
```

---

## Intrinsic definitions

```tablegen
// In include/llvm/IR/Intrinsics.td or a target-specific file

def int_my_sqrt : DefaultAttrsIntrinsic<
  [llvm_float_ty],              // return type
  [llvm_float_ty],              // arg types
  [IntrNoMem, IntrWillReturn, IntrSpeculatable]
>;
// Becomes: declare float @llvm.my.sqrt(float)

// Overloaded intrinsic
def int_my_abs : DefaultAttrsIntrinsic<
  [llvm_anyint_ty],
  [LLVMMatchType<0>],           // arg type matches return type
  [IntrNoMem, IntrWillReturn]
>;
// Becomes: @llvm.my.abs.i32, @llvm.my.abs.i64, etc.

// Target-specific (in IntrinsicsMyISA.td)
def int_myisa_vcmp : DefaultAttrsIntrinsic<
  [llvm_v4i32_ty],
  [llvm_v4i32_ty, llvm_v4i32_ty, ImmArg<ArgIndex<2>>],
  [IntrNoMem]
>;
```

---

## Calling conventions

```tablegen
// In CallingConv.td
def CC_MyISA : CallingConv<[
  // Return i32 in R0
  CCIfType<[i32], CCAssignToReg<[R0, R1]>>,
  // Return f32 in F0
  CCIfType<[f32], CCAssignToReg<[F0]>>,
  // Remaining args on stack, 4-byte aligned
  CCAssignToStack<4, 4>
]>;

def RetCC_MyISA : CallingConv<[
  CCIfType<[i32], CCAssignToReg<[R0]>>,
  CCIfType<[f32], CCAssignToReg<[F0]>>
]>;

def CSR_MyISA : CalleeSavedRegs<(add R19, R20, R21, R22, SP, LR)>;
```

---

## Running TableGen (CMake)

```cmake
# In lib/Target/MyISA/CMakeLists.txt
set(LLVM_TARGET_DEFINITIONS MyISA.td)

tablegen(LLVM MyISAGenRegisterInfo.inc    -gen-register-info)
tablegen(LLVM MyISAGenInstrInfo.inc       -gen-instr-info)
tablegen(LLVM MyISAGenSubtargetInfo.inc   -gen-subtarget)
tablegen(LLVM MyISAGenAsmWriter.inc       -gen-asm-writer)
tablegen(LLVM MyISAGenAsmMatcher.inc      -gen-asm-matcher)
tablegen(LLVM MyISAGenDAGISel.inc         -gen-dag-isel)
tablegen(LLVM MyISAGenCallingConv.inc     -gen-callingconv)
tablegen(LLVM MyISAGenMCCodeEmitter.inc   -gen-emitter)

add_public_tablegen_target(MyISACommonTableGen)
```

**Using a generated `.inc` file in C++:**
```cpp
// Control which section of the .inc to emit with a #define before the include
#define GET_REGINFO_ENUM
#include "MyISAGenRegisterInfo.inc"
#undef GET_REGINFO_ENUM

#define GET_REGINFO_MC_DESC
#include "MyISAGenRegisterInfo.inc"
#undef GET_REGINFO_MC_DESC

#define GET_REGINFO_TARGET_DESC
#include "MyISAGenRegisterInfo.inc"
#undef GET_REGINFO_TARGET_DESC
```

---

## Running TableGen manually (debugging)

```bash
# Dump all records (useful for debugging)
llvm-tblgen --dump-json include/llvm/IR/Intrinsics.td \
  -I include/

# Generate register info for a target
llvm-tblgen -gen-register-info \
  lib/Target/MyISA/MyISA.td \
  -I include/ -I lib/Target/MyISA/

# Check TableGen syntax without generating output
llvm-tblgen --no-warn-on-unused-template-args \
  lib/Target/MyISA/MyISA.td -I include/
```

---

## Common mistakes

- **Do NOT** hand-edit `*Gen*.inc` files — they are overwritten on every rebuild.
- **Do NOT** forget `add_public_tablegen_target(MyISACommonTableGen)` and depend on it — C++ targets that include `.inc` files must wait for TableGen to run first.
- **Do NOT** use `bits<N>` fields wider than needed — TableGen encoding fields must fit within your instruction word width.
- **ALWAYS** include `llvm/Target/Target.td` at the top of your target's `.td` — it defines `Instruction`, `Register`, `RegisterClass`, `CallingConv`, etc.
- **ALWAYS** set `let Namespace = "MyISA"` on registers and instructions — without it, the generated enum names will clash with other targets.
