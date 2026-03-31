# Performance

## Route Table Scale

### FRR Scale Guidelines

| Deployment | Routes | Memory | Recommendations |
|---|---|---|---|
| Small office | <1,000 | 256 MB | Default settings |
| Enterprise | 1,000--50,000 | 1--2 GB | Tune netlink buffer, increase FDs |
| ISP edge | 50,000--200,000 | 4--8 GB | BFD, graceful restart, ECMP tuning |
| Full BGP table | 900,000+ | 16--32 GB | Large netlink buffer, NHG, careful ECMP |
| Full BGP table + IPv6 | 1,000,000+ | 32--64 GB | All optimizations below |

### Netlink Buffer Size

For large routing tables, increase zebra's netlink buffer to avoid message drops:

```
# In /etc/frr/daemons
zebra_options="-s 90000000 --daemon -A 127.0.0.1"
```

The `-s` flag sets the buffer in bytes. Default is ~1 MB. For full BGP tables, use 90 MB or more.

### ECMP Path Limit

```bash
# Set maximum ECMP paths (zebra startup option)
# In /etc/frr/daemons:
zebra_options="--daemon -A 127.0.0.1 -e 64"
```

The `-e` flag limits equal-cost multipath entries. Default is 16. Higher values increase memory per route but improve load distribution.

### File Descriptor Limits

For large BGP deployments with many peers:

```
# In /etc/frr/daemons
MAX_FDS=4096
```

Or set system-wide in `/etc/security/limits.conf`:

```
frr  soft  nofile  65536
frr  hard  nofile  65536
```

## Memory Profiling

### Show Memory Usage by Daemon

```bash
# All daemons
vtysh -c "show memory"

# Specific daemon
vtysh -d bgpd -c "show memory"
vtysh -d zebrad -c "show memory"
vtysh -d ospfd -c "show memory"
```

### Show Memory by Type

```bash
vtysh -c "show memory bgpd"
```

This shows memory allocated to specific data structures (peer entries, path attributes, route entries, etc.).

### Key Memory Consumers

| Data Structure | Daemon | Grows With |
|---|---|---|
| BGP path attributes | bgpd | Unique path attribute combinations |
| BGP route entries | bgpd | Total prefixes x paths |
| BGP peer structures | bgpd | Number of BGP peers |
| OSPF LSA database | ospfd | Network size, area count |
| RIB route nodes | zebra | Total routes from all protocols |
| Nexthop groups | zebra | Unique next-hop combinations |
| Interface structures | zebra | Number of interfaces |

### Detect Memory Leaks

Monitor memory over time:

```bash
#!/bin/bash
# Track memory usage hourly
STAMP=$(date +%Y%m%d-%H%M%S)
echo "=== $STAMP ===" >> /var/log/frr/memory-trend.log
vtysh -c "show memory" >> /var/log/frr/memory-trend.log
```

Schedule via cron:

```
0 * * * * root /usr/local/bin/frr-memory-track.sh
```

Steadily increasing memory with a stable route count indicates a leak. File a bug with the output of `show memory` at multiple time points.

## CPU Profiling

### Show Thread CPU Usage

```bash
# All daemons
vtysh -c "show thread cpu"

# Specific daemon
vtysh -d bgpd -c "show thread cpu"
```

Output columns:

| Column | Meaning |
|---|---|
| Funcname | Function name |
| Calls | Number of invocations |
| Total | Total CPU time |
| Max | Worst-case single invocation |
| Avg | Average time per call |
| Type | Thread type (event, timer, I/O, execute) |

### Identify Hot Functions

```bash
# Sort by total CPU time (look for the top consumers)
vtysh -d bgpd -c "show thread cpu"
```

Common high-CPU functions and what they mean:

| Function | Daemon | Cause | Action |
|---|---|---|---|
| `bgp_process_packet` | bgpd | Large update bursts | Normal during convergence |
| `bgp_best_path_select` | bgpd | Route bestpath recalculation | Check update-groups, peer count |
| `ospf_spf_calculate` | ospfd | SPF runs | Check LSA churn, reduce area size |
| `isis_run_spf` | isisd | SPF runs | Check LSP churn |
| `rib_process` | zebra | Route installation | Normal for large table changes |
| `kernel_read` | zebra | Netlink message processing | Increase netlink buffer |

