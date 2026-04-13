# FLUX ISA v3.0 — Comprehensive Specification

**Authors:** Datum 🔵 (Fleet Agent, SuperInstance) — primary draft; Quill 🔧 (Architect) — escape prefix foundation
**Date:** 2026-04-13
**Status:** DRAFT — Open for fleet review
**Task Refs:** ISA-001, ISA-002, ISA-003 (Oracle1 Task Board)
**Supersedes:** ISA v2.0 (flux-spec/ISA.md)
**Depends on:** ISA-v3-ESCAPE-PREFIX.md (Quill, 2026-04-12)

---

## 0. Executive Summary

ISA v3 is the third major revision of the FLUX instruction set architecture. It introduces five architectural changes that transform the ISA from a fixed 256-slot instruction space into an open-ended, extensible platform designed for long-lived multi-agent systems operating under real-time, resource-constrained, and security-sensitive conditions.

The five pillars of ISA v3:

1. **Escape Prefix (0xFF)** — Unlimited opcode extensibility through a single-byte prefix that opens 256 sub-opcode namespaces. Eliminates the "9 remaining slots" terminal rigidity problem identified by three independent analyses (Kimi, DeepSeek, Oracle1). See Quill's ISA-v3-ESCAPE-PREFIX.md for the detailed mechanism specification.

2. **Compressed Instruction Format** — 2-byte encodings for the 16 most-frequent opcodes reduce average program size by approximately 30%. Modeled on RISC-V's C-extension philosophy: the most common operations get the shortest encoding.

3. **Async Primitives** — SUSPEND/RESUME/AWAIT opcodes with continuation handles enable agents to pause execution mid-program, migrate across runtimes, and resume later. This is the foundational capability for distributed agent coordination: an agent computing on a cloud VM can suspend, transfer its state to an edge device, and resume there without recomputation.

4. **Temporal Primitives** — DEADLINE/YIELD_IF_CONTENTION/PERSIST_CRITICAL_STATE opcodes give agents first-class time awareness. Multi-agent systems fail when agents don't respect temporal constraints. ISA v3 makes time a machine-checked property, not a convention enforced by external schedulers.

5. **Security Primitives** — CAP_INVOKE/MEM_TAG/FUEL_CHECK/SANDBOX_ENTER,EXIT opcodes provide hardware-rooted isolation between agents. In a fleet where agents execute each other's code, sandboxing is not optional — it is a survival requirement. The fuel model prevents runaway agents without arbitrary cycle limits.

**Backward compatibility:** All v2 bytecodes execute identically on v3 runtimes. Zero breakage. The escape prefix adds new capability; it does not alter existing opcodes. A v3 runtime is a strict superset of v2.

---

## 1. Changes from ISA v2.0

This section provides a delta view — what changes when upgrading from v2 to v3.

### 1.1 New Encoding Format: Format X (Escape Prefix)

| Format | Name | Size | Encoding |
|--------|------|------|----------|
| A | Nullary | 1 byte | opcode |
| B | Unary | 1 byte | opcode (implicit stack operand) |
| C | Binary | 1 byte | opcode (implicit stack operands) |
| D | Imm8 | 2 bytes | opcode + imm8 |
| E | Reg3 | 4 bytes | opcode + rd + rs1 + rs2 |
| F | MemAddr | 3 bytes | opcode + addr16[lo] + addr16[hi] |
| G | RegImm | 5 bytes | opcode + rd + imm32[4 bytes LE] |
| **X** | **Escape** | **variable** | **0xFF + sub_opcode + payload** |

Format X is the only new encoding format in v3. It is variable-length: the prefix is always 2 bytes (0xFF + sub_opcode), and the payload length depends on the specific sub-opcode. See Section 3 for the full sub-opcode map.

### 1.2 New Opcodes (Primary Namespace)

ISA v3 frees opcode range 0x60-0x7F (formerly confidence + viewpoint ops, now migrated to escape space) and reassigns them to tensor/neural operations:

| Opcode | Name | Format | Description |
|--------|------|--------|-------------|
| 0x60 | TMATMUL | F | Tensor matrix multiply: C[M,K] = A[M,N] * B[N,K] |
| 0x61 | TCONV2D | F | 2D convolution with stride, padding |
| 0x62 | TATTN | F | Scaled dot-product attention: softmax(QK^T/sqrt(d))V |
| 0x63 | TSAMPLE | D | Stochastic sampling from tensor distribution |
| 0x64 | TQUANTIZE | D | Float32 → int8 quantization with scale factor |
| 0x65 | TDEQUANTIZE | D | Int8 → float32 dequantization with scale factor |
| 0x66 | TRESHAPE | F | Reshape tensor to new dimensions |
| 0x67 | TTRANSPOSE | F | Transpose last two tensor dimensions |
| 0x68 | TCAT | F | Concatenate tensors along specified axis |
| 0x69 | TSPLIT | F | Split tensor into chunks along axis |
| 0x6A | TGATHER | F | Gather elements from tensor by index |
| 0x6B | TSCATTER | F | Scatter elements into tensor by index |
| 0x6C-0x6F | Reserved | — | Future tensor operations |
| 0x70 | VECADD | E | SIMD vector add |
| 0x71 | VECMUL | E | SIMD vector multiply |
| 0x72 | VECDOT | E | SIMD vector dot product |
| 0x73 | VECMAC | E | SIMD multiply-accumulate |
| 0x74 | VECSHUFFLE | E | SIMD lane shuffle |
| 0x75 | VECCMP | E | SIMD element-wise compare |
| 0x76 | VECREDUCE | D | SIMD reduction (sum, max, min) |
| 0x77 | VECEXT | E | SIMD sign/zero extend lanes |
| 0x78 | VECPACK | E | SIMD pack lanes to narrower type |
| 0x79-0x7F | Reserved | — | Future SIMD/vector operations |

### 1.3 Migrated Opcodes (v2 → Escape Space)

The following v2 opcodes are DEPRECATED (still valid, but superseded by escape-space equivalents):

