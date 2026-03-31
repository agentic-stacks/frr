# Protocol Selection Guide

Choose the right routing protocol for your network topology and requirements.

---

## Protocol Comparison

| Criterion | BGP | OSPF | IS-IS | RIP | Babel | EIGRP |
|---|---|---|---|---|---|---|
| **Type** | Path vector (EGP) | Link-state (IGP) | Link-state (IGP) | Distance vector (IGP) | Distance vector (IGP) | Advanced DV (IGP) |
| **Scale** | Internet-scale | ~10K routers | ~10K routers | < 50 routers | < 200 routers | < 500 routers |
| **Convergence** | Seconds–minutes | Sub-second (w/ BFD) | Sub-second (w/ BFD) | 30s–180s | Seconds | Seconds |
| **Complexity** | High | Medium | Medium-High | Low | Low | Medium |
| **Multi-vendor** | Excellent | Excellent | Excellent | Excellent | Limited | Cisco-centric |
| **IPv6** | Full (MP-BGP) | OSPFv3 (separate) | Native dual-stack | RIPng (separate) | Native dual-stack | Yes |
| **MPLS/SR** | LU, VPN, EVPN | SR-MPLS, SRv6 | SR-MPLS, SRv6 | No | No | No |
| **VRF-aware** | Yes (per-AF) | Yes (per-instance) | Yes (per-instance) | Yes (per-instance) | No | Yes |
| **Authentication** | TCP MD5/AO | MD5, HMAC-SHA | HMAC-MD5, HMAC-SHA | MD5 | HMAC-SHA256 | MD5 |
| **Multicast routing** | No (but MP-BGP for MVPN) | MOSPF (rare) | No | No | No | No |
| **Loop prevention** | AS path | SPF (inherent) | SPF (inherent) | Split horizon, poison reverse | Feasibility condition | DUAL algorithm |
| **FRR maturity** | Production | Production | Production | Production | Production | Experimental |

---

## Recommendations by Use Case

### Data Center Leaf-Spine

**Recommended: eBGP**

| Factor | Details |
|---|---|
| Protocol | eBGP with unique ASN per device or AS-path relax |
| Why | Simple, no areas/levels, scales horizontally, unnumbered peering |
| Profile | Use FRR `datacenter` profile |
| Key features | `bgp bestpath as-path multipath-relax`, BGP unnumbered, EVPN |

```
router bgp 65100
 bgp bestpath as-path multipath-relax
 neighbor fabric peer-group
 neighbor fabric remote-as external
 neighbor swp1 interface peer-group fabric
 neighbor swp2 interface peer-group fabric
!
```

**Alternative: eBGP + EVPN** for multi-tenancy with VXLAN overlay.

---

### Enterprise Campus

**Recommended: OSPF**

| Factor | Details |
|---|---|
| Protocol | OSPFv2 + OSPFv3 (or single OSPF with AF support) |
| Why | Widely understood, fast convergence, good tooling, multi-vendor |
| Design | Area 0 backbone + stub/NSSA areas per building/floor |
| Key features | Stub areas, NSSA, BFD, HMAC-SHA authentication |

```
router ospf
 ospf router-id 10.0.0.1
 area 0.0.0.0 authentication message-digest
 area 0.0.0.1 stub
!
interface eth0
 ip ospf area 0.0.0.0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 SECRET
 ip ospf dead-interval minimal hello-multiplier 4
!
```

**Alternative: IS-IS** if dual-stack simplicity is important (single protocol instance for IPv4+IPv6).

---

### ISP / Transit Provider

**Recommended: IS-IS (IGP) + BGP (EGP)**

| Factor | Details |
|---|---|
| IGP | IS-IS for internal topology (native dual-stack, flat namespace) |
| EGP | iBGP with route reflectors for customer/peer/transit routes |
| Why | IS-IS scales well, dual-stack native, segment routing support |
| Key features | IS-IS SR-MPLS/SRv6, BGP communities, BGP-LU, L3VPN |

