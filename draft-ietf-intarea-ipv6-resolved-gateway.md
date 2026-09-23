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
  RFC925:
  RFC1027:
  RFC1918:
  RFC1122:
  RFC2516:
  RFC3046:
  RFC3442:
  RFC6877:
  RFC7217:
  RFC7513:
  RFC8981:
  RFC8585:
  CALICO-FAQ:
    target: https://docs.tigera.io/calico/latest/reference/faq
    title: "Calico Documentation: Frequently Asked Questions"
    author:
      - org: "Project Calico"
    date: 2026
  I-D.ietf-intarea-v4-via-v6:
  RFC5737:
  RFC2132:
  RFC3527:
  RFC3927:
  RFC6105:
  RFC7600:
  RFC8925:
  I-D.ietf-v6ops-6mops:
  I-D.ietf-v6ops-claton:
--- abstract

This document specifies host behavior enabling IPv4 communication
for dual-stack hosts on IPv6-only segments, without subnets, ARP,
tunneling, or translation. Provisioning is unchanged: a host
obtains its IPv4 address and default gateway through ordinary,
unmodified DHCPv4, or by static configuration. Only the gateway
value changes. When that value is a reserved IPv4 sentinel
address, the host resolves the next-hop link-layer address from
the IPv6 neighbor cache rather than via ARP; no other aspect of
IPv4 operation is altered. IPv4 packets are forwarded natively,
end-to-end. The mechanism is incrementally deployable alongside
unmodified hosts with no changes to DHCPv4 infrastructure. This
document requests the allocation of one IPv4 address from
192.0.0.0/24 in the IANA IPv4 Special-Purpose Address Registry to
support this mechanism.

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
IPv4 address, `IPV4-SENTINEL`, that signals to a host stack that
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
at best.

The cost of a new DHCPv4 option is also not confined to client
stacks. An operator's billing, provisioning, inventory,
monitoring, NOC tooling and validation logic all encode what an
IPv4 gateway looks like, and a new option changes that shape in
every one of them at once. A sentinel address changes none of it:
to every system that reads or stores a lease it is an ordinary
IPv4 gateway address, syntactically and semantically valid, and
it passes through unaltered. The only component that has to know
the value is special is the host stack performing next-hop
resolution.

The sentinel address approach requires no changes to
DHCPv4 clients or servers and is incrementally deployable
today. Updated and unmodified hosts coexist on the same segment
indefinitely, and a segment may be converted one host at a time.
There is no flag day or mandatory switch-over point.

The mechanism has been verified to work without changes to
applications or DHCPv4 client configuration on Windows 11, macOS,
Android, iOS, Linux, FreeBSD, and ChromeOS.

This document addresses the host-side first-hop gap left open
by {{I-D.ietf-intarea-v4-via-v6}}, which defines router-to-router
forwarding of IPv4 traffic over IPv6 next-hops. Together, the
two documents provide a complete solution: hosts reach their
first-hop router without ARP, and routers forward IPv4 traffic
across an IPv6-only infrastructure without IPv4 addresses on
any router interface.

## Related Work

The underlying pattern -- a first-hop IPv4 gateway that is not
resolved by ordinary ARP against a shared on-link subnet -- has
been independently reinvented many times, in mutually
incompatible forms.

Proxy ARP {{RFC925}} {{RFC1027}} had a router answer ARP on
behalf of addresses that were not on the requesting host's link.
Point-to-point access architectures, among them PPPoE
{{RFC2516}} and 3GPP bearers, deliver an IPv4 address to a
terminal over a link with no subnet and no neighbor resolution
at all. The softwires IPv4-as-a-Service family {{RFC8585}}
carries IPv4 service across an IPv6-only access network to the
customer edge. Container networking reached the same shape
independently: Calico gives each workload a next-hop of
`169.254.1.1` that is never assigned to any interface and exists
only to be answered by proxy ARP on the host side of the
interface pair {{CALICO-FAQ}}. The hosting-provider /32
configurations described above are a further instance, as are
the pre-standard arrangements for routing IPv4 over IPv6
next-hops that preceded {{RFC8950}}.

