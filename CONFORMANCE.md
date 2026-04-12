# FLUX ISA Conformance Guide

**Version:** 1.1
**Status:** Normative
**Date:** 2025-04-12
**Addresses:** Issues #3, #4, #5, #6

---

## 1. Overview

This document governs the relationship between the FLUX ISA specification (this repo)
and the various FLUX runtime implementations. It defines what "conformant" means,
documents known divergences, and provides a migration path for runtimes.

### Authority

**This repository (`flux-spec`) is the canonical source of truth.** All FLUX
implementations must conform to the specifications defined here:
- `ISA.md` — Opcode map, register conventions, encoding formats, execution model
- `OPCODES.md` — Machine-readable opcode table (generated from `isa_unified.py`)
- `ENCODING-FORMATS.md` — Formal byte-level encoding specification (Formats A–G)
- `FIR.md` — SSA-based intermediate representation
- `A2A.md` — Agent-to-agent protocol
- `SIGNAL.md` — Signal language

The `isa_unified.py` module (in `flux-runtime/src/flux/bytecode/`) is the
programmatic source from which `OPCODES.md` is derived. It represents the
**converged ISA** — the single authoritative opcode table agreed upon by all three
contributing agents (Oracle1, JetsonClaw1, Babel).

---

## 2. Known Implementation Divergence

### 2.1 Legacy Opcode Space (`opcodes.py`)

The Python runtime's `opcodes.py` (`flux-runtime/src/flux/bytecode/opcodes.py`)
uses a **legacy pre-convergence opcode numbering** that predates the unified ISA.
The opcode values in `opcodes.py` do NOT match the canonical spec.

**This is the #1 convergence blocker** (Issues #3 and #5).

#### Key Differences

| Instruction | Canonical Spec (`isa_unified.py`) | Legacy VM (`opcodes.py`) | Match? |
|-------------|----------------------------------|--------------------------|--------|
| HALT | `0x00` | `0x80` | ❌ |
| NOP | `0x01` | `0x00` | ❌ |
| MOV | `0x3A` (Format E, 4B) | `0x01` (Format C, 3B) | ❌ |
| ADD | `0x20` (Format E, 4B) | `0x08` (Format E, 4B) | ❌ |
| SUB | `0x21` | `0x09` | ❌ |
| LOAD | `0x38` (Format E, 4B) | `0x02` (Format C, 3B) | ❌ |
| STORE | `0x39` (Format E, 4B) | `0x03` (Format C, 3B) | ❌ |
| JMP | `0x43` (Format F, 4B) | `0x04` (Format D, 4B) | ❌ |
| JZ | `0x3C` (Format E, 4B) | `0x05` (Format D, 4B) | ❌ |
| PUSH | `0x0C` (Format B, 2B) | `0x20` (Format B, 2B) | ❌ |
| POP | `0x0D` (Format B, 2B) | `0x21` (Format B, 2B) | ❌ |
| MOVI | `0x18` (Format D, 3B) | `0x2B` (Format D, 4B) | ❌ |

#### Format Differences

| Aspect | Canonical Spec | Legacy VM |
|--------|---------------|-----------|
| Number of formats | 7 (A–G) | 6 (A, B, C, D, E, G) |
| Format C | 2B (opcode + imm8) | 3B (opcode + rd + rs1) |
| Format D | 3B (opcode + rd + imm8) | 4B (opcode + rd + imm16) |
| Format E | 4B (opcode + rd + rs1 + rs2) | 4B (opcode + rd + rs1 + rs2) — same |
| Register width | 6-bit (64 registers) | 4-bit implied (16 registers) |
| Endianness | Little-endian (documented) | Little-endian (correct) |

### 2.2 Encoding Format Documentation (Issues #4 and #6)

`ENCODING-FORMATS.md` provides the complete, formal byte-level encoding
specification for all seven instruction formats (A through G). This includes:

- Bit-level format diagrams for each format
- Register encoding rules (6-bit index, bits [7:6] reserved)
- Complete opcode tables organized by format
- Variable-length encoding rules for Format G
- Example encodings with byte offsets
- Endianness specification (little-endian throughout)
- Opcode range → format dispatch table

**Status:** ✅ Resolved. `ENCODING-FORMATS.md` (1,131 lines) is the definitive
encoding reference. All issues about missing format documentation are addressed
by this document.

### 2.3 Vocabulary System Divergence

The `.fluxvocab` system's compiled interpreter (`FLUXVOCAB.md §9`) uses the
legacy opcode space from `opcodes.py`, not the converged ISA. Bytecode generated
by the vocabulary assembler is NOT compatible with the canonical spec. This is
documented in FLUXVOCAB.md §12 (Known Limitations, item 5).

