# MYOS Build Order

## Philosophy

Build all core infrastructure components first — fully, not partially. These are the bones of the system. Once every core component exists and compiles, start wiring them together with cubelets one stage at a time.

**Core components** = the things every cubelet, pod, and orchestrator depends on.
**Extensions** = cubelets, stages, networking, chain, edge — things that plug into the core.

You're building with Claude Code. Every task below is a coding task.

---

## Phase 1: Core Components

Build these in order. Each depends on the ones above it.

---

### Component 1: Repository Structure + NixOS Foundation

**What:** The monorepo structure, Nix flake, and development environment.

**Directory structure:**
```
myos/
├── flake.nix                        # Single entry point for entire system
├── flake.lock
├── CLAUDE.md                        # Claude Code project instructions
│
├── authority-engine/                # Haskell
│   ├── cabal.project
│   ├── authority-engine.cabal
│   └── src/
│       ├── MYOS/Authority/Core.hs       # AV vector type, bounds, clamp
│       ├── MYOS/Authority/MBAC.hs       # Reward, penalty, decay formulas
│       ├── MYOS/Authority/Dimensions.hs # Dimension registry
│       ├── MYOS/Authority/ProofPerl.hs  # STK invariant evaluator
│       ├── MYOS/Authority/PerlFrame.hs  # PCCSCEFVRI value anchor
│       ├── MYOS/Authority/Engine.hs     # Main authorization logic (dual-gate)
│       ├── MYOS/Authority/API.hs        # gRPC service definitions
│       └── MYOS/Authority/Config.hs     # Boot-time config loading
│
├── runtime/                         # Rust workspace
│   ├── Cargo.toml                       # Workspace root
│   ├── cig-service/                     # CIG Neo4j query service
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── graph.rs                 # Neo4j connection + queries
│   │       ├── models.rs                # Cubelet, Edge, STSol types
│   │       └── api.rs                   # gRPC service
│   ├── cubelet-runtime/                 # Cubelet launcher + lifecycle
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── launcher.rs              # Tier 1/2/3 dispatch
│   │       ├── wasm.rs                  # Wasmtime AOT + execution
│   │       ├── container.rs             # crun container launch
│   │       ├── ipc.rs                   # Shared memory ring buffers
│   │       ├── manifest.rs              # Cubelet manifest type
│   │       └── lifecycle.rs             # available→locked→released→cooldown
│   ├── pod-assembly/                    # Pod assembly protocol
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── stsol.rs                 # STSol template loader + resolver
│   │       ├── selection.rs             # Cubelet selection scoring
│   │       ├── assembly.rs              # Locking, deadline-wait, instantiation
│   │       ├── dissolution.rs           # Release, AV update, compatibility update
│   │       └── registry.rs              # Task registry + deduplication
│   ├── pod-orchestrator/                # Pod Orch management
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── dag.rs                   # DAG construction from CIG edges
│   │       ├── pipeline.rs              # Pipeline execution engine
│   │       ├── failure.rs               # Tiered failure response (1-4)
│   │       ├── synthesis.rs             # Result collection + merge
│   │       └── session.rs               # Session pod + sub-pod management
│   ├── data-pipeline/                   # Typed DAG + control messages
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── dag.rs                   # DAG type, edge type checking
│   │       ├── schema.rs                # Schema registry
│   │       ├── flow.rs                  # Pipeline execution (fan-out, fan-in)
│   │       ├── control.rs               # Control message types + routing
│   │       └── logging.rs               # Edge hashing, decision logging
│   ├── kb-service/                      # Knowledge Base service
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── hierarchy.rs             # System KB → Domain KB → Pod KB
│   │       ├── confidence.rs            # Confidence scoring (same math as AV)
│   │       ├── query.rs                 # Mandatory query protocol (exact→semantic→LLM)
│   │       ├── contradiction.rs         # Contradiction detection + escalation
│   │       ├── promotion.rs             # Pod→Domain→System knowledge promotion
│   │       └── erasure.rs               # GDPR right-to-erasure flow
│   ├── conflict-resolution/             # 4-level escalation chain
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── level1.rs                # Direct AV comparison
│   │       ├── level2.rs                # AV-weighted pod vote
│   │       ├── level3.rs                # System Orch arbitration (CIG-informed)
│   │       ├── level4.rs                # Human-in-the-loop
│   │       ├── escalation.rs            # Timeout + auto-escalation logic
│   │       └── safety.rs                # Safety emergency bypass
│   ├── verification/                    # Logging + audit (SQLite first, Ouroboros later)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── onchain.rs               # Decision log (SQLite stub, swap to Ouroboros)
│   │       ├── offchain.rs              # Content-addressed store + hash chains
│   │       ├── fabric.rs                # STF fabric thread routing
│   │       └── merkle.rs                # Merkle root computation
│   └── myos-daemon/                     # Main MYOS daemon (ties everything together)
│       ├── Cargo.toml
│       └── src/
│           └── main.rs                  # Boot sequence, service initialization
│
├── cig/                             # CIG seed data + import scripts
│   ├── schemas/                         # Neo4j schema definitions
│   │   └── constraints.cypher
│   ├── seeds/                           # Generated from cubeletuni registry
│   │   ├── stage0_nodes.jsonl
│   │   ├── stage0_edges.jsonl
│   │   ├── stage3_nodes.jsonl
│   │   ├── stage3_edges.jsonl
│   │   └── ...
│   ├── stsol-templates/                 # STSol pod templates
│   │   ├── ai-safety-guard.yaml
│   │   ├── env-mrv-pod.yaml
│   │   ├── id-issuance-pod.yaml
│   │   └── ...
│   ├── import.sh                        # Bulk import script
│   └── health-check.cypher             # Validation queries
│
├── cubelets/                        # Individual cubelet implementations
│   ├── Cargo.toml                       # Workspace for all cubelets
│   ├── cubelet-macros/                  # #[derive(Cubelet)] proc macro
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── stk-3-a-fairness/              # STK-3-A: AI Fairness Invariant Checker
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── sti-3-b-explainability/         # STI-3-B: Explainability Classifier
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── std-3-a-inference/              # STD-3-A: LLM Inference Runtime
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   └── ...                              # More cubelets added per stage
│
├── config/                          # System configuration
│   ├── dimensions.yaml                  # AV dimension registry
│   ├── invariants.yaml                  # 150 STK invariant definitions
│   ├── action-authority-map.yaml        # Action → AV + STK requirements
│   ├── floors.yaml                      # Entity type → AV floors
│   └── pod-orch-templates.yaml          # Pod Orchestrator templates
│
└── nix/                             # Nix modules and overlays
    ├── modules/
    │   ├── authority-engine.nix          # NixOS service module
    │   ├── neo4j.nix                     # Neo4j with CIG import
    │   ├── qdrant.nix                    # Vector DB service
    │   ├── myos-daemon.nix              # Main daemon service
    │   └── cubelet-runtime.nix          # Wasmtime + crun setup
    └── overlays/
        ├── preempt-rt.nix               # PREEMPT_RT kernel overlay
        └── wasmtime.nix                 # Wasmtime AOT overlay
```

**Tasks in order:**

1. **`flake.nix` skeleton** — Define `nixosConfigurations.core-dev` with Neo4j, Qdrant, and basic packages. Should build with `nix build`. Use `haskell.nix` for the Authority Engine overlay. Rust packages via `crane` or `naersk`.

2. **Rust workspace `Cargo.toml`** — Set up workspace with all crate stubs (`cig-service`, `cubelet-runtime`, `pod-assembly`, `pod-orchestrator`, `data-pipeline`, `kb-service`, `conflict-resolution`, `verification`, `myos-daemon`). Every crate compiles (empty `lib.rs` with module declarations).

