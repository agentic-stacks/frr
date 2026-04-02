# IS-IS Protocol Skill

## Overview

IS-IS (Intermediate System to Intermediate System) is a link-state IGP in FRR, implemented by the `isisd` daemon. IS-IS runs directly on the data link layer (not IP), supports both IPv4 and IPv6 natively via multi-topology extensions, and uses a two-level hierarchy for scalability.

Key facts:

- Daemon: `isisd` (enable in `/etc/frr/daemons` with `isisd=yes`)
- Protocol: ISO/OSI, runs on Layer 2 (not IP)
- Administrative distance: 115
- Dual-stack: Carries IPv4 and IPv6 in a single protocol instance
- Levels: Level-1 (intra-area), Level-2 (inter-area/backbone), Level-1-2 (both)
- Default metric: 10 per interface (narrow), wide metrics support up to 16,777,215
- Hello interval: 10s (broadcast), 10s (point-to-point)
- Hold time: hello-interval x hello-multiplier (default multiplier: 3, so 30s)

## Enable IS-IS

Edit `/etc/frr/daemons`:

```
isisd=yes
```

Reload FRR:

```
sudo systemctl reload frr
```

## Quick-Start: Configure Single-Area IS-IS

```
vtysh -c "configure terminal" -c "
router isis CORE
 net 49.0001.0100.0000.0001.00
 is-type level-1-2
 metric-style wide
 log-adjacency-changes

interface eth0
 ip router isis CORE
 ipv6 router isis CORE
 isis circuit-type level-1-2
 isis network point-to-point

interface lo
 ip router isis CORE
 ipv6 router isis CORE
 isis passive
"
```

Replace:
- `CORE` -- IS-IS process tag (arbitrary name)
- `49.0001.0100.0000.0001.00` -- the NET (Network Entity Title) for this router
- `eth0` -- transit-facing interface
- `lo` -- loopback interface

## NET (Network Entity Title) Addressing

The NET is the ISO address that uniquely identifies each IS-IS router. Format:

```
<AFI>.<Area-ID>.<System-ID>.<SEL>
```

| Field | Length | Example | Meaning |
|---|---|---|---|
| AFI | 1 byte | `49` | Private addressing (always 49 for IP networks) |
| Area ID | 1-13 bytes | `0001` | Area identifier; all routers in same L1 area share this |
| System ID | 6 bytes | `0100.0000.0001` | Unique per router (often derived from loopback IP or MAC) |
| SEL | 1 byte | `00` | Always 00 for a router (NET selector) |

### Derive System ID from Loopback IP

A common convention encodes the loopback IP `10.0.0.1` as system-id `0100.0000.0001`:

| Loopback IP | System ID |
|---|---|
| 10.0.0.1 | 0100.0000.0001 |
| 10.0.0.2 | 0100.0000.0002 |
| 172.16.1.1 | 1720.1600.1001 |
| 192.168.1.254 | 1921.6800.1254 |

### Configure the NET

```
router isis CORE
 net 49.0001.0100.0000.0001.00
```

Multiple area addresses (up to 3) allow area merging/splitting:

```
router isis CORE
 net 49.0001.0100.0000.0001.00
 net 49.0002.0100.0000.0001.00
```

## Configure Router Type (Levels)

| Type | Role | Adjacency | Routing |
|---|---|---|---|
| `level-1` | Intra-area only | L1 only | Routes within one area |
| `level-2-only` | Backbone only | L2 only | Routes between areas |
| `level-1-2` | Both (default) | L1 and L2 | ABR equivalent; leaks routes between levels |

```
router isis CORE
 is-type level-1-2
```

**WARNING:** Changing the IS type drops adjacencies at the removed level. On a production router, this causes immediate traffic loss for routes learned at that level.

## Configure Interface Parameters

### Enable IS-IS on an Interface

```
interface eth0
 ip router isis CORE
 ipv6 router isis CORE
```

Both `ip router isis` and `ipv6 router isis` are needed for dual-stack. Omit one to run single-stack.

### Circuit Type

Restrict which level an interface participates in:

```
interface eth0
 isis circuit-type level-2
```

### Network Type

```
interface eth0
 isis network point-to-point
```

Use point-to-point on links between exactly two routers. This eliminates DIS election and speeds convergence. Default is broadcast.

### Metric

```
interface eth0
 isis metric 100
 isis metric 50 level-1
 isis metric 200 level-2
```

### Timers

