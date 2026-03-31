# Babel Protocol Skill

## Overview

Babel is a distance-vector routing protocol designed for robustness on both wired and wireless mesh networks. It is implemented by the `babeld` daemon in FRR. Babel combines ideas from DSDV, AODV, and EIGRP -- it uses sequence numbers to avoid routing loops, adapts to link quality on wireless links, and converges quickly after topology changes.

Key facts:

- Daemon: `babeld` (enable in `/etc/frr/daemons`)
- Transport: UDP port 6696
- Administrative distance: 125
- Metric: additive link cost (based on rxcost, ETX estimation on wireless)
- Supports: IPv4 and IPv6 simultaneously (dual-stack native)
- Loop avoidance: feasibility conditions with sequence numbers (no counting-to-infinity)
- RFC: 8966 (The Babel Routing Protocol)

> **When to use Babel:** Babel is the right choice for wireless mesh networks, ad-hoc networks, overlay networks, and hybrid wired/wireless environments. It handles lossy links, asymmetric costs, and frequent topology changes gracefully. Prefer OSPF or IS-IS for stable wired-only enterprise or data center networks.

## Enable the Babel Daemon

Edit `/etc/frr/daemons`:

```
babeld=yes
```

Reload FRR:

```
sudo systemctl reload frr
```

## Quick-Start: Wireless Mesh

```
vtysh -c "configure terminal" -c "
router babel
 network eth0
 network wlan0
 redistribute ipv4 connected
 redistribute ipv6 connected

interface eth0
 babel wired
 babel split-horizon

interface wlan0
 babel wireless
 babel channel 1
"
```

Replace:
- `eth0` -- a wired uplink (to the wider network)
- `wlan0` -- a wireless mesh interface
- `babel channel 1` -- the radio channel number for interference tracking

**Note:** Babel begins exchanging routes as soon as `network IFNAME` is configured and the interface is up. No area or AS configuration is required.

## Interface Configuration

### Wired vs Wireless Mode

Every Babel-enabled interface must be classified as wired or wireless. This controls cost estimation and split-horizon behavior.

```
interface eth0
 babel wired

interface wlan0
 babel wireless
```

| Setting | Default Split-Horizon | Cost Model | Use Case |
|---|---|---|---|
| `babel wired` | Enabled | Fixed rxcost | Ethernet, point-to-point links |
| `babel wireless` | Disabled | ETX-based (packet loss) | Wi-Fi, radio, mesh links |

**Default:** `wireless` if not specified.

### Channel Configuration

Channels tell Babel which interfaces share the same radio spectrum (and thus interfere with each other). This affects route selection -- Babel prefers paths through non-interfering channels.

```
interface wlan0
 babel channel 1

interface wlan1
 babel channel 6

interface eth0
 babel channel noninterfering
```

| Channel Setting | Meaning |
|---|---|
| `babel channel (1-254)` | Specific channel number; interfaces on the same channel interfere |
| `babel channel interfering` | Interferes with all other channels |
| `babel channel noninterfering` | Does not interfere with anything (use for wired links) |

### Receive Cost (rxcost)

The base cost for receiving traffic on an interface. Babel adds this to the metric of routes learned through the interface.

```
interface wlan0
 babel rxcost 512

interface eth0
 babel rxcost 96
```

| Link Type | Typical rxcost | Notes |
|---|---|---|
| Wired Ethernet | 96 (default wired) | Low, stable |
| Fast wireless | 256 | Good signal, low loss |
| Marginal wireless | 512-1024 | Moderate packet loss |
| Unreliable wireless | 2048+ | High loss, last resort |

For wireless interfaces, the effective cost is computed using ETX (Expected Transmission Count) based on hello packet loss rates, multiplied by rxcost.

### Hello and Update Intervals

```
interface wlan0
 babel hello-interval 4000
 babel update-interval 20000
```

| Parameter | Default | Range | Purpose |
|---|---|---|---|
| `babel hello-interval` | 4000 ms | 20-655340 ms | Failure detection; shorter = faster failover, more overhead |
| `babel update-interval` | 20000 ms | 20-655340 ms | Full route table refresh interval |
| `babel resend-delay` | 2000 ms | 20-655340 ms | Delay before resending unacknowledged updates |

**Warning:** Very short hello intervals on wireless interfaces increase overhead significantly. On congested wireless links, keep hello-interval >= 2000 ms.

### RTT-Based Cost Adjustment

Enable round-trip-time measurements to penalize high-latency paths:

```
interface wlan0
 babel enable-timestamps
 babel rtt-min 10
 babel rtt-max 500
 babel max-rtt-penalty 150
 babel rtt-decay 42
```

| Parameter | Default | Purpose |
|---|---|---|
| `babel rtt-min` | -- | RTT below this (ms) incurs no penalty |
| `babel rtt-max` | -- | RTT above this (ms) incurs maximum penalty |
| `babel max-rtt-penalty` | 0 | Maximum cost penalty added for high RTT |
| `babel rtt-decay` | -- | Exponential decay factor (1-256) for RTT smoothing |

