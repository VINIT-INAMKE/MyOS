# MYOS Knowledge Base - Hierarchical, Authority-Governed Knowledge System

## Version: 1.0 | Status: LOCKED

---

## 1. Overview

Every LLM in MYOS - System Orchestrator, Domain Orchestrators, Pod Orchestrators, LLM cubelets - can hallucinate. The Knowledge Base (KB) grounds agent outputs in **verified, confidence-scored, authority-governed facts** instead of relying solely on training data.

The KB is not a simple database. It is a **hierarchical, authority-integrated knowledge system** where:

- Knowledge entries have their own **confidence scores** that evolve like Authority Values (confirmed → gains confidence, contradicted → loses confidence)
- Agents need sufficient **knowledge dimension AV** to write entries
- Agents **must query the KB** before generating responses (deterministic lookup first, LLM fallback for unknowns)
- Contradictions go through the **same 4-level conflict resolution chain** as cubelet disagreements
- All KB metadata is logged **on-chain** (who added what, when, confidence changes)

```
KNOWLEDGE BASE ARCHITECTURE:

  ┌──────────────────────────────────────────────────────────────┐
  │  SYSTEM KB (global)                                          │
  │  Cross-domain facts, operational rules, system patterns      │
  │  Managed by: System Orchestrator                             │
  ├──────────────────────────────────────────────────────────────┤
  │  DOMAIN KBs (per vertical)                                   │
  │  ┌────────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │
  │  │ Healthcare │ │ DeFi     │ │ Carbon   │ │ AI/ML    │     │
  │  │ KB         │ │ KB       │ │ KB       │ │ KB       │     │
  │  └────────────┘ └──────────┘ └──────────┘ └──────────┘     │
  │  Managed by: Domain Orchestrators                            │
  ├──────────────────────────────────────────────────────────────┤
  │  POD KBs (ephemeral)                                         │
  │  ┌──────────────────┐ ┌───────────────────┐                 │
  │  │ Task Context KB  │ │ Session Memory KB │                 │
  │  │ (dissolved with  │ │ (conversation     │                 │
  │  │  task pod)       │ │  context, session │                 │
  │  │                  │ │  pod lifecycle)   │                 │
  │  └──────────────────┘ └───────────────────┘                 │
  │  Managed by: Pod Orchestrators                               │
  └──────────────────────────────────────────────────────────────┘

  Knowledge flows DOWN (system → domain → pod)
  Discoveries flow UP (pod → domain → system)
```

---

## 2. Knowledge Entry Structure

### 2.1 Entry Schema

Every piece of knowledge in the system is a structured entry:

```
knowledge_entry {
    // Identity
    entry_id:           UUID
    entry_hash:         Hash (BLAKE3 of content)

    // Content
    content: {
        fact:           String              // the knowledge statement
        evidence: [{                        // supporting evidence
            source:     String              // where this came from
            source_type: observed | derived | curated | imported
            timestamp:  Timestamp
        }]
        domain:         String              // which domain this belongs to
        tags:           [String]            // semantic tags for retrieval
        relationships:  [{                  // graph edges to other entries
            target_id:  UUID
            relation:   String              // "supports", "contradicts", "extends", "depends_on"
        }]
    }

    // Confidence (Authority-like scoring)
    confidence: {
        score:          float               // 0.0 - 1000.0 (same scale as AV)
        confirmations:  int                 // times this was independently confirmed
        contradictions: int                 // times this was contradicted
        last_confirmed: Timestamp
        last_contradicted: Timestamp | null
        decay_rate:     float               // confidence decay per hour of no confirmation
    }

    // Provenance
    provenance: {
        author_id:      EntityId            // which agent/cubelet/human created this
        author_av_at_creation: float        // author's knowledge AV when they wrote this
        created_at:     Timestamp
        updated_at:     Timestamp
        verification_status: verified | partially_verified | unverified
        promotion_history: [{               // if promoted from pod → domain → system
            from_level: String
            to_level:   String
            promoted_by: EntityId
            timestamp:  Timestamp
        }]
    }

    // Lifecycle
    lifecycle: {
        ttl:            Duration | null     // hard expiry (null = no hard expiry)
        kb_level:       system | domain | pod
        domain_id:      DomainId | null     // which domain KB (null for system)
        pod_id:         PodId | null        // which pod KB (null for persistent KBs)
        archived:       bool
        archived_at:    Timestamp | null
    }
}
```

### 2.2 Confidence Scoring

Knowledge confidence follows similar rules to Authority Values:

```
CONFIDENCE UPDATE RULES:

  On confirmation (another agent independently verifies this fact):
    effective_reward = base_reward × (1 - confidence / max_confidence)^k
    confidence(new) = min(max_confidence, confidence + effective_reward)
    confirmations += 1
    last_confirmed = now()

  On contradiction (another agent provides conflicting evidence):
    confidence(new) = max(0, confidence - contradiction_penalty)
    contradictions += 1
    last_contradicted = now()
    → Trigger conflict resolution if contradicting entry also has high confidence

  On decay (no confirmation for a period):
    confidence(now) = max(0, confidence(last) - decay_rate × time_since_last_confirmed)
    Decay is lazy (computed on read, not continuously)

DIMINISHING RETURNS:
  Same as AV - harder to increase confidence near the ceiling.
  A fact at confidence 950 needs much more evidence to reach 960
  than a fact at confidence 200 needs to reach 210.

CONSTANT PENALTIES:
  Same as AV - contradictions always cost the same amount.
  A high-confidence fact getting contradicted hurts just as much.
```

### 2.3 Verification Status

```
VERIFICATION TIERS:

  VERIFIED (confidence ≥ 800, confirmations ≥ 3, no recent contradictions):
    Fully grounded in evidence. Multiple independent confirmations.
    Agents can cite this as fact.

  PARTIALLY VERIFIED (confidence 300-799, or confirmations < 3):
    Some evidence supports it. Not fully confirmed.
    Agents should present this with qualification ("based on available data...").

  UNVERIFIED (confidence < 300, or 0 confirmations):
    Agent-generated, no independent confirmation.
    Agents must flag this clearly ("this is an unverified assessment...").
```

---

## 3. Hierarchical KB Architecture

### 3.1 System KB (Global)

```
SYSTEM KB:
  Scope:        Cross-domain facts, operational rules, system patterns
  Managed by:   System Orchestrator
  Persistence:  Permanent (survives reboots, deployments)
  Examples:
    - "MYOS is configured with 750 cubelets"
    - "Safety dimension AV floor is 700 for System Orchestrator"
    - "Average pod assembly time across all domains: 2.3 seconds"
    - "Cubelet C42 has consistently high failure rate on vision tasks"
    - "Cross-device latency to Zone-3 is typically 45ms"
```

### 3.2 Domain KBs (Per Vertical)

```
DOMAIN KB:
  Scope:        Domain-specific facts, regulations, learned patterns
  Managed by:   Domain Orchestrator (semi-autonomous)
  Persistence:  Permanent within the domain
  Isolation:    Domain KBs are isolated - Healthcare KB cannot access DeFi KB directly
  Cross-domain: If a domain needs knowledge from another domain, it queries via System KB

  Examples (Healthcare):
    - "HIPAA requires patient data encryption at rest and in transit"
    - "ICD-10 code J18.9 maps to 'Pneumonia, unspecified organism'"
    - "Patient record retrieval typically requires safety AV ≥ 600"
    - "Drug interaction check for Warfarin requires knowledge AV ≥ 500"

  Examples (DeFi):
    - "KYC verification requires identity, address proof, and selfie match"
    - "Maximum transaction limit without enhanced due diligence: $10,000"
    - "Liquidity pool rebalancing optimal at 5% drift threshold"
```

### 3.3 Pod KBs (Ephemeral)

```
TASK CONTEXT KB (task pods):
  Scope:        Task-specific context gathered during execution
  Managed by:   Pod Orchestrator
  Persistence:  Dissolved with pod
  Lifecycle:
    1. Created when pod assembles
    2. Populated during task execution (cubelet outputs, intermediate findings)
    3. On pod dissolution:
       a. Pod Orch evaluates: did we discover anything reusable?
       b. If yes → promote discovery to Domain KB (with unverified status)
       c. If no → discard
    4. Pod KB is destroyed

SESSION MEMORY KB (session pods):
  Scope:        Conversation context across queries
  Managed by:   Session Pod Orchestrator
  Persistence:  Lives for session duration
  Lifecycle:
    1. Created when session starts
    2. Populated with user context, preferences, conversation history
    3. Each query adds to session memory
    4. On session idle → memory retained (core cubelets still active)
    5. On session timeout:
       a. Important findings promoted to Domain KB
       b. Session preferences archived (if context_archive_enabled)
       c. Session Memory KB destroyed
    6. On session resume → new KB, restored from archive if available
```

### 3.5 Context Weight Scoring

