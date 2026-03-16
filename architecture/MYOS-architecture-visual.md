# MYOS - Visual Architecture Reference

## Deterministic AI Operating System

---

## 1. System Overview

MYOS is a **distributed operating system** where AI systems operate under mathematically enforced authority, verifiable decision-making, and reproducible builds. It spans centralized servers, decentralized compute nodes, chain validators, and edge sensors - all governed by a single authority model.

```
                    ┌─────────────────────────────────────┐
                    │          FIVE PRINCIPLES             │
                    ├─────────────────────────────────────┤
                    │  1. Deterministic Execution          │
                    │  2. Hardware-Rooted Trust             │
                    │  3. Authority-Controlled Actions      │
                    │  4. Minimal Attack Surface            │
                    │  5. Verifiable Operation              │
                    └─────────────────────────────────────┘
```

### OS Analogy

| Traditional OS          | MYOS Equivalent                                    |
| ----------------------- | -------------------------------------------------- |
| Process                 | Cubelet (named position in Rubik's Lattice)        |
| Job / Process Group     | Pod (STSol assembly - vertical lattice cross-section) |
| Permission / Capability | Authority Value (AV) + STK invariants (dual gate)  |
| Filesystem              | Knowledge Base (on CIG backbone)                   |
| Audit Log               | On-chain / Off-chain via STF fabric threads        |
| Kernel                  | NixOS + PREEMPT_RT + Authority Engine + ProofPerl  |
| Shell                   | Session Pod (AI interaction)                       |
| Type System             | STI framework (logic, ontologies, schemas)         |
| Network Stack           | STF framework (fabric threads, ledger connectors)  |
| Access Policy           | STA framework (intent, ethics, charters)           |

---

## 2. Full System Stack

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│   USER / EXTERNAL SYSTEMS                                                       │
│   (Web API, CLI, Federated MCP, External Chains)                                │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ORCHESTRATION LAYER (4 levels, all LLM-powered)                               │
│                                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                                                                         │   │
│   │   ┌───────────────────────────────────────────────────┐                │   │
│   │   │  SYSTEM ORCHESTRATOR (1, singleton)                │                │   │
│   │   │  Paid LLM (Claude Opus class) | AV Floor: 700     │                │   │
│   │   │  Core server (x86/ARM)                             │                │   │
│   │   └──────────────┬──────────────────┬─────────────────┘                │   │
│   │                  │                  │                                    │   │
│   │        ┌─────────▼──────┐  ┌────────▼────────┐                         │   │
│   │        │ DOMAIN ORCHS   │  │ DEPLOYMENT ORCHS │                         │   │
│   │        │ (per vertical) │  │ (per zone)       │                         │   │
│   │        │ Paid LLM       │  │ Open-source LLM  │                         │   │
│   │        │ AV Floor: 600  │  │ AV Floor: 500    │                         │   │
│   │        │ Semi-autonomous│  │ Infra management │                         │   │
│   │        └───────┬────────┘  └────────┬─────────┘                         │   │
│   │                │   PARALLEL PEERS    │                                   │   │
│   │        ┌───────▼────────────────────▼─────────┐                         │   │
│   │        │      POD ORCHESTRATORS (many)         │                         │   │
│   │        │      Open-source LLM | AV Floor: 400  │                         │   │
│   │        │      Ephemeral, per-task               │                         │   │
│   │        └───────────────────────────────────────┘                         │   │
│   │                                                                         │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│   EXECUTION LAYER - RUBIK'S LATTICE (750 cubelets = 10 stages × 5 frameworks × 15) │
│                                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                                                                         │   │
│   │   THE LATTICE (each cell = 15 cubelets):                                │   │
│   │            STA         STI         STD         STF         STK          │   │
│   │          (Intent)    (Logic)    (Execute)   (Connect)    (Proof)        │   │
│   │   St 0-3  Policies    ML/DL      Runtimes   Ledgers    Invariants     │   │
│   │   St 4-5  Policies    ML/DL      Runtimes   DePIN      Proofs         │   │
│   │   St 6-7  Policies    ML/DL      Runtimes   Treasury   Proofs         │   │
│   │   St 8-9  Policies    ML/DL      Runtimes   Eco/Ethics Proofs         │   │
│   │                                                                         │   │
│   │   THREE MODEL TYPES → THREE ISOLATION TIERS:                            │   │
│   │   ┌─────────────────┐ ┌──────────────────┐ ┌──────────────────────┐    │   │
│   │   │ Math-Bound       │ │ ML/DL Models      │ │ LLM Agents           │    │   │
│   │   │ ~350-400 cubelets│ │ ~200-250 cubelets │ │ ~100-150 cubelets    │    │   │
│   │   │ ─────────────── │ │ ──────────────── │ │ ─────────────────── │    │   │
│   │   │ TIER 1: Wasm +   │ │ TIER 1-2: Wasm    │ │ TIER 3: Linux crun   │    │   │
│   │   │ Unikraft         │ │ (± Unikernel)     │ │ Container            │    │   │
│   │   │ STK, STF cubelets│ │ STI cubelets      │ │ Some STA/STD         │    │   │
│   │   │ ~12ms boot       │ │ ~1ms boot         │ │ ~500ms boot          │    │   │
│   │   │ ~20MB memory     │ │ ~5MB memory       │ │ ~100MB+ memory       │    │   │
│   │   │ Double isolation │ │ Single isolation  │ │ Namespace isolation  │    │   │
│   │   │ Perfect determ.  │ │ Near-perfect      │ │ Non-deterministic    │    │   │
│   │   └─────────────────┘ └──────────────────┘ └──────────────────────┘    │   │
│   │                                                                         │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│   RUNTIME LAYER                                                                 │
│                                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │  NixOS (PREEMPT_RT kernel)                                              │   │
│   │  ┌──────────┐ ┌─────┐ ┌──────────────┐ ┌──────────┐ ┌───────────┐     │   │
│   │  │ RT Sched │ │ IPC │ │ Resource Mgr │ │ Watchdog │ │ Agenix    │     │   │
│   │  │ FIFO     │ │ UDS │ │ cgroups v2   │ │ HW + SW  │ │ Secrets   │     │   │
│   │  └──────────┘ └─────┘ └──────────────┘ └──────────┘ └───────────┘     │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│   HARDWARE LAYER                                                                │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │  x86/ARM (core, compute GPU, validators)                                │   │
│   │  RISC-V  (edge sensors, general compute)                                │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   CROSS-CUTTING SERVICES                                                        │
│                                                                                 │
│   ┌─────────────────────┐  ┌──────────────────────────────────────────────┐    │
│   │  AUTHORITY ENGINE   │  │  VERIFICATION & AUDIT                        │    │
│   │  Haskell (MBAC)     │  │  Ouroboros chain + off-chain logs            │    │
│   │  + ProofPerl (STK   │  │  Routed via STF fabric threads              │    │
│   │    invariant eval)  │  │  Cardano mainnet anchoring                   │    │
│   │  + PerlFrame (value │  │  STK invariant records on-chain             │    │
│   │    anchor, PCCSCEFVRI)│  │  Public readability                        │    │
│   │  Centralized core   │  │  Tiered retention (hot/warm/cold)            │    │
│   │  <5ms response      │  │                                              │    │
│   └─────────────────────┘  └──────────────────────────────────────────────┘    │
│                                                                                 │
│   ┌─────────────────────┐  ┌──────────────────────────────────────────────┐    │
│   │  KNOWLEDGE BASE     │  │  CUBELET INTERACTION GRAPH (CIG)             │    │
│   │  On CIG backbone    │  │  Neo4j: 750 nodes, 9 edge types, 1500+ edges│    │
│   │  3-level hierarchy  │  │  Structural backbone for pods + KB           │    │
│   │  Domain KBs = STF   │  │  Pre-seeded at boot from cubelet registry   │    │
│   │    fabric threads   │  │  Rubik's Moves update the graph             │    │
│   │  Vector DB + Graph  │  │                                              │    │
│   │  Confidence scoring │  │  NETWORKING                                  │    │
│   │  GDPR erasure       │  │  LibP2P P2P mesh (qudag-network)            │    │
│   └─────────────────────┘  │  Post-quantum crypto, STF multiplexed       │    │
│                              │  Federated MCP cross-device tools           │    │
│                              └──────────────────────────────────────────────┘    │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Authority Model (MBAC)

Math-Based Access Control replaces traditional RBAC. Every entity carries a continuously computed, multi-dimensional Authority Value vector.

```
AUTHORITY VALUE STRUCTURE:

    Cubelet_LLM_Agent_07 = {
        execution:      650.0  ████████████████████░░░░░░░░░░  65%
        safety:         880.2  ████████████████████████████░░  88%
        reasoning:      920.5  █████████████████████████████░  92%
        communication:  710.3  ██████████████████████░░░░░░░░  71%
        knowledge:      540.0  █████████████████░░░░░░░░░░░░░  54%
        autonomy:       320.8  ██████████░░░░░░░░░░░░░░░░░░░░  32%
    }

                                0          500         1000
                                |           |           |
                                └─ Floor    └─ Mid      └─ Ceiling


AUTHORITY DYNAMICS:

    ┌──────────────┐     Diminishing returns curve
    │              │
    │   Rewards    │     effective_reward = base × (1 - AV/max)^k
    │   (+AV)      │
    │  ╭──────╮    │     At AV=100: reward ≈ 45    (nearly full)
    │  │      ╰──  │     At AV=500: reward ≈ 25    (half)
    │  │           │     At AV=900: reward ≈  5    (tiny increment)
    └──┼───────────┘
       │
    ┌──┼───────────┐
    │  │           │     Constant penalties
    │  │  ──────── │
    │  │           │     penalty = base_penalty (always the same)
    │  Penalties   │
    │  (-AV)       │     At AV=900: -100
    │              │     At AV=100: -100 (same cost)
    └──────────────┘


HIERARCHICAL FLOORS (immutable invariant):

    System Orch  ████████████████████████████████████  700
    Domain Orch  ██████████████████████████████████    600
    Deploy Orch  ████████████████████████████████      500
    Pod Orch     ██████████████████████████            400
    Cubelet      ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░    0

    floor(System) > floor(Domain) > floor(Deploy) > floor(Pod) > floor(Cubelet)
    This guarantees the hierarchy is ALWAYS maintained, even under worst-case penalties.
```

---

## 4. Pod Lifecycle

```
TASK ARRIVAL → POD ASSEMBLY → EXECUTION → DISSOLUTION

    ┌──────────────┐
    │  Task Arrives │
    │  at System    │
    │  Orchestrator │
    └──────┬───────┘
           │
    ┌──────▼───────────────────────────┐
    │  THREE-LAYER DEDUPLICATION       │
    │  1. Structural: non-overlapping  │
    │  2. Content: hash-based          │
    │  3. Runtime: registry lock       │
    └──────┬───────────────────────────┘
           │
    ┌──────▼──────────────────────────────────────────────┐
    │  CUBELET SELECTION (graph-informed)                   │
    │                                                      │
    │  Phase 1: STSol template → CIG edge resolution       │
    │    Task → Stage mapping → STSol template lookup       │
    │    DEPENDS_ON, PROVES, GOVERNS edges → required slots │
    │                                                      │
    │  Phase 2: AV + compatibility scoring                 │
    │    score = 0.50 × AV[relevant_dim]                   │
    │          + 0.25 × compatibility_bonus                │
    │          + 0.15 × success_rate                       │
    │          - 0.10 × recent_load                        │
    │                                                      │
    │  Assembly mode:                                      │
    │    HIGH priority → PARALLEL (lock immediately)       │
    │    LOW priority  → SEQUENTIAL (lock at end)          │
    └──────┬──────────────────────────────────────────────┘
           │
    ┌──────▼──────────────────────────────────────────────┐
    │  POD INSTANTIATION                                   │
    │                                                      │
    │  ┌──────────────────────────────────────────────┐   │
    │  │  POD                                          │   │
    │  │  ┌────────────────────────────────────────┐   │   │
    │  │  │  Pod Orchestrator (LLM, builds DAG)    │   │   │
    │  │  │  Uses STSol template + CIG edges       │   │   │
    │  │  └────────────────────────────────────────┘   │   │
    │  │       │  Framework ordering: STA→STI→STD→STF→STK  │
    │  │  ┌────▼─────┐  ┌──────────┐  ┌───────────┐  │   │
    │  │  │STA (gov) │→ │STI (logic)│→ │STD (exec) │  │   │
    │  │  └──────────┘  └──────────┘  └─────┬─────┘  │   │
    │  │                               ┌────▼─────┐  │   │
    │  │                               │STF (log) │  │   │
    │  │                               └────┬─────┘  │   │
    │  │                               ┌────▼─────┐  │   │
    │  │                               │STK (prove)│  │   │
    │  │                               └──────────┘  │   │
    │  │       Data Pipeline (typed DAG, CIG-constrained) │
    │  └──────────────────────────────────────────────┘   │
    └──────┬──────────────────────────────────────────────┘
           │
    ┌──────▼──────────────────────────────────────────────┐
    │  DISSOLUTION                                         │
    │  → AV rewards/penalties for each cubelet             │
    │  → Compatibility scores updated                      │
    │  → Cubelets return to pool (cooldown → available)    │
    │  → Pod KB discoveries promoted to Domain KB          │
    │  → Full audit trail logged                           │
    └─────────────────────────────────────────────────────┘
```

---

## 5. Data Pipeline & Control Messages

```
TWO COMMUNICATION CHANNELS:

    DATA PIPELINE (typed DAG - the work product):

        [Sensor] ──Image──→ [Detector] ──Objects──→ [Planner] ──Commands──→ [Motor]
                                  │
                                  └──Stats──→ [Logger]

        - Every edge is type-checked at construction
        - Zero-copy shared memory (local)
        - Serialized + encrypted (cross-device)
        - All edge transfers hashed (off-chain)


    CONTROL MESSAGES (flexible side channel):

        Cubelet ──status_report──→ Pod Orch
        Cubelet ──alert──────────→ Pod Orch
        Pod Orch ──config_update──→ Cubelet
        Pod Orch ──escalation────→ System Orch

        - Authority-gated (config/bypass require AV check)
        - Status/error messages are NEVER gated


TIERED FAILURE RESPONSE:

    ┌─────────────────────────────────────────────────────┐
    │ Tier 1: RETRY                                        │
    │   Cubelet fails → retry with same input (max 2x)    │
    │   No AV penalty                                      │
    ├─────────────────────────────────────────────────────┤
    │ Tier 2: QUARANTINE + COMPENSATE                      │
    │   Retry exhausted → isolate cubelet, find replacement│
    │   AV penalty for quarantined cubelet                 │
    ├─────────────────────────────────────────────────────┤
    │ Tier 3: ADAPTIVE RESTRUCTURE                         │
    │   No replacement → Pod Orch rewires the DAG          │
    │   Exception to "fixed DAG" rule, logged on-chain     │
    ├─────────────────────────────────────────────────────┤
    │ Tier 4: ESCALATE                                     │
    │   Restructure fails → report to System Orch          │
    │   System Orch: dissolve + reassemble, or escalate    │
    └─────────────────────────────────────────────────────┘
```

---

## 6. Conflict Resolution Chain

```
FOUR-LEVEL ESCALATION:

    ~70% of conflicts          ~20%              ~8%            ~2%
    ┌─────────────┐      ┌─────────────┐   ┌──────────┐   ┌──────────┐
    │  LEVEL 1    │      │  LEVEL 2    │   │ LEVEL 3  │   │ LEVEL 4  │
    │  Direct AV  │─────→│  AV-Weighted│──→│ System   │──→│ Human    │
    │  Comparison │ tie/ │  Pod Vote   │low│ Orch     │low│ in the   │
    │             │esc.  │             │cnf│ Arbitrate│cnf│ Loop     │
    │  100ms      │      │  500ms      │   │ 2000ms   │   │ config.  │
    └─────────────┘      └─────────────┘   └──────────┘   └──────────┘
                                                                │
                                                           On timeout:
                                                           SAFEST OPTION
                                                           (no action if
                                                            unclear)

    HUMAN POWERS:
    ┌────────────────────────────────────────────────────────────┐
    │  VOTE:  Participates with max AV weight (1000)             │
    │         Can still be outvoted by collective system          │
    │                                                            │
    │  VETO:  Safety-critical ONLY - overrides all voting math   │
    │         Logged as human_override, does NOT affect any AV   │
    │                                                            │
    │  DELEGATE: Returns decision to System Orch with guidance   │
    └────────────────────────────────────────────────────────────┘

    INVARIANT: Human decisions NEVER affect entity AV.
               The reinforcement model is purely performance-based.
```

---

## 7. Knowledge Base Hierarchy

```
HIERARCHICAL KB WITH CONFIDENCE SCORING:

    ┌──────────────────────────────────────────────────────────────┐
    │  SYSTEM KB (global, permanent)                                │
    │  Cross-domain facts, operational rules                       │
    │  Write: knowledge AV >= 800                                  │
    │  Location: Core server                                       │
    ├──────────────────────────────────────────────────────────────┤
    │                                                              │
    │  ┌────────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │
    │  │ Knowledge  │ │ Finance  │ │ Karbon   │ │ Research │     │
    │  │ Ledger KB  │ │ Ledger KB│ │ Ledger KB│ │ LedgerKB │     │
    │  │ Write:>=500│ │          │ │          │ │          │     │
    │  └────────────┘ └──────────┘ └──────────┘ └──────────┘     │
    │  Mapped to STF fabric threads - co-located with ledger infra │
    │  Isolated from each other - cross-domain via System KB       │
    │  Location: Domain servers                                    │
    ├──────────────────────────────────────────────────────────────┤
    │  Pod KBs (ephemeral, dissolved with pod)                     │
    │  Write: knowledge AV >= 100                                  │
    │  Location: Compute nodes                                     │
    └──────────────────────────────────────────────────────────────┘

    Knowledge flows DOWN ▼    Discoveries flow UP ▲


AGENT QUERY PROTOCOL (mandatory, before every response):

    ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
    │ 1. EXACT LOOKUP │     │ 2. SEMANTIC      │     │ 3. LLM FALLBACK │
    │    (KB search)  │────→│    SEARCH (RAG)  │────→│    (training     │
    │                 │miss │                  │miss │     data only)   │
    │  VERIFIED       │     │  PARTIALLY       │     │  UNVERIFIED      │
    │  (conf >= 800)  │     │  VERIFIED        │     │                  │
    └─────────────────┘     └─────────────────┘     └─────────────────┘

    OUTPUT CHECK: If LLM output contradicts a VERIFIED KB entry → BLOCKED.
                  Contradiction enters the 4-level conflict resolution chain.


CONFIDENCE SCORING (same math as Authority Values):

    ┌──────────────────────────────────────────────────────────────┐
    │  Confirmation:   +reward (diminishing returns near ceiling)  │
    │  Contradiction:  -penalty (constant, always hurts the same)  │
    │  Decay:          confidence erodes if not re-confirmed       │
    │                                                              │
    │  VERIFIED          >= 800, >= 3 confirmations                │
    │  PARTIALLY VERIFIED  300–799                                 │
    │  UNVERIFIED        < 300                                     │
    └──────────────────────────────────────────────────────────────┘

    Storage: Vector DB (Qdrant) + Knowledge Graph (Neo4j), queried in parallel.
    Metadata on-chain (tamper-proof). Content off-chain (deletable for GDPR).
```

---

## 8. Physical Topology

```
                    ┌─────────────────────────────────────┐
                    │     CENTRALIZED CORE (x86/ARM)      │
                    │     System Orch + Authority Engine   │
                    │     CIG (Neo4j) + System KB          │
                    │     NixOS full                       │
                    └────────────────┬────────────────────┘
                                     │
            ┌────────────────────────┼────────────────────────┐
            │                        │                        │
   ┌────────▼────────┐    ┌─────────▼────────┐    ┌──────────▼──────────┐
   │ Domain Server   │    │ Domain Server    │    │ Deployment Node     │
   │ Healthcare      │    │ DeFi            │    │ Zone-1              │
   │ x86 or RISC-V   │    │ x86 (GPU)       │    │ Same arch as zone   │
   │ Paid LLM        │    │ Paid LLM        │    │ Open-source LLM     │
   │ NixOS full      │    │ NixOS full      │    │ NixOS full          │
   └────────┬────────┘    └─────────┬────────┘    └──────────┬──────────┘
            │                       │                        │
   ┌────────▼───────────────────────▼────────────────────────▼──────────┐
   │                    CUBELET COMPUTE NODES                            │
   │                    NixOS full                                      │
   │                                                                    │
   │  LLM Cubelets         ML/DL Cubelets        Math Cubelets          │
   │  ──────────────       ──────────────        ──────────────         │
   │  x86 GPU only         x86 or RISC-V        x86 or RISC-V          │
   │  Tier 3: container    Tier 1: wasm+unikernel   Tier 1 or 2         │
   └────────────────────────────────────────────────────────────────────┘

   ┌────────────────────────────────────────────────────────────────────┐
   │                 CHAIN VALIDATORS                                    │
   │                 x86/ARM dedicated, NixOS full                      │
   │                 Ouroboros consensus, block production               │
   └────────────────────────────────────────────────────────────────────┘

   ┌────────────────────────────────────────────────────────────────────┐
   │               EDGE RISC-V SENSORS (Stage 0, 4 cubelets)            │
   │               NixOS minimal                                        │
   │               Math-bound + quantized ML cubelets (Tier 1-2 only)  │
   │               No LLMs on edge - by design                         │
   │               Authority via RPC to centralized engine              │
   └────────────────────────────────────────────────────────────────────┘
```

---

## 9. Boot & Trust Chain

```
POWER ON → OPERATIONAL (each stage verifies the next):

    ┌──────────────────────────────────────────────────────────────┐
    │  STAGE 0: SoC ROM Bootloader (immutable, burned in silicon) │
    │  Verifies Stage 1 against SoC-burned public key             │
    │  FAIL → HALT                                                │
    └──────────────────────────┬───────────────────────────────────┘
                               │
    ┌──────────────────────────▼───────────────────────────────────┐
    │  STAGE 1: Firmware Verifier (signed, in secure flash)       │
    │  Verifies kernel signature + hash                           │
    │  FAIL → HALT                                                │
    └──────────────────────────┬───────────────────────────────────┘
                               │
    ┌──────────────────────────▼───────────────────────────────────┐
    │  STAGE 2: Linux Kernel (PREEMPT_RT)                         │
    │  Verifies device tree + initrd                              │
    │  Initializes RT scheduler, memory management                │
    └──────────────────────────┬───────────────────────────────────┘
                               │
    ┌──────────────────────────▼───────────────────────────────────┐
    │  STAGE 3: Rust Userspace Init                               │
    │  1. HAL init          6. Authority Engine + ProofPerl       │
    │  2. Config verify     7. CIG load (750 nodes → Neo4j)      │
    │  3. Wasm runtime      8. STK invariant defs verified        │
    │  4. Container runtime 9. Off-chain DB + Qdrant              │
    │  4b. Agenix secrets   10. Chain node (Ouroboros)            │
    │      (decrypt to      11. Network + ANS                     │
    │       tmpfs only)     12. System Orchestrator               │
    │  5. Cubelet registry  13. Boot attestation (on-chain)       │
    │     load              14. SYSTEM READY                       │
    └─────────────────────────────────────────────────────────────┘


KEY HIERARCHY:

    Level 0: SoC Root Key ─── burned in silicon, never exposed
                │
    Level 1: Firmware Key ─── verifies kernel, rotatable
                │
    Level 2: Device Identity Key ─── ML-KEM-768 + ML-DSA (post-quantum)
                │                    managed by qudag-vault-core
    Level 3: Session Keys ─── ephemeral, per-connection, rotated frequently


UPDATES:  NixOS generations (atomic, unlimited rollback)
          Not A/B - any previous generation is one command away
          Build new → switch → verify → rollback if failed
```

---

## 10. Runtime Infrastructure

```
CPU PARTITIONING (4-core example):

    Core 0                Core 1              Core 2              Core 3
    ┌──────────┐         ┌──────────┐        ┌──────────┐       ┌──────────┐
    │ Linux    │         │ P0 Safety│        │ P3 AI    │       │ P5 Comms │
    │ House-   │         │ Watchdog │        │ Inference│       │ Network  │
    │ keeping  │         │          │        │ Cubelets │       │ Chain    │
    │ IRQs     │         │ P1 Auth  │        │          │       │          │
    │ Non-RT   │         │ Engine   │        │ P4 Agent │       │ P6 Logs  │
    │          │         │ IPC      │        │ Runtime  │       │          │
    │          │         │          │        │ Pod Orch │       │ P7 BG    │
    └──────────┘         └──────────┘        └──────────┘       └──────────┘
    NOT used by           Dedicated           Time-shared         Time-shared
    MYOS workloads        pinned              SCHED_FIFO          SCHED_FIFO


PRIORITY MAP:

    P0 (99)  Safety Watchdog          cannot be preempted
    P1 (90)  Authority Engine IPC     <5ms response guaranteed
    P2 (80)  Safety Control Loops     100Hz cycle
    P3 (70)  AI Inference             cubelet execution
    P4 (60)  Agent Runtime            Pod Orch reasoning
    P5 (50)  Communication            network, chain
    P6 (40)  Monitoring & Logging     telemetry
    P7 (30)  Background               cleanup, migration


IPC:

    Cubelets ←──shared memory ring buffer──→ Pod Orch
    Pod Orch ←──shared memory ring buffer──→ System Orch
    Any      ←──Unix domain socket─────────→ Authority Engine
                    │
                    └── SO_PEERCRED kernel auth (UID/GID verified)
                        Socket owned by myos-authority group (0660)
                        NO TCP - local only, no remote exploitation

    Data pipeline: zero-copy shared memory (write once, read same address)
```

---

## 11. Verification & Audit

```
TWO-TIER LOGGING:

    ┌─────────────────────────────────────────────────────────────┐
    │  ON-CHAIN (Ouroboros, tamper-proof, permanent)               │
    │                                                             │
    │  - Authority decisions (allow/deny/escalate)                │
    │  - Conflict resolution outcomes (all 4 levels)              │
    │  - Human override events                                    │
    │  - Safety events (immediate, never batched)                 │
    │  - Pod lifecycle (create, dissolve, quarantine)             │
    │  - KB metadata (create, confirm, contradict, promote)       │
    │  - Boot attestation                                         │
    │                                                             │
    │  Block time: configurable (sub-second to seconds)           │
    │  Validators: MYOS devices (dedicated x86/ARM nodes)         │
    │  Anchoring: periodic merkle root → Cardano mainnet          │
    └─────────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────────┐
    │  OFF-CHAIN (event log + content-addressed, tiered storage)  │
    │                                                             │
    │  - Pipeline edge hashes and data                            │
    │  - Cubelet telemetry (timing, memory, CPU)                  │
    │  - Control messages                                         │
    │  - DAG construction logs                                    │
    │  - Pod Orch reasoning traces                                │
    │  - Session conversations                                    │
    │  - KB content (Vector DB + Graph DB)                        │
    │                                                             │
    │  Tamper-evident: hash chain + periodic merkle root on-chain │
    │                                                             │
    │  Tiered retention:                                          │
    │    HOT  (SSD/RAM)  : last 24h     - sub-ms queries          │
    │    WARM (SSD/HDD)  : last 30 days - ms queries              │
    │    COLD (object)   : 90 days–7yr  - seconds to retrieve     │
    └─────────────────────────────────────────────────────────────┘

    ALL LOGS ARE PUBLICLY READABLE.
    No authentication required for read access.
    Privacy handled by encrypting payload BEFORE entering the pipeline.


MODULAR CHAIN ARCHITECTURE (Kusari evolution path):

    ┌─────────────────────────────────────────────────┐
    │  CONSENSUS ADAPTER INTERFACE                     │
    │  validate_block() | propose_block()              │
    │  finality_check() | sync_state()                 │
    ├─────────────────────────────────────────────────┤
    │  Consensus Module [Haskell] - Ouroboros (today)  │
    │  Storage Module [Rust] - block store + merkle    │
    │  Network Module [Rust] - P2P + peer sync         │
    └─────────────────────────────────────────────────┘

    Each module replaceable independently.
    Tomorrow: swap consensus for Kusari hybrid PoS+BFT.
    Application code doesn't change - only the adapter.
```

---

## 12. Networking & Cross-Device Communication

```
P2P MESH NETWORK:

    ┌──────────┐     qudag-network      ┌──────────┐
    │ Core     │◄═══(LibP2P P2P)═══════►│ Domain   │
    │ Server   │    encrypted, mutual   │ Server   │
    │ Full Val │    auth (ML-DSA)       │ Full Val │
    └────┬─────┘                        └────┬─────┘
         │          ┌──────────┐             │
         │          │ Compute  │             │
         ├──────────┤ Node     ├─────────────┤
         │          └──────────┘             │
    ┌────┴─────┐  ┌──────────┐  ┌───────────┴┐
    │ Edge     │  │ Edge     │  │ Validator  │
    │ RISC-V   │  │ RISC-V   │  │ x86/ARM   │
    │ Light Val│  │ Light Val│  │ Full Val   │
    └──────────┘  └──────────┘  └────────────┘


ENCRYPTION (dual regime):

    Chain + Auth channels:  ML-KEM-768 (post-quantum) + AES-256-GCM
                            Long-lived secrets need quantum resistance

    Data + Pod channels:    X25519 (classical ECDH) + AES-256-GCM
                            Ephemeral data, speed > quantum resistance


CROSS-DEVICE AUTHORITY (dual authorization):

    Device A cubelet wants to use Device B cubelet:

    Device A Auth Engine: "Can my cubelet request this?"  → allow ──┐
    Device B Auth Engine: "Do I allow this remote entity?" → allow ─┤
                                                                    ▼
                                                              BOTH must
                                                              approve
```

---

## 13. Session Pods (AI Interaction)

```
USER STARTS A CONVERSATION:

    ┌─────────────────────────────────────────────────────────────┐
    │  SESSION POD (long-lived, persists for session)             │
    │                                                             │
    │  ┌──────────────────────────────────────────────────────┐  │
    │  │  Session Orchestrator (Pod Orch role, LLM)            │  │
    │  │  Context Manager cubelet (conversation state)         │  │
    │  │  Safety Monitor cubelet (output filtering)            │  │
    │  │  Basic LLM cubelet (handles simple queries)           │  │
    │  └──────────────────────────────────────────────────────┘  │
    │                                                             │
    │  On complex query (capabilities missing):                   │
    │                                                             │
    │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐       │
    │  │ Sub-Pod 1    │ │ Sub-Pod 2    │ │ Sub-Pod 3    │       │
    │  │ Data fetch A │ │ Data fetch B │ │ ML predict   │       │
    │  └──────────────┘ └──────────────┘ └──────────────┘       │
    │      PARALLEL, ephemeral, NO nesting (no sub-sub-pods)     │
    │                                                             │
    │  All results → Session Pod → Safety Monitor → User          │
    └─────────────────────────────────────────────────────────────┘


IDLE BEHAVIOR:

    Query completes → release sub-pod cubelets immediately
    Session idle    → Tier 1: release specialized cubelets
                      Tier 2: KEEP context, safety, basic LLM
    Session timeout → release ALL, archive context
    Session resume  → new Session Pod, restore archived context
```

---

## 13b. Context Continuity (Solving the Context Weight Gap)

The context weight gap - where an LLM loses coherence after long action chains - doesn't apply to MYOS because context is structural, not conversational. Cubelets see only typed inputs. The DAG carries data explicitly. The KB externalizes state. But the Pod Orchestrator (the one LLM with long context) is backed by four mechanisms:

```
WHY MYOS DOESN'T HAVE THE CONTEXT WEIGHT GAP:

    TRADITIONAL AGENT (one LLM, growing context window):

        [action 1] → [action 2] → ... → [action 50] → [action 51]
        ████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ← attention spread thin
        ^ early context          ^ middle fades        ^ recent overweighted

    MYOS (distributed, externalized context):

        Cubelets:      [A]──typed──→[B]──typed──→[C]    (no long context, ever)
        DAG:           structural data flow              (graphs don't forget)
        KB:            persistent, queryable, scored     (notebook, not memory)
        Orchestrator:  bounded scope + checkpointed      (backed up, reloadable)
```

```
DECISION CONTEXT LOGGING (parallel reasoning DAG):

    DATA DAG:       [Sensor] ──data──→ [Model] ──data──→ [Actuator]
                                            ↑
    DECISION DAG:   [build_dag] ──→ [bypass_model] ──→ [reroute_actuator]
                         │                │                    │
                    "why this          "why bypass:        "why reroute:
                     DAG shape"         model timeout"      no replacement"

    Every orchestrator decision that alters the pipeline is logged with:
    - What was decided + what alternatives were considered
    - Why this option (rationale)
    - What context informed it (KB entries, cubelet states, prior decisions)
    - What the expected outcome was
    - Links to prior decisions (forms a queryable reasoning chain)

    Stored in Pod KB. Promoted to Session Memory KB for session pods.
    When orchestrator needs to revisit a decision → reload reasoning, don't reconstruct.
```

```
CONTEXT WEIGHT SCORING (Session Memory KB entries):

    Every context entry carries a relevance weight:

    Entry Type        Base Weight
    ─────────────────────────────
    user_intent         2.0        (user goals are highest priority)
    constraint          1.8        (established rules matter)
    decision            1.5        (past decisions inform future)
    sub_task_result     1.2        (completed work)
    preference          1.0        (nice to have)
    observation         0.8        (transient)

    Weight evolves:
    ┌──────────────────────────────────────────────────────────────┐
    │  On reference (orchestrator reads entry):  weight *= 1.1    │
    │  Distance decay:  1 / (1 + 0.05 × actions_since_creation)  │
    │  Time decay:      1 / (1 + 0.01 × hours_since_referenced)  │
    │                                                              │
    │  current_weight = base × distance_decay × time_decay        │
    │                                                              │
    │  Context reload returns entries sorted by weight DESC.       │
    │  Most relevant first, stale last. Not a flat dump.           │
    └──────────────────────────────────────────────────────────────┘
```

```
CONTEXT CHECKPOINTING:

    Triggers:
    - Every N actions (configurable, default: 10)
    - Before spawning a sub-pod
    - Before pipeline restructure
    - When coherence self-check fires

    Checkpoint contains:
    ┌──────────────────────────────────────────────────────┐
    │  active_goals:          what are we trying to do     │
    │  established_constraints: rules from earlier         │
    │  key_decisions:         links to decision context    │
    │  pipeline_state:        current DAG snapshot         │
    │  open_questions:        unresolved items             │
    │  top_context:           top 20 entries by weight     │
    └──────────────────────────────────────────────────────┘

    On reload: latest checkpoint + top-weighted entries → rebuilt working state.
```

```
COHERENCE SELF-CHECK (Pod Orchestrator monitors its own degradation):

    Three signals:
    ┌─────────────────────────────────────────────────────────────────┐
    │  1. Decision consistency                                        │
    │     New decision contradicts prior decision without new info?   │
    │     → coherence drops                                           │
    │                                                                 │
    │  2. KB alignment                                                │
    │     Output conflicts with high-confidence KB entries?           │
    │     → coherence drops                                           │
    │                                                                 │
    │  3. Goal drift                                                  │
    │     Current action doesn't serve any active goal?               │
    │     → coherence drops                                           │
    └─────────────────────────────────────────────────────────────────┘

    Coherence score: 0.0 ─────────────── 0.4 ──── 0.7 ────── 1.0
                     │                    │         │          │
                     ESCALATE:            RELOAD:   WARNING:   NORMAL
                     dissolve + reassemble pause +   checkpoint
                     fresh orch from       reload    immediately
                     KB checkpoint         from KB

    Checked every 5 actions, after sub-pod synthesis, after restructures.
    Like a human realizing they've lost the plot and re-reading their notes.
```

```
SUB-POD CONTEXT INJECTION:

    Session Pod doesn't just send "do X" to sub-pods.
    It sends a structured context package:

    ┌──────────────────────────────────────────────────────────────┐
    │  sub_pod_context_package                                      │
    │                                                               │
    │  task_description:    "classify these images"                 │
    │  session_goals:       ["build health dashboard", ...]        │
    │  constraints:         ["use HIPAA-compliant models only"]    │
    │  decision_chain:      [why this sub-task was spawned]        │
    │  relevant_kb_entries: [pre-fetched knowledge]                │
    │  user_preferences:    [output format, style, ...]           │
    │                                                               │
    │  hard_constraints:    MUST NOT violate (like system invariants)│
    │  soft_constraints:    SHOULD follow (may deviate with reason) │
    │  contradiction_policy: report_to_parent (don't resolve locally)│
    └──────────────────────────────────────────────────────────────┘

    Sub-pod operates with full session awareness.
    If it finds something contradicting injected context → reports back.
    Session pod updates Session Memory KB if warranted.
```

---

## 14. NixOS Build System

```
SINGLE FLAKE - ENTIRE SYSTEM:

    flake.nix
    ├── modules/
    │   └── myos.nix                    ← ONE composable NixOS module
    │       services.myos.enable        ← toggle MYOS
    │       services.myos.role          ← node role determines everything
    │       services.myos.wasmRuntime   ← swappable (wasmtime | wasmedge)
    │       services.myos.cubeletIsolation  ← per-type tier config
    │       services.myos.authority     ← floor, engine co-location
    │       services.myos.llm           ← tier (paid|open-source), model
    │
    ├── nixosConfigurations/
    │   ├── core-server                 role = "core-server"
    │   ├── domain-server               role = "domain-server"
    │   ├── deployment-node             role = "deployment-node"
    │   ├── compute-gpu                 role = "compute-gpu"
    │   ├── compute-general             role = "compute-general"
    │   ├── chain-validator             role = "chain-validator"
    │   └── edge-sensor                 role = "edge-sensor"
    │
    ├── packages/
    │   ├── authority-engine            Haskell (via haskell.nix)
    │   ├── ouroboros-node              Haskell (via haskell.nix)
    │   ├── myos-runtime                Rust userspace services
    │   ├── cubelet-images              Wasm + unikernel images
    │   └── qudag-*                     All qudag crates
    │
    ├── overlays/
    │   ├── preempt-rt                  PREEMPT_RT kernel overlay
    │   └── unikraft                    Unikraft unikernel builder
    │
    └── disko/                          Declarative disk partitioning
        ├── core-server.nix
        ├── compute-node.nix
        ├── chain-validator.nix
        └── edge-sensor.nix


BUILD → DEPLOY PIPELINE:

    CI builds all    → Push to Cachix    → Nodes pull pre-built
    packages from      (binary cache)       binaries (never compile)
    flake.nix          Signed, verified    Content-addressed (tamper-proof)
                                           Edge RISC-V: cross-compiled


DETERMINISM GUARANTEE:
    Same flake.nix + same flake.lock = identical output.
    Bit-for-bit reproducible. On any machine. Every time.
```

---

## 15. Security Model Summary

```
DEFENSE IN DEPTH:

    Layer 1: Hardware Root of Trust
    ├── SoC ROM → Firmware → Kernel → Userspace (verified chain)
    ├── Post-quantum keys (ML-KEM-768, ML-DSA)
    └── Hardware watchdog (reboot on failure)

    Layer 2: Kernel Isolation
    ├── PREEMPT_RT with CPU partitioning (isolcpus)
    ├── No swap, no dynamic kernel modules, no debug interfaces
    └── cgroups v2 + namespaces + seccomp for containers

    Layer 3: Cubelet Sandboxing
    ├── Tier 1: Wasm + Unikraft (VM + sandbox, no OS surface)
    ├── Tier 2: Wasm sandbox (memory-safe, no syscalls)
    └── Tier 3: Container (seccomp, dropped capabilities, read-only rootfs)

    Layer 4: Authority Control
    ├── Every action checked against MBAC (AV vectors)
    ├── Authority Engine: isolated Haskell process, UDS + SO_PEERCRED
    └── Crash → deny all (fail-closed)

    Layer 5: Secret Management (Agenix)
    ├── Encrypted at rest (.age files, SSH host keys)
    ├── Decrypted only to tmpfs (RAM, never disk)
    └── Each secret owned by its service UID

    Layer 6: Network Security
    ├── Mutual authentication (ML-DSA post-quantum)
    ├── Dual authorization for cross-device operations
    └── Post-quantum encryption for chain + authority channels

    Layer 7: Audit & Verification
    ├── All decisions on-chain (tamper-proof)
    ├── All data off-chain (hash-chained, merkle-anchored)
    └── All logs publicly readable (transparency by default)
```

---

## 16. Technology Stack

| Component           | Technology                             | Language                 |
| ------------------- | -------------------------------------- | ------------------------ |
| Authority Engine    | MBAC, Curry-Howard types               | **Haskell**              |
| Ouroboros Consensus | Slot-based PoS chain                   | **Haskell**              |
| Runtime Services    | Scheduler, IPC, watchdog, resource mgr | **Rust**                 |
| Cubelets (math/ML)  | Compiled to Wasm, WASI-NN for ML       | **Rust → Wasm**          |
| Cross-language      | PyO3 FFI (Rust shell + Python core)    | **Rust + Python**        |
| System Config       | Declarative, reproducible builds       | **Nix**                  |
| P2P Networking      | LibP2P mesh                            | **Rust** (qudag-network) |
| Crypto              | ML-KEM-768, ML-DSA, BLAKE3             | **Rust** (qudag-crypto)  |
| Vector DB           | Semantic search for KB                 | **Qdrant**               |
| Knowledge Graph     | Structured relationships               | **Neo4j**                |
| Unikernels          | Micro-library OS for cubelet VMs       | **Unikraft**             |
| Hypervisor          | Lightweight VM for unikernels          | **Firecracker**          |
| Container Runtime   | OCI-compliant lightweight              | **crun**                 |
| LLM (paid)          | System + Domain Orchs                  | **Claude Opus/Sonnet**   |
| LLM (open-source)   | Deploy + Pod Orchs, cubelets           | **Llama/Phi via Ollama** |

---

## 17. Architecture Document Index

| Doc                                                                                                   | Title             | Focus                                                      |
| ----------------------------------------------------------------------------------------------------- | ----------------- | ---------------------------------------------------------- |
| [00-MYOS-master.md](architecture/00-MYOS-master.md)                                                   | System Overview   | Concept explainer, entry point                             |
| [01-authority-model.md](architecture/01-authority-model.md)                                           | MBAC Authority    | AV vectors, reinforcement, floors/ceilings, Haskell engine |
| [02-conflict-resolution.md](architecture/02-conflict-resolution.md)                                   | Escalation Chain  | 4-level resolution, AV-weighted voting, timeouts           |
| [03-pod-assembly.md](architecture/03-pod-assembly.md)                                                 | Pod Formation     | 3-layer dedup, cubelet selection, session pods             |
| [04-cubelet-io-contract.md](architecture/04-cubelet-io-contract.md)                                   | Data Pipeline     | Typed DAG, control messages, PyO3 FFI, proc macro          |
| [05-pod-orchestrator.md](architecture/05-pod-orchestrator.md)                                         | Pod Orch Behavior | Always LLM, DAG construction, tiered failure               |
| [06-verification-audit.md](architecture/06-verification-audit.md)                                     | Chain & Audit     | Ouroboros, Cardano anchoring, Kusari evolution             |
| [07-boot-trust-chain.md](architecture/07-boot-trust-chain.md)                                         | Boot Sequence     | Secure boot, key hierarchy, init order, Agenix             |
| [08-runtime-infrastructure.md](architecture/08-runtime-infrastructure.md)                             | Runtime           | RT scheduling, 3-tier isolation, IPC, watchdog             |
| [09-communication-networking.md](architecture/09-communication-networking.md)                         | Networking        | P2P mesh, post-quantum crypto, Federated MCP               |
| [10-node-topology-orchestrator-hierarchy.md](architecture/10-node-topology-orchestrator-hierarchy.md) | Topology          | Node types, 4-level hierarchy, NixOS, unikernels           |
| [11-repository-mapping.md](architecture/11-repository-mapping.md)                                     | Repo Mapping      | 38 repos mapped, 10 new components                         |
| [12-knowledge-base.md](architecture/12-knowledge-base.md)                                             | Knowledge Base    | Confidence scoring, contradiction resolution, GDPR         |

---

## 18. Key Invariants (System-Wide)

These invariants must hold at ALL times across the entire system:

| #   | Invariant                                                                                              |
| --- | ------------------------------------------------------------------------------------------------------ |
| 1   | AV never exceeds ceiling (1000) or drops below floor per entity type                                   |
| 2   | Floor hierarchy: System(700) > Domain(600) > Deploy(500) > Pod(400) > Cubelet(0)                       |
| 3   | A cubelet belongs to at most one pod at any time (exclusive)                                           |
| 4   | Every action passes through the Authority Engine before execution                                      |
| 5   | Authority Engine crash → deny all (fail-closed)                                                        |
| 6   | Human decisions never affect any entity's AV                                                           |
| 7   | Safety-critical cubelets MUST use wasm_unikernel (double isolation)                                    |
| 8   | All on-chain data is publicly readable without authentication                                          |
| 9   | Every conflict resolves within bounded time (sum of level timeouts)                                    |
| 10  | LLM output contradicting VERIFIED KB entry is blocked                                                  |
| 11  | No code executes without cryptographic verification from previous boot stage                           |
| 12  | Configuration is immutable at runtime (change requires reboot)                                         |
| 13  | Same flake.nix + flake.lock = bit-for-bit identical output (determinism)                               |
| 14  | Secrets exist only in tmpfs (never on persistent storage)                                              |
| 15  | Cross-device operations require dual authorization (both sides approve)                                |
| 16  | Every pipeline-altering decision is logged with full reasoning chain (decision context)                |
| 17  | Session Memory KB entries carry relevance weights (decay with distance/time, boost on reference)       |
| 18  | Context checkpoints written at configured intervals and before any pipeline-altering decision          |
| 19  | Coherence self-check runs at configured intervals; below reload threshold triggers mandatory KB reload |
| 20  | Sub-pods receive structured context packages with goals, constraints, and decision chain               |
| 21  | Sub-pods report contradictions with injected context to parent rather than resolving locally           |

---

_MYOS - the execution integrity layer for autonomous AI systems: deterministic, secure, authority-bounded, and auditable._
