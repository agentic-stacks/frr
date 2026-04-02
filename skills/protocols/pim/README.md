# PIM Protocol Skill

## Overview

PIM (Protocol Independent Multicast) enables multicast routing in FRR, implemented by the `pimd` daemon (IPv4) and `pim6d` daemon (IPv6). PIM is "protocol independent" because it relies on whatever unicast routing protocol populates the routing table for RPF (Reverse Path Forwarding) checks rather than maintaining its own topology database.

Key facts:

- Daemons: `pimd` (IPv4), `pim6d` (IPv6) -- enable in `/etc/frr/daemons`
- Modes: PIM-SM (Sparse Mode), PIM-DM (Dense Mode), PIM-SSM (Source-Specific Multicast)
- Host membership: IGMP (IPv4), MLD (IPv6)
- RP discovery: static, BSR (Bootstrap Router), Auto-RP
- Inter-domain: MSDP (Multicast Source Discovery Protocol)
- Kernel: Linux 4.19+ for PIM-SM; 5.0+ for EVPN BUM forwarding
- VRF-aware: yes -- most commands accept `[vrf NAME]`

> **PIM-SM vs PIM-SSM:** Sparse Mode requires a Rendezvous Point (RP) where sources register and receivers join. SSM (default range `232.0.0.0/8` for IPv4, `ff3x::/32` for IPv6) bypasses the RP entirely -- receivers specify both source and group, so traffic flows directly from source to receiver via the shortest path tree. Use SSM whenever applications can supply the source address (e.g., IPTV, live streaming).

## Enable the PIM Daemon

Edit `/etc/frr/daemons`:

```
pimd=yes
# pim6d=yes    # uncomment for IPv6 multicast
```

Reload FRR:

```
sudo systemctl reload frr
```

### pimd Startup Options

Set in `/etc/frr/daemons` via `pimd_options`:

| Flag | Purpose |
|---|---|
| Standard FRR flags | See `pimd --help` |

Signals:

| Signal | Behavior |
|---|---|
| `SIGUSR1` | Rotate log file |
| `SIGINT` / `SIGTERM` | Graceful shutdown -- clears installed mroutes first |

## Quick-Start: Static RP Multicast

```
vtysh -c "configure terminal" -c "
interface eth0
 ip pim
 ip igmp
!
interface eth1
 ip pim
 ip igmp
!
router pim
 rp 10.0.0.1 224.0.0.0/4
"
```

This enables PIM and IGMP on two interfaces and designates `10.0.0.1` as the RP for all multicast groups. Every PIM router in the domain must have the same static RP configuration.

> **Warning:** If static RP addresses are inconsistent across routers, multicast will silently blackhole. Use BSR or Auto-RP in larger deployments for automatic RP agreement.

## Rendezvous Point Configuration

### Static RP

```
router pim
 rp 10.0.0.1 224.0.0.0/4
```

Assign specific groups to different RPs:

```
router pim
 rp 10.0.0.1 239.1.0.0/16
 rp 10.0.0.2 239.2.0.0/16
```

### BSR (Bootstrap Router)

BSR dynamically elects an RP across the domain. Configure candidate BSR and candidate RP routers:

On the BSR candidate:

```
router pim
 bsr candidate-bsr priority 200
```

On the RP candidate:

```
router pim
 bsr candidate-rp interval 60 priority 10
```

- Highest BSR priority wins election
- Lowest RP priority wins for each group range
- BSR floods RP-set via bootstrap messages on all PIM interfaces

### Auto-RP

```
router pim
 autorp announce 10.0.0.1
```

Disable Auto-RP discovery if not used:

```
router pim
 no autorp discovery
```

> **Note:** Auto-RP is a legacy Cisco mechanism. Prefer BSR for new deployments. Auto-RP uses dense-mode flooding for discovery, which can cause unexpected traffic.

## IGMP Configuration (IPv4 Host Membership)

### Enable IGMP

```
interface eth0
 ip igmp
```

IGMP is enabled automatically when `ip pim` is configured, but explicit `ip igmp` allows fine-tuning.

### IGMP Commands