```
interface eth0
 isis hello-interval 5
 isis hello-interval 3 level-1
 isis hello-multiplier 4
 isis csnp-interval 30
 isis psnp-interval 5
```

| Timer | Default | Command |
|---|---|---|
| Hello interval | 10s | `isis hello-interval <1-600>` |
| Hello multiplier | 3 | `isis hello-multiplier <2-100>` |
| CSNP interval | 10s | `isis csnp-interval <1-600>` |
| PSNP interval | 2s | `isis psnp-interval <1-120>` |

The hold time (dead interval) = hello-interval x hello-multiplier. Unlike OSPF, IS-IS does not require matching hello/dead timers on both sides of a link.

### Priority (DIS Election)

```
interface eth0
 isis priority 100
 isis priority 64 level-1
```

Default priority is 64. Highest priority wins. Priority 0 does not prevent DIS election (unlike OSPF DR election).

### Passive Interface

```
interface lo
 isis passive
```

Passive interfaces are advertised into IS-IS but do not send or receive hellos.

### Hello Padding

```
interface eth0
 isis hello padding
 isis hello padding during-adjacency-formation
```

Hello padding pads hellos to MTU size to detect MTU mismatches. `during-adjacency-formation` only pads until adjacency is established, saving bandwidth.

### Three-Way Handshake

```
interface eth0
 isis three-way-handshake
```

Enables RFC 5303 point-to-point three-way handshake for reliable adjacency detection.

### Per-Interface Authentication

```
interface eth0
 isis password clear MyInterfacePassword
 isis password md5 MyMD5Password
```

## Configure Metric Style

```
router isis CORE
 metric-style wide
```

| Style | Max Metric | TLV | Use Case |
|---|---|---|---|
| `narrow` | 63 | Old-style TLVs | Legacy interop only |
| `wide` | 16,777,215 | Extended TLVs | Modern networks, required for TE and SR |
| `transition` | Both | Both TLV types | Migration from narrow to wide |

**WARNING:** All routers in an area must agree on metric style. Mismatched styles cause route calculation errors. Always use `wide` for new deployments.

## Configure Authentication

### Area Authentication (Level-1)

```
router isis CORE
 area-password md5 MyAreaPassword
```

### Domain Authentication (Level-2)

```
router isis CORE
 domain-password md5 MyDomainPassword
```

Options: `clear` (plaintext) or `md5`.

**WARNING:** Authentication must match on all routers at the same level. A mismatch prevents LSP acceptance, causing route loss.

## Configure Route Leaking

By default, Level-2 routes are not visible to Level-1-only routers. Route leaking injects L2 routes into the L1 database.

IS-IS route leaking in FRR uses route-maps and redistribution:

```
router isis CORE
 redistribute ipv4 table 254 level-1 route-map L2-TO-L1
```

To leak a default route into Level-1:

```
router isis CORE
 default-information originate ipv4 level-1 always
```

## Configure Redistribution

```
router isis CORE
 redistribute ipv4 connected level-2 metric 100 route-map CONN-TO-ISIS
 redistribute ipv4 static level-2 metric 200
 redistribute ipv4 bgp level-2 route-map BGP-TO-ISIS
 redistribute ipv6 connected level-2 route-map CONN6-TO-ISIS
 redistribute ipv6 static level-2
```

### Default Route Origination

```
router isis CORE
 default-information originate ipv4 level-2 always metric 100
 default-information originate ipv6 level-2 always route-map DEFAULT-CHECK
```

## Configure Overload Bit

The overload bit signals that a router should only be used as a last resort for transit traffic.

```
router isis CORE
 ! Immediate overload (manual maintenance)
 set-overload-bit

 ! Overload on startup for 300 seconds (wait for convergence)
 set-overload-bit on-startup 300
```

Remove overload to resume normal transit:

```
router isis CORE
 no set-overload-bit
```

## Configure Attached Bit

The attached bit on Level-1-2 routers tells Level-1-only routers to use them as default exit points.

```
router isis CORE
 ! Ignore received attached bit (do not install default route)
 attached-bit receive ignore

 ! Do not set the attached bit in own LSPs
 attached-bit send
```

## Configure LSP Parameters

