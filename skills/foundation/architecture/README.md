# FRR Architecture

## Overview

FRR (Free Range Routing) is a routing suite that runs as multiple cooperating daemons on a single Linux (or BSD) host. Each routing protocol runs in its own daemon process. A central daemon called **zebra** aggregates routes from all protocol daemons, selects the best route by administrative distance, and programs the kernel forwarding information base (FIB). A supervisor daemon called **watchfrr** monitors all other daemons and restarts them on failure.

Key architectural facts:

- One daemon per routing protocol (bgpd, ospfd, isisd, etc.)
- Zebra is the single point of contact with the kernel routing table
- All daemons communicate with zebra over Unix domain sockets using the zserv/zapi protocol
- A unified CLI called vtysh connects to all daemons simultaneously over Unix sockets
- watchfrr monitors daemon health by polling VTY sockets and restarts failed daemons

## The Daemon Model

Every routing protocol runs as an independent Unix process. This provides:

| Benefit | Detail |
|---|---|
| Fault isolation | A crash in ospfd does not affect bgpd |
| Independent restart | Restart one protocol without disrupting others |
| Memory isolation | Each daemon has its own address space |
| Privilege separation | Daemons can drop privileges independently |

Each daemon runs an internal event loop (defined in `lib/event.c`). The event loop handles:

- **EVENT_READ / EVENT_WRITE** -- I/O on file descriptors (peer sockets, zapi socket, VTY socket)
- **EVENT_TIMER** -- periodic tasks (keepalives, SPF throttle timers, route aging)
- **EVENT_EVENT** -- high-priority internal events

Some daemons use kernel threads (pthreads) for performance. For example, bgpd runs separate I/O and keepalive threads, each with its own event loop. Cross-thread communication uses the event scheduling API, which is thread-safe.

### Daemon Startup

All daemons share common startup flags inherited from the FRR framework:

```
<daemon> [--daemon] [-A <vty-address>] [-P <vty-port>] \
         [-f <config-file>] [-i <pid-file>] [-z <zapi-socket>] \
         [-u <user>] [-g <group>] [-N <pathspace>]
```

- `--daemon` -- fork into background
- `-A 127.0.0.1` -- bind VTY listener to localhost only
- `-P <port>` -- VTY TCP port (0 disables TCP VTY)
- `-z <path>` -- path to zebra's zapi socket (default: `/var/run/frr/zserv.api`)
- `-N <pathspace>` -- namespace isolation for running multiple FRR instances

## Zebra -- Central Routing Manager

Zebra is the heart of FRR. It has four core responsibilities:

1. **RIB management** -- maintains the Routing Information Base, which holds all routes from all protocol daemons
2. **Best-path selection** -- selects the best route to each destination based on administrative distance
3. **FIB programming** -- installs selected routes into the kernel routing table via netlink (Linux) or routing sockets (BSD)
4. **Interface management** -- tracks interface state, addresses, and link events from the kernel

### Administrative Distance Table

Zebra uses administrative distance to choose between routes from different protocols to the same prefix:

| Source | Administrative Distance |
|---|---|
| Directly connected | 0 |
| Kernel routes | 0 |
| Static routes | 1 |
| EBGP | 20 |
| OSPF | 110 |
| IS-IS / OpenFabric | 115 |
| RIP | 120 |
| IBGP | 200 |

Lower distance wins. Distance 255 means "do not install in FIB" (the route stays in the RIB for redistribution but is not forwarded).

### Zebra-Specific Startup Options

```
zebra [--daemon] [-A 127.0.0.1] [-s <netlink-buffer-size>] \
      [-r] [-e <ecmp-max>] [-K <graceful-restart-time>] [-w]
```

- `-s 90000000` -- increase netlink receive buffer (critical for large routing tables)
- `-r` -- retain routes in kernel on daemon shutdown
- `-e <N>` -- limit ECMP paths (default: 64)
- `-K <seconds>` -- graceful restart sweep timer
- `-w` -- use network namespace VRF backend

## Inter-Daemon Communication (zserv/zapi)

Protocol daemons communicate with zebra using the **zserv** (also called **zapi**) protocol over Unix domain sockets.

### Connection Mechanism

- Zebra listens on a Unix socket at `/var/run/frr/zserv.api`
- Each protocol daemon connects as a client on startup
- The connection is persistent for the lifetime of the daemon
- If the connection drops, the protocol daemon attempts reconnection

### Message Flow

Protocol daemons send these message types to zebra:

| Message | Purpose |
|---|---|
| ROUTE_ADD | Install a route into the RIB |
| ROUTE_DELETE | Remove a route from the RIB |
| REDISTRIBUTE_ADD | Request notifications when routes from other protocols change |
| NEXTHOP_REGISTER | Register interest in nexthop reachability |
| INTERFACE_ADD | Request interface state notifications |
| ROUTER_ID_UPDATE | Request router-id change notifications |

Zebra sends these messages back to protocol daemons:

| Message | Purpose |
|---|---|
| INTERFACE_ADD/DELETE/UP/DOWN | Interface state changes |
| ROUTE_ADD/DELETE | Route redistribution notifications |
| NEXTHOP_UPDATE | Nexthop reachability changes |
| ROUTER_ID_UPDATE | Router ID changed |

