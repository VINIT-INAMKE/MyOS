# MYOS Repository Mapping - Existing Repos to Architecture Layers

## Version: 2.0 | Status: LOCKED

---

## 1. Overview

This document maps every usable repository from the [ruvnet/agenticsorg portfolio](../readme.md) to the MYOS architecture. Each repo is assigned to a specific layer, component, and role. Repos not applicable to MYOS are listed separately with rationale.

Repos are mapped to the **Rubik's Lattice** (10 stages × 5 frameworks × 15 cubelets per cell = 750 total). Each repo's cubelets are assigned lattice positions based on their function. Math-bound cubelets (STK/STF frameworks) run in Tier 1 Wasm+Unikernel isolation. ML/DL cubelets (STI framework) run in Tier 1-2 Wasm. LLM cubelets (some STA/STD) run in Tier 3 containers.

**Total repos evaluated:** 70+
**Repos mapped to MYOS:** 38
**Repos not used:** 32+

---

## 2. Layer Mapping Summary

```
ARCHITECTURE LAYER                    REPOS USED    KEY REPOS
─────────────────────────────────────────────────────────────────────
Centralized Core (System Orch + Auth)     5          Claude-Flow, ANS, qudag-crypto
Domain Orchestrators                      4          SynthLang, PromptLang, GuardRail, SAFLA
Deployment Orchestrators                  3          Agentic DevOps, Edge Functions, Inflight
Pod Orchs & Cubelets                      9          ONNX-Agent, ruv-fann, MidStream, cuda-rust-wasm
Chain & Verification                      6          qudag-dag, qudag-network, qudag-protocol
Communication & Networking                4          Federated MCP, ANS, Agentic Voice
Build System (NixOS)                      0          (new tooling - Nix, haskell.nix, Unikraft)
Edge Sensors                              3          Edge Functions, WiFi-DensePose, Quantum Nav
Cross-Cutting / Utilities                 4          SPARC, Agentic Security, claude-code-flow
```

---

## 3. Detailed Mapping

### 3.1 Centralized Core - System Orchestrator + Authority Engine