### Split Horizon

```
interface eth0
 babel split-horizon

interface wlan0
 no babel split-horizon
```

Defaults: enabled for wired, disabled for wireless. Disabling on wireless is important because mesh topologies have multiple paths, and suppressing announcements can hide valid routes.

### Diversity Routing

Babel can maintain backup routes through different interfaces for faster failover:

```
router babel
 babel diversity

interface wlan0
 babel diversity-factor 128
```

`diversity-factor` (1-256, default 256) controls how aggressively backup routes through different interfaces are maintained. Lower values favor more diverse paths.

### Smoothing

Controls hysteresis for cost changes to avoid route flapping:

```
interface wlan0
 babel smoothing-half-life 4
```

Default: 4 seconds. Set to 0 to disable smoothing. Higher values smooth out transient cost fluctuations on unstable links.

## Redistribution

Babel requires separate redistribution statements for IPv4 and IPv6:

```
router babel
 redistribute ipv4 connected
 redistribute ipv4 static
 redistribute ipv4 ospf
 redistribute ipv6 connected
 redistribute ipv6 static
 redistribute ipv6 kernel
```

Syntax:

```
redistribute <ipv4|ipv6> <babel|bgp|connected|eigrp|isis|kernel|openfabric|ospf|rip|sharp|static|table>
```

**Note:** Unlike other FRR protocols, Babel uses `redistribute ipv4` / `redistribute ipv6` prefixes because it natively handles both address families in a single daemon.

## Filtering

### Distribute Lists

```
router babel
 distribute-list ACLIN in
 distribute-list ACLOUT out
 distribute-list prefix PLIN in
 distribute-list prefix PLOUT out
```

Per-interface filtering:

```
router babel
 distribute-list ACLIN in wlan0
 distribute-list ACLOUT out eth0
```

### Example: Accept Only Specific Prefixes

```
ip prefix-list BABEL-IN seq 10 permit 10.0.0.0/8 le 24
ip prefix-list BABEL-IN seq 20 permit 192.168.0.0/16 le 24
ip prefix-list BABEL-IN seq 100 deny any

router babel
 distribute-list prefix BABEL-IN in
```

## Essential Show Commands

```
! Display all Babel routes with metrics and sources
show babel route

! Show routes for a specific prefix
show babel route 10.0.0.0/24
show babel route 2001:db8::/32

! Show Babel-enabled interfaces with type, cost, channel
show babel interface
show babel interface wlan0

! Show discovered Babel neighbors with cost and address
show babel neighbor

! Show Babel process parameters (timers, diversity, smoothing)
show babel parameters

! Running configuration
show running-config router babel
```

### Debug Commands

```
debug babel common
debug babel filter
debug babel timeout
debug babel interface
debug babel route
debug babel all

show debugging babel
```

## Full Configuration Example

### Wireless Mesh with Wired Uplink

```
router babel
 network eth0
 network wlan0
 network wlan1
 babel diversity
 redistribute ipv4 connected
 redistribute ipv4 static
 redistribute ipv6 connected

interface eth0
 babel wired
 babel split-horizon
 babel rxcost 96
 babel channel noninterfering
 babel hello-interval 4000
 babel update-interval 20000

interface wlan0
 babel wireless
 babel channel 1
 babel rxcost 256
 babel hello-interval 2000
 babel update-interval 10000
 babel enable-timestamps
 babel rtt-min 10
 babel rtt-max 500
 babel max-rtt-penalty 150
 no babel split-horizon

interface wlan1
 babel wireless
 babel channel 6
 babel rxcost 256
 babel hello-interval 2000
 babel update-interval 10000
 no babel split-horizon

ip prefix-list BABEL-OUT seq 10 permit 10.0.0.0/8 le 24
ip prefix-list BABEL-OUT seq 100 deny any
```

### Overlay Network (Wired Only)

```
router babel
 network tun0
 network tun1
 redistribute ipv4 connected
 redistribute ipv6 connected

interface tun0
 babel wired
 babel rxcost 96
 babel channel noninterfering

interface tun1
 babel wired
 babel rxcost 96
 babel channel noninterfering
```

## Babel vs Other Distance-Vector Protocols

| Factor | Babel | RIP | EIGRP |
|---|---|---|---|
| Loop avoidance | Feasibility + sequence numbers | Split horizon + hold-down | DUAL (diffusing computation) |
| Metric | Additive cost (ETX on wireless) | Hop count (max 15) | Composite (BW, delay, etc.) |
| Convergence | Fast (sub-second possible) | Slow (minutes) | Fast |
| Wireless support | Native (ETX, channels, diversity) | None | None |
| Dual-stack | Single daemon, both IPv4/IPv6 | Separate daemons | IPv4 only in FRR |
| Scalability | Medium (mesh networks) | Small (15 hops max) | Medium |
| FRR maturity | Stable | Stable | Alpha |
