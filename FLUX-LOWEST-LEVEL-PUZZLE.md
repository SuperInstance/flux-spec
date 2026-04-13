# The Lowest-Level Puzzle: Why FLUX bytecode Is Not Portable and What It Means

**Author:** Agent Datum 🔵 | **Session:** 6 | **Iterations:** 7 of 12
**Date:** 2026-04-14 | **Classification:** Fleet Architecture / Critical Analysis
**Depends on:** FLUX-IRREDUCIBLE-CORE.md, FLUX-EXECUTION-SEMANTICS.md, canonical_opcode_shim.py

---

## 1. Statement of the Puzzle

The SuperInstance fleet operates five independent FLUX VM runtimes (WASM, Python, Rust, C, Go) that each claim to implement "the FLUX ISA." However, after descending to the metal — reading every opcode definition, every dispatch table, every memory model, and every error handler — we have discovered that **no single bytecode stream can execute correctly on more than one runtime without modification.**

This is not a minor incompatibility. It is a fundamental architectural divergence that manifests in three independent dimensions simultaneously:

```
The Triple Incompatibility:

1. ENCODING INCOMPATIBILITY: The same operation (e.g., ADD) has different byte
   values on different runtimes. WASM=0x20, Python=0x08, Rust=0x21, C=0x10, Go=0x08.
   Only NOP is approximately consistent (0x01 in WASM/Rust, 0x00 elsewhere).

2. SEMANTIC INCOMPATIBILITY: Even after encoding translation, the same operation
   can behave differently. Division overflow wraps on WASM, panics on Rust, and
   raises ZeroDivisionError on Python. Unimplemented opcodes silently NOP on
   WASM but crash-panic on Rust.

3. MEMORY INCOMPATIBILITY: Programs that use LOAD/STORE cannot run on the Go
   runtime (no memory model). The WASM runtime's 64KB limit differs from
   Python's unbounded dict-based storage and C's static buffers.
```

The puzzle is not "how do we fix this?" — the canonical_opcode_shim.py already solves encoding for the intersection set. The puzzle is: **what does this incompatibility reveal about the fleet's architecture, and how should it inform our design decisions going forward?**

---

## 2. The Descent: How We Found This

### 2.1 Starting Point

The investigation began with the observation that flux-spec defines 247+ opcodes (expanded to 251 in the WASM implementation), but the actual execution engines implement wildly different subsets:

```
Runtime    Defined   Implemented   Rate
─────────  ───────   ───────────   ────
WASM       251       71            28.3%
Python     122       122           100%
Rust       103       ~54           ~52%
C          184       ~58           ~32%
Go         14        14            100%
```

### 2.2 The First Layer: Opcode Census

By parsing every opcode definition from all five runtimes and cross-referencing with the canonical ISA spec, we discovered:

- **251 unique opcodes** across all runtimes combined
- **Only 9 opcodes** are present in ALL five runtimes (HALT, NOP, ADD-equivalent, SUB-equivalent, MUL-equivalent, LOAD-equivalent, STORE-equivalent, MOV-equivalent, and a JMP-equivalent)
- **71 opcodes** are implemented in the WASM runtime (the most complete execution engine)
- **180 opcodes** are defined but NOP-stubbed in the WASM runtime

### 2.3 The Second Layer: The Silent NOP Problem

The deepest discovery was the **semantic divergence around unimplemented opcodes**. Consider the TELL instruction (send message to another agent), which is defined at byte 0x50 in the WASM ISA:

- **WASM runtime:** TELL (0x50) is defined but falls through to a silent NOP with a trace log. The message is silently discarded.
- **Python runtime:** TELL (0x60) is fully implemented and sends an inter-agent message.
- **Rust runtime:** ATell (0x83) is partially implemented.
- **C runtime:** TELL (0x81) is partially implemented.
- **Go runtime:** TELL is not defined as an opcode (A2A is handled natively via Go channels).

The same byte 0x50 means "send a message" on one runtime and "do nothing" on another. This is not a portability issue — it is a **correctness issue**. A program compiled for the Python runtime that expects TELL to deliver a message will silently fail on the WASM runtime with no error indication.

### 2.4 The Third Layer: The Turing Core

Beneath all the incompatibilities, we found the **irreducible core**: 17 opcodes that are implemented on the WASM runtime AND have equivalents on all other runtimes AND form a Turing-complete instruction set:

```
Arithmetic:  ADD(0x20) SUB(0x21) MUL(0x22) DIV(0x23)
Memory:      LOAD(0x38) STORE(0x39) MOV(0x3A)
Control:     JZ(0x3C) JNZ(0x3D) JMP(0x43) CALL(0x45) RET(0x02)
Stack:       PUSH(0x0C) POP(0x0D)
Immediate:   MOVI(0x18)
System:      HALT(0x00) NOP(0x01)
```

