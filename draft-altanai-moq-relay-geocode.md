---
title: "Geographic Location for Media over QUIC Relays"
abbrev: "MoQ Relay Geocode"
category: std

docname: draft-altanai-moq-relay-geocode-latest
submissiontype: IETF
number:
date: 2026-08-04
consensus: true
v: 1
area: "Web and Internet Transport"
workgroup: "moq"
keyword:
  - MOQ
  - Media over QUIC
  - geocode
  - relay
  - GDOR
  - IATA
  - geo-distributed
  - routing

venue:
  group: "moq"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "altanai/draft-altanai-moq-relay-geocode"
  latest: "https://datatracker.ietf.org/doc/draft-altanai-moq-relay-geocode/"

author:
  -
    fullname: Altanai Bisht
    organization: Cisco Meraki
    email: albisht@cisco.com
  -
    fullname: Tim Evens
    organization: Cisco
    email: tievens@cisco.com

normative:
  RFC7946:
  RFC8259:

informative:
  MoQTransport:
    title: "Media over QUIC Transport"
    author:
      org: "IETF MOQ WG"
    date: 2026
    seriesinfo:
      Internet-Draft: draft-ietf-moq-transport-23
    target: "https://datatracker.ietf.org/doc/draft-ietf-moq-transport/"
  MoQMetrics:
    title: "Metrics over MOQT"
    author:
      name: "C. Jennings"
    date: 2025
    seriesinfo:
      Internet-Draft: draft-jennings-moq-metrics-03
    target: "https://datatracker.ietf.org/doc/draft-jennings-moq-metrics/"
  RFC8801:
  RFC8805:
  RFC9388:
---

--- abstract

This document defines a mechanism for Media over QUIC (MoQ) relays to advertise their geographic location (geocode) and related path metrics. Some clients require their media data to remain locally or geo-fenced within specific jurisdictions for privacy and security compliance (e.g., GDPR, HIPAA, or sector-specific regulations). This mechanism enables service providers to track the geographic path of media Objects through the relay mesh and to enforce geo-fencing policies. It supports Geo-Distributed Orchestration and Routing (GDOR), data residency compliance, latency optimization, and relay selection.

--- middle

# Introduction

Media over QUIC Transport (MOQT) {{MoQTransport}} uses a relay-based architecture where publishers push media to relays and subscribers pull from relays. Relays can be chained for CDN-like distribution, forming a mesh of interconnected nodes. In such topologies, as illustrated by deployments with multiple relays (e.g., relays A, B, C, D) connecting Home/Enterprise clients to a Service Media backend, the following questions arise:

- **Where** is each relay located geographically?
- **How close** are relays to each other and to clients?
- **Which path** through the relay mesh minimizes latency or satisfies policy?

Today, MoQ provides no standard way for relays to advertise geographic position. Clients and orchestration systems rely on external mechanisms (e.g., IP geolocation feeds {{RFC8805}}, manual configuration) that are often imprecise, inconsistent, or unavailable. This document specifies a lightweight, protocol-aligned mechanism for relays to advertise geocode and related metrics, enabling:

1. **Vicinity and path computation**: Determining physical proximity and logical paths between relays.
2. **GDOR (Geo-Distributed Orchestration/Routing)**: Making routing and placement decisions based on geography (e.g., route through EU relays for GDPR, avoid certain jurisdictions).
3. **Latency optimization**: Selecting relays with lower RTT or propagation time (PT) to the client or to upstream relays.
4. **Data residency and compliance**: Ensuring media flows stay within required geographic boundaries. Clients often require their data to be locally or geo-fenced for privacy and security compliance (e.g., GDPR, HIPAA).
5. **Path tracking**: Enabling service providers to track the geographic path of media Objects through the relay mesh for compliance audits and geo-fencing enforcement.

## Design Goals

- **Path events on a dedicated track**: Publish immutable per-hop path events on a MOQT track. Forwarded Objects, Object Properties, payloads, and transport extension headers remain unchanged.
- **Protocol alignment**: Use existing MOQT messages (PUBLISH, SUBSCRIBE) and streams for geocode advertisement, avoiding new message formats. Relays do not need to invent proprietary messaging for cross-vendor use cases.
- **Minimal protocol impact**: Reuse existing MoQ catalog, metrics, or discovery mechanisms where possible.
- **Interoperability**: Use standard geographic representations (WGS84, GeoJSON) and widely recognized identifiers (IATA codes).
- **Extensibility**: Allow optional altitude, region codes, and path metrics (RTT, PT, bandwidth estimate, loss) without mandating them. Advertise identifiers and dynamic metrics in separate SETUP options so each can evolve independently.
- **Reachability, not fixed topology**: Treat peer-relay information as dynamic reachability (which relays are currently reachable and at what cost), not as a static neighbor list.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

