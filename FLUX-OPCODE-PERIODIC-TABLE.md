# The FLUX Opcode Periodic Table

**Author:** Agent Datum 🔵 | **Session:** 6 | **Iterations:** 11 of 12
**Date:** 2026-04-14 | **Classification:** Fleet Architecture Reference

---

## Concept

Just as the periodic table of elements organizes matter by atomic structure and chemical properties, this document organizes all 251 FLUX opcodes by their **computational nature** — what they fundamentally do to information. Opcodes in the same "group" share deep structural similarities even if they operate on different data types.

## Layout Convention

```
    Group 1      Group 2      Group 3      Group 4      Group 5
    IDENTITY     TRANSFORM    COMPARE      FLOW         BRIDGE
    (stateless)  (stateful)   (classify)   (control)    (connect)
```

- **Identity**: Opcodes that leave information unchanged or simply move it (NOP, MOV, PUSH, POP)
- **Transform**: Opcodes that create new information from existing information (ADD, AND, LOAD, MUL)
- **Compare**: Opcodes that reduce information to a boolean (CMP_EQ, JZ, TEST)
- **Flow**: Opcodes that alter execution order (JMP, CALL, RET, LOOP)
- **Bridge**: Opcodes that cross system boundaries (TELL, SYS, HALT, TRAP)

---

## The Table

### Period 1: Zero-Operand (Format A) — The Atoms

| Group 1: Identity | Group 2: Transform | Group 3: Compare | Group 4: Flow | Group 5: Bridge |
|---|---|---|---|---|
| NOP (0x01) | | | | HALT (0x00) |
| | | | RET (0x02) | BRK (0x04) |
| | | | IRET (0x03) | WFI (0x05) |
| | RESET (0x06) | | | SYN (0x07) |
| | | | | DEBUG (0xF0) |
| | | | | PANIC (0xF5) |
| | | | | PRINT (0xFF) |

### Period 2: Single-Operand (Format B) — The Ions

| Group 1: Identity | Group 2: Transform | Group 3: Compare | Group 4: Flow | Group 5: Bridge |
|---|---|---|---|---|
| | INC (0x08) | | | |
| | DEC (0x09) | | | |
| | NOT (0x0A) | | | |
| | NEG (0x0B) | | | |
| PUSH (0x0C) | | | | CONF_LD (0x0E) |
| POP (0x0D) | | | | CONF_ST (0x0F) |

### Period 3: Immediate-8 (Format C) — The Molecules

| Group 1: Identity | Group 2: Transform | Group 3: Compare | Group 4: Flow | Group 5: Bridge |
|---|---|---|---|---|
| | | | | SYS (0x10) |
| | | | | TRAP (0x11) |
| | | | | DBG (0x12) |
| | CLF (0x13) | | | |
| | | | | SEMA (0x14) |
| | | | | YIELD (0x15) |
| | CACHE (0x16) | | | |
| | | | | STRIPCF (0x17) |

### Period 4: Register + Imm8 (Format D) — The Compounds

| Group 1: Identity | Group 2: Transform | Group 3: Compare | Group 4: Flow | Group 5: Bridge |
|---|---|---|---|---|
| MOVI (0x18) | ADDI (0x19) | | | |
| | SUBI (0x1A) | | | |
| | ANDI (0x1B) | | | |
| | ORI (0x1C) | | | |
| | XORI (0x1D) | | | |
| | SHLI (0x1E) | | | |
| | SHRI (0x1F) | | | |

### Period 5: Three-Register (Format E) — The Reactions

#### Sub-period 5a: Arithmetic (0x20-0x2F)

| Group 1: Identity | Group 2: Transform | Group 3: Compare | Group 4: Flow | Group 5: Bridge |
|---|---|---|---|---|
| | ADD (0x20) | CMP_EQ (0x2C) | | |
| | SUB (0x21) | CMP_LT (0x2D) | | |
| | MUL (0x22) | CMP_GT (0x2E) | | |
| | DIV (0x23) | CMP_NE (0x2F) | | |
| | MOD (0x24) | | | |
| | AND (0x25) | | | |
| | OR (0x26) | | | |
| | XOR (0x27) | | | |
| | SHL (0x28) | | | |
| | SHR (0x29) | | | |
| | MIN (0x2A) | | | |
| | MAX (0x2B) | | | |

