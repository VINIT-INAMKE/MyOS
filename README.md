# MYOS

MYOS is a deterministic operating system for AI. Instead of letting AI agents run freely with unpredictable behavior, MYOS treats them like processes in a traditional OS - bounded, scheduled, monitored, and auditable.

## What it does

- Every AI action goes through a math-based authority check before it executes
- Work is broken into small units called cubelets (think processes), grouped into pods (think jobs)
- All decisions are logged on a tamper-proof chain - anyone can read them
- The system is built entirely with NixOS, so the same config always produces the same output
- Context doesn't live in an LLM's memory - it's externalized to a knowledge base and typed data graphs, so nothing gets lost

## How it's different

Most AI agent frameworks are one LLM doing everything in a growing context window. MYOS decomposes the problem: 750 cubelets each do one thing with typed inputs and outputs, orchestrators coordinate using externalized state, and authority math governs what any entity is allowed to do. No single component holds the whole picture - the architecture does.

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
| How knowledge is stored and governed      | [12-knowledge-base.md](architecture/12-knowledge-base.md)                                             |
