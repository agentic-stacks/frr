# FRR Daemons

## Overview

FRR runs one daemon per routing protocol plus core infrastructure daemons. Every daemon is an independent Unix process with its own event loop, memory space, and VTY interface. All protocol daemons communicate with zebra over Unix sockets using the zserv/zapi protocol.

Enable daemons in `/etc/frr/daemons` by setting them to `yes`, then reload FRR:

```
sudo systemctl reload frr
```

## Core Daemons

These daemons provide FRR's infrastructure. zebra and watchfrr should always be running. staticd is required for static route support.

### zebra

**Role:** Central routing manager, kernel interface, RIB/FIB controller.

zebra is the mandatory core daemon. All other daemons depend on it.

| Responsibility | Detail |
|---|---|
| RIB management | Stores all routes from all protocol daemons in per-VRF routing tables |
| Best-path selection | Selects the best route per prefix by administrative distance |
| FIB programming | Installs/removes routes in the Linux kernel via netlink |
| Interface management | Tracks interface state, IP addresses, link events from kernel |
| Route redistribution | Notifies protocol daemons when routes from other protocols change |
| Nexthop tracking | Monitors nexthop reachability and notifies registered daemons |
| Label management | Allocates MPLS labels to protocol daemons to avoid conflicts |
| VRF management | Manages virtual routing and forwarding instances |

**Startup options:**

```
# /etc/frr/daemons
zebra=yes
zebra_options=" -s 90000000 --daemon -A 127.0.0.1"
```

| Flag | Purpose |
|---|---|
| `-s <size>` | Netlink receive buffer size. Increase for large routing tables or high churn. Default is often too small for production. |
| `-r` | Retain routes in kernel when zebra stops (prevents traffic black-hole during restart) |
| `-e <N>` | Maximum ECMP paths (default: 64) |
| `-K <seconds>` | Graceful restart sweep timer |
| `-w` | Enable network namespace VRF backend |
| `-z <path>` | Custom zapi socket path |

**Key commands:**

```
show ip route [PREFIX] [json]
show ip route summary
show ip route vrf <NAME>
show ipv6 route
show interface [NAME]
show interface brief
show zebra client summary
show zebra dplane [detailed]
show nexthop-group rib
```

**VTY port:** 2601/tcp

### watchfrr

**Role:** Daemon supervisor and health monitor.

watchfrr monitors all other FRR daemons by connecting to their VTY Unix sockets and sending echo commands. If a daemon becomes unresponsive or crashes, watchfrr restarts it.

| Behavior | Detail |
|---|---|
| Health check | Polls each daemon's VTY socket at a configurable interval (default: 5s) |
| Failure detection | Declares daemon down after timeout (default: 10s) or on socket EOF |
| Restart | Executes configured restart command for the failed daemon |
| Backoff | Exponential backoff between restarts (60s min, 600s max) |
| Config writes | vtysh routes `write integrated` through watchfrr |

watchfrr starts automatically with FRR. You do not enable it in the daemons file.

**Key commands:**

```
! Show status of all monitored daemons
show watchfrr

! Exclude a daemon from monitoring (useful during debugging)
watchfrr ignore <daemon>
```

**Startup options:**

| Flag | Purpose |
|---|---|
| `-i <seconds>` | Polling interval (default: 5) |
| `-t <seconds>` | Unresponsive timeout (default: 10) |
| `--min-restart-interval <seconds>` | Minimum delay between restarts (default: 60) |
| `--max-restart-interval <seconds>` | Maximum delay between restarts (default: 600) |
| `-T <seconds>` | Kill timeout for stuck processes (default: 20) |
| `-S <dir>` | VTY socket directory (default: `/var/run/frr`) |
| `--dry` | Monitor only, do not restart (for testing) |

### staticd

**Role:** Static route management.

staticd handles all static route configuration. It runs as a separate daemon (not built into zebra) so that static routes use the same zapi interface as protocol-learned routes.

```
# /etc/frr/daemons
staticd=yes
```

**Configuration example:**

```
ip route 10.0.0.0/8 192.168.1.1
ip route 0.0.0.0/0 192.168.1.1 200
ipv6 route ::/0 fd00::1
ip route 10.10.0.0/16 eth0
ip route 10.20.0.0/16 Null0
```

**VTY port:** 2616/tcp

## Protocol Daemons

Each protocol daemon implements a single routing protocol. Enable only the daemons you need.

### bgpd -- Border Gateway Protocol

The most feature-rich FRR daemon. Supports BGP-4 (RFC 4271), MP-BGP, VPNv4/v6, EVPN, flowspec, and more.

```
# /etc/frr/daemons
bgpd=yes
bgpd_options="  --daemon -A 127.0.0.1"
```

**VTY port:** 2605/tcp

**Use cases:** Internet peering, data center fabrics (EVPN-VXLAN), WAN routing, route policy.

### ospfd -- OSPF v2 (IPv4)