| Command | Purpose |
|---|---|
| `ip igmp` | Enable IGMP (default: v3) |
| `ip igmp version (2-3)` | Set IGMP version |
| `ip igmp query-interval (1-65535)` | Query sending interval (seconds) |
| `ip igmp query-max-response-time (1-65535)` | Max response time (seconds) |
| `ip igmp immediate-leave` | Remove group instantly on IGMPv2 Leave |
| `ip igmp join-group A.B.C.D [A.B.C.D]` | Join group via local socket |
| `ip igmp static-group A.B.C.D [A.B.C.D]` | Simulate receiver without IGMP reports |
| `ip igmp max-groups (0-4294967295)` | Maximum groups per interface |
| `ip igmp max-sources (0-4294967295)` | Maximum sources per group |
| `ip igmp robustness (1-255)` | Robustness variable (default: 2) |
| `ip igmp last-member-query-count (1-255)` | Leave verification query count |
| `ip igmp last-member-query-interval (1-65535)` | Leave query interval (deciseconds) |
| `ip igmp access-list WORD` | Filter IGMP joins by ACL |
| `ip igmp route-map WORD` | Filter IGMP joins by route-map |
| `ip igmp require-router-alert` | Accept only RA-flagged reports |
| `ip igmp proxy` | Forward joins from other interfaces |

### Static Group Example

Force an interface to always forward traffic for a group, even without receivers:

```
interface eth1
 ip igmp static-group 239.1.1.1
```

## MLD Configuration (IPv6 Host Membership)

### Enable MLD

```
interface eth0
 ipv6 mld
```

### MLD Commands

| Command | Purpose |
|---|---|
| `ipv6 mld` | Enable MLD (default: v2) |
| `ipv6 mld version (1-2)` | Set MLD version |
| `ipv6 mld query-interval (1-65535)` | Query frequency (seconds) |
| `ipv6 mld query-max-response-time (1-65535)` | Response threshold (seconds) |
| `ipv6 mld immediate-leave` | Remove group on MLDv1 Done |
| `ipv6 mld join X:X::X:X [Y:Y::Y:Y]` | Join group or (S,G) |
| `ipv6 mld max-groups (0-4294967295)` | Max groups per interface |
| `ipv6 mld max-sources (0-4294967295)` | Max sources per group |
| `ipv6 mld robustness (1-255)` | Robustness variable |
| `ipv6 mld last-member-query-count (1-255)` | Leave query count |
| `ipv6 mld last-member-query-interval (1-65535)` | Leave query interval (deciseconds) |
| `ipv6 mld access-list WORD` | Filter MLD joins by ACL |
| `ipv6 mld route-map WORD` | Filter MLD joins by route-map |
| `ipv6 mld require-router-alert` | Require RA option |

## SSM (Source-Specific Multicast)

### Default SSM Range

IPv4: `232.0.0.0/8` -- no RP required for groups in this range.

### Custom SSM Range

```
ip prefix-list SSM_GROUPS seq 5 permit 239.232.0.0/16
!
router pim
 ssm prefix-list SSM_GROUPS
```

For IPv6:

```
ipv6 prefix-list SSM6_GROUPS seq 5 permit ff3e::/32
!
router pim6
 ssm prefix-list SSM6_GROUPS
```

Applications using SSM must use IGMPv3 (IPv4) or MLDv2 (IPv6) to specify source addresses in their join requests.

## Interface Configuration

### PIM Interface Commands

| Command | Purpose |
|---|---|
| `ip pim` | Enable PIM sparse mode |
| `ip pim passive` | Listen only -- no control packets sent |
| `ip pim drpriority (0-4294967295)` | Designated Router priority |
| `ip pim hello (1-65535) (1-65535)` | Hello interval and hold time (seconds) |
| `ip pim join-prune-interval (5-600)` | Join/prune timer per interface |
| `ip pim use-source A.B.C.D` | Source address for PIM packets |
| `ip pim bsm` | Process bootstrap messages (default: on) |
| `ip pim unicast-bsm` | Accept unicast BSMs |
| `ip pim active-active` | VXLAN MLAG active-active mode |
| `ip pim allowed-neighbors prefix-list WORD` | Restrict PIM peers by prefix-list |
| `ip pim assert-interval (1000-86400000)` | Assert timer (milliseconds) |
| `ip pim override-interval (0-65535)` | LAN prune delay override (ms) |

### PIMv6 Interface Commands

| Command | Purpose |
|---|---|
| `ipv6 pim` | Enable PIMv6 |
| `ipv6 pim passive` | Listen only |
| `ipv6 pim drpriority (0-4294967295)` | DR priority |
| `ipv6 pim hello (1-65535) (1-65535)` | Hello and hold intervals |
| `ipv6 pim use-source X:X::X:X` | Source address selection |
| `ipv6 pim bsm` | Process BSMs |
| `ipv6 pim unicast-bsm` | Accept unicast BSMs |
| `ipv6 pim active-active` | VXLAN MLAG active-active |
| `ipv6 pim allowed-neighbors prefix-list WORD` | Restrict PIM peers |

