# MYOS Verification & Audit - On-Chain Decisions, Off-Chain Data & Consensus Architecture

## Version: 2.0 | Status: LOCKED

---

## 1. Overview

Every action, decision, and data transfer in MYOS is logged. The verification and audit system ensures that the entire history of the system is **tamper-evident, publicly readable, and independently verifiable**.

The system uses a **two-tier storage model**:

```
ON-CHAIN (Ouroboros-based lightweight chain):
  Authority decisions, conflict resolutions, human overrides,
  safety events, pod lifecycle decisions.
  Tamper-proof, publicly readable, permanent.

OFF-CHAIN (event log + content-addressed + time-series DB):
  Pipeline edge hashes, intermediate results, telemetry,
  control messages, performance metrics.
  Hashed, auditable, tiered retention (hot/warm/cold).
```

**All logs are publicly readable.** On-chain data is readable by anyone (general public, auditors, regulators, other systems). Off-chain data is also publicly accessible - if it's logged, it's transparent. This is fundamental: hiding audit data defeats the purpose of a verifiable system.

**STK invariant satisfaction records** are included in verification logs for every pod execution. The 150 formal invariants across 7 families are evaluated by the Authority Engine, and PROVES edges in the CIG are verified and logged on-chain. Verification data is **routed through STF fabric threads** (e.g., EthosLedger, KarbonLedger) so that domain-specific audit records flow to the appropriate off-chain stores based on their fabric thread classification.

---

## 2. On-Chain Layer - Lightweight Ouroboros Chain

### 2.1 Architecture

MYOS runs its own **lightweight blockchain** based on the **Ouroboros** proof-of-stake consensus protocol. This is NOT Cardano mainnet - it is a purpose-built chain optimized for decision logging.

```
MYOS Chain:
  - Consensus: Ouroboros (Haskell core)
  - Purpose: Decision logs only (no smart contracts, no tokens at launch)
  - Validators: MYOS devices themselves
  - Block time: Configurable (sub-second for real-time systems, seconds for non-critical)
  - External anchoring: Periodic merkle root to Cardano mainnet
```

### 2.2 Why Ouroboros

| Property                              | Benefit for MYOS                                                  |
| ------------------------------------- | ----------------------------------------------------------------- |
| Provably secure (formal proofs exist) | Aligns with MYOS's mathematical guarantee philosophy              |
| Haskell implementation                | Shares language/ecosystem with Authority Engine                   |
| Deterministic finality                | Know exactly when a decision is permanently committed             |
| PoS (no mining)                       | Suitable for edge/IoT devices (no wasted computation)             |
| Cardano ecosystem                     | Direct plugin capability to Cardano and any Ouroboros-based chain |

### 2.3 Modular Chain Architecture

The chain is built as three independent modules with clean interfaces, designed for evolution toward Kusari and plugin capability to other chains.

```
┌──────────────────────────────────────────────────────┐
│  CONSENSUS ADAPTER INTERFACE                         │
│                                                      │
│  Every consensus implementation satisfies:           │
│    validate_block(block) → bool                      │
│    propose_block(transactions) → block               │
│    finality_check(block) → finalized | pending       │
│    sync_state(peers) → state                         │
│    get_validators() → validator_set                  │
│                                                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│  ┌────────────────────┐                              │
│  │  CONSENSUS MODULE  │  Ouroboros core [Haskell]    │
│  │                    │  Additional layers can be    │
│  │  Ouroboros (base)  │  added: BFT, PoA, custom     │
│  │  + pluggable       │  Each implements the same    │
│  │    layers          │  adapter interface            │
│  └────────────────────┘                              │
│                                                      │
│  ┌────────────────────┐                              │
│  │  STORAGE MODULE    │  Decision logs [Rust]        │
│  │                    │  Merkle trees                 │
│  │  Block storage     │  State management             │
│  │  Index             │  Query interface              │
│  └────────────────────┘                              │
│                                                      │
│  ┌────────────────────┐                              │
│  │  NETWORK MODULE    │  P2P networking [Rust]       │
│  │                    │  Node discovery               │
│  │  Peer management   │  Block propagation            │
│  │  Sync protocol     │  External anchoring           │
│  └────────────────────┘                              │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Each module can be replaced independently.** This is the key design principle for Kusari evolution.

### 2.4 Consensus Adapter Pattern - Plugin Capability

Because the consensus module implements a standard interface, MYOS can plug into any chain:

```
PLUGIN TARGETS:
  Cardano:         Ouroboros adapter (native - same consensus family)
  Kusari (future): Swap consensus module when Kusari's hybrid PoS+BFT is ready
  Cosmos:          Write a Tendermint/CometBFT adapter
  Custom chains:   Write a new adapter implementing the interface

