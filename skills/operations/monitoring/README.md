# Monitoring

## Syslog Configuration

### Configure Logging in FRR

```
configure terminal
  log syslog informational
  log facility local7
  log record-priority
  log timestamp precision 3
```

### Log Levels

| Level | Keyword | Use Case |
|---|---|---|
| 0 | emergencies | System unusable |
| 1 | alerts | Immediate action required |
| 2 | critical | Critical conditions |
| 3 | errors | Error conditions |
| 4 | warnings | Warning conditions |
| 5 | notifications | Normal but significant |
| 6 | informational | Informational messages |
| 7 | debugging | Debug-level messages |

> **Recommendation:** Use `informational` for production. Use `debugging` only during troubleshooting and disable it afterward -- it generates heavy I/O.

### Log Targets

```
! Log to syslog
log syslog informational

! Log to file
log file /var/log/frr/frr.log informational

! Log to stdout (container use)
log stdout informational

! Log to monitor (vtysh terminal)
log monitor informational
```

### Per-Daemon Log Level

Set different levels for specific daemons in `/etc/frr/daemons`:

```
zebra_options="--daemon -A 127.0.0.1 --log file:/var/log/frr/zebra.log --log-level warning"
bgpd_options="--daemon -A 127.0.0.1 --log file:/var/log/frr/bgpd.log --log-level informational"
```

### Extended Logging (extlog)

FRR supports advanced logging targets with structured data:

```
configure terminal
  log extended MYLOG
    destination syslog
    format rfc5424
    priority informational
    facility local7
    timestamp-precision 6
    structured-data code-location
    structured-data version
    structured-data unique-id
```

Supported formats:

| Format | Description |
|---|---|
| `rfc5424` | Modern syslog with ISO 8601 timestamps, structured data |
| `rfc3164` | Legacy BSD syslog |
| `local-syslog` | RFC3164 without hostname |
| `journald` | systemd native protocol with structured data |

Supported destinations:

| Destination | Syntax | Notes |
|---|---|---|
| syslog | `destination syslog` | Writes to `/dev/log` |
| journald | `destination journald` | Needs systemd socket activation |
| file | `destination file /path/to/file` | Optional permissions |
| fd | `destination fd stdout` | stdout, stderr, or fd number |
| unix socket | `destination unix /path/to/socket` | Auto-detects socket type |
| none | `destination none` | Disables output, keeps config |

### Verify Logging Configuration

```bash
vtysh -c "show logging"
```

## SNMP

FRR uses the AgentX protocol (RFC 2741) to connect to a Net-SNMP master agent. FRR daemons register as AgentX subagents.

### Prerequisites

```bash
# Install Net-SNMP
sudo apt install snmpd snmp libsnmp-dev

# FRR must be compiled with --enable-snmp=agentx
# Package installs usually include this
```

### Configure Net-SNMP Master Agent

Edit `/etc/snmp/snmpd.conf`:

```
# Enable AgentX master
master agentx

# Community string (change in production!)
rocommunity mycomm 10.0.0.0/8

# Exclude heavy OIDs on routers with large tables
view all excluded .1.3.6.1.2.1.4.21     # ipRouteTable
view all excluded .1.3.6.1.2.1.4.22     # ipNetToMediaTable
view all excluded .1.3.6.1.2.1.4.24     # ipCidrRouteTable
view all excluded .1.3.6.1.2.1.4.35     # ipNetToPhysicalPhysAddress
```

> **Warning:** Polling `ipRouteTable` or `ipCidrRouteTable` on a router with a full BGP table can hang snmpd for minutes. Always exclude these OIDs.

### Enable AgentX in FRR Daemons

Add to each daemon's config (e.g., in `frr.conf`):

```
agentx
```

Or enable per-daemon:

```
! In bgpd section
router bgp 65001
  exit
agentx

! In ospfd section
router ospf
  exit
agentx
```

Restart FRR after enabling:

```bash
sudo systemctl restart frr
sudo systemctl restart snmpd
```

### Supported MIBs

| Daemon | MIB | OID Prefix |
|---|---|---|
| zebra | IF-MIB, IP-MIB | .1.3.6.1.2.1.2, .1.3.6.1.2.1.4 |
| bgpd | BGP4-MIB | .1.3.6.1.2.1.15 |
| bgpd | BGP4V2-MIB | .1.3.6.1.3.5.1 |
| ospfd | OSPF-MIB | .1.3.6.1.2.1.14 |
| ospf6d | OSPF-MIB (v3) | .1.3.6.1.2.1.191 |
| ripd | RIPv2-MIB | .1.3.6.1.2.1.23 |

