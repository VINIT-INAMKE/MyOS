# MYOS Runtime Infrastructure - Kernel, Services & Resource Management

## Version: 2.0 | Status: LOCKED

---

## 1. Overview

This document defines how the MYOS runtime operates after boot - the living system that supports cubelets, pods, and orchestrators. It covers the kernel's scheduling and memory model, the service runtime, cubelet isolation, resource management, and the watchdog system.

The three isolation tiers directly map to the three cubelet model types in the 750-cubelet lattice: **Math-bound cubelets** (~350-400, Tier 1 - Wasm+Unikernel) run with double isolation for deterministic, safety-critical computation. **ML/DL cubelets** (~200-250, Tier 1-2) run in Wasm+Unikernel for safety-critical inference or Wasm-native for non-critical models. **LLM cubelets** (~100-150, Tier 3 - containers) require filesystem and GPU access and run in Linux containers. This mapping ensures that the isolation guarantees match the trust and resource profile of each cubelet model type.

```
RUNTIME STACK (Core/Compute Nodes):

  ┌──────────────────────────────────────────────┐
  │  System / Domain / Deployment Orchestrators  │
  │  Pod Orchestrators (LLM)                     │
  │  Cubelets (Wasm+Unikernel / Containers)      │
  ├──────────────────────────────────────────────┤
  │  MYOS Rust Userspace Services                │
  │  (Scheduler, IPC, Resource Mgr, Watchdog)    │
  ├──────────────────────────────────────────────┤
  │  Authority Engine [Haskell - core server]    │
  ├──────────────────────────────────────────────┤
  │  NixOS (PREEMPT_RT kernel, x86_64/RISC-V)   │
  ├──────────────────────────────────────────────┤
  │  Hardware (x86/ARM or RISC-V)                │
  └──────────────────────────────────────────────┘

Note: Authority Engine runs ONLY on the centralized core server.
      Other nodes call it via RPC. See 10-node-topology.md.
      NixOS replaces minimal Linux. See 10-node-topology.md Section 5.
```

---

## 2. Scheduling Model - RT Priority Partitioning

### 2.1 Approach

MYOS uses Linux's SCHED_FIFO real-time scheduler with **CPU partitioning**. MYOS tasks run on dedicated RT cores. Linux housekeeping runs on a separate core.

This is a clean partition: MYOS owns its cores completely.

### 2.2 Priority Levels

```
PRIORITY MAP (SCHED_FIFO priorities, higher number = higher priority):

  P0 (99):  Safety Watchdog
             - Cannot be preempted by anything
             - Monitors all other tasks
             - Hardware watchdog pet
             - Emergency shutdown trigger

  P1 (90):  Authority Engine IPC
             - Handles authorize/update requests
             - Must respond within bounded time
             - Every action in the system depends on this

  P2 (80):  Safety Control Loops
             - Real-time sensor reading
             - Safety constraint checking
             - Actuator safety limits

  P3 (70):  AI Inference Tasks
             - Cubelet execution (Wasm and container)
             - Model inference
             - Math function execution

  P4 (60):  Agent Runtime Tasks
             - Pod Orchestrator reasoning
             - Cubelet state management
             - DAG execution coordination

  P5 (50):  Communication Tasks
             - Network send/receive
             - P2P message handling
             - Chain consensus participation

  P6 (40):  Monitoring & Logging
             - Off-chain event logging
             - Telemetry collection
             - Health reporting

  P7 (30):  Background / Housekeeping
             - Cache cleanup
             - Tier migration (hot → warm → cold)
             - Non-urgent maintenance
```

### 2.3 CPU Pinning

```
4-CORE EXAMPLE:

  Core 0 (housekeeping):    Linux kernel threads, interrupts, non-RT tasks
                            NOT used by MYOS workloads
                            IRQ affinity set to this core only

  Core 1 (safety + auth):   P0 Safety Watchdog
                            P1 Authority Engine IPC
                            P2 Safety control loops
                            Dedicated, pinned, highest priority tasks only

  Core 2 (compute):         P3 AI Inference (Wasm cubelets, containers)
                            P4 Agent Runtime (Pod Orch, cubelet management)
                            Time-shared between P3 and P4 via SCHED_FIFO

  Core 3 (comm + logging):  P5 Communication (qudag-network, chain)
                            P6 Monitoring
                            P7 Background tasks

IMPLEMENTATION:
  isolcpus=1,2,3            # exclude from Linux general scheduler
  taskset / sched_setaffinity()  # pin MYOS tasks to specific cores
  mlockall(MCL_CURRENT | MCL_FUTURE)  # lock all memory, prevent page faults
```

