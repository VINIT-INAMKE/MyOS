# MYOS - Math-Based Operating System for Deterministic AI Execution

## Version: 2.0 | Status: LIVING DOCUMENT

---

## What is MYOS?

MYOS is a **deterministic AI operating system** - a complete execution platform where AI systems operate under mathematically enforced authority, verifiable decision-making, and reproducible builds. It is not a traditional OS that runs on a single machine. It is a **distributed operating system** that spans centralized servers, decentralized compute nodes, chain validators, and edge sensors - all governed by a single, unified authority model.

MYOS answers a fundamental question: **How do you let AI systems act autonomously while guaranteeing they stay within their authority?**

The answer is **Math-Based Access Control (MBAC)** - a continuous, reinforcement-driven authority system where every entity earns and loses trust based on proven performance, not static role assignments. MBAC is backed by a lattice of **750 cubelets** - named, verifiable socio-technical micro-contracts organized across **10 domain stages** and **5 framework layers**, each carrying formal invariants that the system must satisfy at all times.

---

## Who is MYOS For?

MYOS is a general-purpose deterministic AI platform. It serves two primary use cases:

**Safety-Critical Autonomous Systems**

- Industrial robotics, autonomous vehicles, smart grid monitoring
- Environmental sensor networks, energy infrastructure
- Any system where an AI making the wrong decision has physical consequences

**Conversational AI & Knowledge Systems**

- AI assistants with verifiable reasoning chains
- Multi-domain knowledge systems with authority-bounded responses
- AI platforms where trust, compliance, and auditability matter

The same architecture handles both. A robot moving an arm and an AI answering a medical question both go through the same authority pipeline, the same pod assembly protocol, and the same audit trail.

---

## Five Principles

| Principle                        | What It Means                                                                                                                  |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| **Deterministic Execution**      | Same input → same output. No scheduler randomness, no race conditions, no non-deterministic behavior in critical paths.        |
| **Hardware-Rooted Trust**        | Security starts at the SoC. Secure boot chain verifies every layer before it runs. If verification fails, the system halts.    |
| **Authority-Controlled Actions** | Every action passes through the Authority Engine. No entity can act beyond its mathematically computed authority.              |
| **Minimal Attack Surface**       | Only essential components run. Cubelets are sandboxed (Wasm + Unikernels). No unnecessary daemons, drivers, or services.       |
| **Verifiable Operation**         | Every decision is logged on-chain. Every data transfer is hashed off-chain. All logs are publicly readable. Nothing is hidden. |

---

## System Architecture

MYOS is organized into **layers** and **cross-cutting services**. The execution layer is structured as a **Rubik's Lattice** - a 10×5×15 coordinate system of 750 cubelets spanning domain stages (rows), framework layers (columns), and functional indices (depth).

