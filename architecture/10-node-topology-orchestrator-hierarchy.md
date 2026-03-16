# MYOS Node Topology & Orchestrator Hierarchy - Infrastructure, Hardware & LLM Assignment

## Version: 2.0 | Status: LOCKED

---

## 1. Overview

This document defines the **physical and logical topology** of a MYOS deployment: what hardware runs what software, how orchestrators are hierarchically organized, which LLM models are assigned where, and how NixOS and Unikernels are integrated across the stack.

MYOS is a **hybrid centralized-decentralized system**:

- **Centralized core**: System Orchestrator (singleton brain) + Authority Engine - runs on a single powerful server. The core server also hosts the **System KB** and the **CIG** (Neo4j graph database with 750 nodes, 9 edge types, 1500+ edges).
- **Decentralized compute**: Domain Orchestrators, Deployment Orchestrators, cubelets - distributed across zones and nodes
- **Decentralized chain**: Validators run on dedicated x86/ARM servers, chain is not centralized
- **Edge sensors**: RISC-V IoT devices for data collection, light math, and light validation

Node types map to the cubelet lattice stages: **edge RISC-V nodes** run Stage 0-4 cubelets (MYOS core, math-bound and quantized ML only), **GPU compute nodes** run Stage 3 LLM cubelets requiring container isolation and GPU inference, and the **core server** hosts the System KB + CIG (Neo4j) as the central knowledge and graph infrastructure. Stages 0-4 represent the MYOS core activation, while Stages 5-9 are Kusari-era cubelets activated as the system evolves.

---

## 2. Hardware Architecture

### 2.1 Node Types

```
NODE TYPE               HARDWARE        PURPOSE
──────────────────────────────────────────────────────────────────
Core Server             x86/ARM         System Orch + Authority Engine (co-located)
                                        Future: migrate to RISC-V when server chips mature

Domain Server           x86 or RISC-V   Domain Orchestrator (one per domain vertical)
                                        Heavy LLM domains → x86 with GPU
                                        Lighter domains → powerful RISC-V

Deployment Zone Node    Same as zone    Deployment Orchestrator (one per zone/cluster)
                                        Matches the hardware of its managed zone

Compute Node (GPU)      x86 GPU         LLM agent cubelets (require GPU for inference)
                                        Runs in Linux containers

Compute Node (General)  x86 or RISC-V   ML/DL and math cubelets
                                        Runs in Wasm inside Unikraft unikernels

Chain Validator         x86/ARM         Full Ouroboros validators (dedicated hardware)
                                        Block production, consensus participation

Edge Sensor             RISC-V          Sensor data collection
                                        Lightweight math cubelets (local processing)
                                        Light chain validation (merkle proof verification)
```

### 2.2 Hardware Assignment Map

```
┌─────────────────────────────────────────────────────────────────┐
│                    CENTRALIZED CORE                             │
│                   (x86/ARM server)                              │
│                   NixOS full                                    │
│  ┌──────────────────┐  ┌──────────────────┐                    │
│  │ System Orch (1)  │  │ Authority Engine │                    │
│  │ Paid LLM         │  │ (Haskell)        │                    │
│  └──────────────────┘  └──────────────────┘                    │
│  Future: migrate to RISC-V when server chips mature            │
└────────────────────────────┬────────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
┌────────▼────────┐ ┌───────▼───────┐ ┌─────────▼─────────┐
│ Domain Orch:    │ │ Domain Orch:  │ │ Deployment Orch   │
│ Healthcare      │ │ DeFi          │ │ Zone-1            │
│ x86 or RISC-V   │ │ x86 (GPU)     │ │ Same arch as zone │
│ Paid LLM        │ │ Paid LLM      │ │ Open-source LLM   │
│ NixOS full      │ │ NixOS full    │ │ NixOS full         │
└────────┬────────┘ └───────┬───────┘ └─────────┬─────────┘
         │                  │                   │
    ┌────▼────┐        ┌────▼────┐        ┌─────▼────┐
    │Pod Orchs│        │Pod Orchs│        │Pod Orchs │
    │Open-src │        │Open-src │        │Open-src  │
    └────┬────┘        └────┬────┘        └─────┬────┘
         │                  │                   │
┌────────▼──────────────────▼───────────────────▼──────────┐
│                    CUBELET COMPUTE                        │
│                    NixOS full                             │
│                                                          │
│  LLM Cubelets      ML Cubelets       Math Cubelets       │
│  ─────────────     ────────────      ─────────────       │
│  x86 GPU only      x86 or RISC-V    x86 or RISC-V       │
│  Linux container   Wasm+Unikernel   Wasm+Unikernel      │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                 CHAIN VALIDATORS                          │
│                 x86/ARM dedicated                         │
│                 NixOS full                                │
│  Full Ouroboros consensus, block production               │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│               EDGE RISC-V SENSORS                         │
│               NixOS minimal                               │
│                                                          │
│  Sensor agent + Light math cubelets + Light validator     │
│  (data collection + local processing + merkle proofs)    │
└──────────────────────────────────────────────────────────┘
```

### 2.3 RISC-V Strategy

RISC-V is used in two contexts:

1. **Edge sensors** - Always RISC-V. Low-power IoT devices for data collection and light computation. These are the "hands and eyes" of the system.
2. **Compute nodes** - Powerful RISC-V chips (SiFive P870, Milk-V Pioneer, Ventana Veyron) can run ML and math cubelets. Deployed where capable hardware is available.

**x86/ARM is primary today** for the core system, LLM inference (GPU ecosystem), and chain validators. **RISC-V is the future target** - as server-grade RISC-V chips mature, more of the stack migrates to RISC-V.

The architecture is **hardware-agnostic** via the HAL (Hardware Abstraction Layer, Doc 07). Upper layers never directly access hardware - they use HAL interfaces that have implementations per architecture (RISC-V, x86, ARM).

---

## 3. Orchestrator Hierarchy

### 3.1 Four-Level Hierarchy

MYOS uses a **four-level orchestrator hierarchy**. All orchestrators are **LLM-powered**.

```
LEVEL 1: SYSTEM ORCHESTRATOR (1 instance - singleton)
  │
  ├── LEVEL 2a: DOMAIN ORCHESTRATORS (one per vertical)
  │     │
  │     └── LEVEL 4: POD ORCHESTRATORS (many, ephemeral)
  │
  └── LEVEL 2b: DEPLOYMENT ORCHESTRATORS (one per zone/cluster)
        │
        └── LEVEL 4: POD ORCHESTRATORS (many, ephemeral)
```

**Domain and Deployment Orchestrators are PARALLEL PEERS** under System Orchestrator, not hierarchically nested. They coordinate laterally when needed, mediated by System Orchestrator.

### 3.2 System Orchestrator

```
SYSTEM ORCHESTRATOR:
  Count:              1 (singleton)
  Location:           Centralized core server (x86/ARM)
  Co-located with:    Authority Engine (Haskell)
  LLM:                Paid model (Claude/GPT-4 class)
  AV Floor:           700 (on all dimensions)

  Responsibilities:
    - Ultimate decision authority for the entire MYOS instance
    - Task decomposition for top-level goals
    - Creates Domain and Deployment Orchestrators
    - Creates Pod Orchestrators (can create directly, or delegate to Domain/Deployment)
    - Cross-domain and cross-zone coordination
    - Emergency response and system-wide decisions
    - Conflict resolution (Level 3 in escalation chain)
```

### 3.3 Domain Orchestrator

```
DOMAIN ORCHESTRATOR:
  Count:              One per domain vertical (e.g., Healthcare, DeFi, Carbon, AI/ML)
  Location:           Domain server (x86 or powerful RISC-V)
  LLM:                Paid model (Claude/GPT-4 class)
  AV Floor:           600 (between System=700 and Pod=400)
  Autonomy:           SEMI-AUTONOMOUS - operates independently within its domain

  Responsibilities:
    - Full autonomy within its domain vertical
    - Owns domain-specific DSL rules, compliance, and policies
    - Manages domain-relevant cubelets (knows which cubelets are domain-specialized)
    - Creates Pod Orchestrators for domain tasks
    - Enforces domain-specific regulations (HIPAA for Healthcare, KYC/AML for DeFi)
    - Can operate semi-independently from System Orchestrator

  Resources:
    - Has dedicated base resources (pre-allocated cubelet quota, compute budget)
    - Can request overflow resources from Deployment Orchestrators when needed
    - System Orchestrator mediates overflow requests if needed

  Aligns with Kusari KVMs:
    - Domain Orchs are conceptually INSPIRED BY Kusari KVMs
    - Not a 1:1 mapping - multiple domains may share a KVM, or vice versa
    - When MYOS evolves into Kusari, Domains map to KVM execution environments
```

### 3.4 Deployment Orchestrator

```
DEPLOYMENT ORCHESTRATOR:
  Count:              One per physical zone/cluster
  Location:           Same hardware architecture as its managed zone
  LLM:                Open-source model (Llama 3.1 8B, Mistral 7B) via Ollama/Groq
  AV Floor:           500 (between System=700 and Pod=400)

  Responsibilities:
    - Manages infrastructure and environment for its zone
    - Knows node capabilities (RISC-V sensor vs x86 GPU server vs chain validator)
    - Assigns pods to appropriate hardware based on requirements
    - Handles scaling, updates, health monitoring within its zone
    - Manages node lifecycle (add/remove/upgrade nodes)
    - Creates Pod Orchestrators for infrastructure tasks (updates, monitoring, scaling)
    - Environment management (staging, production, canary deployments)
```

### 3.5 Pod Orchestrator

```
POD ORCHESTRATOR:
  Count:              Many, ephemeral (created per task)
  Location:           Wherever the pod runs (any node)
  LLM:                Open-source model (Llama 3.1 8B, Phi-3) via Ollama/Groq
  AV Floor:           400
  Created by:         System, Domain, OR Deployment Orchestrator

  Responsibilities:
    - Same as defined in 05-pod-orchestrator.md
    - DAG construction, pipeline management, cubelet monitoring
    - Result synthesis, failure response, query routing (session pods)
```

### 3.6 Who Can Create Pods?

All three upper orchestrator levels can create Pod Orchestrators:

```
SYSTEM ORCHESTRATOR:    Creates pods for system-level tasks (global coordination,
                        cross-domain queries, emergency response)

DOMAIN ORCHESTRATOR:    Creates pods for domain-specific tasks (Healthcare queries,
                        DeFi transactions, Carbon tracking)

DEPLOYMENT ORCHESTRATOR: Creates pods for infrastructure tasks (node updates,
                         health checks, scaling operations, sensor data aggregation)
```

### 3.7 Domain-Deployment Coordination

Domain and Deployment Orchs are parallel peers. When they need to coordinate:

```
EXAMPLE: Healthcare Domain needs 4 GPU nodes for an ML training task

  1. Healthcare Domain Orch identifies the need for GPU resources
  2. Checks its dedicated base resources - not enough GPUs
  3. Sends overflow request:
     Option A: Direct to Deployment Orch (lateral coordination)
       Healthcare Domain Orch → Deployment Orch Zone-1: "Need 4 GPU nodes"
       Deployment Orch checks availability in its zone
       If available → allocates, responds with node assignments
       If unavailable → responds with rejection

     Option B: Via System Orch (if direct coordination fails or is ambiguous)
       Healthcare Domain Orch → System Orch: "Need 4 GPU nodes, Zone-1 can't fulfill"
       System Orch → queries all Deployment Orchs for GPU availability
       System Orch → assigns from best available zone
```

---

## 4. LLM Model Assignment

### 4.1 Cost Strategy

**Paid LLMs** (Claude/GPT-4 class via API) for high-stakes decisions:

- System Orchestrator - singleton brain, highest impact decisions
- Domain Orchestrators - autonomous domain management, compliance, policy

**Open-source LLMs** (via Ollama self-hosted or Groq API) for operational tasks:

- Deployment Orchestrators - structured infrastructure tasks
- Pod Orchestrators - ephemeral, high volume, lightweight reasoning

### 4.2 Model Assignment

```
ORCHESTRATOR           LLM TIER              MODEL EXAMPLES           HOSTING
────────────────────────────────────────────────────────────────────────────────
System Orch            Paid (top-tier)        Claude Opus, GPT-4      API call
Domain Orch            Paid (top-tier)        Claude Opus/Sonnet      API call
Deployment Orch        Open-source (medium)   Llama 3.1 8B, Mistral   Ollama/Groq
Pod Orch               Open-source (small)    Llama 3.1 8B, Phi-3     Ollama/Groq
```

### 4.3 LLM Cubelet Models

LLM cubelets (the ones in the 750 pool that handle generative tasks) also use open-source models:

```
LLM cubelets use open-source models hosted via Ollama.
Model selection is per-cubelet (configured in cubelet manifest).
Examples: Llama 3.1 70B for reasoning, Mistral 7B for fast responses,
          specialized fine-tuned models for domain tasks.
```

### 4.4 Rationale

```
The rule: if the decision has BUSINESS / SAFETY / COMPLIANCE impact → paid model.
          if the decision is OPERATIONAL / MECHANICAL → open-source model.

System Orch = decides system-wide strategy, can't afford mistakes    → paid
Domain Orch = makes compliance decisions, domain policies             → paid
Deploy Orch = structured infra tasks, scaling, node management        → open-source
Pod Orch    = ephemeral per-task coordination, high volume            → open-source
```

---

## 5. NixOS Integration

### 5.1 Strategy

**NixOS everywhere possible.** Maximizes deterministic builds, reproducibility, and unified fleet management.

```
NODE TYPE           NIXOS PROFILE       RATIONALE
──────────────────────────────────────────────────────────
Core Server         NixOS full          Declarative, reproducible, atomic rollback
Domain Server       NixOS full          Same benefits, unified management
Deployment Node     NixOS full          Consistent fleet management per zone
Compute Node        NixOS full          Reproducible cubelet + unikernel images
Chain Validator     NixOS full          Deterministic chain node builds
Edge Sensor         NixOS minimal       Lightweight but still declarative
```

### 5.2 What NixOS Replaces

| Previous Approach            | NixOS Replacement                | Benefit                                   |
| ---------------------------- | -------------------------------- | ----------------------------------------- |
| Buildroot/Yocto for rootfs   | NixOS system configuration       | Declarative, version-controlled           |
| Manual `.config` for kernel  | NixOS kernel module system       | Reproducible kernel builds                |
| Custom A/B firmware updates  | NixOS generations + rollback     | Atomic, unlimited rollback (not just A/B) |
| Manual 12-step service init  | NixOS systemd units + activation | Declarative service ordering              |
| Custom cgroups v2 setup      | NixOS systemd resource-control   | Declarative resource limits               |
| Custom build pipeline        | Nix derivations + flakes         | Bit-for-bit reproducible builds           |
| Manual dependency management | Nix flakes + lockfiles           | Pinned, hermetic dependency tree          |

### 5.3 NixOS for Haskell (haskell.nix)

The Authority Engine and Ouroboros chain core are built using **haskell.nix**:

```
HASKELL BUILD:
  Tool:       haskell.nix (IOHK's Haskell infrastructure for Nix)
  GHC:        Pinned version (e.g., GHC 9.6.x)
  Target:     x86_64-linux (Authority Engine runs on centralized core)
  Benefits:
    - Reproducible Haskell builds (exact same binary every time)
    - Pinned GHC version (no surprise compiler updates)
    - Cross-compilation support (if needed for future RISC-V migration)
    - Cacheable (Nix binary cache for CI/CD)
```

### 5.4 NixOS Minimal (Edge Sensors)

Edge RISC-V sensors run a stripped NixOS profile:

```
NIXOS MINIMAL PROFILE:
  Base:           NixOS with minimal system closure
  Kernel:         Linux 6.x (PREEMPT_RT, RISC-V)
  Init:           systemd (minimal units)
  Nix store:      Minimal (~100-200MB)
  Services:       Sensor agent, light math cubelets, light validator
  No:             Desktop packages, unnecessary daemons, debug tools

  Cross-compiled from x86 build machine using Nix cross-compilation:
    nix build .#nixosConfigurations.edge-sensor.config.system.build.sdImage \
      --system riscv64-linux
```

### 5.5 System Image as Code

The entire MYOS system is described in a single `flake.nix`:

```
flake.nix
├── nixosConfigurations
│   ├── core-server          # System Orch + Authority Engine
│   ├── domain-server        # Domain Orchestrator node
│   ├── deployment-node      # Deployment Orchestrator node
│   ├── compute-gpu          # LLM cubelet compute (x86 + GPU)
│   ├── compute-general      # ML/math cubelet compute (x86 or RISC-V)
│   ├── chain-validator      # Ouroboros validator node
│   └── edge-sensor          # RISC-V sensor node (minimal)
├── packages
│   ├── authority-engine     # Haskell Authority Engine (via haskell.nix)
│   ├── ouroboros-node       # Haskell Ouroboros consensus (via haskell.nix)
│   ├── myos-runtime         # Rust userspace services
│   ├── cubelet-images       # Wasm + unikernel cubelet images
│   └── qudag-*              # All qudag crates
├── overlays
│   ├── preempt-rt           # PREEMPT_RT kernel overlay
│   └── unikraft             # Unikraft unikernel builder
└── disko                    # Declarative disk partitioning per node type
    ├── core-server.nix      # Core server disk layout
    ├── compute-node.nix     # Compute node (large /nix/store + scratch)
    ├── chain-validator.nix  # Validator (SSD for chain storage)
    └── edge-sensor.nix      # Edge (minimal, SD card layout)
```

### 5.6 Binary Caching (Cachix)

MYOS uses a **binary cache** so that nodes never rebuild from source. All packages are built once in CI, signed, and cached.

```
BINARY CACHE STRATEGY:

  Build:
    CI pipeline builds all packages from flake.nix
    Built outputs are pushed to Cachix (or self-hosted Nix binary cache)
    Each output is signed with a cache-specific signing key

  Deploy:
    Nodes pull pre-built binaries from the cache instead of building locally
    Cache hits are verified by hash (content-addressed - tamper-proof)
    Edge sensors (RISC-V) pull cross-compiled binaries - never compile locally

  Benefits:
    - Compute nodes deploy in seconds (pull, not build)
    - Edge sensors with 256MB RAM can't compile - binary cache is mandatory
    - Identical binaries across all nodes (reproducibility guarantee)
    - Cache can be self-hosted for air-gapped deployments

  Configuration (in flake.nix):
    nix.settings = {
      substituters = [ "https://myos.cachix.org" ];
      trusted-public-keys = [ "myos.cachix.org-1:<key>" ];
    };
```

### 5.7 Declarative Disk Partitioning (Disko)

Node disks are partitioned declaratively using **Disko**, integrated into the NixOS configuration.

```
DISKO INTEGRATION:

  Purpose:
    Declarative, reproducible disk layouts per node type.
    No manual fdisk/parted. Disk layout is code, versioned in Git.

  Per node type:
    Core server:      Large /nix/store, Authority Engine state, chain storage
    Compute node:     Large /nix/store, scratch space for cubelet execution
    Chain validator:  SSD-optimized layout for block storage + merkle trees
    Edge sensor:      Minimal SD card layout (boot + small /nix/store)

  Provisioning flow:
    1. New node boots from NixOS installer
    2. Disko partitions disk according to node type config
    3. NixOS installs with the full node configuration
    4. Node pulls binaries from Cachix (no local compilation)
    5. Node is operational

  Example (compute node):
    disko.devices.disk.main = {
      type = "disk";
      device = "/dev/sda";
      content = {
        type = "gpt";
        partitions = {
          boot = { size = "512M"; type = "EF00"; content.type = "filesystem"; content.format = "vfat"; };
          nix  = { size = "100%"; content.type = "filesystem"; content.format = "ext4"; };
        };
      };
    };
```

---

## 6. Unikernel Integration (Unikraft)

### 6.1 Strategy

**Three isolation tiers. Unikernels for safety-critical. Wasm-native for lightweight. Containers for LLM agents.**

