# BGP Address Families

## Overview

BGP uses address families (AFI/SAFI) to carry different types of routing information over a single session. Each address family must be explicitly activated per neighbor.

## Available Address Families

| Address Family | AFI/SAFI | Purpose |
|---|---|---|
| `ipv4 unicast` | 1/1 | Standard IPv4 routing |
| `ipv4 multicast` | 1/2 | IPv4 multicast RPF |
| `ipv4 vpn` | 1/128 | VPNv4 (MPLS L3VPN) |
| `ipv4 labeled-unicast` | 1/4 | IPv4 with MPLS labels |
| `ipv4 flowspec` | 1/133 | IPv4 FlowSpec rules |
| `ipv6 unicast` | 2/1 | Standard IPv6 routing |
| `ipv6 multicast` | 2/2 | IPv6 multicast RPF |
| `ipv6 vpn` | 2/128 | VPNv6 (MPLS L3VPN) |
| `ipv6 labeled-unicast` | 2/4 | IPv6 with MPLS labels |
| `ipv6 flowspec` | 2/133 | IPv6 FlowSpec rules |
| `l2vpn evpn` | 25/70 | Ethernet VPN (VXLAN fabrics) |

## Activate IPv4 Unicast

By default, FRR activates `ipv4 unicast` for all neighbors (controlled by `bgp default ipv4-unicast`). To explicitly manage it:

```
router bgp 65001
 no bgp default ipv4-unicast
 neighbor 10.0.0.2 remote-as 65002

 address-family ipv4 unicast
  neighbor 10.0.0.2 activate
  network 192.168.1.0/24
 exit-address-family
```

**Recommendation:** Always set `no bgp default ipv4-unicast` and explicitly activate each family. This prevents accidentally advertising IPv4 routes to peers that should only carry other families (e.g., EVPN-only peers).

## Activate IPv6 Unicast

```
router bgp 65001
 no bgp default ipv4-unicast
 neighbor 2001:db8::2 remote-as 65002

 address-family ipv6 unicast
  neighbor 2001:db8::2 activate
  network 2001:db8:1::/48
 exit-address-family
```

### IPv6 Link-Local Peering

Peer using link-local addresses (common in data center fabrics):

```
router bgp 65001
 neighbor eth0 interface remote-as external

 address-family ipv6 unicast
  neighbor eth0 activate
 exit-address-family
```

FRR automatically discovers the link-local address on the named interface. This is the recommended approach for leaf-spine fabrics.

### IPv6 Next-Hop Preference

When a peer has both link-local and global IPv6 addresses, control which next-hop is used:

```
address-family ipv6 unicast
 neighbor 2001:db8::2 nexthop-local unchanged
exit-address-family
```

Or prefer global next-hops:

```
address-family ipv6 unicast
 nexthop prefer-global
exit-address-family
```

## Activate VPNv4 (MPLS L3VPN)

```
router bgp 65001
 no bgp default ipv4-unicast
 neighbor 10.0.0.2 remote-as 65001
 neighbor 10.0.0.2 update-source lo

 address-family ipv4 vpn
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 send-community extended
 exit-address-family
```

**Note:** VPN address families require extended communities for route-target import/export. Always enable `send-community extended` (or `both` / `all`).

### VRF-to-VPN Route Leaking

Export routes from a VRF into VPNv4:

```
router bgp 65001 vrf CUSTOMER-A
 address-family ipv4 unicast
  rd 65001:100
  route-target export 65001:100
  route-target import 65001:100
  redistribute connected
  redistribute static
  label vpn export auto
  export vpn
  import vpn
 exit-address-family
```

Replace:
- `CUSTOMER-A` -- VRF name
- `65001:100` -- route distinguisher and route target
- Adjust `redistribute` to match which routes should enter the VPN

## Activate VPNv6

```
router bgp 65001
 address-family ipv6 vpn
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 send-community extended
 exit-address-family
```

