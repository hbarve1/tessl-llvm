# Link-Time Optimization (LTO) and ThinLTO (LLVM 22)

Reference: [ThinLTO](https://clang.llvm.org/docs/ThinLTO.html) | [LLVM Bitcode](https://llvm.org/docs/BitCodeFormat.html)

**LTO** defers optimization until **link time**, when the linker sees more than one translation unit. LLVM represents intermediate code as **bitcode** (binary LLVM IR). **ThinLTO** is a scalable variant that uses **per-module summaries** and a **thin link** step instead of loading all IR into one huge module first.

---

## Full (monolithic) LTO

1. Each compile unit is emitted as **LLVM bitcode** (`.bc`) or a **mixed object** containing bitcode (depending on platform and driver).
2. The **linker** invokes an **LTO plugin** (historically `LLVMgold.so` with GNU `ld`; modern **LLD** has **integrated LTO**).
3. The plugin loads **all** bitcode, merges it into a single **combined module**, runs a **link-time optimization pipeline**, then emits native object code for the final link.

**Characteristics:**

- Maximum cross-module visibility for the optimizer (inlining, devirtualization, whole-program alias analysis).
- High peak memory and link time for large programs.

**Typical Clang flags:**

```bash
clang -flto -O2 -c a.c -o a.o
clang -flto -O2 -c b.c -o b.o
clang -flto -O2 a.o b.o -o prog
```

Use the **same** `-flto` and optimization level across compile and link steps.

---

## ThinLTO

1. Each `.o` (or bitcode file) carries a **compact summary** of its IR: functions, globals, call graph edges, etc.
2. **Thin link** reads only summaries, builds a **global index** (who can inline whom, import lists), and plans work.
3. **Backend** phases compile each module **mostly in parallel**, **importing** only the pieces of other modules that the index says are needed.

**Characteristics:**

- Much lower peak memory than monolithic LTO.
- Parallelism-friendly; used heavily in Chromium-scale builds.
- Very close to full LTO quality for many workloads when tuned correctly.

**Typical Clang flags:**

```bash
clang -flto=thin -O2 -c a.c -o a.o
clang -flto=thin -O2 -c b.c -o b.o
clang -flto=thin -O2 a.o b.o -o prog
```

**Useful Clang/LLVM tools:**

| Tool | Role |
|------|------|
| `llvm-dis` | Disassemble bitcode to `.ll` text |
| `llvm-as` | Assemble `.ll` to bitcode |
| `llvm-nm` / `llvm-readobj` | Inspect objects and LTO sections |
| `llvm-lto` / `llvm-lto2` | Stand-alone LTO drivers (advanced) |

Exact tool names and driver integration depend on your LLVM build and platform; verify with your installed `clang --help` and LLVM 22 release notes.

---

## Bitcode artifacts

- **Bitcode** is **not** a stable long-term interchange format across arbitrary LLVM versions—pin producer and consumer to the **same** LLVM major for LTO links.
- **Summary index** (ThinLTO) is an internal format; treat it as build plumbing, not a public API for language runtimes.

---

## LLD (integrated linker)

**LLD** (ELF, COFF, Mach-O flavors) can run LTO **without** a separate `LLVMgold`-style plugin when Clang is built with LLD support:

```bash
clang -flto=thin -fuse-ld=lld ...
```

Requirements:

- The **link step** must use an LTO-aware linker (typically LLD or a gold build with the LLVM plugin).
- **LTO codegen** must see the same **target triple** and ABI options as the compile steps.

---

## Out-of-tree projects

- Emit bitcode from your frontend or JIT with the **bitcode writer** (`BitWriter` component) if you participate in a Clang/LLD-based link—this is uncommon for small tools; most frontends still emit object files via the **TargetMachine** MC layer.
- If you only need **whole-program optimization within a single process**, consider **running an `ModulePassManager` on a merged `Module`** in memory instead of file-based LTO.
- For **distributed** ThinLTO builds, follow Clang’s ThinLTO docs for **distributed** flags and index-only links; paths and job APIs change between releases—verify against LLVM 22.

---

## Debugging LTO issues

- **Symbol visibility:** `hidden`, `default`, and COMDAT behavior affect what the linker can merge and devirtualize.
- **ODR violations** show up as mysterious link errors or miscompares—LTO assumes a single definition rule across the whole program.
- **Mixed LTO / non-LTO objects:** safe but may block some cross-module opts; prefer all-LTO for hot code.
- **Version skew:** mixing bitcode from different LLVM versions at link time is unsupported.

---

## Common mistakes

- **Do NOT** mix `-flto` compile with a non-LTO-aware linker without knowing the consequences.
- **Do NOT** assume bitcode files are portable across LLVM majors.
- **Do NOT** forget to pass **optimization level** and **target** flags consistently at link time.
- **ALWAYS** use **`llvm-dis`** on failing modules when diagnosing IR-level LTO bugs.

---

## See also

- [Out-of-tree projects](out-of-tree.md) — CMake components (`BitWriter`, targets)
- [New Pass Manager](new-pass-manager.md) — pipelines that run at compile time vs link time