| Component                     | Repo                         | GitHub                                               | Role in MYOS                                                                                                                               | Integration Notes                                                                                                      |
| ----------------------------- | ---------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| System Orchestrator framework | **Claude-Flow**              | [Link](https://github.com/ruvnet/claude-flow)        | Multi-agent orchestration engine. Becomes the foundation for the singleton System Orchestrator.                                            | Adapt swarm orchestration to single-brain topology. Use coordination primitives for Domain/Deployment Orch management. |
| Agent identity & discovery    | **ANS (Agent Name Service)** | [Link](https://github.com/ruvnet/Agent-Name-Service) | Secure registry for all MYOS entities - cubelets, orchestrators, devices, edge sensors. OWASP-based identity protocol.                     | Core identity layer. Every entity registers in ANS at boot. Cross-device discovery uses ANS lookups.                   |
| Post-quantum cryptography     | **qudag-crypto**             | [Link](https://crates.io/crates/qudag-crypto)        | ML-KEM-768 (key exchange), ML-DSA (signatures), BLAKE3 (hashing). Used by Authority Engine for signing decisions, boot chain verification. | Rust crate. Integrated into Authority Engine (via FFI from Haskell), boot chain, and all authentication flows.         |
| Key management                | **qudag-vault-core**         | [Link](https://crates.io/crates/qudag-vault-core)    | Quantum-resistant key vault. Stores device identity keys, firmware signing keys, session keys.                                             | Rust crate. HAL uses vault for secure storage operations. Keys encrypted at rest using SoC hardware key.               |
| MCP protocol (system level)   | **qudag-mcp**                | [Link](https://crates.io/crates/qudag-mcp)           | MCP server for system-level operations - vault access, exchange, quantum-resistant ops exposed as MCP tools.                               | Bridges qudag ecosystem into MCP tool interface for orchestrators.                                                     |

### 3.2 Domain Orchestrators

| Component                        | Repo           | GitHub                                       | Role in MYOS                                                                                                                                          | Integration Notes                                                                                                                       |
| -------------------------------- | -------------- | -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Domain DSL (prompt optimization) | **SynthLang**  | [Link](https://github.com/ruvnet/SynthLang)  | Mathematically-structured prompt language for domain-specific DSLs. Healthcare DSL, DeFi DSL, Carbon DSL built on SynthLang primitives.               | Each Domain Orch uses SynthLang to define domain-specific prompt templates and optimization rules.                                      |
| Domain DSL (prompt programming)  | **PromptLang** | [Link](https://github.com/ruvnet/promptlang) | Prompt-based programming language. Domains define custom interaction patterns and workflows.                                                          | Complements SynthLang - PromptLang for workflow definition, SynthLang for cost/performance optimization.                                |
| Domain safety & guardrails       | **GuardRail**  | [Link](https://github.com/ruvnet/guardrail)  | API-driven safety framework. Domain Orchs use GuardRail for domain-specific output filtering and compliance checks.                                   | Each domain configures its own GuardRail rules (HIPAA for Healthcare, KYC/AML for DeFi, etc.).                                          |
| Domain safety validation         | **SAFLA**      | [Link](https://github.com/ruvnet/SAFLA)      | Production-ready autonomous AI safety system with hybrid memory and meta-cognitive reasoning. Provides safety validation layer for domain operations. | SAFLA's safety validation integrates with the Authority Engine's safety dimension. Domain Orchs use SAFLA for pre-action safety checks. |

### 3.3 Deployment Orchestrators

| Component                  | Repo                       | GitHub                                              | Role in MYOS                                                                                                                                              | Integration Notes                                                                                               |
| -------------------------- | -------------------------- | --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Infrastructure management  | **Agentic DevOps**         | [Link](https://github.com/agenticsorg/devops)       | Autonomous AI-powered platform for managing cloud infrastructure across providers. Becomes the Deployment Orchestrator's infrastructure management layer. | Each Deployment Orch uses Agentic DevOps for node provisioning, scaling, and health management within its zone. |
| Edge node management       | **Agentic Edge Functions** | [Link](https://github.com/agenticsorg/edge-agents/) | Distributed autonomous agents at the network edge. Manages RISC-V sensor fleet - deployment, updates, monitoring.                                         | Deployment Orchs for edge zones use this as their primary management tool. Handles sensor agent lifecycle.      |
| Real-time event processing | **Inflight Agentics**      | [Link](https://github.com/ruvnet/inflight)          | Real-time event processing for continuous monitoring and autonomous action. Deployment Orchs use for zone health monitoring.                              | Feeds zone health data to Deployment Orch. Triggers scaling/remediation actions within time budgets.            |

### 3.4 Pod Orchestrators & Cubelets

| Component                   | Repo                               | GitHub                                               | Role in MYOS                                                                                                            | Integration Notes                                                                                           |
| --------------------------- | ---------------------------------- | ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Pod Orch framework          | **Claude-Flow** (lightweight mode) | [Link](https://github.com/ruvnet/claude-flow)        | Same orchestration framework as System Orch but configured for lightweight, ephemeral pod coordination.                 | Stripped-down Claude-Flow config. Uses open-source LLM (Llama/Phi) instead of paid model.                   |
| ML/DL inference cubelets    | **ONNX-Agent**                     | [Link](https://github.com/ruvnet/onnx-agent)         | Unified pipeline for training, optimizing, and deploying ONNX models. ML/DL cubelets use ONNX runtime compiled to Wasm. | ONNX models compiled to Wasm modules. Run inside Unikraft unikernels on compute nodes.                      |
| Neural network cubelets     | **ruv-fann**                       | [Link](https://crates.io/crates/ruv-fann)            | Pure Rust implementation of Fast Artificial Neural Network library. Lightweight neural net cubelets.                    | Compiled to Wasm. Deterministic, fast, no external dependencies. Ideal for math/ML cubelet type.            |
| LLM streaming cubelets      | **MidStream**                      | [Link](https://github.com/ruvnet/midstream)          | Real-time LLM streaming platform with inflight data analysis. LLM cubelets use for streaming inference.                 | Rust-based. Runs in Linux containers on x86 GPU nodes. Handles streaming output for generative cubelets.    |
| Swarm coordination cubelets | **ruv-swarm-core**                 | [Link](https://crates.io/crates/ruv-swarm-core)      | Core orchestration and agent traits for multi-agent swarms. Cubelets that coordinate multi-agent behaviors.             | Useful for pods that need swarm-like coordination between cubelets. Compiled to Wasm.                       |
| GPU compute cubelets        | **cuda-rust-wasm**                 | [Link](https://crates.io/crates/cuda-rust-wasm)      | CUDA to Rust transpiler with WebGPU/WASM support. Enables GPU-accelerated cubelets.                                     | Transpiles CUDA kernels to Rust/Wasm. Runs on x86 GPU nodes. Bridges GPU compute into Wasm cubelet model.   |
| Search cubelets             | **bit-parallel-search**            | [Link](https://crates.io/crates/bit-parallel-search) | Blazing fast string search using bit-parallel algorithms. 8x faster than naive search.                                  | Compiled to Wasm. Used as a math cubelet for text/pattern search tasks.                                     |
| Math solver cubelets        | **sublinear**                      | [Link](https://crates.io/crates/sublinear)           | High-performance sublinear-time solver for asymmetric systems.                                                          | Compiled to Wasm. Specialized math cubelet for linear algebra and system solving.                           |
| Data retrieval cubelets     | **FACT**                           | [Link](https://github.com/ruvnet/FACT)               | Fast Augmented Context Tools - LLM data retrieval replacing RAG with deterministic tool execution.                      | Cubelets that need fast context retrieval use FACT instead of RAG. Aligns with MYOS determinism principles. |

### 3.5 Chain & Verification

| Component              | Repo                 | GitHub                                            | Role in MYOS                                                                                                                               | Integration Notes                                                                                                                  |
| ---------------------- | -------------------- | ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| DAG consensus          | **qudag-dag**        | [Link](https://crates.io/crates/qudag-dag)        | DAG consensus with QR-Avalanche algorithm and BFT. Complements the Haskell Ouroboros core.                                                 | Rust crate. Ouroboros (Haskell) handles slot-based consensus; qudag-dag provides DAG-based transaction ordering.                   |
| P2P networking         | **qudag-network**    | [Link](https://crates.io/crates/qudag-network)    | LibP2P-based P2P layer with onion routing, dark addressing, and quantum encryption. Core transport for all device-to-device communication. | Foundation of MYOS networking stack. All channel types (chain, authority, pod, data, MCP, telemetry) multiplex over qudag-network. |
| Protocol orchestration | **qudag-protocol**   | [Link](https://crates.io/crates/qudag-protocol)   | Orchestrates crypto + DAG + network components into a unified protocol. Message framing, routing, sub-protocols.                           | Glue layer between qudag-crypto, qudag-dag, and qudag-network. Defines the wire protocol for all MYOS messages.                    |
| CLI management         | **qudag-cli**        | [Link](https://crates.io/crates/qudag-cli)        | Command-line interface for node management, peer management, token exchange. Used for operator-facing chain management.                    | DevOps and operator tool. Deployment Orchs may invoke CLI commands for node management.                                            |
| Core framework         | **qudag**            | [Link](https://crates.io/crates/qudag)            | Core QuDAG framework - darknet for agent swarms with quantum-resistant communication.                                                      | Meta-crate that ties qudag-\* ecosystem together. MYOS builds on top of this.                                                      |
| Multi-agent payments   | **agentic-payments** | [Link](https://crates.io/crates/agentic-payments) | Ed25519 signature verification with BFT for autonomous payments. Cubelet and node incentive payments.                                      | Integrates with Kusari economic model. Cubelets earn rewards, validators earn fees.                                                |

### 3.6 Communication & Networking

| Component               | Repo              | GitHub                                          | Role in MYOS                                                                                                        | Integration Notes                                                                                      |
| ----------------------- | ----------------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Cross-device MCP        | **Federated MCP** | [Link](https://github.com/ruvnet/federated-mcp) | Distributed MCP runtime for federated AI services. Enables cubelets on different devices to use each other's tools. | Core of cross-device tool coordination. Every device exposes local capabilities as MCP tools.          |
| Voice interface         | **Agentic Voice** | [Link](https://github.com/ruvnet/agentic-voice) | AI-powered chat for real-time communication. Voice input/output for AI interaction sessions.                        | Session Pods can include a voice cubelet for spoken conversation. Feeds into the Session Orchestrator. |
| Voice processing        | **Omnipotent**    | [Link](https://github.com/ruvnet/omnipotent)    | Quantum consciousness voice interface using OpenAI speech API. Real-time voice conversations.                       | Alternative/complementary voice interface. Can be used as a specialized voice cubelet.                 |
| Agent orchestration CLI | **AgentXNG**      | [Link](https://github.com/ruvnet/agentXNG)      | CLI tool for development tasks with Claude. Useful for development-time agent testing.                              | Development tool. Used during cubelet and orchestrator development, not in production runtime.         |

### 3.7 Edge Sensors (RISC-V)

| Component             | Repo                            | GitHub                                                        | Role in MYOS                                                                                              | Integration Notes                                                                                                         |
| --------------------- | ------------------------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| Edge agent runtime    | **Agentic Edge Functions**      | [Link](https://github.com/agenticsorg/edge-agents/)           | Distributed autonomous agents at network edge. Runs on RISC-V sensor nodes as the sensor agent framework. | Primary runtime for edge nodes. Manages sensor data collection, local math cubelet execution, upstream data transmission. |
| WiFi pose estimation  | **WiFi-DensePose**              | [Link](https://github.com/ruvnet/wifi-densepose)              | Privacy-first human pose estimation using WiFi CSI data. Example edge AI sensor capability.               | Specialized edge cubelet. Runs as Wasm module on RISC-V. Demonstrates edge AI processing without cameras.                 |
| GPS-denied navigation | **Quantum Magnetic Navigation** | [Link](https://github.com/ruvnet/quantum-magnetic-navigation) | Navigation using quantum magnetometers for GPS-denied environments. Edge sensor capability.               | Specialized edge cubelet for autonomous vehicle / drone navigation in GPS-denied areas.                                   |

### 3.8 Cross-Cutting / Development Tools

| Component                | Repo                 | GitHub                                                  | Role in MYOS                                                                                        | Integration Notes                                                                                               |
| ------------------------ | -------------------- | ------------------------------------------------------- | --------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Development methodology  | **SPARC**            | [Link](https://github.com/ruvnet/sparc)                 | Specification, Pseudocode, Architecture, Refinement methodology. Used for MYOS development process. | Development methodology. Not runtime code. Guides how MYOS components are specified and built.                  |
| Agentic coding framework | **SPARC 2.0**        | [Link](https://github.com/agenticsorg/sparc2)           | Intelligent coding agent with MCP capabilities. Used to develop MYOS components.                    | Development tool. Can generate MYOS cubelet code, test orchestrator logic, etc.                                 |
| Security scanning        | **Agentic Security** | [Link](https://github.com/agenticsorg/agentic-security) | AI-powered vulnerability detection. Scans MYOS codebase for security issues.                        | CI/CD integration. Runs during development and before deployment. Domain Orchs may use for compliance scanning. |
| Code execution flow      | **claude-code-flow** | [Link](https://github.com/ruvnet/claude-code-flow)      | Code execution and refinement flow. Development-time code optimization.                             | Development tool for iterating on cubelet implementations and orchestrator logic.                               |

### 3.9 Potentially Useful (Secondary)

| Repo                          | GitHub                                                     | Potential Use                                                                                 | Priority                      |
| ----------------------------- | ---------------------------------------------------------- | --------------------------------------------------------------------------------------------- | ----------------------------- |
| **q-star**                    | [Link](https://github.com/ruvnet/q-star)                   | Reinforcement learning framework. Could inform Authority Engine reward/penalty tuning.        | Low - research reference      |
| **NOVA**                      | [Link](https://github.com/ruvnet/nova)                     | Knowledge distillation. Could create smaller cubelet models from larger ones.                 | Medium - cubelet optimization |
| **Agentic Diffusion**         | [Link](https://github.com/ruvnet/agentic-difusion)         | Diffusion-based code generation. Could auto-generate cubelet code.                            | Low - development tool        |
| **DSPy.ts**                   | [Link](https://github.com/ruvnet/dspy.ts)                  | Browser-based AI framework. Could enable browser-based MYOS dashboard.                        | Low - UI only                 |
| **Symbolic Scribe**           | [Link](https://github.com/ruvnet/symbolic-scribe)          | Mathematical prompt engineering. Could inform Authority Engine prompt design.                 | Low - research reference      |
| **temporal-compare**          | [Link](https://crates.io/crates/temporal-compare)          | Temporal prediction benchmarking. Could create specialized temporal cubelets.                 | Low - specialized cubelet     |
| **temporal-attractor-studio** | [Link](https://crates.io/crates/temporal-attractor-studio) | Temporal dynamics prediction. Specialized math cubelet candidate.                             | Low - specialized cubelet     |
| **strange-loop**              | [Link](https://crates.io/crates/strange-loop)              | Hyper-optimized strange loops with temporal consciousness. Research-grade.                    | Low - experimental            |
| **subjective-time-expansion** | [Link](https://crates.io/crates/subjective-time-expansion) | Time perception for AI agents. Could inform cubelet scheduling.                               | Low - experimental            |
| **goalie**                    | [Link](https://crates.io/crates/goalie)                    | AI research assistant with GOAP planning. Could be a specialized Pod Orch template.           | Medium - planning cubelet     |
| **Quantum Agentics**          | [Link](https://github.com/agenticsorg/quantum-agentics)    | Quantum computing for agent task allocation. Future integration with Kusari quantum features. | Low - future                  |
| **Quantum Cryptocurrency**    | [Link](https://github.com/ruvnet/quantum_cryptocurrency)   | Quantum-enhanced cryptocurrency. Concepts may inform Kusari token design.                     | Low - Kusari future           |

---

## 4. Repos NOT Used

These repos are not mapped to MYOS because they are irrelevant to the architecture:

| Repo                           | Reason Not Used                                           |
| ------------------------------ | --------------------------------------------------------- |
| **Electo1 JS**                 | Election prediction - unrelated to MYOS                   |
| **Strawberry Phi**             | GPT fine-tuning app - not applicable                      |
| **ai-gist**                    | GitHub gist management - utility, not architecture        |
| **AIHL**                       | AI Hacker League - non-profit org, not code               |
| **retro-ai-ui**                | Retro terminal UI - aesthetic, not functional             |
| **rUvix**                      | Terminal interface - branding, not architecture           |
| **Infinity UI**                | Sci-fi UI - aesthetic, not functional                     |
| **llamastack**                 | Llama Stack UI - specific to Meta stack                   |
| **supa-ruv**                   | Supabase toolkit - different stack                        |
| **Bot Generator Bot**          | ChatGPT prompt - not applicable                           |
| **Voicebot**                   | Flask-based voicebot - superseded by Agentic Voice        |
| **Prompt Engine**              | Prompt template tool - superseded by SynthLang/PromptLang |
| **ai-video**                   | Video analysis - too narrow for architecture              |
| **Fireflies Webhook**          | Transcript webhook - unrelated                            |
| **Story Development Toolkit**  | Story generation - unrelated                              |
| **AWS ECS Video Processor**    | AWS-specific - not architecture                           |
| **Basic SWARM Algorithm**      | Basic gist - superseded by ruv-swarm-core                 |
| **GenAI-Superstream**          | Data science demo - not architecture                      |
| **GPT Repository**             | GPT catalog - not code                                    |
| **LLM TCO Calculator**         | Cost estimation - utility only                            |
| **MoE Model Implementation**   | PyTorch demo - superseded by ONNX-Agent                   |
| **Cognitive Framework**        | ChatGPT prompt framework - not applicable                 |
| **TikTok Recommender Config**  | Azure config - unrelated                                  |
| **Sentient Systems**           | Cognitive architecture gist - research only               |
| **File Summarization API**     | LlamaIndex demo - not architecture                        |
| **AWS Dev**                    | AWS dev environment - not architecture                    |
| **Agentic Preview**            | Fly.io preview - superseded by Agentic DevOps             |
| **q-space**                    | Azure Quantum deployment - different platform             |
| **pipackager**                 | PyPI tool - Python specific                               |
| **Reflective Engineer**        | LangChain dev environment - different framework           |
| **rUv-dev**                    | Generic dev tool - not architecture                       |
| **Auto-Browser**               | Web automation - not architecture                         |
| **Agentic Robots.txt**         | Web protocol - not architecture                           |
| **Agentic Employment**         | Employment framework - different domain                   |
| **Agentic Search**             | GitHub Copilot extension - IDE specific                   |
| **Agentic-algorithms**         | Algorithm gists - reference only                          |
| **AIDO**                       | Decentralized org without blockchain - different approach |
| **Hello World Agent**          | Tutorial agent - not production                           |
| **Agent Algorithm Repository** | Algorithm gist - reference only                           |
| **rUv MoE Toolkit**            | Self-learning toolkit gist - reference only               |
| **Agentic Reports**            | Report generation - could be a cubelet but too narrow     |
| **SPARC IDE**                  | VSCode distribution - IDE, not architecture               |
| **Genesis UI**                 | Physics simulation UI - different domain                  |
| **CodeSwarm**                  | VSCode remote MCP - IDE specific                          |
| **Dynamo MCP**                 | Template management - development utility                 |
| **AgenticsJS**                 | Search library - JavaScript, not Rust/Haskell             |

---

## 5. Integration Priority

### 5.1 Phase 1 - Foundation (Must Have)

These repos form the core that everything else depends on:

```
PRIORITY 1 (build first):
  qudag-crypto          → All security depends on this
  qudag-vault-core      → Key management for boot chain
  ANS                   → Identity for everything
  qudag-network         → All communication depends on this
  qudag-protocol        → Wire protocol
  qudag-dag             → Chain consensus

PRIORITY 2 (core system):
  Claude-Flow           → System Orchestrator framework
  SAFLA                 → Safety validation
  GuardRail             → Safety guardrails
  Federated MCP         → Cross-device coordination
```

### 5.2 Phase 2 - Cubelets & Domains

```
PRIORITY 3 (cubelet ecosystem):
  ONNX-Agent            → ML inference cubelets
  ruv-fann              → Neural network cubelets
  MidStream             → LLM streaming cubelets
  cuda-rust-wasm        → GPU compute bridge
  bit-parallel-search   → Search cubelets
  sublinear             → Math solver cubelets
  FACT                  → Context retrieval cubelets

PRIORITY 4 (domain layer):
  SynthLang             → Domain DSLs
  PromptLang            → Domain workflows
  Agentic Security      → Compliance scanning
```

### 5.3 Phase 3 - Operations & Edge

```
PRIORITY 5 (deployment):
  Agentic DevOps        → Infrastructure management
  Agentic Edge Functions → Edge sensor fleet
  Inflight Agentics     → Real-time monitoring
  agentic-payments      → Economic model

PRIORITY 6 (edge & voice):
  WiFi-DensePose        → Edge AI demo
  Quantum Mag Nav       → Edge navigation
  Agentic Voice         → Voice interface
  Omnipotent            → Voice processing
```

---

## 6. New Components (Not From Existing Repos)

These components must be built from scratch:

| Component                      | Language               | Reason                                                                                                                                                       |
| ------------------------------ | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Authority Engine**           | Haskell                | Core MBAC math - no existing repo covers this. Built with haskell.nix.                                                                                       |
| **Ouroboros Consensus Core**   | Haskell                | Slot-based consensus - must be custom. Wraps in Rust via FFI.                                                                                                |
| **Consensus Adapter**          | Rust + Haskell         | Pluggable consensus interface for Kusari evolution.                                                                                                          |
| **Cubelet Runtime Manager**    | Rust                   | Manages Wasm + Unikernel + Container lifecycle.                                                                                                              |
| **NixOS System Configuration** | Nix                    | `flake.nix` defining all node profiles.                                                                                                                      |
| **Unikraft Integration**       | Nix + C                | Nix derivations for building Unikraft cubelet images.                                                                                                        |
| **Pod Orchestrator Templates** | Config                 | Template definitions for pod orch types.                                                                                                                     |
| **Dimension Registry**         | Haskell + Config       | Flexible dimension configuration system.                                                                                                                     |
| **Edge Sensor Agent**          | Rust                   | Lightweight agent for RISC-V sensor data collection.                                                                                                         |
| **Authority RPC Client**       | Rust                   | Edge nodes' RPC client to centralized Authority Engine.                                                                                                      |
| **Vector DB (Qdrant)**         | Rust (external)        | Semantic vector search for KB entries. Runs as a NixOS service on compute nodes. Supports similarity queries across System, Domain, and Pod KBs.             |
| **Knowledge Graph DB**         | Rust/Config (external) | Graph-based knowledge relationships and traversal for the KB. Runs as a NixOS service alongside Vector DB. Stores entity-relation mappings for KB entries.   |
| **KB Service Layer**           | Rust                   | Manages hierarchical KB lifecycle (System KB, Domain KBs, Pod KBs). Handles confidence scoring (0-1000), decay rates, and on-chain/off-chain metadata split. |

---

## 7. Dependency Graph

```
qudag-crypto ──────────────┐
                           ▼
qudag-vault-core ──► Boot Chain ──► Kernel ──► Runtime
                           │
qudag-network ─────────────┤
qudag-protocol ────────────┤
qudag-dag ─────────────────┤
                           ▼
ANS ──────────────► Device Identity ──► Peer Discovery
                           │
                           ▼
Claude-Flow ──────► System Orchestrator
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         Domain Orch  Deploy Orch   Chain Node
         (SynthLang)  (Agentic      (qudag-dag +
         (GuardRail)   DevOps)       Ouroboros)
         (SAFLA)      (Inflight)
              │            │
              ▼            ▼
         Pod Orchs ──► Cubelets
         (Claude-Flow)  (ONNX-Agent, ruv-fann,
                         MidStream, cuda-rust-wasm,
                         bit-parallel-search, sublinear)
                           │
                           ▼
                    Federated MCP ──► Cross-Device
                    (federated-mcp)
                           │
                           ▼
                    Edge Sensors
                    (Agentic Edge Functions,
                     WiFi-DensePose, Quantum Nav)
```

---

## 8. Invariants

```
INV-1:  All qudag-* crates are used as Rust dependencies, not forked
INV-2:  Claude-Flow is the ONLY orchestration framework (used at System, Domain, and Pod levels)
INV-3:  ANS is the SINGLE identity registry for all entities (cubelets, orchs, devices)
INV-4:  Every cubelet that can be compiled to Wasm MUST be compiled to Wasm (no native binaries except LLM agents)
INV-5:  New components (Authority Engine, Ouroboros) are built from scratch in Haskell
INV-6:  All builds go through Nix derivations for reproducibility
INV-7:  No repo is used that introduces a dependency on a specific cloud provider (AWS, GCP, Azure)
INV-8:  Security scanning (Agentic Security) runs on every component before deployment
INV-9:  Phase 1 repos must be integrated and tested before Phase 2 work begins
INV-10: Repos are INTEGRATED (wrapped, adapted), not replaced or rewritten
INV-11: Every integrated repo's cubelets must be assigned a lattice position (Framework-Stage-Index)
INV-12: CIG (Cubelet Interaction Graph) must be populated with all integrated cubelet nodes and edges before Phase 2
INV-13: STK invariant definitions from the cubelet registry must be loaded into the Authority Engine at boot
```

---

## 9. Interaction with Other Documents

- **Master Document (00):** Defines the Rubik's Lattice (10×5×15 = 750 cubelets), CIG, STSol templates, and fabric threads that all repos must integrate into.
- **Node Topology (10):** Repo assignments align with hardware topology and lattice stages - edge repos (Stage 0, 4) run on RISC-V, GPU repos (Stage 3) on x86, core repos on x86/ARM.
- **Authority Model (01):** Authority Engine is NEW (Haskell). Enforces STK invariants (150, loaded from cubelet registry). qudag-crypto provides cryptographic primitives for signing decisions.
- **Pod Assembly (03):** Cubelet repos (ONNX-Agent, ruv-fann, etc.) define cubelet capabilities at specific lattice positions. STSol templates define which cubelets assemble into pods.
- **Cubelet I/O (04):** Pipeline data flows through STF fabric threads and qudag-network for cross-device edges. Framework ordering (STA → STI → STD → STF → STK) governs pipeline construction.
- **Pod Orchestrator (05):** Claude-Flow is the base framework for all orchestrator levels. Pod Orchs use open-source models (Llama 8B).
- **Verification & Audit (06):** qudag-dag + Ouroboros (new) form the chain layer. STF fabric threads route domain-specific logs. qudag-protocol defines wire format.
- **Boot & Trust Chain (07):** qudag-crypto + qudag-vault-core provide boot verification. CIG is loaded into Neo4j during boot sequence. STK invariant definitions verified before system goes operational.
- **Runtime Infrastructure (08):** Math-bound cubelets (STK/STF) in Tier 1 Wasm+Unikernel. ML/DL cubelets (STI) in Tier 1-2 Wasm. LLM cubelets in Tier 3 containers.
- **Communication (09):** qudag-network + qudag-protocol + Federated MCP form the full networking stack. STF fabric threads multiplexed across P2P channels.
- **Knowledge Base (12):** FACT repo provides deterministic KB query tooling. CIG (Neo4j) is the KB's structural backbone - 750 nodes with pre-seeded relationships. Qdrant provides vector search on top.