**Migration:** The vocabulary assembler needs to be updated to use canonical
opcode values from `isa_unified.py`.

---

## 3. Conformance Levels

### Level 1 — Core (Must-Have)

An implementation is Level 1 conformant if it:

- [ ] Uses canonical opcode values from `isa_unified.py` / `OPCODES.md`
- [ ] Correctly encodes/decodes all Format A–D opcodes
- [ ] Implements all integer arithmetic (0x20–0x2F) with correct flags
- [ ] Implements LOAD (0x38), STORE (0x39), MOV (0x3A) with Format E encoding
- [ ] Implements JZ (0x3C), JNZ (0x3D), JGT (0x3F), JLT (0x3E) with Format E
- [ ] Implements MOVI16 (0x40), JMP (0x43), JAL (0x44) with Format F
- [ ] Implements PUSH (0x0C), POP (0x0D), RET (0x02), HALT (0x00), NOP (0x01)
- [ ] Follows little-endian encoding for all multi-byte fields
- [ ] Correct three-operand read-before-write semantics
- [ ] Register file: R0 hardwired to 0, R11=SP, R14=FP, R15=LR

### Level 2 — Standard (Should-Have)

Level 1 plus:

- [ ] Float arithmetic (0x30–0x37)
- [ ] FTOI (0x36), ITOF (0x37)
- [ ] Format F (0x40–0x47) and Format G (0x48–0x4F)
- [ ] ENTER (0x4C) / LEAVE (0x4D) frame management
- [ ] Named memory regions with ownership
- [ ] A2A handler callback (0x50–0x5F)

### Level 3 — Extended (Nice-to-Have)

Level 2 plus:

- [ ] Confidence-aware variants (0x60–0x6F)
- [ ] Viewpoint operations (0x70–0x7F)
- [ ] Sensor operations (0x80–0x8F)
- [ ] Extended math/crypto (0x90–0x9F)
- [ ] Collection ops (0xA0–0xAF)
- [ ] Vector/SIMD (0xB0–0xBF)
- [ ] Tensor/neural (0xC0–0xCF)
- [ ] DMA/MMIO/GPU (0xD0–0xDF)
- [ ] Long jumps/coroutines/debug (0xE0–0xEF)
- [ ] Extended system (0xF0–0xFF)

---

## 4. Runtime Conformance Status

| Runtime | Language | Conformance | Notes |
|---------|----------|-------------|-------|
| flux-runtime | Python | **Level 2 (legacy opcodes)** | Uses `opcodes.py` legacy numbering. Functional but NOT opcode-compatible with spec. |
| flux-core | Rust | **Level 2 (target)** | Should target canonical opcodes. |
| flux-zig | Zig | **Level 1 (target)** | Fastest VM; should adopt canonical spec. |
| flux-js | JavaScript | **Level 2 (target)** | JS VM with A2A; should use canonical spec. |

### Python Runtime Migration Path

The Python runtime (`flux-runtime`) needs to transition from `opcodes.py` to
`isa_unified.py` as the canonical opcode source. The recommended approach:

1. **Phase 1:** Add a compatibility shim in `opcodes.py` that provides canonical
   opcode values alongside legacy ones (aliases).
2. **Phase 2:** Update the VM interpreter to use canonical format dispatch
   (the 7-format model from `ENCODING-FORMATS.md §3.8`).
3. **Phase 3:** Update the bytecode encoder/decoder in the FIR pipeline.
4. **Phase 4:** Remove legacy opcode values, making `isa_unified.py` the sole
   source of truth.

---

## 5. Test Vectors

Conformance test vectors are maintained in [flux-conformance](https://github.com/SuperInstance/flux-conformance).
All Level 1 tests must pass for an implementation to claim FLUX compatibility.

### Minimal Smoke Test

The following bytecode sequence should execute on any Level 1 conformant VM:

```
; Load 42 into R1, 17 into R2, add them, halt
0x18 0x01 0x2A   ; MOVI R1, 42       (Format D)
0x18 0x02 0x11   ; MOVI R2, 17       (Format D)
0x20 0x00 0x01 0x02  ; ADD R0, R1, R2  (Format E)
0x00             ; HALT              (Format A)
; Expected: R0 = 59, halted = true
```

---

## 6. Change History

| Date | Version | Change |
|------|---------|--------|
| 2025-04-12 | 1.0 | Initial conformance guide. Documented legacy divergence. |
| 2025-04-12 | 1.1 | Added encoding format cross-references. Resolved issues #3–#6. |

---

*This document is maintained in [flux-spec](https://github.com/SuperInstance/flux-spec).*
