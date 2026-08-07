### Composable Trust Specs

Composable Trust Specs is a collection of interoperable specifications for representing, encoding, hashing, identifying, and verifying machine-generated evidence across distributed and peer-to-peer systems.

The project is designed around a simple principle:

> **Trust should be composable from verifiable, content-addressed evidence rather than dependent on a single centralized authority.**

The initial work focuses on **Verifiable Telemetry Objects (VTOs)**: structured telemetry records that can be deterministically encoded, content-addressed, exchanged between peers, and used as building blocks for observability, diagnostics, reputation, policy, and trust systems.

This repository is an **initial working draft**. The schemas, encoding profiles, digest contexts, and verification semantics are expected to evolve through implementation and interoperability testing.

---

### Goals

Composable Trust Specs aims to provide common building blocks for:

* Verifiable telemetry and measurements
* Peer-to-peer observability
* Content-addressed telemetry
* Cross-implementation interoperability
* Deterministic serialization
* Cryptographic digests and CIDs
* Composable trust and reputation systems
* Machine-to-machine evidence exchange
* Distributed systems and agentic infrastructure

The specifications are intended to be **implementation-independent** and reusable across ecosystems.

---

### Architecture

The initial architecture separates telemetry semantics from encoding and cryptographic identity:

```text
┌───────────────────────────────────────────────┐
│              Application / System             │
│                                               │
│  libp2p • agents • distributed systems • etc. │
└───────────────────────┬───────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────┐
│          Verifiable Telemetry Object          │
│                     (VTO)                     │
│                                               │
│  Measurements • Producer • Network • Metadata │
└───────────────────────┬───────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────┐
│          Deterministic CBOR Encoding          │
└───────────────────────┬───────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────┐
│              Digest Context                   │
│                                               │
│       SHA-256 / Multihash / CID              │
└───────────────────────┬───────────────────────┘
                        │
                        ▼
┌───────────────────────────────────────────────┐
│        Verifiable / Content-Addressed         │
│                  Evidence                     │
└───────────────────────────────────────────────┘
```

The important design boundary is that **VTO defines the semantics of the telemetry**, while the digest context defines how the encoded representation becomes cryptographically identifiable.

This allows the digest and encoding mechanisms to be reused by other telemetry and evidence formats without coupling them to VTO-specific semantics.

---

### Verifiable Telemetry Object (VTO)

A VTO is a structured representation of telemetry produced by a peer, node, service, agent, or other measurement-producing system.

A VTO contains:

* A schema version
* A unique object identifier
* A timestamp
* Producer information
* Measurements
* Network context
* Optional metadata

The initial schema is intentionally small and implementation-friendly. Additional measurements and profiles can be introduced as the specification matures.

### 1. CDDL Schema

The current working schema is:

```cddl
vto = {
  version: uint,
  id: tstr,
  timestamp: uint,

  producer: {
    peer_id: tstr,
    implementation: tstr,
    version: tstr,
  },

  measurements: {
    latency_ms: float64,
    throughput_mbps: float64,
    bandwidth_mbps: float64,
    packet_loss_pct: float64,
    cpu_utilization_pct: float64,
    memory_utilization_pct: float64,
    peer_count: uint,
    active_streams: uint,
    uptime_seconds: uint,
  },

  network: {
    protocol: tstr,
    transport: tstr,
    region: tstr,
  },

  metadata: {
    tags: [* tstr],
    ?notes: tstr,
  }
}
```

The CDDL is a working draft and may change as implementation and interoperability testing progresses.

---

### 2. Example VTO

The following JSON representation is provided as a human-readable diagnostic representation.

JSON is **not** the canonical representation used for hashing or content addressing.