```
router isis CORE
 ! LSP generation throttle (seconds)
 lsp-gen-interval 5
 lsp-gen-interval level-1 2
 lsp-gen-interval level-2 5

 ! LSP refresh interval (seconds, default 900)
 lsp-refresh-interval 900
 lsp-refresh-interval level-1 600

 ! Maximum LSP lifetime (seconds, default 1200)
 max-lsp-lifetime 1200
 max-lsp-lifetime level-2 65535

 ! LSP MTU (bytes, default 1497)
 lsp-mtu 1497

 ! SPF computation interval (seconds)
 spf-interval 1
 spf-interval level-1 1
 spf-interval level-2 5
```

| Parameter | Default | Range |
|---|---|---|
| LSP generation interval | 30s | 1-120s |
| LSP refresh interval | 900s | 1-65235s |
| Max LSP lifetime | 1200s | 350-65535s |
| LSP MTU | 1497 bytes | 128-4352 bytes |
| SPF interval | 1s | 1-120s |

**WARNING:** `max-lsp-lifetime` must be greater than `lsp-refresh-interval`. If LSPs expire before refresh, routes are withdrawn.

## Configure SPF Prefix Priority

Prioritize convergence of critical prefixes (e.g., loopbacks for BGP next-hops):

```
router isis CORE
 spf prefix-priority critical CRITICAL-PREFIXES
 spf prefix-priority high HIGH-PREFIXES
 spf prefix-priority medium MEDIUM-PREFIXES
```

Where `CRITICAL-PREFIXES` is an access-list matching the important prefixes.

## Configure Segment Routing (SR-MPLS)

### Enable SR

```
router isis CORE
 segment-routing on
 segment-routing global-block 16000 23999
 segment-routing node-msd 8
```

### Assign Prefix SIDs

```
router isis CORE
 segment-routing prefix 10.0.0.1/32 index 1
 segment-routing prefix 10.0.0.1/32 index 1 no-php-flag
 segment-routing prefix 10.0.0.1/32 index 1 explicit-null
 segment-routing prefix 2001:db8::1/128 index 101
```

### Absolute SID Labels

```
router isis CORE
 segment-routing prefix 10.0.0.1/32 absolute 16001
```

### Local Block for Adjacency SIDs

```
router isis CORE
 segment-routing global-block 16000 23999 local-block 15000 15999
```

### Verify SR

```
show isis segment-routing node
```

## Configure SRv6

```
router isis CORE
 segment-routing srv6
  locator MY-LOCATOR
  interface eth0
```

### Verify SRv6

```
show isis segment-routing srv6 node
```

## Configure Traffic Engineering

```
router isis CORE
 mpls-te on
 mpls-te router-address 10.0.0.1
 mpls-te router-address ipv6 2001:db8::1
 mpls-te export
```

### Verify TE

```
show isis mpls-te interface
show isis mpls-te interface eth0
show isis mpls-te router
show isis mpls-te database
show isis mpls-te database detail
```

## Configure Fast Reroute

### LFA (Loop-Free Alternates)

```
interface eth0
 isis fast-reroute lfa level-1
 isis fast-reroute lfa level-2
```

Exclude an interface from backup computation:

```
interface eth1
 isis fast-reroute lfa exclude interface eth2
```

### Remote LFA

```
interface eth0
 isis fast-reroute remote-lfa tunnel mpls-ldp level-1

router isis CORE
 fast-reroute remote-lfa prefix-list RLFA-TUNNELS level-1
 fast-reroute remote-lfa maximum-metric 100 level-1
```

### TI-LFA

```
interface eth0
 isis fast-reroute ti-lfa level-1
 isis fast-reroute ti-lfa level-2 node-protection
 isis fast-reroute ti-lfa level-2 node-protection link-fallback
```

### FRR Priority and Tiebreakers

```
router isis CORE
 fast-reroute priority-limit critical level-1
 fast-reroute lfa tiebreaker downstream index 10 level-1
 fast-reroute lfa tiebreaker lowest-backup-metric index 20 level-1
 fast-reroute lfa tiebreaker node-protecting index 30 level-1
 fast-reroute load-sharing disable level-1
```

### Verify FRR

```
show isis fast-reroute summary level-1
show isis fast-reroute summary level-2
show isis route level-1 backup
```

## Configure Flex-Algo (RFC 9350)

### Define Flex-Algo

```
router isis CORE
 flex-algo 128
  advertise-definition
  priority 200
  metric-type delay

 flex-algo 129
  advertise-definition
  metric-type te
  affinity exclude-any RED
  affinity include-any GREEN
```

### Define Affinity Maps

```
affinity-map RED bit-position 0
affinity-map GREEN bit-position 1
affinity-map BLUE bit-position 2
```