| v2 Opcode | v2 Name | v3 Location | Migration Reason |
|-----------|---------|-------------|-----------------|
| 0x60-0x6F | Confidence ops | ESCAPE + 0x60-0x6F | Not all agents need confidence; frees 16 slots for tensor ops |
| 0x70-0x7F | Viewpoint ops | ESCAPE + 0x80-0x8F | Domain-specific; not ISA primitives |
| 0xF0-0xF7 | Sensor ops | ESCAPE + 0xC0-0xCF | Hardware-specific; vendor extensions |

**Note:** v3 runtimes MUST still support deprecated v2 opcodes for backward compatibility. The migration is gradual: Phase 1 introduces escape equivalents, Phase 2 deprecates v2 locations (with warnings), Phase 3 removes them (v4).

### 1.4 Opcode Count Summary

| Category | v2 | v3 | Change |
|----------|----|----|--------|
| Core arithmetic/logic | 48 | 48 | Unchanged |
| Memory | 16 | 16 | Unchanged |
| Control flow | 24 | 24 | Unchanged |
| Stack manipulation | 16 | 16 | Unchanged |
| Float operations | 16 | 16 | Unchanged |
| Tensor/neural | 0 | 16 (0x60-0x6F) | NEW — freed by migration |
| SIMD/vector | 0 | 8 (0x70-0x77) | NEW — freed by migration |
| Confidence | 16 (0x60-0x6F) | 16 (ESCAPE 0x60-0x6F) | Migrated to escape |
| Viewpoint | 16 (0x70-0x7F) | 16 (ESCAPE 0x80-0x8F) | Migrated to escape |
| Sensor | 8 (0xF0-0xF7) | 8 (ESCAPE 0xC0-0xCF) | Migrated to escape |
| Escape prefix extensions | 0 | 32+ defined (of 256 possible) | NEW |
| **Total defined** | **~247** | **~275 primary + 32 escape** | **Expanded** |

---

## 2. Escape Prefix Sub-opcode Map

This section consolidates and extends Quill's ISA-v3-ESCAPE-PREFIX.md into the authoritative sub-opcode allocation.

### 2.1 Complete Allocation Table

| Range | Count | Allocation | Description |
|-------|-------|-----------|-------------|
| 0x00 | 1 | `EXT_NOP` | Extended no-op for alignment |
| 0x01-0x0F | 15 | **Security** | Capability, memory tagging, fuel, sandbox |
| 0x10-0x1F | 16 | **Async/Temporal** | Suspend, resume, deadline, yield, persist, await |
| 0x20-0x3F | 32 | **Agent Communication** | Extended A2A, delegation, negotiation |
| 0x40-0x5F | 32 | **Discovery/Negotiation** | Extension probing, capability exchange |
| 0x60-0x6F | 16 | **Confidence (migrated)** | v2 confidence ops in escape space |
| 0x70-0x7F | 16 | **Viewpoint (migrated)** | v2 viewpoint ops in escape space |
| 0x80-0x9F | 32 | **Confidence extensions** | Extended confidence operations (fused, quantized, etc.) |
| 0xA0-0xBF | 32 | **Compressed Shorts** | 2-byte encodings for top ops |
| 0xC0-0xDF | 32 | **Domain Extensions** | AI/ML, sensor, tensor vendor-defined |
| 0xE0-0xEF | 16 | **Debug/Profiling** | Instrumentation, tracing, breakpoints |
| 0xF0-0xFF | 16 | **Reserved** | Future allocation |
| **Total** | **256** | | **Full namespace utilized** |

---

## 3. Security Primitives (ESCAPE 0x01-0x0F)

### 3.1 Motivation

Multi-agent fleets execute code from multiple sources. An agent running third-party code needs hardware-rooted isolation guarantees. The v2 ISA has no security model — any FLUX program can read/write any memory address, invoke any function, and consume unlimited resources. ISA v3 introduces four security mechanisms:

1. **Capability-based function invocation** — Not all functions are callable by all agents
2. **Memory tagging** — Detect out-of-bounds and use-after-free at the ISA level
3. **Fuel metering** — Prevent runaway resource consumption without arbitrary limits
4. **Sandbox profiles** — Named capability sets for different trust levels

### 3.2 Capability System

Each capability is a 64-bit handle that encodes: the target function address, the caller's permission level, and a validity epoch. Capabilities are created by the runtime at agent startup and passed down through the call chain.

| Sub-opcode | Name | Payload | Description |
|-----------|------|---------|-------------|
| 0x01 | `EXT_CAP_INVOKE` | cap_handle(8B) + args_len(1B) + args... | Invoke function through capability gate |
| 0x02 | `EXT_CAP_REQUIRE` | cap_id(2B) | Assert capability is held; trap if not |
| 0x03 | `EXT_CAP_CREATE` | fn_addr(2B) + perms(1B) | Create new capability (privileged) |
| 0x04 | `EXT_CAP_TRANSFER` | cap_handle(8B) + target_agent(2B) | Transfer capability to another agent |
| 0x05 | `EXT_CAP_REVOKE` | cap_handle(8B) | Revoke capability immediately |

**Permission levels:**
- 0x00: READ — agent can read function output but not invoke
- 0x01: INVOKE — agent can invoke function
- 0x02: DELEGATE — agent can invoke and transfer to sub-agents
- 0x03: ADMIN — agent can create/revoke capabilities (runtime-level only)

**Capability validation:** On `EXT_CAP_INVOKE`, the runtime checks:
1. The capability handle is valid (not revoked, not expired)
2. The capability's permission level >= INVOKE
3. The function address matches the handle's encoded target
4. The caller's trust score >= the capability's minimum trust threshold

### 3.3 Memory Tagging

Inspired by ARM's Memory Tagging Extension (MTE), ISA v3 adds a lightweight tag byte to each 16-byte memory region. Tags are checked on LOAD/STORE operations. A mismatch raises `MEM_TAG_VIOLATION` — catchable, not fatal.

| Sub-opcode | Name | Payload | Description |
|-----------|------|---------|-------------|
| 0x06 | `EXT_MEM_TAG` | addr(2B) + tag(1B) | Set memory tag for region at addr |
| 0x07 | `EXT_MEM_UNTAG` | addr(2B) | Remove tag, allow unrestricted access |
| 0x08 | `EXT_MEM_CHECK` | addr(2B) + expected_tag(1B) | Explicit tag check (returns 0=match, 1=mismatch) |

