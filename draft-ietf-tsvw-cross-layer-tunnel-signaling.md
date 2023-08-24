---
title: "Interop between network and application layer tunnels"
abbrev: "TODO - Abbreviation"
category: info

docname: draft-ietf-tsvw-cross-layer-tunnel-signaling-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
date: {DATE}
consensus: true
v: 3
area: "Internet Engineering Steering Group"
workgroup: "Transport Area"
keyword:
 - MASQUE
 - VPN
 - IPSec
 - Network Tunnelling protocols
venue:
  group: "Transport Area"
  type: "Area"
  mail: ""
  arch: ""
  github: "altanai/interop_network_and_application_layer_tunnels"
  latest: "https://altanai.github.io/interop_network_and_application_layer_tunnels/draft-ietf-tsvw-cross-layer-tunnel-signaling.html"

author:
 -
    fullname: Altanai Bisht
    organization: Cisco Meraki
    email: albisht@cisco.com

normative:

informative:


--- abstract

This document specifies a protocol mechanism for cross-layer signaling between nested tunnels and between tunnels and underlying network devices. The mechanism enables selective exchange of operational state (e.g., congestion signals, path MTU hints, priority hints, and loss/RTT metrics) while preserving end-to-end security and minimizing ossification risk. The document defines a compact TLV-based message format for the trust model between the networking layers and a mode of communication. It also defines negotiation semantics, bindings for representative stacks, and a security model suitable for deployment in common tunnel combinations.

--- middle

# Introduction

Enterprise and consumer deployments often layer multiple tunnels—an L3 VPN carrying QUIC-based application tunnels, or a MASQUE proxy nested inside an IPsec overlay—to combine security policies, traffic engineering, and endpoint features. Each layer introduces its own congestion control, pacing, probing, and telemetry logic. Without coordination, those stacked controllers frequently make conflicting decisions: middleboxes may throttle bulk flows just as an inner media tunnel ramps up, or application-driven retransmissions may amplify congestion signals already handled at the transport layer. These interactions are hard to diagnose and often degrade latency and throughput.

This draft proposes a lightweight, opt-in signaling channel that lets adjacent tunnel layers selectively exchange operational state (e.g., MTU, RTT/loss samples, priority hints) without exposing user payload. Rather than redefining existing tunnels, it defines how they can cooperate: a common TLV vocabulary, negotiation rules that protect security associations, and state machines that bound control traffic.

Specifically, the document contributes:

* A pragmatic taxonomy for L3/L4/L7 tunnels and transforms to reason about which signals each layer can provide or consume.
* A TLV-based message format, negotiation semantics, and security properties suitable for QUIC, MASQUE, IPsec/ESP, and similar tunnels.
* Guidance on transport bindings, pacing/rate sharing, and failure fallback so implementers can deploy the signaling channel without ossifying existing protocol stacks.

## Scope

This draft only specifies a signaling protocol and its bindings for enabling coordination between tunnel endpoints and between nested tunnel layers. It does not prescribe application policies, nor does it mandate changes to existing congestion control algorithms. The protocol is intended to be usable with L3, L4, and L7 tunneling technologies; the draft defines a taxonomy and provides example bindings for QUIC, MASQUE, and IPsec/ESP. Operational guidance and broad surveys of tunnel interactions are out of scope and should be moved to an Informational companion document.

This document uses the following taxonomy to classify tunnels and the endpoints that participate in cross-layer signaling. The taxonomy is intentionally pragmatic: it groups protocols by the layer at which they encapsulate traffic and by the kinds of operational signals they can reasonably provide or consume.

| Class      | Layer                   | Representative protocols                          | Typical signals                                         |
|------------|-------------------------|----------------------------------------------------|---------------------------------------------------------|
| L3 tunnel  | Network                 | IP-in-IP; GRE; IPsec ESP                           | MTU hints; outer path RTT/loss; ECN counters            |
| L4 tunnel  | Transport               | QUIC used as a tunnel; TCP-over-TCP; UDP encapsulation | Congestion window; pacing; RTT; loss events             |
| L7 tunnel  | Application/Session     | HTTP CONNECT; MASQUE; SSH port forwarding          | Application priority; flow intent; media QoS            |
| Transform  | Media/security transform| SRTP; DTLS; TLS tunnel modes                       | Per-flow crypto state; replay windows; timing constraints |


# Problem statement for nested tunnels

## 1. MTU Considerations
Every tunnel encapsulation entails appending extra header and/or footer to the data packets. This call for keeping the data packets small enough to make the headroom for nt exceeding the MTU of the path.
It is a challenge to