```
┌─────────────────────────────────────────────────────────────────────┐
│                        USER / EXTERNAL SYSTEMS                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ORCHESTRATION LAYER                                               │
│   ┌───────────────────────────────────────────────────────────┐     │
│   │  System Orchestrator (1, singleton, paid LLM)             │     │
│   │  ┌─────────────────────┐  ┌─────────────────────────┐    │     │
│   │  │ Domain Orchestrators│  │ Deployment Orchestrators │    │     │
│   │  │ (per vertical,      │  │ (per zone, open-source   │    │     │
│   │  │  paid LLM)          │  │  LLM)                    │    │     │
│   │  └────────┬────────────┘  └────────┬────────────────┘    │     │
│   │           │                        │                      │     │
│   │  ┌────────▼────────────────────────▼────────────────┐    │     │
│   │  │        Pod Orchestrators (many, ephemeral,        │    │     │
│   │  │        open-source LLM)                           │    │     │
│   │  └───────────────────────────────────────────────────┘    │     │
│   └───────────────────────────────────────────────────────────┘     │
│                                                                     │
│   EXECUTION LAYER - THE RUBIK'S LATTICE (750 Cubelets)             │
│   ┌───────────────────────────────────────────────────────────┐     │
│   │              STA(150)  STI(150)  STD(150)  STF(150)  STK(150)  │
│   │  Stage 0-3   Intent    Logic     Runtime   Fabric    Proof     │
│   │  (Foundation) ─────────────────────────────────────────────     │
│   │  Stage 4-5   Intent    Logic     Runtime   Fabric    Proof     │
│   │  (Physical)   ─────────────────────────────────────────────     │
│   │  Stage 6-7   Intent    Logic     Runtime   Fabric    Proof     │
│   │  (Governance) ─────────────────────────────────────────────     │
│   │  Stage 8-9   Intent    Logic     Runtime   Fabric    Proof     │
│   │  (Planetary)  ─────────────────────────────────────────────     │
│   │                                                                 │
│   │  Model Types:                                                   │
│   │  ┌──────────┐  ┌──────────────┐  ┌──────────────────┐         │
│   │  │ Math     │  │ ML/DL Models │  │ LLM Agents       │         │
│   │  │ (Wasm +  │  │ (Wasm +      │  │ (Linux container  │         │
│   │  │ Unikernel│  │  Unikernel)  │  │  on x86 GPU)     │         │
│   │  │ ~350-400 │  │  ~200-250    │  │  ~100-150        │         │
│   │  └──────────┘  └──────────────┘  └──────────────────┘         │
│   └───────────────────────────────────────────────────────────┘     │
│                                                                     │
│   RUNTIME LAYER                                                     │
│   ┌───────────────────────────────────────────────────────────┐     │
│   │  NixOS (PREEMPT_RT kernel)                                │     │
│   │  RT Scheduler │ IPC │ Resource Mgr │ Watchdog             │     │
│   └───────────────────────────────────────────────────────────┘     │
│                                                                     │
│   HARDWARE LAYER                                                    │
│   ┌───────────────────────────────────────────────────────────┐     │
│   │  x86/ARM (core, compute, validators)                      │     │
│   │  RISC-V (edge sensors, general compute)                   │     │
│   └───────────────────────────────────────────────────────────┘     │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  CROSS-CUTTING SERVICES                                             │
│  ┌──────────────────────┐  ┌──────────────────────────────────┐    │
│  │  Authority Engine    │  │  Verification & Audit Chain      │    │
│  │  (Haskell, MBAC)     │  │  (Ouroboros + off-chain logs)    │    │
│  │  Enforces STK        │  │  Records via STF fabric threads  │    │
│  │  invariants          │  │  Decentralized validators        │    │
│  └──────────────────────┘  └──────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Cubelet Interaction Graph (CIG)                              │  │
│  │  Neo4j - 750 nodes, 9 edge types, 1500+ edges                │  │
│  │  Structural backbone for pod assembly and KB                  │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Core Concepts

### Cubelets - The Atomic Units

A **cubelet** is the smallest execution unit in MYOS. There are **750 cubelets** in the system, organized as a **Rubik's Lattice**: 10 domain stages × 5 framework layers × 15 cubelets per cell.

Every cubelet has a **unique identity** defined by its position in the lattice (e.g., `STI-3-B` = Stage 3, Interpretation framework, index B = "Explainability Logic Model"). A cubelet's lattice position is permanent; its model is a versioned artifact deployed to that position.

#### The Five Frameworks (Columns)

Each framework layer represents a different level of abstraction, from intent to proof:

| Framework | Name | Role | What It Contains | Primary Model Type |
| --------- | ---- | ---- | ---------------- | ------------------ |
| **STA** | Abstraction | Intent & Ethics - the "why" and "what" | Charters, policies, rules, standards | Math-bound (formalized rules) + LLM (policy reasoning) |
| **STI** | Interpretation | Logic & Reasoning - the "how to think" | Ontologies, classifiers, schemas, reasoning engines | ML/DL models (XGBoost, small transformers, MoE gates) |
| **STD** | Deliverance | Execution - the "how to run" | Runtimes, services, APIs, dashboards | Mix (pure code + ML inference + LLM runtimes) |
| **STF** | Fabric | Connectivity - the "how to connect" | Bridges, channels, ledger connectors, relays | Math-bound (protocol logic, cryptography) |
| **STK** | Kernel | Proof & Assurance - the "what must always be true" | Formal invariants, proof circuits, constraint checkers. Executed by **ProofPerl** (the STK runtime engine) under **PerlFrame** (the value anchor) | Math-bound (formal verification, ZK proofs) |

The frameworks form a vertical pipeline within each stage: **STA** defines intent → **STI** formalizes logic → **STD** executes → **STF** connects → **STK** proves.

**ProofPerl** is the execution engine for STK invariants - a formal proof runtime that evaluates constraint satisfaction. **PerlFrame** is the value system within STK that anchors all invariants to the PCCSCEFVRI ethical values. Together they form the kernel's enforcement mechanism: ProofPerl runs the checks, PerlFrame defines what the checks mean. See Section "STK Invariant Enforcement" in [01-authority-model.md](01-authority-model.md) for the dual-gate authorization flow.

#### The Ten Stages (Rows)

Each stage represents a domain of operation, from foundational infrastructure to ethical reflexion:

| Tier | Stages | Domain Focus | Activation |
| ---- | ------ | ------------ | ---------- |
| **I. Foundation** | 0 - Identity | DIDs, consent, credentials, guardianship | MYOS core |
| | 1 - Governance | Delegation, quorum, treaties, appeals | MYOS core |
| | 2 - Data | Sovereignty, consent DAGs, lineage, privacy | MYOS core |
| | 3 - Inference/AI | Safety, explainability, fairness, guardrails | MYOS core |
| **II. Physical** | 4 - Sensors/DePIN | Attestation, calibration, energy, rewards | MYOS core |
| | 5 - Assets/RWA | Tokenization, provenance, escrow, settlement | Kusari activation |
| **III. Governance** | 6 - Economy/Treasury | Public-good economy, fiscal policy, budgets | Kusari activation |
| | 7 - Social Intelligence | Knowledge commons, credentials, media integrity | Kusari activation |
| **IV. Planetary** | 8 - Environment | MRV, carbon, biodiversity, circular economy | Kusari activation |
| | 9 - Meta-Ethics | Reflexive ethics, corrigibility, value evolution | Kusari activation |

Stages 0-4 form the **MYOS core** - the deterministic edge AI OS for safety-critical systems. Stages 5-9 activate progressively as MYOS decentralizes into Kusari.

#### Three Model Types

Each cubelet's model defines its computational identity. Models are trained externally and deployed to cubelets; the Authority Engine's reinforcement dynamics (MBAC) govern trust at the system level.

| Type | What It Does | Isolation | Hardware | Approximate Count | Determinism |
| ---- | ------------ | --------- | -------- | ----------------- | ----------- |
| **Math-Bound** | Formal verification, cryptographic proofs, rule engines, scoring functions, ZK circuits | Tier 1: Wasm inside Unikraft unikernel | x86 or RISC-V | ~350-400 | Perfect |
| **ML/DL Model** | Classification, ranking, anomaly detection, NLI, semantic matching, MoE gating | Tier 1-2: Wasm (+ optional Unikernel) | x86 or RISC-V | ~200-250 | Near-perfect (fixed weights) |
| **LLM Agent** | Novel reasoning, intent interpretation, synthesis, user interaction | Tier 3: Linux container | x86 GPU only | ~100-150 | Non-deterministic (outputs audited) |

**The majority of cubelets (~75%) run in the strongest isolation tiers** (Wasm + Unikernel). LLM cubelets are the minority, used only where genuine reasoning capability is required.

A cubelet's model can be updated (retraining for ML/DL, new version for math-bound). When this happens, the model hash changes and the cubelet's **Authority Value resets** - the system must re-earn trust in the updated capability. The lattice position remains permanent.

**All cubelets are exclusive** - a cubelet can belong to only one pod at a time. No sharing, no time-slicing between pods.

> _Full details: [03-pod-assembly.md](03-pod-assembly.md) Section 2_

### The Cubelet Interaction Graph (CIG)

The 750 cubelets are connected by a **directed graph** - the Cubelet Interaction Graph - stored in Neo4j. The CIG defines which cubelets can work together, how they depend on each other, and how proof flows through the lattice.

**Nine edge types:**

| Edge Type | Meaning | Example |
| --------- | ------- | ------- |
| **DEPENDS_ON** | A needs B to operate | STD-2-A depends on STI-2-A for consent logic |
| **PROVES** | A provides evidence satisfying B | STD-8-A proves STK-8-A (MRV invariant) |
| **GOVERNS** | A sets policy/constraints for B | STA-3-B governs STD-3-B (AI safety policy → safety runtime) |
| **INFORMS** | A emits data B consumes | STF-4-A informs STI-8-A (sensor data → climate logic) |
| **DERIVES** | A is algorithmically derived from B | STI-6-A derives from STD-2-B (fiscal logic from data lineage) |
| **ENACTS** | A executes/implements B | STD-1-A enacts STA-1-A (DAO engine enacts governance charter) |
| **FEDERATES** | A synchronizes with peer A' | STF-0-A federates with STF-0-A on another device |
| **CROSSWALKS** | A bridges schema/identity/ledger domains | STF-5-C crosswalks STF-0-B (asset identity ↔ personal identity) |
| **AUDITS** | A produces audit trail for B | STK-3-A audits STD-3-A (AI fairness invariant audits inference) |

The CIG is **pre-seeded** at system initialization with the full lattice structure (~1500 edges). Runtime knowledge accumulates on top of structural knowledge. Pod assembly uses the CIG to resolve dependencies - it doesn't guess which cubelets work together.

**Three link directions:**
- **Vertical** (intra-stage): STA-3-A → STI-3-A → STD-3-A → STF-3-A → STK-3-A
- **Horizontal** (cross-stage): STA-2-A → STA-3-A (data policy feeds AI policy)
- **Diagonal** (cross-domain): STI-3-B → STD-8-A (AI fairness logic informs environmental MRV)

### Pods - Ephemeral Teams (STSol Assemblies)

A **pod** is a temporary team of cubelets assembled to execute a specific task. Pods are:

- **Assembled on demand** by System, Domain, or Deployment Orchestrators
- **Governed by a Pod Orchestrator** (always an LLM) that builds the data pipeline, monitors execution, and synthesizes results
- **Dissolved after task completion** - cubelets return to the pool
- **Structurally informed** by the CIG - each pod corresponds to an **STSol** (Solution Pod), a vertical cross-section of the lattice

An STSol defines the **template** for pod assembly: which cubelet positions are required, which edges must be satisfied, which invariants apply. For example, the `Env-MRV-Pod` STSol requires cubelets from STA-8, STI-8, STD-8, STF-8, STK-8 plus diagonal dependencies from Stage 4 (sensor data).

Pods communicate internally via a **typed data pipeline (DAG)** and **control messages**. The pipeline follows the framework order: STA governs → STI reasons → STD executes → STF records → STK proves. Cross-pod communication is capability-addressed (the requesting cubelet never knows which pod serves its request).

> _Full details: [03-pod-assembly.md](03-pod-assembly.md), [04-cubelet-io-contract.md](04-cubelet-io-contract.md)_

### Authority Engine - The Rules

The **Authority Engine** is a centralized Haskell service that enforces Math-Based Access Control (MBAC). Every entity in MYOS carries a **multi-dimensional Authority Value (AV)** vector:

```
Cubelet_STI_3_B = {
    execution:      650.0,
    safety:         880.2,
    reasoning:      920.5,
    communication:  710.3,
    knowledge:      540.0,
    autonomy:       320.8,
    privacy:        750.0,
    equity:         680.5,
    accountability: 810.0,
    integrity:      790.2
}
```

Authority is **earned through performance** and **lost through failures and inactivity**:

- **Rewards** follow diminishing returns (harder to gain authority at the top)
- **Penalties** are constant (a mistake always costs the same)
- **Decay** erodes unused authority over time (no stale privileges)
- **Floors** guarantee minimum authority per entity type (System=700, Domain=600, Deployment=500, Pod=400, Cubelet=0)
- **Ceiling** is hard-capped at 1000 (no unbounded authority accumulation)

#### STK Invariant Enforcement

The Authority Engine enforces the **150 STK kernel invariants** organized into 7 families. Every action must satisfy the relevant invariants for the cubelets involved:

| Invariant Family | Guarantee | Example Constraints |
| ---------------- | --------- | ------------------- |
| **Privacy & Consent** | Minimal disclosure, data sovereignty | `revocation.window <= 24h`, `consent.required` |
| **Fairness & Equity** | Bounded bias, revenue-share fairness | `bias <= θ`, `revenue_share.fairness` |
| **Verifiability & Determinism** | ZK-proof existence, reproducible execution | `zk_proof.exists`, `hash.reproducible` |
| **Safety & Security** | Refusal on redline, system isolation | `refusal_on_redline`, `system.isolated` |
| **Accountability & Audit** | Signed logs, upgrade audited | `logs.signed`, `upgrade.audited` |
| **Planetary Integrity** | MRV verified, boundary compliance | `MRV.verified`, `planetary_boundary <= 1` |
| **Ethical Reflexivity** | PCCSCEFVRI anchors intact | `ethics.aligned`, `corrigibility.maintained` |

**PCCSCEFVRI** is the ethical value anchor: **P**rivacy, **C**are, **C**onsent, **S**afety, **C**reativity, **E**quity, **F**airness, **V**erifiability, **R**esponsibility, **I**ntegrity. Every STK cubelet's final invariant (the "-O" entry per stage) ensures operations comply with these values.

The Authority Engine runs **centralized on the core server**. All nodes - including edge RISC-V sensors - call it via RPC. It is implemented in Haskell for purity, type safety, and formal verification proximity. Built reproducibly using haskell.nix.

> _Full details: [01-authority-model.md](01-authority-model.md)_

### Fabric Threads - The Connective Tissue

The **STF framework** defines 11+ named fabric threads - interoperability ledgers that connect cubelets across stages and devices:

| Fabric | Domain | Architecture Role |
| ------ | ------ | ----------------- |
| **IdentLedger** | Identity & access | Device identity, ANS registry, DID resolution |
| **EthosLedger** | Ethics & culture | Authority Engine event log, ethical audit trail |
| **KnowledgeLedger** | Education & knowledge | Knowledge Base backbone (Qdrant + Neo4j) |
| **KarbonLedger** | Climate & carbon | Domain KB: Carbon, MRV records |
| **CivicLedger** | Governance | Governance decisions, voting records |
| **TreasuryLedger** | Fiscal flows | Token/fee transactions |
| **DePINFabric** | Sensors & DePIN | Edge device networking, attestation |
| **FinanceLedger** | Economic assets | Asset transactions, settlement |
| **OpenDataLedger** | Data commons | Off-chain event log, data lineage |
| **PerlFrame** | Proof & meta-ethics | STK enforcement runtime (ProofPerl engine) |
| **WaterLedger / AgroLedger / EcoLedger** | Planetary resources | Domain KBs: Environment, agriculture, ecology |

Each fabric thread is an STF cubelet subsystem that handles serialization, routing, and verification for its domain. Fabric threads map to the verification chain's multiplexed channels (doc 09) and the Knowledge Base's domain-specific stores (doc 12).

### Orchestrator Hierarchy - The Brain

MYOS uses a **four-level orchestrator hierarchy**. All orchestrators are LLM-powered.

```
System Orchestrator (1, singleton)
  ├── Domain Orchestrators (per vertical - Healthcare, DeFi, Carbon, AI/ML)
  └── Deployment Orchestrators (per zone/cluster)
        └── Pod Orchestrators (many, ephemeral, per task)
