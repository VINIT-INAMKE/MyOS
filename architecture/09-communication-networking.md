# MYOS Communication & Networking - Device-to-Device, P2P & Cross-System Messaging

## Version: 2.0 | Status: LOCKED

---

## 1. Overview

This document defines how MYOS devices communicate with each other, discover peers, authenticate connections, and exchange data across a distributed network of devices.

P2P channels are **multiplexed per STF fabric thread** - each of the 11+ named ledgers (EthosLedger, KarbonLedger, etc.) gets its own logical stream within the multiplexed connection, ensuring that domain-specific data flows are isolated and prioritized appropriately at the network level. Cross-device pod operations use **CIG FEDERATES edges** to determine which cubelets on remote devices can participate in distributed pods; the FEDERATES edge type in the CIG (Neo4j, 750 nodes, 9 edge types, 1500+ edges) explicitly encodes cross-device cubelet relationships and must be verified before any cross-device pipeline edge is established.

MYOS communication is built on top of existing projects from the repository ecosystem:

```
REPO MAPPING:

  P2P Networking        → qudag-network (LibP2P, onion routing, quantum encryption)
  Protocol Layer        → qudag-protocol (orchestrates crypto + DAG + network)
  Device Discovery      → ANS (Agent Name Service - OWASP-based identity)
  Cross-Device MCP      → Federated MCP (distributed MCP runtime)
  Edge Deployment       → Agentic Edge Functions (distributed agents at edge)
  Cryptographic Layer   → qudag-crypto (post-quantum primitives)
  Key Management        → qudag-vault-core (quantum-resistant vault)
```

These repos are integrated, not replaced. MYOS adds an orchestration layer on top that connects them to the cubelet/pod/authority model.

---

## 2. Network Architecture

### 2.1 Device Network Topology

MYOS devices form a **peer-to-peer mesh network** with no central server.

```
MYOS NETWORK:

  ┌──────────┐     qudag-network     ┌──────────┐
  │ Device A │◄════(P2P LibP2P)═════►│ Device B │
  │ Full     │                       │ Full     │
  │ Validator│     ┌──────────┐      │ Validator│
  └────┬─────┘     │ Device C │      └────┬─────┘
       │           │ Light    │           │
       │           │ Validator│           │
       │           └────┬─────┘           │
       │                │                 │
       ├────────────────┼─────────────────┤
       │                │                 │
  ┌────┴─────┐   ┌─────┴────┐    ┌──────┴─────┐
  │ Device D │   │ Device E │    │ Device F   │
  │ Edge IoT │   │ Edge IoT │    │ Edge IoT   │
  │ Light Val│   │ Light Val│    │ Light Val  │
  └──────────┘   └──────────┘    └────────────┘

  All connections are encrypted.
  Full validators participate in Ouroboros consensus (dedicated x86/ARM nodes).
  Light validators verify merkle proofs only (edge RISC-V nodes).
  Core/compute/domain nodes can host cubelets and run pods.
  Edge sensor nodes collect data + run light math cubelets + light validation.
  Authority Engine runs centralized on core server - all nodes call via RPC.

  See 10-node-topology-orchestrator-hierarchy.md for full topology.
```

### 2.2 Network Layers

```
NETWORK STACK (per device):

  ┌───────────────────────────────────┐
  │  Application Layer                │
  │  Pod cross-pod queries            │
  │  Cubelet data exchange            │
  │  Chain transactions               │
  │  Federated MCP tool calls         │
  ├───────────────────────────────────┤
  │  Protocol Layer (qudag-protocol)  │
  │  Message framing & routing        │
  │  Request/response patterns        │
  │  Stream multiplexing              │
  ├───────────────────────────────────┤
  │  Security Layer (qudag-crypto)    │
  │  Authentication (ML-DSA)          │
  │  Key exchange (ML-KEM / ECDH)     │
  │  Encryption (AES-256-GCM)         │
  ├───────────────────────────────────┤
  │  Transport Layer (qudag-network)  │
  │  LibP2P (TCP/QUIC/WebSocket)      │
  │  Peer discovery (DHT/mDNS)        │
  │  Connection management            │
  │  NAT traversal                    │
  └───────────────────────────────────┘
```