HOW PLUGGING IN WORKS:
  1. MYOS chain runs independently with Ouroboros consensus
  2. When plugging into an external chain:
     - External chain adapter is loaded
     - MYOS decision logs are submitted as transactions to the external chain
     - OR: MYOS chain runs as a sidechain/KVM on the external chain
  3. Application code (logging, verification) doesn't change
     Only the consensus adapter changes
```

### 2.5 Kusari Evolution Path

The MYOS chain is designed to evolve into (or integrate with) Kusari:

```
PHASE 1 (NOW - MYOS v1):
  Standalone Ouroboros chain on MYOS devices
  Periodic merkle root anchor to Cardano mainnet
  Simple PoS consensus among MYOS nodes

PHASE 2 (KUSARI INTEGRATION):
  Option A: MYOS chain becomes a Kusari KVM
    - MYOS authority logic becomes a DSL on Kusari
    - Kusari provides consensus, MYOS provides the domain logic

  Option B: MYOS chain merges its consensus into Kusari
    - Ouroboros layer becomes part of Kusari's hybrid PoS+BFT
    - Additional consensus layers (from Kusari) are added via adapters

  Option C: MYOS chain plugs into Kusari as a sidechain
    - MYOS runs independently but anchors to Kusari instead of Cardano
    - Cross-chain bridge for verification

DESIGN PRINCIPLE:
  Build modularly now so that ANY of these paths is possible
  without rewriting the MYOS chain.
```

### 2.6 Node Types

MYOS devices participate in the chain at different levels based on hardware capability.

```
FULL VALIDATOR NODES:
  Hardware:     Powerful edge devices / servers
  Role:         Produce blocks, validate transactions, full consensus participation
  Storage:      Full chain history
  Requirement:  Sufficient compute for Ouroboros slot leader election

LIGHT VALIDATION NODES:
  Hardware:     Constrained edge devices, IoT
  Role:         Verify merkle proofs, validate individual decisions
  Storage:      Block headers only (not full blocks)
  Capability:   Can verify that a specific decision is in the chain
                without downloading the entire chain
  Cannot:       Produce blocks or participate in consensus

NODE ROLE ASSIGNMENT:
  Determined by device hardware profile at boot:
    cpu_cores ≥ threshold AND memory ≥ threshold → full validator
    otherwise → light validation node

  Role is configured at system boot, not dynamic at runtime.
```

### 2.7 What Goes On-Chain

Only **decisions** go on-chain. Data stays off-chain.

```
ON-CHAIN EVENTS:

Authority decisions:
  - authorize(entity, action) → allow | deny | escalate
  - authority_update(entity, event) → new AV state
  - Every call to the Authority Engine that results in a state change

Conflict resolution outcomes:
  - Level 1 resolution (AV comparison result)
  - Level 2 vote results (weighted tally, confidence, decision)
  - Level 3 System Orch arbitration result
  - Level 4 human decision

Human override events:
  - Human vote
  - Human veto (safety-critical)
  - Human delegate-back

Safety events:
  - Safety emergency triggers
  - Safety watchdog activations
  - Safety-critical veto exercises

