# MYOS Boot & Trust Chain - Power-On to Operational State

## Version: 2.0 | Status: LOCKED

---

## 1. Overview

This document defines everything that happens between a MYOS device powering on and becoming operational. The boot process establishes the **root of trust** - the foundation that every subsequent security guarantee depends on.

```
BOOT SEQUENCE:
  Power On
    → Firmware Verification (root of trust)
    → Kernel Load & Verify
    → Rust Userspace Initialization
    → Service Registration
    → Cubelet Pool Initialization
    → Authority Engine Startup (Haskell)
    → Chain Node Startup (Ouroboros)
    → Network Initialization & Peer Discovery
    → System Orchestrator Activation
    → READY
```

Each stage verifies the next before handing off control. If any stage fails verification, the device halts and enters a **secure failure mode** (no partial boot, no degraded operation without explicit configuration).

The boot process also includes **CIG (Cubelet Interaction Graph) initialization** - loading all 750 cubelet nodes and 1500+ edges (across 9 edge types) into the Neo4j graph database. Additionally, all **STK invariant definitions** (150 formal invariants across 7 families) are loaded and verified before the Authority Engine begins accepting requests. These steps ensure the full lattice topology and its formal guarantees are in place before any pod assembly can occur.

---

## 2. Target Hardware

### 2.1 Primary Platform

```
Core System:    x86_64 / ARM (primary today)
                System Orchestrator, Authority Engine, Domain servers,
                compute nodes, chain validators

Edge Sensors:   RISC-V (always)
                SoCs: SiFive, StarFive, Allwinner D1, Kendryte, Milk-V
                Sensor data collection, light math, light validation

Compute Nodes:  x86_64 (GPU for LLM) + x86 or RISC-V (math/ML)
                Powerful RISC-V (SiFive P870, Ventana Veyron) for general compute

Future:         Migrate core system to RISC-V when server chips mature

See 10-node-topology-orchestrator-hierarchy.md for full hardware assignment map.
```

### 2.2 Hardware Requirements

```
FULL VALIDATOR NODE (produces blocks, runs full orchestration):
  CPU:          4+ RISC-V cores (64-bit, RV64GC)
  Memory:       4GB+ RAM
  Storage:      32GB+ (SSD/eMMC for chain storage)
  Secure Storage: SoC secure enclave or external HSM
  Crypto:       Hardware AES acceleration (optional, improves throughput)
  Network:      Ethernet or WiFi (stable connectivity for consensus)

LIGHT VALIDATION NODE (verifies decisions, runs cubelets):
  CPU:          2+ RISC-V cores
  Memory:       1GB+ RAM
  Storage:      8GB+ (block headers only)
  Secure Storage: SoC secure storage
  Network:      Any (Ethernet, WiFi, LoRa for IoT)

MINIMAL EDGE NODE (submit/read only, constrained IoT):
  CPU:          1 RISC-V core (RV32/RV64)
  Memory:       256MB+ RAM
  Storage:      2GB+
  Secure Storage: Software-simulated (no hardware secure storage)
  Network:      Any
```

### 2.3 Hardware Abstraction Layer (HAL)

MYOS defines a hardware abstraction layer so that the upper stack is hardware-independent.

```
hal_interface {
    // Identity
    get_device_id() → DeviceId
    get_hardware_public_key() → PublicKey
    sign_with_hardware_key(data) → Signature

    // Secure storage
    store_secret(key, value) → result
    retrieve_secret(key) → value
    delete_secret(key) → result

    // Crypto acceleration
    has_crypto_accelerator() → bool
    hardware_encrypt(algo, key, plaintext) → ciphertext
    hardware_decrypt(algo, key, ciphertext) → plaintext

    // Timer
    get_monotonic_time() → Timestamp        // deterministic, monotonic
    get_timer_resolution() → nanoseconds

    // CPU
    get_core_count() → int
    get_core_frequency() → Hz
    set_core_affinity(task, core) → result  // for RT pinning

    // Memory
    get_total_memory() → bytes
    get_available_memory() → bytes
}
```

HAL implementations exist per platform (RISC-V, ARM, x86_64). All upper layers use the HAL interface, never raw hardware access.

