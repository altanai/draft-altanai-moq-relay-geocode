---
title: "Geographic Location for Media over QUIC Relays"
abbrev: "MoQ Relay Geocode"
category: std

docname: draft-altanai-moq-relay-geocode-latest
submissiontype: IETF
number:
date: 2026-11-25
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
    date: 2026
    seriesinfo:
      Internet-Draft: draft-jennings-moq-metrics-03
    target: "https://datatracker.ietf.org/doc/draft-jennings-moq-metrics/"
  RFC8801:
  RFC8805:
  RFC9388:
---

--- abstract

This document defines a mechanism for Media over QUIC (MoQ) relays to advertise their geographic location (geocode) and related path metrics. Some clients require their media data to remain locally or geo-fenced within specific jurisdictions for privacy and security compliance (e.g., GDPR, HIPAA, or sector-specific regulations). This mechanism enables service providers to track the geographic path of media packets through the relay mesh and to enforce geo-fencing policies. It supports Geo-Distributed Orchestration and Routing (GDOR), data residency compliance, latency optimization, and relay selection.

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
5. **Path tracking**: Enabling service providers to correlate hop-by-hop relay forwarding events with media objects for audits and policy enforcement.

## Design Goals

- **Relay identity in SETUP**: Each relay advertises its own stable identity and geocode to its directly connected peer in SETUP.
- **Hop-by-hop evidence**: Relays publish forwarding observations on a dedicated MOQT path-event track. The trace does not modify media Objects, Object Properties, or payloads.
- **Client-directed policy**: A client can express jurisdiction and proximity intent in SETUP. A relay applies authorization and policy, then uses MOQT migration or request redirection when a different relay is required.
- **Minimal protocol impact**: Reuse existing MoQ catalog, metrics, or discovery mechanisms where possible.
- **Interoperability**: Use standard geographic representations (WGS84, GeoJSON) and widely recognized identifiers (IATA codes).
- **Extensibility**: Allow optional altitude, region codes, and path metrics (SRTT, PT, bandwidth estimate, loss) without mandating every metric. Advertise stable relay identity and client intent in separate SETUP options; publish changing measurements on a metrics track.
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

## Sovereign Delivery and Client-Directed Geofencing

A client or service can require a media flow to use only relays in an allowed jurisdiction set. A client can state its policy intent when establishing a session, such as an allowed country or subdivision and whether it prefers a relay near its network attachment. The client does not need to disclose precise coordinates to request a nearby relay. Each relay advertises its own geocode in SETUP; a routing controller or relay uses authenticated relay identities and hop-by-hop path events to evaluate the selected route. If the current relay cannot satisfy the policy, it directs the client to an eligible relay using session migration or request redirection. If no trusted route satisfies a strict policy, the request fails closed.

The advertised geocode is an operator claim. This mechanism provides location and forwarding evidence for policy evaluation; it does not independently prove physical location or regulatory compliance.

## Path and Vicinity Awareness

In a relay mesh (e.g., A↔B↔C↔D), knowing the geocode of each relay allows:

- **Vicinity**: Computing great-circle distance or approximate propagation delay between relays.
- **Path**: Understanding the geographic route (e.g., NY → Relay D → Relay A → SVC) and optimizing it.

## Metrics Integration

Deployments commonly measure **RTT to Relay**, **PT (Propagation Time) to Relay**, bandwidth estimate, and loss as key metrics. Geocode complements these by providing a stable, policy-relevant attribute (location) that does not change with network conditions, while RTT, PT, bandwidth, and loss provide dynamic performance signals.

# Geocode Representation

This section is informative. The identity option uses exactly one primary representation: WGS84 coordinates or an IATA code. Country and subdivision codes are separate jurisdiction claims.

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

An IATA airport code is a human-readable metro-area identifier (e.g., JFK for New York, LHR for London). It may correspond to a nearby airport rather than the relay's facility. It is not a substitute for country or subdivision when enforcing jurisdiction policy.

