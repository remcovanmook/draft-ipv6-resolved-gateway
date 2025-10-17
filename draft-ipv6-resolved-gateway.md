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
title: "IPv6-Resolved IPv4 Gateway"
category: info

docname: draft-ipv6-resolved-gateway-latest
submissiontype: IETF
date:
consensus: true
v: 3
area: Internet
workgroup: Network Working Group
keyword:
 - ipv6
 - ipv4
 - gateway
 - special-purpose address
venue:
  group: Network Working Group
  type: Working Group
  mail: remco@asteroidhq.com
  github: remcovanmook/draft-ipv6-resolved-gateway

author:
 -
    fullname: Remco van Mook
    organization: Asteroid International B.V.
    email: remco@@asteroidhq.comom

normative:
  RFC4861:
    title: Neighbor Discovery for IP version 6 (IPv6)
  RFC8200:
    title: Internet Protocol, Version 6 (IPv6) Specification

informative:

...

--- abstract

This document requests the allocation of a new IPv4 special-purpose address from the IANA IPv4 Special-Purpose Address Registry. The proposed address, 192.0.0.11/32, is intended to serve as a signal to IPv4 hosts in IPv6-only networks that the link-layer resolution for the default gateway should be derived from the IPv6 default gateway learned via IPv6 Router Advertisements and Neighbor Discovery.

This approach enables IPv4 communication without requiring IPv4 subnets or the use of ARP. It maintains backward compatibility with existing IPv4 host software that expects a default gateway IP address, while avoiding the need to implement legacy link-layer protocols.

--- middle

# Introduction

In IPv6-only infrastructure environments, such as modern data centers and ISP networks, IPv4 communication may still be required by applications or systems. However, traditional IPv4 mechanisms like ARP and subnet configuration impose unnecessary complexity in such environments.

Hosts in these environments typically receive IPv6 configuration through SLAAC or DHCPv6, including a default gateway. This document proposes a method by which IPv4 traffic may also be sent without requiring ARP or an IPv4 subnet: by configuring a well-known IPv4 address (192.0.0.11) as the default gateway, and resolving its link-layer address using the IPv6 default gateway learned by the host.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Rationale

The key goal is to enable IPv4 communication in environments that are natively IPv6-only, without relying on dual-stack or tunneling. This is accomplished by decoupling IPv4 next-hop resolution from ARP and instead aligning it with the IPv6 default gateway.

By defining 192.0.0.11 as a special-purpose IPv4 address, hosts can be configured with IPv4 /32 addresses and this default gateway, eliminating the need for any IPv4 subnet or address resolution mechanisms.

# Host Behavior and Next-Hop Resolution

When a host is configured to use 192.0.0.11 as its IPv4 default gateway, the host's operating system should implement the following logic:

* Upon startup or interface configuration, the host listens for IPv6 Router Advertisements (RAs) and records the IPv6 default gateway and associated link-layer address via Neighbor Discovery.

* When sending an IPv4 packet where the next hop is 192.0.0.11, instead of performing an ARP resolution, the host stack consults its IPv6 neighbor cache for the link-layer address associated with the IPv6 default gateway.

* If the IPv6 default gateway is known, and the link-layer address is valid and reachable, the IPv4 packet is sent directly using that link-layer destination address.

* If the IPv6 gateway is not yet known or reachable, the IPv4 packet should be queued or dropped per implementation policy, and a Neighbor Solicitation initiated for the IPv6 gateway.

# Compatibility Considerations

* Hosts continue to use standard IPv4 protocol semantics and packet formats.
* Applications requiring IPv4 continue to function as expected.
* No changes are required to the IPv4 packet format.
* The only change is that 192.0.0.11 is interpreted by the host stack as an indicator to use the link-layer information from the IPv6 default gateway.

# Security Considerations

This approach reduces ARP-related attack surfaces by removing ARP from the network. It assumes integrity of IPv6 neighbor discovery, and any associated risks (e.g., spoofed RAs) are equivalent to standard IPv6 host risks.

Additionally, subnet scanning attacks against IPv4 networks are mitigated, since hosts are only configured with /32 addresses and ARP is not available to discover neighbors.

# IANA Considerations

This document requests the following addition to the IANA IPv4 Special-Purpose Address Registry:

| Address Block | Name | RFC | Allocation Date | Termination Date | Source | Destination | Forwardable | Global | Reserved-by-Protocol |
|--------------|------|-----|-----------------|------------------|---------|-------------|-------------|--------|-------------------|
| 192.0.0.11/32 | IPv6-Resolved Default Gateway | [This document] | [To be assigned] | N/A | False | True | True | No | No |

--- back

# Acknowledgments
{:numbered="false"}

The author would like to thank Tobias Fiebig and Warren Kumari for their input on this document.
