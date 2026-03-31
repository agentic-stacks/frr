# BGP Protocol Skill

## Overview

BGP (Border Gateway Protocol) is the core exterior gateway protocol in FRR, implemented by the `bgpd` daemon. It handles inter-AS routing, MPLS VPNs, EVPN fabrics, FlowSpec, and more. BGP is the most feature-rich protocol in FRR.

Key facts:

- Daemon: `bgpd` (must be enabled in `/etc/frr/daemons` with `bgpd=yes`)
- Default port: TCP 179
- Administrative distance: eBGP = 20, iBGP = 200
- Default timers: hold = 180s, keepalive = 60s
- Supports IPv4, IPv6, VPNv4, VPNv6, L2VPN EVPN, FlowSpec, labeled-unicast address families

## Sub-Topic Index

| File | Covers |
|---|---|
| [sessions.md](sessions.md) | Neighbor config, peer groups, timers, authentication, BFD, graceful restart/shutdown |
| [families.md](families.md) | Address families: IPv4/IPv6 unicast, VPNv4/v6, labeled-unicast, activate/deactivate |
| [policies.md](policies.md) | Route maps on BGP, prefix-list filters, AS path ACLs, soft reconfiguration |
| [communities.md](communities.md) | Standard/extended/large communities, community lists, route map usage |
| [route-reflectors.md](route-reflectors.md) | iBGP scaling with route reflectors, cluster-id, design patterns |
| [confederations.md](confederations.md) | Sub-AS design, when to use confederations vs route reflectors |
| [evpn.md](evpn.md) | EVPN address family, VNI, route types, VXLAN fabric configuration |
| [flowspec.md](flowspec.md) | FlowSpec address family, flow rules, redirect/rate-limit actions |

## Enable bgpd

Edit `/etc/frr/daemons`:

```
bgpd=yes
```

Reload FRR:

```
sudo systemctl reload frr
```

### bgpd Startup Options

Set in `/etc/frr/daemons` via `bgpd_options`:

| Flag | Purpose |
|---|---|
| `-p <port>` | BGP listen port (0 disables listening) |
| `-l <address>` | Bind to specific IP (repeatable) |
| `-n` | Do not install routes in kernel (route-reflector mode) |
| `-Z` | Disable zebra communication entirely |
| `-e <N>` | ECMP multipath limit |
| `-s <size>` | TCP socket send buffer size |

Example for a route reflector that should not install routes:

```
bgpd_options="--daemon -A 127.0.0.1 -n"
```

## Quick-Start: Configure a Basic eBGP Session

```
vtysh -c "configure terminal" -c "
router bgp 65001
 bgp router-id 10.0.0.1
 no bgp ebgp-requires-policy
 neighbor 10.0.0.2 remote-as 65002
 address-family ipv4 unicast
  neighbor 10.0.0.2 activate
  network 192.168.1.0/24
 exit-address-family
"
```

Replace:
- `65001` -- your local AS number
- `10.0.0.1` -- your router-id (typically a loopback IP)
- `10.0.0.2` -- the neighbor's IP address
- `65002` -- the neighbor's AS number
- `192.168.1.0/24` -- the network you want to advertise

**Note:** FRR enables `bgp ebgp-requires-policy` by default. This blocks all eBGP route exchange until you attach a route map. Use `no bgp ebgp-requires-policy` to disable this safety check during initial testing only.

## Essential Show Commands

### Session Status

```
! Summary of all BGP peers with state and prefix counts
show bgp summary

! Detailed info for one neighbor
show bgp neighbors 10.0.0.2

! Brief one-line-per-peer output
show bgp neighbors brief

! JSON output for automation
show bgp summary json
```

### Routing Table

```
! Full IPv4 unicast BGP table
show bgp ipv4 unicast

! Specific prefix lookup
show bgp ipv4 unicast 192.168.1.0/24

! Longer prefixes matching a range
show bgp ipv4 unicast 10.0.0.0/8 longer-prefixes

! All paths to a prefix (not just best)
show bgp ipv4 unicast 192.168.1.0/24 bestpath

! IPv6 BGP table
show bgp ipv6 unicast

! Show routes per VRF
show bgp vrf CUSTOMER1 ipv4 unicast
```

### Route Details

```
! Routes received from a neighbor
show bgp ipv4 unicast neighbors 10.0.0.2 received-routes

! Routes advertised to a neighbor
show bgp ipv4 unicast neighbors 10.0.0.2 advertised-routes

! Routes accepted from a neighbor (after policy)
show bgp ipv4 unicast neighbors 10.0.0.2 routes

! Community-filtered view
show bgp ipv4 unicast community 65001:100
```

### Diagnostics

```
! Show BGP running config
show running-config router bgp

! Why a route was not selected as best
show bgp ipv4 unicast 192.168.1.0/24 bestpath-debug

! BGP update activity
show bgp statistics

! Redistribution status
show bgp ipv4 unicast redistribute
```

## Operational Commands

### Clear Sessions

**WARNING:** Clearing BGP sessions drops peering and causes route reconvergence. This impacts live traffic.

```
! Hard reset -- tears down the TCP session (traffic impact)
clear bgp ipv4 unicast *
clear bgp ipv4 unicast 10.0.0.2

! Soft reset -- re-evaluates policies without dropping the session (safe)
clear bgp ipv4 unicast 10.0.0.2 soft in
clear bgp ipv4 unicast 10.0.0.2 soft out
```

### Save Configuration

```
write memory
```

## BGP Best Path Selection

BGP selects the best route using this ordered sequence:

| Step | Criterion | Prefer |
|---|---|---|
| 1 | Weight | Highest |
| 2 | Local preference | Highest (default 100) |
| 3 | Locally originated | Local routes |
| 4 | AS path length | Shortest |
| 5 | Origin | IGP < EGP < Incomplete |
| 6 | MED | Lowest |
| 7 | eBGP vs iBGP | eBGP |
| 8 | IGP metric to next-hop | Lowest |
| 9 | Multipath | If equal-cost |
| 10 | Router ID | Lowest |
| 11 | Cluster list length | Shortest |
| 12 | Peer address | Lowest |

Tune selection behavior:

```
router bgp 65001
 bgp bestpath as-path multipath-relax
 bgp bestpath compare-routerid
 bgp always-compare-med
 bgp deterministic-med
 bgp bestpath med missing-as-worst
 maximum-paths 4
 maximum-paths ibgp 4
```

## Default Timer Reference

| Timer | Default | Command to Change |
|---|---|---|
| Hold time | 180s | `neighbor PEER timers <keepalive> <hold>` |
| Keepalive | 60s | (set with hold time above) |
| Connect retry | 120s | `neighbor PEER timers connect <seconds>` |
| Advertisement interval (eBGP) | 0s | `neighbor PEER advertisement-interval <seconds>` |
| Advertisement interval (iBGP) | 0s | `neighbor PEER advertisement-interval <seconds>` |
| DelayOpen | disabled | `neighbor PEER timers delayopen <seconds>` |
| Graceful restart time | 120s | `bgp graceful-restart restart-time <seconds>` |
| Stale path time | 360s | `bgp graceful-restart stalepath-time <seconds>` |
| Route flap half-life | 15 min | `bgp dampening <half-life> ...` |