### Socket Permissions

The zapi socket directory (`/var/run/frr/`) must be accessible to all FRR daemons. Daemons typically run as user `frr` in group `frr`. The VTY sockets for vtysh access require membership in the `frrvty` group.

## Routing Table Lifecycle

A route flows through FRR in this sequence:

```
1. Protocol daemon learns route
   (e.g., bgpd receives UPDATE from peer)
         |
         v
2. Protocol daemon selects best path internally
   (e.g., bgpd runs its own best-path algorithm)
         |
         v
3. Protocol daemon sends ROUTE_ADD via zapi
   (includes: prefix, nexthop(s), distance, metric, route type)
         |
         v
4. Zebra receives route, stores in RIB
   (one RIB per address family per VRF)
         |
         v
5. Zebra selects best route per prefix
   (compares administrative distance across all sources)
         |
         v
6. Zebra programs FIB via netlink
   (sends RTM_NEWROUTE / RTM_DELROUTE to kernel)
         |
         v
7. Kernel installs route in forwarding table
   (data plane now forwards packets along this path)
         |
         v
8. Zebra notifies other daemons via redistribution
   (if they registered for REDISTRIBUTE notifications)
```

### Route Removal

When a protocol daemon withdraws a route (ROUTE_DELETE via zapi), zebra removes it from the RIB. If another protocol has a route to the same prefix, zebra selects the next-best route and reprograms the FIB. If no routes remain, zebra removes the prefix from the kernel.

### ECMP

When multiple equal-cost paths exist (same distance and metric), zebra installs them all as ECMP nexthops. The default ECMP limit is 64 paths. Restrict with `zebra -e <N>`.

Verify ECMP routes:

```
vtysh -c "show ip route 10.0.0.0/24"
```

## watchfrr -- Daemon Supervisor

watchfrr monitors all other FRR daemons and restarts them on failure.

### How It Works

1. watchfrr connects to each daemon's VTY Unix socket
2. It sends echo commands at a configurable interval (default: 5 seconds)
3. If a daemon fails to respond within the timeout (default: 10 seconds), watchfrr declares it unresponsive
4. If the VTY socket returns EOF, watchfrr detects immediate failure
5. watchfrr executes the configured restart command for the failed daemon

### Restart Backoff

watchfrr uses exponential backoff between restart attempts:

- Minimum restart interval: 60 seconds (default)
- Maximum restart interval: 600 seconds (default)
- If the time since the last restart exceeds 2x the maximum delay, the backoff resets to minimum
- Otherwise, the delay doubles on each consecutive failure (capped at maximum)

### Key Options

```
watchfrr [--daemon] [-i <poll-interval>] [-t <timeout>] \
         [--min-restart-interval <seconds>] \
         [--max-restart-interval <seconds>] \
         [-S <statedir>] [-N <pathspace>]
```

### Exclude a Daemon from Monitoring

During development or debugging, prevent watchfrr from restarting a daemon:

```
vtysh -c "watchfrr ignore <daemon>"
```

### Check watchfrr Status

```
vtysh -c "show watchfrr"
```

## Architecture Diagram

```
+-------------------------------------------------------------------+
|                         vtysh (CLI)                                |
|  Connects to all daemons via VTY Unix sockets for unified CLI     |
+--------+-----+-----+-----+-----+-----+-----+-----+---------+----+
         |     |     |     |     |     |     |     |           |
         v     v     v     v     v     v     v     v           v
      +-----+-----+-----+-----+-----+-----+-----+------+ +--------+
      |bgpd |ospfd|isisd|ripd |pimd |ldpd |bfdd |staticd| |watchfrr|
      |     |     |     |     |     |     |     |       | |        |
      |     |     |     |     |     |     |     |       | |monitors|
      |     |     |     |     |     |     |     |       | |  all   |
      +--+--+--+--+--+--+--+--+--+--+--+--+--+--+---+--+ +---+----+
         |     |     |     |     |     |     |       |         |
         +-----+-----+-----+-----+-----+-----+------+---------+
                              |
                    zserv/zapi (Unix sockets)
                              |
                              v
                    +---------+---------+
                    |      zebra        |
                    |                   |
                    |  - RIB management |
                    |  - Best-path      |
                    |    selection      |
                    |  - FIB programming|
                    |  - Interface mgmt |
                    +---------+---------+
                              |
                     netlink / routing socket
                              |
                              v
                    +---------+---------+
                    |   Linux Kernel    |
                    |                   |
                    |   FIB (forwarding |
                    |    information    |
                    |      base)        |
                    +-------------------+
```

### Communication Paths

| Path | Protocol | Purpose |
|---|---|---|
| vtysh <-> daemons | Unix socket (VTY) | CLI commands and output |
| protocol daemons <-> zebra | Unix socket (zapi) | Route install/delete, interface events, redistribution |
| zebra <-> kernel | Netlink (Linux) | FIB programming, interface monitoring, route feedback |
| watchfrr <-> daemons | Unix socket (VTY) | Health check echo, status monitoring |
