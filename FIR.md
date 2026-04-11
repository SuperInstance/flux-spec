# FIR v1.0 — FLUX Intermediate Representation

**The canonical SSA-based intermediate representation for the FLUX compiler.**

This document is the authoritative reference for the FIR language. Every frontend
(C, Python, `.flux.md`), every optimizer pass, and every bytecode encoder must
produce or consume FIR that conforms to this specification.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Type System](#2-type-system)
3. [Module Structure](#3-module-structure)
4. [Instruction Set](#4-instruction-set)
5. [SSA Form](#5-ssa-form)
6. [Builder API](#6-builder-api)
7. [Validation Rules](#7-validation-rules)
8. [FIR to Bytecode Encoding](#8-fir-to-bytecode-encoding)
9. [Examples](#9-examples)

---

## 1. Overview

### What FIR Is

FIR (FLUX Intermediate Representation) is a type-safe, SSA-form intermediate
representation that serves as the universal bridge between all FLUX source
languages and the FLUX virtual machine bytecode. It sits at the center of the
compilation pipeline:

```
┌──────────────────┐     ┌─────────────────┐     ┌──────────────┐     ┌───────┐
│  Source Languages │ ── │  FIR (SSA form) │ ── │  Bytecode    │ ── │  VM   │
│  C / Python /     │     │  (this spec)    │     │  Encoder     │     │  Exec │
│  .flux.md         │     │                 │     │              │     │       │
└──────────────────┘     └───────┬─────────┘     └──────────────┘     └───────┘
                                 │
                          ┌──────┴──────┐
                          │ Optimizer  │
                          │ Passes     │
                          └─────────────┘
```

### Role in the Compiler

FIR is the single point of truth for program structure in the FLUX toolchain.
Every frontend (C, Python, `.flux.md`) lowers its parsed AST into FIR, and the
`BytecodeEncoder` translates FIR into FLUX binary bytecode. This decoupling
means adding a new source language requires only a new frontend — the optimizer,
validator, and encoder remain unchanged.

### Design Principles

| Principle | Description |
|-----------|-------------|
| **SSA Form** | Every value is defined exactly once by a single instruction. Block parameters replace traditional phi nodes. |
| **Type-Safe** | All values carry a `FIRType`. Instruction operands must satisfy type rules. The `TypeContext` interns types for fast identity comparison. |
| **Composable** | Modules are self-contained units that can be linked together. Functions reference each other by string name, enabling separate compilation. |
| **Serializable** | The entire IR can be encoded to binary bytecode via the `BytecodeEncoder`, or printed to human-readable text via the `print_fir` function. |
| **Validatable** | The `FIRValidator` enforces structural invariants: every block has a terminator, all block targets exist, values are well-formed. |

### Statistics

| Metric | Value |
|--------|-------|
| Type families | 16 |
| Instruction count | 54 |
| Instruction categories | 8 |
| SSA value model | Single-assignment with block parameters |
| Encoding target | FLUX bytecode (7 formats, variable-length) |

---

## 2. Type System

### Overview

The FIR type system provides 16 type families organized into four groups:
primitives, composites, higher-order, and domain-specific. All types are
immutable frozen dataclasses carrying a `type_id` for fast identity comparison.
Types are interned through a `TypeContext` so that requesting the same type
twice returns the identical Python object (`a is b`).

### BNF Grammar for Type Expressions

```ebnf
<type>       ::= <primitive> | <composite> | <higher_order> | <domain>

<primitive>  ::= <int_type>
               | <float_type>
               | "bool"
               | "unit"
               | "string"

<int_type>   ::= ("i" | "u") <bits>         ; signed or unsigned
<bits>       ::= "8" | "16" | "32" | "64"

<float_type> ::= "f" ("16" | "32" | "64")

<composite>  ::= <ref_type>
               | <array_type>
               | <vector_type>

<ref_type>   ::= "&" <type>

<array_type> ::= "[" <type> ";" <integer> "]"

<vector_type> ::= "<" <integer> "x" <type> ">"

<higher_order> ::= <func_type>
                 | <struct_type>
                 | <enum_type>

<func_type>  ::= "(" <type_list> ")" "->" "(" <type_list> ")"
<type_list>  ::= <type> ("," <type>)*

<struct_type> ::= "struct" "{" <field_list> "}"
<field_list> ::= <ident> ":" <type> ("," <ident> ":" <type>)*

<enum_type>  ::= "enum" "{" <variant_list> "}"
<variant_list> ::= <ident> ("(" <type> ")")? ("," <ident> ("(" <type> ")")?)?

<domain>     ::= <region_type>
               | <capability_type>
               | "agent"
               | "trust"

<region_type>      ::= "region<" <ident> ">"
<capability_type>  ::= "cap(" <ident> "," <ident> ")"
```

### Type Families

#### 2.1 Primitives (5 types)

| Type | Constructor | Parameters | Human-Readable |
|------|------------|------------|----------------|
| `IntType` | `ctx.get_int(bits, signed=True)` | `bits: int` (8/16/32/64), `signed: bool` | `i32`, `u64`, `i8` |
| `FloatType` | `ctx.get_float(bits)` | `bits: int` (16/32/64) | `f32`, `f64` |
| `BoolType` | `ctx.get_bool()` | — | `bool` |
| `UnitType` | `ctx.get_unit()` | — | `unit` |
| `StringType` | `ctx.get_string()` | — | `string` |

**Examples:**

```
i32       → IntType(type_id=0, bits=32, signed=True)   [canonical singleton]
f64       → FloatType(type_id=2, bits=64)
bool      → BoolType(type_id=3)
unit      → UnitType(type_id=4)
string    → StringType(type_id=5)
```

#### 2.2 Composites (3 types)

| Type | Constructor | Parameters | Human-Readable |
|------|------------|------------|----------------|
| `RefType` | `ctx.get_ref(element)` | `element: FIRType` | `&i32`, `&f64` |
| `ArrayType` | `ctx.get_array(element, length)` | `element: FIRType`, `length: int` | `[i32; 10]`, `[f32; 4]` |
| `VectorType` | `ctx.get_vector(element, lanes)` | `element: FIRType`, `lanes: int` | `<4 x f32>`, `<8 x i32>` |

`RefType` represents a pointer or reference to a value of `element` type. It is
the type produced by `alloca` and consumed by `load`/`store`.

`ArrayType` represents a fixed-size, inline array (not a heap-allocated buffer).
It carries both element type and a compile-time length.

`VectorType` represents a SIMD vector with a fixed number of lanes, used for
vector/SIMD operations in the FLUX ISA.

**Examples:**

```
&i32              → RefType(element=i32)
[i32; 10]         → ArrayType(element=i32, length=10)
<4 x f32>         → VectorType(element=f32, lanes=4)
```

#### 2.3 Higher-Order (3 types)

| Type | Constructor | Parameters | Human-Readable |
|------|------------|------------|----------------|
| `FuncType` | `ctx.get_func(params, returns)` | `params: tuple[FIRType]`, `returns: tuple[FIRType]` | `(i32, i32) -> (i32)` |
| `StructType` | `ctx.get_struct(name, fields)` | `name: str`, `fields: tuple[(str, FIRType)]` | `struct { x: f32, y: f32, z: f32 }` |
| `EnumType` | `ctx.get_enum(name, variants)` | `name: str`, `variants: tuple[(str, FIRType | None)]` | `enum { Some(i32), None }` |

`FuncType` describes a function signature with typed parameters and return types.
Parameters and returns are stored as tuples (not lists) for hashability.

`StructType` is a named product type with ordered, named fields. Each field is
a `(name: str, type: FIRType)` pair. Structs are interned by `(name, fields)`
tuple, so two struct definitions with the same name and fields return the same
object.

`EnumType` is a named sum type with variants. Each variant is either a unit
variant `(name, None)` or a data-carrying variant `(name, FIRType)`.

**Examples:**

```
(i32) -> (i32)                         → FuncType(params=(i32,), returns=(i32,))
struct { x: f32, y: f32, z: f32 }      → StructType(name="Vec3", fields=(("x",f32),("y",f32),("z",f32)))
enum { Some(i32), None }                → EnumType(name="Option", variants=(("Some",i32),("None",None)))
```

#### 2.4 Domain-Specific (4 types)

| Type | Constructor | Parameters | Human-Readable |
|------|------------|------------|----------------|
| `RegionType` | `ctx.get_region(name)` | `name: str` | `region<heap>` |
| `CapabilityType` | `ctx.get_capability(perm, resource)` | `permission: str`, `resource: str` | `cap(read, file)` |
| `AgentType` | `ctx.get_agent()` | — | `agent` |
| `TrustType` | `ctx.get_trust()` | — | `trust` |

`RegionType` represents a memory region qualifier for capability-based memory
access (e.g., `stack`, `heap`, `agent-42`). See the FLUX ISA memory model for
details on named regions and ownership transfer.

`CapabilityType` encodes a permission-resource pair for the capability-based
security system (e.g., `cap(read, file)`, `cap(write, network)`).

`AgentType` represents an agent identifier in the A2A (agent-to-agent)
communication protocol. It is used as the type of agent references in Tell,
Ask, Delegate, and TrustCheck instructions.

`TrustType` represents a trust level as a float value (0.0–1.0), used by the
trust subsystem for fleet coordination.

### TypeContext (Interning)

`TypeContext` is the type factory that ensures structural identity: requesting
the same type parameters twice returns the same object. This enables fast
identity comparison (`is`) instead of structural equality (`==`).

```python
ctx = TypeContext()

# Interning: same params → same object
a = ctx.get_int(32, signed=True)
b = ctx.get_int(32, signed=True)
assert a is b   # identity-equal

# Different params → different objects
c = ctx.get_int(32, signed=False)
assert a is not c
```

Each interned type receives a monotonically increasing `type_id` starting at 2
(type_ids 0 and 1 are reserved for the canonical `i32` and `f32` singletons).

**Canonical Singletons:**

| Symbol | Type | `type_id` |
|--------|------|-----------|
| `TypeContext.i32` | `IntType(32, signed=True)` | 0 |
| `TypeContext.f32` | `FloatType(32)` | 1 |

**TypeContext API:**

| Method | Returns | Interning Key |
|--------|---------|---------------|
| `get_int(bits, signed=True)` | `IntType` | `(IntType, (bits, signed))` |
| `get_float(bits)` | `FloatType` | `(FloatType, (bits,))` |
| `get_bool()` | `BoolType` | `(BoolType, ())` |
| `get_unit()` | `UnitType` | `(UnitType, ())` |
| `get_string()` | `StringType` | `(StringType, ())` |
| `get_ref(element)` | `RefType` | `(RefType, (element,))` |
| `get_array(element, length)` | `ArrayType` | `(ArrayType, (element, length))` |
| `get_vector(element, lanes)` | `VectorType` | `(VectorType, (element, lanes))` |
| `get_func(params, returns)` | `FuncType` | `(FuncType, (params, returns))` |
| `get_struct(name, fields)` | `StructType` | `(StructType, (name, fields))` |
| `get_enum(name, variants)` | `EnumType` | `(EnumType, (name, variants))` |
| `get_region(name)` | `RegionType` | `(RegionType, (name,))` |
| `get_capability(perm, resource)` | `CapabilityType` | `(CapabilityType, (perm, resource))` |
| `get_agent()` | `AgentType` | `(AgentType, ())` |
| `get_trust()` | `TrustType` | `(TrustType, ())` |

All types are frozen dataclasses and therefore hashable, enabling use in sets
and as dictionary keys.

---

## 3. Module Structure

### Overview

FIR programs are organized as **modules**, which contain **functions**, which
contain **blocks**, which contain **instructions**. This four-level hierarchy
mirrors the structure of most SSA-based IRs.

```
FIRModule
├── name: str
├── type_ctx: TypeContext
├── functions: dict[str, FIRFunction]
├── structs: dict[str, StructType]
└── globals: list[(name, FIRType, initial_value)]
    │
    └── FIRFunction "gcd"
        ├── name: str
        ├── sig: FuncType
        └── blocks: list[FIRBlock]
            │
            ├── FIRBlock "entry"
            │   ├── label: "entry"
            │   ├── params: [(a, i32), (b, i32)]
            │   └── instructions: [ILt, Branch, ...]
            │
            └── FIRBlock "loop"
                ├── label: "loop"
                ├── params: [(a, i32), (b, i32)]
                └── instructions: [ILt, Branch, ...]
```

### FIRModule

The top-level container for a FIR program.

```python
@dataclass
class FIRModule:
    name: str                                    # Module identifier
    type_ctx: TypeContext                         # Shared type factory
    functions: dict[str, FIRFunction]             # Named function definitions
    structs: dict[str, StructType]                # Named struct type definitions
    globals: list[(name, FIRType, initial_value)]  # Global variable declarations
```

### FIRFunction

A function composed of one or more basic blocks. The first block in the list
is always the **entry block**.

```python
@dataclass
class FIRFunction:
    name: str                # Function name (used for calls)
    sig: FuncType            # Function signature (params, returns)
    blocks: list[FIRBlock]   # Ordered list of basic blocks

    @property
    def entry_block(self) -> FIRBlock:
        """The first block is the entry point."""
        return self.blocks[0]
```

### FIRBlock

A basic block in SSA form. A block receives typed parameters (block arguments)
and contains an ordered sequence of instructions. The last instruction must be
a **terminator**.

```python
@dataclass
class FIRBlock:
    label: str                              # Block label (used for jumps)
    params: list[tuple[str, FIRType]]       # Block arguments (phi-like)
    instructions: list[Instruction]         # Instruction sequence

    @property
    def terminator(self) -> Optional[Instruction]:
        """Return the last instruction, or None if empty."""
        return self.instructions[-1] if self.instructions else None
```

**Block parameters** replace traditional phi nodes in FIR. When control flow
merges at a block, the predecessor passes values as block arguments. For
example:

```
entry(a: i32, b: i32):
    %cmp = ilt %a, %b
    branch %cmp, then(a, b), else(a, b)

then(a: i32, b: i32):
    jump merge(a)

else(a: i32, b: i32):
    jump merge(b)

merge(result: i32):
    return %result
```

Here, `merge` receives `result` from either `then` (passing `a`) or `else`
(passing `b`).

### Human-Readable Syntax

The `print_fir` function renders a module in human-readable text format:

```
module "math"

  type Vec3 = struct { x: f32, y: f32, z: f32 }

  function gcd(a: i32, b: i32) -> i32
  entry(%a:i32, %b:i32):
    %cmp = ilt %b, %a
    branch %cmp, then, else
  then:
    jump merge(%a)
  else:
    jump merge(%b)
  merge(%result:i32):
    return %result
```

### Example: Complete Module

```python
ctx = TypeContext()
i32 = ctx.get_int(32)
f32 = ctx.get_float(32)

module = FIRModule(
    name="math",
    type_ctx=ctx,
    structs={"Vec3": ctx.get_struct("Vec3", (("x", f32), ("y", f32), ("z", f32)))},
    functions={},
    globals=[("counter", i32, 0)],
)
```

---

## 4. Instruction Set

FIR defines **54 instructions** across **8 categories**. Each instruction is a
frozen dataclass implementing the `Instruction` abstract base class, which
provides an `opcode` property (lowercase string mnemonic) and an optional
`result_type` property (the `FIRType` produced, or `None` for void
instructions).

### Instruction Categories

| # | Category | Count | Mnemonics |
|---|----------|-------|-----------|
| 1 | Integer Arithmetic | 6 | `iadd`, `isub`, `imul`, `idiv`, `imod`, `ineg` |
| 2 | Float Arithmetic | 5 | `fadd`, `fsub`, `fmul`, `fdiv`, `fneg` |
| 3 | Bitwise | 6 | `iand`, `ior`, `ixor`, `ishl`, `ishr`, `inot` |
| 4 | Comparison | 11 | `ieq`, `ine`, `ilt`, `igt`, `ile`, `ige`, `feq`, `flt`, `fgt`, `fle`, `fge` |
| 5 | Conversion | 6 | `itrunc`, `zext`, `sext`, `ftrunc`, `fext`, `bitcast` |
| 6 | Memory | 9 | `load`, `store`, `alloca`, `getfield`, `setfield`, `getelem`, `setelem`, `memcpy`, `memset` |
| 7 | Control Flow | 6 | `jump`, `branch`, `switch`, `call`, `return`, `unreachable` |
| 8 | A2A Primitives | 5 | `tell`, `ask`, `delegate`, `trustcheck`, `caprequire` |

### Terminators

The following 5 instructions are terminators — they must appear as the last
instruction in every block:

| Opcode | Description |
|--------|-------------|
| `jump` | Unconditional transfer to a target block |
| `branch` | Conditional transfer to one of two blocks |
| `switch` | Multi-way branch by integer value |
| `return` | Return from the current function |
| `unreachable` | Marks code paths that should never execute |

### 4.1 Integer Arithmetic

All integer arithmetic instructions take integer-typed operands and produce a
result of the same type as the first operand. Division and modulo trap on
division by zero.

```ebnf
<int_bin_op>  ::= <value> "iadd" <value>
                 | <value> "isub" <value>
                 | <value> "imul" <value>
                 | <value> "idiv" <value>
                 | <value> "imod" <value>

<int_unary_op>::= <value> "ineg" <value>
```

| Opcode | Operands | Result Type | Semantics |
|--------|----------|-------------|-----------|
| `iadd` | `lhs: Value`, `rhs: Value` | `lhs.type` (IntType) | `lhs + rhs` |
| `isub` | `lhs: Value`, `rhs: Value` | `lhs.type` (IntType) | `lhs - rhs` |
| `imul` | `lhs: Value`, `rhs: Value` | `lhs.type` (IntType) | `lhs * rhs` |
| `idiv` | `lhs: Value`, `rhs: Value` | `lhs.type` (IntType) | `lhs / rhs` (signed) |
| `imod` | `lhs: Value`, `rhs: Value` | `lhs.type` (IntType) | `lhs % rhs` (signed) |
| `ineg` | `lhs: Value` | `lhs.type` (IntType) | `-lhs` |

**Type Rules:**
- `lhs.type` must be `IntType`
- `rhs.type` must be `IntType` (for binary ops)
- `lhs.type` and `rhs.type` must have the same bit width
- Division by zero is undefined behavior (trap in VM)

**Example:**

```
%sum = iadd %x, %y       ; sum = x + y
%neg = ineg %x            ; neg = -x
```

### 4.2 Float Arithmetic

Float arithmetic mirrors integer arithmetic but operates on `FloatType` values
with IEEE 754 semantics.

```ebnf
<float_bin_op> ::= <value> "fadd" <value>
                  | <value> "fsub" <value>
                  | <value> "fmul" <value>
                  | <value> "fdiv" <value>

<float_unary_op>::= <value> "fneg" <value>
```

| Opcode | Operands | Result Type | Semantics |
|--------|----------|-------------|-----------|
| `fadd` | `lhs: Value`, `rhs: Value` | `lhs.type` (FloatType) | `lhs + rhs` |
| `fsub` | `lhs: Value`, `rhs: Value` | `lhs.type` (FloatType) | `lhs - rhs` |
| `fmul` | `lhs: Value`, `rhs: Value` | `lhs.type` (FloatType) | `lhs * rhs` |
| `fdiv` | `lhs: Value`, `rhs: Value` | `lhs.type` (FloatType) | `lhs / rhs` |
| `fneg` | `lhs: Value` | `lhs.type` (FloatType) | `-lhs` |

**Type Rules:**
- `lhs.type` must be `FloatType`
- `rhs.type` must be `FloatType` (for binary ops)
- Both operands must have the same bit width

**Example:**

```
%dot = fadd %ax, %by     ; dot = ax + by
```

### 4.3 Bitwise

Bitwise operations work on `IntType` values. Shift amounts are masked to the
operand bit width.

```ebnf
<bitwise_bin_op> ::= <value> "iand" <value>
                    | <value> "ior" <value>
                    | <value> "ixor" <value>
                    | <value> "ishl" <value>
                    | <value> "ishr" <value>

<bitwise_unary_op>::= <value> "inot" <value>
```

| Opcode | Operands | Result Type | Semantics |
|--------|----------|-------------|-----------|
| `iand` | `lhs: Value`, `rhs: Value` | `lhs.type` | `lhs & rhs` |
| `ior` | `lhs: Value`, `rhs: Value` | `lhs.type` | `lhs \| rhs` |
| `ixor` | `lhs: Value`, `rhs: Value` | `lhs.type` | `lhs ^ rhs` |
| `ishl` | `lhs: Value`, `rhs: Value` | `lhs.type` | `lhs << rhs` |
| `ishr` | `lhs: Value`, `rhs: Value` | `lhs.type` | `lhs >> rhs` (arithmetic) |
| `inot` | `lhs: Value` | `lhs.type` | `~lhs` |

**Type Rules:**
- All operands must be `IntType`
- Shift amounts are masked: `rhs & (bits - 1)`

**Example:**

```
%mask = iand %flags, %FLAG_ON   ; mask = flags & FLAG_ON
%inv = inot %mask               ; inv = ~mask
```

### 4.4 Comparison

Comparison instructions produce a `BoolType` result. Integer comparisons use
signed ordering; float comparisons use IEEE 754 total ordering.

```ebnf
<cmp_op> ::= <value> ("ieq" | "ine" | "ilt" | "igt" | "ile" | "ige") <value>
            | <value> ("feq" | "flt" | "fgt" | "fle" | "fge") <value>
```

| Opcode | Operands | Result Type | Semantics |
|--------|----------|-------------|-----------|
| `ieq` | `lhs`, `rhs` | `BoolType` | `lhs == rhs` (integer) |
| `ine` | `lhs`, `rhs` | `BoolType` | `lhs != rhs` (integer) |
| `ilt` | `lhs`, `rhs` | `BoolType` | `lhs < rhs` (signed) |
| `igt` | `lhs`, `rhs` | `BoolType` | `lhs > rhs` (signed) |
| `ile` | `lhs`, `rhs` | `BoolType` | `lhs <= rhs` (signed) |
| `ige` | `lhs`, `rhs` | `BoolType` | `lhs >= rhs` (signed) |
| `feq` | `lhs`, `rhs` | `BoolType` | `lhs == rhs` (float) |
| `flt` | `lhs`, `rhs` | `BoolType` | `lhs < rhs` (float) |
| `fgt` | `lhs`, `rhs` | `BoolType` | `lhs > rhs` (float) |
| `fle` | `lhs`, `rhs` | `BoolType` | `lhs <= rhs` (float) |
| `fge` | `lhs`, `rhs` | `BoolType` | `lhs >= rhs` (float) |

**Type Rules:**
- `ieq`/`ine`/`ilt`/`igt`/`ile`/`ige`: both operands must be `IntType`
- `feq`/`flt`/`fgt`/`fle`/`fge`: both operands must be `FloatType`

**Example:**

```
%cmp = ilt %a, %b         ; cmp = (a < b) → bool
%ok = feq %x, %y          ; ok = (x == y) → bool
```

### 4.5 Conversion

Conversion instructions change the type or representation of a value. All take
a single value operand and a `target_type` parameter.

```ebnf
<conv_op> ::= <value> ("itrunc" | "zext" | "sext" | "ftrunc" | "fext" | "bitcast") "to" <type>
```

| Opcode | Operands | Result Type | Semantics |
|--------|----------|-------------|-----------|
| `itrunc` | `value: Value`, `target_type: FIRType` | `target_type` | Truncate integer to narrower type |
| `zext` | `value: Value`, `target_type: FIRType` | `target_type` | Zero-extend integer to wider type |
| `sext` | `value: Value`, `target_type: FIRType` | `target_type` | Sign-extend integer to wider type |
| `ftrunc` | `value: Value`, `target_type: FIRType` | `target_type` | Truncate float to narrower type |
| `fext` | `value: Value`, `target_type: FIRType` | `target_type` | Extend float to wider type |
| `bitcast` | `value: Value`, `target_type: FIRType` | `target_type` | Reinterpret bits without conversion |

**Type Rules:**
- `itrunc`: `IntType` → narrower `IntType` (e.g., `i64` → `i32`)
- `zext`/`sext`: `IntType` → wider `IntType` (e.g., `i32` → `i64`)
- `ftrunc`/`fext`: `FloatType` → narrower/wider `FloatType`
- `bitcast`: same bit-width between any types (e.g., `i32` → `f32`)

**Example:**

```
%narrow = itrunc %x to i16    ; truncate i64 to i16
%wide = zext %y to i64        ; zero-extend i32 to i64
%bits = bitcast %z to f32     ; reinterpret i32 bits as f32
```

### 4.6 Memory

Memory instructions operate on the capability-based linear memory model. The
primary operations are load, store, and stack allocation. Composite types are
accessed through field and element operations.

```ebnf
<mem_op> ::= "load" <type> "," <value> ("+" <integer>)?
            | "store" <value> "," <value> ("+" <integer>)?
            | "alloca" <type> ("," <integer>)?
            | "getfield" <value> "," <string>
            | "setfield" <value> "," <string> "," <value>
            | "getelem" <value> "," <value>
            | "setelem" <value> "," <value> "," <value>
            | "memcpy" <value> "," <value> "," <integer>
            | "memset" <value> "," <integer> "," <integer>
```

| Opcode | Operands | Result Type | Semantics |
|--------|----------|-------------|-----------|
| `load` | `type: FIRType`, `ptr: Value`, `offset: int` | `type` | Load value from `ptr + offset` |
| `store` | `value: Value`, `ptr: Value`, `offset: int` | — (void) | Store value to `ptr + offset` |
| `alloca` | `type: FIRType`, `count: int` | `RefType(type)` | Allocate `count` slots on stack |
| `getfield` | `struct_val: Value`, `field_name: str`, `field_index: int`, `field_type: FIRType` | `field_type` | Extract struct field by index |
| `setfield` | `struct_val: Value`, `field_name: str`, `field_index: int`, `value: Value` | — (void) | Set struct field by index |
| `getelem` | `array_val: Value`, `index: Value`, `elem_type: FIRType` | `elem_type` | Extract array element by index |
| `setelem` | `array_val: Value`, `index: Value`, `value: Value` | — (void) | Set array element by index |
| `memcpy` | `src: Value`, `dst: Value`, `size: int` | — (void) | Copy `size` bytes from src to dst |
| `memset` | `dst: Value`, `value: int`, `size: int` | — (void) | Fill `size` bytes at dst with value |

**Type Rules:**
- `load`: `ptr.type` must be `RefType`; result is the loaded type
- `store`: `ptr.type` must be `RefType`; `value.type` must match the pointee
- `alloca`: result is always `RefType` pointing to the allocated type
- `getfield`/`setfield`: `struct_val.type` must be `StructType`
- `getelem`/`setelem`: `array_val.type` must be `ArrayType` or `RefType(ArrayType)`
- `memcpy`/`memset`: operands must be `RefType` or compatible pointer types

**Example:**

```
%ptr = alloca i32                 ; allocate one i32 on stack
store %x, %ptr                     ; *ptr = x
%y = load i32, %ptr                ; y = *ptr

%xf = getfield %vec, "x", 0, f32   ; xf = vec.x
%elem = getelem %arr, %idx, i32    ; elem = arr[idx]
```

### 4.7 Control Flow

Control flow instructions direct execution between blocks and functions. The
terminator instructions (`jump`, `branch`, `switch`, `return`, `unreachable`)
must be the last instruction in every block.

```ebnf
<ctrl_op> ::= "jump" <label> ("(" <value_list> ")")?
             | "branch" <value> "," <label> ("(" <value_list> ")")? "," <label> ("(" <value_list> ")")?
             | "switch" <value> "," "[" <case_list> "]" "," "default:" <label>
             | "call" "@" <ident> "(" <value_list> ")" (":" <type>)?
             | "return" (<value>)?
             | "unreachable"
```

| Opcode | Operands | Result Type | Semantics |
|--------|----------|-------------|-----------|
| `jump` | `target_block: str`, `args: list[Value]` | — | Transfer to target with block arguments |
| `branch` | `cond: Value`, `true_block: str`, `false_block: str`, `args: list[Value]` | — | Conditional branch on boolean `cond` |
| `switch` | `value: Value`, `cases: dict[int, str]`, `default_block: str` | — | Multi-way branch on integer value |
| `call` | `func: str`, `args: list[Value]`, `return_type: FIRType?` | `return_type` or void | Call a named function |
| `return` | `value: Value?` | — | Return from function (void if `None`) |
| `unreachable` | — | — | Mark unreachable code path |

**Type Rules:**
- `jump`: block arguments must match target block's parameter types
- `branch`: `cond.type` must be `BoolType`
- `switch`: `value.type` must be `IntType`; case keys are integer constants
- `call`: argument types must match the callee's `FuncType.params`
- `return`: value type must match the function's `FuncType.returns`
- `unreachable`: no operands

**Example:**

```
    jump loop(%a, %b)                 ; goto loop, passing a and b
    branch %cmp, then, else           ; if cmp goto then else goto else
    switch %val, [1: one, 2: two], default: other
    %result = call @gcd(%a, %b) : i32
    return %result
    unreachable                        ; should never reach here
```

### 4.8 A2A Primitives

Agent-to-Agent (A2A) instructions implement the inter-agent communication
protocol for the FLUX fleet. These are domain-specific instructions that
require capability tokens for authorization.

```ebnf
<a2a_op> ::= "tell" "@" <ident> "," <value> "," <value>
            | "ask" "@" <ident> "," <value> "," <value> ":" <type>
            | "delegate" "@" <ident> "," <value> "," <value>
            | <value> = "trustcheck" "@" <ident> "," <value> "," <value>
            | "caprequire" <string> "," <string> "," <value>
```

| Opcode | Operands | Result Type | Semantics |
|--------|----------|-------------|-----------|
| `tell` | `target_agent: str`, `message: Value`, `cap: Value` | — | Send one-way message to agent |
| `ask` | `target_agent: str`, `message: Value`, `return_type: FIRType`, `cap: Value` | `return_type` | Blocking request to agent, returns response |
| `delegate` | `target_agent: str`, `authority: Value`, `cap: Value` | — | Delegate task authority to agent |
| `trustcheck` | `agent: str`, `threshold: Value`, `cap: Value` | `BoolType` | Check if agent's trust >= threshold |
| `caprequire` | `capability: str`, `resource: str`, `cap: Value` | — | Assert required capability, trap if missing |

**Type Rules:**
- `tell`: `cap.type` must be a capability token type
- `ask`: returns `return_type`; `cap.type` must be a capability token type
- `delegate`: `authority.type` encodes the delegated permission scope
- `trustcheck`: `threshold.type` should be `FloatType` or `IntType` (trust level)
- `caprequire`: `cap.type` must be a capability token type

**Bytecode Encoding:** All A2A instructions encode as **Format G** (variable
length) with inline agent name and capability data.

**Example:**

```
tell @navigator, %waypoint_msg, %nav_cap
%response = ask @calculator, %query, i32, %calc_cap
delegate @worker, %auth_token, %deleg_cap
%trusted = trustcheck @navigator, %threshold, %trust_cap
caprequire "write", "network", %net_cap
```

### Instruction Quick Reference

```
┌────────────────────────────────────────────────────────────────────┐
│ INTEGER ARITHMETIC                                                 │
│   iadd  lhs, rhs         → lhs + rhs                              │
│   isub  lhs, rhs         → lhs - rhs                              │
│   imul  lhs, rhs         → lhs * rhs                              │
│   idiv  lhs, rhs         → lhs / rhs                              │
│   imod  lhs, rhs         → lhs % rhs                              │
│   ineg  lhs              → -lhs                                   │
├────────────────────────────────────────────────────────────────────┤
│ FLOAT ARITHMETIC                                                   │
│   fadd  lhs, rhs         → lhs + rhs (float)                      │
│   fsub  lhs, rhs         → lhs - rhs (float)                      │
│   fmul  lhs, rhs         → lhs * rhs (float)                      │
│   fdiv  lhs, rhs         → lhs / rhs (float)                      │
│   fneg  lhs              → -lhs (float)                           │
├────────────────────────────────────────────────────────────────────┤
│ BITWISE                                                            │
│   iand  lhs, rhs         → lhs & rhs                              │
│   ior   lhs, rhs         → lhs | rhs                              │
│   ixor  lhs, rhs         → lhs ^ rhs                              │
│   ishl  lhs, rhs         → lhs << rhs                             │
│   ishr  lhs, rhs         → lhs >> rhs (arithmetic)                │
│   inot  lhs              → ~lhs                                   │
├────────────────────────────────────────────────────────────────────┤
│ COMPARISON                                                         │
│   ieq   lhs, rhs         → bool(lhs == rhs)                       │
│   ine   lhs, rhs         → bool(lhs != rhs)                       │
│   ilt   lhs, rhs         → bool(lhs <  rhs) (signed)              │
│   igt   lhs, rhs         → bool(lhs >  rhs) (signed)              │
│   ile   lhs, rhs         → bool(lhs <= rhs) (signed)              │
│   ige   lhs, rhs         → bool(lhs >= rhs) (signed)              │
│   feq   lhs, rhs         → bool(lhs == rhs) (float)               │
│   flt   lhs, rhs         → bool(lhs <  rhs) (float)               │
│   fgt   lhs, rhs         → bool(lhs >  rhs) (float)               │
│   fle   lhs, rhs         → bool(lhs <= rhs) (float)               │
│   fge   lhs, rhs         → bool(lhs >= rhs) (float)               │
├────────────────────────────────────────────────────────────────────┤
│ CONVERSION                                                         │
│   itrunc val to type      → truncate integer                      │
│   zext   val to type      → zero-extend integer                   │
│   sext   val to type      → sign-extend integer                   │
│   ftrunc val to type      → truncate float                        │
│   fext   val to type      → extend float                          │
│   bitcast val to type     → reinterpret bits                      │
├────────────────────────────────────────────────────────────────────┤
│ MEMORY                                                             │
│   load  type, ptr [off]   → *ptr                                  │
│   store val, ptr [off]    → *ptr = val                             │
│   alloca type [, n]       → &stack[n]                             │
│   getfield sv, name, i, t → sv.field_i                            │
│   setfield sv, name, i, v → sv.field_i = v                        │
│   getelem arr, idx, t     → arr[idx]                              │
│   setelem arr, idx, v     → arr[idx] = v                          │
│   memcpy src, dst, n      → copy n bytes                          │
│   memset dst, val, n      → fill n bytes                          │
├────────────────────────────────────────────────────────────────────┤
│ CONTROL FLOW (terminators)                                         │
│   jump  target(args...)    → goto target                          │
│   branch cond, t, f(..)    → if cond goto t else goto f           │
│   switch val, [k:t..], d   → multi-way branch                     │
│   call  @f(args) : ret     → result = f(args)                     │
│   return [val]              → return from function                │
│   unreachable               → trap (never reached)               │
├────────────────────────────────────────────────────────────────────┤
│ A2A PRIMITIVES                                                     │
│   tell   @agent, msg, cap   → send one-way message                │
│   ask    @agent, msg, cap   → blocking request, returns response  │
│   delegate @agent, auth, cap → delegate task authority            │
│   trustcheck @agent, th, cap → bool(trust >= threshold)           │
│   caprequire perm, res, cap → assert capability or trap           │
└────────────────────────────────────────────────────────────────────┘
```

---

## 5. SSA Form

### Overview

FIR uses **Static Single Assignment (SSA)** form, where every value is defined
exactly once by a single instruction. This is the foundational property that
enables most compiler optimizations: constant propagation, dead code
elimination, common subexpression elimination, and more.

### Values

Every computation in FIR produces a `Value`:

```python
@dataclass
class Value:
    id: int           # Unique numeric identifier
    name: str         # Human-readable name (e.g., "x", "tmp1")
    type: FIRType     # The type of this value
    const_value: int | float | None = None  # Constant value (if any)
```

**Value properties:**
- `id`: Monotonically increasing integer assigned by the `FIRBuilder`.
- `name`: Human-readable label for debugging and printing.
- `type`: Every value is statically typed.
- `const_value`: If present, this is a compile-time constant that can be
  materialized directly into bytecode without a runtime instruction.

**Printing format:** Values are printed as `%name` (e.g., `%x`, `%tmp1`).

### Single Assignment

The core SSA invariant: **every value is defined exactly once**. A value is
produced by a single instruction and cannot be redefined. This means:

1. No variable mutation — use new SSA values for new computations
2. Every use of a value dominates the single definition
3. Control flow merge uses block parameters (not phi nodes)

```python
# WRONG — would violate SSA (not possible in FIR):
# %x = iadd %a, %b
# %x = imul %x, %c   ← redefinition of %x

# RIGHT — create new value for each computation:
%x1 = iadd %a, %b
%x2 = imul %x1, %c   ← new value %x2
```

### Block Parameters (Phi-Like Merging)

Instead of traditional phi nodes, FIR uses **block parameters** to merge values
from different control flow paths. When a block receives parameters, every
predecessor block must pass the corresponding values via `jump` or `branch`
arguments.

```
entry(%x:i32, %y:i32):
    %cmp = ilt %x, %y
    branch %cmp, then(%x), else(%y)

then(%val:i32):
    jump merge(%val)

else(%val:i32):
    jump merge(%val)

merge(%result:i32):
    return %result
```

In this example, `merge` receives `result` from either `then` (where it is
`x`) or `else` (where it is `y`). The block parameter `result` acts as a phi
function: `result = phi(%x from then, %y from else)`.

### Dominance

A value `v` defined in block `B` **dominates** a use of `v` in block `C` if
every path from the entry block to `C` passes through `B`. This is the key
property that makes SSA well-formed.

**Current state:** FIR uses a simplified dominance model. The validator
currently checks local value uses within blocks but does not perform full
cross-block dominance analysis. Full dominance tree construction and SSA
validity checking is noted as **future work** (see Section 7).

### SSA to Register Mapping

During bytecode encoding, SSA values are mapped to virtual registers using
a simple scheme:

```
register_number = value.id & 0x3F    # lowest 6 bits → register 0–63
```

This gives a maximum of 64 concurrent live values per function. The register
allocator in the bytecode encoder assigns value IDs sequentially, so later
instructions use higher register numbers. For functions exceeding 64 live
values, register spilling is required (not yet implemented).

### Constant Materialization

Values with a non-None `const_value` are materialized at the start of the
function using `MOVI` instructions before any other instructions are encoded:

```
; SSA value %n (id=0, const_value=10) → register 0
; SSA value %m (id=1, const_value=3)  → register 1
MOVI R0, 10           ; materialize constant n=10
MOVI R1, 3            ; materialize constant m=3
; ... rest of function body ...
```

---

## 6. Builder API

### Overview

The `FIRBuilder` class provides a convenience API for constructing FIR modules
while automatically enforcing SSA invariants. It tracks the current block,
auto-generates value IDs, and creates `Value` objects for instruction results.

### Construction

```python
from flux.fir import TypeContext, FIRBuilder

ctx = TypeContext()
builder = FIRBuilder(ctx)
```

### Module, Function, and Block Construction

```python
# Create a new module
module = builder.new_module("my_module")

# Create a function: i32 add(i32 a, i32 b)
i32 = ctx.get_int(32)
func = builder.new_function(module, "add",
    params=[("a", i32), ("b", i32)],
    returns=[i32]
)

# Create the entry block with block parameters
entry = builder.new_block(func, "entry",
    params=[("a", i32), ("b", i32)]
)

# Set the current block for emission
builder.set_block(entry)
```

### Value Emission

The `_emit()` method is the core mechanism. It appends an instruction to the
current block and, if the instruction produces a result, creates a new `Value`
with an auto-generated ID and name:

```python
def _emit(self, instr: Instruction) -> Value | None:
    """Append instruction to current block. Returns Value if it produces one."""
    self._current_block.instructions.append(instr)
    rt = instr.result_type
    if rt is not None:
        return self._new_value(f"_v{self._next_value_id}", rt)
    return None
```

### Instruction Methods

The builder mirrors every FIR instruction as a method. Methods that produce
values return a `Value`; void methods return `None`:

**Integer Arithmetic:**

| Method | Returns | Signature |
|--------|---------|-----------|
| `iadd(lhs, rhs)` | `Value` | Binary integer add |
| `isub(lhs, rhs)` | `Value` | Binary integer sub |
| `imul(lhs, rhs)` | `Value` | Binary integer mul |
| `idiv(lhs, rhs)` | `Value` | Binary integer div |
| `imod(lhs, rhs)` | `Value` | Binary integer mod |
| `ineg(lhs)` | `Value` | Unary integer negate |

**Float Arithmetic:**

| Method | Returns | Signature |
|--------|---------|-----------|
| `fadd(lhs, rhs)` | `Value` | Float add |
| `fsub(lhs, rhs)` | `Value` | Float sub |
| `fmul(lhs, rhs)` | `Value` | Float mul |
| `fdiv(lhs, rhs)` | `Value` | Float div |
| `fneg(lhs)` | `Value` | Float negate |

**Bitwise:**

| Method | Returns | Signature |
|--------|---------|-----------|
| `iand(lhs, rhs)` | `Value` | Bitwise AND |
| `ior(lhs, rhs)` | `Value` | Bitwise OR |
| `ixor(lhs, rhs)` | `Value` | Bitwise XOR |
| `ishl(lhs, rhs)` | `Value` | Shift left |
| `ishr(lhs, rhs)` | `Value` | Shift right |
| `inot(lhs)` | `Value` | Bitwise NOT |

**Comparison:**

| Method | Returns | Signature |
|--------|---------|-----------|
| `ieq(lhs, rhs)` | `Value` | Integer equal |
| `ine(lhs, rhs)` | `Value` | Integer not equal |
| `ilt(lhs, rhs)` | `Value` | Integer less than |
| `igt(lhs, rhs)` | `Value` | Integer greater than |
| `ile(lhs, rhs)` | `Value` | Integer less or equal |
| `ige(lhs, rhs)` | `Value` | Integer greater or equal |
| `feq(lhs, rhs)` | `Value` | Float equal |
| `flt(lhs, rhs)` | `Value` | Float less than |
| `fgt(lhs, rhs)` | `Value` | Float greater than |
| `fle(lhs, rhs)` | `Value` | Float less or equal |
| `fge(lhs, rhs)` | `Value` | Float greater or equal |

**Conversion:**

| Method | Returns | Signature |
|--------|---------|-----------|
| `itrunc(value, target_type)` | `Value` | Integer truncation |
| `zext(value, target_type)` | `Value` | Zero extension |
| `sext(value, target_type)` | `Value` | Sign extension |
| `ftrunc(value, target_type)` | `Value` | Float truncation |
| `fext(value, target_type)` | `Value` | Float extension |
| `bitcast(value, target_type)` | `Value` | Bit reinterpretation |

**Memory:**

| Method | Returns | Signature |
|--------|---------|-----------|
| `load(type, ptr, offset=0)` | `Value` | Load from pointer |
| `store(value, ptr, offset=0)` | `None` | Store to pointer |
| `alloca(type, count=1)` | `Value` | Stack allocation |
| `getfield(sv, name, idx, ftype)` | `Value` | Get struct field |
| `setfield(sv, name, idx, val)` | `None` | Set struct field |
| `getelem(arr, idx, etype)` | `Value` | Get array element |
| `setelem(arr, idx, val)` | `None` | Set array element |
| `memcpy(src, dst, size)` | `None` | Memory copy |
| `memset(dst, value, size)` | `None` | Memory fill |

**Control Flow:**

| Method | Returns | Signature |
|--------|---------|-----------|
| `jump(block, args=None)` | `None` | Unconditional jump |
| `branch(cond, true, false, args=None)` | `None` | Conditional branch |
| `switch(value, cases, default)` | `None` | Multi-way switch |
| `call(func, args, return_type=None)` | `Value\|None` | Function call |
| `ret(value=None)` | `None` | Return |
| `unreachable()` | `None` | Unreachable marker |

**A2A:**

| Method | Returns | Signature |
|--------|---------|-----------|
| `tell(target, message, cap)` | `None` | Send message |
| `ask(target, message, return_type, cap)` | `Value` | Blocking request |
| `delegate(target, authority, cap)` | `None` | Delegate task |
| `trustcheck(agent, threshold, cap)` | `Value` | Trust check |
| `caprequire(capability, resource, cap)` | `None` | Require capability |

### Example: Building Factorial

```python
ctx = TypeContext()
builder = FIRBuilder(ctx)
i32 = ctx.get_int(32)
bool_t = ctx.get_bool()

mod = builder.new_module("math")
func = builder.new_function(mod, "factorial", [("n", i32)], [i32])

# Create values for parameters
n_val = Value(id=0, name="n", type=i32)
one_val = Value(id=1, name="one", type=i32, const_value=1)
cmp_val = Value(id=2, name="cmp", type=bool_t)

entry = builder.new_block(func, "entry", [("n", i32)])
loop = builder.new_block(func, "loop", [("n", i32), ("acc", i32)])
done = builder.new_block(func, "done", [("result", i32)])

# entry: if n <= 1, goto done(1) else goto loop(n, 1)
builder.set_block(entry)
builder.branch(cmp_val, "done", "loop", [one_val, n_val, one_val])

# loop: compute n * acc, decrement n, repeat
builder.set_block(loop)
prod = builder.imul(n_val, n_val)   # simplified
builder.jump("done", [n_val])

# done: return result
builder.set_block(done)
builder.ret(cmp_val)
```

---

## 7. Validation Rules

### Overview

The `FIRValidator` enforces structural invariants on a `FIRModule` after
construction. Validation is a separate pass that can be run at any point in
the compilation pipeline to ensure the IR is well-formed.

### Validation API

```python
from flux.fir import FIRValidator

validator = FIRValidator()
errors = validator.validate_module(module)

if errors:
    for error in errors:
        print(f"  ERROR: {error}")
else:
    print("Module is valid.")
```

`validate_module` returns a `list[str]`. An empty list indicates the module
is structurally valid.

### Enforced Invariants

#### V1: Every function has at least one block

```
ERROR: Function 'broken' has no blocks
```

A function with zero blocks cannot be entered. This is a hard error — the
validator returns immediately without checking further.

#### V2: Every block has exactly one terminator

```
ERROR: Block 'entry' in function 'f' has no terminator
ERROR: Block 'bad' in function 'f' has 2 terminators (expected 1)
ERROR: Block 'misplaced' in function 'f' has a terminator that is not the last instruction
```

A block must end with exactly one terminator instruction, and that terminator
must be the last instruction in the block. Terminators are: `jump`, `branch`,
`switch`, `return`, `unreachable`.

#### V3: Block target references exist

```
ERROR: Jump in 'f.entry' targets nonexistent block 'missing'
ERROR: Branch in 'f.entry' targets nonexistent block 'then'
```

Every block label referenced by a `jump`, `branch`, or `switch` must match a
block label defined in the same function.

#### V4: Value operands are well-formed

```
ERROR: Instruction in 'f.entry' has non-Value operand: int
ERROR: Instruction in 'f.entry' references Value with negative id: %bad
```

Every `Value` operand in every instruction must:
- Be an instance of `Value` (not a raw integer, string, etc.)
- Have a non-negative `id`

### Validation Gaps (Future Work)

| Check | Status | Notes |
|-------|--------|-------|
| Function has >= 1 block | ✅ Implemented | Hard error |
| Block has exactly 1 terminator | ✅ Implemented | Checks position too |
| Block targets exist | ✅ Implemented | Checks jump/branch/switch |
| Value operands well-formed | ✅ Implemented | Type/name checks |
| Block reachability | ⏳ Not checked | Warn-only (not an error in FIR) |
| Cross-block dominance | ⏳ Not checked | Requires dominance tree |
| Type consistency | ⏳ Not checked | Operands match expected types |
| Block parameter count | ⏳ Not checked | Jump args match block params |
| Function return type | ⏳ Not checked | Return value matches FuncType |

A full SSA validity checker would construct a dominance tree and verify that
every value use is dominated by its definition. This is a significant addition
planned for a future FIR spec revision.

---

## 8. FIR to Bytecode Encoding

### Overview

The `BytecodeEncoder` translates a `FIRModule` into FLUX binary bytecode. The
encoding is a two-pass process: first, all FIR types and instructions are
serialized; second, the result is packaged into the standard FLUX binary format.

### Binary Layout

```
┌──────────────────────────────────────────────────────────┐
│ Header (18 bytes)                                        │
├──────────────────────────────────────────────────────────┤
│ Type Table (variable)                                    │
├──────────────────────────────────────────────────────────┤
│ Name Pool (variable, null-terminated UTF-8 strings)      │
├──────────────────────────────────────────────────────────┤
│ Function Table (n_funcs × 12 bytes)                      │
├──────────────────────────────────────────────────────────┤
│ Code Section (variable, per-function bytecode)           │
└──────────────────────────────────────────────────────────┘
```

#### Header (18 bytes)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 4 | `magic` | `b'FLUX'` |
| 4 | 2 | `version` | `uint16 LE` — always `1` |
| 6 | 2 | `flags` | `uint16 LE` — reserved, `0` |
| 8 | 2 | `n_funcs` | `uint16 LE` — number of functions |
| 10 | 4 | `type_off` | `uint32 LE` — byte offset to type table |
| 14 | 4 | `code_off` | `uint32 LE` — byte offset to code section |

#### Type Table

The type table serializes all interned types from the `TypeContext`. Each type
is prefixed with a type kind tag:

| Tag | Type | Encoding |
|-----|------|----------|
| `0x01` | `IntType` | tag + bits(u8) + signed(u8) |
| `0x02` | `FloatType` | tag + bits(u8) |
| `0x03` | `BoolType` | tag |
| `0x04` | `UnitType` | tag |
| `0x05` | `StringType` | tag |
| `0x06` | `RefType` | tag + element_type_id(u16) |
| `0x07` | `ArrayType` | tag + element_type_id(u16) + length(u32) |
| `0x08` | `VectorType` | tag + element_type_id(u16) + lanes(u8) |
| `0x09` | `FuncType` | tag + nparams(u16) + nreturns(u16) + type_ids |
| `0x0A` | `StructType` | tag + name_len(u16) + name + nfields(u16) + fields |
| `0x0B` | `EnumType` | tag + name_len(u16) + name + nvariants(u16) + variants |
| `0x0C` | `RegionType` | tag + name_len(u16) + name |
| `0x0D` | `CapabilityType` | tag + perm_len(u16) + perm + res_len(u16) + res |
| `0x0E` | `AgentType` | tag |
| `0x0F` | `TrustType` | tag |

#### Name Pool

Null-terminated UTF-8 strings for function names. Each function name appears
once. The function table references names by byte offset into this pool.

#### Function Table

Each entry is 12 bytes:

| Offset | Size | Field |
|--------|------|-------|
| 0 | 4 | `name_offset` — byte offset into name pool |
| 4 | 4 | `entry_offset` — byte offset into code section |
| 8 | 4 | `code_size` — byte length of function's bytecode |

#### Code Section

Per-function bytecode, encoded sequentially. Contains:
1. **Constant materialization** — `MOVI` instructions for all `Value.is_const()`
2. **Instruction encoding** — FIR instructions mapped to FLUX opcodes

### FIR Opcode to Bytecode Opcode Mapping

| FIR Opcode | Bytecode Op | Encoding Format | Layout |
|------------|-------------|-----------------|--------|
| `iadd` | `IADD` (0x08) | E | `[op][rd][rs1][rs2]` |
| `isub` | `ISUB` (0x09) | E | `[op][rd][rs1][rs2]` |
| `imul` | `IMUL` (0x0A) | E | `[op][rd][rs1][rs2]` |
| `idiv` | `IDIV` (0x0B) | E | `[op][rd][rs1][rs2]` |
| `imod` | `IMOD` (0x0C) | E | `[op][rd][rs1][rs2]` |
| `iand` | `IAND` (0x10) | E | `[op][rd][rs1][rs2]` |
| `ior` | `IOR` (0x11) | E | `[op][rd][rs1][rs2]` |
| `ixor` | `IXOR` (0x12) | E | `[op][rd][rs1][rs2]` |
| `ishl` | `ISHL` (0x14) | E | `[op][rd][rs1][rs2]` |
| `ishr` | `ISHR` (0x15) | E | `[op][rd][rs1][rs2]` |
| `fadd` | `FADD` (0x40) | E | `[op][rd][rs1][rs2]` |
| `fsub` | `FSUB` (0x41) | E | `[op][rd][rs1][rs2]` |
| `fmul` | `FMUL` (0x42) | E | `[op][rd][rs1][rs2]` |
| `fdiv` | `FDIV` (0x43) | E | `[op][rd][rs1][rs2]` |
| `ineg` | `INEG` (0x0D) | B | `[op][src]` |
| `fneg` | `FNEG` (0x44) | B | `[op][src]` |
| `inot` | `INOT` (0x13) | B | `[op][src]` |
| `ieq` | `IEQ` (0x19) | E | `[op][rd][rs1][rs2]` |
| `ilt` | `ILT` (0x1A) | E | `[op][rd][rs1][rs2]` |
| `igt` | `IGT` (0x1C) | E | `[op][rd][rs1][rs2]` |
| `ile` | `ILE` (0x1B) | E | `[op][rd][rs1][rs2]` |
| `ige` | `IGE` (0x1D) | E | `[op][rd][rs1][rs2]` |
| `ine` | `IEQ` (0x19) | E | `[op][rd][rs1][rs2]` (same op, negated) |
| `feq` | `FEQ` (0x48) | E | `[op][rd][rs1][rs2]` |
| `flt` | `FLT` (0x49) | E | `[op][rd][rs1][rs2]` |
| `fgt` | `FGT` (0x4B) | E | `[op][rd][rs1][rs2]` |
| `fle` | `FLE` (0x4A) | E | `[op][rd][rs1][rs2]` |
| `fge` | `FGE` (0x4C) | E | `[op][rd][rs1][rs2]` |
| `itrunc/zext/sext/ftrunc/fext/bitcast` | `CAST` (0x38) | C | `[op][src][type_id]` |
| `load` | `LOAD` (0x02) | C | `[op][0][ptr]` |
| `store` | `STORE` (0x03) | C | `[op][val][ptr]` |
| `alloca` | `ALLOCA` (0x27) | B | `[op][0]` |
| `getfield` | `LOAD` (0x02) | E | `[op][0][src][idx]` |
| `setfield` | `STORE` (0x03) | E | `[op][val][src][idx]` |
| `getelem` | `LOAD` (0x02) | E | `[op][0][arr][idx]` |
| `setelem` | `STORE` (0x03) | E | `[op][val][arr][idx]` |
| `memcpy` | `MEMCOPY` (0x33) | E | `[op][0][src][dst]` |
| `memset` | `MEMSET` (0x34) | E | `[op][0][dst][val]` |
| `jump` | `JMP` (0x04) | D | `[op][0][offset:i16]` |
| `branch` | `JZ` (0x05) | D | `[op][cond][offset:i16]` |
| `return` | `RET` (0x28) | C | `[op][0][value]` |
| `unreachable` | `HALT` (0x80) | A | `[op]` |
| `call` | `CALL` (0x07) | D | `[op][0][offset:i16]` |
| `tell` | `TELL` (0x60) | G | `[op][len:u16][msg][cap][agent_name]` |
| `ask` | `ASK` (0x61) | G | `[op][len:u16][msg][cap][agent_name]` |
| `delegate` | `DELEGATE` (0x62) | G | `[op][len:u16][auth][cap][agent_name]` |
| `trustcheck` | `TRUST_CHECK` (0x70) | G | `[op][len:u16][thresh][cap][agent_name]` |
| `caprequire` | `CAP_REQUIRE` (0x74) | G | `[op][len:u16][cap][perm:resource]` |
| `switch` | `JMP` (0x04) | G | `[op][len:u16][val][ncases][cases...][default]` |
| (constant) | `MOVI` (0x2B) | D | `[op][reg][imm16]` |

### Encoding Format Summary

| Format | Size | Layout | Used By |
|--------|------|--------|---------|
| **A** | 1 byte | `[opcode]` | `unreachable` → `HALT` |
| **B** | 2 bytes | `[opcode][reg]` | `ineg`, `fneg`, `inot`, `alloca` |
| **C** | 3 bytes | `[opcode][a][b]` | `load`, `store`, `return`, conversions |
| **D** | 4 bytes | `[opcode][reg][imm16]` | `jump`, `branch`, `call`, `MOVI` |
| **E** | 4 bytes | `[opcode][rd][rs1][rs2]` | All binary arithmetic, comparisons |
| **G** | var | `[opcode][len][data]` | All A2A instructions, `switch` |

### SSA Value to Register Mapping

During encoding, each SSA `Value` is mapped to a register number:

```python
register_number = value.id & 0x3F    # 6 bits → registers 0–63
```

The encoder builds an `instr_to_result` mapping that tracks which instruction
produces which value ID. For binary arithmetic instructions:

```
FIR:   %result = iadd %a, %b
Bytes: [IADD][result.id & 0x3F][a.id & 0x3F][b.id & 0x3F]
```

### Constant Materialization Pass

Before encoding regular instructions, the encoder scans for all `Value` objects
with `is_const() == True` and emits `MOVI` instructions at the function
prologue:

```
; SSA:
;   %n = Value(id=0, name="n", type=i32, const_value=10)
;   %m = Value(id=1, name="m", type=i32, const_value=3)
;
; Bytecode:
;   MOVI R0, 10          ; materialize %n
;   MOVI R1, 3           ; materialize %m
```

Constants are clamped to 16-bit signed range (`-32768` to `32767`). Float
constants are bit-cast to their integer representation.

### Encoding Walkthrough: Simple Add Function

```
FIR:
  module "test"
    function add(a: i32, b: i32) -> i32
    entry(%a:i32, %b:i32):
      %result = iadd %a, %b
      return %result

Bytecode (hex):
  ; Header: FLUX v1, 1 function
  46 4C 55 58   ; magic: "FLUX"
  01 00         ; version: 1
  00 00         ; flags: 0
  01 00         ; n_funcs: 1
  12 00 00 00   ; type_off: 18
  3E 00 00 00   ; code_off: 62

  ; Type table: 2 types (i32 + i32 -> i32)
  03 00         ; 3 types
  01 20 01      ; IntType(32, signed)
  09 02 01      ; FuncType(params=2, returns=1)

  ; Name pool: "add\0"
  61 64 64 00

  ; Function table: add at offset 0, size 7
  00 00 00 00   ; name_offset: 0
  00 00 00 00   ; entry_offset: 0
  07 00 00 00   ; code_size: 7

  ; Code section:
  08 02 00 01   ; IADD rd=2, rs1=0, rs2=1  (%result = %a + %b)
  28 00 02      ; RET 0, value=2           (return %result)
```

---

## 9. Examples

### 9.1 GCD Function

The greatest common divisor (GCD) using Euclid's algorithm:

```
module "math"

  function gcd(a: i32, b: i32) -> i32
  entry(%a:i32, %b:i32):
    %is_zero = ieq %b, %zero
    branch %is_zero, done(a), loop(a, b)
  loop(%a:i32, %b:i32):
    %tmp = imod %a, %b
    jump loop(%b, %tmp)
  done(%result:i32):
    return %result
```

**Builder construction:**

```python
ctx = TypeContext()
builder = FIRBuilder(ctx)
i32 = ctx.get_int(32)

mod = builder.new_module("math")
func = builder.new_function(mod, "gcd",
    params=[("a", i32), ("b", i32)],
    returns=[i32]
)

a_val = Value(id=0, name="a", type=i32)
b_val = Value(id=1, name="b", type=i32)
zero_val = Value(id=2, name="zero", type=i32, const_value=0)

entry = builder.new_block(func, "entry", [("a", i32), ("b", i32)])
loop = builder.new_block(func, "loop", [("a", i32), ("b", i32)])
done = builder.new_block(func, "done", [("result", i32)])

# entry: if b == 0, goto done(a) else goto loop(a, b)
builder.set_block(entry)
is_zero = builder.ieq(b_val, zero_val)
builder.branch(is_zero, "done", "loop", [a_val])

# loop: tmp = a % b; goto loop(b, tmp)
builder.set_block(loop)
tmp = builder.imod(a_val, b_val)
builder.jump("loop", [b_val, tmp])

# done: return result
result_val = Value(id=3, name="result", type=i32)
builder.set_block(done)
builder.ret(result_val)
```

**Bytecode encoding:** This function encodes to approximately 25 bytes of
bytecode, using Format E for `ieq`/`imod`, Format D for `branch`/`jump`,
and Format C for `return`.

### 9.2 Factorial Function

Recursive factorial using a loop:

```
module "math"

  function factorial(n: i32) -> i32
  entry(%n:i32):
    %cmp = ilt %n, %one
    branch %cmp, base, loop(n, one)
  base(%result:i32):
    return %result
  loop(%n:i32, %acc:i32):
    %next_n = isub %n, %one
    %new_acc = imul %n, %acc
    %done = ieq %next_n, %one
    branch %done, base(new_acc), loop(next_n, new_acc)
```

**Builder construction:**

```python
ctx = TypeContext()
builder = FIRBuilder(ctx)
i32 = ctx.get_int(32)

mod = builder.new_module("math")
func = builder.new_function(mod, "factorial",
    params=[("n", i32)],
    returns=[i32]
)

n_val = Value(id=0, name="n", type=i32)
one_val = Value(id=1, name="one", type=i32, const_value=1)

entry = builder.new_block(func, "entry", [("n", i32)])
base = builder.new_block(func, "base", [("result", i32)])
loop = builder.new_block(func, "loop", [("n", i32), ("acc", i32)])

# entry: if n < 1 goto base(1) else goto loop(n, 1)
builder.set_block(entry)
cmp = builder.ilt(n_val, one_val)
builder.branch(cmp, "base", "loop", [one_val])

# base: return result
result_val = Value(id=2, name="result", type=i32)
builder.set_block(base)
builder.ret(result_val)

# loop: next_n = n - 1; new_acc = n * acc; if next_n == 1 goto base(new_acc)
builder.set_block(loop)
next_n = builder.isub(n_val, one_val)
new_acc = builder.imul(n_val, n_val)  # simplified
done = builder.ieq(next_n, one_val)
builder.branch(done, "base", "loop", [next_n, new_acc])
```

### 9.3 A2A TELL Operation

Sending a message to another agent in the FLUX fleet:

```
module "fleet"

  function report_status(status: i32, cap_token: i32) -> unit
  entry(%status:i32, %cap:i32):
    %msg = zext %status to i64
    tell @coordinator, %msg, %cap
    return
```

**Builder construction:**

```python
ctx = TypeContext()
builder = FIRBuilder(ctx)
i32 = ctx.get_int(32)
i64 = ctx.get_int(64)
unit_t = ctx.get_unit()

mod = builder.new_module("fleet")
func = builder.new_function(mod, "report_status",
    params=[("status", i32), ("cap_token", i32)],
    returns=[unit_t]
)

status = Value(id=0, name="status", type=i32)
cap = Value(id=1, name="cap", type=i32)

entry = builder.new_block(func, "entry",
    params=[("status", i32), ("cap", i32)])
builder.set_block(entry)

# Convert status to i64 message
msg = builder.zext(status, i64)

# Send to coordinator agent
builder.tell("coordinator", msg, cap)

# Return unit
builder.ret(None)
```

**Bytecode encoding:** The `tell` instruction encodes as Format G:

```
TELL [len:u16] [msg_reg:u8] [cap_reg:u8] "coordinator\0"
```

The agent name `"coordinator"` is embedded inline as UTF-8 bytes within the
variable-length Format G payload. The runtime VM extracts the agent name and
routes the message through the A2A handler callback.

### 9.4 Max Function with Block Parameter Merging

```
module "math"

  function max(a: i32, b: i32) -> i32
  entry(%a:i32, %b:i32):
    %cmp = ilt %b, %a
    branch %cmp, then(a), else(b)
  then(%a:i32):
    jump merge(%a)
  else(%b:i32):
    jump merge(%b)
  merge(%result:i32):
    return %result
```

This example demonstrates the block parameter merging pattern. The `merge`
block receives `result` from either the `then` path (where it is `a`) or the
`else` path (where it is `b`).

---

## Appendix A: Type Rendering Reference

The `print_fir` function renders types in human-readable form:

| FIRType | Rendering |
|---------|-----------|
| `IntType(32, True)` | `i32` |
| `IntType(32, False)` | `u32` |
| `IntType(64, True)` | `i64` |
| `FloatType(32)` | `f32` |
| `FloatType(64)` | `f64` |
| `BoolType()` | `bool` |
| `UnitType()` | `unit` |
| `StringType()` | `string` |
| `RefType(IntType(32))` | `&i32` |
| `ArrayType(IntType(32), 10)` | `[i32; 10]` |
| `VectorType(FloatType(32), 4)` | `<4 x f32>` |
| `FuncType((i32,), (i32,))` | `(i32) -> (i32)` |
| `StructType("Vec3", ...)` | `struct { x: f32, y: f32, z: f32 }` |
| `EnumType("Option", ...)` | `enum { Some(i32), None }` |
| `RegionType("heap")` | `region<heap>` |
| `CapabilityType("read", "file")` | `cap(read, file)` |
| `AgentType()` | `agent` |
| `TrustType()` | `trust` |

## Appendix B: Terminator Set

The set of terminator opcodes (must be last instruction in a block):

```
{"jump", "branch", "switch", "return", "unreachable"}
```

The `is_terminator(instr)` function returns `True` if an instruction is in
this set. Non-terminator instructions (all arithmetic, memory, comparison,
conversion, and A2A instructions) must never appear after a terminator in a
block.

## Appendix C: FIR Module Printing Format

The canonical text format produced by `print_fir`:

```
module "<name>"

  type <struct_name> = struct { <field>: <type>, ... }

  function <name>(<params>) -> <returns> {
  <label>(%<name>:<type>, ...):
      <instruction>
      <instruction>
      ...
  <label>(%<name>:<type>, ...):
      ...
  }
```

**Complete example:**

```
module "math"

  type Vec3 = struct { x: f32, y: f32, z: f32 }

  function add(a: i32, b: i32) -> i32
  entry(%a:i32, %b:i32):
    %_v0 = iadd %a, %b
    return %_v0

  function vec_length(v: struct { x: f32, y: f32, z: f32 }) -> f32
  entry(%v:struct { x: f32, y: f32, z: f32 }):
    %_v0 = getfield %v, "x"
    %_v1 = getfield %v, "y"
    %_v2 = getfield %v, "z"
    %_v3 = fmul %_v0, %_v0
    %_v4 = fmul %_v1, %_v1
    %_v5 = fmul %_v2, %_v2
    %_v6 = fadd %_v3, %_v4
    %_v7 = fadd %_v6, %_v5
    return %_v7
```

---

*This specification is maintained in [flux-spec](https://github.com/SuperInstance/flux-spec).
All FIR implementations, frontends, optimizers, and encoders must conform to this document.*

*Implementation reference: `flux-runtime/src/flux/fir/`*
