# OpenFabric Protocol Skill

## Overview

OpenFabric is a simplified link-state routing protocol for data center fabrics in FRR, implemented by the `fabricd` daemon. Derived from IS-IS, OpenFabric strips away features unnecessary in structured DC topologies (areas, multiple levels, CLNS) while retaining efficient SPF-based forwarding and link-state flooding.

Key facts:

- Daemon: `fabricd` (enable in `/etc/frr/daemons`)
- Config file: `fabricd.conf`
- Based on: IS-IS (simplified for spine-leaf topologies)
- Addressing: ISO NET (Network Entity Title) required
- Fabric tiers: 0-14 (automatic or manual assignment)
- Supports: IPv4, IPv6, authentication, overload bit, purge originator
- Single flooding domain (no areas or levels)

> **When to choose OpenFabric:** Use OpenFabric in structured DC fabrics (spine-leaf, fat-tree) where you want link-state routing without the complexity of full IS-IS configuration (levels, areas, DIS election) and without the operational overhead of BGP. OpenFabric auto-discovers topology tiers and optimizes flooding. Prefer IS-IS if you need multi-area, multi-level, or segment routing. Prefer BGP if you need policy-rich path selection or internet peering integration.

## OpenFabric vs IS-IS vs BGP for DC Fabrics

| Feature | OpenFabric | IS-IS | BGP (DC overlay) |
|---|---|---|---|
| Protocol type | Link-state | Link-state | Path-vector |
| Configuration complexity | Low | Medium | High |
| Topology awareness | Automatic tiers | Manual levels/areas | None (policy-driven) |
| Flooding efficiency | Optimized for fabric | General purpose | N/A (no flooding) |
| Convergence speed | Fast (SPF) | Fast (SPF) | Slower (best-path) |
| Segment routing | No | Yes | Yes (SR-MPLS, SRv6) |
| EVPN integration | No | Underlay only | Full |
| Multi-vendor support | Limited | Broad | Broad |
| Scaling | DC-scale | Enterprise/SP-scale | Internet-scale |

## Enable the OpenFabric Daemon

Edit `/etc/frr/daemons`:

```
fabricd=yes
```

Reload FRR:

```
sudo systemctl reload frr
```

### fabricd Startup Options

Set in `/etc/frr/daemons` via `fabricd_options`:

| Flag | Purpose |
|---|---|
| `--dummy_as_loopback` | Treat dummy interfaces as loopback interfaces |

## Quick-Start: Spine-Leaf OpenFabric

**Leaf switch (tier 0):**

```
vtysh -c "configure terminal" -c "
interface lo
 ip router openfabric FABRIC1
!
interface eth-spine1
 ip router openfabric FABRIC1
!
interface eth-spine2
 ip router openfabric FABRIC1
!
router openfabric FABRIC1
 net 49.0001.0000.0000.0001.00
 fabric-tier 0
"
```

**Spine switch (tier 1):**

```
vtysh -c "configure terminal" -c "
interface lo
 ip router openfabric FABRIC1
!
interface eth-leaf1
 ip router openfabric FABRIC1
!
interface eth-leaf2
 ip router openfabric FABRIC1
!
interface eth-leaf3
 ip router openfabric FABRIC1
!
router openfabric FABRIC1
 net 49.0001.0000.0000.1001.00
 fabric-tier 1
"
```

The domain name (`FABRIC1`) must match across all routers in the fabric. The NET (Network Entity Title) must be unique per router.

## NET (Network Entity Title)

The NET identifies each router in ISO format:

```
49.0001.0000.0000.0001.00
│  │     │              │
│  │     │              └── N-selector (always 00 for NET)
│  │     └──────────────── System ID (6 bytes, must be unique)
│  └────────────────────── Area ID
└───────────────────────── AFI (49 = private)
```

**Recommendation:** Encode the loopback IP into the System ID for easy correlation:

- Loopback `10.0.0.1` becomes System ID `0100.0000.0001`
- Loopback `10.0.1.25` becomes System ID `0100.0001.0025`

```
router openfabric FABRIC1
 net 49.0001.0100.0000.0001.00
```

## Fabric Tiers

OpenFabric assigns tiers to classify a router's position in the fabric hierarchy. Tiers optimize flooding -- a router only floods LSPs to peers in adjacent tiers.

| Tier | Typical role |
|---|---|
| 0 | Leaf / ToR switch |
| 1 | Spine switch |
| 2 | Super-spine |
| 3-14 | Higher tiers (rare) |

### Manual Tier Assignment

```
router openfabric FABRIC1
 fabric-tier 0
```

### Automatic Tier Discovery

If `fabric-tier` is not configured, OpenFabric attempts to discover the tier automatically based on the topology. This works well in regular spine-leaf topologies but may produce unexpected results in irregular or asymmetric fabrics.

