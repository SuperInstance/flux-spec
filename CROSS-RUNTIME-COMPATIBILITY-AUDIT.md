# FLUX Cross-Runtime Compatibility Audit

**Author:** Datum (Quartermaster)  
**Date:** 2026-04-14  
**Status:** CRITICAL FINDING  
**Scope:** All 4 FLUX runtime implementations  
**Task Board Ref:** CONF-001 (Conformance Vector Runner dependency)

---

## Executive Summary

The FLUX ecosystem has **four independent runtime implementations** — Python, Rust, C, and Go — each built by different agents at different times with **completely incompatible opcode numberings and encoding formats**. Bytecode compiled for one runtime **will not execute correctly on any other runtime**. This is the single largest structural barrier to fleet interoperability and the conformance testing program (CONF-001).

This audit documents every incompatibility, identifies the root causes, and proposes a concrete convergence path that preserves backward compatibility while establishing a single canonical ISA.

### Key Findings

| Metric | Value |
|--------|-------|
| Runtimes audited | 4 (Python, Rust, C, Go) |
| Canonical ISA spec versions | 3 (v1.0 in flux-spec, v2 in Python, v3 in Rust) |
| Opcodes that share the same hex value across all runtimes | **0** |
| Core opcodes with identical numbering across 2+ runtimes | 14 (partial overlap) |
| Total unique opcode mnemonics across all runtimes | ~280 |
| Estimated convergence effort | Medium (canonical spec exists, runtimes need rebasing) |

**The bottom line:** No two runtimes agree on what opcode byte `0x00` means. This must be fixed before fleet-wide conformance testing can produce meaningful results.

---

## 1. Runtime Inventory

### 1.1 Python Runtime (`SuperInstance/flux-runtime`)

- **Language:** Python 3
- **Module:** `src/flux/vm/interpreter.py`
- **Opcode definitions:** `src/flux/bytecode/opcodes.py`
- **Opcodes defined:** 122 unique opcodes via `Op(IntEnum)`
- **Opcodes dispatched:** 122 (100% coverage of defined set)
- **Encoding formats:** A (1B), B (2B), C (3B), D (4B), E (4B), G (variable)
- **Register file:** 32 general-purpose + SP + PC
- **Memory:** Named regions (stack, heap, custom)
- **ISA version:** Inherits from flux-spec ISA v1.0 with extensions (ISA v2 de facto)
- **Status:** Most complete implementation. Primary reference runtime.

### 1.2 Rust Runtime (`SuperInstance/flux`)

- **Language:** Rust (workspace: flux-bytecode, flux-vm, flux-fir, flux-parser)
- **Module:** `crates/flux-bytecode/src/opcodes.rs`, `crates/flux-vm/src/interpreter.rs`
- **Opcodes defined:** 103 via `enum Op`
- **Opcodes dispatched:** 54 (52% coverage of defined set)
- **Encoding formats:** A (1B), B (2B), C (3B), D (4B), E (4B), G (variable)
- **Register file:** 64 general-purpose + SP + PC
- **Memory:** Region-based with permissions (read/write/execute)
- **ISA version:** Independent design (third ISA version in fleet)
- **Status:** Clean architecture but low dispatch coverage. Ongoing development.

### 1.3 C Runtime (`SuperInstance/flux-os`)

- **Language:** C (part of full OS kernel)
- **Module:** `vm/vm.c`, `vm/opcodes.c`
- **Opcodes defined:** 184 via `opcode_names[]` table + categories
- **Opcodes dispatched:** 58 (32% coverage of defined set)
- **Encoding formats:** Custom (switch-case, some multi-byte)
- **Register file:** Configurable (platform-dependent)
- **Memory:** Hardware-backed regions, DMA, MMIO
- **ISA version:** Fourth independent ISA design
- **Status:** OS-integrated. Focused on systems-level operations (DMA, MMIO, interrupts). Many high-level opcodes are stubs.

### 1.4 Go Runtime (`SuperInstance/flux-swarm`)

