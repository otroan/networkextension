---
title: Extending the Network - Host Requirements
docname: draft-troan-v6ops-extending-network-reqs-latest
date: 2023-10-05
ipr: trust200902
area: Internet
wg: v6ops
cat: info
v: 3

author:
   -
    ins: O. Troan
    name: Ole Troan
    org: Cisco Systems Inc.
    email: otroan@cisco.com
   -
    ins: N. Buraglio
    name: Nick Buraglio
    org: Energy Sciences Network
    email: buraglio@forwardingplane.net


normative:
  RFC2119:

informative:
  RFC7788:
  RFC8415:
  RFC4389:
  RFC6296:

--- abstract

This memo describes the behaviour of a node that when connecting to a network, offers connectivity via itself to other nodes (or entities needing addresses) connected to it. The focus is on IPv6 connectivity, but the same principles could be applied to IPv4.

--- middle

# Introduction

A node may have downstream links or host virtual machines or for other reasons extend the upstream network. This memo describes the requirements for such a node. The capabilities of the upstream network determines what mechanism the Extending Node (EN) can use. The EN may use one or more of the following mechanisms:

 - Homenet Control Protocol (HNCP) {{?RFC7788}}
 - DHCPv6 Prefix Delegation (PD) {{?RFC8415}}
 - Ethernet Bridging
 - ND Proxy {{!RFC4389}}
 - Address masquerading
 - NPTv6 {{RFC6296}}
 - NAPT66
 - Multilink Subnet routing

The upstream link, that is the network link that the EN is connected to upstream may or may not support protocols to aid in forther network extensions. The EN may posses some cpacity to detect the capabilities of the upstream link.
The EN acquires addresses on the upstream link as described in {{!RFC7084}}. The upstream link is either numbered via SLAAC, via DHCPv6 address assignment or it's "unnumbered". The EN should follow the requirements in {{RFC7084}}, specifically requirements W-1 to W-5.

Further extending the upstream network introduces a level of recursion into the network that does not otherwise exist. The mechanisms used by the first EN to extend the upstream network may potentially be used by another EN downstream to further extend the network, thus introducing the potential for recursion to the level of network resource exhaustion. Additionally, many of the mechanisms described do not have robust loop protection or detection, so caution must be exercised to prevent such loops from occurring. While the IPv6 address space is large, it is not infinite, therefore, assigning a /64 to each node is not feasible if the network is extended to enough levels, of the address plan did not account for such potential or if the initial network block is not sized appropriately. These considerations will determine if SLAAC is suitable for all the downstream links.

There are two essential problems with extending the network: addressing and routing. There are multiple approaches depending on the use cases and the limitations of the upstream network and implementations.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
{{!RFC 2119}}.

## Upstream link capabilities

- HNCP supported
- DHCPv6 PD supported
    - Multiple prefixes supported
    - Only a single prefix supported
    - Only /64 prefixes supported
- Ethernet bridging supported
- SLAAC: /64 prefix assignment supported. No address limits.
- SLAAC: /64 prefix assignment supported. Address limits.
- DHCPv6: /64 prefix with individual addresses assigned via DHCPv6.
    - Single address
    - Multiple addresses

A /64 prefix may be advertised on the upstream link for stateless address autoconfiguration (SLAAC), but it may still limit the number of addresses a node may use. Similarly, the network may use 802.1x authentication limiting MAC addresses a host may use, or control the addresses used by the node using DHCPv6 address assignment.

The network may or may not support DHCPv6 prefix assignment. If it does, it may support multiple prefixes, or it may support only a single prefix. It may support prefixes of any length, or it may support only /64 prefixes. If the network supports HNCP, it may restrict access to participate in the HNCP network.

The only mechanism that is guaranteed to work is one that is self-similar. That means that the upstream network cannot distinguish traffic from a downstream node from the traffic coming from the directly attached node itself. The only mechanism that is self-similar is NAPT66.

The EN must therefore go through a set of heuristics to determine which method works for a given network.

## Considerations

The EN must take address stability into consideration. While a single address is "simple" to renumber. Renumbering a network with a large number of addresses is not. Unless statically assigned, global addresses are ephemeral. Ahile IPv6 addresses and prefixes come with a lifetime, there is no guarantee [needs explained further --nb] as it has been seen in operational networks that networks keeps this promise. The EN must support renumbering of the downstream network in a methodical, repeatable, and consistent manner. Both "announced" via correct use of address/prefix lifetimes and "unannounced" by detecting that the upstream network has renumbered.
In addition the EN should support the use of ULA addresses on the downstream network. And connect that to the upstream network using one of NPTv6, NAT66 or NAPT66 depending on the circumstances.

In performing these functions, the EN operates as a router. It has a duality upstream, acting as a host to acquire addresses on the upstream interface. It acts as a router and forwards packets between interfaces. In addition to forwarding packets it must support acting as a router as specified in {{?RFC4861}}. It should also be able to act as a DHCPv6 server for addresses and prefixes.
For the purpose of source address selection, it must follow the weak host model. The upstream interface may be assigned only a link-local address and it that case the source address selection algorithm must pick a source address from another interface.

The EN doesn't necessarily know a priori how much address space it needs. It also depends on which addressing model that is used. A single /64 is enough to number an infinitely large downstream network. But if each host or container is supposed to be numbered with an individual /64 that will not scale well in many networks.

# Mechanisms for network extensions

## DHCPv6 Prefix Delegation