Link-state IGP for IPv4 networks. Implements OSPF version 2 (RFC 2328).

```
# /etc/frr/daemons
ospfd=yes
ospfd_options="  --daemon -A 127.0.0.1"
```

**VTY port:** 2604/tcp

**Use cases:** Enterprise campus networks, IPv4 IGP underlay.

### ospf6d -- OSPF v3 (IPv6)

Link-state IGP for IPv6 networks. Implements OSPF version 3 (RFC 5340).

```
# /etc/frr/daemons
ospf6d=yes
```

**VTY port:** 2606/tcp

**Use cases:** IPv6 IGP, dual-stack networks.

### isisd -- IS-IS

Link-state IGP supporting both IPv4 and IPv6 in a single protocol instance. Implements ISO 10589 with RFC 5308 extensions.

```
# /etc/frr/daemons
isisd=yes
```

**VTY port:** 2608/tcp

**Use cases:** Service provider networks, data center underlay, segment routing.

### ripd -- RIP v2 (IPv4)

Distance-vector IGP for IPv4. Implements RIP version 2 (RFC 2453).

```
# /etc/frr/daemons
ripd=yes
```

**VTY port:** 2602/tcp

**Use cases:** Simple/legacy networks, lab environments.

### ripngd -- RIPng (IPv6)

Distance-vector IGP for IPv6. Implements RIPng (RFC 2080).

```
# /etc/frr/daemons
ripngd=yes
```

**VTY port:** 2603/tcp

**Use cases:** Simple IPv6 networks, lab environments.

### pimd -- PIM (IPv4 Multicast)

Protocol Independent Multicast for IPv4. Supports PIM-SM, PIM-SSM, and IGMP.

```
# /etc/frr/daemons
pimd=yes
```

**VTY port:** 2611/tcp

**Use cases:** IPv4 multicast routing, IPTV, video distribution.

### pim6d -- PIM (IPv6 Multicast)

Protocol Independent Multicast for IPv6. Supports PIMv6 and MLD.

```
# /etc/frr/daemons
pim6d=yes
```

**Use cases:** IPv6 multicast routing.

### ldpd -- LDP (MPLS)

Label Distribution Protocol for MPLS label-switched paths. Implements RFC 5036.

```
# /etc/frr/daemons
ldpd=yes
```

**VTY port:** 2612/tcp

**Use cases:** MPLS L2/L3 VPNs, traffic engineering, service provider networks.

### bfdd -- Bidirectional Forwarding Detection

Sub-second failure detection for routing protocol adjacencies. Implements BFD (RFC 5880).

```
# /etc/frr/daemons
bfdd=yes
```

**VTY port:** 2617/tcp

**Use cases:** Fast failure detection for BGP, OSPF, IS-IS sessions.

### babeld -- Babel

Distance-vector routing protocol designed for wireless mesh networks. Implements RFC 8966.

```
# /etc/frr/daemons
babeld=yes
```

**VTY port:** 2609/tcp

**Use cases:** Mesh networks, overlay networks, networks with frequently changing topology.

### pbrd -- Policy-Based Routing

Enables routing decisions based on criteria beyond destination address (source address, DSCP, etc.).

```
# /etc/frr/daemons
pbrd=yes
```

**VTY port:** 2615/tcp

**Use cases:** Source-based routing, traffic steering, service chaining.

### fabricd -- OpenFabric

IS-IS variant optimized for data center fabric topologies.

```
# /etc/frr/daemons
fabricd=yes
```

**VTY port:** 2618/tcp

**Use cases:** Data center spine-leaf fabrics.

### vrrpd -- Virtual Router Redundancy Protocol

First-hop redundancy with virtual IP failover. Implements VRRP v2/v3 (RFC 5798).

```
# /etc/frr/daemons
vrrpd=yes
```

**VTY port:** 2619/tcp

**Use cases:** Gateway redundancy, high availability.

### eigrpd -- EIGRP

Enhanced Interior Gateway Routing Protocol (Cisco-compatible).

```
# /etc/frr/daemons
eigrpd=yes
```

**VTY port:** 2613/tcp

**Use cases:** Cisco interoperability, migration scenarios.

### nhrpd -- NHRP

Next Hop Resolution Protocol for DMVPN (Dynamic Multipoint VPN) deployments. Implements RFC 2332.

```
# /etc/frr/daemons
nhrpd=yes
```

**VTY port:** 2614/tcp

**Use cases:** DMVPN hub-and-spoke and spoke-to-spoke tunnels.

### pathd -- Segment Routing Path Computation

Manages Segment Routing (SR) path computation, including SR-TE policies and PCEP (Path Computation Element Protocol).

```
# /etc/frr/daemons
pathd=yes
```

**Use cases:** Segment routing traffic engineering, PCEP-based path computation.

### sharpd -- Super Happy Advanced Routing Process (Utility)

A developer/testing daemon for exercising zebra's internal APIs. Not for production use.

```
# /etc/frr/daemons
sharpd=yes
```

