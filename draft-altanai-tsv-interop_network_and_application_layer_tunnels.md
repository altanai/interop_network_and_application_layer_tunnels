---
title: "Interop between network and application layer tunnels"
abbrev: "TODO - Abbreviation"
category: info

docname: draft-todo-yourname-protocol-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
 - MASQUE
 - VPN
 - IPSec
 - Network Tunnelling protocols
venue:
  group: "Transport Area"
  type: "Area"
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

author:
 -
    fullname: Altanai Bisht
    organization: Cisco Meraki
    email: albisht@cisco.com

normative:

informative:


--- abstract

VPN not only provides secure communication over an untrusted network like site-to-site networking through the Internet, but also provides authentication, encryption and compression. VPNs are formed by tunnels which provide the secure medium for exchange.
Tunneling encapsulating and encrypting the original protocol with its own protocol.


--- middle

# Introduction


VPN not only provides secure communication over an untrusted network like site-to-site networking through the Internet, but also provides authentication, encryption and compression.
A path for information exchange on a network passes via many interconnected networks. This makes it vulnerable to rewriting or interception at various points.  Tunneling overcomes this as this process encapsulates the packet, including header and data of one protocol, inside the payload field of another protocol appending its own headers. This ensures the inner protocol  is protected in transit. Tunneling can build a secure interconnection called Virtual Private Network(VPN) that provides a private subnet to pass traffic between the tunneled endpoints. This setup caters to enterprises and smaller home networks alike. The egress points provided a gateway for outgoing traffic destined for the internet but only after anonymizing the source. The protocols supporting such network level tunneling include GRE, L2P, IPSec, OpenVPN and even proprietary protocols such as AutoVPN. With the advent of QUIC, multiple proxied stream- and datagram-based flows inside an HTTP connection, such as MASQUE, are quickly gaining quick popularity and widespread adoption. However there may be real world use-cases where network tunnels nest application tunnels, which leads to large latency and quality degradation.

Some instances of lower layer protocol

* Generic Routing Encapsulation (GRE) {{?RFC2784}}
* IP-in-IP {{?RFC1853}}
* IPSec
* WireGuard
* OpenVPN
* AutoVPN

Some instances of upper layer protocols

* SSH and  Secure Real-time Transport Protocol (SRTP) {{?RFC5764}} operate at the application layer.
* Stream Control Transmission Protocol (SCTP)
* DTLS.

- SSL/TLS tunnelling

- A L3 tunneling protocol, such as IPSec ({{?RFC4301}}), can provde secure path to almost all protocols in the TCP/IP suite. IPSec is a frame work of open standards to provide secure and private IP networks. Authentication Header (AH) and Encapsulating Security Payload (ESP) are part of the IPSec protocol suite, where Security Parameter Index(SPI) uniquely identifies a connetion and each packet in the flow bears a Sequence Number (SN) and a integtrity checksum called ICV (integrity check value). ESP packets inside UDP packets traversing through NATs {{?RFC3948}}.

- Segment Routing enables a network to enforce a flow by transporting unicast packets through a specific forwarding path. While ideally this would be decided by IGP shortest path or BGP best path. A segment can represent any instruction, topological or service-based

# Problem statement for nested tunnels

