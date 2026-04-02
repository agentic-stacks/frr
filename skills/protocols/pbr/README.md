# PBR Protocol Skill

## Overview

PBR (Policy-Based Routing) enables forwarding decisions based on packet fields other than the destination IP address, implemented by the `pbrd` daemon in FRR. PBR matches traffic using source/destination addresses, ports, protocols, DSCP, and other fields, then forwards via specified nexthop groups or modifies VRF lookup tables.

Key facts:

- Daemon: `pbrd` (enable in `/etc/frr/daemons`)
- Platform: Linux only (some match/set actions unsupported by default kernel dataplane)
- Kernel mechanism: `ip rule` entries + routing tables with nexthop routes
- Default table range: 10000-4294966272
- Sequence range: 1-700 per PBR map
- Interacts with: zebra (installs rules/routes into kernel)

> **When to use PBR:** Use PBR when destination-based routing is insufficient -- for example, steering traffic from specific sources through a firewall, load-balancing across multiple ISP links by source subnet, or directing VoIP traffic to a low-latency path. PBR adds complexity; prefer IGP/BGP-based traffic engineering when possible.

## Enable the PBR Daemon

Edit `/etc/frr/daemons`:

```
pbrd=yes
```

Reload FRR:

```
sudo systemctl reload frr
```

## Quick-Start: Route by Source

Forward traffic from 10.10.10.0/24 destined to 9.9.9.0/24 through a specific nexthop group:

```
vtysh -c "configure terminal" -c "
nexthop-group ISP-A
 nexthop 4.5.6.7
!
pbr-map POLICY-ROUTE seq 100
 match src-ip 10.10.10.0/24
 match dst-ip 9.9.9.0/24
 set nexthop-group ISP-A
!
interface eth0
 pbr-policy POLICY-ROUTE
"
```

Replace:
- `10.10.10.0/24` -- the source subnet to match
- `9.9.9.0/24` -- the destination subnet to match
- `4.5.6.7` -- the nexthop IP to forward matched traffic through
- `eth0` -- the ingress interface

## Nexthop Groups

Nexthop groups define where matched traffic is forwarded. They support ECMP, weighted load balancing, backup paths, and MPLS labels.

### Single Nexthop Group

```
nexthop-group UPSTREAM-A
 nexthop 10.0.0.1
```

### ECMP Nexthop Group

```
nexthop-group LOAD-BALANCE
 nexthop 10.0.0.1
 nexthop 10.0.1.1
 nexthop 10.0.2.1
```

### Weighted ECMP

```
nexthop-group WEIGHTED-ISP
 nexthop 10.0.0.1 weight 3
 nexthop 10.0.1.1 weight 1
```

### Interface-Specific Nexthop

```
nexthop-group VIA-ETH1
 nexthop 10.0.0.1 eth1
```

With `onlink` flag (nexthop is on a directly connected subnet even if ARP says otherwise):

```
nexthop-group VIA-ETH1
 nexthop 10.0.0.1 eth1 onlink
```

### VRF-Aware Nexthop

```
nexthop-group VRF-EXIT
 nexthop 10.0.0.1 nexthop-vrf INTERNET
```

### MPLS Labels

```
nexthop-group MPLS-PATH
 nexthop 10.0.0.1 label 100/200
```

Up to 16 labels separated by `/`, range 16-1048575.

### Backup Nexthop Group

```
nexthop-group BACKUP-PATH
 nexthop 10.99.0.1
!
nexthop-group PRIMARY-PATH
 backup-group BACKUP-PATH
 nexthop 10.0.0.1
```

### Resilient Hashing (Linux 5.19+)

```
nexthop-group RESILIENT-ECMP
 resilient buckets 128 idle-timer 120 unbalanced-timer 300
 nexthop 10.0.0.1
 nexthop 10.0.1.1
 nexthop 10.0.2.1
```

Resilient hashing minimizes flow redistribution when a nexthop fails. The `resilient` command must be the first entry in the group.

> **Warning:** Resilient hashing requires Linux kernel 5.19 or later.

### Blackhole Nexthop

```
nexthop-group DROP-IT
 nexthop blackhole
```

## PBR Maps

PBR maps contain sequenced rules with match conditions and set actions. Rules are evaluated in sequence order (lowest first); the first match wins.

### Create a PBR Map Rule

```
pbr-map MAP-NAME seq SEQUENCE
 match ...
 set ...
```

Sequence numbers range from 1 to 700.

### Match Conditions

| Match Command | Description |
|---|---|
| `match src-ip A.B.C.D/M` | Source IPv4 prefix |
| `match dst-ip A.B.C.D/M` | Destination IPv4 prefix |
| `match src-ip X:X::X:X/M` | Source IPv6 prefix |
| `match dst-ip X:X::X:X/M` | Destination IPv6 prefix |
| `match src-port (1-65535)` | Source TCP/UDP port |
| `match dst-port (1-65535)` | Destination TCP/UDP port |
| `match ip-protocol PROTOCOL` | IP protocol name from `/etc/protocols` |
| `match mark (1-4294967295)` | Kernel/dataplane packet mark |
| `match dscp DSCP_VALUE` | DSCP value (numeric or named: cs0-cs7, af11-af43, ef, va) |
| `match ecn (0-3)` | ECN field value |

