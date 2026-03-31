# VRRP Protocol Skill

## Overview

VRRP (Virtual Router Redundancy Protocol) provides gateway redundancy in FRR, implemented by the `vrrpd` daemon. VRRP allows multiple routers on the same LAN segment to share a virtual IP address -- when the Master fails, a Backup with the highest priority takes over, providing seamless default gateway failover for hosts.

Key facts:

- Daemon: `vrrpd` (enable in `/etc/frr/daemons`)
- Versions: VRRPv2 (RFC 3768, IPv4 only), VRRPv3 (RFC 5798, IPv4 + IPv6)
- VRID range: 1-255
- Priority range: 1-254 (255 is reserved for the acting Master)
- Virtual MACs: `00:00:5e:00:01:XX` (IPv4), `00:00:5e:00:02:XX` (IPv6) where XX = VRID in hex
- Kernel requirement: Linux 5.1+
- Requires: macvlan interfaces configured externally before VRRP activation

> **When to use VRRP:** Deploy VRRP when hosts on a LAN need a resilient default gateway and cannot run a dynamic routing protocol themselves. VRRP is simpler than running OSPF/BGP on every host. Do not use VRRP if hosts already participate in routing (e.g., containerized workloads with BGP).

## Enable the VRRP Daemon

Edit `/etc/frr/daemons`:

```
vrrpd=yes
```

Reload FRR:

```
sudo systemctl reload frr
```

## Macvlan Prerequisite

> **Critical:** VRRP in FRR requires macvlan sub-interfaces with the correct VRRP virtual MAC addresses. These must be created externally before FRR starts VRRP. FRR does not create macvlan devices.

### Create Macvlan Devices with iproute2

For VRID 10 on interface `eth0`:

```bash
# IPv4 macvlan (MAC 00:00:5e:00:01:0a for VRID 10)
ip link add vrrp4-10-eth0 link eth0 addrgenmode random type macvlan mode bridge
ip link set vrrp4-10-eth0 address 00:00:5e:00:01:0a
ip link set vrrp4-10-eth0 up

# IPv6 macvlan (MAC 00:00:5e:00:02:0a for VRID 10)
ip link add vrrp6-10-eth0 link eth0 addrgenmode random type macvlan mode bridge
ip link set vrrp6-10-eth0 address 00:00:5e:00:02:0a
ip link set vrrp6-10-eth0 up
```

### Create Macvlan Devices with ifupdown2

```
auto vrrp4-10-eth0
iface vrrp4-10-eth0
    vrrp-macvlan-device eth0
    hwaddress 00:00:5e:00:01:0a
    addrgenmode random

auto vrrp6-10-eth0
iface vrrp6-10-eth0
    vrrp-macvlan-device eth0
    hwaddress 00:00:5e:00:02:0a
    addrgenmode random
```

> **Warning:** The macvlan must be in `bridge` mode. The `addrgenmode random` setting is required for IPv6 to prevent SLAAC from generating an unwanted link-local address on the macvlan.

## Quick-Start: Basic VRRP

```
vtysh -c "configure terminal" -c "
interface eth0
 vrrp 10 version 3
 vrrp 10 ip 192.168.1.1
 vrrp 10 priority 200
"
```

This creates VRID 10 on `eth0` with virtual IP `192.168.1.1` and priority 200. The router with the highest priority becomes Master.

## Configuration Commands

### Create a VRRP Instance

```
interface eth0
 vrrp 10 version 3
```

Version 2 is available for legacy interop but supports IPv4 only:

```
interface eth0
 vrrp 10 version 2
```

### Assign Virtual IP Addresses

IPv4:

```
interface eth0
 vrrp 10 ip 192.168.1.1
 vrrp 10 ip 192.168.1.2
```

IPv6:

```
interface eth0
 vrrp 10 ipv6 fe80::1
 vrrp 10 ipv6 2001:db8::1
```

> **Note:** VRRPv3 supports both IPv4 and IPv6 on the same VRID, but each address family uses a separate macvlan with a distinct virtual MAC.

### Set Priority

```
interface eth0
 vrrp 10 priority 200
```

| Priority | Meaning |
|---|---|
| 255 | Reserved -- assigned automatically to the current Master |
| 200-254 | Typical primary router |
| 100 (default) | Standard priority |
| 1-99 | Lower priority, backup role |

