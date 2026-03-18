# MYOS Authority Model - Math-Based Access Control (MBAC)

## Version: 2.0 | Status: LOCKED

---

## 1. Overview

MYOS replaces traditional Role-Based Access Control (RBAC) with **Math-Based Access Control (MBAC)**. In MBAC, every entity in the system - cubelets, Pod Orchestrators, and the System Orchestrator - carries a continuously computed **Authority Value (AV)**. Authority is not a static role assignment; it is a living, reinforcement-driven score that reflects an entity's proven competence, reliability, and recent activity.

MBAC operates in tandem with the **STK (Kernel) framework** - 150 formal invariants organized into 7 families that define the ethical and operational constraints the system must satisfy at all times. While AV governs _how much_ authority an entity has, STK invariants govern _what constraints_ apply to any given action. Together, MBAC + STK form the complete authority system: AV determines permission level, STK determines permission conditions.

The AV dimension registry aligns with **PCCSCEFVRI** - the 10-value ethical anchor (**P**rivacy, **C**are, **C**onsent, **S**afety, **C**reativity, **E**quity, **F**airness, **V**erifiability, **R**esponsibility, **I**ntegrity) that the STK invariants enforce. Each AV dimension maps to one or more PCCSCEFVRI values, ensuring the reinforcement learning loop has an explicit ethical objective function.

**MYOS is not limited to edge robotics or physical actuators.** It is a general-purpose deterministic AI execution platform. The same authority model governs physical control systems (sensors, actuators, industrial controllers) AND conversational AI interactions (user queries, reasoning chains, knowledge synthesis). A user talking to an AI assistant goes through the same authority pipeline as a robot moving an arm - both are tasks, both assemble pods, both are governed by MBAC.

### Why Not RBAC?

Traditional RBAC has fundamental flaws in autonomous systems:

| Problem                  | RBAC Behavior                                              | MBAC Behavior                                                                                     |
| ------------------------ | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Stale privilege          | Admin is admin forever, even if dormant for months         | Dormant entity decays in AV, loses effective authority                                            |
| Equal privilege illusion | Two admins have identical power regardless of track record | Two entities at same "role level" have different AV - the more competent one has higher influence |
| No earned trust          | New admin has same power as veteran admin on day one       | Authority must be earned through successful task execution                                        |
| Permanent privilege      | Once granted, hard to revoke without explicit action       | Authority self-corrects - poor performance automatically reduces AV                               |

---

## 2. Authority Value (AV) Structure

### 2.1 Multi-Dimensional Vector

Each entity's authority is represented as a **multi-dimensional vector**, not a single scalar. Each dimension corresponds to a domain of authority.

```
AV = {
    dimension_1:  float,
    dimension_2:  float,
    dimension_3:  float,
    ...
}
```

**Dimensions are NOT hardcoded.** MYOS uses a **Flexible Dimension Registry** - the set of authority dimensions is configured per deployment. Different deployments can define different dimensions based on their use case.

### 2.2 Flexible Dimension Registry

The dimension registry is defined at system configuration time and is immutable at runtime.

```
dimension_registry {
    dimensions: [
        {
            name:        String        // unique identifier (e.g., "execution")
            description: String        // human-readable purpose
            av_max:      float         // ceiling for this dimension (can differ per dim)
            av_floor: {                // floor per entity type (can differ per dim)
                system_orchestrator: float
                pod_orchestrator:    float
                cubelet:             float
            }
            decay_rate:  float         // decay rate can differ per dimension
        }
    ]
}
```

#### Standard Dimensions

The following dimensions are defined as standard, organized by category. Deployments select which ones they need. All standard dimensions are mapped to the PCCSCEFVRI ethical values they enforce.

**Physical / Edge Dimensions:**

| Dimension   | Purpose                                             | Use Case                                            | PCCSCEFVRI Mapping |
| ----------- | --------------------------------------------------- | --------------------------------------------------- | ------------------ |
| `execution` | Authority over task execution and control decisions | Motor control, process execution, actuator commands | Verifiability |
| `safety`    | Authority over safety-critical decisions            | Emergency stops, hazard avoidance, safety overrides | Safety |
| `resource`  | Authority over resource allocation                  | Memory, CPU, I/O, sensor access, bandwidth          | Responsibility |
| `policy`    | Authority over policy and rule interpretation       | Compliance, regulation, operational rules           | Fairness |

**AI Interaction Dimensions:**

| Dimension       | Purpose                                                          | Use Case                                                                    | PCCSCEFVRI Mapping |
| --------------- | ---------------------------------------------------------------- | --------------------------------------------------------------------------- | ------------------ |
| `reasoning`     | Authority over complex reasoning chains and multi-step logic     | Deep analysis, multi-hop inference, mathematical proof, strategic planning  | Verifiability |
| `communication` | Authority over user-facing responses and commitments             | What the AI can say, tone, making promises, providing guarantees            | Care, Integrity |
| `knowledge`     | Authority over which knowledge domains can be accessed and cited | Domain expertise boundaries, data access levels, citation authority         | Responsibility |
| `autonomy`      | Authority over independent action without human confirmation     | Self-directed task execution, unsupervised decisions, autonomous operations | Consent |

**Ethical / Governance Dimensions (PCCSCEFVRI-aligned):**

