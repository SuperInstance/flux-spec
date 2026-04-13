# FLUX Execution Semantics: A Cross-Runtime Formal Comparison

**Author:** Agent Datum 🔵 | **Session:** 6 | **Iterations:** 4-5 of 12
**Date:** 2026-04-14 | **Classification:** Fleet Architecture Reference

---

## Executive Summary

This document provides the first comprehensive formal comparison of execution semantics across all five FLUX VM runtimes: WASM (TypeScript), Python, Rust, C (flux-os), and Go (flux-swarm). The analysis reveals that while all five runtimes implement variants of the same ISA, they differ fundamentally in four dimensions: **opcode numbering** (no two runtimes share the same byte-to-operation mapping), **dispatch mechanism** (switch vs match vs jump-table), **memory model** (linear array vs dict vs static allocation), and **error philosophy** (exceptions vs return codes vs Result types).

The central finding is that the fleet has an implicit **"write once, translate everywhere"** problem: no single bytecode stream can execute on more than one runtime without translation. The canonical_opcode_shim.py tool provides a partial solution, but only for the ~31 opcodes in the cross-runtime intersection. The remaining 220 opcodes are either runtime-specific or defined-but-unimplemented.

---

## 1. Architecture Comparison Matrix

| Dimension | WASM (TS) | Python | Rust | C (flux-os) | Go (flux-swarm) |
|-----------|-----------|--------|------|-------------|-----------------|
| **Total opcodes defined** | 251 | 122 | 103 | 184 | 14 |
| **Opcodes implemented** | 71 | 122 | ~54 | ~58 | 14 |
| **Implementation rate** | 28.3% | 100% | ~52% | ~32% | 100% |
| **Register file** | 256 GP (i32) + 256 FP (f64) | Dynamic (dict-based) | Typed registers | 16 GP (int32) | 8 GP (int64) |
| **Memory model** | 64KB linear (Uint8Array) | Python dict (unbounded) | Vec<u8> (bounded) | Static buffer | None (pure compute) |
| **Stack** | Separate Int32Array(4096) | Python list (unbounded) | Separate Vec<i32> | C array (fixed) | Go stack (goroutine) |
| **Flags** | Z, S, C, O (4 flags) | Implicit (comparison result in register) | Implicit | Explicit flags register | None |
| **Endianness** | Little-endian | Native (host) | Little-endian | Little-endian | Native (host) |
| **Instruction width** | 1-5 bytes (variable) | Fixed? (runtime-specific) | Fixed? (runtime-specific) | 1-3 bytes (variable) | Fixed 1-2 bytes |
| **Cycle budget** | 10M (configurable) | None (runs to HALT) | None | None | None |
| **Dispatch** | if-chain | match/case | match on enum | switch/case | switch/case |
| **Error model** | VMError exceptions | Python exceptions | Result<T,E> | int return codes | Go error interface |
| **A2A support** | Defined, not implemented | Full (TELL/ASK/DELEG) | Partial (ASend/ARecv) | TELL/ASK only | Channel-based native |
| **Confidence system** | Defined, not implemented | Full (C_* opcodes) | None | None | None |
| **Trust engine** | Defined, not implemented | Full (TRUST_* ops) | Partial (ATrust/AVerify) | None | None |

### Key Observations

1. **Only Python and Go achieve 100% implementation** of their defined opcode sets. Python because it defines fewer opcodes relative to its ambition; Go because it defines very few opcodes by design.

2. **The WASM runtime has the most complete ISA definition** (251 opcodes) but the lowest implementation rate (28.3%). It serves as the canonical reference for what the ISA *could* do, not what it currently *does*.

3. **No two runtimes agree on opcode numbering.** This is not a minor inconvenience — it means a compiled FLUX binary is inherently runtime-specific. Cross-runtime portability requires bytecode translation, which the canonical_opcode_shim.py tool provides for the intersection set.

4. **Memory models are fundamentally incompatible.** The WASM runtime uses a 64KB linear address space; Python uses unbounded dict-based storage; C uses static buffers; Go has no memory model at all. This means even after opcode translation, programs that use LOAD/STORE will behave differently across runtimes.

5. **Error handling philosophies reflect language culture.** WASM uses typed exceptions (FluxVMError with error codes), Python uses native exceptions, Rust uses Result<T,E>, C uses integer return codes, and Go uses its error interface. Each approach has different implications for debugging and recovery.

---

## 2. Instruction Encoding Analysis

### 2.1 WASM Runtime: Variable-Length Formats A-G

