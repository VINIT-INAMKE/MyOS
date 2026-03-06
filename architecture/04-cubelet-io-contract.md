# MYOS Cubelet I/O Contract - Data Pipeline & Control Messages

## Version: 1.0 | Status: LOCKED

---

## 1. Overview

Cubelets communicate through **two distinct channels**:

1. **Data Pipeline** - typed, directed acyclic graph (DAG) carrying the actual work product from cubelet to cubelet
2. **Control Messages** - flexible messages for coordination, status reporting, and queries between cubelets and their Pod Orchestrator

These channels are architecturally separate. Data flows through the pipeline deterministically. Control messages flow through a side channel for coordination and management.

```
DATA PIPELINE (typed DAG):
  sensor → filter → model → actuator
  Each arrow is typed. Output type must match input type.

CONTROL MESSAGES (flexible):
  cubelet ──status──→ Pod Orchestrator
  Pod Orch ──config──→ cubelet
  cubelet ──alert──→ Pod Orchestrator
  Pod Orch ──query──→ System Orchestrator
```

---

## 2. Data Pipeline

### 2.1 Pipeline as a Directed Acyclic Graph (DAG)

The data pipeline is a DAG where:

- **Nodes** = cubelets (each performs one transformation)
- **Edges** = typed data flows (output of one cubelet feeds input of the next)

```
Example: Image Classification Pipeline

  [Image Resizer] ──ResizedImage──→ [Classifier] ──ClassificationResult──→ [Response Writer]
       ↑                                                                         ↓
    RawImage                                                                 UserResponse
```

```
Example: Robot Arm Control Pipeline

  [Camera] ──Image──→ [Object Detector] ──ObjectPosition──→ [Path Planner] ──MotorCommands──→ [Motor Controller]
```

```
Example: Fan-out (parallel branches)

  [Sensor Reader] ──RawSignal──→ [Noise Filter] ──CleanSignal──┬──→ [Anomaly Detector] ──Alert──→ [Alert Handler]
                                                                │
                                                                └──→ [Data Logger] ──LogEntry──→ [Storage]
```

### 2.2 Type System

Every cubelet declares its input and output types at registration (in its cubelet manifest).

```
cubelet_io_declaration {
    cubelet_id:     UUID
    inputs: [{
        port_name:  String          // e.g., "image_in"
        type_id:    String          // e.g., "image.rgb.640x480"
        schema:     SchemaDefinition // full type definition
        required:   bool            // must be connected
    }]
    outputs: [{
        port_name:  String          // e.g., "resized_out"
        type_id:    String          // e.g., "image.rgb.224x224"
        schema:     SchemaDefinition
    }]
}
```

**Type compatibility rule:**

```
An edge from Cubelet A's output to Cubelet B's input is valid if and only if:
    A.output.type_id == B.input.type_id
    OR
    A.output.type_id is a subtype of B.input.type_id (if subtyping is defined)

If types don't match → pipeline assembly error → pod cannot start.
```

### 2.3 Multi-Port Cubelets

Some cubelets have multiple input or output ports.

```
Example: Data Merger cubelet
  inputs:
    - port "data_a": FinancialReport
    - port "data_b": FinancialReport
  outputs:
    - port "merged": MergedReport

Example: Classifier cubelet
  inputs:
    - port "image": ResizedImage
  outputs:
    - port "label": ClassificationLabel
    - port "confidence": ConfidenceScore
    - port "heatmap": AttentionHeatmap
```

Each port is independently typed and independently wired in the DAG.

### 2.4 DAG Construction

The **Pod Orchestrator** (an LLM) builds the DAG dynamically for each task.

```
DAG construction process:
  1. Pod Orch receives: task descriptor + assigned cubelets list
  2. Pod Orch examines each cubelet's input/output declarations
  3. Pod Orch reasons about the optimal pipeline structure for the task
  4. Pod Orch constructs the DAG (which cubelet connects to which)
  5. System validates type compatibility on every edge
  6. If all edges are type-compatible → DAG is finalized
  7. If any edge has a type mismatch → assembly error → report to System Orch
```

Since the Pod Orchestrator is an LLM, DAG construction involves reasoning about:

- The best ordering of cubelets for the task
- Whether fan-out (parallel branches) is beneficial
- Whether any cubelets can be skipped for this specific task
- How to handle multiple output ports (which downstream cubelets need which outputs)

### 2.5 DAG Mutability - Fixed with Optional Bypass

Once the pod starts execution, the core DAG structure is **fixed**. However, the Pod Orchestrator can apply **limited modifications**:

```
ALLOWED mid-execution:
  - BYPASS a node: skip a cubelet in the pipeline
    (e.g., skip noise filter if signal is already clean)
  - INSERT a node: add an additional cubelet at a point in the pipeline
    (e.g., insert an extra validation step if confidence is low)
  - These modifications require:
    a. Type compatibility is maintained after the change
    b. Pod Orchestrator has sufficient authority
    c. The modification is logged as a pipeline_modification event

NOT ALLOWED mid-execution:
  - Complete rewiring of the DAG (requires pod dissolution and reassembly)
  - Removing a cubelet that has already produced output used downstream
  - Changing the fundamental flow direction
```

**Exception:** During the tiered failure response (see Pod Orchestrator document), Tier 3 (Adaptive Restructure) allows more aggressive DAG changes. This is a controlled exception triggered by failure recovery, not normal operation.

### 2.6 Data Flow Execution

```
pipeline_execution:
  1. Initial input enters the first cubelet(s) in the DAG
  2. Cubelet processes input → produces output
  3. Output is:
     a. Type-checked against the expected output type
     b. Hashed for logging (off-chain)
     c. Passed to the next cubelet(s) in the DAG
  4. Repeat until the final cubelet(s) produce the pipeline's output
  5. Final output is delivered to the Pod Orchestrator for synthesis/delivery

Error handling:
  - If a cubelet fails to produce output within its time budget → timeout error
  - If a cubelet produces output that fails type checking → type error
  - Both errors trigger the Pod Orchestrator's tiered failure response
```

### 2.7 Fan-Out and Fan-In

**Fan-out:** One cubelet's output feeds multiple downstream cubelets in parallel.

```
[Cubelet A] ──→ [Cubelet B]
             └──→ [Cubelet C]
             └──→ [Cubelet D]

All three receive the same output from A.
B, C, D execute in parallel.
```

**Fan-in:** Multiple cubelets' outputs feed into one downstream cubelet (requires multi-port input).

```
[Cubelet B] ──→┐
[Cubelet C] ──→├──→ [Merger Cubelet]
[Cubelet D] ──→┘

Merger has three input ports, one for each upstream cubelet.
Merger waits for ALL inputs before processing.
```

**Fan-in timeout:** If one branch takes too long, the merger can proceed with partial inputs (if configured) or timeout (triggering failure response).

---

## 3. Control Messages

### 3.1 Purpose

Control messages handle everything that is NOT the work product:

- Health status reports
- Alerts and warnings
- Configuration changes
- Cross-pod query requests
- Escalation triggers
- Performance metrics

### 3.2 Message Structure

```
control_message {
    message_id:     UUID
    timestamp:      Timestamp
    sender:         EntityId        // cubelet or Pod Orchestrator
    recipient:      EntityId        // Pod Orchestrator, System Orchestrator, or cubelet
    message_type:   MessageType
    priority:       urgent | normal | low
    payload:        MessagePayload  // flexible, type depends on message_type
    authority_context: AuthorityContext  // sender's AV snapshot
}
```

### 3.3 Message Types

| Message Type     | Direction              | Purpose                                                                    |
| ---------------- | ---------------------- | -------------------------------------------------------------------------- |
| `status_report`  | Cubelet → Pod Orch     | Regular health/progress update                                             |
| `alert`          | Cubelet → Pod Orch     | Something unexpected detected (low confidence, anomaly, resource pressure) |
| `error`          | Cubelet → Pod Orch     | Cubelet encountered a failure                                              |
| `config_update`  | Pod Orch → Cubelet     | Change cubelet parameters (e.g., confidence threshold, processing mode)    |
| `bypass_command` | Pod Orch → Cubelet     | Instruct cubelet to pass through data without processing                   |
| `query_request`  | Cubelet → Pod Orch     | Request data/action from outside the pod                                   |
| `query_response` | Pod Orch → Cubelet     | Response to a query_request                                                |
| `escalation`     | Pod Orch → System Orch | Escalate an issue beyond pod scope                                         |
| `pod_status`     | Pod Orch → System Orch | Pod-level health report                                                    |
| `directive`      | System Orch → Pod Orch | System-level instruction to the pod                                        |