Each of these is the same idea, constrained to one domain, and
each was previously kept there by a condition that has since
expired. There was no standardised transport for IPv4 routes
with IPv6 next-hops before {{RFC8950}}. Host stacks were treated
as unchangeable, an assumption disproved by the deployment of
{{RFC8925}} Option 108 and of CLAT across mainstream operating
systems. Every prior instance lived inside a walled domain -- a
single vendor, access technology, or orchestrator -- and so
never needed an interoperable code point. And IPv4 service on an
IPv6-only network was generally assumed to call for translation.

Those conditions no longer hold. The contribution of this
document is therefore not a new mechanism, but a general and
interoperable form of one that has repeatedly been built in
private and left undocumented.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the symbolic name `IPV4-SENTINEL` for the IPv4
sentinel address whose allocation is requested in {{iana}}. The
allocated value is not yet assigned; `IPV4-SENTINEL` denotes
whatever value IANA assigns.

\[RFC Editor: please replace all occurrences of `IPV4-SENTINEL`
with the IPv4 address assigned by IANA (TBD1), and remove this
note.\]

Where a concrete value is needed for illustration, examples and
figures in this document use `192.0.2.11`, drawn from the
TEST-NET-1 documentation range {{RFC5737}}. That value is
illustrative only; it is not the requested allocation and has no
special meaning.

# Problem Statement

IPv4 next-hop resolution on a local link depends on ARP, which
requires an IPv4 subnet to be configured on the link. In an
IPv6-only segment, no such subnet exists. Existing solutions
-- dual-stack, tunneling, and translation-based approaches
such as NAT64 -- generally require changes beyond the local
segment. This document closes the first-hop resolution gap
for dual-stack hosts on IPv6-only segments without requiring
changes to host software, packet formats, or DHCPv4 clients.

# Host Behavior and Next-Hop Resolution {#host-behavior}

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

When a host is configured to use `IPV4-SENTINEL` as its IPv4 default
gateway, the host's operating system MUST implement the following
logic:

1. The host MUST maintain a functional IPv6 Neighbor Discovery
   implementation per {{RFC4861}} on the same interface,
   including default router discovery and neighbor cache
   maintenance. No additional action specific to this mechanism
   is required at interface configuration time.
2. When the next hop for an IPv4 packet is `IPV4-SENTINEL`, the host
   MUST NOT perform ARP. Instead, it consults the IPv6 default
   router list and neighbor cache for the link-layer address,
   scoped to the interface on which `IPV4-SENTINEL` is configured
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

Host stacks MUST treat `IPV4-SENTINEL` as a sentinel address
signalling that IPv6-based next-hop resolution is to be used,
regardless of other address configuration on the interface.
This behavior is unconditional and not dependent on any
additional signaling.

When a DHCPv4 lease configuring `IPV4-SENTINEL` expires and
is not renewed, the host SHOULD remove `IPV4-SENTINEL` as the
IPv4 default gateway and cease IPv6-based resolution on that
interface. For statically configured deployments, removal
is governed by local administrative policy.

The following pseudocode defines the resolution logic:

~~~
on interface I (where IPV4-SENTINEL is the configured IPv4 gateway):

  if next-hop(pkt) == IPV4-SENTINEL:

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

## Multi-Homed Hosts {#multihomed}

Cross-interface resolution MUST NOT be performed. On multi-homed
hosts, each interface independently resolves `IPV4-SENTINEL`
against its own IPv6 neighbor cache state.

The sentinel is an interface-scoped next-hop token, not the
address of a particular router. It denotes "the IPv6 default
router on this interface" and carries no topological
information. Two interfaces configured with the same value
resolve it to two different link-layer addresses, by way of two
different IPv6 default router lists. This is the property IPv6
link-local next-hops already have, where a route is meaningful
only together with the interface it is attached to.

Every IPv4 route whose next hop is the sentinel is therefore a
triple, not a pair:

~~~
(destination prefix, outgoing interface, via IPV4-SENTINEL)
~~~

A host holding a sentinel default route on each of two
interfaces holds two distinct routes rather than a conflict:

~~~
0.0.0.0/0   dev if1   via IPV4-SENTINEL
0.0.0.0/0   dev if2   via IPV4-SENTINEL
~~~

They are disambiguated exactly as the equivalent IPv6 routes
with link-local next-hops are: by outgoing interface, and by
whatever metric or policy the host already applies when choosing
between two default routes. Resolution then proceeds per the
algorithm above, independently on each interface, against that
interface's own neighbor cache.

DHCPv4 needs no extension to express this. A DHCPv4 client
already keeps configuration per interface, and a lease obtained
on one interface supplies the Router Option (Option 3) default
for that interface alone. More specific routes are carried per
interface in the same way, through the Classless Static Route
Option (Option 121) {{RFC3442}}, whose router field accepts the
sentinel as it would any other next-hop value. Both sides of the
exchange are thus already interface-scoped; see
{{implementation}} for demonstrations.

Source address selection is unaffected by this mechanism. Once
an egress interface has been selected, the host chooses a source
address from that interface exactly as it does today. The
sentinel is never a candidate source address and never appears
in a forwarded packet (see {{ingress}}).

# Router Behavior

## End-to-End Packet Flow

In this model, end hosts are assigned IPv4 addresses with a /32
prefix length. No IPv4 prefix is configured on the link; a
sending host directs all IPv4 traffic to the first-hop router
using the link-layer address derived from the IPv6 neighbor cache.
A host cannot resolve another host's IPv4 address on the
local link without router assistance, so all intra-segment IPv4
traffic is forwarded by the first-hop router.

This mechanism does not rely on ICMPv4 redirects
({{RFC1122}}, Section 3.2.2.2). A redirect conveys a next-hop
address alone, and so cannot express the (destination,
interface, sentinel) form that next-hop resolution takes here
(see {{multihomed}}); on a multi-homed host the result would be
ambiguous. Redirects are in any case widely filtered or ignored
in current deployments. Where traffic is to be steered towards
one of several first-hop routers, that selection is made
through Default Router Preference {{RFC4191}} in Router
Advertisements, which operates per interface and is already
required by this mechanism for router selection.

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
{{RFC5737}} and are not globally routable.

~~~
Host A                      Router R1
IPv4: 198.51.100.1/32       (no IPv4 address configured)
IPv4 gw: 192.0.2.11         IPv6 link-local: fe80::R1
IPv6: 2001:db8:1::1

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

## Router-to-Host Binding Durability {#durability}

A first-hop router MUST be able, for the lifetime of a host's
IPv4 address assignment, to deliver IPv4 traffic to that host's
/32 without depending on opportunistically learned neighbor
cache state.

The failure mode this prohibits is the following. A router that
learns a host's link-layer address only as a side effect of
traffic, and holds the resulting IPv6 neighbor cache entry under
ordinary garbage collection, will discard that entry once the
host has been idle long enough. If the host's /32 route is
derived from that entry, the route is withdrawn with it, and
inbound traffic that would have prompted re-resolution never
arrives, because it is discarded upstream for want of a route.
Connectivity then does not recover, and the host cannot recover
it, since the host has no way to learn that it has become
unreachable from outside. Outbound traffic from the host
repairs the binding, but a host with no reason to send will
remain unreachable indefinitely.

Satisfying this requirement is a matter of router
implementation and operator configuration, and the mechanisms
are outside the scope of this document. They include deriving
the binding from DHCPv4 lease state rather than from traffic,
provoking Neighbor Discovery from that lease state, populating
the binding from an orchestration or route distribution system,
and anchoring it to a delegated prefix. A separate document is
expected to catalogue acceptable configurations.

## Router Ingress Behavior {#ingress}

Routers MUST treat `IPV4-SENTINEL` as an interface-scoped
address, valid only on the interface on which it is
configured, and only for locally-terminated traffic.
`IPV4-SENTINEL` does not appear in IPv4 fragment headers;
fragmentation behavior is unchanged by this mechanism.
Specifically:

