# Debugging

FRR debug commands, log management, packet captures, core dumps, and tracing. Use these tools for deep diagnosis when troubleshooting decision trees require protocol-level visibility.

> **WARNING:** Debug commands generate high log volume. Never leave debug enabled in production. Always disable debug when diagnosis is complete. Failing to do so can cause high CPU, disk exhaustion, and service degradation.

## Show Current Debug State

Before enabling anything, check what is already active:

```
show debugging
```

This lists all currently enabled debug flags across the daemon you are connected to. Run it in each daemon context (bgpd, ospfd, zebra, etc.) via vtysh.

## FRR Debug Commands

### BGP Debug

```
! All BGP updates (high volume)
debug bgp updates

! Updates for a specific peer only
debug bgp updates <A.B.C.D|X:X::X:X> [in|out]

! BGP session events (OPEN, NOTIFICATION, state changes)
debug bgp neighbor-events

! Keepalive messages
debug bgp keepalives

! AS-path and filter evaluation
debug bgp filters

! Route bestpath selection
debug bgp bestpath <PREFIX>

! BFD integration
debug bgp bfd

! Graceful restart events
debug bgp graceful-restart

! Label allocation (MPLS/VPN)
debug bgp labelpool

! Update groups
debug bgp update-groups

! Zebra communication
debug bgp zebra

! All BGP debug at once (extreme volume -- use only on test systems)
debug bgp all
```

### OSPF Debug

```
! OSPF packet types: hello, dd, ls-request, ls-update, ls-ack
debug ip ospf packet <TYPE> [send|recv] [detail]

! ISM (Interface State Machine) events
debug ip ospf ism [status|events|timers]

! NSM (Neighbor State Machine) events
debug ip ospf nsm [status|events|timers]

! LSA generation, flooding, installation
debug ip ospf lsa [generate|flooding|install|refresh]

! SPF calculation
debug ip ospf spf

! Route table calculation
debug ip ospf rt

! Zebra communication
debug ip ospf zebra [send|recv]

! NSSA handling
debug ip ospf nssa
```

### IS-IS Debug

```
debug isis events
debug isis packet-dump
debug isis snp-packets
debug isis update-packets
debug isis spf-events
debug isis route-events
debug isis lsp-gen
debug isis adj-packets
```

### Zebra Debug

```
! Route install/remove events
debug zebra rib [detailed]

! Kernel communication
debug zebra kernel [msgdump [send|recv]]

! FPM (Forwarding Plane Manager)
debug zebra fpm

! Nexthop tracking
debug zebra nht [detailed]

! Pseudowire events
debug zebra pw

! MPLS processing
debug zebra mpls [detailed]

! Interface events
debug zebra events

! Packet communication with daemons
debug zebra packet [send|recv] [detail]

! Dataplane interactions
debug zebra dplane [detailed]
```

### Other Daemons

```
! BFD
debug bfd peer <A.B.C.D> [multihop|local-address <A.B.C.D>]
debug bfd zebra
debug bfd network

! PIM
debug pim events
debug pim packets
debug pim trace
debug pim zebra

! RIP
debug rip events
debug rip packet [send|recv]
debug rip zebra

! VRRP
debug vrrp [<(1-255)>] [{proto|autoconfigure|packets|sockets|ndisc|arp|zebra}]
```

## Disable Debug

Always disable debug after investigation:

```
! Disable a specific debug
no debug bgp updates

! Disable all debug in current daemon
undebug all
```

Verify with `show debugging` that no flags remain active.

## Log Levels

| Level | Keyword | Use |
|---|---|---|
| 0 | emergencies | System unusable |
| 1 | alerts | Immediate action needed |
| 2 | critical | Critical conditions |
| 3 | errors | Error conditions |
| 4 | warnings | Warning conditions |
| 5 | notifications | Significant normal events |
| 6 | informational | Informational (production default) |
| 7 | debugging | Debug messages (troubleshooting only) |

### Set Log Level at Runtime

```
configure terminal
  log syslog debugging
  log file /var/log/frr/frr.log debugging
  log record-priority
  log timestamp precision 3
```

### Enable Monitor Logging (Live Terminal)

```
terminal monitor
```

This streams log output to your current vtysh session. Disable with `terminal no monitor`.

> **WARNING:** `terminal monitor` combined with high debug levels can flood your terminal and may cause vtysh to become unresponsive. Use targeted debug flags, not blanket `debug all`.

