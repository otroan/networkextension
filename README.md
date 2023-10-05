# networkextension
IETF draft on Network Extensions

## Abstract
Extending the network is a common use case. If it's just a user's host connecting having VMs, if it's tethering, or if it's attaching a network device to an existing network.

## Introduction
How can a host extend the network? How to acquire a "presence" on the northbound link?

1. Participate in HNCP and get assigned one or more prefixes via the HNCP protocol
2. Act as a DHCPv6 PD client and get assigned a prefix
3. Ethernet bridging
4. Take an address from the northbound link (ND proxy)
5. Share the address(es) of the host.
	1. Address sharing / stealing
	2. NAPT66

## Issues to consider
1. Address stability
2. Upstream link capability detection.
	1. Does it support multiple addresses?
	2. SLAAC supported? How many addresses does it support? Hard to detect?
	3. If PD, prefix size, how many prefixes?

How to address southbound interfaces?
1. Direct assignment from an acquired prefix?
2. Using SLAAC
3. Using DHCPv6 IA_NA
4. Using DHCPv6 IA_PD

Does the acquired prefix have to be a /64?
Does the downstream prefix have to be a /64?

"Multilink-subnet routing"?
In IPv6 an address does not have to be part of a prefix. Routing is achieved by using the link-local prefix, that's always directly connected.

The network extending device is either a full blown router or an Ethernet switch or a NAT or something else.

To extend the network a node should try in preference order:
1) HNCP
2) DHCPv6 PD
3) Ethernet Bridging


## Southbound addressing