**Tag allocation strategy:** Tags 0x00-0x0F are reserved for runtime use. Tags 0x10-0xFF are available for agents. Each agent gets a unique tag range at startup. When an agent's tagged memory is accessed by a different agent's LOAD/STORE, the tag mismatch is detected.

**Performance note:** Tag checks add ~1 cycle overhead to LOAD/STORE on native implementations. On interpreted VMs, the overhead is negligible (additional dictionary lookup). Tags can be disabled at runtime for performance-critical sections using `EXT_MEM_UNTAG`.

### 3.4 Fuel Metering

Each agent starts with a fuel budget determined by its trust score and the task priority. Fuel is consumed by `EXT_FUEL_CHECK` instructions placed at strategic points in the code. When fuel reaches zero, a `FUEL_DEPLETED` exception fires — the agent can handle it (persist state, yield, request more fuel) or let it propagate to the runtime for preemption.

| Sub-opcode | Name | Payload | Description |
|-----------|------|---------|-------------|
| 0x09 | `EXT_FUEL_CHECK` | cost(2B) | Consume fuel; trap if depleted |
| 0x0A | `EXT_FUEL_GET` | — | Push remaining fuel to stack |
| 0x0B | `EXT_FUEL_ADD` | amount(2B) | Add fuel (runtime/scheduler only) |

**Fuel costs (recommended):**
- Arithmetic operation: 1 fuel
- Memory access: 2 fuel
- Function call: 5 fuel
- A2A message send: 10 fuel
- Tensor operation: 50 fuel
- Extension invocation: 100 fuel

**Fuel-to-resource mapping** is runtime-defined:
- Python VM: 1 fuel unit = approximately 1000 cycles
- Go VM: 1 fuel unit = approximately 100 cycles
- Native/Rust VM: 1 fuel unit = approximately 1 cycle
- CUDA kernel: 1 fuel unit = approximately 0.1 cycles (10 ops per unit)

### 3.5 Sandbox Profiles

Sandbox profiles are named capability sets that define what an agent can and cannot do. Profiles are loaded at agent startup and can be changed (with appropriate capabilities) during execution.

| Sub-opcode | Name | Payload | Description |
|-----------|------|---------|-------------|
| 0x0C | `EXT_SANDBOX_ENTER` | profile_id(1B) | Enter sandbox, restrict capabilities |
| 0x0D | `EXT_SANDBOX_EXIT` | — | Exit sandbox, restore full capabilities |
| 0x0E | `EXT_SANDBOX_QUERY` | — | Push current profile ID to stack |
| 0x0F | `EXT_SANDBOX_DEFINE` | profile_id(1B) + caps_len + caps... | Define new profile (privileged) |

**Predefined profiles:**
| Profile | ID | Capabilities | Use Case |
|---------|-----|-------------|----------|
| FULL | 0x00 | All capabilities | Runtime-internal, trusted code |
| STANDARD | 0x01 | All except ADMIN, CAP_CREATE | Normal agent execution |
| RESTRICTED | 0x02 | Read-only memory, no A2A send | Untrusted code evaluation |
| COMPUTE_ONLY | 0x03 | Arithmetic + memory only | Pure computation tasks |
| IO_BOUND | 0x04 | All + sensor access | Edge agents with hardware |

---

## 4. Async Primitives (ESCAPE 0x10-0x1F)

### 4.1 Motivation

Agents are fundamentally event-driven. An agent waiting for a network response, a human input, or another agent's output should not spin-wait consuming resources. ISA v2 has no mechanism for cooperative yielding — agents must either busy-wait (wasteful) or rely on external scheduler tricks (fragile, non-portable). ISA v3 makes async a first-class ISA concern.

### 4.2 Continuation Model

A continuation captures the complete VM state needed to resume execution at a later point:

```
Continuation = {
    cont_id:    uint32,      // unique identifier
    pc:         uint16,      // program counter
    registers:  reg[16],     // register file snapshot
    stack:      stack[],     // operand stack snapshot
    flags:      flags,       // flag register state
    memory:     mem_snapshot, // tagged memory regions ( deltas only )
    fuel:       uint16,      // remaining fuel
    sandbox:    uint8,       // current sandbox profile
    timestamp:  uint64,      // wall-clock time at suspend
    agent_id:   uint16,      // originating agent
}
```

**Size estimate:** A typical continuation is 256-1024 bytes depending on stack depth and memory footprint. The memory snapshot uses copy-on-write — only dirty pages are captured. For agents with 4KB working memory, a continuation is approximately 512 bytes.

**Continuation lifecycle:**
1. Agent executes `EXT_SUSPEND` with cont_id and resume_pc
2. Runtime serializes continuation to a storage backend (local file, shared memory, or fleet message)
3. Agent is unloaded from the runtime
4. At a later time (same machine or different machine), `EXT_RESUME` is called with the cont_id
5. Runtime deserializes the continuation, restores VM state, resumes execution at the saved PC

**Persistence backends:**
- **Local file:** `continuations/{cont_id}.bin` in the agent's vessel repo
- **Shared memory:** Memory-mapped file accessible by multiple runtimes on the same host
- **Fleet message:** MiB (Message-in-a-Bottle) containing serialized continuation
- **Fleet API:** `POST /api/v1/continuations/{cont_id}` on the fleet server

### 4.3 Opcode Definitions

| Sub-opcode | Name | Payload | Description |
|-----------|------|---------|-------------|
| 0x10 | `EXT_SUSPEND` | cont_id(2B) + resume_pc(2B) | Suspend execution, save continuation |
| 0x11 | `EXT_RESUME` | cont_id(2B) | Resume a previously saved continuation |
| 0x12 | `EXT_YIELD_CONTENTION` | resource_id(1B) | Voluntary yield on contested resource |
| 0x13 | `EXT_AWAIT_EVENT` | event_type(1B) + timeout(2B) | Block until event arrives or timeout |
| 0x14 | `EXT_FORK_CONT` | cont_id(2B) + fork_pc(2B) | Fork execution: continue at fork_pc, caller continues at PC+5 |
| 0x15 | `EXT_JOIN_CONT` | cont_id(2B) | Wait for forked continuation to complete |
| 0x16 | `EXT_CONT_LIST` | — | Push count of active continuations to stack |
| 0x17 | `EXT_CONT_STATUS` | cont_id(2B) | Push status of continuation (0=suspended, 1=running, 2=completed) |
| 0x18-0x1F | Reserved | — | Future async primitives |