```
TIER    CUBELET TYPE        ISOLATION                     RATIONALE
─────────────────────────────────────────────────────────────────────────────
Tier 1  Safety-critical     Wasm inside Unikraft          Double isolation: Wasm sandbox + VM
        math cubelets       (wasm_unikernel)              Tiny footprint (~4-16MB per unikernel)
        Safety-critical                                   ~10ms boot time
        ML/DL cubelets                                    No OS attack surface
                                                          WASI-NN for ML inference

Tier 2  Non-critical math   Wasm sandbox only             Single isolation: Wasm sandbox
        Lightweight ML      (wasm_native)                 ~1ms boot, ~5MB memory
        Data transforms                                   No hypervisor overhead
                                                          Same Wasm portability + memory safety
                                                          Use when double isolation is overkill

Tier 3  LLM cubelets        Linux container               Needs filesystem (model files)
        Cross-language      (container)                   Needs network (API calls to LLM services)
        cubelets (PyO3)                                   Needs debugging tools
                                                          Larger memory footprint (~2GB+)
```

Tier selection is declared in the cubelet manifest and enforced by the Cubelet Launcher (see 08-runtime-infrastructure.md Section 3). Safety-critical cubelets MUST use Tier 1.

### 6.2 Unikraft Architecture

```
UNIKERNEL CUBELET:
  ┌─────────────────────────────────┐
  │  Wasm Module (cubelet code)     │  ← Application
  ├─────────────────────────────────┤
  │  Wasmtime Runtime               │  ← Wasm execution
  ├─────────────────────────────────┤
  │  Unikraft Micro-library OS      │  ← Minimal OS (scheduler, memory, IPC)
  ├─────────────────────────────────┤
  │  KVM / Firecracker / QEMU       │  ← Hypervisor
  ├─────────────────────────────────┤
  │  Host NixOS                     │  ← Host operating system
  └─────────────────────────────────┘
```

### 6.3 Unikernel Benefits vs Containers

| Property        | Unikernel (Unikraft)        | Container (crun)           |
| --------------- | --------------------------- | -------------------------- |
| Boot time       | ~10ms                       | ~500ms                     |
| Memory overhead | ~4-16MB                     | ~50-100MB                  |
| Attack surface  | Zero shell, zero filesystem | Full Linux userspace       |
| Escape risk     | VM isolation (hypervisor)   | Container escape possible  |
| Debugging       | Limited (no SSH, no shell)  | Full (exec, logs, strace)  |
| Filesystem      | None                        | Read-only rootfs + tmpfs   |
| Network         | None (for Wasm cubelets)    | Filtered network namespace |

### 6.4 Unikernel Images Built by Nix

Unikernel images are built as Nix derivations:

```
UNIKERNEL BUILD PIPELINE:
  1. Cubelet code compiled to Wasm (via Rust → wasm32-wasi)
  2. Wasmtime runtime linked with Unikraft micro-libs
  3. Unikraft image built as a Nix derivation
  4. Image cryptographically signed
  5. Image stored in Nix store (content-addressed, reproducible)
  6. At runtime: hypervisor boots image, Wasm module executes
```

---

## 7. Edge RISC-V Sensor Nodes

### 7.1 What Runs on Edge

Edge RISC-V sensor nodes run a lightweight subset of the MYOS stack:

```
EDGE NODE COMPONENTS:
  ┌──────────────────────────────────────┐
  │  Sensor Agent (Rust)                 │  ← Data collection from physical sensors
  │  Light Math Cubelets (Wasm)          │  ← Local real-time processing
  │  Light Chain Validator               │  ← Merkle proof verification (not full consensus)
  │  Network Client (qudag-network lite) │  ← Connect to MYOS network, send data upstream
  ├──────────────────────────────────────┤
  │  NixOS Minimal                       │
  ├──────────────────────────────────────┤
  │  Linux PREEMPT_RT (RISC-V)          │
  ├──────────────────────────────────────┤
  │  RISC-V Hardware                     │
  └──────────────────────────────────────┘
```

### 7.2 Edge Capabilities

```
EDGE NODE CAN:
  ✅ Collect sensor data (temperature, GPS, accelerometer, camera, etc.)
  ✅ Run lightweight math cubelets locally (threshold detection, filtering, FFT)
  ✅ Verify chain merkle proofs (light validation)
  ✅ Buffer data when disconnected (offline mode)
  ✅ Send data to MYOS compute nodes for heavy processing

EDGE NODE CANNOT:
  ❌ Run LLM inference (no GPU, insufficient memory)
  ❌ Participate in consensus (not a full validator)
  ❌ Run Pod or Domain Orchestrators
  ❌ Host pods (cubelets are standalone, not in pods)
  ❌ Run Authority Engine (calls centralized instance via RPC)
```

### 7.3 Edge Authority Model

Edge nodes do NOT run an Authority Engine locally. They call the centralized Authority Engine via RPC:

```
EDGE AUTHORITY FLOW:
  1. Edge node sensor agent wants to execute an action
  2. Edge agent calls Authority Engine via network (RPC over qudag-network)
  3. Authority Engine evaluates (centralized, same rules for everyone)
  4. Response: allow | deny | escalate
  5. Edge agent proceeds or blocks based on response

  OFFLINE FALLBACK:
    If network is unavailable:
    - Edge node uses cached authority rules (last known good)
    - Only safety-critical actions are allowed in offline mode
    - All decisions made offline are logged locally and synced when reconnected
```

---

