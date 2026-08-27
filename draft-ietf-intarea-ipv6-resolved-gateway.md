---
title: IPv6-Resolved IPv4 Gateway
abbrev: IPv6-Resolved IPv4 GW
docname: draft-ietf-intarea-ipv6-resolved-gateway-latest
date: 2026-08-27
category: std
ipr: trust200902
area: "Internet"
workgroup: "Internet Area Working Group"
submissiontype: IETF
consensus: true
v: 3
keyword: Internet-Draft

venue:
  group: "Internet Area"
  type: "Working Group"
  mail: "int-area@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/int-area/"
  github: "remcovanmook/draft-ipv6-resolved-gateway"
  latest: "https://remcovanmook.github.io/draft-ipv6-resolved-gateway/draft-ietf-intarea-ipv6-resolved-gateway.html"

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: R. van Mook
    name: Remco van Mook
    org: Asteroid International B.V.
    email: remco@asteroidhq.com

normative:
  RFC4191:
  RFC4861:
  RFC6890:
  RFC8950:

informative:
  RFC1027:
  RFC1122:
  I-D.ietf-intarea-v4-via-v6:
  RFC5737:
  RFC2132:
  RFC3927:
  RFC6105:
  RFC7600:
  RFC8925:
  I-D.ietf-v6ops-6mops:
  I-D.ietf-v6ops-claton:
--- abstract

This document specifies host behavior enabling IPv4 communication
for dual-stack hosts on IPv6-only segments, without
subnets, ARP, tunneling, or translation. Hosts that receive
`192.0.0.11/32` as their IPv4 default gateway address resolve
the next-hop link-layer address from the IPv6 neighbor cache
rather than via ARP. IPv4 packets are forwarded
natively, end-to-end. The mechanism is incrementally deployable
alongside unmodified hosts with no changes to DHCPv4
infrastructure. This document
requests the allocation of `192.0.0.11/32` in the IANA IPv4
Special-Purpose Address Registry to support this mechanism.

--- middle

# Introduction

An IPv4 address functions as a service endpoint identifier --
either for a client seeking access to a service, or a server
providing one. The BSD socket API, present in virtually every
operating system and language runtime, expresses this directly:
a call to `connect(AF_INET, "192.0.2.1", 80)` is a statement
about which service to reach, not about routing or link-layer
resolution. The host implements IPv4 natively;
routers recognise and forward IPv4 packets and will continue to
do so. What this document changes is solely how the host resolves
the link-layer next-hop for the first hop.

Networks transitioning to IPv6-only segments still need to carry
IPv4 traffic for dual-stack hosts. Traditional mechanisms such as
dual-stack, tunneling, and translation all reintroduce IPv4 at
the infrastructure level. This document defines a sentinel
IPv4 address, `192.0.0.11`, that signals to a host stack that
link-layer resolution for the IPv4 default gateway is derived
from the link-layer address entry for the IPv6 default router in
the neighbor cache {{RFC4861}}, rather than via ARP. The IPv4
routing table entry is unchanged; only the next-hop resolution
path is modified. This eliminates the need for IPv4 subnets and
ARP on the local segment, removing the requirement for tunneling
or translation at the first hop.

This problem is already being solved in production, but
inconsistently. Hosting providers including Hetzner, OVH, and
Scaleway independently deploy /32 host addresses with off-link
IPv4 gateways using per-OS workarounds (Linux pointopoint, netplan
on-link: true, explicit post-up routes). None of this is
documented in any RFC, and the implementations are not
interoperable across providers. This draft standardises the
pattern with a single sentinel address that host stacks can
implement natively. Alternative
approaches (a new DHCPv4 option, or implicit behaviour when
no router is specified) would both require DHCPv4 client
changes across every OS implementation; given typical
deployment timescales, meaningful coverage would take a decade
at best. The sentinel address approach requires no changes to
DHCPv4 clients or servers and is incrementally deployable
today.

The mechanism has been verified to work without changes to
applications or DHCPv4 configuration on Windows 11, macOS,
Android, iOS, Linux, FreeBSD, and ChromeOS.