**Use cases:** FRR development, stress testing zebra, validating zapi behavior.

## Daemon Lifecycle

### Start All Daemons

```
sudo systemctl start frr
```

This starts zebra, watchfrr, and every daemon enabled in `/etc/frr/daemons`.

### Stop All Daemons

**WARNING:** Stopping FRR drops all protocol sessions and removes FRR-installed routes from the kernel (unless zebra was started with `-r`).

```
sudo systemctl stop frr
```

### Restart All Daemons

**WARNING:** This is disruptive. All BGP sessions drop, OSPF adjacencies reset, and the network reconverges.

```
sudo systemctl restart frr
```

### Reload Configuration (Non-Disruptive)

```
sudo systemctl reload frr
```

This uses `frr-reload.py` to compute and apply only the configuration delta. It also starts any newly enabled daemons.

### Enable a New Daemon

1. Edit `/etc/frr/daemons` and set the daemon to `yes`
2. Add any protocol configuration to `/etc/frr/frr.conf`
3. Reload:

```
sudo systemctl reload frr
```

### Disable a Daemon

1. Remove the daemon's configuration from `frr.conf` (or the per-daemon config file)
2. Set the daemon to `no` in `/etc/frr/daemons`
3. Restart FRR (reload alone cannot stop a running daemon):

```
sudo systemctl restart frr
```

### Check Daemon Status

```
sudo systemctl status frr
```

Or from within vtysh:

```
show watchfrr
```

### Graceful Restart

Some protocols support graceful restart, which preserves forwarding state during a daemon restart. This is configured per-protocol (e.g., BGP graceful-restart). Zebra supports a graceful restart sweep timer:

```
# In daemons file:
zebra_options=" -K 30 --daemon -A 127.0.0.1"
```

The `-K 30` flag tells zebra to sweep stale routes 30 seconds after restart.

## Command-Line Options

All FRR daemons share a common set of command-line options inherited from the FRR library:

| Flag | Purpose |
|---|---|
| `--daemon` or `-d` | Fork into background |
| `-f <file>` | Configuration file path |
| `-i <file>` | PID file path |
| `-z <path>` | zapi socket path (default: `/var/run/frr/zserv.api`) |
| `-A <address>` | VTY bind address (use `127.0.0.1` for security) |
| `-P <port>` | VTY TCP port (use `0` to disable TCP VTY) |
| `-u <user>` | Run as user (default: `frr`) |
| `-g <group>` | Run as group (default: `frr`) |
| `-N <pathspace>` | Path namespace (for multiple FRR instances on one host) |
| `-o <name>` | Override default VRF name (zebra only) |
| `--log <target>` | Log destination (stdout, syslog, file) |
| `--log-level <level>` | Log level: emergencies, alerts, critical, errors, warnings, notifications, informational, debugging |
| `-v` or `--version` | Show version and exit |
| `-h` or `--help` | Show usage and exit |

These flags are typically set in the `_options` variables in `/etc/frr/daemons`:

```
# /etc/frr/daemons
zebra_options=" -s 90000000 --daemon -A 127.0.0.1"
bgpd_options="  --daemon -A 127.0.0.1"
ospfd_options=" --daemon -A 127.0.0.1"
```

## Daemon-to-Zebra Registration

When a protocol daemon starts, it performs this registration sequence with zebra:

```
1. Daemon opens Unix socket connection to zebra
   (connects to /var/run/frr/zserv.api)
        |
        v
2. Daemon sends HELLO message
   (identifies itself: daemon type, instance, capabilities)
        |
        v
3. Daemon requests interface information
   (INTERFACE_ADD: zebra sends current interface state)
        |
        v
4. Daemon requests router ID
   (ROUTER_ID_ADD: zebra sends the current router ID)
        |
        v
5. Daemon registers for route redistribution (if needed)
   (REDISTRIBUTE_ADD: zebra sends existing routes of requested types)
        |
        v
6. Daemon registers nexthop tracking (if needed)
   (NEXTHOP_REGISTER: zebra will notify on reachability changes)
        |
        v
7. Daemon is operational
   (begins installing/withdrawing routes via ROUTE_ADD/ROUTE_DELETE)
```

### Verify Registration

Check which daemons are connected to zebra:

```
vtysh -c "show zebra client summary"
```

This shows each connected daemon, its socket FD, the number of routes it has installed, and message counters.

### VTY Port Reference

| Daemon | Default VTY Port |
|---|---|
| zebra | 2601 |
| ripd | 2602 |
| ripngd | 2603 |
| ospfd | 2604 |
| bgpd | 2605 |
| ospf6d | 2606 |
| isisd | 2608 |
| babeld | 2609 |
| pimd | 2611 |
| ldpd | 2612 |
| eigrpd | 2613 |
| nhrpd | 2614 |
| pbrd | 2615 |
| staticd | 2616 |
| bfdd | 2617 |
| fabricd | 2618 |
| vrrpd | 2619 |
