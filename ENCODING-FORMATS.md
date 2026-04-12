# FLUX VM — Instruction Encoding Format Specification

**Version:** 1.0.0
**Status:** Normative
**Canonical Source:** `isa_unified.py` (see [Cross-Reference](#8-cross-reference-to-isa_unifiedpy))

---

## 1. Overview

### 1.1 Purpose

This document defines the byte-level encoding of every FLUX VM instruction. It is the single authoritative reference for assemblers, disassemblers, JIT compilers, interpreters, and conformance test suites. Any divergence from this specification between runtimes is a bug.

### 1.2 Design Principles

| Principle | Rule |
|-----------|------|
| **Byte-aligned** | Every instruction starts at a byte boundary. No sub-byte packing or bit-interleaving. |
| **Little-endian** | All multi-byte numeric fields (imm16, u16 length) store the least-significant byte at the lowest address. |
| **6-bit register encoding** | Each register operand occupies a full byte; bits [5:0] hold the register index (0–63), bits [7:6] are reserved and **must** be zero. |
| **Format-from-opcode** | There are no explicit format bits in the opcode byte. The instruction format is determined solely by the opcode value (see [dispatch table, §3.8](#38-opcode-range--format-dispatch)). |
| **Variable-length instructions** | Instruction sizes range from 1 byte (Format A) to 5+ bytes (Format G with payload). |
| **Read-before-write** | For Format E three-register instructions, the VM reads `rs1` and `rs2` before writing `rd`. `ADD R3, R3, R5` is well-defined. |

### 1.3 Format Summary

| Format | Fixed Size | Variable? | Layout (byte offsets) | Primary Use |
|--------|-----------|-----------|-----------------------|-------------|
| **A** | 1 byte | No | `[+0: opcode]` | System control, halt, debug |
| **B** | 2 bytes | No | `[+0: opcode][+1: rd]` | Unary register operations |
| **C** | 2 bytes | No | `[+0: opcode][+1: imm8]` | Immediate-only system/debug ops |
| **D** | 3 bytes | No | `[+0: opcode][+1: rd][+2: imm8]` | Register + short immediate |
| **E** | 4 bytes | No | `[+0: opcode][+1: rd][+2: rs1][+3: rs2]` | Three-register arithmetic, logic, A2A, etc. |
| **F** | 4 bytes | No | `[+0: opcode][+1: rd][+2: imm16_lo][+3: imm16_hi]` | Register + 16-bit immediate |
| **G** | 5 bytes | Yes (payload) | `[+0: opcode][+1: rd][+2: rs1][+3: u16_lo][+4: u16_hi][+5 … +N: payload]` | Two-register + displacement, variable-length data |

---

## 2. Register Encoding Rules

### 2.1 Register Byte Format

Every register operand occupies exactly one byte with the following bit layout:

```
 7   6   5   4   3   2   1   0
┌───┬───┬───────────────────────┐
│   0   │       reg[5:0]        │
│ rsvd  │    Register Index     │
└───┴───┴───────────────────────┘
 ^^^^^   ^^^^^^^^^^^^^^^^^^^^
 bits    bits
 [7:6]   [5:0]
```

- **Bits [7:6] (reserved):** MUST be zero. A runtime MUST reject (trap on) any instruction where these bits are non-zero.
- **Bits [5:0] (register index):** Unsigned integer 0–63, selecting one of 64 logical registers.

### 2.2 Register Map

> **Note — ISA Reconciliation:** The ISA specification ([ISA.md §3](ISA.md)) describes the register file as a three-bank logical model (GP R0–R15, Float F0–F15, Vector V0–V15 = 48 registers). This encoding section defines the canonical unified 64-register space (R0–R63). The three-bank model is a logical/ABI convenience view; the encoding uses R0–R63 directly. Implementations must use this 64-register unified encoding for instruction decode.

| Register Range | Count | Category | ABI Convention |
|----------------|-------|----------|----------------|
| R0–R7 | 8 | General Purpose (GP) | Arguments, return values, scratch |
| R8–R15 | 8 | Frame / Callee-Saved (FP) | Callee-saved temporaries, frame management |
| R16–R31 | 16 | Temporary (Temp) | Caller-saved intermediates, compiler temporaries |
| R32–R47 | 16 | Floating-Point (f32) | IEEE 754 single-precision, accessed by float opcodes |
| R48–R63 | 16 | Special / Vector / Reserved | SIMD vectors, system registers, future expansion |

### 2.3 Notable Fixed-Function Registers

| Register | ABI Name | Role | Notes |
|----------|----------|------|-------|
| R0 | `ZERO` | Hardwired constant zero | Writes are silently discarded |
| R11 | `SP` | Stack pointer | Stack grows downward |
| R12 | `RID` | Region ID | Implicit memory-region selector |
| R13 | `TKN` | Trust token | A2A trust metadata |
| R14 | `FP` | Frame pointer | Callee-saved |
| R15 | `LR` | Link register | Return address for `CALL`/`JAL` |

### 2.4 Register Encoding Examples

| Register | Decimal Index | Binary `[7:0]` | Hex Byte |
|----------|--------------|-----------------|----------|
| R0 | 0 | `00 000000` | `0x00` |
| R1 | 1 | `00 000001` | `0x01` |
| R5 | 5 | `00 000101` | `0x05` |
| R10 | 10 | `00 001010` | `0x0A` |
| R15 | 15 | `00 001111` | `0x0F` |
| R31 | 31 | `00 011111` | `0x1F` |
| R63 | 63 | `00 111111` | `0x3F` |

---

## 3. Bit-Level Format Diagrams

### 3.1 Format A — Opcode Only (1 byte)

System control and debug instructions with no operands.

```
 Offset 0:
 7   6   5   4   3   2   1   0
┌───────────────────────────────┐
│         opcode[7:0]           │
└───────────────────────────────┘
```

**Opcode ranges:** `0x00–0x07`, `0xF0–0xFF`
**Total size:** 1 byte

### 3.2 Format B — Single Register (2 bytes)

Unary register operations (increment, decrement, push, pop, bitwise NOT, etc.).

```
 Offset 0:                          Offset 1:
 7   6   5   4   3   2   1   0     7   6   5   4   3   2   1   0
┌───────────────────────────────┐   ┌───┬───┬───────────────────────┐
│         opcode[7:0]           │   │ 0   0 │        rd[5:0]        │
└───────────────────────────────┘   └───┴───┴───────────────────────┘
```

**Opcode range:** `0x08–0x0F`
**Total size:** 2 bytes

### 3.3 Format C — Immediate Byte (2 bytes)

System calls, traps, and debug operations that take a single 8-bit immediate operand.

```
 Offset 0:                          Offset 1:
 7   6   5   4   3   2   1   0     7   6   5   4   3   2   1   0
┌───────────────────────────────┐   ┌───────────────────────────────┐
│         opcode[7:0]           │   │         imm8[7:0]             │
└───────────────────────────────┘   └───────────────────────────────┘
```

**Opcode range:** `0x10–0x17`
**Total size:** 2 bytes

### 3.4 Format D — Register + Imm8 (3 bytes)

Register operations with an 8-bit immediate (load immediate, short arithmetic with constants, shift by immediate).

```
 Offset 0:                          Offset 1:                          Offset 2:
 7   6   5   4   3   2   1   0     7   6   5   4   3   2   1   0     7   6   5   4   3   2   1   0
┌───────────────────────────────┐   ┌───┬───┬───────────────────────┐   ┌───────────────────────────────┐
│         opcode[7:0]           │   │ 0   0 │        rd[5:0]        │   │         imm8[7:0]            │
└───────────────────────────────┘   └───┴───┴───────────────────────┘   └───────────────────────────────┘
```

**Opcode range:** `0x18–0x1F` (also `0x69`, `0xA0`)
**Total size:** 3 bytes
**Sign extension:** `imm8` is sign-extended to 32 bits for arithmetic operations (MOVI, ADDI, SUBI). For logical/shift operations (ANDI, ORI, XORI, SHLI, SHRI), `imm8` is zero-extended.

### 3.5 Format E — Three Register (4 bytes)

The workhorse format. All three register fields use the same encoding (§2.1). For unary or two-operand instructions, unused register bytes are consumed by the encoding but semantically ignored by the VM.

```
 Offset 0:      Offset 1:      Offset 2:      Offset 3:
 ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
 │  opcode   │  │    rd     │  │    rs1    │  │    rs2    │
 │  [7:0]    │  │   [5:0]   │  │   [5:0]   │  │   [5:0]   │
 └───────────┘  └───────────┘  └───────────┘  └───────────┘
   1 byte        1 byte        1 byte        1 byte
```

**Opcode ranges:** `0x20–0x3F`, `0x50–0x5F`, `0x60–0x6F`, `0x70–0x7F`, `0x80–0x8F`, `0x90–0x9F`, `0xA1–0xA3`, `0xA5–0xAF`, `0xB0–0xBF`, `0xC0–0xCF`
**Total size:** 4 bytes

**Semantic notes:**
- **Three-operand ops** (ADD, SUB, MUL, etc.): All three fields active. `rd = rs1 op rs2`.
- **Unary ops** (FTOI, ITOF, MOV, SWP, ABS, SIGN, etc.): `rd` and `rs1` active; `rs2` byte consumed but **ignored**.
- **Conditional branches** (JZ, JNZ, JLT, JGT): `rd` is the condition register; `rs1` is the signed offset; `rs2` byte consumed but **ignored**.

### 3.6 Format F — Register + Imm16 (4 bytes)

One register and one 16-bit immediate. The immediate is stored in **little-endian** order: low byte at offset +2, high byte at offset +3.

```
 Offset 0:      Offset 1:      Offset 2:          Offset 3:
 ┌───────────┐  ┌───────────┐  ┌───────────────┐  ┌───────────────┐
 │  opcode   │  │    rd     │  │  imm16[7:0]   │  │  imm16[15:8]  │
 │  [7:0]    │  │   [5:0]   │  │  (low byte)   │  │  (high byte)  │
 └───────────┘  └───────────┘  └───────────────┘  └───────────────┘
   1 byte        1 byte         1 byte              1 byte
```

**Opcode ranges:** `0x40–0x47`, `0xE0–0xEF`
**Total size:** 4 bytes

**Signedness rules:**
- **Unsigned** (`MOVI16`, `ADDI16`, `SUBI16`, `TRACE`, `PROF_ON`, `PROF_OFF`, `WATCH`): `imm16` is zero-extended to 32 bits.
- **Signed** (`JMP`, `JAL`, `CALL`, `JMPL`, `JALL`, `CALLL`, `TAIL`, `LOOP`, `SELECT`, `SWITCH`, `COYIELD`, `FAULT`, `HANDLER`): `imm16` is sign-extended to 32 bits.

### 3.7 Format G — Two Register + Imm16 + Optional Payload (5+ bytes)

Two registers and a 16-bit immediate/displacement. May be followed by a variable-length payload (see §5 for detailed variable-length rules).

```
 Offset 0:      Offset 1:      Offset 2:      Offset 3:          Offset 4:
 ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────────┐  ┌───────────────┐
 │  opcode   │  │    rd     │  │    rs1    │  │  u16[7:0]     │  │  u16[15:8]    │
 │  [7:0]    │  │   [5:0]   │  │   [5:0]   │  │  (low byte)   │  │  (high byte)  │
 └───────────┘  └───────────┘  └───────────┘  └───────────────┘  └───────────────┘
   1 byte        1 byte        1 byte         1 byte              1 byte

 Offset 5 … N (optional, only when opcode requires variable-length data):
 ┌─────────────────────────────────────────────────────────────┐
 │                      payload[0 … N-5]                      │
 │              (raw bytes, N-5 = payload length)              │
 └─────────────────────────────────────────────────────────────┘
```

**Opcode ranges:** `0x48–0x4F`, `0xA4`, `0xD0–0xDF`
**Fixed header size:** 5 bytes
**Maximum total size:** 5 + 65535 = **65540 bytes**

### 3.8 Opcode Range → Format Dispatch

The instruction format is determined solely by the opcode byte. The following table is the definitive dispatch map:

| Opcode Range | Format | Size (fixed) | Category |
|-------------|--------|-------------|----------|
| `0x00–0x07` | **A** | 1 byte | System control / interrupt / debug |
| `0x08–0x0F` | **B** | 2 bytes | Single-register operations |
| `0x10–0x17` | **C** | 2 bytes | Immediate-only system / debug |
| `0x18–0x1F` | **D** | 3 bytes | Register + imm8 |
| `0x20–0x3F` | **E** | 4 bytes | Integer arithmetic, float, memory, control |
| `0x40–0x47` | **F** | 4 bytes | Register + imm16 (short jumps, loads) |
| `0x48–0x4F` | **G** | 5 bytes | Reg + reg + imm16 (memory offset, frame) |
| `0x50–0x5F` | **E** | 4 bytes | Agent-to-Agent (A2A) fleet primitives |
| `0x60–0x6F` | **E** | 4 bytes | Confidence-aware variants (`C_*`) |
| `0x69` | **D** | 3 bytes | `C_THRESH` (confidence threshold) |
| `0x70–0x7F` | **E** | 4 bytes | Viewpoint ops (Babel) |
| `0x80–0x8F` | **E** | 4 bytes | Sensor / hardware I/O (JetsonClaw1) |
| `0x90–0x9F` | **E** | 4 bytes | Extended math / crypto |
| `0xA0` | **D** | 3 bytes | `LEN` (collection length) |
| `0xA1–0xA3` | **E** | 4 bytes | Collection ops (concat, at, setat) |
| `0xA4` | **G** | 5 bytes | `SLICE` (collection slice) |
| `0xA5–0xAF` | **E** | 4 bytes | Collection + crypto ops |
| `0xB0–0xBF` | **E** | 4 bytes | Vector / SIMD |
| `0xC0–0xCF` | **E** | 4 bytes | Tensor / neural |
| `0xD0–0xDF` | **G** | 5 bytes | Extended memory / MMIO / GPU |
| `0xE0–0xEF` | **F** | 4 bytes | Long jumps, coroutines, debug |
| `0xF0–0xFF` | **A** | 1 byte | Extended system control / illegal |

> **Note:** `0x69` (`C_THRESH`) and `0xA0` (`LEN`) are the only opcodes in their respective ranges that use Format D instead of Format E. Decoders MUST treat these as special cases within the `0x60–0x6F` and `0xA0–0xAF` ranges.

---

## 4. Complete Opcode Tables by Format

### 4.1 Format A — Opcode Only (1 byte)

#### System Control (`0x00–0x03`)

| Hex | Mnemonic | Category | Description |
|-----|----------|----------|-------------|
| `0x00` | `HALT` | system | Stop execution. Set `halted = True`. |
| `0x01` | `NOP` | system | No operation. Pipeline synchronization point. |
| `0x02` | `RET` | system | Return from subroutine. Pop return address from stack. |
| `0x03` | `IRET` | system | Return from interrupt handler. Restore saved state. |

#### Interrupt / Debug (`0x04–0x07`)

| Hex | Mnemonic | Category | Description |
|-----|----------|----------|-------------|
| `0x04` | `BRK` | debug | Breakpoint. Trap to debugger. |
| `0x05` | `WFI` | system | Wait for interrupt. Enter low-power idle. |
| `0x06` | `RESET` | system | Soft reset. Clear register file to zero. |
| `0x07` | `SYN` | system | Memory barrier / synchronize. |

#### Extended System / Debug (`0xF0–0xFF`)

| Hex | Mnemonic | Category | Description |
|-----|----------|----------|-------------|
| `0xF0` | `HALT_ERR` | system | Halt with error (check flags). |
| `0xF1` | `REBOOT` | system | Warm reboot (preserve memory). |
| `0xF2` | `DUMP` | debug | Dump register file to debug output. |
| `0xF3` | `ASSERT` | debug | Assert flags; halt if violation. |
| `0xF4` | `ID` | system | Return agent ID to R0. |
| `0xF5` | `VER` | system | Return ISA version to R0. |
| `0xF6` | `CLK` | system | Return clock cycle count to R0. |
| `0xF7` | `PCLK` | system | Return performance counter to R0. |
| `0xF8` | `WDOG` | system | Kick watchdog timer. |
| `0xF9` | `SLEEP` | system | Enter low-power sleep (wake on interrupt). |
| `0xFA` | _RESERVED_ | reserved | Reserved. |
| `0xFB` | _RESERVED_ | reserved | Reserved. |
| `0xFC` | _RESERVED_ | reserved | Reserved. |
| `0xFD` | _RESERVED_ | reserved | Reserved. |
| `0xFE` | _RESERVED_ | reserved | Reserved. |
| `0xFF` | `ILLEGAL` | system | Illegal instruction trap. |

**Format A total: 19 defined + 5 reserved = 24 opcodes**

---

### 4.2 Format B — Single Register (2 bytes)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0x08` | `INC` | `rd` | arithmetic | `rd = rd + 1` |
| `0x09` | `DEC` | `rd` | arithmetic | `rd = rd - 1` |
| `0x0A` | `NOT` | `rd` | logic | `rd = ~rd` (bitwise NOT) |
| `0x0B` | `NEG` | `rd` | arithmetic | `rd = -rd` (arithmetic negate) |
| `0x0C` | `PUSH` | `rd` | stack | Push rd onto stack. SP -= 4. |
| `0x0D` | `POP` | `rd` | stack | Pop stack into rd. SP += 4. |
| `0x0E` | `CONF_LD` | `rd` | confidence | Load confidence for rd into accumulator. |
| `0x0F` | `CONF_ST` | `rd` | confidence | Store confidence accumulator to rd. |

**Format B total: 8 opcodes**

---

### 4.3 Format C — Immediate Byte (2 bytes)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0x10` | `SYS` | `imm8` | system | System call with code `imm8`. |
| `0x11` | `TRAP` | `imm8` | system | Software interrupt, dispatch to vector `imm8`. |
| `0x12` | `DBG` | `imm8` | debug | Debug print register `imm8`. |
| `0x13` | `CLF` | `imm8` | system | Clear condition flag bits `imm8`. |
| `0x14` | `SEMA` | `imm8` | concurrency | Semaphore operation (wait/signal by `imm8`). |
| `0x15` | `YIELD` | `imm8` | concurrency | Yield execution for `imm8` cycles. |
| `0x16` | `CACHE` | `imm8` | system | Cache control: flush(0), invalidate(1), both(2). |
| `0x17` | `STRIPCF` | `imm8` | confidence | Strip confidence from next `imm8` ops. |

**Format C total: 8 opcodes**

---

### 4.4 Format D — Register + Imm8 (3 bytes)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0x18` | `MOVI` | `rd, imm8` | move | `rd = sign_extend(imm8)` |
| `0x19` | `ADDI` | `rd, imm8` | arithmetic | `rd = rd + sign_extend(imm8)` |
| `0x1A` | `SUBI` | `rd, imm8` | arithmetic | `rd = rd - sign_extend(imm8)` |
| `0x1B` | `ANDI` | `rd, imm8` | logic | `rd = rd & zero_extend(imm8)` |
| `0x1C` | `ORI` | `rd, imm8` | logic | `rd = rd \| zero_extend(imm8)` |
| `0x1D` | `XORI` | `rd, imm8` | logic | `rd = rd ^ zero_extend(imm8)` |
| `0x1E` | `SHLI` | `rd, imm8` | shift | `rd = rd << imm8` (logical left shift) |
| `0x1F` | `SHRI` | `rd, imm8` | shift | `rd = rd >> imm8` (arithmetic right shift) |
| `0x69` | `C_THRESH` | `rd, imm8` | confidence | Skip next instruction if `crd < imm8/255` |
| `0xA0` | `LEN` | `rd, imm8` | collection | `rd = length of collection at `imm8`` |

**Format D total: 10 opcodes**

---

### 4.5 Format E — Three Register (4 bytes)

#### Integer Arithmetic (`0x20–0x2F`)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0x20` | `ADD` | `rd, rs1, rs2` | arithmetic | `rd = rs1 + rs2` |
| `0x21` | `SUB` | `rd, rs1, rs2` | arithmetic | `rd = rs1 - rs2` |
| `0x22` | `MUL` | `rd, rs1, rs2` | arithmetic | `rd = rs1 * rs2` (32-bit truncated) |
| `0x23` | `DIV` | `rd, rs1, rs2` | arithmetic | `rd = rs1 / rs2` (signed, traps on /0) |
| `0x24` | `MOD` | `rd, rs1, rs2` | arithmetic | `rd = rs1 % rs2` (signed, traps on /0) |
| `0x25` | `AND` | `rd, rs1, rs2` | logic | `rd = rs1 & rs2` |
| `0x26` | `OR` | `rd, rs1, rs2` | logic | `rd = rs1 \| rs2` |
| `0x27` | `XOR` | `rd, rs1, rs2` | logic | `rd = rs1 ^ rs2` |
| `0x28` | `SHL` | `rd, rs1, rs2` | shift | `rd = rs1 << (rs2 & 0x3F)` |
| `0x29` | `SHR` | `rd, rs1, rs2` | shift | `rd = rs1 >> (rs2 & 0x3F)` (arithmetic) |
| `0x2A` | `MIN` | `rd, rs1, rs2` | arithmetic | `rd = min(rs1, rs2)` |
| `0x2B` | `MAX` | `rd, rs1, rs2` | arithmetic | `rd = max(rs1, rs2)` |
| `0x2C` | `CMP_EQ` | `rd, rs1, rs2` | compare | `rd = (rs1 == rs2) ? 1 : 0` |
| `0x2D` | `CMP_LT` | `rd, rs1, rs2` | compare | `rd = (rs1 < rs2) ? 1 : 0` (signed) |
| `0x2E` | `CMP_GT` | `rd, rs1, rs2` | compare | `rd = (rs1 > rs2) ? 1 : 0` (signed) |
| `0x2F` | `CMP_NE` | `rd, rs1, rs2` | compare | `rd = (rs1 != rs2) ? 1 : 0` |

#### Float / Memory / Control (`0x30–0x3F`)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0x30` | `FADD` | `rd, rs1, rs2` | float | `rd = f(rs1) + f(rs2)` |
| `0x31` | `FSUB` | `rd, rs1, rs2` | float | `rd = f(rs1) - f(rs2)` |
| `0x32` | `FMUL` | `rd, rs1, rs2` | float | `rd = f(rs1) * f(rs2)` |
| `0x33` | `FDIV` | `rd, rs1, rs2` | float | `rd = f(rs1) / f(rs2)` |
| `0x34` | `FMIN` | `rd, rs1, rs2` | float | `rd = fmin(rs1, rs2)` |
| `0x35` | `FMAX` | `rd, rs1, rs2` | float | `rd = fmax(rs1, rs2)` |
| `0x36` | `FTOI` | `rd, rs1, -` | convert | `rd = int(f(rs1))`. rs2 ignored. |
| `0x37` | `ITOF` | `rd, rs1, -` | convert | `rd = float(rs1)`. rs2 ignored. |
| `0x38` | `LOAD` | `rd, rs1, rs2` | memory | `rd = mem[rs1 + rs2]` |
| `0x39` | `STORE` | `rd, rs1, rs2` | memory | `mem[rs1 + rs2] = rd` |
| `0x3A` | `MOV` | `rd, rs1, -` | move | `rd = rs1`. rs2 ignored. |
| `0x3B` | `SWP` | `rd, rs1, -` | move | `swap(rd, rs1)`. rs2 ignored. |
| `0x3C` | `JZ` | `rd, rs1, -` | control | If `rd == 0`: `PC += rs1`. rs2 ignored. |
| `0x3D` | `JNZ` | `rd, rs1, -` | control | If `rd != 0`: `PC += rs1`. rs2 ignored. |
| `0x3E` | `JLT` | `rd, rs1, -` | control | If `rd < 0`: `PC += rs1`. rs2 ignored. |
| `0x3F` | `JGT` | `rd, rs1, -` | control | If `rd > 0`: `PC += rs1`. rs2 ignored. |

#### Agent-to-Agent (`0x50–0x5F`)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0x50` | `TELL` | `rd, rs1, rs2` | a2a | Send message `rs2` to agent `rs1`, tag `rd`. |
| `0x51` | `ASK` | `rd, rs1, rs2` | a2a | Request `rs2` from agent `rs1`, response → `rd`. |
| `0x52` | `DELEG` | `rd, rs1, rs2` | a2a | Delegate task `rs2` to agent `rs1`. |
| `0x53` | `BCAST` | `rd, rs1, rs2` | a2a | Broadcast `rs2` to fleet, tag `rd`. |
| `0x54` | `ACCEPT` | `rd, rs1, rs2` | a2a | Accept delegated task, context → `rd`. |
| `0x55` | `DECLINE` | `rd, rs1, rs2` | a2a | Decline task with reason `rs2`. |
| `0x56` | `REPORT` | `rd, rs1, rs2` | a2a | Report status `rs2` to supervisor `rd`. |
| `0x57` | `MERGE` | `rd, rs1, rs2` | a2a | Merge results from `rs1`, `rs2` → `rd`. |
| `0x58` | `FORK` | `rd, rs1, rs2` | a2a | Spawn child agent; state from `rs2`, ID → `rd`. |
| `0x59` | `JOIN` | `rd, rs1, rs2` | a2a | Wait for child `rs1`, result → `rd`. |
| `0x5A` | `SIGNAL` | `rd, rs1, rs2` | a2a | Emit signal `rs2` on channel `rd`. |
| `0x5B` | `AWAIT` | `rd, rs1, rs2` | a2a | Wait for signal `rs2`, data → `rd`. |
| `0x5C` | `TRUST` | `rd, rs1, rs2` | a2a | Set trust level `rs2` for agent `rs1`. |
| `0x5D` | `DISCOV` | `rd, rs1, rs2` | a2a | Discover fleet agents, list → `rd`. |
| `0x5E` | `STATUS` | `rd, rs1, rs2` | a2a | Query agent `rs1` status, result → `rd`. |
| `0x5F` | `HEARTBT` | `rd, rs1, rs2` | a2a | Emit heartbeat, load → `rd`. |

#### Confidence-Aware Variants (`0x60–0x6F`)

> **Note:** `0x69` (`C_THRESH`) is Format D (§4.4), not Format E. All others are Format E.

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0x60` | `C_ADD` | `rd, rs1, rs2` | confidence | `rd = rs1+rs2; crd = min(crs1,crs2)` |
| `0x61` | `C_SUB` | `rd, rs1, rs2` | confidence | `rd = rs1-rs2; crd = min(crs1,crs2)` |
| `0x62` | `C_MUL` | `rd, rs1, rs2` | confidence | `rd = rs1*rs2; crd = crs1*crs2` |
| `0x63` | `C_DIV` | `rd, rs1, rs2` | confidence | `rd = rs1/rs2; crd = crs1*crs2*(1-ε)` |
| `0x64` | `C_FADD` | `rd, rs1, rs2` | confidence | Float add + confidence propagation. |
| `0x65` | `C_FSUB` | `rd, rs1, rs2` | confidence | Float sub + confidence propagation. |
| `0x66` | `C_FMUL` | `rd, rs1, rs2` | confidence | Float mul + confidence propagation. |
| `0x67` | `C_FDIV` | `rd, rs1, rs2` | confidence | Float div + confidence propagation. |
| `0x68` | `C_MERGE` | `rd, rs1, rs2` | confidence | `crd = weighted_avg(crs1, crs2)` |
| `0x6A` | `C_BOOST` | `rd, rs1, rs2` | confidence | `crd = min(crd + rs2, 1.0)` |
| `0x6B` | `C_DECAY` | `rd, rs1, rs2` | confidence | `crd = crd * rs2` per cycle. |
| `0x6C` | `C_SOURCE` | `rd, rs1, rs2` | confidence | Set confidence source: sensor(0)/model(1)/human(2). |
| `0x6D` | `C_CALIB` | `rd, rs1, rs2` | confidence | Calibrate confidence against ground truth. |
| `0x6E` | `C_EXPLY` | `rd, rs1, rs2` | confidence | Apply confidence to control flow weight. |
| `0x6F` | `C_VOTE` | `rd, rs1, rs2` | confidence | Weighted vote consensus. |

#### Viewpoint Operations — Babel (`0x70–0x7F`)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0x70` | `V_EVID` | `rd, rs1, rs2` | viewpoint | Evidentiality: source type `rs2` → `rd`. |
| `0x71` | `V_EPIST` | `rd, rs1, rs2` | viewpoint | Epistemic stance: certainty level. |
| `0x72` | `V_MIR` | `rd, rs1, rs2` | viewpoint | Mirative: unexpectedness marker. |
| `0x73` | `V_NEG` | `rd, rs1, rs2` | viewpoint | Negation scope: predicate vs proposition. |
| `0x74` | `V_TENSE` | `rd, rs1, rs2` | viewpoint | Temporal viewpoint alignment. |
| `0x75` | `V_ASPEC` | `rd, rs1, rs2` | viewpoint | Aspectual viewpoint: complete/ongoing. |
| `0x76` | `V_MODAL` | `rd, rs1, rs2` | viewpoint | Modal force: necessity/possibility. |
| `0x77` | `V_POLIT` | `rd, rs1, rs2` | viewpoint | Politeness register mapping. |
| `0x78` | `V_HONOR` | `rd, rs1, rs2` | viewpoint | Honorific level → trust tier. |
| `0x79` | `V_TOPIC` | `rd, rs1, rs2` | viewpoint | Topic-comment structure binding. |
| `0x7A` | `V_FOCUS` | `rd, rs1, rs2` | viewpoint | Information focus marking. |
| `0x7B` | `V_CASE` | `rd, rs1, rs2` | viewpoint | Case-based scope assignment. |
| `0x7C` | `V_AGREE` | `rd, rs1, rs2` | viewpoint | Agreement (gender/number/person). |
| `0x7D` | `V_CLASS` | `rd, rs1, rs2` | viewpoint | Classifier → type mapping. |
| `0x7E` | `V_INFL` | `rd, rs1, rs2` | viewpoint | Inflection → control flow mapping. |
| `0x7F` | `V_PRAGMA` | `rd, rs1, rs2` | viewpoint | Pragmatic context switch. |

#### Sensor / Hardware I/O — JetsonClaw1 (`0x80–0x8F`)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0x80` | `SENSE` | `rd, rs1, rs2` | sensor | Read sensor `rs1`, channel `rs2` → `rd`. |
| `0x81` | `ACTUATE` | `rd, rs1, rs2` | sensor | Write `rd` to actuator `rs1`, channel `rs2`. |
| `0x82` | `SAMPLE` | `rd, rs1, rs2` | sensor | ADC sample channel `rs1`, average `rs2` → `rd`. |
| `0x83` | `ENERGY` | `rd, rs1, rs2` | sensor | Available → `rd`, used → `rs1`. |
| `0x84` | `TEMP` | `rd, rs1, rs2` | sensor | Temperature sensor read → `rd`. |
| `0x85` | `GPS` | `rd, rs1, rs2` | sensor | GPS coordinates → `rd`, `rs1`. |
| `0x86` | `ACCEL` | `rd, rs1, rs2` | sensor | Accelerometer (3-axis) → `rd`, `rs1`, `rs2`. |
| `0x87` | `DEPTH` | `rd, rs1, rs2` | sensor | Depth/pressure sensor → `rd`. |
| `0x88` | `CAMCAP` | `rd, rs1, rs2` | sensor | Capture camera frame `rs1` → buffer `rd`. |
| `0x89` | `CAMDET` | `rd, rs1, rs2` | sensor | Detection on buffer `rd`, N → `rs1`. |
| `0x8A` | `PWM` | `rd, rs1, rs2` | sensor | PWM: pin `rs1`, duty `rd`, freq `rs2`. |
| `0x8B` | `GPIO` | `rd, rs1, rs2` | sensor | GPIO: pin `rs1`, direction `rs2`. |
| `0x8C` | `I2C` | `rd, rs1, rs2` | sensor | I2C: addr `rs1`, reg `rs2`, data `rd`. |
| `0x8D` | `SPI` | `rd, rs1, rs2` | sensor | SPI: send `rd`, receive → `rd`, cs=`rs1`. |
| `0x8E` | `UART` | `rd, rs1, rs2` | sensor | UART: send `rd` bytes from buf `rs1`. |
| `0x8F` | `CANBUS` | `rd, rs1, rs2` | sensor | CAN bus: send `rd` with ID `rs1`. |

#### Extended Math / Crypto (`0x90–0x9F`)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0x90` | `ABS` | `rd, rs1, -` | math | `rd = abs(rs1)`. rs2 ignored. |
| `0x91` | `SIGN` | `rd, rs1, -` | math | `rd = sign(rs1)`. rs2 ignored. |
| `0x92` | `SQRT` | `rd, rs1, -` | math | `rd = sqrt(rs1)`. rs2 ignored. |
| `0x93` | `POW` | `rd, rs1, rs2` | math | `rd = rs1 ^ rs2`. |
| `0x94` | `LOG2` | `rd, rs1, -` | math | `rd = log2(rs1)`. rs2 ignored. |
| `0x95` | `CLZ` | `rd, rs1, -` | math | `rd = count_leading_zeros(rs1)`. |
| `0x96` | `CTZ` | `rd, rs1, -` | math | `rd = count_trailing_zeros(rs1)`. |
| `0x97` | `POPCNT` | `rd, rs1, -` | math | `rd = popcount(rs1)`. |
| `0x98` | `CRC32` | `rd, rs1, rs2` | crypto | `rd = crc32(rs1, rs2)`. |
| `0x99` | `SHA256` | `rd, rs1, rs2` | crypto | SHA-256: msg `rs1`, len `rs2` → `rd`. |
| `0x9A` | `RND` | `rd, rs1, rs2` | math | `rd = random in [rs1, rs2]`. |
| `0x9B` | `SEED` | `rd, rs1, -` | math | Seed PRNG with `rs1`. |
| `0x9C` | `FMOD` | `rd, rs1, rs2` | float | `rd = fmod(rs1, rs2)`. |
| `0x9D` | `FSQRT` | `rd, rs1, -` | float | `rd = fsqrt(rs1)`. |
| `0x9E` | `FSIN` | `rd, rs1, -` | float | `rd = sin(rs1)`. |
| `0x9F` | `FCOS` | `rd, rs1, -` | float | `rd = cos(rs1)`. |

#### Collection + Crypto (`0xA1–0xAF`)

> **Note:** `0xA0` (`LEN`) is Format D (§4.4), `0xA4` (`SLICE`) is Format G (§4.7).

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0xA1` | `CONCAT` | `rd, rs1, rs2` | collection | `rd = concat(rs1, rs2)`. |
| `0xA2` | `AT` | `rd, rs1, rs2` | collection | `rd = rs1[rs2]`. |
| `0xA3` | `SETAT` | `rd, rs1, rs2` | collection | `rs1[rs2] = rd`. |
| `0xA5` | `REDUCE` | `rd, rs1, rs2` | collection | `rd = fold(rs1, rs2)`. |
| `0xA6` | `MAP` | `rd, rs1, rs2` | collection | `rd = map(rs1, fn rs2)`. |
| `0xA7` | `FILTER` | `rd, rs1, rs2` | collection | `rd = filter(rs1, fn rs2)`. |
| `0xA8` | `SORT` | `rd, rs1, rs2` | collection | `rd = sort(rs1, cmp rs2)`. |
| `0xA9` | `FIND` | `rd, rs1, rs2` | collection | `rd = index of rs2 in rs1` (-1 if absent). |
| `0xAA` | `HASH` | `rd, rs1, rs2` | crypto | `rd = hash(rs1, algorithm rs2)`. |
| `0xAB` | `HMAC` | `rd, rs1, rs2` | crypto | `rd = hmac(rs1, key rs2)`. |
| `0xAC` | `VERIFY` | `rd, rs1, rs2` | crypto | `rd = verify signature rs2 on data rs1`. |
| `0xAD` | `ENCRYPT` | `rd, rs1, rs2` | crypto | `rd = encrypt rs1 with key rs2`. |
| `0xAE` | `DECRYPT` | `rd, rs1, rs2` | crypto | `rd = decrypt rs1 with key rs2`. |
| `0xAF` | `KEYGEN` | `rd, rs1, rs2` | crypto | Keypair: pub → `rs1`, priv → `rs2`. |

#### Vector / SIMD (`0xB0–0xBF`)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0xB0` | `VLOAD` | `rd, rs1, rs2` | vector | Load vector from `mem[rs1]`, length `rs2`. |
| `0xB1` | `VSTORE` | `rd, rs1, rs2` | vector | Store vector `rd` to `mem[rs1]`, length `rs2`. |
| `0xB2` | `VADD` | `rd, rs1, rs2` | vector | `rd[i] = rs1[i] + rs2[i]`. |
| `0xB3` | `VMUL` | `rd, rs1, rs2` | vector | `rd[i] = rs1[i] * rs2[i]`. |
| `0xB4` | `VDOT` | `rd, rs1, rs2` | vector | `rd = Σ rs1[i] * rs2[i]`. |
| `0xB5` | `VNORM` | `rd, rs1, rs2` | vector | `rd = sqrt(Σ rs1[i]²)`. |
| `0xB6` | `VSCALE` | `rd, rs1, rs2` | vector | `rd[i] = rs1[i] * rs2` (scalar). |
| `0xB7` | `VMAXP` | `rd, rs1, rs2` | vector | `rd[i] = max(rs1[i], rs2[i])`. |
| `0xB8` | `VMINP` | `rd, rs1, rs2` | vector | `rd[i] = min(rs1[i], rs2[i])`. |
| `0xB9` | `VREDUCE` | `rd, rs1, rs2` | vector | Reduce vector with op `rs2`. |
| `0xBA` | `VGATHER` | `rd, rs1, rs2` | vector | `rd[i] = mem[rs1[rs2[i]]]`. |
| `0xBB` | `VSCATTER` | `rd, rs1, rs2` | vector | `mem[rs1[rs2[i]]] = rd[i]`. |
| `0xBC` | `VSHUF` | `rd, rs1, rs2` | vector | Shuffle lanes by index `rs2`. |
| `0xBD` | `VMERGE` | `rd, rs1, rs2` | vector | Merge vectors by mask `rs2`. |
| `0xBE` | `VCONF` | `rd, rs1, rs2` | vector | Vector confidence propagation. |
| `0xBF` | `VSELECT` | `rd, rs1, rs2` | vector | Conditional select by confidence mask. |

#### Tensor / Neural (`0xC0–0xCF`)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0xC0` | `TMATMUL` | `rd, rs1, rs2` | tensor | `rd = rs1 @ rs2` (matrix multiply). |
| `0xC1` | `TCONV` | `rd, rs1, rs2` | tensor | 2D convolution. |
| `0xC2` | `TPOOL` | `rd, rs1, rs2` | tensor | Max/avg pooling. |
| `0xC3` | `TRELU` | `rd, rs1, -` | tensor | `rd = max(0, rs1)`. |
| `0xC4` | `TSIGM` | `rd, rs1, -` | tensor | `rd = 1 / (1 + exp(-rs1))`. |
| `0xC5` | `TSOFT` | `rd, rs1, rs2` | tensor | Softmax over dimension `rs2`. |
| `0xC6` | `TLOSS` | `rd, rs1, rs2` | tensor | Loss: type `rs2`, pred `rs1`. |
| `0xC7` | `TGRAD` | `rd, rs1, rs2` | tensor | `rd = ∂loss/∂rs1`, lr=`rs2`. |
| `0xC8` | `TUPDATE` | `rd, rs1, rs2` | tensor | SGD update: `rd -= rs2 * rs1`. |
| `0xC9` | `TADAM` | `rd, rs1, rs2` | tensor | Adam optimizer step. |
| `0xCA` | `TEMBED` | `rd, rs1, rs2` | tensor | Embedding lookup: token `rs1`, table `rs2`. |
| `0xCB` | `TATTN` | `rd, rs1, rs2` | tensor | Self-attention: Q=`rs1`, K=V=`rs2`. |
| `0xCC` | `TSAMPLE` | `rd, rs1, rs2` | tensor | Sample from distribution `rs1`, temp `rs2`. |
| `0xCD` | `TTOKEN` | `rd, rs1, rs2` | tensor | Tokenize: text `rs1`, vocab `rs2` → `rd`. |
| `0xCE` | `TDETOK` | `rd, rs1, rs2` | tensor | Detokenize: tokens `rs1`, vocab `rs2` → `rd`. |
| `0xCF` | `TQUANT` | `rd, rs1, rs2` | tensor | Quantize: fp32 `rs1` → int8, scale `rs2`. |

**Format E total: 157 opcodes** (excluding `0x69` and `0xA0` which are Format D, and `0xA4` which is Format G; those are counted under their actual encoding format)

---

### 4.6 Format F — Register + Imm16 (4 bytes)

#### Short Register + Imm16 (`0x40–0x47`)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0x40` | `MOVI16` | `rd, imm16` | move | `rd = imm16` (unsigned). |
| `0x41` | `ADDI16` | `rd, imm16` | arithmetic | `rd = rd + imm16`. |
| `0x42` | `SUBI16` | `rd, imm16` | arithmetic | `rd = rd - imm16`. |
| `0x43` | `JMP` | `rd, imm16` | control | `PC += sign_extend(imm16)`. rd unused. |
| `0x44` | `JAL` | `rd, imm16` | control | `rd = PC; PC += sign_extend(imm16)`. |
| `0x45` | `CALL` | `rd, imm16` | control | `PUSH(PC); PC = rd + imm16`. |
| `0x46` | `LOOP` | `rd, imm16` | control | `rd--; if rd > 0: PC -= imm16`. |
| `0x47` | `SELECT` | `rd, imm16` | control | `PC += imm16 * rd` (computed jump). |

#### Long Jumps / Coroutines / Debug (`0xE0–0xEF`)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0xE0` | `JMPL` | `rd, imm16` | control | Long relative jump: `PC += sign_extend(imm16)`. |
| `0xE1` | `JALL` | `rd, imm16` | control | `rd = PC; PC += sign_extend(imm16)`. |
| `0xE2` | `CALLL` | `rd, imm16` | control | `PUSH(PC); PC = rd + imm16`. |
| `0xE3` | `TAIL` | `rd, imm16` | control | Pop frame; `PC = rd + imm16`. |
| `0xE4` | `SWITCH` | `rd, imm16` | control | Context switch: save state, jump `imm16`. |
| `0xE5` | `COYIELD` | `rd, imm16` | control | Coroutine yield: save, jump to `imm16`. |
| `0xE6` | `CORESUM` | `rd, imm16` | control | Coroutine resume: restore, jump to `rd`. |
| `0xE7` | `FAULT` | `rd, imm16` | system | Raise fault code `imm16`, context `rd`. |
| `0xE8` | `HANDLER` | `rd, imm16` | system | Install fault handler at `PC + imm16`. |
| `0xE9` | `TRACE` | `rd, imm16` | debug | Log `rd`, tag `imm16`. |
| `0xEA` | `PROF_ON` | `rd, imm16` | debug | Start profiling region `imm16`. |
| `0xEB` | `PROF_OFF` | `rd, imm16` | debug | End profiling region `imm16`. |
| `0xEC` | `WATCH` | `rd, imm16` | debug | Watchpoint: break on write to `rd + imm16`. |
| `0xED` | _RESERVED_ | — | reserved | Reserved. |
| `0xEE` | _RESERVED_ | — | reserved | Reserved. |
| `0xEF` | _RESERVED_ | — | reserved | Reserved. |

**Format F total: 21 defined + 3 reserved = 24 opcodes**

---

### 4.7 Format G — Two Register + Imm16 (5+ bytes)

#### Memory Offset / Frame Management (`0x48–0x4F`)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0x48` | `LOADOFF` | `rd, rs1, imm16` | memory | `rd = mem[rs1 + sign_extend(imm16)]`. |
| `0x49` | `STOREOFF` | `rd, rs1, imm16` | memory | `mem[rs1 + sign_extend(imm16)] = rd`. |
| `0x4A` | `LOADI` | `rd, rs1, imm16` | memory | `rd = mem[mem[rs1] + imm16]` (indirect load). |
| `0x4B` | `STOREI` | `rd, rs1, imm16` | memory | `mem[mem[rs1] + imm16] = rd` (indirect store). |
| `0x4C` | `ENTER` | `rd, rs1, imm16` | stack | Prologue: push regs; `SP -= imm16`; `rd = old SP`. |
| `0x4D` | `LEAVE` | `rd, rs1, imm16` | stack | Epilogue: `SP += imm16`; pop regs; `rd = return value`. |
| `0x4E` | `COPY` | `rd, rs1, imm16` | memory | `memcpy(rd, rs1, imm16)` — copy `imm16` bytes. |
| `0x4F` | `FILL` | `rd, rs1, imm16` | memory | `memset(rd, rs1, imm16)` — fill `imm16` bytes. |

#### Collection Slice (`0xA4`)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0xA4` | `SLICE` | `rd, rs1, imm16` | collection | `rd = rs1[0:imm16]`. |

#### Extended Memory / MMIO / GPU (`0xD0–0xDF`)

| Hex | Mnemonic | Operands | Category | Semantics |
|-----|----------|----------|----------|-----------|
| `0xD0` | `DMA_CPY` | `rd, rs1, imm16` | memory | DMA copy `imm16` bytes: `rd ← rs1`. |
| `0xD1` | `DMA_SET` | `rd, rs1, imm16` | memory | DMA fill `imm16` bytes at `rd` with `rs1`. |
| `0xD2` | `MMIO_R` | `rd, rs1, imm16` | memory | `rd = io[rs1 + imm16]`. |
| `0xD3` | `MMIO_W` | `rd, rs1, imm16` | memory | `io[rs1 + imm16] = rd`. |
| `0xD4` | `ATOMIC` | `rd, rs1, imm16` | memory | Atomic RMW: `rd = swap(mem[rs1+imm16], rd)`. |
| `0xD5` | `CAS` | `rd, rs1, imm16` | memory | Compare-and-swap at `rs1 + imm16`. |
| `0xD6` | `FENCE` | `rd, rs1, imm16` | memory | Memory fence: type `imm16` (acq/rel/full). |
| `0xD7` | `MALLOC` | `rd, rs1, imm16` | memory | Allocate `imm16` bytes, handle → `rd`. |
| `0xD8` | `FREE` | `rd, rs1, imm16` | memory | Free allocation at `rd`. |
| `0xD9` | `MPROT` | `rd, rs1, imm16` | memory | Protect: start=`rd`, len=`rs1`, flags=`imm16`. |
| `0xDA` | `MCACHE` | `rd, rs1, imm16` | memory | Cache: op=`imm16`, addr=`rd`, len=`rs1`. |
| `0xDB` | `GPU_LD` | `rd, rs1, imm16` | memory | GPU load to device mem, offset `imm16`. |
| `0xDC` | `GPU_ST` | `rd, rs1, imm16` | memory | GPU store from device mem. |
| `0xDD` | `GPU_EX` | `rd, rs1, imm16` | compute | GPU execute kernel `rd`, grid=`rs1`, block=`imm16`. |
| `0xDE` | `GPU_SYNC` | `rd, rs1, imm16` | compute | GPU synchronize device `imm16`. |
| `0xDF` | _RESERVED_ | — | reserved | Reserved. |

**Format G total: 24 defined + 1 reserved = 25 opcodes**

---

## 5. Format G — Variable-Length Encoding Rules

Format G is the only encoding format that supports variable-length instructions beyond the fixed 5-byte header. This section specifies the rules for variable-length payloads.

### 5.1 Header Structure

Every Format G instruction begins with a **5-byte fixed header**:

```
Offset  Bytes   Field          Description
------  -----   -----          -----------
+0      1       opcode         Instruction opcode (determines payload interpretation)
+1      1       rd             Destination register [5:0]
+2      1       rs1            Source register 1 [5:0]
+3      1       u16[7:0]      Low byte of 16-bit field (little-endian)
+4      1       u16[15:8]     High byte of 16-bit field (little-endian)
```

### 5.2 Fixed-Length Mode (No Payload)

Most current Format G opcodes use **only the 5-byte header**. In this mode, the u16 field serves as a direct immediate operand:

- `LOADOFF`: `u16` is a signed displacement added to `rs1`.
- `COPY` / `FILL`: `u16` is an unsigned byte count.
- `ENTER` / `LEAVE`: `u16` is the frame size in bytes.
- `DMA_CPY` / `DMA_SET`: `u16` is the transfer size in bytes.
- `MALLOC`: `u16` is the allocation size in bytes.

**Total instruction size:** 5 bytes (header only).

### 5.3 Variable-Length Mode (With Payload)

When an opcode requires inline data (e.g., inline strings for A2A messages, region descriptors, serialized payloads), the u16 field at offsets +3/+4 is reinterpreted as a **payload length prefix**:

```
┌─────────┬─────────┬─────────┬──────────┬──────────┬──────────────────────────────────┐
│ opcode  │   rd    │   rs1   │ len[7:0] │ len[15:8]│         payload[0 … len-1]        │
└─────────┴─────────┴─────────┴──────────┴──────────┴──────────────────────────────────┘
  +0         +1        +2        +3          +4           +5  …  +(5 + len - 1)
```

**Rules:**

1. **Length field (`u16`):** Unsigned 16-bit integer, stored little-endian at offsets +3/+4. Specifies the number of payload bytes that follow the header.
2. **Maximum payload size:** 65535 bytes (`0xFFFF`).
3. **Maximum total instruction size:** 5 (header) + 65535 (payload) = **65540 bytes**.
4. **Zero-length payload:** When `u16 = 0x0000`, the instruction is exactly 5 bytes (equivalent to fixed-length mode). Decoders MUST handle this without special casing.
5. **Payload alignment:** The payload begins at byte offset +5 relative to the instruction start. It has **no alignment guarantee** beyond byte alignment.
6. **Payload content:** The interpretation of payload bytes is opcode-specific. The payload is treated as an opaque byte sequence by the encoding layer; semantic interpretation is the responsibility of the execution unit.

### 5.4 Payload Length Computation

To determine the total instruction size at decode time:

```
header_size   = 5
payload_len   = u16_at(offset +3)     // read LE uint16 from header bytes 3-4
total_size    = header_size + payload_len
next_pc       = current_pc + total_size
```

### 5.5 Opcode-Specific Payload Semantics

| Opcode Pattern | u16 Interpretation | Payload Content |
|----------------|--------------------|-----------------|
| Memory offset ops (`LOADOFF`, `STOREOFF`, `LOADI`, `STOREI`) | Signed displacement / offset | None (fixed 5 bytes) |
| Frame ops (`ENTER`, `LEAVE`) | Unsigned frame size | None (fixed 5 bytes) |
| Bulk memory (`COPY`, `FILL`, `DMA_*`) | Unsigned byte count | None (fixed 5 bytes) |
| A2A inline message ops | Unsigned payload length | Message data (UTF-8 string, serialized struct, etc.) |
| Region ops | Unsigned descriptor length | Region descriptor bytes |
| Future variable-length ops | As defined by opcode | As defined by opcode |

---

## 6. Example Encodings

All examples show the instruction bytes in hexadecimal, with byte offsets labeled.

### 6.1 `HALT` (Format A, opcode `0x00`)

```
Assembly:   HALT

Encoding:
  Offset 0:  0x00

Hex stream:  00
Size:        1 byte
```

### 6.2 `ADD R1, R2, R3` (Format E, opcode `0x20`)

```
Assembly:   ADD R1, R2, R3

Encoding:
  Offset 0:  0x20              opcode = ADD
  Offset 1:  0x01              rd   = R1
  Offset 2:  0x02              rs1  = R2
  Offset 3:  0x03              rs2  = R3

Hex stream:  20 01 02 03
Size:        4 bytes
Semantics:   R1 = R2 + R3
```

### 6.3 `MOVI R5, 42` (Format D, opcode `0x18`)

```
Assembly:   MOVI R5, 42

Encoding:
  Offset 0:  0x18              opcode = MOVI
  Offset 1:  0x05              rd   = R5
  Offset 2:  0x2A              imm8 = 42 (decimal)

Hex stream:  18 05 2A
Size:        3 bytes
Semantics:   R5 = 42
```

### 6.4 `MOVI16 R1, 0x1234` (Format F, opcode `0x40`)

```
Assembly:   MOVI16 R1, 0x1234

Encoding:
  Offset 0:  0x40              opcode = MOVI16
  Offset 1:  0x01              rd       = R1
  Offset 2:  0x34              imm16_lo = 0x34  (low byte)
  Offset 3:  0x12              imm16_hi = 0x12  (high byte)

Hex stream:  40 01 34 12
Size:        4 bytes
Semantics:   R1 = 0x1234 (4660 decimal)
```

### 6.5 `JMP -8` (Format F, opcode `0x43`)

```
Assembly:   JMP -8

Encoding:
  Offset 0:  0x43              opcode = JMP
  Offset 1:  0x00              rd       = R0 (unused)
  Offset 2:  0xF8              imm16_lo = 0xF8  (two's complement of -8)
  Offset 3:  0xFF              imm16_hi = 0xFF  (sign-extended)

Hex stream:  43 00 F8 FF
Size:        4 bytes
Semantics:   PC += -8 (sign_extend(0xFFF8) = -8)
```

### 6.6 `LOADOFF R3, R10, 0x0100` (Format G fixed, opcode `0x48`)

```
Assembly:   LOADOFF R3, R10, 0x0100

Encoding:
  Offset 0:  0x48              opcode = LOADOFF
  Offset 1:  0x03              rd       = R3
  Offset 2:  0x0A              rs1      = R10
  Offset 3:  0x00              imm16_lo = 0x00  (low byte of 256)
  Offset 4:  0x01              imm16_hi = 0x01  (high byte of 256)

Hex stream:  48 03 0A 00 01
Size:        5 bytes (fixed, no payload)
Semantics:   R3 = mem[R10 + 256]
```

### 6.7 `TELL R5, R3, R10` (Format E, opcode `0x50`)

```
Assembly:   TELL R5, R3, R10

Encoding:
  Offset 0:  0x50              opcode = TELL
  Offset 1:  0x05              rd   = R5 (tag)
  Offset 2:  0x03              rs1  = R3 (target agent)
  Offset 3:  0x0A              rs2  = R10 (message pointer)

Hex stream:  50 05 03 0A
Size:        4 bytes
Semantics:   Send message at address R10 to agent R3, tag R5.
```

### 6.8 `TELL "Hello"` (Format G variable-length, hypothetical inline A2A)

This example demonstrates the variable-length payload mode for an inline A2A message.

```
Assembly:   TELL_inline R5, R3, "Hello"

Encoding:
  Offset 0:  0x50              opcode = TELL (Format G variant)
  Offset 1:  0x05              rd       = R5 (tag)
  Offset 2:  0x03              rs1      = R3 (target agent)
  Offset 3:  0x05              len_lo   = 5 (payload = 5 bytes)
  Offset 4:  0x00              len_hi   = 0
  Offset 5:  0x48              'H' (0x48)
  Offset 6:  0x65              'e' (0x65)
  Offset 7:  0x6C              'l' (0x6C)
  Offset 8:  0x6C              'l' (0x6C)
  Offset 9:  0x6F              'o' (0x6F)

Hex stream:  50 05 03 05 00 48 65 6C 6C 6F
Size:        10 bytes (5 header + 5 payload)
```

### 6.9 Multi-instruction Sequence: Sum of Two Numbers

```
Assembly:
    MOVI R1, 10          ; R1 = 10
    MOVI R2, 20          ; R2 = 20
    ADD  R3, R1, R2      ; R3 = R1 + R2
    HALT

Encoding (byte offsets from instruction start):
  Addr 0x00:  18 01 0A          ; MOVI R1, 10
  Addr 0x03:  18 02 14          ; MOVI R2, 20
  Addr 0x06:  20 03 01 02       ; ADD R3, R1, R2
  Addr 0x0A:  00                ; HALT

Hex stream:  18 01 0A 18 02 14 20 03 01 02 00
Total size:  11 bytes (3 + 3 + 4 + 1)
```

### 6.10 Multi-instruction Sequence: Function Call with ENTER/LEAVE

```
Assembly:
    MOVI16 R1, 42        ; R1 = 42
    ENTER  R14, R0, 16   ; save frame, allocate 16 bytes, R14 = old SP
    ; ... body ...
    ADD    R1, R1, R2    ; R1 = R1 + R2
    LEAVE  R1, R0, 16    ; deallocate 16 bytes, R1 = return value
    RET

Encoding:
  Addr 0x00:  40 01 2A 00       ; MOVI16 R1, 42
  Addr 0x04:  4C 0E 00 10 00    ; ENTER R14, R0, 16
  Addr 0x09:  20 01 01 02       ; ADD R1, R1, R2
  Addr 0x0D:  4D 01 00 10 00    ; LEAVE R1, R0, 16
  Addr 0x12:  00                ; RET

Hex stream:  40 01 2A 00 4C 0E 00 10 00 20 01 01 02 4D 01 00 10 00 00
Total size:  19 bytes
```

---

## 7. Alignment and Padding Rules

### 7.1 Instruction Alignment

All instructions are **byte-aligned**. There are no alignment constraints on instruction boundaries beyond byte alignment. An instruction may begin at any byte address.

### 7.2 No Inter-Instruction Padding

The FLUX VM bytecode format does **not** insert padding bytes between instructions. Instructions are packed contiguously:

```
┌────────────┬────────────┬────────────┬───┐
│ Inst. N    │ Inst. N+1  │ Inst. N+2  │…  │
│ (1–5+ B)   │ (1–5+ B)   │ (1–5+ B)   │   │
└────────────┴────────────┴────────────┴───┘
```

The program counter (`PC`) advances by exactly the instruction's computed size after each decode-execute cycle. For fixed-size formats this is trivial; for Format G the decoder reads the u16 length prefix to compute total instruction size before advancing `PC`.

### 7.3 Register Byte Padding

The 6-bit register field occupies a full byte with bits [7:6] reserved (must be zero). These bits are **not** reused for other purposes. This means:

- A register byte always consumes 1 byte of encoding space.
- The high 2 bits are wasted encoding space, reserved for future expansion to 8-bit register indices (256 registers).
- Decoders MUST mask the register byte with `0x3F` to extract the 6-bit index, or MUST trap if bits [7:6] are non-zero.

### 7.4 Immediate Field Padding

- **`imm8`** (Format C, D): Exactly 8 bits, no padding. Occupies 1 byte.
- **`imm16`** (Format F, G): Exactly 16 bits, no padding. Occupies 2 bytes in little-endian order. No alignment constraint on the 16-bit value — it can straddle any byte boundary (though since it always starts at offset +2 or +3 within the instruction, and instructions are byte-aligned, this is trivially satisfied).

### 7.5 Format G Payload Alignment

The variable-length payload in Format G has **no alignment requirement**. It begins at offset +5 within the instruction (immediately after the u16 length prefix). The payload is a raw byte sequence; no alignment padding is inserted before or within it.

### 7.6 Stack Alignment

While not strictly an encoding rule, the FLUX calling convention requires:

- `PUSH` / `POP` operate on 32-bit (4-byte) values.
- `SP` (R11) should be maintained at 4-byte alignment by convention, though the ISA does not enforce this at the hardware level.
- `ENTER` / `LEAVE` frame sizes are specified in bytes via the imm16 field; it is the programmer's/assembler's responsibility to keep frames aligned.

---

## 8. Cross-Reference to `isa_unified.py`

### 8.1 Canonical Source Location

The authoritative machine-readable definition of all opcodes and their format assignments is in:

```
flux-runtime/src/flux/bytecode/isa_unified.py
```

Specifically:
- `build_unified_isa()` — constructs the full opcode list
- `OpcodeDef` dataclass — per-opcode metadata (opcode, mnemonic, format, operands, description, category, source)
- `FormatType` — format string constants (`"A"` through `"G"`)
- `OpcodeDef.byte_size()` — returns the fixed instruction size per format

### 8.2 Format Size Map (from `byte_size()`)

```python
sizes = {
    "A": 1,   # opcode only
    "B": 2,   # opcode + register
    "C": 2,   # opcode + imm8
    "D": 3,   # opcode + register + imm8
    "E": 4,   # opcode + 3 registers
    "F": 4,   # opcode + register + imm16
    "G": 5,   # opcode + 2 registers + imm16 (+ optional payload)
}
```

> **Note:** The `byte_size()` method returns the fixed header size for Format G (5 bytes). For Format G opcodes with variable-length payloads, the total instruction size is `5 + payload_length`, where `payload_length` is the u16 value read from the header bytes at offsets +3/+4.

### 8.3 Opcode Dispatch Implementation

A conformant decoder should dispatch on the opcode byte using the ranges defined in §3.8. Pseudocode:

```python
def decode_instruction(bytecode, pc):
    opcode = bytecode[pc]

    if opcode <= 0x07:                         # Format A
        size = 1
    elif opcode <= 0x0F:                       # Format B
        rd = bytecode[pc + 1] & 0x3F
        size = 2
    elif opcode <= 0x17:                       # Format C
        imm8 = bytecode[pc + 1]
        size = 2
    elif opcode <= 0x1F:                       # Format D
        rd   = bytecode[pc + 1] & 0x3F
        imm8 = bytecode[pc + 2]
        size = 3
    elif opcode <= 0x3F:                       # Format E
        rd  = bytecode[pc + 1] & 0x3F
        rs1 = bytecode[pc + 2] & 0x3F
        rs2 = bytecode[pc + 3] & 0x3F
        size = 4
    elif opcode <= 0x47:                       # Format F
        rd   = bytecode[pc + 1] & 0x3F
        imm16 = bytecode[pc + 2] | (bytecode[pc + 3] << 8)
        size = 4
    elif opcode <= 0x4F:                       # Format G
        rd   = bytecode[pc + 1] & 0x3F
        rs1  = bytecode[pc + 2] & 0x3F
        imm16 = bytecode[pc + 3] | (bytecode[pc + 4] << 8)
        size = 5  # + payload if applicable (see §5)
    elif opcode <= 0x5F:                       # Format E (A2A)
        rd  = bytecode[pc + 1] & 0x3F
        rs1 = bytecode[pc + 2] & 0x3F
        rs2 = bytecode[pc + 3] & 0x3F
        size = 4
    elif opcode <= 0x6F:                       # Format E (confidence)
        if opcode == 0x69:                     # C_THRESH is Format D
            rd   = bytecode[pc + 1] & 0x3F
            imm8 = bytecode[pc + 2]
            size = 3
        else:
            rd  = bytecode[pc + 1] & 0x3F
            rs1 = bytecode[pc + 2] & 0x3F
            rs2 = bytecode[pc + 3] & 0x3F
            size = 4
    elif opcode <= 0xCF:                       # Format E (viewpoint, sensor, math, collection, vector, tensor)
        if opcode == 0xA0:                     # LEN is Format D
            rd   = bytecode[pc + 1] & 0x3F
            imm8 = bytecode[pc + 2]
            size = 3
        elif opcode == 0xA4:                   # SLICE is Format G
            rd   = bytecode[pc + 1] & 0x3F
            rs1  = bytecode[pc + 2] & 0x3F
            imm16 = bytecode[pc + 3] | (bytecode[pc + 4] << 8)
            size = 5
        else:
            rd  = bytecode[pc + 1] & 0x3F
            rs1 = bytecode[pc + 2] & 0x3F
            rs2 = bytecode[pc + 3] & 0x3F
            size = 4
    elif opcode <= 0xDF:                       # Format G (memory/MMIO/GPU)
        rd   = bytecode[pc + 1] & 0x3F
        rs1  = bytecode[pc + 2] & 0x3F
        imm16 = bytecode[pc + 3] | (bytecode[pc + 4] << 8)
        size = 5
    elif opcode <= 0xEF:                       # Format F (long jumps)
        rd   = bytecode[pc + 1] & 0x3F
        imm16 = bytecode[pc + 2] | (bytecode[pc + 3] << 8)
        size = 4
    else:                                      # Format A (0xF0–0xFF)
        size = 1

    return (opcode, size, ...)                 # Advance PC by size
```

### 8.4 Related Specification Documents

| Document | Location | Content |
|----------|----------|---------|
| ISA.md | `flux-spec/ISA.md` | Full ISA specification (execution model, flags, conformance) |
| OPCODES.md | `flux-spec/OPCODES.md` | Machine-readable opcode table (generated from `isa_unified.py`) |
| SIGNAL.md | `flux-spec/SIGNAL.md` | Signal protocol specification |
| A2A.md | `flux-spec/A2A.md` | Agent-to-Agent communication protocol |
| FIR.md | `flux-spec/FIR.md` | FLUX Intermediate Representation |

---

## Appendix A: Opcode Count Summary

| Format | Defined | Reserved | Total |
|--------|---------|----------|-------|
| A | 19 | 5 | 24 |
| B | 8 | 0 | 8 |
| C | 8 | 0 | 8 |
| D | 10 | 0 | 10 |
| E | 157 | 0 | 157 |
| F | 21 | 3 | 24 |
| G | 24 | 1 | 25 |
| **Grand Total** | **247** | **9** | **256** |

## Appendix B: Little-Endian Encoding Reference

### Encoding a u16 value

To encode a 16-bit unsigned value `v` into two bytes at offsets `lo` and `hi`:

```
byte[lo] = v & 0xFF          // low byte (bits 7:0)
byte[hi] = (v >> 8) & 0xFF   // high byte (bits 15:8)
```

### Decoding a u16 value

To decode two bytes at offsets `lo` and `hi` into a 16-bit unsigned value:

```
v = byte[lo] | (byte[hi] << 8)
```

### Sign-extending a u16 to i32

For jump offsets and signed immediates:

```
i32_value = u16_value if u16_value < 0x8000 else u16_value - 0x10000
```

### Sign-extending an imm8 to i32

For Format D arithmetic operations:

```
i32_value = imm8 if imm8 < 0x80 else imm8 - 0x100
```

---

*This specification is maintained in [flux-spec](https://github.com/SuperInstance/flux-spec). All FLUX VM implementations, assemblers, and toolchains MUST conform to the encoding rules defined herein.*