### 2.4 Time Budgets

Each priority level has a bounded worst-case execution time (WCET) per scheduling cycle.

```
time_budgets {
    P0_safety_watchdog:     1ms per cycle       // must complete within 1ms
    P1_authority_ipc:       5ms per request      // authority check must respond in 5ms
    P2_safety_control:      10ms per cycle       // safety loop runs at 100Hz
    P3_ai_inference:        configurable per model (10ms - 1000ms)
    P4_agent_runtime:       50ms per scheduling slot
    P5_communication:       20ms per message batch
    P6_monitoring:          100ms per collection cycle
    P7_background:          best-effort (no guarantee)
}

OVERRUN HANDLING:
  If a task exceeds its time budget:
    P0-P2:  CRITICAL - log, alert, but task continues (safety can't be killed)
    P3-P4:  Kill the task, report to Pod Orchestrator, trigger Tier 1 retry
    P5-P7:  Defer to next cycle, log the overrun
```

---

## 3. Cubelet Isolation - Three-Tier Model

### 3.1 Isolation Strategy by Cubelet Type

```
TIER 1 - DOUBLE ISOLATION:   Wasm inside Unikraft unikernel
  Math cubelets (safety-critical, deterministic)
  ML/DL cubelets (safety-critical inference)

TIER 2 - WASM NATIVE:        Wasm sandbox only (no unikernel)
  Non-safety-critical math (data transforms, filtering, aggregation)
  Lightweight ML (small models, feature extraction)

TIER 3 - CONTAINER:          Linux container (crun)
  LLM agent cubelets (need filesystem, network, GPU)
  Cross-language cubelets (PyO3 Python, need runtime)
```

The **Cubelet Launcher** is a single abstraction that dispatches to the right isolation backend:

```
CUBELET LAUNCHER:

  CubeletLauncher.launch(manifest) →
      match manifest.isolation_tier:
          "wasm_unikernel"  → Firecracker → Unikraft + Wasm runtime
          "wasm_native"     → Wasm runtime directly on host NixOS
          "container"       → crun Linux container

  Tier selection is declared in the cubelet manifest:
      safety_critical: true  → wasm_unikernel (double isolation, mandatory)
      safety_critical: false, needs_os: false → wasm_native (lightweight)
      needs_os: true → container (filesystem, network, GPU)

  All three tiers share:
      - Same typed I/O contract (cubelet_io_declaration)
      - Same control message protocol
      - Same authority context in IPC
      - Same resource budget enforcement
```

**Why three tiers instead of two?**

```
Tier 2 (wasm_native) fills the gap between double isolation and containers:
  - Boot time:   ~1ms   (vs ~12ms unikernel, ~500ms container)
  - Memory:      ~5MB   (vs ~20MB unikernel, ~100MB container)
  - Isolation:   Wasm sandbox (memory safe, no syscalls, portable)
  - Missing:     No VM boundary (hypervisor layer skipped)
  - Use for:     High-volume, non-safety-critical cubelets where
                 the ~12ms unikernel boot overhead matters at scale
```

See **10-node-topology-orchestrator-hierarchy.md** Section 6 for full unikernel architecture and rationale.

### 3.2 Wasm Sandboxes (Math + ML/DL Cubelets)

Math functions and ML/DL models run as **WebAssembly modules** in a Wasm runtime.

```
WASM RUNTIME: Wasmtime (default) - swappable via NixOS module config

WHY WASM FOR MATH/ML:
  - Deterministic execution (same input → same output, guaranteed)
  - Memory safety (no buffer overflows, no use-after-free)
  - Sandboxed by default (no filesystem, no network, no syscalls)
  - Near-native performance (AOT compiled at boot)
  - Portable (same Wasm module runs on RISC-V, ARM, x86)
  - Lightweight (microsecond startup, kilobytes of overhead)
  - Perfect for pure functions (no side effects)

ML INFERENCE INTERFACE - WASI-NN:
  ML cubelets use WASI-NN as the standard inference API.
  WASI-NN is a WASI (WebAssembly System Interface) proposal for
  neural network inference, supported by both Wasmtime and WasmEdge.

  Benefits:
    - Standard API: cubelet code compiles against wasi-nn, not a specific runtime
    - Runtime swappable: Wasmtime's wasi-nn crate or any future implementation
    - Backend agnostic: WASI-NN can dispatch to ONNX, OpenVINO, or TensorFlow Lite
    - No vendor lock-in: ML cubelets are portable across Wasm runtimes

  ML cubelet code:
    use wasi_nn;
    let graph = wasi_nn::load(&model_bytes, wasi_nn::GRAPH_ENCODING_ONNX)?;
    let context = wasi_nn::init_execution_context(graph)?;
    wasi_nn::set_input(context, 0, &input_tensor)?;
    wasi_nn::compute(context)?;
    let output = wasi_nn::get_output(context, 0)?;

WASM CUBELET CAPABILITIES:
  ✅ Read input from data pipeline
  ✅ Write output to data pipeline
  ✅ Send control messages (via host function callback)
  ✅ Read its own configuration
  ❌ No filesystem access
  ❌ No network access
  ❌ No spawning processes
  ❌ No accessing other cubelets' memory
  ❌ No modifying its own Wasm module

RESOURCE LIMITS:
  memory:       Fixed per cubelet (e.g., 64MB max)
  execution:    Time budget enforced by the Wasm runtime (fuel metering)
  stack:        Fixed stack size (e.g., 1MB)
```

