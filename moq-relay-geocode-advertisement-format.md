# MoQ Relay Geocode Advertisement Format

*Related document for draft-altanai-moq-relay-geocode*

This document gives informative JSON examples for the relay identity and
client-intent Setup Options, and the dynamic metrics and path-event tracks.
Networked JSON is encoded as UTF-8 without a byte order mark, following
[RFC 8259](https://www.rfc-editor.org/rfc/rfc8259.html#section-8.1). UTF-7 is
not used.

Relay identity is sent by each relay in its own SETUP message and is also
available on the `identity` track in the Track Namespace
`("relay-geocode.moq.arpa", "v1", <relay-id>)`. Dynamic metrics and forwarding events use the `metrics` and
`path-events` tracks in that namespace; they are not placed in SETUP.

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

## Client Policy Intent (`GEOCODE_INTENT`)

The client can request allowed jurisdictions and/or a nearby relay without
sending exact client coordinates:

```json
{
  "allowed_jurisdictions": ["US", "US-NY"],
  "near_me": true,
  "desired_geocode": { "type": "iata", "code": "JFK" }
}
```

This option expresses policy intent; it does not grant authorization or
authenticate the client. The receiving relay applies its authorization and
trust policy before selecting or redirecting to a relay.

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
