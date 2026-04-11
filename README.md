# flux-spec

**The canonical FLUX language specification.** All implementations must conform to this spec.

## What This Defines

- **FLUX ISA v1.0** — Complete instruction set architecture (opcodes, encoding, register ABI)
- **FIR (FLUX Intermediate Representation)** — SSA-based IR specification
- **A2A Protocol** — Agent-to-agent communication protocol
- **.flux.md Format** — Markdown-native source file grammar
- **.fluxvocab Format** — Vocabulary interchange format
- **Conformance Tests** — Bytecode test vectors all VMs must pass

## ISA Summary

- **247 opcodes** across 17 categories
- **Variable-length instruction encoding** (1–5 bytes, little-endian)
- **48 registers** (16 GP + 16 FP + 16 SIMD) with explicit ABI (R0=zero, R11=SP, R14=FP, R15=LR)
- **7 instruction formats**: A (1B system) through G (5B reg+reg+imm16)
- **Capability-based memory** with named regions, ownership, and borrowers
- **Confidence-OPTIONAL** parallel register file for uncertainty propagation
- **A2A primitives** for agent communication (TELL, ASK, DELEGATE, BROADCAST, etc.)
- **Viewpoint ops** for cross-linguistic semantics (Babel's 16 V_* opcodes)
- **Sensor ops** for edge hardware I/O (JetsonClaw1's 16 S_* opcodes)
- **Tensor ops** for neural network primitives (matmul, conv, attention, etc.)

## Documents

| Document | Status | Description |
|----------|--------|-------------|
| [ISA.md](ISA.md) | SHIPPED | Complete ISA v1.0 specification — formats, registers, memory, execution model, all 247 opcodes |
| [OPCODES.md](OPCODES.md) | SHIPPED | Machine-readable opcode reference table (generated from isa_unified.py) |
| [FIR.md](FIR.md) | SHIPPED | Complete FIR v1.0 specification — type system (16 families), 54 instructions, SSA form, builder API, validation rules, bytecode encoding |
| [A2A.md](A2A.md) | SHIPPED | Complete A2A Protocol v1.0 — 16 opcodes, 52-byte message format, INCREMENTS+2 trust engine, capability system, Signal language (28 ops) |
| .flux.md grammar specification | Pending | Markdown-native source file grammar |
| .fluxvocab interchange format | Pending | Vocabulary interchange format |
| Conformance test vectors | Pending | Bytecode test vectors all VMs must pass |

## ISA Statistics

```
Total opcode slots:   256
Defined opcodes:      247
Reserved slots:        9
Confidence variants:  16
A2A primitives:       16
Viewpoint ops (Babel): 16
Sensor ops (JC1):     16
Vector/SIMD ops:      16
Tensor/neural ops:    16
Crypto ops:            8
```

By source attribution:
- Converged (shared):  ~140 ops
- Oracle1:             ~50 ops
- JetsonClaw1:         ~50 ops
- Babel:               16 ops

## Related Repos

| Repo | Language | Purpose |
|------|----------|---------|
| [flux-os](https://github.com/SuperInstance/flux-os) | C | Reference ISA implementation (microkernel) |
| [flux](https://github.com/SuperInstance/flux) | Rust | Production runtime |
| [flux-runtime](https://github.com/SuperInstance/flux-runtime) | Python | Reference implementation + vocabulary system |
| [flux-vocabulary](https://github.com/SuperInstance/flux-vocabulary) | Python | Standalone vocabulary library |
| [flux-conformance](https://github.com/SuperInstance/flux-conformance) | JSON | Cross-runtime test suite |
| [flux-lsp](https://github.com/SuperInstance/flux-lsp) | TypeScript | Language server |
| [flux-ide](https://github.com/SuperInstance/flux-ide) | TypeScript | Web IDE |

## License

MIT