- It MUST NOT be injected into any routing protocol.
- It MUST NOT trigger overlapping-subnet checks.
- It MUST NOT appear as source or destination in any
  forwarded packet.

A router MAY respond to ICMPv4 echo requests addressed to
`IPV4-SENTINEL` and MAY generate ICMPv4 Time Exceeded messages using
`IPV4-SENTINEL` as the source address. All such messages are
interface-local.

ICMPv4 error generation on IPv6-only transit routers is out of
scope; see {{RFC7600}}.

## Backward Compatibility: Router ARP Response {#arp-compat}

The use of `IPV4-SENTINEL` as the DHCPv4 Router Option (Option 3)
value is fully conformant with {{RFC2132}}, which imposes no
requirement that the router address be reachable via ARP on
the same subnet. Unlike {{RFC1027}} (proxy ARP), the router
is not proxying for a remote host; it owns this address on
the interface for the purpose of link-layer reachability.

Unmodified hosts receiving `IPV4-SENTINEL` as their IPv4 default
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
requests for `IPV4-SENTINEL`, so the router's ARP response
behavior is triggered only by unmodified hosts.

The ARP responder is migration machinery, and it is under
operator control. It exists so that unmodified hosts keep
working while a segment is being converted. An operator SHOULD
enable it for as long as unmodified hosts remain on the segment,
and SHOULD disable it once none do. The two-tier model is a
transitional state rather than the end state: on a fully
migrated segment no ARP is exchanged at all, the router's own
responder included.

The requirement in {{host-behavior}} that a host MUST NOT
perform ARP for the sentinel is correspondingly strict, and this
document deliberately defines no host-side fallback to ARP. A
sanctioned fallback would mean that no operator could safely
retire the responder on any segment, since a conformant host
might resort to ARP at any time. The zero-ARP end state would be
foreclosed permanently, and the return would be the masking of
a misconfiguration that is better surfaced (see
{{security-considerations}}).

# Deployment Considerations

This mechanism applies granularly at the segment level. A network
may contain a mix of IPv4-only, dual-stack, and IPv6-only
segments; this mechanism is applicable specifically to IPv6-only
segments carrying dual-stack hosts and does not affect other
segment types.

## Applicability

Hosts without a functional IPv6 implementation on the relevant
interface cannot perform Neighbor Discovery and are outside
the scope of this document.

## Relationship to Other Transition Mechanisms

This mechanism complements {{RFC8925}} (IPv6-Only Preferred
Option). RFC 8925 allows hosts to signal a preference for
IPv6-only operation, but operators must still provide IPv4
for hosts or applications that require it. This document
provides exactly that fallback: IPv4 connectivity on an
IPv6-only segment, without requiring a dual-stack
infrastructure or an IPv4 subnet on the local link. A host
that receives Option 108 and transitions to IPv6-only
operation retains functional IPv4 connectivity via
`IPV4-SENTINEL` without any additional configuration.

This mechanism fits within the IPv6-mostly network deployment
model described in {{I-D.ietf-v6ops-6mops}}, specifically
the case where native IPv4 connectivity is provided to
dual-stack hosts on an IPv6-only segment.

CLAT {{I-D.ietf-v6ops-claton}} and this mechanism serve
different demand profiles, and are not alternatives for the same
host. CLAT provides outbound IPv4 reachability by translating
into a pool shared at the PLAT; address sharing is the point of
it, and a host behind a CLAT has no IPv4 address of its own that
a remote peer can address. This mechanism provides the
complementary thing: a per-host IPv4 address that is routable
end to end, retains inbound reachability, and appears to the
host stack as ordinary IPv4 configuration. A host that needs
only outbound access to IPv4 services is well served by CLAT; a
host that is itself an IPv4 service endpoint, or that runs
software requiring a real IPv4 address, is not.