<!-- ![image](https://github.com/altanai/interop_network_and_application_layer_tunnels/assets/1989657/757a59d4-a63d-4284-aede-8fb2d66f99bc) -->


Among the wide scope of problems related to nested tunnels, here are the issues specific to multi layer tunnels alone

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

Control limits can be configured at multiple network nodes or levels of network stack, from applications to WAN port level. The simplest rate control techniques are based on thresholding. On an advanced level, traffic shaping policies are used for limiting traffic by steady state or burst rate. Rate limiting applied indifferently creates performance degradation especially for real time applications. For example a real time stream in a nested tunnel can be starved by middlebox's rate-control to adapt to bandwidth as cross traffic on the parent tunnel tries to occupy the majority share.
Pacing is a means to control the flow of packets from the sender to avoid such short term congestion. A perfectly paced sender spreads packets evenly over time, not exceeding the capacity of the network. However practical implementations of pacing often suffer from incorrect bandwidth estimation, packetization, scheduling delays or other computational overheads. At the network level any of the packets from the application whether carrying information or non-critical probing appear the same. A network tunneling protocol treats all the payload uniformly while applying start, recovery and congestion avoidance rules. However the loss or timeout of any probe packet could trigger aggressive congestion control from its host application such as increased redundant data on media encoder to overcome retransmission and so on. From the network perspective this fills up the congestion window and can then trigger rate limiting after some cycles. On the other hand absence of any intelligent rate control can result in underutilization of the congestion window.


## 5. Obfuscated prioritization

Since not all traffic is equally important to an organization, it is a common practice to prioritize some kinds of traffic. Different network service providers implement various kinds of prioritization logic on various levels of complexities. The classification can span from a low, medium or high bucket based approach to real time classification using machine learning models. Some prioritize traffic using static routing while others may tag or add markers such as DSCP. The markings were meant for L3 capable devices such as routers or multi-layer switches and designed to be persistent through the network for example Expedited Forwarding(EF) was meant for Voice traffic, where loss and subsequent retransmission is detrimental to user experience. However while tunneling the traffic through another tunnel, these efforts mostly goto vain by either getting overwritten or getting hidden under encapsulating protocol's header.

## 6. Multi-layer stats collection

Multiple applications or tunneling protocols may simultaneously be sending probe signals such as heartbeat, ping, hello or applying a custom feedback mechanism such as RTCP for stats collection. Besides the fact that leads to costly overhead, there can also be situations of information disparity. For example the loss observed by the network layer out of band telemetry may be high but the loss observed by the application layer’s in-band tunnel may be low. This leads to conflicting  observations of the network’s state.

## 7. Retransmission of lost packets

QUIC manages network stats for Loss Detection and Congestion Control {{?RFC9002}}. It detects loss by sending separate monotonically Increasing Packet Numbers as well as separate packet number spaces for each encryption level so that acknowledgment of packets is recognizable across levels of encryption. This prevents spurious retransmissions while still enabling fast retransmit.
On the other hand a L3 tunnel encapsulation such as ESP only adds a monotonically increasing sequence number in the header. All nested tunnels on this tunnel are subjected to HoL blocking as the packets wait in the transmission queue to be received and processed. While there exist no means to avoid spurious retransmissions from the sender, the receiver can use an anti-replay window to detect duplicates and discard.
A cumulative retransmission from both layers of a nested tunnel not only contributes to complexity in jitter and packet reordering, but also to latency and overhead by impacting other traffic's performance.

# Proposed Solution

Although nested tunnel are a realworld scenario which simplifies the configuration from an enterprise policy prespective and may appear more secure, we see from the probelem statement above that there are major tradeoffs when it comes to retransmission and managing overheads. The following is proposed detection mechanism for multiple tunneling layers and the communication model between the layers to overcome some of the shortcomings from nested tuneling.

## 1. Sync Congestion control across layers

QUIC does not mandate a particular congestion control yet suggests TCP Reno. Other techniques such as Cubic or bbr could also be applied and there is work in progress on other approaches too. However as mentioned in the problem statement the unsynchronised congestion control across layers is an issue for the latency or quality sensitive applications.
To overcome this, a synchronization mechanism is proposed which communicates the congestion control across layers. This would lead to layers deciding whether to completely mute their congestion control operation, prioritizing higher layer’s congestion controller when there is one available or select a variety of algorithms from many available that not only matches its own traffic requirements but also the network it is traversing. Alternatively this can help in path selection as well as nodes synchronize their congestion control parameters. For example a media streaming application with its own flow and congestion control can be put on a separate group for path selection than collectively prefer a delay based congestion control, so as to not conflict with other other groups of traffic.
The transport layer congestion controller may react faster to changes in the network as compared to application layer controllers. Although the usage of the information is upto the implementer, it would be advised to sync the congestion control algorithms across layers.

## 2. Share stats accross layers

Network related Stats are collected at various layers as defined in the problem statement 6. The picture derived from these instantaneous may sometimes be conflicting with the observations made at another layer. There can be no preference of one over another as different layers have their own pros and cons. For example
Timestamps collected at lower layers are more precise and better synced with NTP to avoid clock skews.
Lower layers also collect  RTT without being impacted by CPU cycles as is the case with application layer’s timestamps.
On the other hand application specific traffic management at higher layers is better at avoiding slow start and adapting to low bandwidth by a variety of techniques such as Bitrate ladder, encoder adding redundant data or just rate adaptation by pacing.
Hence it is proposed that the network layer should provide an interface for explicitly sharing collected network stats with higher layers such as applications. This will in turn lead to more fine tuning of the bandwidth and base RTT estimation across layers. For example QUIC datagram acknowledgements provide information on the send timestamps as well as round trip time to infer network transit delay. This can be shared with the application layer above such as a WebRTC application to infer and self-regulate its video quality and/or redundancy.

## 3. Firewall capabilities for end system

This solution provides host-based stateful packet filtering can provide some firewall capabilities at the end systems such as blocking IP traffic based on source and/or destination, specific protocols or ports.

## 4. Prioritization of traffic accrosss the layerd tunnels

To prevent the performance impact, degradation or starvation of certain data streams it is critical to apply prioritization. While application based prioitization can meet the requirnemnts here, Dynamic prioritization scehmes are more effective at balancing the load.

## 5. Trust model between the layers

The proposed inter- layer communication protocol provides the ability to exchange updated network state as well as packetization and congestion controller preferences. However network stats from the network layer may be aggregated and at a frequency different that what is required by the higher layer. Therefore as part of this synchronization protocol a flow and path identifier mechanism is suggested.

The design should
- apply policies accross the tunneled data for example if network administrator uses an IPSec policy that requires ESP encryption and communication only with trusted computers in the Active Directory domain, then the inner tunnels traffic should not be able to reach non listed servers.

The design should not
- cause bleeching of information where one actors remove the addiotnal secutity infornmation

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
There are concerns of deciphering user information from analysinf traffic

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
