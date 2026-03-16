# MYOS Pod Orchestrator - Behavior, Lifecycle & Runtime Management

## Version: 2.0 | Status: LOCKED

---

## 1. Overview

The Pod Orchestrator is the **intelligent coordinator and synthesizer** inside every running pod. It is always an **LLM-based agent** - never a simple rule engine or scheduler. It reasons about pipeline structure, monitors cubelet health, handles failures adaptively, and synthesizes results from parallel branches.

Pod Orchestrators are **NOT cubelets** from the 750 cubelet pool. They are a distinct entity class, created by the System Orchestrator and registered for reuse.

Pod Orchestrators use **STSol templates** and **CIG edges** to guide DAG construction. Rather than reasoning about pipeline structure from scratch, the Pod Orch consults the STSol template for the task type to identify required cubelet positions, then uses CIG DEPENDS_ON and PROVES edges to build the pipeline within structural constraints. The LLM reasoning focuses on optimization within the graph structure, not inventing the structure from scratch.

```
KEY PROPERTIES:
  - Always LLM-powered
  - Created by System Orchestrator (not from cubelet pool)
  - Has its own AV vector (subject to MBAC, floor = 400)
  - Builds and manages the data pipeline DAG
  - Synthesizes results using LLM reasoning
  - Full visibility into pod cubelets' AV
  - Registered in a registry for reuse after task completion
```

---

## 2. Pod Orchestrator Nature

### 2.1 Entity Class

Pod Orchestrators are architecturally distinct from cubelets:

| Property         | Cubelet                                        | Pod Orchestrator                                    |
| ---------------- | ---------------------------------------------- | --------------------------------------------------- |
| Source           | From the 750 cubelet pool                      | Created by System Orchestrator                      |
| Intelligence     | Varies (math function, ML model, or LLM agent) | Always LLM                                          |
| Role             | Data processing (one specific function)        | Coordination, synthesis, management                 |
| AV Floor         | 0                                              | 400                                                 |
| DAG Position     | Node in the pipeline                           | Manager of the pipeline (not a node)                |
| Data Processing  | Yes (its primary job)                          | Synthesis only (merging results, formatting output) |
| Control Messages | Can send to Pod Orch only                      | Can send to cubelets and System Orch                |
| Reuse            | Selected from pool per task                    | Cloned from registry or instantiated from template  |

### 2.2 Always LLM-Based

Pod Orchestrators require LLM capabilities because they:

1. **Build DAGs dynamically** - reasoning about optimal pipeline structure for each specific task, not following a template
2. **Synthesize results** - merging outputs from parallel branches requires understanding and composing coherent results, not concatenating data
3. **Handle failures intelligently** - the tiered failure response requires reasoning about what went wrong and what to do about it
4. **Manage cubelets contextually** - deciding whether to bypass, retry, or replace a cubelet based on the specific situation
5. **Route queries** - deciding whether a session pod can handle a query or needs a sub-pod (as LLM fallback after capability check)

### 2.3 Relationship to Orchestrator Hierarchy

MYOS has a **four-level orchestrator hierarchy** (see 10-node-topology-orchestrator-hierarchy.md):

```
System Orchestrator (1, singleton) → Paid LLM (Claude Opus / GPT-4 class)
  ├── Domain Orchestrators (per vertical) → Paid LLM (Claude Sonnet class)
  └── Deployment Orchestrators (per zone) → Open-source LLM (Llama 3.1 8B / Mistral 7B via Ollama/Groq)
        └── Pod Orchestrators (many, ephemeral) → Open-source LLM (Llama 3.1 8B / Phi-3 via Ollama/Groq)

Domain and Deployment Orchs are PARALLEL PEERS under System Orch.
All three upper levels (System, Domain, Deployment) can create Pod Orchestrators.

LLM COST STRATEGY:
  Paid models for high-stakes decisions (System + Domain Orchs)
  Open-source models for operational tasks (Deployment + Pod Orchs)

AV FLOORS:
  System: 700 > Domain: 600 > Deployment: 500 > Pod: 400 > Cubelet: 0

Model assignment is defined in system configuration at boot.
```

---

## 3. Pod Orchestrator Lifecycle

### 3.1 Creation

Pod Orchestrators are created using a **template + registry hybrid** model.