### 3.4 Control Message Flow Rules

```
WITHIN POD:
  Cubelet → Pod Orchestrator:       ALWAYS allowed
  Pod Orchestrator → Cubelet:       ALWAYS allowed
  Cubelet → Cubelet:                NOT allowed (must go through Pod Orch)

OUTSIDE POD:
  Pod Orchestrator → System Orch:   ALWAYS allowed
  System Orch → Pod Orchestrator:   ALWAYS allowed
  Cubelet → System Orch:            NOT allowed (must go through Pod Orch)
  Cubelet → External:               NOT allowed (must go through Pod Orch → System Orch)
```

### 3.5 Authority-Gated Control Messages

Certain control messages require authority checks:

```
config_update:     Pod Orch must have sufficient policy AV
bypass_command:    Pod Orch must have sufficient execution AV
escalation:        Checked by Authority Engine before System Orch receives it
directive:         System Orch must have sufficient authority (always does, floor = 700)
```

Status reports, alerts, and errors are NOT authority-gated - any cubelet can always report its status.

---

## 4. Pipeline Logging - Two-Tier Model

### 4.1 Overview

All pipeline activity is logged, but stored in two tiers based on importance:

```
ON-CHAIN (blockchain/ledger):
  Decision logs only - authority decisions, conflict resolutions,
  human overrides, safety events.
  Tamper-proof, expensive, permanent.

OFF-CHAIN (database):
  Everything else - pipeline edge hashes, intermediate results,
  telemetry, performance data, control messages.
  Hashed and auditable, cheaper storage.
```

### 4.2 What Goes On-Chain

```
on_chain_events:
  - Authority decisions (authorize → allow/deny/escalate)
  - Conflict resolution outcomes (all levels)
  - Human override events
  - Safety-critical events (emergency stops, safety violations)
  - Pod creation and dissolution decisions
  - Cubelet quarantine events
  - Pipeline restructure decisions (Tier 3 failure response)
```

Each on-chain event includes:

```
on_chain_entry {
    event_id:       UUID
    event_type:     String
    timestamp:      Timestamp
    participants:   [EntityId]
    decision:       String
    authority_snapshot: AuthorityVector[]
    input_hash:     Hash
    output_hash:    Hash
    signature:      Signature (device key)
}
```

### 4.3 What Goes Off-Chain

```
off_chain_events:
  - Every pipeline edge data hash (intermediate results)
  - Cubelet execution telemetry (timing, resource usage)
  - Control messages (status reports, alerts, config changes)
  - DAG construction logs
  - Pod Orchestrator reasoning traces
  - Performance metrics
  - Query/response logs for AI interactions
```

Each off-chain event includes:

```
off_chain_entry {
    event_id:       UUID
    event_type:     String
    timestamp:      Timestamp
    source:         EntityId
    data_hash:      Hash            // hash of the actual data
    data_payload:   Bytes | null    // actual data (if retention allows)
    pod_id:         PodId
    pipeline_edge:  (source_cubelet, target_cubelet) | null
}
```

### 4.4 Decision Context Logging

Pipeline logging captures _what_ happened. Decision context logging captures _why_ it happened. Every Pod Orchestrator decision that alters the pipeline is logged with its full reasoning chain.