---

## 3. Device Discovery & Identity

### 3.1 ANS - Agent Name Service

Devices register their identity and capabilities in ANS (Agent Name Service), the OWASP-based secure registry.

```
ANS REGISTRATION (at boot, Step 9):

  device registers:
    device_id:          UUID (from hardware identity)
    public_key:         PostQuantumPublicKey (ML-KEM-768)
    signing_key:        PostQuantumSigningKey (ML-DSA)
    capabilities: [
        "cubelet.vision.object_detection",
        "cubelet.math.fft",
        "cubelet.llm.claude_sonnet",
        ...
    ]
    node_type:          full_validator | light_validator
    network_addresses:  [multiaddr]         // LibP2P multiaddresses
    hardware_profile: {
        arch:           "riscv64"
        cores:          4
        memory:         "4GB"
        crypto_accel:   true
    }
    authority_vector:   AuthorityVector     // current AV of the device-level entity
    boot_attestation:   Hash                // reference to on-chain boot attestation
```

### 3.2 Peer Discovery

Devices find each other using a combination of ANS and qudag-network:

```
DISCOVERY FLOW:

  1. BOOTSTRAP (first boot):
     - Device contacts bootstrap nodes (preconfigured addresses)
     - Bootstrap nodes provide initial peer list
     - Device joins the LibP2P DHT (distributed hash table)

  2. ANS LOOKUP:
     - Device queries ANS for peers with specific capabilities
     - Example: "Find devices with cubelet.vision.depth_estimation"
     - ANS returns matching device records with network addresses

  3. QUDAG-NETWORK DISCOVERY:
     - LibP2P mDNS for local network discovery (devices on same LAN)
     - LibP2P DHT for wide-area discovery
     - Peer exchange protocol (known peers share their peer lists)

  4. CONNECTION ESTABLISHMENT:
     - Device initiates connection to discovered peer
     - Mutual authentication (see Section 4)
     - Secure channel established
     - Peer added to device's active peer table

  5. CONTINUOUS DISCOVERY:
     - Periodic ANS refresh (re-register, discover new peers)
     - DHT walks for new peer discovery
     - Peer liveness checks (ping/pong)
     - Stale peers removed from active peer table
```

### 3.3 Capability-Based Peer Selection

