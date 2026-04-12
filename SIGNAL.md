# Signal Language Specification v1.0

**Version:** 1.0
**Status:** Draft
**Date:** 2026-04-12
**Author:** Super Z ⚡

---

## 1. Overview

Signal is an **agent-first-class JSON language** for multi-agent coordination and computation. It is the high-level interface to the FLUX bytecode VM — any agent can write programs in Signal, compile them to FLUX bytecode, and execute them on any FLUX runtime.

### Design Principles

1. **JSON IS the AST.** There is no separate parse step. A Signal program is a JSON object. Every tool that can read JSON can read Signal. Every agent that can produce JSON can write Signal programs.

2. **Agent-first.** Communication primitives (tell, ask, delegate, broadcast) are first-class operations, not library calls. Multi-agent coordination (branch, fork, co-iterate, discuss, synthesize, reflect) is built into the language.

3. **Bytecode-compilable.** Signal compiles to FLUX FORMAT_A-G bytecodes via the SignalCompiler. The compilation is deterministic and produces a source map for debugging.

4. **Confidence-native.** Every operation can carry a confidence score. Uncertainty is not an error condition — it is a first-class value that propagates through computation.

5. **Schema-versioned.** Every protocol primitive carries a `$schema` field for forward/backward compatibility. Unknown fields go into `meta`, not errors.

### Relationship to Other Specifications

```
Signal Language (this document)
    │
    ├── compiles to → FLUX Bytecode (ISA.md)
    ├── uses types from → FIR.md (type system)
    ├── transport via → A2A Protocol (A2A.md)
    └── primitives defined in → flux-a2a-prototype/protocol.py
```

### File Extension

Signal programs use the `.signal.json` extension. The language ID for tooling is `signal`.

---

## 2. Program Structure

A Signal program is a JSON object with the following top-level fields:

```json
{
  "program": "program-name",
  "lang": "signal",
  "$schema": "flux.signal/v1",
  "version": "1.0",
  "meta": {
    "author": "Agent Name",
    "description": "What this program does",
    "created": "2026-04-12T00:00:00Z"
  },
  "ops": [
    { "op": "...", ... },
    ...
  ]
}
```

### 2.1 Top-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `program` | string | Yes | Program identifier. Must be unique within a deployment. |
| `lang` | string | Yes | Always `"signal"`. |
| `$schema` | string | No | Schema version URI. Recommended: `"flux.signal/v1"`. |
| `version` | string | No | Program version. Semantic versioning recommended. |
| `meta` | object | No | Arbitrary metadata. Author, description, created, tags. |
| `ops` | array | Yes | Ordered list of Signal operations. Execution is sequential unless control flow ops (if, loop, branch, fork) alter flow. |

### 2.2 Operation Object

Every element in the `ops` array is a JSON object with at minimum an `"op"` field:

```json
{
  "op": "operation-name",
  "...": "operation-specific fields"
}
```

Operations execute in array order unless altered by control flow. Each operation may produce side effects (agent messages, register mutations, branch spawning) and/or return a value.

---

## 3. Data Binding

### 3.1 let — Bind a Value

Binds a name to a value in the program's register space. The compiler allocates a register and stores the value.