## 8. Repo Mapping to Architecture

### 8.1 Centralized Core

| Component           | Repo(s)                            | Role                                        |
| ------------------- | ---------------------------------- | ------------------------------------------- |
| System Orchestrator | **Claude-Flow**                    | Multi-agent orchestration framework         |
| Authority Engine    | _New (Haskell)_ + **qudag-crypto** | Haskell math core + post-quantum primitives |
| Identity & Registry | **ANS**                            | Agent/device identity and discovery         |
| MCP Integration     | **qudag-mcp** + **Federated MCP**  | MCP protocol at system level                |

### 8.2 Domain Orchestrators

| Component             | Repo(s)                        | Role                             |
| --------------------- | ------------------------------ | -------------------------------- |
| Domain Orch Framework | **Claude-Flow** (extended)     | LLM orchestration per domain     |
| Domain DSL Runtime    | **SynthLang** + **PromptLang** | Domain-specific prompt languages |
| Domain Safety         | **GuardRail** + **SAFLA**      | Safety validation, guardrails    |
| Domain Compliance     | **Agentic Security**           | Security scanning, compliance    |

### 8.3 Deployment Orchestrators

| Component             | Repo(s)                    | Role                         |
| --------------------- | -------------------------- | ---------------------------- |
| Deploy Orch Framework | **Agentic DevOps**         | Infrastructure management    |
| Edge Deployment       | **Agentic Edge Functions** | Edge sensor fleet management |
| Scaling & Monitoring  | **Inflight Agentics**      | Real-time event processing   |

### 8.4 Pod Orchestrators & Cubelets

| Component           | Repo(s)                       | Role                            |
| ------------------- | ----------------------------- | ------------------------------- |
| Pod Orch Framework  | **Claude-Flow** (lightweight) | Ephemeral task orchestration    |
| ML Cubelets         | **ONNX-Agent**                | ONNX model inference cubelets   |
| Neural Net Cubelets | **ruv-fann**                  | Fast neural network (pure Rust) |
| LLM Cubelets        | **MidStream**                 | Real-time LLM streaming         |
| Swarm Cubelets      | **ruv-swarm-core**            | Swarm coordination primitives   |
| GPU Compute         | **cuda-rust-wasm**            | CUDA→Rust→Wasm transpilation    |
| Search Cubelets     | **bit-parallel-search**       | Fast string search              |
| Math Cubelets       | **sublinear**                 | High-performance solvers        |

### 8.5 Chain & Verification

| Component      | Repo(s)                         | Role                                |
| -------------- | ------------------------------- | ----------------------------------- |
| Consensus      | _New (Haskell)_ + **qudag-dag** | Ouroboros core + DAG consensus      |
| P2P Network    | **qudag-network**               | LibP2P mesh, quantum encryption     |
| Protocol       | **qudag-protocol**              | Orchestrates crypto + DAG + network |
| Crypto         | **qudag-crypto**                | ML-KEM-768, ML-DSA, BLAKE3          |
| Secure Storage | **qudag-vault-core**            | Quantum-resistant key vault         |
| Payments       | **agentic-payments**            | Ed25519 multi-agent payments        |

### 8.6 Communication

| Component            | Repo(s)                            | Role                         |
| -------------------- | ---------------------------------- | ---------------------------- |
| Federated Services   | **Federated MCP**                  | Cross-node MCP federation    |
| Agent Discovery      | **ANS**                            | Agent name resolution        |
| Voice/Conversational | **Agentic Voice** + **Omnipotent** | Voice interfaces             |
| Streaming            | **MidStream**                      | Real-time LLM data streaming |

### 8.7 Build System

| Component          | Tool                      | Role                                |
| ------------------ | ------------------------- | ----------------------------------- |
| System image build | **Nix flakes**            | Declarative, reproducible images    |
| Haskell builds     | **haskell.nix**           | Authority Engine + Ouroboros builds |
| Cubelet images     | **Nix derivations**       | Wasm + unikernel images             |
| Edge firmware      | **Nix cross-compilation** | RISC-V firmware images              |
| Unikernel images   | **Nix + Unikraft**        | Math/ML cubelet unikernels          |

### 8.8 Edge Sensors

| Component        | Repo(s)                         | Role                       |
| ---------------- | ------------------------------- | -------------------------- |
| Sensor Framework | **Agentic Edge Functions**      | Edge agent runtime         |
| Sensor AI        | **WiFi-DensePose**              | WiFi-based pose estimation |
| Navigation       | **Quantum Magnetic Navigation** | GPS-denied navigation      |

---

## 9. Configuration