```
CREATION FLOW:

  1. System Orchestrator needs a Pod Orch for a new pod
  2. Check REGISTRY: is there an existing Pod Orch with relevant experience?
     │
     ├─ YES (registry hit):
     │    Clone the registered Pod Orch
     │    AV carries over with decay applied for idle time
     │    Inherited strategies and learned patterns from previous tasks
     │    Adapted for the new task context
     │
     └─ NO (registry miss):
          Instantiate from TEMPLATE
          AV set to template default values
          Fresh start, no prior experience
          Template selected based on task type
```

### 3.2 Templates

Templates define the base configuration of a Pod Orchestrator.

```
pod_orch_templates {
    "data-pipeline-orch": {
        description:    "Manages linear and branching data processing pipelines"
        base_av: {
            execution: 500, safety: 500, resource: 450, policy: 450,
            reasoning: 500, communication: 400, knowledge: 400, autonomy: 400
        }
        specialization:  "pipeline construction, data flow optimization"
        llm_model:       configured_pod_orch_model   // from system config
    },

    "ai-session-orch": {
        description:    "Manages user conversation sessions with sub-pod spawning"
        base_av: {
            execution: 450, safety: 500, resource: 450, policy: 450,
            reasoning: 550, communication: 550, knowledge: 500, autonomy: 400
        }
        specialization:  "conversation management, query routing, response synthesis"
        llm_model:       configured_pod_orch_model
    },

    "safety-critical-orch": {
        description:    "Manages safety-critical operations with conservative decision-making"
        base_av: {
            execution: 500, safety: 600, resource: 500, policy: 500,
            reasoning: 450, communication: 400, knowledge: 400, autonomy: 400
        }
        specialization:  "safety monitoring, conservative escalation, fail-safe defaults"
        llm_model:       configured_pod_orch_model
    },

    "hybrid-orch": {
        description:    "General-purpose orchestrator for mixed workloads"
        base_av: {
            execution: 500, safety: 500, resource: 450, policy: 450,
            reasoning: 500, communication: 450, knowledge: 450, autonomy: 400
        }
        specialization:  "general coordination, adaptive to task type"
        llm_model:       configured_pod_orch_model
    }
}
```

Templates are defined at system configuration time and are immutable at runtime.

### 3.3 Registry

The Pod Orchestrator registry stores previously used Pod Orchs for reuse.

```
pod_orch_registry {
    entries: [{
        registry_id:        UUID
        template_origin:    String          // which template it was created from
        created_at:         Timestamp
        last_used_at:       Timestamp
        total_tasks_managed: int
        success_rate:       float
        authority_vector:   AuthorityVector  // current AV (with decay applied)
        specialization_history: [String]    // types of tasks it has managed
        compatibility_data: {               // how well it works with specific cubelets
            cubelet_id: float
            ...
        }
    }]
}
```

**Registry lookup criteria:**

```
find_pod_orch(task):
    candidates = registry.filter(entry =>
        entry.template_origin matches task.type
        AND entry.success_rate >= min_success_rate
        AND entry.authority_vector meets floor requirements
    )
    sort candidates by:
        1. Relevance to task type (specialization match)
        2. Success rate
        3. AV on relevant dimensions
        4. Compatibility with assigned cubelets
    return top candidate (or null → use template)
```

### 3.4 AV Behavior

```
REUSED FROM REGISTRY:
  AV carries over from previous usage.
  Decay is applied for idle time since last use:
    AV(now) = max(floor, AV(last_used) - decay_rate × idle_time)
  After the new task, AV is updated based on performance (success/violation/risk).

NEWLY CREATED FROM TEMPLATE:
  AV is set to the template's base_av values.
  After the task, AV is updated based on performance.
  Pod Orch is registered in the registry for future reuse.
```

### 3.5 Dissolution

When a pod is dissolved:

```
1. Pod Orch's AV is updated based on task outcome
2. Pod Orch is saved/updated in the registry
3. Pod Orch's LLM session is terminated
4. Resources are freed
```

Pod Orchestrators are NOT subject to cooldown (unlike cubelets). They can be immediately reused if a matching task arrives.

---

## 4. Core Responsibilities

### 4.1 DAG Construction

