# MoQ Relay Geocode Advertisement Format

*Related document for draft-altanai-moq-relay-geocode*

This document defines the JSON schemas and examples for relay geocode
identity, path metrics, and reachability. It is a supporting
specification for [draft-altanai-moq-relay-geocode](https://datatracker.ietf.org/doc/draft-altanai-moq-relay-geocode/).

Advertisements are intentionally split so that stable identifiers and
dynamic metrics can be carried separately (e.g., as distinct MOQT SETUP
options: `GEOCODE_IDENTITY` and `GEOCODE_METRICS`).

---

## Identity Schema (`GEOCODE_IDENTITY`)

Stable geographic identifiers for a relay.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "MoQ Relay Geocode Identity",
  "type": "object",
  "required": ["relay_id", "latitude", "longitude"],
  "properties": {
    "relay_id": { "type": "string", "description": "Unique relay identifier" },
    "latitude": { "type": "number", "minimum": -90, "maximum": 90 },
    "longitude": { "type": "number", "minimum": -180, "maximum": 180 },
    "altitude": { "type": "number", "description": "Meters above WGS84 ellipsoid" },
    "iata_code": { "type": "string", "pattern": "^[A-Z]{3}$", "description": "IATA airport/metro code" },
    "country": { "type": "string", "pattern": "^[A-Z]{2}$", "description": "ISO 3166-1 alpha-2" },
    "subdivision": { "type": "string", "description": "ISO 3166-2 subdivision" }
  }
}
```

### Example identity

```json
{
  "relay_id": "relay-D",
  "latitude": 40.6413,
  "longitude": -73.7781,
  "altitude": 4,
  "iata_code": "JFK",
  "country": "US",
  "subdivision": "US-NY"
}
```

---

## Dynamic Metrics Track

Each sample is timestamped and identifies the peer and measurement scope.
SRTT is measured for an adjacent QUIC connection, not end-to-end latency.

```json
{
  "sampled_at": "2026-11-25T20:00:00Z",
  "peer_id": "relay-a",
  "srtt_ms": 8.2,
  "pt_ms": 5.1,
  "bw_kbps": 50000,
  "loss": 0.001
}
```

## Field Reference

### Identity fields

| Field | Required | Description |
|-------|----------|-------------|
| relay_id | Yes | Unique relay identifier |
| latitude | Yes | Decimal degrees, -90 to 90 (WGS84) |
| longitude | Yes | Decimal degrees, -180 to 180 (WGS84) |
| altitude | No | Meters above WGS84 ellipsoid |
| iata_code | No | 3-letter IATA airport/metro code |
| country | No | ISO 3166-1 alpha-2 |
| subdivision | No | ISO 3166-2 subdivision |

### Dynamic metrics fields

| Field | Required | Description |
|-------|----------|-------------|
| rtt_ms | No | RTT to this relay (ms) |
| pt_ms | No | Propagation time to relay (ms) |
| bw_kbps | No | Bandwidth estimate (kbps) |
| loss | No | Loss ratio in range 0.0 to 1.0 |
| reachable | No | Dynamic reachability to other relays with optional path metrics |


## Dynamic Metrics Track

Each sample is timestamped and identifies its peer and measurement scope.
SRTT values are per adjacent QUIC connection; they are not end-to-end latency.

```json
{
  "sampled_at": "2026-11-25T20:00:00Z",
  "peer_id": "relay-a",
  "srtt_ms": 8.2,
  "pt_ms": 5.1,
  "bw_kbps": 50000,
  "loss": 0.001
}
```

## Hop-by-Hop Path Event (`path-events` track)

Each event correlates a relay forwarding action to the MOQT Object and records
the relay's ingress and egress peers. The byte count is the serialized MOQT
Object length, including its header, properties, and payload, but excluding
QUIC framing. `connection_object_bytes_received_total` is a monotonic counter
for that ingress connection and uses `counter_epoch` to distinguish resets.

```json
{
  "version": 1,
  "object": {
    "namespace": ["example", "media"],
    "track": "video",
    "group_id": 12,
    "object_id": 7
  },
  "relay_id": "relay-d",
  "ingress_peer": "publisher-edge",
  "egress_peer": "relay-a",
  "direction": "publisher-to-subscriber",
  "geocode": { "type": "iata", "code": "JFK" },
  "observed_at": "2026-11-25T20:00:00Z",
  "ingress_srtt_ms": 16.4,
  "egress_srtt_ms": 8.2,
  "object_bytes_received": 1200,
  "connection_object_bytes_received_total": 987654321,
  "counter_epoch": "connection-7"
}
```

An event reports the relay's forwarding action. It does not prove that the
egress peer received the Object or independently prove the relay's location.
The relay can publish one event per downstream forwarding action when an
Object is replicated to multiple peers.

Networked JSON is UTF-8 without a byte order mark per RFC 8259; UTF-7 is not used.
