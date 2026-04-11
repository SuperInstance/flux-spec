# .flux.md File Format Specification v1.0

**The canonical reference for the `.flux.md` source file format.** All parsers,
compilers, and tooling must conform to this specification.

---

## Table of Contents

1. [Overview](#1-overview)
2. [File Structure](#2-file-structure)
3. [Parsing Rules](#3-parsing-rules)
4. [Directive Sections](#4-directive-sections)
5. [Code Block Dialects](#5-code-block-dialects)
6. [AST Node Reference](#6-ast-node-reference)
7. [Compilation Pipeline](#7-compilation-pipeline)
8. [Complete Working Example](#8-complete-working-example)

---

## 1. Overview

`.flux.md` is a markdown-native source format for FLUX programs. It embraces the
literate programming philosophy: human-readable documentation and machine-compiled
code coexist within a single file. Any `.flux.md` file can be opened in a standard
markdown viewer and read as documentation, while the FLUX toolchain can extract
and compile its embedded code. This dual nature means `.flux.md` serves
simultaneously as specification, documentation, and executable source — a single
artifact that eliminates drift between docs and code.

The format is valid CommonMark markdown augmented with YAML frontmatter,
FLUX-specific heading directives (`## agent:`, `## fn:`), and classified fenced
code blocks. The parser converts markdown source into a `FluxModule` AST. The
compiler then extracts native code blocks, compiles them through the appropriate
frontend (C, Python) into FIR, and encodes FIR to FLUX bytecode:

```
┌───────────────┐     ┌──────────────┐     ┌───────────┐     ┌──────────┐     ┌─────┐
│  .flux.md     │ ── │ FluxMDParser │ ── │ FluxModule│ ── │ FIR      │ ── │ VM  │
│  (this spec)  │     │ → AST        │     │ → AST     │     │ → Bytecode│    │ Exec│
└───────────────┘     └──────────────┘     └───────────┘     └──────────┘     └─────┘
```

### Statistics

| Metric | Value |
|--------|-------|
| Supported code block dialects | 8+ (flux, flux-type, fluxfn, json, yaml, toml, python, c, rust, …) |
| Frontmatter keys | 6 (flux, agent, trust, tiles, capabilities, dependencies) |
| AST node types | 10 (FluxModule, Heading, Paragraph, CodeBlock variants, ListBlock, ListItem, AgentDirective, FluxTypeError) |
| Directive types | 2 (`## agent:`, `## fn:`) |

---

## 2. File Structure

A `.flux.md` file consists of two regions: optional YAML frontmatter followed by
the markdown body.

```
┌────────────────────────────┐
│  YAML Frontmatter          │  (optional, `---` delimited)
│  key: value pairs          │
├────────────────────────────┤
│  Markdown Body             │  (required)
│  ├── Headings              │
│  ├── Paragraphs            │
│  ├── Code Blocks           │
│  ├── Lists                 │
│  └── Directives            │
└────────────────────────────┘
```

### YAML Frontmatter

An optional `---` delimited block at the start of the file. **Regex:**
`^---[ \t]*\n(.*?)\n---[ \t]*(?:\n|$)` (DOTALL).

The parser is lightweight — it handles flat `key: value` pairs with automatic
type coercion. Multi-line YAML values are returned as raw strings. Lines starting
with `#` and blank lines within frontmatter are ignored.

#### Supported Keys

| Key | Type | Description | Example |
|-----|------|-------------|---------|
| `flux` | `string` | FLUX spec version | `"1.0"` |
| `agent` | `string` | Agent name/identifier | `"navigator"` |
| `trust` | `float` (0.0–1.0) | Trust level for fleet coordination | `0.85` |
| `tiles` | `list[string]` | Tile identifiers the agent occupies | `["nav-core", "path-plan"]` |
| `capabilities` | `list[string]` | Capability tokens the agent holds | `["read", "network"]` |
| `dependencies` | `list[string]` | Module names this file depends on | `["math.flux.md"]` |

#### Type Coercion

| Pattern | Coerced Type |
|---------|-------------|
| `true` / `yes` (case-insensitive) | `bool` → `True` |
| `false` / `no` (case-insensitive) | `bool` → `False` |
| Valid integer literal | `int` |
| Valid float literal | `float` |
| Everything else | `str` |

#### Example

```yaml
---
flux: 1.0
agent: navigator
trust: 0.85
---
```

### Minimal Valid File

```markdown
---
flux: 1.0
agent: hello-world
trust: 1.0
---

# Hello World

A minimal FLUX module with inline documentation.

## fn: greet(name: string) -> string

```python
def greet(name):
    return "Hello, " + name + "!"
```
```

---

## 3. Parsing Rules

The parser processes source in four steps: (1) extract frontmatter, (2) parse
body into raw AST, (3) classify code blocks, (4) extract agent directives.

### Line-Level Priority

Lines are tested against patterns in strict order:

| Priority | Pattern | Node Produced |
|----------|---------|---------------|
| 1 | Blank line | Skipped |
| 2 | Fence open `` ``` `` or `~~~` | Code block parsing |
| 3 | Heading `#{1,6}` | `Heading` |
| 4 | Unordered list `- ` / `* ` / `+ ` | List parsing |
| 5 | Ordered list `\d+. ` | List parsing |
| 6 | Everything else | Paragraph |

### Regex Patterns

| Element | Pattern | Notes |
|---------|---------|-------|
| Heading | `^(#{1,6})\s+(.+)$` | Group 1 = level, Group 2 = text |
| Fence open | `^(`{3,}\|~{3,})([\w-]*)\s*(.*?)$` | Group 2 = lang, Group 3 = meta |
| Unordered list | `^(\s*)[-*+]\s+(.+)$` | Group 1 = indent, Group 2 = text |
| Ordered list | `^(\s*)\d+\.\s+(.+)$` | Group 1 = indent, Group 2 = text |
| Agent directive | `^##\s+(?:agent\|fn)\s*[:\s]\s*(.+)$` | Case-insensitive |

### Code Fences

The closing fence is the first subsequent line **starting with** the same fence
characters. Unterminated fences produce a `FluxTypeError`.

### Lists

Consecutive items of the same type (ordered or unordered) are grouped. A single
blank line between same-type items does **not** break the list. A different type
or non-list element terminates the block.

### Paragraphs

Fallback: non-blank, non-heading, non-fence, non-list lines are accumulated.
Lines are joined with spaces. Paragraphs break at blank lines, headings, fences,
or lists.

---

## 4. Directive Sections

### Overview

Directive sections use level-2 headings with `agent:` or `fn:` prefix to declare
agents and functions. During parsing step 4, these headings are extracted into
`AgentDirective` AST nodes with collected body content.

### Recognition

**Regex:** `^##\s+(?:agent|fn)\s*[:\s]\s*(.+)$` (case-insensitive)

Valid forms: `## agent: navigator`, `## fn: compute`, `## fn process(x: i32) -> i32`

### Argument Forms (BNF)

```ebnf
<directive_args>   ::= <simple_name> | <flags> | <signature>

<simple_name>      ::= <word>                        ; no ( , or ->
<flags>            ::= <word> ("," <word>)*           ; has , but no ( or ->
<signature>        ::= <word>? "(" <params>? ")" ("->" <type>)?
<params>           ::= <param> ("," <param>)*
<param>            ::= <word> ":" <type>
```

**Detection logic:** `(` or `->` present → signature; `,` present → flags; otherwise → simple name.

| Form | Input | Parsed `args` |
|------|-------|---------------|
| Simple name | `cleanup` | `{"name": "cleanup"}` |
| Flags | `hot-path, vectorize` | `{"flags": ["hot-path", "vectorize"]}` |
| Signature | `cross_product(a: Vec4, b: Vec4) -> Vec4` | `{"signature": "cross_product(a: Vec4, b: Vec4) -> Vec4", "name": "cross_product"}` |

For signatures, the function name is extracted via `(\w+)\s*\(`. If absent, only
`"signature"` is set.

### Body Collection Rules

After a directive heading, subsequent nodes are collected as body until:

1. **Another directive heading** — any `## agent:` or `## fn:`
2. **Same-or-higher heading** — level ≤ 2 (only H1 and H2)
3. **Non-collectable node** — anything that is not a code block or paragraph

**Collected:** `FluxCodeBlock`, `DataBlock`, `NativeBlock`, `Paragraph`
**Terminates:** `ListBlock`, `ListItem`, `Heading` (at same/higher level)

### Full Directive BNF

```ebnf
<directive>        ::= "##" " " ("agent" | "fn") (":" | " ") <directive_args>
                       <newline> <directive_body>

<directive_body>   ::= (<code_block> | <paragraph>)*
<code_block>       ::= "```" <lang> <newline> <content> <newline> "```"
<paragraph>        ::= <text_line> (<newline> <text_line>)*
```

---

## 5. Code Block Dialects

### Classification Sets

**FLUX dialects:** `{"flux", "flux-type", "fluxfn"}`
**Data dialects:** `{"json", "yaml", "toml", "yml"}`

All other tags → `NativeBlock`.

| Tag | AST Type | Category |
|-----|----------|----------|
| `flux` | `FluxCodeBlock` | FLUX assembly instructions |
| `flux-type` | `FluxCodeBlock` | Type definitions |
| `fluxfn` | `FluxCodeBlock` | Function definitions |
| `json`, `yaml`, `yml`, `toml` | `DataBlock` | Structured data |
| `python` | `NativeBlock` | Compiled via Python frontend |
| `c` | `NativeBlock` | Compiled via C frontend |
| `rust`, `*` | `NativeBlock` | Pass-through (not compiled) |

### `FluxCodeBlock` — FLUX Assembly

Contains FLUX/FIR-level code. Not independently compiled from `.flux.md` in the
current implementation; serves as inline FIR documentation.

````markdown
```flux
%ptr = alloca Vec3
store %v, %ptr
%result = load Vec3, %ptr
```
````

### `DataBlock` — Structured Data

Contains JSON, YAML, or TOML data. Embedded in the AST for tooling and testing.
Not compiled to bytecode.

````markdown
```json
{"sensors": ["lidar", "radar"], "rate_hz": 100}
```
````

### `NativeBlock` — Compiled Source Code

Python and C blocks are compiled through their respective frontends to FIR and
then to bytecode. Other languages are parsed but not compiled.

**Python** (`PythonFrontendCompiler`): `def`, assignments, arithmetic,
comparison, `if`/`elif`/`else`, `while`, `for`/`range`, `return`, `print`,
calls, literals.

**C** (`CFrontendCompiler`): `int`/`float` functions, variables, arithmetic,
comparison, `if`/`else`, `while`, `for`, `return`, calls, literals.

### Code Block Metadata

The `meta` field captures everything after the language tag on the opening fence:

````markdown
```python title="fib" dataflow="pure"
def fib(n): ...
```
````

Here `meta = 'title="fib" dataflow="pure"'`.

---

## 6. AST Node Reference

### Node Hierarchy

```
LocatedNode (abstract base — carries SourceSpan)
├── FluxModule          — Root: frontmatter + children
├── Heading             — Markdown heading (level 1–6)
├── Paragraph           — Plain text paragraph
├── CodeBlock           — Generic fenced code block (intermediate)
│   ├── FluxCodeBlock   — FLUX dialect (flux, flux-type, fluxfn)
│   ├── DataBlock       — Data dialect (json, yaml, toml)
│   └── NativeBlock     — Other dialect (python, c, rust, …)
├── ListBlock           — Ordered or unordered list
├── ListItem            — Single list item
├── AgentDirective      — ## agent: / ## fn: directive
└── FluxTypeError       — Parse error node
```

### `SourceSpan`

All coordinates **1-indexed**. Immutable (`frozen=True`).

| Field | Type | Description |
|-------|------|-------------|
| `line_start` | `int` | First line |
| `line_end` | `int` | Last line (inclusive) |
| `col_start` | `int` | First column |
| `col_end` | `int` | Last column (inclusive) |

### Node Definitions

**`FluxModule`** — Root of every parsed document.

| Field | Type | Description |
|-------|------|-------------|
| `frontmatter` | `dict[str, Any]` | Parsed YAML key-value pairs |
| `children` | `list[LocatedNode]` | Ordered body nodes |

**`Heading`** — Markdown heading.

| Field | Type | Description |
|-------|------|-------------|
| `level` | `int` | 1–6 |
| `text` | `str` | Content without `#` prefix |

**`Paragraph`** — Joined text lines.

| Field | Type | Description |
|-------|------|-------------|
| `text` | `str` | Lines joined with spaces |

**`CodeBlock`** — Base for all fenced code blocks.

| Field | Type | Description |
|-------|------|-------------|
| `lang` | `str` | Language tag from fence |
| `content` | `str` | Raw content between fences |
| `meta` | `str` | Metadata after language tag |

**`FluxCodeBlock(CodeBlock)`** — FLUX dialect. No additional fields.
**`DataBlock(CodeBlock)`** — Data dialect. No additional fields.
**`NativeBlock(CodeBlock)`** — Compiled language. No additional fields.

**`ListBlock`** — List container.

| Field | Type | Description |
|-------|------|-------------|
| `ordered` | `bool` | `True` for `1.` style, `False` for `-` style |
| `items` | `list[ListItem]` | Ordered items |

**`ListItem`** — Single list item.

| Field | Type | Description |
|-------|------|-------------|
| `text` | `str` | Item text after bullet/number |
| `children` | `list[LocatedNode]` | Nested sub-items (default: `[]`) |

**`AgentDirective`** — Extracted `## agent:` / `## fn:` with body.

| Field | Type | Description |
|-------|------|-------------|
| `name` | `str` | Directive kind: `"agent"` or `"fn"` |
| `args` | `dict[str, Any]` | Parsed arguments (see Section 4) |
| `body` | `list[LocatedNode]` | Collected body nodes (default: `[]`) |

`args` shapes: `{"name": str}` | `{"flags": list[str]}` | `{"signature": str, "name"?: str}`

**`FluxTypeError`** — Parse error.

| Field | Type | Description |
|-------|------|-------------|
| `message` | `str` | Human-readable error description |

---

## 7. Compilation Pipeline

`FluxCompiler.compile_md()` converts `.flux.md` source to FLUX bytecode:

```
┌────────────────────────────────────────────────────────────────┐
│  Step 1: Parse     source → FluxMDParser → FluxModule AST      │
│  Step 2: Extract   frontmatter metadata (not yet used)         │
│  Step 3: Find      NativeBlock nodes where lang ∈ {c, python}  │
│  Step 4: Compile   first NativeBlock → Frontend → FIRModule    │
│  Step 5: Encode    FIRModule → BytecodeEncoder → bytes         │
└────────────────────────────────────────────────────────────────┘
```

### Steps

**Step 1 — Parse:** `FluxMDParser.parse(source)` → `FluxModule`. Performs all
four parsing phases. Errors accumulated in `parser.errors`.

**Step 2 — Extract metadata:** `FluxModule.frontmatter` is available but not
used by the current compiler (future: embed trust/capabilities in bytecode).

**Step 3 — Find native blocks:** Iterates `doc.children` (top-level only;
`AgentDirective.body` is **not** traversed). Filters to `lang` in `{"c", "python"}`.

**Step 4 — Compile first block:** Only the **first** compilable `NativeBlock`
is compiled. C → `CFrontendCompiler`, Python → `PythonFrontendCompiler`.

**Step 5 — Encode:** `FIRModule` → `BytecodeEncoder.encode()` → `bytes`.

### Empty Module

If no compilable blocks are found, a minimal empty `FIRModule` is produced and
encoded to valid (empty) bytecode.

### Known Limitations

| Limitation | Description |
|------------|-------------|
| Single-block | Only the first `NativeBlock` is compiled |
| No directive body compilation | Code inside `AgentDirective.body` not compiled |
| Frontmatter unused | Metadata parsed but not embedded in bytecode |
| Two frontends only | Only C and Python supported |
| No link resolution | `dependencies` list not resolved |

---

## 8. Complete Working Example

```markdown
---
flux: 1.0
agent: vector-math
trust: 0.95
tiles:
  - simd-core
  - math-lib
capabilities:
  - read
  - compute
---

# Vector Math Library

A FLUX module providing 3D and 4D vector operations for the
navigation subsystem. Documentation and executable code coexist.

## fn: dot_product(a: Vec4, b: Vec4) -> f32

Computes the dot product of two 4D vectors using element-wise
multiplication and summation.

```python
def dot_product(a, b):
    result = 0.0
    for i in range(4):
        result = result + a[i] * b[i]
    return result
```

The dot product is the core primitive for projection and lighting.

## fn: cross_product(a: Vec3, b: Vec3) -> Vec3

Computes the cross product of two 3D vectors.

```python
def cross_product(a, b):
    x = a[1] * b[2] - a[2] * b[1]
    y = a[2] * b[0] - a[0] * b[2]
    z = a[0] * b[1] - a[1] * b[0]
    return (x, y, z)
```

### Type Reference

```flux-type
type Vec3 = struct { x: f32, y: f32, z: f32 }
type Vec4 = struct { x: f32, y: f32, z: f32, w: f32 }
```

## agent: navigator

The navigator agent uses vector math for path planning.

### Configuration

```json
{
  "max_speed": 2.5,
  "update_rate_hz": 50
}
```

### Initialization

```c
int navigator_init() {
    int status = init_sensor_fusion();
    if (status != 0) {
        return -1;
    }
    set_update_rate(50);
    return 0;
}
```
```

### Parse Tree (Abbreviated)

```
FluxModule
├── frontmatter: {"flux": "1.0", "agent": "vector-math", "trust": 0.95, ...}
├── Heading(1, "Vector Math Library")
├── Paragraph("A FLUX module providing...")
├── AgentDirective("fn", {"signature": "dot_product(a: Vec4, b: Vec4) -> f32", "name": "dot_product"})
│   ├── Paragraph("Computes the dot product...")
│   └── NativeBlock(lang="python", ...)
├── Paragraph("The dot product is the core...")
├── AgentDirective("fn", {"signature": "cross_product(a: Vec3, b: Vec3) -> Vec3", "name": "cross_product"})
│   └── NativeBlock(lang="python", ...)
├── FluxCodeBlock(lang="flux-type", ...)
├── AgentDirective("agent", {"name": "navigator"})
│   ├── Paragraph("The navigator agent uses...")
│   ├── DataBlock(lang="json", ...)
│   └── NativeBlock(lang="c", ...)
```

### Compilation Path

The compiler searches top-level `FluxModule.children` for `NativeBlock` nodes
with `lang` of `"c"` or `"python"`. In this example, top-level native blocks
are absent (they live inside directive bodies), so the compiler would produce
an empty module. Moving a code block to the top level triggers compilation:

```
Python source → PythonFrontendCompiler → FIRModule → BytecodeEncoder → bytes
```

---

*Specification v1.0 — Conforms to flux-runtime parser v1.0, FIR v1.0, FLUX ISA v1.0.*