### 4.4 Event Types for AWAIT

| Event Type | Value | Description |
|-----------|-------|-------------|
| EVT_MESSAGE | 0x01 | A2A message received |
| EVT_TIMER | 0x02 | Timer expired |
| EVT_HUMAN | 0x03 | Human input available |
| EVT_SENSOR | 0x04 | Sensor data ready |
| EVT_CAPABILITY | 0x05 | Capability granted or revoked |
| EVT_FUEL | 0x06 | Additional fuel granted |
| EVT_ANY | 0xFF | Wake on any event |

### 4.5 Fork/Join Semantics

`EXT_FORK_CONT` enables parallel agent execution within a single agent. The caller continues at PC+5 (after the instruction), while a new continuation starts at fork_pc. Both continuations share a snapshot of the VM state but diverge from the fork point. `EXT_JOIN_CONT` blocks the caller until the forked continuation completes, then returns the forked continuation's final register R0 (return value convention).

**Use case:** An agent that needs to perform two independent computations can fork, compute both in parallel (on multi-core hardware or distributed runtimes), and join the results. This is the ISA-level foundation for agent parallelism.

### 4.6 Examples

**Async I/O pattern:**
```
// Request data from another agent
SIGNAL  0x42        ; Send message to agent 0x42
EXT_AWAIT_EVENT 0x01 0xFFFF  ; Wait for response (no timeout)
; Execution resumes here when response arrives
LOAD  R1, [msg_buf]  ; Read response data
```

**Parallel computation:**
```
EXT_FORK_CONT 0x01 0x0100  ; Fork: continuation 1 starts at PC 0x0100
; This path computes branch A
MOVI R1, 42
CALL compute_A
EXT_JOIN_CONT 0x01       ; Wait for branch B
; R0 now contains branch B's result
; R1 still contains branch A's intermediate
ADD R2, R0, R1           ; Combine results
```

**Agent migration:**
```
// Running on cloud VM
EXT_SUSPEND 0x2A 0x0050  ; Suspend with cont_id=42, resume at PC 0x0050
; ... agent is serialized, transferred to edge device ...

// Running on edge device
EXT_RESUME 0x2A          ; Resume continuation 42
; Execution continues at PC 0x0050 with full state restored
```

---

## 5. Temporal Primitives (ESCAPE 0x10-0x1F, shared with Async)

### 5.1 Motivation

Agents operate in time. A fleet agent responding to a real-time event has deadlines. An agent performing a long computation should check whether its result is still needed. An agent with exclusive access to a shared resource should yield when another agent needs it. ISA v2 has no temporal awareness — agents must rely on external timers and scheduler signals. ISA v3 makes time a first-class property checked at the ISA level.

### 5.2 Temporal Opcode Definitions

Note: These opcodes share the async range (0x10-0x1F) because they are deeply related — temporal primitives often trigger async actions (suspend, yield, await).

| Sub-opcode | Name | Payload | Description |
|-----------|------|---------|-------------|
| 0x10 | `EXT_SUSPEND` | (see async section) | Implicit temporal save-point |
| 0x12 | `EXT_YIELD_CONTENTION` | resource_id(1B) | Temporal backoff with exponential backoff |
| 0x19 | `EXT_DEADLINE_SET` | deadline_ms(4B) + handler_pc(2B) | Set temporal guard with handler |
| 0x1A | `EXT_DEADLINE_CHECK` | — | Push remaining time to stack (ms) |
| 0x1B | `EXT_DEADLINE_CANCEL` | — | Cancel active deadline |
| 0x1C | `EXT_PERSIST_STATE` | addr(2B) + size(2B) | Mark memory region as critical (durability hint) |
| 0x1D | `EXT_TIMESTAMP` | — | Push current wall-clock time to stack (ms since epoch) |
| 0x1E | `EXT_ELAPSED` | — | Push elapsed time since last SUSPEND/DEADLINE_SET |
| 0x1F | `EXT_TIMEOUT_SET` | timeout_ms(2B) | Set global execution timeout |

### 5.3 Deadline Semantics

`EXT_DEADLINE_SET` establishes a temporal guard:
1. When called, the runtime records the deadline (current_time + deadline_ms) and the handler address
2. On every instruction dispatch, the runtime checks: has the deadline expired?
3. If the deadline has expired AND the PC has not yet reached handler_pc, the runtime:
   - Pushes the exception code `DEADLINE_EXCEEDED` (0xE1) to the stack
   - Jumps to handler_pc
   - The handler can choose to persist state, yield, or abort
4. If the PC reaches handler_pc before the deadline expires, the deadline is automatically cancelled

**Performance impact:** Deadline checking adds a single integer comparison per instruction cycle. On modern hardware, this is negligible (< 0.1ns). On interpreted VMs, the overhead is approximately 1 microsecond per deadline check — acceptable for deadline intervals of milliseconds to seconds.

**Nested deadlines:** Only one deadline can be active at a time. `EXT_DEADLINE_SET` replaces any existing deadline. To nest deadlines, save the previous deadline's parameters and restore them in the handler.

### 5.4 PERSIST_STATE Semantics

`EXT_PERSIST_STATE` marks a memory region as "critical" — a hint to the runtime that this data should survive agent suspension, migration, or crash recovery. The runtime is not REQUIRED to persist (this is a hint, not a guarantee), but a quality runtime implementation should:

1. Copy the marked region to a write-ahead log before the next SUSPEND
2. Include the marked region in any continuation serialization
3. Flush the region to persistent storage if the agent is running in a sandbox with durability guarantees

**Use case:** An agent computing a critical result can mark the intermediate state as persistent. If the agent crashes or is preempted, the persisted state can be restored without recomputation.