## PIMv6 Configuration

### Enable PIMv6

```
router pim6
 rp 2001:db8::1 ff00::/8
!
interface eth0
 ipv6 pim
 ipv6 mld
```

### Embedded RP (IPv6 Only)

PIMv6 supports learning the RP address from the multicast group address itself (RFC 3956, `FF70::/12` range):

```
router pim6
 embedded-rp
 embedded-rp group-list EMBEDDED_RP_FILTER
 embedded-rp limit 100
```

### BSR for IPv6

```
router pim6
 bsr candidate-bsr priority 200
 bsr candidate-rp interval 60 priority 10
```

### SPT Switchover Control

Prevent switchover from shared tree to shortest-path tree (keeps traffic through RP):

```
router pim
 spt-switchover infinity-and-beyond
```

For specific groups only:

```
router pim6
 spt-switchover infinity-and-beyond prefix-list NO_SPT_GROUPS
```

## Multicast Routing Table and OIL

### Static Multicast Routes

Override RPF lookups with static entries in the Multicast RIB:

```
ip mroute 10.1.0.0/24 10.0.0.1
ip mroute 10.1.0.0/24 10.0.0.1 150
```

The optional distance (150) controls preference over dynamic routes.

### RPF Lookup Modes

```
router pim
 rpf-lookup-mode MODE
```

| Mode | Behavior |
|---|---|
| `urib-only` | Use unicast RIB exclusively |
| `mrib-only` | Use multicast RIB exclusively |
| `mrib-then-urib` | Try MRIB first, fall back to URIB (default) |
| `lower-distance` | Prefer the route with lower admin distance |
| `longer-prefix` | Prefer the more specific route |

### Multicast Boundaries

Restrict multicast traffic at interface level:

```
interface eth2
 ip multicast boundary oil NO_MULTICAST_PREFIXES
 ip multicast boundary MULTICAST_ACL
```

### ECMP for Multicast

```
router pim
 ecmp
 ecmp rebalance
```

`ecmp` load-balances across equal-cost nexthops. `ecmp rebalance` redistributes flows when an interface goes down.

## MSDP (Multicast Source Discovery Protocol)

MSDP shares active source information between PIM-SM domains, typically between Anycast RP peers or across autonomous system boundaries.

### Configure an MSDP Peer

```
router pim
 msdp peer 10.0.0.2 source 10.0.0.1
```

With eBGP loop detection (requires `bgp send-extra-data zebra`):

```
router pim
 msdp peer 10.0.0.2 source 10.0.0.1 as 65002
```

### MSDP Mesh Groups

A mesh group prevents SA flooding loops among a set of MSDP peers:

```
router pim
 msdp mesh-group ANYCAST_RP member 10.0.0.2
 msdp mesh-group ANYCAST_RP member 10.0.0.3
 msdp mesh-group ANYCAST_RP source 10.0.0.1
```

### MSDP Tuning

| Command | Purpose |
|---|---|
| `msdp timers (1-65535) (1-65535) [(1-65535)]` | Keep-alive, hold, retry intervals |
| `msdp peer A.B.C.D sa-filter ACL in/out` | Filter SA advertisements |
| `msdp peer A.B.C.D sa-limit AMOUNT` | Max SAs learned from peer |
| `msdp peer A.B.C.D password WORD` | MD5 authentication |
| `msdp originator-id A.B.C.D` | Custom originator ID |
| `msdp shutdown` | Administratively disable all sessions |

### Full MSDP Example: Anycast RP

Two RPs sharing the same anycast address `10.99.0.1` with MSDP:

**RP-1 (loopback 10.0.0.1, anycast 10.99.0.1):**

```
interface lo
 ip address 10.0.0.1/32
 ip address 10.99.0.1/32
!
router pim
 rp 10.99.0.1 224.0.0.0/4
 msdp peer 10.0.0.2 source 10.0.0.1
 msdp mesh-group ANYCAST member 10.0.0.2
 msdp mesh-group ANYCAST source 10.0.0.1
```

**RP-2 (loopback 10.0.0.2, anycast 10.99.0.1):**

```
interface lo
 ip address 10.0.0.2/32
 ip address 10.99.0.1/32
!
router pim
 rp 10.99.0.1 224.0.0.0/4
 msdp peer 10.0.0.1 source 10.0.0.2
 msdp mesh-group ANYCAST member 10.0.0.1
 msdp mesh-group ANYCAST source 10.0.0.2
```

## Router-Level PIM Commands