```json
{
  "version": 1,
  "id": "vto-000001",
  "timestamp": 1785926400,
  "producer": {
    "peer_id": "12D3KooWExamplePeer",
    "implementation": "go-libp2p",
    "version": "0.45.0"
  },
  "measurements": {
    "latency_ms": 14.37,
    "throughput_mbps": 843.52,
    "bandwidth_mbps": 921.11,
    "packet_loss_pct": 0.04,
    "cpu_utilization_pct": 41.8,
    "memory_utilization_pct": 63.2,
    "peer_count": 142,
    "active_streams": 51,
    "uptime_seconds": 87234
  },
  "network": {
    "protocol": "gossipsub",
    "transport": "quic-v1",
    "region": "ap-south-1"
  },
  "metadata": {
    "tags": [
      "observability",
      "libp2p",
      "telemetry"
    ]
  }
}
```

This representation is intended to make the structure easy to inspect, debug, and test.

Implementations MUST NOT use JSON serialization for the canonical digest.

---

### 3. Canonical Encoding

The initial VTO profile uses **deterministic CBOR** as its canonical serialization.

The canonicalization boundary is:

```text
VTO
 │
 ▼
Deterministic CBOR
 │
 ▼
Digest
 │
 ▼
Multihash
 │
 ▼
CID
```

This provides a deterministic byte representation suitable for cryptographic hashing and content addressing.

The actual encoded bytes should be generated directly by the implementation rather than manually constructed or copied from a diagnostic representation.

For example:

```text
Base64:
<implementation-generated deterministic CBOR bytes>
```

or:

```text
Hex:
<implementation-generated deterministic CBOR bytes>
```

Implementations SHOULD include test vectors containing:

1. The logical VTO object
2. The deterministic CBOR bytes
3. The resulting digest
4. The resulting CID

This will allow implementations to validate interoperability.

---

### 4. Digest

The current draft uses:

```text
SHA-256
```

as the initial digest algorithm.

The digest MUST be computed over the deterministic CBOR serialization:

```text
digest = SHA-256(deterministic_cbor(vto))
```

The resulting digest can then be represented as a multihash and mapped to a CID.

Example:

```text
Digest:
<implementation-generated SHA-256 digest>
```

```text
CID:
<implementation-generated CID>
```

The exact CID representation will be finalized alongside the digest-context specification.

---

### 5. Digest Context

The digest context defines the boundary between the logical VTO and its cryptographic identity.

The current design requirements are:

* Operate directly on deterministic CBOR
* Avoid JSON canonicalization
* Support native CBOR numeric types
* Produce stable multihashes
* Map naturally to CIDs
* Remain independent of VTO semantics
* Be reusable by other CBOR-based telemetry formats
* Provide deterministic interoperability test vectors

The digest context should be treated as a reusable primitive rather than as a VTO-specific implementation detail.

This allows future specifications to use the same mechanism:

```text
Other Evidence Object
        │
        ▼
   Deterministic CBOR
        │
        ▼
    Digest Context
        │
        ▼
     Multihash
        │
        ▼
        CID
```

---

### 6. Floating-Point Fields

The following fields currently use IEEE-754 binary64 floating-point values.

| Field                    | Type    | Units        | Source                   |
| ------------------------ | ------- | ------------ | ------------------------ |
| `latency_ms`             | float64 | milliseconds | measured RTT             |
| `throughput_mbps`        | float64 | Mbps         | moving average           |
| `bandwidth_mbps`         | float64 | Mbps         | estimated link bandwidth |
| `packet_loss_pct`        | float64 | %            | transport statistics     |
| `cpu_utilization_pct`    | float64 | %            | host metrics             |
| `memory_utilization_pct` | float64 | %            | host metrics             |

Floating-point handling is an important part of the canonicalization profile because even apparently equivalent numeric values can result in different byte representations if implementations perform implicit conversions.

The initial profile therefore requires:

* IEEE-754 binary64 encoding
* Native floating-point representation in CBOR
* No conversion to decimal strings
* Deterministic CBOR encoding
* Digest computation directly over canonical CBOR bytes
* No JSON canonicalization step

### NaN and signed zero

The treatment of exceptional floating-point values is still under specification.

In particular, the final profile needs to define whether:

* NaN values are permitted
* NaN payloads must be preserved
* `+0.0` and `-0.0` must remain distinguishable
* infinities are permitted