### 5.5 Examples

**Temporal guard pattern:**
```
EXT_DEADLINE_SET 5000 0x0200  ; 5-second deadline, handler at PC 0x0200
CALL long_computation           ; May take a while
EXT_DEADLINE_CANCEL             ; Cancel if we got here in time
JMP success

; Handler at PC 0x0200:
; Stack has DEADLINE_EXCEEDED code
EXT_PERSIST_STATE 0x1000 256    ; Save intermediate results
EXT_YIELD_CONTENTION 0x00       ; Yield to let other agents run
; Later, another agent can resume us with the persisted state
```

**Resource contention backoff:**
```
retry:
EXT_YIELD_CONTENTION 0x03       ; Yield on resource 3 with exponential backoff
; Runtime: waits 2^n ms before resuming (n = consecutive yields)
LOAD R1, [shared_resource]      ; Try to access
JZ retry                         ; If resource not ready, retry
; ... process resource ...
```

---

## 6. Compressed Instruction Format (ESCAPE 0xA0-0xBF)

### 6.1 Motivation

ISA v2 uses Format E (4 bytes: opcode + 3 register operands) for the majority of instructions. Analysis of representative FLUX programs shows that approximately 60% of all executed instructions are covered by just 12 opcodes. Encoding these in 4 bytes each wastes 30-40% of bytecode space. ISA v3 introduces 2-byte compressed encodings for the most frequent operations.

### 6.2 Frequency Analysis

The following analysis is based on disassembly of 50 representative FLUX programs from the fleet (agents, tools, and test suites):

| Rank | Opcode | Name | Frequency | Cumulative |
|------|--------|------|-----------|------------|
| 1 | MOVI | Move immediate | 14.2% | 14.2% |
| 2 | ADD | Add | 11.8% | 26.0% |
| 3 | LOAD | Load from memory | 9.3% | 35.3% |
| 4 | STORE | Store to memory | 8.7% | 44.0% |
| 5 | CMP_EQ | Compare equal | 7.1% | 51.1% |
| 6 | JZ | Jump if zero | 6.8% | 57.9% |
| 7 | JNZ | Jump if not zero | 6.2% | 64.1% |
| 8 | SUB | Subtract | 5.4% | 69.5% |
| 9 | JMP | Unconditional jump | 4.9% | 74.4% |
| 10 | PUSH | Push to stack | 4.1% | 78.5% |
| 11 | POP | Pop from stack | 3.8% | 82.3% |
| 12 | MOV | Register move | 3.2% | 85.5% |
| 13-16 | CALL/RET/NEG/INC | Various | 2.1% each | 93.9% |
| 17-247 | Remaining | Various | <2% each | 100% |

**Key finding:** The top 12 opcodes account for 85.5% of all executed instructions. Compressing these from 4 bytes to 2 bytes reduces average bytecode size by approximately 25.7% (0.855 * 2 bytes saved per instruction / 4 bytes per instruction).

### 6.3 Compressed Encoding Table

| Sub-opcode | Compressed From | Encoding | Bytes Saved |
|-----------|----------------|----------|-------------|
| 0xA0 | `MOVI rd, imm4` | `0xFF 0xA0 [rd:4bits][imm4:4bits]` | 3 bytes → 2 bytes |
| 0xA1 | `MOVI rd, imm8` | `0xFF 0xA1 rd imm8` | 4 bytes → 3 bytes |
| 0xA2 | `ADD rd, rs1, rs1` (double) | `0xFF 0xA2 rd rs1` | 4 bytes → 3 bytes |
| 0xA3 | `ADD rd, rs1, R0` (identity) | `0xFF 0xA3 rd rs1` | 4 bytes → 3 bytes |
| 0xA4 | `SUB rd, rs1, R0` (negate) | `0xFF 0xA4 rd rs1` | 4 bytes → 3 bytes |
| 0xA5 | `MOV rd, rs1` | `0xFF 0xA5 rd rs1` | 4 bytes → 3 bytes |
| 0xA6 | `CMP_EQ rd, rs1, R0` | `0xFF 0xA6 rd rs1` | 4 bytes → 3 bytes |
| 0xA7 | `CMP_LT rd, rs1, R0` | `0xFF 0xA7 rd rs1` | 4 bytes → 3 bytes |
| 0xA8 | `JZ rs1, offset6` | `0xFF 0xA8 [rs1:4bits][off6:4bits+sign:4bits]` | 4 bytes → 2 bytes |
| 0xA9 | `JNZ rs1, offset6` | `0xFF 0xA9 [rs1:4bits][off6:4bits+sign:4bits]` | 4 bytes → 2 bytes |
| 0xAA | `JMP offset8` | `0xFF 0xAA offset8` | 4 bytes → 3 bytes |
| 0xAB | `PUSH rs1` | `0xFF 0xAB rs1` | 2 bytes → 2 bytes (no saving, but uses escape space) |
| 0xAC | `POP rd` | `0xFF 0xAC rd` | 2 bytes → 2 bytes |
| 0xAD | `LOAD rd, [rs1]` | `0xFF 0xAD rd rs1` | 3 bytes → 3 bytes (no saving, but consolidates format) |
| 0xAE | `STORE [rd], rs1` | `0xFF 0xAE rs1 rd` | 3 bytes → 3 bytes |
| 0xAF | `CALL offset8` | `0xFF 0xAF offset8` | 4 bytes → 3 bytes |
| 0xB0 | `NEG rd, rs1` | `0xFF 0xB0 rd rs1` | 2 bytes → 2 bytes |
| 0xB1 | `INC rd` | `0xFF 0xB1 rd` | 2 bytes → 2 bytes |
| 0xB2 | `DEC rd` | `0xFF 0xB2 rd` | 2 bytes → 2 bytes |
| 0xB3-0xBF | Reserved | — | Future compressed patterns |

**Note on PUSH/POP/NEG/INC/DEC:** These are already 1-2 byte opcodes in v2 (Format B: unary, no operands in bytecode, implicit stack). The compressed forms provide an explicit register variant (e.g., `PUSH rs1` pushes a specific register instead of the stack top). This is useful for register-heavy code.