| Command | Purpose |
|---|---|
| `router pim [vrf NAME]` | Enter PIM configuration context |
| `rp A.B.C.D A.B.C.D/M` | Static RP for group range |
| `rp keep-alive-timer (1-65535)` | RP keepalive timeout (seconds) |
| `keep-alive-timer (1-65535)` | S,G flow timeout |
| `join-prune-interval (1-65535)` | Join/prune interval (default: 60s) |
| `register-suppress-time (1-65535)` | FHR register suppression period |
| `register-accept-list PLIST` | Filter register sources |
| `packets (1-255)` | Batch packet processing count |
| `send-v6-secondary` | Include IPv6 secondaries in hellos |
| `join-filter route-map WORD` | Filter incoming joins |

## Show Commands

### PIM State

| Command | Output |
|---|---|
| `show ip pim interface` | PIM-enabled interfaces, DR, neighbors |
| `show ip pim neighbor` | PIM peer status and uptime |
| `show ip pim join` | Received join information |
| `show ip pim upstream [A.B.C.D]` | S,G upstream state |
| `show ip pim state` | S,G forwarding state and OIL |
| `show ip pim rp-info [A.B.C.D/M]` | Configured and learned RPs |
| `show ip pim nexthop` | RPF nexthop tracking |
| `show ip pim assert` | Assert winner per interface |
| `show ip pim group-type` | SSM range membership |

### Multicast Routes

| Command | Output |
|---|---|
| `show ip mroute [A.B.C.D] [fill] [json]` | Installed S,G mroutes in kernel |
| `show ip mroute count [json]` | Per-interface packet counts |
| `show ip mroute summary [json]` | Mroute totals |
| `show ip multicast [count]` | Interface multicast statistics |

### IGMP State

| Command | Output |
|---|---|
| `show ip igmp interface` | IGMP interface statistics |
| `show ip igmp groups [INTERFACE] [detail]` | Active multicast groups |
| `show ip igmp sources` | Learned IGMP sources |
| `show ip igmp join` | Static join entries |
| `show ip igmp statistics` | Protocol counters |

### BSR State

| Command | Output |
|---|---|
| `show ip pim bsr` | Current BSR and uptime |
| `show ip pim bsr candidate-bsr` | Local BSR candidacy |
| `show ip pim bsr candidate-rp` | Local RP candidacy |
| `show ip pim bsr candidate-rp-database` | Learned RP candidates |
| `show ip pim bsr groups` | Group-to-RP mappings |
| `show ip pim bsm-database` | Stored BSM fragments |

### MSDP State

| Command | Output |
|---|---|
| `show ip msdp peer` | MSDP peer connections and SA counts |
| `show ip msdp mesh-group` | Mesh-group membership and status |

### RPF Verification

| Command | Output |
|---|---|
| `show ip rpf` | Entire Multicast RIB |
| `show ip rpf A.B.C.D` | RPF lookup for a specific source |
| `show ip pim nexthop-lookup SRC [GROUP]` | Nexthop per configured RPF mode |

### PIMv6 Show Commands

| Command | Output |
|---|---|
| `show ipv6 pim interface` | PIMv6-enabled interfaces |
| `show ipv6 pim neighbor [detail]` | IPv6 PIM peers |
| `show ipv6 pim join` | PIMv6 join state |
| `show ipv6 pim upstream` | S,G upstream state |
| `show ipv6 pim rp-info` | Configured RPs |
| `show ipv6 pim state` | Forwarding state and OIL |
| `show ipv6 mroute [X:X::X:X]` | IPv6 mroutes |
| `show ipv6 mld interface` | MLD interface state |
| `show ipv6 mld groups` | MLD group membership |
| `show ipv6 mld joins` | MLD join details |

> **Tip:** Append `json` to most show commands for machine-parseable output. Use `vrf NAME` or `vrf all` for VRF-scoped queries.

## Troubleshooting

### RPF Failures

RPF (Reverse Path Forwarding) failures are the most common cause of multicast blackholes.

**Symptom:** `show ip mroute` shows `(S,G)` entries with no OIL (Outgoing Interface List) or missing entries entirely.

**Diagnose:**

```
show ip rpf 10.1.1.1
show ip pim rpf
debug pim nht detail
```

**Common causes:**

| Cause | Fix |
|---|---|
| No unicast route to source | Add static route or fix IGP |
| Source reachable via wrong interface | Check asymmetric routing; add `ip mroute` |
| ECMP causing RPF flaps | Use `router pim` > `ecmp` or add static mroute |
| VRF mismatch | Ensure PIM and unicast routes are in same VRF |
| RPF lookup mode wrong for topology | Change `rpf-lookup-mode` |

### No Multicast Traffic Forwarding