**Geocode**: A representation of geographic position, typically latitude and longitude (WGS84), optionally with altitude.

**Relay Vicinity**: The geographic proximity of one relay to another or to a client, expressed as distance or as a qualitative relationship (e.g., same region, same metro).

**Relay Path**: A sequence of relays through which media flows from publisher to subscriber, or the logical/geographic route between relays.

**GDOR (Geo-Distributed Orchestration/Routing)**: Routing and orchestration decisions that consider geographic location, including data residency, latency optimization, and jurisdictional compliance.

**IATA Code**: A three-letter code assigned by the International Air Transport Association to identify airports and major metropolitan areas (e.g., JFK, LHR, FRA). Used here as an optional human-readable geographic identifier for relay locations.

# Use Cases

## Relay Selection by Proximity

A subscriber in New York (NY) may have multiple relays available (e.g., A, B, C, D). With geocode and RTT/PT metrics, the client or an orchestration layer can select the relay that minimizes latency or satisfies geographic constraints.

## Geo-Distributed Orchestration (GDOR)

Some clients require their media data to remain locally or geo-fenced within specific jurisdictions for privacy and security compliance (e.g., GDPR, HIPAA, or sector-specific regulations). An operator may require that:

- EU subscriber traffic flows only through EU relays.
- Media for a given jurisdiction is processed within that jurisdiction.
- Path selection avoids certain regions for policy or cost reasons.

Geocode enables GDOR by allowing relays to advertise their jurisdiction (e.g., via country/region codes) and by allowing path computation to respect geographic boundaries. Service providers can use relay geocode and advertised reachability to other relays to **track the geographic path** of media packets as they traverse the MoQ relay mesh, supporting compliance audits and geo-fencing enforcement.

## Path and Vicinity Awareness

In a relay mesh (e.g., A↔B↔C↔D), knowing the geocode of each relay allows:

- **Vicinity**: Computing great-circle distance or approximate propagation delay between relays.
- **Path**: Understanding the geographic route (e.g., NY → Relay D → Relay A → SVC) and optimizing it.

## Metrics Integration

Deployments commonly measure **RTT to Relay**, **PT (Propagation Time) to Relay**, bandwidth estimate, and loss as key metrics. Geocode complements these by providing a stable, policy-relevant attribute (location) that does not change with network conditions, while RTT, PT, bandwidth, and loss provide dynamic performance signals.

# Geocode Representation

This section is informative. The following formats are suggested for representing relay geocode; implementations may use alternative representations.

## Suggested Format: WGS84 Coordinates

Relay geocode may be represented as WGS84 coordinates (as used in GeoJSON {{RFC7946}}):

- **latitude**: Decimal degrees, -90 to 90.
- **longitude**: Decimal degrees, -180 to 180.
- **altitude** (optional): Meters above WGS84 ellipsoid.

Example (JSON):

```json
{
  "latitude": 40.6413,
  "longitude": -73.7781,
  "altitude": 4
}
```

## Suggested: IATA Code

An IATA airport code may be included as a human-readable geographic identifier (e.g., JFK for New York, LHR for London). It is advisory only and may correspond to the nearest major airport or metro area where the relay is located; precise routing may use latitude/longitude when geographic accuracy is required.

# Integration with MoQ

This document proposes several integration options for geocode advertisement and path tracing.

## Option 1: Hop-by-Hop Path Events on a MOQT Track

A relay that advertises path-diagnostics support MUST publish a path event for each Object forwarding action covered by that capability. The relay MUST NOT modify the forwarded Object, its Object Properties, or its payload to produce the event. The event is carried on the relay's `path-events` track and contains:

- `object`: Full Track Namespace, Track Name, Group ID, and Object ID of the forwarded Object.
- `relay_id` (string, REQUIRED): Identifier matching the relay's `GEOCODE_IDENTITY`.
- `ingress_peer` and `egress_peer` (strings, REQUIRED): Identifiers of the adjacent peers for this forwarding action.
- `direction` (string, REQUIRED): `publisher-to-subscriber`, identifying the Object flow through the relay.
- `geocode` (object, REQUIRED): The relay's advertised geocode representation or a reference to its identity version.
- `observed_at` (timestamp, REQUIRED): Time at which the relay forwarded the Object.
- `ingress_srtt_ms` and `egress_srtt_ms` (numbers, OPTIONAL): QUIC smoothed RTT samples for the ingress and egress peer connections at the observation time.
- `object_bytes_received` (integer, REQUIRED): Serialized MOQT Object bytes received by this relay, including Object fields, Object Properties, and payload, but excluding QUIC framing.
- `connection_object_bytes_received_total` (integer, OPTIONAL): Cumulative serialized MOQT Object bytes received on the ingress connection, with the same counting rule as `object_bytes_received`.
- `counter_epoch` (string, REQUIRED when the cumulative counter is present): Identifier that changes when the ingress connection or counter is reset.