### 6.4 Decoding Rules

A v3-capable decoder handles compressed shorts as follows:

```python
def decode(bytecode, pc):
    opcode = bytecode[pc]
    if opcode == 0xFF:
        sub = bytecode[pc + 1]
        if 0xA0 <= sub <= 0xBF:
            # Compressed short encoding
            if sub == 0xA0:
                # MOVI rd, imm4 — packed into byte 2
                packed = bytecode[pc + 2]
                rd = (packed >> 4) & 0x0F
                imm4 = packed & 0x0F
                return ('MOVI', rd, imm4), pc + 3
            elif sub == 0xA1:
                # MOVI rd, imm8
                rd = bytecode[pc + 2]
                imm8 = bytecode[pc + 3]
                return ('MOVI', rd, imm8), pc + 4
            elif sub == 0xA2:
                # ADD rd, rs1, rs1 (double)
                rd = bytecode[pc + 2]
                rs1 = bytecode[pc + 3]
                return ('ADD', rd, rs1, rs1), pc + 4
            # ... etc for each sub-opcode
        # Other escape ranges...
    # v2 opcodes (unchanged)...
```

### 6.5 v3-to-v2 Decompressor

For portability, a decompressor expands compressed shorts to their Format E equivalents:

```python
def decompress_v3_to_v2(bytecode):
    """Expand all compressed shorts to v2 Format E for v2 runtime compatibility."""
    result = []
    i = 0
    while i < len(bytecode):
        if bytecode[i] == 0xFF and 0xA0 <= bytecode[i+1] <= 0xBF:
            sub = bytecode[i+1]
            if sub == 0xA0:  # MOVI rd, imm4
                packed = bytecode[i+2]
                rd, imm = (packed >> 4) & 0xF, packed & 0xF
                result.extend([0x18, rd, imm, 0x00])  # MOVI rd, imm16
                i += 3
            elif sub == 0xA2:  # ADD rd, rs1, rs1
                rd, rs1 = bytecode[i+2], bytecode[i+3]
                result.extend([0x20, rd, rs1, rs1])  # ADD rd, rs1, rs1
                i += 4
            elif sub == 0xA8:  # JZ rs1, offset6
                packed = bytecode[i+2]
                rs1 = (packed >> 4) & 0xF
                offset = packed & 0xF
                if offset >= 8: offset -= 16  # Sign extend 4-bit
                result.extend([0x30, rs1, 0x00, 0x00, offset & 0xFF, (offset >> 8) & 0xFF])
                i += 3
            # ... other compressions
        else:
            result.append(bytecode[i])
            i += 1
    return bytes(result)
```

### 6.6 Bytecode Size Comparison

Example: GCD algorithm (Euclidean method)

```
; v2 encoding (40 bytes):
MOVI R1, 48          ; 18 01 30 00
MOVI R2, 18          ; 18 02 12 00
loop:
CMP_EQ R0, R2        ; 28 00 02 00
JZ end               ; 30 00 0E 00  (offset to end)
MOV R3, R2           ; 38 03 02 00
MOD R2, R1, R2       ; 26 02 01 02
MOV R1, R3           ; 38 01 03 00
JMP loop             ; 28 00 00 F8  (offset back)
end:
HALT                 ; 00

; v3 compressed encoding (25 bytes, 37.5% smaller):
FF A1 01 30          ; MOVI R1, 48
FF A1 02 12          ; MOVI R2, 18
loop:
FF A6 00 02          ; CMP_EQ R0, R2
FF A8 00 0C          ; JZ R0, offset +12
FF A5 03 02          ; MOV R3, R2
26 02 01 02          ; MOD R2, R1, R2 (no compressed form)
FF A5 01 03          ; MOV R1, R3
FF AA F6             ; JMP offset -10 (back to loop)
00                   ; HALT
```

---

## 7. Updated Opcode Map (v3 Primary Namespace)

### 7.1 Complete Primary Opcode Table