The Pod Orchestrator's first job is to build the data pipeline.

```
dag_construction(task, cubelets):
    1. Examine each cubelet's I/O declarations (input/output ports with types)
    2. Reason about the optimal pipeline structure:
       - What is the logical order of operations for this task?
       - Which cubelets should run in sequence?
       - Which can run in parallel (fan-out)?
       - Where do results need to merge (fan-in)?
       - Are any cubelets optional for this specific task?
    3. Construct the DAG:
       - Wire output ports to input ports
       - Validate type compatibility on every edge
       - Define execution order (topological sort of the DAG)
    4. If type mismatch found:
       - Attempt to resolve (is there a type adapter cubelet?)
       - If unresolvable → report assembly error to System Orch
    5. Finalize DAG → pipeline ready for execution
```

### 4.2 Pipeline Execution Management

During execution, the Pod Orchestrator:

```
- Initiates data flow by sending initial input to the first cubelet(s)
- Monitors each cubelet's execution time against its time budget
- Receives control messages from cubelets (status, alerts, errors)
- Makes bypass/insert decisions if needed (within fixed DAG constraints)
- Handles fan-in synchronization (waits for all branches or applies timeout)
- Detects and responds to failures (tiered failure response)
```

### 4.3 Cubelet Monitoring

The Pod Orchestrator has **full visibility** into its cubelets' Authority Values.

```
monitoring_data (per cubelet):
    cubelet_id:         UUID
    authority_vector:   AuthorityVector     // full, real-time AV
    execution_status:   idle | processing | completed | error
    current_task_time:  Duration            // how long current operation is taking
    time_budget_remaining: Duration
    resource_usage: {
        memory:         bytes
        cpu_time:       Duration
    }
    output_confidence:  float | null        // if cubelet reports confidence
    error_count:        int                 // errors during this pod session
    last_output_hash:   Hash | null
```

Pod Orch uses AV visibility for:

- Trusting high-AV cubelet outputs more during synthesis
- Prioritizing high-AV cubelets' alerts
- Deciding which cubelet to trust when outputs conflict (before formal conflict resolution)
- Informing its reasoning about failure response

### 4.4 Result Synthesis

The Pod Orchestrator synthesizes results from the pipeline using its **LLM capabilities**.

```
synthesis_scenarios:

  LINEAR PIPELINE:
    Input → A → B → C → Output
    Pod Orch receives C's output as the final result.
    Synthesis = formatting, validation, adding context.

  FAN-OUT (parallel branches):
    Input → A → [B, C, D] → results
    Pod Orch receives outputs from B, C, D.
    Synthesis = LLM reasons about the outputs:
      - Are they consistent?
      - How to merge them into a coherent result?
      - If contradictory → may trigger conflict resolution
      - Compose a single response that incorporates all perspectives

  MULTI-QUERY SESSION:
    Pod Orch tracks conversation context across multiple queries.
    Synthesis includes context from previous queries.
    Response is contextually aware, not just answering the current query in isolation.
```

Synthesis is NOT a simple concatenation or merge function. It is LLM-driven reasoning about the outputs.

### 4.5 Query Routing (Session Pods)

In session pods (AI interaction), the Pod Orchestrator routes queries:

```
query_routing(query):
    1. CAPABILITY CHECK (deterministic):
       required = extract_required_capabilities(query)
       available = session_pod.capabilities()
       missing = required - available

       if missing is empty:
           → handle within session pod
           DONE

    2. LLM FALLBACK (if capability check is ambiguous):
       Pod Orch LLM evaluates:
         - "Can my current cubelets handle this, or do I need a sub-pod?"
         - "Is this query simple enough for the basic LLM cubelet?"
         - "Does this need specialized knowledge/models?"
       → handle locally OR request sub-pod from System Orch
```

Capability check runs first (fast, deterministic). LLM fallback runs only when the capability check can't clearly determine the answer.

---

## 5. Tiered Failure Response

When a cubelet underperforms or fails during execution, the Pod Orchestrator applies a **tiered response**. Each tier fires only if the previous tier didn't resolve the issue.

### 5.1 Tier 1 - Retry