The two interact cleanly. A CLAT function SHOULD be disabled
when native IPv4 connectivity is available
{{I-D.ietf-v6ops-claton}}, and this mechanism supplies exactly
that native connectivity through ordinary DHCPv4. The existing
trigger therefore applies with no modification.

The per-host address need not be globally unique. On client
segments, a /32 drawn from {{RFC1918}} space with translation at
the network edge works with this mechanism unchanged. What the
host requires is a routable IPv4 identity on the segment, not a
public one.

## Access Provider Deployments

The common access-network case is a single public IPv4 address
per subscriber, delivered over an IPv6-only access network. This
mechanism addresses that case directly, and removes several
costs usually treated as unavoidable.

No IPv4 space is carved per service area, because no segment
carries an IPv4 prefix. No addresses are consumed by subnet and
broadcast overhead, and none are stranded by a mismatch between
a subnet's size and the number of subscribers actually attached
to it. ARP is not present at MAC-domain scale. The pool is flat:
any /32 may be assigned to any subscriber anywhere in the
network, and reclaimed and reissued without regard to topology.

Per-subscriber service differentiation follows from the same
property. One access segment can serve subscribers holding
public IPv4, subscribers holding addresses from shared CGN
space, and subscribers with no IPv4 at all, at the same time.
The distinction lies entirely in what DHCPv4 provisioning
returns, and nowhere in the configuration of the segment.

The model is not specific to one access technology. It applies
to FTTH and DOCSIS, to enterprise and campus wireless, and to
fixed wireless access over a mobile operator's IPv6-only
transport, in each case as a per-subscriber /32 over IPv6-only
infrastructure. An operator running several of these obtains a
single IPv4 service architecture across all of them. Handsets
remain a separate case, well served by 464XLAT {{RFC6877}}.

## Neighbor Discovery Prerequisites

Because IPv4 next-hop resolution now depends on Neighbor
Discovery, this mechanism inherits ND's operational
prerequisites. A fault in ND becomes an IPv4 fault as well as an
IPv6 one.

Specifically: ICMPv6 must be permitted by host firewalls, for
Neighbor Solicitation and Neighbor Advertisement as well as
Router Solicitation and Router Advertisement; multicast must be
delivered correctly across the segment, including by any MLD
snooping in the path, so that solicited-node multicast reaches
its target; and link-layer filtering must not discard frames
addressed to solicited-node multicast groups.

None of these is a new requirement. Each is already a
prerequisite for functioning IPv6 on the segment, and a segment
on which they are not met does not have working IPv6 service.
They are stated here because an operator whose IPv6 service is
nominally working, but whose ND health has never been examined
closely, may otherwise meet the dependency for the first time as
an IPv4 symptom.

## Intra-Segment Traffic

Because hosts carry /32 addresses, traffic between two hosts on
the same segment is forwarded by the first-hop router rather
than switched directly. The cost of this is topology-dependent,
and is frequently overstated.

Where the first-hop router is the access switch itself, as in an
L3 access design or a datacenter leaf, the forwarded path
traverses the same physical links the switched path would have
used, and is forwarded in hardware at line rate. The additional
cost is a forwarding lookup, not additional wire. The case that
does cost bandwidth twice is a router on a stick behind a shared
L2 domain, where each intra-segment packet crosses the uplink
once in each direction.

Guidance therefore follows the demand rather than the mechanism.
On the segments this mechanism most obviously suits -- access
and hosting segments, where lateral IPv4 between customers is
suppressed by policy or simply absent -- the cost rounds to
zero. On enterprise segments with an L3 access layer it is a
forwarding lookup. A segment carrying heavy east-west IPv4
traffic behind a shared L2 domain, reached by a router on a
stick, is the case that should not adopt this mechanism first:
such a segment is better left dual-stack, or moved to IPv6 for
its east-west traffic before the IPv4 first hop is converted.

## Layer 2 Integrity Features

DHCP snooping, Dynamic ARP Inspection and IP Source Guard were
specified against an assumption of a shared subnet with on-link
bindings. Their behaviour with an off-subnet gateway and /32
leases needs to be validated per platform before deployment.