Pod lifecycle decisions:
  - Pod creation (task_id, cubelet roster, Pod Orch assignment)
  - Pod dissolution (outcome, AV updates triggered)
  - Cubelet quarantine events
  - Pipeline restructure decisions (Tier 3 failure response)

System configuration events:
  - Boot verification (secure boot chain hashes)
  - Configuration load (config hash, signature verification)
  - Dimension registry changes (only via reboot)
```

### 2.8 On-Chain Entry Structure

```
on_chain_entry {
    entry_id:           UUID
    block_number:       uint64
    block_hash:         Hash
    entry_type:         EventType
    timestamp:          Timestamp

    // Who
    participants:       [EntityId]
    authority_snapshots: [AuthorityVector]   // AV of all participants at decision time

    // What
    decision_type:      String
    decision_input_hash: Hash               // hash of the input that led to this decision
    decision_output:    DecisionResult
    decision_reasoning: String | null       // for LLM orchestrator decisions

    // Verification
    device_id:          DeviceId
    device_signature:   Signature           // signed by the device's hardware key
    merkle_proof:       MerkleProof         // proof of inclusion in the block's merkle tree
}
```

### 2.9 Cardano Mainnet Anchoring

Periodic merkle root anchoring to Cardano for external verifiability.

```
ANCHORING PROCESS:

  1. Every N blocks (configurable, e.g., every 100 blocks):
     - Compute merkle root of the last N blocks
     - Create a Cardano transaction containing the merkle root
     - Submit to Cardano mainnet

  2. The Cardano transaction contains:
     anchor_data {
         myos_chain_id:      String          // identifies this MYOS deployment
         block_range:        (start, end)    // which MYOS blocks are covered
         merkle_root:        Hash            // root of the merkle tree
         timestamp:          Timestamp
         validator_signatures: [Signature]   // signed by MYOS validators
     }

  3. Anyone can verify:
     - Fetch the Cardano transaction
     - Fetch the MYOS blocks in the range
     - Recompute the merkle root
     - Compare with the anchored root
     - If they match → MYOS chain data is verified as untampered

ANCHORING FREQUENCY:
  Configurable per deployment:
    Real-time critical:    every 10 blocks (~minutes)
    Standard:              every 100 blocks (~hours)
    Low-frequency:         every 1000 blocks (~days)

  More frequent = more Cardano transaction costs, stronger guarantees
  Less frequent = cheaper, but longer window of unanchored data
```

---

## 3. Off-Chain Layer - Event Log + Content-Addressed + Time-Series

### 3.1 Hybrid Storage Model

The off-chain database combines two storage paradigms:

```
EVENT LOG (append-only, content-addressed):
  Every event is appended to an immutable log.
  Each event's payload is stored by its content hash (like Git objects).

  Benefits:
    - Temporal ordering: events are ordered by time
    - Content dedup: identical payloads stored once
    - Tamper evidence: changing any event breaks the hash chain
    - Replay capability: reconstruct any state by replaying events

TIME-SERIES INDEX:
  On top of the event log, a time-series index enables efficient queries:
    - "Show me all events for Pod X between 10:00 and 10:05"
    - "Show me telemetry for Cubelet C42 over the last hour"
    - "Show me all pipeline edge transfers for Task T1"

  The index is a secondary structure - the event log is the source of truth.
  If the index is corrupted, it can be rebuilt from the event log.
```

### 3.2 What Goes Off-Chain

```
OFF-CHAIN EVENTS:

Pipeline data flow:
  - Every edge transfer hash (cubelet A output → cubelet B input)
  - Input/output data (actual payload, not just hashes)
  - Type validation results
  - Fan-in/fan-out synchronization events

Cubelet telemetry:
  - Execution time per cubelet per task
  - Resource usage (memory, CPU)
  - Output confidence scores
  - Error counts and types

