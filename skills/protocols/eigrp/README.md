# EIGRP Protocol Skill

## Overview

EIGRP (Enhanced Interior Gateway Routing Protocol) is an advanced distance-vector protocol originally developed by Cisco. FRR includes an EIGRP implementation via the `eigrpd` daemon, but it is in **alpha status** with significant limitations compared to Cisco IOS EIGRP.

Key facts:

- Daemon: `eigrpd` (enable in `/etc/frr/daemons`)
- Transport: IP protocol 88
- Administrative distance: 90 (internal), 170 (external)
- Metric: composite of bandwidth, delay, reliability, load, MTU
- Algorithm: DUAL (Diffusing Update Algorithm) -- loop-free convergence
- AS number: required, must match on all neighbors (1-65535)
- IPv4 only in FRR (no IPv6/EIGRP for IPv6 support)

> **Warning -- Alpha Status:** FRR's EIGRP implementation is incomplete and intended primarily for interoperability testing with Cisco routers. It lacks many features present in Cisco EIGRP (see Caveats section below). Do not use FRR EIGRP in production unless you have thoroughly tested your specific use case.

> **When to use EIGRP in FRR:** The primary use case is interoperating with existing Cisco EIGRP networks where replacing Cisco hardware with FRR-based routers. For greenfield deployments, prefer OSPF or IS-IS. For mesh/wireless, use Babel.

## Enable the EIGRP Daemon

Edit `/etc/frr/daemons`:

```
eigrpd=yes
```

Reload FRR:

```
sudo systemctl reload frr
```

## Quick-Start: Basic EIGRP

```
vtysh -c "configure terminal" -c "
router eigrp 100
 network 10.0.0.0/24
 network 192.168.1.0/24
 passive-interface eth1
"
```

Replace:
- `100` -- the EIGRP Autonomous System (AS) number (must match all EIGRP neighbors)
- `10.0.0.0/24` -- a transit network between routers
- `192.168.1.0/24` -- a LAN subnet to advertise
- `eth1` -- a host-facing interface (suppress EIGRP hellos)

**Note:** All EIGRP routers in the same domain must share the same AS number. Mismatched AS numbers prevent neighbor formation entirely -- no error is logged, hellos are silently ignored.

## AS Number

The Autonomous System number identifies the EIGRP routing domain. It is a local identifier (not a BGP ASN) and must be consistent across all routers in the EIGRP domain.

```
router eigrp 100
```

Valid range: 1-65535.

Multiple EIGRP instances with different AS numbers can run simultaneously:

```
router eigrp 100
 network 10.0.0.0/24

router eigrp 200
 network 172.16.0.0/16
```

## Network Statements

Network statements enable EIGRP on interfaces whose addresses fall within the specified range and advertise those networks to neighbors.

```
router eigrp 100
 network 10.0.0.0/24
 network 192.168.1.0/24
 network 172.16.0.0/16
```

Unlike Cisco IOS, FRR uses standard prefix notation (CIDR) rather than wildcard masks:

| FRR Syntax | Cisco IOS Equivalent |
|---|---|
| `network 10.0.0.0/24` | `network 10.0.0.0 0.0.0.255` |
| `network 10.0.0.0/8` | `network 10.0.0.0 0.255.255.255` |
| `network 192.168.1.0/24` | `network 192.168.1.0 0.0.0.255` |

## Passive Interfaces

Suppress EIGRP hello packets on an interface while still advertising its network:

```
router eigrp 100
 ! Suppress specific interface
 passive-interface eth1

 ! Make all interfaces passive, then selectively activate
 passive-interface default
 no passive-interface eth0
```

## EIGRP Metrics

EIGRP uses a composite metric computed from five components. The classic formula (K1=1, K2=0, K3=1, K4=0, K5=0) simplifies to:

```
metric = (bandwidth + delay) * 256
```

Where:
- **bandwidth** = 10^7 / (minimum bandwidth in kbps along the path)
- **delay** = sum of delays along the path (in tens of microseconds)

Full metric components:

| Component | Range | Description |
|---|---|---|
| Bandwidth | 1-4294967295 | Minimum bandwidth along path (kbps) |
| Delay | 0-4294967295 | Cumulative delay along path (10s of microseconds) |
| Reliability | 0-255 | Link reliability (255 = 100% reliable) |
| Load | 1-255 | Link utilization (255 = fully loaded) |
| MTU | 1-65535 | Minimum MTU along path |

### Set Metrics on Redistributed Routes

```
router eigrp 100
 redistribute static metric 100000 10 255 1 1500
 redistribute connected metric 100000 10 255 1 1500
```

The five metric values after `metric` are: bandwidth, delay, reliability, load, MTU.

## Redistribution