This document addresses the host-side first-hop gap left open
by {{I-D.ietf-intarea-v4-via-v6}}, which defines router-to-router
forwarding of IPv4 traffic over IPv6 next-hops. Together, the
two documents provide a complete solution: hosts reach their
first-hop router without ARP, and routers forward IPv4 traffic
across an IPv6-only infrastructure without IPv4 addresses on
any router interface.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Problem Statement

IPv4 next-hop resolution on a local link depends on ARP, which
requires an IPv4 subnet to be configured on the link. In an
IPv6-only segment, no such subnet exists. Existing solutions
-- dual-stack, tunneling, and translation-based approaches
such as NAT64 -- generally require changes beyond the local
segment. This document closes the first-hop resolution gap
for dual-stack hosts on IPv6-only segments without requiring
changes to host software, packet formats, or DHCPv4 clients.

# Host Behavior and Next-Hop Resolution

This mechanism activates only when a functional IPv6
implementation is present on the same interface, sufficient
to perform Neighbor Discovery per {{RFC4861}} and process
Router Advertisements. Without IPv6 on the interface, the
sentinel address has no effect and the host continues to
behave as if no gateway were configured.

For the outbound path, link-local IPv6 operation is sufficient.
For return traffic to reach the host via the
{{I-D.ietf-intarea-v4-via-v6}} forwarding model, the first-hop
router must advertise the host's /32 route with a routable IPv6
next-hop (GUA or ULA); a link-local address is not valid as an
inter-router next-hop. Operators deploying this mechanism in
conjunction with {{I-D.ietf-intarea-v4-via-v6}} MUST therefore
ensure the host has a routable IPv6 address on the interface.

A host implementing this mechanism SHOULD be configured with a
/32 prefix length for its IPv4 address. With a broader prefix,
the host will continue to ARP for addresses it considers on-link,
defeating the purpose of the mechanism for local segment traffic.
A /32 ensures all IPv4 traffic is directed to the first-hop router
via the sentinel, with no on-link ARP possible.

When a host is configured to use `192.0.0.11` as its IPv4 default
gateway, the host's operating system MUST implement the following
logic:

1. The host MUST maintain a functional IPv6 Neighbor Discovery
   implementation per {{RFC4861}} on the same interface,
   including default router discovery and neighbor cache
   maintenance. No additional action specific to this mechanism
   is required at interface configuration time.
2. When the next hop for an IPv4 packet is `192.0.0.11`, the host
   MUST NOT perform ARP. Instead, it consults the IPv6 default
   router list and neighbor cache for the link-layer address,
   scoped to the interface on which `192.0.0.11` is configured
   as the IPv4 default gateway.
3. If the IPv6 default router link-layer address is in a usable
   NUD state (REACHABLE, STALE, DELAY, or PROBE per {{RFC4861}}),
   the IPv4 packet is sent in a link-layer frame addressed to
   that destination.
4. If no reachable IPv6 default router is known after the
   interface has completed initial configuration (i.e., at
   least one RA has been processed), the packet MAY be queued
   or dropped per implementation policy. If a last-known
   router address is available, a Neighbor Solicitation
   SHOULD be sent to that address. For behavior prior to
   first RA reception, see the startup paragraph below.

Host stacks MUST treat `192.0.0.11` as a sentinel address
signalling that IPv6-based next-hop resolution is to be used,
regardless of other address configuration on the interface.
This behavior is unconditional and not dependent on any
additional signaling.

When a DHCPv4 lease configuring `192.0.0.11` expires and
is not renewed, the host SHOULD remove `192.0.0.11` as the
IPv4 default gateway and cease IPv6-based resolution on that
interface. For statically configured deployments, removal
is governed by local administrative policy.

Cross-interface resolution MUST NOT be performed. On multi-homed
hosts, each interface independently resolves `192.0.0.11` against
its own IPv6 neighbor cache state.

The following pseudocode defines the resolution logic:

~~~
on interface I (where 192.0.0.11 is the configured IPv4 gateway):

  if next-hop(pkt) == 192.0.0.11:

    routers = default_router_list(I)

    if routers is empty:
      if not first_ra_received(I):  /* startup: MUST queue */
        queue pkt
      else:                         /* mid-operation: MAY drop */
        queue or drop pkt
      send Router Solicitation on I  /* ff02::2; subject to
                                        RFC 4861 rate limiting */
      return

    selected = select from routers by:
      1. highest Default Router Preference (RFC 4191)
      2. NUD state == REACHABLE   /* if preference equal */
      3. implementation-defined   /* if reachability equal */

    lladdr = neighbor_cache(I, selected).lladdr

    if lladdr is valid and NUD state != INCOMPLETE:
      /* STALE, DELAY, and PROBE are usable; see RFC 4861 s7.3.3 */
      send pkt in link-layer frame with dst = lladdr
    else:
      queue or drop pkt           /* per implementation policy */
      send Neighbor Solicitation for selected router on I
~~~

Router selection uses Default Router Preference as defined in
{{RFC4191}}. When multiple routers have equal preference and
reachability, the tiebreaker is implementation-defined; use
of most-recently-heard RA is one reasonable approach.

If the selected IPv6 default router becomes unreachable during
an active session, the host SHOULD re-evaluate the default
router list and select an alternative. Existing transport
sessions will be disrupted if the link-layer next-hop changes;
this is consistent with IPv6 router failure behavior and is not
specific to this mechanism. Packets queued for a router that
has become unreachable SHOULD be flushed and re-evaluated
against the updated router selection.

Each time an interface transitions to an operational state
and begins RA solicitation, a host may receive DHCPv4
configuration before any RA has been processed. Until a
reachable IPv6 default router is known, IPv4 packets MUST
be queued rather than silently dropped, pending RA reception.
Implementations SHOULD bound this queue duration to avoid
indefinite resource consumption. On queue timeout, packets
SHOULD be dropped and an ICMPv4 Host Unreachable message
MAY be generated toward the sending application.

# Router Behavior

## End-to-End Packet Flow

In this model, end hosts are assigned IPv4 addresses with a /32
prefix length. No IPv4 prefix is configured on the link; a
sending host directs all IPv4 traffic to the first-hop router
using the link-layer address derived from the IPv6 neighbor cache.
A host cannot resolve another host's IPv4 address on the
local link without router assistance; direct host-to-host
IPv4 communication on the segment may occur via ICMPv4 redirect
({{RFC1122}}, Section 3.2.2.2) from the gateway, but cannot be
initiated by the host alone.

For return traffic to reach end hosts, operators MUST ensure
that host /32 routes with an IPv6 next-hop per {{RFC8950}}
are present in the routing infrastructure, allowing routers
to forward IPv4 traffic toward the correct first-hop without
requiring IPv4 addresses on any router interface. The mechanism
by which the routing infrastructure learns these host routes is
outside the scope of this document.

The following diagram illustrates the end-to-end packet flow.
Router-to-router forwarding uses {{RFC8950}}; no IPv4 address
is configured on any router interface. IPv4 addresses used in
the diagram are from the documentation ranges defined in
{{?RFC5737}} and are not globally routable.

~~~
Host A                      Router R1
IPv4: 198.51.100.1/32       (no IPv4 address configured)
IPv6: 2001:db8:1::1         IPv6 link-local: fe80::R1

  [1] IPv4 pkt (src: 198.51.100.1, dst: 203.0.113.5)
      L2 dst: MAC(R1) -- resolved from ND cache, no ARP
  ---------------------------------------------------->

Router R1                   Router R2
(no IPv4 address)           (no IPv4 address configured)
IPv6 link-local: fe80::R1   IPv6 link-local: fe80::R2

  [2] FIB lookup: 203.0.113.5/32 via fe80::R2 (RFC 8950)
      L2 dst: MAC(R2) -- resolved from ND cache, no ARP
  ---------------------------------------------------->

Router R2                   Host B
(no IPv4 address)           IPv4: 203.0.113.5/32
IPv6 link-local: fe80::R2   IPv6: 2001:db8:2::2

  [3] FIB lookup: 203.0.113.5/32 via fe80::HostB (ND)
      L2 dst: MAC(HostB) -- resolved from ND cache, no ARP
  ---------------------------------------------------->
  [4] IPv4 packet delivered to Host B
~~~

No ARP is exchanged at any point.

## Router Ingress Behavior