3. **Haskell cabal project** — `authority-engine.cabal` with dependencies (`grpc-haskell`, `aeson`, `text`, `vector`, `QuickCheck`). All module stubs compile.

4. **CLAUDE.md** — Project instructions for Claude Code: build commands (`nix build .#authority-engine`, `nix build .#myos-runtime`), test commands, architecture summary, key invariants to never violate.

**Exit criteria:** `nix flake check` passes. `nix build .#authority-engine` produces Haskell binary. `nix build .#myos-runtime` produces Rust binary. Neo4j starts via `nixos-rebuild switch`.

---

### Component 2: Cubelet Interaction Graph (CIG) Service

**What:** Neo4j populated with the cubelet lattice. A Rust gRPC service to query it.

**Detailed tasks:**

1. **Neo4j schema** (`cig/schemas/constraints.cypher`):
   ```cypher
   CREATE CONSTRAINT cubelet_id IF NOT EXISTS FOR (c:Cubelet) REQUIRE c.lattice_id IS UNIQUE;
   CREATE INDEX cubelet_stage IF NOT EXISTS FOR (c:Cubelet) ON (c.stage);
   CREATE INDEX cubelet_framework IF NOT EXISTS FOR (c:Cubelet) ON (c.framework);
   ```
   Node labels: `Cubelet`. Properties: `lattice_id`, `framework` (STA|STI|STD|STF|STK), `stage` (0-9), `index` (A-O), `title`, `summary`, `type`, `cubelet_type` (math_bound|ml_dl|llm_agent), `model_hash`, `model_version`, `status` (available|locked|cooldown|error|disabled), `stf_threads` (string array), `stk_invariants` (string array).

   Edge types: `DEPENDS_ON`, `PROVES`, `GOVERNS`, `INFORMS`, `DERIVES`, `ENACTS`, `FEDERATES`, `CROSSWALKS`, `AUDITS`. Each edge has `weight` (float), `created_at` (timestamp).

2. **Seed data generator** — Script that reads `cubeletuni/05-cubeletmasterregistry.md` (the YAML sections) and `cubeletuni/10-full-latticeseeds.md` and produces `*.jsonl` files for Neo4j bulk import. Start with Stage 3 only (75 nodes). Expand later.

3. **Import script** (`cig/import.sh`) — Runs `neo4j-admin database import` or Cypher `LOAD CSV` to populate. Idempotent (can re-run safely). Called during NixOS boot via a systemd oneshot unit that runs after Neo4j starts.

4. **Health check queries** (`cig/health-check.cypher`):
   ```cypher
   // Node count per stage
   MATCH (c:Cubelet) RETURN c.stage, count(c) ORDER BY c.stage;
   // Orphan check (nodes with no edges)
   MATCH (c:Cubelet) WHERE NOT (c)--() RETURN c.lattice_id;
   // Every STD has a PROVES edge to at least one STK
   MATCH (d:Cubelet {framework:'STD'}) WHERE NOT (d)-[:PROVES]->(:Cubelet {framework:'STK'}) RETURN d.lattice_id;
   // Giant component check
   CALL gds.wcc.stream({nodeProjection:'Cubelet', relationshipProjection:{ALL:{type:'*'}}}) YIELD componentId, nodeId RETURN componentId, count(nodeId) ORDER BY count(nodeId) DESC LIMIT 5;
   ```

5. **CIG Rust service** (`runtime/cig-service/`):

   **`models.rs`** — Types:
   ```rust
   pub struct CubeletNode {
       pub lattice_id: String,      // "STI-3-B"
       pub framework: Framework,     // STA, STI, STD, STF, STK
       pub stage: u8,                // 0-9
       pub index: char,              // A-O
       pub title: String,
       pub cubelet_type: CubeletType, // MathBound, MlDl, LlmAgent
       pub model_hash: Option<String>,
       pub stk_invariants: Vec<String>,
       pub stf_threads: Vec<String>,
       pub status: CubeletStatus,
   }

   pub struct CigEdge {
       pub from: String,             // lattice_id
       pub to: String,               // lattice_id
       pub edge_type: EdgeType,      // DependsOn, Proves, Governs, ...
       pub weight: f64,
   }

   pub struct STSolTemplate {
       pub template_id: String,      // "AI-Safety-Guard"
       pub primary_stage: u8,
       pub required_positions: Vec<STSolPosition>,
       pub diagonal_dependencies: Vec<STSolPosition>,
       pub stk_required: Vec<String>,
   }
   ```

   **`graph.rs`** — Neo4j queries via `neo4rs` crate:
   - `get_cubelet(lattice_id: &str) -> Result<CubeletNode>`
   - `get_edges(lattice_id: &str, edge_type: Option<EdgeType>) -> Result<Vec<CigEdge>>`
   - `get_cubelets_by_stage(stage: u8) -> Result<Vec<CubeletNode>>`
   - `get_cubelets_by_framework(framework: Framework) -> Result<Vec<CubeletNode>>`
   - `resolve_stsol(template_id: &str) -> Result<Vec<CubeletNode>>` — follows DEPENDS_ON/PROVES/GOVERNS edges from the template to find all required cubelets
   - `get_governing_policies(lattice_id: &str) -> Result<Vec<CubeletNode>>` — follows GOVERNS edges backwards to find STA cubelets
   - `get_stk_invariants(cubelet_ids: &[String]) -> Result<Vec<String>>` — union of all stk_invariants for a set of cubelets
   - `update_cubelet_status(lattice_id: &str, status: CubeletStatus) -> Result<()>`
   - `health_check() -> Result<CigHealthReport>`

   **`api.rs`** — gRPC service (using `tonic`):
   ```protobuf
   service CigService {
       rpc GetCubelet(CubeletRequest) returns (CubeletResponse);
       rpc GetEdges(EdgesRequest) returns (EdgesResponse);
       rpc ResolveSTSol(STSolRequest) returns (STSolResponse);
       rpc GetGoverningPolicies(CubeletRequest) returns (CubeletsResponse);
       rpc GetSTKInvariants(CubeletIdsRequest) returns (InvariantsResponse);
       rpc UpdateCubeletStatus(StatusUpdateRequest) returns (StatusUpdateResponse);
       rpc HealthCheck(Empty) returns (HealthReport);
   }
   ```

6. **STSol template files** (`cig/stsol-templates/`):
   ```yaml
   # ai-safety-guard.yaml
   template_id: "AI-Safety-Guard"
   primary_stage: 3
   description: "AI inference with safety verification"
   required_positions:
     - lattice_id: "STA-3-B"
       role: governing_policy
       priority: required
     - lattice_id: "STI-3-B"
       role: reasoning_engine
       priority: required
     - lattice_id: "STD-3-A"
       role: execution
       priority: required
     - lattice_id: "STF-3-A"
       role: fabric_connector
       priority: preferred
     - lattice_id: "STK-3-A"
       role: proof_checker
       priority: required
   diagonal_dependencies: []
   stk_required:
     - "safety.refusal_on_redline"
     - "fairness<=θ"
     - "explainability.required"
   ```

**Exit criteria:** Neo4j has 75 nodes (Stage 3) with all edges. `GetCubelet("STI-3-B")` returns full node data. `ResolveSTSol("AI-Safety-Guard")` returns 5 cubelet positions. `HealthCheck` passes all validation queries.

---

### Component 3: Authority Engine

**What:** Haskell MBAC implementation with ProofPerl, gRPC API, NixOS service.

**Detailed tasks:**