Implementations should not introduce additional normalization rules until these behaviors are explicitly defined by the profile.

---

### 7. Integer Fields

The current integer fields are:

| Field            | Type |
| ---------------- | ---- |
| `version`        | uint |
| `timestamp`      | uint |
| `peer_count`     | uint |
| `active_streams` | uint |
| `uptime_seconds` | uint |

The timestamp is currently represented as an unsigned integer and is expected to use Unix epoch seconds.

The timestamp representation and precision may be revisited as the specification develops.

---

### 8. Producer Identity

The `producer` section identifies the implementation that generated the telemetry.

```cddl
producer: {
  peer_id: tstr,
  implementation: tstr,
  version: tstr,
}
```

For peer-to-peer systems, `peer_id` provides an important association between the telemetry object and the producing peer.

However, a producer identifier alone does **not** establish that the telemetry is authentic.

Future profiles may define mechanisms for:

* Signing VTOs
* Binding telemetry to cryptographic peer identities
* Attesting measurements
* Establishing provenance
* Verifying the measurement environment

These mechanisms are intentionally separate from the initial serialization and digest work.

---

### 9. Network Context

The `network` section describes the context in which the measurements were produced.

```cddl
network: {
  protocol: tstr,
  transport: tstr,
  region: tstr,
}
```

Example:

```json
{
  "protocol": "gossipsub",
  "transport": "quic-v1",
  "region": "ap-south-1"
}
```

The current fields are intentionally descriptive. Future profiles may define controlled vocabularies or registries for protocols and transports.

---

### 10. Metadata

Metadata provides optional context without changing the core measurement model.

```cddl
metadata: {
  tags: [* tstr],
  ?notes: tstr,
}
```

Tags can be used to associate VTOs with applications, environments, experiments, or observability workflows.

For example:

```json
{
  "tags": [
    "observability",
    "libp2p",
    "telemetry"
  ]
}
```

Metadata SHOULD NOT be used to introduce security-critical semantics without an explicit specification.

---

### 11. Verification Model

The initial VTO design separates **integrity** from **authenticity**.

A digest provides a stable identifier for the exact encoded object:

```text
VTO
 │
 ▼
Deterministic CBOR
 │
 ▼
SHA-256
 │
 ▼
Multihash / CID
```

This establishes content integrity: a verifier can determine whether the bytes correspond to the referenced object.

It does not, by itself, establish:

* Who produced the object
* Whether the producer is trusted
* Whether the measurement is accurate
* Whether the measurement environment is trustworthy
* Whether the producer intentionally reported false data

These concerns belong to higher-level trust and verification profiles.

This separation is fundamental to the composable design of the project.

---

### 12. From Telemetry to Composable Trust

VTOs are intended to serve as primitive evidence objects rather than as complete trust decisions.

A higher-level system could compose multiple independently verifiable objects:

```text
             ┌─────────────┐
             │    VTO #1   │
             └──────┬──────┘
                    │
             ┌──────▼──────┐
             │    VTO #2   │
             └──────┬──────┘
                    │
             ┌──────▼──────┐
             │    VTO #3   │
             └──────┬──────┘
                    │
                    ▼
          ┌───────────────────┐
          │ Evidence / Trust  │
          │     Policy        │
          └─────────┬─────────┘
                    │
                    ▼
          ┌───────────────────┐
          │ Decision / Action  │
          └───────────────────┘
```

This enables future systems to build trust from multiple sources of evidence without requiring every participant to use the same centralized trust authority.

Potential applications include:

* P2P network observability
* Network performance assessment
* Agent-to-agent coordination
* Distributed reputation
* Service-level evidence
* Routing and peer selection
* Security monitoring
* Infrastructure diagnostics
* Decentralized agent systems

---

### 13. Implementation Status

This repository currently contains an **initial working draft**.

