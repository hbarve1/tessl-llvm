---
name: lower-struct-types
description: Lower source-language struct, union, and tuple types to LLVM 22 IR. Covers creating StructType, computing field offsets, emitting GEP for field access, packed vs. padded layouts, unions via largest-member types, passing structs by value vs. pointer, and the alloca+mem2reg pattern for struct locals.
---

# Skill: Lower Struct / Union / Tuple Types to LLVM 22 IR

Use this skill when your language has composite types (records, structs, tuples, classes, tagged unions) and you need to map them to LLVM IR for storage, field access, and function passing.

---

## Step 0 — Key rules

- All pointers in LLVM 22 are opaque (`ptr`) — never use `%MyStruct*`. Always pair a GEP with the explicit struct type.
- LLVM `StructType` models the layout; `GEP` with `i32` constant indices accesses fields.
- Use the alloca + mem2reg pattern for struct locals — let `PromotePass` eliminate the alloca when possible.
- Track your source-level type → `StructType*` mapping in your CodeGen context.

---

## Step 1 — Create StructType

```cpp
#include "llvm/IR/DerivedTypes.h"

// Named (identified) struct — opaque until body is set; supports recursion
llvm::StructType *PointTy = llvm::StructType::create(Ctx, "Point");
PointTy->setBody({B.getInt32Ty(), B.getInt32Ty()}); // { i32 x, i32 y }

// Literal (anonymous) struct — body given at creation; no recursion
llvm::StructType *PairTy = llvm::StructType::get(Ctx,
    {B.getInt32Ty(), B.getDoubleTy()}); // { i32, double }

// Packed struct — no padding, fields may be unaligned
llvm::StructType *PackedTy = llvm::StructType::get(Ctx,
    {B.getInt8Ty(), B.getInt32Ty()}, /*isPacked=*/true);
```

---

## Step 2 — Query field offsets and sizes

Use `DataLayout` to get the byte offset of a field and the struct's allocation size:

```cpp
const llvm::DataLayout &DL = M->getDataLayout();

// Byte offset of field index 1 in PointTy
uint64_t Offset1 = DL.getStructLayout(PointTy)->getElementOffset(1);

// Total allocation size in bytes (including trailing padding)
uint64_t TotalSize = DL.getTypeAllocSize(PointTy);

// Size in bits (for debug info)
uint64_t SizeBits = DL.getTypeSizeInBits(PointTy);
```

---

## Step 3 — Alloca for struct locals

Always alloca in the **entry block**, then let mem2reg / SROA optimize it:

```cpp
// Helper: always insert allocas at the top of the entry block
llvm::AllocaInst *createEntryAlloca(llvm::Function *F,
                                     llvm::Type *Ty,
                                     const llvm::Twine &Name = "") {
  llvm::IRBuilder<> EntryB(&F->getEntryBlock(),
                            F->getEntryBlock().begin());
  return EntryB.CreateAlloca(Ty, nullptr, Name);
}

// Allocate a Point struct
llvm::AllocaInst *PtAlloca = createEntryAlloca(F, PointTy, "pt");
```

---

## Step 4 — Field access via GEP

GEP syntax: `getelementptr <StructType>, ptr <base>, i32 0, i32 <fieldIndex>`

```cpp
// Store into field 0 (x) of a Point alloca
llvm::Value *XPtr = B.CreateStructGEP(PointTy, PtAlloca, 0, "pt.x");
B.CreateStore(B.getInt32(42), XPtr);

// Load from field 1 (y)
llvm::Value *YPtr = B.CreateStructGEP(PointTy, PtAlloca, 1, "pt.y");
llvm::Value *Y    = B.CreateLoad(B.getInt32Ty(), YPtr, "y");
```

`CreateStructGEP(Ty, Ptr, Idx)` emits `GEP Ty, Ptr, i32 0, i32 Idx` — the canonical form for struct field access.

---

## Step 5 — Nested structs

```cpp
// struct Rect { Point tl; Point br; };
llvm::StructType *RectTy = llvm::StructType::create(Ctx, "Rect");
RectTy->setBody({PointTy, PointTy});

llvm::AllocaInst *R = createEntryAlloca(F, RectTy, "rect");

// Access Rect.tl.x — two levels of GEP
llvm::Value *TLPtr  = B.CreateStructGEP(RectTy,  R,    0, "tl");    // Rect.tl
llvm::Value *TLXPtr = B.CreateStructGEP(PointTy, TLPtr, 0, "tl.x"); // Point.x
B.CreateStore(B.getInt32(0), TLXPtr);
```

---

## Step 6 — Arrays inside structs