```

| Level          | Count            | LLM                               | AV Floor | Scope                                 |
| -------------- | ---------------- | ---------------------------------- | -------- | ------------------------------------- |
| **System**     | 1 (singleton)    | Paid (Claude Opus class)           | 700      | Entire MYOS instance                  |
| **Domain**     | One per vertical | Paid (Claude Sonnet class)         | 600      | Full autonomy within domain           |
| **Deployment** | One per zone     | Open-source (Llama 8B via Ollama)  | 500      | Infrastructure + environment per zone |
| **Pod**        | Many, ephemeral  | Open-source (Llama 8B / Phi-3)     | 400      | Single task execution                 |

**Domain and Deployment Orchestrators are parallel peers** under System, not hierarchically nested. They coordinate laterally when needed. All three upper levels (System, Domain, Deployment) can create Pod Orchestrators.

Orchestrators use the **CIG** to inform decisions: pod assembly follows DEPENDS_ON/PROVES/GOVERNS edges, conflict resolution consults the STA policies and STK invariants that govern the involved cubelets, and knowledge queries route through STF fabric threads.

> _Full details: [05-pod-orchestrator.md](05-pod-orchestrator.md), [10-node-topology-orchestrator-hierarchy.md](10-node-topology-orchestrator-hierarchy.md)_

### Conflict Resolution - When Entities Disagree

MYOS resolves conflicts through a **4-level escalation chain**:

| Level       | Mechanism                                                          | Resolves          | Timeout |
| ----------- | ------------------------------------------------------------------ | ----------------- | ------- |
| **Level 1** | Direct AV comparison on relevant dimension                         | ~70% of conflicts | 100ms   |
| **Level 2** | AV-weighted voting within the pod (quorum + confidence threshold)  | ~20% of conflicts | 500ms   |
| **Level 3** | System Orchestrator arbitration (informed by CIG GOVERNS edges)    | ~8% of conflicts  | 5s      |
| **Level 4** | Human-in-the-loop (veto only on safety-critical)                   | ~2% of conflicts  | configurable |

At Level 3, the System Orchestrator consults the **STA cubelets** that govern the conflicting cubelets (via GOVERNS edges in the CIG) and the **STK invariants** that constrain them. This provides domain-aware arbitration rather than generic weighted scoring.

If any level times out, the conflict escalates. If Level 4 times out, the system defaults to the **safest available option**. Human decisions do NOT affect entity AV - the reinforcement model is purely performance-based.

> _Full details: [02-conflict-resolution.md](02-conflict-resolution.md)_

### Verification & Audit - The Ledger

Every decision and action is logged using a **two-tier system**, routed through STF fabric threads:

**On-chain** (Ouroboros-based lightweight chain):

- Authority decisions, conflict resolutions, human overrides, safety events, pod lifecycle decisions
- Tamper-proof, publicly readable, permanent
- STK invariant satisfaction records (which PROVES edges were satisfied per action)
- Anchored to Cardano mainnet for external verification
- Designed to evolve into Kusari (see [kusari.md](../../kusari.md))

**Off-chain** (event log + content-addressed store + time-series DB):

- Pipeline edge hashes, intermediate results, telemetry, control messages
- Routed through domain-specific fabric threads (KarbonLedger for carbon data, FinanceLedger for asset transactions, etc.)
- Tiered retention: hot (SSD, minutes-hours) → warm (HDD, days-weeks) → cold (archive, months-years)

**All logs are publicly readable.** Anyone - auditors, regulators, the general public - can read any log. Transparency is not optional.

> _Full details: [06-verification-audit.md](06-verification-audit.md)_

---

## Physical Topology

MYOS is a **hybrid centralized-decentralized system**:

```
┌──────────────────────────────────┐
│  CENTRALIZED CORE (x86/ARM)     │
│  System Orch + Authority Engine  │
│  CIG (Neo4j) + System KB        │
│  NixOS full                      │
└──────────────┬───────────────────┘
               │
  ┌────────────┼────────────┐
  │            │            │
  ▼            ▼            ▼
