# Upgrades

> **Warning:** Before upgrading, always check the [FRR release notes](https://frrouting.org/release/) and [known issues](https://github.com/FRRouting/frr/issues) for your target version. Breaking changes, deprecated commands, and migration requirements vary between releases.

## Pre-Upgrade Checklist

Complete every item before starting the upgrade:

| Step | Command / Action | Purpose |
|---|---|---|
| 1. Check current version | `vtysh -c "show version"` | Know your starting point |
| 2. Back up configuration | `sudo cp -a /etc/frr /etc/frr.bak.$(date +%Y%m%d)` | Restore point |
| 3. Back up running config | `vtysh -c "write memory"` then backup | Ensure saved config matches running |
| 4. Record route table | `vtysh -c "show ip route summary" > /tmp/routes-before.txt` | Post-upgrade comparison |
| 5. Record BGP state | `vtysh -c "show bgp summary" > /tmp/bgp-before.txt` | Post-upgrade comparison |
| 6. Record OSPF state | `vtysh -c "show ip ospf neighbor" > /tmp/ospf-before.txt` | Post-upgrade comparison |
| 7. Check disk space | `df -h /usr /var` | Ensure space for new packages |
| 8. Read release notes | Check target version changelog | Identify breaking changes |
| 9. Test in lab | Upgrade a non-production router first | Catch issues early |
| 10. Schedule window | Coordinate with NOC/team | Minimize impact |

### Save a Complete State Snapshot

```bash
#!/bin/bash
# save-state.sh -- run before and after upgrade
STAMP=$(date +%Y%m%d-%H%M%S)
DIR="/var/backup/frr/upgrade-${STAMP}"
mkdir -p "$DIR"
vtysh -c "show version" > "$DIR/version.txt"
vtysh -c "show running-config" > "$DIR/running-config.txt"
vtysh -c "show ip route summary" > "$DIR/route-summary.txt"
vtysh -c "show ip route" > "$DIR/routes.txt"
vtysh -c "show bgp summary" > "$DIR/bgp-summary.txt"
vtysh -c "show ip ospf neighbor" > "$DIR/ospf-neighbors.txt"
vtysh -c "show isis neighbor" > "$DIR/isis-neighbors.txt"
vtysh -c "show bfd peers brief" > "$DIR/bfd-peers.txt"
vtysh -c "show interface brief" > "$DIR/interfaces.txt"
cp -a /etc/frr "$DIR/etc-frr"
echo "State saved to $DIR"
```

## Package Upgrade (Debian/Ubuntu)

### Upgrade from FRR APT Repository

```bash
# 1. Update package lists
sudo apt update

# 2. Check available version
apt-cache policy frr

# 3. Upgrade FRR packages
sudo apt upgrade frr frr-pythontools

# 4. FRR restarts automatically via systemd
# Verify it came back
sudo systemctl status frr
vtysh -c "show version"
```

### Upgrade to a Specific Version

```bash
sudo apt install frr=10.2-1~deb12u1 frr-pythontools=10.2-1~deb12u1
```

### Hold/Unhold a Version

```bash
# Prevent accidental upgrades
sudo apt-mark hold frr frr-pythontools

# Allow upgrades again
sudo apt-mark unhold frr frr-pythontools
```

### RHEL/Rocky/AlmaLinux

```bash
# Check available version
sudo dnf info frr

# Upgrade
sudo dnf upgrade frr

# Restart (may not auto-restart)
sudo systemctl restart frr
```

## Source Upgrade

### Build and Install New Version

```bash
# 1. Stop FRR
sudo systemctl stop frr

# 2. Download and build
cd /usr/src
git clone https://github.com/FRRouting/frr.git frr-new
cd frr-new
git checkout frr-10.2

./bootstrap.sh
./configure \
  --prefix=/usr \
  --includedir=\${prefix}/include \
  --bindir=\${prefix}/bin \
  --sbindir=\${prefix}/lib/frr \
  --sysconfdir=/etc/frr \
  --localstatedir=/var/run/frr \
  --with-moduledir=/usr/lib/frr/modules \
  --enable-configfile-mask=0640 \
  --enable-logfile-mask=0640 \
  --enable-snmp=agentx \
  --enable-multipath=64 \
  --with-pkg-git-version

make -j$(nproc)

# 3. Install (overwrites binaries)
sudo make install

# 4. Start FRR
sudo systemctl start frr
vtysh -c "show version"
```

> **Warning:** Compile with the same `./configure` flags as the original build. Use `frr --version` or check `/usr/lib/frr/zebra --help` to review enabled features before upgrading.

## Container Upgrade

### Docker

```bash
# 1. Pull new image
docker pull quay.io/frrouting/frr:10.2.0

# 2. Stop current container
docker stop frr

# 3. Remove old container (config is in a volume)
docker rm frr

# 4. Start with new image
docker run -d --name frr \
  --privileged \
  --network host \
  -v /etc/frr:/etc/frr \
  -v /var/log/frr:/var/log/frr \
  quay.io/frrouting/frr:10.2.0

# 5. Verify
docker exec frr vtysh -c "show version"
```

### Kubernetes (Helm)

```bash
# Update image tag in values.yaml
# image:
#   tag: "10.2.0"

helm upgrade frr ./frr-chart -f values.yaml
kubectl rollout status daemonset/frr
```

## Graceful Restart During Upgrades

Graceful restart allows FRR to restart without dropping forwarded traffic. The kernel retains routes while daemons restart.

### Enable Graceful Restart for BGP

```
router bgp 65001
  bgp graceful-restart
  bgp graceful-restart preserve-fw-state
  bgp graceful-restart stalepath-time 300
  bgp graceful-restart restart-time 120
```

### Enable Graceful Restart for OSPF

```
router ospf
  graceful-restart grace-period 120
  graceful-restart helper enable
```

### Upgrade with Graceful Restart

```bash
# Zebra preserves routes during restart with -K flag
# The daemons file should include:
# zebra_options="-s 90000000 --daemon -A 127.0.0.1 -K 120"

# Perform upgrade (package or source)
sudo apt upgrade frr frr-pythontools

# Or restart manually with GR
sudo systemctl restart frr
```

The `-K 120` flag tells zebra to keep routes for 120 seconds during restart. This prevents traffic loss while daemons reload.

### Verify Graceful Restart Worked

```bash
# Check that routes were preserved (not withdrawn/re-added)
vtysh -c "show bgp neighbor 10.0.0.1" | grep -i "graceful"
vtysh -c "show ip route summary"
```

## Rollback Procedure

### Quick Rollback (Package)

```bash
# 1. Stop FRR
sudo systemctl stop frr

# 2. Restore config
sudo rm -rf /etc/frr
sudo cp -a /etc/frr.bak.YYYYMMDD /etc/frr

# 3. Downgrade package
sudo apt install frr=9.1-1~deb12u1 frr-pythontools=9.1-1~deb12u1

# 4. Start FRR
sudo systemctl start frr
vtysh -c "show version"
```

### Quick Rollback (Source)

```bash
# 1. Stop FRR
sudo systemctl stop frr

# 2. Rebuild old version
cd /usr/src/frr-old
sudo make install

# 3. Restore config
sudo rm -rf /etc/frr
sudo cp -a /etc/frr.bak.YYYYMMDD /etc/frr

# 4. Start FRR
sudo systemctl start frr
```

## 9.x to 10.x Migration

FRR 10.x includes several changes that may affect your configuration:

### Key Changes

| Area | Change | Action |
|---|---|---|
| CLI syntax | Some deprecated commands removed | Run `frr-reload.py --test` with new binary to find issues |
| BGP | Default `bgp ebgp-requires-policy` behavior | Ensure route-maps exist for eBGP peers |
| OSPF | Timer default changes | Compare before/after timer values |
| VRF | Netns-based VRF is default | Check VRF configuration syntax |
| Logging | Extended logging available | Review log configuration |
| Build | New dependencies | Check `configure` requirements |

### Migration Steps

```bash
# 1. Test configuration compatibility
# (with new binary installed but FRR stopped)
/usr/lib/frr/zebra --config_file /etc/frr/frr.conf --dryrun

# 2. Use frr-reload.py in test mode
sudo /usr/lib/frr/frr-reload.py --test /etc/frr/frr.conf

# 3. Fix any reported syntax errors in the config

# 4. Start FRR and verify
sudo systemctl start frr
```

### Use frr-reload.py for Configuration Changes

```bash
# Preview what changes frr-reload.py would make
sudo /usr/lib/frr/frr-reload.py --test /etc/frr/frr.conf

# Apply changes (live reload without full restart)
sudo /usr/lib/frr/frr-reload.py --reload /etc/frr/frr.conf

# Target a specific daemon
sudo /usr/lib/frr/frr-reload.py --reload --daemon bgpd /etc/frr/bgpd.conf

# With debug output
sudo /usr/lib/frr/frr-reload.py --reload --debug /etc/frr/frr.conf
```

## Post-Upgrade Verification

Run these checks immediately after upgrading:

```bash
# 1. Version check
vtysh -c "show version"

# 2. All daemons running
vtysh -c "show daemons"

# 3. Compare route tables
vtysh -c "show ip route summary"
# Compare with pre-upgrade snapshot

# 4. BGP peers re-established
vtysh -c "show bgp summary"

# 5. OSPF neighbors back to Full
vtysh -c "show ip ospf neighbor"

# 6. No errors in logs
sudo journalctl -u frr --since "10 minutes ago" --priority=err --no-pager

# 7. Traffic forwarding (from a remote host)
ping -c 5 <destination_through_router>
traceroute <destination_through_router>
```

### Automated Post-Upgrade Comparison

```bash
#!/bin/bash
# compare-state.sh BEFORE_DIR AFTER_DIR
BEFORE=$1
AFTER=$2

echo "=== Version Change ==="
diff "$BEFORE/version.txt" "$AFTER/version.txt"

echo
echo "=== Route Summary Diff ==="
diff "$BEFORE/route-summary.txt" "$AFTER/route-summary.txt"

echo
echo "=== BGP Summary Diff ==="
diff "$BEFORE/bgp-summary.txt" "$AFTER/bgp-summary.txt"

echo
echo "=== OSPF Neighbor Diff ==="
diff "$BEFORE/ospf-neighbors.txt" "$AFTER/ospf-neighbors.txt"

echo
echo "=== Interface Diff ==="
diff "$BEFORE/interfaces.txt" "$AFTER/interfaces.txt"

echo
echo "=== Config Diff ==="
diff -u "$BEFORE/running-config.txt" "$AFTER/running-config.txt"
```
