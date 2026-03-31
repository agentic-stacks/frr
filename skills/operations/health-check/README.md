# Health Check

## Quick Health Summary

Run these commands in sequence to assess overall FRR health:

```bash
# 1. Service status
sudo systemctl status frr

# 2. Daemon status via watchfrr
vtysh -c "show watchfrr"

# 3. All daemons responding
vtysh -c "show daemons"

# 4. Route summary
vtysh -c "show ip route summary"
vtysh -c "show ipv6 route summary"

# 5. Interface state
vtysh -c "show interface brief"

# 6. Zebra client connections
vtysh -c "show zebra client summary"
```

## Daemon Health

### Check systemd Service Status

```bash
sudo systemctl status frr
# Expected: active (running)

# Check if FRR will start on boot
sudo systemctl is-enabled frr
```

### Show Running Daemons

```bash
vtysh -c "show daemons"
```

Output lists every daemon that is currently running and connected to zebra. Compare against `/etc/frr/daemons` to confirm all enabled daemons started.

### Check Individual Daemon Process

```bash
# Verify specific daemon is running
pgrep -a bgpd
pgrep -a ospfd
pgrep -a zebra
```

## Watchfrr Monitoring

Watchfrr supervises all FRR daemons, automatically restarting any that become unresponsive.

### View Watchfrr Status

```bash
vtysh -c "show watchfrr"
```

This displays each daemon's state as seen by watchfrr. Healthy daemons show as responsive. Any daemon in a restart loop indicates a serious problem.

### Temporarily Ignore a Daemon

During debugging, prevent watchfrr from restarting a daemon you are investigating:

```bash
vtysh
  configure
    watchfrr ignore ospfd
```

> **Warning:** Do not leave a daemon in ignored state in production. Watchfrr will not restart it if it crashes.

## Protocol-Specific Health Checks

### BGP Health

```bash
# Peer summary -- check Established state and prefix counts
vtysh -c "show bgp summary"
vtysh -c "show bgp ipv4 unicast summary"
vtysh -c "show bgp ipv6 unicast summary"

# Detailed peer state for a specific neighbor
vtysh -c "show bgp neighbor 10.0.0.1"

# Check for peers not in Established state
vtysh -c "show bgp summary json" | jq '.ipv4Unicast.peers | to_entries[] | select(.value.state != "Established")'
```

| BGP State | Meaning | Action |
|---|---|---|
| Idle | Session not initiated | Check neighbor config, reachability |
| Connect | TCP connection in progress | Check firewalls, port 179 |
| Active | TCP failed, retrying | Check remote peer, ACLs |
| OpenSent | OPEN sent, waiting for reply | Check AS numbers, router-id |
| OpenConfirm | OPEN received, waiting for keepalive | Check capability mismatches |
| Established | Session up, exchanging routes | Healthy |

### OSPF Health

```bash
# Neighbor table -- check Full state
vtysh -c "show ip ospf neighbor"

# Interface status
vtysh -c "show ip ospf interface"

# Database summary
vtysh -c "show ip ospf database"

# Route table
vtysh -c "show ip ospf route"

# Check for stuck neighbors (not reaching Full)
vtysh -c "show ip ospf neighbor json" | jq '.neighbors | to_entries[] | select(.value[0].nbrState != "Full")'
```

| OSPF State | Meaning | Action |
|---|---|---|
| Down | No hellos received | Check interface, area config |
| Init | Hello received but not bidirectional | Check hello/dead timers, MTU |
| 2-Way | Bidirectional, no adjacency needed | Normal for DROther-DROther |
| ExStart | Master/slave negotiation | Check MTU mismatch |
| Exchange | DBD exchange | Check MTU, authentication |
| Loading | LSA exchange | Typically transient |
| Full | Fully adjacent | Healthy |

### IS-IS Health

```bash
# Adjacency table
vtysh -c "show isis neighbor"
vtysh -c "show isis neighbor detail"

# Interface status
vtysh -c "show isis interface"

# LSDB summary
vtysh -c "show isis database"

# Routes
vtysh -c "show isis route"
```