Domain       Deploy      Chain
Servers      Nodes       Validators
(x86/RV)     (zone HW)   (x86/ARM)
NixOS full   NixOS full  NixOS full
Domain KB    Deploy KB
  │            │
  ▼            ▼
Compute Nodes            Edge Sensors
(x86 GPU + x86/RV)      (RISC-V)
NixOS full               NixOS minimal
LLM + ML cubelets        Math + quantized ML cubelets
                         (Stages 0, 4 primarily)
```

| Node Type                  | Hardware                | What Runs                                                | NixOS Profile |
| -------------------------- | ----------------------- | -------------------------------------------------------- | ------------- |
| **Core Server**            | x86/ARM (RISC-V future) | System Orch + Authority Engine + CIG + System KB        | Full          |
| **Domain Server**          | x86 or RISC-V           | Domain Orchestrator + Domain KB (semi-autonomous)       | Full          |
| **Deployment Node**        | Same as managed zone    | Deployment Orchestrator (infra + env)                   | Full          |
| **Compute Node (GPU)**     | x86 with GPU            | LLM agent cubelets (containers, Tier 3)                 | Full          |
| **Compute Node (General)** | x86 or RISC-V           | Math/ML cubelets (Wasm + Unikraft, Tier 1-2)           | Full          |
| **Chain Validator**        | x86/ARM dedicated       | Ouroboros consensus, block production                   | Full          |
| **Edge Sensor**            | RISC-V                  | Stage 4 sensor cubelets + STK invariant checkers (Tier 1) | Minimal     |

> _Full details: [10-node-topology-orchestrator-hierarchy.md](10-node-topology-orchestrator-hierarchy.md)_

---

## Technology Stack

### Languages

| Component                | Language        | Rationale                                                  |
| ------------------------ | --------------- | ---------------------------------------------------------- |
| Authority Engine         | **Haskell**     | Pure functions, type safety, formal verification proximity |
| Ouroboros Consensus Core | **Haskell**     | Same as Authority Engine - mathematical correctness        |
| Runtime Services         | **Rust**        | Memory safety, zero-cost abstractions, RISC-V/Wasm targets |
| Cubelets (math/ML)       | **Rust → Wasm** | Compiled to Wasm for sandboxed, portable execution         |
| System Configuration     | **Nix**         | Declarative, reproducible, deterministic builds            |

### Key Technologies

| Technology                             | Role in MYOS                                                                                                     |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **NixOS**                              | Base OS on all nodes. Declarative configuration, atomic rollback, reproducible builds. Replaces Buildroot/Yocto. |
| **Unikraft**                           | Unikernel framework for math/ML cubelet isolation. Wasm-inside-unikernel = double isolation.                     |
| **Wasmtime**                           | WebAssembly runtime for cubelet sandboxing. AOT compiled at boot for near-native performance.                    |
| **Neo4j**                              | Cubelet Interaction Graph (CIG) storage. Pre-seeded with 750 nodes + 1500 edges. Also used for KB graph layer. |
| **Qdrant**                             | Vector DB for KB semantic search. Queried in parallel with Neo4j.                                               |
| **LibP2P** (qudag-network)             | P2P mesh networking. Device discovery, encrypted transport, NAT traversal.                                       |
| **Ouroboros**                          | Slot-based PoS consensus (Haskell core). Lightweight chain for decision logging.                                 |
| **Post-quantum crypto** (qudag-crypto) | ML-KEM-768 (key exchange), ML-DSA (signatures), BLAKE3 (hashing). Chain + auth channels.                         |
| **Ollama / Groq**                      | Open-source LLM hosting for Deployment and Pod Orchestrators. Zero API cost.                                     |

### Build System

The entire MYOS system is described in a single `flake.nix`:

```
flake.nix
├── nixosConfigurations/     # One per node type (core, domain, compute, edge, validator)
├── packages/                # Authority Engine, Ouroboros, runtime, cubelets, qudag-*
└── overlays/                # PREEMPT_RT kernel, Unikraft builder
```

Every system image is **bit-for-bit reproducible**. Same `flake.nix` + same `flake.lock` = identical output on any machine, every time. This is MYOS's first principle (deterministic execution) applied to the build process itself.

> _Full details: [10-node-topology-orchestrator-hierarchy.md](10-node-topology-orchestrator-hierarchy.md) Section 5_

---

## Existing Repository Integration

MYOS builds on **38 existing repositories** from the ruvnet/agenticsorg portfolio:

| Category          | Key Repos                                                                       | Role                                     |
| ----------------- | ------------------------------------------------------------------------------- | ---------------------------------------- |
| **Orchestration** | Claude-Flow                                                                     | System/Domain/Pod Orchestrator framework |
| **Identity**      | ANS (Agent Name Service)                                                        | All entity identity and discovery        |
| **Crypto**        | qudag-crypto, qudag-vault-core                                                  | Post-quantum encryption, key management  |
| **Networking**    | qudag-network, qudag-protocol                                                   | P2P mesh, wire protocol                  |
| **Consensus**     | qudag-dag                                                                       | DAG-based transaction ordering           |
| **Cubelets**      | ONNX-Agent, ruv-fann, MidStream, cuda-rust-wasm, sublinear, bit-parallel-search | ML/math/LLM cubelet implementations      |
| **Safety**        | SAFLA, GuardRail                                                                | Safety validation, guardrails            |
| **Domain DSLs**   | SynthLang, PromptLang                                                           | Domain-specific prompt languages         |
| **DevOps**        | Agentic DevOps, Agentic Edge Functions                                          | Infrastructure + edge management         |
| **Cross-Device**  | Federated MCP                                                                   | Distributed MCP tool coordination        |
| **Edge AI**       | WiFi-DensePose, Quantum Magnetic Navigation                                     | Edge sensor capabilities                 |

**10 new components** must be built from scratch: Authority Engine (Haskell), Ouroboros Consensus Core (Haskell), Consensus Adapter, Cubelet Runtime Manager, NixOS System Configuration, Unikraft Integration, Pod Orch Templates, Dimension Registry, Edge Sensor Agent, Authority RPC Client.

> _Full details: [11-repository-mapping.md](11-repository-mapping.md)_

---

## Knowledge Base - Grounding Agent Outputs in Verified Facts

Every LLM in MYOS can hallucinate. The **Knowledge Base (KB)** ensures agent outputs are grounded in verified, confidence-scored facts instead of training data guesses.

### Hierarchical Structure

```
System KB (global rules, cross-domain facts)
  ├── Domain KBs (per vertical - mapped to STF fabric threads)
  │     ├── KarbonLedger KB (carbon, MRV, environment)
  │     ├── FinanceLedger KB (assets, settlement)
  │     ├── KnowledgeLedger KB (education, credentials)
  │     └── ... (one per active fabric thread)
  └── Pod KBs (ephemeral - task context + session memory)

