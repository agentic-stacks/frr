# LDP Protocol Skill

## Overview

LDP (Label Distribution Protocol) is the MPLS label signaling protocol in FRR, implemented by the `ldpd` daemon. LDP automatically distributes labels for IGP-learned prefixes, enabling MPLS forwarding across the network. It implements RFC 5036 along with related standards (RFC 6720, 6667, 5919, 5561, 7552, 4447, 3031).

Key facts:

- Daemon: `ldpd` (enable in `/etc/frr/daemons`)
- Transport: TCP port 646, UDP port 646 (discovery)
- Multicast: 224.0.0.2 (All Routers) for link-local discovery
- Default hello: interval 5s, holdtime 15s
- Label retention: Liberal (stores all received labels)
- Label distribution: Downstream Unsolicited
- Label control: Independent (default), Ordered (optional)
- Address families: IPv4 and IPv6 (dual-stack per RFC 7552)
- Requires: kernel MPLS support, zebra running first

> **When to use LDP:** LDP is the standard choice for distributing labels in MPLS core networks. Use LDP when you need basic MPLS transport for L3VPN (with MP-BGP), L2VPN/VPLS pseudowires, or traffic engineering fallback paths. For strict TE tunnel signaling, use RSVP-TE instead (not in FRR). For simpler segment routing deployments, consider IS-IS or OSPF SR extensions.

## Kernel MPLS Prerequisites

LDP requires kernel MPLS support. On Linux, load the MPLS modules and enable forwarding:

```bash
# Load required kernel modules
sudo modprobe mpls_router
sudo modprobe mpls_iptunnel

# Enable MPLS forwarding
sudo sysctl -w net.mpls.conf.lo.input=1
sudo sysctl -w net.mpls.platform_labels=1048575

# Enable MPLS on each LDP-facing interface
sudo sysctl -w net.mpls.conf.eth0.input=1
sudo sysctl -w net.mpls.conf.eth1.input=1
```

Make persistent in `/etc/sysctl.d/90-mpls.conf`:

```
net.mpls.conf.lo.input = 1
net.mpls.platform_labels = 1048575
net.mpls.conf.eth0.input = 1
net.mpls.conf.eth1.input = 1
```

And in `/etc/modules-load.d/mpls.conf`:

```
mpls_router
mpls_iptunnel
```

> **Warning:** If `net.mpls.platform_labels` is 0 (the default), the kernel cannot install any MPLS forwarding entries. Set it to at least 1048575 before starting ldpd.

## Enable the LDP Daemon

Edit `/etc/frr/daemons`:

```
ldpd=yes
```

Reload FRR:

```
sudo systemctl reload frr
```

**Prerequisite:** The zebra daemon must be running before ldpd starts.

### ldpd Startup Options

| Flag | Purpose |
|---|---|
| `--ctl_socket` | Override default ldpd.sock control socket path |

## Quick-Start: Basic LDP on a P Router

```
vtysh -c "configure terminal" -c "
mpls ldp
 router-id 10.255.0.1
 address-family ipv4
  discovery transport-address 10.255.0.1
  interface eth0
  interface eth1
 exit-address-family
"
```

Replace:
- `10.255.0.1` -- your MPLS router-id (typically a loopback IP reachable via IGP)
- `eth0`, `eth1` -- interfaces facing other LDP routers

**Note:** The `discovery transport-address` should be your loopback address, which must be reachable via IGP (OSPF/IS-IS). LDP establishes TCP sessions between transport addresses, not link addresses.

## Router-ID and Basic Settings

```
mpls ldp
 router-id 10.255.0.1
```

### Ordered Control

By default, LDP uses Independent Label Distribution Control (labels are assigned immediately). To require a downstream label before assigning a local label:

```
mpls ldp
 ordered-control
```

Use ordered control when strict label path consistency is required (e.g., MPLS-TE environments).

## Address Family Configuration

### IPv4

```
mpls ldp
 address-family ipv4
  discovery transport-address 10.255.0.1
  interface eth0
  interface eth1
 exit-address-family
```

### IPv6

```
mpls ldp
 address-family ipv6
  discovery transport-address 2001:db8::1
  interface eth0
  interface eth1
 exit-address-family
```

### Dual-Stack

```
mpls ldp
 dual-stack transport-connection prefer ipv4
 address-family ipv4
  discovery transport-address 10.255.0.1
  interface eth0
 exit-address-family
 address-family ipv6
  discovery transport-address 2001:db8::1
  interface eth0
 exit-address-family
```

By default, dual-stack prefers IPv6 per RFC 7552. Use `dual-stack transport-connection prefer ipv4` to override.

## Discovery Timers

```
mpls ldp
 discovery hello holdtime 15
 discovery hello interval 5
```

| Parameter | Default | Range |
|---|---|---|
| Hello interval | 5s | 1-65535s |
| Hello holdtime | 15s | 1-65535s |

## Neighbor Configuration

### MD5 Authentication

```
mpls ldp
 neighbor 10.255.0.2 password SECRET123
```

> **Warning:** LDP MD5 passwords are stored in plain text in the configuration file. Restrict read access to `/etc/frr/frr.conf`.

### Session Holdtime