```json
{ "op": "let", "name": "x", "value": 42 }
{ "op": "let", "name": "greeting", "value": "hello world" }
{ "op": "let", "name": "pi", "value": 3.14159 }
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Variable name. Must be unique per program. |
| `value` | any | Yes | Value to bind. Integer, float, string, boolean, or reference to another name. |

**Bytecode:** Integer values in [-128, 127] emit `MOVI` (Format D, 2 bytes). Larger values emit `MOVI16` (Format F, 3 bytes). String values emit `MOV` (Format E, 3 bytes) as register references.

**Semantics:** `let` creates a binding in the register allocator. If `name` was previously bound, the existing register is reused (no new allocation). Values are immutable — subsequent `let` with the same name is an error unless `set` is used.

### 3.2 get — Read a Value

Reads the current value of a previously bound name into a new register.

```json
{ "op": "get", "name": "x", "into": "x_copy" }
```

### 3.3 set — Update a Value

Updates the value of a previously bound name.

```json
{ "op": "set", "name": "x", "value": 100 }
```

---

## 4. Arithmetic Operations

### 4.1 add, sub, mul, div, mod

Binary arithmetic operations that support multi-argument chaining. The result is stored in the register named by `into` (or auto-generated if omitted).

```json
{ "op": "add", "args": [3, 5], "into": "sum" }
{ "op": "mul", "args": ["x", "y", "z"], "into": "product" }
{ "op": "div", "args": ["total", "count"], "into": "average" }
{ "op": "mod", "args": ["n", 2], "into": "parity" }
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `args` | array | Yes | Two or more arguments. Each is an integer literal or a name referencing a bound value. |
| `into` | string | No | Name for the result register. Auto-generated if omitted. |

**Multi-argument semantics:** With 2 args, emits one binary operation. With 3+ args, emits chained binary operations: `add(a, b, c)` → `R = a + b; R = R + c`.

**Bytecode:**

| Signal | Opcode | Format |
|--------|--------|--------|
| `add` | 0x20 | E (rd, rs1, rs2) |
| `sub` | 0x21 | E |
| `mul` | 0x22 | E |
| `div` | 0x23 | E |
| `mod` | 0x24 | E |

**Errors:** Division by zero produces a runtime fault. `mod` by zero produces a runtime fault.

---

## 5. Comparison Operations

### 5.1 eq, neq, lt, lte, gt, gte

Binary comparison operations that produce a boolean result (0 = false, nonzero = true).

```json
{ "op": "eq", "args": ["x", 0], "into": "is_zero" }
{ "op": "gt", "args": ["score", "threshold"], "into": "above" }
{ "op": "lte", "args": ["count", 10], "into": "within_limit" }
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `args` | array | Yes | Exactly 2 arguments. Each is a literal or bound name. |
| `into` | string | No | Name for the result register. Auto-generated if omitted. |

**Bytecode:**

| Signal | Opcode | Format |
|--------|--------|--------|
| `eq` | 0x2C | E (rd, rs1, rs2) |
| `neq` | 0x2F | E |
| `lt` | 0x2D | E |
| `gt` | 0x2E | E |
| `lte` | 0x2D | E (same as lt, with swapped args or adjusted semantics) |
| `gte` | 0x2E | E (same as gt, with swapped args) |

---

## 6. Logic Operations

### 6.1 and, or, not, xor

Bitwise and logical operations.

```json
{ "op": "and", "args": ["flag_a", "flag_b"], "into": "both" }
{ "op": "or", "args": ["x", "y"], "into": "either" }
{ "op": "not", "args": ["is_zero"], "into": "is_nonzero" }
{ "op": "xor", "args": ["a", "b"], "into": "diff" }
```

**Bytecode:**

| Signal | Opcode | Format |
|--------|--------|--------|
| `and` | 0x25 | E |
| `or` | 0x26 | E |
| `not` | 0x0A | B (rd) |
| `xor` | 0x27 | E |

---

## 7. Agent Communication Operations

These are Signal's first-class agent communication primitives. They compile to FLUX A2A opcodes (0x50-0x5B).

### 7.1 tell — Send Information

Send a message to a specific agent.

```json
{ "op": "tell", "to": "oracle1", "what": "status report ready" }
{ "op": "tell", "to": "jetsonclaw1", "what": "cuda kernel compiled", "tag": "build" }
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | Yes | Target agent identifier. |
| `what` | string | Yes | Message content or data reference. |
| `tag` | string | No | Optional tag for categorization. Stored in the message header. |

**Bytecode:** `TELL (0x50)`, Format E (rd, rs1=agent, rs2=data).

**Semantics:** Sends a one-way message. No response expected. The message enters the target agent's mailbox. If the target is not running, the message is queued.