```
Trigger: Cubelet returns an error or times out (first occurrence)

Action:
  1. Pod Orch sends retry command to the cubelet
  2. Cubelet re-executes with the same input
  3. If retry succeeds → continue pipeline
  4. If retry fails → escalate to Tier 2

Constraints:
  - Max retries: configurable (default: 2)
  - Retry timeout: same as original time budget
  - Same input data (no modification)

Logging:
  - Retry event logged off-chain
  - No AV penalty for the cubelet (transient errors happen)
```

### 5.2 Tier 2 - Quarantine + Compensate

```
Trigger: Cubelet fails after max retries

Action:
  1. QUARANTINE the cubelet:
     - Stop sending data to it
     - Isolate it from the pipeline
     - Mark as quarantined in pod state
     - Log quarantine event (ON-CHAIN - this is a decision)

  2. COMPENSATE:
     Option A: Redistribute work to other cubelets in the pod
       - If another cubelet has overlapping capability
       - Pod Orch rewires the edge to the alternative cubelet

     Option B: Request replacement from System Orchestrator
       - System Orch finds a suitable cubelet (same capability, available)
       - New cubelet is added to the pod (hot-swap)
       - Pod Orch wires the new cubelet into the DAG
       - Pipeline resumes from the point of failure

  3. AV PENALTY:
     - Quarantined cubelet receives a violation event
     - AV penalty applied to the relevant dimension
     - Severity: moderate (multiplier = 2.0)

Logging:
  - Quarantine event: on-chain
  - Compensation details: off-chain
  - AV penalty: logged by Authority Engine
```

### 5.3 Tier 3 - Adaptive Restructure

```
Trigger: Tier 2 compensation fails (no alternative cubelet, replacement unavailable)

Action:
  1. Pod Orch reasons about the entire DAG:
     - Can the task succeed without this capability?
     - Can the pipeline be restructured to work around the gap?
     - Can remaining cubelets be re-ordered to compensate?

  2. RESTRUCTURE the DAG:
     - Reroute data through different paths
     - Add new cubelets from the pool (via System Orch request)
     - Merge or split pipeline branches
     - Change execution order

  3. This is an EXCEPTION to the "fixed DAG" rule:
     - Pipeline restructure requires authority check
     - Pod Orch must have sufficient execution + resource AV
     - Restructure is logged ON-CHAIN (it's a significant decision)

  4. If restructure succeeds → resume with new DAG
     If restructure fails → escalate to Tier 4

Logging:
  - Restructure decision: on-chain
  - New DAG definition: off-chain
  - All cubelet changes: off-chain
```

### 5.4 Tier 4 - Escalate to System Orchestrator

```
Trigger: Tier 3 restructure fails (task cannot be completed with available resources)

Action:
  1. Pod Orch sends full failure report to System Orchestrator:
     - What went wrong
     - What tiers were attempted
     - Current pod state
     - What resources would be needed to succeed

  2. System Orchestrator decides:
     Option A: Dissolve pod, reassemble with different cubelets
     Option B: Dissolve pod, requeue task for later
     Option C: Dissolve pod, escalate to human (if safety-critical)
     Option D: Provide additional resources and let Pod Orch try again

  3. Pod Orch AV is updated:
     - If failure was due to cubelet issues (not Pod Orch's fault): no penalty
     - If failure was due to poor DAG design or bad coordination: penalty for Pod Orch
     - System Orch LLM makes this judgment

Logging:
  - Escalation event: on-chain
  - Full failure report: off-chain
```

### 5.5 Tier Summary

```
Tier    Trigger              Action                    Scope           AV Impact
────────────────────────────────────────────────────────────────────────────────
  1     First failure        Retry cubelet             Single cubelet  None
  2     Retry exhausted      Quarantine + replace      Single cubelet  Cubelet penalized
  3     Replacement fails    Restructure DAG           Entire pipeline Pod Orch judged
  4     Restructure fails    Escalate to System Orch   Pod-level       Pod Orch may be penalized
```

---

## 6. Pod Orchestrator in Session Pods

When managing an AI interaction session, the Pod Orchestrator has additional responsibilities.

### 6.1 Conversation Context Management

```
The Session Pod's Pod Orch:
  - Maintains conversation history across queries
  - Tracks user intent and preferences within the session
  - Uses context for query routing (is this a follow-up or new topic?)
  - Passes relevant context to sub-pods when they're spawned
```