```
mpls ldp
 neighbor 10.255.0.2 session holdtime 30
```

Range: 15-65535 seconds. The lower holdtime between two neighbors is used.

### TTL Security (GTSM)

GTSM (Generalized TTL Security Mechanism) is enabled by default. To disable per neighbor:

```
mpls ldp
 neighbor 10.255.0.2 ttl-security disable
```

Or set an expected hop count:

```
mpls ldp
 neighbor 10.255.0.2 ttl-security hops 2
```

## Targeted Sessions

Targeted LDP sessions enable label distribution between non-adjacent routers (e.g., for pseudowires or remote LFA).

```
mpls ldp
 address-family ipv4
  neighbor 10.255.0.5 targeted
 exit-address-family
```

Targeted hello timers are separate from link discovery timers:

```
mpls ldp
 discovery targeted-hello holdtime 90
 discovery targeted-hello interval 10
```

## Interface-Level Settings

### Disable GTSM on an Interface

```
mpls ldp
 address-family ipv4
  interface eth0
   ttl-security disable
  exit
 exit-address-family
```

### Disable Extra Hello on Session Establishment

```
mpls ldp
 address-family ipv4
  interface eth0
   disable-establish-hello
  exit
 exit-address-family
```

## LDP-IGP Synchronization

LDP-IGP sync prevents traffic blackholing during LDP convergence by raising the IGP metric on interfaces where LDP is not yet converged.

### With OSPF

```
router ospf
 mpls ldp-sync
```

Or per interface:

```
interface eth0
 ip ospf mpls ldp-sync
```

### With IS-IS

```
router isis CORE
 mpls ldp-sync
```

## L2VPN Pseudowires

LDP can signal pseudowires for L2VPN/VPLS services:

```
l2vpn CUSTOMER-A type vpls
 member pseudowire mpw0
  neighbor lsr-id 10.255.0.2
  pw-id 100
```

- `mpw0` -- the pseudowire interface (create with `ip link add mpw0 type mpwtun`)
- `neighbor lsr-id` -- the remote PE router-id
- `pw-id` -- must match on both ends

## Show Commands

### Show Neighbors

```
show mpls ldp neighbor
show mpls ldp neighbor 10.255.0.2
show mpls ldp neighbor 10.255.0.2 detail
show mpls ldp neighbor 10.255.0.2 capabilities
```

### Show Discovery

```
show mpls ldp ipv4 discovery
show mpls ldp ipv4 discovery detail
show mpls ldp ipv6 discovery
```

### Show Interfaces

```
show mpls ldp ipv4 interface
show mpls ldp ipv6 interface
```

### Show Label Bindings

```
show mpls ldp ipv4 binding
show mpls ldp ipv6 binding
```

Shows local and remote label mappings. This is the primary command for verifying MPLS label distribution.

### Show MPLS Table (Zebra)

```
show mpls table
```

Shows the kernel MPLS forwarding entries installed by zebra.

## Debug Commands

| Command | What It Shows |
|---|---|
| `debug mpls ldp discovery` | Hello packet send/receive events |
| `debug mpls ldp errors` | Protocol errors |
| `debug mpls ldp event` | Session and state machine events |
| `debug mpls ldp labels` | Label allocation and distribution |
| `debug mpls ldp messages` | LDP protocol messages (recv/sent) |
| `debug mpls ldp zebra` | Zebra interaction (route/label installs) |

## Complete Configuration Example

MPLS core with two P routers running LDP over OSPF:

**Router P1 (10.255.0.1):**

```
! /etc/frr/frr.conf -- Router P1
!
interface lo
 ip address 10.255.0.1/32
!
interface eth0
 ip address 10.0.12.1/30
 ip ospf area 0
 ip ospf network point-to-point
!
interface eth1
 ip address 10.0.13.1/30
 ip ospf area 0
 ip ospf network point-to-point
!
router ospf
 ospf router-id 10.255.0.1
 network 10.255.0.1/32 area 0
 mpls ldp-sync
!
mpls ldp
 router-id 10.255.0.1
 neighbor 10.255.0.2 password MPLS-SECRET
 neighbor 10.255.0.3 password MPLS-SECRET
 address-family ipv4
  discovery transport-address 10.255.0.1
  interface eth0
  interface eth1
 exit-address-family
```

**Router P2 (10.255.0.2):**

```
! /etc/frr/frr.conf -- Router P2
!
interface lo
 ip address 10.255.0.2/32
!
interface eth0
 ip address 10.0.12.2/30
 ip ospf area 0
 ip ospf network point-to-point
!
router ospf
 ospf router-id 10.255.0.2
 network 10.255.0.2/32 area 0
 mpls ldp-sync
!
mpls ldp
 router-id 10.255.0.2
 neighbor 10.255.0.1 password MPLS-SECRET
 address-family ipv4
  discovery transport-address 10.255.0.2
  interface eth0
 exit-address-family
```

## Verify MPLS Forwarding

After LDP converges, verify end-to-end:

```
! Check LDP neighbors are up
show mpls ldp neighbor

! Check labels are exchanged
show mpls ldp ipv4 binding

! Check kernel MPLS table
show mpls table

! Trace the label-switched path
traceroute mpls ipv4 10.255.0.2/32
```