The expected analysis is favourable. The first-hop router sits
on a trusted port, so its ARP reply for the sentinel is not
subject to inspection and passes unchanged. An unmodified host's
ARP request validates against that host's own sender binding,
which the DHCPv4 /32 lease supplies in the usual way. IP Source
Guard sees the leased address and permits it. None of this
depends on the gateway being a member of the host's subnet.

The residue is where attention is warranted. A first-hop router
on an untrusted port needs a static permit for the sentinel.
Platforms vary in whether inspection examines the ARP target
address as well as the sender, and in how gratuitous ARP is
treated. Some platforms carry binding machinery beyond these
three features, with its own subnet assumptions. Where a static
permit is required, a single universal entry suffices for an
entire deployment, because the sentinel holds the same value on
every segment.

This apparatus becomes unnecessary for IPv4 as migration
completes. Once no unmodified hosts remain and the router's ARP
responder has been retired, there is no ARP on the segment to
inspect, and host impersonation is an ND concern met by
ND-specific protections (see {{security-considerations}}).

## Multiple Gateways and Redundancy

Because the sentinel is resolved from the neighbor cache and
never by election, any number of routers on a segment may
present it at the same time, with no coordination between them
and no shared virtual address. Each updated host resolves the
sentinel to whichever IPv6 default router it has itself
selected. The requirement on the routers reduces to holding
distinct IPv6 identities, which they must do in any case. The
effect is anycast-like, though this is not anycast in the
technical sense: one value, many holders, and selection made
independently by each host.

That holds for the updated tier. Unmodified hosts still ARP for
the sentinel, and where more than one router answers, the last
reply received wins. The resulting nondeterminism is benign,
since every answering router is a valid first hop, but it is
nondeterminism nonetheless; an operator who wants a determinate
answer for the ARP tier can run a conventional FHRP in front of
it. That choice affects unmodified hosts only.

Every router a host may select MUST be able to forward that
host's IPv4 traffic, and MUST have a return path for it per
{{RFC8950}}. Where some routers on a segment qualify and others
do not, operators MUST steer host selection with Default Router
Preference {{RFC4191}}. On segments where several routers
advertise equal preference, as is common in datacenter ECMP
fabrics, hosts may select inconsistently according to RA timing;
operators SHOULD set preferences explicitly wherever a
deterministic outcome is wanted.

Return traffic needs no additional machinery. Where route
origination is configured correctly, every router on the segment
originates a route to each host /32, and inbound traffic is
distributed active-active by ordinary ECMP. The two directions
heal independently: the downstream path converges at the pace of
the routing protocol, and the upstream path at the pace of RA
processing and Neighbor Unreachability Detection on the host.
Operators running stateful functions on segment routers inherit
the usual consequences of ECMP path asymmetry, and should apply
the same measures they would in any other ECMP deployment.

No failover MAC address is involved. An updated host elects
nothing and inherits nothing; it follows its own IPv6 default
router selection, and a change of router is a change of
link-layer destination like any other. An unmodified host either
accepts the timing of its ARP cache expiring and re-resolving,
or sits behind a conventional FHRP as above.

More generally, the two things a first-hop redundancy protocol
is usually asked for are a gateway address that survives a
change of owner and a floating service address that does the
same. Both are the same requirement: an address that must appear
on-link while the device holding it may change. This model meets
the first by removing the address from the link altogether, and
the second by route origination, where the address is originated
by whichever member currently owns it, and originated
conditionally by the active member for stateful pairs.

## First-Hop Failover Pacing

IPv4 first-hop failover under this mechanism is paced entirely
by IPv6 default router selection. There is no IPv4-specific
detection mechanism and no IPv4-specific latency: whatever a
host's IPv6 first-hop failover behaviour is on a segment, its
IPv4 behaviour is now the same.

For a planned decommission, the departing router sends a final
Router Advertisement with a Router Lifetime of zero
({{RFC4861}}, Section 6.2.5). Hosts remove it from the default
router list on receipt and select another immediately, so a
planned failover need not involve detection at all. Operators
SHOULD ensure this advertisement is sent by any procedure that
stops an RA daemon or removes a router from service; where it is
omitted, a planned change degrades into the unplanned case.

