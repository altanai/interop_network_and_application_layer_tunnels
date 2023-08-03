---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
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

TODO Abstract


--- middle

# Introduction

- A L3 tunneling protocol, such as IPSec, can provde secure path to almost all protocols in the TCP/IP suite
IPSec is a frame work of open standards to provide secure and private IP networks.

- Segment Routing enables a network to enforce a flow by transporting unicast packets through a specific forwarding path. While ideally this would be decided by IGP shortest path or BGP best path. A segment can represent any instruction, topological or service-based

# Problem statement for nested tunnels 

Among the wide scope of problems related to nested tunnels, here are the issues specific to multi layer tunnels alone

## 1. MTU Considerations 
Every tunnel encapsulation entails appending extra header and/or footer to the data packets. This call for keeping the data packets small enough to make the headroom for nt exceeding the MTU of the path.  
It is a challenge to 

## 2. Multiple encryption overhead


## 3. Multi layer Congestion control 

## 4. Rate Control conflicts 
Bandwidth limits can be configured at multiple point in a node or network stack, from application level to WAN port level. The simple rate control techniques are based on threshold. Often traffic shaping polcies can be condifgured for limiting by steady state or burst rate. 

## 5. Obfuscated prioritization 
Different network service providers implement various kindes of prioritization logic on varioud levels of complexities. The classification can span from low, medium or high bucket based approach to realtime classification using machine learning models. Some prioritise traffic using static routing while others may tag or add marker such as DSCP. However while tunneling the traffic through another tunnel, these efforts mostly goto vain. 


# Proposed Solution 




## Trust model between the layers 

The design should not
cause blkeeching of information where one actors remove the addiotnal secutity infornmation 

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations
There are numerous performance issues related to over encryption in the probelm space. While there is large scope for improvements in the draft, the main crux is to build a trust model between the tunneling layers that avoids the need for multiple layers of cryptographic protocols.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