VRF-to-VPNv6 leaking follows the same pattern as VPNv4, under `address-family ipv6 unicast` in the VRF instance.

## Activate Labeled Unicast

Labeled unicast attaches MPLS labels to standard unicast prefixes:

```
router bgp 65001
 address-family ipv4 labeled-unicast
  neighbor 10.0.0.2 activate
  network 10.1.0.0/24 label-index 100
 exit-address-family
```

### IPv6 Labeled Unicast

```
router bgp 65001
 address-family ipv6 labeled-unicast
  neighbor 2001:db8::2 activate
 exit-address-family
```

## Activate L2VPN EVPN

See [evpn.md](evpn.md) for full EVPN configuration. Basic activation:

```
router bgp 65001
 address-family l2vpn evpn
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 send-community both
  advertise-all-vni
 exit-address-family
```

## Activate FlowSpec

See [flowspec.md](flowspec.md) for full FlowSpec configuration. Basic activation:

```
router bgp 65001
 address-family ipv4 flowspec
  neighbor 10.0.0.2 activate
 exit-address-family
```

## Deactivate an Address Family

```
router bgp 65001
 address-family ipv4 unicast
  no neighbor 10.0.0.2 activate
 exit-address-family
```

This stops route exchange in that family without tearing down the BGP session itself.

## Change the Default Address Family

Control which families are activated automatically for new neighbors:

```
router bgp 65001
 ! Disable automatic IPv4 unicast activation (recommended)
 no bgp default ipv4-unicast

 ! Or enable a different default
 bgp default ipv6-unicast
 bgp default l2vpn-evpn
```

## Announce Networks

### Static Network Announcement

```
address-family ipv4 unicast
 network 192.168.1.0/24
 network 10.0.0.0/8 route-map SET-COMMUNITY
exit-address-family
```

The prefix must exist in the routing table (connected, static, or from another protocol) unless `no bgp network import-check` is set.

### Redistribute Routes

```
address-family ipv4 unicast
 redistribute connected route-map CONNECTED-OUT
 redistribute static
 redistribute ospf route-map OSPF-TO-BGP
exit-address-family
```

### Aggregate Routes

```
address-family ipv4 unicast
 aggregate-address 10.0.0.0/16 summary-only
 aggregate-address 172.16.0.0/12 as-set
exit-address-family
```

| Option | Effect |
|---|---|
| (none) | Advertise aggregate AND specific prefixes |
| `summary-only` | Suppress specific prefixes, advertise only the aggregate |
| `as-set` | Include AS path information from contributing routes |
| `route-map NAME` | Apply route map to the aggregate |
| `matching-MED-only` | Only aggregate routes with matching MED |
| `suppress-map NAME` | Selectively suppress specific contributing routes |

## Per-Family Show Commands

```
! IPv4 unicast table
show bgp ipv4 unicast

! IPv6 unicast table
show bgp ipv6 unicast

! VPNv4 table
show bgp ipv4 vpn

! EVPN table
show bgp l2vpn evpn

! FlowSpec table
show bgp ipv4 flowspec

! Labeled unicast
show bgp ipv4 labeled-unicast

! Routes in a specific VRF
show bgp vrf CUSTOMER-A ipv4 unicast

! Summary per family
show bgp ipv4 unicast summary
show bgp ipv6 unicast summary
show bgp l2vpn evpn summary
```

## Multi-Family Session Example

A single session carrying IPv4 unicast, IPv6 unicast, and EVPN:

```
router bgp 65001
 bgp router-id 10.0.0.1
 no bgp default ipv4-unicast
 neighbor 10.0.0.2 remote-as 65001
 neighbor 10.0.0.2 update-source lo

 address-family ipv4 unicast
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 next-hop-self
 exit-address-family

 address-family ipv6 unicast
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 next-hop-self
 exit-address-family

 address-family l2vpn evpn
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 send-community both
  advertise-all-vni
 exit-address-family
```