Knowledge flows DOWN. Discoveries flow UP (promoted on pod dissolution).
```

### Structural Backbone: The CIG

The Knowledge Base is built on top of the **Cubelet Interaction Graph**. The CIG provides the structural skeleton - which cubelets relate to which, through what edge types. Runtime knowledge (facts, confidence scores, verified outputs) accumulates as properties on CIG nodes and edges. The KB doesn't start empty; it starts with 750 nodes and 1500+ structural relationships.

### How Agents Use the KB

Every agent follows a mandatory query protocol:

1. **Deterministic KB lookup** (exact match) → if found with high confidence, use directly → response marked **VERIFIED**
2. **Semantic KB search** (RAG-style) → if relevant entries found, inject into LLM context → response marked **PARTIALLY VERIFIED**
3. **LLM fallback** (no KB support) → LLM reasons from training data → response marked **UNVERIFIED**

If an LLM output **contradicts a verified KB entry**, the output is **blocked** and the contradiction enters the conflict resolution chain.

### Authority Integration

- Knowledge entries have **confidence scores** (0-1000, same math as AV - diminishing returns on confirmation, constant penalty on contradiction, decay over time)
- Agents need sufficient **knowledge dimension AV** to write to the KB (System KB requires 800, Domain requires 500, Pod requires 100)
- Initial confidence scales with the author's AV - high-authority agents' knowledge starts more trusted
- Contradictions go through the **same 4-level conflict resolution chain** as cubelet disagreements

### Storage

**Vector DB** (Qdrant) for semantic search + **Knowledge Graph** (Neo4j, shared with CIG) for structured relationships. Queried in parallel, results merged. KB metadata on-chain (tamper-proof audit trail), content off-chain.

> _Full details: [12-knowledge-base.md](12-knowledge-base.md)_

---

## The Autopoietic Loop

MYOS is not a static system. The lattice includes a self-governing feedback loop where the system evaluates its own behavior and evolves its policies:

```
Stage 9: Meta-Ethics (Ethics-Runtime-Pod)
    │  Evaluates system behavior against PCCSCEFVRI
    │  Updates STA policies, tightens/loosens STK invariants
    ▼