```
router eigrp 100
 redistribute static [metric BW DELAY RELIABILITY LOAD MTU]
 redistribute connected [metric BW DELAY RELIABILITY LOAD MTU]
 redistribute ospf [metric BW DELAY RELIABILITY LOAD MTU]
 redistribute rip [metric BW DELAY RELIABILITY LOAD MTU]
 redistribute bgp [metric BW DELAY RELIABILITY LOAD MTU]
```

Full syntax:

```
redistribute <babel|bgp|connected|isis|kernel|openfabric|ospf|rip|sharp|static|table>
  [metric (1-4294967295) (0-4294967295) (0-255) (1-255) (1-65535)]
```

**Warning:** Always specify metric values when redistributing into EIGRP. Without explicit metrics, redistributed routes may receive a metric of infinity and be unreachable.

## VRF Support

```
router eigrp 100 vrf CUSTOMER1
 network 10.100.0.0/24
```

## Essential Show Commands

```
! EIGRP topology table (successor and feasible successor routes)
show ip eigrp topology
show ip eigrp vrf CUSTOMER1 topology

! EIGRP-enabled interfaces
show ip eigrp interface
show ip eigrp vrf CUSTOMER1 interface

! EIGRP neighbor table
show ip eigrp neighbor
show ip eigrp vrf CUSTOMER1 neighbor

! Running configuration
show running-config router eigrp
```

### Debug Commands

```
debug eigrp packets
debug eigrp transmit
show debugging eigrp
```

## Full Configuration Example

### EIGRP with Redistribution

```
router eigrp 100
 network 10.0.0.0/24
 network 10.0.1.0/24
 network 192.168.1.0/24
 passive-interface eth2
 passive-interface lo
 redistribute static metric 100000 10 255 1 1500
 redistribute connected metric 100000 10 255 1 1500
```

### Multi-AS Interop with Cisco

```
! AS 100 -- connects to Cisco EIGRP domain
router eigrp 100
 network 10.0.0.0/24

! Cisco router config (for reference):
! router eigrp 100
!  network 10.0.0.0 0.0.0.255
!  no auto-summary
```

## Caveats: FRR EIGRP vs Cisco EIGRP

**Warning:** FRR's EIGRP implementation is missing many features that Cisco IOS EIGRP provides. Review this table carefully before deployment.

| Feature | Cisco EIGRP | FRR EIGRP |
|---|---|---|
| Status | Production | **Alpha** |
| IPv6 (EIGRP for IPv6) | Yes | **No** |
| Named mode | Yes | **No** |
| Stub routing | Yes | **No** |
| Route summarization | Yes | **No** |
| Authentication (MD5/SHA) | Yes | **No** |
| K-value tuning | Yes | **No** |
| Wide metrics | Yes | **No** |
| Graceful restart / NSF | Yes | **No** |
| Route-maps on redistribute | Yes | **No** |
| Offset lists | Yes | **No** |
| Distribute lists | Yes | **Limited** |
| BFD integration | Yes | **No** |
| DUAL algorithm | Full | Basic |
| Interop with Cisco | -- | Basic (tested for simple topologies) |

### Known Limitations

1. **No authentication** -- EIGRP neighbors cannot be authenticated. Do not run on untrusted networks.
2. **No route summarization** -- All prefixes are advertised individually. This can cause excessive routing table size.
3. **No stub routing** -- Cannot configure stub sites to reduce query scope.
4. **No named mode** -- Only classic AS-number mode is supported.
5. **Limited DUAL implementation** -- Complex stuck-in-active (SIA) scenarios may not be handled correctly.
6. **No IPv6** -- Only IPv4 is supported. For IPv6, use another protocol.
7. **Sparse documentation** -- Community support is limited compared to OSPF/BGP.

### Recommendation

For new FRR deployments, prefer OSPF (for structured networks) or Babel (for mesh/wireless). Use EIGRP only when you must interoperate with an existing Cisco EIGRP domain and cannot migrate to OSPF.

## EIGRP vs Other IGPs

| Factor | EIGRP | OSPF | RIP | Babel |
|---|---|---|---|---|
| Type | Advanced distance-vector | Link-state | Distance-vector | Distance-vector |
| Convergence | Fast (DUAL) | Fast (SPF) | Slow | Fast |
| Metric | Composite (BW+delay) | Cost (bandwidth) | Hop count | Additive cost |
| Loop avoidance | DUAL feasibility | SPF (loop-free by design) | Split horizon + timers | Feasibility + seq numbers |
| FRR status | **Alpha** | Stable | Stable | Stable |
| IPv6 | No (in FRR) | OSPFv3 | RIPng | Native dual-stack |
| Scalability | Medium | Large | Small | Medium |
| Complexity | Moderate | Moderate | Very low | Low |