When a System Orchestrator needs a cubelet from another device (the local pool doesn't have the required capability), it queries the network:

```
cross_device_cubelet_request:
  1. System Orch determines needed capability (not available locally)
  2. Query ANS: "Which devices have capability X?"
  3. ANS returns list of devices with capability X
  4. Filter by:
     a. Device is online and reachable
     b. Device's cubelet with capability X is available (not locked)
     c. Device's cubelet AV meets the threshold
  5. Select best device (highest AV, lowest latency, best compatibility)
  6. Request cubelet allocation from remote device's System Orch
  7. Remote System Orch approves/denies based on its own authority model
  8. If approved: cubelet is locked on remote device, communication channel established
```

---

## 4. Authentication & Encryption

### 4.1 Mutual Authentication

Every device-to-device connection requires mutual authentication.

```
AUTHENTICATION PROTOCOL:

  1. Device A initiates connection to Device B
  2. A sends: { device_id_A, public_key_A, nonce_A }
  3. B verifies A's identity against ANS registry
  4. B sends: { device_id_B, public_key_B, nonce_B, sign(nonce_A, private_key_B) }
  5. A verifies B's signature on nonce_A (proves B holds its private key)
  6. A sends: { sign(nonce_B, private_key_A) }
  7. B verifies A's signature on nonce_B (proves A holds its private key)
  8. Both parties verified → proceed to key exchange

SIGNATURE ALGORITHM: ML-DSA (post-quantum) via qudag-crypto
```

### 4.2 Key Exchange

After authentication, devices establish a shared session key.

```
KEY EXCHANGE:

  FOR CHAIN + AUTHENTICATION CHANNELS:
    Algorithm:  ML-KEM-768 (post-quantum key encapsulation)
    Via:        qudag-crypto
    Result:     256-bit shared secret
    Purpose:    Encrypt chain transactions, authority data, authentication tokens
    Rationale:  Long-lived secrets need quantum resistance

  FOR PIPELINE DATA CHANNELS:
    Algorithm:  X25519 (ECDH) - classical
    Via:        standard libp2p TLS
    Result:     256-bit shared secret
    Purpose:    Encrypt cubelet data in transit
    Rationale:  Ephemeral data, speed matters, classical is sufficient

  SESSION KEY ROTATION:
    Chain/auth:   Rotate every 1 hour or 1000 messages (whichever first)
    Data:         Rotate every 10 minutes or 10000 messages
    Rotation is seamless (new key negotiated before old one expires)
```

### 4.3 Encryption

```
DATA ENCRYPTION (all channels):
  Algorithm:    AES-256-GCM (authenticated encryption)
  Nonce:        96-bit, counter-based (deterministic, no random nonce issues)
  AAD:          Message header (authenticated but not encrypted)

MESSAGE INTEGRITY:
  Every message includes:
    - AES-GCM authentication tag (integrity + authenticity)
    - Sender's device_id (authenticated via AAD)
    - Sequence number (prevents replay attacks)
    - Timestamp (prevents message delay attacks)
```

---

## 5. Communication Channels

### 5.1 Channel Types

MYOS devices communicate over several logical channel types, all multiplexed over the same physical connection.

```
CHANNEL TYPES:

  CHAIN CHANNEL:
    Purpose:    Ouroboros consensus messages, block propagation, transaction submission
    Encryption: Post-quantum (ML-KEM-768 + AES-256-GCM)
    Priority:   High (consensus must be timely)
    Protocol:   qudag-protocol chain sub-protocol

  AUTHORITY CHANNEL:
    Purpose:    Cross-device authority queries, AV synchronization
    Encryption: Post-quantum
    Priority:   High
    Protocol:   qudag-protocol authority sub-protocol

  POD CHANNEL:
    Purpose:    Cross-pod communication (capability-based routing)
    Encryption: Classical (ECDH + AES-256-GCM)
    Priority:   Medium
    Protocol:   qudag-protocol pod sub-protocol

  DATA CHANNEL:
    Purpose:    Pipeline data transfer between cubelets on different devices
    Encryption: Classical (ECDH + AES-256-GCM)
    Priority:   Medium-High (latency sensitive)
    Protocol:   qudag-protocol data sub-protocol

  MCP CHANNEL:
    Purpose:    Federated MCP tool calls across devices
    Encryption: Classical
    Priority:   Medium
    Protocol:   Federated MCP protocol

  TELEMETRY CHANNEL:
    Purpose:    Health reports, resource usage, monitoring data
    Encryption: Classical
    Priority:   Low
    Protocol:   qudag-protocol telemetry sub-protocol
```

### 5.2 Channel Multiplexing

All channels run over a single physical connection using LibP2P stream multiplexing.

```
MULTIPLEXING:
  Transport:    LibP2P (TCP or QUIC)
  Multiplexer:  yamux or mplex
  Each channel type gets its own multiplexed stream
  Streams are independent (one blocked stream doesn't affect others)
  Priority is enforced at the application layer (higher priority channels
  are serviced first when bandwidth is constrained)
```

### 5.3 Fabric Thread Taxonomy

STF fabric threads are classified into five categories:

| Category | Threads | Purpose |
| -------- | ------- | ------- |
| **Social Infrastructure Protocols (SIPs)** | KnowledgeLedger, ResearchLedger, TalentLedger, EthosLedger, KarbonLedger | Core societal infrastructure — education, research, ethics, climate |
| **Sectoral Protocols** | AgroLedger, HealthLedger, EduLedger, CityLedger | Domain-specific verticals — agriculture, healthcare, education, smart cities |
| **Cross-Domain Bridges** | TokenBridge, IdentLedger, SchemaLedger | Identity, schema, and value transfer across domains |
| **Ethical and Cultural Threads** | EthosLedger, consent frameworks | Value alignment and cultural stewardship |
| **Resilience and Adaptation** | Stewardship maps, feedback loops | System health and adaptive governance |

Each thread is a logical stream multiplexed over the P2P channel (see Section 5). Thread.Sync Rubik's Moves ensure consistency across devices.

---

## 6. Cross-Device Pod Communication

### 6.1 Distributed Pods

A pod can span multiple devices. Some cubelets may be local, others remote.

```
DISTRIBUTED POD EXAMPLE:

  Device A (has vision cubelets):
    ┌─────────────────────┐
    │ [Camera Cubelet]    │──── data pipeline ────┐
    │ [Object Detector]   │                       │
    └─────────────────────┘                       │
                                                  │ (crosses network)
  Device B (has planning cubelets):               │
    ┌─────────────────────┐                       │
    │ [Path Planner]      │◄──────────────────────┘
    │ [Motor Controller]  │
    └─────────────────────┘

  Pod Orchestrator runs on one device (e.g., Device A).
  Data pipeline edges that cross devices use the DATA CHANNEL.
  Control messages cross devices via the POD CHANNEL.
```

### 6.2 Cross-Device Pipeline Edge

When a data pipeline edge crosses a device boundary:

```
CROSS-DEVICE EDGE:
  1. Producer cubelet (Device A) writes output to local shared memory
  2. MYOS network service on Device A:
     a. Reads the output
     b. Serializes (protobuf)
     c. Encrypts (AES-256-GCM via data channel)
     d. Sends over qudag-network to Device B
  3. MYOS network service on Device B:
     a. Receives and decrypts
     b. Deserializes
     c. Writes to local shared memory for consumer cubelet
  4. Consumer cubelet (Device B) reads input from local shared memory

LATENCY:
  Local edge (same device):   <1ms (shared memory, zero-copy)
  Cross-device edge:          10-100ms (serialization + network + deserialization)

  Pod Orchestrator accounts for cross-device latency in time budgets.
  Pipeline edges that cross devices have relaxed time budgets.
```

### 6.3 Cross-Device Authority

When a cubelet on Device A needs an authority check, and the Authority Engine runs on Device A:

- Local IPC, <5ms

When a cross-device operation needs authority validation:

- The requesting device's Authority Engine evaluates the local entity's authority
- The responding device's Authority Engine evaluates whether it allows the remote access
- Both must approve for the cross-device operation to proceed

```
CROSS-DEVICE AUTHORITY:
  Device A cubelet wants to use Device B's cubelet:
    1. Device A Authority Engine: "Does my cubelet have authority to make this request?"
       → If denied → operation blocked
    2. Device B Authority Engine: "Does this remote entity have authority on my device?"
       → If denied → operation blocked
    3. Both approved → operation proceeds

  This is DUAL AUTHORIZATION - both sides must agree.
```

---

## 7. Federated MCP Integration

### 7.1 Cross-Device Tool Coordination

Federated MCP enables cubelets on different devices to use each other's tools and capabilities as if they were local.

```
FEDERATED MCP:
  - Each device exposes its capabilities as MCP tools
  - Remote devices can discover and call these tools via MCP protocol
  - Tool calls are:
    a. Authenticated (device identity verified)
    b. Authority-checked (both sides)
    c. Encrypted (via MCP channel)
    d. Logged (off-chain)

EXAMPLE:
  Device A has a cubelet with "sensor.temperature" capability.
  Device B needs temperature data for its pod's task.

  Device B's Pod Orch → System Orch → Federated MCP →
    discovers Device A has "sensor.temperature" tool →
    calls the tool via MCP →
    receives temperature reading →
    feeds into Device B's pod pipeline
```

### 7.2 MCP Tool Registry

```
Each device maintains a local MCP tool registry:
  tool_registry {
      local_tools: [{
          tool_name:      String
          cubelet_id:     UUID
          description:    String
          input_schema:   SchemaDefinition
          output_schema:  SchemaDefinition
          authority_required: AuthorityVector
      }]
      remote_tools: [{       // discovered from other devices
          tool_name:      String
          device_id:      DeviceId
          latency_estimate: Duration
          last_seen:      Timestamp
      }]
  }
```

---

## 8. Network Resilience

### 8.1 Connection Failure Handling

```
PEER DISCONNECTION:
  1. Connection drops detected via LibP2P heartbeat
  2. Attempt reconnection (exponential backoff, max 5 retries)
  3. If reconnection fails:
     a. Peer marked as "unreachable" in peer table
     b. If cross-device pipeline edges are affected:
        - Pipeline data is buffered locally (bounded buffer)
        - Pod Orchestrator notified of degraded connectivity
        - If buffer fills → pipeline pauses → Pod Orch decides (wait or restructure)
     c. If consensus is affected (peer was a validator):
        - Ouroboros handles validator absence natively (BFT tolerance)
        - Chain continues as long as >2/3 validators are online
  4. Periodic reconnection attempts in background
  5. When peer returns → resync, replay buffered data, resume normal operation
```

### 8.2 Network Partition

```
NETWORK PARTITION (group of devices split from the main network):

  CHAIN BEHAVIOR:
    - Partitioned validators cannot reach consensus with the main group
    - Partitioned group forms a minority (cannot produce blocks if <1/3 of validators)
    - Decision logs are buffered locally on partitioned devices
    - When partition heals → chain state resynchronizes
    - Buffered decisions are submitted and ordered

  POD BEHAVIOR:
    - Distributed pods spanning the partition boundary are affected
    - Pod Orchestrator on one side loses contact with cubelets on the other
    - Pod Orch triggers Tier 2/3 failure response (quarantine unreachable cubelets, restructure)
    - System Orch may dissolve affected pods and reassemble with local cubelets only

  AUTHORITY BEHAVIOR:
    - Each device's Authority Engine operates independently
    - AV updates for cross-device events are buffered
    - When partition heals → AV states are reconciled
    - Reconciliation uses "latest timestamp wins" for concurrent updates
```

### 8.3 Offline Mode

```
OFFLINE MODE (single device, no network):
  - Device operates as a standalone MYOS system
  - Local cubelets only (no cross-device pods)
  - Authority Engine operates locally
  - Chain node buffers transactions (writes to local log)
  - When connectivity is restored:
    a. Rejoin P2P network
    b. Resync chain state
    c. Submit buffered transactions
    d. Re-register in ANS
    e. Resume cross-device operations
```

---

## 9. Bandwidth Management

### 9.1 Traffic Prioritization

```
PRIORITY ORDER (when bandwidth is constrained):

  1. Chain consensus messages       (must arrive on time for slot schedule)
  2. Safety-critical data           (emergency stops, safety alerts)
  3. Authority queries/updates      (actions are blocked without these)
  4. Pipeline data (active pods)    (ongoing task execution)
  5. Cross-pod queries              (can tolerate slight delay)
  6. MCP tool calls                 (can tolerate slight delay)
  7. Telemetry and monitoring       (can be delayed or sampled)
  8. Background sync                (peer discovery, state reconciliation)

ENFORCEMENT:
  Application-level traffic shaping via qudag-network
  Higher priority channels get bandwidth first
  Lower priority channels are throttled under congestion
```

### 9.2 Rate Limiting

```
rate_limiting {
    per_peer_max_bandwidth:     configurable (e.g., 10 MB/s)
    per_channel_limits: {
        chain:          "2 MB/s"
        authority:      "500 KB/s"
        data:           "5 MB/s"
        pod:            "1 MB/s"
        mcp:            "1 MB/s"
        telemetry:      "500 KB/s"
    }
    burst_allowance:    2x rate for 5 seconds
}
```

---

## 10. Configuration

```
communication_config {
    // P2P Network
    network {
        transport:              "quic"          // or "tcp"
        listen_addresses:       ["/ip4/0.0.0.0/udp/9000/quic"]
        bootstrap_nodes:        ["/ip4/x.x.x.x/udp/9000/quic/p2p/QmXXX"]
        max_peers:              50
        peer_discovery:         ["dht", "mdns"]
        connection_timeout_ms:  5000
        heartbeat_interval_ms:  10000
    }

    // ANS
    ans {
        registry_endpoint:      "ans://discovery.myos.network"
        registration_interval:  300             // re-register every 5 minutes
        capability_broadcast:   true
    }

    // Encryption
    crypto {
        chain_key_exchange:     "ml-kem-768"    // post-quantum
        data_key_exchange:      "x25519"        // classical
        symmetric_cipher:       "aes-256-gcm"
        chain_key_rotation_interval:    3600    // 1 hour
        data_key_rotation_interval:     600     // 10 minutes
    }

    // Channels
    channels {
        multiplexer:            "yamux"
        max_streams_per_peer:   64
    }

    // Resilience
    resilience {
        reconnect_max_retries:  5
        reconnect_base_delay_ms: 1000
        reconnect_max_delay_ms: 30000
        cross_device_buffer_size: "10MB"
        offline_mode_enabled:   true
    }

    // Federated MCP
    federated_mcp {
        enabled:                true
        tool_discovery_interval: 60             // seconds
        max_remote_tool_calls:  100             // per minute
    }

    // Bandwidth
    bandwidth {
        per_peer_max:           "10MB/s"
        priority_enforcement:   true
    }
}
```

---

## 11. Invariants (Must Always Hold)

```
INV-1:  All device-to-device connections are mutually authenticated
INV-2:  Chain and authority channels use post-quantum encryption (ML-KEM-768)
INV-3:  Data channels use classical encryption (X25519 + AES-256-GCM)
INV-4:  Cross-device operations require dual authorization (both devices approve)
INV-5:  Peer identity is verified against ANS before connection is accepted
INV-6:  Session keys are rotated before expiry (no expired key usage)
INV-7:  Network partition does not cause data loss (buffering + reconciliation)
INV-8:  Offline mode preserves all operations locally for later sync
INV-9:  Bandwidth prioritization ensures chain consensus messages are never starved
INV-10: Cross-device pipeline edges have relaxed time budgets accounting for network latency
INV-11: STF fabric threads are multiplexed across P2P channels
INV-12: Cross-device cubelet operations require dual authorization and CIG FEDERATES edge verification
INV-13: Federated MCP tool calls are authenticated, authority-checked, and logged
INV-14: Connection failures trigger automatic reconnection with exponential backoff
```

---

## 12. Interaction with Other Documents

- **Boot & Trust Chain (07):** Network services initialized at Step 9. Device registered in ANS. Peers discovered.
- **Runtime Infrastructure (08):** Network services run at P5 priority on the communication core. IPC handles cross-device data routing. Container cubelets' network egress is enforced by L7 HTTP policy (per-endpoint, per-method, per-path) and routed through a Privacy Router for LLM API calls — see doc 08 sections 3.5-3.6.
- **Authority Model (01):** Cross-device authority uses dual authorization. Post-quantum crypto for authority channels.
- **Pod Assembly (03):** Cross-device cubelet selection uses ANS for discovery and qudag-network for transport.
- **Cubelet I/O (04):** Cross-device pipeline edges add serialization + encryption overhead. Latency accounted for in time budgets.
- **Verification & Audit (06):** Chain consensus messages flow over the chain channel. Off-chain events from remote devices are synced via telemetry channel.
- **Master Document (00-master.md) - CIG & Lattice:** CIG FEDERATES edges define which cubelets can participate in cross-device pod operations. The network layer verifies FEDERATES edges before establishing cross-device pipeline connections.
- **STF Fabric Threads:** The 11+ named ledgers are multiplexed as separate logical streams across P2P channels, ensuring domain-specific data isolation at the network level. Each fabric thread's priority maps to the bandwidth management priority scheme.
- **STK Invariants:** Cross-device operations must satisfy STK invariants on both sides. Dual authorization includes STK invariant verification in addition to AV threshold checks.
- **Existing Repos:** qudag-network (P2P), qudag-protocol (framing), qudag-crypto (encryption), ANS (identity), Federated MCP (tool coordination), Agentic Edge Functions (edge agent deployment).
- **Knowledge Base (12-knowledge-base.md):** Cross-device KB queries (e.g., System KB lookups from domain servers, Domain KB queries from compute nodes) flow through the network stack. Edge sensor data feeds into the KB as sensor data entries with high decay rates, reflecting the ephemeral nature of real-time observations.