### 7.2 ask — Request Information

Send a request to an agent and await a response.

```json
{ "op": "ask", "from": "oracle1", "what": "current fleet status", "into": "status" }
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from` | string | Yes | Source agent identifier. (`to` is accepted as an alias.) |
| `what` | string | Yes | Query content. |
| `into` | string | No | Name for the response register. Auto-generated if omitted. |

**Bytecode:** `ASK (0x51)`, Format E (rd=response, rs1=agent, rs2=query).

**Semantics:** Synchronous request-response. The requesting agent blocks until the target responds or the request times out. The response is stored in the `into` register.

### 7.3 delegate — Assign a Task

Assign a computational task to another agent.

```json
{ "op": "delegate", "to": "jetsonclaw1", "task": "compile cuda kernel" }
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | Yes | Target agent identifier. |
| `task` | string | Yes | Task description or task reference. |

**Bytecode:** `DELEG (0x52)`, Format E (rd, rs1=agent, rs2=task).

**Semantics:** Fire-and-forget task assignment. The delegating agent continues execution. The target agent receives the task and processes it asynchronously. No response is awaited.

### 7.4 broadcast — Send to All

Send a message to all listening agents.

```json
{ "op": "broadcast", "what": "fleet census complete", "tag": "announcement" }
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `what` | string | Yes | Message content. |
| `tag` | string | No | Optional tag. |

**Bytecode:** `BCAST (0x53)`, Format E (rd, rs1=0 (all), rs2=data).

**Semantics:** Sends to all agents subscribed to the broadcast channel. No response expected.

---

## 8. Control Flow

### 8.1 seq — Sequential Execution

Execute a list of operations in order. This is the default execution model, but `seq` is useful for nesting operations within control structures.

```json
{
  "op": "seq",
  "body": [
    { "op": "let", "name": "x", "value": 1 },
    { "op": "add", "args": ["x", 1], "into": "x" },
    { "op": "tell", "to": "log", "what": "incremented" }
  ]
}
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `body` | array | Yes | Operations to execute in sequence. |

**Bytecode:** Operations are compiled inline in order. No control flow bytes emitted.

### 8.2 if — Conditional Execution

Execute a block based on a condition.

```json
{
  "op": "if",
  "cond": "is_ready",
  "then": [
    { "op": "tell", "to": "fleet", "what": "system ready" }
  ],
  "else": [
    { "op": "tell", "to": "fleet", "what": "system not ready" }
  ]
}
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `cond` | string | Yes | Name of a register containing the condition (0 = false, nonzero = true). |
| `then` | array | Yes | Operations to execute if condition is true. |
| `else` | array | No | Operations to execute if condition is false. |

**Bytecode:** `CMP_EQ` + `JZ` to else block, `JMP` over else block, with back-patched relative offsets.

### 8.3 loop — Counted Iteration

Execute a block a fixed number of times.