```
topology_config {
    // Core
    core {
        hardware:               "x86_64"
        nixos_profile:          "core-server"
        system_orch_model:      "claude-opus"           // paid
        authority_engine:       true                     // co-located
        future_migration:       "riscv64"               // when chips mature
    }

    // Domain Orchestrators
    domains {
        default_hardware:       "x86_64"                // or "riscv64" for lighter domains
        nixos_profile:          "domain-server"
        llm_model:              "claude-sonnet"          // paid
        av_floor:               600

        instances: [
            { name: "healthcare", hardware: "x86_64", compliance: ["hipaa", "gdpr"] },
            { name: "defi",       hardware: "x86_64", compliance: ["kyc", "aml"] },
            { name: "carbon",     hardware: "riscv64" },
            { name: "ai-ml",      hardware: "x86_64"  }
        ]
    }

    // Deployment Orchestrators
    deployments {
        hardware:               "zone_match"            // matches zone hardware
        nixos_profile:          "deployment-node"
        llm_model:              "llama-3.1-8b"          // open-source via Ollama
        av_floor:               500
    }

    // Compute
    compute {
        gpu_nodes {
            hardware:           "x86_64"
            nixos_profile:      "compute-gpu"
            cubelet_isolation:  "container"             // LLM cubelets
        }
        general_nodes {
            hardware:           "x86_64 | riscv64"
            nixos_profile:      "compute-general"
            cubelet_isolation:  "unikernel"             // math/ML cubelets
        }
    }

    // Chain
    chain {
        hardware:               "x86_64"
        nixos_profile:          "chain-validator"
        dedicated:              true
    }

    // Edge
    edge {
        hardware:               "riscv64"
        nixos_profile:          "edge-sensor"            // NixOS minimal
        capabilities:           ["sensor", "light_math", "light_validator"]
        authority_mode:         "rpc"                    // calls centralized Authority Engine
    }

    // LLM Models
    llm {
        paid {
            provider:           "anthropic"              // or "openai"
            system_orch:        "claude-opus"
            domain_orch:        "claude-sonnet"
        }
        open_source {
            hosting:            "ollama"                 // or "groq"
            deployment_orch:    "llama-3.1-8b"
            pod_orch:           "llama-3.1-8b"
            llm_cubelets:       "configurable"           // per cubelet manifest
        }
    }

    // Unikernels
    unikernels {
        framework:              "unikraft"
        hypervisor:             "firecracker"            // or "qemu", "kvm"
        wasm_runtime:           "wasmtime"
        applies_to:             ["math", "ml_dl"]        // cubelet types
    }
}
```

### 9.1 NixOS Composable Module (services.myos)

The entire MYOS runtime is packaged as a **single composable NixOS module** with typed options. Instead of 7 separate nixosConfigurations with duplicated config, each node imports one module and sets its role.

```nix
# The MYOS NixOS module - one module, all node types
{ config, lib, pkgs, ... }:
{
  options.services.myos = {
    enable = lib.mkEnableOption "MYOS runtime";

    role = lib.mkOption {
      type = lib.types.enum [
        "core-server" "domain-server" "deployment-node"
        "compute-gpu" "compute-general" "chain-validator" "edge-sensor"
      ];
      description = "Node role - determines which services, isolation tiers, and profiles activate";
    };

    wasmRuntime = lib.mkOption {
      type = lib.types.enum [ "wasmtime" "wasmedge" ];
      default = "wasmtime";
      description = "Wasm runtime for cubelet execution (swappable)";
    };

    cubeletIsolation = lib.mkOption {
      type = lib.types.submodule {
        options = {
          math    = lib.mkOption { type = lib.types.enum [ "wasm_unikernel" "wasm_native" ]; default = "wasm_unikernel"; };
          ml      = lib.mkOption { type = lib.types.enum [ "wasm_unikernel" "wasm_native" ]; default = "wasm_unikernel"; };
          llm     = lib.mkOption { type = lib.types.enum [ "container" ];                     default = "container"; };
        };
      };
      default = {};
      description = "Isolation tier per cubelet type";
    };

    authority = lib.mkOption {
      type = lib.types.submodule {
        options = {
          floor = lib.mkOption { type = lib.types.int; default = 0; };
          engine = lib.mkOption { type = lib.types.bool; default = false; description = "Co-locate Authority Engine on this node"; };
        };
      };
      default = {};
    };

    llm = lib.mkOption {
      type = lib.types.submodule {
        options = {
          tier    = lib.mkOption { type = lib.types.enum [ "paid" "open-source" "none" ]; default = "none"; };
          model   = lib.mkOption { type = lib.types.str; default = ""; };
        };
      };
      default = {};
    };

    unikernel = lib.mkOption {
      type = lib.types.submodule {
        options = {
          hypervisor = lib.mkOption { type = lib.types.enum [ "firecracker" "qemu" "kvm" ]; default = "firecracker"; };
        };
      };
      default = {};
    };
  };

  # Role-based activation (simplified)
  config = lib.mkIf config.services.myos.enable {
    # Each role activates the right services, isolation, and profiles
    # No duplication - role determines everything
  };
}
```

```
NODE CONFIGURATION EXAMPLES:

  Core server:
    services.myos = {
      enable = true;
      role = "core-server";
      authority = { floor = 700; engine = true; };
      llm = { tier = "paid"; model = "claude-opus"; };
    };

  Compute (general):
    services.myos = {
      enable = true;
      role = "compute-general";
      wasmRuntime = "wasmtime";
      cubeletIsolation.math = "wasm_unikernel";
      cubeletIsolation.ml = "wasm_unikernel";
    };

  Edge sensor:
    services.myos = {
      enable = true;
      role = "edge-sensor";
      cubeletIsolation.math = "wasm_native";    # no unikernel on edge
      authority = { floor = 0; };
    };

FLAKE SIMPLIFICATION:
  Instead of 7 separate nixosConfigurations with duplicated config:
    nixosConfigurations.core-server = nixpkgs.lib.nixosSystem {
      modules = [ ./modules/myos.nix { services.myos = { enable = true; role = "core-server"; ... }; } ];
    };
    nixosConfigurations.edge-sensor = nixpkgs.lib.nixosSystem {
      modules = [ ./modules/myos.nix { services.myos = { enable = true; role = "edge-sensor"; ... }; } ];
    };
    # Same module, different options - no duplication
```