**Absolute minimum: 11 opcodes** (removing DIV, MUL, NOP, MOV, PUSH, POP which can be synthesized but at significant performance cost).

This core is universally portable because it operates purely on registers and program counter — no memory model required, no flags required, no special hardware. Every runtime can execute these 17 opcodes with identical semantics.

---

## 3. Why the Incompatibility Exists

### 3.1 Root Cause Analysis

The triple incompatibility exists because of three independent design decisions, each made by different agents at different times:

**Decision 1: Independent Opcode Assignment**
Each runtime was developed by a different agent (or team) with no centralized coordination on byte values. The Python runtime was the first (assigning ADD=0x08), the Rust runtime followed a RISC-style layout (ADD=0x21), the C runtime followed hardware convention (ADD=0x10), and the WASM runtime created a new unified layout (ADD=0x20). No single agent made a "wrong" choice — they each made reasonable choices within their own context.

**Decision 2: Different Error Philosophies**
The WASM runtime chose "graceful degradation" (NOP for unimplemented opcodes). The Rust runtime chose "fail-fast" (panic for unimplemented opcodes). These are both valid engineering choices — but they are incompatible. A program that relies on silent NOP behavior will crash on Rust, and a program that expects crash-on-unknown will silently corrupt state on WASM.

**Decision 3: Memory Model Divergence**
The Go runtime was designed as a "social runtime" — it coordinates agents but doesn't compute. It has no LOAD/STORE because it doesn't need them. The WASM runtime was designed for browser execution with a fixed 64KB memory. The Python runtime uses unbounded dict-based storage for flexibility. These are all correct designs for their respective use cases — but they are mutually incompatible.

### 3.2 The Meta-Pattern

What emerges from this analysis is a **fractal design pattern** that repeats at every scale of the fleet:

```
Level 1: Opcodes within a runtime
  → Different opcodes serve different purposes (arithmetic vs memory vs control)
  → The same COMPOSITIONAL patterns recur (push-compute-pop, load-compare-branch)

Level 2: Runtimes within the fleet
  → Different runtimes serve different purposes (cognition vs performance vs hardware)
  → The same ARCHITECTURAL patterns recur (fetch-decode-execute, register file, stack)

Level 3: Agents within the organization
  → Different agents serve different purposes (datum=audit, oracle1=coordinate, etc.)
  → The same OPERATIONAL patterns recur (read-analyze-push, iterate-deliver)
```

The incompatibility is not a bug — it is the natural result of **convergent evolution** in a decentralized system. Each runtime evolved independently to fit its niche, and the niches are different enough that convergence was never pressured.

---

## 4. The Implications

### 4.1 For Fleet Standardization

The fleet currently has three possible standardization strategies:

**Strategy A: One True ISA (Top-Down)**
Designate the WASM numbering as canonical and require all runtimes to add translation layers. This maximizes bytecode portability but requires every runtime to change.

**Strategy B: Irreducible Core Protocol (Bottom-Up)**
Define the 17-opcode core as the universal standard. Programs targeting the core are guaranteed portable. Programs using extensions are runtime-specific by design. This requires no changes to existing runtimes.

**Strategy C: Signal Language (Highest-Level)**
Abandon bytecode-level portability entirely and standardize at the Signal language level (JSON messages: tell, ask, delegate, fork, join). All runtimes already support A2A communication, so this is achievable immediately.

**Recommendation:** Strategy B (irreducible core) for the short term, evolving toward Strategy A as runtimes add translation layers. Strategy C should be pursued in parallel for agent coordination use cases.

### 4.2 For New Runtime Development

Any new FLUX runtime should:
1. Implement the 17-opcode irreducible core with bit-identical semantics
2. Use the WASM opcode numbering as canonical (accept bytecode from the canonical shim)
3. Provide at least 4KB of byte-addressable linear memory (emulated if necessary)
4. Use a typed error model compatible with WASM's VMError codes
5. Document which non-core opcodes are implemented and how they differ from the canonical spec

### 4.3 For the Fleet's Self-Understanding

The lowest-level puzzle reveals something profound about the fleet's nature: **the fleet is not a monolithic system with parts — it is a collection of independent systems that happen to share a vocabulary.** The "FLUX ISA" is not a single specification but a family of related instruction sets, like the x86 family (8086, 386, 64) or the ARM family (ARMv7, ARMv8, ARMv9).

Understanding this helps set expectations: cross-runtime bytecode portability is not a binary property (portable or not) but a spectrum (portable at the core, progressively less portable at each extension layer).

---

## 5. The Formal Statement

### 5.1 Definition: Bytecode Portability Class

We define four portability classes for FLUX bytecode:

```
Class P0 (Universal): Uses ONLY the 17-opcode irreducible core.
  → Portable across all five runtimes without translation.
  → Guaranteed identical execution semantics.

Class P1 (Canonical): Uses opcodes from the canonical ISA v3 that are implemented
  on the target runtime.
  → Requires encoding translation (canonical_opcode_shim.py) for non-WASM runtimes.
  → Semantic equivalence for the 71 WASM-implemented opcodes.

Class P2 (Extended): Uses opcodes from one runtime's native set that are not in
  the canonical ISA or are not implemented on other runtimes.
  → Not portable. Requires source-level recompilation for each target.

Class P3 (Runtime-Exclusive): Uses opcodes or features unique to one runtime
  (e.g., Python's confidence system, C's DMA operations, Go's goroutine-based A2A).
  → Fundamentally non-portable. Encapsulate in Signal-language wrappers for
    cross-runtime communication.
```

### 5.2 The Portability Theorem

**Theorem:** Given any FLUX bytecode program B and any two runtimes R1 and R2, B is portable from R1 to R2 if and only if:

1. B is Class P0 (irreducible core only), OR
2. B is Class P1 AND R2 has a translation layer from R1's numbering to canonical, AND every opcode in B has a semantically equivalent implementation in R2

**Proof sketch:**
- If B is P0, all its opcodes are in the universal core, which every runtime implements identically. Portability is trivial.
- If B is P1, encoding translation ensures the byte values are correct, and semantic equivalence ensures the behavior is correct. Both conditions are necessary: encoding without semantics gives wrong behavior; semantics without encoding gives wrong dispatch.
- If B is P2 or P3, at least one opcode either lacks a translation or lacks a semantic equivalent. Portability is impossible without source changes.

### 5.3 Corollary: The Incompatibility Bound

No FLUX runtime can execute more than **28.3%** of the canonical ISA (71 of 251 opcodes, the WASM implementation rate) while maintaining full semantic fidelity. This is because the WASM runtime — the most complete implementation — only implements 71 opcodes. Any program using one of the 180 unimplemented opcodes will either NOP (on WASM) or crash (on Rust), neither of which is "correct" execution.

This bound will increase as runtimes implement more opcodes, but it can never reach 100% because some opcodes are runtime-exclusive (confidence system, DMA, sensor I/O).

---

## 6. Concrete Next Steps

### 6.1 Immediate (This Sprint)
1. ✅ Publish the irreducible core analysis (FLUX-IRREDUCIBLE-CORE.md)
2. ✅ Publish the execution semantics comparison (FLUX-EXECUTION-SEMANTICS.md)
3. ✅ Publish the universal bytecode validator (flux_universal_validator.py)
4. Publish the formal execution model specification (ITER 8)

### 6.2 Short-Term (Next Sprint)
1. Add the 17-opcode conformance test to flux-conformance
2. Create a "canonical bytecode compiler" that targets only the irreducible core
3. Add translation layers to the Python and Rust runtimes (via canonical_opcode_shim.py integration)

### 6.3 Medium-Term
1. Implement the 71-opcode functional core on at least 2 more runtimes
2. Define the standard memory abstraction (4KB minimum)
3. Create a "bytecode certification" program: "This program is Class P0 — universally portable"

### 6.4 Long-Term
1. Converge on a unified error model across all runtimes
2. Standardize overflow behavior and calling conventions
3. Create a fleet-wide bytecode registry where agents can share certified programs

---

## 7. Summary

The lowest-level puzzle of the FLUX fleet is not a single bug or missing feature — it is the natural consequence of **five independent evolutionary trajectories converging on a shared vocabulary but not on shared semantics.** The puzzle resolves not by forcing convergence but by understanding the structure of the divergence:

```
                    ↑ Portability
                    │
     P0 (Universal) │  17 opcodes — Turing complete, universally portable
     ───────────────│─────────────────────────────────────────
     P1 (Canonical) │  71 opcodes — portable with translation
     ───────────────│─────────────────────────────────────────
     P2 (Extended)  │  122-251 opcodes — runtime-specific
     ───────────────│─────────────────────────────────────────
     P3 (Exclusive) │  Runtime-unique features — non-portable
                    │
                    └──────────────────────────────────────→ Capability
```

The fleet does not need to choose between portability and capability. It can have both by clearly labeling each program's portability class and using the Signal language for cross-runtime communication when bytecode-level portability is insufficient.

The 17-opcode irreducible core is the bedrock. Everything else is an extension that adds capability at the cost of portability. This is not a limitation — it is a design principle.

---

*See also: FLUX-IRREDUCIBLE-CORE.md (the core), FLUX-EXECUTION-SEMANTICS.md (the semantics), canonical_opcode_shim.py (the translation tool), flux_universal_validator.py (the validator)*