Multiple match statements in a single rule form an AND condition -- all must match.

### Set Actions

| Set Command | Description |
|---|---|
| `set nexthop-group NAME` | Forward via nexthop group |
| `set nexthop A.B.C.D\|X:X::X:X` | Forward via single nexthop |
| `set nexthop blackhole` | Drop matched traffic |
| `set vrf NAME` | Change VRF lookup table |
| `set queue-id (0-65535)` | Set egress queue |

> **Note:** Packet mangling operations (rewrite source/destination IP, rewrite ports, set PCP/VLAN) are defined in FRR but unsupported by the default Linux kernel dataplane.

### Example: Multi-Rule PBR Map

```
pbr-map MULTI-POLICY seq 100
 match src-ip 10.10.0.0/16
 match dst-port 443
 match ip-protocol tcp
 set nexthop-group HTTPS-PATH
!
pbr-map MULTI-POLICY seq 200
 match src-ip 10.20.0.0/16
 set nexthop-group BACKUP-ISP
!
pbr-map MULTI-POLICY seq 300
 match dscp ef
 set nexthop-group LOW-LATENCY
```

## Apply PBR Policy to Interfaces

PBR maps are applied to ingress interfaces:

```
interface eth0
 pbr-policy MULTI-POLICY
!
interface eth1
 pbr-policy MULTI-POLICY
```

> **Warning:** PBR policies do not cascade to sub-interfaces. Each sub-interface requires explicit `pbr-policy` assignment.

To remove a policy:

```
interface eth0
 no pbr-policy MULTI-POLICY
```

## Table ID Range

PBR internally creates Linux routing tables. Control the table ID range:

```
pbr table range 10000 50000
```

Range: 10000-4294966272. Avoid overlapping with tables used by VRFs or other applications.

## Kernel Interaction

When a PBR map is applied, FRR (via zebra) installs:

1. **`ip rule`** entries matching the configured conditions (source, destination, protocol, port, DSCP, mark)
2. **Routing table** entries with default routes pointing to the specified nexthops

Verify with Linux commands:

```bash
ip rule show
ip route show table 10000
```

## Show Commands

### Show PBR Maps

```
show pbr map
show pbr map MULTI-POLICY
show pbr map MULTI-POLICY detail
```

Shows rule details, match conditions, set actions, and installation status (whether the rule is active in the kernel).

### Show Nexthop Groups

```
show pbr nexthop-groups
show pbr nexthop-groups ISP-A
```

Shows group validity, installation status, and contained nexthops.

### Show Interfaces

```
show pbr interface
show pbr interface eth0
```

Shows which policies are applied to which interfaces.

### JSON Output

All show commands support `json` suffix:

```
show pbr map json
show pbr nexthop-groups json
show pbr interface json
```

## Debug Commands

| Command | What It Shows |
|---|---|
| `debug pbr events` | PBR event processing |
| `debug pbr map` | PBR map evaluation and matching |
| `debug pbr nht` | Nexthop tracking events |
| `debug pbr zebra` | Zebra interaction (rule/route installs) |

## Complete Configuration Example

Dual-ISP policy routing with DSCP-based path selection:

```
! /etc/frr/frr.conf
!
! Define nexthop groups for each ISP
nexthop-group ISP-PRIMARY
 nexthop 203.0.113.1 eth0
!
nexthop-group ISP-SECONDARY
 nexthop 198.51.100.1 eth1
!
nexthop-group ISP-ECMP
 nexthop 203.0.113.1 eth0
 nexthop 198.51.100.1 eth1
!
! VoIP traffic (EF DSCP) goes to primary (low latency)
pbr-map DUAL-ISP seq 100
 match dscp ef
 set nexthop-group ISP-PRIMARY
!
! Bulk traffic (CS1) goes to secondary (high bandwidth)
pbr-map DUAL-ISP seq 200
 match dscp cs1
 set nexthop-group ISP-SECONDARY
!
! Server subnet always uses primary
pbr-map DUAL-ISP seq 300
 match src-ip 10.100.0.0/24
 set nexthop-group ISP-PRIMARY
!
! Everything else load-balances
pbr-map DUAL-ISP seq 400
 match src-ip 10.0.0.0/8
 set nexthop-group ISP-ECMP
!
! Apply to LAN-facing interfaces
interface eth2
 pbr-policy DUAL-ISP
!
interface eth3
 pbr-policy DUAL-ISP
```

## Verify PBR Operation

```
! Check PBR maps are installed
show pbr map

! Check nexthop groups are valid
show pbr nexthop-groups

! Check interfaces have policies
show pbr interface

! Verify kernel rules
! (from Linux shell)
ip rule show
ip route show table 10000
```
