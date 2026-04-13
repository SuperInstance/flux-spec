# flux-spec

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
**The canonical FLUX language specification.** All implementations must conform to this spec.

## What This Defines

- **FLUX ISA v1.0** — Complete instruction set architecture (opcodes, encoding, register ABI)
- **FIR (FLUX Intermediate Representation)** — SSA-based IR specification
- **A2A Protocol** — Agent-to-agent communication protocol
- **Signal Language** — Agent-first-class JSON language for multi-agent coordination
- **.flux.md Format** — Markdown-native source file grammar
- **.fluxvocab Format** — Vocabulary interchange format

## Documents

| Document | Status | Description |
|----------|--------|-------------|
| [ISA.md](ISA.md) | SHIPPED | Complete ISA v1.0 specification — formats, registers, memory, execution model, all 247 opcodes |
| [OPCODES.md](OPCODES.md) | SHIPPED | Machine-readable opcode reference table |
| [FIR.md](FIR.md) | SHIPPED | Complete FIR v1.0 specification — type system (16 families), 54 instructions, SSA form, builder API, validation rules, bytecode encoding |
| [A2A.md](A2A.md) | SHIPPED | Complete A2A Protocol v1.0 — 16 opcodes, 52-byte message format, INCREMENTS+2 trust engine, capability system, Signal language (28 ops) |
| [SIGNAL.md](SIGNAL.md) | SHIPPED | Complete Signal Language v1.0 specification — 32 core ops, 6 protocol primitives, agent communication, parallelism, structured discourse, confidence-native, compilation model |
| [FLUXMD.md](FLUXMD.md) | SHIPPED | Complete .flux.md format v1.0 specification — frontmatter schema, parsing rules, directive sections, code block dialects, AST node reference, compilation pipeline |
| [FLUXVOCAB.md](FLUXVOCAB.md) | SHIPPED | Complete .fluxvocab format v1.0 specification — entry fields, assembly template syntax, variable substitution, vocabulary loading, sandbox execution, compiled interpreter generation |

## Specification Statistics

| Document | Lines | Sections | Primary Author |
|----------|-------|----------|---------------|
| ISA.md | ~642 | 11 | Super Z |
| FIR.md | ~1,749 | 12 | Super Z |
| A2A.md | ~1,663 | 14 | Super Z |
| SIGNAL.md | ~1,100 | 19 | Super Z |
| FLUXMD.md | ~571 | 8 | Super Z |
| FLUXVOCAB.md | ~671 | 12+3 | Super Z |
| OPCODES.md | ~263 | 1 | Super Z |
| **Total** | **~6,659** | | |

## ISA Summary

- **247 opcodes** across 17 categories
- **Variable-length instruction encoding** (1–5 bytes, little-endian)
- **7 instruction formats**: A (1B system) through G (5B reg+reg+imm16)
- **Confidence-OPTIONAL** parallel register file for uncertainty propagation
- **A2A primitives** for agent communication (TELL, ASK, DELEGATE, BROADCAST)
- **Viewpoint ops** for cross-linguistic semantics
- **Sensor ops** for edge hardware I/O
- **Tensor ops** for neural network primitives

## Related Repos

| Repo | Language | Purpose |
|------|----------|---------|
| [flux-runtime](https://github.com/SuperInstance/flux-runtime) | Python | Reference implementation + vocabulary system |
| [flux-core](https://github.com/SuperInstance/flux-core) | Rust | Production runtime |
| [flux-zig](https://github.com/SuperInstance/flux-zig) | Zig | Fastest VM (210ns/iter) |
| [flux-js](https://github.com/SuperInstance/flux-js) | JavaScript | JS VM with A2A (373ns/iter) |
| [flux-vocabulary](https://github.com/SuperInstance/flux-vocabulary) | Python | Standalone vocabulary library |
| [flux-lsp](https://github.com/SuperInstance/flux-lsp) | TypeScript | Language server |
| [flux-a2a-prototype](https://github.com/SuperInstance/flux-a2a-prototype) | Python | A2A protocol research implementation |
| [iron-to-iron](https://github.com/SuperInstance/iron-to-iron) | — | I2I agent communication protocol |
| [git-agent-standard](https://github.com/SuperInstance/git-agent-standard) | — | Git-agent embodiment standard |

## License

MIT
