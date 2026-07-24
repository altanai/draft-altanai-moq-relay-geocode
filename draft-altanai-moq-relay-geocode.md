---
title: "Geographic Location for Media over QUIC Relays"
abbrev: "MoQ Relay Geocode"
category: std

docname: draft-altanai-moq-relay-geocode-latest
submissiontype: IETF
number:
date: 2026-07-24
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
  RFC2119:
  RFC8174:
  RFC7946:
    title: "The GeoJSON Format"
    date: 2016-08
    author:
      - name: H. Butler
      - name: M. Daly
      - name: A. Doyle
      - name: S. Gillies
      - name: S. Hagen
      - name: T. Schaub

informative:
  MoQTransport:
    title: "Media over QUIC Transport"
    author:
      org: "IETF MOQ WG"
    date: 2026
    seriesinfo:
      Internet-Draft: draft-ietf-moq-transport-17
    target: "https://datatracker.ietf.org/doc/draft-ietf-moq-transport/"
  MoQMetrics:
    title: "Metrics over MOQT"
    author:
      name: "C. Jennings"
    date: 2025
    seriesinfo:
      Internet-Draft: draft-jennings-moq-metrics-02
    target: "https://datatracker.ietf.org/doc/draft-jennings-moq-metrics/"
  RFC8801:
    title: "Discovering Provisioning Domain Names and Data"
    date: 2020-07
  RFC9388:
    title: "Content Delivery Network Interconnection (CDNI) Footprint Types: Country Subdivision Code and Footprint Union"
    date: 2023-06
  RFC8805:
    title: "A Format for Self-Published IP Geolocation Feeds"
    date: 2020-08
---

--- abstract

This document defines a mechanism for Media over QUIC (MoQ) relays to advertise their geographic location (geocode) and related path metrics. Some clients require their media data to remain locally or geo-fenced within specific jurisdictions for privacy and security compliance (e.g., GDPR, HIPAA, or sector-specific regulations). This mechanism enables service providers to track the geographic path of media packets through the relay mesh and to enforce geo-fencing policies. It supports Geo-Distributed Orchestration and Routing (GDOR), data residency compliance, latency optimization, and relay selection. The specification includes optional IATA airport codes as human-readable geographic identifiers for major relay locations.

--- middle

# Introduction

Media over QUIC Transport (MOQT) {{MoQTransport}} uses a relay-based architecture where publishers push media to relays and subscribers pull from relays. Relays can be chained for CDN-like distribution, forming a mesh of interconnected nodes. In such topologies—as illustrated by deployments with multiple relays (e.g., relays A, B, C, D) connecting Home/Enterprise clients to a Service Media backend—the following questions arise:

- **Where** is each relay located geographically?
- **How close** are relays to each other and to clients?
- **Which path** through the relay mesh minimizes latency or satisfies policy?

Today, MoQ provides no standard way for relays to advertise geographic position. Clients and orchestration systems rely on external mechanisms (e.g., IP geolocation, manual configuration) that are often imprecise, inconsistent, or unavailable. This document specifies a lightweight, protocol-aligned mechanism for relays to advertise geocode and related metrics, enabling:

1. **Vicinity and path computation**: Determining physical proximity and logical paths between relays.
2. **GDOR (Geo-Distributed Orchestration/Routing)**: Making routing and placement decisions based on geography (e.g., route through EU relays for GDPR, avoid certain jurisdictions).
3. **Latency optimization**: Selecting relays with lower RTT or propagation time (PT) to the client or to upstream relays.
4. **Data residency and compliance**: Ensuring media flows stay within required geographic boundaries. Clients often require their data to be locally or geo-fenced for privacy and security compliance (e.g., GDPR, HIPAA).
5. **Path tracking**: Enabling service providers to track the geographic path of media packets through the relay mesh for compliance audits and geo-fencing enforcement.

## Design Goals

- **Path trace in-band (primary)**: Use MOQT extension headers on objects for path tracing as the RECOMMENDED integration. Each relay/proxy appends device name and direction; clients can verify the geographic path from received objects.
- **Protocol alignment**: Use existing MOQT messages (PUBLISH, SUBSCRIBE) and streams for geocode advertisement, avoiding new message formats. Relays do not need to invent proprietary messaging for cross-vendor use cases.
- **Minimal protocol impact**: Reuse existing MoQ catalog, metrics, or discovery mechanisms where possible.
- **Interoperability**: Use standard geographic representations (WGS84, GeoJSON) and widely recognized identifiers (IATA codes).
- **Extensibility**: Allow optional altitude, region codes, and path metrics (RTT, PT) without mandating them.

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

Geocode enables GDOR by allowing relays to advertise their jurisdiction (e.g., via country/region codes) and by allowing path computation to respect geographic boundaries. Service providers can use relay geocode and the advertised neighbor topology to **track the geographic path** of media packets as they traverse the MoQ relay mesh, supporting compliance audits and geo-fencing enforcement.

## Path and Vicinity Awareness

In a relay mesh (e.g., A↔B↔C↔D), knowing the geocode of each relay allows:

- **Vicinity**: Computing great-circle distance or approximate propagation delay between relays.
- **Path**: Understanding the geographic route (e.g., NY → Relay D → Relay A → SVC) and optimizing it.