### 6.2 Sub-Pod Coordination

```
When the Session Pod spawns sub-pods:
  1. Pod Orch formulates the sub-task for the sub-pod
  2. Sends sub-task to System Orch for sub-pod assembly
  3. Waits for sub-pod result(s)
  4. If multiple sub-pods (fan-out):
     - Collects all results
     - Synthesizes a coherent response using LLM reasoning
  5. Passes synthesized result through Safety Monitor cubelet
  6. Delivers filtered response to user
```

### 6.3 Coherence Self-Check

The Pod Orchestrator monitors its own context degradation during long pipelines and multi-query sessions. When coherence drops, it triggers a context reload rather than continuing with degraded reasoning.

```
COHERENCE MONITORING:

  The Pod Orch tracks coherence signals:

  1. Decision consistency:
     - Each new decision is compared against the decision context log
     - If a new decision contradicts a prior decision without an
       explicit reason (new information, changed constraints),
       the coherence score drops

  2. KB alignment:
     - After generating a plan or response, the Pod Orch queries the KB
       for entries relevant to its output
     - If the output conflicts with high-confidence KB entries,
       the coherence score drops

  3. Goal drift:
     - The Pod Orch compares its current action against the active goals
       from the latest context checkpoint
     - If the current action doesn't serve any active goal,
       the coherence score drops

COHERENCE SCORING:
  coherence_score starts at 1.0 (full coherence)
  Each detected inconsistency: coherence_score -= 0.15
  Each action aligned with goals and KB: coherence_score += 0.02 (slow recovery)
  Score is bounded [0.0, 1.0]

COHERENCE THRESHOLDS:
  coherence ≥ 0.7:   Normal operation. No action needed.
  coherence < 0.7:   WARNING - write a context checkpoint immediately.
                      Log a coherence_warning control message.
  coherence < 0.4:   RELOAD - pause pipeline execution.
                      Load latest context checkpoint from Session Memory KB.
                      Reload top-weighted context entries (see 12-knowledge-base.md §3.5-3.6).
                      Rebuild working state. Resume execution.
  coherence < 0.2:   ESCALATE - context is irrecoverably degraded.
                      Escalate to System Orchestrator.
                      System Orch may dissolve and re-assemble the pod
                      with a fresh orchestrator seeded from the KB checkpoint.

SELF-CHECK FREQUENCY:
  - Every N actions (default: 5, configurable via pod_orchestrator_config)
  - After every sub-pod result synthesis
  - After every pipeline restructure
  - Triggered manually by System Orch directive
```

This is analogous to a human realizing they've lost the thread and re-reading their notes before continuing. The orchestrator doesn't try to power through degraded context - it stops, reloads from externalized state, and resumes with a clear picture.

### 6.4 Tiered Release Management

```
The Session Pod's Pod Orch manages idle behavior:
  - Monitors time since last user query
  - Releases specialized cubelets after idle_window expires
  - Keeps core cubelets (Context Manager, Safety Monitor, basic LLM)
  - On session timeout → initiates full dissolution
  - On new query after partial release → requests re-assembly of needed cubelets
```

---

## 7. Configuration

```
pod_orchestrator_config {
    // LLM model assignment (see 10-node-topology for full cost strategy)
    system_orch_model:      "claude-opus-tier"       // paid, powerful model
    domain_orch_model:      "claude-sonnet-tier"     // paid, domain reasoning
    deployment_orch_model:  "llama-8b-ollama"        // open-source, operational
    pod_orch_model:         "llama-8b-ollama"        // open-source, operational

    // AV
    pod_orch_av_floor:      400                      // minimum AV on all dimensions
    av_decay_rate:          0.5                      // per hour of inactivity

    // Templates
    templates_path:         "/config/pod_orch_templates/"

    // Registry
    registry_max_size:      1000                     // max stored Pod Orchs
    registry_min_success_rate: 0.6                   // minimum success rate to keep in registry
    registry_eviction_policy: "lowest_success_rate"  // when full, evict worst performers

    // Failure response
    max_retries_per_cubelet: 2
    retry_timeout_multiplier: 1.0                    // same as original time budget
    quarantine_penalty_severity: "moderate"           // 2.0x base penalty
    restructure_authority_threshold: 600              // min execution AV to restructure

    // Session pods
    idle_window_seconds:    10
    session_timeout_seconds: 300
    max_sub_pods:           5

    // Coherence self-check
    coherence_check_interval: 5             // check every N actions
    coherence_warning_threshold: 0.7
    coherence_reload_threshold: 0.4
    coherence_escalate_threshold: 0.2
    coherence_recovery_rate: 0.02           // per aligned action
    coherence_penalty_rate: 0.15            // per inconsistency
}
```