### 3.3 Linux Containers (LLM Agent Cubelets)

LLM-infused agents run in **lightweight Linux containers** because they need more system capabilities.

```
CONTAINER RUNTIME: crun (lightweight, OCI-compliant)

WHY CONTAINERS FOR LLM AGENTS:
  - LLM inference requires larger memory (model weights)
  - LLM agents may need limited filesystem access (model files)
  - LLM agents may need controlled network access (for API calls to LLM services)
  - Container cgroups provide resource limits
  - Namespace isolation prevents interference between agents

CONTAINER CUBELET CAPABILITIES:
  ✅ Read input from data pipeline
  ✅ Write output to data pipeline
  ✅ Send control messages to Pod Orchestrator
  ✅ Read model files (read-only mount)
  ✅ Network access (filtered - only to authorized LLM API endpoints)
  ✅ Limited filesystem (tmpfs for scratch, read-only for models)
  ❌ No access to host filesystem
  ❌ No access to other containers
  ❌ No access to kernel interfaces
  ❌ No privileged operations

SECURITY:
  seccomp:      Restrictive syscall filter (allowlist of safe syscalls)
  capabilities: All dropped except minimum required (no CAP_SYS_ADMIN)
  read-only rootfs: Container filesystem is read-only
  no-new-privileges: Cannot gain additional privileges after start
  user namespace: Runs as unprivileged user inside container

RESOURCE LIMITS (cgroups v2):
  memory:       Fixed per cubelet (e.g., 2GB for large LLM)
  cpu:          CPU time quota (e.g., 50% of one core)
  io:           I/O bandwidth limit
  pids:         Max process count (prevent fork bombs)
```

### 3.4 Orchestrator Isolation

Pod Orchestrators and the System Orchestrator are LLM-based and run in containers with slightly higher privileges than regular LLM cubelets.

```
ORCHESTRATOR CONTAINER:
  Same base as LLM cubelet containers, but with:
    - Higher memory limit (orchestrators need more context)
    - Higher CPU quota (LLM reasoning takes more compute)
    - Access to the IPC channel (for Authority Engine communication)
    - Access to the cubelet registry (read-only)
    - Access to the Pod Orch registry (read-write for System Orch)

  Still restricted:
    - No host filesystem access
    - No raw network access (only via qudag-network APIs)
    - seccomp + capability drops still applied
```

### 3.5 Container Network Policy Enforcement

Tier 3 containers (LLM cubelets, orchestrators, cross-language cubelets) require network access for API calls (Ollama, external LLM services). MYOS enforces **L7 HTTP-level policy** on all container egress, adopting the pattern from NVIDIA OpenShell's policy engine.

```
CONTAINER NETWORK POLICY (per-cubelet, declarative):

  Each container cubelet's manifest declares allowed network endpoints:

  network_policy:
    allowed_endpoints:
      - binary: "*"                    # applies to all processes in container
        endpoints:
          - host: "localhost"
            port: 11434               # Ollama API
            methods: [POST]
            paths: ["/api/generate", "/api/chat", "/api/embeddings"]
          - host: "localhost"
            port: 7687                # Neo4j (for CIG queries)
            methods: [POST]
            paths: ["*"]
    default: deny                     # all other traffic blocked

  ENFORCEMENT:
    - L7 proxy intercepts all HTTP/HTTPS traffic from the container
    - Each request checked against: binary → endpoint → method → path
    - Non-matching requests are DENIED and logged
    - DNS resolution restricted to allowed hostnames only

  HOT-RELOAD:
    - Network policies can be updated WITHOUT restarting the container
    - Pod Orchestrator can tighten or relax endpoint access mid-execution
    - Hot-reload signal triggers policy re-evaluation on next request
    - Policy changes are authority-gated (requires sufficient AV on policy dimension)
    - All policy changes logged to verification chain
```