1. **`MYOS/Authority/Core.hs`** — Foundation types:
   ```haskell
   -- Authority Value: a single dimension
   newtype AV = AV { unAV :: Double }
     deriving (Eq, Ord, Show)

   -- Smart constructor: clamps to [0, avMax]
   mkAV :: Double -> Double -> AV
   mkAV maxVal x = AV (max 0 (min maxVal x))

   -- Authority Vector: multiple dimensions
   newtype AuthorityVector = AuthorityVector (Map DimensionName AV)

   -- Dimension name (execution, safety, reasoning, etc.)
   newtype DimensionName = DimensionName Text

   -- Entity types with their floors
   data EntityType = SystemOrch | DomainOrch | DeploymentOrch | PodOrch | Cubelet

   -- Floor lookup
   entityFloor :: EntityType -> DimensionName -> Config -> AV
   ```

2. **`MYOS/Authority/MBAC.hs`** — The math:
   ```haskell
   -- Diminishing returns reward
   effectiveReward :: Double -> AV -> Double -> Double -> AV
   -- effectiveReward baseReward currentAV avMax k =
   --   baseReward * (1 - unAV currentAV / avMax) ** k

   -- Constant penalty
   applyPenalty :: Double -> Double -> AV -> AV -> AV
   -- applyPenalty basePenalty severityMultiplier currentAV floor =
   --   max floor (currentAV - basePenalty * severityMultiplier)

   -- Lazy decay
   applyDecay :: Double -> Double -> AV -> AV -> AV
   -- applyDecay decayRate inactivityHours currentAV floor =
   --   max floor (currentAV - decayRate * inactivityHours)

   -- Full update rule
   updateAV :: UpdateRequest -> Config -> AuthorityVector -> AuthorityVector
   ```
   **QuickCheck properties:**
   - `prop_avNeverExceedsCeiling`: After any sequence of updates, no dimension exceeds avMax
   - `prop_avNeverBelowFloor`: After any sequence of updates, no dimension drops below floor
   - `prop_rewardDiminishes`: effectiveReward at AV=900 < effectiveReward at AV=100
   - `prop_penaltyConstant`: penalty at AV=900 == penalty at AV=100
   - `prop_floorHierarchy`: floor(SystemOrch) > floor(DomainOrch) > ... > floor(Cubelet) always

3. **`MYOS/Authority/Dimensions.hs`** — Dimension registry:
   ```haskell
   data DimensionDef = DimensionDef
     { dimName        :: DimensionName
     , dimDescription :: Text
     , dimAvMax       :: Double
     , dimFloors      :: Map EntityType Double
     , dimDecayRate   :: Double
     , dimPCCSCEFVRI  :: [Text]         -- ethical value mapping
     }

   -- Load from config at boot, immutable after
   loadDimensionRegistry :: Config -> Either ConfigError DimensionRegistry
   ```

4. **`MYOS/Authority/ProofPerl.hs`** — STK invariant evaluator:
   ```haskell
   data Invariant = Invariant
     { invId            :: Text           -- "safety.refusal_on_redline"
     , invExpression    :: InvExpr        -- parsed expression AST
     , invParameters    :: Map Text Double -- { "redline_threshold": 0.95 }
     , invPerlFrame     :: [Text]         -- PCCSCEFVRI anchors
     , invFamily        :: InvFamily      -- SafetySecurity | PrivacyConsent | ...
     , invStage         :: Int
     , invLocked        :: Bool
     }

   data InvResult = Pass | Fail Text | Indeterminate Text

   -- Pure evaluation function
   evaluate :: Invariant -> SystemState -> InvResult

   -- Evaluate all relevant invariants for a set of cubelets
   evaluateAll :: [Invariant] -> SystemState -> [(Text, InvResult)]

   -- Load invariants from config at boot
   loadInvariants :: Config -> Either ConfigError [Invariant]
   ```
   Start with 3 invariants hardcoded, then load from YAML.

5. **`MYOS/Authority/PerlFrame.hs`** — Value anchor:
   ```haskell
   data PCCSCEFVRIValue = Privacy | Care | Consent | Safety | Creativity
                        | Equity | Fairness | Verifiability | Responsibility | Integrity

   -- Precedence order for invariant conflict resolution
   familyPrecedence :: InvFamily -> Int
   -- SafetySecurity = 0 (highest), PrivacyConsent = 1, ... EthicalReflexivity = 6 (lowest)

   -- Check if an invariant maps to a given PCCSCEFVRI value
   anchoredTo :: Invariant -> PCCSCEFVRIValue -> Bool
   ```

6. **`MYOS/Authority/Engine.hs`** — Main authorization logic:
   ```haskell
   data AuthDecision = Allow | Deny Text | Escalate Text

   data AuthRequest = AuthRequest
     { reqEntityId   :: EntityId
     , reqAction     :: ActionType
     , reqCubeletIds :: [Text]       -- lattice IDs involved
     , reqContext    :: ActionContext
     }

   -- Dual-gate authorization
   authorize :: AuthRequest -> EngineState -> AuthDecision
   -- 1. Gate 1: AV check — entity.AV[relevant_dim] >= required_authority
   -- 2. Gate 2: STK invariant check — all stk_required invariants pass via ProofPerl
   -- 3. Both pass → Allow. AV fails → Escalate. STK fails → Deny (always).

   -- State management
   data EngineState = EngineState
     { entities        :: Map EntityId AuthorityVector
     , dimensionReg    :: DimensionRegistry
     , invariants      :: [Invariant]
     , actionAuthMap   :: Map ActionType ActionRequirements
     , config          :: Config
     }
   ```

7. **`MYOS/Authority/API.hs`** — gRPC service:
   ```protobuf
   service AuthorityEngine {
       rpc Authorize(AuthorizeRequest) returns (AuthorizeResponse);
       rpc GetAuthority(EntityIdRequest) returns (AuthorityVectorResponse);
       rpc UpdateAuthority(UpdateRequest) returns (AuthorityVectorResponse);
       rpc CompareAuthority(CompareRequest) returns (CompareResponse);
       rpc AuditTrail(AuditRequest) returns (stream AuditEvent);
       rpc EvaluateInvariants(InvariantsRequest) returns (InvariantsResponse);
   }
   ```

8. **`MYOS/Authority/Config.hs`** — Boot-time config loading:
   - Load `config/dimensions.yaml` → DimensionRegistry
   - Load `config/invariants.yaml` → [Invariant]
   - Load `config/action-authority-map.yaml` → ActionAuthMap
   - Load `config/floors.yaml` → Floor definitions
   - Verify config signature (BLAKE3 hash against known hash)
   - All config immutable after load

9. **NixOS service module** (`nix/modules/authority-engine.nix`):
   ```nix
   services.myos.authority-engine = {
     enable = true;
     configDir = ./config;
     listenAddress = "127.0.0.1:50051";
     heapSize = "256m";
   };
   ```
   Systemd unit with `Restart=always`, watchdog timer, `DenyAllOnCrash` behavior.

10. **Config files:**
    - `config/dimensions.yaml` — Start with 4 dimensions: execution, safety, reasoning, knowledge
    - `config/invariants.yaml` — Start with 3 invariants: `safety.refusal_on_redline`, `fairness<=θ`, `explainability.required`
    - `config/action-authority-map.yaml` — 5 action types with AV + STK requirements
    - `config/floors.yaml` — SystemOrch=700, DomainOrch=600, DeploymentOrch=500, PodOrch=400, Cubelet=0

**Exit criteria:** `nix build .#authority-engine` produces binary. `Authorize` returns Allow/Deny/Escalate. QuickCheck passes 10M random sequences. ProofPerl evaluates 3 invariants correctly. gRPC server responds within 5ms.

---

### Component 4: Cubelet Runtime

**What:** Launches cubelets in the correct isolation tier based on their type.

**Detailed tasks:**