---

## 3. Firmware-Level Root of Trust

### 3.1 Boot Chain

```
STAGE 0: SoC ROM Bootloader (immutable, in silicon)
  - Loads Stage 1 from secure storage
  - Verifies Stage 1 signature against SoC-burned public key
  - If verification fails → HALT (no boot)

STAGE 1: Firmware Verifier (signed, in secure flash)
  - Loads the kernel image
  - Verifies kernel signature against firmware's embedded public key
  - Verifies kernel hash against expected hash
  - If verification fails → HALT
  - Sets up initial memory layout
  - Transfers control to kernel

STAGE 2: Kernel Loader (part of Linux boot)
  - Linux kernel initializes
  - Verifies device tree and initial ramdisk
  - Sets up memory management, interrupt handlers
  - Initializes PREEMPT_RT scheduler
  - Launches init process (Rust userspace)

STAGE 3: Rust Userspace Initialization
  - Verifies MYOS configuration file (signature + hash)
  - Loads dimension registry, authority floors, system config
  - Initializes Wasm runtime (for math/ML cubelets)
  - Initializes container runtime (for LLM agent cubelets)
  - Starts Authority Engine (Haskell process via IPC)
  - Starts Chain Node (Ouroboros)
  - Starts Network services (qudag-network)
  - Registers device in ANS (Agent Name Service)
  - Activates System Orchestrator (LLM)
```

### 3.2 Verification at Each Stage

Each stage produces a verification record:

```
stage_verification {
    stage:          String
    image_hash:     Hash (BLAKE3 via qudag-crypto)
    signature:      Signature (ML-DSA for post-quantum, or Ed25519)
    expected_hash:  Hash (from previous stage or configuration)
    verified:       bool
    timestamp:      Timestamp (hardware monotonic clock)
}
```

All stage verifications are collected into a **boot attestation record** and written on-chain as the first action after the chain node starts (see Section 5).

### 3.3 Key Hierarchy

```
KEY HIERARCHY:

Level 0: SoC Root Key (burned into silicon/eFuse)
  - Used ONLY to verify Stage 1 firmware
  - Never exposed outside the SoC
  - Cannot be changed

Level 1: Firmware Signing Key (embedded in Stage 1)
  - Used to verify kernel images
  - Can be rotated via secure firmware update
  - Stored in SoC secure storage

Level 2: Device Identity Key (generated at first boot, stored in secure storage)
  - Post-quantum keypair (ML-KEM-768 + ML-DSA via qudag-crypto)
  - Used for device authentication, chain transactions, message signing
  - Managed by qudag-vault-core
  - Registered in ANS with device capabilities

Level 3: Session Keys (ephemeral, generated per connection)
  - Used for device-to-device data encryption
  - Classical (AES-256-GCM with ECDH key exchange) for pipeline data
  - Post-quantum (ML-KEM-768) for chain and authentication channels
  - Rotated frequently, never stored
```

### 3.4 Secure Storage (qudag-vault-core)

Device keys are managed by qudag-vault-core:

```
vault_operations {
    // Key generation
    generate_device_keypair() → (PublicKey, encrypted_private_key)
    generate_session_keypair() → (PublicKey, ephemeral_private_key)

    // Key storage
    store_key(key_id, encrypted_key) → result
    retrieve_key(key_id) → encrypted_key
    destroy_key(key_id) → result

    // Signing
    sign(key_id, data) → Signature
    verify(public_key, data, signature) → bool

    // Encryption
    encrypt(key_id, plaintext) → ciphertext
    decrypt(key_id, ciphertext) → plaintext
}
```

All private keys are encrypted at rest using the SoC hardware key (Level 0). If the SoC is compromised, all keys must be rotated.

---

## 4. Kernel Initialization

### 4.1 Linux Kernel Configuration

MYOS uses a **minimal Linux kernel** with PREEMPT_RT patches on RISC-V.