For an unplanned failure, detection is by Neighbor
Unreachability Detection, which is traffic-driven. A host with
active flows through the failed router moves the neighbor entry
through DELAY and PROBE to FAILED and selects another router
within single-digit seconds, comparable to a conventional FHRP
in its default configuration. A host with nothing in flight
detects nothing, by design, and drops the router when its
advertised lifetime expires; the next packet it sends drives
resolution. Where sub-second failover is required it is
engineered rather than assumed, exactly as with a conventional
FHRP, by pairing the first hop with a liveness mechanism such as
BFD or by shortening RA intervals.

Two further notes. On platforms where the IPv4 and IPv6 stacks
are separate, the IPv4 path inherits whatever the operating
system's NUD implementation concludes, so failover timing is a
property of that stack rather than of this document. And DHCPv4
lease renewal or release does not affect next-hop selection: the
lease governs the host's address and the configured gateway
value, while resolution of that value is driven solely by
Neighbor Discovery.

## Return Path Provisioning

{{durability}} requires that a first-hop router be able to
deliver to each host /32 for the lifetime of the assignment. The
mechanisms are outside the scope of this document, but one
practical point is worth recording.

Lease-triggered origination is existing practice rather than a
new invention. Where the first-hop router is the DHCPv4 relay,
or observes relayed traffic, it already holds the lease binding;
together with the host's link-layer address from Neighbor
Discovery, that is sufficient to originate the {{RFC8950}} route
for the host's /32 when the lease is granted and to withdraw it
when the lease ends. Broadband network gateways originate
subscriber routes on exactly this basis today. This is an
illustration only: route origination belongs to {{RFC8950}} and
{{I-D.ietf-intarea-v4-via-v6}}, and nothing in this document
constrains how it is done.

The choice of IPv6 next-hop identity for those routes deserves
care. An IPv6 link-local address is not a stable identifier for
a host under current address-generation practice. An
implementation that moves from an interface-identifier-derived
address to a semantically opaque one {{RFC7217}} may do so
during interface bring-up, in the middle of the DHCPv4 exchange,
so a return route keyed to the first link-local address observed
can be stale before the host has finished configuring.

Operators SHOULD therefore key return routes to a stable
next-hop identity wherever one is available: a global address
the provisioning system already knows, an address anchored to a
delegated prefix, or a binding derived from ND snooping or a
source address validation mechanism {{RFC7513}}, using the
link-local address only as a fallback. Where only link-local
addresses are available, origination has to track changes of
identity through Neighbor Discovery rather than assume that the
first address seen persists. Where a global address is used, it
MUST be a stable one: temporary addresses {{RFC8981}} are
designed to rotate and are unsuitable as a next-hop identity.

## DHCPv4 Relay Considerations

A relay agent's interface toward the segment needs no IPv4
address, since the segment carries no IPv4 prefix. The relay
does need a routable IPv4 address for the giaddr field, but that
address need not be on the segment: one loopback address per
relaying router is sufficient, whatever the number of segments
it relays for.

Where giaddr would otherwise be conflated with address pool
selection, the two are decoupled by the link selection
sub-option {{RFC3527}}. In practice the need for this is
reduced, because flat /32 pools largely dissolve the question of
which pool serves which segment. Where segment identity is still
wanted for policy or accounting, it is carried by the relay
agent information option circuit identifier {{RFC3046}}.

Relay agents that enforce on-link gateway validation may reject
or flag `IPV4-SENTINEL` as an invalid Router Option value.
Operators SHOULD verify relay agent behaviour in their
deployment before relying on this mechanism.

ICMPv4 messages sourced by a first-hop router on the segment use
the sentinel and are interface-local (see {{ingress}}). ICMPv4
generated further along the path, by IPv6-only transit routers
with no interface-local sentinel to use, is a separate problem
addressed by {{RFC7600}}, which allocates `192.0.0.8/32` for
that purpose.