### Reset CPU Counters

```bash
vtysh -c "clear thread cpu"
```

Useful for measuring CPU impact of a specific operation (clear counters, perform operation, check counters).

## Convergence Tuning

### BGP Convergence

```
router bgp 65001
  ! Reduce keepalive/hold timers (default 60/180)
  neighbor 10.0.0.1 timers 10 30

  ! Reduce advertisement interval (default 30s for eBGP, 0 for iBGP)
  neighbor 10.0.0.1 advertisement-interval 5

  ! Enable next-hop tracking (faster reaction to nexthop changes)
  ! (enabled by default in modern FRR)

  ! Reduce BGP update delay after restart
  bgp update-delay 5

  address-family ipv4 unicast
    ! Enable add-path for faster convergence with route reflectors
    neighbor 10.0.0.1 addpath-tx-all-paths
  exit-address-family
```

### OSPF Convergence

```
router ospf
  ! Reduce SPF timers (initial-delay, min-holdtime, max-holdtime in ms)
  timers throttle spf 50 100 5000

  ! Reduce LSA generation timers
  timers throttle lsa all 0 50 5000

interface eth0
  ! Reduce hello/dead intervals (default 10/40)
  ip ospf hello-interval 1
  ip ospf dead-interval 4

  ! Or use minimal dead interval with multiplier
  ip ospf dead-interval minimal hello-multiplier 4
```

### IS-IS Convergence

```
router isis CORE
  ! Reduce SPF timers
  spf-interval level-1 50 100 5000
  spf-interval level-2 50 100 5000

  ! Reduce LSP generation interval
  lsp-gen-interval level-1 50 100 5000
  lsp-gen-interval level-2 50 100 5000

interface eth0
  ! Reduce hello interval
  isis hello-interval 1
  isis hello-multiplier 4
```

## BFD for Fast Failover

BFD provides sub-second failure detection, much faster than protocol hello timers.

### Configure BFD

```
bfd
  peer 10.0.0.1
    receive-interval 300
    transmit-interval 300
    detect-multiplier 3
  exit
```

This detects failure in 900 ms (3 x 300 ms).

### Link BFD to BGP

```
router bgp 65001
  neighbor 10.0.0.1 bfd
```

### Link BFD to OSPF

```
interface eth0
  ip ospf bfd
```

### Link BFD to IS-IS

```
interface eth0
  isis bfd
```

### BFD Timer Guidelines

| Environment | TX/RX Interval | Multiplier | Detection Time |
|---|---|---|---|
| Directly connected | 50 ms | 3 | 150 ms |
| Single-hop with jitter | 300 ms | 3 | 900 ms |
| Multi-hop / cloud | 1000 ms | 3 | 3 s |

> **Warning:** Aggressive BFD timers (under 100 ms) can cause false positives on loaded systems or virtual machines. Test thoroughly in your environment before deploying sub-100 ms timers.

## Graceful Restart

Graceful restart preserves forwarding state during daemon restarts, preventing traffic loss.

### BGP Graceful Restart

```
router bgp 65001
  bgp graceful-restart
  bgp graceful-restart preserve-fw-state
  bgp graceful-restart stalepath-time 300
  bgp graceful-restart restart-time 120
  bgp graceful-restart select-defer-time 45
```

| Parameter | Default | Purpose |
|---|---|---|
| `stalepath-time` | 360 s | How long to retain stale routes |
| `restart-time` | 120 s | Advertised restart timer to peers |
| `select-defer-time` | 360 s | Delay bestpath after restart |

### OSPF Graceful Restart

```
router ospf
  graceful-restart grace-period 120
  graceful-restart helper enable
```

### Zebra Route Retention

```
# In /etc/frr/daemons
zebra_options="--daemon -A 127.0.0.1 -K 120"
```

The `-K` flag tells zebra to keep routes in the kernel for the specified seconds during a restart. Essential for hitless restarts.

## Kernel FIB Optimization

### Use Route Replace

For IPv6, enable route replacement instead of delete-then-add:

```
# Zebra startup option
zebra_options="--daemon -A 127.0.0.1 --v6-rr-semantics"
```

