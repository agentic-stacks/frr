# BFD Protocol Skill

## Overview

BFD (Bidirectional Forwarding Detection) is a lightweight protocol for rapid failure detection in FRR, implemented by the `bfdd` daemon. BFD operates independently from routing protocols and notifies them when a forwarding path fails, enabling sub-second convergence that would otherwise take seconds or minutes with protocol-native keepalives.

Key facts:

- Daemon: `bfdd` (enable in `/etc/frr/daemons`)
- Transport: UDP port 3784 (single-hop), UDP port 4784 (multi-hop)
- Default timers: transmit 300ms, receive 300ms, detect-multiplier 3
- Detection time: multiplier x negotiated interval (default: 900ms)
- Supports: single-hop, multi-hop, echo mode, profiles, VRF
- Integrates with: BGP, OSPF, OSPFv3, IS-IS, PIM, RIP, static routes

> **When to use BFD:** Add BFD whenever routing protocol keepalive timers are too slow for your convergence requirements. BFD is essential on links where the physical layer does not signal failures (e.g., switched Ethernet with intermediate L2 devices, ECMP bundles). Do not use BFD if the link already provides fast failure signaling (e.g., direct fiber with LOS detection).

## Enable the BFD Daemon

Edit `/etc/frr/daemons`:

```
bfdd=yes
```

Reload FRR:

```
sudo systemctl reload frr
```

### bfdd Startup Options

Set in `/etc/frr/daemons` via `bfdd_options`:

| Flag | Purpose |
|---|---|
| `--dplaneaddr <type>:<address>[:<port>]` | Distributed BFD data plane socket (UNIX, IPv4, IPv6) |
| `--vrfs <vrf-list>` | Monitor specific VRFs (default: all including default) |

## Quick-Start: BFD with BGP

```
vtysh -c "configure terminal" -c "
bfd
 peer 10.0.0.2
  no shutdown
 !
!
router bgp 65001
 neighbor 10.0.0.2 remote-as 65002
 neighbor 10.0.0.2 bfd
"
```

This creates a single-hop BFD session to 10.0.0.2 and tells BGP to tear down the neighbor immediately when BFD detects failure.

## Peer Configuration

### Create a Single-Hop Peer

```
bfd
 peer 10.0.0.2
  detect-multiplier 3
  receive-interval 300
  transmit-interval 300
  no shutdown
```

Single-hop peers monitor directly connected neighbors. BFD validates that the TTL is 255 (ensuring the packet traversed exactly one hop).

### Create a Multi-Hop Peer

```
bfd
 peer 10.1.1.1 multihop local-address 10.0.0.1
  detect-multiplier 3
  receive-interval 300
  transmit-interval 300
  no shutdown
```

Multi-hop peers listen on UDP port 4784 and accept packets with TTL less than 254. The `local-address` is mandatory for multi-hop sessions.

> **Warning:** Echo mode is incompatible with multi-hop sessions per RFC 5883.

### Create an Interface-Bound Peer

```
bfd
 peer 10.0.0.2 interface eth0
  no shutdown
```

Bind a session to a specific interface. Required when the same peer address is reachable on multiple interfaces.

### Create a VRF-Aware Peer

```
bfd
 peer 10.0.0.2 vrf CUSTOMER-A
  no shutdown
```

### IPv6 Peer

```
bfd
 peer 2001:db8::2 local-address 2001:db8::1 interface eth0
  no shutdown
```

`local-address` is mandatory for IPv6 BFD sessions.

## Session Timers

| Parameter | Command | Default | Range |
|---|---|---|---|
| Detect multiplier | `detect-multiplier (1-255)` | 3 | 1-255 |
| Receive interval | `receive-interval (10-4294967)` | 300ms | 10ms-4294967ms |
| Transmit interval | `transmit-interval (10-4294967)` | 300ms | 10ms-4294967ms |
| Echo receive interval | `echo receive-interval (10-4294967)` | 50ms | 10ms-4294967ms |
| Echo transmit interval | `echo transmit-interval (10-4294967)` | 50ms | 10ms-4294967ms |

**Detection time calculation:** If the local system has `detect-multiplier 3` with `transmit-interval 300` and the remote system has `receive-interval 200`, the remote system detects failure after 3 x 300 = 900ms.

**Aggressive timers example (50ms detection):**

```
bfd
 peer 10.0.0.2
  detect-multiplier 5
  receive-interval 10
  transmit-interval 10
  no shutdown
```

> **Warning:** Sub-100ms timers place significant CPU load on the router. Test thoroughly before deploying aggressive timers in production. Use BFD profiles to standardize timer values across the network.

## Profiles

Profiles define reusable BFD parameters that can be applied to multiple peers. Peer-level settings override profile settings.

### Define a Profile

```
bfd
 profile FAST-DETECT
  detect-multiplier 3
  receive-interval 150
  transmit-interval 150
 !
 profile SLOW-DETECT
  detect-multiplier 5
  receive-interval 1000
  transmit-interval 1000
```

### Apply a Profile to a Peer

```
bfd
 peer 10.0.0.2
  profile FAST-DETECT
  no shutdown
```

### Apply a Profile via Routing Protocol

```
router bgp 65001
 neighbor 10.0.0.2 bfd profile FAST-DETECT
```

```
interface eth0
 ip ospf bfd profile FAST-DETECT
```

If a profile is deleted, all peers using it revert to default timer values.

## Echo Mode

Echo mode sends packets that the remote system loops back without processing, allowing the local system to measure the entire forwarding path round-trip.

