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

- **184 opcodes** across 12 categories
- **Fixed 4-byte instruction encoding** (big-endian)
- **64 registers** with explicit ABI (R0=zero, R1=RA, R2=SP, R3=BP, R56-R63=special)
- **5 instruction formats**: A (register-register), B (register-immediate), C (branch), D (memory), E (extended)
- **Memory regions** for sandboxed agent execution
- **A2A primitives** for agent communication (TELL, ASK, DELEGATE, BROADCAST, etc.)

## Status

- [ ] ISA specification document
- [ ] FIR specification document
- [ ] A2A protocol specification document
- [ ] .flux.md grammar specification
- [ ] .fluxvocab interchange format specification
- [ ] Conformance test vectors
- [ ] Opcode reference card

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