> **Warning:** In production, always configure `fabric-tier` explicitly. Automatic discovery can misclassify routers during partial bring-up or topology changes, causing suboptimal flooding.

## Router Configuration

### Authentication

```
router openfabric FABRIC1
 domain-password md5 s3cretFabr1c
```

All routers in the domain must share the same password. Supports `clear` (plaintext) or `md5`.

### Overload Bit

Set the overload bit to drain traffic from a router before maintenance:

```
router openfabric FABRIC1
 set-overload-bit
```

Other routers will route around this node. Remove to restore transit:

```
router openfabric FABRIC1
 no set-overload-bit
```

> **Tip:** Set overload bit before performing maintenance. Wait for traffic to drain (monitor with interface counters), then proceed with upgrades or changes. Remove overload bit when the router is ready to carry traffic again.

### Adjacency Logging

```
router openfabric FABRIC1
 log-adjacency-changes
```

Logs neighbor up/down events to syslog -- essential for fabric monitoring.

### Purge Originator Identification

```
router openfabric FABRIC1
 purge-originator
```

Enables RFC 6232 purge originator tracking for debugging spurious LSP purges.

### Attached Bit

Control inter-area attached bit behavior:

```
router openfabric FABRIC1
 attached-bit receive ignore
 attached-bit send
```

## Timer Tuning

| Command | Default | Range | Purpose |
|---|---|---|---|
| `lsp-gen-interval (1-120)` | 1s | 1-120s | Min time between LSP regeneration |
| `lsp-refresh-interval (1-65235)` | 900s | 1-65235s | LSP refresh period |
| `max-lsp-lifetime (360-65535)` | 1200s | 360-65535s | Maximum LSP lifetime |
| `spf-interval (1-120)` | 1s | 1-120s | Min time between SPF calculations |

Example for aggressive convergence:

```
router openfabric FABRIC1
 lsp-gen-interval 1
 spf-interval 1
```

> **Warning:** `max-lsp-lifetime` must be greater than `lsp-refresh-interval`. If it is not, LSPs will expire before they are refreshed, causing route flaps.

## Interface Configuration

### Activate OpenFabric on an Interface

```
interface eth0
 ip router openfabric FABRIC1
```

The domain name must match the `router openfabric` instance.

### Interface Commands

| Command | Purpose |
|---|---|
| `ip router openfabric WORD` | Activate OpenFabric on interface |
| `openfabric metric (0-16777215)` | Set interface metric (default: auto-cost) |
| `openfabric hello-interval (1-600)` | Hello sending interval (seconds) |
| `openfabric hello-multiplier (2-100)` | Holding time = hello-interval x multiplier |
| `openfabric passive` | Advertise prefix but do not form adjacencies |
| `openfabric csnp-interval (1-600)` | CSNP interval (seconds) |
| `openfabric psnp-interval (1-120)` | PSNP interval (seconds) |
| `openfabric password [clear \| md5] WORD` | Per-interface authentication |

### Passive Interfaces

Use passive on loopbacks and host-facing ports where no OpenFabric neighbor exists:

```
interface lo
 ip router openfabric FABRIC1
 openfabric passive
```

### Manual Metric

Override auto-cost with a fixed metric:

```
interface eth-spine1
 openfabric metric 100
```

Lower metric is preferred. Auto-cost derives the metric from interface bandwidth.

## Full Configuration Example

### Three-Tier Fabric (Leaf / Spine / Super-Spine)

**Leaf-1:**

```
interface lo
 ip address 10.0.0.1/32
 ip router openfabric DC1
 openfabric passive
!
interface eth-spine1
 ip address 10.1.1.1/31
 ip router openfabric DC1
!
interface eth-spine2
 ip address 10.1.2.1/31
 ip router openfabric DC1
!
router openfabric DC1
 net 49.0001.0100.0000.0001.00
 fabric-tier 0
 log-adjacency-changes
 domain-password md5 Fabr1cS3cret
```

**Spine-1:**

```
interface lo
 ip address 10.0.1.1/32
 ip router openfabric DC1
 openfabric passive
!
interface eth-leaf1
 ip address 10.1.1.0/31
 ip router openfabric DC1
!
interface eth-leaf2
 ip address 10.1.3.0/31
 ip router openfabric DC1
!
interface eth-superspine1
 ip address 10.2.1.1/31
 ip router openfabric DC1
!
router openfabric DC1
 net 49.0001.0100.0001.0001.00
 fabric-tier 1
 log-adjacency-changes
 domain-password md5 Fabr1cS3cret
```

**Super-Spine-1:**