### Dataplane Queue Tuning

```
configure terminal
  zebra dplane limit 5000
```

This sets the maximum number of pending route updates in the dataplane queue. Increase for large-scale route changes. The default is suitable for most deployments.

### FPM (Forwarding Plane Manager)

For hardware-offloaded forwarding or external FIB managers:

```
configure terminal
  fpm address 10.0.0.50 port 2620
  fpm use-next-hop-groups
  fpm use-route-replace
```

### Monitor Dataplane Performance

```bash
vtysh -c "show zebra dplane"
vtysh -c "show zebra dplane providers"
```

Look for queue depth and error counters. Consistently high queue depth means the kernel cannot keep up with route programming rate.

## Nexthop Tracking (NHT)

NHT allows protocols to react immediately to nexthop reachability changes rather than waiting for periodic scans.

### Verify NHT Status

```bash
vtysh -c "show ip nht"
vtysh -c "show ipv6 nht"
```

### Allow NHT via Default Route

By default, a nexthop resolved only via the default route is considered unreachable. To change this:

```
configure terminal
  ip nht resolve-via-default
  ipv6 nht resolve-via-default
```

> **Warning:** Enabling `resolve-via-default` means all nexthops are considered reachable if a default route exists. This can mask real reachability problems.

### NHT Route-Map Filtering

```
configure terminal
  ip nht route-map NHT_FILTER

route-map NHT_FILTER permit 10
  match ip address prefix-list VALID_NEXTHOPS

ip prefix-list VALID_NEXTHOPS seq 10 permit 10.0.0.0/8 le 32
```

### Nexthop Group Retention

```
configure terminal
  zebra nexthop-group keep 600
```

Controls how long unused nexthop groups are retained (seconds). Default is 180. Increase for environments with flapping nexthops to reduce churn.

## Recommendations by Scale

### Small Office (<1,000 Routes)

```
# /etc/frr/daemons -- default settings are sufficient
zebra_options="--daemon -A 127.0.0.1"
bgpd_options="--daemon -A 127.0.0.1"
```

- Default timers
- BFD optional
- 256 MB--1 GB memory

### Enterprise (1,000--50,000 Routes)

```
# /etc/frr/daemons
zebra_options="-s 16000000 --daemon -A 127.0.0.1"

# In frr.conf
router ospf
  timers throttle spf 50 100 5000

bfd
  peer 10.0.0.1
    receive-interval 300
    transmit-interval 300
    detect-multiplier 3
```

- Increase netlink buffer to 16 MB
- Enable BFD for critical links
- Tune OSPF SPF timers
- 1--4 GB memory

### ISP Edge (50,000--200,000 Routes)

```
# /etc/frr/daemons
zebra_options="-s 90000000 --daemon -A 127.0.0.1 -e 32 -K 120"
bgpd_options="--daemon -A 127.0.0.1"
MAX_FDS=4096

# In frr.conf
router bgp 65001
  bgp graceful-restart
  bgp graceful-restart preserve-fw-state
  neighbor PEERS timers 10 30
  neighbor PEERS advertisement-interval 5
  neighbor PEERS bfd
```

- 90 MB netlink buffer
- Graceful restart enabled
- BFD on all peers
- Reduced BGP timers
- 4--8 GB memory

### Full BGP Table (900,000+ Routes)

```
# /etc/frr/daemons
zebra_options="-s 90000000 --daemon -A 127.0.0.1 -e 64 -K 120 --v6-rr-semantics"
bgpd_options="--daemon -A 127.0.0.1"
MAX_FDS=65536

# System tuning
# /etc/sysctl.conf
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# In frr.conf
router bgp 65001
  bgp graceful-restart
  bgp graceful-restart preserve-fw-state
  bgp graceful-restart stalepath-time 600
  bgp graceful-restart restart-time 240
  bgp update-delay 10

  address-family ipv4 unicast
    maximum-paths 64
  exit-address-family

zebra dplane limit 10000
```

- Maximum netlink buffer
- 64-way ECMP
- Graceful restart with extended timers
- IPv6 route-replace semantics
- System TCP buffer tuning
- 16--64 GB memory
- Monitor with `show memory` and `show thread cpu` regularly