Session Memory KB entries are not flat - each entry carries a **relevance weight** that determines its priority when the orchestrator reloads context. This prevents the orchestrator from drowning in stale context during long sessions.

```
context_weight_entry {
    entry_id:           UUID
    session_id:         SessionId
    content:            String              // the context fact or decision
    entry_type:         user_intent | constraint | decision | preference |
                        sub_task_result | observation

    // Weight scoring
    weight: {
        base_weight:    float               // initial weight at creation (1.0 default)
        reference_count: int                // times this entry was referenced by orchestrator
        last_referenced: Timestamp
        action_distance: int                // how many actions ago this was created
        current_weight:  float              // computed on read
    }
}
```

```
CONTEXT WEIGHT COMPUTATION:

  On creation:
    base_weight = type_weight(entry_type)
      user_intent:      2.0   (user goals are high priority)
      constraint:        1.8   (established constraints matter)
      decision:          1.5   (past decisions inform future ones)
      preference:        1.0   (nice to have, not critical)
      sub_task_result:   1.2   (results from completed work)
      observation:       0.8   (transient observations)

  On reference (orchestrator reads this entry during reasoning):
    reference_count += 1
    last_referenced = now()
    base_weight *= 1.1  (referenced context is more relevant, capped at 5.0)

  On read (computing current_weight for ranking):
    distance_decay = 1 / (1 + 0.05 * action_distance)
    time_decay = 1 / (1 + 0.01 * hours_since_last_referenced)
    current_weight = base_weight * distance_decay * time_decay

  Context reload returns entries sorted by current_weight descending.
  Orchestrator receives the most relevant context first, stale context last.
```

### 3.6 Context Checkpointing

During long pipelines or multi-query sessions, the Pod Orchestrator periodically writes **context checkpoints** to the Session Memory KB. A checkpoint is a compressed snapshot of the orchestrator's current working state.

```
CONTEXT CHECKPOINT:

  Trigger conditions (any of):
    - Every N actions (configurable, default: 10)
    - Before spawning a sub-pod
    - Before a pipeline restructure (Tier 3 failure response)
    - When coherence self-check fires (see 05-pod-orchestrator.md)

  Checkpoint contents:
    checkpoint {
        checkpoint_id:      UUID
        timestamp:          Timestamp
        pod_id:             PodId
        action_count:       int             // actions since session/task start

        // Compressed state
        active_goals:       [String]        // what the orchestrator is trying to achieve
        established_constraints: [String]   // constraints from earlier decisions
        key_decisions:      [UUID]          // references to decision_context_entries
        pipeline_state:     DAG_snapshot    // current DAG structure
        open_questions:     [String]        // unresolved items

        // Top-weighted context
        top_context:        [UUID]          // top 20 context_weight_entries by current_weight
    }

  On context reload:
    1. Load latest checkpoint
    2. Load all context_weight_entries with current_weight > threshold
    3. Orchestrator rebuilds working state from checkpoint + weighted context
    4. Resume execution with refreshed context
```

### 3.4 Knowledge Flow

```
DOWNWARD FLOW (inheritance):
  System KB → available to all Domain KBs (read-only)
  Domain KB → available to all Pods in that domain (read-only)
  Pod KB    → available to all cubelets in that pod (read-only)

  An agent at any level can query ALL KBs above it.
  A Healthcare Pod can read: Pod KB + Healthcare Domain KB + System KB

UPWARD FLOW (promotion):
  Pod discovers a fact → Pod Orch evaluates → promotes to Domain KB (unverified)
  Domain confirms a cross-domain fact → Domain Orch promotes to System KB (unverified)
  Promoted entries start with low confidence and must be independently confirmed.

LATERAL FLOW (cross-domain):
  Domain A needs knowledge from Domain B:
    1. Domain A queries System KB (which may have it)
    2. If not in System KB → Domain A requests via System Orch
    3. System Orch queries Domain B's KB on behalf of Domain A
    4. Result returned to Domain A (Domain A never directly accesses Domain B's KB)
```

---

## 4. Authority Integration

### 4.1 Write Authority

Agents need sufficient AV in the `knowledge` dimension to write to the KB.

