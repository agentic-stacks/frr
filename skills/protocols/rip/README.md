# RIP Protocol Skill

## Overview

RIP (Routing Information Protocol) is a distance-vector interior gateway protocol in FRR, implemented by the `ripd` (RIPv1/v2) and `ripngd` (RIPng/IPv6) daemons. RIP uses hop count as its metric (max 15, 16 = unreachable) and converges via periodic broadcasts and triggered updates.

Key facts:

- Daemons: `ripd` for IPv4, `ripngd` for IPv6 (enable in `/etc/frr/daemons`)
- Transport: UDP port 520 (RIPv1/v2), UDP port 521 (RIPng)
- Administrative distance: 120
- Metric: hop count (1-15 valid, 16 = infinity)
- Default timers: update 30s, timeout 180s, garbage-collection 120s
- RIPv1: classful, broadcast only, no authentication
- RIPv2: classless (VLSM/CIDR), multicast 224.0.0.9, MD5/plain-text auth
- RIPng: IPv6, multicast ff02::9, no built-in authentication

> **When to use RIP:** RIP is appropriate for small, flat networks (under ~15 hops) where simplicity matters more than convergence speed. Common use cases include legacy environments, stub networks with a single exit, lab/education setups, and networks where OSPF/IS-IS complexity is not justified. Avoid RIP for networks requiring fast convergence or more than 15 hops.

## Enable RIP Daemons

Edit `/etc/frr/daemons`:

```
ripd=yes
ripngd=yes
```

Reload FRR:

```
sudo systemctl reload frr
```

## Quick-Start: Single-Subnet RIPv2

```
vtysh -c "configure terminal" -c "
router rip
 version 2
 network 10.0.0.0/24
 network 192.168.1.0/24
 passive-interface eth1
 timers basic 30 180 120
"
```

Replace:
- `10.0.0.0/24` -- the transit network between routers
- `192.168.1.0/24` -- a LAN subnet to advertise
- `eth1` -- the interface facing hosts (suppress RIP hellos)

**Note:** RIPv2 multicasts to 224.0.0.9 by default. All routers sharing a subnet must run the same RIP version to form adjacencies.

## Quick-Start: RIPng (IPv6)

```
vtysh -c "configure terminal" -c "
router ripng
 network eth0
 network eth2
 redistribute connected
"
```

RIPng uses interface names or IPv6 prefixes in `network` statements.

## Network Statements

Network statements enable RIP on matching interfaces and define which prefixes to advertise.

```
! Match by subnet -- enables RIP on all interfaces in this range
router rip
 network 10.0.0.0/8
 network 172.16.0.0/16

! Match by interface name -- enables RIP on that interface specifically
router rip
 network eth0
 network lo
```

For RIPng:

```
router ripng
 network eth0
 network 2001:db8::/32
```

## Version Control

### Global Version Setting

```
router rip
 version 2
```

| Setting | Sends | Accepts |
|---|---|---|
| `version 1` | v1 only | v1 only |
| `version 2` | v2 only | v2 only |
| (default, no statement) | v2 | v1 and v2 |

### Per-Interface Version Override

```
interface eth0
 ip rip send version 2
 ip rip receive version 1 2
```

| Command | Effect |
|---|---|
| `ip rip send version 1` | Send RIPv1 only on this interface |
| `ip rip send version 2` | Send RIPv2 only on this interface |
| `ip rip send version 1 2` | Send both v1 and v2 |
| `ip rip receive version 1` | Accept RIPv1 only |
| `ip rip receive version 2` | Accept RIPv2 only |
| `ip rip receive version 1 2` | Accept both versions |
| `ip rip v2-broadcast` | Send RIPv2 via broadcast instead of multicast |

## Passive Interfaces

Passive interfaces receive RIP updates but do not send them. Use on LAN-facing interfaces and loopbacks.

```
router rip
 ! Suppress RIP on a specific interface
 passive-interface eth1

 ! Make all interfaces passive, then selectively activate
 passive-interface default
 no passive-interface eth0
```

For RIPng:

```
router ripng
 passive-interface eth1
```

## Timers

```
router rip
 timers basic UPDATE TIMEOUT GARBAGE
```

| Timer | Default | Purpose |
|---|---|---|
| UPDATE | 30s | Interval between full routing table broadcasts |
| TIMEOUT | 180s | Time before a route is declared unreachable (no updates received) |
| GARBAGE | 120s | Time an unreachable route stays in the table before deletion |

**Warning:** Timer values must match on all routers in the RIP domain. Mismatched timers cause route flapping and blackholes.

For RIPng:

```
router ripng
 timers basic 30 180 120
```

## Authentication (RIPv2 Only)

RIPv1 and RIPng have no authentication support. RIPv2 supports plain-text and MD5 authentication per interface.

### Plain-Text Authentication

```
interface eth0
 ip rip authentication mode text
 ip rip authentication string MySecret123
```

**Warning:** Plain-text passwords are visible in packet captures. Use MD5 in production.

### MD5 Authentication with Key Chain

```
! Define the key chain
key chain RIP-KEYS
 key 1
  key-string S3cureP@ss
  accept-lifetime 00:00:00 Jan 1 2025 infinite
  send-lifetime 00:00:00 Jan 1 2025 infinite

! Apply to interface
interface eth0
 ip rip authentication mode md5
 ip rip authentication key-chain RIP-KEYS
```

Key chain rotation: define multiple keys with overlapping lifetimes for hitless key rollover.

## Redistribution

### Redistribute into RIP

```
router rip
 redistribute static
 redistribute connected
 redistribute ospf
 redistribute bgp route-map BGP-TO-RIP

 ! Set a default metric for all redistributed routes
 default-metric 5
```