1. **`cubelet-runtime/src/manifest.rs`** — Cubelet manifest type (mirrors CIG node + runtime info):
   ```rust
   pub struct CubeletManifest {
       pub lattice_id: String,
       pub cubelet_type: CubeletType,    // MathBound, MlDl, LlmAgent
       pub framework: Framework,
       pub stage: u8,
       pub model_hash: String,
       pub capabilities: Vec<Capability>,
       pub constraints: ResourceConstraints, // max_memory, max_cpu_ms, deterministic
       pub stk_invariants: Vec<String>,
       pub stf_threads: Vec<String>,
       pub io_declaration: IoDeclaration,    // input ports, output ports, schemas
   }
   ```

2. **`cubelet-runtime/src/launcher.rs`** — Tier dispatch:
   ```rust
   pub enum IsolationTier {
       Tier1WasmUnikernel,  // Math-bound: Wasm inside Unikraft (strongest)
       Tier2WasmNative,     // ML/DL: Wasm sandbox only
       Tier3Container,      // LLM: Linux crun container
   }

   impl CubeletLauncher {
       pub fn resolve_tier(manifest: &CubeletManifest) -> IsolationTier {
           match manifest.cubelet_type {
               CubeletType::MathBound => IsolationTier::Tier1WasmUnikernel,
               CubeletType::MlDl => IsolationTier::Tier2WasmNative,
               CubeletType::LlmAgent => IsolationTier::Tier3Container,
           }
       }

       pub async fn launch(&self, manifest: &CubeletManifest) -> Result<RunningCubelet> {
           let tier = Self::resolve_tier(manifest);
           match tier {
               Tier1WasmUnikernel => self.launch_wasm_unikernel(manifest).await,
               Tier2WasmNative => self.launch_wasm_native(manifest).await,
               Tier3Container => self.launch_container(manifest).await,
           }
       }
   }
   ```
   For Sprint 1, implement `Tier2WasmNative` and `Tier3Container` only. Unikraft (Tier 1) is a later optimization — use Tier 2 as fallback.

3. **`cubelet-runtime/src/wasm.rs`** — Wasmtime integration:
   - AOT compile `.wasm` → `.cwasm` at boot (pre-warm)
   - Instantiate module with WASI capabilities (minimal: clock, random, args)
   - Fuel metering for execution time limits
   - Memory limits from manifest constraints
   - Typed I/O: host functions for reading input / writing output

4. **`cubelet-runtime/src/container.rs`** — Container launch:
   - Use `crun` (lightweight OCI runtime) or Podman
   - Resource limits from manifest: CPU shares, memory limit, no network (unless STF cubelet)
   - Mount input/output via shared memory or Unix socket
   - Health check: container alive + responsive within timeout

5. **`cubelet-runtime/src/ipc.rs`** — Inter-cubelet communication:
   - Shared memory ring buffer for intra-pod data pipeline
   - Unix domain socket for control messages to Pod Orchestrator
   - `SO_PEERCRED` authentication on control socket
   - Zero-copy: write once to shared memory, all downstream cubelets read from same address

6. **`cubelet-runtime/src/lifecycle.rs`** — State machine:
   ```rust
   pub enum CubeletStatus {
       Available,
       Locked { pod_id: PodId },
       Cooldown { until: Instant },
       Error { reason: String },
       Disabled,
   }

   pub fn transition(current: CubeletStatus, event: LifecycleEvent) -> CubeletStatus {
       // available → locked (selected for pod)
       // locked → available (pod completes, no cooldown)
       // locked → cooldown (pod completes, cooldown needed)
       // cooldown → available (cooldown expires)
       // locked → available (pod dissolved/failed)
       // any → error (error detected)
       // error → available (recovered)
   }
   ```

**Exit criteria:** A Wasm cubelet loads, receives typed input, produces typed output, respects resource limits. A container cubelet launches, responds to health check, receives input via IPC. Lifecycle state machine transitions correctly.

---

### Component 5: Data Pipeline

**What:** Typed DAG construction, schema registry, pipeline execution with framework ordering.

**Detailed tasks:**

1. **`data-pipeline/src/schema.rs`** — Schema registry:
   ```rust
   pub struct SchemaRegistry {
       schemas: HashMap<TypeId, SchemaDefinition>,
   }

   pub struct SchemaDefinition {
       pub type_id: String,          // "classification.label"
       pub fields: Vec<FieldDef>,
       pub version: semver::Version,
   }

   impl SchemaRegistry {
       pub fn check_compatibility(&self, output_type: &str, input_type: &str) -> bool;
       pub fn register(&mut self, schema: SchemaDefinition) -> Result<()>;
   }
   ```

2. **`data-pipeline/src/dag.rs`** — DAG type with validation:
   ```rust
   pub struct PipelineDAG {
       nodes: Vec<PipelineNode>,      // cubelet lattice_ids
       edges: Vec<PipelineEdge>,      // typed data flow
   }

   pub struct PipelineEdge {
       pub from: String,              // source cubelet lattice_id
       pub from_port: String,         // output port name
       pub to: String,                // dest cubelet lattice_id
       pub to_port: String,           // input port name
       pub type_id: String,           // schema type flowing through this edge
   }

   impl PipelineDAG {
       /// Validate: no cycles, all edges type-checked, framework ordering respected
       pub fn validate(&self, schema_reg: &SchemaRegistry) -> Result<()>;
       /// Check framework ordering: STA before STI before STD before STF before STK
       pub fn validate_framework_ordering(&self) -> Result<()>;
       /// Topological sort for execution order
       pub fn execution_order(&self) -> Result<Vec<String>>;
   }
   ```

3. **`data-pipeline/src/flow.rs`** — Pipeline execution:
   ```rust
   pub async fn execute_pipeline(
       dag: &PipelineDAG,
       cubelets: &HashMap<String, RunningCubelet>,
       authority: &AuthorityClient,
   ) -> Result<PipelineResult> {
       let order = dag.execution_order()?;
       for node_id in order {
           // 1. Authority check: authorize this cubelet for this step
           // 2. Collect inputs from upstream edges
           // 3. Execute cubelet with inputs
           // 4. Type-check output against declared schema
           // 5. Hash output (off-chain logging)
           // 6. Route output to downstream edges
           // 7. If STF cubelet: log to fabric thread
           // 8. If STK cubelet: verify invariant, log result
       }
   }
   ```

4. **`data-pipeline/src/control.rs`** — Control message routing:
   ```rust
   pub enum ControlMessage {
       StatusReport { cubelet_id: String, status: CubeletHealth },
       Alert { cubelet_id: String, severity: Severity, message: String },
       ConfigUpdate { cubelet_id: String, config: serde_json::Value },
       Escalation { cubelet_id: String, conflict: ConflictRecord },
   }
   // StatusReport and Alert are NEVER authority-gated
   // ConfigUpdate requires AV check
   // Cubelet → Pod Orch only. Never cubelet → cubelet directly.
   ```

5. **`data-pipeline/src/logging.rs`** — Edge hashing + decision logging:
   - Every pipeline edge data transfer: BLAKE3 hash of (input + output + cubelet_id + timestamp)
   - Every authority decision: full decision context (entity, action, AV snapshot, STK results)
   - Logged to verification service (SQLite in Component 8, Ouroboros later)

**Exit criteria:** A 3-node DAG (STD→STI→STK) is constructed, validated (type-check + framework ordering), executed, and every step is logged with hashes.

---

### Component 6: Pod Assembly

**What:** STSol template resolution → cubelet selection → locking → pod instantiation.

**Detailed tasks:**

1. **`pod-assembly/src/stsol.rs`** — Template loading:
   - Load STSol YAML templates from `cig/stsol-templates/`
   - `resolve(template_id)` → calls CIG service to follow DEPENDS_ON/PROVES edges → returns full list of required cubelet positions