Control messages:
  - All control messages between cubelets and Pod Orchestrator
  - Status reports, alerts, configuration changes
  - Query requests and responses

Pod Orchestrator activity:
  - DAG construction logs (what was wired and why)
  - Synthesis reasoning traces
  - Query routing decisions (capability check results)
  - Bypass/insert decisions

Performance metrics:
  - Pod assembly duration
  - Task completion time
  - Cubelet selection scores
  - Compatibility score updates

Session data (AI interaction):
  - Conversation history
  - Query/response pairs
  - Sub-pod spawn/dissolution events
  - Context state snapshots
```

### 3.3 Off-Chain Entry Structure

```
off_chain_entry {
    entry_id:       UUID
    sequence_number: uint64              // monotonic, for ordering
    timestamp:      Timestamp
    event_type:     String

    // Source
    source_entity:  EntityId
    pod_id:         PodId | null
    task_id:        TaskId | null

    // Content (content-addressed)
    content_hash:   Hash                 // hash of the payload
    content:        Bytes                // actual data (stored by content_hash)

    // Pipeline context (if applicable)
    pipeline_edge:  (source_cubelet, target_cubelet) | null
    input_hash:     Hash | null
    output_hash:    Hash | null

    // Chain link
    previous_hash:  Hash                 // hash of the previous entry (hash chain)
}
```

### 3.4 Tamper Evidence for Off-Chain Data

Off-chain data is not on the blockchain, but it IS tamper-evident:

```
HASH CHAIN:
  Each off-chain entry includes the hash of the previous entry.
  Changing any entry breaks the chain from that point forward.

  entry_hash(N) = hash(entry(N).content + entry(N).metadata + entry_hash(N-1))

MERKLE ROOTS:
  Periodically, a merkle root of recent off-chain entries is computed
  and submitted ON-CHAIN as a "data integrity checkpoint."

  data_integrity_checkpoint {
      checkpoint_id:     UUID
      off_chain_range:   (sequence_start, sequence_end)
      merkle_root:       Hash
      entry_count:       uint64
      timestamp:         Timestamp
  }

  This is an ON-CHAIN event - so anyone can:
    1. Fetch the off-chain entries in the range
    2. Recompute the merkle root
    3. Compare with the on-chain checkpoint
    4. Verify integrity

CHECKPOINT FREQUENCY:
  Configurable (e.g., every 1000 off-chain entries, or every 5 minutes)
```

### 3.5 Tiered Retention (Hot / Warm / Cold)

Off-chain data is retained using a tiered storage model:

```
HOT TIER:
  Storage:    Fast SSD / in-memory
  Data:       Last 24 hours (configurable)
  Access:     Sub-millisecond queries
  Purpose:    Real-time monitoring, active debugging, live dashboards

WARM TIER:
  Storage:    Standard SSD / HDD
  Data:       Last 30 days (configurable)
  Access:     Millisecond queries
  Purpose:    Post-incident analysis, performance trending, audit queries
  Format:     Compressed, indexed

COLD TIER:
  Storage:    Object storage (S3, MinIO) / tape / archival
  Data:       Everything older than 30 days
  Access:     Seconds to minutes (retrieval required)
  Purpose:    Regulatory compliance, long-term audit, legal discovery
  Format:     Compressed, content-addressed (deduped)
  Retention:  Configurable per deployment:
                Regulated industries: 7 years minimum
                Standard: 1 year
                Minimal: 90 days

TIER MIGRATION:
  Automatic background process moves data between tiers.
  Migration is transparent - queries search across all tiers.
  Content hashes are preserved across tiers (data integrity maintained).
  Merkle roots reference data regardless of which tier it's in.