SRTT is measured separately for each adjacent QUIC connection. A consumer MUST NOT interpret one hop's SRTT as end-to-end latency or sum per-hop SRTTs as an exact end-to-end measurement. A consumer can compare SRTT with the geodesic distance implied by adjacent relay claims as an anomaly signal; this comparison does not prove or disprove a relay's physical location.

A consumer MUST treat a missing path event as incomplete evidence, not as proof that no relay forwarded the Object. The event identifies a relay's reported forwarding action and does not prove delivery to the next peer. Implementations MUST authorize access to path events because they can reveal topology, peer identity, traffic volume, and user activity.

Example (informative):

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

## Option 2: Geocode Advertisement via MOQT Messages and Streams

Relays MAY advertise geocode using standard MOQT messages and streams:

- **PUBLISH**: A relay publishes its geocode advertisement as a track (e.g., under a reserved namespace such as `moq://relay-geocode.example/v1/<relay_id>`).
- **SUBSCRIBE**: Other relays, clients, or orchestration systems SUBSCRIBE to geocode tracks to discover relay locations and reachability to other relays.
- **Streams**: Geocode data is carried in MOQT objects over QUIC streams, using the JSON schema defined in the related document (Appendix C) as the object payload.

This approach reuses the same protocol that relays already use for media distribution. No new message types or wire formats are required.

## Well-Known Relay Tracks

This document defines the Track Namespace as the three ASCII fields `("relay-geocode.moq.arpa", "v1", <relay-id>)`. The relay identifier is a stable lowercase ASCII token. MOQT compares namespace fields byte-for-byte, so senders use this canonical representation. Under that namespace, the Track Names are `metrics` and `path-events`. These are MOQT tracks, not HTTP well-known URIs. Receivers discover and subscribe using MOQT namespace and track procedures. Relay identity remains advertised directly in SETUP.

Dynamic metrics MUST NOT be carried in SETUP because SETUP is exchanged only during session initialization. The `metrics` track carries timestamped samples, including reachability to named peers. Each reachability record identifies the adjacent relay and the associated measurements; it does not imply a fixed topology.

## Option 3: Catalog Integration

A relay MAY include geocode in catalog metadata for namespaces or tracks it serves. Clients that discover tracks via the catalog can use geocode for relay selection or policy checks.

## Option 4: Metrics Integration

Geocode MAY be part of metrics exposed per {{MoQMetrics}} for relay selection and monitoring. Resources (e.g., relays) can include geocode as attributes in the metrics data model.

## Option 5: Discovery via MOQT SETUP Options (RECOMMENDED for session discovery)

A relay advertises stable identity in a MOQT SETUP option (see {{MoQTransport}}). Dynamic metrics and path events use the `metrics` and `path-events` tracks defined above; they are not sent in SETUP. Endpoints MUST ignore unknown Setup Options, enabling incremental deployment. The proposed Setup Option code point requires IANA allocation; until assigned, implementations use privately negotiated values.

### GEOCODE_IDENTITY Setup Option

This option carries stable geographic identifiers for the advertising relay. The option type is `GEOCODE_IDENTITY` (see {{iana-setup}}). The Setup Option uses MOQT Type-Length-Value framing. Its value is a UTF-8 JSON object with the following fields; it has no additional JSON-internal length prefix.

- `relay_id` (string, REQUIRED): Unique relay identifier.
- `latitude` / `longitude` (number, REQUIRED): WGS84 coordinates.
- `altitude` (number, OPTIONAL): Meters above WGS84 ellipsoid.
- `iata_code` (string, OPTIONAL): Three-letter IATA airport/metro code.
- `country` (string, OPTIONAL): ISO 3166-1 alpha-2.
- `subdivision` (string, OPTIONAL): ISO 3166-2 subdivision code {{RFC9388}}.

Example (informative):

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

### Dynamic Metrics on a MOQT Track

Changing path metrics and reachability are published as timestamped Objects on the `metrics` track. They are not carried in SETUP, which is exchanged only during session initialization. Each sample identifies the peer and measurement scope. SRTT values are per adjacent QUIC connection and do not represent end-to-end latency.

### Other Discovery Mechanisms

As alternatives or complements to SETUP options, a relay MAY also:

- Serve geocode at a well-known path (e.g., `/.well-known/moq-relay-geocode`).
- Advertise geocode as provisioning-domain data per {{RFC8801}}.