```
# Step 1: Verify PIM neighbors
show ip pim neighbor

# Step 2: Verify IGMP groups on receiver interface
show ip igmp groups

# Step 3: Verify mroute state
show ip mroute

# Step 4: Check OIL (Outgoing Interface List)
show ip pim state

# Step 5: Verify RP assignment
show ip pim rp-info
```

### RP Issues

| Symptom | Diagnosis |
|---|---|
| No RP shown | `show ip pim rp-info` -- check static config or BSR |
| Wrong RP elected | `show ip pim bsr` -- verify priorities |
| Register messages dropped | `show ip pim upstream` -- check register-accept-list |
| Sources not registering at RP | Verify PIM enabled on source-side interfaces |

### IGMP Issues

| Symptom | Diagnosis |
|---|---|
| Groups not appearing | `show ip igmp groups` -- check IGMP version mismatch |
| Slow group leave | Set `ip igmp immediate-leave` or reduce last-member timers |
| Too many groups | Set `ip igmp max-groups` and check `ip igmp access-list` |

### MSDP Issues

| Symptom | Diagnosis |
|---|---|
| Peer not connecting | Verify TCP reachability, source address, and BGP config |
| No SAs learned | Check `show ip msdp peer` -- verify sa-filter and sa-limit |
| SA loop detection failing | Enable `bgp send-extra-data zebra` for AS-path info |

## Debug Commands

| Command | Purpose |
|---|---|
| `debug pim events` | Timer and state change events |
| `debug pim packets` | PIM packet generation and handling |
| `debug pim nht [detail]` | Nexthop tracking and RPF lookups |
| `debug pim bsm` | Bootstrap message processing |
| `debug pim autorp` | Auto-RP protocol events |
| `debug pim zebra` | ZAPI event handling |
| `debug igmp` | IGMP protocol activity |
| `debug mroute` | Kernel MFC interaction |
| `debug pimv6 events` | PIMv6 state changes |
| `debug pimv6 packets` | PIMv6 packet handling |
| `debug pimv6 nht [detail]` | PIMv6 nexthop tracking |
| `debug mld` | MLD protocol activity |
| `debug mroute6` | IPv6 kernel MFC interaction |
| `debug pim packet-dump` | Full packet content dump |
| `debug pim trace` | Code execution tracing |

> **Warning:** `debug pim packet-dump` and `debug pim trace` produce extremely verbose output. Use only for short-duration troubleshooting on specific interfaces. Disable promptly to avoid log flooding.

## Clear Commands

| Command | Purpose |
|---|---|
| `clear ip pim interfaces` | Reset PIM interfaces |
| `clear ip pim oil` | Rescan OIL |
| `clear ip pim bsr-data` | Clear BSM scope data |
| `clear ip igmp interfaces` | Reset IGMP interfaces |
| `clear ip mroute` | Clear all mroutes |
| `clear ip mroute count` | Zero packet counters |
| `clear ip msdp peer A.B.C.D` | Reset MSDP peer connection |
| `clear ipv6 pim interfaces` | Reset PIMv6 interfaces |
| `clear ipv6 pim oil` | Rescan IPv6 OIL |
| `clear ipv6 mld interfaces` | Reset MLD state |
| `clear ipv6 mroute` | Clear IPv6 mroutes |

## Full Example: Dual-Stack Multicast with BSR

```
! === Enable daemons in /etc/frr/daemons ===
! pimd=yes
! pim6d=yes

! === IPv4 PIM with BSR ===
interface eth0
 ip pim
 ip igmp
!
interface eth1
 ip pim
 ip igmp
!
interface lo
 ip pim
!
router pim
 bsr candidate-bsr priority 200
 bsr candidate-rp interval 60 priority 10
 ecmp
 ecmp rebalance

! === IPv6 PIM with static RP ===
interface eth0
 ipv6 pim
 ipv6 mld
!
interface eth1
 ipv6 pim
 ipv6 mld
!
interface lo
 ipv6 pim
!
router pim6
 rp 2001:db8::1 ff00::/8
```

## Quick Reference: Verification Checklist

```bash
# 1. PIM neighbors up?
vtysh -c "show ip pim neighbor"

# 2. RP learned?
vtysh -c "show ip pim rp-info"

# 3. IGMP groups present on receiver-side interface?
vtysh -c "show ip igmp groups"

# 4. Mroute installed with OIL?
vtysh -c "show ip mroute"

# 5. RPF correct for source?
vtysh -c "show ip rpf 10.1.1.1"

# 6. Kernel forwarding active?
vtysh -c "show ip mroute count"
```