2. **`pod-assembly/src/selection.rs`** — Cubelet scoring:
   ```rust
   pub fn score_cubelet(
       candidate: &CubeletNode,
       slot: &STSolPosition,
       already_selected: &[CubeletNode],
   ) -> f64 {
       let av_score = candidate.av[&slot.relevant_dimension];
       let compat = compatibility_bonus(candidate, already_selected);
       let success = candidate.success_rate;
       let load = candidate.recent_task_load;
       0.50 * av_score + 0.25 * compat + 0.15 * success - 0.10 * load
   }
   ```

3. **`pod-assembly/src/assembly.rs`** — Full assembly flow:
   ```rust
   pub async fn assemble_pod(
       task: &TaskDescriptor,
       cig: &CigClient,
       authority: &AuthorityClient,
       runtime: &CubeletRuntime,
   ) -> Result<Pod> {
       // 1. Deduplicate: check task registry for active duplicate
       // 2. Resolve STSol template → required cubelet positions
       // 3. For each position (scarce first):
       //    a. Query CIG for available cubelets at that position
       //    b. Filter: available status, AV >= threshold
       //    c. Score candidates
       //    d. Select top candidate
       //    e. Lock cubelet (update status in CIG)
       // 4. Authority check: all cubelets meet pod requirements
       // 5. STK invariant pre-check: are all stk_required satisfiable?
       // 6. Create Pod manifest
       // 7. Assign Pod Orchestrator (from template or registry)
       // 8. Launch all cubelets via runtime
       // 9. Return running Pod
   }
   ```

4. **`pod-assembly/src/dissolution.rs`** — Pod teardown:
   ```rust
   pub async fn dissolve_pod(pod: &Pod, outcome: PodOutcome) -> Result<()> {
       // 1. Stop all cubelet execution
       // 2. For each cubelet:
       //    a. Update AV: reward (success) or penalty (failure)
       //    b. Update compatibility scores (pairwise)
       //    c. Release lock (update status in CIG)
       //    d. Apply cooldown if needed
       // 3. Promote Pod KB discoveries to Domain KB
       // 4. Log dissolution event (verification service)
       // 5. Destroy Pod KB
   }
   ```

5. **`pod-assembly/src/registry.rs`** — Task registry + dedup:
   - In-memory HashMap of active tasks (task_id → PodId)
   - Content-based dedup: `task_id = BLAKE3(task_type + context_params)`
   - Lock acquisition: atomic compare-and-swap on task_id

**Architecture notes (from finalComp/repo audit):**
- STSol templates now reference Soul Papers (confidence=1000 KB entries) as the founding document for each solution — the Soul Paper defines the normative intent the pod must fulfill
- Assembly must validate cross-framework handshake contracts (doc 13) before pipeline activation — each cubelet pair crossing a framework boundary must have a signed handshake proving type compatibility and authority delegation
- Pod lifecycle tracks STD/STS lifecycle phases: Aggregation → Coordination → Execution → Evaluation → Adaptation → Sunset — dissolution triggers differ per phase

**Exit criteria:** `assemble_pod("AI-Safety-Guard")` resolves STSol → selects 3-5 cubelets from CIG → locks them → launches pipeline → returns running Pod. `dissolve_pod` releases all cubelets with correct AV updates.

---

### Component 7: Pod Orchestrator

**What:** LLM-powered coordinator that builds DAGs, manages pipelines, handles failures.

**Detailed tasks:**

1. **`pod-orchestrator/src/dag.rs`** — DAG construction from CIG:
   ```rust
   pub fn build_dag(
       task: &TaskDescriptor,
       cubelets: &[CubeletNode],
       cig_edges: &[CigEdge],
       schema_reg: &SchemaRegistry,
   ) -> Result<PipelineDAG> {
       // 1. Start with framework ordering: STA → STI → STD → STF → STK
       // 2. Use CIG DEPENDS_ON edges to wire data flow
       // 3. Use CIG PROVES edges to connect STD outputs to STK verifiers
       // 4. Use CIG GOVERNS edges to place STA policy cubelets upstream
       // 5. Validate all edges type-check against schema registry
       // 6. LLM reasoning (via Ollama) for ambiguous cases:
       //    - Multiple valid orderings within a framework tier
       //    - Fan-out optimization decisions
       //    - Skipping optional cubelets
   }
   ```

2. **`pod-orchestrator/src/pipeline.rs`** — Execution management:
   - Start pipeline execution (delegates to data-pipeline crate)
   - Monitor cubelet health during execution
   - Collect intermediate results
   - Synthesize final output from parallel branches (LLM reasoning for synthesis)

3. **`pod-orchestrator/src/failure.rs`** — Tiered failure response:
   ```rust
   pub async fn handle_failure(
       cubelet_id: &str,
       error: &CubeletError,
       pod: &mut Pod,
   ) -> Result<FailureRecovery> {
       // Tier 1: Retry (same cubelet, same input, max 2x)
       // Tier 2: Quarantine + Compensate (isolate cubelet, AV penalty, find replacement)
       // Tier 3: Adaptive Restructure (rewire DAG, logged on-chain)
       // Tier 4: Escalate to System Orch (pod cannot self-recover)
   }
   ```

4. **`pod-orchestrator/src/session.rs`** — Session pod management:
   - Long-lived pod for user conversations
   - Context manager cubelet (conversation state)
   - Safety monitor cubelet (output filtering)
   - Sub-pod spawning for complex queries (fan-out, no nesting)
   - Tiered release: specialized cubelets released on idle, core kept until session timeout

**Architecture notes (from finalComp/repo audit):**
- Pod Orch manages framework lifecycle phase transitions for its cubelets — tracking each cubelet's position in the Aggregation → Coordination → Execution → Evaluation → Adaptation → Sunset lifecycle
- DAG construction must respect DSL verb hierarchy: STA `select{}` → STI `bind{}` → STD `realize{}` → STF `weave{}` → STK `enforce{}` — this is the canonical ordering from the DSL spec, not just framework alphabetical order
- Rollout controls for pod changes: canary deployment (route subset of traffic to new DAG), soak period (minimum observation window before full cutover), auto-rollback if `proof_failure_rate` exceeds threshold
- Coherence self-check now includes cross-framework handshake validation — Pod Orch must verify all handshake contracts remain valid after any DAG restructure or cubelet replacement
- **OpenClaw patterns:** Reasoning budget (minimal/standard/deep) per Pod Orch, per-session tool allowlisting as defense-in-depth complementing MBAC
- **NemoClaw patterns:** Inference Router already implemented in cubelet-runtime (transparent routing, privacy-aware, SecretResolver integration)
- **Workflow gates:** Lobster-inspired approval checkpoints in STSol templates for human-in-the-loop at pipeline stages (not just conflict resolution Level 4)

**Exit criteria:** Pod Orch receives task + cubelets → builds DAG respecting framework ordering → executes pipeline → synthesizes result → handles cubelet failure with tier 1-2 recovery. Session pod persists across multiple queries.

---

### Component 8: Conflict Resolution

**What:** 4-level escalation chain.

**Detailed tasks:**

1. **`conflict-resolution/src/level1.rs`** — Direct AV comparison (100ms timeout)
2. **`conflict-resolution/src/level2.rs`** — AV-weighted pod vote (500ms timeout, quorum >= 3, confidence >= 0.6)
3. **`conflict-resolution/src/level3.rs`** — System Orch arbitration (2000ms, CIG-informed scoring with STK compliance 0.25 + STA alignment 0.20 + safety 0.25 + AV 0.10 + resource 0.10 + vote 0.10)
4. **`conflict-resolution/src/level4.rs`** — Human-in-the-loop (configurable timeout, vote/veto/delegate)
5. **`conflict-resolution/src/escalation.rs`** — Timeout auto-escalation, cascade depth tracking (max 3), safety emergency bypass
6. **`conflict-resolution/src/safety.rs`** — Safety emergency: bypass all levels, execute safest action immediately, P0 priority

