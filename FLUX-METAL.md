# FLUX Metal: Formal Execution Model Specification

**Author:** Agent Datum 🔵 | **Session:** 6 | **Iterations:** 8 of 12
**Date:** 2026-04-14 | **Classification:** Fleet Architecture / Formal Spec
**Status:** Draft v0.1 — Reference Implementation Target

---

## 0. Purpose

This document provides a **mathematically precise** specification of the FLUX Virtual Machine execution model. It is intended as the authoritative reference for runtime implementors who need to achieve bitwise-identical execution across different hardware platforms and programming languages.

Unlike the ISA spec (which defines *what* opcodes exist) and the execution semantics comparison (which analyzes *how* runtimes differ), this document defines *exactly what each operation does* in terms that admit no ambiguity.

---

## 1. Machine State

The FLUX VM state is a 6-tuple:

```
M = (R, F, S, P, H, T)
```

| Component | Type | Description |
|-----------|------|-------------|
| R | registers | Register file |
| F | flags | Condition flags |
| S | stack | Call/data stack |
| P | program_counter | Current instruction pointer |
| H | halted | Execution state |
| T | memory | Addressable memory |

### 1.1 Register File R

```
R = (r[0], r[1], ..., r[255], f[0], f[1], ..., f[255])
```

- 256 general-purpose integer registers: r[i] in Z/2^32 (unsigned 32-bit, two's complement interpretation for signed operations)
- 256 floating-point registers: f[i] in IEEE 754 binary64 (double precision)
- All registers initialized to 0 on machine reset

### 1.2 Condition Flags F

```
F = (f_z, f_s, f_c, f_o)
```

| Flag | Set Condition |
|------|---------------|
| f_z (zero) | result == 0 |
| f_s (sign) | result & 0x80000000 != 0 (negative in two's complement) |
| f_c (carry) | unsigned overflow in arithmetic operations |
| f_o (overflow) | signed overflow in arithmetic operations |

Flags are set by arithmetic and comparison operations. Control flow operations may read flags (in runtimes that use flag-based branching; the canonical WASM runtime uses register-based branching instead).

### 1.3 Stack S

```
S = (s[0], s[1], ..., s[stack_ptr - 1])
```

- LIFO stack of 32-bit values
- Maximum depth: 4096 entries (implementation minimum)
- stack_ptr: index of next free slot (0 = empty)
- Stack overflow: error when push would exceed maximum depth
- Stack underflow: error when pop on empty stack

### 1.4 Program Counter P

```
P in {0, 1, ..., |bytecode| - 1}
```

- Byte address of the next instruction to execute
- P advances by instruction_width after each fetch
- Control flow instructions modify P directly
- P >= |bytecode| is equivalent to HALT

### 1.5 Halted Flag H

```
H in {running, halted, error}
```

- running: normal execution continues
- halted: HALT instruction executed, machine stops gracefully
- error: fatal error (division by zero, stack overflow, etc.)

### 1.6 Memory T

```
T: {0, 1, ..., MEM_SIZE - 1} -> {0, 1, ..., 255}
```

- Byte-addressable linear memory
- Minimum size: 4096 bytes (4KB)
- Reference implementation: 65536 bytes (64KB)
- Out-of-bounds access: error

---

## 2. Instruction Encoding

### 2.1 Format Definitions

```
Format A: [op]                                1 byte
Format B: [op][rd]                            2 bytes
Format C: [op][imm8]                          2 bytes
Format D: [op][rd][imm8]                      3 bytes
Format E: [op][rd][rs1][rs2]                  4 bytes
Format F: [op][rd][imm16_lo][imm16_hi]        4 bytes
Format G: [op][rd][rs1][imm16_lo][imm16_hi]   5 bytes
```

All multi-byte immediate fields are **little-endian**.

### 2.2 Immediate Value Interpretation

| Width | Signed Range | Unsigned Range |
|-------|-------------|----------------|
| imm8 | -128 to 127 | 0 to 255 |
| imm16 | -32768 to 32767 | 0 to 65535 |

Signed interpretation uses two's complement.

---

## 3. Instruction Semantics

For each instruction, we define the state transition: M -> M'. Undefined fields in the instruction format are denoted _ (ignored).

### 3.1 System Control

**HALT (0x00, Format A)**
```
H := halted
```
Machine stops. No register or memory modification.

**NOP (0x01, Format A)**
```
(no state change)
```
No operation. May be used for alignment, timing, or as a breakpoint placeholder.

**RET (0x02, Format A)**
```
Precondition: stack_ptr > 0
P := S[stack_ptr - 1]
stack_ptr := stack_ptr - 1
```
Pop return address from stack and jump to it.

**BRK (0x04, Format A)**
```
emit(event: breakpoint, pc=P)
```
Signal to attached debugger. Execution continues.

### 3.2 Single Register (Format B)

**INC rd (0x08, Format B: [0x08][rd])**
```
r[rd] := (r[rd] + 1) mod 2^32
set_flags(r[rd])
```

**DEC rd (0x09, Format B: [0x09][rd])**
```
r[rd] := (r[rd] - 1) mod 2^32
set_flags(r[rd])
```

**NOT rd (0x0A, Format B: [0x0A][rd])**
```
r[rd] := bitwise_complement(r[rd])
set_flags(r[rd])
```

**NEG rd (0x0B, Format B: [0x0B][rd])**
```
r[rd] := (-r[rd]) mod 2^32  [two's complement negation]
set_flags(r[rd])
```

**PUSH rd (0x0C, Format B: [0x0C][rd])**
```
Precondition: stack_ptr < MAX_STACK
S[stack_ptr] := r[rd]
stack_ptr := stack_ptr + 1
```

**POP rd (0x0D, Format B: [0x0D][rd])**
```
Precondition: stack_ptr > 0
stack_ptr := stack_ptr - 1
r[rd] := S[stack_ptr]
```

### 3.3 Immediate-Only (Format C)

**SYS imm8 (0x10, Format C: [0x10][imm8])**
```
syscall(imm8, r[0..7])  [platform-defined side effect]
```

**YIELD imm8 (0x15, Format C: [0x15][imm8])**
```
emit(event: yield, cycles=imm8)
```
Cooperative multitasking hint. No state change.

### 3.4 Register + Immediate 8 (Format D)

**MOVI rd, imm8 (0x18, Format D: [0x18][rd][imm8])**
```
r[rd] := sign_extend(imm8, 32)
```
Sign-extend 8-bit immediate to 32 bits: if imm8 >= 128, result = imm8 - 256.

**ADDI rd, imm8 (0x19, Format D: [0x19][rd][imm8])**
```
r[rd] := (r[rd] + sign_extend(imm8, 32)) mod 2^32
set_flags(r[rd])
```

**SUBI rd, imm8 (0x1A, Format D: [0x1A][rd][imm8])**
```
r[rd] := (r[rd] - sign_extend(imm8, 32)) mod 2^32
set_flags(r[rd])
```

### 3.5 Integer Arithmetic (Format E)

**ADD rd, rs1, rs2 (0x20, Format E: [0x20][rd][rs1][rs2])**
```
result := (r[rs1] + r[rs2]) mod 2^32
r[rd] := result
f_c := (r[rs1] + r[rs2]) > 0xFFFFFFFF  [unsigned carry]
f_o := sign(r[rs1]) == sign(r[rs2]) AND sign(result) != sign(r[rs1])
set_flags(result)
```

**SUB rd, rs1, rs2 (0x21, Format E: [0x21][rd][rs1][rs2])**
```
result := (r[rs1] - r[rs2]) mod 2^32
r[rd] := result
f_c := r[rs1] < r[rs2]  [unsigned borrow]
f_o := sign(r[rs1]) != sign(r[rs2]) AND sign(result) != sign(r[rs1])
set_flags(result)
```

**MUL rd, rs1, rs2 (0x22, Format E: [0x22][rd][rs1][rs2])**
```
result := (r[rs1] * r[rs2]) mod 2^32  [lower 32 bits of product]
r[rd] := result
set_flags(result)
```
Note: This is the lower 32 bits of the full 64-bit product. The upper 32 bits are discarded.

**DIV rd, rs1, rs2 (0x23, Format E: [0x23][rd][rs1][rs2])**
```
Precondition: r[rs2] != 0  [else: error E_DIV_ZERO]
r[rd] := truncating_signed_division(r[rs1], r[rs2])
set_flags(r[rd])
```
Signed division truncates toward zero (C99 behavior). Division of INT32_MIN by -1 is undefined (error).

**MOD rd, rs1, rs2 (0x24, Format E: [0x24][rd][rs1][rs2])**
```
Precondition: r[rs2] != 0  [else: error E_DIV_ZERO]
r[rd] := r[rs1] - truncating_signed_division(r[rs1], r[rs2]) * r[rs2]
set_flags(r[rd])
```
Remainder has the sign of the dividend.

**AND/OR/XOR/SHL/SHR** follow standard bitwise semantics with 32-bit masking.

### 3.6 Comparison (Format E)

**CMP_EQ rd, rs1, rs2 (0x2C)**
```
r[rd] := 1 if r[rs1] == r[rs2] else 0
```

**CMP_LT rd, rs1, rs2 (0x2D)**
```
r[rd] := 1 if signed(r[rs1]) < signed(r[rs2]) else 0
```

**CMP_GT rd, rs1, rs2 (0x2E)**
```
r[rd] := 1 if signed(r[rs1]) > signed(r[rs2]) else 0
```

**CMP_NE rd, rs1, rs2 (0x2F)**
```
r[rd] := 1 if r[rs1] != r[rs2] else 0
```

### 3.7 Memory Operations

**LOAD rd, rs1, rs2 (0x38, Format E)**
```
addr := (r[rs1] + r[rs2]) mod MEM_SIZE
r[rd] := T[addr] | (T[addr+1] << 8) | (T[addr+2] << 16) | (T[addr+3] << 24)
```
Load 32-bit little-endian value from memory at rs1+rs2.

**STORE rd, rs1, rs2 (0x39, Format E)**
```
addr := (r[rs1] + r[rs2]) mod MEM_SIZE
val := r[rd]
T[addr] := val & 0xFF
T[addr+1] := (val >> 8) & 0xFF
T[addr+2] := (val >> 16) & 0xFF
T[addr+3] := (val >> 24) & 0xFF
```
Store 32-bit little-endian value to memory at rs1+rs2.

### 3.8 Control Flow

**JZ rd, rs1 (0x3C, Format E: [0x3C][rd][rs1][_])**
```
if r[rd] == 0:
    P := P + sign_extend_16(r[rs1])
```
Conditional jump: if rd is zero, add signed rs1 to PC. Note: in the reference implementation, rs1 is treated as a signed 8-bit offset from the register value.

**JNZ rd, rs1 (0x3D)** — same as JZ but condition is r[rd] != 0.

**JMP imm16 (0x43, Format F: [0x43][_][lo][hi])**
```
P := P + sign_extend_16(imm16)
```
Unconditional relative jump.

**CALL imm16 (0x45, Format F: [0x45][_][lo][hi])**
```
Precondition: stack_ptr < MAX_STACK
S[stack_ptr] := P  [save return address = PC after this instruction]
stack_ptr := stack_ptr + 1
P := P + sign_extend_16(imm16)
```

**LOOP rd, imm16 (0x46, Format F: [0x46][rd][lo][hi])**
```
r[rd] := (r[rd] - 1) mod 2^32
if r[rd] > 0:
    P := P - sign_extend_16(imm16)
```

---

## 4. Helper: set_flags

```
set_flags(result):
    f_z := (result & 0xFFFFFFFF) == 0
    f_s := (result & 0x80000000) != 0
```

Only zero and sign flags are set by default. Carry and overflow are set explicitly by ADD and SUB.

---

## 5. Error Codes

| Code | Name | Trigger |
|------|------|---------|
| E000 | INVALID_OPCODE | Opcode byte not in ISA |
| E001 | INVALID_REGISTER | Register index > 255 |
| E002 | STACK_OVERFLOW | PUSH when stack is full |
| E003 | STACK_UNDERFLOW | POP when stack is empty |
| E004 | DIV_ZERO | DIV/MOD with divisor = 0 |
| E005 | MEM_OUT_OF_BOUNDS | LOAD/STORE address outside [0, MEM_SIZE) |
| E006 | CYCLE_BUDGET | Exceeded maximum cycle count |
| E007 | HALTED | Instruction executed after HALT |

---

## 6. Conformance Requirements

A runtime is FLUX Metal compliant if and only if:

1. All 17 irreducible core opcodes produce **bitwise-identical** results on identical inputs
2. All errors are reported using the error codes above
3. The machine state tuple (R, F, S, P, H, T) is observable and reproducible
4. Little-endian byte ordering is used for all multi-byte values
5. Two's complement is used for all signed integer operations

A conformance test verifies compliance by:
- Encoding a test program using only core opcodes
- Initializing R, S, T to known values
- Executing to HALT
- Comparing final R, S, T against expected values

---

## 7. Implementation Notes

- The canonical (WASM) runtime implements 71 opcodes with these exact semantics
- The Python runtime implements additional opcodes (confidence, trust, evolution) with semantics defined in FLUX-OPCODE-PRIMITIVE-THEORY.md
- Runtimes may add extensions in the 0x50-0xFF range without breaking core compliance
- The 0xFF escape prefix for v3 extensions is reserved and must not be used as a standalone opcode

---

*This specification is a living document. Submit corrections via pull request to flux-spec/FLUX-METAL.md.*
