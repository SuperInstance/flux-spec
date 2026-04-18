# flux-spec

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
**The canonical FLUX language specification.** All implementations must conform to this spec.

## What This Defines

- **FLUX ISA v3.0** — Next-generation instruction set architecture with escape prefix extensibility, security primitives, async/temporal operations, and compressed instruction format
- **FLUX ISA v1.0** — Foundational instruction set architecture (opcodes, encoding, register ABI)
- **FIR (FLUX Intermediate Representation)** — SSA-based IR specification
- **A2A Protocol** — Agent-to-agent communication protocol
- **Signal Language** — Agent-first-class JSON language for multi-agent coordination
- **FLUX Programs** — Real-world algorithm implementations proving the VM runs actual computation
- **.flux.md Format** — Markdown-native source file grammar
- **.fluxvocab Format** — Vocabulary interchange format

## Documents

| Document | Status | Description |
|----------|--------|-------------|
| [ISA-v3.md](ISA-v3.md) | DRAFT | ISA v3.0 comprehensive specification — escape prefix (0xFF), security primitives (CAP_INVOKE, MEM_TAG, FUEL, SANDBOX), async primitives (SUSPEND, RESUME, FORK, JOIN), temporal primitives (DEADLINE, YIELD, PERSIST_STATE), compressed instruction format (2-byte), tensor/SIMD ops, migration guide v2→v3 |
| [FLUX-PROGRAMS.md](FLUX-PROGRAMS.md) | SHIPPED | Five real FLUX programs with hand-crafted bytecode: Euclidean GCD, Fibonacci, Bubble Sort, Sieve of Eratosthenes, Matrix Multiplication — includes assembly, raw bytecode, correctness traces, complexity analysis |
| [ISA-v3-ESCAPE-PREFIX.md](ISA-v3-ESCAPE-PREFIX.md) | PROPOSAL | Quill's escape prefix mechanism design — 0xFF prefix with 256 sub-opcode namespaces, extension discovery protocol, compressed shorts |
| [ISA.md](ISA.md) | SHIPPED | Complete ISA v1.0 specification — formats, registers, memory, execution model, all 247 opcodes |
| [OPCODES.md](OPCODES.md) | SHIPPED | Machine-readable opcode reference table (v2/v1 unified) |
| [FIR.md](FIR.md) | SHIPPED | Complete FIR v1.0 specification — type system (16 families), 54 instructions, SSA form, builder API, validation rules, bytecode encoding |
| [A2A.md](A2A.md) | SHIPPED | Complete A2A Protocol v1.0 — 16 opcodes, 52-byte message format, INCREMENTS+2 trust engine, capability system, Signal language (28 ops) |
| [SIGNAL.md](SIGNAL.md) | SHIPPED | Complete Signal Language v1.0 specification — 32 core ops, 6 protocol primitives, agent communication, parallelism, structured discourse, confidence-native, compilation model |
| [SIGNAL-AMENDMENT-1.md](SIGNAL-AMENDMENT-1.md) | SHIPPED | Signal Amendment 1 — proposed resolutions for 6 open questions |
| [FLUXMD.md](FLUXMD.md) | SHIPPED | Complete .flux.md format v1.0 specification — frontmatter schema, parsing rules, directive sections, code block dialects, AST node reference, compilation pipeline |
| [FLUXVOCAB.md](FLUXVOCAB.md) | SHIPPED | Complete .fluxvocab format v1.0 specification — entry fields, assembly template syntax, variable substitution, vocabulary loading, sandbox execution, compiled interpreter generation |
| [ENCODING-FORMATS.md](ENCODING-FORMATS.md) | SHIPPED | Formal encoding format specification — Formats A through G with bit-level layouts |

## Specification Statistics

| Document | Lines | Sections | Primary Author |
|----------|-------|----------|---------------|
| ISA-v3.md | ~829 | 11 | Datum |
| FLUX-PROGRAMS.md | ~579 | 5 + traces | Datum |
| ISA-v3-ESCAPE-PREFIX.md | ~376 | 10 | Quill |
| ISA.md | ~642 | 11 | Super Z |
| FIR.md | ~1,749 | 12 | Super Z |
| A2A.md | ~1,663 | 14 | Super Z |
| SIGNAL.md | ~1,100 | 19 | Super Z |
| ENCODING-FORMATS.md | ~1,600 | — | Super Z |
| FLUXMD.md | ~571 | 8 | Super Z |
| FLUXVOCAB.md | ~671 | 12+3 | Super Z |
| OPCODES.md | ~263 | 1 | Super Z |
| **Total** | **~10,043** | | |

## ISA Summary

### v3.0 (Draft)
- **Escape prefix (0xFF)** — Unlimited opcode extensibility via 256 sub-opcode namespaces
- **Security primitives** — CAP_INVOKE, MEM_TAG, FUEL_CHECK, SANDBOX_ENTER/EXIT for multi-agent isolation
- **Async primitives** — SUSPEND, RESUME, AWAIT, FORK_CONT, JOIN_CONT for agent migration and parallelism
- **Temporal primitives** — DEADLINE_SET, YIELD_CONTENTION, PERSIST_STATE, TIMESTAMP for time-aware execution
- **Compressed instruction format** — 2-byte encodings for top 12 opcodes, ~30% code size reduction
- **24 new tensor/SIMD opcodes** — In freed 0x60-0x7F space (migrated confidence/viewpoint to escape)
- **Full v2 backward compatibility** — All v2 bytecodes execute identically on v3 runtimes
- **68 new conformance test vectors** — 62 shipped to flux-conformance, covering all new primitives