Routers MUST treat `192.0.0.11` as an interface-scoped
address, valid only on the interface on which it is
configured, and only for locally-terminated traffic.
`192.0.0.11` does not appear in IPv4 fragment headers;
fragmentation behavior is unchanged by this mechanism.
Specifically:

- It MUST NOT be injected into any routing protocol.
- It MUST NOT trigger overlapping-subnet checks.
- It MUST NOT appear as source or destination in any
  forwarded packet.

A router MAY respond to ICMPv4 echo requests addressed to
`192.0.0.11` and MAY generate ICMPv4 Time Exceeded messages using
`192.0.0.11` as the source address. All such messages are
interface-local.

ICMPv4 error generation on IPv6-only transit routers is out of
scope; see {{RFC7600}}.

## Backward Compatibility: Router ARP Response

The use of `192.0.0.11` as the DHCPv4 Router Option (Option 3)
value is fully conformant with {{RFC2132}}, which imposes no
requirement that the router address be reachable via ARP on
the same subnet. Unlike {{RFC1027}} (proxy ARP), the router
is not proxying for a remote host; it owns this address on
the interface for the purpose of link-layer reachability.

Unmodified hosts receiving `192.0.0.11` as their IPv4 default
gateway will issue an ARP request for it. A router SHOULD respond
to such ARP requests with its own MAC address. This is not proxy
ARP: no subnet exists, no remote host is being proxied.

This enables a two-tier deployment model on the same L2 segment:

- **Unmodified hosts:** router answers ARP; IPv4 forwarding works
  with zero host-side changes.
- **Updated hosts:** link-layer address resolved from IPv6 neighbor
  cache; ARP eliminated entirely.

Both tiers interoperate, allowing incremental deployment.
The router requires no per-host state to support both
tiers simultaneously: updated hosts will not send ARP
requests for `192.0.0.11`, so the router's ARP response
behavior is triggered only by unmodified hosts.

# Deployment Considerations

This mechanism applies granularly at the segment level. A network
may contain a mix of IPv4-only, dual-stack, and IPv6-only
segments; this mechanism is applicable specifically to IPv6-only
segments carrying dual-stack hosts and does not affect other
segment types.

This mechanism complements {{RFC8925}} (IPv6-Only Preferred
Option). RFC 8925 allows hosts to signal a preference for
IPv6-only operation, but operators must still provide IPv4
for hosts or applications that require it. This document
provides exactly that fallback: IPv4 connectivity on an
IPv6-only segment, without requiring a dual-stack
infrastructure or an IPv4 subnet on the local link. A host
that receives Option 108 and transitions to IPv6-only
operation retains functional IPv4 connectivity via
`192.0.0.11` without any additional configuration.

This mechanism fits within the IPv6-mostly network deployment
model described in {{I-D.ietf-v6ops-6mops}}, specifically
the case where native IPv4 connectivity is provided to
dual-stack hosts on an IPv6-only segment.

CLAT {{I-D.ietf-v6ops-claton}} provides an alternative approach
to IPv4 connectivity via translation. CLAT notes that a CLAT
function SHOULD be disabled when native IPv4 connectivity is
available; this mechanism provides exactly that native
connectivity, making CLAT unnecessary on segments where it
is deployed.

Hosts without a functional IPv6 implementation on the relevant
interface cannot perform Neighbor Discovery and are outside
the scope of this document.

On segments with multiple routers advertising equal Default
Router Preference (common in datacenter ECMP fabrics), hosts
may make inconsistent router selections based on RA timing.
Operators SHOULD configure explicit Default Router Preference
values per {{RFC4191}} to ensure deterministic behavior.

Implementations in which IPv4 and IPv6 stacks are managed by
separate processes (as is common on mobile operating systems)
will require inter-process communication to expose the IPv6
neighbor cache to the IPv4 forwarding path. This is an
implementation consideration and does not affect the on-wire
behavior defined in this document.

DHCPv4 relay agents that enforce on-link gateway validation
may reject or flag `192.0.0.11` as an invalid router option.
Operators SHOULD verify relay agent behavior in their
deployment before relying on this mechanism.

# Security Considerations

## ARP Attack Surface Reduction