### 3.6 Privacy Router (LLM API Proxy)

All LLM cubelet API calls route through a **Privacy Router** — a credential-aware proxy that prevents leaking secrets into LLM context. Adopted from NVIDIA OpenShell's inference routing pattern.

```
PRIVACY ROUTER:

  PURPOSE:
    LLM cubelets call external APIs (Ollama, cloud LLM services).
    The cubelet code must NEVER see or handle API credentials directly.
    The Privacy Router intercepts outgoing LLM requests and:
      1. Strips any caller credentials from the request
      2. Injects the correct backend credentials (API keys, tokens)
      3. Routes to the correct backend endpoint
      4. Strips any credential-bearing headers from the response

  ARCHITECTURE:
    Container cubelet → HTTP request → Privacy Router (sidecar) → Backend API
                                            │
                                     Credentials injected
                                     from agenix vault
                                     (never in container env)

  SECURITY PROPERTIES:
    - API keys are NEVER mounted in the container filesystem
    - API keys are NEVER passed as environment variables to cubelets
    - API keys exist only in the Privacy Router process memory
    - Privacy Router runs as a separate process with its own agenix secret
    - If a cubelet is compromised, it cannot extract API credentials
    - All requests/responses are logged (with credentials redacted)

  CONFIGURATION:
    privacy_router:
      backends:
        ollama:
          upstream: "http://localhost:11434"
          auth: none                         # local Ollama needs no auth
        openai:
          upstream: "https://api.openai.com"
          auth:
            type: bearer
            secret: "agenix://openai-api-key"  # resolved from agenix vault
        anthropic:
          upstream: "https://api.anthropic.com"
          auth:
            type: header
            header_name: "x-api-key"
            secret: "agenix://anthropic-api-key"
```