```
decision_context_entry {
    decision_id:        UUID
    timestamp:          Timestamp
    pod_id:             PodId
    orchestrator_id:    EntityId

    // The decision itself
    decision_type:      dag_construction | cubelet_bypass | pipeline_restructure |
                        sub_pod_spawn | failure_response | query_routing
    decision:           String          // what was decided
    alternatives:       [String]        // what other options were considered
    rationale:          String          // why this option was chosen

    // Context that informed the decision
    input_context: {
        triggering_event:   String      // what caused this decision point
        relevant_kb_entries: [UUID]     // KB entries consulted
        cubelet_states:     [{EntityId, AV_snapshot, status}]
        pipeline_state:     DAG_snapshot
        prior_decisions:    [UUID]      // links to earlier decision_context_entries
    }

    // Outcome tracking
    expected_outcome:   String          // what the orchestrator expects to happen
    actual_outcome:     String | null   // filled after execution (retrospective)
    outcome_matched:    bool | null     // did reality match expectation?
}
```

Decision context entries are stored **off-chain** in the Pod KB and linked to corresponding on-chain events (authority decisions, restructures). This creates a parallel reasoning DAG alongside the data DAG:

```
DATA DAG:       [A] ──data──→ [B] ──data──→ [C]
                                    ↑
DECISION DAG:   [build_dag] ──informed──→ [bypass_B] ──informed──→ [reroute_to_C]
                     │                         │                        │
                   rationale               rationale                rationale

The decision DAG preserves WHY the data DAG looks the way it does.
When the orchestrator needs to revisit a decision, it reloads
the decision context - not reconstruct reasoning from degraded memory.
```

Decision context entries are:

- Written to Pod KB immediately when decisions are made
- Linked via `prior_decisions` to form the reasoning chain
- Promoted to Session Memory KB for session pods (persists across queries)
- Queryable by the orchestrator to reload reasoning context mid-pipeline

```

### 4.4 Tamper Evidence

Off-chain data is NOT on a blockchain but IS tamper-evident:

```

Each off-chain entry is hashed.
Hashes are chained: entry_hash(N) = hash(entry(N) + entry_hash(N-1))
Periodic merkle root of off-chain entries is anchored on-chain.

This means:

- Any tampering with off-chain data breaks the hash chain
- The on-chain merkle root can verify the integrity of the off-chain data
- Full auditability without putting everything on-chain

```

---

## 5. Data Serialization

### 5.1 Wire Format

Data flowing through pipeline edges must be serialized. The system uses a standard serialization format.

```

Supported formats (deployment-configurable):

- Protocol Buffers (default - compact, typed, fast)
- MessagePack (for lightweight edge deployments)
- CBOR (for constrained devices)
- JSON (for debugging/development only - not recommended for production)

```

### 5.2 Schema Registry

All type schemas are registered at system boot in a **schema registry**.

```

schema_registry {
schemas: [{
type_id: String // e.g., "image.rgb.224x224"
version: int
definition: SchemaDefinition // protobuf/msgpack schema
hash: Hash // hash of the schema definition
}]
}

```

Schema registry is:
- Immutable at runtime
- Loaded at boot
- Signed and verified
- Shared across all cubelets and orchestrators

### 5.3 Schema Versioning

If a cubelet is updated with a new schema version:

```

type_id: "image.rgb.224x224"
version: 2

Backward compatibility:
If version 2 is a superset of version 1 → compatible (edge is valid)
If version 2 breaks version 1 → incompatible (edge is invalid, assembly error)

```

Schema versioning rules are enforced at DAG construction time.

---

## 6. Invariants (Must Always Hold)

```

INV-1: Every pipeline edge is type-checked at DAG construction time
INV-2: Type mismatch → assembly error → pod cannot start
INV-3: Data pipeline is a DAG (no cycles)
INV-4: Core DAG structure is fixed after pod start (bypass/insert are limited exceptions)
INV-5: Cubelet-to-cubelet direct communication is NOT allowed (must go through Pod Orch via control messages)
INV-6: Every pipeline edge data transfer is hashed and logged (off-chain)
INV-7: Every authority decision is logged (on-chain)
INV-8: Off-chain data integrity is verifiable via on-chain merkle roots
INV-9: Control messages from cubelets can only go to their Pod Orchestrator (never directly to System Orch or external)
INV-10: Status reports and error messages are never authority-gated (cubelets can always report)
INV-11: Fan-in nodes wait for all required inputs (or timeout)
INV-12: Schema registry is immutable at runtime
INV-13: Cross-language cubelets (Python ML code) use PyO3 FFI with Rust ownership enforcement at the boundary

