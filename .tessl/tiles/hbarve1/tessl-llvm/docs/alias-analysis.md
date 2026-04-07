# Alias Analysis (LLVM 22)

Reference: [AliasAnalysis.h](https://github.com/llvm/llvm-project/blob/llvmorg-22.1.2/llvm/include/llvm/Analysis/AliasAnalysis.h) | [MemoryLocation](https://llvm.org/doxygen/classllvm_1_1MemoryLocation.html)

**Alias analysis** answers whether two memory references **may**, **must**, or **cannot** refer to the **same** underlying object. Almost every memory optimization (load/store motion, dead store elimination, vectorization) depends on it.

---

## Core concepts

### `AliasResult`

Typical outcomes when comparing two locations:

| Result | Meaning |
|--------|---------|
| `NoAlias` | Never overlap for the accessed sizes |
| `MustAlias` | Always the same object / same base for the accessed range |
| `MayAlias` | Conservative unknown |
| `PartialAlias` | Overlap but not identical footprint |

### `MemoryLocation`

A memory reference is described by:

- A **pointer `Value*`** (or pseudo-location for unknown memory)
- A **size** in bytes (`LocationSize`)
- Optional **AAMDNodes** (TBAA and related metadata)

Construct helpers include `MemoryLocation::get` / `getBeforeOrAfter` for `LoadInst`, `StoreInst`, etc.

### Mod/ref on calls

**ModRef** information classifies whether a call **reads**, **writes**, or **neither** for a given `MemoryLocation`:

- `ModRefInfo::NoModRef`, `Ref`, `Mod`, `ModRef`

Passes use this to hoist loads past calls or to eliminate stores.

---

## How passes query alias information (NPM)

In the **New Pass Manager**, alias results are obtained through the **analysis manager** stack, not `getAnalysis<AliasAnalysis>()`. The usual pattern from a **function** pass is to obtain the **`AAResults`** facade for that function’s context (the exact helper may be `getAA(FAM)`-style depending on your pass boilerplate—**verify the signature in LLVM 22’s `AliasAnalysis.h` and `PassBuilder.cpp`** when you wire code).

Conceptually:

```cpp
// Illustrative — confirm API names against LLVM 22 headers:
// AAResults AA = ... obtained from FunctionAnalysisManager for Function F;

MemoryLocation LocA = MemoryLocation::get(LI);
MemoryLocation LocB = MemoryLocation::get(SI);
AliasResult AR = AA.alias(LocA, LocB);
if (AR == AliasResult::NoAlias) {
  // Safe to reorder / CSE under the right additional checks
}
```

Always re-query after IR changes that may affect memory behavior; **invalidate** analyses via `PreservedAnalyses` and `FAM.invalidate` when you break assumptions.

---

## IR-level hints (prefer these first)

Before writing a custom AA pass, use what LLVM already understands:

| Mechanism | Effect |
|-----------|--------|
| `noalias` on parameters / returns | Asserts non-aliasing with other pointers per LangRef rules |
| `tbaa` metadata | Type-based alias analysis |
| `scoped noalias` domains and scopes | Control-flow-scoped non-aliasing |
| Function attributes | `argmemonly`, `readnone`, `readonly`, etc. (see [attributes-metadata.md](attributes-metadata.md)) |

A precise frontend that emits **`noalias`** and **TBAA** often yields large wins without any C++ alias analysis code.

---

## Custom alias analysis (in-tree)

Adding a **new** alias analysis implementation is **advanced** and tightly coupled to LLVM’s internal AA pipeline. The high-level steps:

1. Study **`BasicAA`** and any **scoped** AA implementations in `llvm/lib/Analysis/` as templates.
2. Implement an **analysis** using the NPM pattern (`AnalysisInfoMixin`, `run()` returning a result type that participates in the AA stack—**match the LLVM 22 pattern used by sibling analyses**).
3. Register the analysis in the **PassBuilder** / `PassRegistry.def` the same way other **function analyses** are registered.
4. Ensure it composes correctly with **`AAResults`** (ordering and delegation are easy to get wrong).

**Out-of-tree:** shipping a third-party AA that plugs into `opt` like in-tree passes is uncommon; most teams push facts into IR (**metadata/attributes**) or run a **custom pass** that uses **`AAResults`** only as a **client**, not as a new provider.

---

## Language-specific example (metadata, not a new AA)

If your language guarantees that **immutable global tables** never alias mutable heap objects, you may:

1. Mark loads from those tables with **distinct TBAA** nodes, or
2. Pass pointers with **`noalias`** at call boundaries where the language rules allow it.

That reuses existing **BasicAA + TBAA** instead of forking AA.

---

## Common mistakes

- **Do NOT** cache `AAResults` across transformations without invalidation.
- **Do NOT** confuse **pointer equality** with **alias results**—two different `Value*` can `MustAlias` (e.g., different `GEP`s with same object).
- **Do NOT** ignore **partial alias** when doing range-sensitive reasoning.
- **ALWAYS** verify **ModRef** before moving instructions past **calls**.

---

## See also

- [Attributes & metadata](attributes-metadata.md) — TBAA, `noalias`, function attributes
- [New Pass Manager](new-pass-manager.md) — `FunctionAnalysisManager`, analysis registration