**Architecture notes (from finalComp/repo audit):**
- Level 3 arbitration uses STK+ resilience data (anomalous AV trajectories, circuit breaker state) — the System Orch can factor in whether a cubelet's AV has been volatile or whether its circuit breaker has tripped recently
- STK invariant conflicts resolved by family precedence (from PerlFrame) with Hard > Soft > Adaptive constraint ordering — Hard constraints (e.g., `safety.refusal_on_redline`) are immutable and always win; Soft constraints require Level 4 authority to override; Adaptive constraints auto-tune within bounds
- Inter-kernel federation disputes routed through treaty ledger — when cubelets from different kernel domains conflict, the treaty ledger (from fabric thread taxonomy) provides the adjudication record

**Exit criteria:** Two cubelets disagree → escalation chain runs L1→L2→L3→L4 with correct timeouts. Timeout at any level triggers next level. Safety emergency bypasses everything.

---

### Component 9: Verification Service

**What:** Decision logging (SQLite now, Ouroboros later) + off-chain content store.

**Detailed tasks:**

1. **`verification/src/onchain.rs`** — SQLite decision log:
   ```rust
   pub struct DecisionRecord {
       pub decision_id: Uuid,
       pub entity_id: String,
       pub action: String,
       pub av_snapshot: serde_json::Value,
       pub stk_results: Vec<(String, InvResult)>,
       pub decision: AuthDecision,
       pub timestamp: DateTime<Utc>,
       pub signature: Vec<u8>,      // BLAKE3 + ML-DSA
   }
   ```
   Trait `OnChainStore` with SQLite impl now, Ouroboros impl later.

2. **`verification/src/offchain.rs`** — Content-addressed store:
   - Pipeline edge data hashed with BLAKE3
   - Stored in content-addressed directory: `{hash[0:2]}/{hash[2:4]}/{hash}`
   - Tiered retention: hot (in-memory LRU) → warm (SSD) → cold (configurable archive)

3. **`verification/src/fabric.rs`** — STF fabric thread routing:
   - Route decision logs to the correct fabric thread based on cubelet's `stf_threads` field
   - EthosLedger: authority events. KarbonLedger: environmental data. ResearchLedger: AI inference logs.

4. **`verification/src/merkle.rs`** — Periodic merkle root:
   - Batch off-chain hashes → compute merkle tree → store root in on-chain log
   - Frequency configurable (every N minutes or every N records)

**Architecture notes (from finalComp/repo audit):**
- Statistical monitoring thresholds: `proof_success >= 98%`, `telemetry_coverage >= 90%`, `interop_success >= 95%` — these are the health gates; breaching any threshold triggers alert escalation
- Maturity level tracking: L1 (visible) → L2 (interoperable) → L3 (provable), with FCR (First-Call Resolution) metric — cubelets and pods advance maturity levels as they demonstrate sustained compliance
- STK+ observability thread: merkle root every 5 minutes, BLAKE3 hash chains — the 5-minute cadence is the default; configurable per deployment but must not exceed 10 minutes for compliance
- Decision context logging with `DecisionType` enum (already defined in `data-pipeline/src/logging.rs`) — all verification records must include the decision type for audit filtering

**Exit criteria:** Every authority decision is logged with full context. Off-chain data is content-addressed and tamper-evident. Merkle roots link off-chain to on-chain.

---

### Component 10: Knowledge Base Service

**What:** Hierarchical KB on CIG backbone, confidence scoring, mandatory query protocol.

**Detailed tasks:**

1. **`kb-service/src/hierarchy.rs`** — System KB → Domain KBs → Pod KBs
   - System KB: global, on core server, write requires knowledge AV >= 800
   - Domain KBs: one per active fabric thread, write requires knowledge AV >= 500
   - Pod KBs: ephemeral, created/destroyed with pods, write requires knowledge AV >= 100
   - Knowledge flows down (inheritance), discoveries flow up (promotion)

2. **`kb-service/src/confidence.rs`** — Same math as MBAC AV:
   ```rust
   pub fn confirm(current_confidence: f64, base_reward: f64, max: f64) -> f64 {
       current_confidence + base_reward * (1.0 - current_confidence / max)
   }
   pub fn contradict(current_confidence: f64, base_penalty: f64, floor: f64) -> f64 {
       (current_confidence - base_penalty).max(floor)
   }
   pub fn decay(current_confidence: f64, hours_since_confirm: f64, rate: f64, floor: f64) -> f64 {
       (current_confidence - rate * hours_since_confirm).max(floor)
   }
   ```

3. **`kb-service/src/query.rs`** — Mandatory three-step protocol:
   - Step 1: Exact match in Neo4j (CIG node properties) → VERIFIED if confidence >= 800
   - Step 2: Semantic search in Qdrant (vector similarity) → PARTIALLY VERIFIED
   - Step 3: LLM fallback (no KB support) → UNVERIFIED
   - Contradiction check: if LLM output contradicts VERIFIED entry → BLOCK + escalate

4. **`kb-service/src/promotion.rs`** — Knowledge flows up:
   - Pod dissolution → evaluate Pod KB entries → promote to Domain KB with 0.5x confidence discount
   - Domain → System promotion with 0.7x confidence discount
   - Requires sufficient knowledge AV from the promoting agent

5. **`kb-service/src/erasure.rs`** — GDPR right-to-erasure:
   - Off-chain: delete from Qdrant + Neo4j properties
   - On-chain: pseudonymize author_id, log erasure event
   - Cascade: track promoted copies, erase all

**Architecture notes (from finalComp/repo audit):**
- Soul Papers stored as System KB entries with confidence=1000 (non-decaying) — these are the founding documents for each STSol and are never subject to confidence decay or contradiction override
- Domain KBs mapped to fabric thread taxonomy: SIPs (System Integrity Proofs), Sectoral (domain-specific), Cross-Domain (shared between stages), Ethical (PerlFrame-anchored), Resilience (STK+ recovery knowledge)
- 18+ named ledgers from thread taxonomy feed into domain KB structure — each ledger (EthosLedger, KarbonLedger, IdentLedger, ResearchLedger, TreatyLedger, etc.) has a corresponding Domain KB partition that receives its verified entries

**Exit criteria:** KB query returns VERIFIED/PARTIAL/UNVERIFIED results. Contradictions are blocked and escalated. Confidence scores update correctly. Pod KB promotes knowledge on dissolution.

---

## Phase 2: First Domain Vertical + End-to-End Integration

Once all 10 core components exist and compile, wire them together — not with isolated cubelets, but with a **complete domain vertical**. Pick a real use case, build every cubelet it needs across all 5 frameworks (STA→STI→STD→STF→STK), and test the full pipeline with real data and real models.

### Why Domain Verticals, Not Individual Cubelets

A single cubelet proves nothing — it's just a function in a sandbox. A full domain vertical proves the **entire architecture works**: the lattice selects cubelets correctly, the pipeline follows framework ordering, the authority engine checks real invariants, the KB grounds real outputs, and the logging captures everything.

It also forces you to **train real models early**. You can't test STI cubelets (ML/DL classifiers, rankers, anomaly detectors) without trained models. Building domain-by-domain means you train the models the domain needs, deploy them to their lattice positions, and validate them against the STK invariants for that domain.

---

### First Domain: AI Safety Verification (Stage 3)

**Why this first:** It's the most immediately useful domain, exercises all three model types, and is the core differentiator of MYOS — AI that verifies its own safety.

**STSol template:** `AI-Safety-Guard`

**Cubelets to build (one per framework, minimum viable):**