```
interface lo
 ip address 10.0.2.1/32
 ip router openfabric DC1
 openfabric passive
!
interface eth-spine1
 ip address 10.2.1.0/31
 ip router openfabric DC1
!
interface eth-spine2
 ip address 10.2.2.0/31
 ip router openfabric DC1
!
router openfabric DC1
 net 49.0001.0100.0002.0001.00
 fabric-tier 2
 log-adjacency-changes
 domain-password md5 Fabr1cS3cret
```

## Show Commands

| Command | Output |
|---|---|
| `show openfabric summary` | Router ID, uptime, adjacency count |
| `show openfabric hostname` | Hostname-to-System-ID mappings |
| `show openfabric interface [detail] [IFNAME]` | Interface state, metric, hello timers |
| `show openfabric neighbor [detail] [SYSID]` | Neighbor adjacencies and state |
| `show openfabric database [detail] [LSPID]` | Link-state database contents |
| `show openfabric topology` | Calculated paths and costs |

### Example Show Output

```
router# show openfabric summary
Area DC1:
  fabricd process is running
  System Id: 0100.0000.0001
  Up time: 02:15:33
  Number of interfaces: 3
  Number of adjacencies: 2

router# show openfabric neighbor
Area DC1:
  System Id           Interface     State  Holdtime  SNPA
  0100.0001.0001      eth-spine1    Up     26        aabb.cc00.0100
  0100.0001.0002      eth-spine2    Up     28        aabb.cc00.0200

router# show openfabric topology
Area DC1:
  System Id           Metric  Next-Hop        Interface
  0100.0000.0001      --
  0100.0001.0001      10      0100.0001.0001  eth-spine1
  0100.0001.0002      10      0100.0001.0002  eth-spine2
  0100.0002.0001      20      0100.0001.0001  eth-spine1
```

## Debug Commands

| Command | Purpose |
|---|---|
| `debug openfabric adj-packets` | Adjacency packet processing |
| `debug openfabric events` | Protocol events |
| `debug openfabric lsp-gen` | LSP generation |
| `debug openfabric lsp-sched` | LSP scheduling |
| `debug openfabric packet-dump` | Full packet content dump |
| `debug openfabric route-events` | Route installation events |
| `debug openfabric snp-packets` | SNP (CSNP/PSNP) packet processing |
| `debug openfabric spf-events` | SPF calculation events |
| `debug openfabric update-packets` | Update packet processing |
| `debug openfabric flooding` | Flooding behavior |
| `debug openfabric bfd` | BFD integration events |
| `debug openfabric ldp-sync` | LDP synchronization events |
| `debug openfabric lfa` | Loop-Free Alternate calculation |
| `debug openfabric sr-events` | Segment routing events |
| `debug openfabric te-events` | Traffic engineering events |
| `debug openfabric tx-queue` | Transmit queue operations |
| `show debugging openfabric` | Active debug flags |

> **Warning:** `debug openfabric packet-dump` produces extremely verbose output. Use only for short-duration troubleshooting and disable promptly.

## Troubleshooting

### Adjacency Not Forming

| Check | Command / Action |
|---|---|
| Domain name matches? | `show openfabric summary` on both sides |
| Interface activated? | `show openfabric interface` -- must show `Active` |
| NET configured? | `show openfabric summary` -- System Id must be present |
| Authentication matches? | `domain-password` must be identical on all routers |
| L2 connectivity? | Verify with `ping` or `ip neigh` on the link |
| Hello interval compatible? | Must be close enough for holdtime to not expire |

### Routes Missing

| Cause | Fix |
|---|---|
| Overload bit set | `show openfabric database detail` -- check OL flag |
| LSP expired | Verify `max-lsp-lifetime` > `lsp-refresh-interval` |
| SPF not converged | Check `debug openfabric spf-events` |
| Interface passive | Passive interfaces advertise prefixes but do not form adjacencies |
| Tier misconfiguration | Wrong tier can cause flooding issues -- verify with `show openfabric database detail` |

### Fabric Tier Issues

| Symptom | Diagnosis |
|---|---|
| Auto-tier produces wrong value | Set `fabric-tier` explicitly |
| Flooding incomplete | Verify all adjacent routers have consecutive tier numbers |
| Some routers not receiving LSPs | Check tier assignment chain from source to destination |

## Quick Reference: Verification Checklist

```bash
# 1. Daemon running?
vtysh -c "show openfabric summary"

# 2. All adjacencies up?
vtysh -c "show openfabric neighbor"

# 3. Database synchronized?
vtysh -c "show openfabric database"

# 4. Topology and paths correct?
vtysh -c "show openfabric topology"

# 5. Routes installed in RIB?
vtysh -c "show ip route openfabric"

# 6. Interface metrics reasonable?
vtysh -c "show openfabric interface detail"
```