```
KERNEL CONFIGURATION:

  Base:           Linux 6.x (latest LTS) with PREEMPT_RT
  Architecture:   x86_64 (core/compute), RISC-V 64-bit RV64GC (edge/compute)
  Build system:   NixOS (declarative system configuration, replaces Buildroot/Yocto)

  ENABLED:
    - PREEMPT_RT (full preemption)
    - cgroups v2 (resource limits for cubelets)
    - namespaces (PID, network, mount, user - for container isolation)
    - seccomp (syscall filtering)
    - SCHED_FIFO / SCHED_RR (real-time scheduling)
    - CPU isolation (isolcpus for RT cores)
    - Memory locking (mlockall for RT processes)
    - Device tree support (RISC-V hardware description)

  DISABLED (stripped for minimal attack surface):
    - General-purpose networking daemons (no sshd, no telnetd by default)
    - Unnecessary filesystems (no NFS, no CIFS)
    - Unnecessary drivers (only hardware-specific drivers included)
    - Swap (no swap - all memory is physical, deterministic)
    - Dynamic module loading (all modules compiled in - no runtime module insertion)
    - Debug interfaces in production (kgdb, ftrace disabled)
```

### 4.2 CPU Partitioning

```
CPU LAYOUT (example: 4-core system):

  Core 0:  Linux housekeeping (non-RT)
           - Kernel threads, interrupts, background tasks
           - isolcpus excludes this from RT workload

  Core 1:  MYOS Safety + Authority Engine
           - Safety watchdog (highest priority, P0)
           - Authority Engine IPC handler (P1)
           - Pinned, dedicated, no other tasks

  Core 2:  MYOS Cubelets (RT workload)
           - AI inference (P3)
           - Math cubelets (P3)
           - Agent runtime (P4)
           - Scheduled by MYOS userspace scheduler

  Core 3:  MYOS Communication + Chain
           - Network services (P5)
           - Chain node (P5)
           - Logging (P6)
           - Background tasks (P7)

For 2-core systems:
  Core 0: Linux + Communication + Chain
  Core 1: Safety + Authority + Cubelets (time-partitioned)
```

### 4.3 Memory Layout

```
MEMORY REGIONS (pre-allocated at boot, no dynamic allocation in RT paths):

  Kernel region:        Linux kernel (RX only from userspace)
  Authority Engine:     Isolated Haskell process heap (fixed size, pinned)
  Wasm Runtime:         Wasm cubelet sandbox memory (fixed pool)
  Container Runtime:    LLM agent container memory (cgroup-limited)
  Shared IPC:           Message passing buffers (controlled R/W)
  Chain Storage:        Block storage, merkle trees (mmap'd)
  Network Buffers:      Send/receive buffers (fixed pool)
  Logging Buffer:       Ring buffer for off-chain events (fixed size, overflow → drop oldest)

No malloc in RT paths. All buffers are pre-allocated at boot.
Dynamic allocation only in non-RT paths (Linux housekeeping core).
```

---

## 5. Service Initialization Sequence

After the kernel is running, MYOS services start in a defined order.