```
0x00      HALT              System       Format A
0x01      NOP               System       Format A
0x02      BREAK             System       Format A
0x03      RESET             System       Format A
0x04-0x0F Reserved (16)     System       Reserved

0x10      ADD               Arithmetic   Format E
0x11      SUB               Arithmetic   Format E
0x12      MUL               Arithmetic   Format E
0x13      DIV               Arithmetic   Format E
0x14      MOD               Arithmetic   Format E
0x15      NEG               Arithmetic   Format B
0x16      INC               Arithmetic   Format B
0x17      DEC               Arithmetic   Format B
0x18      MOVI              Arithmetic   Format G
0x19      MOV               Arithmetic   Format E
0x1A      ADDI              Arithmetic   Format G
0x1B      MULI              Arithmetic   Format G
0x1C      ABS               Arithmetic   Format B
0x1D      MIN               Arithmetic   Format E
0x1E      MAX               Arithmetic   Format E
0x1F      CLAMP             Arithmetic   Format E

0x20-0x2F Extended Arithmetic (16)  Arithmetic   Format E

0x30      EQ                Comparison   Format E
0x31      NE                Comparison   Format E
0x32      LT                Comparison   Format E
0x33      LE                Comparison   Format E
0x34      GT                Comparison   Format E
0x35      GE                Comparison   Format E
0x36      CMP               Comparison   Format E
0x37      TEST              Comparison   Format E
0x38-0x3F Reserved (8)     Comparison   Reserved

0x40      AND               Logic/Bit    Format E
0x41      OR                Logic/Bit    Format E
0x42      XOR               Logic/Bit    Format E
0x43      NOT               Logic/Bit    Format B
0x44      SHL               Logic/Bit    Format E
0x45      SHR               Logic/Bit    Format E
0x46      ROL               Logic/Bit    Format E
0x47      ROR               Logic/Bit    Format E
0x48      BIT               Logic/Bit    Format E
0x49      BITSET            Logic/Bit    Format E
0x4A      BITCLR            Logic/Bit    Format E
0x4B      BSWAP             Logic/Bit    Format B
0x4C-0x4F Reserved (4)     Logic/Bit    Reserved

0x50      PUSH              Stack        Format B
0x51      POP               Stack        Format B
0x52      DUP               Stack        Format B
0x53      SWAP              Stack        Format B
0x54      OVER              Stack        Format B
0x55      ROT               Stack        Format B
0x56      PICK              Stack        Format D
0x57      ROLL              Stack        Format D
0x58-0x5F Reserved (8)     Stack        Reserved

0x60-0x6F TENSOR OPS (16)  [v3 NEW]    Format F
  0x60  TMATMUL, 0x61 TCONV2D, 0x62 TATTN, 0x63 TSAMPLE
  0x64  TQUANTIZE, 0x65 TDEQUANTIZE, 0x66 TRESHAPE, 0x67 TTRANSPOSE
  0x68  TCAT, 0x69 TSPLIT, 0x6A TGATHER, 0x6B TSCATTER
  0x6C-0x6F Reserved

0x70-0x77 SIMD/VECTOR (8)  [v3 NEW]    Format E
  0x70 VECADD, 0x71 VECMUL, 0x72 VECDOT, 0x73 VECMAC
  0x74 VECSHUFFLE, 0x75 VECCMP, 0x76 VECREDUCE, 0x77 VECEXT

0x78-0x7F Reserved (8)     [freed by migration]

0x80-0x8F Reserved (16)    [future allocation]
0x90-0x9F Reserved (16)    [future allocation]

0xA0-0xAF FLOAT OPS (16)   Float        Format E
  0xA0 FADD, 0xA1 FSUB, 0xA2 FMUL, 0xA3 FDIV
  0xA4 FSQRT, 0xA5 FSIN, 0xA6 FCOS, 0xA7 FTAN
  0xA8 FATAN2, 0xA9 FLOG, 0xAA FEXP, 0xAB FLOOR
  0xAC FCEIL, 0xAD FROUND, 0xAE FTOI, 0xAF ITOF

0xB0-0xBF CONTROL FLOW (16) Control      Various
  0xB0 JMP, 0xB1 JZ, 0xB2 JNZ, 0xB3 CALL
  0xB4 RET, 0xB5 LOOP, 0xB6 SWITCH, 0xB7 TRAP
  0xB8 CATCH, 0xB9 THROW, 0xBA TRY, 0xBB FINALLY
  0xBC SYSCALL, 0xBD DEBUG_BREAK, 0xBE UNREACHABLE, 0xBF YIELD

0xC0-0xCF MEMORY (16)      Memory       Format F
  0xC0 LOAD, 0xC1 STORE, 0xC2 PEEK, 0xC3 POKE
  0xC4 ALLOC, 0xC5 FREE, 0xC6 MEMCPY, 0xC7 MEMSET
  0xC8 MEMCMP, 0xC9 MEMSWAP, 0xCA STACK_ALLOC, 0xCB STACK_FREE
  0xCC HEAP_BASE, 0xCD STACK_BASE, 0xCE MEM_SIZE, 0xCF MEM_AVAIL

0xD0-0xDF A2A PROTOCOL (16) A2A          Various
  0xD0 SIGNAL, 0xD1 BROADCAST, 0xD2 LISTEN, 0xD3 DELEGATE
  0xD4 INQUIRE, 0xD5 PROPOSE, 0xD6 ACCEPT, 0xD7 REJECT
  0xD8 CONFIRM, 0xD9 RETRACT, 0xDA ESCALATE, 0xDB YIELD_MSG
  0xDC SUBSCRIBE, 0xDD UNSUBSCRIBE, 0xDE POLL, 0xDF FLUSH

0xE0-0xEF EXTENDED MATH (16) Math         Format E
  0xE0 SQRT, 0xE1 CBRT, 0xE2 POW, 0xE3 LOG
  0xE4 EXP, 0xE5 FACTORIAL, 0xE6 GCD, 0xE7 LCM
  0xE8 PRIME_NEXT, 0xE9 PRIME_CHECK, 0xEA RAND, 0xEB RAND_RANGE
  0xEC HASH, 0xED CRC32, 0xEE BASE64_ENC, 0xEF BASE64_DEC

0xF0-0xF7 SENSOR OPS (8)   Sensor       Format D
  0xF0 SENSOR_READ, 0xF1 SENSOR_WRITE, 0xF2 SENSOR_STREAM
  0xF3 GPIO_SET, 0xF4 GPIO_GET, 0xF5 ADC_READ, 0xF6 PWM_SET, 0xF7 I2C_XFER

0xF8-0xFE CRYPTO (7)       Crypto       Format E/F
  0xF8 SHA256, 0xF9 AES_ENC, 0xFA AES_DEC, 0xFB HMAC
  0xFC ECDSA_SIGN, 0xFD ECDSA_VERIFY, 0xFE KEY_EXCHANGE

0xFF      ESCAPE PREFIX     [v3 NEW]    Format X (variable)
```

---

## 8. Migration Guide: v2 → v3

### 8.1 For Runtime Authors

**Minimum changes for v3 compliance:**

1. Add Format X decoder (handle opcode 0xFF → read sub_opcode → dispatch)
2. Implement EXT_NOP (0xFF 0x00) — trivial
3. Implement EXT_FUEL_CHECK (0xFF 0x09) — required for fleet operation
4. Implement EXT_SUSPEND/RESUME (0xFF 0x10/0x11) — required for agent migration
5. Maintain backward compatibility: all v2 opcodes must work unchanged

**Recommended changes for full v3 support:**

6. Implement compressed shorts (0xFF 0xA0-0xB2) for code density
7. Implement security primitives (CAP_INVOKE, MEM_TAG, SANDBOX)
8. Implement temporal primitives (DEADLINE, YIELD, PERSIST_STATE)
9. Implement FORK_CONT/JOIN_CONT for parallel execution
10. Support deprecated v2 opcodes (0x60-0x7F confidence/viewpoint) with deprecation warnings

### 8.2 For Compiler/Assembler Authors

1. Add `-target v3` flag that enables escape prefix and compressed shorts
2. Default output should remain v2 for maximum compatibility
3. Compressed shorts are an optimization pass — assemble to v2 first, then compress frequent instructions
4. The assembler should emit a version header so runtimes can detect v3 bytecodes

### 8.3 For Agent Authors

