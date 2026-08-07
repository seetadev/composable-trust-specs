# Verifiable Telemetry Objects (VTO)

## Abstract

This document defines the Verifiable Telemetry Object (VTO), a structured, content-addressable representation of telemetry produced by peers, nodes, services, and distributed systems.

VTO provides a common telemetry artifact that can be deterministically encoded using CBOR and subsequently referenced by a cryptographic digest. The specification defines the VTO data model, field semantics, CBOR representation, floating-point handling requirements, and the interface between the telemetry object and a CBOR-specific digest context.

VTO is designed to compose with generic trust and evidence mechanisms without defining a trust decision itself. In particular, the VTO telemetry artifact can be bound to a generic Composable Proof Binding (CPB) layer through a typed digest reference. An Agent Attestation Composition (AAC) is described as an example composition.

This document intentionally separates telemetry semantics from cryptographic binding: VTO defines what is being measured and represented; the underlying digest context defines how the encoded representation is identified and verified. Neither layer subsumes the other.

This document is an initial working draft and is intended to support implementation, interoperability testing, and further discussion within the IETF community.

---

## Status of This Memo

This document is an Internet-Draft and is in full conformance with all provisions of BCP 78 and BCP 79.

Internet-Drafts are working documents of the Internet Engineering Task Force (IETF). Note that other groups may also distribute working documents as Internet-Drafts. The list of current Internet-Drafts is at [https://datatracker.ietf.org/drafts/current/](https://datatracker.ietf.org/drafts/current/).

Internet-Drafts are draft documents valid for a maximum of six months and may be updated, replaced, or obsoleted by other documents at any time. It is inappropriate to use Internet-Drafts as reference material or to cite them other than as "work in progress."

---

## 1. Introduction

Distributed systems increasingly depend on telemetry to make decisions about network performance, peer selection, service health, resource utilization, and system behavior.

In centralized systems, telemetry is commonly associated with a trusted collection point. In decentralized and peer-to-peer systems, telemetry may instead be generated independently by many peers and exchanged across administrative and organizational boundaries.

This creates a requirement for a telemetry representation that is:

* structured;
* implementation-independent;
* deterministic to encode;
* independently identifiable;
* suitable for content addressing;
* composable with evidence and trust mechanisms; and
* usable across heterogeneous implementations.

The Verifiable Telemetry Object (VTO) addresses this requirement by defining a compact telemetry artifact whose canonical representation can be encoded using deterministic CBOR and subsequently identified through a cryptographic digest.

VTO deliberately does not attempt to define a complete trust framework. A cryptographic digest establishes the identity of an encoded object, but does not establish that the producer is trustworthy or that the reported measurement is correct. Authenticity, provenance, attestation, reputation, and policy are therefore treated as separate concerns.

The resulting architecture is based on two layers with a typed join:

```text
                 ┌─────────────────────────────┐
                 │      Trust / Evidence       │
                 │        Applications          │
                 └──────────────┬──────────────┘
                                │
                                │
                 ┌──────────────▼──────────────┐
                 │             AAC              │
                 │   Worked Composition Model  │
                 └──────────────┬──────────────┘
                                │
                                │ typed digest reference
                                │
                 ┌──────────────▼──────────────┐
                 │             CPB              │
                 │ Generic Binding Mechanism    │
                 └──────────────┬──────────────┘
                                │
                                │
                 ┌──────────────▼──────────────┐
                 │             VTO              │
                 │    Telemetry Artifact       │
                 └──────────────┬──────────────┘
                                │
                                │
                 ┌──────────────▼──────────────┐
                 │       Deterministic CBOR     │
                 └──────────────┬──────────────┘
                                │
                                ▼
                         Digest Context
```

The organizing principle is:

> **Two layers, one join, neither absorbs the other.**

VTO defines telemetry semantics. CPB provides a generic binding mechanism. The typed digest reference joins the two without requiring CPB to understand VTO-specific telemetry semantics.

---

## 2. Conventions and Terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in BCP 14 when, and only when, they appear in all capitals.

### 2.1. Verifiable Telemetry Object

A **Verifiable Telemetry Object (VTO)** is a structured representation of measurements produced by a telemetry producer.

### 2.2. Producer

A **producer** is the peer, node, service, or implementation that generates a VTO.

### 2.3. Canonical Representation

The **canonical representation** is the deterministic CBOR byte sequence over which the digest is calculated.

### 2.4. Digest Context

A **digest context** defines the encoding and interpretation rules necessary to calculate a cryptographic digest over a specific data representation.

This document specifies requirements for a CBOR digest context but does not define the complete digest-context algorithm.

### 2.5. Typed Digest Reference

A **typed digest reference** identifies the representation and digest context used to bind an encoded VTO to an external evidence or trust mechanism.

### 2.6. CPB

**Composable Proof Binding (CPB)** is a generic binding layer that allows evidence objects to be associated with higher-level trust and verification mechanisms.

CPB is outside the normative scope of this document except where required to describe the VTO composition model.

### 2.7. AAC

**Agent Attestation Composition (AAC)** is a worked composition example demonstrating how VTO telemetry can be associated with a higher-level attestation or evidence structure.

AAC is not required for generation or verification of a VTO.

---

## 3. Design Goals

VTO has the following design goals:

1. Provide a compact representation of telemetry.
2. Allow telemetry generated by different implementations to be interoperable.
3. Provide a deterministic byte representation suitable for cryptographic hashing.
4. Support floating-point measurements without requiring their conversion to strings.
5. Permit content-addressable identification.
6. Separate telemetry semantics from cryptographic binding.
7. Permit VTO to be composed with generic evidence and trust mechanisms.
8. Remain independent of any particular peer-to-peer implementation.

VTO does not attempt to:

* establish that telemetry is truthful;
* establish producer trustworthiness;
* define a reputation system;
* define an attestation architecture;
* define a trust policy;
* mandate a particular transport protocol; or
* define CPB semantics.

---

## 4. VTO Data Model

A VTO consists of five top-level components:

* `version`
* `id`
* `timestamp`
* `producer`
* `measurements`
* `network`
* `metadata`

The initial VTO schema is expressed using CDDL [RFC8610].

### 4.1. CDDL

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

The schema is an initial working definition. Future revisions MAY introduce optional fields or additional profiles.

Implementations MUST NOT interpret fields introduced by a future VTO version as having semantics defined by this document unless those semantics are explicitly specified.

---

## 5. VTO Field Semantics

### 5.1. version

`version` identifies the VTO schema version.

The value is an unsigned integer.

### 5.2. id

`id` provides an application-level identifier for the VTO.

The `id` field is distinct from the cryptographic digest or CID associated with the encoded VTO. Implementations MUST NOT assume that `id` provides cryptographic integrity.

### 5.3. timestamp

`timestamp` identifies the time at which the telemetry represented by the VTO was generated.

The initial profile represents the timestamp as an unsigned integer using Unix epoch seconds.

Future profiles MAY define additional timestamp precision.

### 5.4. producer

The `producer` structure identifies the implementation that generated the telemetry.

```cddl
producer: {
  peer_id: tstr,
  implementation: tstr,
  version: tstr,
}
```

`peer_id` identifies the producing peer where a peer identity is available.

`implementation` identifies the software implementation producing the VTO.

`version` identifies the implementation version.

Producer information is descriptive and does not by itself provide cryptographic authentication.

---

## 6. Measurements

The `measurements` structure contains the telemetry values represented by the VTO.

```cddl
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
}
```

### 6.1. Latency

`latency_ms` represents measured latency in milliseconds.

The measurement basis SHOULD be documented by the producer when the underlying protocol permits multiple latency definitions.

### 6.2. Throughput

`throughput_mbps` represents measured or calculated throughput in megabits per second.

Where the value represents an average or moving average, the measurement interval SHOULD be available through an associated profile or metadata field.

### 6.3. Bandwidth

`bandwidth_mbps` represents an estimate of available or observed link bandwidth in megabits per second.

The estimation method is implementation-specific unless constrained by a future profile.

### 6.4. Packet Loss

`packet_loss_pct` represents packet loss as a percentage.

The observation interval and measurement methodology are outside the scope of the base VTO definition.

### 6.5. CPU Utilization

`cpu_utilization_pct` represents host CPU utilization as a percentage.

### 6.6. Memory Utilization

`memory_utilization_pct` represents host memory utilization as a percentage.

### 6.7. Peer Count

`peer_count` represents the number of peers observed by the producer at the time associated with the measurement.

### 6.8. Active Streams

`active_streams` represents the number of active streams observed by the producer.

### 6.9. Uptime

`uptime_seconds` represents the producer's uptime in seconds.

---

## 7. Network Context

The `network` structure describes the environment in which the telemetry was collected.

```cddl
network: {
  protocol: tstr,
  transport: tstr,
  region: tstr,
}
```

`protocol` identifies the relevant application or networking protocol.

`transport` identifies the transport used by the relevant communication.

`region` provides deployment or geographic context.

The base VTO specification does not define controlled vocabularies for these values.

Implementations MAY define additional profiles that establish registries or controlled vocabularies.

---

## 8. Metadata

The `metadata` structure provides additional non-core information.

```cddl
metadata: {
  tags: [* tstr],
  ?notes: tstr,
}
```

`tags` is a list of textual labels.

`notes` is an optional textual description.

Metadata MUST NOT be interpreted as establishing security, trust, or attestation semantics unless such semantics are explicitly defined by another specification.

---

## 9. Canonical CBOR Representation

A VTO intended for digesting MUST be represented using deterministic CBOR.

The canonicalization process is:

```text
VTO data model
      |
      v
Deterministic CBOR
      |
      v
Digest Context
      |
      v
Digest
      |
      v
Multihash / CID
```

JSON is provided only as a diagnostic and human-readable representation.

JSON canonicalization MUST NOT be used as an intermediate step when calculating the VTO digest.

An implementation MUST calculate the digest directly over the canonical CBOR byte sequence.

---

## 10. Floating-Point Encoding

The VTO data model intentionally permits floating-point measurements.

The initial profile uses IEEE-754 binary64 values for floating-point fields.

The following fields are currently floating-point fields:

| Field                    | Representation    | Units        | Measurement Basis               |
| ------------------------ | ----------------- | ------------ | ------------------------------- |
| `latency_ms`             | IEEE-754 binary64 | milliseconds | measured RTT                    |
| `throughput_mbps`        | IEEE-754 binary64 | Mbps         | measured or averaged throughput |
| `bandwidth_mbps`         | IEEE-754 binary64 | Mbps         | estimated link bandwidth        |
| `packet_loss_pct`        | IEEE-754 binary64 | percent      | transport statistics            |
| `cpu_utilization_pct`    | IEEE-754 binary64 | percent      | host metrics                    |
| `memory_utilization_pct` | IEEE-754 binary64 | percent      | host metrics                    |

A VTO implementation:

* MUST preserve the binary floating-point value when encoding the VTO;
* MUST NOT convert floating-point values to decimal strings;
* MUST use deterministic CBOR;
* MUST calculate the digest over the resulting CBOR bytes; and
* MUST NOT introduce an intermediate JSON representation.

### 10.1. Exceptional Floating-Point Values

The treatment of NaN, infinities, and signed zero requires further specification.

In particular, the digest context MUST define whether:

* NaN values are permitted;
* NaN payloads are preserved;
* positive and negative zero are distinct;
* positive and negative infinity are permitted; and
* any floating-point normalization is applied before hashing.

Until the digest context defines these behaviors, implementations SHOULD avoid emitting exceptional floating-point values.

The final digest-context specification will determine the normative treatment of these values.

---

## 11. Integer Encoding

The following fields are represented as unsigned integers:

| Field            | Type |
| ---------------- | ---- |
| `version`        | uint |
| `timestamp`      | uint |
| `peer_count`     | uint |
| `active_streams` | uint |
| `uptime_seconds` | uint |

Implementations MUST use the CBOR integer representation corresponding to the value represented by the field.

Implementations MUST NOT encode integer values as strings.

---

## 12. Digest Boundary

The cryptographic digest boundary is explicitly defined as the deterministic CBOR representation of the VTO.

Conceptually:

```text
digest = H(deterministic_cbor(VTO))
```

where `H` is selected by the applicable digest context.

This document does not define a single mandatory digest algorithm.

The digest context is responsible for specifying:

1. The serialization format.
2. The canonicalization rules.
3. Numeric encoding requirements.
4. Floating-point behavior.
5. The digest algorithm.
6. The resulting multihash representation.
7. Any CID mapping requirements.

The digest context MUST operate on the CBOR representation rather than on the logical JSON representation.

---

## 13. Typed Digest Reference

The digest is the join between the VTO telemetry artifact and higher-level evidence or trust systems.

A typed digest reference identifies at least:

* the object representation;
* the applicable digest context; and
* the resulting digest.

Conceptually:

```text
              VTO
               |
               v
       Deterministic CBOR
               |
               v
        Digest Context
               |
               v
       Typed Digest Reference
               |
               v
              CPB
               |
               v
       Evidence / Trust System
```

The typed digest reference allows a generic binding layer to refer to a VTO without requiring the binding layer to interpret VTO measurement semantics.

This separation is intentional.

---

## 14. Relationship to CPB

CPB provides a generic mechanism for binding evidence to other objects or trust systems.

VTO is a telemetry-specific artifact.

The two specifications therefore operate at different layers.

```text
+------------------------------------+
|        Trust / Evidence Layer      |
+------------------------------------+
                 |
                 |
+------------------------------------+
|               CPB                  |
|       Generic Binding Layer        |
+------------------------------------+
                 |
          Typed Digest Reference
                 |
+------------------------------------+
|               VTO                  |
|       Telemetry Artifact           |
+------------------------------------+
                 |
+------------------------------------+
|        Deterministic CBOR          |
+------------------------------------+
```

CPB MUST NOT need to understand the individual semantics of VTO measurements to bind a VTO object.

Conversely, VTO does not define CPB semantics.

This separation permits the same telemetry artifact to be used by multiple evidence and trust mechanisms.

---

## 15. Worked Composition: AAC

An Agent Attestation Composition (AAC) can use VTO as a telemetry artifact.

A conceptual composition is:

```text
+-----------------------------+
| Agent / Peer                |
+--------------+--------------+
               |
               | produces
               v
+-----------------------------+
| VTO                         |
| telemetry artifact          |
+--------------+--------------+
               |
               | deterministic CBOR
               v
+-----------------------------+
| Digest Context              |
+--------------+--------------+
               |
               | typed digest
               v
+-----------------------------+
| CPB                         |
| generic proof binding       |
+--------------+--------------+
               |
               v
+-----------------------------+
| AAC                         |
| attestation composition     |
+-----------------------------+
```

AAC is included as a composition example and does not change the VTO data model.

The AAC specification MAY define additional authenticity, provenance, or attestation semantics.

---

## 16. Example VTO

The following example illustrates the logical structure of a VTO.

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

This JSON representation is illustrative only.

The canonical digest MUST be calculated from the deterministic CBOR representation generated from the VTO data model.

---

## 17. Example Encoded Object

An implementation conforming to this document SHOULD provide a test vector containing the exact CBOR bytes emitted for the example above.

For example:

```text
CBOR (hex):

<implementation-generated deterministic CBOR bytes>
```

or:

```text
CBOR (base64):

<implementation-generated deterministic CBOR bytes>
```

The test vector SHOULD additionally contain the digest:

```text
Digest:

<implementation-generated digest>
```

and, where a CID mapping is defined:

```text
CID:

<implementation-generated CID>
```

The actual byte sequence MUST be generated by the implementation rather than manually constructed.

---

## 18. Interoperability Requirements

Interoperability is a primary objective of the VTO specification.

Two independent implementations receiving the same VTO data SHOULD produce identical deterministic CBOR bytes.

Consequently, when using the same digest context, independent implementations SHOULD produce identical digests and content identifiers.

An interoperability test vector SHOULD contain:

* the logical VTO;
* the deterministic CBOR byte sequence;
* the digest context identifier;
* the digest algorithm;
* the digest;
* the multihash; and
* the CID, where applicable.

A future version of this document SHOULD define a normative set of interoperability vectors.

---

## 19. Security Considerations

A VTO digest establishes integrity of the represented bytes but does not establish the correctness or truthfulness of the telemetry.

For example, a malicious producer can generate a perfectly valid VTO containing false measurements. The resulting object can still have a valid cryptographic digest.

Applications MUST therefore distinguish between:

* **integrity** — whether the referenced bytes have changed;
* **authenticity** — whether the object originated from the claimed producer;
* **provenance** — how and where the measurement was produced;
* **measurement validity** — whether the measurement accurately reflects the underlying system; and
* **trust** — whether an application should rely upon the measurement.

A future VTO profile MAY define digital signatures or other producer-authentication mechanisms.

Such mechanisms are intentionally outside the base VTO representation.

### 19.1. Replay

Telemetry can become stale or be replayed by another participant.

Applications using VTO SHOULD consider:

* timestamps;
* freshness windows;
* sequence numbers;
* producer identity; and
* application-specific replay protection.

The base VTO specification does not define a universal replay-prevention mechanism.

### 19.2. Measurement Manipulation

A producer can manipulate host-level measurements before generating a VTO.

Cryptographic encoding does not prevent this behavior.

Where stronger guarantees are required, an application MAY combine VTO with hardware attestation, execution-environment attestation, signed measurement sources, or other provenance mechanisms.

### 19.3. Privacy

Telemetry can reveal information about:

* network topology;
* geographic deployment;
* peer relationships;
* resource utilization;
* service activity; and
* operational behavior.

Implementations SHOULD consider whether fields such as `peer_id`, `region`, and `notes` disclose sensitive operational information.

Applications SHOULD avoid placing unnecessary sensitive information in VTO metadata.

---

## 20. IANA Considerations

This document has no IANA actions at this time.

A future version MAY define IANA registries for:

* VTO versions;
* measurement identifiers;
* protocol identifiers;
* transport identifiers;
* digest contexts; and
* typed digest reference profiles.

Such registries should only be introduced once interoperability requirements are sufficiently stable.

---

## 21. Open Issues

The following topics remain open in this working draft:

1. Final VTO versioning semantics.
2. Exact timestamp precision.
3. Float handling for NaN and infinities.
4. Signed-zero handling.
5. Exact CBOR deterministic encoding profile.
6. Digest context identifier.
7. Digest algorithm registry.
8. Multihash mapping.
9. CID mapping.
10. Producer authentication.
11. Provenance and attestation.
12. Measurement methodology profiles.
13. Controlled vocabularies for protocols and transports.
14. Extension mechanisms.
15. Normative interoperability test vectors.

These issues should be resolved through implementation and interoperability testing rather than being prematurely constrained by the initial schema.

---

## 22. Implementation and Deployment Considerations

The initial VTO implementation is expected to be exercised in peer-to-peer networking environments, including libp2p implementations.

The implementation experience should focus on:

* generating VTOs from real telemetry;
* preserving native floating-point values;
* deterministic CBOR serialization;
* generating reproducible digests;
* producing multihash and CID representations;
* exchanging VTOs between peers; and
* validating objects generated by different implementations.

Implementations SHOULD expose test-vector generation as part of their development workflow.

A real encoded VTO instance generated by an implementation is particularly important for validating the digest-context specification because it establishes the actual byte-level representation produced by deployed software.

---

## 23. Relationship to Existing CBOR Work

The VTO encoding model builds on CBOR and deterministic encoding.

The digest context is intentionally separated from the VTO semantic specification because the requirements for digesting a CBOR object can differ from the requirements for representing a telemetry object.

In particular, VTO measurements include floating-point values. A digest context that prohibits floating-point values would therefore be unsuitable for the VTO representation.

The VTO work consequently provides the concrete telemetry structure and encoded examples required to define a CBOR-specific digest context.

The digest context itself is expected to be specified separately and may be reusable for CBOR-based objects other than VTO.

---

## 24. IETF MAPRG Discussion

This document is intended as a starting point for discussion within the IETF Measurement Analysis for Protocols Research Group (MAPRG).

The proposed work focuses on the telemetry layer and its deterministic representation.

The intended composition is:

```text
VTO
  |
  | telemetry artifact
  |
  +----> typed digest reference ----> CPB
                                      |
                                      +----> AAC
```

The document does not attempt to make VTO a complete trust or attestation framework.

The authors welcome implementation feedback, measurement methodology discussion, interoperability testing, and review of the proposed separation between telemetry semantics and generic evidence binding.

---

## 25. Future Work

Future versions of this document may address:

* VTO signatures;
* authenticated telemetry;
* measurement provenance;
* hardware-backed measurements;
* trusted execution environments;
* telemetry aggregation;
* streaming VTOs;
* VTO chaining;
* temporal relationships between VTOs;
* peer-to-peer VTO exchange;
* measurement methodology profiles;
* privacy-preserving telemetry;
* selective disclosure;
* richer CID and multihash profiles; and
* integration with additional evidence and trust frameworks.

These mechanisms should build on the base VTO artifact without unnecessarily coupling telemetry semantics to a particular trust architecture.

---

## 26. References

### 26.1. Normative References

[RFC8610]

Birkholz, H., Vigano, C., and C. Bormann, "Concise Data Definition Language (CDDL): A Notational Convention to Express CBOR Data Structures", RFC 8610.

[RFC8949]

Bormann, C. and P. Hoffman, "Concise Binary Object Representation (CBOR)", RFC 8949.

### 26.2. Informative References

[CBOR42]

Caballero, et al., "CBOR-42", Internet-Draft.

[CPB]

Mih, S., et al., "Composable Proof Binding", Internet-Draft, work in progress.

[AAC]

Mih, S., et al., "Agent Attestation Composition", work in progress.

---

## Appendix A. Digest and CID Mapping

This appendix is informative.

A VTO can be represented as a content-addressed object using a digest context and a multihash/CID representation.

The conceptual mapping is:

```text
VTO
 |
 | deterministic CBOR
 v
CBOR bytes
 |
 | digest context
 v
Digest
 |
 | multihash
 v
Multihash
 |
 | CID codec
 v
CID
```

For example:

```text
Digest algorithm:
SHA-256

Digest:
<digest bytes>

Multihash:
<multihash>

CID:
<CID>
```

The exact multihash and CID parameters are implementation/profile dependent until standardized by the applicable digest context.

This appendix does not define a new digest algorithm or CID codec.

---

## Appendix B. Initial Test Vector Structure

This appendix is informative.

A VTO interoperability test vector SHOULD have the following form:

```yaml
vto:
  version: 1
  id: vto-000001
  timestamp: 1785926400

  producer:
    peer_id: 12D3KooWExamplePeer
    implementation: go-libp2p
    version: 0.45.0

  measurements:
    latency_ms: 14.37
    throughput_mbps: 843.52
    bandwidth_mbps: 921.11
    packet_loss_pct: 0.04
    cpu_utilization_pct: 41.8
    memory_utilization_pct: 63.2
    peer_count: 142
    active_streams: 51
    uptime_seconds: 87234

  network:
    protocol: gossipsub
    transport: quic-v1
    region: ap-south-1

  metadata:
    tags:
      - observability
      - libp2p
      - telemetry

encoding:
  format: deterministic-cbor
  bytes_hex: "<generated bytes>"

digest:
  context: "<digest-context>"
  algorithm: "<algorithm>"
  digest_hex: "<generated digest>"

multihash:
  value: "<generated multihash>"

cid:
  value: "<generated CID>"
```

The test vector format itself is not normative.

---

## Appendix C. Design Rationale

The VTO design intentionally keeps four concerns separate:

```text
+--------------------+
| Telemetry Semantics|
|        VTO         |
+---------+----------+
          |
          v
+--------------------+
| Wire Representation|
|   Deterministic    |
|       CBOR         |
+---------+----------+
          |
          v
+--------------------+
| Cryptographic      |
| Identity           |
| Digest Context     |
+---------+----------+
          |
          v
+--------------------+
| Evidence / Trust   |
| Binding            |
|       CPB          |
+--------------------+
```

This separation provides several benefits.

First, telemetry formats can evolve independently of trust frameworks.

Second, a generic binding mechanism can reference a VTO without understanding every telemetry field.

Third, a digest context can be reused for additional CBOR-based evidence objects.

Finally, implementation and interoperability work can proceed incrementally: the VTO schema and real encoded objects can be defined and tested before the complete digest context is standardized.

The immediate implementation objective is therefore to establish the exact relationship between:

1. the VTO schema;
2. the actual CBOR bytes emitted by implementations;
3. floating-point representation;
4. the digest context; and
5. the resulting multihash/CID.

This provides the foundation for subsequent work on CPB and AAC composition.


### Feedback 

Please reach out to manu@libp2p.io and johanna@libp2p.io for any question, thoughts and feedback. We are collaborating with Steven Mih on developing a composable trust spec by focusing on VTO x CBP X AAC.