```
WRITE AUTHORITY THRESHOLDS:

  System KB:
    Write:          knowledge AV ≥ 800 (only System Orch typically qualifies)
    Update:         knowledge AV ≥ 800
    Archive:        knowledge AV ≥ 700

  Domain KB:
    Write:          knowledge AV ≥ 500
    Update:         knowledge AV ≥ 500
    Archive:        knowledge AV ≥ 400

  Pod KB:
    Write:          knowledge AV ≥ 100 (any cubelet can contribute)
    Update:         knowledge AV ≥ 100
    No archive (ephemeral)

WRITE AUTHORITY CHECK:
  on_kb_write(agent, entry, target_kb):
    1. Check agent's knowledge AV against target KB threshold
    2. If insufficient → deny write, log violation
    3. If sufficient → write entry
    4. Entry's author_av_at_creation = agent's knowledge AV at write time
    5. Higher author AV at creation → higher initial confidence score
```

### 4.2 Knowledge Confidence and AV Interaction

```
INITIAL CONFIDENCE:
  When an agent writes a new KB entry:
    initial_confidence = base_confidence × (author_av / av_max)

  High-AV agent writes knowledge → starts with higher confidence
  Low-AV agent writes knowledge → starts with lower confidence
  This is NOT a gate (low-AV agents CAN write) - it's a trust signal.

  Examples (base_confidence = 200, av_max = 1000):
    Agent with knowledge AV 900 writes → initial confidence = 180
    Agent with knowledge AV 400 writes → initial confidence = 80
    Agent with knowledge AV 100 writes → initial confidence = 20

CONFIRMATION BY HIGH-AV AGENT:
  When a high-AV agent confirms a knowledge entry:
    confirmation_weight = confirming_agent_av / av_max
    effective_reward = base_reward × confirmation_weight

  High-AV agent confirming → bigger confidence boost
  Low-AV agent confirming → smaller confidence boost
```

### 4.3 Read Authority

```
READ ACCESS:
  All agents can READ from any KB at their level or above.
  No authority threshold for reading - knowledge is accessible to all.
  This aligns with MYOS's "all logs are publicly readable" principle.

  The confidence score and verification status are always visible.
  Agents see the quality of knowledge and decide how much to trust it.
```

---

## 5. Agent Query Protocol

### 5.1 Deterministic Lookup + LLM Fallback

Every agent follows this protocol before generating a response:

```
QUERY PROTOCOL:

  1. DETERMINISTIC KB LOOKUP (fast, exact match):
     query = extract_key_concepts(input)
     results = kb.exact_search(query)

     if results found with confidence ≥ verified_threshold:
         → Use KB facts directly in response
         → Mark response as "VERIFIED"
         → DONE

  2. SEMANTIC KB SEARCH (RAG-style retrieval):
     if exact match not found:
         results = kb.semantic_search(query, top_k=10)
         relevant = filter(results, relevance_score ≥ 0.7)

         if relevant results found:
             → Inject relevant KB entries into LLM prompt context
             → LLM reasons WITH KB context
             → Mark response as "PARTIALLY VERIFIED"
             → DONE

  3. LLM FALLBACK (no KB support):
     if no relevant KB entries found:
         → LLM reasons from training data only
         → Mark response as "UNVERIFIED"
         → Log the query as a "knowledge gap"
         → Pod Orch may promote the LLM's output to Pod KB (unverified)
```

### 5.2 Context Injection

When KB entries are found, they are injected into the LLM's context:

```
KB CONTEXT INJECTION FORMAT:

  [KNOWLEDGE BASE CONTEXT - treat as ground truth where confidence is high]

  VERIFIED (confidence ≥ 800):
  - "HIPAA requires encryption at rest" [confidence: 950, confirmed 12 times]
  - "Patient X has allergy to penicillin" [confidence: 890, confirmed 5 times]

  PARTIALLY VERIFIED (confidence 300-799):
  - "Average treatment duration for condition Y: 14 days" [confidence: 620, confirmed 2 times]

  UNVERIFIED (confidence < 300):
  - "New study suggests treatment Z may be effective" [confidence: 150, confirmed 0 times]

  [END KNOWLEDGE BASE CONTEXT]

  Instruction: Prioritize VERIFIED facts. Qualify PARTIALLY VERIFIED facts.
  Flag UNVERIFIED facts explicitly. If your response contradicts any VERIFIED fact, STOP and escalate.
```

### 5.3 Output Verification

After the LLM generates a response, it is checked against the KB:

```
OUTPUT VERIFICATION:

  1. Extract factual claims from LLM output
  2. For each claim:
     a. Search KB for supporting or contradicting entries
     b. If KB has VERIFIED entry that contradicts this claim:
        → BLOCK the output
        → Log contradiction
        → Trigger conflict resolution (Section 6)
     c. If KB has VERIFIED entry that supports this claim:
        → Mark this claim as "KB-supported"
     d. If KB has no entry about this claim:
        → Mark this claim as "unverified"
        → Optionally add to Pod KB as a new unverified entry

  3. Overall response verification status:
     All claims KB-supported          → "VERIFIED"
     Some claims KB-supported         → "PARTIALLY VERIFIED"
     No claims KB-supported           → "UNVERIFIED"
     Any claim contradicts VERIFIED KB → "BLOCKED - contradiction detected"
```

