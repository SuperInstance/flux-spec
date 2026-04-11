# .fluxvocab Format Specification

**Version:** 1.0
**Status:** Draft
**Date:** 2026-04-12
**Author:** Super Z ⚡ (Cartographer, SuperInstance)

---

## 1. Overview

`.fluxvocab` is the **vocabulary interchange format** for FLUX. It defines named instruction patterns — called **words** — that map natural language or structured input to FLUX bytecode sequences via an assembly template. The format draws from Forth's word-based paradigm: each word is a reusable, composable unit of behavior that expands to assembly, assembles to bytecode, and executes in the sandbox VM.

A vocabulary is a collection of words organized by domain (math, loops, strings, etc.). Vocabularies are loaded by the FLUX interpreter at runtime and matched against input text. The first matching word's assembly template is substituted, assembled, and executed.

### Design Goals

- **Natural language interface:** Words match human-readable patterns (e.g., `"compute $a + $b"`) and expand to bytecode
- **Composable:** Words can reference each other, and vocabulary folders can be layered
- **Zero dependencies:** `.fluxvocab` files are plain text with no build step
- **Self-contained execution:** The `compile_interpreter()` tool generates standalone Python modules with inline VM and assembler
- **Domain-organized:** Vocabularies are grouped into folders (core, math, loops, etc.) and loaded hierarchically

### File Extension

`.fluxvocab` — plain text files loaded by the FLUX vocabulary system.

---

## 2. File Structure

A `.fluxvocab` file contains one or more vocabulary entries, separated by horizontal rules.

```
---
<entry-block-1>
---
<entry-block-2>
---
<entry-block-3>
```

The separator is `---` at the start of a line (optionally followed by whitespace). An optional leading `---` before the first entry is tolerated — the split produces an empty first block which is silently skipped.

Entries are parsed in declaration order. Within a file, this order determines match priority: the first matching word wins.

### BNF Grammar

```
vocab_file      ::= ["---" EOL] entry_block ("---" EOL entry_block)*
entry_block     ::= metadata_field EOL (metadata_field EOL)* [assembly_body]
metadata_field  ::= field_key ":" WHITESPACE field_value
field_key       ::= "pattern" | "expand" | "result" | "name" | "description" | "tags"
field_value     ::= quoted_string | inline_value | pipe_block
quoted_string   ::= '"' [^"]* '"'
inline_value    ::= [^\n]+
pipe_block      ::= "|" EOL indented_line (EOL indented_line)*
indented_line   ::= WHITESPACE assembly_line
assembly_body   ::= assembly_line (EOL assembly_line)*
assembly_line   ::= [mnemonic [operand_list]] [comment]
```

---

## 3. Entry Fields

### 3.1 `pattern` — Matching Pattern (Required)

Defines the natural language or structured text pattern this word matches against. Patterns contain **variable placeholders** (`$varname`) that capture numeric arguments from input.

#### Syntax

```
pattern: "<text with $var placeholders>"
```

or (unquoted):

```
pattern: <text with $var placeholders>
```

#### Variable Placeholders

A dollar sign followed by one or more word characters: `$varname`.

Each unique `$varname` in the pattern becomes a **named regex capture group** `(?P<varname>\d+)` at match time. Captures are **digits only** — the regex matches one or more decimal digits.

#### Pattern Compilation

At load time, the pattern string is compiled into a regex:

1. Split the pattern on `$varname` boundaries using `re.split(r'(\$\w+)', pattern)`
2. Literal parts → `re.escape(part)` (escape regex special characters)
3. `$varname` parts → `(?P<varname>\d+)`
4. Join all parts into a complete regex
5. Compile with `re.IGNORECASE`

#### Matching Semantics

- Matching uses `regex.search(text)` — **substring match**, not full match
- First match wins (no priority scoring or specificity ranking)
- Match priority is determined by load order: alphabetical by filename within a folder, declaration order within a file
- Case-insensitive matching

#### Examples