## Metrics Integration

Deployments commonly measure **RTT to Relay** and **PT (Propagation Time) to Relay** as key metrics. Geocode complements these by providing a stable, policy-relevant attribute (location) that does not change with network conditions, while RTT/PT provide dynamic performance signals.

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

### Option 1: Path Trace via MOQT Extension Headers (RECOMMENDED)

To enable clients to verify the geographic path of media objects as they traverse relays and proxies, relays and proxies MAY append a path-trace entry to a MOQT object extension header as objects are forwarded. Path tracing uses the extension header mechanism described in {{MoQTransport}} (Section 10.2.1.2).

Each path-trace entry in the extension header SHOULD include:

- **device**: Name or identifier of the relay or proxy (e.g., `relay_id`, hostname, or a compact label such as `relay-D`).
- **direction**: The direction of traversal at this device—e.g., `ingress` (object received from upstream toward publisher) or `egress` (object sent toward downstream subscriber). Alternative encodings such as `upstream`/`downstream` or `in`/`out` may be used.

Additional fields such as `iata_code`, `country`, or `subdivision` MAY be included for compliance and geo-fencing verification.

Example path-trace entry (informative):

```json
{ "device": "relay-D", "direction": "egress", "iata_code": "JFK" }
```

Clients that receive objects can inspect the extension header to reconstruct the relay path (e.g., `relay-D (JFK) egress → relay-A (FRA) ingress → relay-A egress`) for compliance audits or geo-fencing enforcement.

This is the RECOMMENDED for path tracing as it travels in-band with media and enables clients to verify the geographic path directly from received objects.

### Option 2: Geocode Advertisement via MOQT Messages and Streams

Relays MAY advertise geocode using standard MOQT messages and streams:

- **PUBLISH**: A relay publishes its geocode advertisement as a track (e.g., under a reserved namespace such as `moq://relay-geocode.moq.arpa/v1/<relay_id>`).
- **SUBSCRIBE**: Other relays, clients, or orchestration systems SUBSCRIBE to geocode tracks to discover relay locations and neighbor topology.
- **Streams**: Geocode data is carried in MOQT objects over QUIC streams, using the JSON schema defined in the related document (Appendix C) as the object payload.

This approach reuses the same protocol that relays already use for media distribution. No new message types or wire formats are required.

### Option 3: Catalog Integration

A relay MAY include geocode in catalog metadata for namespaces or tracks it serves. Clients that discover tracks via the catalog can use geocode for relay selection or policy checks.

### Option 4: Metrics Integration

Geocode MAY be part of metrics exposed per {{MoQMetrics}} for relay selection and monitoring. Resources (e.g., relays) can include geocode as attributes in the metrics data model.

### Option 5: Discovery (Well-Known URI or Setup Option)

A relay MAY serve geocode at a well-known path (e.g., `/.well-known/moq-relay-geocode`) or via a Setup Option in SETUP, for deployments that prefer HTTP-based or session-level discovery.

---

The reserved namespace, extension header type, and exact encoding for the path trace are left to a companion specification or a future revision of {{MoQTransport}}.

# Vicinity and Path Computation

## Great-Circle Distance

Given two relays with geocode (lat1, lon1) and (lat2, lon2), the approximate great-circle distance in kilometers can be computed using the Haversine formula or an equivalent. This provides a stable measure of physical proximity independent of network topology.

## Path Metrics

When relays advertise neighbor topology (e.g., `neighbors` with `rtt_ms` and `pt_ms` per the schema in Appendix C), a client or orchestration system can:

1. **Build a relay graph**: Nodes = relays, edges = neighbor relationships with RTT/PT.
2. **Compute paths**: Shortest path by RTT, by PT, or by geographic distance.
3. **Apply GDOR constraints**: Filter paths to those that stay within required jurisdictions (using `country`, `subdivision`).

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

This document does not require IANA actions. If a well-known URI or a MoQ parameter type is registered in the future, the appropriate IANA registry would be updated.

--- back

# Appendix A. Example Topology (Informative)

The following example illustrates a typical relay mesh topology:

```
Home/Ent (NY)                    Relay Mesh                    SVC Media MS
    |                          A --- B
    |                           \   /
    +----(16ms RTT)-------------> D
    |                           /   \
    |                          C --- (to SVC)
```

- **Relay D**: JFK (NY), serves Home/Ent with 16ms RTT.
- **Relay A, B, C**: Interconnected; A connects to SVC.
- Geocode for each enables path selection (e.g., NY client → D → A → SVC) and GDOR (e.g., ensure D and A are in permitted jurisdictions).

# Appendix B. IATA Code Reference (Informative)

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
| GRU  | São Paulo |

Operators SHOULD use the official IATA code list for authoritative mappings.

# Appendix C. Relay Advertisement Format (Informative)

The JSON schema and example for relay geocode advertisement are defined in the related document `moq-relay-geocode-advertisement-format.md`, which accompanies this draft. Implementations may use that schema when advertising geocode via Option 2 (PUBLISH/SUBSCRIBE), Option 3 (Catalog), Option 4 (Metrics), or Option 5 (Discovery).

# Acknowledgments
{:numbered="false"}

The authors thank the participants of the Media over QUIC Working Group for
discussion and review feedback that helped shape this document.