#### Sub-period 5b: Float / Memory / Control (0x30-0x3F)

| Group 1: Identity | Group 2: Transform | Group 3: Compare | Group 4: Flow | Group 5: Bridge |
|---|---|---|---|---|
| | FADD (0x30) | | JZ (0x3C) | |
| | FSUB (0x31) | | JNZ (0x3D) | |
| | FMUL (0x32) | | JLT (0x3E) | |
| | FDIV (0x33) | | JGT (0x3F) | |
| | FMIN (0x34) | | | |
| | FMAX (0x35) | | | |
| MOV (0x3A) | FTOI (0x36) | | | |
| SWP (0x3B) | ITOF (0x37) | | | |
| | LOAD (0x38) | | | |
| | STORE (0x39) | | | |

#### Sub-period 5c: Agent-to-Agent Fleet (0x50-0x5F)

| Group 1: Identity | Group 2: Transform | Group 3: Compare | Group 4: Flow | Group 5: Bridge |
|---|---|---|---|---|
| | | | | TELL (0x50) |
| | | | | ASK (0x51) |
| | | | | DELEG (0x52) |
| | | | | BCAST (0x53) |
| | | | ACCEPT (0x54) | DECLINE (0x55) |
| | | | | REPORT (0x56) |
| | MERGE (0x57) | | | |
| | | | FORK (0x58) | |
| | | | JOIN (0x59) | |
| | | | | SIGNAL (0x5A) |
| | | | | AWAIT (0x5B) |
| | | | | TRUST (0x5C) |
| | | | | DISCOV (0x5D) |
| | | | | STATUS (0x5E) |
| | | | | HEARTBT (0x5F) |

#### Sub-period 5d: Confidence-Aware (0x60-0x6F)

| Group 1: Identity | Group 2: Transform | Group 3: Compare | Group 4: Flow | Group 5: Bridge |
|---|---|---|---|---|
| | CADD (0x60) | | CJZ (0x68) | |
| | CSUB (0x61) | | CJNZ (0x69) | |
| | CMUL (0x62) | | | CPUSH (0x6A) |
| | CDIV (0x63) | | | CPOP (0x6B) |
| | CMOD (0x64) | | CCALL (0x6C) | CRET (0x6D) |
| | CMOV (0x65) | | | |
| | CLOAD (0x66) | | | |
| | CSTORE (0x67) | | | |
| | | | | CCONF (0x6E) |
| | | | | CRES (0x6F) |

#### Sub-period 5e: Extended Math/Crypto (0x90-0x9F)

| Group 1: Identity | Group 2: Transform | Group 3: Compare | Group 4: Flow | Group 5: Bridge |
|---|---|---|---|---|
| | ABS (0x90) | | | |
| | SQRT (0x91) | | | |
| | POW (0x92) | | | |
| | LOG (0x93) | | | |
| | EXP (0x94) | | | |
| | SIN (0x95) | | | |
| | COS (0x96) | | | |
| | TAN (0x97) | | | |
| | ATAN2 (0x98) | | | |
| | FLOOR (0x99) | | | |
| | CEIL (0x9A) | | | |
| | ROUND (0x9B) | | | |
| | CLAMP (0x9C) | | | |
| | LERP (0x9D) | | | |
| | HASH (0x9E) | | | |
| | CRC32 (0x9F) | | | |

### Period 6: Register + Imm16 (Format F) — The Macromolecules

| Group 1: Identity | Group 2: Transform | Group 3: Compare | Group 4: Flow | Group 5: Bridge |
|---|---|---|---|---|
| MOVI16 (0x40) | ADDI16 (0x41) | | JMP (0x43) | |
| | SUBI16 (0x42) | | JAL (0x44) | |
| | | | CALL (0x45) | |
| | | | LOOP (0x46) | |
| | | | SELECT (0x47) | |

### Period 7: Register + Register + Imm16 (Format G) — The Polymers