### Verify SNMP Is Working

```bash
# Test OSPF MIB
snmpwalk -c mycomm -v2c localhost .1.3.6.1.2.1.14.1.1

# Test BGP MIB
snmpwalk -c mycomm -v2c localhost .1.3.6.1.2.1.15

# Check AgentX connection in FRR logs
sudo grep "AgentX" /var/log/frr/frr.log
# Expected: "NET-SNMP version X.X.X AgentX subagent connected"
```

## SNMP Traps

### Configure Trap Destination

Edit `/etc/snmp/snmpd.conf`:

```
# Send traps to management station
trapsink 10.0.0.100 mycomm
trap2sink 10.0.0.100 mycomm

# Or to a local snmptrapd
trapsink localhost mycomm
```

### Configure Trap Receiver

Edit `/etc/snmp/snmptrapd.conf`:

```
authCommunity log,execute mycomm

# Handle BGP traps with a script
traphandle .1.3.6.1.4.1.3317.1.2.2 /usr/local/bin/bgp-trap-handler.sh
```

### Enable BGP Traps in FRR

```
router bgp 65001
  bgp snmp traps rfc4273
  ! Or use the newer MIB:
  ! bgp snmp traps bgp4-mibv2

  ! Enable L3VPN notifications
  bgp snmp traps rfc4382
```

### BGP Trap Types

| MIB | Trap | Event |
|---|---|---|
| RFC 4273 | bgpEstablishedNotification | Peer reaches Established |
| RFC 4273 | bgpBackwardTransNotification | Peer leaves Established |
| BGP4V2-MIB | bgp4V2EstablishedNotification | Peer state change (v2) |
| RFC 4382 | mplsL3VpnVrfUp | VRF becomes operational |
| RFC 4382 | mplsL3VpnVrfDown | VRF goes down |

### Test Trap Delivery

```bash
# Start trap listener in foreground
sudo snmptrapd -f -Lo -c /etc/snmp/snmptrapd.conf

# On the FRR router, reset a BGP peer to trigger a trap
vtysh -c "clear bgp neighbor 10.0.0.1"
```

## BMP (BGP Monitoring Protocol)

BMP provides real-time BGP data streaming to external collectors (e.g., OpenBMP, pmacct, Wireshark).

### Enable BMP Module

Add `-M bmp` to bgpd startup options in `/etc/frr/daemons`:

```
bgpd_options="--daemon -A 127.0.0.1 -M bmp"
```

Restart bgpd after changing daemons file.

### Configure BMP Target

```
router bgp 65001
  ! Create a BMP target group
  bmp targets COLLECTOR1
    ! Active connection to collector
    bmp connect 10.0.0.200 port 11019 min-retry 1000 max-retry 64000

    ! Monitor pre-policy and post-policy routes
    bmp monitor ipv4 unicast pre-policy
    bmp monitor ipv4 unicast post-policy
    bmp monitor ipv6 unicast pre-policy
    bmp monitor ipv6 unicast post-policy

    ! Enable route mirroring (copies raw BGP messages)
    bmp mirror

    ! Send statistics every 60 seconds
    bmp stats interval 60000
  exit
```

### Configure a Passive BMP Listener

```
router bgp 65001
  bmp targets PASSIVE1
    bmp listener 0.0.0.0 port 11019
    bmp monitor ipv4 unicast post-policy
  exit
```

### BMP Buffer Management

```
router bgp 65001
  ! Limit memory for route mirror buffering (bytes)
  bmp mirror buffer-limit 268435456
```

### Monitor BMP from Other VRFs

```
router bgp 65001
  bmp targets COLLECTOR1
    bmp import-vrf-view VRF_CUSTOMER1
  exit
```

## Prometheus / Metrics

FRR does not have a native Prometheus exporter. Use one of these approaches:

### Option 1: SNMP Exporter

```bash
# Install prometheus-snmp-exporter
sudo apt install prometheus-snmp-exporter

# Configure /etc/prometheus/snmp.yml with FRR MIBs
# Then add to prometheus.yml:
```

```yaml
scrape_configs:
  - job_name: 'frr-snmp'
    static_configs:
      - targets: ['router1:161']
    metrics_path: /snmp
    params:
      module: [frr]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: snmp-exporter:9116
```

### Option 2: Custom vtysh Scraper