| Dimension          | Purpose                                                         | Use Case                                                         | PCCSCEFVRI Mapping |
| ------------------ | --------------------------------------------------------------- | ---------------------------------------------------------------- | ------------------ |
| `privacy`          | Authority over personal data handling and disclosure            | Data minimization, consent enforcement, anonymization            | Privacy, Consent |
| `equity`           | Authority over fairness-sensitive decisions                     | Bias-bounded inference, equitable resource distribution          | Equity, Fairness |
| `accountability`   | Authority over audit and transparency operations               | Log access, audit trail verification, compliance reporting       | Responsibility, Integrity |
| `planetary`        | Authority over environmental and sustainability decisions       | MRV verification, carbon accounting, resource stewardship        | (Planetary Integrity - extends PCCSCEFVRI) |

These ethical dimensions are activated when MYOS operates in governance-sensitive contexts (Stages 5-9 of the cubelet lattice) or when regulatory compliance requires explicit tracking of privacy, equity, or environmental impact.

#### Deployment Configuration Examples

**Edge Robotics Deployment (Stages 0, 4):**

```
dimensions: [execution, safety, resource, policy]
```

**AI Interaction Platform (Stages 0-3):**

```
dimensions: [execution, safety, reasoning, communication, knowledge, autonomy]
```

**Hybrid (Robotics with AI Interface, Stages 0-4):**

```
dimensions: [execution, safety, resource, policy, reasoning, communication, knowledge, autonomy]
```

**Full Governance Deployment (Stages 0-9, Kusari-era):**

```
dimensions: [execution, safety, resource, policy, reasoning, communication, knowledge, autonomy,
             privacy, equity, accountability, planetary]
```

**Custom / Domain-Specific:**
Deployments can define entirely custom dimensions (e.g., `financial_authority`, `medical_authority`, `legal_authority`) as long as they follow the dimension registry schema. Custom dimensions should be mapped to the PCCSCEFVRI value(s) they enforce.

#### Dimension Independence

Each dimension is **fully independent**:

- Rewards/penalties are applied to the specific dimension relevant to the event
- Decay is tracked per dimension (idle in reasoning ≠ idle in execution)
- Floors are defined per dimension per entity type
- Ceilings can differ per dimension (e.g., safety ceiling = 1000, reasoning ceiling = 500)

The authority engine operates generically over whatever dimensions are registered. No code changes are needed when dimensions are added or removed - only configuration changes.

**Example (AI Interaction deployment):**

```
Cubelet_LLM_Agent_07 = {
    execution:      650.0,
    safety:         880.2,
    reasoning:      920.5,
    communication:  710.3,
    knowledge:      540.0,
    autonomy:       320.8
}
```

This cubelet excels at reasoning tasks but has limited autonomy (most of its actions require confirmation).

```
Cubelet_Math_FFT_12 = {
    execution:      990.0,
    safety:         950.0,
    reasoning:      100.0,     // math functions don't reason
    communication:  0.0,       // math functions don't communicate
    knowledge:      0.0,       // not applicable
    autonomy:       980.0      // highly trusted, can act independently
}
```