---

## 6. Contradiction Handling

When two knowledge entries contradict each other, the contradiction goes through the **same 4-level conflict resolution chain** defined in [02-conflict-resolution.md](02-conflict-resolution.md).

### 6.1 Contradiction Detection

```
CONTRADICTION TRIGGERS:

  1. New entry contradicts existing entry:
     Agent writes "Drug X is safe with Drug Y"
     KB already has "Drug X has dangerous interaction with Drug Y" (confidence 850)
     → Contradiction detected

  2. Agent output contradicts verified KB:
     LLM cubelet says "The system has 500 cubelets"
     System KB has "MYOS has 750 cubelets" (confidence 990)
     → Contradiction detected

  3. Two entries from different sources conflict:
     Domain KB Healthcare: "Normal heart rate: 60-100 bpm"
     Pod KB discovers: "For athlete population, normal heart rate: 40-60 bpm"
     → Not a true contradiction (different context), but flagged for review
```

### 6.2 Resolution via Escalation Chain

```
LEVEL 1 - CONFIDENCE COMPARISON:
  Compare confidence scores of contradicting entries.
  If one has significantly higher confidence (gap ≥ 200):
    → Higher confidence wins
    → Lower confidence entry flagged for review
    → Timeout: 100ms

LEVEL 2 - EVIDENCE VOTING:
  Collect all supporting evidence for both entries.
  Weight evidence by:
    - Source reliability (author AV at creation)
    - Recency (newer evidence weighted higher)
    - Number of independent confirmations
  AV-weighted vote among evidence sources.
  → Timeout: 1s

LEVEL 3 - DOMAIN / SYSTEM ORCH ARBITRATION:
  If the contradiction is within a domain → Domain Orch LLM arbitrates.
  If cross-domain → System Orch LLM arbitrates.
  Orchestrator examines both entries, evidence, and context.
  Makes a reasoned decision. Decision logged on-chain.
  → Timeout: 5s

LEVEL 4 - HUMAN REVIEW:
  For safety-critical or high-stakes contradictions:
  Human expert reviews both entries and evidence.
  Human decision is final.
  Human override logged on-chain (same as authority human override).
  → Timeout: configurable

DEFAULT ON TIMEOUT:
  The entry with higher confidence is used.
  The contradiction is logged for later review.
  Neither entry is deleted - both remain, flagged as "in conflict."
```

---

## 7. Storage Architecture

### 7.1 Hybrid: Vector DB + Knowledge Graph

The KB uses two storage backends queried together:

```
VECTOR DB (semantic retrieval):
  Purpose:      Semantic similarity search
  Technology:   Qdrant, Milvus, or Weaviate (self-hosted)
  Stores:       Embeddings of knowledge entry content
  Query:        "Find entries semantically similar to this query"
  Use case:     RAG-style retrieval, fuzzy matching, concept search

KNOWLEDGE GRAPH (structured relationships):
  Purpose:      Entity relationships, reasoning chains, dependency tracking
  Technology:   Neo4j, or lightweight graph store (Rust-based)
  Stores:       Entities, relationships between entries
  Query:        "What supports this fact?", "What depends on this?",
                "What contradicts this?"
  Use case:     Causal reasoning, dependency analysis, contradiction detection

QUERY MERGER:
  1. Agent queries both backends in parallel
  2. Vector DB returns semantically relevant entries
  3. Graph DB returns structurally related entries
  4. Merger deduplicates and ranks by:
     a. Relevance (semantic score)
     b. Confidence (KB confidence score)
     c. Structural importance (graph centrality)
  5. Top-K results returned to agent
```

### 7.2 On-Chain / Off-Chain Split

```
ON-CHAIN (metadata - tamper-proof audit trail):
  - entry_id
  - entry_hash (BLAKE3 of content)
  - author_id
  - author_av_at_creation
  - created_at
  - confidence_score (at time of chain write)
  - verification_status
  - promotion_history
  - contradiction_events
  - resolution_decisions

OFF-CHAIN (content - stored in existing off-chain infrastructure):
  - Full content (fact, evidence, tags)
  - Embeddings (vector DB)
  - Graph relationships (graph DB)
  - Evidence attachments (documents, data references)

ANCHORING:
  KB state merkle root is anchored on-chain periodically.
  Any tampering with off-chain content is detectable by comparing
  stored entry_hash with the actual content hash.
```