## Crash Logs

FRR automatically writes crash logs to:

```
/var/tmp/frr.<daemon>.crashlog
```

Check for recent crashes:

```
ls -lt /var/tmp/frr.*.crashlog
```

Read the most recent crash:

```
cat /var/tmp/frr.<daemon>.crashlog
```

The crash log contains a backtrace captured by FRR's signal handler. It may be incomplete compared to a full gdb backtrace from a core dump.

## Core Dumps

### Enable Core Dumps

1. Set system-level core dump limits:

```bash
# Temporary (current shell)
ulimit -c unlimited

# Persistent for FRR service (systemd)
sudo mkdir -p /etc/systemd/system/frr.service.d
sudo tee /etc/systemd/system/frr.service.d/coredump.conf <<EOF
[Service]
LimitCORE=infinity
EOF
sudo systemctl daemon-reload
sudo systemctl restart frr
```

2. Configure core dump location:

```bash
# Set core pattern (system-wide)
echo '/var/tmp/core.%e.%p.%t' | sudo tee /proc/sys/kernel/core_pattern

# Or use systemd-coredump (recommended on systemd systems)
echo 'kernel.core_pattern=|/lib/systemd/systemd-coredump %P %u %g %s %t %c %h' | sudo tee /etc/sysctl.d/50-coredump.conf
sudo sysctl --system
```

3. Verify FRR was compiled with debug symbols:

```bash
file /usr/lib/frr/bgpd
# Should show "not stripped" for debug symbols
```

### Locate Core Dumps

```bash
# If using core_pattern
ls -lt /var/tmp/core.*

# If using systemd-coredump
coredumpctl list frr
coredumpctl list bgpd

# Get info about latest crash
coredumpctl info bgpd
```

### Analyze with GDB

```bash
# Direct core file
gdb /usr/lib/frr/<daemon> /var/tmp/core.<daemon>.<pid>.<timestamp>

# Or via systemd-coredump
coredumpctl gdb bgpd
```

Essential GDB commands once loaded:

```
(gdb) bt full              # Full backtrace with local variables
(gdb) info threads         # List all threads
(gdb) thread apply all bt  # Backtrace for every thread
(gdb) frame <N>            # Select stack frame N for inspection
(gdb) print <variable>     # Inspect variable values
(gdb) info locals          # Show local variables in current frame
```

> **Tip:** Install FRR debug symbol packages if available (`frr-dbgsym` or `frr-debuginfo`) for the most useful backtraces. Without symbols, backtraces show hex addresses instead of function names.

## Packet Captures

### BGP (TCP Port 179)

```bash
# Capture all BGP traffic on any interface
sudo tcpdump -i any -nn -s0 -w /tmp/bgp.pcap tcp port 179

# Filter to a specific peer
sudo tcpdump -i eth0 -nn -s0 -w /tmp/bgp.pcap host 10.0.0.1 and tcp port 179

# Live decode (no write)
sudo tcpdump -i any -nn -v tcp port 179
```

### OSPF (IP Protocol 89)

```bash
# Capture all OSPF traffic
sudo tcpdump -i any -nn -s0 -w /tmp/ospf.pcap proto 89

# Specific interface
sudo tcpdump -i eth0 -nn -s0 -w /tmp/ospf.pcap proto 89

# OSPF multicast only (hello, all-routers)
sudo tcpdump -i eth0 -nn -s0 -w /tmp/ospf.pcap 'dst 224.0.0.5 or dst 224.0.0.6'
```

### IS-IS (Layer 2, EtherType 0x83)

```bash
# IS-IS runs directly on Layer 2, not IP
sudo tcpdump -i eth0 -nn -s0 -w /tmp/isis.pcap iso

# Alternative filter by multicast MAC
sudo tcpdump -i eth0 -nn -s0 -w /tmp/isis.pcap ether dst 01:80:c2:00:00:14 or ether dst 01:80:c2:00:00:15 or ether dst 09:00:2b:00:00:05
```

### BFD (UDP Ports 3784/3785)

```bash
sudo tcpdump -i any -nn -s0 -w /tmp/bfd.pcap udp port 3784 or udp port 3785
```

### General Tips