Stage 0-1: Identity + Governance
    │  Authority rules updated, governance policies revised
    ▼
Stage 2-5: Data + AI + Sensors + Assets
    │  Cubelets operate under updated rules
    │  Execution results logged to verification chain
    ▼
Stage 6-8: Economy + Culture + Environment
    │  Outcomes evaluated against planetary/social objectives
    ▼
Stage 9: Cycle repeats (feeds back to reflection)
```

In architecture terms:
- Stage 9 cubelets have the **highest authority requirements** - only the System Orchestrator and top-tier cubelets can modify meta-ethical constraints
- STK invariant updates require **Level 4 human-in-the-loop** approval - no automated system can weaken its own ethical constraints
- The AV reward/penalty/decay cycle IS the reinforcement learning: the system learns which cubelets to trust for which tasks through continuous performance evaluation against defined values

---

## Lattice Governance - Rubik's Moves

The lattice is not static. It evolves through **Rubik's Moves** - formal, STK-constrained governance operations that recompose the lattice. Every structural change to the lattice (enabling a new stage, updating a cubelet's model, modifying an STK invariant, adding a fabric thread) is a Rubik's Move.

### Three Move Types

| Move | What It Does | Pre-conditions | Post-conditions |
| ---- | ------------ | -------------- | --------------- |
| **Rotate.Stage(k → k+1)** | Promotes a stage from draft to active - enables all cubelets in that stage | All STD cubelets at stage k have PROVES edges satisfied by STK cubelets. All STA policies at stage k are finalized. Level 4 (human) approval. | ENACTS edges added from STD to next-stage cubelets. Stage k+1 cubelets become available for pod assembly. |
| **Lock.Invariants(S)** | Freezes a set of STK invariants - prevents any modification | Invariants in set S are fully verified (all PROVES edges satisfied). System Orchestrator approval. | Blocked: any update that would weaken a locked invariant. Locked invariants can only be unlocked via Level 4 human + system reboot. |
| **Thread.Sync(FabricId)** | Ensures consistent CROSSWALKS and FEDERATES edges for a fabric thread across all devices | Fabric thread exists on at least 2 devices. All STF cubelets for the thread are active. | Spanning tree built per fabric. All FEDERATES edges verified. Cross-device queries for this thread are enabled. |

### Move Execution Protocol

```
1. Move proposed by System Orchestrator (or human operator)
2. Authority Engine checks:
   a. Proposer has sufficient AV on relevant dimensions
   b. All pre-conditions for the move type are satisfied
   c. ProofPerl evaluates: does the move violate any locked STK invariants?
3. If all checks pass:
   a. Move is executed atomically (all-or-nothing)
   b. CIG is updated (new edges, updated node properties)
   c. Move is logged on-chain as a governance event
   d. All affected devices receive the update via STF fabric sync
4. If any check fails:
   a. Move is rejected
   b. Rejection reason is logged
   c. Proposer can modify and re-submit