This math cubelet has near-max execution and autonomy (it's deterministic and proven) but zero communication authority (it never talks to users).

### 2.3 Why Multi-Dimensional?

A single scalar conflates unrelated competencies. A cubelet that excels at motor control (execution) should not automatically gain influence over safety decisions. A cubelet that is great at reasoning should not automatically gain authority to make autonomous decisions. Multi-dimensional vectors enforce **domain-specific authority**: an entity earns trust independently in each domain based on its performance in that domain.

In the cubelet lattice, framework layers naturally map to different dimension strengths:

- **STK cubelets** (invariant checkers) accumulate high `safety`, `accountability`, and `execution` AV - they prove things correctly or they don't
- **STI cubelets** (reasoning/logic) accumulate high `reasoning`, `knowledge`, and `equity` AV - they classify, rank, and reason
- **STD cubelets** (runtimes) accumulate high `execution` and `communication` AV - they execute and deliver
- **STF cubelets** (fabric/connectors) accumulate high `execution` and `privacy` AV - they handle data in transit
- **STA cubelets** (policies) accumulate high `policy`, `accountability`, and `autonomy` AV - they define and interpret rules

This natural alignment means the lattice structure reinforces domain-specific authority: cubelets earn trust in the dimensions their framework layer exercises most.

### 2.4 Dimension Comparison Rule - "Relevant Dimension Match"

When a task or decision requires authority evaluation, **only the relevant dimension is compared**.

```
Task: motor_control_adjustment
Relevant dimension: execution

Cubelet A: execution = 800
Cubelet B: execution = 450

→ Cubelet A has authority for this task
```

```
Task: emergency_shutdown_decision
Relevant dimension: safety

Cubelet A: safety = 300
Cubelet B: safety = 920

→ Cubelet B has authority for this task (despite lower execution AV)
```

**A cubelet is selected for a task based on its strength in the task's relevant dimension, not its overall average or total.**

### 2.5 Multiple Relevant Dimensions

Some tasks may span multiple dimensions. In such cases:

```
relevant_dimensions = [execution, safety]

Comparison: use the PRIMARY dimension specified by the task descriptor.
Secondary dimensions serve as tiebreakers if primary dimension values are equal.
```

Task descriptors must always specify a primary dimension.

---

## 3. Authority Bounds

### 3.1 Hard Ceiling

Every AV dimension has a hard ceiling that can never be exceeded.

```
0 ≤ AV_dimension ≤ AV_max

Default AV_max = 1000 (configurable per deployment)
```

No entity, regardless of performance history, can exceed `AV_max` in any dimension. This prevents unbounded authority accumulation and ensures the system remains balanced.

### 3.2 Diminishing Returns on Rewards

Authority growth follows a **diminishing returns curve**. The closer an entity is to the ceiling, the harder it is to gain more authority.

```
effective_reward = base_reward × (1 - AV_current / AV_max)^k

Where:
  base_reward  = the raw reward for a successful action
  AV_current   = current AV in the relevant dimension
  AV_max       = hard ceiling
  k            = diminishing returns exponent (k > 0, default k = 1)
```

**Examples (AV_max = 1000, base_reward = 50, k = 1):**

| Current AV | Effective Reward | Explanation                      |
| ---------- | ---------------- | -------------------------------- |
| 100        | 45.0             | Early stage - nearly full reward |
| 500        | 25.0             | Mid-range - half reward          |
| 800        | 10.0             | High performer - small gains     |
| 950        | 2.5              | Near ceiling - tiny increments   |
| 990        | 0.5              | Almost maxed - negligible gain   |

This creates a natural distribution where most entities cluster in the mid-range, a few reach high AV, and hitting the ceiling is rare and meaningful.

### 3.3 Constant Penalties

Unlike rewards, **penalties do not diminish**. A mistake costs the same amount of AV regardless of the entity's current authority level.

```
effective_penalty = base_penalty  (constant, no scaling)
```

**Rationale:** A high-authority entity making a mistake is just as bad (or worse) than a low-authority entity making one. Diminishing penalties would make high-AV entities immune to consequences.

**Examples (base_penalty = 100):**

| Current AV | Penalty Applied | New AV               |
| ---------- | --------------- | -------------------- |
| 900        | -100            | 800                  |
| 500        | -100            | 400                  |
| 50         | -100            | 0 (clamped to floor) |

### 3.4 Decay on Inactivity

Dormant entities lose authority over time. This prevents stale privilege accumulation.

```
decay = decay_rate × time_since_last_activity

AV(t) = max(AV_floor, AV(t-1) - decay)
```

Decay is applied **per dimension** independently. An entity active in execution tasks but idle in safety decisions will decay only in the safety dimension.

**Decay parameters:**

- `decay_rate`: AV points lost per unit time of inactivity (configurable)
- `time_since_last_activity`: measured per dimension (last time the entity made a decision or executed a task in that domain)

---

## 4. Hierarchical Authority Floors

### 4.1 Floor Definitions

Different entity types have different minimum authority floors. An entity's AV can never drop below its floor, regardless of penalties or decay.

```
Entity Type              AV Floor    Rationale
─────────────────────────────────────────────────────────────
System Orchestrator      700         Must always maintain system-level authority
Domain Orchestrator      600         Semi-autonomous domain management
Deployment Orchestrator  500         Zone infrastructure management
Pod Orchestrator         400         Must always outrank its cubelets
Cubelet                  0           Can lose all authority
```

### 4.2 Floor Invariant

The system enforces a strict hierarchical invariant:

```
floor(System) > floor(Domain) > floor(Deployment) > floor(Pod) > floor(Cubelet)

700 > 600 > 500 > 400 > 0
```

This guarantees that:

- A System Orchestrator can **always** override any other orchestrator
- A Domain Orchestrator can **always** override Deployment and Pod Orchestrators
- A Deployment Orchestrator can **always** override Pod Orchestrators
- A Pod Orchestrator can **always** override its cubelets
- The hierarchy is maintained even in worst-case scenarios (all penalties applied, maximum decay)

See **Node Topology (10-node-topology-orchestrator-hierarchy.md)** for the full four-level orchestrator hierarchy and hardware assignments.

### 4.3 Floor Behavior

When a penalty or decay would push an entity below its floor:

```
AV(new) = max(AV_floor, AV(current) - penalty)

Example:
  Pod Orchestrator AV = 420
  Penalty = 100
  AV(new) = max(400, 420 - 100) = max(400, 320) = 400  ← clamped to floor
```

The entity is not destroyed or demoted. It sits at its floor, still functional but with minimal authority for its level. Recovery requires successful task execution to earn AV back.

### 4.4 Floor Configuration

Floors are defined at system configuration time (boot) and are **immutable at runtime**. No entity, including the System Orchestrator or human operator, can change floors without a system reconfiguration and reboot.

```
system_config {
    authority_floors {
        system_orchestrator: 700
        pod_orchestrator: 400
        cubelet: 0
    }
    authority_ceiling: 1000
    diminishing_returns_exponent: 1.0
    decay_rate: 0.5          // AV points per hour of inactivity
    base_penalty: 100
}
```

---

## 5. Authority Update Model (Reinforcement)

### 5.1 Update Rule

Authority is updated after every task completion, decision outcome, or time interval.

```
AV(t+1) = clamp(
    AV(t)
    + effective_reward(success_events)
    - base_penalty × violation_count
    - risk_penalty × risk_events
    - decay_rate × inactivity_duration,

    AV_floor,     // lower bound
    AV_max        // upper bound
)
```

### 5.2 Event Types

| Event          | Effect on AV   | Scaling                                         |
| -------------- | -------------- | ----------------------------------------------- |
| **Success**    | + reward       | Diminishing returns (harder to gain at high AV) |
| **Violation**  | - penalty      | Constant (always hurts the same)                |
| **Risk Event** | - risk_penalty | Constant (safety events are always severe)      |
| **Inactivity** | - decay        | Linear with time (longer idle = more loss)      |

### 5.3 Success Events

A success event is generated when:

- A cubelet's inference result is validated as correct
- A cubelet's proposed action is executed without issues
- A pod completes its assigned task within time and resource budgets
- An orchestrator's coordination decision leads to successful pod completion

Each success event carries a **base_reward** value determined by the task difficulty and domain.

### 5.4 Violation Events

A violation event is generated when:

- A cubelet produces an output that fails validation
- A cubelet attempts an action beyond its authority
- A cubelet's output causes a downstream failure
- An orchestrator makes a coordination decision that leads to pod failure

Each violation carries a **base_penalty** value. Safety violations carry a higher penalty multiplier.

```
penalty = base_penalty × severity_multiplier

Severity levels:
  minor:    multiplier = 1.0  (e.g., suboptimal output)
  moderate: multiplier = 2.0  (e.g., failed task)
  severe:   multiplier = 5.0  (e.g., safety violation)
  critical: multiplier = 10.0 (e.g., caused system-level incident)
```

### 5.5 Risk Events

Risk events are distinct from violations. A risk event occurs when:

- A cubelet's action is flagged by the safety subsystem (even if no actual harm occurred)
- A cubelet operates near the boundary of its authority
- Environmental conditions make a cubelet's domain of operation hazardous

Risk penalties are preventive - they reduce authority to limit future exposure.

### 5.6 Update Frequency

AV updates are **event-driven**, not periodic.

```
Trigger: task completion, violation detected, risk event detected
Processor: Authority Engine (Haskell)
Effect: immediate AV recalculation for involved entities
```

Decay is the exception - it is computed on-demand when an entity's AV is queried after a period of inactivity, not as a background timer.

```
on_query(entity_id):
    time_idle = now() - last_activity(entity_id)
    if time_idle > 0:
        apply_decay(entity_id, time_idle)
    return AV(entity_id)
```

This is a lazy evaluation pattern - decay is applied at read time, not write time. Avoids unnecessary computation for dormant entities.

---

## 6. Authority in Action - Decision Flow

### 6.1 Action Authorization

Before any action is executed in the system, it must pass **two gates**: AV permission and STK invariant satisfaction.

```
1. Entity requests to perform action involving cubelets [C1, C2, ...]
2. System determines required_authority for the action
   (which dimension, what minimum AV is needed)
3. System resolves STK invariants for involved cubelets
   (via PROVES edges in the CIG)
4. System queries Authority Engine:
   authorize(entity_id, action, cubelet_ids, context) → allow | deny | escalate
5. Authority Engine evaluates (both gates must pass):
   Gate 1 - AV check:
     if entity.AV[relevant_dimension] ≥ required_authority → pass
     else → escalate or deny
   Gate 2 - STK invariant check:
     for each relevant STK invariant:
       evaluate invariant against current state
     if all pass → pass
     if any fail → deny (regardless of AV - invariants are absolute)
6. Both gates pass → allow
   AV fails, can escalate → escalate
   STK invariant fails → deny (no escalation possible for invariant violations)
```

### 6.2 Required Authority Mapping

Every action type in the system has a predefined required authority level per dimension, plus a set of STK invariants that must be satisfied.

```
action_authority_map {
    read_sensor: {
        av_required:  { execution: 100 }
        stk_required: ["device.attested"]                          // Stage 4 invariant
    }
    write_actuator: {
        av_required:  { execution: 500, safety: 600 }
        stk_required: ["device.attested", "safety.refusal_on_redline"]
    }
    modify_pod_composition: {
        av_required:  { resource: 700 }
        stk_required: ["audit.complete"]
    }
    emergency_stop: {
        av_required:  { safety: 200 }                              // low threshold - anyone should be able to stop
        stk_required: []                                           // no invariant gate - always allowed
    }
    override_safety_rule: {
        av_required:  { safety: 950 }                              // extremely high
        stk_required: ["safety.refusal_on_redline", "ethics.aligned", "logs.signed"]
    }
    cross_pod_communication: {
        av_required:  { policy: 300 }
        stk_required: ["privacy.min_disclosure"]                   // Stage 2 invariant
    }
    process_personal_data: {
        av_required:  { privacy: 600, execution: 300 }
        stk_required: ["consent.required", "privacy.min_disclosure", "revocation.window<=24h"]
    }
    carbon_credit_verification: {
        av_required:  { execution: 500, planetary: 400 }
        stk_required: ["MRV.verified", "planetary_boundary<=1"]    // Stage 8 invariants
    }
}
```

This map is defined at system configuration time and is immutable at runtime. The `stk_required` field references STK cubelet invariant IDs from the cubelet registry.

---

## 7. Authority Engine - Haskell Implementation Contract

### 7.1 Deployment & Build

The Authority Engine runs **centralized** on the core server (x86/ARM), co-located with the System Orchestrator. Edge nodes and distributed compute nodes call it via RPC over qudag-network. It is built using **haskell.nix** for reproducible builds and deployed as a NixOS systemd service.

### 7.2 STK Invariant Enforcement

Beyond AV-based permission checks, the Authority Engine enforces **150 STK kernel invariants** - formal constraints that must hold for any action involving the corresponding cubelets. STK invariants are organized into 7 families:

| Family | Guarantee | Example Invariants |
| ------ | --------- | ------------------ |
| **Privacy & Consent** | Minimal disclosure, data sovereignty | `consent.required`, `revocation.window <= 24h` |
| **Fairness & Equity** | Bounded bias, revenue fairness | `bias <= θ`, `revenue_share.fairness` |
| **Verifiability & Determinism** | Proofs exist, execution reproducible | `zk_proof.exists`, `hash.reproducible` |
| **Safety & Security** | Redline refusal, system isolation | `refusal_on_redline`, `system.isolated` |
| **Accountability & Audit** | Signed logs, audited upgrades | `logs.signed`, `upgrade.audited` |
| **Planetary Integrity** | MRV verified, boundary compliance | `MRV.verified`, `planetary_boundary <= 1` |
| **Ethical Reflexivity** | PCCSCEFVRI anchors intact | `ethics.aligned`, `corrigibility.maintained` |

**Authorization with STK invariants:**

```
1. Entity requests action involving cubelets [C1, C2, C3]
2. Authority Engine checks AV: entity.AV[relevant_dim] >= required_authority
3. Authority Engine resolves STK invariants for C1, C2, C3
   (via PROVES edges in the CIG - which STK cubelets validate these operations)
4. ProofPerl evaluates each relevant STK invariant:
   - Invariant expression loaded from cubelet registry
   - ProofPerl executes formal check against current system state
   - Result: pass | fail | indeterminate
   if all pass → allow
   if any fail → deny (regardless of AV)
   if indeterminate → escalate to conflict resolution
5. Invariant check results logged on-chain
```

**STK invariants cannot be overridden by high AV.** A cubelet with AV 999 in execution is still blocked if the action violates `refusal_on_redline`. AV governs permission level; STK invariants govern permission conditions. Both must pass.

#### ProofPerl - The STK Enforcement Runtime

**ProofPerl** is the execution engine that evaluates STK invariants. It is a formal proof runtime embedded in the Authority Engine that:

- Evaluates constraint expressions (e.g., `bias <= θ`, `revocation.window <= 24h`, `planetary_boundary <= 1`) against current system state
- Supports parameterized thresholds (θ, ε, Δt, Lmin, Rmin) defined per invariant in the cubelet registry
- Produces deterministic pass/fail/indeterminate results (same input → same output, always)
- Runs inside the Authority Engine's Haskell process (compiled as a Haskell DSL)
- Cannot be modified at runtime - invariant expressions are loaded at boot and are immutable

**PerlFrame** is the value anchor within ProofPerl that maps invariants to PCCSCEFVRI ethical values. Each invariant carries a `perlframe_anchor` field:

```
invariant {
    id:               "safety.refusal_on_redline"
    expression:       "action.risk_score < redline_threshold"
    parameters:       { redline_threshold: 0.95 }
    perlframe_anchor: ["Safety"]                    // PCCSCEFVRI mapping
    family:           "safety_security"
    stage:            3
    locked:           true                          // cannot be weakened via Rubik's Move
}
```

ProofPerl is NOT a separate service - it is a module within the Authority Engine's Haskell codebase, sharing its purity and type safety guarantees. Invariant evaluation is a pure function: `evaluate(invariant, system_state) → pass | fail | indeterminate`.

STK invariant definitions are loaded from the cubelet registry at boot and are **immutable at runtime**. Updating invariants requires a Rubik's Move (see Lattice Governance in [00-MYOS-master.md](00-MYOS-master.md)) with Level 4 (human-in-the-loop) approval and system reconfiguration.

#### Adaptive Constraint Types

Each STK invariant carries a constraint type that determines how its thresholds can evolve:

| Type | Behavior | Example |
| ---- | -------- | ------- |
| **Hard** | Threshold never changes. Cannot be weakened via Rubik's Move. Equivalent to `invLocked = True`. | `safety.refusal_on_redline` |
| **Soft** | Threshold can be adjusted via Rubik's Move with Level 4 human approval. | `explainability.required` (min_score adjustable) |
| **Adaptive** | Threshold auto-tunes based on entity AV history. Entities with high safety AV may get slightly relaxed thresholds; those with violation history get tightened. | Fairness theta per-entity adjustment |

Adaptive constraints use a feedback loop: ProofPerl evaluates the invariant → result + entity AV history fed to PerlFrame → PerlFrame computes adjusted threshold → next evaluation uses the new threshold. The adjustment range is bounded by the invariant's `min_threshold` and `max_threshold` parameters.

### 7.3 Language Choice Rationale

The Authority Engine is implemented in **Haskell** for the following reasons:

- **Pure functions**: Authority computations are side-effect-free. Same input always produces same output. Deterministic by construction.
- **Type safety**: Invalid authority states (negative AV, exceeded ceiling, broken floor invariant) are unrepresentable in Haskell's type system via smart constructors and refined types.
- **Property-based testing**: QuickCheck/Hedgehog can verify invariants (e.g., "AV never exceeds ceiling after any sequence of operations") over millions of random test cases.
- **Formal verification proximity**: Haskell code can be reasoned about mathematically, and if needed, ported to proof assistants (Agda, Idris) for formal certification. Via the Curry-Howard correspondence, MBAC invariants expressed as Haskell types are automatically enforced by the compiler - types are propositions, programs are proofs.
- **Algebraic data types**: The authority model (vectors, events, decisions) maps naturally to Haskell ADTs and pattern matching.

### 7.4 API Contract (IPC / gRPC Interface)

The Authority Engine runs as an **isolated service** communicating with the Rust runtime via IPC (Unix socket or gRPC).

```
service AuthorityEngine {

    // Query current authority state
    rpc GetAuthority(EntityId) → AuthorityVector

    // Authorize an action
    rpc Authorize(AuthorizeRequest) → AuthorizeResponse
        where AuthorizeRequest = {
            entity_id:   EntityId,
            action:      ActionType,
            context:     ActionContext
        }
        where AuthorizeResponse = Allow | Deny | Escalate {
            decision_id:   UUID,
            reasoning:     String,
            timestamp:     Timestamp,
            signature:     Signature
        }

    // Update authority after an event
    rpc UpdateAuthority(UpdateRequest) → AuthorityVector
        where UpdateRequest = {
            entity_id:   EntityId,
            event:       Event (Success | Violation | RiskEvent | Inactivity),
            severity:    SeverityLevel,
            context:     EventContext
        }

    // Query authority history for audit
    rpc AuditTrail(AuditRequest) → [AuthorityEvent]
        where AuditRequest = {
            entity_id:    EntityId,
            time_range:   (Timestamp, Timestamp),
            dimension:    Option<DimensionName>
        }

    // Get the policy rules for a domain
    rpc GetPolicy(DomainId) → PolicyRules

    // Compare two entities on a specific dimension
    rpc CompareAuthority(CompareRequest) → CompareResponse
        where CompareRequest = {
            entity_a:     EntityId,
            entity_b:     EntityId,
            dimension:    DimensionName
        }
        where CompareResponse = {
            winner:       EntityId,
            margin:       float,
            confidence:   float
        }
}
```

### 7.5 Isolation Guarantees

- The Authority Engine runs in its own **memory-isolated process**.
- It has **no direct access** to the kernel, hardware, network, or any other system component.
- It receives input only via IPC and produces output only via IPC.
- It maintains its own internal state (authority vectors, history) in its own memory space.
- If the Authority Engine crashes, the system defaults to **deny all actions** until it recovers.
- The Authority Engine **cannot modify its own authority rules at runtime**. Rules are loaded at boot from a signed, verified configuration.

### 7.6 Determinism Contract

The Authority Engine must satisfy:

```
For all inputs I, timestamps T:
    authorize(I, T) at time T₁ == authorize(I, T) at time T₂

That is: given the same input and the same logical timestamp,
the Authority Engine always produces the same output,
regardless of when (wall-clock time) the computation runs.
```

This is naturally guaranteed by Haskell's purity - no IO in the core computation.

---

## 8. Human Relationship to Authority Model

### 8.1 Human is Outside the Reinforcement Loop

Human decisions do **not** affect cubelet AV. The reinforcement model is purely performance-based.

```
Human approves Cubelet A's proposal   → Cubelet A gets NO AV reward from human approval
Human rejects Cubelet B's proposal    → Cubelet B gets NO AV penalty from human rejection
```

**Rationale:** If human opinions influenced AV, cubelets would optimize for human approval rather than task performance. The authority model must reflect objective outcomes, not subjective preferences.

### 8.2 Human Authority in the System

Human operators participate in the conflict resolution chain (see: Conflict Resolution document) but are not entities in the MBAC model. They do not have an AV vector. Their authority is structural, not earned.

### 8.3 Human Override Logging

All human interventions are logged separately from the authority reinforcement system.

```
human_override_log {
    override_id:      UUID
    human_operator_id: String
    action_overridden: ActionType
    original_decision: AuthorizeResponse
    human_decision:    HumanDecision
    reasoning:         String (optional, human-provided)
    timestamp:         Timestamp
    signature:         Signature (human's auth credential)
}
```

Human overrides are auditable but do not feed back into the authority model.

---

## 9. System Constants and Configuration

All MBAC parameters are defined at system configuration time and are immutable at runtime.

```
mbac_config {
    // Bounds
    av_max:                     1000

    // Floors (hierarchical invariant enforced - see 10-node-topology.md)
    floor_system_orchestrator:  700
    floor_domain_orchestrator:  600
    floor_deployment_orchestrator: 500
    floor_pod_orchestrator:     400
    floor_cubelet:              0

    // Reward model
    base_reward:                50
    diminishing_returns_k:      1.0

    // Penalty model
    base_penalty:               100
    severity_multipliers {
        minor:      1.0
        moderate:   2.0
        severe:     5.0
        critical:   10.0
    }

    // Risk
    risk_penalty:               75

    // Decay
    decay_rate_per_hour:        0.5

    // Dimension Registry (flexible - configured per deployment)
    // Dimensions are mapped to PCCSCEFVRI ethical values
    dimension_registry {
        dimensions: [
            // Physical / Edge dimensions
            {
                name: "execution"
                description: "Authority over task execution and control decisions"
                pccscefvri: ["Verifiability"]
                av_max: 1000
                av_floor: { system_orch: 700, pod_orch: 400, cubelet: 0 }
                decay_rate_per_hour: 0.5
            },
            {
                name: "safety"
                description: "Authority over safety-critical decisions"
                pccscefvri: ["Safety"]
                av_max: 1000
                av_floor: { system_orch: 700, pod_orch: 400, cubelet: 0 }
                decay_rate_per_hour: 0.3   // slower decay - safety trust is hard-earned
            },
            // AI interaction dimensions
            // reasoning, communication, knowledge, autonomy (see section 2.2)

            // Ethical / Governance dimensions (PCCSCEFVRI-aligned)
            {
                name: "privacy"
                description: "Authority over personal data handling and disclosure"
                pccscefvri: ["Privacy", "Consent"]
                av_max: 1000
                av_floor: { system_orch: 700, pod_orch: 400, cubelet: 0 }
                decay_rate_per_hour: 0.4
            },
            {
                name: "equity"
                description: "Authority over fairness-sensitive decisions"
                pccscefvri: ["Equity", "Fairness"]
                av_max: 1000
                av_floor: { system_orch: 700, pod_orch: 400, cubelet: 0 }
                decay_rate_per_hour: 0.4
            },
            {
                name: "accountability"
                description: "Authority over audit and transparency operations"
                pccscefvri: ["Responsibility", "Integrity"]
                av_max: 1000
                av_floor: { system_orch: 700, pod_orch: 400, cubelet: 0 }
                decay_rate_per_hour: 0.3
            },
            {
                name: "planetary"
                description: "Authority over environmental and sustainability decisions"
                pccscefvri: []   // extends PCCSCEFVRI with planetary integrity
                av_max: 1000
                av_floor: { system_orch: 700, pod_orch: 400, cubelet: 0 }
                decay_rate_per_hour: 0.2   // slow decay - environmental trust is long-term
            }
        ]
    }

    // STK Invariant Families (loaded from cubelet registry at boot)
    stk_invariant_families {
        privacy_consent:          { count: ~20, stages: [0, 2, 5, 7] }
        fairness_equity:          { count: ~20, stages: [3, 5, 6, 7] }
        verifiability_determinism:{ count: ~25, stages: [0, 1, 2, 3, 4] }
        safety_security:          { count: ~25, stages: [0, 3, 4, 8] }
        accountability_audit:     { count: ~20, stages: [1, 5, 6] }
        planetary_integrity:      { count: ~20, stages: [4, 8] }
        ethical_reflexivity:      { count: ~20, stages: [9] }
        // Total: 150 STK invariants (15 per stage × 10 stages)
    }

    // Action authority map (see section 6.2 for full spec with stk_required)
    action_authority_map {
        // Physical actions
        read_sensor:              { av: { execution: 100 }, stk: ["device.attested"] }
        write_actuator:           { av: { execution: 500, safety: 600 }, stk: ["safety.refusal_on_redline"] }
        emergency_stop:           { av: { safety: 200 }, stk: [] }
        override_safety_rule:     { av: { safety: 950 }, stk: ["ethics.aligned", "logs.signed"] }

        // AI interaction actions
        answer_simple_query:      { av: { communication: 200, reasoning: 100 }, stk: [] }
        deep_analysis:            { av: { reasoning: 600, knowledge: 500 }, stk: ["explainability.required"] }
        make_commitment:          { av: { communication: 800, autonomy: 700 }, stk: ["logs.signed"] }
        autonomous_decision:      { av: { autonomy: 800, safety: 500 }, stk: ["safety.refusal_on_redline"] }
        cross_domain_reasoning:   { av: { reasoning: 700, knowledge: 600 }, stk: ["privacy.min_disclosure"] }

        // Governance actions (Stages 5-9)
        process_personal_data:    { av: { privacy: 600 }, stk: ["consent.required", "revocation.window<=24h"] }
        carbon_verification:      { av: { planetary: 400 }, stk: ["MRV.verified", "planetary_boundary<=1"] }

        // System actions
        cross_pod_communication:  { av: { policy: 300 }, stk: ["privacy.min_disclosure"] }
        modify_pod_composition:   { av: { resource: 700 }, stk: ["audit.complete"] }
    }
}
```

This configuration is:

- Loaded at boot
- Cryptographically signed
- Verified against a known hash before the Authority Engine starts
- Cannot be modified without a system reboot and re-signing

---

## 10. Invariants (Must Always Hold)

The following invariants must be provably maintained by the Authority Engine at all times:

```
INV-1:  ∀ entity, dimension: AV_floor ≤ AV[dimension] ≤ AV_max
INV-2:  floor(System Orchestrator) > floor(Pod Orchestrator) > floor(Cubelet)
INV-3:  ∀ action: authorize(entity, action) == allow  ⟹  entity.AV[relevant_dim] ≥ required_authority(action)
INV-4:  ∀ penalty: AV(new) = max(AV_floor, AV(current) - penalty)
INV-5:  ∀ reward: AV(new) = min(AV_max, AV(current) + effective_reward)
INV-6:  effective_reward ≤ base_reward  (diminishing returns guarantee)
INV-7:  ∀ entity: AV history is append-only and tamper-evident
INV-8:  Authority Engine crash → system defaults to deny-all
INV-9:  Configuration is immutable at runtime
INV-10: Human decisions do not alter any entity's AV
```

INV-11: Dimension registry is immutable at runtime (no adding/removing dimensions without reboot)
INV-12: All dimensions in the registry follow the same floor hierarchy invariant independently
INV-13: ∀ action: if any STK invariant in stk_required fails → deny (regardless of AV)
INV-14: STK invariant definitions are immutable at runtime (loaded from cubelet registry at boot)
INV-15: STK invariant updates require Level 4 (human-in-the-loop) approval + system reboot
INV-16: ∀ cubelet model update: AV resets to floor (new model must re-earn trust)
INV-17: Cubelet lattice position (Framework-Stage-Index) is permanent and immutable
INV-18: ∀ AV dimension: maps to at least one PCCSCEFVRI value (ethical traceability)
INV-19: Adaptive constraint thresholds are bounded by min_threshold and max_threshold parameters
INV-20: Adaptive threshold adjustments are logged on-chain with before/after values
```

These invariants should be verified via property-based testing (QuickCheck) and, for safety-critical deployments, via formal proof.

---

## 11. AI Interaction Model - Hybrid Session Pods

MYOS is not limited to physical systems. When a user interacts with the AI platform (conversation, query, analysis request), the interaction is modeled as a **task** governed by the same MBAC authority system.

### 11.1 Session Pod Model

When a user starts a conversation, a **Session Pod** is assembled. This is a long-lived pod that persists for the duration of the user's session.

```

SESSION POD (long-lived)
┌─────────────────────────────────────────────────┐
│ Session Orchestrator (Pod Orchestrator role) │
│ Context Manager cubelet (conversation state) │
│ Safety Monitor cubelet (output filtering) │
│ Basic LLM cubelet (handles simple queries) │
└─────────────────────────────────────────────────┘

```

**Session Pod responsibilities:**
- Track conversation context across multiple queries
- Handle simple queries directly (no sub-pod needed)
- Route complex queries to sub-pods
- Filter all outputs through safety checks before user sees them
- Maintain session state (not just per-query state)

### 11.2 Sub-Pod Spawning for Complex Queries

When a query requires capabilities not present in the Session Pod, the Session Orchestrator requests a **sub-pod** from the System Orchestrator.

```

Query routing decision (capability check):

1. Session Orchestrator receives user query
2. Determine required capabilities for the query
3. Check: does the Session Pod have all required capabilities?
   YES → handle directly within Session Pod
   NO → request sub-pod from System Orchestrator

```

**Query routing is deterministic** - it is a capability check, not a judgment call. If the Session Pod lacks a required capability, a sub-pod is spawned. No LLM-based routing decision.

**Sub-pod behavior:**
- Assembled by the System Orchestrator using the standard pod assembly protocol
- Contains specialized cubelets (data loaders, ML models, domain experts)
- Has its own Pod Orchestrator
- Communicates results ONLY back to the Session Pod
- Dissolves immediately after completing the query
- Authority ceiling: cannot exceed the Session Pod's authority (inherited ceiling)

### 11.3 Parallel Sub-Pods (Fan-Out, No Nesting)

For complex queries requiring multiple independent capabilities:

```

User: "Compare the financial performance of Company A vs Company B
and predict next quarter trends"

Session Pod spawns:
Sub-Pod 1: Financial data retrieval (Company A)
Sub-Pod 2: Financial data retrieval (Company B)
Sub-Pod 3: Trend prediction model

All three run IN PARALLEL.
No sub-pod can spawn its own children.
Session Pod collects all results and synthesizes the response.

```

**Nesting rules:**
- Session Pod → can spawn multiple sub-pods (fan-out)
- Sub-Pod → CANNOT spawn sub-sub-pods (no nesting)
- If a sub-pod determines it needs additional capabilities, it reports back to the Session Pod, which spawns another sub-pod

### 11.4 Idle Session - Tiered Cubelet Release

When a session is idle (user not actively sending queries):

```

TIERED RELEASE POLICY
─────────────────────
Tier 1 - RELEASE IMMEDIATELY on idle:
Specialized cubelets (ML models, data loaders, domain experts)
These are expensive and in demand.
Released as soon as the current query is answered.

Tier 2 - KEEP during session:
Context Manager (holds conversation state - cheap, essential)
Safety Monitor (lightweight, always needed)
Session Orchestrator (manages the session lifecycle)

On next query:
If specialized cubelets are needed → re-assemble via sub-pod
Context and safety are already present → instant response for simple queries

On session end (user disconnects or timeout):
All remaining cubelets released
Context state archived (if persistence is configured)
Session Pod fully dissolved

```

**Session idle timeout:** configurable per deployment (e.g., 5 minutes of no queries → dissolve session pod entirely)

### 11.5 Authority Dimensions for AI Interaction

AI interaction tasks use the AI-specific dimensions from the flexible dimension registry:

| Action | Relevant Dimensions | Example |
|---|---|---|
| Answer a factual question | `communication`, `knowledge` | "What is the capital of France?" |
| Perform multi-step reasoning | `reasoning`, `knowledge` | "Explain why this algorithm is O(n log n)" |
| Make a commitment or promise | `communication`, `autonomy` | "I will have this report ready by 5pm" |
| Access sensitive knowledge domain | `knowledge`, `safety` | "Show me the patient medical records" |
| Take autonomous action | `autonomy`, `execution` | "I've scheduled the deployment for midnight" |
| Synthesize across domains | `reasoning`, `knowledge`, `communication` | "Compare quantum and classical approaches to this problem" |

The same MBAC rules apply: the cubelet must have sufficient AV in the relevant dimension(s) to perform the action. If not, the action is denied or escalated through the conflict resolution chain.

---

## 12. Interaction with Other Documents

- **Master Document (00-MYOS-master.md):** Defines the Rubik's Lattice (10×5×15 = 750 cubelets), three model types, and the autopoietic loop that gives MBAC its ethical objective function.
- **Conflict Resolution (02-conflict-resolution.md):** AV comparison is the basis for Level 1 conflict resolution. AV-weighted voting drives Level 2. At Level 3, the System Orchestrator consults STA policies and STK invariants via the CIG's GOVERNS and PROVES edges.
- **Pod Assembly (03-pod-assembly.md):** Cubelet selection uses AV from the flexible dimension registry. The CIG's DEPENDS_ON and PROVES edges constrain which cubelets can be assembled together. STSol templates define which cubelet positions are required.
- **Cubelet I/O Contract (04-cubelet-io-contract.md):** The typed data pipeline follows the framework order (STA → STI → STD → STF → STK). Each pipeline edge carries type information validated by the schema registry.
- **Verification & Audit (06-verification-audit.md):** All authority decisions (AV checks + STK invariant evaluations) are logged on-chain. STK invariant satisfaction records include which PROVES edges were verified.
- **Boot Trust Chain (07-boot-trust-chain.md):** The Authority Engine loads the dimension registry, STK invariant definitions, and action authority map from signed configuration at boot. The cubelet registry (750 entries with invariant bindings) is loaded from the CIG.
- **Knowledge Base (12-knowledge-base.md):** The `knowledge` dimension AV governs write access to the Knowledge Base. Agents must have sufficient knowledge AV to create or update KB entries. KB confidence scores use the same 0–1000 scale and diminishing-returns math as Authority Values. The KB's structural backbone is the CIG (same Neo4j instance).
