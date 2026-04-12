# FLUX ISA v1.0 — Complete Specification

**The canonical instruction set architecture for the FLUX Virtual Machine.**

This document is the authoritative reference for all FLUX implementations. Every VM, compiler, assembler, and debugger must conform to the encoding formats, register conventions, and execution semantics defined here.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Encoding Formats](#2-encoding-formats)
3. [Register File](#3-register-file)
4. [Memory Model](#4-memory-model)
5. [Execution Model](#5-execution-model)
6. [Condition Flags](#6-condition-flags)
7. [Opcode Map](#7-opcode-map)
8. [Opcode Reference](#8-opcode-reference)
9. [Confidence-Aware Variants](#9-confidence-aware-variants)
10. [Agent-to-Agent Primitives](#10-agent-to-agent-primitives)
11. [Conformance Requirements](#11-conformance-requirements)

**Opcode Reference Subsections:** [8.1 System Control](#81-system-control-0x00-0x03-format-a) · [8.2 Interrupt/Debug](#82-interruptdebug-0x04-0x07-format-a) · [8.3 Single Register](#83-single-register-0x08-0x0f-format-b) · [8.4 Immediate](#84-immediate-0x10-0x17-format-c) · [8.5 Register + Imm8](#85-register--imm8-0x18-0x1f-format-d) · [8.6 Integer Arithmetic](#86-integer-arithmetic-0x20-0x2f-format-e) · [8.7 Float / Memory / Control](#87-float--memory--control-0x30-0x3f-format-e) · [8.8 Register + Imm16](#88-register--imm16-0x40-0x47-format-f) · [8.9 Reg + Reg + Imm16](#89-reg--reg--imm16-0x48-0x4f-format-g) · [8.10 Long Jumps / Coroutines](#810-long-jumps--coroutines--debug-0xe0-0xef-format-f) · [8.11 Extended Memory / MMIO / GPU](#811-extended-memory--mmio--gpu-0xd0-0xdf-format-g) · [8.12 Viewpoint Operations](#812-viewpoint-operations--babel-0x70-0x7f-format-e) · [8.13 Sensor / Hardware I/O](#813-sensor--hardware-io--jetsonclaw1-0x80-0x8f-format-e) · [8.14 Extended Math / Crypto](#814-extended-math--crypto-0x90-0x9f-format-e) · [8.15 Collection / Crypto](#815-collection--crypto-0xa0-0xaf-format-deg)

---

## 1. Overview

The FLUX ISA is a 256-slot instruction set designed for a stack-based virtual machine with register operands. It supports integer and floating-point arithmetic, bitwise operations, memory management with capability-based regions, control flow, inter-agent communication (A2A protocol), confidence propagation, viewpoint semantics, sensor I/O, vector/SIMD operations, tensor primitives, and extended math/cryptography.

### Design Principles

- **Variable-length encoding** (1–5 bytes) for compact bytecode
- **Little-endian** for all multi-byte fields
- **Three-operand format** for arithmetic (rd, rs1, rs2)
- **Register + immediate formats** for common patterns (MOVI, ADDI, JMP)
- **Capability-based memory** with named regions and ownership transfer
- **Confidence-OPTIONAL** — base opcodes work without confidence; C_* variants propagate confidence alongside computation
- **ISA convergence** — unified from three agent contributions: Oracle1 (115 base), JetsonClaw1 (128 hardware), Babel (120 multilingual)

### Statistics

| Metric | Value |
|--------|-------|
| Total opcode slots | 256 |
| Defined opcodes | 247 |
| Reserved slots | 9 |
| Encoding formats | 7 (A through G) |
| Confidence variants | 16 |
| A2A primitives | 16 |
| Viewpoint ops (Babel) | 16 |
| Sensor ops (JetsonClaw1) | 16 |
| Vector/SIMD ops | 16 |
| Tensor/neural ops | 16 |
| Crypto ops | 8 |

---

## 2. Encoding Formats

FLUX uses seven instruction formats (A–G). The format is determined by the opcode value — no explicit format bits in the instruction. Multi-byte immediates use **little-endian** byte order throughout.

### Format Summary

| Format | Size | Layout | Opcode Range |
|--------|------|--------|-------------|
| **A** | 1 byte | `[opcode]` | 0x00–0x03, 0xF0–0xFF |
| **B** | 2 bytes | `[opcode][rd]` | 0x08–0x0F |
| **C** | 2 bytes | `[opcode][imm8]` | 0x10–0x17 |
| **D** | 3 bytes | `[opcode][rd][imm8]` | 0x18–0x1F |
| **E** | 4 bytes | `[opcode][rd][rs1][rs2]` | 0x20–0x3F, 0x50–0x5F, 0x60–0x6F, 0x70–0x7F, 0x80–0x8F, 0x90–0x9F, 0xB0–0xBF, 0xC0–0xCF |
| **F** | 4 bytes | `[opcode][rd][imm16_hi][imm16_lo]` | 0x40–0x47, 0xE0–0xEF |
| **G** | 5 bytes | `[opcode][rd][rs1][imm16_hi][imm16_lo]` | 0x48–0x4F, 0xA4, 0xD0–0xDF |

### Format A — System (1 byte)

Single byte. No operands. Used for system control, halt, and debug operations.

```
┌─────────┐
│ opcode  │  0x00–0x03, 0xF0–0xFF
└─────────┘
```

**Opcodes:** HALT, NOP, RET, IRET, BRK, WFI, RESET, SYN, HALT_ERR, REBOOT, DUMP, ASSERT, ID, VER, CLK, PCLK, WDOG, SLEEP, ILLEGAL

### Format B — Single Register (2 bytes)

One register operand. Used for increment, decrement, push, pop, and bitwise unary operations.

```
┌─────────┬─────────┐
│ opcode  │   rd    │  rd: 0–15
└─────────┴─────────┘
```

**Opcodes:** INC, DEC, NOT, NEG, PUSH, POP, CONF_LD, CONF_ST

### Format C — Immediate (2 bytes)

One 8-bit immediate operand. Used for system calls, traps, and debug operations.

```
┌─────────┬─────────┐
│ opcode  │  imm8   │  imm8: unsigned 8-bit
└─────────┴─────────┘
```

**Opcodes:** SYS, TRAP, DBG, CLF, SEMA, YIELD, CACHE, STRIPCF

### Format D — Register + Imm8 (3 bytes)

One register and one 8-bit immediate. Used for load-immediate and short arithmetic with constants.

```
┌─────────┬─────────┬─────────┐
│ opcode  │   rd    │  imm8   │  imm8: signed (sign-extended)
└─────────┴─────────┴─────────┘
```

**Opcodes:** MOVI, ADDI, SUBI, ANDI, ORI, XORI, SHLI, SHRI

### Format E — Three Register (4 bytes)

Three register operands. The workhorse format for arithmetic, logic, memory, control flow, A2A, confidence, viewpoint, sensor, vector, and tensor operations.

```
┌─────────┬─────────┬─────────┬─────────┐
│ opcode  │   rd    │   rs1   │   rs2   │  all registers: 0–15
└─────────┴─────────┴─────────┴─────────┘
```

For unary operations (FTOI, ITOF, MOV, SWP, ABS, etc.), rs2 is ignored by the VM but still consumes the encoding byte.

For conditional branches (JZ, JNZ, JLT, JGT), rs1 holds the offset and rs2 is ignored.

**Opcodes:** All arithmetic (ADD through CMP_NE), float (FADD through JGT), memory (LOAD, STORE, MOV, SWP), control (JZ through JGT), A2A (TELL through HEARTBT), confidence (C_ADD through C_VOTE), viewpoint (V_EVID through V_PRAGMA), sensor (SENSE through CANBUS), extended math (ABS through FCOS), vector (VLOAD through VSELECT), tensor (TMATMUL through TQUANT).

### Format F — Register + Imm16 (4 bytes)

One register and one 16-bit immediate (little-endian). Used for large immediates and long jumps.

```
┌─────────┬─────────┬──────────┬──────────┐
│ opcode  │   rd    │ imm16_hi │ imm16_lo │
└─────────┴─────────┴──────────┴──────────┘
```

imm16 is unsigned for MOVI16/ADDI16/SUBI16 and signed for jumps (JMP, JAL, CALL, etc.).

**Opcodes:** MOVI16, ADDI16, SUBI16, JMP, JAL, CALL, LOOP, SELECT, JMPL, JALL, CALLL, TAIL, SWITCH, COYIELD, CORESUM, FAULT, HANDLER, TRACE, PROF_ON, PROF_OFF, WATCH

### Format G — Register + Register + Imm16 (5 bytes)

Two registers and one 16-bit immediate offset. Used for memory operations with displacement, frame management, and extended I/O.

```
┌─────────┬─────────┬─────────┬──────────┬──────────┐
│ opcode  │   rd    │   rs1   │ imm16_hi │ imm16_lo │
└─────────┴─────────┴─────────┴──────────┴──────────┘
```

**Opcodes:** LOADOFF, STOREOFF, LOADI, STOREI, ENTER, LEAVE, COPY, FILL, SLICE, DMA_CPY through GPU_SYNC

### Opcode Range → Format Dispatch

| Opcode Range | Format |
|-------------|--------|
| 0x00–0x03 | A |
| 0x04–0x07 | A |
| 0x08–0x0F | B |
| 0x10–0x17 | C |
| 0x18–0x1F | D |
| 0x20–0x3F | E |
| 0x40–0x47 | F |
| 0x48–0x4F | G |
| 0x50–0x6F | E |
| 0x70–0xCF | E |
| 0xD0–0xDF | G |
| 0xE0–0xEF | F |
| 0xF0–0xFF | A |

---

## 3. Register File

The FLUX register file provides a unified 64-register encoding space (R0–R63), organized into functional categories for ABI convenience. The encoding specification ([ENCODING-FORMATS.md §2.2](ENCODING-FORMATS.md)) defines the canonical register map. The three-bank logical view below (GP, float, vector) maps directly onto this unified space: R0–R7 = argument/scratch, R8–R15 = callee-saved, R16–R31 = temporaries, R32–R47 = float, R48–R63 = vector/special.

### General-Purpose Registers (R0–R15)

16 signed 32-bit integer registers. All arithmetic operates on these.

| Register | ABI Name | Purpose | Callee-Saved |
|----------|----------|---------|-------------|
| R0 | ZERO | Hardwired to 0 (writes ignored) | — |
| R1 | — | Scratch / expression evaluation | No |
| R2 | — | Scratch / expression evaluation | No |
| R3 | — | Scratch / expression evaluation | No |
| R4 | — | Argument / return value | No |
| R5 | — | Argument / return value | No |
| R6 | — | Argument | No |
| R7 | — | Argument | No |
| R8 | — | Callee-saved | Yes |
| R9 | — | Callee-saved | Yes |
| R10 | — | Callee-saved | Yes |
| R11 | SP | Stack pointer | Yes |
| R12 | RID | Region ID (implicit ABI) | Yes |
| R13 | TKN | Trust token (implicit ABI) | Yes |
| R14 | FP | Frame pointer | Yes |
| R15 | LR | Link register (return address) | Yes |

### Floating-Point Registers (F0–F15)

16 IEEE 754 single-precision (32-bit) floating-point registers.

### Vector Registers (V0–V15)

16 SIMD registers, each 128 bits (16 bytes). Used for vector/SIMD operations (VLOAD, VSTORE, VADD, etc.).

> **Note — Encoding Reconciliation:** The encoding specification ([ENCODING-FORMATS.md §2.2](ENCODING-FORMATS.md)) defines a unified 64-register space (R0–R63) with 6-bit register fields in the instruction encoding. The three-bank model above (GP R0–R15, Float F0–F15/R32–R47, Vector V0–V15/R48–R63) is the logical/ABI view; the canonical encoding uses R0–R63 directly. The mapping is: R0–R7 = arguments/scratch, R8–R15 = callee-saved/frame, R16–R31 = temporaries, R32–R47 = floating-point, R48–R63 = vector/special. Implementations must use the 64-register unified encoding for instruction decode.

### Confidence Register File (Parallel)

A parallel register file indexed by rd, storing confidence values (0.0–1.0) alongside each general-purpose register. Confidence is propagated by C_* opcodes and can be loaded/stored with CONF_LD/CONF_ST. Confidence-OPTIONAL means base opcodes work without touching the confidence file; C_* variants explicitly propagate it.

---

## 4. Memory Model

FLUX uses a **capability-based linear memory** model. Memory is organized into named regions, each with an owner and optional read-only borrowers.

### Memory Regions

Each `MemoryRegion` has:
- **name** — string identifier (e.g., "stack", "heap", "agent-42")
- **data** — contiguous bytearray
- **size** — byte count
- **owner** — string identifier of the owning agent
- **borrowers** — list of read-only borrower identifiers

Regions are created, destroyed, and transferred via Format G opcodes (REGION_CREATE, REGION_DESTROY, REGION_TRANSFER) or programmatically via the MemoryManager API.

### Stack Convention

- The stack grows **downward** (from high address to low address)
- SP (R11) points to the current top of stack
- PUSH/POP operate on 32-bit (4-byte) values
- ENTER allocates a frame: saves FP, allocates `frame_size * 4` bytes
- LEAVE restores FP and deallocates the frame

### Default Regions

On initialization, the VM creates two regions:
- **stack** — size 65536 bytes, owner "system"
- **heap** — size 65536 bytes, owner "system"

### Heap Allocation

MALLOC (0xD7) and FREE (0xD8) provide dynamic heap allocation with 16-byte immediate size parameter.

### Typed Memory Access

All memory access uses **little-endian** byte order:
- `read_i32` / `write_i32` — signed 32-bit integers
- `read_f32` / `write_f32` — IEEE 754 single-precision floats
- `read(offset, size)` / `write(offset, data)` — raw byte access

---

## 5. Execution Model

The FLUX VM uses a **fetch-decode-execute** loop with a cycle budget.

### Initialization

1. Bytecode is loaded into the VM
2. Register file is zeroed
3. Stack pointer is set to the top of the stack region
4. Condition flags are cleared
5. Program counter starts at 0

### Execution Loop

```
while running and not halted and cycles < max_cycles:
    fetch instruction at PC
    decode operands based on opcode
    execute operation
    update flags if applicable
    advance PC by instruction size
    increment cycle count
```

### Cycle Budget

Default maximum: **10,000,000 cycles** (10M). Configurable per VM instance. When exceeded, execution halts gracefully (not an error).

### Jump Semantics

All jump offsets are **relative to the PC after the instruction has been fully fetched**. That is, the PC is already pointing past the instruction when the offset is applied. A JMP with offset 0 is a no-op; offset -4 jumps back to the JMP instruction itself.

### Three-Operand Read-Before-Write

For Format E instructions (rd, rs1, rs2), the VM reads rs1 and rs2 **before** writing rd. This means `ADD R3, R3, R5` (rd=3, rs1=3, rs2=5) correctly computes R3 = R3 + R5 without clobbering the source.

---

## 6. Condition Flags

Four condition flags track the result of arithmetic and comparison operations:

| Flag | Name | Set When |
|------|------|----------|
| **Zero** (Z) | Zero flag | Result is exactly 0 |
| **Sign** (S) | Sign flag | Result is negative (bit 31 set) |
| **Carry** (C) | Carry flag | Unsigned borrow occurred (a < b in subtraction) |
| **Overflow** (O) | Overflow flag | Signed overflow occurred |

### Flag-Setting Instructions

- All arithmetic ops (ADD, SUB, MUL, etc.) set flags via `_set_flags(result)`
- CMP/ICMP sets flags via `_set_cmp_flags(a, b)` — tracks carry/overflow from subtraction
- TEST performs bitwise AND without storing result, sets flags only
- SETCC reads flags and writes 0 or 1 to a register based on a condition code

### Condition Codes (SETCC)

| Code | Condition | Flags Tested |
|------|-----------|-------------|
| 0 | EQ | Z == 1 |
| 1 | NE | Z == 0 |
| 2 | LT | S == 1 |
| 3 | GE | S == 0 |
| 4 | GT | Z == 0 and S == 0 |
| 5 | LE | Z == 1 or S == 1 |
| 6 | CS/HS | C == 1 |
| 7 | CC/LO | C == 0 |
| 8 | VS | O == 1 |
| 9 | VC | O == 0 |

### Flag-Based Branches

| Opcode | Condition | Description |
|--------|-----------|-------------|
| JE | Z == 1 | Jump if equal |
| JNE | Z == 0 | Jump if not equal |
| JG | Z == 0 and S == 0 | Jump if greater (signed) |
| JL | S == 1 | Jump if less (signed) |
| JGE | S == 0 | Jump if greater or equal |
| JLE | Z == 1 or S == 1 | Jump if less or equal |

---

## 7. Opcode Map

### Visual Overview

```
0x00-0x03  A  ██████ System Control (HALT, NOP, RET, IRET)
0x04-0x07  A  ██████ Interrupt/Debug (BRK, WFI, RESET, SYN)
0x08-0x0F  B  ██████ Single Register (INC, DEC, NOT, NEG, PUSH, POP, CONF_LD, CONF_ST)
0x10-0x17  C  ██████ Immediate (SYS, TRAP, DBG, CLF, SEMA, YIELD, CACHE, STRIPCF)
0x18-0x1F  D  ██████ Reg+Imm8 (MOVI, ADDI, SUBI, ANDI, ORI, XORI, SHLI, SHRI)
0x20-0x2F  E  ██████ Integer Arithmetic (ADD, SUB, MUL, DIV, MOD, AND, OR, XOR, SHL, SHR, MIN, MAX, CMP_EQ/LT/GT/NE)
0x30-0x3F  E  ██████ Float/Memory/Control (FADD, FSUB, FMUL, FDIV, FMIN, FMAX, FTOI, ITOF, LOAD, STORE, MOV, SWP, JZ, JNZ, JLT, JGT)
0x40-0x47  F  ██████ Reg+Imm16 (MOVI16, ADDI16, SUBI16, JMP, JAL, CALL, LOOP, SELECT)
0x48-0x4F  G  ██████ Reg+Reg+Imm16 (LOADOFF, STOREOFF, LOADI, STOREI, ENTER, LEAVE, COPY, FILL)
0x50-0x5F  E  ██████ Agent-to-Agent (TELL, ASK, DELEG, BCAST, ACCEPT, DECLINE, REPORT, MERGE, FORK, JOIN, SIGNAL, AWAIT, TRUST, DISCOV, STATUS, HEARTBT)
0x60-0x6F  E  ██████ Confidence (C_ADD, C_SUB, C_MUL, C_DIV, C_FADD/SUB/MUL/DIV, C_MERGE, C_THRESH, C_BOOST, C_DECAY, C_SOURCE, C_CALIB, C_EXPLY, C_VOTE)
0x70-0x7F  E  ██████ Viewpoint — Babel (V_EVID, V_EPIST, V_MIR, V_NEG, V_TENSE, V_ASPEC, V_MODAL, V_POLIT, V_HONOR, V_TOPIC, V_FOCUS, V_CASE, V_AGREE, V_CLASS, V_INFL, V_PRAGMA)
0x80-0x8F  E  ██████ Sensor — JetsonClaw1 (SENSE, ACTUATE, SAMPLE, ENERGY, TEMP, GPS, ACCEL, DEPTH, CAMCAP, CAMDET, PWM, GPIO, I2C, SPI, UART, CANBUS)
0x90-0x9F  E  ██████ Extended Math/Crypto (ABS, SIGN, SQRT, POW, LOG2, CLZ, CTZ, POPCNT, CRC32, SHA256, RND, SEED, FMOD, FSQRT, FSIN, FCOS)
0xA0-0xAF  D/E ██████ String/Collection (LEN, CONCAT, AT, SETAT, SLICE, REDUCE, MAP, FILTER, SORT, FIND, HASH, HMAC, VERIFY, ENCRYPT, DECRYPT, KEYGEN)
0xB0-0xBF  E  ██████ Vector/SIMD (VLOAD, VSTORE, VADD, VMUL, VDOT, VNORM, VSCALE, VMAXP, VMINP, VREDUCE, VGATHER, VSCATTER, VSHUF, VMERGE, VCONF, VSELECT)
0xC0-0xCF  E  ██████ Tensor/Neural (TMATMUL, TCONV, TPOOL, TRELU, TSIGM, TSOFT, TLOSS, TGRAD, TUPDATE, TADAM, TEMBED, TATTN, TSAMPLE, TTOKEN, TDETOK, TQUANT)
0xD0-0xDF  G  ██████ Extended Memory/I/O (DMA_CPY, DMA_SET, MMIO_R, MMIO_W, ATOMIC, CAS, FENCE, MALLOC, FREE, MPROT, MCACHE, GPU_LD, GPU_ST, GPU_EX, GPU_SYNC)
0xE0-0xEF  F  ██████ Long Jumps/Calls/Debug (JMPL, JALL, CALLL, TAIL, SWITCH, COYIELD, CORESUM, FAULT, HANDLER, TRACE, PROF_ON, PROF_OFF, WATCH)
0xF0-0xFF  A  ██████ Extended System (HALT_ERR, REBOOT, DUMP, ASSERT, ID, VER, CLK, PCLK, WDOG, SLEEP, ILLEGAL)
```

---

## 8. Opcode Reference

### 8.1 System Control (0x00–0x03, Format A)

| Hex | Mnemonic | Description |
|-----|----------|-------------|
| 0x00 | **HALT** | Stop execution. Sets `halted = True`. |
| 0x01 | **NOP** | No operation. Pipeline synchronization point. |
| 0x02 | **RET** | Return from subroutine. Pops return address from stack and sets PC. If stack is empty (returning from top-level), halts the VM. |
| 0x03 | **IRET** | Return from interrupt handler. Restores saved processor state. |

### 8.2 Interrupt/Debug (0x04–0x07, Format A)

| Hex | Mnemonic | Description |
|-----|----------|-------------|
| 0x04 | **BRK** | Breakpoint. Traps to debugger. |
| 0x05 | **WFI** | Wait for interrupt. Enters low-power idle state. |
| 0x06 | **RESET** | Soft reset. Clears register file to zero. |
| 0x07 | **SYN** | Memory barrier / synchronize. Ensures all pending memory operations complete before continuing. |

### 8.3 Single Register (0x08–0x0F, Format B)

| Hex | Mnemonic | Operands | Description |
|-----|----------|----------|-------------|
| 0x08 | **INC** | rd | rd = rd + 1. Sets flags. |
| 0x09 | **DEC** | rd | rd = rd - 1. Sets flags. |
| 0x0A | **NOT** | rd | rd = bitwise NOT of rd. Sets flags. |
| 0x0B | **NEG** | rd | rd = -rd (arithmetic negation). Sets flags. |
| 0x0C | **PUSH** | rd | Push rd onto stack. SP decremented by 4. |
| 0x0D | **POP** | rd | Pop from stack into rd. SP incremented by 4. |
| 0x0E | **CONF_LD** | rd | Load confidence register for rd into the confidence accumulator. |
| 0x0F | **CONF_ST** | rd | Store confidence accumulator to the confidence register for rd. |

### 8.4 Immediate (0x10–0x17, Format C)

| Hex | Mnemonic | Operands | Description |
|-----|----------|----------|-------------|
| 0x10 | **SYS** | imm8 | System call with code imm8. |
| 0x11 | **TRAP** | imm8 | Software interrupt. Dispatches to vector imm8. |
| 0x12 | **DBG** | imm8 | Debug print register imm8. |
| 0x13 | **CLF** | imm8 | Clear condition flag bits specified by imm8. |
| 0x14 | **SEMA** | imm8 | Semaphore operation (wait/signal based on imm8). |
| 0x15 | **YIELD** | imm8 | Yield execution for imm8 cycles. Cooperative multitasking. |
| 0x16 | **CACHE** | imm8 | Cache control: flush (0), invalidate (1), flush+invalidate (2). |
| 0x17 | **STRIPCF** | imm8 | Strip confidence metadata from the next imm8 instructions. Used by edge devices to skip confidence propagation for performance. |

### 8.5 Register + Imm8 (0x18–0x1F, Format D)

| Hex | Mnemonic | Operands | Description |
|-----|----------|----------|-------------|
| 0x18 | **MOVI** | rd, imm8 | rd = sign_extend(imm8). Load an 8-bit signed immediate into a register. |
| 0x19 | **ADDI** | rd, imm8 | rd = rd + sign_extend(imm8). |
| 0x1A | **SUBI** | rd, imm8 | rd = rd - sign_extend(imm8). |
| 0x1B | **ANDI** | rd, imm8 | rd = rd & zero_extend(imm8). |
| 0x1C | **ORI** | rd, imm8 | rd = rd \| zero_extend(imm8). |
| 0x1D | **XORI** | rd, imm8 | rd = rd ^ zero_extend(imm8). |
| 0x1E | **SHLI** | rd, imm8 | rd = rd << imm8 (logical left shift). |
| 0x1F | **SHRI** | rd, imm8 | rd = rd >> imm8 (arithmetic right shift). |

### 8.6 Integer Arithmetic (0x20–0x2F, Format E)

| Hex | Mnemonic | Operands | Description |
|-----|----------|----------|-------------|
| 0x20 | **ADD** | rd, rs1, rs2 | rd = rs1 + rs2. Sets flags. |
| 0x21 | **SUB** | rd, rs1, rs2 | rd = rs1 - rs2. Sets flags. |
| 0x22 | **MUL** | rd, rs1, rs2 | rd = rs1 * rs2. Sets flags. 32-bit truncated result. |
| 0x23 | **DIV** | rd, rs1, rs2 | rd = rs1 / rs2 (signed integer division). Traps on division by zero. |
| 0x24 | **MOD** | rd, rs1, rs2 | rd = rs1 % rs2 (signed modulo). Traps on division by zero. |
| 0x25 | **AND** | rd, rs1, rs2 | rd = rs1 & rs2. Sets flags. |
| 0x26 | **OR** | rd, rs1, rs2 | rd = rs1 \| rs2. Sets flags. |
| 0x27 | **XOR** | rd, rs1, rs2 | rd = rs1 ^ rs2. Sets flags. |
| 0x28 | **SHL** | rd, rs1, rs2 | rd = rs1 << (rs2 & 0x3F). Sets flags. |
| 0x29 | **SHR** | rd, rs1, rs2 | rd = rs1 >> (rs2 & 0x3F). Sets flags. Arithmetic (sign-extending) right shift. |
| 0x2A | **MIN** | rd, rs1, rs2 | rd = min(rs1, rs2). Sets flags. |
| 0x2B | **MAX** | rd, rs1, rs2 | rd = max(rs1, rs2). Sets flags. |
| 0x2C | **CMP_EQ** | rd, rs1, rs2 | rd = 1 if rs1 == rs2, else 0. |
| 0x2D | **CMP_LT** | rd, rs1, rs2 | rd = 1 if rs1 < rs2 (signed), else 0. |
| 0x2E | **CMP_GT** | rd, rs1, rs2 | rd = 1 if rs1 > rs2 (signed), else 0. |
| 0x2F | **CMP_NE** | rd, rs1, rs2 | rd = 1 if rs1 != rs2, else 0. |

### 8.7 Float / Memory / Control (0x30–0x3F, Format E)

| Hex | Mnemonic | Operands | Description |
|-----|----------|----------|-------------|
| 0x30 | **FADD** | rd, rs1, rs2 | rd = float(rs1) + float(rs2). |
| 0x31 | **FSUB** | rd, rs1, rs2 | rd = float(rs1) - float(rs2). |
| 0x32 | **FMUL** | rd, rs1, rs2 | rd = float(rs1) * float(rs2). |
| 0x33 | **FDIV** | rd, rs1, rs2 | rd = float(rs1) / float(rs2). Traps on zero. |
| 0x34 | **FMIN** | rd, rs1, rs2 | rd = fmin(rs1, rs2). |
| 0x35 | **FMAX** | rd, rs1, rs2 | rd = fmax(rs1, rs2). |
| 0x36 | **FTOI** | rd, rs1, - | rd = int(float(rs1)). Float-to-integer conversion. rs2 ignored. |
| 0x37 | **ITOF** | rd, rs1, - | rd = float(rs1). Integer-to-float conversion. rs2 ignored. |
| 0x38 | **LOAD** | rd, rs1, rs2 | rd = mem[rs1 + rs2]. Load 32-bit value from memory. |
| 0x39 | **STORE** | rd, rs1, rs2 | mem[rs1 + rs2] = rd. Store 32-bit value to memory. |
| 0x3A | **MOV** | rd, rs1, - | rd = rs1. Register copy. rs2 ignored. |
| 0x3B | **SWP** | rd, rs1, - | swap(rd, rs1). Exchange two register values. rs2 ignored. |
| 0x3C | **JZ** | rd, rs1, - | If rd == 0: PC += rs1. rs2 ignored. |
| 0x3D | **JNZ** | rd, rs1, - | If rd != 0: PC += rs1. rs2 ignored. |
| 0x3E | **JLT** | rd, rs1, - | If rd < 0: PC += rs1. rs2 ignored. |
| 0x3F | **JGT** | rd, rs1, - | If rd > 0: PC += rs1. rs2 ignored. |

### 8.8 Register + Imm16 (0x40–0x47, Format F)

| Hex | Mnemonic | Operands | Description |
|-----|----------|----------|-------------|
| 0x40 | **MOVI16** | rd, imm16 | rd = imm16. Load 16-bit unsigned immediate. |
| 0x41 | **ADDI16** | rd, imm16 | rd = rd + imm16. |
| 0x42 | **SUBI16** | rd, imm16 | rd = rd - imm16. |
| 0x43 | **JMP** | rd, imm16 | PC += sign_extend(imm16). Unconditional relative jump. rd ignored. |
| 0x44 | **JAL** | rd, imm16 | rd = PC; PC += sign_extend(imm16). Jump and link (save return address). |
| 0x45 | **CALL** | rd, imm16 | PUSH(PC); PC = rd + imm16. Indirect call with 16-bit offset. |
| 0x46 | **LOOP** | rd, imm16 | rd--; if rd > 0: PC -= imm16. Hardware loop construct. |
| 0x47 | **SELECT** | rd, imm16 | PC += imm16 * rd. Computed jump (branch table dispatch). |

### 8.9 Reg + Reg + Imm16 (0x48–0x4F, Format G)

| Hex | Mnemonic | Operands | Description |
|-----|----------|----------|-------------|
| 0x48 | **LOADOFF** | rd, rs1, imm16 | rd = mem[rs1 + sign_extend(imm16)]. Load with 16-bit displacement. |
| 0x49 | **STOREOFF** | rd, rs1, imm16 | mem[rs1 + sign_extend(imm16)] = rd. Store with 16-bit displacement. |
| 0x4A | **LOADI** | rd, rs1, imm16 | rd = mem[mem[rs1] + imm16]. Indirect load (load from address stored in memory). |
| 0x4B | **STOREI** | rd, rs1, imm16 | mem[mem[rs1] + imm16] = rd. Indirect store. |
| 0x4C | **ENTER** | rd, rs1, imm16 | Push registers; SP -= imm16; rd = old SP. Function prologue. |
| 0x4D | **LEAVE** | rd, rs1, imm16 | SP += imm16; pop registers; rd = return value. Function epilogue. |
| 0x4E | **COPY** | rd, rs1, imm16 | memcpy(rd, rs1, imm16). Copy imm16 bytes from address rs1 to rd. |
| 0x4F | **FILL** | rd, rs1, imm16 | memset(rd, rs1, imm16). Fill imm16 bytes at address rd with value rs1. |

### 8.10 Long Jumps / Coroutines / Debug (0xE0–0xEF, Format F)

| Hex | Mnemonic | Operands | Description |
|-----|----------|----------|-------------|
| 0xE0 | **JMPL** | rd, imm16 | Long relative jump: PC += sign_extend(imm16). rd unused. |
| 0xE1 | **JALL** | rd, imm16 | Long jump-and-link: rd = PC; PC += sign_extend(imm16). |
| 0xE2 | **CALLL** | rd, imm16 | Long call: PUSH(PC); PC = rd + imm16. |
| 0xE3 | **TAIL** | rd, imm16 | Tail call: pop frame; PC = rd + imm16. |
| 0xE4 | **SWITCH** | rd, imm16 | Context switch: save state, jump imm16. |
| 0xE5 | **COYIELD** | rd, imm16 | Coroutine yield: save state, jump to imm16. |
| 0xE6 | **CORESUM** | rd, imm16 | Coroutine resume: restore state, jump to rd. |
| 0xE7 | **FAULT** | rd, imm16 | Raise fault code imm16, context rd. |
| 0xE8 | **HANDLER** | rd, imm16 | Install fault handler at PC + imm16. |
| 0xE9 | **TRACE** | rd, imm16 | Log rd, tag imm16. |
| 0xEA | **PROF_ON** | rd, imm16 | Start profiling region imm16. |
| 0xEB | **PROF_OFF** | rd, imm16 | End profiling region imm16. |
| 0xEC | **WATCH** | rd, imm16 | Watchpoint: break on write to rd + imm16. |

### 8.11 Extended Memory / MMIO / GPU (0xD0–0xDF, Format G)

| Hex | Mnemonic | Operands | Description |
|-----|----------|----------|-------------|
| 0xD0 | **DMA_CPY** | rd, rs1, imm16 | DMA copy imm16 bytes: rd ← rs1. |
| 0xD1 | **DMA_SET** | rd, rs1, imm16 | DMA fill imm16 bytes at rd with value rs1. |
| 0xD2 | **MMIO_R** | rd, rs1, imm16 | Memory-mapped I/O read: rd = io[rs1 + imm16]. |
| 0xD3 | **MMIO_W** | rd, rs1, imm16 | Memory-mapped I/O write: io[rs1 + imm16] = rd. |
| 0xD4 | **ATOMIC** | rd, rs1, imm16 | Atomic read-modify-write: rd = swap(mem[rs1+imm16], rd). |
| 0xD5 | **CAS** | rd, rs1, imm16 | Compare-and-swap at address rs1 + imm16. |
| 0xD6 | **FENCE** | rd, rs1, imm16 | Memory fence: type imm16 (acquire/release/full). |
| 0xD7 | **MALLOC** | rd, rs1, imm16 | Allocate imm16 bytes on heap. Handle → rd. |
| 0xD8 | **FREE** | rd, rs1, imm16 | Free allocation at address rd. |
| 0xD9 | **MPROT** | rd, rs1, imm16 | Memory protect: start=rd, len=rs1, flags=imm16. |
| 0xDA | **MCACHE** | rd, rs1, imm16 | Cache management: op=imm16, addr=rd, len=rs1. |
| 0xDB | **GPU_LD** | rd, rs1, imm16 | GPU load to device memory, offset imm16. |
| 0xDC | **GPU_ST** | rd, rs1, imm16 | GPU store from device memory. |
| 0xDD | **GPU_EX** | rd, rs1, imm16 | GPU execute kernel rd, grid=rs1, block=imm16. |
| 0xDE | **GPU_SYNC** | rd, rs1, imm16 | Synchronize GPU device imm16. |

### 8.12 Viewpoint Operations — Babel (0x70–0x7F, Format E)

Viewpoint operations encode cross-linguistic semantic categories from the Babel contributor. They enable agents to reason about evidentiality, epistemic stance, tense, aspect, modality, politeness, and pragmatic context — concepts that vary across human languages but can be unified for AI coordination.

| Mnemonic | Operands | Semantics | Flags |
|----------|----------|-----------|-------|
| **V_EVID** | rd, rs1, rs2 | Classify the evidentiality (source of information) for the proposition in rs1. Source type rs2 (0=direct, 1=reported, 2=inferred, 3=hearsay) is stored in rd. Used to tag agent assertions with how the knowledge was obtained. | — |
| **V_EPIST** | rd, rs1, rs2 | Attach an epistemic stance (certainty level) to rs1. The certainty degree rs2 (0=unknown through 1=certain) is written to rd. Enables agents to qualify beliefs explicitly rather than relying solely on confidence values. | — |
| **V_MIR** | rd, rs1, rs2 | Mark rs1 with a mirative (unexpectedness) value. Mirativity degree rs2 (0=expected through 1=highly surprising) is stored in rd. Useful for highlighting novel information in multi-agent discourse. | — |
| **V_NEG** | rd, rs1, rs2 | Apply negation scope to rs1. Scope type rs2 (0=predicate negation, 1=proposition negation, 2=metalinguistic negation) determines how the negation interacts with inference. Result in rd. | — |
| **V_TENSE** | rd, rs1, rs2 | Align the temporal viewpoint of rs1. Tense marker rs2 (0=past, 1=present, 2=future, 3=anterior) is applied and the annotated result stored in rd. | — |
| **V_ASPEC** | rd, rs1, rs2 | Set the aspectual viewpoint (completeness) for rs1. Aspect rs2 (0=perfective/completed, 1=imperfective/ongoing, 2=habitual) determines how the event is framed temporally. Result in rd. | — |
| **V_MODAL** | rd, rs1, rs2 | Apply modal force to rs1. Modality rs2 encodes necessity/possibility (0=must, 1=should, 2=may, 3=can). Result in rd influences agent planning constraints. | — |
| **V_POLIT** | rd, rs1, rs2 | Map rs1 to a politeness register. Level rs2 (0=formal, 1=polite, 2=casual, 3=intimate) is stored in rd. Affects how agents phrase messages in A2A communication. | — |
| **V_HONOR** | rd, rs1, rs2 | Translate an honorific level rs2 (0=none through 3=highest) into a trust tier for agent rs1. The computed tier is stored in rd. Bridges cultural pragmatic conventions with fleet trust semantics. | — |
| **V_TOPIC** | rd, rs1, rs2 | Bind a topic-comment structure: rs1 is the topic (what is being discussed), rs2 is the comment (new information about the topic). The bound structure handle is stored in rd. | — |
| **V_FOCUS** | rd, rs1, rs2 | Mark information focus on rs1. Focus type rs2 (0=broad, 1=narrow, 2=contrastive) identifies which constituent carries new or contrastive information. Result in rd. | — |
| **V_CASE** | rd, rs1, rs2 | Assign case-based scope to rs1. Case marker rs2 (0=nominative through 7=instrumental, following typological conventions) determines the grammatical role in a multi-agent coordination context. Result in rd. | — |
| **V_AGREE** | rd, rs1, rs2 | Check grammatical agreement features between rs1 and rs2. Agreement dimensions include gender (bit 0), number (bit 1), and person (bit 2). The agreement bitmask is stored in rd. | Z |
| **V_CLASS** | rd, rs1, rs2 | Map a classifier rs2 to a type category for rs1. Different languages use distinct classifier systems (counters, measure words); this opcode normalizes them into a shared type space stored in rd. | — |
| **V_INFL** | rd, rs1, rs2 | Translate morphological inflection rs2 into a control-flow effect. Inflection patterns (e.g., imperative, subjunctive, conditional) modulate how the agent interprets the associated instruction. Result in rd. | — |
| **V_PRAGMA** | rd, rs1, rs2 | Switch the pragmatic context for subsequent operations. Context rs2 selects a predefined pragmatic frame (e.g., negotiation, instruction, narrative). The previous context handle is saved to rd. | — |

### 8.13 Sensor / Hardware I/O — JetsonClaw1 (0x80–0x8F, Format E)

Sensor opcodes provide direct hardware I/O access for edge and robotic agents, contributed by the JetsonClaw1 platform. They abstract common peripheral protocols (I2C, SPI, UART, PWM, GPIO, CAN bus) and integrated sensors (camera, GPS, IMU, temperature).

| Mnemonic | Operands | Semantics | Flags |
|----------|----------|-----------|-------|
| **SENSE** | rd, rs1, rs2 | Read a value from sensor rs1 on channel rs2. The sampled value is stored in rd. Blocks until data is available. | — |
| **ACTUATE** | rd, rs1, rs2 | Write the value rd to actuator rs1 on channel rs2. Used to control motors, servos, LEDs, and other output devices. | — |
| **SAMPLE** | rd, rs1, rs2 | Perform ADC sampling on channel rs1, averaging over rs2 readings. The averaged result is stored in rd. Higher rs2 values yield smoother readings at the cost of latency. | — |
| **ENERGY** | rd, rs1, rs2 | Query the energy subsystem. Available energy budget (in millijoules) is stored in rd; consumed energy since last query is stored in rs1. rs2 is reserved for future use. | — |
| **TEMP** | rd, rs1, rs2 | Read the on-board temperature sensor in millidegrees Celsius. Result stored in rd. rs1 and rs2 select the sensor instance and averaging mode respectively. | — |
| **GPS** | rd, rs1, rs2 | Read GPS coordinates. Latitude (fixed-point Q24.8 degrees) is stored in rd; longitude in rs1. rs2 selects the fix mode (0=no-fix, 1=2D, 2=3D). | — |
| **ACCEL** | rd, rs1, rs2 | Read 3-axis accelerometer data. X-axis (milli-g) in rd, Y-axis in rs1, Z-axis in rs2. | — |
| **DEPTH** | rd, rs1, rs2 | Read depth or pressure sensor. Distance (mm) or pressure (Pa) stored in rd. rs1 selects sensor type; rs2 selects averaging filter. | — |
| **CAMCAP** | rd, rs1, rs2 | Capture a camera frame. Camera index rs1 determines which camera; frame is stored into buffer rd. rs2 specifies resolution mode. Blocks until frame is acquired. | — |
| **CAMDET** | rd, rs1, rs2 | Run object detection on image buffer rd. The number of detected objects is stored in rs1; detection metadata is written to a result buffer at address rs2. | — |
| **PWM** | rd, rs1, rs2 | Configure PWM output on pin rs1. Duty cycle (0–255) is specified by rd; frequency in Hz by rs2. | — |
| **GPIO** | rd, rs1, rs2 | Read or write a GPIO pin. Pin number in rs1; direction in rs2 (0=input, 1=output). For reads, the pin value is stored in rd. For writes, rd is driven onto the pin. | — |
| **I2C** | rd, rs1, rs2 | Perform an I2C transaction. Device address in rs1, register offset in rs2. Data to write is taken from rd; on read, received data is stored in rd. | — |
| **SPI** | rd, rs1, rs2 | Perform an SPI transaction. Send the value in rd to device with chip-select rs1. The received value is stored back in rd. rs2 specifies clock speed mode. | — |
| **UART** | rd, rs1, rs2 | Transmit rd bytes from buffer rs1 over the UART serial port. rs2 specifies baud rate configuration. Returns bytes actually sent in rd. | Z |
| **CANBUS** | rd, rs1, rs2 | Transmit a CAN bus frame. Message payload length in rd, CAN identifier in rs1. rs2 specifies priority and bus selection. Returns transmission status in rd. | Z |

### 8.14 Extended Math / Crypto (0x90–0x9F, Format E)

Extended math operations provide common transcendental functions, bit manipulation utilities, pseudorandom number generation, and cryptographic primitives essential for secure multi-agent communication.

| Mnemonic | Operands | Semantics | Flags |
|----------|----------|-----------|-------|
| **ABS** | rd, rs1, - | Compute the absolute value of rs1. Result stored in rd. rs2 is ignored. | Z, S |
| **SIGN** | rd, rs1, - | Extract the sign of rs1: rd = -1 if negative, 0 if zero, +1 if positive. rs2 is ignored. | Z |
| **SQRT** | rd, rs1, - | Compute the integer square root of rs1. The largest integer n such that n² ≤ rs1 is stored in rd. rs2 is ignored. | Z |
| **POW** | rd, rs1, rs2 | Compute rs1 raised to the power rs2. Result truncated to 32-bit integer and stored in rd. Traps on negative base with non-integer exponent. | Z, S |
| **LOG2** | rd, rs1, - | Compute the base-2 logarithm of rs1 (integer floor). Result stored in rd. Traps if rs1 ≤ 0. rs2 is ignored. | Z |
| **CLZ** | rd, rs1, - | Count the number of leading zero bits in rs1 (from bit 31). Result (0–32) stored in rd. rs2 is ignored. | Z |
| **CTZ** | rd, rs1, - | Count the number of trailing zero bits in rs1 (from bit 0). Result (0–32) stored in rd. Undefined if rs1 = 0. rs2 is ignored. | Z |
| **POPCNT** | rd, rs1, - | Count the number of set (1) bits in rs1 (population count). Result (0–32) stored in rd. rs2 is ignored. | Z |
| **CRC32** | rd, rs1, rs2 | Compute CRC-32 checksum over rs2 bytes starting at memory address rs1. The 32-bit checksum is stored in rd. | Z |
| **SHA256** | rd, rs1, rs2 | Compute SHA-256 hash of rs2 bytes at address rs1. The 256-bit digest is written to a 32-byte buffer whose handle is stored in rd. | — |
| **RND** | rd, rs1, rs2 | Generate a pseudorandom integer in the inclusive range [rs1, rs2]. Result stored in rd. Uses the PRNG seeded by SEED. | — |
| **SEED** | rd, rs1, - | Seed the pseudorandom number generator with rs1. rd is set to the previous seed value (or 0 if unset). rs2 is ignored. | — |
| **FMOD** | rd, rs1, rs2 | Compute the floating-point remainder: rd = fmod(f(rs1), f(rs2)). The result has the same sign as f(rs1). | Z |
| **FSQRT** | rd, rs1, - | Compute the IEEE 754 single-precision square root of f(rs1). Result stored in rd. Traps if f(rs1) < 0. rs2 is ignored. | Z |
| **FSIN** | rd, rs1, - | Compute the IEEE 754 single-precision sine of f(rs1) (interpreted as radians). Result stored in rd. rs2 is ignored. | Z |
| **FCOS** | rd, rs1, - | Compute the IEEE 754 single-precision cosine of f(rs1) (interpreted as radians). Result stored in rd. rs2 is ignored. | Z |

### 8.15 Collection / Crypto (0xA0–0xAF, Format D/E/G)

Collection opcodes provide higher-level data structure operations (concatenation, indexing, slicing, sorting, mapping, filtering) and cryptographic primitives (hashing, HMAC, signing, encryption, key generation). Note that `LEN` (0xA0) uses Format D and `SLICE` (0xA4) uses Format G; all others use Format E.

| Mnemonic | Operands | Semantics | Flags |
|----------|----------|-----------|-------|
| **LEN** | rd, imm8 | (Format D) Load the length of the collection at memory address imm8 into rd. Works for strings, lists, and byte arrays. | Z |
| **CONCAT** | rd, rs1, rs2 | Concatenate collection rs1 with collection rs2. A new collection is allocated and its handle stored in rd. Original collections are not modified. | — |
| **AT** | rd, rs1, rs2 | Index into collection rs1 at position rs2 (0-based). The element value is stored in rd. Traps if rs2 is out of bounds. | — |
| **SETAT** | rd, rs1, rs2 | Set element at index rs2 of collection rs1 to value rd. Traps if rs2 is out of bounds. Returns the previous value in rd (if applicable). | — |
| **SLICE** | rd, rs1, imm16 | (Format G) Extract a sub-collection from rs1, from index 0 to imm16 (exclusive). A new collection handle is stored in rd. Traps if imm16 exceeds the collection length. | — |
| **REDUCE** | rd, rs1, rs2 | Reduce (fold) collection rs1 using the binary function at address rs2. The accumulator result is stored in rd. The function takes (accumulator, element) and returns the new accumulator. | — |
| **MAP** | rd, rs1, rs2 | Apply the function at address rs2 to each element of collection rs1. A new collection with the transformed elements is stored in rd. | — |
| **FILTER** | rd, rs1, rs2 | Filter collection rs1 using the predicate function at address rs2. A new collection containing only elements for which the predicate returns non-zero is stored in rd. | — |
| **SORT** | rd, rs1, rs2 | Sort collection rs1 in ascending order using the comparator function at address rs2. The sorted result (new collection) is stored in rd. | — |
| **FIND** | rd, rs1, rs2 | Search collection rs1 for element rs2. If found, the 0-based index is stored in rd; otherwise rd = -1. | Z |
| **HASH** | rd, rs1, rs2 | Compute a cryptographic hash of the data at memory address rs1 (rs2 bytes). Algorithm is selected by the lower bits of rs2 (0=SHA-256). The digest handle is stored in rd. | — |
| **HMAC** | rd, rs1, rs2 | Compute an HMAC over the data at memory address rs1 using the key at memory address rs2. The HMAC digest handle is stored in rd. | — |
| **VERIFY** | rd, rs1, rs2 | Verify a digital signature. The signature is at memory address rs2; the signed data is at address rs1. Returns 1 in rd if valid, 0 if invalid. | Z |
| **ENCRYPT** | rd, rs1, rs2 | Encrypt the data at memory address rs1 using the key at address rs2. The ciphertext handle is stored in rd. Uses AES-256-GCM by default. | — |
| **DECRYPT** | rd, rs1, rs2 | Decrypt the ciphertext at memory address rs1 using the key at address rs2. The plaintext handle is stored in rd. Traps if authentication fails (for AEAD modes). | — |
| **KEYGEN** | rd, rs1, rs2 | Generate a cryptographic key pair. The public key handle is stored in rs1; the private key handle in rs2. The key type identifier is stored in rd. | — |

---

## 9. Confidence-Aware Variants (0x60–0x6F)

Confidence-aware opcodes perform the same computation as their base counterparts **and** propagate confidence metadata through a parallel register file. Confidence-OPTIONAL means that programs can use base opcodes (ADD, SUB, etc.) without any confidence overhead; C_* variants are used only when confidence tracking is desired.

### Confidence Propagation Rules

| Operation | Confidence Rule |
|-----------|----------------|
| ADD/SUB | crd = min(crs1, crs2) — confidence is the lower bound |
| MUL | crd = crs1 * crs2 — joint probability |
| DIV | crd = crs1 * crs2 * (1 - epsilon) — division introduces uncertainty |
| MERGE | crd = weighted_average(crs1, crs2) |
| THRESHOLD | Skip next instruction if crd < imm8/255 |
| BOOST | crd = min(crd + rs2, 1.0) — increase confidence, capped at 1.0 |
| DECAY | crd = crd * rs2 — decrease confidence each cycle |
| SOURCE | Set confidence source metadata (sensor/model/human) |
| CALIBRATE | Adjust confidence against ground truth |
| EXPLAIN | Apply confidence weight to control flow branching |
| VOTE | crd = sum(crs_i * weight_i) / sum(weight_i) — weighted consensus |

| Hex | Mnemonic | Operands | Description |
|-----|----------|----------|-------------|
| 0x60 | **C_ADD** | rd, rs1, rs2 | rd = rs1+rs2; crd = min(crs1, crs2) |
| 0x61 | **C_SUB** | rd, rs1, rs2 | rd = rs1-rs2; crd = min(crs1, crs2) |
| 0x62 | **C_MUL** | rd, rs1, rs2 | rd = rs1*rs2; crd = crs1 * crs2 |
| 0x63 | **C_DIV** | rd, rs1, rs2 | rd = rs1/rs2; crd = crs1*crs2*(1-epsilon) |
| 0x64 | **C_FADD** | rd, rs1, rs2 | Float add + confidence propagation |
| 0x65 | **C_FSUB** | rd, rs1, rs2 | Float sub + confidence propagation |
| 0x66 | **C_FMUL** | rd, rs1, rs2 | Float mul + confidence propagation |
| 0x67 | **C_FDIV** | rd, rs1, rs2 | Float div + confidence propagation |
| 0x68 | **C_MERGE** | rd, rs1, rs2 | Merge confidences: crd = weighted average |
| 0x69 | **C_THRESH** | rd, imm8 | Skip next instruction if crd < imm8/255 (Format D) |
| 0x6A | **C_BOOST** | rd, rs1, rs2 | Boost crd by rs2 factor (max 1.0) |
| 0x6B | **C_DECAY** | rd, rs1, rs2 | Decay crd by factor rs2 per cycle |
| 0x6C | **C_SOURCE** | rd, rs1, rs2 | Set confidence source metadata (0=sensor, 1=model, 2=human) |
| 0x6D | **C_CALIB** | rd, rs1, rs2 | Calibrate confidence against ground truth |
| 0x6E | **C_EXPLY** | rd, rs1, rs2 | Apply confidence to control flow weight |
| 0x6F | **C_VOTE** | rd, rs1, rs2 | Weighted vote: crd = sum(crs*crs_i) / sum(crs) |

---

## 10. Agent-to-Agent Primitives (0x50–0x5F)

The A2A opcodes implement inter-agent communication within the FLUX fleet. They operate on the VM's registered A2A handler callback.

| Hex | Mnemonic | Operands | Description |
|-----|----------|----------|-------------|
| 0x50 | **TELL** | rd, rs1, rs2 | Send message rs2 to agent rs1, with tag rd. One-way communication. |
| 0x51 | **ASK** | rd, rs1, rs2 | Request data rs2 from agent rs1. Response stored in rd. Blocking. |
| 0x52 | **DELEG** | rd, rs1, rs2 | Delegate task rs2 to agent rs1. Returns task ID in rd. |
| 0x53 | **BCAST** | rd, rs1, rs2 | Broadcast message rs2 to entire fleet. Tag in rd. |
| 0x54 | **ACCEPT** | rd, rs1, rs2 | Accept a delegated task. Context loaded into rd. |
| 0x55 | **DECLINE** | rd, rs1, rs2 | Decline a delegated task with reason rs2. |
| 0x56 | **REPORT** | rd, rs1, rs2 | Report task status rs2 to supervisor rd. |
| 0x57 | **MERGE** | rd, rs1, rs2 | Merge results from agents rs1 and rs2 into rd. |
| 0x58 | **FORK** | rd, rs1, rs2 | Spawn child agent. Initial state in rs2. Child ID in rd. |
| 0x59 | **JOIN** | rd, rs1, rs2 | Wait for child agent rs1 to complete. Result in rd. |
| 0x5A | **SIGNAL** | rd, rs1, rs2 | Emit named signal rs2 on channel rd. Non-blocking. |
| 0x5B | **AWAIT** | rd, rs1, rs2 | Wait for signal rs2. Signal data stored in rd. Blocking. |
| 0x5C | **TRUST** | rd, rs1, rs2 | Set trust level rs2 (0.0-1.0) for agent rs1. |
| 0x5D | **DISCOV** | rd, rs1, rs2 | Discover available fleet agents. Agent list in rd. |
| 0x5E | **STATUS** | rd, rs1, rs2 | Query agent rs1 for status. Result in rd. |
| 0x5F | **HEARTBT** | rd, rs1, rs2 | Emit heartbeat with load metric. Response in rd. |

---

## 11. Conformance Requirements

A FLUX implementation is conformant if it meets all of the following:

### Must-Have (Level 1 — Core)

- [ ] All Format A, B, C, D opcodes implemented (0x00–0x1F)
- [ ] All integer arithmetic opcodes (0x20–0x2F)
- [ ] LOAD, STORE, MOV (0x38–0x3A)
- [ ] JZ, JNZ, JGT, JLT (0x3C–0x3F)
- [ ] MOVI16, JMP, JAL (0x40–0x44)
- [ ] PUSH, POP, RET (0x0C, 0x0D, 0x02)
- [ ] HALT, NOP (0x00, 0x01)
- [ ] Correct flag behavior for all arithmetic ops
- [ ] Correct three-operand read-before-write semantics
- [ ] Correct little-endian encoding/decoding for all formats
- [ ] Stack grows downward, 32-bit values

### Should-Have (Level 2 — Standard)

- [ ] Float arithmetic (0x30–0x37)
- [ ] FTOI, ITOF (0x36–0x37)
- [ ] Format F and G opcodes (0x40–0x4F)
- [ ] ENTER/LEAVE frame management (0x4C–0x4D)
- [ ] Named memory regions with ownership
- [ ] A2A handler callback mechanism (0x50–0x5F)

### Nice-to-Have (Level 3 — Extended)

- [ ] Confidence-aware variants (0x60–0x6F)
- [ ] Viewpoint operations (0x70–0x7F)
- [ ] Sensor operations (0x80–0x8F)
- [ ] Extended math/crypto (0x90–0x9F)
- [ ] Collection ops (0xA0–0xAF)
- [ ] Vector/SIMD (0xB0–0xBF)
- [ ] Tensor/neural (0xC0–0xCF)
- [ ] DMA/MMIO/GPU ops (0xD0–0xDF)

### Test Vectors

Conformance test vectors are maintained in [flux-conformance](https://github.com/SuperInstance/flux-conformance). All Level 1 tests must pass for an implementation to claim FLUX compatibility.

---

## Appendix A: Source Attribution

| Opcode Range | Primary Contributor | Description |
|-------------|-------------------|-------------|
| 0x00–0x3F | Converged | Core ISA — system, arithmetic, memory, control |
| 0x40–0x4F | Converged + JC1 | Large immediates, jumps, frame management |
| 0x50–0x5F | Converged | A2A fleet communication primitives |
| 0x60–0x6F | Converged + JC1 + Oracle1 | Confidence propagation system |
| 0x70–0x7F | Babel | Viewpoint/linguistic operations (16 ops) |
| 0x80–0x8F | JetsonClaw1 | Sensor/hardware I/O operations (16 ops) |
| 0x90–0x9F | Oracle1 + JC1 + Converged | Extended math, crypto, float |
| 0xA0–0xAF | Oracle1 | String/collection operations |
| 0xB0–0xBF | JetsonClaw1 | Vector/SIMD operations |
| 0xC0–0xCF | JetsonClaw1 + Oracle1 | Tensor/neural operations |
| 0xD0–0xDF | JetsonClaw1 + Oracle1 | Extended memory, DMA, GPU |
| 0xE0–0xEF | Converged + JC1 + Oracle1 | Long jumps, coroutines, debug |
| 0xF0–0xFF | Converged + JetsonClaw1 | Extended system control |

## Appendix B: Opcode Quick Reference Card

See [OPCODES.md](OPCODES.md) for the complete machine-readable opcode table generated from `isa_unified.py`.

---

*This specification is maintained in [flux-spec](https://github.com/SuperInstance/flux-spec). All FLUX implementations must conform to this document.*