```
bfd
 peer 10.0.0.2
  echo-mode
  echo transmit-interval 50
  transmit-interval 2000
  no shutdown
```

**Best practice:** When enabling echo mode, increase the control-plane `transmit-interval` (e.g., 2000ms) to reduce CPU overhead. The echo packets handle fast detection; the control-plane packets only maintain the session.

Constraints:
- Incompatible with multi-hop sessions
- Both peers must support echo mode for bidirectional echo
- RTT measurements available for IPv4 echo on Linux

To disable echo reception:

```
bfd
 peer 10.0.0.2
  echo receive-interval disabled
```

## Session Control

| Command | Effect |
|---|---|
| `shutdown` | Administratively disable peer; sends AdminDown diagnostic |
| `no shutdown` | Enable peer (peers start in shutdown state) |
| `passive-mode` | Wait for remote to initiate; do not send control packets first |
| `minimum-ttl (1-254)` | Multi-hop: minimum acceptable TTL (default 254) |

`passive-mode` is useful on hub routers in hub-spoke topologies -- the hub waits for spokes to initiate BFD sessions.

## Integration with Routing Protocols

### BGP

```
router bgp 65001
 neighbor 10.0.0.2 remote-as 65002
 neighbor 10.0.0.2 bfd
```

Optional strict mode -- prevents BGP from establishing until BFD is up:

```
 neighbor 10.0.0.2 bfd strict
```

With hold-time (delay BGP teardown after BFD goes down):

```
 neighbor 10.0.0.2 bfd strict hold-time 30
```

Check-control-plane-failure (for graceful restart scenarios):

```
 neighbor 10.0.0.2 bfd check-control-plane-failure
```

### OSPF

```
interface eth0
 ip ospf bfd
```

Or with a profile:

```
interface eth0
 ip ospf bfd profile FAST-DETECT
```

OSPF creates one BFD peer per discovered neighbor on the interface.

### OSPFv3

```
interface eth0
 ipv6 ospf6 bfd
```

### IS-IS

```
interface eth0
 isis bfd
```

IS-IS creates a single BFD session per interface using IPv6.

### PIM

```
interface eth0
 ip pim bfd
```

### RIP

```
interface eth0
 ip rip bfd
```

Or set a global default profile for all RIP BFD sessions:

```
router rip
 bfd default-profile FAST-DETECT
```

### Static Routes

Monitor a static route with BFD -- the route installs only when BFD is up:

```
ip route 192.168.10.0/24 10.0.0.2 bfd
```

Multi-hop static route monitoring:

```
ip route 192.168.10.0/24 10.0.0.2 bfd multi-hop source 10.0.0.1
```

With profile:

```
ip route 192.168.10.0/24 10.0.0.2 bfd profile FAST-DETECT
```

## Show and Debug Commands

### Show Peers

```
show bfd peers
show bfd peers brief
show bfd peer 10.0.0.2
show bfd vrf CUSTOMER-A peers
```

**Sample output fields:** Session ID, Remote ID, Status (up/down), Uptime, Diagnostics, Peer Type (configured/dynamic), Local/Remote timers, RTT measurements.

### Show Counters

```
show bfd peers counters
show bfd peer 10.0.0.2 counters
```

Shows control/echo packet counts, session state events, and Zebra notifications.

### Clear Counters

```
clear bfd peers counters
```

Resets packet counters but preserves session event counts.

### Show Static Route BFD Status

```
show bfd static route
```

### Show Distributed BFD Status

```
show bfd distributed
```

### Debug Commands

| Command | What It Shows |
|---|---|
| `debug bfd peer` | Peer creation/removal, state transitions |
| `debug bfd network` | Socket failures, unexpected messages |
| `debug bfd zebra` | Interface, address, VRF, registration events |
| `debug bfd distributed` | Data plane messaging and connection events |

Enable logging:

```
configure terminal
 log file /var/log/frr/frr.log debugging
```

### JSON Output

All show commands support `json` suffix:

```
show bfd peers json
show bfd peer 10.0.0.2 counters json
```

## Complete Configuration Example

Hub router with BFD profiles and multiple protocol integrations:

```
! /etc/frr/frr.conf
!
bfd
 profile FAST
  detect-multiplier 3
  receive-interval 150
  transmit-interval 150
 !
 profile WAN
  detect-multiplier 5
  receive-interval 500
  transmit-interval 500
 !
 ! Explicitly configured peer for static route monitoring
 peer 10.99.0.1
  profile WAN
  no shutdown
 !
!
router bgp 65001
 neighbor 10.0.0.2 remote-as 65002
 neighbor 10.0.0.2 bfd profile FAST
 neighbor 10.0.1.2 remote-as 65003
 neighbor 10.0.1.2 bfd strict
!
router ospf
 ospf router-id 10.255.0.1
!
interface eth0
 ip ospf area 0
 ip ospf bfd profile FAST
!
interface eth1
 isis bfd profile FAST
 isis network point-to-point
!
! Static route conditional on BFD
ip route 172.16.0.0/16 10.99.0.1 bfd profile WAN
```

## Distributed BFD

Distributed BFD separates the control plane (bfdd) from the data plane (vendor/custom hardware). The data plane handles packet TX/RX, while bfdd manages session configuration and distributes state to routing protocols.

Start bfdd with the data plane socket:

```
bfdd --dplaneaddr unix:/var/run/bfdd_dplane.sock
```

Monitor with:

```
show bfd distributed
```

This is an advanced feature for hardware-offloaded BFD implementations.
