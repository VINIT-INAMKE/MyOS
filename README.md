# MYOS

MYOS is a deterministic operating system for AI. Instead of letting AI agents run freely with unpredictable behavior, MYOS treats them like processes in a traditional OS - bounded, scheduled, monitored, and auditable.

## What it does

- **750 cubelets** organized as a Rubik's Lattice (10 domain stages x 5 framework layers x 15 per cell) - each with a named role, typed I/O, and formal invariant bindings
- **Three model types** - math-bound (~350, strongest isolation), ML/DL (~250, Wasm), LLM (~100, containers) - the right tool at the right security level
- **Dual-gate authority** - every action must pass both an AV (Authority Value) permission check AND STK kernel invariant satisfaction, enforced by ProofPerl
- **Graph-informed assembly** - pods are assembled using STSol templates and the Cubelet Interaction Graph (CIG), not blind scoring
- **Fabric-routed verification** - all decisions logged on-chain, domain data routed through named STF fabric threads (EthosLedger, KarbonLedger, etc.)
- **Reproducible everything** - NixOS from boot to deployment, same config always produces same output
- **Knowledge on structure** - KB built on the CIG backbone (Neo4j), domain KBs mapped to fabric threads, never starts empty

## How it's different

Most AI agent frameworks are one LLM doing everything in a growing context window. MYOS decomposes the problem into a structured lattice:

- **5 frameworks** define the pipeline: STA (intent) -> STI (logic) -> STD (execution) -> STF (connectivity) -> STK (proof)
- **10 stages** span from Identity (Stage 0) through Meta-Ethics (Stage 9) - Stages 0-4 are the MYOS core, Stages 5-9 activate with Kusari
- **150 STK invariants** across 7 families (safety, privacy, fairness, verifiability, accountability, planetary integrity, ethical reflexivity) enforce formal constraints no entity can bypass, anchored to the PCCSCEFVRI ethical values
- **Rubik's Moves** (Rotate.Stage, Lock.Invariants, Thread.Sync) govern lattice evolution - every structural change is STK-constrained and logged on-chain
- The **autopoietic loop** (Stage 9 -> Stage 0 feedback) gives the system a defined ethical objective function for its reinforcement learning

No single component holds the whole picture - the architecture does.

## Where to start

| What you want                             | Read this                                                                                             |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Understand the whole system in one page   | [MYOS Architecture Visual](architecture/MYOS-architecture-visual.md)                                  |
| Understand the core concepts              | [00-MYOS-master.md](architecture/00-MYOS-master.md)                                                   |
| How authority and permissions work        | [01-authority-model.md](architecture/01-authority-model.md)                                           |
| How conflicts between agents get resolved | [02-conflict-resolution.md](architecture/02-conflict-resolution.md)                                   |
| How teams of cubelets are assembled       | [03-pod-assembly.md](architecture/03-pod-assembly.md)                                                 |
| How data flows between cubelets           | [04-cubelet-io-contract.md](architecture/04-cubelet-io-contract.md)                                   |
| How the pod orchestrator works            | [05-pod-orchestrator.md](architecture/05-pod-orchestrator.md)                                         |
| How everything is verified and audited    | [06-verification-audit.md](architecture/06-verification-audit.md)                                     |
| How the system boots securely             | [07-boot-trust-chain.md](architecture/07-boot-trust-chain.md)                                         |
| Runtime scheduling and isolation          | [08-runtime-infrastructure.md](architecture/08-runtime-infrastructure.md)                             |
| Networking and cross-device communication | [09-communication-networking.md](architecture/09-communication-networking.md)                         |
| Node types and orchestrator hierarchy     | [10-node-topology-orchestrator-hierarchy.md](architecture/10-node-topology-orchestrator-hierarchy.md) |
| How repos map to architecture layers      | [11-repository-mapping.md](architecture/11-repository-mapping.md)                                     |
| How knowledge is stored and governed      | [12-knowledge-base.md](architecture/12-knowledge-base.md)                                             |

## Cubelet Universe (source definitions)

The full 750-cubelet registry, interaction graph spec, Neo4j import scripts, and implementation roadmap live in [`cubeletuni/`](../cubeletuni/). These are the source definitions that the architecture docs reference.