Full redistribution syntax:

```
redistribute (babel|bgp|connected|eigrp|isis|kernel|openfabric|ospf|sharp|static|table)
  [metric (0-16)] [route-map WORD]
```

### Originate a Default Route

```
router rip
 default-information originate
```

### RIP-Only Static Routes

Inject a route into RIP without adding it to the kernel table:

```
router rip
 route 10.99.0.0/24
```

### Redistribute into RIPng

```
router ripng
 redistribute connected
 redistribute static
 redistribute ospf6
```

## Offset Lists

Offset lists add a metric value to routes matching an access list, applied inbound or outbound, optionally per interface.

```
! Add 3 hops to routes matching ACL "REMOTE-NETS" received on eth0
router rip
 offset-list REMOTE-NETS in 3 eth0

! Add 2 hops to all advertised routes
router rip
 offset-list 0 out 2
```

Access list `0` matches all routes.

## Distance

Override the default administrative distance (120):

```
router rip
 ! Global distance
 distance 100

 ! Per-source distance
 distance 90 10.0.0.1/32

 ! Per-source with access-list filter
 distance 150 10.0.0.2/32 UNTRUSTED-PREFIXES
```

## Split Horizon

Prevents advertising routes back through the interface they were learned on:

```
interface eth0
 ip rip split-horizon
 ! or with poisoned reverse (advertise back with metric 16)
 ip rip split-horizon poisoned-reverse
```

For RIPng:

```
interface eth0
 ipv6 ripng split-horizon [poisoned-reverse]
```

## Distribute Lists (Route Filtering)

Filter routes on ingress or egress per interface:

```
! Filter with access list
router rip
 distribute-list ACL-NAME in eth0
 distribute-list ACL-NAME out eth0

! Filter with prefix list
router rip
 distribute-list prefix PL-NAME in eth0
 distribute-list prefix PL-NAME out eth0
```

For RIPng:

```
router ripng
 ipv6 distribute-list ACLNAME in eth0
 ipv6 distribute-list prefix PLNAME out eth0
```

## ECMP

```
router rip
 allow-ecmp [1-MULTIPATH_NUM]
```

Controls the number of equal-cost paths RIP installs for the same prefix. Default is 1 (no ECMP).

## BFD Integration

Enable BFD for fast failure detection on RIP neighbors:

```
interface eth0
 ip rip bfd
 ip rip bfd profile FAST-DETECT

router rip
 bfd default-profile FAST-DETECT
```

## Static RIP Neighbors

For non-multicast networks (e.g., NBMA), define unicast neighbors:

```
router rip
 neighbor 10.0.0.2
 neighbor 10.0.0.3
```

## RIPng Route Aggregation

```
router ripng
 aggregate-address 2001:db8::/32
```

## Essential Show Commands

### RIP (IPv4)

```
! Display RIP routing table
show ip rip
show ip rip vrf CUSTOMER1

! RIP process status: timers, version, networks, distance
show ip rip status
show ip rip vrf CUSTOMER1 status

! Running config
show running-config router rip
```

### RIPng (IPv6)

```
! RIPng routing table
show ipv6 ripng
show ipv6 ripng vrf NAME

! RIPng status
show ipv6 ripng status

! Running config
show running-config router ripng
```

### Debug Commands

```
! RIP debug
debug rip events
debug rip packet
debug rip zebra
show debugging rip

! RIPng debug
debug ripng events
debug ripng packet
debug ripng packet recv
debug ripng packet send
debug ripng zebra
show debugging ripng
```

### Clear Routes

```
clear ip rip
clear ip rip vrf CUSTOMER1
clear ipv6 ripng
clear ipv6 ripng vrf NAME
```

## Full Configuration Example

### RIPv2 with Authentication and Redistribution

```
key chain RIP-AUTH
 key 1
  key-string R1pK3y2025

interface eth0
 ip rip authentication mode md5
 ip rip authentication key-chain RIP-AUTH
 ip rip split-horizon poisoned-reverse

interface eth1
 ip rip authentication mode md5
 ip rip authentication key-chain RIP-AUTH

interface eth2
 ! LAN -- no config needed, passive below

router rip
 version 2
 network 10.0.0.0/24
 network 10.0.1.0/24
 network 192.168.1.0/24
 passive-interface eth2
 passive-interface lo
 timers basic 30 180 120
 default-metric 3
 redistribute static route-map STATIC-TO-RIP
 redistribute connected
 distance 120

route-map STATIC-TO-RIP permit 10
 match ip address prefix-list PL-STATIC
 set metric 5

ip prefix-list PL-STATIC seq 10 permit 10.99.0.0/16 le 24
```

### RIPng Minimal Config

```
router ripng
 network eth0
 network eth1
 passive-interface eth2
 redistribute connected
 aggregate-address 2001:db8:1::/48
```

## RIP vs Other IGPs

| Factor | RIP | OSPF | IS-IS |
|---|---|---|---|
| Algorithm | Distance-vector (Bellman-Ford) | Link-state (Dijkstra) | Link-state (Dijkstra) |
| Metric | Hop count (max 15) | Cost (bandwidth-based) | Wide metric |
| Convergence | Slow (minutes) | Fast (sub-second with BFD) | Fast (sub-second with BFD) |
| Scalability | Small networks only | Hundreds of routers | Thousands of routers |
| Complexity | Very low | Moderate | Moderate-high |
| VLSM/CIDR | v2 only | Yes | Yes |
| Auth | v2: MD5/text | MD5/SHA | IS-IS auth |