```json
{
  "op": "loop",
  "count": 10,
  "body": [
    { "op": "tell", "to": "monitor", "what": "iteration" }
  ]
}
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `count` | integer or string | Yes | Number of iterations. Integer literal or name of a bound register. |
| `body` | array | Yes | Operations to execute per iteration. |

**Bytecode:** `MOVI` counter + `LOOP (0x46)` with back-reference offset.

### 8.4 while — Conditional Loop

Repeatedly execute a block while a condition is true. (Currently implemented as an alias for `if` in the compiler — full while support is planned.)

```json
{
  "op": "while",
  "cond": "has_work",
  "body": [
    { "op": "delegate", "to": "worker", "task": "process next" }
  ]
}
```

---

## 9. Parallelism and Concurrency

### 9.1 branch — Parallel Exploration

Spawn multiple parallel execution paths and merge results.

```json
{
  "op": "branch",
  "strategy": "parallel",
  "branches": [
    {
      "label": "fast-path",
      "weight": 1.0,
      "body": [
        { "op": "ask", "from": "cache", "what": "result", "into": "cached" }
      ]
    },
    {
      "label": "slow-path",
      "weight": 0.5,
      "body": [
        { "op": "delegate", "to": "compute", "task": "calculate" }
      ]
    }
  ],
  "merge": {
    "strategy": "weighted_confidence",
    "timeout_ms": 30000,
    "fallback": "first_complete"
  }
}
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `strategy` | string | No | `"parallel"` (default), `"sequential"`, or `"competitive"`. |
| `branches` | array | Yes | Array of branch bodies. |
| `branches[].label` | string | No | Branch identifier. |
| `branches[].weight` | float | No | Confidence weight for merge (0.0-1.0, default 1.0). |
| `branches[].body` | array | Yes | Operations in this branch. |
| `merge.strategy` | string | No | `"weighted_confidence"` (default), `"consensus"`, `"vote"`, `"best"`, `"all"`, `"first_complete"`, `"last_writer_wins"`. |
| `merge.timeout_ms` | integer | No | Merge timeout in milliseconds (default 30000). |
| `merge.fallback` | string | No | Fallback merge strategy on timeout (default `"first_complete"`). |

**Bytecode:** `FORK (0x58)` per branch + `JOIN (0x59)` for synchronization + `MERGE (0x57)` for result combination.

**Semantics:**
- **parallel:** All branches execute concurrently. Merge waits for all to complete (or timeout).
- **sequential:** Branches execute in order. Each sees the results of previous branches.
- **competitive:** All branches race. Only the first to complete is used (others are cancelled).

### 9.2 fork — Agent Inheritance

Create a child agent that inherits specified state from the parent.

```json
{
  "op": "fork",
  "id": "explorer-1",
  "inherit": {
    "state": ["knowledge_base", "trust_graph"],
    "context": true,
    "trust_graph": true,
    "message_history": false
  },
  "mutations": [
    { "type": "strategy", "changes": { "risk_tolerance": "high" } }
  ],
  "on_complete": "merge",
  "conflict_mode": "negotiate"
}
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | No | Fork identifier. Auto-generated UUID if omitted. |
| `inherit.state` | array | No | List of register names to inherit. Default: all. |
| `inherit.context` | boolean | No | Inherit execution context (default true). |
| `inherit.trust_graph` | boolean | No | Inherit trust relationships (default false). |
| `inherit.message_history` | boolean | No | Inherit message history (default false). |
| `mutations` | array | No | Changes to apply to the forked agent. |
| `mutations[].type` | string | No | Mutation type: `"prompt"`, `"context"`, `"strategy"`, `"capability"`. |
| `mutations[].changes` | object | No | Key-value pairs describing the mutation. |
| `on_complete` | string | No | `"collect"` (default), `"discard"`, `"signal"`, `"merge"`. |
| `conflict_mode` | string | No | `"parent_wins"`, `"child_wins"` (default), `"negotiate"`. |

**Bytecode:** `FORK (0x58)` + state serialization + `JOIN (0x59)` + conflict resolution.

### 9.3 merge — Join Results

Combine results from parallel branches with a merge strategy.

```json
{
  "op": "merge",
  "strategy": "consensus",
  "into": "final_result"
}
```

**Bytecode:** `MERGE (0x57)`, Format E.

---

## 10. Structured Agent Discourse

### 10.1 discuss — Multi-Agent Discussion

Facilitate a structured discussion between multiple agents.

```json
{
  "op": "discuss",
  "id": "design-review",
  "format": "peer_review",
  "topic": "Should we use binary or JSON for A2A messages?",
  "participants": [
    { "agent": "oracle1", "stance": "pro", "role": "moderator" },
    { "agent": "superz", "stance": "neutral", "role": "reviewer" },
    { "agent": "jetsonclaw1", "stance": "con", "role": "reviewer" }
  ],
  "turn_order": "round_robin",
  "until": { "condition": "consensus", "max_rounds": 5 },
  "confidence": 0.8
}
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | No | Discussion identifier. Auto-generated UUID if omitted. |
| `format` | string | No | `"debate"`, `"brainstorm"`, `"review"`, `"negotiate"`, `"peer_review"`. |
| `topic` | string | Yes | The question or topic under discussion. |
| `participants` | array | Yes | List of participating agents. |
| `participants[].agent` | string | Yes | Agent identifier. |
| `participants[].stance` | string | No | `"pro"`, `"con"`, `"neutral"`, `"devil's_advocate"`, `"moderator"`. |
| `participants[].role` | string | No | Participant role description. |
| `turn_order` | string | No | `"round_robin"` (default), `"priority"`, `"free_for_all"`, `"moderated"`. |
| `until.condition` | string | No | `"consensus"`, `"timeout"`, `"rounds"`, `"majority"`, `"best_argument"`. |
| `until.max_rounds` | integer | No | Maximum discussion rounds (when condition is "rounds"). |