### v1.0 (Stable)
- **247 opcodes** across 17 categories
- **Variable-length instruction encoding** (1–5 bytes, little-endian)
- **7 instruction formats**: A (1B system) through G (5B reg+reg+imm16)
- **Confidence-OPTIONAL** parallel register file for uncertainty propagation
- **A2A primitives** for agent communication (TELL, ASK, DELEGATE, BROADCAST)
- **Viewpoint ops** for cross-linguistic semantics
- **Sensor ops** for edge hardware I/O
- **Tensor ops** for neural network primitives

### Conformance

| Suite | Vectors | Status |
|-------|---------|--------|
| v2 core (conformance-vectors.json) | 113 | All passing against Python reference VM |
| v3 extensions (conformance-vectors-v3.json) | 62 | Shipped — 7 categories, runner included |
| v3 unit tests (test_conformance_v3.py) | 28 | All passing |
| v3 benchmarks (benchmark_flux.py) | — | Performance harness shipped |
| **Total** | **175+** | |

## Real FLUX Programs

The FLUX VM runs real algorithms, not just exercises. Five non-trivial programs are documented in [FLUX-PROGRAMS.md](FLUX-PROGRAMS.md):

| Program | Instructions | Bytes | ISA Features Demonstrated |
|---------|-------------|-------|---------------------------|
| Euclidean GCD | 10 | 24 | MOD, CMP_EQ, JNZ, loops |
| Fibonacci (iterative) | 12 | 29 | ADD, MOV, DEC, rotation |
| Bubble Sort | ~35 | ~140 | LOADOFF, STOREOF, conditional branch, memory |
| Sieve of Eratosthenes | ~25 | ~100 | Memory arrays, nested loops, boolean flags |
| Matrix Multiply (2x2) | 64 | ~260 | LOADOFF, STOREOF, MUL, memory addressing |

## Related Repos

| Repo | Language | Purpose |
|------|----------|---------|
| [flux-runtime](https://github.com/SuperInstance/flux-runtime) | Python | Reference implementation — 2,360 tests, vocabulary system, A2A |
| [flux-core](https://github.com/SuperInstance/flux-core) | Rust | Production runtime — VM, assembler, disassembler |
| [flux-conformance](https://github.com/SuperInstance/flux-conformance) | Python | Conformance test suite — 113 v2 + 62 v3 vectors + v3 runner + benchmarks |
| [flux-runtime-wasm](https://github.com/SuperInstance/flux-runtime-wasm) | TypeScript | WASM target — 170+ opcodes, 44 tests, assembler, markdown-to-bytecode |
| [flux-zig](https://github.com/SuperInstance/flux-zig) | Zig | Fastest VM (210ns/iter benchmark) |
| [flux-js](https://github.com/SuperInstance/flux-js) | JavaScript | JS VM with A2A (373ns/iter benchmark) |
| [flux-swarm](https://github.com/SuperInstance/flux-swarm) | Go | Distributed agent runtime — Go implementation with swarm coordination |
| [flux-vocabulary](https://github.com/SuperInstance/flux-vocabulary) | Python | Standalone vocabulary library — 606 terms across 132 domains |
| [flux-lsp](https://github.com/SuperInstance/flux-lsp) | TypeScript | Language server — IDE integration |
| [flux-ide](https://github.com/SuperInstance/flux-ide) | TypeScript | IDE — markdown-to-bytecode agent-native development |
| [flux-a2a-signal](https://github.com/SuperInstance/flux-a2a-signal) | Python | A2A Signal Protocol — 476KB specification implementation |
| [flux-a2a-prototype](https://github.com/SuperInstance/flux-a2a-prototype) | Python | A2A protocol research implementation |
| [flux-os](https://github.com/SuperInstance/flux-os) | C | OS-level FLUX — Pure C hardware-agnostic operating system |
| [iron-to-iron](https://github.com/SuperInstance/iron-to-iron) | — | I2I agent communication protocol |
| [git-agent-standard](https://github.com/SuperInstance/git-agent-standard) | — | Git-agent embodiment standard |
| [oracle1-vessel](https://github.com/SuperInstance/oracle1-vessel) | Python | Fleet coordination — task board, dispatch, bottles |

## Version History

| Date | Version | Change | Author |
|------|---------|--------|--------|
| 2026-04-13 | v3.0 DRAFT | ISA v3 comprehensive spec, FLUX programs collection, conformance vectors | Datum |
| 2026-04-12 | — | Escape prefix proposal, encoding formats spec | Quill |
| 2026-04-11 | v2.0 | ISA convergence, 88→113 conformance vectors | Oracle1 |
| 2026-04-10 | v1.1 | Signal Amendment 1 | Quill |
| 2026-03-XX | v1.0 | Initial 7 documents shipped | Super Z |

## License

MIT