# Integration with MoQ

This document proposes several integration options for geocode advertisement and path tracing.

## Relay Geocode Identity in SETUP

Each endpoint sends a SETUP message at the beginning of a MOQT session. A relay that advertises this capability MUST include its own `GEOCODE_IDENTITY` Setup Option in its SETUP message. In a relay-to-relay session, each relay sends its own identity. The option describes only the sender; it does not advertise the complete route or other relays.

The value is a JSON object encoded as UTF-8, without a byte order mark, as specified in {{RFC8259}}. It contains:

- `relay_id` (string, REQUIRED): Stable identifier for the relay.
- `geocode` (object, REQUIRED): Exactly one primary representation, either `{"type":"wgs84","latitude":40.6413,"longitude":-73.7781}` or `{"type":"iata","code":"JFK"}`.
- `country` (string, OPTIONAL): ISO 3166-1 alpha-2 jurisdiction code.
- `subdivision` (string, OPTIONAL): ISO 3166-2 subdivision code {{RFC9388}}.
- `location_precision` (string, REQUIRED): `exact`, `coarse`, or `jurisdiction-only`.

An IATA code identifies an airport or metro area and MUST NOT be used alone to enforce a jurisdiction boundary. An endpoint that requires jurisdiction enforcement uses the country or subdivision claim and its configured trust policy. A relay identity option is a claim by the sender, not independent proof of physical location.

Example (informative):

```json
{
  "relay_id": "relay-d",
  "geocode": { "type": "wgs84", "latitude": 40.7, "longitude": -74.0 },
  "country": "US",
  "subdivision": "US-NY",
  "location_precision": "coarse"
}
```

## Client Geocode Intent in SETUP

A client can include a `GEOCODE_INTENT` Setup Option to state its route policy. The value is a UTF-8 JSON object containing one or both of:

- `allowed_jurisdictions` (array of strings, OPTIONAL): ISO 3166-1 alpha-2 country codes or ISO 3166-2 subdivision codes.
- `near_me` (boolean, OPTIONAL): Requests a relay near the client's current network attachment. This flag does not convey client coordinates or assert a precise client location.
- `desired_geocode` (object, OPTIONAL): One WGS84 or IATA target using the same tagged representation as `GEOCODE_IDENTITY`.

The receiving relay MUST authenticate and authorize the request before applying it. A relay MUST NOT treat the intent as authorization to disclose relay or client location information. A relay that cannot satisfy a strict jurisdiction policy MUST NOT serve the affected request through a route that contains an unknown or disallowed relay.

If the client must move to another relay, the relay uses the MOQT migration mechanism appropriate to the scope: a control-stream GOAWAY with a new session URI for session migration, or a REDIRECT request error when redirecting an individual request. These mechanisms do not define route selection or policy; this document defines the geocode policy inputs used by the relay or controller.

## Well-Known Relay Tracks

This document defines the Track Namespace as the three fields `("relay-geocode.moq.arpa", "v1", <relay-id>)`. Under that namespace, the Track Names are `identity`, `metrics`, and `path-events`. These are MOQT tracks, not HTTP well-known URIs. Receivers discover and subscribe using MOQT namespace and track procedures. Identity is also advertised directly in SETUP so the directly connected peer can learn it without subscribing to a track.

Dynamic metrics MUST NOT be carried in SETUP because SETUP is exchanged only during session initialization. The `metrics` track carries timestamped samples, including reachability to named peers. Each reachability record identifies the adjacent relay and the associated measurements; it does not imply a fixed topology.

## Hop-by-Hop Path Events

A relay that advertises path-diagnostics support MUST publish a path event for each Object forwarding action covered by that capability. The relay MUST NOT modify the forwarded Object, its Object Properties, or its payload to produce the event. The event is carried on the relay's `path-events` track and contains:

- `object`: Full Track Namespace, Track Name, Group ID, and Object ID of the forwarded Object.
- `relay_id` (string, REQUIRED): Identifier matching the relay's `GEOCODE_IDENTITY`.
- `ingress_peer` and `egress_peer` (strings, REQUIRED): Identifiers of the adjacent peers for this forwarding action.
- `direction` (string, REQUIRED): `publisher-to-subscriber` or `subscriber-to-publisher`.
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

## Authorization and Policy Enforcement

The deployment policy determines which authenticated clients can provide geocode intent and which clients can subscribe to relay identity, metrics, and path-event tracks. Relays MUST validate the authorization scope before accepting intent or exposing diagnostic tracks. When the deployment requires strict geofencing, the controller MUST exclude routes with untrusted, unknown, or disallowed jurisdiction claims. A deployment that allows fallback MUST make the policy result visible to the client and in the path evidence.

# Vicinity and Path Computation

## Great-Circle Distance

Given two relays with geocode (lat1, lon1) and (lat2, lon2), the approximate great-circle distance in kilometers can be computed using the Haversine formula or an equivalent. This provides a stable measure of physical proximity independent of network topology.

## Path Metrics

When relays advertise reachability and timestamped metrics on the `metrics` track, a client or orchestration system can:

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

- Geocode reveals relay location. A relay's `location_precision` tells the receiver whether the claim is exact, coarse, or jurisdiction-only. Client `near_me` intent does not contain client coordinates, but the relay can still infer network location from the connection.
- Path events can reveal topology, peer identifiers, traffic volume, and user activity. Access to identity, metrics, and path-event tracks MUST be authorized and scoped to the requesting client or controller.
- Deployments that do not disclose facility coordinates can advertise only a jurisdiction or a coarse geocode. IATA codes are not precise locations and are not sufficient for jurisdiction enforcement.

## Integrity

- SETUP is integrity-protected by the authenticated transport session, but that alone does not establish that the relay's location claim is true. Compliance deployments MUST bind the relay identity to an operator trust process and validate the advertised jurisdiction against that identity.
- SRTT-to-geocode comparison is an anomaly signal only. Routing, congestion, asymmetric paths, and endpoint processing affect measured RTT; SRTT does not prove physical location.
- The path-event track records relay assertions about forwarding. Deployments requiring audit integrity need authenticated publishers and a trust policy for event records; a relay can omit or falsify its own observations.

## Misuse

- A malicious or compromised relay can advertise a false location, omit path events, or report misleading counters. A strict geofence MUST reject unknown or untrusted claims. A route trace is not independent proof of physical location or successful delivery.
- `GEOCODE_INTENT` reveals the client's jurisdiction policy and preference. Relays MUST apply authorization to this option and MUST NOT require precise client coordinates to honor `near_me`.

# IANA Considerations
{#iana}

## MOQT Setup Option Types
{#iana-setup}

This document requests registration of the following MOQT Setup Option types in the appropriate MOQT registry (see {{MoQTransport}}):

| Name | Value | Description | Reference |
|------|-------|-------------|-----------|
| GEOCODE_IDENTITY | TBD1 | Relay geographic identity (stable identifiers) | This document |
| GEOCODE_INTENT | TBD2 | Client jurisdiction and proximity intent | This document |

Until values are assigned, experimental use MUST use privately negotiated option types.

## Other Registrations

Changing metrics are published on the `metrics` track, not in SETUP. This document does not define a new MOQT Object Property or modify the MOQT Object or Subgroup Header formats. The Track Namespace and Track Names defined in this document are application identifiers carried in existing MOQT tracks.

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

The JSON schema and examples for relay identity, client intent, dynamic metrics, and hop-by-hop path events are defined in the related document `moq-relay-geocode-advertisement-format.md`, which accompanies this draft.

# Acknowledgments
{:numbered="false"}

The authors thank the participants of the Media over QUIC Working Group for
discussion and review feedback that helped shape this document.