```bash
#!/bin/bash
# frr-metrics.sh -- expose FRR metrics as Prometheus text format
# Serve with: while true; do { echo -e "HTTP/1.1 200 OK\n"; bash frr-metrics.sh; } | nc -l -p 9100 -q 1; done

# BGP peer count
ESTABLISHED=$(vtysh -c "show bgp summary json" | jq '[.ipv4Unicast.peers | to_entries[] | select(.value.state == "Established")] | length')
echo "frr_bgp_peers_established $ESTABLISHED"

# Total routes
TOTAL=$(vtysh -c "show ip route summary json" | jq '.routes')
echo "frr_routes_total $TOTAL"

# FIB routes
FIB=$(vtysh -c "show ip route summary json" | jq '.fib')
echo "frr_fib_routes_total $FIB"
```

### Option 3: frr_exporter (Community)

Third-party Prometheus exporters exist for FRR. Search for `frr_exporter` on GitHub. These typically poll vtysh or SNMP and expose metrics at `/metrics`.

## Log Rotation

### Configure logrotate

Create `/etc/logrotate.d/frr`:

```
/var/log/frr/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 frr frr
    sharedscripts
    postrotate
        # Signal daemons to reopen log files
        kill -USR1 $(cat /var/run/frr/zebra.pid 2>/dev/null) 2>/dev/null || true
        kill -USR1 $(cat /var/run/frr/bgpd.pid 2>/dev/null) 2>/dev/null || true
        kill -USR1 $(cat /var/run/frr/ospfd.pid 2>/dev/null) 2>/dev/null || true
    endscript
}
```

### Test logrotate

```bash
sudo logrotate -d /etc/logrotate.d/frr   # dry-run
sudo logrotate -f /etc/logrotate.d/frr   # force rotation
```

### systemd Journal Size

If using journald for FRR logs:

```bash
# Check journal disk usage
journalctl --disk-usage

# Set max size in /etc/systemd/journald.conf
# SystemMaxUse=500M
# MaxRetentionSec=30day

sudo systemctl restart systemd-journald
```

## What to Alert On

### Critical Alerts (Page Immediately)

| Condition | Detection | Command |
|---|---|---|
| FRR service down | systemd | `systemctl is-active frr` |
| Daemon crash/restart | watchfrr logs | `grep "restart" /var/log/frr/frr.log` |
| All BGP peers down | Zero established peers | `show bgp summary json` |
| OSPF adjacency lost | Neighbor count drops to 0 | `show ip ospf neighbor json` |
| No routes in FIB | Route count = 0 | `show ip route summary` |
| RIB/FIB mismatch | Routes != FIB count | `show ip route summary` |

### Warning Alerts (Investigate Soon)

| Condition | Detection | Command |
|---|---|---|
| Single BGP peer down | Peer not Established | `show bgp summary json` |
| BGP prefix count drop >10% | Compare with baseline | `show bgp summary json` |
| OSPF neighbor not Full | State stuck in ExStart/Exchange | `show ip ospf neighbor json` |
| Interface flapping | Link up/down events | `show interface` counters |
| High memory usage | Memory approaching limit | `show memory` |
| High CPU on thread | Single thread consuming CPU | `show thread cpu` |
| BFD session down | BFD peer state change | `show bfd peers` |
| Log errors increasing | Error rate spike | `grep -c "error" /var/log/frr/frr.log` |

### Informational (Dashboard/Trending)

| Metric | Command | Purpose |
|---|---|---|
| Total routes by protocol | `show ip route summary` | Capacity planning |
| BGP prefixes received per peer | `show bgp summary` | Peer health trending |
| Memory per daemon | `show memory` | Leak detection |
| Uptime | `show version` | Track restart events |
| Convergence time | BFD + timer config | SLA verification |

### Example Nagios/Icinga Check

```bash
#!/bin/bash
# check_frr_bgp -- Nagios-compatible BGP check
# Returns: 0=OK, 1=WARNING, 2=CRITICAL

EXPECTED_PEERS=${1:-1}
ESTABLISHED=$(vtysh -c "show bgp summary json" 2>/dev/null | \
  jq '[.ipv4Unicast.peers // {} | to_entries[] | select(.value.state == "Established")] | length' 2>/dev/null)

if [ -z "$ESTABLISHED" ]; then
    echo "CRITICAL: Cannot query BGP status"
    exit 2
elif [ "$ESTABLISHED" -eq 0 ]; then
    echo "CRITICAL: No BGP peers established (expected $EXPECTED_PEERS)"
    exit 2
elif [ "$ESTABLISHED" -lt "$EXPECTED_PEERS" ]; then
    echo "WARNING: $ESTABLISHED/$EXPECTED_PEERS BGP peers established"
    exit 1
else
    echo "OK: $ESTABLISHED BGP peers established"
    exit 0
fi
```