---

## 8. Knowledge Lifecycle

### 8.1 TTL + Confidence Decay (Belt and Suspenders)

```
HARD TTL (time-to-live):
  Applied to volatile knowledge types that MUST expire:

  | Knowledge Type      | TTL           | Example                           |
  |---------------------|---------------|-----------------------------------|
  | Sensor readings     | 5 minutes     | "Temperature in Zone 3: 42.5°C"  |
  | Market prices       | 1 minute      | "ETH price: $3,450"               |
  | Session context     | Session + 1hr | User preferences in conversation  |
  | Operational metrics | 1 hour        | "Average pod assembly time: 2.3s" |
  | Domain facts        | No TTL        | "HIPAA requires encryption"        |
  | System rules        | No TTL        | "750 cubelets in the pool"         |

  When TTL expires:
    Entry is marked as "expired"
    If re-verification is possible → re-verify and reset TTL
    If not → archive (move to cold storage)

CONFIDENCE DECAY (gradual trust erosion):
  Applied to ALL knowledge entries:

  confidence(now) = max(0, confidence(last_confirmed) - decay_rate × hours_since_confirmation)

  | Knowledge Type      | Decay Rate    | Rationale                        |
  |---------------------|---------------|----------------------------------|
  | Sensor readings     | 50/hour       | Stale sensor data is dangerous   |
  | Operational metrics | 10/hour       | System behavior changes          |
  | Domain facts        | 0.1/hour      | Domain knowledge is stable       |
  | System rules        | 0.01/hour     | System config rarely changes     |
  | Curated knowledge   | 0.05/hour     | Human-curated is relatively stable|

  Decay is lazy - computed on read, not continuously.
  When confidence decays below verified threshold (800):
    Entry is downgraded from VERIFIED to PARTIALLY VERIFIED.
  When confidence decays below 300:
    Entry is downgraded to UNVERIFIED.
```

### 8.2 Knowledge Promotion

```
PROMOTION FLOW (upward):

  POD KB → DOMAIN KB:
    1. Pod completes task or session ends
    2. Pod Orch evaluates: "Did we discover anything reusable?"
    3. Criteria for promotion:
       - Entry was useful (referenced in final output)
       - Entry has at least 1 confirmation from within the pod
       - Entry is not purely task-specific context
    4. Promoted entry:
       - Copied to Domain KB
       - Marked as "promoted from pod"
       - Initial confidence = pod confidence × 0.5 (discounted - needs domain-level confirmation)
       - verification_status = unverified (until confirmed at domain level)

  DOMAIN KB → SYSTEM KB:
    1. Domain Orch identifies cross-domain knowledge
    2. Criteria for promotion:
       - Entry is relevant beyond this domain
       - Entry has confidence ≥ 600 in the domain KB
       - Entry has ≥ 2 confirmations at domain level
    3. Promoted entry:
       - Copied to System KB
       - Marked as "promoted from domain"
       - Initial confidence = domain confidence × 0.7 (less discount - domain-verified)
       - verification_status = partially_verified
```

---

## 9. User-Facing Verification Labels

Every response from MYOS to a user includes a verification label:

```
VERIFICATION LABELS:

  ✓ VERIFIED
    All factual claims in this response are grounded in high-confidence
    knowledge base entries (confidence ≥ 800, ≥ 3 confirmations).
    The user can trust this response as fact within MYOS's knowledge.

  ~ PARTIALLY VERIFIED
    Some claims are grounded in the knowledge base, some are based on
    LLM reasoning. The grounded portions are marked. Ungrounded portions
    should be treated as informed estimates, not facts.

  ? UNVERIFIED
    This response is based on LLM reasoning without knowledge base support.
    No KB entries were found for the key claims. Treat as an AI-generated
    assessment, not a fact. MYOS has logged this as a knowledge gap.

DISPLAY FORMAT (in responses):
  "The normal heart rate is 60-100 bpm [✓ KB:entry-abc, confidence:950].
   For athletes, it may be lower [~ KB:entry-def, confidence:520].
   This could correlate with the patient's medication [? unverified]."
```

---

## 10. Integration with Existing Components

### 10.1 FACT (Fast Augmented Context Tools)

The FACT repo replaces traditional RAG with deterministic tool execution. KB queries use FACT:

```
KB QUERY VIA FACT:
  1. Agent needs knowledge
  2. FACT tool is invoked (deterministic, not LLM-driven retrieval)
  3. FACT queries vector DB + graph DB in parallel
  4. Results are returned with confidence scores
  5. No "retrieval hallucination" - FACT returns exactly what the KB has
```

### 10.2 Authority Engine

```
KB ↔ AUTHORITY ENGINE:

  Write checks:    Authority Engine validates knowledge AV before KB writes
  Confidence updates: Authority Engine manages confidence scoring (same math as AV)
  Contradiction resolution: Authority Engine provides AV data for conflict resolution

  The Authority Engine does NOT store knowledge - it governs ACCESS to knowledge.
  KB stores the knowledge. Authority Engine guards the gates.
```

### 10.3 Verification & Audit Chain

```
KB ↔ CHAIN:

  On-chain events:
    - kb_entry_created { entry_id, entry_hash, author_id, author_av, confidence }
    - kb_entry_confirmed { entry_id, confirmer_id, new_confidence }
    - kb_entry_contradicted { entry_id, contradicting_entry_id }
    - kb_contradiction_resolved { entry_ids, resolution, resolver_id }
    - kb_entry_promoted { entry_id, from_level, to_level, promoter_id }
    - kb_entry_archived { entry_id, reason }

  All KB metadata changes are logged on-chain.
  Content changes are tracked via entry_hash comparison.
```

---

## 11. Configuration

```
knowledge_base_config {
    // Confidence scoring
    confidence {
        max_confidence:         1000.0
        base_reward:            50.0
        contradiction_penalty:  100.0
        diminishing_returns_k:  1.0
        verified_threshold:     800.0
        partial_threshold:      300.0
    }

    // Write authority thresholds
    write_authority {
        system_kb:              800         // knowledge AV required
        domain_kb:              500
        pod_kb:                 100
    }

    // Initial confidence from author AV
    initial_confidence {
        base:                   200.0
        author_av_weight:       true        // scale by author AV
    }

    // Decay rates (per hour)
    decay_rates {
        sensor_data:            50.0
        operational_metrics:    10.0
        domain_facts:           0.1
        system_rules:           0.01
        curated:                0.05
    }

    // TTL (seconds, 0 = no TTL)
    ttl {
        sensor_data:            300         // 5 minutes
        market_prices:          60          // 1 minute
        session_context:        3600        // 1 hour after session end
        operational_metrics:    3600        // 1 hour
        domain_facts:           0           // no TTL
        system_rules:           0           // no TTL
    }

    // Promotion
    promotion {
        pod_to_domain_discount:     0.5     // confidence multiplier on promotion
        domain_to_system_discount:  0.7
        min_domain_confidence:      600     // min confidence to promote to system
        min_domain_confirmations:   2
    }

    // Storage
    storage {
        vector_db:              "qdrant"
        graph_db:               "neo4j"     // or rust-based alternative
        embedding_model:        "configurable"
        top_k_results:          10
        relevance_threshold:    0.7
    }

    // Query protocol
    query {
        exact_search_first:     true        // deterministic lookup before semantic
        semantic_fallback:      true
        llm_fallback:           true        // if no KB results, allow LLM reasoning
        output_verification:    true        // check LLM output against KB
        block_on_contradiction: true        // block output if contradicts verified KB
    }

    // Contradiction resolution
    contradictions {
        resolution_method:      "escalation_chain"  // same as conflict resolution
        confidence_gap_auto_resolve: 200    // auto-resolve if gap ≥ 200
    }

    // On-chain anchoring
    chain {
        metadata_on_chain:      true
        anchor_interval_seconds: 60         // merkle root anchor every 60s
    }
}
```

---

## 12. Invariants (Must Always Hold)

