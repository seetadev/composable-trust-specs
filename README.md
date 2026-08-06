# VTO (Verifiable Telemetry Object) – Initial Working Draft

## 1. CDDL Schema (Draft)

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

---

# 2. Example Encoded Object

Canonical diagnostic representation

```json
{
  "version":1,
  "id":"vto-000001",
  "timestamp":1785926400,
  "producer":{
    "peer_id":"12D3KooWExamplePeer",
    "implementation":"go-libp2p",
    "version":"0.45.0"
  },
  "measurements":{
    "latency_ms":14.37,
    "throughput_mbps":843.52,
    "bandwidth_mbps":921.11,
    "packet_loss_pct":0.04,
    "cpu_utilization_pct":41.8,
    "memory_utilization_pct":63.2,
    "peer_count":142,
    "active_streams":51,
    "uptime_seconds":87234
  },
  "network":{
    "protocol":"gossipsub",
    "transport":"quic-v1",
    "region":"ap-south-1"
  },
  "metadata":{
    "tags":[
      "observability",
      "libp2p",
      "telemetry"
    ]
  }
}
```

---

# 3. Example Encoded Bytes

For the first implementation, we'll serialize the object using deterministic CBOR.

Example representation:

```
Base64:
pmF2ZXJzaW9uAWJpZ...
```

or

```
Hex:
A6
67657273696F6E01
6269646A76746F2D303030303031
...
```

(We'll generate the actual encoded bytes directly from the implementation to avoid ambiguity.)

---

# 4. Digest

Digest algorithm (current placeholder)

```
SHA-256
```

Example

```
Digest (hex)

3d7d7cb98b...
```

```
CID

bafy...
```

The final digest should always be computed over the deterministic CBOR serialization.

---

# 5. Float-bearing Fields

The following fields are encoded as IEEE-754 floating-point values.

| Field                  | Type    | Units        | Source                   |
| ---------------------- | ------- | ------------ | ------------------------ |
| latency_ms             | float64 | milliseconds | measured RTT             |
| throughput_mbps        | float64 | Mbps         | moving average           |
| bandwidth_mbps         | float64 | Mbps         | estimated link bandwidth |
| packet_loss_pct        | float64 | %            | transport statistics     |
| cpu_utilization_pct    | float64 | %            | host metrics             |
| memory_utilization_pct | float64 | %            | host metrics             |

---

# 6. Integer Fields

| Field          | Type |
| -------------- | ---- |
| version        | uint |
| timestamp      | uint |
| peer_count     | uint |
| active_streams | uint |
| uptime_seconds | uint |

---

# 7. Float Handling Requirements (Initial)

The digest context should preserve floating-point measurements exactly as emitted by the implementation.

Requirements:

* IEEE-754 binary64 encoding
* No conversion to decimal strings
* Preserve NaN payloads if permitted by the profile
* Preserve positive/negative zero
* Deterministic CBOR encoding
* Digest computed over canonical CBOR bytes
* No JSON canonicalization step

---

# 8. Digest Context Requirements

The digest context should:

* operate directly on deterministic CBOR
* support native floating-point values
* produce stable multihashes
* map naturally to CIDs
* remain independent of VTO semantics
* be reusable by other CBOR-based telemetry formats

This draft should provide Anton with enough structure to begin defining the CBOR digest context while Johanna and I continue refining the complete VTO specification.