```

### Constraints

- **No move can weaken a locked STK invariant** - this is the fundamental safety guarantee. Once an invariant is locked, the lattice cannot evolve in ways that violate it.
- **Stage activation (Rotate.Stage) is irreversible** - once a stage is active, its cubelets cannot be de-activated. They can be individually disabled (AV set to 0) but the stage itself remains.
- **Every move is logged on-chain** with full reasoning, pre/post-condition checks, and the identity of the proposer.
- **Moves are the mechanism for Kusari activation** - enabling Stages 5-9 is a series of Rotate.Stage moves, each requiring its own pre-conditions and approvals.

> _Full details: Rubik's Move specification in CIG document (08-cubeletinteragraph in cubeletuni)_

---

## Kusari Evolution

MYOS is designed to evolve into **Kusari** - a Layer 1/0 modular decentralized architecture solution (dArch). The lattice makes the evolution path concrete:

| MYOS Phase | Lattice Activation | Kusari Equivalent |
| ---------- | ------------------ | ----------------- |
| **MYOS Core** (Stages 0-4) | Identity, Governance, Data, AI, Sensors | Foundational infrastructure - runs on centralized + edge nodes |
| **Kusari Phase 1** (Stage 5) | Assets/RWA | Asset tokenization, Kusari KVMs, economic primitives |
| **Kusari Phase 2** (Stages 6-7) | Economy + Social Intelligence | Treasury, governance DAOs, knowledge commons |
| **Kusari Phase 3** (Stages 8-9) | Environment + Meta-Ethics | Planetary stewardship, autonomous ethical governance |

Additional connection points:

| MYOS Concept                           | Kusari Equivalent                                           |
| -------------------------------------- | ----------------------------------------------------------- |
| Domain Orchestrators                   | Inspired by Kusari KVMs (not 1:1 mapping)                   |
| Domain DSLs (Healthcare, DeFi, Carbon) | Kusari Domain-Specific Languages                            |
| Ouroboros chain                        | Evolves into Kusari consensus via Consensus Adapter pattern |
| Node types (core, compute, edge)       | Maps to Kusari Supreme, Master, Archival, Lite nodes        |
| Cubelet compute marketplace            | Kusari resource marketplace (GPU, CPU, storage)             |
| STF fabric threads                     | Kusari interoperable ledgers                                |

The **Consensus Adapter pattern** ensures MYOS can switch consensus mechanisms without changing upper layers. Today it runs Ouroboros with Cardano mainnet anchoring. Tomorrow it can plug into Kusari's native consensus.

> _Kusari overview: [kusari.md](../../kusari.md)_
> _Consensus adapter: [06-verification-audit.md](06-verification-audit.md)_

---

## Session Pods - AI Interaction Model

When a user starts a conversation with MYOS, the system assembles a **Session Pod** - a long-lived pod that persists for the session:

```
SESSION POD (long-lived)
┌─────────────────────────────────────────────┐
│  Session Orchestrator (Pod Orch role, LLM)  │
│  Context Manager cubelet (conversation state)│
│  Safety Monitor cubelet (output filtering)   │
│  Basic LLM cubelet (simple queries)          │
└─────────────────────────────────────────────┘
```

**Simple queries** are handled directly within the Session Pod. **Complex queries** trigger sub-pod spawning:

```
User: "Compare financial performance of Company A vs B and predict trends"

Session Pod spawns (fan-out, parallel):
  Sub-Pod 1: Financial data retrieval (Company A)
  Sub-Pod 2: Financial data retrieval (Company B)
  Sub-Pod 3: Trend prediction model