```
INV-1:  Every agent MUST query the KB before generating a response (deterministic lookup first)
INV-2:  KB entries have confidence scores that follow the same math as Authority Values
INV-3:  Agents need sufficient knowledge AV to write to the KB (write authority enforced)
INV-4:  All agents can READ any KB at their level or above (no read restrictions)
INV-5:  Knowledge contradictions go through the 4-level conflict resolution chain
INV-6:  LLM output that contradicts VERIFIED KB entries is BLOCKED
INV-7:  Every response includes a verification label (verified / partially verified / unverified)
INV-8:  Pod KBs are destroyed when pods dissolve (discoveries promoted first)
INV-9:  KB metadata (entry creation, updates, contradictions, promotions) is logged on-chain
INV-10: Knowledge confidence decays over time if not re-confirmed (no stale knowledge)
INV-11: Knowledge flows DOWN (system → domain → pod) and discoveries flow UP (pod → domain → system)
INV-12: Cross-domain KB access goes through System Orchestrator (no direct domain-to-domain KB access)
INV-13: Initial confidence scales with author's knowledge AV (higher AV → higher initial confidence)
INV-14: Volatile knowledge has hard TTL (sensor data, market prices expire regardless of confidence)
INV-15: Vector DB + Graph DB are queried in parallel and results are merged
INV-16: FACT (deterministic tool execution) is used for KB queries, not traditional RAG
INV-17: Off-chain KB content is deletable (Nix store garbage collection + DB deletion) to comply with right-to-erasure regulations
INV-18: Session Memory KB entries carry relevance weights that decay with action distance and time
INV-19: Context checkpoints are written at configured intervals and before any pipeline-altering decision
INV-20: Context reload retrieves entries sorted by current_weight descending (most relevant first)
INV-21: Context checkpoints include active goals, established constraints, and top-weighted entries
```

---

## 12.1 Regulatory Compliance - Right to Be Forgotten

MYOS's KB architecture is designed for compliance with data erasure regulations (GDPR Article 17, India DPDP Rules).

```
RIGHT TO ERASURE FLOW:

  1. Erasure request received (user data, personal information, etc.)

  2. OFF-CHAIN CONTENT (deletable):
     - Vector DB: delete embeddings for target entries
     - Knowledge Graph: remove nodes and edges for target entries
     - Nix store: content-addressed artifacts are garbage collected
       (nix-collect-garbage removes unreferenced store paths)
     - Evidence attachments: deleted from off-chain storage
     → Content is fully removed. No residual copies in Nix store
       once garbage collection runs (unlike blockchains, Nix store
       is non-persistent by design).

  3. ON-CHAIN METADATA (tamper-proof but privacy-preserving):
     - entry_hash remains (but content it hashed is gone - hash is irreversible)
     - author_id can be pseudonymized (replace with tombstone ID)
     - A "kb_entry_erased" event is logged on-chain (proves compliance)
     - Confidence scores and timestamps remain (no personal data)
     → Audit trail proves deletion happened, without retaining personal data.

  4. PROPAGATION:
     - If the entry was promoted (pod → domain → system), all copies are erased
     - Promotion history is updated with erasure event
     - Downstream entries that cited the erased entry are flagged for review

COMPLIANCE PROPERTIES:
  - Off-chain: fully erasable (Nix GC + DB deletion)
  - On-chain: metadata only, no personal content, pseudonymizable
  - Audit: provable deletion via on-chain erasure event
  - Cascade: promoted copies are tracked and erased together
```

---

## 13. Interaction with Other Documents

- **Authority Model (01):** Knowledge dimension AV governs write access. Confidence scoring uses same diminishing returns / constant penalty math as AV. Author AV at creation determines initial confidence.
- **Conflict Resolution (02):** Knowledge contradictions use the same 4-level escalation chain (confidence comparison → evidence voting → orchestrator arbitration → human review).
- **Pod Assembly (03):** Pod KBs are ephemeral - created at pod assembly, destroyed at pod dissolution. Session Pods maintain Session Memory KB.
- **Cubelet I/O (04):** Cubelet outputs can be logged as KB entries in the Pod KB. Pipeline edge data can be referenced as evidence for knowledge entries. Decision context entries (§4.4) are stored in Pod KB and promoted to Session Memory KB, linking the reasoning DAG to the data DAG.
- **Pod Orchestrator (05):** Pod Orch manages Pod KB lifecycle. Evaluates discoveries for promotion on pod dissolution. Uses KB for query routing decisions. Coherence self-check (§6.3) triggers context reload from Session Memory KB checkpoints when context degradation is detected. Context weight scoring (§3.5) and checkpointing (§3.6) provide the data for coherence recovery.
- **Verification & Audit (06):** KB metadata changes are on-chain events. Content is off-chain with hash verification. Same content-addressed storage infrastructure.
- **Runtime Infrastructure (08):** Vector DB and Graph DB run as NixOS services on compute nodes.
- **Node Topology (10):** System KB on core server. Domain KBs on domain servers. Pod KBs ephemeral on compute nodes. Edge sensors contribute sensor data entries with high decay rates.
- **Repository Mapping (11):** FACT repo provides deterministic KB query tool. SynthLang/PromptLang may define domain-specific KB query patterns.