```

### 3.6 Retention Configuration

```
retention_config {
    hot_tier {
        duration:       "24h"
        storage_type:   "ssd"
        max_size:       "10GB"
    }
    warm_tier {
        duration:       "30d"
        storage_type:   "hdd"
        max_size:       "500GB"
        compression:    "zstd"
    }
    cold_tier {
        duration:       "7y"            // or "forever" for regulated
        storage_type:   "object_store"
        compression:    "zstd"
        dedup:          true            // content-addressed dedup
    }

    // Override per data type
    overrides {
        "authority_decision":   { min_retention: "forever" }  // never delete
        "safety_event":         { min_retention: "forever" }
        "telemetry":            { min_retention: "90d" }
        "session_conversation": { min_retention: "30d" }
    }
}
```

### 3.7 Statistical Monitoring Thresholds

The verification system continuously monitors system health metrics against formal thresholds:

| Metric | Threshold | Action on Violation |
| ------ | --------- | ------------------- |
| `proof_success_rate` | >= 98% | Halt Rubik's Move rollouts if below |
| `telemetry_coverage` | >= 90% | Flag cubelets without telemetry for review |
| `interop_success_rate` | >= 95% | Suspend cross-device queries on failing fabric threads |
| `payout_variance_std_dev` | < 0.1 | Audit economic model for fairness drift |
| `zk_verifier_failure_rate` | < 2% over 24h | Halt all rollouts, trigger STK alert |

These thresholds are defined in system configuration and enforced by the STK+ observability thread (see doc 08).

---

## 4. Public Readability

### 4.1 Principle

**All logs are publicly readable.** This is non-negotiable. If data is logged, it is transparent.

```
ON-CHAIN DATA:
  Readable by: anyone (general public, auditors, regulators, other systems)
  Access: blockchain explorer, API, direct node query
  No authentication required for read access

OFF-CHAIN DATA:
  Readable by: anyone with access to the MYOS data API
  Access: REST/gRPC API, data export
  No authentication required for read access

  Note: off-chain data contains hashes and metadata.
  Actual payload content is also readable.
  If privacy is needed for specific data (e.g., personal data in AI interactions),
  the data should be encrypted BEFORE entering the pipeline.
  The audit trail still shows encrypted blobs - transparency of the process
  is maintained even if payload content is encrypted.
```

### 4.2 Privacy Considerations

```
MYOS does NOT redact or restrict audit data. However:

  - Cubelets processing sensitive data should encrypt payloads
  - Off-chain entries store encrypted content (hash is of encrypted data)
  - The audit trail shows THAT a decision was made, by WHOM, WHEN
  - The audit trail may not show raw payload if it was encrypted
  - Encryption keys are managed outside the audit system

This preserves:
  - Process transparency: everyone can verify the decision chain
  - Data privacy: sensitive payloads are encrypted at the application level
  - Regulatory compliance: GDPR-style "right to forget" is handled by
    key destruction (encrypted data becomes unreadable, but the audit
    record of decisions remains)