### BFD Health

```bash
# BFD session status
vtysh -c "show bfd peers"
vtysh -c "show bfd peers brief"
vtysh -c "show bfd peers counters"
```

## Route Table Verification

### Compare Route Counts by Protocol

```bash
vtysh -c "show ip route summary"
```

Example output:

```
Route Source         Routes    FIB
connected            4         4
static               2         2
ospf                 150       150
bgp                  50000     50000
------
Totals               50156     50156
```

The `Routes` column shows routes in the RIB. The `FIB` column shows routes installed in the kernel forwarding table. A mismatch indicates routes not being programmed.

### Verify a Specific Route

```bash
vtysh -c "show ip route 10.0.0.0/24"
vtysh -c "show ip route 10.0.0.0/24 json"
```

### Check Nexthop Tracking (NHT)

```bash
vtysh -c "show ip nht"
vtysh -c "show ipv6 nht"
```

NHT shows which nexthops are being tracked by protocols like BGP. Unresolved nexthops indicate reachability problems.

## Zebra FIB vs Kernel Verification

### Compare FIB and Kernel Routes

```bash
# FRR's view
vtysh -c "show ip route"

# Kernel's view
ip route show

# Compare counts
echo "FRR routes:" && vtysh -c "show ip route summary" | tail -1
echo "Kernel routes:" && ip route show | wc -l
```

### Check Dataplane Status

```bash
vtysh -c "show zebra dplane"
```

Look for:

| Counter | Healthy Value | Problem Indicator |
|---|---|---|
| Route updates enqueued | Incrementing | Stalled at same value |
| Route updates dequeued | Matches enqueued | Large gap = backlog |
| Route update errors | 0 or very low | Incrementing = kernel issues |
| Queue depth | 0 or near 0 | Consistently high = bottleneck |

### Check Zebra Client Status

```bash
vtysh -c "show zebra client summary"
```

Shows each connected daemon and its route/nexthop statistics. Use this to verify all expected daemons are connected.

## Interface State

### Summary of All Interfaces

```bash
vtysh -c "show interface brief"
```

### Detailed Interface Check

```bash
vtysh -c "show interface eth0"
```

Key fields to check:

| Field | Healthy | Problem |
|---|---|---|
| Line protocol | up | down |
| Link ups/downs | Low count | High count = flapping |
| Input/Output errors | 0 | Non-zero = physical issue |
| Speed | Expected value | auto-negotiation problems |

### Detect Flapping Interfaces

```bash
# Check kernel for carrier changes
ip -s link show eth0

# Check FRR logs for interface events
sudo grep -i "link" /var/log/frr/frr.log | tail -20
```

## Log Review

### Check FRR Log for Errors

```bash
# Recent errors and warnings
sudo grep -E "(error|warning|crash|assert|signal)" /var/log/frr/frr.log | tail -50

# BGP-specific issues
sudo grep -i "bgp" /var/log/frr/frr.log | grep -iE "(error|notification|cease)" | tail -20

# OSPF-specific issues
sudo grep -i "ospf" /var/log/frr/frr.log | grep -iE "(error|mismatch|auth)" | tail -20

# Watchfrr restart events
sudo grep -i "watchfrr" /var/log/frr/frr.log | tail -20
```

### Check systemd Journal

```bash
sudo journalctl -u frr --since "1 hour ago" --no-pager
sudo journalctl -u frr --priority=err --no-pager | tail -30
```

### Verify Log Configuration

```bash
vtysh -c "show logging"
```

## Scripted Health Check

Save this as `/usr/local/bin/frr-health-check.sh`:

```bash
#!/bin/bash
# FRR Health Check Script
# Returns non-zero on any failure

ERRORS=0

echo "=== FRR Health Check ==="
echo "Date: $(date)"
echo

# 1. Service running
echo "--- Service Status ---"
if systemctl is-active --quiet frr; then
    echo "PASS: FRR service is running"
else
    echo "FAIL: FRR service is not running"
    ERRORS=$((ERRORS + 1))
fi

# 2. Expected daemons running
echo
echo "--- Daemon Status ---"
EXPECTED_DAEMONS=$(grep -E '^\w+=yes' /etc/frr/daemons | cut -d= -f1)
RUNNING_DAEMONS=$(vtysh -c "show daemons" 2>/dev/null | tail -1)
for daemon in $EXPECTED_DAEMONS; do
    if echo "$RUNNING_DAEMONS" | grep -qw "$daemon"; then
        echo "PASS: $daemon is running"
    else
        echo "FAIL: $daemon is enabled but not running"
        ERRORS=$((ERRORS + 1))
    fi
done

# 3. BGP peer check (if bgpd enabled)
if echo "$RUNNING_DAEMONS" | grep -qw "bgpd"; then
    echo
    echo "--- BGP Peers ---"
    DOWN_PEERS=$(vtysh -c "show bgp summary json" 2>/dev/null | \
        jq -r '.ipv4Unicast.peers // {} | to_entries[] | select(.value.state != "Established") | .key' 2>/dev/null)
    if [ -z "$DOWN_PEERS" ]; then
        echo "PASS: All BGP peers established"
    else
        for peer in $DOWN_PEERS; do
            echo "FAIL: BGP peer $peer is not Established"
            ERRORS=$((ERRORS + 1))
        done
    fi
fi

# 4. OSPF neighbor check (if ospfd enabled)
if echo "$RUNNING_DAEMONS" | grep -qw "ospfd"; then
    echo
    echo "--- OSPF Neighbors ---"
    NOT_FULL=$(vtysh -c "show ip ospf neighbor json" 2>/dev/null | \
        jq -r '.neighbors // {} | to_entries[] | .value[] | select(.nbrState != "Full" and .nbrState != "2-Way") | .neighborIp' 2>/dev/null)
    if [ -z "$NOT_FULL" ]; then
        echo "PASS: All OSPF neighbors in expected state"
    else
        for nbr in $NOT_FULL; do
            echo "FAIL: OSPF neighbor $nbr not Full"
            ERRORS=$((ERRORS + 1))
        done
    fi
fi

# 5. Route table sanity
echo
echo "--- Route Table ---"
ROUTE_SUMMARY=$(vtysh -c "show ip route summary" 2>/dev/null)
TOTAL_ROUTES=$(echo "$ROUTE_SUMMARY" | grep "^Totals" | awk '{print $2}')
FIB_ROUTES=$(echo "$ROUTE_SUMMARY" | grep "^Totals" | awk '{print $3}')
if [ "$TOTAL_ROUTES" = "$FIB_ROUTES" ]; then
    echo "PASS: RIB ($TOTAL_ROUTES) matches FIB ($FIB_ROUTES)"
else
    echo "WARN: RIB ($TOTAL_ROUTES) differs from FIB ($FIB_ROUTES)"
    ERRORS=$((ERRORS + 1))
fi

# 6. Interface check
echo
echo "--- Interfaces ---"
DOWN_IFACES=$(vtysh -c "show interface brief json" 2>/dev/null | \
    jq -r 'to_entries[] | select(.value.operState == "down" and .value.adminState == "up") | .key' 2>/dev/null)
if [ -z "$DOWN_IFACES" ]; then
    echo "PASS: No admin-up/oper-down interfaces"
else
    for iface in $DOWN_IFACES; do
        echo "FAIL: Interface $iface is admin-up but oper-down"
        ERRORS=$((ERRORS + 1))
    done
fi

# Summary
echo
echo "=== Results ==="
if [ $ERRORS -eq 0 ]; then
    echo "ALL CHECKS PASSED"
    exit 0
else
    echo "$ERRORS CHECK(S) FAILED"
    exit 1
fi
```

Make it executable and schedule it:

```bash
chmod +x /usr/local/bin/frr-health-check.sh

# Run on demand
sudo /usr/local/bin/frr-health-check.sh

# Schedule via cron (every 5 minutes)
echo "*/5 * * * * root /usr/local/bin/frr-health-check.sh >> /var/log/frr/health-check.log 2>&1" | \
  sudo tee /etc/cron.d/frr-health-check
```