### Configure Preemption

Preemption is enabled by default. A higher-priority Backup will take over from a lower-priority Master:

```
interface eth0
 vrrp 10 preempt
```

Disable preemption (lower-priority Master keeps the role until it fails):

```
interface eth0
 no vrrp 10 preempt
```

> **Warning:** Disabling preemption can cause the "wrong" router to remain Master after recovery. Only disable preemption when flapping is a concern and the network can tolerate suboptimal forwarding paths.

### Configure Advertisement Interval

```
interface eth0
 vrrp 10 advertisement-interval 1000
```

The interval is in **milliseconds** and must be a multiple of 10. Range: 10-40950.

| Interval | Typical use |
|---|---|
| 100 ms | Fast failover (sub-second) |
| 1000 ms (1s) | Default, suitable for most deployments |
| 5000+ ms | Slow convergence, reduces control traffic |

> **Note:** All VRRP routers for the same VRID must use the same advertisement interval. Mismatched intervals cause Master election instability.

### Administrative Shutdown

```
interface eth0
 vrrp 10 shutdown
```

### Checksum Interoperability

VRRPv3 checksum behavior varies across vendors. If interop issues occur:

```
interface eth0
 vrrp 10 checksum-with-ipv4-pseudoheader
```

This includes the IPv4 pseudoheader in the checksum (some vendors always do this; RFC 5798 is ambiguous).

## Global Defaults

Set defaults applied to all newly created VRRP instances:

```
vrrp default priority 150
vrrp default advertisement-interval 500
vrrp default preempt
```

## Autoconfigure

Automatically create VRRP instances from existing macvlan interfaces on the system:

```
vrrp autoconfigure [version 3]
```

This scans for macvlan devices with VRRP-pattern MAC addresses and creates matching VRRP routers. Useful when macvlan setup is handled by external orchestration.

## Full Configuration Example

### Dual-Stack VRRP with Two Routers

**Router A (primary):**

```bash
# Create macvlans (run before FRR starts)
ip link add vrrp4-10-eth0 link eth0 addrgenmode random type macvlan mode bridge
ip link set vrrp4-10-eth0 address 00:00:5e:00:01:0a
ip link set vrrp4-10-eth0 up

ip link add vrrp6-10-eth0 link eth0 addrgenmode random type macvlan mode bridge
ip link set vrrp6-10-eth0 address 00:00:5e:00:02:0a
ip link set vrrp6-10-eth0 up
```

```
! FRR configuration
interface eth0
 ip address 192.168.1.2/24
 ipv6 address 2001:db8::2/64
 vrrp 10 version 3
 vrrp 10 priority 200
 vrrp 10 ip 192.168.1.1
 vrrp 10 ipv6 fe80::1
 vrrp 10 ipv6 2001:db8::1
 vrrp 10 preempt
 vrrp 10 advertisement-interval 1000
```

**Router B (backup):**

```bash
# Create macvlans (same MACs -- they are per-VRID, not per-router)
ip link add vrrp4-10-eth0 link eth0 addrgenmode random type macvlan mode bridge
ip link set vrrp4-10-eth0 address 00:00:5e:00:01:0a
ip link set vrrp4-10-eth0 up

ip link add vrrp6-10-eth0 link eth0 addrgenmode random type macvlan mode bridge
ip link set vrrp6-10-eth0 address 00:00:5e:00:02:0a
ip link set vrrp6-10-eth0 up
```

```
! FRR configuration
interface eth0
 ip address 192.168.1.3/24
 ipv6 address 2001:db8::3/64
 vrrp 10 version 3
 vrrp 10 priority 100
 vrrp 10 ip 192.168.1.1
 vrrp 10 ipv6 fe80::1
 vrrp 10 ipv6 2001:db8::1
 vrrp 10 preempt
 vrrp 10 advertisement-interval 1000
```

Both routers share VRID 10 with virtual IP `192.168.1.1`. Router A (priority 200) becomes Master. If Router A fails, Router B (priority 100) takes over.

## Show Commands

| Command | Output |
|---|---|
| `show vrrp [json]` | All VRRP instances with state, priority, addresses |
| `show vrrp interface eth0 [json]` | VRRP instances on specific interface |
| `show vrrp 10 [json]` | Specific VRID status |
| `show vrrp interface eth0 10 [json]` | Specific VRID on specific interface |