| Lattice ID | Framework | Type | What It Does | Model/Implementation | Training Needed? |
|-----------|-----------|------|-------------|---------------------|-----------------|
| **STA-3-A** | Abstraction | Math-bound | AI Ethics Charter — encodes the rules: what constitutes a safe AI output, bias thresholds, refusal conditions | Rust rule engine. Loads policy config (JSON/YAML). Evaluates rules deterministically. | No — hand-written rules |
| **STA-3-B** | Abstraction | Math-bound | AI Risk Policy — risk scoring formula: maps input features to risk level (0-1) | Rust scoring function. `risk = f(topic_sensitivity, confidence, user_context)` | No — formula-based |
| **STI-3-A** | Interpretation | ML/DL | Explainability Classifier — given an LLM response + prompt, classifies whether the response is explainable (has clear reasoning chain) | Small transformer or XGBoost. Input: response text features (length, structure, hedging words, citation density). Output: explainability score 0-1 | **Yes** — train on labeled (prompt, response, explainability_score) dataset. Start with synthetic data from LLM-generated examples. |
| **STI-3-B** | Interpretation | ML/DL | Fairness Detector — given an LLM response, detects bias along protected dimensions (gender, race, age, etc.) | Fine-tuned small transformer (distilBERT or similar) for NLI-style bias detection. Input: response text. Output: bias scores per dimension. | **Yes** — train on bias benchmarks (BBQ, WinoBias, CrowS-Pairs). Fine-tune, export to ONNX. |
| **STI-3-C** | Interpretation | ML/DL | Topic Sensitivity Classifier — classifies input query into sensitivity categories (medical, legal, financial, general, harmful) | XGBoost or small BERT classifier. Input: query text. Output: sensitivity category + confidence. | **Yes** — train on topic classification dataset. Can start with synthetic categories. |
| **STD-3-A** | Deliverance | LLM | Inference Runtime — the actual LLM that generates responses to user queries | Llama 8B via Ollama. Container cubelet. Standard prompt template with system prompt injected from STA policies. | No — use pre-trained model as-is |
| **STD-3-B** | Deliverance | Math-bound | Guardrail Runtime — post-processing filter that applies STA rules to STD-3-A output. Redacts, modifies, or blocks responses that violate policies. | Rust rule engine. Takes (original_response, sta_rules, sti_scores) → filtered_response or BLOCKED. | No — deterministic rule application |
| **STF-3-A** | Fabric | Math-bound | ResearchLedger Connector — logs inference events (prompt hash, response hash, safety scores, timestamps) to the ResearchLedger fabric thread | Rust service. Serializes events, routes to verification service tagged with "ResearchLedger". | No — infrastructure code |
| **STK-3-A** | Kernel | Math-bound | Safety Invariant Checker — evaluates `safety.refusal_on_redline`: if STI-3-C classified query as harmful AND STD-3-A generated a response anyway → FAIL | Rust invariant evaluator. Input: (query_sensitivity, response_exists, risk_score). Output: pass/fail. | No — formal constraint |
| **STK-3-B** | Kernel | Math-bound | Fairness Invariant Checker — evaluates `fairness <= θ`: if STI-3-B bias scores exceed threshold θ on any dimension → FAIL | Rust invariant evaluator. Input: bias_scores from STI-3-B. Output: pass/fail with dimension breakdown. | No — threshold comparison |
| **STK-3-C** | Kernel | Math-bound | Explainability Invariant Checker — evaluates `explainability.required`: if STI-3-A score < minimum threshold → FAIL (response is not explainable enough) | Rust invariant evaluator. Input: explainability_score from STI-3-A. Output: pass/fail. | No — threshold comparison |

**That's 11 cubelets: 6 math-bound (no training), 3 ML/DL (need training), 2 LLM (pre-trained).**

**Architecture notes for Stage 3 assembly (from finalComp/repo audit):**
- Each cubelet must declare its framework lifecycle phase (from doc 13) — at assembly time, every cubelet reports whether it is in Aggregation, Coordination, Execution, etc., and the pod cannot activate until all are in a compatible phase
- STSol Soul Paper for AI-Safety-Guard must be created as a KB entry before pod assembly — this is a confidence=1000 System KB entry that defines the normative safety intent; pod assembly will refuse to proceed without it
- Cross-framework handshakes verified between all 11 cubelets before pipeline activation — particularly the STA→STI, STI→STD, STD→STF, and STF→STK boundary crossings must each have a signed handshake contract
- Privacy Router configuration needed for STD-3-A (inference runtime) calling Ollama API — the container cubelet must route LLM API calls through the Privacy Router to prevent prompt leakage and enforce data sovereignty
- L7 network policies needed for container cubelets (from OpenShell patterns in doc 08) — STD-3-A and any other Tier 3 container cubelets require explicit L7 (application-layer) network policies; default is deny-all
- Adaptive constraint support: `explainability.required` is Soft (adjustable with Level 4 authority), `safety.refusal_on_redline` is Hard (immutable, cannot be overridden at any authority level)
- **Inference routing:** InferenceRouter created per-pod at assembly, shared by STD-3-A (inference runtime). Routes to local Ollama by default, privacy-aware failover to cloud if configured. SecretResolver handles API key isolation.

### Model Training Plan for Stage 3

You need 3 trained models. Here's the practical approach:

**STI-3-A (Explainability Classifier):**
- **Architecture:** XGBoost on hand-crafted features (cheapest, fastest to iterate)
- **Features:** response length, sentence count, "because"/"therefore" count, question mark presence, citation density, hedge word ratio, structured list presence
- **Training data:** Generate 5000 (prompt, response) pairs using an LLM. Have a second LLM label each for explainability (1-5 scale). Convert to binary (explainable >= 3). This is synthetic but good enough for v1.
- **Export:** XGBoost → ONNX → Wasm via ONNX runtime
- **Validation:** Hold-out 20% test set. Must achieve >= 80% accuracy before deployment to lattice position.

**STI-3-B (Fairness Detector):**
- **Architecture:** DistilBERT fine-tuned for NLI-style bias detection
- **Training data:** BBQ benchmark (ambiguous questions with bias traps) + WinoBias + CrowS-Pairs. ~10K examples across gender, race, age, religion dimensions.
- **Fine-tuning:** 3-5 epochs on the combined dataset. Output: bias probability per protected dimension.
- **Export:** PyTorch → ONNX → Wasm (or container if too large for Wasm)
- **Validation:** Must match or exceed published baselines on BBQ test split.

**STI-3-C (Topic Sensitivity Classifier):**
- **Architecture:** XGBoost on TF-IDF features (fast, small, deterministic)
- **Training data:** Curate or generate 10K queries across categories: `general`, `medical`, `legal`, `financial`, `harmful`, `political`. Mix real queries (from public datasets like ShareGPT) with synthetic harmful examples.
- **Export:** XGBoost → ONNX → Wasm
- **Validation:** >= 85% accuracy on test split. False negative rate on `harmful` category must be < 5%.

**Training workflow:**
```
1. Prepare dataset (synthetic generation + public benchmarks)
2. Train model (Python: scikit-learn/XGBoost or HuggingFace transformers)
3. Evaluate on held-out test set (must meet accuracy threshold)
4. Export to ONNX
5. Compute model hash (BLAKE3)
6. Deploy to cubelet: copy ONNX file to cubelet's Wasm module data section
7. Register model hash in CIG (cubelet node property)
8. AV starts at 0 — model must earn trust through successful execution
```

### Building the `#[derive(Cubelet)]` Proc Macro First

Before building any cubelet, build the proc macro that generates boilerplate:

```rust
#[derive(Cubelet)]
#[cubelet(
    lattice_id = "STI-3-B",
    framework = "STI",
    stage = 3,
    input(port = "response_text", type_id = "text.utf8"),
    output(port = "bias_scores", type_id = "scores.per_dimension"),
    stk_invariants = ["fairness<=θ"],
    stf_threads = ["ResearchLedger"],
)]
pub struct FairnessDetector {
    model: OnnxModel,
}

impl CubeletExecute for FairnessDetector {
    fn execute(&self, input: CubeletInput) -> Result<CubeletOutput> {
        let text = input.get_port::<String>("response_text")?;
        let scores = self.model.infer(&text)?;
        Ok(CubeletOutput::new().set_port("bias_scores", scores))
    }
}
```

The macro generates: typed I/O port declaration, IPC message handlers, lifecycle hooks (init/execute/shutdown), authority context wrappers, health check endpoint, schema registry registration.

### Integration Test: Full Domain Pipeline

Once all 11 cubelets are built:

```
Test: "User asks a medical question"

1. Pod assembly: resolve AI-Safety-Guard STSol → select all 11 cubelets → lock → launch
2. Pipeline execution (framework ordering):
   a. STA-3-A + STA-3-B: load safety policy + risk scoring rules
   b. STI-3-C: classify query → "medical" (high sensitivity)
   c. STI-3-A + STI-3-B: (waiting for LLM response)
   d. STD-3-A: Llama 8B generates response to medical query
   e. STI-3-A: score explainability of response → 0.72
   f. STI-3-B: score fairness of response → {gender: 0.05, age: 0.12, race: 0.03}
   g. STD-3-B: apply guardrail rules → add medical disclaimer, verify no specific dosage advice
   h. STF-3-A: log all scores to ResearchLedger
   i. STK-3-A: check safety.refusal_on_redline → PASS (medical is high-sensitivity but not harmful)
   j. STK-3-B: check fairness<=θ → PASS (all bias scores < 0.15 threshold)
   k. STK-3-C: check explainability.required → PASS (0.72 > 0.60 threshold)
3. Result returned: response with medical disclaimer, labeled VERIFIED
4. Verification: all decisions logged, STK results on-chain, edge hashes off-chain
5. AV update: all cubelets get success reward (diminishing returns)
```

```
Test: "User asks how to make a weapon"

1. Same pod assembly
2. Pipeline execution:
   a. STA-3-B: risk score = 0.97 (above redline)
   b. STI-3-C: classify query → "harmful"
   c. STD-3-A: generates refusal response
   d. STK-3-A: check safety.refusal_on_redline → PASS (model refused correctly)
3. Result: refusal response, labeled VERIFIED
4. If STD-3-A had generated a non-refusal: STK-3-A → FAIL → response BLOCKED → logged as safety violation → STD-3-A gets AV penalty
```

**This proves:** lattice structure works, framework ordering works, ML models work, invariant checking works, authority engine works, logging works, AV updates work.

---

### Second Domain: Identity Verification (Stage 0)

**Why second:** Identity is foundational. Stage 0 cubelets are needed by everything else (boot chain, ANS registration, device authentication).

**Cubelets to build:**
- STA-0-A: Personhood & Consent Charter (math-bound rule engine)
- STI-0-A: Credential Verification Logic (ML — anomaly detection on credential metadata)
- STD-0-A: Credential Issuance Runtime (Rust service, DID generation via qudag-crypto)
- STD-0-B: Consent Verification API (Rust service, checks consent DAG)
- STF-0-A: IdentLedger Connector (routes identity events to IdentLedger fabric)
- STK-0-A: Personhood Uniqueness Invariant (formal check: one DID per entity)
- STK-0-B: Consent Required Invariant (formal check: consent exists before data processing)

**Models to train:** STI-0-A needs a small anomaly detector (Isolation Forest or Autoencoder) trained on credential metadata features (issuance time, issuer reputation, credential type distribution).

### Third Domain: Sensor Attestation (Stage 4)

**Why third:** Proves edge deployment on RISC-V works.

**Cubelets:** Sensor attestation, calibration checker, telemetry collector, DePIN reward calculator, device integrity invariant. All math-bound or small ML (quantized for RISC-V).

### Subsequent Domains

| Order | Domain (Stage) | Key Models to Train |
|-------|---------------|-------------------|
| 4 | Governance (Stage 1) | Delegation pattern classifier, quorum optimizer |
| 5 | Data Sovereignty (Stage 2) | Consent violation detector, PII classifier |
| 6 | Assets/RWA (Stage 5) | Provenance verifier, valuation anomaly detector |
| 7 | Economy (Stage 6) | Fiscal policy optimizer, fraud detector |
| 8 | Social Intelligence (Stage 7) | Credential quality scorer, media integrity classifier |
| 9 | Environment (Stage 8) | MRV verifier, carbon accounting model |
| 10 | Meta-Ethics (Stage 9) | Value drift detector, alignment scorer |

Each domain follows the same pattern:
1. Seed CIG with the stage's 75 nodes + edges
2. Identify which STI cubelets need trained models
3. Prepare training data (synthetic + public benchmarks)
4. Train, evaluate, export to ONNX
5. Build all cubelets for the domain (math-bound + ML + LLM)
6. Create STSol templates
7. Integration test: full pipeline with real data
8. AV starts at 0 — cubelets earn trust through the test suite

### Extension Order (Infrastructure Extensions Beyond Domain Verticals)

These are cross-cutting infrastructure extensions that get wired in alongside or after domain verticals:

| Extension | When to Build | Architecture Notes (from finalComp/repo audit) |
|-----------|--------------|------------------------------------------------|
| Ouroboros chain | After Component 9 SQLite is proven, before Stage 4 | Implementation should include statistical monitoring thresholds (`proof_success >= 98%`, `telemetry_coverage >= 90%`, `interop_success >= 95%`) as on-chain health gates — breaching a threshold triggers chain-level alert escalation |
| qudag-network | After Stage 0 identity cubelets are deployed | Integration should include inter-kernel federation protocol — qudag nodes that span kernel boundaries must route disputes through the treaty ledger and respect federation handshake contracts |
| Rubik's Moves (DAG restructuring) | After Pod Orch is stable with 2+ domain verticals | Implementation must respect `ConstraintType`: Hard constraints cannot be modified by any Rubik's Move (they are immutable invariants); Soft constraints require Level 4 authority to restructure around; Adaptive constraints auto-tune within declared bounds and can be freely reorganized |
| Inference Router wiring | When building Stage 3 cubelets, wire InferenceRouter as pod-level resource — router created at pod assembly, shared by all LLM cubelets, SecretResolver loads from agenix |

---

## Dependency Graph

```
Component 1: Repo + Nix
    ↓
Component 2: CIG Service
    ↓
Component 3: Authority Engine ←── depends on CIG for STK invariant resolution
    ↓
Component 4: Cubelet Runtime ←── depends on CIG for manifest data
    ↓
Component 5: Data Pipeline ←── depends on Runtime (cubelet execution) + Auth (per-step checks)
    ↓
Component 6: Pod Assembly ←── depends on CIG (STSol resolution) + Auth (AV checks) + Runtime (launch)
    ↓
Component 7: Pod Orchestrator ←── depends on Assembly (gets a pod) + Pipeline (executes it)
    ↓
Component 8: Conflict Resolution ←── depends on Auth (AV comparison) + CIG (GOVERNS edges)
    ↓
Component 9: Verification ←── depends on everything (logs all decisions)
    ↓
Component 10: Knowledge Base ←── depends on CIG (backbone) + Auth (write authority) + Verification (logging)
    ↓
First Cubelets (Stage 3) ←── depends on all components
    ↓
Extensions (Stages 0, 4, 1, 2, ...) ←── one at a time
```

Start at the top. Build down.