```

---

## 7. Cross-Language Cubelets (PyO3 FFI)

### 7.1 Rust-Python Bridge for ML Cubelets

Many ML/data pipelines use Python libraries (PyTorch, scikit-learn, pandas). MYOS supports **cross-language cubelets** where the cubelet shell is Rust (for safety, memory guarantees, typed I/O) and the inner computation calls Python via **PyO3**.

```

CROSS-LANGUAGE CUBELET ARCHITECTURE:

┌─────────────────────────────────────┐
│ Rust Cubelet Shell │ ← Typed I/O, authority context, lifecycle
│ ├── Input deserialization (Rust) │
│ ├── PyO3 call into Python │ ← FFI boundary
│ │ └── Python ML code │ ← Model inference, data transforms
│ ├── Output serialization (Rust) │
│ └── Authority/logging (Rust) │
└─────────────────────────────────────┘

The Rust shell enforces: - Typed input/output (schema registry) - Memory limits (ownership model prevents leaks across FFI) - Time budgets (Rust controls the execution timeout) - Authority context (Rust handles IPC with Pod Orch)

Python handles: - ML model inference (PyTorch, ONNX, sklearn) - Data transformation (pandas, numpy) - Domain-specific computation

PyO3 provides: - Zero-copy data passing where possible (numpy arrays → Rust slices) - Rust ownership model enforced at the boundary - Python GIL management handled by PyO3 - Rust structs exposed as Python classes (#[pyclass])

```

### 7.2 When to Use Cross-Language vs Pure Wasm

```

PURE WASM (Rust → wasm32-wasi):
Use for: Math functions, signal processing, crypto, ONNX inference
Isolation: Wasm + Unikernel (double isolation)
Deterministic: Yes (bit-for-bit reproducible)

CROSS-LANGUAGE (Rust + PyO3 → Python):
Use for: PyTorch models, sklearn pipelines, pandas transforms,
any ML code that depends on the Python ecosystem
Isolation: Linux container (Python runtime needs filesystem)
Deterministic: Best-effort (Python floating point, GPU non-determinism)
Note: These run in containers like LLM cubelets, NOT in unikernels

````

### 7.3 Cubelet Proc Macro (Boilerplate Reduction)

Cubelet authors can use a `#[derive(Cubelet)]` proc macro to auto-generate boilerplate:

```rust
#[derive(Cubelet)]
#[cubelet(
    name = "image_classifier",
    input(port = "image_in", type_id = "image.rgb.224x224"),
    output(port = "label_out", type_id = "classification.label"),
    output(port = "confidence_out", type_id = "classification.confidence")
)]
struct ImageClassifier {
    model: OnnxModel,
}

// The derive macro generates:
//   - cubelet_io_declaration (typed ports for schema registry)
//   - IPC message handlers (status reports, config updates)
//   - Authority context wrappers
//   - Serialization/deserialization for input/output
//   - Health check endpoint
//   - Cubelet lifecycle hooks (init, execute, shutdown)
````

This reduces cubelet authoring from ~200 lines of boilerplate to a struct definition + derive attribute.

---

## 8. Interaction with Other Documents

- **Authority Model (01-authority-model.md):** Authority-gated control messages use the MBAC system. AV from the flexible dimension registry determines which control actions are allowed.
- **Conflict Resolution (02-conflict-resolution.md):** Conflicts detected during pipeline execution (disagreement between cubelets) are resolved through the escalation chain.
- **Pod Assembly (03-pod-assembly.md):** DAG construction happens after pod assembly. Cubelet I/O declarations (from cubelet manifests) are used for capability matching during assembly AND for type checking during DAG construction.
- **Pod Orchestrator (05-pod-orchestrator.md):** Pod Orch builds the DAG, manages control messages, handles pipeline failures, and synthesizes results.
- **Knowledge Base (12-knowledge-base.md):** Cubelet outputs flowing through the data pipeline can be logged as KB entries when they meet confidence thresholds. Pipeline edge data (input/output hashes, intermediate results) can be referenced as evidence for KB entries, linking verified knowledge back to the deterministic pipeline that produced it.