---

## 8. Invariants (Must Always Hold)

```
INV-1:  Pod Orchestrators are always LLM-based (never rule engines)
INV-2:  Pod Orchestrators are NOT cubelets (distinct entity class)
INV-3:  Pod Orch AV never drops below floor (400) on any dimension
INV-4:  System Orchestrator is always a more powerful LLM than Pod Orchestrators
INV-5:  Tiered failure response follows order: Retry → Quarantine → Restructure → Escalate
INV-6:  DAG restructure (Tier 3) requires authority check and is logged on-chain
INV-7:  Pod Orch has full visibility into its cubelets' AV at all times
INV-8:  Pod Orch never processes data in the pipeline (synthesis of results is not pipeline processing)
INV-9:  Query routing uses capability check first, LLM fallback only when ambiguous
INV-10: Quarantined cubelets always receive an AV violation event
INV-11: Pod Orch registry preserves AV history across reuse
INV-12: Newly created Pod Orchs start at template default AV, not at floor
INV-13: Fan-out synthesis is always LLM-reasoned, never simple concatenation
INV-14: Coherence self-check runs at configured intervals during pipeline execution
INV-15: Coherence below reload threshold triggers mandatory context reload from KB checkpoint
INV-16: Coherence below escalation threshold triggers mandatory escalation to System Orch
INV-17: Every decision that alters the pipeline is logged with its full reasoning chain (decision context)
INV-18: DAG construction must respect CIG structural constraints (DEPENDS_ON, GOVERNS, PROVES edges)
INV-19: DAG construction must follow framework ordering (STA → STI → STD → STF → STK)
INV-20: STSol template is consulted before free-form DAG construction (graph-first, LLM-second)
INV-21: Pod Orch model assignment follows the LLM cost strategy: open-source for Pod/Deployment, paid for System/Domain
```

---

## 9. Interaction with Other Documents

- **Master Document (00-MYOS-master.md):** Defines STSol templates, the Rubik's Lattice, and framework ordering that Pod Orchestrators use for DAG construction. Pod Orchs instantiate STSol templates into running pods.
- **Authority Model (01-authority-model.md):** Pod Orchs are subject to MBAC dual-gate system (AV + STK invariants) with floor = 400. AV is multi-dimensional using the flexible dimension registry. STK invariants are verified at pipeline completion - every pod DAG must terminate at an STK cubelet.
- **Conflict Resolution (02-conflict-resolution.md):** Pod Orch facilitates Level 2 (pod vote) but does not vote itself. It routes domain-specific conflicts to the Domain Orchestrator before escalating to System Orch (Level 3). For cross-framework disagreements (STA vs STD), STA policy takes precedence.
- **Pod Assembly (03-pod-assembly.md):** Pod Orch is assigned during pod assembly. Created from template or cloned from registry. Receives the STSol template and CIG edges for graph-informed DAG construction.
- **Cubelet I/O Contract (04-cubelet-io-contract.md):** Pod Orch builds the DAG using CIG structural constraints and framework ordering. Manages the pipeline, handles control messages, and is responsible for pipeline logging (off-chain) and decision logging (on-chain). Decision context logging creates a parallel reasoning DAG alongside the data DAG.
- **Node Topology (10-node-topology-orchestrator-hierarchy.md):** Defines the LLM cost strategy - Pod Orchs use open-source models (Llama 8B / Phi-3), not paid models. This is the authoritative source for model assignment.
- **Knowledge Base (12-knowledge-base.md):** The Pod Orchestrator manages the Pod KB lifecycle - creating it at pod assembly and destroying it at dissolution. The CIG provides structural backbone; the Pod KB adds runtime knowledge on top. The coherence self-check mechanism uses context checkpoints stored in the KB to reload context when degradation is detected.