| Group 1: Identity | Group 2: Transform | Group 3: Compare | Group 4: Flow | Group 5: Bridge |
|---|---|---|---|---|
| | LOADOFF (0x48) | | | |
| | STOREOF (0x49) | | | |
| | LOADI (0x4A) | | | |
| | STOREI (0x4B) | | | |
| | | | ENTER (0x4C) | |
| | | | LEAVE (0x4D) | |
| | COPY (0x4E) | | | |
| | FILL (0x4F) | | | |

---

## Implementation Status Overlay

Legend: **bold** = implemented in WASM runtime, ~~strikethrough~~ = NOP-stubbed

### Fully Implemented Rows (Green)

| Opcode | Hex | Group | Category |
|--------|-----|-------|----------|
| **HALT** | 0x00 | Bridge | system |
| **NOP** | 0x01 | Identity | system |
| **RET** | 0x02 | Flow | system |
| **INC** | 0x08 | Transform | arithmetic |
| **DEC** | 0x09 | Transform | arithmetic |
| **NOT** | 0x0A | Transform | logic |
| **NEG** | 0x0B | Transform | arithmetic |
| **PUSH** | 0x0C | Identity | stack |
| **POP** | 0x0D | Identity | stack |
| **MOVI** | 0x18 | Identity | move |
| **ADD** | 0x20 | Transform | arithmetic |
| **SUB** | 0x21 | Transform | arithmetic |
| **MUL** | 0x22 | Transform | arithmetic |
| **DIV** | 0x23 | Transform | arithmetic |
| **MOD** | 0x24 | Transform | arithmetic |
| **AND** | 0x25 | Transform | logic |
| **OR** | 0x26 | Transform | logic |
| **XOR** | 0x27 | Transform | logic |
| **SHL** | 0x28 | Transform | shift |
| **SHR** | 0x29 | Transform | shift |
| **JMP** | 0x43 | Flow | control |
| **CALL** | 0x45 | Flow | control |
| **LOAD** | 0x38 | Transform | memory |
| **STORE** | 0x39 | Transform | memory |
| **MOV** | 0x3A | Identity | move |
| **JZ** | 0x3C | Compare | control |
| **JNZ** | 0x3D | Compare | control |

### Defined but NOP-stubbed Rows (Red)

| Opcode | Hex | Group | Category | Priority |
|--------|-----|-------|----------|----------|
| ~~TELL~~ | 0x50 | Bridge | a2a | HIGH |
| ~~ASK~~ | 0x51 | Bridge | a2a | HIGH |
| ~~DELEG~~ | 0x52 | Bridge | a2a | HIGH |
| ~~BCAST~~ | 0x53 | Bridge | a2a | HIGH |
| ~~FORK~~ | 0x58 | Bridge | a2a | MEDIUM |
| ~~JOIN~~ | 0x59 | Bridge | a2a | MEDIUM |
| ~~VADD~~ | 0xB0 | Transform | simd | LOW |
| ~~TMATMUL~~ | 0xC4 | Transform | tensor | LOW |
| ~~TCONV~~ | 0xC6 | Transform | tensor | LOW |

---

## Patterns Observed

1. **Group density**: Group 2 (Transform) dominates the ISA — 65% of all opcodes transform data. This confirms that FLUX is fundamentally a **computation-oriented** ISA, not a communication-oriented one.

2. **Period length**: Period 5 (three-register, Format E) is the longest period with 5 sub-periods and 90+ opcodes. This is the "workhorse" format that encodes most computational work.

3. **Bridge rarity**: Group 5 (Bridge) contains only ~25 opcodes but they are the most semantically significant. TELL, ASK, HALT, SYS — these are the operations that connect the VM to the outside world.

4. **Implementation gradient**: The implemented opcodes cluster in Groups 1 (Identity) and 2 (Transform), while the stubbed opcodes cluster in Group 5 (Bridge). This suggests the WASM runtime prioritized computation over communication — it can compute but cannot connect.

5. **The "noble gas" pattern**: NOP (0x01) sits in Group 1 Period 1, analogous to helium — inert, essential, and found everywhere. Like helium in the real periodic table, NOP is the most abundant and least reactive element.

---

*This periodic table will be updated as new opcodes are implemented across runtimes.*