**Semantics:** The discussion primitive orchestrates multi-turn agent discourse. Each participant contributes in turn order. The discussion terminates when the `until` condition is met. The result is a summary with consensus level and key arguments.

### 10.2 synthesize — Result Combination

Combine multiple results into a unified output.

```json
{
  "op": "synthesize",
  "method": "map_reduce",
  "sources": [
    { "type": "branch_result", "ref": "exploration" },
    { "type": "external", "ref": "human_feedback" }
  ],
  "output_type": "decision",
  "confidence_mode": "propagate"
}
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `method` | string | No | `"map_reduce"` (default), `"ensemble"`, `"chain"`, `"vote"`, `"weighted_merge"`, `"best_effort"`. |
| `sources` | array | Yes | Input sources. |
| `sources[].type` | string | No | `"branch_result"`, `"fork_result"`, `"discuss_result"`, `"external"`, `"variable"`. |
| `sources[].ref` | string | Yes | Reference to the source data. |
| `output_type` | string | No | `"code"`, `"spec"`, `"question"`, `"decision"`, `"summary"`, `"value"`. |
| `confidence_mode` | string | No | `"propagate"` (default), `"min"`, `"max"`, `"average"`. |

**Semantics:**
- **map_reduce:** Apply a map function to each source, then reduce to single output.
- **ensemble:** Combine outputs using weighted averaging (for numeric results).
- **chain:** Pipe outputs sequentially through transformation stages.
- **vote:** Majority rule across discrete choices.
- **weighted_merge:** Confidence-weighted combination of all sources.

### 10.3 reflect — Meta-Cognition

Enable an agent to reason about its own state and strategy.

```json
{
  "op": "reflect",
  "target": "strategy",
  "method": "introspection",
  "output": "adjustment",
  "confidence": 0.7
}
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `target` | string | No | `"strategy"`, `"progress"`, `"uncertainty"`, `"confidence"`, `"all"`. |
| `method` | string | No | `"introspection"` (default), `"benchmark"`, `"comparison"`, `"statistical"`. |
| `output` | string | No | `"adjustment"`, `"question"`, `"branch"`, `"log"`, `"signal"`. |

**Semantics:** The reflect primitive enables agents to examine and modify their own behavior. It is the basis for self-improvement loops — an agent can reflect on its strategy, identify weaknesses, and branch into a modified approach.

---

## 11. Asynchronous Operations

### 11.1 yield — Suspend Execution

Temporarily suspend the current agent's execution, yielding the CPU to other agents.

```json
{ "op": "yield", "cycles": 1 }
{ "op": "yield" }
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `cycles` | integer | No | Number of cycles to yield (default 1). |

**Bytecode:** `YIELD (0x15)`, Format C (opcode, cycles).

### 11.2 await — Wait for Signal

Block until a specific signal or result is available.

```json
{ "op": "await", "signal": "cuda-ready", "into": "cuda_result" }
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `signal` | string | Yes | Signal name to wait for. |
| `into` | string | No | Name for the result register. |

**Bytecode:** `AWAIT (0x5B)`, Format E.