```cpp
// struct Buffer { i32 len; [256 x i8] data; }
llvm::ArrayType *DataTy = llvm::ArrayType::get(B.getInt8Ty(), 256);
llvm::StructType *BufTy = llvm::StructType::get(Ctx, {B.getInt32Ty(), DataTy});

llvm::AllocaInst *Buf = createEntryAlloca(F, BufTy, "buf");

// GEP to data[5]
llvm::Value *DataPtr = B.CreateStructGEP(BufTy, Buf, 1, "data");
llvm::Value *ElemPtr = B.CreateGEP(DataTy, DataPtr,
    {B.getInt32(0), B.getInt32(5)}, "data.5");
B.CreateStore(B.getInt8('A'), ElemPtr);
```

---

## Step 7 — Unions

LLVM has no union type. Model a union as an array of bytes sized for the largest member:

```cpp
// union U { i32 i; double d; } — double is larger (8 bytes)
uint64_t MaxSize = DL.getTypeAllocSize(B.getDoubleTy()); // 8 bytes
llvm::ArrayType *UnionTy = llvm::ArrayType::get(B.getInt8Ty(), MaxSize);
llvm::AllocaInst *U = createEntryAlloca(F, UnionTy, "u");

// Write as i32 (active member = int)
llvm::Value *AsI32 = B.CreateBitCast(U, B.getPtrTy()); // ptr is already ptr in LLVM 22
B.CreateStore(B.getInt32(42),
    // In LLVM 22, all ptrs are opaque — store directly with explicit type
    B.CreatePointerCast(U, B.getPtrTy()));
// Actually in LLVM 22 with opaque pointers, just use the alloca ptr directly:
B.CreateStore(B.getInt32(42), U); // store i32 42, ptr %u

// Read as double (active member = double)
llvm::Value *D = B.CreateLoad(B.getDoubleTy(), U, "u.d");
```

---

## Step 8 — Pass structs to functions

**Small structs (2 fields or fewer):** pass by value if the calling convention allows it.

```cpp
// By value — LLVM will lower this per the platform ABI (sret, registers, etc.)
llvm::FunctionType *FT = llvm::FunctionType::get(B.getVoidTy(), {PointTy}, false);
// Pass: B.CreateCall(FT, Fn, {PointValue});
```

**Large structs:** pass by pointer (using `sret` attribute for returns):

```cpp
// Return a large struct by pointer — caller allocates, passes ptr as first arg
llvm::FunctionType *FT = llvm::FunctionType::get(
    B.getVoidTy(),
    {B.getPtrTy()}, // sret parameter
    false);
llvm::Function *Fn = llvm::Function::Create(FT, llvm::Function::ExternalLinkage,
                                              "makePoint", *M);
// Mark the first parameter as sret(PointTy)
llvm::AttrBuilder AB(Ctx);
AB.addStructRetAttr(PointTy);
Fn->addParamAttrs(0, AB);
```

---

## Step 9 — Copy a struct

```cpp
// memcpy the whole struct
B.CreateMemCpy(
    Dst,                                   // destination ptr
    llvm::MaybeAlign(DL.getABITypeAlign(PointTy)), // dst alignment
    Src,                                   // source ptr
    llvm::MaybeAlign(DL.getABITypeAlign(PointTy)), // src alignment
    DL.getTypeAllocSize(PointTy)           // size in bytes
);
```

---

## Step 10 — Recursive types (linked list nodes)

Use named (identified) structs and set the body after the type is registered:

```cpp
// struct Node { i32 val; ptr next; }
llvm::StructType *NodeTy = llvm::StructType::create(Ctx, "Node");
// ptr is opaque — no need for Node** to represent ptr-to-Node
NodeTy->setBody({B.getInt32Ty(), B.getPtrTy()});

// Access next pointer
llvm::Value *NextPtr = B.CreateStructGEP(NodeTy, NodeAlloca, 1, "next");
llvm::Value *Next    = B.CreateLoad(B.getPtrTy(), NextPtr, "next.val");
```

---

## Common mistakes

- **Never** use `getPointerElementType()` — it's removed in LLVM 22. Always pass the explicit type to `CreateLoad`, `CreateStore`, and `CreateGEP`.
- **Never** alloca structs at the point of use — always alloca in the entry block so `PromotePass` / SROA can optimize them.
- **Never** hand-compute byte offsets and emit byte GEPs — use `CreateStructGEP` with the struct type and field index.
- **Always** use `DataLayout::getStructLayout()` for field offsets in debug info — not hand-computed values.
- **Always** use `StructType::create` (not `StructType::get`) for recursive types — set the body after the type is named.