| Pattern | Compiled Regex | Matches | Captures |
|---------|---------------|---------|----------|
| `compute $a + $b` | `compute\ (?P<a>\d+)\ \+\ (?P<b>\d+)` | `"compute 3 + 4"` | `a="3", b="4"` |
| `factorial of $n` | `factorial\ of\ (?P<n>\d+)` | `"factorial of 10"` | `n="10"` |
| `is $x prime` | `is\ (?P<x>\d+)\ prime` | `"is 17 prime"` | `x="17"` |
| `load $val into R$reg` | `load\ (?P<val>\d+)\ into\ R(?P<reg>\d+)` | `"load 42 into R3"` | `val="42", reg="3"` |

Note the last example: `$var` can appear **inside register operands** (`R${reg}`), enabling dynamic register selection via substitution.

### 3.2 `expand` — Assembly Template (Required)

Defines the FLUX assembly instructions that the word expands to when matched. Supports two forms:

#### Inline Form (single instruction)

```
expand: HALT
```

The entire text after `expand:` (stripped of leading whitespace) is the assembly.

#### Block Form (multi-line)

```
expand: |
    MOVI R0, ${a}
    MOVI R1, ${b}
    IADD R0, R0, R1
    HALT
```

The `|` marker indicates a YAML-style block scalar. Lines after `|` must be indented. Leading whitespace on each line is stripped. Empty lines are preserved but skipped by the assembler.

#### Substitution Syntax

Captured values from the pattern are substituted into the assembly template using `${varname}` syntax:

- `${varname}` → replaced with the captured string value
- This is **literal string replacement**, not regex — `assembly.replace("${varname}", value)`
- Braces are required: `$varname` without braces is NOT substituted in the expand body

#### Assembly Subset

The vocabulary assembler supports a **subset of FLUX assembly** — 26 opcodes from the old opcode space:

| Category | Mnemonics | Format |
|----------|-----------|--------|
| No-operand | `NOP`, `HALT`, `RET`, `YIELD` | 1 byte |
| Single register | `INC`, `DEC`, `PUSH`, `POP`, `NOT`, `INEG` | 2 bytes |
| Two register | `MOV Rd, Rs` | 3 bytes |
| Three register | `IADD`, `ISUB`, `IMUL`, `IDIV`, `IMOD`, `AND`, `OR`, `XOR`, `SHL`, `SHR`, `CMP` | 4 bytes |
| Reg + imm16 | `MOVI Rd, imm`, `JZ Rd, off`, `JNZ Rd, off` | 4 bytes |
| Imm16 only | `JMP offset` | 3 bytes |

**Important limitations:**
- No labels or symbolic references — loops use hardcoded relative offsets
- No forward references — all jump offsets must be manually calculated
- Comments: `;`, `//`, `#` prefix (inline `;` comments are stripped)
- Registers: `R0`–`R15` or bare `0`–`15`
- Immediates: signed 16-bit integers, little-endian encoding
- `CMP Ra, Rb` stores result (-1/0/+1) into R13 (comparison result register)

#### Example

Pattern match: `"compute 3 + 4"` → captures `{a: "3", b: "4"}`

Template:
```
expand: |
    MOVI R0, ${a}
    MOVI R1, ${b}
    IADD R0, R0, R1
    HALT
```

After substitution:
```
MOVI R0, 3
MOVI R1, 4
IADD R0, R0, R1
HALT
```

Assembled bytecode (13 bytes): `[0x2B, 0x00, 0x03, 0x00, 0x2B, 0x01, 0x04, 0x00, 0x08, 0x00, 0x00, 0x01, 0x80]`

### 3.3 `result` — Result Register (Optional)

Specifies which register holds the result value after execution. Default: `R0`.

```
result: R5
```

The `R` prefix is stripped; only the number is parsed (0–15).

### 3.4 `name` — Word Name (Optional)

Human-readable identifier for the word. Used for method naming in compiled interpreters (sanitized to lowercase alphanumeric with underscores). If omitted, auto-derived from the first 40 characters of the pattern.