---

## 12. Confidence Operations

### 12.1 confidence — Set Confidence Threshold

Set the confidence level for a register value. Affects how uncertainty propagates through subsequent operations.

```json
{ "op": "confidence", "for": "result", "level": 0.85 }
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `for` | string | Yes | Name of the register whose confidence is being set. |
| `level` | float | Yes | Confidence level (0.0-1.0). Encoded as uint8 (level * 255). |

**Bytecode:** `C_THRESH (0x69)`, Format D (opcode, rd, imm8).

**Semantics:** Confidence is monotonically decreasing through computation chains. A `confidence` operation sets the initial confidence for a value. Subsequent operations may reduce it (e.g., division by uncertain values, data from untrusted agents). The minimum confidence across all inputs propagates to outputs.

---

## 13. Co-Iteration (Advanced)

### 13.1 co_iterate — Multi-Agent Shared Traversal

Multiple agents traverse the same program simultaneously, with configurable conflict detection and convergence checking.

```json
{
  "op": "co_iterate",
  "id": "code-review",
  "agents": ["superz", "oracle1", "jetsonclaw1"],
  "shared_state_mode": "merge",
  "conflict_resolution": "vote",
  "convergence": {
    "metric": "agreement",
    "threshold": 0.95
  },
  "merge_type": "trust_weighted"
}
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | No | Co-iteration identifier. |
| `agents` | array | Yes | List of participating agent identifiers. |
| `shared_state_mode` | string | No | `"conflict"`, `"merge"` (default), `"partitioned"`, `"isolated"`. |
| `conflict_resolution` | string | No | `"priority"`, `"vote"` (default), `"last_writer"`, `"reject"`, `"branch"`. |
| `convergence.metric` | string | No | `"agreement"` (default), `"confidence_delta"`, `"value_stability"`. |
| `convergence.threshold` | float | No | Convergence threshold (0.0-1.0, default 0.95). |
| `merge_type` | string | No | `"sequential_consensus"`, `"parallel_merge"`, `"majority_vote"`, `"trust_weighted"` (default). |

**Semantics:** Co-iteration enables collaborative processing of a shared program. Each agent traverses the same operations but may produce different intermediate results. The conflict_resolution strategy determines how divergent results are handled. The convergence metric determines when iteration can stop.

---

## 14. Execution Modes

Signal programs support three execution modes, ranging from interpreted to self-modifying:

| Mode | Description | Maturity |
|------|-------------|----------|
| `script` | Interpret Signal JSON directly, no compilation | Stable |
| `compile` | Compile to FLUX bytecode, then execute on VM | Stable |
| `meta_compile` | Compile a program that produces programs (self-improving) | Research |

The default mode is `compile`. The execution mode can be specified in the program metadata:

```json
{
  "program": "self-improver",
  "lang": "signal",
  "meta": { "execution_mode": "meta_compile" },
  "ops": [...]
}
```

---

## 15. Complete Operation Reference

| Category | Operation | Opcode | Format | Page |
|----------|-----------|--------|--------|------|
| **Data** | `let` | MOVI/MOVI16/MOV | D/F/E | Data Binding |
| **Arithmetic** | `add` | 0x20 | E | Arithmetic |
| | `sub` | 0x21 | E | |
| | `mul` | 0x22 | E | |
| | `div` | 0x23 | E | |
| | `mod` | 0x24 | E | |
| **Logic** | `and` | 0x25 | E | Logic |
| | `or` | 0x26 | E | |
| | `not` | 0x0A | B | |
| | `xor` | 0x27 | E | |
| **Comparison** | `eq` | 0x2C | E | Comparison |
| | `neq` | 0x2F | E | |
| | `lt` | 0x2D | E | |
| | `gt` | 0x2E | E | |
| | `lte` | 0x2D | E | |
| | `gte` | 0x2E | E | |
| **Agent Comms** | `tell` | 0x50 | E | Agent Comms |
| | `ask` | 0x51 | E | |
| | `delegate` | 0x52 | E | |
| | `broadcast` | 0x53 | E | |
| **Control Flow** | `seq` | (inline) | — | Control Flow |
| | `if` | CMP_EQ+JZ | E+F | |
| | `loop` | MOVI+LOOP | D+F | |
| | `while` | CMP_EQ+JZ | E+F | |
| **Parallelism** | `branch` | FORK+JOIN+MERGE | E | Parallelism |
| | `fork` | FORK | E | |
| | `merge` | MERGE | E | |
| **Discourse** | `discuss` | (protocol) | — | Discourse |
| | `synthesize` | (protocol) | — | |
| | `reflect` | (protocol) | — | |
| | `co_iterate` | (protocol) | — | |
| **Async** | `yield` | 0x15 | C | Async |
| | `await` | 0x5B | E | |
| **Confidence** | `confidence` | 0x69 | D | Confidence |