| Component                       | Status              |
| ------------------------------- | ------------------- |
| VTO conceptual model            | 🧪 Working draft    |
| CDDL schema                     | 🧪 Working draft    |
| Deterministic CBOR profile      | 🧪 In progress      |
| Digest context                  | 🧪 In progress      |
| SHA-256 profile                 | 🧪 Initial proposal |
| Multihash representation        | 🧪 In progress      |
| CID representation              | 🧪 In progress      |
| Canonical test vectors          | ⏳ Planned           |
| Cross-language interoperability | ⏳ Planned           |
| VTO signatures                  | ⏳ Future work       |
| Provenance / attestation        | ⏳ Future work       |
| Trust / reputation profiles     | ⏳ Future work       |

The specification should be considered experimental until interoperability tests and implementation feedback have been incorporated.

---

### 14. Interoperability and Test Vectors

Interoperability is a primary goal of this repository.

Implementations should eventually be able to take the same logical VTO and independently produce:

```text
Identical VTO
      │
      ├── Implementation A
      │
      ├── Implementation B
      │
      └── Implementation C
               │
               ▼
      Identical deterministic CBOR
               │
               ▼
         Identical digest
               │
               ▼
          Identical CID
```

The repository will therefore include canonical test vectors as the implementation matures.

A test vector should contain:

```text
- Input VTO
- Deterministic CBOR bytes
- Digest algorithm
- Digest bytes
- Multihash
- CID
```

These vectors will provide a common interoperability target for implementations in different languages and ecosystems.

---

# 15. Relationship to libp2p

VTOs are particularly relevant to peer-to-peer systems where telemetry can be generated at the edge and exchanged between independently operated peers.

The initial examples use libp2p terminology such as:

* Peer IDs
* GossipSub
* QUIC
* Peer counts
* Active streams

However, the VTO specification is **not intended to be libp2p-specific**.

The format should remain useful for other peer-to-peer networks, distributed systems, agent networks, and infrastructure platforms.

libp2p provides an important initial implementation and experimentation environment, while the specification is intended to remain broadly reusable.

---

### 16. Design Principles

The project currently follows these principles:

### Determinism

The same logical object should result in the same canonical byte representation.

### Content Addressability

Evidence should be independently identifiable through cryptographic content addressing.

### Composability

Small verifiable objects should be usable as building blocks for larger trust systems.

### Separation of Concerns

Telemetry semantics, serialization, cryptographic identity, authenticity, and trust decisions should remain independently specified.

### Interoperability

The specification should be implementable across languages and ecosystems without relying on a particular implementation.

### Minimalism

The core object should remain small enough to be generated and exchanged efficiently at the edge.

### Extensibility

Future profiles should be able to add capabilities without requiring the core model to become tightly coupled to a particular application.

---

### 17. Current Work

The immediate implementation priorities are:

1. Finalize the VTO CDDL.
2. Implement deterministic CBOR serialization.
3. Generate canonical CBOR test vectors.
4. Define the digest context.
5. Validate SHA-256 → multihash → CID interoperability.
6. Establish cross-language test vectors.
7. Clarify floating-point canonicalization.
8. Define optional authenticity and signing profiles.
9. Explore provenance and attestation mechanisms.
10. Develop higher-level trust composition profiles.

The initial digest-context work should be sufficiently self-contained for implementation while the broader VTO specification continues to evolve.

---

### Contributing

This repository is an open working space for developing composable specifications for verifiable evidence and trust.

Contributions are welcome across:

* Specification design
* CDDL
* CBOR encoding
* Digest contexts
* Multihash and CID integration
* Test vectors
* Interoperability testing
* Implementations
* Security analysis
* Formalization
* Trust and reputation models

When proposing changes, please distinguish between:

1. **Wire-format requirements**
2. **Canonicalization requirements**
3. **Cryptographic requirements**
4. **Semantic requirements**
5. **Application-specific behavior**

Keeping these layers separate is important for maintaining composability and interoperability.

---

### Status

**Experimental / Initial Working Draft**

The contents of this repository should not yet be considered a finalized standard.

The specifications will evolve through implementation, interoperability testing, security review, and community feedback.


