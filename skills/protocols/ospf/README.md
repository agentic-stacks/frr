# OSPF Protocol Skill

## Overview

OSPF (Open Shortest Path First) is a link-state interior gateway protocol (IGP) in FRR, implemented by the `ospfd` (OSPFv2) and `ospf6d` (OSPFv3) daemons. OSPF computes shortest-path-first routes using Dijkstra's algorithm and organizes the network into areas for scalability.

Key facts:

- Daemons: `ospfd` for IPv4, `ospf6d` for IPv6 (enable in `/etc/frr/daemons`)
- Protocol: IP protocol 89 (not TCP/UDP)
- Administrative distance: 110
- Multicast addresses: 224.0.0.5 (AllSPFRouters), 224.0.0.6 (AllDRouters)
- Default hello: 10s, dead: 40s (broadcast/point-to-point)
- Metric: cost = reference-bandwidth / interface-bandwidth (default ref: 100 Mbps)

## Sub-Topic Index

| File | Covers |
|---|---|
| [ospfv2.md](ospfv2.md) | OSPFv2 full config: router-id, network/interface config, area types, virtual links, auth, timers, cost, passive interfaces, redistribution, graceful restart, OSPF-TE, segment routing |
| [ospfv3.md](ospfv3.md) | OSPFv3 differences from v2: IPv6, interface-based config, instance IDs, IPsec/keychain auth, area config, redistribution |

## Enable OSPF Daemons

Edit `/etc/frr/daemons`:

```
ospfd=yes
ospf6d=yes
```

Reload FRR:

```
sudo systemctl reload frr
```

### ospfd Startup Options

Set in `/etc/frr/daemons` via `ospfd_options`:

| Flag | Purpose |
|---|---|
| `-a` | Enable the OSPF API server |
| `-n <instance>` | Run as a specific OSPF instance |

### ospf6d Startup Options

Set in `/etc/frr/daemons` via `ospf6d_options`:

| Flag | Purpose |
|---|---|
| Standard FRR flags | `-A`, `-d`, `-f`, `-z` etc. |

## Quick-Start: Configure Single-Area OSPF

```
vtysh -c "configure terminal" -c "
router ospf
 ospf router-id 10.0.0.1
 network 10.0.0.0/24 area 0
 network 192.168.1.0/24 area 0
 passive-interface eth1
"
```

Replace:
- `10.0.0.1` -- your router-id (typically a loopback IP)
- `10.0.0.0/24` -- the backbone transit network
- `192.168.1.0/24` -- a LAN you want to advertise
- `eth1` -- the interface facing hosts (no OSPF hellos sent)

**Note:** Unlike BGP, OSPF begins forming adjacencies immediately once a `network` statement matches an interface. Verify timers match on both ends of every link.

## OSPF Area Types

| Area Type | External LSAs | Summary LSAs | Default Route | Use Case |
|---|---|---|---|---|
| Normal (backbone 0) | Yes | Yes | No | Core transit area |
| Normal (non-zero) | Yes | Yes | No | Standard numbered area |
| Stub | No | Yes | Injected by ABR | Leaf area, no external routes needed |
| Totally Stubby | No | No | Injected by ABR | Leaf area, single exit point |
| NSSA | Type-7 (local) | Yes | Optional | Stub area that needs local redistribution |
| Totally NSSA | Type-7 (local) | No | Injected by ABR | NSSA with single exit point |

## LSA Types

| Type | Name | Originated By | Scope |
|---|---|---|---|
| 1 | Router LSA | Every router | Area |
| 2 | Network LSA | DR on multi-access links | Area |
| 3 | Summary LSA (network) | ABR | Inter-area |
| 4 | Summary LSA (ASBR) | ABR | Inter-area |
| 5 | AS External LSA | ASBR | AS-wide |
| 7 | NSSA External LSA | ASBR in NSSA | NSSA area |

## Route Type Codes

