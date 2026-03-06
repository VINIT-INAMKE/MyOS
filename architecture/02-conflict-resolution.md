# MYOS Conflict Resolution - Escalation Chain & AV-Weighted Voting

## Version: 1.0 | Status: LOCKED

---

## 1. Overview

MYOS uses a **4-level escalation chain** to resolve conflicts between entities (cubelets, Pod Orchestrators, System Orchestrator). Each level introduces a broader scope of evaluation and higher authority. The system is designed so that the vast majority of conflicts resolve at Level 1 (direct AV comparison), with each subsequent level handling progressively rarer and more complex disagreements.

```
Expected resolution distribution:
  Level 1 (AV comparison):     ~70% of conflicts
  Level 2 (AV-weighted vote):  ~20% of conflicts
  Level 3 (System Orch):       ~8% of conflicts
  Level 4 (Human-in-the-loop): ~2% of conflicts
```

Every level has a **timeout**. If a level cannot resolve within its deadline, the conflict automatically escalates to the next level. If Level 4 (human) times out, the system defaults to the **safest available option**.

---

## 2. What Constitutes a Conflict?

A conflict occurs when two or more entities disagree on an action, decision, or resource allocation within the system.

### 2.1 Conflict Types

| Conflict Type                       | Description                                                                       | Example                                                                                                                             |
| ----------------------------------- | --------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| **Action disagreement**             | Two cubelets propose different actions for the same situation                     | Physical: Cubelet A says "turn left", Cubelet B says "turn right". AI: Cubelet A recommends answer X, Cubelet B recommends answer Y |
| **Resource contention**             | Multiple entities request the same limited resource                               | Physical: Two pods both need access to the same sensor. AI: Two sub-pods both need the same LLM cubelet                             |
| **Authority dispute**               | An entity challenges whether another entity has sufficient authority              | Cubelet questions Pod Orchestrator's decision on reasoning depth or actuator command                                                |
| **Escalation request**              | A lower-authority entity disagrees with a higher-authority entity's decision      | Cubelet disagrees with Pod Orch and escalates                                                                                       |
| **Cross-pod coordination conflict** | Two pods' actions would interfere with each other                                 | Physical: Pod A's motor command conflicts with Pod B's safety constraint. AI: Two sub-pods return contradictory analysis results    |
| **Reasoning disagreement**          | Cubelets disagree on the reasoning chain or conclusion for an AI interaction task | LLM cubelet and knowledge cubelet disagree on the factual basis of a response                                                       |

### 2.2 Conflict Detection

Conflicts are detected by:

1. **Pod Orchestrator** - detects intra-pod conflicts (cubelets within the same pod disagree)
2. **System Orchestrator** - detects inter-pod conflicts (pods' actions interfere with each other)
3. **Authority Engine** - detects authority violations (entity attempts action beyond its AV)
4. **Self-report** - an entity explicitly raises a disagreement via the escalation API

### 2.3 Conflict Record

Every detected conflict generates a conflict record:

```
conflict_record {
    conflict_id:        UUID
    conflict_type:      ConflictType
    timestamp:          Timestamp
    participants:       [EntityId]
    proposals:          [Proposal]          // what each participant wants to do
    relevant_dimension: DimensionName       // which AV dimension is relevant
    context:            ConflictContext      // task, environment, constraints
    resolution_level:   Level (1|2|3|4)     // which level resolved it
    resolution:         Resolution          // final decision
    resolution_time:    Duration            // how long resolution took
}
```

This record is immutable and sent to the Verification & Audit system.

---

## 3. Level 1 - Direct AV Comparison

### 3.1 Mechanism

The simplest and fastest resolution. Two entities disagree; the one with the higher Authority Value on the **relevant dimension** wins.

```
Input:
  entity_a:  AV[relevant_dimension] = X
  entity_b:  AV[relevant_dimension] = Y
  relevant_dimension: determined by the task/action type

Decision:
  if X > Y:
      winner = entity_a
  elif Y > X:
      winner = entity_b
  elif X == Y:
      → escalate to Level 2 (tie cannot be resolved here)

Outcome:
  winner's proposal is accepted
  loser's proposal is rejected
  loser is notified of the decision
```

### 3.2 Loser's Options

After losing at Level 1, the losing entity can:

1. **Accept** - acknowledge the decision, no escalation. This is the expected behavior for most conflicts.
2. **Escalate** - request Level 2 resolution. This triggers a pod-wide vote.

The decision to escalate is the losing entity's right. It is not penalized for escalating (no AV loss for requesting escalation). However, **frivolous escalation** - escalating and then losing at Level 2 with low vote support - may result in a minor AV penalty (configurable).

### 3.3 Time Budget

```
level_1_timeout: 100ms (configurable)
```

If Level 1 cannot resolve within the timeout (e.g., AV query fails, Authority Engine unresponsive), the conflict auto-escalates to Level 2.

### 3.4 Logging

```
level_1_log {
    conflict_id:     UUID
    winner:          EntityId
    loser:           EntityId
    winner_av:       float
    loser_av:        float
    dimension:       DimensionName
    margin:          float (winner_av - loser_av)
    loser_accepted:  bool
    escalated:       bool
    resolution_time: Duration
}
```

---

## 4. Level 2 - AV-Weighted Pod Vote

### 4.1 Trigger

Level 2 is triggered when:

- The Level 1 loser escalates
- Level 1 results in a tie (equal AV)
- Level 1 times out

### 4.2 Mechanism

The Pod Orchestrator (an LLM agent, NOT a cubelet - see 05-pod-orchestrator.md) initiates a **vote among all cubelets in the pod**. The Pod Orchestrator is the **facilitator** of the vote, not a voter. Each cubelet's vote is **weighted by its AV on the relevant dimension**.

```
Input:
  proposal_a: (proposed by entity_a)
  proposal_b: (proposed by entity_b)
  voters: all cubelets in the pod (including a and b)
  relevant_dimension: from the conflict context

Voting:
  Each voter casts a vote for one proposal.
  Each vote carries weight = voter.AV[relevant_dimension]

Tallying:
  score_a = sum of AV[relevant_dimension] for all voters who chose proposal_a
  score_b = sum of AV[relevant_dimension] for all voters who chose proposal_b
  total   = score_a + score_b

Decision:
  if score_a > score_b:
      winner = proposal_a
      confidence = score_a / total
  elif score_b > score_a:
      winner = proposal_b
      confidence = score_b / total
  else:
      → escalate to Level 3 (exact tie)
```

### 4.3 Quorum Requirement

A minimum number of voters must participate for the vote to be valid.

```
quorum = max(3, ceil(pod_size × 0.5))

If fewer than quorum cubelets are available to vote:
    → auto-escalate to Level 3
```

**Rationale:** A vote between only 2 entities is just Level 1 again. Quorum ensures a meaningful collective decision.

### 4.4 Confidence Threshold

The vote result must meet a minimum confidence level to be accepted.

```
confidence_threshold: 0.6 (configurable, default 60%)

if confidence ≥ confidence_threshold:
    → decision accepted at Level 2
elif confidence < confidence_threshold:
    → low-confidence result → escalate to Level 3
```

**Example:**

```
Cubelet A: "go left"   (AV=800, weight=800)
Cubelet B: "go right"  (AV=450, weight=450)
Cubelet C: "go right"  (AV=600, weight=600)
Cubelet D: "go left"   (AV=300, weight=300)

"go left"  = 800 + 300 = 1100
"go right" = 450 + 600 = 1050

total = 2150
confidence("go left") = 1100 / 2150 = 0.511 (51.1%)

0.511 < 0.6 (threshold) → LOW CONFIDENCE → escalate to Level 3
```

### 4.5 Vote Eligibility

Not all cubelets in a pod may be eligible to vote:

```
Eligible to vote:
  - Active cubelets in the pod (not in error state)
  - Cubelets with AV[relevant_dimension] > 0
  - Cubelets not involved in an unrelated ongoing task that prevents interruption

NOT eligible:
  - The Pod Orchestrator itself (it is the facilitator, not a voter at Level 2)
  - Cubelets in error/recovery state
  - Cubelets with AV[relevant_dimension] = 0 (no authority in this domain)
```

### 4.6 Time Budget

```
level_2_timeout: 500ms (configurable)

Breakdown:
  vote_collection_window: 400ms (time for cubelets to cast votes)
  tallying_and_decision:  100ms
```

If the vote does not complete within the timeout, or if quorum is not met within the collection window, the conflict auto-escalates to Level 3.

### 4.7 Logging

```
level_2_log {
    conflict_id:        UUID
    proposal_a:         Proposal
    proposal_b:         Proposal
    votes: [{
        voter:          EntityId
        voted_for:      proposal_a | proposal_b
        voter_av:       float
        weight:         float
    }]
    score_a:            float
    score_b:            float
    confidence:         float
    quorum_met:         bool
    voters_count:       int
    eligible_count:     int
    decision:           proposal_a | proposal_b | escalated
    resolution_time:    Duration
}
```

---

## 5. Level 3 - System Orchestrator Arbitration

### 5.1 Trigger

Level 3 is triggered when:

- Level 2 vote has low confidence (below threshold)
- Level 2 results in an exact tie
- Level 2 fails quorum
- Level 2 times out
- The conflict is cross-pod (two different pods disagree - Level 2 cannot handle this because voting is intra-pod)

### 5.2 Mechanism

The System Orchestrator evaluates the conflict with **full system context**.

```
Input:
  conflict_record (from Level 1 and/or Level 2)
  proposal_a, proposal_b
  AV of all participants
  vote results (if Level 2 was attempted)
  system-wide state:
      - other active pods and their tasks
      - resource availability
      - safety status
      - environmental conditions

Evaluation:
  System Orchestrator has high AV floor (700) and typically high earned AV.
  It uses:
    1. Authority comparison (its own AV vs. participants')
    2. System-wide context (does one proposal conflict with other pods?)
    3. Safety analysis (is one proposal safer?)
    4. Policy rules (does one proposal violate system policy?)

Decision:
  if System Orch is confident:
      → make decision, log reasoning
  elif safety-critical:
      → escalate to Level 4 (human)
  elif not confident:
      → escalate to Level 4 (human)
```

### 5.3 System Orchestrator Decision Criteria

The System Orchestrator evaluates proposals using a weighted scoring model:

```
proposal_score = (
    w1 × av_alignment          // does the proposal align with high-AV entities?
  + w2 × safety_score          // how safe is this proposal?
  + w3 × resource_efficiency   // how resource-efficient?
  + w4 × policy_compliance     // does it comply with system policy?
  + w5 × vote_support          // how much Level 2 vote support did it get?
)

Where w1..w5 are system-configured weights.
```

If the score difference between proposals exceeds a **confidence margin**, the System Orchestrator decides. Otherwise, it escalates.

```
confidence_margin: 0.15 (configurable)

if abs(score_a - score_b) / max(score_a, score_b) ≥ confidence_margin:
    → decide (pick higher score)
else:
    → escalate to Level 4
```

### 5.4 Cross-Pod Conflicts

For conflicts between pods (not within a pod), Level 3 is the **first** resolution level (Levels 1 and 2 are intra-pod only).

```
Cross-pod conflict flow:
  Pod Orch A detects conflict with Pod B
  → Pod Orch A sends conflict report to System Orchestrator
  → System Orchestrator evaluates (Level 3)
  → If unresolvable → Level 4 (human)
```

### 5.5 Time Budget

```
level_3_timeout: 2000ms (configurable)

Breakdown:
  context_gathering:    500ms
  evaluation:           1000ms
  decision_and_logging: 500ms
```

Timeout → auto-escalate to Level 4.

### 5.6 Logging

```
level_3_log {
    conflict_id:          UUID
    system_orch_id:       EntityId
    system_orch_av:       AuthorityVector
    proposals_evaluated:  [Proposal with scores]
    context_snapshot:     SystemStateSnapshot
    decision_criteria: {
        av_alignment:       float
        safety_score:       float
        resource_efficiency: float
        policy_compliance:  float
        vote_support:       float
    }
    score_a:              float
    score_b:              float
    confidence_margin:    float
    decision:             proposal_a | proposal_b | escalated
    reasoning:            String
    resolution_time:      Duration
}
```

---

## 6. Level 4 - Human-in-the-Loop

### 6.1 Trigger

Level 4 is triggered when:

- Level 3 System Orchestrator cannot reach confidence margin
- The conflict involves a safety-critical decision
- Level 3 times out
- Any level explicitly flags the conflict as "requires human judgment"

### 6.2 Human Notification

When a conflict reaches Level 4, the human operator receives a **conflict briefing**:

```
conflict_briefing {
    conflict_id:          UUID
    urgency:              high | medium | low
    conflict_type:        ConflictType
    conflict_summary:     String (human-readable)

    proposal_a: {
        description:      String
        proposed_by:      EntityId
        proposer_av:      float (relevant dimension)
        safety_assessment: String
    }

    proposal_b: {
        description:      String
        proposed_by:      EntityId
        proposer_av:      float (relevant dimension)
        safety_assessment: String
    }

    level_2_vote_results: {       // if applicable
        score_a:          float
        score_b:          float
        confidence:       float
        voter_count:      int
    }

    level_3_evaluation: {         // if applicable
        system_orch_recommendation: proposal_a | proposal_b
        system_orch_confidence:     float
        reasoning:                  String
    }

    deadline:             Timestamp (when timeout fallback triggers)
}
```

### 6.3 Human Actions

The human operator can:

#### 6.3.1 Vote (MAX AV Weight)

The human participates as a voter with the **maximum possible AV weight** (AV_max in the relevant dimension).

```
human_vote_weight = AV_max (e.g., 1000)

The vote is added to the Level 2 vote tally (retroactively recalculated):
  new_score_a = level_2_score_a + (1000 if human votes for a, else 0)
  new_score_b = level_2_score_b + (1000 if human votes for b, else 0)
  new_confidence = max(new_score_a, new_score_b) / (new_score_a + new_score_b)
```

This means the human is the strongest single voter but can still be outweighed if the entire system collectively disagrees.

#### 6.3.2 Veto (Safety-Critical ONLY)

On **safety-critical decisions only**, the human has **hard veto power**. A veto overrides all voting math and directly imposes the human's chosen action.

```
Veto conditions:
  - The conflict must be tagged as safety-critical
  - Safety-critical tag is determined by:
      a) The relevant dimension is "safety"
      b) The action_authority_map marks the action as safety-critical
      c) Any participant flagged a safety concern during escalation
  - If the conflict is NOT safety-critical, the human can only vote (no veto)

Veto effect:
  - Human's chosen proposal is immediately executed
  - All other proposals are discarded
  - The veto is logged as a "human_override" event (distinct from normal decisions)
  - The veto does NOT affect any entity's AV (human is outside the reinforcement model)
```

#### 6.3.3 Delegate Back

The human can delegate the decision back to the System Orchestrator with instructions:

```
delegate {
    instruction:  "Prioritize safety over speed in this case"
    constraints:  [additional constraints the human wants applied]
}

→ System Orchestrator re-evaluates with the human's additional constraints
→ If still unresolvable → returns to human with updated analysis
```

### 6.4 Time Budget and Timeout Fallback

```
level_4_timeout: configurable per deployment

Examples:
  real-time robotics:     5000ms (5 seconds)
  industrial monitoring:  60000ms (1 minute)
  non-urgent analysis:    300000ms (5 minutes)
```

**If the human does not respond within the timeout:**

```
timeout_fallback:
  1. Identify the SAFEST option among the proposals
     - "Safest" is determined by the safety_score from Level 3 evaluation
     - If no safety scores available, default to NO ACTION (inaction)
  2. Execute the safest option
  3. Log as "timeout_fallback" event
  4. Notify human that a fallback decision was made
  5. Human can still review and override after the fact (post-hoc correction)
```

**Definition of "safest option":**

```
Safest option priority:
  1. The option with the highest safety_score from Level 3
  2. If equal safety scores → the option proposed by the entity with higher safety AV
  3. If still equal → NO ACTION (do nothing)
  4. If inaction is explicitly marked as unsafe → execute the option with the
     lowest resource impact (minimum change to current state)
```

### 6.5 Logging

```
level_4_log {
    conflict_id:            UUID
    human_operator_id:      String
    briefing_sent_at:       Timestamp
    human_response_at:      Timestamp | null (if timeout)
    human_action:           vote | veto | delegate | timeout
    human_decision:         proposal_a | proposal_b | null
    is_safety_critical:     bool
    veto_used:              bool
    timeout_fallback_used:  bool
    fallback_option:        Proposal | NO_ACTION
    reasoning:              String (human-provided, optional)
    resolution_time:        Duration
}
```

Human override logs are stored **separately** from the authority reinforcement log. They are part of the audit trail but do not feed into AV calculations.

---

## 7. Timeout Summary - All Levels

```
Level    Timeout (default)    On Timeout
──────────────────────────────────────────────────────
L1       100ms                → escalate to L2
L2       500ms                → escalate to L3
L3       2000ms               → escalate to L4
L4       deployment-specific  → safest option fallback
```

**Total worst-case resolution time (all levels exhaust their timeout):**

```
worst_case = 100 + 500 + 2000 + L4_timeout
           = 2600ms + L4_timeout

For real-time robotics (L4 = 5000ms):  7600ms (7.6 seconds)
For industrial monitoring (L4 = 60s):  62.6 seconds
```

Timeouts are configurable per deployment to match the safety profile of the environment.

---

## 8. Special Cases

### 8.1 Safety Emergency Override

In a **safety emergency** (detected by the safety watchdog, not by normal conflict resolution), the escalation chain is bypassed entirely:

```
Safety emergency detected:
  → Immediate execution of the safest action
  → No voting, no escalation, no human wait
  → Safety watchdog has P0 kernel priority
  → Log the emergency action
  → Notify all levels after the fact
```

Safety emergencies are not conflicts. They are system-level interrupts that override all other processing.

### 8.2 Single-Entity Pod

If a pod has only one cubelet, Level 2 (pod vote) is meaningless. Conflicts in single-cubelet pods escalate directly from Level 1 to Level 3.

```
if pod_size == 1:
    Level 1 → Level 3 (skip Level 2)
```

### 8.3 Cascading Conflicts

If resolving one conflict triggers another conflict (e.g., accepting proposal A causes a resource contention with another pod):

```
Cascading conflict:
  → New conflict is created with a reference to the original
  → New conflict enters escalation chain at Level 1
  → If cascading depth exceeds max_cascade_depth (default: 3):
      → Escalate entire chain to Level 4 (human)
      → Flag as "complex multi-conflict scenario"
```

### 8.4 Conflict Between Orchestrators

If two Pod Orchestrators disagree (cross-pod conflict), the conflict starts at **Level 3** (System Orchestrator), since Level 1 and 2 are intra-pod mechanisms.

If the System Orchestrator is involved in a conflict (e.g., its decision is challenged), the conflict goes **directly to Level 4** (human), since there is no higher system authority.

```
Pod Orch vs Pod Orch:     → Level 3 (System Orch arbitrates)
Any entity vs System Orch: → Level 4 (Human arbitrates)
```

---

## 9. Invariants (Must Always Hold)

```
INV-1:  Every conflict has a resolution within bounded time (sum of all timeouts)
INV-2:  Every conflict produces a conflict_record in the audit log
INV-3:  A conflict can only escalate upward (L1→L2→L3→L4), never downward
INV-4:  Timeout at any level → automatic escalation (never stuck)
INV-5:  Human veto is available ONLY for safety-critical decisions
INV-6:  Human decisions do not affect any entity's AV
INV-7:  Safety emergency bypasses all escalation levels
INV-8:  Cascading conflict depth is bounded (max_cascade_depth)
INV-9:  Level 2 requires quorum to produce a valid result
INV-10: Level 2 requires confidence above threshold to resolve
INV-11: The safest option is always deterministically identifiable (fallback never fails)
```

---

## 10. Configuration

```
conflict_resolution_config {
    // Level 1
    level_1_timeout_ms:              100

    // Level 2
    level_2_timeout_ms:              500
    level_2_vote_collection_ms:      400
    level_2_quorum_min:              3
    level_2_quorum_ratio:            0.5
    level_2_confidence_threshold:    0.6

    // Level 3
    level_3_timeout_ms:              2000
    level_3_confidence_margin:       0.15
    level_3_scoring_weights {
        av_alignment:                0.25
        safety_score:                0.30
        resource_efficiency:         0.15
        policy_compliance:           0.15
        vote_support:                0.15
    }

    // Level 4
    level_4_timeout_ms:              5000     // deployment-specific
    level_4_human_vote_weight:       1000     // AV_max

    // Special
    max_cascade_depth:               3
    escalation_penalty_enabled:      false    // penalize frivolous escalation?
    escalation_penalty_amount:       5        // minor AV penalty if enabled

    // Safety override
    safety_emergency_bypass:         true     // emergencies skip all levels
}
```

All values are defined at system configuration time and are immutable at runtime.

---

## 11. Interaction with Other Documents

- **Authority Model (01-authority-model.md):** AV comparison is the foundation of Level 1 resolution. AV-weighted voting drives Level 2.
- **Pod Assembly (03-pod-assembly.md):** Cross-pod conflicts detected during assembly or execution route through Level 3 (System Orchestrator).
- **Knowledge Base (12-knowledge-base.md):** When an LLM output contradicts a verified KB entry, the contradiction is treated as a conflict and routed through the same 4-level escalation chain (AV comparison → pod vote → System Orch → human-in-the-loop). KB contradiction resolution follows identical timeout and fallback rules.
