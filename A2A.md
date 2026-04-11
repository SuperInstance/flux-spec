# FLUX A2A Protocol v1.0 — Complete Specification

**The canonical Agent-to-Agent communication protocol for the FLUX Virtual Machine.**

This document defines the official A2A protocol: how autonomous agents within the FLUX fleet communicate, coordinate, delegate, and establish trust — all at the bytecode level. Every VM runtime, compiler, and agent framework that supports inter-agent communication must conform to this specification.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Bytecode Opcodes](#3-bytecode-opcodes)
4. [FIR Instructions](#4-fir-instructions)
5. [Message Format](#5-message-format)
6. [Trust Engine — INCREMENTS+2](#6-trust-engine--increments2)
7. [Capability System](#7-capability-system)
8. [Signal Language](#8-signal-language)
9. [Transport Layer](#9-transport-layer)
10. [Conformance Requirements](#10-conformance-requirements)

---

## 1. Overview

### What A2A Is

The A2A (Agent-to-Agent) protocol is a **first-class inter-agent communication system** embedded directly into the FLUX Virtual Machine's instruction set architecture. Unlike conventional multi-agent frameworks where agents communicate through high-level message brokers, chat APIs, or external orchestration layers, FLUX agents communicate through **bytecode**. A2A is not an add-on library or a protocol bolted on top of the VM — it is part of the ISA itself, occupying a dedicated opcode space at the lowest level of execution.

This design means that agent communication carries the same performance guarantees, security model, and determinism as any other VM operation. Messages are not strings passed through a socket; they are structured binary payloads routed through a trust-gated dispatch system, with every interaction tracked, scored, and metered by the VM's built-in trust engine.

### Design Principles

A2A is built on four pillars:

- **Dedicated opcode space** — The 16 A2A opcodes occupy bytecodes `0x50` through `0x5F`, using Format E (four bytes: opcode + three register operands). This gives A2A primitives the same encoding density and decode efficiency as core arithmetic and memory operations.

- **FIR (FLUX Intermediate Representation) instructions** — At the compiler level, A2A operations are expressed as typed FIR instructions (`Tell`, `Ask`, `Delegate`, `TrustCheck`, `CapRequire`) before lowering to bytecode. This enables type checking, capability validation, and optimization at the IR level.

- **Binary message format** — All inter-agent messages conform to a fixed 52-byte header plus variable-length payload. The format is fully specified with little-endian fields, enabling zero-copy deserialization and deterministic wire compatibility across all FLUX runtimes.

- **Trust engine and capability gating** — Every message is subject to a trust gate based on the INCREMENTS+2 composite trust model. Agents must hold sufficient trust scores and possess the required capability tokens before communication is permitted. This is enforced at the VM level, not at the application level.

### Philosophy

The core philosophy of A2A can be summarized in one principle: **agents communicate through bytecode, not chat. Git signals, not conversation.**

In a FLUX fleet, an agent does not "ask" another agent a question in natural language. Instead, it issues an `ASK` opcode that requests a specific typed result, receives a structured response in a register, and continues execution deterministically. Coordination happens through signals (named, typed events), delegation patterns (fork/join), and broadcast channels — all expressible in the Signal language and compilable to FLUX bytecode.

This approach eliminates the ambiguity, latency overhead, and security risks of natural-language agent coordination while enabling the same emergent multi-agent behaviors: task distribution, result aggregation, consensus, specialization, and hierarchical organization.

---

## 2. Architecture

### Three-Layer Design

A2A is structured in three distinct layers, each with clear boundaries and responsibilities:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Layer 3: Transport                           │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │ In-Process   │  │    HTTP      │  │    WebSocket / gRPC   │  │
│  │  Mailbox     │  │  (future)    │  │      (future)         │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬────────────┘  │
│         │                 │                      │               │
├─────────┴─────────────────┴──────────────────────┴───────────────┤
│                    Layer 2: Message Format                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  52-byte header + variable payload                       │   │
│  │  sender_uuid | receiver_uuid | conversation_id | type   │   │
│  │  priority | trust_token | capability_token | payload   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│                    Layer 1: Bytecode / FIR                       │
│  ┌────────────────┐    ┌────────────────────────────────────┐   │
│  │ FIR: Tell()    │───>│  Bytecode: 0x50 [rd][rs1][rs2]    │   │
│  │ FIR: Ask()     │───>│  Bytecode: 0x51 [rd][rs1][rs2]    │   │
│  │ FIR: Delegate()│───>│  Bytecode: 0x52 [rd][rs1][rs2]    │   │
│  │ FIR: TrustChk()│───>│  Bytecode: 0x5C [rd][rs1][rs2]    │   │
│  │ FIR: CapReq()  │───>│  Bytecode: (via CAP_REQUIRE)      │   │
│  └────────────────┘    └────────────────────────────────────┘   │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│                 Trust Engine (INCREMENTS+2)                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  6-dimensional composite trust score                    │    │
│  │  T = α·T_hist + β·T_cap + γ·T_lat + δ·T_con +          │    │
│  │      ε·T_det + ζ·T_aud                                  │    │
│  │  Gate: T >= threshold (default 0.3)                     │    │
│  └─────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

### AgentCoordinator

The `AgentCoordinator` is the central runtime component that manages all inter-agent communication. It provides four core services:

1. **Registration** — Agents register with the coordinator upon initialization, receiving a unique 128-bit UUID and a mailbox handle. Registration requires declaring the agent's capabilities (a set of typed capability tokens).

2. **Mailbox Delivery** — Each registered agent owns a mailbox (a FIFO queue of messages). The coordinator routes incoming messages to the appropriate mailbox based on the receiver UUID in the message header.

3. **Trust-Gated Dispatch** — Before delivering a message, the coordinator evaluates the sender's trust score against the receiver's trust threshold. If the score is below the threshold, the message is rejected with a `TRUST_TOO_LOW` error and the sender's trust is penalized.

4. **Capability Verification** — The coordinator checks that the sender holds a capability token matching the operation type requested. Capability tokens are computed from the agent's declared capabilities at registration time and verified on every dispatch.

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Agent A    │     │ Coordinator  │     │   Agent B    │
│              │     │              │     │              │
│ TELL opcode  │────>│ Trust Gate   │────>│ Mailbox      │
│ 0x50 rd,rs1, │     │ Cap Check    │     │ Receive      │
│     rs2      │     │ Route        │     │ Process      │
│              │     │              │     │              │
│ <────result──│     │              │     │ ASK response │
└──────────────┘     └──────────────┘     └──────────────┘
```

### Signal Compiler

The Signal language provides a high-level JSON-based DSL for expressing multi-agent programs. The signal compiler (`signal_compiler.py`) translates Signal programs into FLUX bytecode. This is the primary authoring interface for A2A programs:

```
Signal JSON  ──>  signal_compiler.py  ──>  FLUX Bytecode
                                   │
                                   ├── FIR (optional, for debugging)
                                   └── Opcode listing (assembly)
```

### Message Flow

The complete lifecycle of a `TELL` opcode from issuance to delivery:

```
  Agent A (sender)                    VM Runtime                    Agent B (receiver)
  ─────────────────                   ──────────                    ──────────────────
  1. Execute: TELL R1, R2, R3
     rd=R1 (tag), rs1=R2 (target),
     rs2=R3 (message handle)
        │
        v
  2. VM decodes opcode 0x50
     Reads register operands
     Looks up A2A handler callback
        │
        v
  3. Construct message:
     - sender_uuid = current agent
     - receiver_uuid = lookup(R2)
     - message_type = 0x50 (TELL)
     - trust_token = current trust
     - capability_token = agent caps
     - payload = memory[R3..]
        │
        v
  4. Submit to AgentCoordinator
     ┌─────────────────────────────┐
     │ Trust Gate: T(sender) >= θ? │──No──> Return error to R1
     │ Cap Check: cap valid?       │──No──> Return error to R1
     │ Route: find receiver mailbox│
     └──────────────┬──────────────┘
                    │ Yes
                    v
  5. Deliver to Agent B mailbox
     Message enqueued in B's FIFO
        │
        v
  6. Agent B dequeues message
     Processes payload
     (TELL is fire-and-forget;
      no response expected)
        │
        v
  7. R1 = 0 (success tag)
     Execution continues
```

---

## 3. Bytecode Opcodes

All A2A opcodes occupy the range `0x50`–`0x5F` and use **Format E** (four bytes: `[opcode][rd][rs1][rs2]`). All register operands are 4-bit indices (0–15) mapped to the general-purpose register file.

### Format E Encoding

```
Byte 0     Byte 1     Byte 2     Byte 3
┌─────────┬─────────┬─────────┬─────────┐
│ opcode  │   rd    │   rs1   │   rs2   │
│ 0x5X    │ 4 bits  │ 4 bits  │ 4 bits  │
└─────────┴─────────┴─────────┴─────────┘
```

### Opcode Summary Table

| Hex | Mnemonic | Blocking | Trust Req | Cap Req | Description |
|-----|----------|----------|-----------|---------|-------------|
| 0x50 | **TELL** | No | T ≥ 0.3 | send | Send message to agent |
| 0x51 | **ASK** | Yes | T ≥ 0.3 | request | Request data from agent |
| 0x52 | **DELEG** | No | T ≥ 0.5 | delegate | Delegate task to agent |
| 0x53 | **BCAST** | No | T ≥ 0.3 | broadcast | Broadcast to fleet |
| 0x54 | **ACCEPT** | No | T ≥ 0.2 | none | Accept delegated task |
| 0x55 | **DECLINE** | No | T ≥ 0.2 | none | Decline with reason |
| 0x56 | **REPORT** | No | T ≥ 0.3 | report | Report task status |
| 0x57 | **MERGE** | No | T ≥ 0.3 | merge | Merge agent results |
| 0x58 | **FORK** | No | T ≥ 0.4 | fork | Spawn child agent |
| 0x59 | **JOIN** | Yes | T ≥ 0.4 | none | Wait for child agent |
| 0x5A | **SIGNAL** | No | T ≥ 0.1 | signal | Emit named signal |
| 0x5B | **AWAIT** | Yes | T ≥ 0.1 | none | Wait for signal |
| 0x5C | **TRUST** | No | admin | admin | Set trust level |
| 0x5D | **DISCOV** | Yes | T ≥ 0.2 | discover | Discover fleet agents |
| 0x5E | **STATUS** | Yes | T ≥ 0.2 | status | Query agent status |
| 0x5F | **HEARTBT** | No | T ≥ 0.1 | none | Emit heartbeat |

### 3.1 TELL — Send Message (0x50)

```
TELL rd, rs1, rs2    →    [0x50][rd][rs1][rs2]
```

**Semantics:** Send a message to another agent. The message payload is addressed by `rs2` (a memory address or handle containing the message bytes). The target agent is identified by `rs1` (an agent ID or registered name). A delivery tag is written to `rd` for tracking purposes.

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Output | Delivery tag (0 = success, nonzero = error code) |
| rs1 | Input | Target agent identifier |
| rs2 | Input | Message payload address/handle |

**Blocking:** No. Returns immediately after enqueueing. Delivery is asynchronous.

**Trust Requirement:** Sender trust T ≥ 0.3 toward receiver.

**Capability Required:** `send`

**Error Codes (rd):**
- `0x00` — Success, message enqueued
- `0x01` — Agent not found (rs1 invalid)
- `0x02` — Trust too low (sender below receiver threshold)
- `0x03` — Capability denied (sender lacks `send` capability)
- `0x04` — Mailbox full (receiver's queue at capacity)
- `0x05` — Invalid payload (rs2 address out of bounds)

**Example:**
```asm
MOVI  R2, 42        ; target agent ID
MOVI16 R3, msg_buf  ; payload address
TELL  R1, R2, R3    ; send message, tag in R1
JNZ   R1, error     ; check for delivery error
```

### 3.2 ASK — Request Data (0x51)

```
ASK rd, rs1, rs2    →    [0x51][rd][rs1][rs2]
```

**Semantics:** Send a request to another agent and block until a response is received. The request payload is at `rs2`. The response is written into the memory region addressed by `rd`. This is the primary request-response primitive.

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Output | Response buffer address (response written here) |
| rs1 | Input | Target agent identifier |
| rs2 | Input | Request payload address/handle |

**Blocking:** Yes. The sending agent is suspended until the target agent responds or the request times out (configurable, default 30 seconds). During suspension, the agent's cycle budget is paused.

**Trust Requirement:** T ≥ 0.3.

**Capability Required:** `request`

**Error Codes (rd, first byte):**
- `0x00` — Success, response payload follows
- `0x01` — Agent not found
- `0x02` — Trust too low
- `0x03` — Capability denied
- `0x04` — Timeout (receiver did not respond)
- `0x05` — Receiver declined the request

**Example:**
```asm
MOVI  R2, 17        ; target agent ID
MOVI16 R3, req_buf  ; request payload
MOVI16 R1, resp_buf ; response buffer
ASK   R1, R2, R3    ; request, blocks until response
; R1 now contains response data
```

### 3.3 DELEG — Delegate Task (0x52)

```
DELEG rd, rs1, rs2  →    [0x52][rd][rs1][rs2]
```

**Semantics:** Delegate a task to another agent, transferring execution authority. The task specification is at `rs2`. A task ID is returned in `rd` for tracking. The receiving agent must `ACCEPT` or `DECLINE` the task.

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Output | Task ID (used for tracking and JOIN) |
| rs1 | Input | Target agent identifier |
| rs2 | Input | Task specification address/handle |

**Blocking:** No. Returns task ID immediately. Use `JOIN` to wait for completion.

**Trust Requirement:** T ≥ 0.5 (higher than TELL due to authority transfer).

**Capability Required:** `delegate`

**Delegation Lifecycle:**
```
Delegator              Target
    │                     │
    ├─ DELEG ─────────────>│  (task offered)
    │                     ├── ACCEPT ───────────>│  (task accepted)
    │                     │   or                  │
    │                     ├── DECLINE ──────────>│  (task rejected)
    │                     │                     │
    │   (task executes)   │                     │
    │                     │                     │
    ├─ JOIN ──────────────┤  (wait for result)  │
    │<──── result ────────┤                     │
    │                     │                     │
```

### 3.4 BCAST — Broadcast (0x53)

```
BCAST rd, rs1, rs2  →    [0x53][rd][rs1][rs2]
```

**Semantics:** Broadcast a message to all agents in the fleet (or to a filtered subset). The broadcast payload is at `rs2`. `rs1` contains a filter bitmask: `0x0000` = all agents, other values filter by agent group. A broadcast tag is written to `rd`.

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Output | Broadcast tag (0 = all delivered, nonzero = partial) |
| rs1 | Input | Group filter bitmask (0 = all agents) |
| rs2 | Input | Broadcast payload address/handle |

**Blocking:** No. Returns immediately.

**Trust Requirement:** T ≥ 0.3.

**Capability Required:** `broadcast`

**Note:** Broadcast bypasses individual agent mailboxes and writes directly to all matching agents' broadcast queues. Trust is checked per-receiver; agents with insufficient trust do not receive the message but do not cause an error for the sender.

### 3.5 ACCEPT — Accept Task (0x54)

```
ACCEPT rd, rs1, rs2  →    [0x54][rd][rs1][rs2]
```

**Semantics:** Accept a delegated task. `rs1` contains the task ID (from a previous `DELEG` directed at this agent). The task context is loaded into the memory region at `rd`. `rs2` is reserved for future use (accept parameters).

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Output | Task context address (loaded from delegation) |
| rs1 | Input | Task ID to accept |
| rs2 | Input | Reserved (accept parameters, currently ignored) |

**Blocking:** No. Loads task context into memory and returns immediately.

**Trust Requirement:** T ≥ 0.2 (low; acceptance is cooperative).

**Capability Required:** None.

### 3.6 DECLINE — Decline Task (0x55)

```
DECLINE rd, rs1, rs2  →    [0x55][rd][rs1][rs2]
```

**Semantics:** Decline a delegated task with a reason code. `rs1` contains the task ID. `rs2` contains a reason code (0 = overloaded, 1 = incapable, 2 = policy violation, 3+ = custom). The delegator is notified.

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Output | Status (0 = declined successfully) |
| rs1 | Input | Task ID to decline |
| rs2 | Input | Reason code |

**Blocking:** No.

**Trust Requirement:** T ≥ 0.2.

**Capability Required:** None.

**Reason Codes:**
| Code | Meaning |
|------|---------|
| 0 | Overloaded (insufficient resources) |
| 1 | Incapable (missing required capabilities) |
| 2 | Policy violation (task violates agent policy) |
| 3 | Already busy (another task in progress) |
| 4+ | Implementation-defined |

### 3.7 REPORT — Report Status (0x56)

```
REPORT rd, rs1, rs2  →    [0x56][rd][rs1][rs2]
```

**Semantics:** Report task status to a supervisor or tracking system. `rd` identifies the supervisor/tracker. `rs1` contains the task ID. `rs2` contains a status word.

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Input | Supervisor/tracker identifier |
| rs1 | Input | Task ID being reported |
| rs2 | Input | Status word (progress, result, error) |

**Blocking:** No.

**Trust Requirement:** T ≥ 0.3.

**Capability Required:** `report`

**Status Word Layout (rs2):**
```
Bits 31-24: status code (0=pending, 1=running, 2=done, 3=error)
Bits 23-0:  progress (0-16777215, scaled 0-100%)
```

### 3.8 MERGE — Merge Results (0x57)

```
MERGE rd, rs1, rs2  →    [0x57][rd][rs1][rs2]
```

**Semantics:** Merge results from two agent sources. `rs1` and `rs2` identify two result sets (memory addresses or handles). The merged result is written to `rd`. The merge strategy depends on the data type: numeric results are averaged, byte arrays are concatenated, structured data follows the agent's declared merge function.

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Output | Merged result address |
| rs1 | Input | First result set address |
| rs2 | Input | Second result set address |

**Blocking:** No.

**Trust Requirement:** T ≥ 0.3.

**Capability Required:** `merge`

### 3.9 FORK — Spawn Child Agent (0x58)

```
FORK rd, rs1, rs2  →    [0x58][rd][rs1][rs2]
```

**Semantics:** Spawn a new child agent. `rs2` contains the address of the child agent's initial state (bytecode, registers, capabilities). The child agent receives a new UUID and is registered with the coordinator. The child ID is written to `rd`.

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Output | Child agent ID |
| rs1 | Input | Initial trust level for child (0-1023, scaled) |
| rs2 | Input | Initial state address (bytecode/registers) |

**Blocking:** No. The child begins executing asynchronously.

**Trust Requirement:** T ≥ 0.4 (parent must be trusted to spawn agents).

**Capability Required:** `fork`

### 3.10 JOIN — Wait for Child (0x59)

```
JOIN rd, rs1, rs2  →    [0x59][rd][rs1][rs2]
```

**Semantics:** Wait for a child agent (identified by `rs1`) to complete execution. The child's final result is written to `rd`. `rs2` is reserved for timeout parameters. If the child has already completed, the result is returned immediately.

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Output | Child agent's final result |
| rs1 | Input | Child agent ID |
| rs2 | Input | Reserved (timeout, 0 = infinite) |

**Blocking:** Yes. Suspends the parent until the child completes or times out.

**Trust Requirement:** T ≥ 0.4.

**Capability Required:** None (forking agent inherently has join rights on its children).

### 3.11 SIGNAL — Emit Signal (0x5A)

```
SIGNAL rd, rs1, rs2  →    [0x5A][rd][rs1][rs2]
```

**Semantics:** Emit a named signal on a channel. Signals are the primary coordination primitive for event-driven agent programs. `rd` is the channel ID, `rs1` is the signal name (memory address of a null-terminated string or signal ID), and `rs2` contains optional signal data.

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Input | Channel ID |
| rs1 | Input | Signal name/address |
| rs2 | Input | Signal data (optional) |

**Blocking:** No. Signals are non-blocking events.

**Trust Requirement:** T ≥ 0.1 (minimal; signals are informational).

**Capability Required:** `signal`

### 3.12 AWAIT — Wait for Signal (0x5B)

```
AWAIT rd, rs1, rs2  →    [0x5B][rd][rs1][rs2]
```

**Semantics:** Suspend execution until a named signal is received on a channel. `rd` is the channel ID, `rs1` is the signal name to wait for, and `rs2` is a timeout value (0 = wait indefinitely). Signal data is written to the memory region following `rd`.

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Input/Output | Channel ID / signal data written here |
| rs1 | Input | Signal name/address to await |
| rs2 | Input | Timeout (cycles, 0 = infinite) |

**Blocking:** Yes. Suspends until signal or timeout.

**Trust Requirement:** T ≥ 0.1.

**Capability Required:** None.

### 3.13 TRUST — Set Trust Level (0x5C)

```
TRUST rd, rs1, rs2  →    [0x5C][rd][rs1][rs2]
```

**Semantics:** Manually adjust the trust level for an agent. `rs1` is the target agent ID. `rs2` is the new trust value (0–1023, mapped to 0.0–1.0). This is a privileged operation that requires admin capability. The previous trust value is returned in `rd`.

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Output | Previous trust value (for rollback) |
| rs1 | Input | Target agent ID |
| rs2 | Input | New trust value (0-1023) |

**Blocking:** No.

**Trust Requirement:** Admin privilege (T ≥ 0.8 or system role).

**Capability Required:** `admin`

### 3.14 DISCOV — Discover Agents (0x5D)

```
DISCOV rd, rs1, rs2  →    [0x5D][rd][rs1][rs2]
```

**Semantics:** Query the fleet for available agents matching criteria. `rs1` contains a capability filter (0 = all agents). `rs2` is reserved. A list of matching agent IDs is written to memory starting at `rd`, with the count in the first 4 bytes.

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Output | Address of agent list (count + IDs) |
| rs1 | Input | Capability filter bitmask |
| rs2 | Input | Reserved |

**Blocking:** Yes. Returns snapshot of fleet state.

**Trust Requirement:** T ≥ 0.2.

**Capability Required:** `discover`

### 3.15 STATUS — Query Status (0x5E)

```
STATUS rd, rs1, rs2  →    [0x5E][rd][rs1][rs2]
```

**Semantics:** Query another agent's current status (running, idle, error, trust score, capabilities). `rs1` is the target agent ID. Status data is written to memory at `rd`.

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Output | Status data address |
| rs1 | Input | Target agent ID |
| rs2 | Input | Status flags (0 = all, 1 = brief) |

**Blocking:** Yes.

**Trust Requirement:** T ≥ 0.2.

**Capability Required:** `status`

**Status Data Format (at rd):**
```
Offset  Size   Field
0       4      agent_state (0=idle, 1=running, 2=error, 3=paused)
4       4      cycle_count
8       4      mailbox_depth
12      4      trust_score (scaled 0-1023)
16      4      capability_count
20      4      active_tasks
```

### 3.16 HEARTBT — Heartbeat (0x5F)

```
HEARTBT rd, rs1, rs2  →    [0x5F][rd][rs1][rs2]
```

**Semantics:** Emit a heartbeat signal indicating the agent is alive and operational. `rd` is the current load metric (0–255, representing CPU utilization or task queue depth). `rs1` is the heartbeat interval hint. `rs2` is reserved. The coordinator records the heartbeat timestamp and updates the agent's liveness status.

| Operand | Role | Semantics |
|---------|------|-----------|
| rd | Input | Load metric (0-255) |
| rs1 | Input | Interval hint (cycles between heartbeats) |
| rs2 | Input | Reserved |

**Blocking:** No.

**Trust Requirement:** T ≥ 0.1 (minimal; heartbeat is operational).

**Capability Required:** None.

---

## 4. FIR Instructions

The FLUX Intermediate Representation (FIR) provides typed, SSA-based instructions that sit between high-level Signal programs and low-level bytecode. FIR A2A instructions carry full type information and are validated before bytecode emission.

### Instruction Set

#### `Tell(target_agent: AgentId, message: Payload, cap: CapToken) -> Tag`

Send a one-way message. Returns a delivery tag for tracking.

```
%1 = Tell(agent:"compute-worker", payload:%msg_data, cap:cap_send)
```

**Type Rules:**
- `target_agent` must be of type `AgentId` (resolved to a 128-bit UUID at link time)
- `message` must be of type `Payload` (any serializable FIR value)
- `cap` must be of type `CapToken` with permission `send`
- Return type is `Tag` (uint32)

#### `Ask(target_agent: AgentId, message: Payload, return_type: Type, cap: CapToken) -> Response`

Send a request and await a typed response.

```
%2 = Ask(agent:"data-store", payload:%query, return_type:i32, cap:cap_request)
```

**Type Rules:**
- `target_agent`: `AgentId`
- `message`: `Payload`
- `return_type`: any valid FIR type (i32, f32, string, struct, etc.)
- `cap`: `CapToken` with permission `request`
- Return type matches `return_type`

#### `Delegate(target_agent: AgentId, authority: AuthLevel, cap: CapToken) -> TaskId`

Delegate a task with a specified authority level.

```
%3 = Delegate(agent:"worker-pool", authority:full, cap:cap_delegate)
```

**Type Rules:**
- `target_agent`: `AgentId`
- `authority`: `AuthLevel` (enum: `read_only`, `limited`, `full`)
- `cap`: `CapToken` with permission `delegate`
- Return type is `TaskId` (uint32)

#### `TrustCheck(agent: AgentId, threshold: f64, cap: CapToken) -> bool`

Assert that the trust score for an agent meets a threshold.

```
%4 = TrustCheck(agent:"external-service", threshold:0.7, cap:cap_admin)
```

**Type Rules:**
- `agent`: `AgentId`
- `threshold`: `f64` in range [0.0, 1.0]
- `cap`: `CapToken` with permission `admin`
- Return type is `bool`

#### `CapRequire(capability: CapName, resource: ResourceId, cap: CapToken) -> unit`

Assert that the current agent holds a specific capability for a resource.

```
CapRequire(capability:"write", resource:"region://heap-42", cap:cap_self)
```

**Type Rules:**
- `capability`: `CapName` (string literal, one of: read, write, execute, delegate, admin)
- `resource`: `ResourceId` (URI-formatted resource identifier)
- `cap`: `CapToken` (self-referencing capability token)
- Return type is `unit` (panics on failure)

### BNF Syntax for FIR A2A Instructions

```ebnf
a2a_instruction
  = tell_inst | ask_inst | delegate_inst | trust_check_inst | cap_require_inst

tell_inst
  = "Tell" "(" "agent" ":" string_literal ","
             "payload" ":" ssa_value ","
             "cap" ":" ssa_value ")" "->" type_annotation

ask_inst
  = "Ask" "(" "agent" ":" string_literal ","
            "payload" ":" ssa_value ","
            "return_type" ":" type ","
            "cap" ":" ssa_value ")" "->" type_annotation

delegate_inst
  = "Delegate" "(" "agent" ":" string_literal ","
                "authority" ":" auth_level ","
                "cap" ":" ssa_value ")" "->" type_annotation

trust_check_inst
  = "TrustCheck" "(" "agent" ":" string_literal ","
                   "threshold" ":" float_literal ","
                   "cap" ":" ssa_value ")" "->" "bool"

cap_require_inst
  = "CapRequire" "(" "capability" ":" cap_name ","
                   "resource" ":" string_literal ","
                   "cap" ":" ssa_value ")"

auth_level
  = "read_only" | "limited" | "full"

cap_name
  = "read" | "write" | "execute" | "delegate" | "admin"

ssa_value
  = "%" identifier | literal_value
```

### FIR-to-Bytecode Lowering

| FIR Instruction | Bytecode | Notes |
|----------------|----------|-------|
| `Tell(agent, msg, cap)` | `0x50 [rd] [rs1] [rs2]` | rs1 = agent, rs2 = msg addr, rd = tag |
| `Ask(agent, msg, type, cap)` | `0x51 [rd] [rs1] [rs2]` | rd = resp buf, rs1 = agent, rs2 = req addr |
| `Delegate(agent, auth, cap)` | `0x52 [rd] [rs1] [rs2]` | rd = task id, rs1 = agent, rs2 = task spec |
| `TrustCheck(agent, thresh, cap)` | `0x5C [rd] [rs1] [rs2]` | rd = prev trust, rs1 = agent, rs2 = new val |
| `CapRequire(cap, res, tok)` | Inline check + trap | Emits CAP_REQUIRE syscall (0x10) |

---

## 5. Message Format

All inter-agent messages conform to a fixed binary format. The header is exactly 52 bytes, followed by a variable-length payload. All multi-byte fields use **little-endian** byte order, consistent with the FLUX ISA convention.

### Header Layout

```
Offset  Size   Field                 Description
──────  ────   ─────                 ───────────
0       16     sender_uuid           128-bit sender agent UUID
16      16     receiver_uuid         128-bit receiver agent UUID
32       8     conversation_id       64-bit conversation identifier
40       1     message_type          uint8: maps to opcode (0x50-0x5F)
41       1     priority              uint8: 0 (lowest) to 15 (highest)
42       4     trust_token           uint32 LE: sender's trust token
46       4     capability_token      uint32 LE: sender's capability token
50       2     in_reply_to           uint16 LE: message ID being replied to
52      var    payload               arbitrary bytes (0 to 2^32-1 bytes)
```

### Visual Layout

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+ 0
|                      sender_uuid (bytes 0-15)                   |
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+ 16
|                    receiver_uuid (bytes 0-15)                   |
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+ 32
|                      conversation_id (64-bit LE)                |
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+ 40
| msg_type | priority |      trust_token (32-bit LE)      |cap | 46
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+ 48
|  capability_token (cont) |  in_reply_to (16-bit LE)      |     50
+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+ 52
|                                                               |
|                      payload (variable length)                 |
|                                                               |
+---------------------------------------------------------------+
```

### Field Definitions

#### sender_uuid (16 bytes, offset 0)
128-bit UUID of the sending agent, encoded as raw bytes in network byte order (big-endian for the UUID itself, within the little-endian message frame). Generated at agent registration time by the AgentCoordinator.

#### receiver_uuid (16 bytes, offset 16)
128-bit UUID of the intended recipient. For `BCAST`, this field is set to all-zeros (`0x00000000-0000-0000-0000-000000000000`) to indicate fleet-wide delivery.

#### conversation_id (8 bytes, offset 32)
64-bit identifier linking related messages into a conversation. Monotonically increasing per sender-receiver pair. Enables correlation of requests and responses.

#### message_type (1 byte, offset 40)
Maps directly to the A2A opcode that generated the message:

| Value | Opcode | Direction |
|-------|--------|-----------|
| 0x50 | TELL | One-way send |
| 0x51 | ASK | Request |
| 0x52 | DELEG | Task delegation |
| 0x53 | BCAST | Broadcast |
| 0x54 | ACCEPT | Task acceptance |
| 0x55 | DECLINE | Task rejection |
| 0x56 | REPORT | Status report |
| 0x57 | MERGE | Result merge |
| 0x58 | FORK | Agent spawn notification |
| 0x59 | JOIN | Completion notification |
| 0x5A | SIGNAL | Named signal |
| 0x5B | AWAIT | Signal acknowledgment |
| 0x5C | TRUST | Trust adjustment |
| 0x5D | DISCOV | Discovery response |
| 0x5E | STATUS | Status response |
| 0x5F | HEARTBT | Heartbeat |

#### priority (1 byte, offset 41)
Message priority level, 0 (background) to 15 (critical). Used by the transport layer for queue ordering. Higher-priority messages are delivered first when mailboxes are contended.

Default priorities:
- `TELL`, `BCAST`, `SIGNAL`, `HEARTBT`: priority 7 (normal)
- `ASK`, `AWAIT`: priority 8 (normal-high)
- `DELEG`, `REPORT`: priority 10 (high)
- `ACCEPT`, `DECLINE`: priority 12 (urgent)
- `TRUST`: priority 14 (system-critical)

#### trust_token (4 bytes, offset 42)
Unsigned 32-bit integer encoding the sender's composite trust score at time of sending, scaled to 0–4294967295 (mapping 0.0–1.0). The receiver verifies this against its locally cached trust score for the sender. Significant discrepancies trigger a trust re-evaluation.

#### capability_token (4 bytes, offset 46)
Unsigned 32-bit integer computed from the sender's declared capabilities and the operation type. The token is a hash: `HMAC-SHA256(sender_uuid || declared_caps || opcode, fleet_key)[0:4]`. The receiver recomputes and verifies.

#### in_reply_to (2 bytes, offset 50)
Unsigned 16-bit message ID that this message is replying to. Zero indicates a new message (not a reply). Used for correlating `ASK` responses, `DELEG` accept/decline, and `STATUS` query results.

#### payload (variable, offset 52)
Arbitrary bytes. The interpretation depends on the message type and the agents' shared protocol. The FLUX VM does not interpret the payload; it is opaque data routed by the transport layer.

Maximum payload size: 4,294,967,295 bytes (2^32 - 1), limited by transport layer configuration. Default mailbox capacity: 64 messages or 1 MB total payload, whichever is reached first.

---

## 6. Trust Engine — INCREMENTS+2

### Overview

The INCREMENTS+2 trust engine is a six-dimensional composite trust model that evaluates agent reliability across multiple behavioral axes. Every inter-agent message is subject to a trust gate: messages from agents below the receiver's trust threshold are silently dropped (for broadcasts) or rejected with error code `0x02` (for directed messages).

The name "INCREMENTS+2" reflects the six dimensions: the four original INCREMENTS axes (history, capability, latency, consistency) plus two additions (determinism, audit).

### Composite Trust Formula

```
T = α · T_history  +  β · T_capability  +  γ · T_latency
  + δ · T_consistency  +  ε · T_determinism  +  ζ · T_audit
```

| Symbol | Weight | Dimension | Range |
|--------|--------|-----------|-------|
| α | 0.30 | T_history | [0.0, 1.0] |
| β | 0.25 | T_capability | [0.0, 1.0] |
| γ | 0.20 | T_latency | [0.0, 1.0] |
| δ | 0.15 | T_consistency | [0.0, 1.0] |
| ε | 0.05 | T_determinism | [0.0, 1.0] |
| ζ | 0.05 | T_audit | {0.0, 1.0} |

Weight sum: 0.30 + 0.25 + 0.20 + 0.15 + 0.05 + 0.05 = **1.00**

### Time Decay

Trust scores decay over time to reflect the recency requirement. Stale trust (from old interactions) is progressively downweighted:

```
T_composite *= (1 − λ · elapsed / max_age)
```

| Parameter | Value | Meaning |
|-----------|-------|---------|
| λ | 0.01 / sec | Decay rate |
| max_age | 3600 sec | Maximum interaction age considered |
| elapsed | seconds since last interaction | Recency factor |

At `elapsed = 0` (just interacted): multiplier = 1.0 (no decay)
At `elapsed = 1800` (30 minutes): multiplier = 0.95
At `elapsed = 3600` (1 hour): multiplier = 0.90
At `elapsed > 3600`: capped at `max_age`, multiplier = 0.90 (minimum)

### Dimension 1: T_history (weight α = 0.30)

Tracks the historical success rate of interactions using an exponential moving average (EMA) of binary outcomes.

```
T_history = EMA(success_rate, α_ema = 0.1)
```

Where `success_rate` is updated after each interaction:
- Success (message delivered, response received, task completed): outcome = 1.0
- Failure (timeout, error, declined): outcome = 0.0
- Update rule: `T_history = α_ema · outcome + (1 − α_ema) · T_history`

**Behavior:** An agent that consistently succeeds will converge to T_history ≈ 1.0. A single failure causes a 10% drop. An agent that fails repeatedly decays toward 0.0. The α_ema = 0.1 smoothing factor means the most recent ~10 interactions dominate the score.

**Cold Start:** New agents begin with T_history = 0.5 (neutral prior).

### Dimension 2: T_capability (weight β = 0.25)

Measures how well the agent's declared capabilities match the actual operations it performs successfully.

```
T_capability = avg(capability_match[i] for i in last 50 interactions)
```

Where `capability_match` for a single interaction:
- 1.0: The agent performed exactly the operation its declared capabilities promised
- 0.8: Close match (e.g., read when write was expected but read-only was acceptable)
- 0.5: Partial match (performed a degraded version of the requested operation)
- 0.0: Mismatch (claimed a capability but failed to deliver)

**Behavior:** Agents that overclaim capabilities are penalized. Agents that consistently deliver within their declared scope maintain high scores. The sliding window of 50 interactions ensures responsiveness to capability drift.

### Dimension 3: T_latency (weight γ = 0.20)

Evaluates response timeliness using inverse linear interpolation between a target latency and a maximum acceptable latency.

```
T_latency = clamp((max_lat − actual_lat) / (max_lat − target_lat), 0.0, 1.0)
```

| Parameter | Value |
|-----------|-------|
| target_lat | 10 ms (ideal response time) |
| max_lat | 1000 ms (1 second, worst acceptable) |

**Examples:**
- actual = 10 ms → T_latency = 1.0 (perfect)
- actual = 100 ms → T_latency = (1000 − 100) / (1000 − 10) = 900/990 ≈ 0.909
- actual = 500 ms → T_latency = 500/990 ≈ 0.505
- actual = 1000 ms → T_latency = 0.0 (at limit)
- actual = 2000 ms → T_latency = 0.0 (clamped)

**Behavior:** Fast agents are rewarded. Agents near the latency cap are heavily penalized. The asymmetric weighting (target at 10ms, max at 1000ms) reflects the expectation that most A2A operations should complete in milliseconds.

### Dimension 4: T_consistency (weight δ = 0.15)

Measures the stability of response times using the inverse of the coefficient of variation.

```
T_consistency = 1 − CV(latencies)
              = 1 − (σ / μ)
```

Where σ is the standard deviation and μ is the mean of the last 20 response latencies.

**Examples:**
- All responses at 50ms: σ = 0, CV = 0, T_consistency = 1.0
- Responses at 50ms ± 5ms: σ ≈ 5, μ = 50, CV = 0.1, T_consistency = 0.9
- Responses at 50ms ± 50ms: σ ≈ 50, μ = 50, CV = 1.0, T_consistency = 0.0

**Behavior:** Consistent agents (low jitter) are preferred even if their absolute latency is moderate. Highly variable agents are distrusted because unpredictability complicates scheduling and timeout management.

**Edge case:** If μ = 0 (no interactions yet), T_consistency = 1.0 (benefit of the doubt).

### Dimension 5: T_determinism (weight ε = 0.05)

Tracks behavioral determinism: whether an agent produces the same output for the same input across repeated interactions.

```
T_determinism = match_count / sample_count   (over last 10 repeated inputs)
```

Where a "repeated input" is detected by comparing the payload hash of incoming messages. If the same payload hash is seen more than once, the outputs are compared.

**Examples:**
- Agent always returns the same result for the same input: T_determinism = 1.0
- Agent returns different results 30% of the time: T_determinism = 0.7
- Agent is stochastic by nature (e.g., randomized algorithms): T_determinism may be low, which is expected for certain agent types

**Behavior:** This dimension has a low weight (0.05) because some legitimate agents are non-deterministic. It primarily catches agents that are malfunctioning or have been tampered with.

### Dimension 6: T_audit (weight ζ = 0.05)

Binary dimension indicating whether the agent has valid audit records.

```
T_audit = 1.0  if audit_records_exist AND audit_records_valid
         0.0  otherwise
```

**Behavior:** Agents with no audit trail or invalid/corrupted records receive T_audit = 0.0. This acts as a gate for unknown or unverified agents. The low weight (0.05) means audit alone doesn't determine trust, but the lack of audit prevents achieving the highest trust levels.

### Trust Gate

Messages are filtered at dispatch time:

```
deliver(message) :=
  if T_composite(sender, receiver) >= receiver.trust_threshold:
    enqueue(message, receiver.mailbox)
  else:
    reject(message, reason=TRUST_TOO_LOW)
    penalize(sender, delta=-0.05)
```

Default trust threshold: **0.3** (configurable per agent, range [0.0, 1.0]).

Repeated trust failures escalate penalties:
- 1st rejection: delta = -0.05
- 2nd rejection (within 60s): delta = -0.10
- 3rd+ rejection (within 60s): delta = -0.20, cooldown = 300s

### Example Computation

Given an agent with the following interaction profile:

| Dimension | Computation | Score |
|-----------|-------------|-------|
| T_history | 9 successes, 1 failure in last 10 → EMA ≈ 0.90 | 0.90 |
| T_capability | 45/50 capability matches in window | 0.90 |
| T_latency | Mean latency 80ms → (1000−80)/(1000−10) ≈ 0.929 | 0.93 |
| T_consistency | Latencies: 70, 80, 75, 85, 90 → σ=7.9, μ=80, CV=0.099 | 0.90 |
| T_determinism | 9/10 repeated inputs matched | 0.90 |
| T_audit | Valid audit records present | 1.00 |

Composite (no decay, elapsed = 0):
```
T = 0.30 × 0.90 + 0.25 × 0.90 + 0.20 × 0.93
  + 0.15 × 0.90 + 0.05 × 0.90 + 0.05 × 1.00

T = 0.270 + 0.225 + 0.186 + 0.135 + 0.045 + 0.050

T = 0.911
```

This agent easily passes the default trust threshold of 0.3 and would be considered highly trusted.

---

## 7. Capability System

### Overview

The capability system provides fine-grained access control for agent operations. Capabilities are not just permissions — they are cryptographic tokens that prove an agent's authorization to perform specific operations on specific resources. The system integrates with FLUX's capability-based memory model (named regions with ownership and borrowers).

### Capability Tokens

A capability token is a 32-bit value computed from:
```
capability_token = HMAC-SHA256(
    agent_uuid || declared_capabilities || operation_opcode,
    fleet_key
)[0:4]
```

The `fleet_key` is a shared secret established at fleet initialization. All agents in the fleet can verify tokens but only the coordinator can issue them.

### Capability Types

| Type | Value | Description |
|------|-------|-------------|
| `read` | 0x01 | Read access to a resource |
| `write` | 0x02 | Write/modify access to a resource |
| `execute` | 0x04 | Execute bytecode or trigger actions |
| `delegate` | 0x08 | Delegate authority to other agents |
| `admin` | 0x10 | Administrative operations (trust, registration) |

Capabilities are bitmask-combinable. For example, `read | write` = `0x03` grants both read and write access.

### Operations

#### CAP_REQUIRE — Assert Capability

Before performing a protected operation, an agent must assert it holds the required capability:

```
CAP_REQUIRE capability, resource
```

If the assertion fails:
- In debug mode: traps to the debugger with `FAULT` (0xE7), error code `CAPABILITY_DENIED`
- In production mode: sets the Zero flag and skips the next instruction (similar to `C_THRESH` behavior)

**Bytecode encoding:** Uses `SYS 0x10` with capability and resource encoded in following registers.

#### CAP_REQUEST — Request Capability

An agent may request a new capability from the coordinator or from another agent:

```
CAP_REQUEST capability, resource, from_agent
```

This initiates a handshake:
1. Requester sends capability request
2. Granter evaluates the request
3. Granter responds with `CAP_GRANT` or `CAP_REVOKE`

#### CAP_GRANT — Grant Capability

```
CAP_GRANT capability, resource, to_agent
```

Grants a capability to another agent. The granter must hold `delegate` or `admin` capability for the resource. The granted capability is recorded in the coordinator's capability table and a new capability token is issued to the recipient.

#### CAP_REVOKE — Revoke Capability

```
CAP_REVOKE capability, resource, from_agent
```

Revokes a previously granted capability. The granter must hold `admin` capability for the resource. Revocation is immediate: the target agent's capability token is invalidated, and any in-flight operations using the old token will fail at the next trust gate check.

### Integration with Memory Regions

Capabilities extend to FLUX's named memory regions:

```
Agent "compute-1" declares:
  capabilities: {
    "region://shared-buffer": [read, write],
    "region://model-weights": [read],
    "a2a:send": [execute],
    "a2a:request": [execute]
  }
```

When Agent "compute-1" attempts to write to "region://model-weights", the capability system checks its declared capabilities, finds only `read`, and denies the operation. This is enforced at the VM level, not at the agent level.

### Capability Delegation Chains

Capabilities can be delegated through chains:

```
Admin → Manager (delegate, read, write) → Worker (read, write) → Subworker (read)
```

Each link in the chain reduces the delegation depth by one. Maximum delegation depth: **4 levels** (configurable). Delegation chains are tracked by the coordinator and can be audited.

---

## 8. Signal Language

### Overview

The Signal language is a high-level JSON-based DSL for authoring multi-agent FLUX programs. Signals provide a structured way to express coordination, computation, and communication patterns that compile down to FLUX bytecode through the `signal_compiler.py` tool.

### Signal JSON Format

A Signal program is a JSON object with the following top-level structure:

```json
{
  "name": "pipeline-name",
  "version": "1.0",
  "agents": ["agent-a", "agent-b", "agent-c"],
  "program": {
    "op": "seq",
    "steps": [...]
  }
}
```

### Operations (28 total)

#### Arithmetic (10 ops)

| Op | Description | Arguments |
|----|-------------|-----------|
| `let` | Bind a variable | `name`, `value` |
| `add` | Integer addition | `a`, `b` |
| `sub` | Integer subtraction | `a`, `b` |
| `mul` | Integer multiplication | `a`, `b` |
| `div` | Integer division | `a`, `b` |
| `mod` | Integer modulo | `a`, `b` |
| `eq` | Equality comparison | `a`, `b` |
| `neq` | Not-equal comparison | `a`, `b` |
| `lt` | Less-than comparison | `a`, `b` |
| `lte` | Less-than-or-equal | `a`, `b` |
| `gt` | Greater-than comparison | `a`, `b` |
| `gte` | Greater-than-or-equal | `a`, `b` |
| `and` | Bitwise AND | `a`, `b` |
| `or` | Bitwise OR | `a`, `b` |
| `not` | Bitwise NOT | `a` |
| `xor` | Bitwise XOR | `a`, `b` |

#### A2A Communication (4 ops)

| Op | Description | Arguments |
|----|-------------|-----------|
| `tell` | Send message to agent | `target`, `message`, `tag` |
| `ask` | Request data from agent | `target`, `message`, `into` |
| `delegate` | Delegate task to agent | `target`, `task`, `into` |
| `broadcast` | Broadcast to fleet | `message`, `tag`, `filter` |

#### Control Flow (7 ops)

| Op | Description | Arguments |
|----|-------------|-----------|
| `seq` | Sequential execution | `steps` (array) |
| `if` | Conditional branch | `cond`, `then`, `else` |
| `loop` | Fixed-count loop | `count`, `body` |
| `while` | Conditional loop | `cond`, `body` |
| `branch` | Multi-way branch | `value`, `cases` |
| `fork` | Parallel execution | `branches` (array) |
| `merge` | Join parallel results | `sources`, `into`, `strategy` |

#### Advanced (3 ops)

| Op | Description | Arguments |
|----|-------------|-----------|
| `confidence` | Set confidence level | `value`, `source` |
| `yield` | Yield execution | `cycles` |
| `await` | Wait for signal | `signal`, `channel`, `timeout` |

### BNF Grammar

```ebnf
signal_program
  = "{"
      '"name"' ":" string ", "
      '"version"' ":" version_string ", "
      '"agents"' ":" agent_list ", "
      '"program"' ":" signal_expr
    "}"

agent_list
  = "[" string ("," string)* "]"

signal_expr
  = arithmetic_expr | a2a_expr | control_expr | advanced_expr

arithmetic_expr
  = let_expr | binary_expr | unary_expr

let_expr
  = '{"op": "let", "name": ' string ', "value": ' signal_expr '}'

binary_expr
  = '{"op": ' binary_op ', "a": ' signal_expr ', "b": ' signal_expr '}'

binary_op
  = '"add"' | '"sub"' | '"mul"' | '"div"' | '"mod"'
  | '"eq"' | '"neq"' | '"lt"' | '"lte"' | '"gt"' | '"gte"'
  | '"and"' | '"or"' | '"xor"'

unary_expr
  = '{"op": "not", "a": ' signal_expr '}'

a2a_expr
  = tell_expr | ask_expr | delegate_expr | broadcast_expr

tell_expr
  = '{"op": "tell", '
    '"target": ' string ', '
    '"message": ' signal_expr ', '
    '"tag": ' string
  '}'

ask_expr
  = '{"op": "ask", '
    '"target": ' string ', '
    '"message": ' signal_expr ', '
    '"into": ' string
  '}'

delegate_expr
  = '{"op": "delegate", '
    '"target": ' string ', '
    '"task": ' signal_expr ', '
    '"into": ' string
  '}'

broadcast_expr
  = '{"op": "broadcast", '
    '"message": ' signal_expr ', '
    '"tag": ' string ', '
    '"filter": ' signal_expr
  '}'

control_expr
  = seq_expr | if_expr | loop_expr | while_expr | branch_expr
  | fork_expr | merge_expr

seq_expr
  = '{"op": "seq", "steps": [' signal_expr (',' signal_expr)* ']}'

if_expr
  = '{"op": "if", '
    '"cond": ' signal_expr ', '
    '"then": ' signal_expr ', '
    '"else": ' signal_expr
  '}'

loop_expr
  = '{"op": "loop", "count": ' signal_expr ', "body": ' signal_expr '}'

while_expr
  = '{"op": "while", "cond": ' signal_expr ', "body": ' signal_expr '}'

branch_expr
  = '{"op": "branch", '
    '"value": ' signal_expr ', '
    '"cases": [' case (',' case)* ']'
  '}'

case
  = '{"match": ' signal_expr ', "body": ' signal_expr '}'

fork_expr
  = '{"op": "fork", "branches": [' signal_expr (',' signal_expr)* ']}'

merge_expr
  = '{"op": "merge", '
    '"sources": [' string (',' string)* '], '
    '"into": ' string ', '
    '"strategy": ' merge_strategy
  '}'

merge_strategy
  = '"average"' | '"concat"' | '"first"' | '"vote"'

advanced_expr
  = confidence_expr | yield_expr | await_expr

confidence_expr
  = '{"op": "confidence", "value": ' signal_expr ', "source": ' string '}'

yield_expr
  = '{"op": "yield", "cycles": ' signal_expr '}'

await_expr
  = '{"op": "await", '
    '"signal": ' string ', '
    '"channel": ' signal_expr ', '
    '"timeout": ' signal_expr
  '}'
```

### Example: Multi-Agent Compute Pipeline

This Signal program demonstrates a three-stage compute pipeline with data parallelism:

```json
{
  "name": "image-processing-pipeline",
  "version": "1.0",
  "agents": ["splitter", "worker-a", "worker-b", "aggregator"],
  "program": {
    "op": "seq",
    "steps": [
      {
        "op": "tell",
        "target": "splitter",
        "message": {"op": "let", "name": "task", "value": "split-image"},
        "tag": "split-done"
      },
      {
        "op": "await",
        "signal": "split-done",
        "channel": 1,
        "timeout": 0
      },
      {
        "op": "fork",
        "branches": [
          {
            "op": "delegate",
            "target": "worker-a",
            "task": "process-chunk-0",
            "into": "result-a"
          },
          {
            "op": "delegate",
            "target": "worker-b",
            "task": "process-chunk-1",
            "into": "result-b"
          }
        ]
      },
      {
        "op": "merge",
        "sources": ["result-a", "result-b"],
        "into": "final-result",
        "strategy": "concat"
      },
      {
        "op": "tell",
        "target": "aggregator",
        "message": {"op": "let", "name": "data", "value": "final-result"},
        "tag": "pipeline-complete"
      }
    ]
  }
}
```

### Compilation: Signal to FLUX Bytecode

The `signal_compiler.py` performs the following compilation pipeline:

```
Signal JSON
    │
    ├── 1. Parse and validate JSON structure
    ├── 2. Resolve agent references (string → UUID)
    ├── 3. Type-check all expressions
    ├── 4. Lower to FIR (SSA form)
    ├── 5. Optimize FIR (dead code elimination, constant folding)
    ├── 6. Allocate registers (linear scan)
    ├── 7. Emit FLUX bytecode (Format E for A2A ops)
    └── 8. Output: .flux bytecode binary
```

Key compilation rules:
- `tell` → `TELL` (0x50) + payload setup
- `ask` → `ASK` (0x51) + response buffer allocation
- `delegate` → `DELEG` (0x52) + task serialization
- `broadcast` → `BCAST` (0x53) + filter setup
- `fork` → multiple `DELEG` ops (one per branch)
- `merge` → `MERGE` (0x57) + result collection
- `await` → `AWAIT` (0x5B)
- `seq` → linear concatenation of branch bytecodes
- `if` → `JZ`/`JNZ` + two bytecode paths
- `loop` → `LOOP` (0x46) or `MOVI` + `DEC` + `JGT` pattern

---

## 9. Transport Layer

### Design

The A2A transport layer is an abstraction that decouples message routing from the underlying communication mechanism. The default implementation uses in-process mailboxes for maximum performance within a single FLUX VM instance. The abstraction is designed to be extensible to network transports for cross-process and cross-machine communication.

### Transport Interface

All transports must implement the following interface:

```
Transport:
  register_agent(agent_id: UUID, capabilities: CapSet) -> Handle
  unregister_agent(agent_id: UUID) -> void
  send_message(msg: A2AMessage) -> DeliveryResult
  get_messages(agent_id: UUID, max_count: int) -> List[A2AMessage]
  get_message_count(agent_id: UUID) -> int
  broadcast(msg: A2AMessage, filter: Bitmask) -> int
```

### In-Process Mailbox (Default)

The default transport uses per-agent FIFO queues implemented as in-memory buffers:

```
┌──────────────────────────────────────────────────────┐
│                   AgentCoordinator                    │
│                                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐ │
│  │  Mailbox A  │  │  Mailbox B  │  │  Mailbox C   │ │
│  │  ┌───┐┌───┐ │  │  ┌───┐     │  │  (empty)     │ │
│  │  │msg││msg│ │  │  │msg│     │  │              │ │
│  │  └───┘└───┘ │  │  └───┘     │  │              │ │
│  │  head  tail │  │  head  tail│  │  head  tail  │ │
│  └─────────────┘  └─────────────┘  └──────────────┘ │
│                                                       │
│  Trust Table:                                         │
│  ┌──────────────┬──────────────────────────────┐     │
│  │ sender       │ composite_trust             │     │
│  ├──────────────┼──────────────────────────────┤     │
│  │ agent-A      │ 0.911                      │     │
│  │ agent-B      │ 0.743                      │     │
│  │ agent-C      │ 0.512                      │     │
│  └──────────────┴──────────────────────────────┘     │
│                                                       │
│  Capability Table:                                    │
│  ┌──────────────┬──────────────────────────────┐     │
│  │ agent        │ capabilities (bitmask)       │     │
│  ├──────────────┼──────────────────────────────┤     │
│  │ agent-A      │ 0x17 (read|write|exec)      │     │
│  │ agent-B      │ 0x0B (read|write|deleg)     │     │
│  │ agent-C      │ 0x03 (read|write)           │     │
│  └──────────────┴──────────────────────────────┘     │
└──────────────────────────────────────────────────────┘
```

**Mailbox properties:**
- FIFO ordering (strict)
- Default capacity: 64 messages
- Priority queueing: higher-priority messages inserted ahead of lower-priority
- Atomic enqueue/dequeue (thread-safe in multi-threaded VMs)

### Message Ordering Guarantees

| Guarantee | Description |
|-----------|-------------|
| FIFO within priority | Messages of the same priority are delivered in send order |
| Priority ordering | Higher-priority messages are dequeued first |
| Causal ordering (future) | For cross-transport: messages from A arrive at B in the order A sent them |
| Exactly-once (in-process) | In-process delivery guarantees exactly-once semantics |
| At-least-once (network) | Network transports may retry, requiring idempotency |

### Extensible Transports (Future)

The transport abstraction supports pluggable implementations:

| Transport | Status | Description |
|-----------|--------|-------------|
| In-Process Mailbox | **Stable** | Default; zero-copy, FIFO queues |
| HTTP/REST | Planned | POST-based message delivery for cross-process |
| WebSocket | Planned | Bidirectional real-time communication |
| gRPC | Planned | High-performance RPC with streaming support |
| Shared Memory | Planned | Zero-copy inter-process via mmap |

Transport plugins register with the coordinator at initialization:

```
coordinator = AgentCoordinator(transport=InProcessMailboxTransport())
# or
coordinator = AgentCoordinator(transport=HttpTransport(base_url="https://fleet.example.com"))
```

### Error Handling

All transport errors are surfaced through the A2A opcode return values:

| Error | Code | Recovery |
|-------|------|----------|
| Agent not found | 0x01 | Use `DISCOV` to find valid agents |
| Trust too low | 0x02 | Wait for trust to decay-reset or earn trust through successful interactions |
| Capability denied | 0x03 | Request capability via `CAP_REQUEST` |
| Mailbox full | 0x04 | Retry after delay; recipient should process pending messages |
| Invalid payload | 0x05 | Check payload address bounds and format |
| Timeout | 0x04 (ASK) | Increase timeout or implement async pattern with SIGNAL/AWAIT |
| Transport failure | 0xFF | Transport-specific recovery (reconnect, failover) |

---

## 10. Conformance Requirements

A FLUX implementation claims A2A conformance at one of three levels. Each level requires all provisions of the levels below it.

### Level 1 — Core (Minimum Viable A2A)

An implementation at Level 1 supports basic inter-agent messaging:

- [ ] **TELL** (0x50): Send messages between agents
- [ ] **ASK** (0x51): Request-response pattern with blocking semantics
- [ ] **52-byte message header**: Correct layout, little-endian fields, UUID handling
- [ ] **Trust gate**: Composite trust evaluation with at minimum T_history and T_capability dimensions
- [ ] **Trust threshold**: Default 0.3, configurable per agent
- [ ] **Capability verification**: At minimum `send` and `request` capabilities
- [ ] **In-process mailbox**: FIFO delivery, basic priority support
- [ ] **Error codes**: Return at minimum codes 0x00–0x05 in rd

### Level 2 — Standard (Full Opcode Set + Tooling)

An implementation at Level 2 supports the complete A2A opcode set and the Signal compilation toolchain:

- [ ] All 16 A2A opcodes (0x50–0x5F) with correct encoding (Format E) and semantics
- [ ] **INCREMENTS+2 trust engine**: All six dimensions with specified weights and time decay
- [ ] **Capability system**: All five capability types (read, write, execute, delegate, admin)
- [ ] **CAP_REQUIRE / CAP_REQUEST / CAP_GRANT / CAP_REVOKE**: Full dynamic capability management
- [ ] **Capability delegation chains**: Up to 4 levels of delegation depth
- [ ] **Signal compiler**: Parse Signal JSON, validate, emit FLUX bytecode
- [ ] **FIR A2A instructions**: All five FIR instructions with type checking
- [ ] **Memory region integration**: Capability-based access to named memory regions
- [ ] **Priority queueing**: 16-level priority with correct ordering
- [ ] **Agent lifecycle**: FORK/JOIN for child agent management

### Level 3 — Extended (Production Fleet)

An implementation at Level 3 supports production fleet deployment:

- [ ] **Transport plugins**: Pluggable transport interface (HTTP, WebSocket, gRPC, shared memory)
- [ ] **Trust persistence**: Trust scores survive VM restarts (serialized to stable storage)
- [ ] **Capability delegation chains**: Full audit trail of delegation history
- [ ] **Audit records**: Tamper-evident log of all trust and capability changes
- [ ] **Trust penalization escalation**: Progressive penalties for repeated trust failures
- [ ] **Cross-transport causal ordering**: Messages maintain causal order across network boundaries
- [ ] **Heartbeat monitoring**: Coordinator tracks agent liveness via HEARTBT, triggers reconnection
- [ ] **Fleet discovery**: DISCOV with capability-based filtering across multiple VM instances
- [ ] **Signal language completeness**: All 28 operations compilable with no feature gaps

### Test Vectors

Conformance test vectors for A2A are maintained in [flux-conformance](https://github.com/SuperInstance/flux-conformance). The following test categories are defined:

| Category | Count | Description |
|----------|-------|-------------|
| Opcode encoding | 16 | One test per opcode verifying Format E encoding |
| Message format | 8 | Header serialization/deserialization, endianness |
| Trust computation | 12 | Known-answer tests for each dimension and composite |
| Trust gate | 6 | Messages above/below threshold, escalation |
| Capability check | 8 | Grant, revoke, require, delegation chain |
| Transport delivery | 10 | FIFO ordering, priority, broadcast, error cases |
| Signal compilation | 10 | Each operation compiles to valid bytecode |
| FIR lowering | 5 | Each FIR instruction lowers to correct opcodes |

All Level 1 tests must pass for an implementation to claim A2A compatibility. Level 2 and Level 3 tests are required for the respective conformance levels.

---

## Appendix A: Opcode Quick Reference

| Hex | Mnemonic | Encoding | Blocking | Summary |
|-----|----------|----------|----------|---------|
| 0x50 | TELL | `50 rd rs1 rs2` | No | Send message to agent |
| 0x51 | ASK | `51 rd rs1 rs2` | Yes | Request data from agent |
| 0x52 | DELEG | `52 rd rs1 rs2` | No | Delegate task to agent |
| 0x53 | BCAST | `53 rd rs1 rs2` | No | Broadcast to fleet |
| 0x54 | ACCEPT | `54 rd rs1 rs2` | No | Accept delegated task |
| 0x55 | DECLINE | `55 rd rs1 rs2` | No | Decline with reason |
| 0x56 | REPORT | `56 rd rs1 rs2` | No | Report task status |
| 0x57 | MERGE | `57 rd rs1 rs2` | No | Merge agent results |
| 0x58 | FORK | `58 rd rs1 rs2` | No | Spawn child agent |
| 0x59 | JOIN | `59 rd rs1 rs2` | Yes | Wait for child agent |
| 0x5A | SIGNAL | `5A rd rs1 rs2` | No | Emit named signal |
| 0x5B | AWAIT | `5B rd rs1 rs2` | Yes | Wait for signal |
| 0x5C | TRUST | `5C rd rs1 rs2` | No | Set trust level |
| 0x5D | DISCOV | `5D rd rs1 rs2` | Yes | Discover fleet agents |
| 0x5E | STATUS | `5E rd rs1 rs2` | Yes | Query agent status |
| 0x5F | HEARTBT | `5F rd rs1 rs2` | No | Emit heartbeat |

## Appendix B: Trust Weight Summary

```
INCREMENTS+2 Composite Trust:

  T = 0.30·T_history + 0.25·T_capability + 0.20·T_latency
    + 0.15·T_consistency + 0.05·T_determinism + 0.05·T_audit

  Decay: T *= (1 - 0.01 · elapsed/3600)
  Gate:  T >= 0.3 (default threshold)
```

## Appendix C: Message Type Constants

| Constant | Value | Opcode | Direction |
|----------|-------|--------|-----------|
| MSG_TELL | 0x50 | TELL | Sender → Receiver |
| MSG_ASK | 0x51 | ASK | Requester → Responder |
| MSG_ASK_RESP | 0x51 | ASK | Responder → Requester |
| MSG_DELEG | 0x52 | DELEG | Delegator → Worker |
| MSG_BCAST | 0x53 | BCAST | Broadcaster → Fleet |
| MSG_ACCEPT | 0x54 | ACCEPT | Worker → Delegator |
| MSG_DECLINE | 0x55 | DECLINE | Worker → Delegator |
| MSG_REPORT | 0x56 | REPORT | Worker → Supervisor |
| MSG_MERGE | 0x57 | MERGE | Agents → Aggregator |
| MSG_FORK | 0x58 | FORK | Parent → Coordinator |
| MSG_JOIN | 0x59 | JOIN | Child → Parent |
| MSG_SIGNAL | 0x5A | SIGNAL | Emitter → Channel |
| MSG_AWAIT_ACK | 0x5B | AWAIT | Channel → Waiter |
| MSG_TRUST | 0x5C | TRUST | Admin → Coordinator |
| MSG_DISCOV_RESP | 0x5D | DISCOV | Coordinator → Requester |
| MSG_STATUS_RESP | 0x5E | STATUS | Agent → Requester |
| MSG_HEARTBT | 0x5F | HEARTBT | Agent → Coordinator |

---

*This specification is maintained in [flux-spec](https://github.com/SuperInstance/flux-spec) as part of the canonical FLUX language specification. All FLUX implementations supporting inter-agent communication must conform to this document.*
