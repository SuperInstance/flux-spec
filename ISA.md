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
| Defined opcodes | ~240 |
| Reserved slots | ~16 |
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

The FLUX register file contains 48 registers organized into three banks.

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