**Implementation note:** When implementing the Privacy Router and L7 network policies, study the actual Rust source code in NVIDIA OpenShell (https://github.com/NVIDIA/OpenShell, Apache 2.0). The policy engine (`src/policy/`) and gateway routing code provide proven patterns for HTTP interception, Landlock integration, and hot-reloadable policy evaluation. Do not reimagine these from scratch.

---

## 4. Inter-Process Communication (IPC)

### 4.1 IPC Model

All communication between MYOS components uses **message passing over shared memory regions** with access control.

```
IPC CHANNELS:

  Cubelet → Pod Orchestrator:
    Channel type:  Shared memory ring buffer
    Direction:     Bidirectional
    Messages:      Control messages (status, alerts, queries)
    Access:        Cubelet can write, Pod Orch can read/write

  Pod Orchestrator → System Orchestrator:
    Channel type:  Shared memory ring buffer
    Direction:     Bidirectional
    Messages:      Pod status, escalation, cross-pod queries
    Access:        Pod Orch can write, System Orch can read/write

  Any component → Authority Engine:
    Channel type:  Unix domain socket (synchronous request/response)
    Direction:     Request/response
    Messages:      authorize(), update_authority(), get_authority()
    Access:        Any MYOS component can send requests
    Latency:       <5ms response time guaranteed
    Authentication:
      Kernel-level:  SO_PEERCRED - kernel verifies connecting process's UID/GID
                     before the Authority Engine processes the request.
                     Only processes running as MYOS service UIDs are accepted.
      Socket perms:  Socket file owned by myos-authority group (mode 0660).
                     Only MYOS service processes are in this group.
      No network:    UDS only - no TCP, no network stack, no remote exploitation.
                     Edge nodes use qudag-network RPC (encrypted), not local UDS.

  Data Pipeline edges:
    Channel type:  Shared memory with typed buffers
    Direction:     Unidirectional (producer → consumer)
    Messages:      Pipeline data (typed, schema-validated)
    Access:        Producer can write, consumer can read
    Zero-copy:     Data is written once, consumer reads from same memory
```

### 4.2 Authority Context in IPC

Every IPC message carries an **authority context**:

```
ipc_message {
    header: {
        sender:             EntityId
        recipient:          EntityId
        message_type:       String
        priority:           int (maps to scheduling priority)
        timestamp:          Timestamp
        authority_context: {
            sender_av:      AuthorityVector     // sender's current AV
            required_av:    AuthorityVector      // minimum AV needed to process this message
        }
    }
    payload:    Bytes
}
```

The recipient checks the authority context before processing. If the sender's AV is insufficient, the message is rejected and a violation event is logged.

### 4.3 IPC Buffer Management

```
All IPC buffers are pre-allocated at boot.
No dynamic allocation during runtime.

ring_buffer_config {
    cubelet_to_pod_orch:    4KB per cubelet (1024 messages × 4 bytes avg)
    pod_orch_to_system:     16KB per pod
    data_pipeline_edge:     configurable per edge (default 1MB)
    authority_engine:       64KB (request/response queue)
}

OVERFLOW:
  If a ring buffer is full:
    Control messages: drop oldest, log the drop
    Pipeline data: block producer until consumer catches up (backpressure)
    Authority requests: never drop - queue until processed (bounded by time budget)
```

---

## 5. Resource Management

### 5.1 Resource Budgets

Every entity in the system operates within a resource budget assigned at creation time.

```
resource_budget {
    entity_id:      EntityId
    cpu_quota:      float           // fraction of a core (e.g., 0.5 = half a core)
    memory_limit:   bytes           // hard limit, kill on exceed
    io_bandwidth:   bytes_per_sec   // I/O rate limit
    network_quota:  bytes_per_sec   // network bandwidth limit (containers only)
    time_budget:    Duration        // max execution time per task
}
```

### 5.2 Resource Enforcement

```
ENFORCEMENT MECHANISMS:

  CPU:      cgroups v2 cpu.max (for containers)
            Wasm fuel metering (for Wasm cubelets)
            SCHED_FIFO priority (for RT tasks)

  Memory:   cgroups v2 memory.max (for containers)
            Wasm linear memory limit (for Wasm cubelets)
            OOM kill on exceed (container is killed, not the system)

  I/O:      cgroups v2 io.max (for containers)
            Not applicable for Wasm (no I/O access)

  Network:  Network namespace + tc (traffic control) for containers
            Not applicable for Wasm (no network access)

  Time:     Wasm fuel metering (instruction count → time estimate)
            Container timeout (watchdog kills container on overrun)
```

### 5.3 Resource Accounting

Resource usage is tracked per entity and reported off-chain.

```
resource_usage_report {
    entity_id:          EntityId
    reporting_period:   (Timestamp, Timestamp)
    cpu_used:           Duration            // actual CPU time consumed
    memory_peak:        bytes               // peak memory usage
    memory_avg:         bytes               // average memory usage
    io_read:            bytes               // total I/O read
    io_write:           bytes               // total I/O write
    network_sent:       bytes               // total network sent
    network_received:   bytes               // total network received
    overruns:           int                 // number of time budget overruns
    oom_events:         int                 // number of out-of-memory events
}
```

Resource reports are used by the System Orchestrator for pod assembly decisions (overloaded cubelets get lower load-balance scores).

---

## 6. Watchdog System

### 6.1 Multi-Level Watchdog

```
LEVEL 1: Hardware Watchdog
  - Hardware timer on the SoC
  - Must be "pet" by the P0 Safety Watchdog task every N milliseconds
  - If not pet → hardware reset (full device reboot)
  - Catches: kernel hang, safety task crash, total system failure

LEVEL 2: Software Safety Watchdog (P0 priority)
  - Monitors all other MYOS services
  - Checks:
    a. Authority Engine is responsive (heartbeat)
    b. System Orchestrator is responsive (heartbeat)
    c. All active Pod Orchestrators are responsive
    d. No cubelet has exceeded its time budget by >2x
    e. Safety control loops are running on schedule
  - If any check fails:
    a. Log the failure (off-chain, immediately)
    b. Attempt recovery (restart the failed service)
    c. If recovery fails → safety shutdown

LEVEL 3: Pod-Level Watchdog (per Pod Orchestrator)
  - Pod Orchestrator monitors its cubelets
  - Catches: cubelet crash, hang, output timeout
  - Response: tiered failure response (retry → quarantine → restructure → escalate)
```

### 6.2 Safety Shutdown

```
safety_shutdown_sequence:
  1. Safety Watchdog detects unrecoverable failure
  2. All actuators commanded to safe state (stop, neutral position)
  3. All active pods are emergency-dissolved
  4. Safety event written on-chain (if chain is available)
  5. Device enters safe mode:
     - No task execution
     - Network remains active (for remote diagnosis)
     - Chain node remains active (for event logging)
     - Human notification sent
  6. Await human intervention to restart normal operation
```

---

## 7. Service Lifecycle Management

### 7.1 Services

MYOS runs several long-lived services managed as **NixOS systemd units** with secrets provisioned via **agenix**.

```
MYOS SERVICES:

  myos-authority-engine     [Haskell]   P1 priority
    The Authority Engine process
    Communicates via Unix domain socket IPC
    Restarts: automatic (critical service)
    Secrets: Authority config signing key (via agenix)

  myos-system-orchestrator  [Container] P4 priority
    System Orchestrator LLM
    Manages pod assembly, task decomposition
    Restarts: automatic (critical service)
    Secrets: LLM API key (Claude/GPT-4) (via agenix)

  myos-chain-node           [Rust]      P5 priority
    Ouroboros consensus + block storage + P2P
    Restarts: automatic (chain must stay live)
    Secrets: Validator signing key, chain identity key (via agenix)

  myos-network              [Rust]      P5 priority
    qudag-network P2P layer + ANS registration
    Restarts: automatic
    Secrets: Device identity key, P2P encryption key (via agenix)

  myos-federated-mcp        [Rust]      P5 priority
    Federated MCP runtime for cross-device coordination
    Restarts: automatic

  myos-offchain-db           [Rust]      P6 priority
    Event log + content-addressed store + time-series index
    Restarts: automatic

  myos-watchdog             [Rust]      P0 priority
    Safety watchdog (highest priority)
    Restarts: N/A (if this dies, hardware watchdog reboots)

  myos-scheduler            [Rust]      P1 priority
    Userspace scheduler coordinator
    Manages CPU pinning, priority assignment, time budgets
    Restarts: automatic
```

### 7.1.1 Secret Provisioning (Agenix)

All MYOS service secrets are managed via **agenix** - secrets are encrypted at rest using SSH host keys and decrypted only at runtime into `/run/agenix/`.

```
AGENIX SECRET MODEL:

  At rest:
    Secrets stored as .age files in the Git repo (encrypted, safe to commit)
    Encrypted using SSH host keys (ed25519) of authorized nodes
    No plaintext secrets anywhere on disk or in configuration

  At runtime:
    NixOS activation decrypts secrets to /run/agenix/<secret-name>
    Secrets exist only in tmpfs (RAM) - never written to persistent storage
    Each secret has owner:group matching the service that needs it
    Services reference secrets by path: config.age.secrets.<name>.path

  Secret types:
    authority-config-signing-key    → owner: myos-authority
    llm-api-key                     → owner: myos-orchestrator
    chain-validator-key             → owner: myos-chain
    device-identity-key             → owner: myos-network
    cachix-signing-key              → owner: root (build-time only)
    vector-db-credentials           → owner: myos-kb

  NixOS integration:
    age.identityPaths = [ "/etc/ssh/ssh_host_ed25519_key" ];
    age.secrets.llm-api-key = {
      file = ./secrets/llm-api-key.age;
      owner = "myos-orchestrator";
      group = "myos-orchestrator";
    };
    # Referenced in service config:
    environment.LLM_API_KEY_FILE = config.age.secrets.llm-api-key.path;

  Security properties:
    - Secrets never in Nix store (not content-addressed, not cached)
    - Secrets deleted on reboot (tmpfs)
    - Only the owning service UID can read its secrets
    - Rotation: re-encrypt .age files with new keys, rebuild node
```

### 7.2 Service Dependencies

```
DEPENDENCY GRAPH:

  myos-watchdog          → none (starts first, no dependencies)
  myos-scheduler         → myos-watchdog
  myos-authority-engine  → myos-scheduler
  myos-offchain-db       → myos-scheduler
  myos-chain-node        → myos-offchain-db
  myos-network           → myos-chain-node
  myos-federated-mcp     → myos-network
  myos-system-orchestrator → myos-authority-engine + myos-network + myos-chain-node
```

### 7.3 Service Health Monitoring

```
health_check (per service, every 1 second):
  1. Heartbeat: is the service process alive?
  2. Responsiveness: does it respond to a health probe within 100ms?
  3. Resource usage: is it within its resource budget?
  4. Error rate: has it exceeded error threshold in the last minute?

Health states:
  healthy:    all checks pass
  degraded:   alive but slow or high error rate
  unhealthy:  unresponsive or over resource limits
  dead:       process not running

State transitions:
  healthy → degraded:   alert logged, continue monitoring
  degraded → unhealthy: attempt restart
  unhealthy → dead:     auto-restart, log on-chain if critical service
  dead → healthy:       restart succeeded, resume normal operation
  dead → dead:          restart failed, escalate (safety shutdown if critical)
```

---

## 8. STK+ Enhancement Threads

The STK kernel evolves beyond basic invariant checking into **STK+** — additional runtime threads that provide resilience, observability, and adaptive governance.

### 8.1 Resilience Layer
- Self-healing logic: if a cubelet consistently fails (AV drops below threshold), the system automatically quarantines it and attempts pod restructure
- Auto-recovery: if Authority Engine restarts from crash, cached state is restored from the last on-chain checkpoint
- Circuit breaker pattern: if a fabric thread exceeds error rate threshold (>5% over 5 minutes), cross-device queries on that thread are suspended until health recovers

### 8.2 Observability and Explainability
- Every authority decision includes a human-readable explanation generated from the FailContext (observed value, threshold, invariant family)
- XAI composer: an STD cubelet that takes ProofPerl results and generates natural language explanations for audit dashboards
- BLAKE3 hash of all logs committed as merkle root every 5 minutes for tamper detection
- Archive retention: 180 days for hot/warm, configurable years for cold

### 8.3 Adaptive Governance
- Hot-swappable network policies for container cubelets (from OpenShell L7 pattern, see Section 3.5)
- STK invariant thresholds can be adjusted via Rubik's Move (Lock.Invariants for safety-critical, Soft constraints for operational)
- Policy bundles flow from PerlFrame → ProofPerl; telemetry flows from ProofPerl → PerlFrame for adaptive threshold tuning
- Invariant ConstraintType: Hard (never changes), Soft (Rubik's Move required), Adaptive (auto-tuned based on AV history)

### 8.4 Inter-Kernel Federation
- Multiple MYOS instances can federate their STK kernels via treaty verification
- Remote attestation handshake: kernels exchange signed attestation of identity, code hash, and policy config
- Treaty ledger: a shared fabric thread where federated kernels publish proofs of their integrity
- M-of-N key authorization for kernel upgrades across federated instances
- Cross-kernel invariant verification: if Instance A references data from Instance B, Instance B's STK proves the data satisfies B's invariants before A accepts it

### 8.5 Quantum-Resilient Crypto Switching
- The STK supports hot-switching cryptographic algorithms without system restart
- Parallel old/new crypto during transition periods (dual-sign, verify both)
- Survival analysis targeting 2035 for post-quantum readiness (ML-KEM-768, ML-DSA via qudag-crypto)
- Fallback: if ZK verifier fails >2% over 24 hours, halt all Rubik's Move rollouts until resolved

---

## 9. Configuration

```
runtime_config {
    // Scheduling
    scheduler {
        rt_policy:              "SCHED_FIFO"
        rt_cores:               [1, 2, 3]
        housekeeping_cores:     [0]
        irq_affinity:           [0]         // all IRQs to housekeeping core
    }

    // Cubelet isolation (three-tier)
    isolation {
        wasm_runtime:           "wasmtime"  // swappable via NixOS module
        wasm_aot_compile:       true        // AOT compile at boot
        wasi_nn_backend:        "onnx"      // WASI-NN inference backend
        container_runtime:      "crun"
        container_seccomp:      "strict"    // restrictive syscall filter
        default_wasm_memory:    "64MB"
        default_container_memory: "2GB"
        tiers: {
            wasm_unikernel:     { hypervisor: "firecracker", for: ["safety_critical_math", "safety_critical_ml"] }
            wasm_native:        { direct: true, for: ["non_critical_math", "lightweight_ml", "data_transforms"] }
            container:          { runtime: "crun", for: ["llm_agents", "cross_language", "orchestrators"] }
        }
    }

    // IPC
    ipc {
        cubelet_buffer_size:    "4KB"
        pod_orch_buffer_size:   "16KB"
        pipeline_edge_default:  "1MB"
        authority_queue_size:   "64KB"
    }

    // Watchdog
    watchdog {
        hardware_timeout_ms:    5000        // 5 seconds
        software_check_interval_ms: 1000   // 1 second
        service_health_interval_ms: 1000
        max_restart_attempts:   3
        safety_shutdown_on_critical_failure: true
    }

    // Resource defaults
    resources {
        default_cpu_quota:      0.25        // quarter of a core
        default_time_budget_ms: 1000        // 1 second per task
        overrun_multiplier:     2.0         // kill at 2x time budget
    }

    // Service management
    services {
        auto_restart:           true
        restart_delay_ms:       1000
        max_restart_failures:   3           // after 3 failed restarts → escalate
    }
}
```

---

## 10. Invariants (Must Always Hold)

```
INV-1:  RT cores are exclusively owned by MYOS (isolcpus, no Linux scheduler interference)
INV-2:  Safety watchdog runs at highest priority (P0) and cannot be preempted
INV-3:  No dynamic memory allocation in RT execution paths
INV-4:  Wasm cubelets have no filesystem, network, or syscall access
INV-5:  Container cubelets have restrictive seccomp + dropped capabilities
INV-6:  Every IPC message carries authority context
INV-7:  Pipeline data uses zero-copy shared memory (write once, read from same memory)
INV-8:  Resource budgets are enforced by kernel mechanisms (cgroups, Wasm fuel), not application code
INV-9:  Hardware watchdog reboots the device if software watchdog dies
INV-10: Critical services (Authority Engine, System Orch) auto-restart; if restart fails → safety shutdown
INV-11: Service dependencies are respected (no service starts before its dependencies)
INV-12: All cubelets execute within their time budget or are killed
INV-13: Math-bound and ML/DL cubelets (STK, STF, STI frameworks) run in Tier 1-2 isolation
INV-14: LLM cubelets run in Tier 3 containers - never in Wasm+Unikernel
INV-15: Orchestrator containers have higher resource limits but same security restrictions as cubelet containers
INV-16: Cubelet Launcher is the single entry point for all cubelet lifecycle - dispatch is by isolation tier, not by runtime
INV-17: ML cubelets use WASI-NN as the standard inference interface - cubelet code never imports a specific runtime
INV-18: Safety-critical cubelets MUST use wasm_unikernel tier (double isolation) - wasm_native is only for non-safety-critical
INV-19: All container cubelet network egress passes through L7 policy enforcement - no direct outbound connections
INV-20: Container network policies are per-cubelet and enforce endpoint + HTTP method + path restrictions
INV-21: Network policy hot-reload requires authority check (policy dimension AV) before applying
INV-22: All LLM API calls route through the Privacy Router - cubelets never handle API credentials
INV-23: API credentials exist only in Privacy Router process memory, sourced from agenix vault - never in container filesystem or environment
INV-24: STK+ resilience thread monitors cubelet AV trajectories for anomalous drops
INV-25: Observability merkle root committed every 5 minutes (configurable)
INV-26: Inter-kernel federation requires mutual remote attestation before any data exchange
INV-27: Crypto algorithm switching uses dual-sign period — both old and new must verify
```

---

## 11. Interaction with Other Documents

- **Boot & Trust Chain (07):** Runtime infrastructure is initialized during boot Steps 2-10. CPU layout, memory layout, and service startup order defined there.
- **Authority Model (01):** Authority Engine runs as a dedicated Haskell process on the safety core (Core 1). IPC latency <5ms.
- **Pod Assembly (03):** Cubelet isolation (Wasm vs container) determines how cubelets are launched when pods are assembled.
- **Cubelet I/O (04):** Data pipeline uses shared memory IPC with typed buffers. Control messages use ring buffer IPC.
- **Pod Orchestrator (05):** Pod Orch runs in a container with elevated resource limits. Communicates with cubelets via IPC.
- **Verification & Audit (06):** Off-chain DB is a service managed by the service lifecycle system. Watchdog events are logged on-chain.
- **Master Document (00-master.md) - Cubelet Lattice:** The 750 cubelets (10 stages x 5 frameworks x 15 per cell) map to isolation tiers by model type: Math-bound (~350-400) to Tier 1, ML/DL (~200-250) to Tier 1-2, LLM (~100-150) to Tier 3. The lattice structure determines which isolation tier is enforced for each cubelet.
- **CIG (Cubelet Interaction Graph):** The CIG's 9 edge types define inter-cubelet relationships that the runtime must respect when enforcing isolation boundaries. CONTAINS and FEDERATES edges determine which cubelets can share isolation contexts.
- **STF Fabric Threads:** The 11+ named ledgers influence resource allocation at runtime - fabric threads carrying safety-critical data (e.g., EthosLedger) require Tier 1 isolation for all cubelets in their path.
- **Existing Repos:** Wasm runtime hosts cubelets compiled from ONNX-Agent and ruv-fann. Container runtime hosts LLM agents.
- **Knowledge Base (12-knowledge-base.md):** Vector DB and Knowledge Graph DB run as NixOS-managed services on compute nodes. These services support KB queries, semantic search, and knowledge graph traversal for the hierarchical KB (System, Domain, Pod levels).