When used within an administrative domain DHCPv6 Prefix Delegation is better named DHCPv6 Prefix Assignment.
DHCPv6 PD allows the client to request a prefix of any length. To more effectively use address space, the EN should expect a "flat" PD deployment and not a hierarchical one. Meaning the EN should request a prefix (a /64) for each of it's downstream links. The upstream network decides to what extent it wants to adhere to the prefix hint given by the client. It may assign a /64 for each of the IA_PDs in the client's request or it may assign a single /64 or an even longer prefix to the EN.
Which downstream addressing methods are available to the EN is determined by the size of the prefix. E.g. if the EN is assingned a single /64 prefix, but has two downstream links, it has to use one of the DHCPv6 based addressing methods.

## Ethernet Bridging

Bridging the upstream Ethernet segment the downstream Ethernet segment makes all nodes on the downstream segment appear as if they are on the upstream segment. It will look like all nodes are on the same IPv6 link.
That assumes the downstream datalinks are Ethernet of course. We learnt back in the 80s that building huge Ethernet collision domains wasn't always a good idea. Protocols like ND and mDNS are very chatty and it exposes all downstream nodes to a lot of chattiness.
While a standard Ethhernet uses an MTU of 1500, many networks use jumbo frames. The EN must be able to handle jumbo frames.

Using bridging would also do nothing to alleivate the sizes required for the upstream network ND caches, or any limits upstream devices has in the terms of number of addresses they support.

## ND Proxy {{!RFC4389}}

This is when an address from the upstream prefix is used on a downstream link. The EN must defend this address and respond to ND address resolution requests for the address.

## Address masquerading

If only a single address is available on the upstream link, the EN may number the downstream network using ULA addresses and connect to the upstream network using NAPT66.

If there is a requirement of absoulte address stability in the downstream network NAT66 or NPTv6 is used. With the downstream network being numbered from the ULA address space.

Utilizing ULA addressing in a dual stacked network also implies that the address selection defined in {{RFC6724}} is followed and in the presence of IPv4 addressing and a destination A record, IPv6 will not be selected [This will chance if draft-ietf-6man-rfc6724-update is published as an update to RFC6724].

## NPTv6
[Stopping point 7-Oct-2023]

# Methods of address assignment

Depending on the size of the acquired prefix as well as user preference / configuration, the EN may use one of the following methods to assign addresses to downstream nodes.

These same mechanisms must be supported to acquire addresses on the upstream link. These are described in {{!RFC7084}}.

## SLAAC

A separate /64 prefix is assigned to each downstream link. The EN advertises the /64 in a PIO option in the RA, with the A-flag on.

## DHCPv6 address assignment

A separate IPv6 prefix of any length (larger than /128) is assigned to each link. The EN acts as a DHCPv6 IA_NA server and assigns addresses from the prefix to downstream nodes on that particular link. The EN also assigns an addresses to itself from the prefix on the downstream interface.
The EN advertises the prefix in an RA with the A-flag off, and the RA M-flag on.

## Prefixless Address Assignment (PLAA)

The EN is assigned a block of addresses (an IPv6 prefix).
The EN assigns addresses to downstream nodes without assigning a prefix to the downstream links. The EN acts as a DHCPv6 IA_NA server and assigns addresses from the block of addresses to downstream nodes. As addresses are assigned individually, the same address block (prefix) is used to number nodes on all downstream links. The EN installs a route in it's forwarding table for each address assigned.
The EN can also assign an addresses to itself from the block on a virtual interface.

The EN advertises itself as a default router in an RA. The M-flag is on, and the RA is sent without any PIO option.

## DHCPv6 prefix delegation

DHCPv6 prefix delegation (or more correctly prefix assignment when it is used within the same administrative domain) assigns a prefix to the attached node, not an address to be used on the interface attached to the link. The EN will install a route for the address (or prefix) assigned to the downstream node, with the nodes link-local address as next-hop. The downstream node needs to assign the address or create an address from the prefix on a virtual interface. The EN will not do ND (address resolution) on the downstream link for the assigned address or prefix.

The downstream link can be unnumbered as in the above case. The EN advertises itself as a default router in an RA. The M-flag is on, and the RA is sent without any PIO option.

A downstream node should request a prefix if it is capable to do so. It should also request an address using the IA_NA option if it is capable to do so.

DHCPv6 prefix delegation can be seen as a superset of DHCPv6 address assignment. As it can also assign individual addresses as well as address prefixes.

## Manual configuration

In many cases the network extensions are containers or virtual machines running directly on the EN. In these cases as well as to number it's own interfaces it can assign addresses to those directly. Using whatever host specific tools exist for this. Unless the numbered entities need to be onlink to each other, prefixless address assignment should work well for this.

# Topics for further considerations

Support for arbitrary topology downstream. Each node participates in a routing protocol (if trusted) and advertises it's own address or prefix in the downstream network.
Each node acts as a DHCPv6 relay. Sending all DHCPv6 requests to the "root" EN.

To extend the network a node should try in preference order:
1) HNCP
2) DHCPv6 PD
3) Ethernet Bridging

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Security Considerations
=======================
TBD

# IANA Considerations
TBD

# Acknowledgments

The following people provided advice or review comments that
substantially improved this document: Brian Carpenter, Owen DeLong

# Text to add
Owen: “Any of the mechanisms which do not provide complete end-to-end address transparency and connectivity should be considered sub-standard and implemented only in circumstances where no better alternative exists.”

Brian:
To avoid a polemic here, maybe the draft should draw a clear line between solutions that do preserve e2e addresses and those that don't, with a note that the latter bring with them some or all of the documented issues with NAT. The topic is nicely described at https://www.rfc-editor.org/rfc/rfc6296.html#section-5 and of course in RFC 2993.

BFD keepalive.
Two pronged approach for the PD to detect possible lost forwarding state.
- Monitor counters for forward progress
- Send probes to itself and see if they come back.