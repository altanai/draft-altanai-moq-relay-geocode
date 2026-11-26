# MoQ Relay Geocode Advertisement Format

*Related document for draft-altanai-moq-relay-geocode*

This document defines JSON examples for relay geocode identity, path metrics, and reachability. Networked JSON is encoded as UTF-8 without a byte order mark per RFC 8259; UTF-7 is not used. It is a supporting
specification for [draft-altanai-moq-relay-geocode](https://datatracker.ietf.org/doc/draft-altanai-moq-relay-geocode/).

Advertisements are intentionally split so that stable identifiers and
dynamic metrics can be carried separately (e.g., as distinct MOQT SETUP
options: `GEOCODE_IDENTITY` and `GEOCODE_METRICS`).

---

## Relay Identity (`GEOCODE_IDENTITY`)

The `geocode` member contains exactly one primary representation: WGS84 or
IATA. Country and subdivision are separate jurisdiction claims. IATA codes
identify an airport or metro area and do not establish a jurisdiction.

```json
{
  "relay_id": "relay-d",
  "geocode": {
    "type": "wgs84",
    "latitude": 40.7,
    "longitude": -74.0
  },
  "country": "US",
  "subdivision": "US-NY",
  "location_precision": "coarse"
}
```

An IATA representation uses this form instead:

```json
{
  "relay_id": "relay-d",
  "geocode": { "type": "iata", "code": "JFK" },
  "country": "US",
  "subdivision": "US-NY",
  "location_precision": "jurisdiction-only"
}
```

## Metrics Schema (`GEOCODE_METRICS`)

Dynamic path metrics and reachability. Topology in MoQ deployments is
dynamic; the `reachable` array describes current reachability to other
relays, not a fixed neighbor list.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "MoQ Relay Geocode Metrics",
  "type": "object",
  "properties": {
    "rtt_ms": { "type": "number", "description": "RTT to this relay (ms)" },
    "pt_ms": { "type": "number", "description": "Propagation time to relay (ms)" },
    "bw_kbps": { "type": "number", "description": "Bandwidth estimate (kbps)" },
    "loss": { "type": "number", "minimum": 0, "maximum": 1, "description": "Loss ratio 0.0-1.0" },
    "reachable": {
      "type": "array",
      "description": "Currently reachable relays and path metrics (dynamic, not a fixed topology)",
      "items": {
        "type": "object",
        "required": ["relay_id"],
        "properties": {
          "relay_id": { "type": "string" },
          "rtt_ms": { "type": "number" },
          "pt_ms": { "type": "number" },
          "bw_kbps": { "type": "number" },
          "loss": { "type": "number", "minimum": 0, "maximum": 1 }
        }
      }
    }
  }
}
```

### Example metrics

```json
{
  "rtt_ms": 16,
  "pt_ms": 12,
  "bw_kbps": 50000,
  "loss": 0.001,
  "reachable": [
    { "relay_id": "relay-A", "rtt_ms": 7, "pt_ms": 3, "bw_kbps": 80000, "loss": 0.0 },
    { "relay_id": "relay-C", "rtt_ms": 12, "pt_ms": 5, "bw_kbps": 40000, "loss": 0.002 }
  ]
}
```

---

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

### Metrics fields

| Field | Required | Description |
|-------|----------|-------------|
| rtt_ms | No | RTT to this relay (ms) |
| pt_ms | No | Propagation time to relay (ms) |
| bw_kbps | No | Bandwidth estimate (kbps) |
| loss | No | Loss ratio in range 0.0 to 1.0 |
| reachable | No | Dynamic reachability to other relays with optional path metrics |
