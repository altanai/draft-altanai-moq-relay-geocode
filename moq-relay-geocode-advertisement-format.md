# MoQ Relay Geocode Advertisement Format

*Related document for draft-altanai-moq-relay-geocode*

This document defines the JSON schema and example for relay geocode advertisement. It is a supporting specification for [draft-altanai-moq-relay-geocode](https://datatracker.ietf.org/doc/draft-altanai-moq-relay-geocode/).

---

## JSON Schema

The following JSON structure defines the relay geocode advertisement. It MAY be carried in MoQ catalog extensions, metrics (draft-jennings-moq-metrics), or a dedicated discovery resource (e.g., well-known URI).

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "MoQ Relay Geocode",
  "type": "object",
  "required": ["relay_id", "latitude", "longitude"],
  "properties": {
    "relay_id": { "type": "string", "description": "Unique relay identifier" },
    "latitude": { "type": "number", "minimum": -90, "maximum": 90 },
    "longitude": { "type": "number", "minimum": -180, "maximum": 180 },
    "altitude": { "type": "number", "description": "Meters above WGS84 ellipsoid" },
    "iata_code": { "type": "string", "pattern": "^[A-Z]{3}$", "description": "IATA airport/metro code" },
    "country": { "type": "string", "pattern": "^[A-Z]{2}$", "description": "ISO 3166-1 alpha-2" },
    "subdivision": { "type": "string", "description": "ISO 3166-2 subdivision" },
    "rtt_ms": { "type": "number", "description": "RTT to this relay (ms), if advertised" },
    "pt_ms": { "type": "number", "description": "Propagation time to relay (ms), if advertised" },
    "neighbors": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "relay_id": { "type": "string" },
          "rtt_ms": { "type": "number" },
          "pt_ms": { "type": "number" }
        }
      },
      "description": "Adjacent relays and inter-relay metrics"
    }
  }
}
```

---

## Example Advertisement

Relay D in New York area, with path metrics:

```json
{
  "relay_id": "relay-D",
  "latitude": 40.6413,
  "longitude": -73.7781,
  "altitude": 4,
  "iata_code": "JFK",
  "country": "US",
  "subdivision": "US-NY",
  "rtt_ms": 16,
  "pt_ms": 12,
  "neighbors": [
    { "relay_id": "relay-A", "rtt_ms": 7, "pt_ms": 3 },
    { "relay_id": "relay-C", "rtt_ms": 12, "pt_ms": 5 }
  ]
}
```

---

## Field Reference

| Field | Required | Description |
|-------|----------|-------------|
| relay_id | Yes | Unique relay identifier |
| latitude | Yes | Decimal degrees, -90 to 90 (WGS84) |
| longitude | Yes | Decimal degrees, -180 to 180 (WGS84) |
| altitude | No | Meters above WGS84 ellipsoid |
| iata_code | No | 3-letter IATA airport/metro code |
| country | No | ISO 3166-1 alpha-2 |
| subdivision | No | ISO 3166-2 subdivision |
| rtt_ms | No | RTT to this relay (ms) |
| pt_ms | No | Propagation time to relay (ms) |
| neighbors | No | Array of adjacent relays with inter-relay metrics |
