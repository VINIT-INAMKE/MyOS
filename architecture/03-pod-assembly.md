# MYOS Pod Assembly Protocol - Task Decomposition, Cubelet Selection & Pod Formation

## Version: 2.0 | Status: LOCKED

---

## 1. Overview

Pod assembly is the process by which the System Orchestrator transforms an incoming task into a functioning team of cubelets, governed by a Pod Orchestrator, ready to execute. This document defines every step of that process - from task arrival to pod activation.

Pod assembly is **graph-informed**: the Cubelet Interaction Graph (CIG) provides the structural backbone for cubelet selection. Rather than blind scoring across the 750-cubelet pool, the system uses **STSol templates** (pre-defined vertical cross-sections of the Rubik's Lattice) to identify which cubelet positions are required, then selects the best available cubelets to fill those positions. The CIG's DEPENDS_ON, PROVES, and GOVERNS edges constrain which cubelets can work together.

The protocol is designed around six principles:

1. **No duplicate tasks** - every active task is unique in the system
2. **Graph-informed teams** - the CIG defines which cubelets can work together; selection follows structural relationships
3. **Best available within structure** - among structurally valid candidates, cubelets are selected by AV with compatibility scoring
4. **Fair rotation** - high-performing cubelets don't monopolize all work
5. **Graceful degradation** - if the perfect cubelet isn't available, the system adapts
6. **Priority-aware assembly** - urgent tasks assemble fast, non-urgent tasks wait for ideal teams

---

## 2. Cubelet Model

### 2.1 What is a Cubelet?

A cubelet is the atomic execution unit in MYOS. There are **750 cubelets** in the system, organized as a **Rubik's Lattice**: 10 domain stages × 5 framework layers × 15 cubelets per cell. Each cubelet has a **unique identity** defined by its lattice position (e.g., `STI-3-B` = Stage 3, Interpretation framework, index B = "Explainability Logic Model").

A cubelet has defined inputs, defined outputs, and a single responsibility. Its **lattice position is permanent**; its model is a versioned artifact deployed to that position.

### 2.2 Cubelet Types

| Type                  | Description                                                                      | Determinism                                                | State                                         | Authority Profile                                                     | Isolation Tier | Approximate Count |
| --------------------- | -------------------------------------------------------------------------------- | ---------------------------------------------------------- | --------------------------------------------- | --------------------------------------------------------------------- | -------------- | ----------------- |
| **Math-Bound**        | Rule engines, ZK proofs, scoring functions, formal verification, crypto          | Fully deterministic                                        | Stateless                                     | Typically high execution AV, earned through consistent correct output | Tier 1 (Wasm + Unikernel) | ~350-400 |
| **ML/DL Model**       | Classification, ranking, anomaly detection, NLI, semantic matching, MoE gating   | Near-perfect (fixed weights, fixed input → fixed output)   | Stateless (model weights are read-only)       | AV earned through inference accuracy and reliability                  | Tier 1-2 (Wasm) | ~200-250 |
| **LLM Agent**         | Novel reasoning, intent interpretation, synthesis, user interaction               | Non-deterministic (generative)                             | Stateful (conversation history, task context) | Most volatile AV - highest reward potential and highest penalty risk  | Tier 3 (container) | ~100-150 |

**Framework-type alignment** (cubelets naturally cluster by model type within frameworks):

| Framework | Primary Model Type | Rationale |
| --------- | ------------------ | --------- |
| **STK** (Kernel) | Math-bound (all 150) | Invariants are formal constraints - proofs, not inference |
| **STF** (Fabric) | Math-bound (most of 150) | Protocol logic, crypto bridges, routing algorithms |
| **STI** (Interpretation) | ML/DL (most of 150) | Classification, ranking, reasoning - trained model territory |
| **STA** (Abstraction) | Mix (math-bound + LLM) | Formalized policy rules + novel policy reasoning |
| **STD** (Deliverance) | Mix (all three) | Pure services + ML inference + LLM runtimes |

### 2.3 Cubelet Exclusivity

**All cubelets are exclusive.** A cubelet can belong to only one pod at a time. It cannot be shared.

```
RULE: If Pod A has Cubelet C42, then Pod B CANNOT use Cubelet C42.
      Pod B must find another cubelet with similar capabilities.
```

This means the system must maintain a pool of cubelets with overlapping capabilities - multiple cubelets that can perform similar functions, even if at different AV levels.

### 2.4 Cubelet Manifest

Every cubelet registers a manifest at system boot:

```
cubelet_manifest {
    cubelet_id:         UUID
    cubelet_type:       math_bound | ml_dl | llm_agent
    name:               String
    version:            String

    // Lattice identity (permanent - defines the cubelet's role in the architecture)
    lattice_id:         String          // e.g., "STI-3-B"
    framework:          STA | STI | STD | STF | STK
    stage:              int             // 0-9
    index:              char            // A-O

    // Model identity (versioned - changes on retraining/update)
    model_hash:         BLAKE3          // hash of deployed model artifact
    model_version:      String          // semver

    // CIG relationships (loaded from graph at boot)
    cig_edges: {
        depends_on:     [CubeletId]     // cubelets this one needs
        proves:         [CubeletId]     // STK invariants this one satisfies
        governed_by:    [CubeletId]     // STA cubelets that constrain this one
        informs:        [CubeletId]     // cubelets this one feeds data to
    }

    // STK invariant bindings
    stk_invariants:     [InvariantId]   // e.g., ["safety.refusal_on_redline", "fairness<=θ"]

    // STF fabric thread bindings
    stf_threads:        [FabricId]      // e.g., ["EthosLedger", "ResearchLedger"]

    // Capability declaration
    capabilities: [{
        capability_id:  String          // e.g., "vision.object_detection"
        input_schema:   SchemaDefinition
        output_schema:  SchemaDefinition
        constraints: {
            max_input_size:     int     // bytes
            max_execution_time: int     // milliseconds
            memory_required:    int     // bytes
            deterministic:      bool
        }
    }]

    // Current state
    authority_vector:   AuthorityVector  // from Authority Engine
    status:             available | locked | cooldown | error | disabled
    locked_by:          PodId | null
    cooldown_until:     Timestamp | null

    // Performance history (summary)
    total_tasks:        int
    success_rate:       float
    avg_execution_time: Duration

    // Compatibility data
    compatibility_scores: {
        cubelet_id: float       // historical co-pod performance scores
        ...
    }
}
```

### 2.5 Cubelet Lifecycle in a Pod

```
available
    │
    ├──selected──→ locked (exclusive to one pod)
    │                 │
    │                 ├──task completes──→ released
    │                 │                      │
    │                 │                      ├──→ cooldown (rotation period)
    │                 │                      │       │
    │                 │                      │       └──cooldown expires──→ available
    │                 │                      │
    │                 │                      └──no cooldown needed──→ available
    │                 │
    │                 └──pod dissolved (failure/abort)──→ released → available
    │
    └──error detected──→ error (requires recovery)
                            │
                            └──recovered──→ available
```

### 2.6 Cooldown (Rotation Policy)

After a cubelet completes a task and is released from a pod, it enters a **cooldown period** before it can be selected again. This prevents the same high-AV cubelets from being monopolized.

```
cooldown_duration = base_cooldown × (1 + consecutive_task_count / max_consecutive)

Where:
  base_cooldown:          configurable (e.g., 5 seconds)
  consecutive_task_count: how many tasks this cubelet has done back-to-back
  max_consecutive:        threshold after which cooldown grows (e.g., 5)
```

**Examples:**

| Consecutive Tasks | Cooldown Duration (base = 5s) |
| ----------------- | ----------------------------- |
| 1                 | 6.0s                          |
| 2                 | 7.0s                          |
| 3                 | 8.0s                          |
| 5                 | 10.0s                         |
| 10                | 15.0s                         |

A cubelet that has been idle for a long time has `consecutive_task_count = 0` and gets the minimum cooldown.

**Cooldown resets to 0 when:**

- The cubelet has been idle (not selected) for longer than `cooldown_reset_threshold` (configurable, e.g., 60 seconds)

---

## 3. Task Uniqueness - Three-Layer Deduplication

### 3.1 Why Three Layers?

Tasks arrive from various sources and contexts. A single dedup mechanism is insufficient:

- **Layer 1 (Structural)** prevents overlap at the planning stage
- **Layer 2 (Content)** catches duplicate submissions from different sources
- **Layer 3 (Runtime)** prevents race conditions during concurrent pod assembly

### 3.2 Layer 1 - Task Decomposition (Structural Uniqueness)

When a high-level goal arrives at the System Orchestrator, it is decomposed into independent, non-overlapping sub-tasks.

```
Goal: "Monitor all reactor temperatures"

Decomposition:
  Sub-task T1: "Monitor temperature in Zone 1"  (scope: zone_1)
  Sub-task T2: "Monitor temperature in Zone 2"  (scope: zone_2)
  Sub-task T3: "Monitor temperature in Zone 3"  (scope: zone_3)
```

Each sub-task has:

- A unique scope/context that distinguishes it from other sub-tasks
- No overlapping responsibility with any other sub-task
- Independent success/failure criteria

**Decomposition rules:**

1. Sub-tasks must be **fully independent** - completing or failing one does not affect another
2. Sub-tasks must cover the **entire goal** - no gaps
3. Sub-tasks must not **overlap** - no shared scope or responsibility
4. The decomposition is logged as part of the task record

**Who performs decomposition?**
The System Orchestrator performs decomposition. For complex goals, it may use an LLM-infused cubelet (a planner cubelet) to assist with decomposition, but the System Orchestrator validates and approves the decomposition before proceeding.

### 3.3 Layer 2 - Task Identity (Content-Based Deduplication)

Each sub-task is assigned a **task identity** based on its content.

```
task_id = hash(task_type + context_parameters)

Examples:
  hash("monitor_temp" + "zone_1") → 0xABC1
  hash("monitor_temp" + "zone_2") → 0xABC2  ← different (different zone)
  hash("monitor_temp" + "zone_1") → 0xABC1  ← DUPLICATE (same zone)
```

**Hash function:** Cryptographic hash (SHA-256 or BLAKE3 via qudag-crypto) over the canonical serialization of the task descriptor.

**Dedup check:**

```
on_new_task(task):
    task_id = hash(task.type + task.context)
    if task_registry.contains(task_id):
        if task_registry[task_id].status == active:
            → REJECT: "Duplicate task already active"
        elif task_registry[task_id].status == completed:
            → ALLOW: task can be re-executed (new instance)
    else:
        → ALLOW: new unique task
```

**Key distinction:** Same task type with different parameters is NOT a duplicate. Same task type with same parameters IS a duplicate (if the previous instance is still active).

### 3.4 Layer 3 - Task Registry (Runtime Locking)

The System Orchestrator maintains a **global task registry** - a live map of all active tasks and their assigned pods.

```
task_registry {
    task_id: {
        task_descriptor:    TaskDescriptor
        status:             pending | assembling | active | completing | completed | failed
        assigned_pod:       PodId | null
        created_at:         Timestamp
        started_at:         Timestamp | null
        completed_at:       Timestamp | null
        locked:             bool
    }
}
```

**Locking protocol:**

```
1. System Orchestrator receives task
2. Compute task_id (Layer 2 hash)
3. Attempt to acquire lock on task_id in registry
   - If lock acquired → proceed with pod assembly
   - If already locked → reject (another assembly is in progress)
4. Lock is released when:
   - Pod assembly completes → task status = active (lock held by pod)
   - Pod assembly fails → task status = failed (lock released)
   - Pod completes task → task status = completed (lock released)
```

This prevents race conditions where two parallel assembly processes try to create pods for the same task.

---

## 4. Cubelet Selection Algorithm

### 4.1 Overview

Once a task is validated and locked, the System Orchestrator must select cubelets to fill the pod. Selection follows a **two-phase process**:

**Phase 1 - STSol template resolution (graph-informed):**
1. **Task → Stage mapping** - determine which lattice stage(s) the task belongs to
2. **STSol template lookup** - find the pre-defined pod template for this task type
3. **CIG edge resolution** - follow DEPENDS_ON, PROVES, GOVERNS edges to identify all required cubelet positions

**Phase 2 - Candidate scoring (AV + compatibility):**
4. **Capability match** - cubelet must have the required capability at the specified lattice position
5. **Authority Value** - higher AV on the relevant dimension is preferred
6. **Compatibility score** - cubelets that have worked well together before get a bonus
7. **Availability** - cubelet must be available (not locked, not in cooldown)
8. **Rotation** - cubelets in cooldown are skipped

### 4.2 STSol Template Model

Each task type maps to an **STSol template** - a vertical cross-section of the lattice defining which cubelet positions are required.

```
stsol_template {
    template_id:        String          // e.g., "Env-MRV-Pod"
    primary_stage:      int             // e.g., 8 (Environment)
    description:        String

    // Required cubelet positions (from CIG)
    required_positions: [
        {
            lattice_id:         "STA-8-A"   // Planetary Stewardship Charter
            framework:          STA
            role:               "governing_policy"
            priority:           required
        },
        {
            lattice_id:         "STI-8-A"   // Climate Data Logic
            framework:          STI
            role:               "reasoning_engine"
            priority:           required
        },
        {
            lattice_id:         "STD-8-A"   // MRV Runtime
            framework:          STD
            role:               "execution"
            priority:           required
        },
        {
            lattice_id:         "STF-8-A"   // KarbonLedger connector
            framework:          STF
            role:               "fabric_connector"
            priority:           required
        },
        {
            lattice_id:         "STK-8-A"   // MRV Verification Invariant
            framework:          STK
            role:               "proof_checker"
            priority:           required
        }
    ]

    // Cross-stage dependencies (from CIG diagonal edges)
    diagonal_dependencies: [
        {
            lattice_id:         "STD-4-A"   // Sensor Attestation Runtime
            role:               "data_source"
            priority:           preferred    // can degrade without it
        }
    ]

    // STK invariants that must be satisfied
    stk_required:       ["MRV.verified", "planetary_boundary<=1", "device.attested"]
}
```

**Fallback for unrecognized tasks:** If no STSol template matches, the System Orchestrator falls back to capability-based slot filling (legacy mode). This uses the traditional capability slot model:

```
task_capability_requirements {
    task_id:    UUID
    slots: [
        {
            slot_id:            "vision"
            required_capability: "vision.object_detection"
            required_av_threshold: 300       // minimum AV in relevant dimension
            relevant_dimension:  "execution"
            priority:            required | preferred | optional
        },
        {
            slot_id:            "planning"
            required_capability: "path.planning"
            required_av_threshold: 400
            relevant_dimension:  "execution"
            priority:            required
        },
        {
            slot_id:            "safety_monitor"
            required_capability: "safety.anomaly_detection"
            required_av_threshold: 500
            relevant_dimension:  "safety"
            priority:            required
        }
    ]
}
```

**Slot priorities:**

- `required` - pod cannot start without this slot filled
- `preferred` - pod can start without it but is degraded
- `optional` - nice to have, no degradation if missing

### 4.3 Candidate Scoring

For each capability slot, the System Orchestrator computes a **selection score** for every eligible cubelet.

```
selection_score(cubelet, slot, other_selected_cubelets) =
    w1 × cubelet.AV[slot.relevant_dimension]
  + w2 × compatibility_bonus(cubelet, other_selected_cubelets)
  + w3 × cubelet.success_rate
  - w4 × cubelet.recent_task_load    // penalize overworked cubelets

Where:
  w1 = 0.50  (AV weight - most important)
  w2 = 0.25  (compatibility - team chemistry matters)
  w3 = 0.15  (track record)
  w4 = 0.10  (load balancing)
```

### 4.4 Compatibility Bonus

Compatibility is measured by historical co-pod performance. When two cubelets have previously been in the same pod and the pod succeeded, their compatibility score increases.

```
compatibility_bonus(cubelet_X, selected_cubelets) =
    average(
        cubelet_X.compatibility_scores[c.cubelet_id]
        for c in selected_cubelets
        if c.cubelet_id in cubelet_X.compatibility_scores
    )

If cubelet_X has never worked with any of the selected cubelets:
    compatibility_bonus = 0 (neutral, not negative)
```

**Compatibility score update (after pod completion):**

```
on_pod_complete(pod):
    for each pair (cubelet_A, cubelet_B) in pod.cubelets:
        if pod.outcome == success:
            cubelet_A.compatibility_scores[cubelet_B.id] += compat_reward
            cubelet_B.compatibility_scores[cubelet_A.id] += compat_reward
        elif pod.outcome == failure:
            cubelet_A.compatibility_scores[cubelet_B.id] -= compat_penalty
            cubelet_B.compatibility_scores[cubelet_A.id] -= compat_penalty
```

### 4.5 Selection Algorithm (Per Slot)

```
select_cubelet(slot, already_selected_cubelets):

    1. Find all cubelets with matching capability:
       candidates = cubelets.filter(c =>
           c.capabilities.contains(slot.required_capability)
       )

    2. Filter by availability:
       candidates = candidates.filter(c =>
           c.status == available
           AND c.cooldown_until < now()
       )

    3. Filter by AV threshold:
       qualified = candidates.filter(c =>
           c.AV[slot.relevant_dimension] >= slot.required_av_threshold
       )

    4. If qualified is empty:
       → mark this slot for deadline-wait (see Section 5)
       → use all candidates (including below-threshold) as fallback pool

    5. Score all candidates:
       for each c in (qualified or fallback_pool):
           c.score = selection_score(c, slot, already_selected_cubelets)

    6. Sort by score descending

    7. Select top candidate

    8. Return selected cubelet (or null if no candidates at all)
```

### 4.6 Slot Fill Order

Slots are filled in order of priority and scarcity:

```
fill_order = sort(slots, by = (
    priority DESC,        // required > preferred > optional
    candidate_count ASC   // scarcer capabilities first
))
```

**Why scarce capabilities first?** If a rare capability has only 2 eligible cubelets, fill that slot first before those cubelets get locked by another concurrent assembly. Common capabilities (many eligible cubelets) can wait.

---

## 5. Deadline-Wait and Substitution

### 5.1 Problem

The ideal cubelet for a slot may be currently locked in another pod. The system must decide: wait for it, or substitute with a less ideal cubelet.

### 5.2 Deadline Calculation

```
deadline = base_wait_time × (1 / priority_weight) × scarcity_factor

Where:
  base_wait_time:   configurable (e.g., 10 seconds)
  priority_weight:  task priority (higher priority → shorter deadline)
      critical:   4.0  → deadline = base × 0.25
      high:       2.0  → deadline = base × 0.50
      medium:     1.0  → deadline = base × 1.00
      low:        0.5  → deadline = base × 2.00

  scarcity_factor:  total_cubelets_with_capability / available_cubelets_with_capability
      If 10 exist, 5 available: factor = 10/5 = 2.0 (longer wait, scarce now)
      If 10 exist, 9 available: factor = 10/9 = 1.1 (short wait, plenty available)
      If 10 exist, 0 available: factor = 10/1 = 10.0 (long wait, all busy - capped)
      Factor is capped at max_scarcity_factor (e.g., 10.0)
```

**Examples (base_wait_time = 10s):**

| Priority | Scarcity Factor        | Deadline |
| -------- | ---------------------- | -------- |
| Critical | 1.0 (plenty available) | 2.5s     |
| Critical | 5.0 (scarce)           | 12.5s    |
| High     | 1.0                    | 5.0s     |
| High     | 3.0                    | 15.0s    |
| Medium   | 1.0                    | 10.0s    |
| Low      | 1.0                    | 20.0s    |
| Low      | 5.0                    | 100.0s   |

### 5.3 During the Wait

While waiting for the ideal cubelet to become available:

```
on_cubelet_released(cubelet):
    if cubelet matches a waiting slot:
        if cubelet.AV[relevant_dim] >= slot.required_av_threshold:
            → cancel wait timer
            → select this cubelet
            → resume pod assembly
```

The system monitors cubelet releases in real-time. As soon as a matching cubelet becomes available, it is claimed.

### 5.4 Deadline Expiry - Substitution

If the deadline expires without the ideal cubelet becoming available:

```
on_deadline_expired(slot):
    1. Take the best available cubelet from the fallback pool
       (candidates that are available but below the AV threshold)

    2. If fallback pool is empty:
       a. If slot.priority == required:
          → pod assembly FAILS
          → task returns to queue
          → retry after backoff period
       b. If slot.priority == preferred:
          → proceed without this slot
          → mark pod as "degraded" (preferred capability missing)
       c. If slot.priority == optional:
          → proceed without this slot
          → no degradation flag

    3. If substitute cubelet is selected:
       → mark slot as "degraded" in pod manifest
       → log substitution event:
          {
              slot_id:              String
              ideal_av_threshold:   float
              substitute_cubelet:   CubeletId
              substitute_av:        float
              av_gap:               float (threshold - substitute_av)
              reason:               "deadline_expired"
          }
```

### 5.5 Degraded Pod Behavior

A pod with one or more degraded slots operates normally but with additional monitoring:

```
degraded_pod {
    degraded_slots: [{
        slot_id:          String
        degradation_type: substituted | missing
        original_requirement: AV threshold
        actual_av:        float | null
    }]

    additional_monitoring:
        - Pod Orchestrator checks degraded cubelet outputs more frequently
        - Confidence thresholds for actions are raised (more conservative)
        - System Orchestrator is notified (may provide replacement later)

    replacement_policy:
        - If the ideal cubelet becomes available during pod execution:
            → System Orchestrator notifies Pod Orchestrator
            → Pod Orchestrator can hot-swap the degraded cubelet
              (if the task state allows safe replacement)
            → Hot-swap is optional, not mandatory
}
```

---

## 6. Assembly Mode - Priority-Driven Strategy

### 6.1 Two Assembly Modes

The assembly strategy adapts based on task priority and cubelet availability.

```
HIGH PRIORITY (critical, high)
──────────────────────────────
  Mode: PARALLEL ASSEMBLY
  - Lock available cubelets IMMEDIATELY as they are selected
  - Fill multiple slots concurrently
  - Short deadline for missing slots
  - Substitute aggressively when deadline expires
  - Rationale: urgent task, can't wait, can't lose available cubelets to other assemblies

LOW PRIORITY (medium, low)
──────────────────────────
  Mode: SEQUENTIAL ASSEMBLY
  - Mark cubelets as CANDIDATES (don't lock yet)
  - Fill slots one at a time, in priority order
  - Long deadline for missing slots
  - Wait for ideal cubelets before locking
  - Final lock: all cubelets locked at once when all slots resolved
  - Rationale: no rush, don't hog resources, higher-priority tasks may need these cubelets
```

### 6.2 Why Not Always Parallel?

Parallel assembly locks cubelets immediately. If a low-priority task locks 5 cubelets while a critical task arrives 2 seconds later, the critical task may not find the cubelets it needs. Sequential assembly for low-priority tasks prevents resource hoarding.

### 6.3 Why Not Always Sequential?

Sequential assembly delays locking. If a high-priority task selects a cubelet as "candidate" but doesn't lock it, another concurrent assembly can steal it. For urgent tasks, locking immediately is essential.

### 6.4 Assembly Mode Decision

```
determine_assembly_mode(task):
    if task.priority in [critical, high]:
        return PARALLEL
    elif task.priority in [medium, low]:
        return SEQUENTIAL
```

### 6.5 Parallel Assembly Flow

```
parallel_assembly(task):
    slots = task.capability_requirements.slots
    sort slots by (priority DESC, scarcity ASC)

    for each slot IN PARALLEL:
        cubelet = select_cubelet(slot, already_selected)
        if cubelet found:
            LOCK cubelet immediately (exclusive)
            add to pod_roster
        else:
            start deadline_wait(slot)
            on_cubelet_available: lock and add
            on_deadline_expired: substitute or fail

    when all required slots filled (or failed):
        if all required slots filled:
            → instantiate pod
        else:
            → release all locked cubelets
            → requeue task with backoff
```

### 6.6 Sequential Assembly Flow

```
sequential_assembly(task):
    slots = task.capability_requirements.slots
    sort slots by (priority DESC, scarcity ASC)
    candidates = []

    for each slot IN ORDER:
        cubelet = select_cubelet(slot, candidates)
        if cubelet found:
            mark as CANDIDATE (don't lock)
            add to candidates list
        else:
            start deadline_wait(slot)
            on_cubelet_available: mark as candidate
            on_deadline_expired: substitute or fail

    when all slots resolved:
        // Final lock phase - atomic
        for each candidate:
            if candidate.status == available:
                LOCK candidate
            else:
                // Candidate was taken by a higher-priority assembly
                // Find next best available cubelet for this slot
                replacement = find_replacement(slot)
                if replacement:
                    LOCK replacement
                else:
                    → assembly fails for this slot

        if all required slots locked:
            → instantiate pod
        else:
            → release all locked cubelets
            → requeue task with backoff
```

---

## 7. Pod Instantiation

### 7.1 Pod Manifest

Once all required slots are filled and cubelets are locked, the pod is instantiated.

```
pod_manifest {
    pod_id:             UUID
    task_id:            UUID
    task_descriptor:    TaskDescriptor
    created_at:         Timestamp

    // Team composition
    cubelets: [{
        cubelet_id:     UUID
        lattice_id:     String               // e.g., "STI-3-B"
        slot_id:        String
        capability:     String
        model_hash:     BLAKE3               // deployed model version
        av_at_selection: AuthorityVector     // snapshot at selection time
        is_degraded:    bool                  // substitution flag
    }]

    // STSol template used (if graph-informed assembly)
    stsol_template:     String | null        // e.g., "Env-MRV-Pod" or null for legacy

    // STK invariants applicable to this pod
    stk_invariants:     [InvariantId]        // union of all cubelets' stk_invariants

    // Pod Orchestrator assignment
    pod_orchestrator: {
        orch_id:        UUID                  // Pod Orch is NOT a cubelet (distinct entity)
        template_origin: String               // e.g., "data-pipeline-orch"
        av_at_assignment: AuthorityVector
    }

    // Degradation status
    degraded_slots:     [DegradedSlot]        // empty if fully qualified
    missing_preferred:  [SlotId]              // preferred slots not filled
    missing_optional:   [SlotId]              // optional slots not filled

    // Resource allocation
    time_budget:        Duration
    memory_budget:      int (bytes)
    authority_budget: {
        max_authority_per_action:  float       // per-dimension caps
    }

    // Assembly metadata
    assembly_mode:      parallel | sequential
    assembly_duration:  Duration
    substitutions_made: int
    compatibility_score: float                // average team compatibility
}
```

### 7.2 Pod Orchestrator Assignment

The Pod Orchestrator is **NOT a cubelet** from the 750 pool. It is a distinct entity class - always LLM-based - created by **System, Domain, or Deployment Orchestrators** using a **template + registry hybrid** model.

```
Pod Orchestrator assignment:
  1. Creating orchestrator (System, Domain, or Deployment) checks the Pod Orch REGISTRY:
     - Is there an existing Pod Orch with relevant experience (similar task type)?
     - If YES → clone/reuse it (AV carries over with decay for idle time)
     - If NO  → instantiate from a TEMPLATE (AV set to template defaults)

  2. Templates: "data-pipeline-orch", "ai-session-orch", "safety-critical-orch", "hybrid-orch"
     Selected based on task type.

  3. LLM model: Open-source (Llama 3.1 8B / Phi-3 via Ollama/Groq)
     System Orchestrator uses paid model (Claude Opus class).
     Domain Orchestrators use paid model (Claude Sonnet class).
     Deployment Orchestrators use open-source model.
     See 10-node-topology-orchestrator-hierarchy.md for full LLM assignment.

  4. After task completion, Pod Orch is saved/updated in the registry for future reuse.

  WHO CREATES PODS:
     System Orch:      System-level tasks (global coordination, emergency)
     Domain Orch:      Domain-specific tasks (Healthcare, DeFi, etc.)
     Deployment Orch:  Infrastructure tasks (scaling, updates, health checks)
```

**Pod Orchestrator requirements:**

- Must have AV above the Pod Orchestrator floor (400) on ALL dimensions
- Prefers Pod Orchs with high compatibility scores with the assigned cubelets
- NOT subject to cubelet cooldown/rotation rules (can be reused immediately)

See **Pod Orchestrator document (05-pod-orchestrator.md)** for full lifecycle, creation, and runtime behavior details.

### 7.3 Pod States

```
pending       → task accepted, assembly not started
assembling    → cubelet selection in progress
active        → all cubelets locked, Pod Orchestrator running, task executing
completing    → task finished, results being validated
completed     → task successfully done, cubelets released
failed        → task failed, cubelets released, logged
dissolved     → pod manually dissolved (by System Orch or human)
```

```
State transitions:
  pending → assembling       (System Orch starts assembly)
  assembling → active        (all required slots filled)
  assembling → failed        (required slot unfillable after all retries)
  active → completing        (task execution finished)
  completing → completed     (results validated)
  completing → failed        (results invalid)
  active → failed            (unrecoverable error during execution)
  active → dissolved         (System Orch or human dissolves pod)
  any → dissolved            (emergency dissolution)
```

### 7.4 Post-Assembly Actions

After pod instantiation:

```
1. Notify all cubelets of their pod assignment and teammates
2. Pod Orchestrator receives full pod manifest and task descriptor
3. Pod Orchestrator establishes intra-pod communication channels
4. Pod Orchestrator creates execution plan for the task
5. Task registry updated: status = active, assigned_pod = pod_id
6. System Orchestrator monitors pod health externally
7. Verification & Audit receives pod_instantiation_event
```

---

## 8. Cubelet Communication Within Pod

### 8.1 Intra-Pod Communication

Cubelets within a pod can communicate **only with their teammates**. They cannot see or address cubelets outside their pod.

```
Communication rules:
  ✅ Cubelet A (Pod 1) → Cubelet B (Pod 1)     // same pod, allowed
  ✅ Cubelet A (Pod 1) → Pod Orch (Pod 1)      // same pod, allowed
  ❌ Cubelet A (Pod 1) → Cubelet X (Pod 2)     // different pod, BLOCKED
  ❌ Cubelet A (Pod 1) → Pod Orch (Pod 2)      // different pod, BLOCKED
```

### 8.2 Cross-Pod Communication

If a cubelet needs information or action from outside its pod:

```
1. Cubelet sends request to its Pod Orchestrator
2. Pod Orchestrator evaluates the request
3. Pod Orchestrator sends the request to System Orchestrator
   (via capability-based routing)
4. System Orchestrator identifies which pod has the relevant capability
5. System Orchestrator forwards request to target Pod Orchestrator
6. Target Pod Orchestrator routes to the appropriate cubelet
7. Response follows the reverse path

Flow:
  Cubelet A (Pod 1)
      → Pod Orch (Pod 1)
          → System Orchestrator
              → Pod Orch (Pod 2)
                  → Cubelet X (Pod 2)
                  ← response
              ← response
          ← response
      ← response
  Cubelet A receives response
```

### 8.3 Capability-Based Routing

The System Orchestrator does not expose pod internals. Cross-pod communication is **capability-addressed**, not entity-addressed.

```
cross_pod_request {
    from_pod:           PodId
    requested_capability: "vision.depth_estimation"
    payload:            RequestData
    priority:           normal | urgent
    timeout:            Duration
}

System Orchestrator:
    1. Look up which active pod has a cubelet with "vision.depth_estimation"
    2. If found: forward to that pod's orchestrator
    3. If not found: return "capability_not_available"
    4. If multiple pods have it: pick the one with the highest-AV cubelet for that capability
```

The requesting cubelet never knows which pod or which cubelet served its request. This maintains pod isolation.

---

## 9. Pod Dissolution

### 9.1 Normal Dissolution (Task Complete)

```
on_task_complete(pod):
    1. Pod Orchestrator validates results
    2. Results sent to System Orchestrator
    3. System Orchestrator validates and accepts
    4. Authority Engine updates:
       - Success event for each cubelet (AV reward)
       - Success event for Pod Orchestrator
       - Compatibility scores updated for all pairs
    5. All cubelets released (status → cooldown → available)
    6. Pod Orchestrator released
    7. Task registry updated: status = completed
    8. Pod record archived in audit log
    9. Pod dissolved
```

### 9.2 Failure Dissolution

```
on_task_failure(pod):
    1. Pod Orchestrator reports failure reason
    2. System Orchestrator evaluates:
       - Which cubelet(s) caused the failure?
       - Was it a cubelet error, a coordination error, or external?
    3. Authority Engine updates:
       - Violation event for responsible cubelet(s) (AV penalty)
       - If Pod Orchestrator's coordination failed: penalty for Pod Orch
       - If external factor: no penalty (but risk event recorded)
    4. All cubelets released
    5. Task may be requeued (with different cubelets)
    6. Pod dissolved
```

### 9.3 Emergency Dissolution

```
on_emergency_dissolve(pod):
    1. Triggered by System Orchestrator or human operator
    2. All cubelets immediately stop execution
    3. Current state is snapshot and saved
    4. All cubelets released (no cooldown - immediate availability)
    5. No AV update (emergency dissolution is not a performance event)
    6. Task may be reassigned to a new pod
    7. Pod dissolved
    8. Emergency event logged
```

---

## 10. Session Pods and Sub-Pods (AI Interaction)

MYOS is not limited to physical control tasks. User conversations and AI interactions follow the same pod assembly protocol with specific adaptations.

### 10.1 Session Pod Assembly

When a user starts a conversation, the System Orchestrator assembles a **Session Pod** - a long-lived pod that persists for the session.

```
Session Pod composition (fixed slots):
  REQUIRED:
    - Session Orchestrator    (Pod Orch role, manages conversation flow)
    - Context Manager         (tracks conversation state across queries)
    - Safety Monitor          (filters all outputs before user sees them)
    - Basic LLM cubelet       (handles simple queries directly)

  Selection follows standard cubelet selection algorithm:
    - Highest AV with rotation
    - Compatibility scoring between selected cubelets
    - Threshold check on all dimensions
```

Session Pods differ from standard task pods:

- **Long-lived** - they don't dissolve after one query
- **Multi-query** - they handle a stream of queries, not a single task
- **Tiered release** - specialized cubelets are released on idle, core cubelets are kept

### 10.2 Query Routing - Capability Check

When a query arrives at the Session Pod, routing is deterministic:

```
on_query(query):
    required_capabilities = extract_capabilities(query)
    session_capabilities = session_pod.available_capabilities()

    missing = required_capabilities - session_capabilities

    if missing is empty:
        → handle within Session Pod (no sub-pod)
    else:
        → request sub-pod from System Orchestrator
          with capability_requirements = missing
```

This is a **capability check**, not an LLM judgment call. If the Session Pod lacks a required capability, a sub-pod is spawned. No ambiguity.

### 10.3 Sub-Pod Spawning

Sub-pods are assembled using the standard pod assembly protocol (Sections 3-7 above) with these constraints:

```
sub_pod_constraints {
    // Authority ceiling: sub-pod cannot exceed Session Pod's authority
    max_authority_per_dimension: session_pod.authority_vector

    // No nesting: sub-pods cannot spawn sub-sub-pods
    can_spawn_children: false

    // Ephemeral: sub-pod dissolves after query completion
    lifecycle: ephemeral

    // Communication: results go ONLY to the Session Pod
    allowed_communication: [session_pod_id]
}
```

### 10.3.1 Sub-Pod Context Injection Protocol

When spawning a sub-pod, the Session Pod doesn't just send a task description - it injects a structured **context package** so the sub-pod operates with full awareness of the session's established reasoning, not in a vacuum.

```
sub_pod_context_package {
    // The sub-task itself
    task_description:       String
    expected_output_type:   TypeSpec

    // Session context (from Session Memory KB)
    injected_context: {
        // Active goals - what the session is trying to achieve overall
        session_goals:      [String]

        // Established constraints - decisions and rules from earlier in the session
        // that the sub-pod must not contradict
        constraints:        [String]

        // Relevant decision chain - the reasoning that led to this sub-task
        // (references to decision_context_entries, see 04-cubelet-io-contract.md §4.4)
        decision_chain:     [UUID]

        // Relevant KB entries - pre-fetched knowledge the sub-pod will need
        // (avoids redundant KB queries and ensures consistency with parent's view)
        relevant_kb_entries: [UUID]

        // User preferences - communication style, output format, constraints
        // the user has established in this session
        user_preferences:   [{key: String, value: String}]
    }

    // Boundary rules
    context_rules: {
        // Sub-pod MUST NOT produce results that contradict these constraints
        hard_constraints:   [String]

        // Sub-pod SHOULD align with these but may deviate with justification
        soft_constraints:   [String]

        // If sub-pod encounters a contradiction with injected context,
        // it reports back rather than resolving independently
        contradiction_policy: report_to_parent
    }
}
```

```
CONTEXT INJECTION FLOW:

  Session Pod Orch prepares sub-task:
    1. Formulate task description from user query decomposition
    2. Query Session Memory KB for top-weighted context entries relevant to this sub-task
    3. Collect active goals from latest context checkpoint
    4. Collect constraints from decision context log
    5. Bundle into sub_pod_context_package
    6. Send to System Orch for sub-pod assembly

  Sub-Pod receives context package:
    1. Load injected context into its Pod KB at creation
    2. Treat hard_constraints as inviolable (same as system invariants)
    3. Use relevant_kb_entries before querying the broader KB hierarchy
    4. On contradiction with injected context → report_to_parent (do not resolve locally)

  Sub-Pod returns results:
    1. Result includes output data + any new discoveries
    2. If sub-pod found information that updates/contradicts injected context,
       it flags this explicitly in the result
    3. Session Pod Orch evaluates flags and updates Session Memory KB if warranted

This ensures sub-pods don't produce results that conflict with
established session context, even though they operate independently.
```

### 10.4 Parallel Sub-Pods (Fan-Out)

For complex queries, the Session Pod can spawn multiple sub-pods in parallel:

```
on_complex_query(query):
    sub_tasks = decompose_query(query)  // standard task decomposition

    for each sub_task IN PARALLEL:
        request_sub_pod(sub_task)

    collect all sub_pod results
    synthesize response within Session Pod
    deliver to user through Safety Monitor
```

**Fan-out rules:**

- Session Pod can spawn N sub-pods in parallel (bounded by max_active_sub_pods config)
- Sub-pods run independently - no inter-sub-pod communication
- Session Pod collects and synthesizes all results
- If any sub-pod fails, Session Pod handles gracefully (partial results or retry)

### 10.5 Idle Session - Tiered Cubelet Release

```
TIERED RELEASE POLICY
─────────────────────
IMMEDIATELY on query completion:
  Release all sub-pod cubelets (standard pod dissolution)

On session idle (no new query within idle_window):
  Tier 1 RELEASE: Specialized cubelets in Session Pod (if any)
    - Expensive, in demand, release first
  Tier 2 KEEP: Core cubelets
    - Context Manager (holds conversation state)
    - Safety Monitor (lightweight, always needed)
    - Session Orchestrator (manages lifecycle)
    - Basic LLM cubelet (needed for next query)

On session timeout (no query for session_timeout_duration):
  Release ALL cubelets including core
  Archive context state
  Dissolve Session Pod entirely

On session resume (user returns after timeout):
  Assemble new Session Pod
  Restore archived context (if available)
  Fresh cubelet selection (team may differ from previous session)
```

### 10.6 Configuration - Session Pod Specific

```
session_pod_config {
    idle_window_seconds:        10      // time before releasing tier 1 cubelets
    session_timeout_seconds:    300     // 5 minutes before full dissolution
    max_active_sub_pods:        5       // max parallel sub-pods per session
    context_archive_enabled:    true    // save context on session end
    context_archive_ttl:        3600    // 1 hour before archived context expires
}
```

---

## 11. Configuration

```
pod_assembly_config {
    // Cubelet selection weights
    selection_weights {
        av_weight:              0.50
        compatibility_weight:   0.25
        success_rate_weight:    0.15
        load_balance_weight:    0.10
    }

    // Cooldown / rotation
    cooldown_base_seconds:      5
    cooldown_max_consecutive:   5
    cooldown_reset_threshold:   60    // seconds of idle before cooldown resets

    // Deadline / wait
    deadline_base_seconds:      10
    deadline_max_scarcity:      10.0  // scarcity factor cap
    priority_weights {
        critical:   4.0
        high:       2.0
        medium:     1.0
        low:        0.5
    }

    // Assembly mode
    parallel_priorities:        [critical, high]
    sequential_priorities:      [medium, low]

    // Task dedup
    task_registry_max_size:     10000
    task_hash_algorithm:        "blake3"

    // Pod limits
    max_cubelets_per_pod:       20
    max_active_pods:            50
    max_assembly_retries:       3
    assembly_retry_backoff:     5     // seconds

    // Compatibility
    compatibility_reward:       10.0
    compatibility_penalty:      5.0

    // Quorum for pod orchestrator selection
    pod_orch_min_av_all_dimensions: 400
}
```

---

## 11. Invariants (Must Always Hold)

```
INV-1:  A cubelet belongs to at most one pod at any time
INV-2:  No two active tasks have the same task_id (content hash)
INV-3:  Every required slot must be filled before a pod becomes active
INV-4:  Pod Orchestrator AV ≥ pod_orchestrator_floor on ALL dimensions
INV-5:  Cubelet selection score is deterministic given the same inputs
INV-6:  Parallel assembly locks cubelets immediately; sequential assembly defers locking
INV-7:  Deadline is always finite and bounded (base × max_factor × max_scarcity)
INV-8:  Substituted cubelets are flagged as "degraded" in the pod manifest
INV-9:  Cross-pod communication is capability-addressed, never entity-addressed
INV-10: Pod dissolution always releases all cubelets (no orphaned locks)
INV-11: Cooldown is applied after every pod release (rotation guarantee)
INV-12: Compatibility scores are symmetric (A→B == B→A)
INV-13: Task registry lock is released on both success and failure (no deadlocks)
INV-14: Assembly mode is determined solely by task priority (deterministic)
INV-15: Sub-pods cannot spawn sub-sub-pods (no nesting, fan-out only)
INV-16: Sub-pod authority cannot exceed parent Session Pod authority on any dimension
INV-17: Query routing is a deterministic capability check, not an LLM judgment
INV-18: Session Pod core cubelets (Context Manager, Safety Monitor, Session Orch) are released only on session timeout
INV-19: All sub-pod results pass through the Session Pod's Safety Monitor before reaching the user
INV-20: Sub-pods receive a structured context package (not just a task description) at spawn
INV-21: Sub-pods treat injected hard_constraints as inviolable - violations trigger report_to_parent
INV-22: Sub-pods that encounter contradictions with injected context report back rather than resolving locally
INV-23: STSol template assembly uses CIG DEPENDS_ON edges to resolve all required positions
INV-24: Every pod must include at least one STK cubelet for proof verification (no unproven pods)
INV-25: Cubelet model_hash at selection time must match model_hash at execution time (no mid-pod model swaps)
INV-26: Cubelet lattice position (Framework-Stage-Index) is permanent - only model version changes
INV-27: AV resets to floor when a cubelet's model_hash changes (re-earn trust for updated models)
```

---

## 13. Interaction with Other Documents

- **Master Document (00-MYOS-master.md):** Defines the Rubik's Lattice (10×5×15 = 750), five frameworks, ten stages, three model types, and STSol pod concept. Pod assembly is the runtime instantiation of STSol templates.
- **Authority Model (01-authority-model.md):** Cubelet selection uses AV from the flexible dimension registry. AV updates (rewards/penalties) happen at pod dissolution. STK invariants are enforced at assembly time - a pod cannot activate if its cubelets' required invariants cannot be satisfied. AV resets when cubelet model_hash changes.
- **Conflict Resolution (02-conflict-resolution.md):** Conflicts between cubelets within a pod are resolved using the escalation chain. Cross-pod conflicts route through the Domain Orchestrator, then System Orchestrator (Level 3). The CIG GOVERNS edges inform Level 3 arbitration.
- **Cubelet I/O Contract (04-cubelet-io-contract.md):** The typed data pipeline within a pod follows framework ordering (STA → STI → STD → STF → STK). CIG DEPENDS_ON edges inform DAG construction.
- **Pod Orchestrator (05-pod-orchestrator.md):** Pod Orchestrators are NOT cubelets. They use STSol templates to guide DAG construction and CIG edges to validate pipeline structure.
- **Verification & Audit (06-verification-audit.md):** Every assembly, substitution, dissolution, and task dedup event is logged immutably. STSol template usage and STK invariant satisfaction are recorded on-chain.
- **Knowledge Base (12-knowledge-base.md):** A Pod KB (ephemeral, scoped to the pod) is created at pod assembly and destroyed at pod dissolution. The CIG provides the structural backbone - the Pod KB adds runtime knowledge on top of structural relationships. Session Pods maintain a Session Memory KB that persists for the session lifetime. Discoveries made within a pod can be promoted to Domain or System KB through the Pod Orchestrator.
