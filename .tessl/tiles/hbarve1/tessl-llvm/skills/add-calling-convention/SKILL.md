---
name: add-calling-convention
description: Add or wire a calling convention in an LLVM 22 in-tree target or document IR-level ABI choices for out-of-tree frontends. Covers CallingConv IDs, TableGen CallingConv.td, CCState/CCValAssign hooks, ISel lowering, and tests.
---

# Skill: Add a Custom Calling Convention (LLVM 22)

Use this skill when the user needs a **new or customized ABI** for a **target inside the LLVM tree**, or when an **out-of-tree frontend** must choose among existing conventions and attributes.

**Scope boundary:** Truly new register/stack rules require **target TableGen + lowering changes inside LLVM**. Out-of-tree projects that only **consume** an installed LLVM should not expect to add a global `CallingConv::ID` without rebuilding LLVM.

---

## Step 0 — Classify the request

| Goal | Where to work |
|------|----------------|
| Use standard C/fast call on existing target | IR only: `setCallingConv`, parameter attributes |
| Language needs special struct return / copies | `sret`, `byval`, `inalloca`, alignment attrs |
| **New** register/stack layout for a target | In-tree: `CallingConv.h`, `*CallingConv.td`, `*ISelLowering.cpp` |
| Document/embedder custom protocol | Often **metadata + existing CC**, or a dedicated in-tree target fork |

Read [calling-conventions.md](../../docs/calling-conventions.md) for terminology (`CCState`, `CCValAssign`, related IR features).

---

## Step 1 — IR-level wiring (all projects)

1. Pick an existing `llvm::CallingConv::ID` that matches the platform (usually `C` or `Fast`).
2. On each `llvm::Function` and on `CallInst`/`InvokeInst`, call `setCallingConv`.
3. Add parameter/return attributes (`sret`, `byval`, `nest`, `swiftself`, etc.) per LangRef.
4. Run `verifyModule` and compile with `llc` (or your driver) for the intended triple.

```cpp
F->setCallingConv(CallingConv::C);
F->addParamAttrs(0, ParamAttrBuilder...); // e.g. ByVal, StructRet
```

If this satisfies the ABI, **stop**—no TableGen work needed.

---

## Step 2 — Reserve a CallingConv ID (in-tree only)

1. Open `llvm/include/llvm/IR/CallingConv.h`.
2. Add a new enumerator in the correct **numeric range** for your class of conventions (follow comments in that file; avoid reserved values).
3. If the convention is user-visible in IR text, update **LangRef** (`LangRef.rst`) with the spelling and number.
4. Rebuild; fix any `switch (CC)` exhaustiveness warnings in tree.

---

## Step 3 — Target TableGen (`*CallingConv.td`)

1. Locate the target’s calling-convention definitions, e.g. `llvm/lib/Target/X86/X86CallingConv.td` (pattern: `Target/*/*CallingConv.td`).
2. Add a `def` / `multiclass` instance for your convention that hooks **CCAssignFn**-style logic the same way sibling conventions do.
3. Regenerate TableGen outputs via CMake (`cmake --build --target <Target>CommonTableGen` or full target build)—**never** hand-edit `*Gen*.inc`.

If you are unsure which multiclass to extend, **clone the closest existing convention** (e.g. a variant of `Fast`) and adjust register/stack rules incrementally.

---

## Step 4 — ISel lowering (C++)

In the target’s `*ISelLowering.cpp` (and related files):

1. **`LowerFormalArguments`** — handle the new `CallingConv` when assigning incoming values to ` SDValue`s / registers.
2. **`LowerCall` / `LowerCallTo`** — match outgoing call setup (caller-side).
3. **`CanLowerReturn` / `LowerReturn`** — return-value rules (registers, sret, multiple values).

Use `CCState` / `CCValAssign` consistently with the TableGen side. Mismatches here cause **silent ABI bugs**.

Search the target for `getCallingConv()` / `callconv` dispatch to find existing patterns.

---

## Step 5 — Frontend (optional)

If Clang (or another frontend) must expose the convention:

- Add an attribute or calling-convention keyword in the frontend.
- Map it to `setCallingConv` on the IR function and calls.
- Add semantic checks (e.g. incompatible with varargs).

---

## Step 6 — Tests

1. **IR test** — `.ll` showing `call`/`define` with the new `cc` number or name (as printed by LLVM).
2. **Assembly or MIR test** — show expected registers/stack for a **minimal** function (one or two args + return).
3. **Negative test** — invalid combination of CC + attributes if the verifier should reject it.

Run `llvm-lit` on the new files.

---

## Step 7 — Review checklist

```
[ ] CallingConv enum value unique and documented
[ ] TableGen + generated inc files regenerated
[ ] LowerFormalArguments / LowerCall / LowerReturn updated
[ ] Varargs path considered (if applicable)
[ ] Clang/frontend mapping (if user-facing)
[ ] lit tests for IR + machine level
[ ] Compared against platform ABI doc or psABI
```

---

## Out-of-tree summary

You **cannot** complete Steps 2–4 from a standalone plugin; they live in the LLVM target sources. Out-of-tree frontends should:

- Use **existing** `CallingConv` values and **attributes**, and/or
- Vendor a **patched LLVM** with the new convention if the ABI is non-standard.

---

## Common mistakes

- **Do NOT** change only the IR `cc` without matching backend assignment logic.
- **Do NOT** reuse a reserved calling-convention number.
- **Do NOT** edit generated TableGen inc files manually.
- **ALWAYS** test **varargs** and **sret/byval** interactions for your CC.

---

## See also

- [calling-conventions.md](../../docs/calling-conventions.md)
- [codegen.md](../../docs/codegen.md)
- [tablegen.md](../../docs/tablegen.md)
- [add-intrinsic](../add-intrinsic/SKILL.md) — if the ABI also needs a new intrinsic