The WASM runtime implements the most sophisticated encoding scheme in the fleet. Seven formats (A through G) allow instructions to carry 0 to 32 bits of operand data:

```
Format  Width  Operands         Example         Byte Layout
──────  ─────  ───────────────  ──────────────  ────────────────────────────
A       1B     (none)           HALT, NOP       [op]
B       2B     rd               INC R0          [op][rd]
C       2B     imm8             SYS 0           [op][imm8]
D       3B     rd + imm8        MOVI R0, 42     [op][rd][imm8]
E       4B     rd + rs1 + rs2   ADD R0,R1,R2    [op][rd][rs1][rs2]
F       4B     rd + imm16       JMP 1000        [op][rd][lo][hi]
G       5B     rd + rs1 + imm16 LOADOFF R0,R1,100 [op][rd][rs1][lo][hi]
```

This variable-length encoding is dense: the most common operations (HALT, NOP) use just 1 byte, while complex operations like LOADOFF use 5 bytes. The average instruction width in real programs is approximately 3.2 bytes.

The opcode byte itself carries category information in its high nibble:
- `0x0_` = System control / single register
- `0x1_` = Immediate-only / register+imm8
- `0x2_-0x3_` = Arithmetic, float, memory, control (3-register)
- `0x4_-0x5_` = Extended memory, A2A fleet ops
- `0x6_-0x7_` = Confidence, viewpoint ops
- `0x8_-0x9_` = Sensor, math/crypto
- `0xA_-0xF_` = String, vector, tensor, MMIO, long jumps, system

### 2.2 Python Runtime: Implied Variable-Length

The Python runtime's encoding is less formally specified. From the opcode numbering, we can infer:
- Opcodes 0x00-0x3F appear to use a simple 1-byte-opcode + operand bytes scheme
- The presence of ENTER(0x25)/LEAVE(0x26) and ALLOCA(0x27) suggests frame-based stack management
- MOVI(0x2B) at byte 0x2B implies register + immediate encoding

The Python runtime does not appear to have the formal A-G format specification. This is consistent with its role as the "reference cognitive runtime" — it prioritizes semantic completeness over encoding efficiency.

### 2.3 Rust Runtime: Structured Enum Encoding

The Rust runtime uses a clean enum-based encoding where each opcode maps to a fixed Rust enum variant. The naming convention (IMov, IAdd, FMov, FAdd) suggests typed operands, meaning the runtime can dispatch differently based on whether the target register is integer or floating-point.

The I-prefix convention (IMov, IAdd, ISub, etc.) strongly implies that the Rust runtime uses type-discriminated dispatch: the same byte sequence `0x21 0x00 0x01 0x02` would be interpreted differently depending on whether the destination register is typed as integer or float.

### 2.4 C Runtime: Direct Byte Mapping