In the updated-host deployment model, ARP is eliminated from the
segment entirely. In the unmodified-host model, ARP is constrained
to the sentinel address from a known source (the first-hop
router), eliminating the class of ARP-based network reconnaissance
and spoofing attacks that are possible in conventional subnet
deployments. Broadcast traffic is reduced to a single predictable
ARP exchange per unmodified host at startup, compared to
continuous ARP traffic across a conventional subnet.

The mechanism relies on the integrity of IPv6 Neighbor Discovery.
Rogue RA risks apply as in any IPv6 deployment and can be
mitigated with RA Guard {{RFC6105}}. Subnet scanning is
mitigated since hosts carry /32 addresses only.

A host receiving `192.0.0.11` as its IPv4 default gateway
on a network that does not implement this mechanism will issue
an ARP request that receives no response, causing IPv4
connectivity to fail silently. Operators SHOULD ensure
`192.0.0.11` is only offered via DHCPv4 on segments where
the mechanism is deployed. DHCPv4 snooping and dynamic ARP
inspection, where used, MUST be configured to permit ARP
responses for `192.0.0.11` from the first-hop router.

This mechanism does not interact with IPv4 link-local address
configuration per {{RFC3927}}. A host configured with
`192.0.0.11` as its gateway and a link-local IPv4 source
address will follow the same resolution logic defined in
Section 4 (Host Behavior and Next-Hop Resolution).

## Universal Gateway Address

A consequence of the IANA allocation and the ARP behavior
defined in Section 5.3 is that `192.0.0.11` can serve as a
topology-independent gateway address in any deployment where
routers respond to ARP for it, not limited to IPv6-only
segments. This document does not specify or require this
behavior in IPv4-only deployments; it is an emergent property
of the allocation.

In segments where routers respond to ARP for `192.0.0.11`,
it functions as a universal gateway address. Unlike a
conventional per-subnet gateway address, it does not reveal
network topology and carries no subnet membership information.
ARP cache poisoning of `192.0.0.11` is less valuable than
poisoning a conventional gateway address, as it carries no
topological information for an attacker to exploit; it can
only redirect local-segment traffic, mitigated by dynamic ARP
inspection. Rogue RA attacks achieve the same redirection and
are mitigated by RA Guard {{RFC6105}}.

As `192.0.0.11` MUST NOT appear as source or destination in
any forwarded packet per Section 5.2, conformant deployments
render it unreachable from any device not on the local segment.
This eliminates it as a target for off-link attacks. As
Source=False in the IANA registry (see IANA Considerations),
no conformant off-link device will originate packets with
`192.0.0.11` as source, precluding volumetric attacks using
this address.

IPv6 has long used specific link-local addresses (fe80::) as
next-hop addresses, topology-independent identifiers that
work on any segment without carrying subnet membership
information. `192.0.0.11` provides the same property for IPv4.

# Implementation Requirements

An implementation is conformant if it satisfies all MUST
and MUST NOT requirements in Sections 4 and 5.

A reference implementation in Linux userspace is available
at https://github.com/remcovanmook/v4-with-v6-nh.
A full conformance test suite will be documented prior to
IETF Last Call.

# IANA Considerations

This document requests that IANA assign `192.0.0.11/32` in the
"IANA IPv4 Special-Purpose Address Registry" {{RFC6890}} as follows:

| Field                | Value                          |
|----------------------|--------------------------------|
| Address Block        | 192.0.0.11/32                  |
| Name                 | IPv4 Gateway via IPv6 Resolution |
| RFC                  | This document                  |
| Allocation Date      | (date of publication)          |
| Termination Date     | N/A                            |
| Source               | True                           |
| Destination          | True                           |
| Forwardable          | False                          |
| Globally Reachable   | False                          |
| Reserved-by-Protocol | False                          |

The Destination=True designation reflects that `192.0.0.11`
may appear as a destination in ICMPv4 messages received by
the router on a local interface (see Section 5.2). It does
not imply global reachability; Forwardable=False and
Globally Reachable=False together preclude any use of this
address beyond the local link.

--- back

# Acknowledgements

The author thanks Tobias Fiebig, Warren Kumari, Jen Linkova,
David Lamparter, and Jordi Palet Martinez for their feedback
and review. An earlier version of this work was presented as
a lightning talk at RIPE 91 in Bucharest (October 2025) and
at IETF 124 in Montreal (November 2025).