### Assign Interface Affinities

```
interface eth0
 isis affinity flex-algo RED
```

### Assign Prefix SIDs for Flex-Algo

```
router isis CORE
 segment-routing prefix 10.0.0.1/32 algorithm 128 index 501
 segment-routing prefix 10.0.0.1/32 algorithm 129 index 601
```

### Verify Flex-Algo

```
show isis flex-algo
show isis flex-algo 128
show isis topology algorithm 128
show isis route algorithm 128
show isis segment-routing node algorithm 128
```

## Configure Multi-Topology

Multi-topology allows separate SPF computations for IPv4 and IPv6, so each protocol can follow different paths.

Enable on the router:

```
router isis CORE
 topology ipv6-unicast
```

IS-IS multi-topology support in FRR allows IPv4 and IPv6 topologies to diverge. Without multi-topology, IPv4 and IPv6 share the same shortest-path tree.

## Configure Hostname Mapping

```
router isis CORE
 hostname dynamic
```

Dynamic hostname maps system-ids to hostnames in `show` output for readability.

## Configure Logging

```
router isis CORE
 log-adjacency-changes
 log-pdu-drops
```

## Configure Purge Originator

```
router isis CORE
 purge-originator
```

Adds the purge originator identification TLV (RFC 6232) to purged LSPs, aiding in troubleshooting.

## Configure Advertise High Metrics

```
router isis CORE
 advertise-high-metrics
```

Sets all interface metrics to the maximum value, effectively draining traffic (similar to OSPF max-metric).

## Configure Advertise Passive Only

```
router isis CORE
 advertise-passive-only
```

Only advertises prefixes from passive interfaces. Transit link prefixes are suppressed.

## Essential Show Commands

```
! Process summary
show isis summary
show isis summary json

! Hostname database
show isis hostname

! Interface status
show isis interface
show isis interface detail
show isis interface eth0

! Neighbor state
show isis neighbor
show isis neighbor detail
show isis neighbor detail SYSTEMID

! Link-state database
show isis database
show isis database detail
show isis database detail LSPID

! Topology (shortest-path tree)
show isis topology
show isis topology level-1
show isis topology level-2

! Routing table
show isis route
show isis route level-1
show isis route level-2
show isis route prefix-sid

! VRF-aware queries
show isis vrf CUSTOMER1 summary
show isis vrf all neighbor
```

## Complete IS-IS Configuration Example

```
! Router R1 -- L1-L2 router with SR and TI-LFA
affinity-map RED bit-position 0
affinity-map GREEN bit-position 1

router isis CORE
 net 49.0001.0100.0000.0001.00
 is-type level-1-2
 metric-style wide
 log-adjacency-changes
 hostname dynamic
 !
 segment-routing on
 segment-routing global-block 16000 23999
 segment-routing node-msd 8
 segment-routing prefix 10.0.0.1/32 index 1
 segment-routing prefix 2001:db8::1/128 index 101
 !
 mpls-te on
 mpls-te router-address 10.0.0.1
 !
 redistribute ipv4 connected level-2 route-map CONN-TO-ISIS
 default-information originate ipv4 level-1 always

interface lo
 ip router isis CORE
 ipv6 router isis CORE
 isis passive

interface eth0
 ip router isis CORE
 ipv6 router isis CORE
 isis circuit-type level-2
 isis network point-to-point
 isis metric 10
 isis fast-reroute ti-lfa level-2 node-protection
 isis password md5 BackboneSecret

interface eth1
 ip router isis CORE
 ipv6 router isis CORE
 isis circuit-type level-1
 isis network point-to-point
 isis metric 10
 isis fast-reroute ti-lfa level-1

interface eth2
 ip router isis CORE
 ipv6 router isis CORE
 isis passive
```

## Debugging

```
! Adjacency debugging
debug isis adj-packets
debug isis events

! SPF computation
debug isis spf-events

! LSP/SNP packets
debug isis update-packets
debug isis snp-packets

! Routing decisions
debug isis route-events

! Feature-specific
debug isis sr-events
debug isis te-events
debug isis lfa
debug isis flooding
debug isis bfd
debug isis ldp-sync

! Raw packet dumps
debug isis packet-dump

! Database sync
debug isis tx-queue
```

**WARNING:** IS-IS debug output can be extremely verbose in large networks. Always remove debug commands after troubleshooting with `no debug isis` or `undebug all`.