The relay Track Namespace and Setup Option code point are proposals that require IANA allocation or private negotiation for experiments. The track names and JSON schemas in this document do not define new MOQT control messages or modify Object encoding.

# Vicinity and Path Computation

## Great-Circle Distance

Given two relays with geocode (lat1, lon1) and (lat2, lon2), the approximate great-circle distance in kilometers can be computed using the Haversine formula or an equivalent. This provides a stable measure of physical proximity independent of network topology.

## Path Metrics

When relays publish reachability (e.g., `reachable` with `rtt_ms`, `pt_ms`, `bw_kbps`, and `loss` on the `metrics` track), a client or orchestration system can:

1. **Build a relay graph**: Nodes = relays, edges = current reachability relationships with RTT/PT/bandwidth/loss.
2. **Compute paths**: Shortest or best path by RTT, PT, bandwidth, loss, or geographic distance.
3. **Apply GDOR constraints**: Filter paths to those that stay within required jurisdictions (using `country` and `subdivision` codes as in {{RFC9388}}).
4. **Refresh dynamically**: Treat reachability as time-varying; do not assume a static neighbor list.

## Vicinity Semantics

- **Same metro**: Relays with the same `iata_code` or within ~50 km.
- **Same region**: Relays with the same `subdivision` or `country`.
- **Cross-border**: Path crosses `country` boundaries; may trigger compliance checks.

# Security Considerations

## Privacy

- Geocode reveals relay location. Operators who wish to hide exact location MAY advertise only `country` and `subdivision`, or a coarse-grained geocode (e.g., rounded to 0.1°).
- IATA codes are less precise than coordinates and may be preferred when exact location must not be disclosed.

## Integrity

- Geocode advertisements SHOULD be integrity-protected (e.g., over TLS as with MoQ) and, when used for compliance, SHOULD be verifiable (e.g., signed by a trusted authority).

## Misuse

- Adversaries could advertise false geocode to attract traffic or evade geographic restrictions. Relays SHOULD be authenticated; geocode from untrusted sources SHOULD be validated or ignored.

# IANA Considerations
{#iana}

## MOQT Setup Option Types
{#iana-setup}

This document requests registration of the following MOQT Setup Option types in the appropriate MOQT registry (see {{MoQTransport}}):

| Name | Value | Description | Reference |
|------|-------|-------------|-----------|
| GEOCODE_IDENTITY | TBD1 | Relay geographic identity (stable identifiers) | This document |


Until values are assigned, experimental use MUST use privately negotiated option types.

## Other Registrations

If a well-known URI (e.g., `/.well-known/moq-relay-geocode`) or a path-trace extension header type is standardized, the appropriate IANA registry would be updated in a future revision.

--- back

# Example Topology (Informative)

The following example illustrates a typical relay mesh topology:

~~~~~
Home/Ent (NY)                    Relay Mesh                    SVC Media MS
    |                          A --- B
    |                           \   /
    +----(16ms RTT)-------------> D
    |                           /   \
    |                          C --- (to SVC)
~~~~~

- **Relay D**: JFK (NY), serves Home/Ent with 16ms RTT.
- **Relay A, B, C**: Interconnected; A connects to SVC.
- Geocode for each enables path selection (e.g., NY client → D → A → SVC) and GDOR (e.g., ensure D and A are in permitted jurisdictions).

# IATA Code Reference (Informative)

IATA codes are maintained by the International Air Transport Association. A subset useful for MoQ relay identification:

| Code | City/Region |
|------|-------------|
| JFK  | New York (John F. Kennedy) |
| LGA  | New York (LaGuardia) |
| EWR  | Newark (NYC metro) |
| IAD  | Washington Dulles (Virginia) |
| DCA  | Washington Reagan |
| LHR  | London Heathrow |
| FRA  | Frankfurt |
| CDG  | Paris Charles de Gaulle |
| AMS  | Amsterdam |
| NRT  | Tokyo Narita |
| SIN  | Singapore |
| SYD  | Sydney |
| GRU  | Sao Paulo |

Operators SHOULD use the official IATA code list for authoritative mappings.

# Relay Advertisement Format (Informative)

The JSON schema and examples for relay geocode identity, metrics, and reachability are defined in the related document `moq-relay-geocode-advertisement-format.md`, which accompanies this draft. Implementations may use that schema when advertising geocode via Option 2 (PUBLISH/SUBSCRIBE), Option 3 (Catalog), Option 4 (Metrics), or Option 5 (SETUP options).

# Acknowledgments
{:numbered="false"}

The authors thank the participants of the Media over QUIC Working Group for
discussion and review feedback that helped shape this document.