## Host Implementation Considerations

Implementations in which IPv4 and IPv6 stacks are managed by
separate processes (as is common on mobile operating systems)
will require inter-process communication to expose the IPv6
neighbor cache to the IPv4 forwarding path. This is an
implementation consideration and does not affect the on-wire
behavior defined in this document.


# Security Considerations {#security-considerations}

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

A host receiving `IPV4-SENTINEL` as its IPv4 default gateway
on a network that does not implement this mechanism will issue
an ARP request that receives no response, causing IPv4
connectivity to fail silently. Operators SHOULD ensure
`IPV4-SENTINEL` is only offered via DHCPv4 on segments where
the mechanism is deployed. DHCPv4 snooping and dynamic ARP
inspection, where used, MUST be configured to permit ARP
responses for `IPV4-SENTINEL` from the first-hop router.

This mechanism does not interact with IPv4 link-local address
configuration per {{RFC3927}}. A host configured with
`IPV4-SENTINEL` as its gateway and a link-local IPv4 source
address will follow the same resolution logic defined in
{{host-behavior}}.

## Universal Gateway Address

A consequence of the IANA allocation and the ARP behavior
defined in {{arp-compat}} is that `IPV4-SENTINEL` can serve as a
topology-independent gateway address in any deployment where
routers respond to ARP for it, not limited to IPv6-only
segments. This document does not specify or require this
behavior in IPv4-only deployments; it is an emergent property
of the allocation.

In segments where routers respond to ARP for `IPV4-SENTINEL`,
it functions as a universal gateway address. Unlike a
conventional per-subnet gateway address, it does not reveal
network topology and carries no subnet membership information.
ARP cache poisoning of `IPV4-SENTINEL` is less valuable than
poisoning a conventional gateway address, as it carries no
topological information for an attacker to exploit; it can
only redirect local-segment traffic, mitigated by dynamic ARP
inspection. Rogue RA attacks achieve the same redirection and
are mitigated by RA Guard {{RFC6105}}.

As `IPV4-SENTINEL` MUST NOT appear as source or destination in
any forwarded packet per {{ingress}}, conformant deployments
render it unreachable from any device not on the local segment.
This eliminates it as a target for off-link attacks. As
Source=False in the IANA registry (see IANA Considerations),
no conformant off-link device will originate packets with
`IPV4-SENTINEL` as source, precluding volumetric attacks using
this address.

IPv6 has long used specific link-local addresses (fe80::) as
next-hop addresses, topology-independent identifiers that
work on any segment without carrying subnet membership
information. `IPV4-SENTINEL` provides the same property for IPv4.

# Implementation Requirements {#implementation}

An implementation is conformant if it satisfies all MUST
and MUST NOT requirements in Sections 4 and 5.

A reference implementation in Linux userspace is available
at https://github.com/remcovanmook/v4-with-v6-nh.
A full conformance test suite will be documented prior to
IETF Last Call.

# IANA Considerations {#iana}

This document requests that IANA assign a single IPv4 address from
the 192.0.0.0/24 IETF Protocol Assignments block in the
"IANA IPv4 Special-Purpose Address Registry" {{RFC6890}}. The
address `192.0.0.11/32` is suggested. The assigned value is
referred to as `IPV4-SENTINEL` (TBD1) throughout this document.

| Field                | Value                          |
|----------------------|--------------------------------|
| Address Block        | TBD1 (suggested: 192.0.0.11/32) |
| Name                 | IPv4 Gateway via IPv6 Resolution |
| RFC                  | This document                  |
| Allocation Date      | (date of publication)          |
| Termination Date     | N/A                            |
| Source               | True                           |
| Destination          | True                           |
| Forwardable          | False                          |
| Globally Reachable   | False                          |
| Reserved-by-Protocol | False                          |

The Destination=True designation reflects that `IPV4-SENTINEL`
may appear as a destination in ICMPv4 messages received by
the router on a local interface (see {{ingress}}). It does
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