```

### 4.3 Query API

```
audit_query_api {
    // On-chain queries
    get_decision(entry_id) → OnChainEntry
    get_decisions_by_entity(entity_id, time_range) → [OnChainEntry]
    get_decisions_by_type(event_type, time_range) → [OnChainEntry]
    get_block(block_number) → Block
    verify_decision(entry_id) → VerificationResult (merkle proof)

    // Off-chain queries
    get_event(entry_id) → OffChainEntry
    get_events_by_pod(pod_id, time_range) → [OffChainEntry]
    get_events_by_cubelet(cubelet_id, time_range) → [OffChainEntry]
    get_pipeline_trace(task_id) → [OffChainEntry]  // full pipeline data flow
    get_telemetry(entity_id, time_range) → TimeSeries

    // Cross-tier queries
    get_full_audit_trail(task_id) → {
        on_chain:  [OnChainEntry],     // all decisions for this task
        off_chain: [OffChainEntry]     // all data events for this task
    }

    // Verification
    verify_off_chain_integrity(sequence_range) → {
        hash_chain_valid: bool,
        merkle_root_matches_on_chain: bool,
        entries_count: uint64
    }
}
```

---

## 5. Decision Commitment Objects

Important decisions generate **commitment objects** - structured records that capture the full context of a decision for future verification.

```
decision_commitment {
    // Identity
    commitment_id:      UUID
    decision_type:      String

    // Context
    timestamp:          Timestamp
    device_id:          DeviceId
    pod_id:             PodId | null
    task_id:            TaskId | null

    // Participants
    deciding_entity:    EntityId
    affected_entities:  [EntityId]
    authority_snapshots: {
        entity_id: AuthorityVector      // AV of each participant at decision time
    }

    // Decision
    input_hash:         Hash            // hash of the input that led to this decision
    output:             DecisionResult
    reasoning:          String | null   // for LLM orchestrator decisions
    confidence:         float | null    // if applicable

    // Verification
    output_hash:        Hash
    signature:          Signature       // signed by the deciding entity's key

    // Chain reference
    on_chain_entry_id:  UUID            // reference to the on-chain log entry
    block_number:       uint64
    merkle_proof:       MerkleProof
}
```

Commitment objects are stored on-chain and can be independently verified by anyone.

---

## 6. Safety Event Logging

Safety events have special logging treatment - they are **always on-chain, always immediate, never batched**.

```
safety_event_log {
    event_id:           UUID
    severity:           warning | critical | emergency
    timestamp:          Timestamp (hardware clock, not system clock)
    device_id:          DeviceId

    trigger: {
        source:         String          // what detected the safety issue
        description:    String
        sensor_data_hash: Hash | null   // hash of sensor data at time of event
    }

    action_taken: {
        action:         String          // e.g., "emergency_stop", "quarantine_cubelet"
        authority_used: AuthorityVector // who authorized this action
        bypassed_normal_flow: bool     // did this bypass the escalation chain?
    }

    system_state_snapshot: {
        active_pods:    [PodId]
        active_tasks:   [TaskId]
        affected_entities: [EntityId]
    }

    signature:          Signature
}
```

**Safety events bypass normal batching.** They are written to the chain immediately as a priority transaction, not queued with other events.

---

## 7. Boot Verification Log

Every system boot is logged on-chain to establish the trust chain.

```
boot_verification_log {
    boot_id:            UUID
    device_id:          DeviceId
    timestamp:          Timestamp

    boot_chain: [
        {
            stage:      "rom_bootloader"
            hash:       Hash
            signature:  Signature
            verified:   bool
        },
        {
            stage:      "firmware_verifier"
            hash:       Hash
            signature:  Signature
            verified:   bool
        },
        {
            stage:      "kernel_loader"
            hash:       Hash
            signature:  Signature
            verified:   bool
        },
        {
            stage:      "runtime_init"
            hash:       Hash
            signature:  Signature
            verified:   bool
        }
    ]

    config_loaded: {
        config_hash:        Hash
        dimension_registry: [DimensionName]
        authority_floors:   { entity_type: floor_value }
    }

    hardware_attestation: {
        hardware_public_key: PublicKey
        attestation_cert:    Certificate
        firmware_version:    String
    }

    boot_signature:     Signature       // entire boot record signed by hardware key
}
```

---

## 8. Configuration

```
verification_audit_config {
    // On-chain
    chain {
        consensus:              "ouroboros"
        block_time_ms:          1000            // 1 second blocks
        validators_min:         3
        consensus_module:       "haskell_ouroboros_core"
        storage_module:         "rust_block_store"
        network_module:         "rust_p2p_network"
    }

    // Cardano anchoring
    cardano_anchor {
        enabled:                true
        anchor_frequency_blocks: 100
        cardano_network:        "mainnet"       // or "testnet"
        anchor_wallet:          "addr1..."
    }

    // Off-chain
    off_chain {
        storage_backend:        "event_log_content_addressed"
        hash_algorithm:         "blake3"
        checkpoint_frequency:   1000            // entries between integrity checkpoints
        time_series_index:      true
    }

    // Retention
    retention:                  // see Section 3.6

    // Public access
    public_api {
        enabled:                true
        rate_limit:             1000            // requests per minute
        cors_origins:           ["*"]           // publicly accessible
    }

    // Safety events
    safety {
        immediate_write:        true            // bypass batching
        priority_transaction:   true
    }

    // Kusari evolution
    kusari_compatibility {
        modular_consensus:      true
        adapter_interface:      "v1"
        future_adapters:        ["bft", "poa", "kusari_hybrid"]
    }
}
```

---

## 9. Invariants (Must Always Hold)

```
INV-1:  All on-chain data is publicly readable by anyone without authentication
INV-2:  All off-chain data is publicly readable via the query API
INV-3:  On-chain entries are immutable (append-only, no updates, no deletes)
INV-4:  Off-chain entries are append-only with hash chain integrity
INV-5:  Off-chain merkle roots are periodically anchored on-chain
INV-6:  Safety events are written to chain immediately (never batched)
INV-7:  Every system boot produces an on-chain boot verification log
INV-8:  Consensus module implements the standard adapter interface
INV-9:  Consensus module can be replaced without changing application code
INV-10: Cardano anchoring is optional and configurable per deployment
INV-11: Tier migration preserves content hashes (data integrity across tiers)
INV-12: Cold tier data is always retrievable (never permanently deleted unless configured)
INV-13: STK invariant satisfaction (PROVES edges verified) is logged on-chain for every pod execution
INV-14: STF fabric threads route domain-specific data to appropriate off-chain stores
INV-15: Light validation nodes can verify individual decisions via merkle proofs
INV-16: The chain is designed to evolve into or integrate with Kusari without rewrite
INV-17: Decision commitment objects are self-contained and independently verifiable
INV-18: Statistical monitoring thresholds are evaluated continuously, not on-demand
INV-19: Threshold violations trigger automatic protective actions (halt, suspend, audit)
```

---

## 10. Interaction with Other Documents

- **Authority Model (01-authority-model.md):** Every authority decision is logged on-chain. AV snapshots are included in decision commitments.
- **Conflict Resolution (02-conflict-resolution.md):** All conflict resolution outcomes (all 4 levels) are logged on-chain with full context.
- **Pod Assembly (03-pod-assembly.md):** Pod creation, dissolution, and cubelet quarantine are on-chain events. Assembly telemetry is off-chain.
- **Cubelet I/O Contract (04-cubelet-io-contract.md):** Pipeline edge hashes are logged off-chain. On-chain vs off-chain split is defined here.
- **Pod Orchestrator (05-pod-orchestrator.md):** Pod Orch decisions that affect authority (quarantine, restructure, escalation) go on-chain. DAG construction and synthesis traces go off-chain.
- **Kusari (kusari.md):** The MYOS chain is designed to evolve into a Kusari KVM, sidechain, or consensus contributor. Modular architecture enables this without rewrite.
- **Master Document (00-master.md):** The cubelet lattice (750 cubelets = 10 stages x 5 frameworks x 15 per cell) defines the complete verification surface. Every cell in the lattice has STK invariants that must be satisfied and logged.
- **Authority Model (01) - STK Dual-Gate:** STK invariant satisfaction is enforced by the Authority Engine's dual-gate model. PROVES edges in the CIG are verified before any pod execution proceeds, and results are logged on-chain.
- **STF Fabric Threads:** The 11+ named ledgers (EthosLedger, KarbonLedger, etc.) route domain-specific verification data to appropriate off-chain stores. Each fabric thread determines which off-chain tier and retention policy applies to its data.
- **Knowledge Base (12-knowledge-base.md):** KB metadata changes (create, update, deprecate entries) are on-chain events. KB content is stored off-chain with hash verification using the same content-addressed storage infrastructure. KB confidence scores (0-1000) follow the same mathematical model as Authority Vectors.