```
INITIALIZATION ORDER (dependencies → must start in this order):

  1. HAL Initialization
     - Detect hardware capabilities
     - Initialize crypto accelerator (if available)
     - Read device identity from secure storage
     - Output: hal_context

  2. Configuration Loader
     - Load MYOS config from verified storage
     - Verify config signature (using device key)
     - Verify config hash
     - Parse dimension registry, authority floors, system parameters
     - Output: system_config

  3. Wasm Runtime
     - Initialize Wasmtime/WasmEdge runtime
     - Pre-compile Wasm cubelet modules (AOT compilation)
     - Output: wasm_runtime_handle

  4. Container Runtime
     - Initialize container engine (lightweight, e.g., crun)
     - Pre-pull container images for LLM agent cubelets
     - Verify container image signatures
     - Output: container_runtime_handle

  4b. Secret Provisioning (Agenix)
     - NixOS activation triggers agenix decryption
     - Encrypted .age secret files decrypted using SSH host keys
     - Decrypted secrets placed in /run/agenix/ (tmpfs, RAM only)
     - Each secret owned by its service UID (filesystem permissions)
     - Secrets available to services before they start
     - Output: /run/agenix/* secret paths

  5. Authority Engine (Haskell)
     - Start Haskell process (separate memory-isolated process)
     - Load authority configuration (dimension registry, floors, ceilings)
     - Load signing key from /run/agenix/authority-config-signing-key
     - Initialize authority state (load from persistent storage or fresh)
     - Establish IPC channel with Rust userspace
     - Output: authority_engine_handle

  6. Cubelet Pool Registration
     - Load cubelet manifests from verified storage
     - Register all 750 cubelets (capabilities, I/O schemas, types)
     - Initialize cubelet AV from persistent storage (or defaults for fresh)
     - Mark all cubelets as "available"
     - Output: cubelet_registry

  7. Off-Chain Database
     - Initialize event log + content-addressed store
     - Initialize time-series index
     - Load last checkpoint (resume from where we left off)
     - Output: off_chain_db_handle

  8. Chain Node (Ouroboros)
     - Start consensus module (Haskell Ouroboros core)
     - Start storage module (Rust block store)
     - Start network module (Rust P2P)
     - Sync with peers (if existing chain) or genesis (if first boot)
     - Determine role: full validator or light validation node
     - Output: chain_node_handle

  9. Network Services
     - Initialize qudag-network (LibP2P P2P layer)
     - Initialize qudag-protocol (protocol handler)
     - Initialize Federated MCP (cross-device MCP runtime)
     - Register device in ANS (Agent Name Service) with:
       - device_id
       - hardware_public_key
       - capabilities (what cubelets this device has)
       - node_type (full validator / light validation)
     - Discover peers via ANS + qudag-network
     - Establish secure connections with known peers
     - Output: network_handle

  10. System Orchestrator Activation
      - Start System Orchestrator LLM (powerful model)
      - Load orchestration config (pod templates, task decomposition rules)
      - Initialize Pod Orchestrator registry
      - Output: system_orch_handle

  11. Boot Attestation
      - Collect all stage verification records
      - Create boot_verification_log (see 06-verification-audit.md)
      - Write boot attestation ON-CHAIN (first chain transaction)
      - Output: boot_attestation_id

  12. SYSTEM READY
      - All services initialized
      - Device is operational
      - Begin accepting tasks
```

### 5.1 Initialization Failure Handling

```
If any step fails:
  Steps 1-4:  HALT. Device cannot operate without hardware, config, or runtimes.
  Step 5:     HALT. Authority Engine is mandatory. No operation without it.
  Step 6:     WARN. If some cubelets fail to register, continue with reduced pool.
              Log failed cubelets. They can be re-registered later.
  Step 7:     WARN. If off-chain DB fails, buffer events in memory ring buffer.
              Attempt DB recovery in background.
  Step 8:     DEGRADE. If chain node can't sync, operate in "offline mode."
              Buffer on-chain events locally. Sync when connectivity is restored.
  Step 9:     DEGRADE. If network fails, operate as standalone device.
              No cross-device communication. Local pods only.
  Step 10:    HALT. System Orchestrator is mandatory. No operation without it.
  Step 11:    WARN. If attestation can't be written (chain not ready), buffer it.

HALT = device does not become operational. Human intervention required.
DEGRADE = device operates with reduced capabilities. Logged as degraded_boot event.
WARN = device operates normally but with a logged warning.
```

---

## 6. Firmware Update Process

Firmware and kernel updates must go through a secure update process.

```
UPDATE PROCESS:

  1. Update package arrives (signed by the update authority)
  2. Verify package signature (post-quantum: ML-DSA)
  3. Verify package hash
  4. Extract:
     - New firmware image (if firmware update)
     - New kernel image (if kernel update)
     - New MYOS config (if config update)
     - New cubelet manifests (if cubelet update)
  5. Build new system image via Nix derivation (reproducible)
  6. Switch to new NixOS generation (atomic)
  7. Reboot into new generation
  8. New boot chain verifies the updated images
  9. If boot succeeds → new generation is active
  10. If boot fails → rollback to previous generation (automatic, nixos-rebuild switch --rollback)

NIXOS GENERATIONS (replaces A/B partitioning):
  NixOS maintains unlimited generations (not just A/B)
  Each generation is a complete, immutable system configuration
  Rollback to ANY previous generation is one command away
  All generations are stored in /nix/store (content-addressed, immutable)
  Atomic switching - either fully applied or not at all
```

