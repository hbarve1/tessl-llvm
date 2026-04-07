# Calling Conventions (LLVM 22)

Reference: [LangRef Calling Conventions](https://llvm.org/docs/LangReference.html#callingconv) | [CallingConv.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.2/llvm/include/llvm/IR/CallingConv.h)

Calling conventions control **how arguments and return values are passed in the ABI**: registers vs stack, who cleans the stack, varargs handling, and special rules for sret, byval, and inalloca.

---

## IR and C++ API

### `CallingConv::ID`

Every `llvm::Function` and each `CallInst` / `InvokeInst` carries a calling convention ID (`unsigned` / `CallingConv::ID`).

```cpp
#include "llvm/IR/CallingConv.h"

F->setCallingConv(CallingConv::C);
CI->setCallingConv(CallingConv::Fast);

unsigned CC = F->getCallingConv();
```

Common values (see `llvm/IR/CallingConv.h` for the full enum):

| Name | Typical use |
|------|-------------|
| `C` | Default C ABI on the target |
| `Fast` | Fast calling convention (often more registers) |
| `Cold` | Cold code path |
| `AnyReg` / `Swift` / etc. | Language- or target-specific ABIs |
| `WebKit_JS`, `AnyReg` | Specialized runtimes |

**Verifier rule:** the convention on a `call` must be compatible with the callee’s function type and target capabilities.

---

## Where conventions are defined

1. **Shared enum** — `llvm/include/llvm/IR/CallingConv.h`  
   Global IDs used across targets and in LangRef.

2. **TableGen** — target-specific files such as  
   `llvm/lib/Target/X86/X86CallingConv.td` (other targets mirror this pattern).  
   These records describe **which registers and stack slots** are used for each convention the target supports.

3. **Lowering** — `TargetLowering::LowerFormalArguments`, `LowerCall`, `LowerReturn`, and helpers that consume **CCState** / **CCValAssign** (see `llvm/lib/CodeGen/CallingConvLower.*`).

---

## `CCState` and `CCValAssign` (codegen)

When the backend lowers calls, it walks formal parameters and call operands and **assigns** each to a register class or stack slot.

- **`CCValAssign`** — one logical placement: register, stack offset, flags (e.g. sext, zext, inreg).
- **`CCState`** — mutable state while assigning (stack offset, used registers).

Custom conventions usually provide **custom assignment logic** (a **CCAssignFn**) invoked from the target’s `CallingConv.td` or from hand-written lowering, so that arguments are laid out consistently for prologue/epilogue and for callers.

---

## Adding a **target-specific** convention (in-tree)

High-level checklist:

1. **Reserve an ID** — Add an entry to `CallingConv` in `include/llvm/IR/CallingConv.h` and document it in LangRef if it is user-visible. Follow the numbering rules already used in that file (avoid collisions with reserved ranges).

2. **TableGen** — In the target’s `*CallingConv.td`, define a `CallingConv` record (or extend an existing multiclass) so TableGen emits the correct **CCAssignFn** hooks for incoming args, outgoing calls, and returns.

3. **ISel / lowering** — In `*ISelLowering.cpp`, ensure `LowerFormalArguments`, `LowerCall`, and `CanLowerReturn` handle the new CC (often by dispatching on `CLI.CallConv` / `F.getCallingConv()`).

4. **Clang or front-end** — If the language needs a keyword or attribute, wire it in the frontend so it emits `setCallingConv` on the IR function and calls.

5. **Tests** — Assembly or MIR tests that show register/stack layout for a small function using the new CC.

6. **Run TableGen** — Rebuild the target’s TableGen outputs (`*Gen*.inc`) via CMake; never edit generated files by hand.

For file names and class names, mirror the closest existing convention in your target (for example a variant of `Fast` or `C`).

---

## Out-of-tree and frontends

- **If you only need a standard ABI**, set `CallingConv::C` (or `Fast`, etc.) and rely on the installed LLVM target. No TableGen changes.

- **If you need a custom layout**, you normally **fork or patch the in-tree target** that you ship with your toolchain, because assignment rules live in target TableGen and lowering. Pure out-of-tree projects cannot inject a new global `CallingConv::ID` into an unmodified LLVM without rebuilding LLVM.

- **Interop trick:** some embedders use **existing** conventions plus **parameter attributes** (`byval`, `sret`, `inalloca`, `nest`, alignment) to model special ABIs without a new CC enum value—at the cost of staying within what the target already supports.

---

## Related IR features

| Feature | Role |
|---------|------|
| `sret` | Hidden pointer return for large structs |
| `byval` | Callee gets a copy; caller owns memory |
| `inalloca` | Caller-allocated memory for args (MSVC-style) |
| `nest` | Static chain / display pointer |
| `swiftself` / `swifterror` | Swift ABI hooks on some targets |

Combine these with the correct `CallingConv` for your language.

---

## Common mistakes

- **Do NOT** set a calling convention on the call that disagrees with the callee function without knowing the target supports that combination.
- **Do NOT** forget **varargs** (`...`) rules: many conventions use different paths for varargs vs fixed args.
- **Do NOT** hand-edit `*Gen*.inc` from CallingConv TableGen—regenerate via the build.
- **ALWAYS** verify with **llc** / **Clang** output and a small ABI test before relying on a new CC in production.

---

## See also

- [Code generation](codegen.md) — `TargetLowering`, ISel, `MachineFunction`
- [TableGen](tablegen.md) — multiclass patterns for targets
- [Attributes & metadata](attributes-metadata.md) — parameter attributes that interact with the ABI