```
name: addition
name: factorial-compute
```

### 3.5 `description` — Description (Optional)

Free-text description of what the word does.

```
description: Add two numbers and return the sum in R0
```

### 3.6 `tags` — Category Tags (Optional)

Comma-separated tags for categorization and filtering.

```
tags: math, arithmetic, core
tags: loop, iteration, control-flow
```

---

## 4. Field Reference Table

| Field | Required | Default | Format | Description |
|-------|----------|---------|--------|-------------|
| `pattern` | Yes | — | Quoted or unquoted string | Natural language matching pattern with `$var` placeholders |
| `expand` | Yes | — | Inline or `|` block | Assembly template with `${var}` substitution |
| `result` | No | `R0` | `R<n>` (n = 0–15) | Register holding the result |
| `name` | No | First 40 chars of pattern | Identifier | Human-readable name |
| `description` | No | Empty | Free text | What this word does |
| `tags` | No | Empty | Comma-separated | Category tags |

---

## 5. Complete Examples

### 5.1 Basic Arithmetic Word

```
---
pattern: "compute $a + $b"
expand: |
    MOVI R0, ${a}
    MOVI R1, ${b}
    IADD R0, R0, R1
    HALT
result: R0
name: addition
description: Add two numbers and return the sum
tags: math, arithmetic
```

### 5.2 Word with Loop (Hardcoded Offset)

```
---
pattern: "sum from 1 to $n"
expand: |
    MOVI R0, 0
    MOVI R1, 1
    MOVI R2, ${n}
    MOVI R3, 5
    DEC R3
@loop:
    IADD R0, R0, R1
    INC R1
    CMP R1, R2
    JNZ R0, @loop
    HALT
result: R0
name: sum-to-n
description: Compute the sum of integers from 1 to n
tags: math, loop
```

Note: The `JNZ R0, @loop` line demonstrates that the vocabulary assembler does NOT support labels. This entry would need to be rewritten with a calculated relative offset (e.g., `JNZ R0, -5`) to actually assemble correctly. This is a known limitation.

### 5.3 Word with Dynamic Register Selection

```
---
pattern: "load $val into R$reg"
expand: |
    MOVI R${reg}, ${val}
    HALT
result: R${reg}
name: load-immediate
description: Load an immediate value into a specified register
tags: core, register
```

### 5.4 Multi-Word Vocabulary File

```
---
pattern: "compute $a + $b"
expand: |
    MOVI R0, ${a}
    MOVI R1, ${b}
    IADD R0, R0, R1
    HALT
result: R0
name: addition
tags: math
---
pattern: "compute $a - $b"
expand: |
    MOVI R0, ${a}
    MOVI R1, ${b}
    ISUB R0, R0, R1
    HALT
result: R0
name: subtraction
tags: math
---
pattern: "compute $a * $b"
expand: |
    MOVI R0, ${a}
    MOVI R1, ${b}
    IMUL R0, R0, R1
    HALT
result: R0
name: multiplication
tags: math
---
pattern: "compute $a / $b"
expand: |
    MOVI R0, ${a}
    MOVI R1, ${b}
    IDIV R0, R0, R1
    HALT
result: R0
name: division
tags: math
```

---

## 6. Vocabulary Loading

### 6.1 Folder Structure

Vocabularies are organized into directories:

```
vocabularies/
├── core/
│   └── basic.fluxvocab       # Core patterns (load, store, halt)
├── math/
│   ├── arithmetic.fluxvocab   # +, -, *, / operations
│   └── sequences.fluxvocab    # Fibonacci, factorial
├── loops/
│   └── basic.fluxvocab        # Loop patterns
├── examples/
│   ├── maritime.fluxvocab     # Domain-specific examples
│   └── math_decomposed.fluxvocab
└── custom/                    # User-defined vocabularies
    └── README.md
```

### 6.2 Loading Order

When `Vocabulary.load_folder(path)` is called:

1. List all files in the directory (via `os.listdir`)
2. Sort alphabetically
3. For each `.fluxvocab` file (in sorted order):
   a. Split on `^---\s*$` to extract entry blocks
   b. Parse each entry block
   c. Add valid entries to the vocabulary in order
4. Skip `.fluxtpl` template files (loaded separately)

Files are loaded alphabetically within a folder. Entries within a file maintain declaration order. This means:

- `arithmetic.fluxvocab` loads before `sequences.fluxvocab`
- Within a file, the first entry has highest match priority

### 6.3 Layered Loading

Multiple vocabulary folders can be loaded in sequence:

```python
vocab = Vocabulary()
vocab.load_folder("vocabularies/core/")     # Loaded first (highest priority)
vocab.load_folder("vocabularies/math/")     # Loaded second
vocab.load_folder("vocabularies/loops/")    # Loaded third
```

Later-loaded entries do NOT override earlier ones. Match priority is strictly first-match-wins across all loaded entries, in load order.

### 6.4 Deduplication

Each file path is tracked in `_loaded_paths` to prevent loading the same file twice.

---

## 7. Template Files (.fluxtpl)

A separate format for reusable assembly templates. Not directly executable — used as building blocks for `.fluxvocab` entries.

```
name: factorial
result: R0
description: Compute factorial of n

assembly:
    MOVI R2, 1
    MOVI R1, ${n}
    ; ... loop body ...
    HALT
```

Fields:
- `name:` — Template identifier
- `result:` — Result register
- `description:` — What the template does
- `assembly:` — Assembly body (block after the keyword)

Templates are loaded into `BytecodeTemplate` objects and stored in the vocabulary registry, but are not matched against input text. They serve as a library of reusable assembly snippets.

---

## 8. Interpreter Dispatch

When the FLUX interpreter receives input text, it follows this dispatch order:

1. **Vocabulary match** — Iterate all loaded entries, try `regex.search(text)` on each. If matched, substitute `${var}` in assembly, assemble to bytecode, execute in sandbox VM.

2. **Assembly detection** — If no vocabulary match and the first 3 lines start with known mnemonics (case-insensitive), treat as raw assembly text.

3. **Hex bytecode detection** — If no vocabulary match and input is an even-length hex string (≥4 chars), decode and execute directly.

4. **Inline math** — If no vocabulary match and input matches `(\d+)\s*([+\-*/×÷])\s*(\d+)`, evaluate directly in Python without VM execution.

5. **Error** — If nothing matched, return an error `SandboxResult`.

---

## 9. Compiled Interpreter Generation

The `compile_interpreter()` tool generates a **standalone Python module** from vocabulary folders. Each vocabulary entry becomes a native Python method in the generated class.

### Generated Module Structure

```python
# Generated by flux compile-interpreter
import struct
from typing import Optional

class Result:
    def __init__(self, success, value, cycles, error): ...

class _VM:
    # Inline sandbox VM (16 registers, cycle-limited)
    def execute(self, bytecode): ...

class _asm:
    # Inline assembler (26-opcode subset)
    def assemble(self, text): ...

class AutopilotFlux:
    def __init__(self):
        self._patterns = []  # (compiled_regex, bound_method) pairs

    def addition(self, a, b):
        assembly = f"MOVI R0, {a}\nMOVI R1, {b}\nIADD R0, R0, R1\nHALT"
        bytecode = _asm().assemble(assembly)
        result = _VM().execute(bytecode)
        return Result(result.success, result.registers[0], result.cycles, result.error)

    def run(self, text):
        for pattern, method in self._patterns:
            m = pattern.search(text)
            if m:
                return method(**m.groupdict())
        return Result(False, None, 0, "No matching vocabulary word")
```

### Properties

- **Zero external dependencies** — only `struct` and `typing` from stdlib
- **No runtime pattern matching** — patterns compiled directly into method dispatch
- **Inline VM** — identical to `SandboxVM` but embedded, no imports
- **Inline assembler** — subset of `assemble_text()` covering the 26 opcodes used by vocabularies

