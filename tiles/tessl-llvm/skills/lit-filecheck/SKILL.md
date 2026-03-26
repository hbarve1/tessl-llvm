---
name: lit-filecheck
description: Write lit tests with FileCheck for LLVM 20 IR transforms, pass output, and codegen. Covers RUN lines, CHECK patterns, common directives, negative checks, and running tests with llvm-lit.
---

# Skill: Write Lit / FileCheck Tests (LLVM 20)

Use this skill when the user wants to add tests for an LLVM pass, IR transform, codegen output, or compiler frontend.

---

## Step 0 — Tools required

```bash
# Verify tools are on PATH
llvm-lit --version       # the lit test runner
FileCheck --version      # the pattern matcher
opt --version            # for IR transform tests
llc --version            # for codegen tests
```

For out-of-tree projects, set `LLVM_TOOLS_BINARY_DIR` in cmake and use:
```bash
${LLVM_TOOLS_BINARY_DIR}/llvm-lit
${LLVM_TOOLS_BINARY_DIR}/FileCheck
```

---

## Step 1 — Test file structure

Lit tests are plain text files (`.ll` for IR, `.c`/`.cpp` for Clang tests):

```llvm
; RUN: opt -passes=<pass-name> -S %s | FileCheck %s
;
; A short comment explaining what this test covers.

define i32 @test_basic(i32 %x, i32 %y) {
entry:
; CHECK-LABEL: @test_basic
; CHECK:       %sum = add i32 %x, %y
  %sum = add i32 %x, %y
  ret i32 %sum
; CHECK:       ret i32 %sum
}
```

- `%s` — expands to the test file path
- `-S` — print IR as text (not bitcode)
- `FileCheck %s` — reads CHECK patterns from the same file

---

## Step 2 — RUN line patterns

```
; RUN: <command> | FileCheck %s
; RUN: <command> | FileCheck %s --check-prefix=OPT
; RUN: <command> 2>&1 | FileCheck %s              ; capture stderr too
; RUN: not <command> 2>&1 | FileCheck %s          ; expect non-zero exit
```

Multiple RUN lines run sequentially; each must pass:

```llvm
; RUN: opt -passes=instcombine -S %s | FileCheck %s --check-prefix=IC
; RUN: opt -passes=simplifycfg -S %s | FileCheck %s --check-prefix=SCFG
```

---

## Step 3 — FileCheck directives

### Basic matching

```
; CHECK:       <pattern>      — match pattern anywhere on the next matching line
; CHECK-NEXT:  <pattern>      — match on the very next line after previous CHECK
; CHECK-SAME:  <pattern>      — match on the same line as previous CHECK
; CHECK-NOT:   <pattern>      — assert pattern does NOT appear until next CHECK
; CHECK-EMPTY:                — assert next line is empty
; CHECK-LABEL: <pattern>      — like CHECK but resets the search position (use for function names)
```

### Captures and variables

```
; CHECK: %[[REG:[a-z0-9.]+]] = add i32
; CHECK: ret i32 %[[REG]]
```

`%[[REG:[a-z0-9.]+]]` captures the register name into `REG` for later use.

Numeric variables:
```
; CHECK: define i32 @foo(i32 %x)
; CHECK: [[#LINE:]]:
; CHECK: line [[#LINE+1]]
```

### Negative checks

```
; CHECK-NOT: call void @expensive_function
; CHECK-NOT: unreachable
```

`CHECK-NOT` asserts the pattern never appears between the previous and next `CHECK`.

---

## Step 4 — IR transform test (opt)

```llvm
; RUN: opt -passes=instcombine -S %s | FileCheck %s
; RUN: opt -passes=instcombine -disable-output %s  ; smoke test: must not crash

; Test: x + 0 → x
define i32 @add_zero(i32 %x) {
; CHECK-LABEL: @add_zero(
; CHECK-NEXT:    ret i32 %x
  %r = add i32 %x, 0
  ret i32 %r
}

; Test: x * 1 → x
define i32 @mul_one(i32 %x) {
; CHECK-LABEL: @mul_one(
; CHECK-NEXT:    ret i32 %x
  %r = mul i32 %x, 1
  ret i32 %r
}

; Negative test: should NOT eliminate a volatile load
define i32 @volatile_load(ptr %p) {
; CHECK-LABEL: @volatile_load(
; CHECK:         load volatile i32
; CHECK-NOT:     ret i32 0
  %v = load volatile i32, ptr %p
  ret i32 %v
}
```