```
router isis CORE
 net 49.0001.0100.0000.0001.00
 is-type level-2-only
 segment-routing on
 segment-routing prefix 10.0.0.1/32 index 1
!
router bgp 65000
 neighbor RR-CLIENTS peer-group
 neighbor RR-CLIENTS remote-as 65000
 neighbor RR-CLIENTS update-source Loopback0
 address-family ipv4 unicast
  neighbor RR-CLIENTS route-reflector-client
!
```

**Alternative: OSPF (IGP)** if the operations team is more familiar with OSPF; works fine up to a few hundred routers.

---

### Small Office / Branch

**Recommended: OSPF (single area) or Static Routes**

| Factor | Details |
|---|---|
| Protocol | OSPF single area, or static routes with BFD |
| Why | Simple, minimal configuration, fast convergence with BFD |
| When static | Single uplink, no redundancy needed |
| When OSPF | Dual uplinks, need automatic failover |

```
! Simple dual-uplink with OSPF
router ospf
 ospf router-id 10.0.0.100
 passive-interface default
 no passive-interface eth0
 no passive-interface eth1
!
interface eth0
 ip ospf area 0.0.0.0
!
interface eth1
 ip ospf area 0.0.0.0
!
```

---

### Wireless Mesh Network

**Recommended: Babel**

| Factor | Details |
|---|---|
| Protocol | Babel |
| Why | Designed for wireless; handles asymmetric links, high churn, lossy paths |
| Key features | Link quality sensing, dual-stack native, low overhead |

```
router babel
 network eth0
 network wlan0
 redistribute connected
!
interface wlan0
 babel wireless
 babel channel 1
!
interface eth0
 babel wired
!
```

**Alternative: OSPF** if all links are reliable point-to-point tunnels over wireless.

---

### DMVPN / NHRP Overlay

**Recommended: eBGP or OSPF over DMVPN**

| Factor | Details |
|---|---|
| Protocol | eBGP (hub-spoke) or OSPF (NBMA/point-to-multipoint) |
| Underlay | NHRP for spoke-to-spoke shortcut paths |
| Why BGP | Better policy control, scales to hundreds of spokes |
| Why OSPF | Simpler for small deployments (< 50 spokes) |

```
! NHRP + OSPF example
interface tunnel0
 ip ospf network point-to-multipoint
 ip ospf area 0.0.0.0
 ip nhrp authentication SECRET
 ip nhrp network-id 1
 ip nhrp nhs 10.0.0.1 nbma 198.51.100.1
!
```

---

## Protocol Migration Paths

### RIP to OSPF

1. Deploy OSPF on all routers with higher administrative distance
2. Verify OSPF adjacencies and full route table
3. Redistribute RIP into OSPF temporarily
4. Lower OSPF administrative distance (or raise RIP)
5. Remove RIP configuration
6. Remove redistribution

### OSPF to IS-IS

1. Deploy IS-IS alongside OSPF on all routers
2. Redistribute between protocols during transition
3. Verify IS-IS adjacencies and route convergence
4. Shift traffic by adjusting administrative distance
5. Remove OSPF configuration
6. Remove redistribution

### IGP to BGP (DC migration)

1. Deploy eBGP on spine-leaf links alongside existing IGP
2. Redistribute IGP into BGP initially
3. Move prefix advertisements to BGP `network` statements
4. Verify full reachability via BGP
5. Remove IGP configuration and redistribution

### Single-Protocol to SR-MPLS

1. Verify kernel MPLS support (modules loaded, labels configured)
2. Enable segment routing in IS-IS or OSPF
3. Assign prefix SIDs to loopbacks
4. Verify label forwarding: `show mpls table`
5. Deploy SR-TE policies or LDP interop as needed

---

## Decision Flowchart

```
Start
  |
  v
Internet/multi-AS? ──Yes──> BGP (+ IGP for internal)
  |
  No
  |
  v
Wireless mesh? ──Yes──> Babel
  |
  No
  |
  v
> 500 routers? ──Yes──> IS-IS (or OSPF with careful area design)
  |
  No
  |
  v
Multi-vendor? ──Yes──> OSPF (most universal IGP)
  |
  No
  |
  v
Cisco-only? ──Yes──> EIGRP (if supported) or OSPF
  |
  No
  |
  v
< 10 routers, single uplink? ──Yes──> Static routes
  |
  No
  |
  v
Default ──> OSPF
```