---

## 16. Compilation Model

### 16.1 Compiler Pipeline

```
Signal JSON Program
    │
    ├─ Parse (JSON parse — no separate parse step needed)
    │
    ├─ Register Allocation (names → register numbers)
    │
    ├─ Code Generation (ops → FORMAT_A-G bytecodes)
    │
    ├─ Label Resolution (back-patch jumps with relative offsets)
    │
    └─ Emit (append explicit HALT)
```

### 16.2 Register Allocator

The compiler maintains a register map from names to register numbers:
- Names are allocated sequentially: first `let` gets R0, second gets R1, etc.
- Reusing a name reuses the register (no new allocation).
- Register overflow (64 registers exceeded) produces a compilation error.
- System registers (R29=FP, R30=SP, R31=PC) are not available for user allocation.

### 16.3 Source Map

The compiler produces a source map (byte offset → JSON line number) for debugging. If a runtime error occurs at bytecode offset N, the source map identifies which line in the Signal JSON caused it.

### 16.4 CompiledSignal Result

```json
{
  "bytecode": [0x18, 0, 42, ...],
  "register_map": { "x": 0, "y": 1 },
  "label_map": { "_loop_start": 12, "_loop_end": 24 },
  "source_map": { "0": 1, "3": 2, "5": 3 },
  "errors": []
}
```

---

## 17. Examples

### 17.1 Hello World (Agent Communication)

```json
{
  "program": "hello-fleet",
  "lang": "signal",
  "ops": [
    { "op": "tell", "to": "fleet", "what": "Hello from Signal!" }
  ]
}
```

### 17.2 Fibonacci (Computation)

```json
{
  "program": "fibonacci",
  "lang": "signal",
  "ops": [
    { "op": "let", "name": "a", "value": 0 },
    { "op": "let", "name": "b", "value": 1 },
    { "op": "let", "name": "n", "value": 10 },
    { "op": "loop", "count": "n", "body": [
      { "op": "let", "name": "temp", "value": "b" },
      { "op": "add", "args": ["a", "b"], "into": "b" },
      { "op": "set", "name": "a", "value": "temp" }
    ]},
    { "op": "tell", "to": "monitor", "what": "fib(10) complete" }
  ]
}
```

### 17.3 Parallel Research (Branch + Ask)

```json
{
  "program": "parallel-research",
  "lang": "signal",
  "ops": [
    { "op": "branch", "strategy": "parallel", "branches": [
      {
        "label": "cache-check",
        "body": [
          { "op": "ask", "from": "cache", "what": "answer", "into": "cached" }
        ]
      },
      {
        "label": "compute",
        "body": [
          { "op": "delegate", "to": "researcher", "task": "find answer" }
        ]
      }
    ],
    "merge": { "strategy": "first_complete", "timeout_ms": 5000 }}
  ]
}
```

### 17.4 Design Review (Discuss)