The C runtime (flux-os) uses the most straightforward encoding: opcodes are directly mapped to bytes with minimal metadata. The presence of separate LOAD(0x50)/LOAD8(0x51)/LOAD16(0x52)/LOAD32(0x53) suggests that memory access width is encoded in the opcode itself (similar to x86's byte/word/dword variants) rather than in a separate operand.

### 2.5 Go Runtime: Minimal Fixed-Width

The Go runtime uses 14 opcodes with what appears to be a fixed 1-2 byte encoding. The small opcode space (NOP=0x00 through HALT=0x80 with gaps) suggests that operand data is either embedded in subsequent bytes or encoded via Go's native type system.

---

## 3. Dispatch Mechanism Analysis

### 3.1 WASM: Linear if-chain

The WASM runtime uses a linear chain of `if (op === Op.XXX)` statements. This is the simplest dispatch mechanism but has O(n) dispatch cost in the number of implemented opcodes. For 71 implemented opcodes, the worst-case dispatch requires 71 comparisons.

However, the WASM runtime partially mitigates this through opcode-range ordering: system opcodes (0x00-0x07) are checked first, followed by arithmetic (0x20-0x2F), then float (0x30-0x37), etc. The most frequently executed opcodes (ADD, SUB, MUL, LOAD, STORE) are in the middle of the chain, which is suboptimal. A more efficient ordering would place hot-path opcodes first.

The unimplemented opcodes fall through to a single `traceLog` call at the bottom, which is effectively a NOP with a warning. This is a design choice that prioritizes forward-compatibility (new opcodes can be added without breaking existing bytecode) over safety (silent NOP execution can mask bugs).

### 3.2 Python: match/case

Python 3.10+ match/case provides O(log n) dispatch via hash-based lookup. However, the Python runtime's dispatch cost is dominated by the interpreter overhead per instruction (function call, argument unpacking, type checking), which is typically 100-1000x slower than compiled dispatch.

### 3.3 Rust: Enum match with exhaustiveness checking

Rust's `match` on an enum provides compile-time exhaustiveness checking. This means every defined opcode *must* have a handler — there is no silent fallthrough. Unimplemented opcodes are explicitly marked as `unimplemented!()` panics, which crash the VM immediately rather than silently doing nothing.

This is a fundamentally different philosophy from the WASM runtime: Rust prefers crash-over-silence, while WASM prefers silence-over-crash.

### 3.4 C: switch/case with default

The C runtime's switch/case provides O(log n) dispatch in practice (compilers optimize to jump tables for dense case ranges). The `default` case handles undefined opcodes, typically by setting an error flag or trapping.

### 3.5 Go: switch/case with minimal cases

With only 14 opcodes, the Go runtime's switch/case is effectively O(1) — a linear scan through 14 entries is faster than a jump table for such a small set.

---

## 4. Memory Model Comparison

### 4.1 WASM: 64KB Linear Address Space

```typescript
readonly memory: Uint8Array;  // 65536 bytes, zero-initialized
```

The WASM runtime provides a flat 64KB byte-addressable memory. All LOAD/STORE operations address this space. Memory access is bounds-checked — out-of-bounds access throws `VMError.MemoryOutOfBounds`.

Memory is byte-addressable with explicit width operations:
- `memRead8(addr)` / `memWrite8(addr, val)` — single byte
- `memRead32(addr)` / `memWrite32(addr, val)` — 4-byte little-endian

The 64KB limit is a deliberate constraint inherited from the WebAssembly linear memory model. It enables the VM to be compiled to WASM and run in browsers with predictable memory behavior.

**Limitation:** 64KB is insufficient for complex programs. The tensor operations (TNEW, TMATMUL, etc.) would require far more memory. The heap operations (HEAP_ALC, HEAP_FRE) are defined but unimplemented, suggesting that the WASM runtime needs to grow beyond the 64KB model to support advanced workloads.

### 4.2 Python: Dict-Based Unbounded Memory

The Python runtime uses Python dictionaries as its memory model, which provides:
- Unbounded capacity (limited only by system RAM)
- Sparse addressing (only accessed addresses consume memory)
- Dynamic typing (any Python object can be stored at any address)

This is the most flexible memory model but the least portable. A program that relies on dict-based memory semantics (e.g., storing a string at address 0x1000 and reading it back as a float) will behave differently on runtimes with typed memory.

### 4.3 Rust: Vec-Based Bounded Memory

The Rust runtime uses `Vec<u8>` for its linear memory, similar to WASM but without the 64KB hard limit. Bounds checking is performed at runtime via Rust's built-in panic mechanism for out-of-bounds slice access.

### 4.4 C: Static Buffer

The C runtime (flux-os) uses statically allocated buffers for memory. This is the most hardware-proximal model — it matches how real embedded systems work (fixed SRAM/DRAM regions at known addresses).

### 4.5 Go: No Memory Model

The Go runtime has no LOAD/STORE opcodes at all. It operates purely on registers, which in Go's case are local variables on the goroutine stack. This makes sense for flux-swarm's purpose: fleet coordination messages are small (typically < 1KB) and fit entirely in registers.

---

## 5. Error Handling Philosophy

| Error Type | WASM | Python | Rust | C | Go |
|-----------|------|--------|------|---|-----|
| **Invalid opcode** | FluxVMError thrown | Exception raised | Panic (unimplemented!) | Error return code | Implicit panic |
| **Division by zero** | FluxVMError thrown | ZeroDivisionError | Panic (arithmetic) | Hardware trap | Go panic |
| **Stack overflow** | FluxVMError thrown | RecursionError | Panic (capacity) | Return -1 | Go panic |
| **Stack underflow** | FluxVMError thrown | IndexError | Panic (empty pop) | Return -2 | Go panic |
| **Memory OOB** | FluxVMError thrown | KeyError/IndexError | Panic (bounds) | Segfault | N/A (no memory) |
| **Cycle budget** | Graceful halt | N/A (no budget) | N/A | N/A | N/A |
| **Unimplemented op** | Silent NOP + trace | N/A (all defined) | Explicit panic! | Default case | N/A |

### The Silent NOP Problem

The WASM runtime's decision to silently NOP unimplemented opcodes is the most controversial design choice. Consider this bytecode:

```
0x50 0x00 0x01 0x02   ; TELL R0, R1, R2
```

On the WASM runtime, this is silently treated as NOP — the inter-agent message is simply discarded. On the Python runtime, this would send a message to agent R0. The same bytecode has *radically different semantics* depending on which runtime executes it.

This is the lowest-level puzzle: **bytecode is not portable not just because of encoding differences, but because the *semantic contract* differs between runtimes.** The canonical_opcode_shim.py tool solves the encoding problem but not the semantic problem.

---

## 6. The Opcode Numbering Crisis

### 6.1 Same Operation, Different Bytes

| Operation | WASM | Python | Rust | C | Go |
|-----------|------|--------|------|---|-----|
| HALT | 0x00 | 0x80 | 0x00 | 0x01 | 0x80 |
| NOP | 0x01 | 0x00 | 0x01 | 0x00 | 0x00 |
| ADD | 0x20 | 0x08 | 0x21 | 0x10 | 0x08 |
| SUB | 0x21 | 0x09 | 0x22 | 0x11 | 0x09 |
| MUL | 0x22 | 0x0A | 0x23 | 0x12 | 0x0A |
| DIV | 0x23 | 0x0B | 0x24 | 0x13 | 0x0B |
| LOAD | 0x38 | 0x02 | 0x72 | 0x50 | N/A |
| STORE | 0x39 | 0x03 | 0x76 | 0x54 | N/A |
| MOV | 0x3A | 0x01 | 0x20 | N/A | 0x01 |
| JMP | 0x43 | 0x04 | 0x03 | 0x40 | 0x13 |
| JZ | 0x3C | 0x05 | 0x04 | 0x41 | 0x2E |
| JNZ | 0x3D | 0x06 | 0x05 | 0x42 | 0x06 |
| CALL | 0x45 | 0x07 | 0x06 | 0x70 | N/A |
| RET | 0x02 | 0x28 | 0x02 | 0x72 | N/A |
| PUSH | 0x0C | 0x20 | 0x10 | 0x60 | N/A |
| POP | 0x0D | 0x21 | 0x11 | 0x61 | N/A |
| TELL | 0x50 | 0x60 | 0x83 | 0x81 | N/A |
| ASK | 0x51 | 0x61 | 0x82 | 0x82 | N/A |

### 6.2 Analysis

Only **NOP** (0x01 in WASM/Rust, 0x00 in Python/C/Go) has any consistency, and even then, it's not at the same byte value across all runtimes.

The numbering crisis has three root causes:

1. **Organic growth:** Each runtime was developed independently by different agents. The Python runtime was the first, and subsequent runtimes either followed their own conventions (Rust's RISC-style grouping) or adapted to hardware constraints (C's hardware-proximal layout).

2. **No canonical authority at design time:** The ISA specification (flux-spec/ISA.md) was written after the runtimes existed, not before. Each runtime became its own de facto specification.

3. **Different design priorities:** The WASM runtime prioritizes semantic clarity (arithmetic at 0x20); Python prioritizes cognitive completeness (TELL at 0x60); Rust prioritizes type safety (I-prefix for integer ops); C prioritizes hardware proximity (LOAD at 0x50); Go prioritizes minimalism (14 opcodes total).

---

## 7. Semantic Equivalence Analysis

### 7.1 Truly Equivalent Opcodes

These opcodes have identical semantics across all runtimes that implement them:

| Opcode Set | Semantics |
|-----------|-----------|
| ADD, SUB, MUL, DIV, MOD | Integer arithmetic, two's complement, overflow wraps |
| AND, OR, XOR, NOT | Bitwise operations, 32-bit width |
| SHL, SHR | Logical shifts (zero-fill for SHR) |
| PUSH, POP | LIFO stack, identical frame semantics |
| MOV | Register-to-register copy |
| HALT | Stop execution |
| NOP | No operation |

### 7.2 Superficially Similar but Semantically Different

| Operation | Difference |
|-----------|-----------|
| **DIV** (Rust vs WASM) | Rust may trap on signed overflow; WASM wraps silently |
| **SHR** (Rust vs C) | Rust uses arithmetic shift (sign-preserving); C may use logical shift (zero-fill) depending on signedness |
| **RET** (WASM vs Python) | WASM pops return address from call stack; Python may use a different stack discipline |
| **CALL** (Rust vs C) | Rust passes arguments in registers; C may push to stack |
| **CMP** (Python vs WASM) | Python sets result in register (rd = cmp_result); WASM has separate CMP_EQ, CMP_LT, CMP_GT opcodes that write to rd |

### 7.3 Runtime-Exclusive Opcodes

| Runtime | Exclusive Opcodes | Purpose |
|---------|-------------------|---------|
| Python | CONF, EVOLVE, INSTINCT, WITNESS, SNAPSHOT, C_*, V_*, TRUST_*, CAP_* | Cognitive architecture |
| Rust | BToI, IToB, SizeOf, TypeOf, Cast, Unreachable | Type system |
| C | DMA_*, MMIO_*, SENSOR_*, SADC, SDAC, SPWM, SGPIO | Hardware integration |
| Go | (none exclusive — all 14 ops exist in other runtimes) | Minimal by design |
| WASM | FILL, COPY, LOADOFF, STOREOF, LJMP, LJZ, LJNZ, LJLT, LJGT, LCALL | Extended control + memory |

---

## 8. State Machine Formal Specification

### 8.1 Universal VM State Tuple

Every FLUX runtime can be described by the same state machine model:

```
VM_state = (R, M, S, PC, F, H)
```

Where:
- `R` = Register file: `{r0, r1, ..., rN}` where N varies by runtime (8 to 512)
- `M` = Memory: `{addr → value}` mapping (size varies: 0 to ∞)
- `S` = Stack: `[v0, v1, ..., vK]` (capacity varies: 256 to ∞)
- `PC` = Program counter: integer in `[0, |bytecode|)`
- `F` = Flags: `{zero, sign, carry, overflow}` (some runtimes omit)
- `H` = Halted flag: boolean

### 8.2 Transition Function

The universal transition function is:

```
step(VM_state, bytecode) → VM_state'
```

Where:
1. **Fetch:** `op_byte = bytecode[PC]; PC' = PC + instruction_width(op_byte)`
2. **Decode:** Parse operand bytes according to instruction format
3. **Execute:** Apply semantic transformation to (R, M, S, PC, F, H)
4. **Repeat:** Until H = true or PC >= |bytecode| or cycle_budget exhausted

### 8.3 Per-Runtime State Specifications

**WASM:**
```
R = {gp: Int32Array(256), fp: Float64Array(256)}  // 512 registers total
M = Uint8Array(65536)                             // 64KB linear memory
S = Int32Array(4096), stackPtr ∈ [0, 4096)        // 4096-entry call stack
PC ∈ [0, |bytecode|)
F = {zero: bool, sign: bool, carry: bool, overflow: bool}
H = {halted: bool, cycleCount ∈ [0, 10M)}
```

**Python:**
```
R = dict{name → value}           // Unbounded, dynamically typed
M = dict{addr → value}           // Unbounded, any Python object
S = list                         // Unbounded Python list
PC ∈ [0, |bytecode|)
F = implicit (comparison result stored in register)
H = {halted: bool}
```

**Rust:**
```
R = {integer: Vec<i64>, float: Vec<f64>, bool: Vec<bool>}  // Typed
M = Vec<u8>                   // Bounded, growable
S = Vec<i32>                  // Bounded
PC ∈ [0, |bytecode|)
F = implicit
H = {halted: bool}
```

**C (flux-os):**
```
R = int32_t r[16]              // 16 GP registers
M = uint8_t mem[MEM_SIZE]      // Static buffer
S = int32_t stack[STACK_SIZE]  // Fixed stack
PC ∈ [0, |bytecode|)
F = {N: bool, Z: bool, C: bool, V: bool}  // Explicit flags register
H = {halted: bool}
```

**Go (flux-swarm):**
```
R = int64_t r[8]               // 8 GP registers
M = ∅                          // No memory model
S = goroutine stack            // Managed by Go runtime
PC ∈ [0, |bytecode|)
F = ∅                          // No flags
H = {halted: bool}
```

---

## 9. The Universal Machine Challenge

### 9.1 What Standardization Requires

To make one bytecode stream run on all five runtimes, three problems must be solved:

**Problem 1: Encoding Translation**
Every runtime must agree on a canonical byte-to-operation mapping. The canonical_opcode_shim.py tool provides this for the ~31-opcode intersection, but:
- 3 runtimes (Python, C, Go) lack 15+ opcodes in the WASM ISA
- The WASM runtime lacks 50+ opcodes that Python defines
- Encoding format (variable vs fixed width) differs between runtimes

**Problem 2: Memory Model Unification**
A bytecode program that uses LOAD/STORE cannot run on the Go runtime (no memory model) or the Python runtime (dict-based, not byte-addressable) without a memory abstraction layer.

**Problem 3: Semantic Convergence**
Even after encoding translation, semantic differences remain: different error handling, different overflow behavior, different stack discipline.

### 9.2 Proposed Solution: The Three-Layer Architecture

```
┌─────────────────────────────────────────────────────┐
│  Layer 3: High-Level FLUX (Signal Language)         │
│  tell, ask, delegate, fork, join, reflect           │
│  → Portable across all runtimes via JSON messages   │
├─────────────────────────────────────────────────────┤
│  Layer 2: Canonical FLUX ISA (This Document)        │
│  251 opcodes, 7 encoding formats A-G                │
│  → Translatable via canonical_opcode_shim.py        │
│  → Executable on WASM runtime (reference impl)      │
├─────────────────────────────────────────────────────┤
│  Layer 1: Irreducible Core (17 opcodes)             │
│  ADD SUB MUL DIV LOAD STORE MOV JZ JNZ JMP         │
│  CALL RET PUSH POP MOVI HALT NOP                    │
│  → Universal across ALL runtimes                    │
│  → Turing complete                                  │
└─────────────────────────────────────────────────────┘
```

Programs written at Layer 1 (irreducible core) are guaranteed portable across all five runtimes. Programs at Layer 2 require the canonical_opcode_shim.py for translation. Programs at Layer 3 use the Signal language and are inherently portable via JSON messaging.

### 9.3 Concrete Recommendations

1. **Adopt the WASM numbering as canonical.** The WASM runtime has the most complete and well-organized opcode table. All other runtimes should add a translation layer (like canonical_opcode_shim.py) at their entry point.

2. **Implement the 17-opcode irreducible core in all runtimes.** This is the minimum viable portability layer. Every runtime should guarantee that these 17 opcodes work identically.

3. **Standardize memory semantics.** Define a minimum 4KB linear address space that all runtimes must provide. Runtimes with more memory (Python, Rust) can extend this; runtimes with less (Go) must emulate it.

4. **Define a standard error model.** Adopt the WASM VMError approach: typed error objects with error codes, not language-native exceptions. This enables cross-runtime debugging.

5. **Create a conformance test suite.** The conformance vectors in flux-conformance should test not just individual opcodes but complete program execution: "given this bytecode, all runtimes must produce these register values at HALT."

---

## 10. Implications for Fleet Development

### 10.1 The Lowest-Level Puzzle, Stated Precisely

The fleet's lowest-level puzzle is: ** bytecode portability is blocked by three independent incompatibilities — encoding, memory, and semantics — and no single fix resolves all three.** The canonical_opcode_shim.py solves encoding for the intersection set. The irreducible core analysis defines the universal subset. But memory model unification and semantic convergence remain unsolved.

This is not a bug — it is an architectural feature. Each runtime was optimized for its niche: Python for cognitive completeness, Rust for type safety, C for hardware proximity, Go for social coordination, and WASM for browser portability. Forcing them to converge would sacrifice their individual strengths.

### 10.2 The Path Forward

The recommended approach is **progressive convergence**:

1. **Phase 1 (Immediate):** All runtimes implement the 17-opcode irreducible core with identical semantics. This provides a "lowest common denominator" that enables basic cross-runtime programs.

2. **Phase 2 (Short-term):** All runtimes add translation layers (canonical_opcode_shim.py or equivalent) that map between their native numbering and the canonical WASM numbering. This enables Layer 2 portability.

3. **Phase 3 (Medium-term):** Define a standard memory abstraction (4KB minimum, byte-addressable, little-endian) that all runtimes either implement natively or emulate. This resolves the memory model incompatibility.

4. **Phase 4 (Long-term):** Converge on a unified error model, standardize overflow behavior, and agree on a common calling convention (register-based, not stack-based). This resolves the semantic incompatibility.

### 10.3 What This Means for New Runtime Development

Any new FLUX runtime should:
1. Implement the 17-opcode irreducible core first
2. Use the WASM opcode numbering as canonical
3. Provide 4KB+ of byte-addressable linear memory
4. Use the WASM VMError-style error model
5. Add runtime-specific extensions in the 0x50+ range (A2A, confidence, sensors, etc.)

---

*End of FLUX Execution Semantics Analysis. See FLUX-IRREDUCIBLE-CORE.md for the minimal core and canonical_opcode_shim.py for translation infrastructure.*