- Always use `-s0` (capture full packets), `-nn` (no DNS/port name resolution), and `-w` (write to file)
- Analyze pcap files in Wireshark for protocol-level decode
- Limit capture duration with `-c <count>` (packet count) or `-G <seconds>` (rotation interval)
- On busy routers, add host filters to avoid capturing unrelated traffic and filling disk

> **WARNING:** Packet captures on production routers consume CPU and disk I/O. Use targeted filters and set file size limits (`-C <MB>`) to prevent disk exhaustion.

## Tracing (LTTng / USDT)

FRR supports static tracepoints with zero overhead when disabled. Two backends are available:

### Check If Tracing Is Compiled In

```bash
# USDT probes
readelf -n /usr/lib/frr/bgpd | grep provider

# LTTng tracepoints
lttng list --userspace | grep frr
```

### LTTng Tracing

> Requires FRR compiled with `--enable-lttng=yes` and `liblttng-ust` installed.

```bash
# Create a session
lttng create frr-debug --output=/tmp/frr-trace

# Enable all BGP tracepoints
lttng enable-event --userspace 'frr_bgp:*'

# Enable all library tracepoints (memory, threading, scheduling)
lttng enable-event --userspace 'frr_libfrr:*'

# Enable zlog messages as trace events
lttng enable-event --userspace 'lttng_ust_tracelog:*'

# Start tracing
lttng start

# ... reproduce the problem ...

# Stop and view
lttng stop
lttng view
lttng destroy
```

If the daemon runs in daemonizing mode (`-d`), preload the fork handler:

```bash
LD_PRELOAD=liblttng-ust-fork.so /usr/lib/frr/bgpd
```

For systemd, add to a service override:

```
[Service]
Environment="LD_PRELOAD=liblttng-ust-fork.so"
```

### USDT Probes

> Requires FRR compiled with `--enable-usdt=yes`.

Available BGP tracepoints: `packet_read`, `open_process`, `update_process`, `notification_process`, `keepalive_process`, `refresh_process`, `capability_process`, `input_filter`, `output_filter`, `process_update`.

```bash
# Discover probes
readelf -n /usr/lib/frr/bgpd

# Use with SystemTap
stap -L 'process("/usr/lib/frr/bgpd").mark("*")'
```

> **Note:** USDT probes have higher overhead than LTTng. Use LTTng for continuous "flight recorder" tracing; use USDT for ad-hoc investigation on systems where LTTng is not available.

## Common Debug Workflows

### Diagnose a BGP Session Problem

1. Check current state:
   ```
   show bgp summary
   show bgp neighbor <IP>
   ```

2. Enable targeted debug:
   ```
   debug bgp neighbor-events
   debug bgp updates <PEER_IP>
   terminal monitor
   ```

3. Watch for events (OPEN, NOTIFICATION, state transitions).

4. If TCP-level issue suspected, capture packets:
   ```bash
   sudo tcpdump -i any -nn -s0 -w /tmp/bgp-debug.pcap host <PEER_IP> and tcp port 179
   ```

5. Disable debug:
   ```
   no debug bgp neighbor-events
   no debug bgp updates <PEER_IP>
   terminal no monitor
   ```

### Diagnose an OSPF Adjacency Problem

1. Check neighbor state:
   ```
   show ip ospf neighbor
   show ip ospf interface <IFACE>
   ```

2. Enable targeted debug:
   ```
   debug ip ospf packet hello recv detail
   debug ip ospf nsm status
   terminal monitor
   ```

3. Look for hello mismatches (area, timers, authentication, MTU).

4. If needed, capture packets:
   ```bash
   sudo tcpdump -i <IFACE> -nn -s0 -w /tmp/ospf-debug.pcap proto 89
   ```

5. Disable debug:
   ```
   no debug ip ospf packet hello recv detail
   no debug ip ospf nsm status
   terminal no monitor
   ```

### Diagnose a Route Installation Failure

1. Check protocol table vs RIB:
   ```
   show bgp ipv4 unicast <PREFIX>
   show ip route <PREFIX>
   ```

2. Enable zebra debug:
   ```
   debug zebra rib detailed
   debug zebra kernel
   terminal monitor
   ```

3. Clear and re-add the route (or trigger a BGP soft-reset) to see the installation path:
   ```
   clear bgp ipv4 unicast <PEER_IP> soft in
   ```

4. Check zebra's client communication:
   ```
   show zebra client summary
   debug zebra packet send detail
   ```

5. Disable debug:
   ```
   undebug all
   terminal no monitor
   ```
