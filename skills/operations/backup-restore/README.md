# Backup and Restore

## What to Back Up

| Path / Item | Contents | Critical |
|---|---|---|
| `/etc/frr/frr.conf` | Integrated configuration (all daemons) | Yes |
| `/etc/frr/daemons` | Which daemons are enabled, startup options | Yes |
| `/etc/frr/vtysh.conf` | vtysh settings (username, service integrated-config) | Yes |
| `/etc/frr/*.conf` | Per-daemon configs (if not using integrated config) | Yes |
| `/etc/frr/support_bundle_commands.conf` | Support bundle definitions | No |
| `/etc/frr/frr.conf.sav` | Previous config (auto-saved by `write memory`) | Useful |
| `/var/log/frr/` | FRR log files | Optional |

### Verify Configuration is Saved Before Backup

Always write the running configuration to disk before taking a backup:

```bash
vtysh -c "write memory"
```

This ensures the files on disk match what is actually running.

## Manual Backup

### Back Up the Entire /etc/frr Directory

```bash
sudo cp -a /etc/frr /etc/frr.bak.$(date +%Y%m%d)
```

### Back Up to a Tarball

```bash
sudo tar czf /var/backup/frr-config-$(date +%Y%m%d-%H%M%S).tar.gz /etc/frr/
```

### Back Up Running Config (Not Just Saved Config)

```bash
vtysh -c "show running-config" | sudo tee /var/backup/frr-running-$(date +%Y%m%d).conf
```

This captures the actual running state, which may differ from the saved config if changes were made without `write memory`.

### Back Up to a Remote Host

```bash
sudo tar czf - /etc/frr/ | ssh backup-server "cat > /backups/frr-$(hostname)-$(date +%Y%m%d).tar.gz"
```

## Automated Backup with Cron

### Daily Backup Script

Save as `/usr/local/bin/frr-backup.sh`:

```bash
#!/bin/bash
# FRR Configuration Backup
# Keeps 30 days of backups

BACKUP_DIR="/var/backup/frr"
RETENTION_DAYS=30
STAMP=$(date +%Y%m%d-%H%M%S)
HOSTNAME=$(hostname -s)

mkdir -p "$BACKUP_DIR"

# Save running config to disk first
vtysh -c "write memory" 2>/dev/null

# Create backup
tar czf "${BACKUP_DIR}/${HOSTNAME}-frr-${STAMP}.tar.gz" /etc/frr/ 2>/dev/null

# Also save a plain-text running config for easy diffing
vtysh -c "show running-config" > "${BACKUP_DIR}/${HOSTNAME}-running-${STAMP}.conf" 2>/dev/null

# Remove backups older than retention period
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +${RETENTION_DAYS} -delete
find "$BACKUP_DIR" -name "*.conf" -mtime +${RETENTION_DAYS} -delete

echo "$(date): Backup complete -> ${BACKUP_DIR}/${HOSTNAME}-frr-${STAMP}.tar.gz"
```

### Install the Cron Job

```bash
chmod +x /usr/local/bin/frr-backup.sh

# Run daily at 2:00 AM
echo "0 2 * * * root /usr/local/bin/frr-backup.sh >> /var/log/frr/backup.log 2>&1" | \
  sudo tee /etc/cron.d/frr-backup
```

### Backup Before Every Config Change (Git-Based)

For change tracking, use a git repository:

```bash
# Initialize
sudo git -C /etc/frr init
sudo git -C /etc/frr add -A
sudo git -C /etc/frr commit -m "Initial FRR config"

# After each change
vtysh -c "write memory"
sudo git -C /etc/frr add -A
sudo git -C /etc/frr commit -m "$(date): config change description"

# View history
sudo git -C /etc/frr log --oneline
sudo git -C /etc/frr diff HEAD~1
```

## Restore Procedure

### Restore from Tarball

```bash
# 1. Stop FRR
sudo systemctl stop frr

# 2. Move current config aside
sudo mv /etc/frr /etc/frr.broken

# 3. Extract backup
sudo tar xzf /var/backup/frr-config-20260101-020000.tar.gz -C /

# 4. Verify file permissions
sudo chown -R frr:frr /etc/frr
sudo chmod 640 /etc/frr/*.conf
sudo chmod 640 /etc/frr/daemons

# 5. Start FRR
sudo systemctl start frr

# 6. Verify
vtysh -c "show daemons"
vtysh -c "show running-config"
```

### Restore from Directory Copy

```bash
sudo systemctl stop frr
sudo rm -rf /etc/frr
sudo cp -a /etc/frr.bak.20260101 /etc/frr
sudo systemctl start frr
```

### Restore Running Config Without Restart

Use `frr-reload.py` to apply a saved configuration to running daemons without restarting:

```bash
# Preview changes
sudo /usr/lib/frr/frr-reload.py --test /var/backup/frr-running-20260101.conf

# Apply changes
sudo /usr/lib/frr/frr-reload.py --reload /var/backup/frr-running-20260101.conf
```

> **Warning:** `frr-reload.py` cannot handle all configuration changes. Structural changes (VRF creation, router-id changes) may still require a restart.

## Disaster Recovery

### Complete Recovery on a New Machine

```bash
# 1. Install FRR (same version as original)
# Follow the standard installation procedure for your OS

# 2. Copy backup to new machine
scp backup-server:/backups/frr-router1-20260101.tar.gz /tmp/

# 3. Extract config
sudo tar xzf /tmp/frr-router1-20260101.tar.gz -C /

# 4. Verify daemons file matches available daemons
cat /etc/frr/daemons

# 5. Fix permissions
sudo chown -R frr:frr /etc/frr
sudo chmod 640 /etc/frr/*.conf

# 6. Enable and start FRR
sudo systemctl enable frr
sudo systemctl start frr

# 7. Verify routing
vtysh -c "show daemons"
vtysh -c "show ip route summary"
```

### Recover from a Git-Based Backup

```bash
# View available snapshots
sudo git -C /etc/frr log --oneline

# Restore to a specific commit
sudo systemctl stop frr
sudo git -C /etc/frr checkout <commit-hash> -- .
sudo systemctl start frr
```

## Configuration Diff

### Compare Running Config to Saved Config

```bash
# Running config
vtysh -c "show running-config" > /tmp/running.conf

# Saved config
cat /etc/frr/frr.conf > /tmp/saved.conf

# Diff
diff -u /tmp/saved.conf /tmp/running.conf
```

### Compare Two Backup Files

```bash
diff -u /var/backup/frr-running-20260101.conf /var/backup/frr-running-20260201.conf
```

### Compare Against a Remote Router

```bash
# Get remote config
ssh router2 "vtysh -c 'show running-config'" > /tmp/router2.conf

# Get local config
vtysh -c "show running-config" > /tmp/router1.conf

# Compare
diff -u /tmp/router1.conf /tmp/router2.conf
```

### Use frr-reload.py for a Safe Diff

```bash
# This shows exactly what commands would be issued to transform
# the running config into the target config
sudo /usr/lib/frr/frr-reload.py --test /etc/frr/frr.conf
```

This is safer than a raw `diff` because it understands FRR configuration semantics (e.g., command ordering, implicit defaults).