---

## 7. Configuration

```
boot_config {
    // Hardware
    target_arch:            "riscv64"
    hal_implementation:     "riscv_sifive"      // or "riscv_starfive", "arm_cortex", etc.

    // Secure boot
    firmware_signing_algo:  "ml-dsa-65"         // post-quantum
    kernel_hash_algo:       "blake3"
    config_signing_algo:    "ml-dsa-65"

    // Key management
    vault_backend:          "qudag-vault-core"
    device_key_algo:        "ml-kem-768"        // post-quantum key exchange
    device_sign_algo:       "ml-dsa-65"         // post-quantum signature

    // Kernel
    kernel_version:         "6.x-preempt-rt"
    build_system:           "nixos"
    rt_cores:               [1, 2, 3]           // cores reserved for RT
    housekeeping_cores:     [0]                  // cores for Linux housekeeping

    // Memory
    authority_engine_heap:  "256MB"
    wasm_pool_size:         "512MB"
    container_memory_limit: "1GB"
    ipc_buffer_size:        "64MB"
    logging_ring_buffer:    "128MB"

    // Update
    ab_partitioning:        false  # replaced by NixOS generations
    rollback_on_boot_fail:  true

    // Initialization
    halt_on_critical_failure: true
    degrade_on_network_failure: true
    offline_mode_enabled:   true
}
```

---

## 8. Invariants (Must Always Hold)

```
INV-1:  No code executes without cryptographic verification by the previous boot stage
INV-2:  Boot chain failure at any stage → HALT (no partial boot)
INV-3:  Device identity key is post-quantum (ML-KEM-768 + ML-DSA)
INV-4:  All private keys are encrypted at rest using SoC hardware key
INV-5:  System configuration is signature-verified before loading
INV-6:  No dynamic kernel module loading in production
INV-7:  No swap (all memory is physical)
INV-8:  RT cores are dedicated (isolcpus) - no Linux housekeeping on RT cores
INV-9:  No dynamic memory allocation in RT paths (all pre-allocated at boot)
INV-10: Authority Engine and System Orchestrator failure → device HALTS
INV-11: NixOS generations ensure rollback on failed updates (replaces A/B partitioning)
INV-12: Boot attestation is written on-chain as the first transaction
INV-13: CIG must be fully loaded (750 nodes, all edges) before pod assembly is enabled
INV-14: STK invariant definitions must be loaded and verified before Authority Engine accepts requests
INV-15: Service initialization follows strict dependency order
INV-16: Firmware updates are signature-verified (post-quantum)
```

---

## 9. Interaction with Other Documents

- **Authority Model (01):** Authority Engine starts at Step 5. Configuration (dimension registry, floors) loaded at Step 2.
- **Pod Assembly (03):** Cubelet pool registered at Step 6. System Orchestrator activated at Step 10.
- **Cubelet I/O (04):** Wasm runtime (Step 3) and container runtime (Step 4) provide cubelet isolation.
- **Pod Orchestrator (05):** Pod Orch registry initialized at Step 10 as part of System Orchestrator activation.
- **Verification & Audit (06):** Chain node started at Step 8. Boot attestation written at Step 11.
- **Existing Repos:** qudag-crypto (key hierarchy), qudag-vault-core (key management), ANS (device registration, Step 9).
- **Master Document (00-master.md) - CIG & Lattice:** CIG initialization (750 nodes + 1500+ edges into Neo4j) occurs during the boot service sequence. The full cubelet lattice (10 stages x 5 frameworks x 15 per cell) must be loaded before pod assembly is enabled.
- **Authority Model (01) - STK Invariants:** All 150 STK invariant definitions across 7 families are loaded and verified during boot. The Authority Engine will not accept requests until invariant verification is complete.
- **Knowledge Base (12-knowledge-base.md):** KB services (Vector DB, Knowledge Graph DB) are initialized during the boot service sequence. KB state is loaded from persistent storage to restore the hierarchical knowledge base (System KB, Domain KBs).