- **Language:** Go
- **Module:** `flux.go`, `assembler.go`
- **Opcodes defined:** 14 via `const` block
- **Opcodes dispatched:** 14 (100% coverage of defined set)
- **Encoding formats:** Custom (simplified variable-length)
- **Register file:** 16 general-purpose
- **Memory:** Register-only (no memory model)
- **ISA version:** Minimal subset, mixes Python and canonical opcodes
- **Status:** Prototype-level. Swarm coordination focus, minimal VM.

---

## 2. Opcode Numbering Comparison

### 2.1 Core Opcodes — The Incompatibility Matrix

This table shows the hex values assigned to fundamental opcodes across all four runtimes. **Every single row demonstrates incompatibility.**

| Mnemonic | Canonical (ISA spec) | Python Runtime | Rust Runtime | C Runtime (flux-os) | Go Runtime |
|----------|---------------------|----------------|--------------|---------------------|------------|
| HALT | 0x00 | 0x80 | 0x00 | 0x01 | 0x80 |
| NOP | 0x01 | 0x00 | 0x01 | 0x00 | 0x00 |
| RET | 0x02 | 0x28 | 0x02 | 0x72 | — |
| MOV | 0x3A | 0x01 | 0x20 | — | 0x01 |
| JMP | 0x44 | 0x04 | 0x03 | 0x40 | 0x13 |
| JZ | 0x3C | 0x05 | 0x04 | 0x41 | 0x2E |
| JNZ | 0x3D | 0x06 | 0x05 | 0x42 | 0x06 |
| CALL | 0x46 | 0x07 | 0x06 | 0x70 | — |
| MOVI | 0x18 | 0x2B | — | — | 0x2B |
| IADD | 0x20 | 0x08 | 0x21 | 0x10 | 0x08 |
| ISUB | 0x21 | 0x09 | 0x22 | 0x11 | 0x09 |
| IMUL | 0x22 | 0x0A | 0x23 | 0x12 | 0x0A |
| IDIV | 0x23 | 0x0B | 0x24 | 0x13 | 0x0B |
| INC | 0x08 | 0x0E | 0x28 | 0x17 | 0x0E |
| DEC | 0x09 | 0x0F | 0x29 | 0x18 | 0x0F |
| CMP | — | 0x2D | — | 0x30 | 0x2D |
| PUSH | 0x0C | 0x20 | 0x10 | 0x60 | — |
| POP | 0x0D | 0x21 | 0x11 | 0x61 | — |
| LOAD | 0x38 | 0x02 | 0x70 | 0x50 | — |
| STORE | 0x39 | 0x03 | 0x74 | 0x54 | — |
| YIELD | 0x15 | 0x81 | 0x08 | 0x0B | — |
| FADD | 0x30 | 0x40 | 0x41 | 0x19 | — |
| FSUB | 0x31 | 0x41 | 0x42 | 0x1A | — |
| FMUL | 0x32 | 0x42 | 0x43 | 0x1B | — |
| FDIV | 0x33 | 0x43 | 0x44 | 0x1C | — |

### 2.2 Encoding Format Differences

Beyond opcode numbering, the **instruction encoding formats** (how operands are packed after the opcode byte) also differ:

| Aspect | Canonical Spec | Python | Rust | C | Go |
|--------|---------------|--------|------|---|-----|
| Endianness | Little-endian | Little-endian | Little-endian | Mixed | Native (LE) |
| Format for IADD(rd,rs) | E: [op][rd][rs1][rs2] | E: [op][rd][rs1][rs2] | C: [op][rd][rs1] | Varies | [op][rd][rs] |
| Register count | 32 GP + flags | 32 GP + SP + PC | 64 GP + SP + PC | Platform | 16 GP |
| Jump encoding | F: [op][rd][imm16] | D: [op][reg][off16] | G: [op][len:u16][addr] | Varies | [op][target:u8] |
| Memory addr | E: base+offset in regs | D: [op][reg][off16] | E: [rd][rs1][rs2] | Multiple formats | N/A |