```json
{
  "program": "design-review",
  "lang": "signal",
  "ops": [
    { "op": "discuss",
      "format": "peer_review",
      "topic": "ISA opcode allocation for neural ops",
      "participants": [
        { "agent": "oracle1", "stance": "pro" },
        { "agent": "superz", "stance": "neutral" },
        { "agent": "jetsonclaw1", "stance": "con" }
      ],
      "turn_order": "round_robin",
      "until": { "condition": "consensus", "max_rounds": 5 }
    },
    { "op": "synthesize", "method": "vote", "output_type": "decision" }
  ]
}
```

---

## 18. Open Questions

> **Note on Amendment 1:** [SIGNAL-AMENDMENT-1.md](SIGNAL-AMENDMENT-1.md) proposes resolutions
> to these open questions. As of the formal peer review (see [Issue #7](https://github.com/SuperInstance/flux-spec/issues/7)),
> Resolution 1 (Zone Partition) has been APPROVED with deferral of specific range assignments
> to post-convergence. Resolutions 2–6 have been DEFERRED pending further implementation
> work. See SIGNAL-AMENDMENT-1.md for full proposals and the review document for detailed
> verdicts. Once resolutions are approved and merged, the open questions below will be
> updated accordingly.

1. **Opcode conflict at 0x60-0x69:** The flux-a2a-prototype's ISA mapping documents a FATAL collision between A2A opcodes (TELL=0x60, ASK=0x61) and Oracle1's FORMAT spec (CONF_ADD=0x60, CONF_SUB=0x61). The runtime's signal_compiler.py uses 0x50-0x5B for A2A ops, which avoids the conflict. Should the prototype's mapping be updated to match? **→ Resolved by SIGNAL-AMENDMENT-1 Resolution 1 (Zone Partition, APPROVED).**

2. **Protocol primitives bytecode:** The discuss, synthesize, reflect, and co_iterate primitives are defined as dataclasses but do not have direct bytecode encodings. They operate at the protocol layer above the VM. Should they have VM-level opcodes for efficiency? **→ See SIGNAL-AMENDMENT-1 Resolution 2 (DEFERRED — 0x70-0x73 collides with Babel viewpoint ops AND runtime trust ops).**

3. **Error handling:** No formal error handling model exists. Should Signal include try/catch/raise operations? How do runtime errors propagate through branch/fork constructs? **→ See SIGNAL-AMENDMENT-1 Resolution 3 (DEFERRED).**

4. **Type system integration:** Signal currently uses dynamic typing (any value can be in any register). Should it integrate with FIR's 16 type families for static type checking? **→ See SIGNAL-AMENDMENT-1 Resolution 4 (Progressive typing, DEFERRED).**

5. **Network transport:** The current implementation assumes local (in-process) message passing. How should Signal programs address agents across network boundaries? **→ See SIGNAL-AMENDMENT-1 Resolution 5 (Hierarchical URI, DEFERRED).**

6. **Persistence:** Can a Signal program serialize its state to disk and resume later? What does checkpoint/restart look like? **→ See SIGNAL-AMENDMENT-1 Resolution 6 (Checkpoint opcodes, DEFERRED).**

---

## 19. Relationship to flux-a2a-prototype

The flux-a2a-prototype repository contains a richer implementation of the Signal language concepts:

| Feature | flux-runtime (SignalCompiler) | flux-a2a-prototype |
|---------|------------------------------|-------------------|
| Core ops (let, add, tell, etc.) | 32 ops, compiled to bytecode | Interpreter + compiler |
| Protocol primitives (branch, fork, etc.) | Basic (FORK/JOIN/MERGE bytecodes) | Full dataclass implementations |
| Type system | None | FUTS (8 base types, 6 paradigms) |
| Cross-language compilation | None | Bidirectional bridge with Dijkstra routing |
| Testing | In flux-runtime test suite | 184+ tests in 15 files |

**Recommendation:** The protocol primitives from flux-a2a-prototype should be considered the canonical definitions. The SignalCompiler in flux-runtime provides the compilation path to bytecode. The two should be unified: prototype defines the primitives, runtime compiles them.

⚡