All run in parallel → Session Pod collects results → LLM synthesis → Safety Monitor → User
```

Sub-pods **cannot nest** (no sub-sub-pods). If a sub-pod needs more capability, it reports back to the Session Pod, which spawns another sub-pod.

**Idle sessions** use tiered release: specialized cubelets are released immediately, core cubelets (context, safety) are kept until session timeout.

> _Full details: [01-authority-model.md](01-authority-model.md) Section 11, [03-pod-assembly.md](03-pod-assembly.md) Section 10_

---

## Architecture Document Index

| Doc                                                                                          | Title                                                 | Scope                                                                                                 |
| -------------------------------------------------------------------------------------------- | ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| **[00-MYOS-master.md](00-MYOS-master.md)**                                                   | This document                                         | System overview, concept explainer                                                                    |
| **[01-authority-model.md](01-authority-model.md)**                                           | Math-Based Access Control (MBAC)                      | AV vectors, flexible dimensions, reinforcement, floors/ceilings, Haskell engine, AI interaction model |
| **[02-conflict-resolution.md](02-conflict-resolution.md)**                                   | Escalation Chain & AV-Weighted Voting                 | 4-level escalation, voting quorum, timeouts, human-in-the-loop                                        |
| **[03-pod-assembly.md](03-pod-assembly.md)**                                                 | Task Decomposition, Cubelet Selection & Pod Formation | CIG-informed assembly, STSol templates, cubelet scoring, session pods                                |
| **[04-cubelet-io-contract.md](04-cubelet-io-contract.md)**                                   | Data Pipeline & Control Messages                      | Typed DAG (STA→STI→STD→STF→STK flow), control messages, two-tier logging                            |
| **[05-pod-orchestrator.md](05-pod-orchestrator.md)**                                         | Pod Orchestrator Behavior & Lifecycle                 | Always LLM, template+registry, DAG construction, tiered failure response                              |
| **[06-verification-audit.md](06-verification-audit.md)**                                     | On-Chain Decisions & Off-Chain Data                   | Ouroboros chain, STF fabric routing, Cardano anchoring, Kusari evolution, consensus adapter           |
| **[07-boot-trust-chain.md](07-boot-trust-chain.md)**                                         | Power-On to Operational State                         | Secure boot, key hierarchy, NixOS kernel, CIG initialization, service startup                        |
| **[08-runtime-infrastructure.md](08-runtime-infrastructure.md)**                             | Kernel, Services & Resource Management                | RT scheduling, three isolation tiers, IPC, watchdog                                                   |
| **[09-communication-networking.md](09-communication-networking.md)**                         | Device-to-Device & P2P Messaging                      | P2P mesh, mutual auth, post-quantum crypto, STF fabric channels, Federated MCP                       |
| **[10-node-topology-orchestrator-hierarchy.md](10-node-topology-orchestrator-hierarchy.md)** | Infrastructure, Hardware & LLM Assignment             | Node types, hardware map, 4-level orchestrator hierarchy, NixOS, Unikraft, LLM cost strategy          |
| **[11-repository-mapping.md](11-repository-mapping.md)**                                     | Existing Repos to Architecture Layers                 | 38 repos mapped, integration priority, dependency graph, new components needed                        |
| **[12-knowledge-base.md](12-knowledge-base.md)**                                             | Hierarchical Knowledge System                         | CIG backbone, confidence scoring, authority-governed writes, vector+graph storage                     |

---

## Key Design Decisions (Summary)

| Decision                  | Choice                                                       | Rationale                                                                 |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------- |
| Cubelet structure         | 10×5×15 Rubik's Lattice (750 named cubelets)                | Domain decomposition across stages and frameworks, not arbitrary pool     |
| Three model types         | Math-bound (~350), ML/DL (~250), LLM (~100)                 | Majority runs in strongest isolation; LLMs only where reasoning needed   |
| CIG graph backbone        | Neo4j with 9 edge types, pre-seeded                          | Pod assembly follows graph, not blind scoring                            |
| STK kernel invariants     | 150 invariants across 7 families                             | Formal ethical/safety constraints, not configurable away                  |
| PCCSCEFVRI ethical anchor | 10-value ethical foundation across all STK invariants        | Explicit objective function for the RL loop                              |
| STF fabric threads        | 11+ named ledgers per domain                                 | Domain-specific routing for verification and KB                          |
| Staged Kusari activation  | Stages 0-4 MYOS core, 5-9 activate with Kusari              | Progressive decentralization, not a rewrite                              |
| Access control model      | MBAC (math-based) over RBAC                                  | No stale privileges, earned trust, self-correcting                        |
| Authority dimensions      | Flexible registry, expanded by PCCSCEFVRI values             | Deployment-configurable (edge robotics, AI platform, hybrid)              |
| Cubelet exclusivity       | All exclusive (no sharing)                                   | Eliminates race conditions, simplifies reasoning                          |
| Cubelet identity          | Lattice position permanent, model versioned                  | AV resets on model change; system re-earns trust                         |
| Orchestrator intelligence | Always LLM (all levels)                                      | Need reasoning for DAG construction, synthesis, failure handling          |
| Orchestrator hierarchy    | 4 levels (System → Domain + Deploy → Pod)                    | Domain autonomy + zone management + ephemeral tasks                       |
| LLM cost strategy         | Paid for System+Domain, open-source for Deploy+Pod           | High-stakes decisions get quality, operational tasks get cost efficiency  |
| Conflict resolution       | AV comparison → weighted vote → System Orch → human          | 70/20/8/2% distribution, CIG-informed at Level 3                         |
| Cubelet isolation         | Wasm+Unikernel for math/ML, containers for LLM               | Double isolation for deterministic cubelets, debuggability for LLM agents |
| Chain consensus           | Ouroboros (Haskell) + Cardano anchoring                      | Proven PoS, evolves into Kusari via consensus adapter                     |
| Build system              | NixOS everywhere                                             | Bit-for-bit reproducible builds, declarative, atomic rollback             |
| OS base                   | NixOS (not bare Linux)                                       | Replaces Buildroot/Yocto, gains declarative config, unlimited rollback    |
| Kernel                    | Linux PREEMPT_RT                                             | Proven RT, RISC-V support, pragmatic over custom kernel                   |
| Core hardware             | x86/ARM today, RISC-V future                                 | Practical (GPU ecosystem), aspirational (open ISA)                        |
| Edge hardware             | Always RISC-V                                                | Low power, open ISA, sensor-grade SoCs                                    |
| Authority Engine location | Centralized (core server)                                    | Haskell on x86, no RISC-V GHC risk, single source of truth                |
| Authority Engine language | Haskell                                                      | Purity, type safety, formal verification proximity                        |
| Logs accessibility        | All public                                                   | Transparency is fundamental - hiding audit data defeats the purpose       |
| Knowledge base            | Hierarchical (system + domain + pod) on CIG backbone         | Structural graph + runtime knowledge, not empty-start                     |
| KB contradictions         | Same 4-level conflict resolution chain                       | Consistent conflict handling across cubelets and knowledge                |
| KB storage                | Vector DB + Knowledge Graph (shared with CIG)                | Semantic search + structural relationships, queried in parallel           |
| Unverified output         | Tiered labels (verified / partial / unverified)              | Transparent about certainty, users know what's grounded                   |
| Human in authority model  | Outside reinforcement loop                                   | Human opinions don't change AV - prevents optimizing for human approval   |
| Session model             | Hybrid session pods + sub-pods                               | Long-lived sessions with ephemeral sub-task teams, no nesting             |

---

## What Makes MYOS Different?

**From traditional operating systems:**
MYOS is not a desktop or server OS. It's a distributed AI execution platform with built-in authority, consensus, and audit. It treats AI as a first-class citizen with mathematically bounded autonomy. Its 750 cubelets form a structured lattice spanning from identity to meta-ethics - not a flat process pool.

**From other blockchain platforms:**
MYOS is not primarily a blockchain. The chain is a verification layer, not the execution layer. AI workloads run in pods with real-time scheduling, not in smart contracts. STF fabric threads provide domain-specific ledger infrastructure that evolves into Kusari's interoperable ledger ecosystem.

**From AI orchestration frameworks:**
MYOS goes below the application layer. It controls the kernel, the boot chain, the hardware, and the build system. Authority is enforced at every layer, not just at the API level. The Cubelet Interaction Graph provides structural relationships - not just task routing - enabling graph-informed assembly and domain-aware conflict resolution.

**From RBAC/traditional access control:**
MBAC is continuous, not discrete. Authority is a living score that evolves with performance. There are no "admin" or "user" roles - only authority vectors earned through proven competence. STK kernel invariants provide formal ethical constraints that no entity can bypass, anchored to the PCCSCEFVRI value system.

**From conventional middleware:**
MYOS is not middleware sitting on top of an OS. The lattice structure - 5 frameworks providing intent, logic, execution, connectivity, and proof for each of 10 domain stages - gives MYOS the formal structure of an operating system: STK is the kernel, STD is userspace, STF is the network stack, STI is the type system, STA is access control policy. The Rubik's Lattice is the contribution - no existing system has this structure.

---

_MYOS aims to become the execution integrity layer for autonomous AI systems - deterministic, secure, authority-bounded, and auditable. Every intent becomes logic; every logic becomes verifiable action; every action yields proof; every proof feeds back into reflection._