| Code | Meaning | Source |
|---|---|---|
| `O` | Intra-area | Router and Network LSAs within the area |
| `O IA` | Inter-area | Summary LSAs from ABRs |
| `O E1` | External Type 1 | Cost = OSPF internal + external metric |
| `O E2` | External Type 2 | Cost = external metric only (default) |
| `O N1` | NSSA Type 1 | Type-7 LSA, metric like E1 |
| `O N2` | NSSA Type 2 | Type-7 LSA, metric like E2 |

## Essential Show Commands

### Process Status

```
! OSPF process overview with router-id, areas, SPF stats
show ip ospf
show ip ospf json

! OSPF-enabled interfaces with cost, state, timers
show ip ospf interface
show ip ospf interface eth0

! VRF-aware queries
show ip ospf vrf CUSTOMER1
show ip ospf vrf all
```

### Neighbor State

```
! All OSPF neighbors with state and DR/BDR role
show ip ospf neighbor

! Detailed neighbor info including adjacency timers
show ip ospf neighbor detail

! JSON output for automation
show ip ospf neighbor json
```

### LSDB and Routes

```
! Full link-state database
show ip ospf database

! Only self-originated LSAs
show ip ospf database self-originate

! Filter by LSA type
show ip ospf database router
show ip ospf database network
show ip ospf database summary
show ip ospf database external
show ip ospf database nssa-external

! Detailed view of a specific LSA
show ip ospf database router detail

! OSPF routing table
show ip ospf route
show ip ospf route detail

! ABR and ASBR reachability
show ip ospf border-routers
```

### Diagnostics

```
! Running OSPF config
show running-config router ospf

! Graceful restart status
show ip ospf graceful-restart helper detail

! MPLS-TE status
show ip ospf mpls-te interface
show ip ospf mpls-te router
show ip ospf mpls-te database

! External route summarization
show ip ospf summary-address detail
```

## Operational Commands

### Clear OSPF Process

**WARNING:** Clearing the OSPF process resets all adjacencies and triggers full SPF reconvergence. This causes temporary traffic disruption across all OSPF areas.

```
! Reset OSPF process -- drops all adjacencies (traffic impact)
clear ip ospf process

! Reset specific OSPF instance
clear ip ospf 1 process

! Reset neighbor adjacencies
clear ip ospf neighbor
```

### Save Configuration

```
write memory
```

## Default Timer Reference

| Timer | Default | Command to Change |
|---|---|---|
| Hello interval (broadcast) | 10s | `ip ospf hello-interval <1-65535>` |
| Dead interval (broadcast) | 40s | `ip ospf dead-interval <1-65535>` |
| Hello interval (NBMA) | 30s | `ip ospf hello-interval <1-65535>` |
| Dead interval (NBMA) | 120s | `ip ospf dead-interval <1-65535>` |
| Retransmit interval | 5s | `ip ospf retransmit-interval <1-65535>` |
| Transmit delay | 1s | `ip ospf transmit-delay <1-65535>` |
| SPF throttle (delay) | 200ms | `timers throttle spf <delay> <initial> <max>` |
| SPF throttle (initial hold) | 1000ms | (set with SPF throttle above) |
| SPF throttle (max hold) | 10000ms | (set with SPF throttle above) |
| LSA throttle | 5000ms | `timers throttle lsa all <0-5000>` |
| LSA min arrival | 1000ms | `timers lsa min-arrival <0-5000>` |
| Reference bandwidth | 100 Mbps | `auto-cost reference-bandwidth <1-4294967>` |
| Grace period | 120s | `graceful-restart grace-period <1-1800>` |
| Router priority | 1 | `ip ospf priority <0-255>` |

## Network Types

| Type | Hello | DR/BDR | Use Case |
|---|---|---|---|
| Broadcast | 10s | Yes | Ethernet LAN segments |
| Non-broadcast (NBMA) | 30s | Yes | Frame Relay, ATM (legacy) |
| Point-to-point | 10s | No | Serial links, GRE tunnels, /31 subnets |
| Point-to-multipoint | 30s | No | Hub-and-spoke NBMA, partial mesh |