## 2. Multiple encryption overhead
Encryption provides protection against theft. While network tunnels may connect a user to a private network, it cannot protect against intentionally harmful behavior of a compromised endpoint, thereby exposing all inner sites to that endpoint. An application tunnel such as MASQUE maintains separate authorization for separate applications and thus with its own end to end encryption, reduces this attack surface as a compromised application endpoint cannot expose the information from other application tunnels. However, overlaying multiple layers of end to end encryption not only adds more compute but also time to read. Additionally rekey and other challenge-response mechanisms add more weight to the flow.

<!-- ![image](https://github.com/altanai/interop_network_and_application_layer_tunnels/assets/1989657/4e6d7895-9728-4390-8c31-d808d1eeb34c) -->


## 3. Multi layer Congestion control
Buffers are employed as means to store excess traffic during congestion periods which leads to queing. Queing can be implemented at ingress or egress points. It be as simple as FIFO, switch specific such as RR, WRR or router specific such as PQ,WFQ,CBWFQ, LLQ. More enhanced queing schmes involve AQM such as FQCODEL, PIE, CAKE. To overcome the congestion and optimize bandwidth usage there exist Congestion control algorithms. For example loss based transport layer congestion control such as TCP NewReno {{?RFC6582}} or CUBIC {{?RFC8312}} or delay based congestion control such as these at application layer, SCReAM, NADA or GCC (Google Congestion Control) for Real-Time Communication {{?I-D.ietf-rmcat-gcc-02}} which is already applied to many WebRTC implementations.
Issues arise when more than one congestion control algorithms are trying to adapt to the bandwidth and varaible congestion window.

## 4. Rate Control conflicts and flawed pacing

Rate limits can be imposed by applications, host stacks, middleboxes, or WAN links, and uncoordinated policies often starve latency-sensitive traffic inside nested tunnels. A middlebox reacting to bulk flow bursts, for example, can clamp the shared parent tunnel just as an inner real-time stream needs bandwidth, producing oscillations and excessive buffering.
Pacing helps smooth bursts, but practical implementations rarely have synchronized bandwidth estimates. Because outer tunnels treat all encapsulated packets uniformly, the loss of a single inner probe can trigger aggressive retransmissions or redundant encoding at the application, further bloating the congestion window. Conversely, overly conservative pacing wastes capacity. Nested tunnels therefore need a channel to exchange pacing budgets and rate intents so limits are applied once, at the most informed layer, instead of repeatedly at each encapsulation boundary.


## 5. Obfuscated prioritization

Since not all traffic is equally important to an organization, it is a common practice to prioritize some kinds of traffic. Different network service providers implement various kinds of prioritization logic on various levels of complexities. The classification can span from a low, medium or high bucket based approach to real time classification using machine learning models. Some prioritize traffic using static routing while others may tag or add markers such as DSCP. The markings were meant for L3 capable devices such as routers or multi-layer switches and designed to be persistent through the network for example Expedited Forwarding(EF) was meant for Voice traffic, where loss and subsequent retransmission is detrimental to user experience. However while tunneling the traffic through another tunnel, these efforts mostly goto vain by either getting overwritten or getting hidden under encapsulating protocol's header.

## 6. Multi-layer stats collection

Multiple applications or tunneling protocols may simultaneously be sending probe signals such as heartbeat, ping, hello or applying a custom feedback mechanism such as RTCP for stats collection. Besides the fact that leads to costly overhead, there can also be situations of information disparity. For example the loss observed by the network layer out of band telemetry may be high but the loss observed by the application layer’s in-band tunnel may be low. This leads to conflicting  observations of the network’s state.

## 7. Retransmission of lost packets

QUIC manages network stats for Loss Detection and Congestion Control {{?RFC9002}}. It detects loss by sending separate monotonically Increasing Packet Numbers as well as separate packet number spaces for each encryption level so that acknowledgment of packets is recognizable across levels of encryption. This prevents spurious retransmissions while still enabling fast retransmit.
On the other hand a L3 tunnel encapsulation such as ESP only adds a monotonically increasing sequence number in the header. All nested tunnels on this tunnel are subjected to HoL blocking as the packets wait in the transmission queue to be received and processed. While there exist no means to avoid spurious retransmissions from the sender, the receiver can use an anti-replay window to detect duplicates and discard. A cumulative retransmission from both layers of a nested tunnel not only contributes to complexity in jitter and packet reordering, but also to latency and overhead by impacting other traffic's performance.

# Proposed Solution

Nested tunnels are common in modern deployments, so the draft focuses on pragmatic coordination primitives instead of radically changing existing stacks. The solution space is organized around a lightweight TLV-based control plane that sits alongside existing tunneling protocols and offers the following architectural pillars:

1. **Capability-driven enablement:** Every tunnel layer first proves support with an authenticated opt-in handshake before exchanging any hints.
2. **Shared taxonomy:** L3/L4/L7 tunnels expose the same minimal TLV vocabulary, allowing implementations to reuse parsing logic even when traversing different combinations of tunnels.
3. **Context scoping:** Signals can be aggregated per outer tunnel or scoped per inner flow, depending on negotiated privacy and rate budgets.
4. **Fail-safe behavior:** If hints are missing or revoked, tunnels revert to their native operation without tearing down user traffic.

The rest of this section details how the protocol accomplishes those goals in concrete terms.

## Protocol design recommendations

The signaling channel transports only operational metadata using authenticated TLVs; it never carries user payload. Endpoints enforce opt-in and minimize wire footprint so the channel remains deployable even inside constrained outer tunnels.

### Transport bindings

* **QUIC-based L4/L7 tunnels:** Endpoints advertise support via a new QUIC transport parameter that includes rate limits and the allowed TLV set. Once both sides confirm support, they exchange control data either through a dedicated frame type (reliable delivery) or a reserved DATAGRAM flow ID (unordered delivery). All messages inherit QUIC confidentiality and integrity protection, and datagrams are sized <= Path MTU to avoid amplification.
* **Outer L3 tunnels:** If the outer tunnel exposes option space (GRE keys, ESP optional headers, SRv6 TLVs), signaling TLVs ride as authenticated options protected by the existing Security Association. For tunnels without option space, peers may encapsulate TLVs inside a tiny control packet (e.g., UDP payload with a well-known codepoint) that is encrypted/MACed to the tunnel endpoints. Implementations should rate-limit such packets to avoid DoS amplification.

### Message format

The wire format uses a canonical TLV envelope so middle layers can forward or aggregate signals without re-serializing:

* `Version` (8 bits) identifies the signaling revision.
* `Message Type` (8 bits) enumerates semantic buckets such as `CC_HINT`, `MTU_HINT`, `PRIORITY_HINT`, `STATS_SUMMARY`, and `KEEPALIVE`.
* `Length` (16 bits) encodes the total length of the payload in octets.
* `Payload`: sequence of nested TLVs, each carrying a specific metric (RTT in microseconds, ECN counters, loss in ppm, pacing fraction, etc.).

Messages MUST be authenticated (MAC or signature) and SHOULD be batched when possible. Unknown message types are ignored to reduce ossification risk.

### Negotiation

Negotiation happens during the parent tunnel’s setup procedure: QUIC transport parameters, IKEv2 exchanges for IPsec, WireGuard handshake TLVs, or MASQUE-specific HTTP headers. Each negotiation includes:

* allowed message types per direction;
* authentication mode (per-message MAC vs. signature piggybacked on existing keys);
* privacy level (aggregate per outer tunnel vs. per inner flow identifier);
* signaling rate policy (token bucket or fixed cadence) and loss/replay tolerance.

Endpoints drop any signaling observed before negotiation completes.

### State machine

The control plane follows a minimal four-state machine: `UNNEGOTIATED` → `NEGOTIATED` → `ACTIVE` → `TEARDOWN`.

* `UNNEGOTIATED`: default state; peers exchange capability TLVs within the parent handshake.
* `NEGOTIATED`: peers derive MAC keys, set rate limits, and send a `KEEPALIVE` probe. Receipt of an authenticated acknowledgment advances the state.
* `ACTIVE`: peers exchange operational hints. Senders retransmit lost control frames using capped exponential backoff. Rate policing suppresses bursts and ensures the TLV channel never exceeds negotiated ceilings.
* `TEARDOWN`: triggered when a peer withdraws support or consecutive probes fail. Pending hints are discarded, and endpoints revert to baseline tunnel behavior. Returning to `UNNEGOTIATED` requires a new handshake.

An ASCII sketch of the state diagram is shown below; annotations highlight the triggers that advance or regress the machine.

```
          +----------------+
          | UNNEGOTIATED  |
          +--------+------+
             |
      capability TLVs ok |
             v
          +--------+------+
          |  NEGOTIATED  |
          +----+---+-----+
            |   |
    KEEPALIVE ack       |   | capability revoked / timeout
            v   |
          +---+----+------+
          |     ACTIVE    |
          +---+----+------+
            |   ^
  persistent loss /     |   | exponential backoff exhausted
  rate limit violation  v   |
          +---+----+------+
          |    TEARDOWN   |
          +----------------+
```

## Cross-layer coordination themes

### 1. Congestion-control alignment

`CC_HINT` messages expose each layer’s congestion-control posture (algorithm identifier, current cwnd, pacing gain). Outer tunnels can choose to mute redundant algorithms or harmonize pacing windows so that only one layer performs multiplicative backoff. For example, a media tunnel running BBR can advertise its pacing window so the underlying QUIC tunnel avoids additional slow start. When hints stop arriving, peers revert to autonomous congestion control after a grace period $T_{grace}$ negotiated during setup.

### 2. Shared telemetry and health

`STATS_SUMMARY` aggregates RTT samples, loss ratios, ECN counters, and queueing delay estimates gathered at the outer tunnel. Application tunnels can ingest those stats to refine bitrate ladders or to decide whether to trigger key-frame retransmissions. The nested TLV format allows precise units (microseconds, ppm) so receivers do not misinterpret values.

### 3. Rate control and pacing hints

`CC_HINT` and `KEEPALIVE` messages double as pacing beacons. Outer tunnels can advertise remaining token-bucket capacity, enabling inner tunnels to throttle proactively rather than hitting hard outer limits. Conversely, an inner tunnel can mark urgent flows with `PRIORITY_HINT` so the outer tunnel applies low-latency queue disciplines or ECN marking thresholds tailored to that flow class.

### 4. Path and MTU agility

`MTU_HINT` TLVs communicate observed outer-path MTU and black-holed segments. Inner tunnels can promptly adjust segmentation and avoid fragmentation cascades. When the outer tunnel detects diverging paths for different inner flows, it can tag each hint with a flow-group identifier so only the affected inner tunnel reacts.

### 5. Policy and trust guarantees

Authentication modes ensure only legitimate tunnel endpoints inject hints. The protocol REQUIRES that any policy-relevant hint (e.g., `PRIORITY_HINT`) be signed or MACed; middleboxes may only aggregate or drop hints but cannot forge higher priority. Administrative policies from IPsec SAs or MASQUE routing tables can be mirrored into TLVs so that inner tunnels learn which destinations or DSCP values are permissible without exposing user payload.

### 6. Resilience and fallback

Control traffic adheres to negotiated rate limits and uses exponential backoff on loss to avoid congesting the very tunnels it tries to optimize. If the TLV channel stalls, endpoints continue user traffic uninterrupted but log telemetry gaps so operators can diagnose misconfiguration. Optional `KEEPALIVE` frames help detect path breakages faster than relying solely on data-plane loss.

### Example: MASQUE over IPsec tunnel stack

The following example illustrates how MASQUE flows multiplexed over QUIC can ride an IPsec tunnel while still exchanging cross-layer hints. The outer IPsec peers share MTU and ECN observations upward, while the MASQUE proxy returns application priorities and pacing budgets downward.

```
                                             +------+              +------+
                             MASQUE tunnel   |      |              |      |
                                             |      |              |      |
        +--------------+   +-----------------+------+              |      |          +---------+ stream1
        |  client 1    +---+----------------->      | MASQUE in    |      |          |         +---->
        |              |Stream               |      |              |      +----------+MASQUE   |
        |              |   Multiplxed        |      |  IPSec tunnel|      |stream    |         |
        |              |                     |      +--------------+      |multiplxed|Proxy    +---->
        |              +--------------------->      |              |      +----------+         |
        +--------------+   |                 |      |              |      |          |         | stream2
                           |                 |      +--------------+      |          +---------+
                           +-----------------+------+              |      |
                                             |      |              |      |
                                             |      |              |      |
        +--------------+   Stream            |      |              |      +----->
        |  client 2    +---------------------> IPSec|              |IPSec |       stream3
        +--------------+                     | VPN  |              |VPN   |
                                             | Peer |              |Peer  |
                                             |      |              |      |
                                             +------+              +------+
```

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations
There are numerous performance issues related to over encryption and conflicting algorithms in the probelm space. While there is large scope for improvements in the draft, the main crux is to build a trust model between the tunneling layers that avoids the need for multiple layers of cryptographic protocols.

## 1. Loss of sync messages
If a lower layer, such as L3 network tunneling, fails to share the network stats with higher layers, such as MASQUE in application layer, then it would impact the sender's ability to recognize congestion as effectively and provide markings.

## 2. Misuse of inter-layer messages
The application layer may notoriously try to mis-use the information from lower layers to enhance its rate adaptation or pacing unfairly.
The inner layer can also add ECN marking to all the traffic so as to win the prioritization.
Alternatively the outer tunneling protocols may mis-report information to the inner tunnels. For example the network tunnel may suggest RTT to be more the observed value so that the inner tunnel does not completely utilize the bandwidth.

## 3. Traffic analysis concerns
There are concerns of deciphering information from analyzing traffic patterns across layers however if the inter-layer synchronization mechanism is restricted to the sender side this concern is avoided.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