---

## Step 5 — Pass output test (custom pass)

```llvm
; RUN: opt -load-pass-plugin=%llvmshlibdir/MyPass%pluginext \
; RUN:     -passes=my-pass -S %s 2>&1 | FileCheck %s

; CHECK: MyPass: processing function foo

define void @foo() {
  ret void
}
```

For in-tree passes, just use `-passes=my-pass` without `--load-pass-plugin`.

---

## Step 6 — Codegen test (llc)

```llvm
; RUN: llc -mtriple=x86_64-unknown-linux-gnu -O2 < %s | FileCheck %s

define i32 @add(i32 %a, i32 %b) {
entry:
; CHECK-LABEL: add:
; CHECK:       leal (%rdi,%rsi), %eax
; CHECK:       retq
  %r = add i32 %a, %b
  ret i32 %r
}
```

Test for a specific CPU feature:
```llvm
; RUN: llc -mtriple=x86_64 -mattr=+avx2 < %s | FileCheck %s --check-prefix=AVX2
; AVX2: vmovdqu
```

---

## Step 7 — Frontend / compiler test

```llvm
; For a custom compiler binary:
; RUN: my-compiler %s -emit-llvm -S -o - | FileCheck %s
; RUN: my-compiler %s -o %t && %t | FileCheck %s --check-prefix=OUTPUT

; OUTPUT: Hello, World!
```

---

## Step 8 — lit.cfg.py (test suite configuration)

In the root of your test directory, create `lit.cfg.py`:

```python
import lit.formats
config.name = "MyProject"
config.test_format = lit.formats.ShTest(True)
config.test_source_root = os.path.dirname(__file__)
config.suffixes = [".ll", ".c"]

# Substitutions for tool paths
config.substitutions.append(("%opt",      "/path/to/opt"))
config.substitutions.append(("%filecheck","/path/to/FileCheck"))
config.substitutions.append(("%llvmshlibdir", "/path/to/llvm/lib"))
config.substitutions.append(("%pluginext",
    ".dylib" if sys.platform == "darwin" else ".so"))
```

---

## Step 9 — Running tests

```bash
# Run all tests in a directory
llvm-lit test/ -v

# Run a single test file
llvm-lit test/Transforms/my-pass.ll -v

# Run with parallelism
llvm-lit test/ -j8

# Filter by name
llvm-lit test/ --filter "instcombine"

# Show output of failing tests
llvm-lit test/ --show-all
```

---

## Step 10 — Debugging a failing test

```bash
# Print the exact FileCheck invocation
llvm-lit test/my-test.ll -v --show-all

# Run the RUN line manually (replace %s with the file path)
opt -passes=my-pass -S test/my-test.ll | FileCheck test/my-test.ll

# Check what the pass actually emits:
opt -passes=my-pass -S test/my-test.ll

# Use FileCheck --dump-input to show which lines matched/failed:
opt -passes=my-pass -S test/my-test.ll | \
  FileCheck test/my-test.ll --dump-input=fail
```

---

## Common patterns

```llvm
; Match any register name
; CHECK: %{{[a-z0-9.]+}} = add

; Match optional whitespace
; CHECK: ret{{[ \t]+}}i32

; Match a function definition regardless of attributes
; CHECK-LABEL: define {{.*}}@my_func(

; Assert a block is completely eliminated (no label in output)
; CHECK-NOT: dead_block:

; Test that a call is inlined (no call instruction remains)
; CHECK-NOT: call {{.*}}@small_func
```

---

## Common mistakes

- **Do NOT** use `CHECK:` without `CHECK-LABEL:` for function boundaries — without labels, patterns from one function can match in another.
- **Do NOT** match on internal register names like `%0`, `%1` — they change easily; use named values or `%[[VAR:...]]` captures instead.
- **Do NOT** write overly specific assembly patterns in codegen tests — they break on minor compiler changes; match essential parts only.
- **ALWAYS** add a smoke-test `RUN` line with `-disable-output` to catch crashes separately from correctness.
- **ALWAYS** use `CHECK-LABEL:` at each function so FileCheck resets its search position.