---

## 10. Invariants (Must Always Hold)

```
INV-1:  System Orchestrator is a singleton (exactly ONE instance in the entire MYOS deployment)
INV-2:  System Orchestrator and Authority Engine are co-located on the same physical server
INV-3:  Authority Engine runs centralized - edge nodes call it via RPC, never run locally
INV-4:  Domain Orchestrators are semi-autonomous - they operate independently within their domain
INV-5:  Domain and Deployment Orchestrators are parallel peers, not hierarchically nested
INV-6:  All orchestrator levels can create Pod Orchestrators (System, Domain, Deployment)
INV-7:  LLM cubelets (requiring GPU) run ONLY on x86 GPU nodes
INV-8:  Math and ML cubelets can run on x86 OR RISC-V (capability-based assignment)
INV-9:  Chain validators run on dedicated x86/ARM hardware
INV-10: Edge RISC-V sensors run NixOS minimal with sensor agent + light math + light validator
INV-11: NixOS is used on ALL node types (full or minimal profile)
INV-12: Edge RISC-V nodes run only math-bound and quantized ML cubelets (no LLMs)
INV-13: CIG (Neo4j) runs on the core server, replicated to domain servers
INV-14: Three isolation tiers: wasm_unikernel (safety-critical), wasm_native (non-critical), container (LLM/cross-lang)
INV-15: Paid LLMs for System + Domain Orchs; open-source LLMs for Deployment + Pod Orchs
INV-16: Hardware architecture is abstracted via HAL - upper layers never access hardware directly
INV-17: Edge nodes in offline mode use cached authority rules; all offline decisions are synced on reconnect
INV-18: The entire system image is reproducibly buildable from a single flake.nix
INV-19: Domain Orchs have dedicated base resources + can request overflow from Deployment Orchs
INV-20: The entire MYOS runtime is a single composable NixOS module (services.myos) - node role determines activated services
INV-21: Wasm runtime is swappable via NixOS module config - cubelet code targets WASI/WASI-NN interfaces, not a specific runtime
```

---

## 11. Interaction with Other Documents

- **Authority Model (01):** New floor hierarchy: System=700, Domain=600, Deployment=500, Pod=400, Cubelet=0. Authority Engine runs centralized on core server. Edge nodes call via RPC.
- **Conflict Resolution (02):** Escalation chain now includes Domain Orch level. Domain Orch resolves domain-specific conflicts before escalating to System Orch.
- **Pod Assembly (03):** Pods can be created by System, Domain, or Deployment Orchs. Pod Orch model assignment uses open-source LLMs.
- **Cubelet I/O (04):** Math/ML cubelets run in Wasm-inside-unikernel. Pipeline edges between unikernels use virtio/shared memory.
- **Pod Orchestrator (05):** Pod Orch LLM model updated to open-source. System Orch model is paid. Domain and Deployment Orchs are new entity types.
- **Verification & Audit (06):** Chain validators on dedicated x86/ARM. Ouroboros built via haskell.nix.
- **Boot & Trust Chain (07):** NixOS replaces Buildroot/Yocto. NixOS generations replace A/B partitioning. Different boot profiles per node type.
- **Runtime Infrastructure (08):** Unikernels for math/ML isolation. NixOS systemd replaces custom service management. CPU partitioning applies to x86/ARM core server.
- **Communication & Networking (09):** Network topology updated for centralized core + distributed compute model. Edge sensors connect as lightweight network clients.
- **Master Document (00-master.md) - Cubelet Lattice:** The 750-cubelet lattice (10 stages x 5 frameworks x 15 per cell) maps to node types: Stages 0-4 (MYOS core) run on edge and general compute nodes, Stages 5-9 (Kusari-era) activate on capable nodes as the system evolves. STSol templates (pre-defined pod templates as vertical lattice cross-sections) determine which node types are needed for a given pod.
- **CIG (Cubelet Interaction Graph):** The CIG (Neo4j, 750 nodes, 1500+ edges) runs on the core server and is replicated to domain servers. CIG FEDERATES edges define cross-device cubelet relationships that the topology must support.
- **STK Invariants & PCCSCEFVRI:** The 150 STK invariants enforced by the Authority Engine on the core server apply globally. PCCSCEFVRI serves as the ethical value anchor across all node types.
- **Knowledge Base (12-knowledge-base.md):** KB hierarchy maps to node topology: System KB resides on the core server (global knowledge), Domain KBs reside on domain servers (vertical-specific knowledge), and Pod KBs are ephemeral on compute nodes (task-scoped knowledge created and destroyed with pods). Edge sensors contribute sensor data entries to the KB with high decay rates.