---

## 10. Sandbox Execution Model

Vocabulary words execute in a sandboxed VM with the following constraints:

| Constraint | Value | Purpose |
|-----------|-------|---------|
| Max registers | 16 (R0–R15) | Fixed register file |
| Max stack depth | 256 entries | Prevent stack overflow |
| Max cycles | 1,000,000 | Prevent infinite loops |
| Memory access | None | No LOAD/STORE in vocabulary subset |
| I/O | None | No SYS calls, no network |
| Error handling | Catch all | Errors stored in result, never crash host |

### Success Condition

A result is considered successful when:
- `result.error is None` (no runtime errors)
- `result.halted is True` (VM reached HALT instruction)
- `result.cycles < max_cycles` (didn't time out)

---

## 11. Relationship to Other Formats

| Format | Purpose | Connection |
|--------|---------|------------|
| `.flux.md` | Module-level source format | Can contain ````fluxvocab` code blocks that define vocabulary words inline |
| `.fluxvocab` | Vocabulary interchange format | Standalone vocabulary definition files loaded by the interpreter |
| `.fluxtpl` | Assembly template files | Reusable assembly snippets, not directly matched |
| `.ese` | Extended semantic entries | Alternative entry format (same parsing pipeline as `.fluxvocab`) |

### Integration with .flux.md

A `.flux.md` file can define vocabulary words inline using a `## vocabulary:` section:

```markdown
## vocabulary: core_actions
Core vocabulary words for basic operations.

```fluxvocab
:double ( n -- 2n )
  IADD R0, R0, R0
  RET

:swap ( a b -- b a )
  MOV R2, R0
  MOV R0, R1
  MOV R1, R2
  RET
```
```

Note: This inline syntax (Forth-style `:wordname` with stack effects) is a different format from the standalone `.fluxvocab` pattern/expand format. The inline format is processed by the `.flux.md` parser's vocabulary section handler, while the standalone format is loaded by `Vocabulary.load_folder()`.

---

## 12. Known Limitations and Open Issues

1. **Digits-only captures** — `$var` only matches `\d+`. Vocabulary words cannot accept string or symbolic arguments. This is a deliberate simplification for the current implementation.

2. **No labels in assembly subset** — The vocabulary assembler does not support labels (`@name:`) or forward references. Loops must use hardcoded relative offsets, making complex control flow fragile and error-prone.

3. **Substring matching ambiguity** — `regex.search()` can cause false positives. `"add 3 and 4"` might match both `"add $a and $b"` and `"compute $a + $b"` depending on vocabulary order.

4. **No priority scoring** — Match priority is purely order-dependent. There is no specificity ranking (longer/more-specific patterns don't get priority over shorter/more-general ones).

5. **Dual ISA problem** — The vocabulary assembler uses the old opcode space (from `opcodes.py`), not the converged ISA (from `isa_unified.py`). Compiled interpreters will produce bytecode incompatible with the canonical spec.

6. **No type validation** — Template substitution is pure string replacement. There is no type checking, range validation, or overflow detection on substituted values.

7. **Empty entries silently skipped** — If both `pattern` and `expand` are empty, the entry is silently ignored rather than reported as an error.

8. **Frontmatter YAML limitations** — The parser's YAML frontmatter handler doesn't support multi-line values, nested structures, or anchors. Complex metadata silently falls back to raw string assignment.

---

## Appendix A: Vocabulary Entry Data Model

```python
@dataclass
class VocabEntry:
    pattern: str           # Raw pattern string with $var placeholders
    expand: str            # Assembly template with ${var} substitution
    result: int            # Result register number (0-15)
    name: str              # Human-readable name
    description: str       # What this word does
    tags: list[str]        # Category tags
    regex: Pattern         # Compiled regex (built from pattern)

    def compile(self) -> Pattern:
        """Compile pattern string into regex with digit-only captures."""

    def match(self, text: str) -> Optional[dict]:
        """Try to match text, return captured groups or None."""
```

## Appendix B: Supported Assembly Opcodes (Vocabulary Subset)

| Mnemonic | Opcode | Format | Encoding | Description |
|----------|--------|--------|----------|-------------|
| `NOP` | 0x00 | A | `[0x00]` | No operation |
| `HALT` | 0x80 | A | `[0x80]` | Stop execution |
| `RET` | 0x28 | A | `[0x28]` | Return |
| `YIELD` | 0x81 | A | `[0x81]` | Yield to scheduler |
| `INC Rn` | 0x08 | B | `[0x08][n]` | Increment register |
| `DEC Rn` | 0x09 | B | `[0x09][n]` | Decrement register |
| `PUSH Rn` | 0x20 | B | `[0x20][n]` | Push to stack |
| `POP Rn` | 0x21 | B | `[0x21][n]` | Pop from stack |
| `NOT Rn` | 0x13 | B | `[0x13][n]` | Bitwise NOT |
| `INEG Rn` | 0x0D | B | `[0x0D][n]` | Negate |
| `MOV Rd, Rs` | 0x01 | C | `[0x01][d][s]` | Copy register |
| `MOVI Rd, imm` | 0x2B | D | `[0x2B][d][imm:LE16]` | Load immediate |
| `IADD Rd, Ra, Rb` | 0x08 | E | `[0x08][d][a][b]` | Integer add |
| `ISUB Rd, Ra, Rb` | 0x09 | E | `[0x09][d][a][b]` | Integer subtract |
| `IMUL Rd, Ra, Rb` | 0x0A | E | `[0x0A][d][a][b]` | Integer multiply |
| `IDIV Rd, Ra, Rb` | 0x0B | E | `[0x0B][d][a][b]` | Integer divide |
| `IMOD Rd, Ra, Rb` | 0x0C | E | `[0x0C][d][a][b]` | Integer modulo |
| `AND Rd, Ra, Rb` | 0x10 | E | `[0x10][d][a][b]` | Bitwise AND |
| `OR Rd, Ra, Rb` | 0x11 | E | `[0x11][d][a][b]` | Bitwise OR |
| `XOR Rd, Ra, Rb` | 0x12 | E | `[0x12][d][a][b]` | Bitwise XOR |
| `SHL Rd, Ra, Rb` | 0x14 | E | `[0x14][d][a][b]` | Shift left |
| `SHR Rd, Ra, Rb` | 0x15 | E | `[0x15][d][a][b]` | Shift right |
| `CMP Ra, Rb` | 0x18 | E | `[0x18][0][a][b]` | Compare (result in R13) |
| `JMP offset` | 0x04 | D | `[0x04][0][off:LE16]` | Unconditional jump |
| `JZ Rd, offset` | 0x05 | D | `[0x05][d][off:LE16]` | Jump if zero |
| `JNZ Rd, offset` | 0x06 | D | `[0x06][d][off:LE16]` | Jump if not zero |

## Appendix C: Complete Working Example

```fluxvocab
---
pattern: "add $a and $b"
expand: |
    MOVI R0, ${a}
    MOVI R1, ${b}
    IADD R0, R0, R1
    HALT
result: R0
name: word_add
description: Add two numbers
tags: math, arithmetic, core
---
pattern: "negate $x"
expand: |
    MOVI R0, ${x}
    INEG R0
    HALT
result: R0
name: word_negate
description: Negate a number
tags: math, unary
---
pattern: "double $x"
expand: |
    MOVI R0, ${x}
    IADD R0, R0, R0
    HALT
result: R0
name: word_double
description: Double a number
tags: math, unary
---
pattern: "max of $a and $b"
expand: |
    MOVI R0, ${a}
    MOVI R1, ${b}
    CMP R0, R1
    MOV R0, R1
    HALT
result: R0
name: word_max
description: Return the larger of two numbers
tags: math, comparison
```