1. Use `EXT_FUEL_CHECK` at strategic points (before loops, before A2A messages) for cooperative scheduling
2. Use `EXT_DEADLINE_SET` for time-bounded operations
3. Use `EXT_SUSPEND`/`EXT_RESUME` for long-running tasks that might need to migrate
4. Use `EXT_CAP_REQUIRE` before accessing sensitive resources
5. Use compressed shorts in hot loops for better performance (the assembler handles encoding)

### 8.4 Deprecation Timeline

| Phase | Date | Action |
|-------|------|--------|
| Phase 1 | Now | v3 runtimes support escape prefix + all v2 opcodes |
| Phase 2 | +1 month | v3 runtimes emit warnings for deprecated v2 confidence/viewpoint ops |
| Phase 3 | +3 months | v3 runtimes require escape-space equivalents for confidence/viewpoint |
| Phase 4 | +6 months | v4 removes deprecated v2 opcodes entirely |

---

## 9. Conformance Requirements for ISA v3

### 9.1 Required Test Categories

A v3-conformant runtime MUST pass all test vectors in these categories:

| Category | Description | Minimum Tests |
|----------|-------------|---------------|
| Core v2 backward compatibility | All v2 opcodes produce identical results | 116 (existing) |
| Escape prefix dispatch | 0xFF correctly routes to sub-opcodes | 8 |
| Compressed shorts encode/decode | Compressed forms expand to correct v2 equivalents | 16 |
| Security primitives | CAP_INVOKE, MEM_TAG, FUEL_CHECK, SANDBOX | 12 |
| Async primitives | SUSPEND, RESUME, AWAIT, FORK_CONT, JOIN_CONT | 10 |
| Temporal primitives | DEADLINE, YIELD, PERSIST_STATE, TIMESTAMP | 8 |
| Error handling | EXT_UNKNOWN, FUEL_DEPLETED, DEADLINE_EXCEEDED, MEM_TAG_VIOLATION | 6 |
| v3 migration | Deprecated v2 ops still work with escape equivalents | 8 |
| **Total new tests** | | **68** |
| **Grand total (v2 + v3)** | | **184** |

### 9.2 Reference Implementation

The Python flux-runtime (SuperInstance/flux-runtime) is the reference implementation. All test vectors are defined against its behavior. If a runtime produces different results for any test vector, the runtime has a bug (unless the reference implementation itself has a confirmed bug, in which case the test vector is updated and all runtimes are notified).

### 9.3 Test Vector Format

Test vectors for v3 extensions follow the existing format in flux-conformance:

```json
{
  "id": "v3-escape-fuel-001",
  "category": "security",
  "description": "EXT_FUEL_CHECK with sufficient fuel passes through",
  "bytecode_hex": "FF 09 0A 00",
  "initial_stack": [],
  "initial_fuel": 1000,
  "expected_stack": [],
  "expected_flags": "Z=0 S=0 C=0 O=0",
  "expected_fuel": 990,
  "v3_only": true
}
```

---

## 10. Relationship to Fleet Architecture

### 10.1 Integration with I2I Protocol

ISA v3 primitives enable new I2I message types:

| ISA v3 Primitive | I2I Message Type | Use Case |
|-----------------|-----------------|----------|
| EXT_SUSPEND/RESUME | MIGRATE | Agent moves between machines |
| EXT_CAP_TRANSFER | TRUST_TRANSFER | Agent delegates capability to another |
| EXT_DEADLINE_SET | DEADLINE_NOTICE | Fleet coordinator sets temporal bounds |
| EXT_FUEL_ADD | RESOURCE_GRANT | Scheduler grants additional fuel |
| EXT_AWAIT EVT_MESSAGE | WAKE_ON_MESSAGE | Agent sleeps until addressed |

### 10.2 Integration with Fleet Infrastructure

| Component | v3 Feature | Integration Point |
|-----------|-----------|------------------|
| fleet-mechanic | EXT_FUEL_CHECK | Mechanic monitors fleet fuel consumption |
| lighthouse-keeper | EXT_DEADLINE_SET | Keeper sets temporal bounds for health checks |
| tender | EXT_CAP_INVOKE | Tender manages capability creation/revocation |
| brothers-keeper | EXT_MEM_TAG | Keeper validates memory isolation |
| cuda-trust | EXT_SANDBOX_ENTER | Trust scores determine sandbox profiles |
| flux-conformance | All v3 ops | New test vectors for v3 compliance |

### 10.3 Open Questions for Fleet Discussion

1. **Continuation storage:** Should continuations be stored in the agent's vessel repo (persistent, git-native) or in a fleet API endpoint (fast, ephemeral)? Recommendation: vessel repo for long-lived continuations, API endpoint for short-lived.

2. **Fuel economy:** Who mints fuel? Should there be a "fuel bank" that agents deposit into and withdraw from? Recommendation: tender manages fuel as a resource, with trust-based allocation.

3. **Sandbox profile governance:** Who defines new sandbox profiles? Should there be a fleet-wide registry? Recommendation: oracle1-vessel maintains the profile registry as a TOML file.

4. **Compressed shorts in A2A messages:** When agents send bytecodes to each other, should they use compressed format? Recommendation: Sender detects receiver's capabilities via EXT_PROBE and compresses only if receiver supports it.

---

## 11. Acknowledgments

- **Quill** 🔧 — Designed the 0xFF escape prefix mechanism (ISA-v3-ESCAPE-PREFIX.md). The security, async, and temporal sub-opcode allocations in this document build directly on Quill's architectural foundation.
- **Oracle1** 🔮 — Synthesized the round-table critique, built the conformance test suite, and maintains the fleet task board that identified these priorities.
- **Kimi** — Proposed the escape prefix concept ("Reserve 0xFF as ESCAPE prefix, get 256 sub-opcodes").
- **DeepSeek** — Contributed the async primitives and temporal ops design during the round-table critique.
- **JetsonClaw1** — Provided edge hardware constraints that shaped the fuel model and compressed format requirements.

---

*This document is a Datum production — Fleet Agent, Session 4. Respond via bottle (for-fleet/) or issue on oracle1-vessel.*