### Example Show Output

```
router# show vrrp
 Virtual Router ID                    10
 Protocol Version                     3
 Autoconfigured                       No
 Shutdown                             No
 Interface                            eth0
 VRRP interface (v4)                  vrrp4-10-eth0
 VRRP interface (v6)                  vrrp6-10-eth0
 Primary IP (v4)                      192.168.1.2
 Primary IP (v6)                      fe80::a00:27ff:fe12:3456
 Virtual MAC (v4)                     00:00:5e:00:01:0a
 Virtual MAC (v6)                     00:00:5e:00:02:0a
 Status (v4)                          Master
 Status (v6)                          Master
 Priority                             200
 Effective Priority (v4)              200
 Effective Priority (v6)              200
 Preempt Mode                         Yes
 Accept Mode                          Yes
 Advertisement Interval               1000 ms
 Master Advertisement Interval (v4)   1000 ms
 Master Advertisement Interval (v6)   1000 ms
 Advertisements Tx (v4)               1542
 Advertisements Tx (v6)               1542
 Advertisements Rx (v4)               0
 Advertisements Rx (v6)               0
 IPv4 Addresses                       1
  192.168.1.1
 IPv6 Addresses                       2
  fe80::1
  2001:db8::1
```

## Debug Commands

```
debug vrrp [protocol | autoconfigure | packets | sockets | ndisc | arp | zebra]
```

| Keyword | Purpose |
|---|---|
| `protocol` | State machine transitions |
| `autoconfigure` | Autoconfiguration events |
| `packets` | Advertisement send/receive |
| `sockets` | Raw socket operations |
| `ndisc` | IPv6 Neighbor Discovery |
| `arp` | Gratuitous ARP activity |
| `zebra` | ZAPI communication |

## Troubleshooting

### Master Not Elected

| Check | Command / Action |
|---|---|
| Kernel version >= 5.1? | `uname -r` |
| Macvlan exists and is up? | `ip link show type macvlan` |
| Macvlan in bridge mode? | Recreate with `type macvlan mode bridge` |
| MAC address correct? | `ip link show vrrp4-10-eth0` -- verify `00:00:5e:00:01:XX` |
| Parent interface has IPv4? | Required for IPv4 VRRP -- `ip addr show eth0` |
| `addrgenmode random` set? | Required for IPv6 -- `ip -d link show vrrp6-10-eth0` |

### Advertisements Not Received

| Cause | Fix |
|---|---|
| Firewall blocking multicast | Allow VRRP: protocol 112, dst `224.0.0.18` (v2/v3) or `ff02::12` (v3) |
| Multicast snooping on bridge | `echo 0 > /sys/devices/virtual/net/<bridge>/bridge/multicast_snooping` |
| ESXi security policy | Enable Promiscuous Mode (pre-6.7) or MAC Learning (6.7+) |
| Interval mismatch | All routers for same VRID must use identical `advertisement-interval` |

### Traffic Not Forwarding Through Virtual IP

| Cause | Fix |
|---|---|
| Master not forwarding | Verify `net.ipv4.ip_forward=1` on Master |
| Backup attracting traffic | Set `net.ipv4.conf.eth0.ignore_routes_with_linkdown=1` |
| ARP not updating on failover | Master sends gratuitous ARPs -- check `debug vrrp arp` |
| Protodown not clearing | Check `ip -d link show` -- macvlan should not be protodown on Master |

### VRRPv3 Checksum Interop

If a multi-vendor environment shows peers not recognizing each other:

```
interface eth0
 vrrp 10 checksum-with-ipv4-pseudoheader
```

or

```
interface eth0
 no vrrp 10 checksum-with-ipv4-pseudoheader
```

Match the setting to what the peer vendor expects. RFC 5798 section 5.2.8 is ambiguous about IPv4 checksum calculation.

## Quick Reference: Verification Checklist

```bash
# 1. Macvlan devices exist and are up?
ip link show type macvlan

# 2. VRRP state and priority?
vtysh -c "show vrrp"

# 3. Master elected correctly?
vtysh -c "show vrrp" | grep Status

# 4. Virtual IP reachable from host?
ping 192.168.1.1

# 5. Debug advertisement flow
vtysh -c "debug vrrp packets"
```