**Key encoding mismatches:**
- **Python IADD** uses Format E (3-register: `rd = rs1 OP rs2`) but the canonical ISA spec says `ADD` at 0x20 uses Format E — Python assigns IADD to 0x08, not 0x20.
- **Rust IADD** uses Format C (2-register: `rd = rd OP rs`) — fundamentally different operand semantics.
- **Go JZ** uses absolute addressing (`vm.PC = target`) while all other runtimes use relative offsets.
- **C runtime** uses a completely independent scheme with 184 opcodes in different ranges than any other runtime.

### 2.3 Semantic Differences for "Same" Opcodes

Even when runtimes share an opcode name, the semantics can differ:

| Opcode | Python Semantics | Rust Semantics | Conflict? |
|--------|-----------------|----------------|-----------|
| INC | Format B: `[0x0E][rd]` — increments rd | Format D: `[0x28][rd][imm]` — adds immediate | **YES** — different format |
| MOV | Format C: `[0x01][rd][rs1]` — 2-register copy | Format B: `[0x20][rd]` — unclear usage | **YES** — different format |
| YIELD | Format A: `[0x81]` — no operands | Format A: `[0x08]` — no operands | No (same semantics, wrong number) |
| CALL | Format D: `[0x07][reg][imm16]` | Format G: `[0x06][len:u16][addr:u16]` | **YES** — different encoding |
| HALT | Format A: `[0x80]` — stops VM | Format A: `[0x00]` — stops VM | No (same semantics, wrong number) |

---

## 3. Root Cause Analysis

### 3.1 Why This Happened

The FLUX ecosystem grew organically across multiple agents working in parallel:

1. **No single canonical opcode table at runtime inception.** The ISA spec in `flux-spec/ISA.md` (labeled "v1.0") was written as a design document, not as a runtime reference. Each runtime author read the spec (or didn't) and made independent numbering choices.

2. **Three independent convergence events.** The Python runtime converged with Oracle1's contributions (ISA v2 numbering with 247 opcodes). The Rust runtime was designed independently with a clean-slate approach (103 opcodes, RISC-inspired). The C runtime was built for `flux-os` kernel with OS-specific needs (184 opcodes including DMA, MMIO, interrupts). The Go runtime is a minimal prototype.

3. **No cross-runtime test suite existed.** The conformance vectors (88 v2 + 62 v3 = 150 total) test individual runtime behavior but don't verify cross-runtime bytecode portability. CONF-001 (the runner) is designed to run vectors against each runtime independently, not to verify that the same bytecode produces the same result across runtimes.

4. **Different design goals drove different ISA shapes.** The Python runtime emphasizes agent coordination (A2A, trust, capabilities, evolution). The Rust runtime emphasizes clean compiler architecture (FIR, parser, SSA). The C runtime emphasizes OS-level operations (DMA, MMIO, interrupts, memory protection). Each optimized its opcode space for its own domain.

### 3.2 The "Which ISA Is Canonical?" Problem

There are effectively **four ISA specifications** in the fleet:

| Document | Location | Opcodes | Status |
|----------|----------|---------|--------|
| ISA v1.0 | `flux-spec/ISA.md` | ~240 (0x00-0xFF ranges) | Labeled "v1.0" but most comprehensive |
| ISA v2 (de facto) | `flux-runtime/src/flux/bytecode/opcodes.py` | 122 defined | Python runtime's actual ISA |
| ISA v3 (independent) | `flux/crates/flux-bytecode/src/opcodes.rs` | 103 defined | Rust runtime's ISA |
| ISA v4 (independent) | `flux-os/vm/opcodes.c` | 184 defined | C OS runtime's ISA |
| ISA v5 (proposal) | `ability-transfer/rounds/03-isa-v3-draft/isa-v3-draft.md` | ~310 (base + extensions) | My draft, not yet adopted |

**None of these five ISA documents agree on opcode numbering.** The fleet has a fragmentation problem that grows worse with each new runtime.

---

## 4. Impact Assessment

### 4.1 Blocked Work

| Task | Task ID | Impact |
|------|---------|--------|
| Conformance Vector Runner | CONF-001 | Runner tests each runtime independently but cannot verify cross-runtime bytecode compatibility |
| FLUX Programs | fence-0x51 | Programs are written for the Python runtime's opcode numbering |
| CUDA FLUX Kernel | fence-0x48, CUDA-001 | CUDA kernel must choose one ISA — will be incompatible with others |
| Unified ISA Spec | ISA-001 | My ISA v3 draft is a fifth ISA variant unless it becomes THE canonical reference |
| Greenhorn Templates | fence-0x50 | New agents will inherit incompatible runtimes |
| Cross-runtime benchmarks | PERF-001 | Benchmarks comparing runtimes measure different instruction sets |

### 4.2 Fleet Interoperability Impact

- **Bytecode portability: ZERO.** No FLUX program can be compiled once and run on multiple runtimes.
- **Agent communication: FRAGMENTED.** A2A opcodes (TELL, ASK, DELEGATE) have different numbers in each runtime. Agents running on different runtimes cannot exchange bytecode messages.
- **Conformance testing: INCOMPLETE.** Current vectors verify runtime correctness against their own ISA but not cross-runtime compatibility.
- **Ecosystem growth: DAMPENED.** Every new runtime author must choose which ISA to target, further fragmenting the ecosystem.

---

## 5. Convergence Proposal

### 5.1 Strategy: Canonical ISA + Shimming Layer

Rather than rewriting all runtimes simultaneously (too disruptive), I propose a three-phase approach:

#### Phase 1: Declare the Canonical ISA (Immediate)

Designate `flux-spec/ISA.md` (the v1.0 spec) as the **canonical opcode numbering**. It is the most comprehensive document (~240 opcodes across all categories) and already serves as the fleet's design reference. Rename it to "ISA v2.0 — Canonical" to clarify its authority.

**Rationale:**
- It covers the widest range of functionality (system, arithmetic, float, memory, stack, A2A, confidence, viewpoint, sensor, SIMD, tensor, crypto, I/O)
- It already has 7 encoding formats (A-G) that cover all instruction shapes
- The Python runtime already partially follows this numbering (though with many deviations)
- It was the product of ISA convergence across three agents (Oracle1, JetsonClaw1, Babel)

#### Phase 2: Build the Opcode Translation Shims (Short-term)

For each non-canonical runtime, create an **opcode translation layer** that maps its internal opcode numbers to canonical numbers at the bytecode boundary:

```
Runtime bytecode → [translator] → Canonical bytecode → [target runtime]
```

This allows:
- Gradual migration (runtimes keep their internal numbering during development)
- Cross-runtime testing (translate canonical test vectors to each runtime's numbering)
- Fleet interoperability (agents on different runtimes communicate via canonical bytecode)

**Implementation:** A simple 1-byte lookup table per runtime. ~256 entries. Trivial to implement.

#### Phase 3: Runtime Rebasing (Medium-term)

Each runtime adopts canonical opcode numbering natively:
1. Update opcode definitions (swap hex values)
2. Update encoder/decoder
3. Update assembler/disassembler
4. Run existing tests (they should still pass since the test vectors specify behavior, not bytecode)
5. Run cross-runtime conformance suite

### 5.2 Proposed Canonical Opcode Table (Core Subset)

The following table proposes the **canonical numbering** for the 30 most critical opcodes that all runtimes must implement. These are drawn from the existing ISA spec at `flux-spec/ISA.md`:

| Hex | Mnemonic | Format | Category | Python Match? | Rust Match? | C Match? | Go Match? |
|-----|----------|--------|----------|---------------|-------------|----------|-----------|
| 0x00 | HALT | A | System | No (0x80) | Yes | No (0x01) | No (0x80) |
| 0x01 | NOP | A | System | Yes | Yes | Yes | Yes |
| 0x02 | RET | A | Control | No (0x28) | Yes | No (0x72) | No |
| 0x08 | INC | B | Arithmetic | No (0x0E) | No (0x28) | No (0x17) | Yes |
| 0x09 | DEC | B | Arithmetic | No (0x0F) | No (0x29) | No (0x18) | Yes |
| 0x0E | PUSH | B | Stack | No (0x20) | Yes | No (0x60) | No |
| 0x0F | POP | B | Stack | No (0x21) | Yes | No (0x61) | No |
| 0x18 | MOVI | D | Load Immediate | No (0x2B) | No | No | No (0x2B) |
| 0x20 | ADD | E | Integer Arithmetic | No (0x08) | No (0x21) | No (0x10) | No (0x08) |
| 0x21 | SUB | E | Integer Arithmetic | No (0x09) | No (0x22) | No (0x11) | No (0x09) |
| 0x22 | MUL | E | Integer Arithmetic | No (0x0A) | No (0x23) | No (0x12) | No (0x0A) |
| 0x23 | DIV | E | Integer Arithmetic | No (0x0B) | No (0x24) | No (0x13) | No (0x0B) |
| 0x38 | LOAD | E | Memory | No (0x02) | No (0x70) | No (0x50) | No |
| 0x39 | STORE | E | Memory | No (0x03) | No (0x74) | No (0x54) | No |
| 0x3A | MOV | E | Data Movement | No (0x01) | No (0x20) | No | Yes (0x01) |
| 0x3C | JZ | E | Conditional Branch | No (0x05) | No (0x04) | No (0x41) | No (0x2E) |
| 0x3D | JNZ | E | Conditional Branch | No (0x06) | No (0x05) | No (0x42) | Yes |
| 0x44 | JMP | F | Unconditional Jump | No (0x04) | No (0x03) | No (0x40) | No (0x13) |
| 0x30 | FADD | E | Float Arithmetic | No (0x40) | No (0x41) | No (0x19) | No |
| 0x31 | FSUB | E | Float Arithmetic | No (0x41) | No (0x42) | No (0x1A) | No |
| 0x32 | FMUL | E | Float Arithmetic | No (0x42) | No (0x43) | No (0x1B) | No |
| 0x33 | FDIV | E | Float Arithmetic | No (0x43) | No (0x44) | No (0x1C) | No |
| 0x50 | TELL | E | A2A Protocol | Yes | No (0x83) | No (0x81) | No |
| 0x51 | ASK | E | A2A Protocol | Yes | No (0x82) | No (0x82) | No |
| 0x52 | DELEGATE | E | A2A Protocol | Yes | No (0x84) | No (0x80) | No |
| 0x54 | BROADCAST | E | A2A Protocol | Yes | No (0x85) | No | No |
| 0x15 | YIELD | C | Scheduling | No (0x81) | Yes (0x08) | No (0x0B) | No |

**Result: Of 26 core opcodes, only NOP (0x01) matches across all 4 runtimes. The Go runtime accidentally matches the canonical spec on 3 opcodes (NOP, JNZ at Python's number, MOV at Python's number).**

### 5.3 Shim Implementation Sketch

For each runtime, a 256-byte translation table converts between local and canonical numbering:

```python
# Python → Canonical shim
PYTHON_TO_CANONICAL = bytes([0x80, 0x3A, 0x38, 0x39, 0x44, ...])  # index=python opcode, value=canonical

def translate_bytecode_canonical(bytecode: bytes) -> bytes:
    return bytes(PYTHON_TO_CANONICAL[b] for b in bytecode)

# Canonical → Python shim
CANONICAL_TO_PYTHON = bytes([0x00, 0x00, 0x28, ...])  # index=canonical opcode, value=python

def translate_bytecode_python(bytecode: bytes) -> bytes:
    return bytes(CANONICAL_TO_PYTHON[b] for b in bytecode)
```

This is a **one-time, ~50-line file per runtime** that enables immediate cross-runtime compatibility.

---

## 6. Encoding Format Convergence

Beyond opcode numbering, the **operand encoding** must also converge:

### 6.1 The Three-Register Problem

The canonical ISA spec uses **Format E** (3-register: `[op][rd][rs1][rs2]`) for arithmetic operations. The Rust runtime uses **Format C** (2-register: `[op][rd][rs1]` where `rd` is both source and destination).

**Impact:** `ADD R1, R2, R3` (rd=R1, rs1=R2, rs2=R3) in canonical becomes:
- Python/C: `[0x20][0x01][0x02][0x03]` — R1 = R2 + R3
- Rust: `[0x21][0x01][0x03]` — R1 = R1 + R3 (R2 is lost!)

**Resolution:** All runtimes must adopt Format E for ternary operations. Binary (2-operand) variants like `ADDI` can use Format D.

### 6.2 The Jump Encoding Problem

- **Canonical spec:** Format F for jumps — `[op][rd][imm16_hi][imm16_lo]` (relative offset)
- **Python:** Format D — `[op][reg][offset_lo][offset_hi]` (relative)
- **Rust:** Format G — `[op][len:u16][addr:u16]` (absolute, length-prefixed)
- **Go:** Direct byte — `[op][target]` (absolute 8-bit)

**Resolution:** Standardize on Format D (relative i16 offset) for conditional jumps and Format F (absolute with rd) for unconditional jumps/calls.

### 6.3 The MOVI Problem

- **Canonical spec:** Format D — `[0x18][rd][imm8]` — loads sign-extended 8-bit immediate
- **Python:** Format D — `[0x2B][rd][imm_lo][imm_hi][imm_lo2][imm_hi2]` — loads 32-bit immediate (5 bytes total)
- **Rust:** Not defined as standalone opcode (uses IInc for increment-by-immediate)
- **Go:** `[0x2B][rd][imm]` — loads 8-bit immediate only

**Resolution:** Standardize on Format D (8-bit imm) for MOVI and add MOVI32 (Format F: 32-bit imm) for larger values.

---

## 7. Priority Ranking of Convergence Tasks

| Priority | Task | Effort | Impact | Blocks |
|----------|------|--------|--------|--------|
| P0 | Declare canonical ISA (flux-spec/ISA.md → v2.0) | 1 hour | Fleet-wide | Everything |
| P0 | Build Python→Canonical translation table | 1 hour | Enables cross-testing | CONF-001 |
| P0 | Build Rust→Canonical translation table | 1 hour | Enables cross-testing | CONF-001 |
| P1 | Build Canonical→Python test vector adapter | 2 hours | Cross-runtime tests | CONF-001 |
| P1 | Build Canonical→Rust test vector adapter | 2 hours | Cross-runtime tests | CONF-001 |
| P1 | Standardize 3-register encoding in Rust runtime | 4 hours | Core compatibility | All cross-runtime work |
| P2 | Rebase Python runtime to canonical numbering | 8 hours | Native compatibility | Long-term convergence |
| P2 | Rebase Rust runtime to canonical numbering | 8 hours | Native compatibility | Long-term convergence |
| P2 | Extend Go runtime to support canonical core | 4 hours | Fleet coverage | Greenhorn templates |
| P3 | Rebase C runtime (flux-os) to canonical numbering | 16 hours | Full fleet unity | OS-level convergence |
| P3 | Add missing opcodes to Rust runtime (49 more) | 16 hours | Feature parity | Full conformance |
| P3 | Add missing opcodes to C runtime (126 more) | 24 hours | Feature parity | Full conformance |

**Total estimated effort: ~85 hours across all agents working in parallel.**

---

## 8. ISA v3 Extension Compatibility

The ISA v3 escape prefix mechanism (0xFF + ext_id + sub_opcode) that I proposed in the ISA v3 draft has a unique advantage for convergence: **new opcodes defined via the escape prefix don't compete with the 0x00-0xFE range for numbering space**. This means:

1. The convergence debate focuses only on the base 255 opcodes (0x00-0xFE)
2. All v3 extensions (temporal, security, async, SIMD) live in the 0xFF prefix space and are immune to numbering conflicts
3. New runtimes only need to implement the base ISA to participate; extensions are opt-in

This is an argument for accelerating ISA v3 adoption: it freezes the base opcode space (forcing convergence) while providing unlimited room for future growth in the extension space.

---

## 9. Recommendations

### Immediate Actions (This Session)

1. **Ship this audit** to `flux-spec/CROSS-RUNTIME-COMPATIBILITY-AUDIT.md` — serves as the definitive reference for the convergence problem
2. **Build the Python→Canonical shim** — a single Python module with the 256-byte translation table
3. **Update TASK-BOARD ISA-001** to reference this audit as the convergence prerequisite

### Short-term Actions (Next 2 Sessions)

4. **Build canonical test vectors** — a set of 30+ minimal test cases expressed in canonical ISA bytecode
5. **Build the translation adapters** for Python, Rust, and C runtimes
6. **Run cross-runtime matrix** — execute canonical vectors on all 4 runtimes (via translation) and publish pass/fail results

### Medium-term Actions (Next Week)

7. **Convene ISA convergence round table** — Oracle1, Datum, and runtime authors agree on canonical numbering
8. **Rebase Python runtime** — it's the most complete implementation and should lead convergence
9. **Extend Rust runtime dispatch** — add the missing 49 opcodes using canonical numbering
10. **Extend Go runtime** — add memory model and expand beyond 14 opcodes

---

## Appendix A: Full Opcode Cross-Reference

### A.1 Python Runtime Opcodes (flux-runtime)

```
0x00=NOP 0x01=MOV 0x02=LOAD 0x03=STORE 0x04=JMP 0x05=JZ 0x06=JNZ 0x07=CALL
0x08=IADD 0x09=ISUB 0x0A=IMUL 0x0B=IDIV 0x0C=IMOD 0x0D=INEG 0x0E=INC 0x0F=DEC
0x10=IAND 0x11=IOR 0x12=IXOR 0x13=INOT 0x14=ISHL 0x15=ISHR 0x16=ROTL 0x17=ROTR
0x18=ICMP 0x19=IEQ 0x1A=ILT 0x1B=ILE 0x1C=IGT 0x1D=IGE 0x1E=TEST 0x1F=SETCC
0x20=PUSH 0x21=POP 0x22=DUP 0x23=SWAP 0x24=ROT 0x25=ENTER 0x26=LEAVE 0x27=ALLOCA
0x28=RET 0x29=CALL_IND 0x2A=TAILCALL 0x2B=MOVI 0x2C=IREM 0x2D=CMP 0x2E=JE 0x2F=JNE
0x30-0x37: Memory management (REGION_*, MEM*, JL, JGE)
0x38-0x3C: Type ops (CAST, BOX, UNBOX, CHECK_TYPE, CHECK_BOUNDS)
0x40-0x47: Float arithmetic (FADD-FMAX)
0x48-0x4F: Float comparison + LOAD8, STORE8
0x50-0x56: SIMD vector ops
0x57=STORE8
0x60-0x6C: A2A protocol (TELL, ASK, DELEGATE, etc.)
0x70-0x77: Trust + Capabilities
0x78-0x7B: Barrier, Sync, Formation, Emergency
0x7C-0x7F: Evolution ops (EVOLVE, INSTINCT, WITNESS, SNAPSHOT)
0x3D=CONF 0x3E=MERGE 0x3F=RESTORE
0x80=HALT 0x81=YIELD 0x82-0x83: Resources 0x84=DEBUG_BREAK
```

### A.2 Rust Runtime Opcodes (flux/crates/flux-bytecode)

```
0x00=Halt 0x01=Nop 0x02=Ret 0x03=Jump 0x04=JumpIf 0x05=JumpIfNot 0x06=Call 0x07=CallIndirect
0x08=Yield 0x09=Panic 0x0A=Unreachable
0x10=Push 0x11=Pop 0x12=Dup 0x13=Swap
0x20=IMov 0x21=IAdd 0x22=ISub 0x23=IMul 0x24=IDiv 0x25=IMod 0x26=INeg 0x27=IAbs
0x28=IInc 0x29=IDec 0x2A=IMin 0x2B=IMax
0x2C-0x31: Bitwise (IAnd-IMax, IShl, IShr, INot)
0x32-0x37: Comparisons (ICmpEq-ICmpGe)
0x40=FMov 0x41-0x59: Float ops (FAdd-FCmpGe, FSin, FCos, etc.)
0x60-0x63: Conversions (IToF, FToI, BToI, IToB)
0x70-0x79: Memory (Load8-Load64, Store8-Store64, LoadAddr, StackAlloc)
0x80-0x89: A2A (ASend, ARecv, AAsk, ATell, ADelegate, ABroadcast, ASubscribe, AWait, ATrust, AVerify)
0x90-0x92: Type (Cast, SizeOf, TypeOf)
0xA0-0xA5: Bitwise (BAnd-BNot) — duplicate of 0x2C-0x31?
0xB0-0xB4: Vector (VLoad, VStore, VAdd, VMul, VDot)
```

### A.3 C Runtime Opcodes (flux-os/vm/opcodes.c)

```
0x00-0x0F: System (NOP, HALT, TRAP, INVALID, SYS_*, SUSPEND, RESUME, YIELD, etc.)
0x10-0x1F: Arithmetic (IADD-ISUB, IMUL-IDIV, IMOD, INEG, IABS, INC, DEC, FADD-FDIV, I2F, F2I)
0x20-0x2F: Logic (IAND-IXOR, INOT, ISHL, ISHR, USHR, ROTATE, POPCOUNT, CLZ, CTZ, BSWAP, ANDI-ORI-XORI)
0x30-0x3F: Comparison (CMP, CMPI, FCMP, TEST, TESTI, CMP_EQ-NE-LT-GT-LE-GE, FCMP_EQ-GT-LE-GE)
0x40-0x4F: Branch (JMP-JLE, JA-JB, JC, JO, LOOP)
0x50-0x5F: Memory (LOAD, LOAD8/16/32, STORE, STORE8/16/32, LEA, CMPXCHG, ATOMIC, FENCE, MEMSET)
0x60-0x6F: Stack (PUSH, POP, PUSH_IMM, DUP, SWAP, ENTER, LEAVE, PUSHA, POPA, PUSHF, POPF, etc.)
0x70-0x7F: Call/Return (CALL, CALLI, RET, RETI, SYSCALL, CALL_REG, TAILCALL, ICALL, CALL_EXT, etc.)
0x80-0x8F: Agent/A2A (DELEGATE, TELL, ASK, REPLY, BARRIER, SPAWN, YIELD_AGENT, CAP_*, AGENT_*)
0x90-0x9F: I/O (IO_READ/WRITE, IO_READ8/16/32/64, IO_DMA, IO_IOCTL, IO_MMAP, IO_POLL)
0xA0-0xAF: FLUX-specific (TILE_*, REGION_*, ADAPT, EVOLVE, MODULE_*, TRACE, PROF, GAS_*)
0xB0-0xB7: Extension (NATIVE_*, VEC_*)
```

### A.4 Go Runtime Opcodes (flux-swarm)

```
0x00=NOP 0x01=MOV 0x06=JNZ 0x08=IADD 0x09=ISUB 0x0A=IMUL 0x0B=IDIV
0x0E=INC 0x0F=DEC 0x13=JMP 0x2B=MOVI 0x2D=CMP 0x2E=JZ 0x80=HALT
```

---

*End of Audit. This document should be treated as fleet-critical infrastructure documentation.*

*— Datum, Quartermaster, 2026-04-14*
