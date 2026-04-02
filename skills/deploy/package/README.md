# Install FRR from Packages

## Overview

FRR provides official pre-built packages for Debian/Ubuntu (deb) and RHEL/CentOS/Fedora (rpm). Packages are the recommended installation method for production deployments -- they include systemd integration, correct file permissions, and the frr user/group already configured.

Official repositories:
- **Debian/Ubuntu**: https://deb.frrouting.org/
- **RHEL/CentOS/Fedora**: https://rpm.frrouting.org/
- **Snap**: https://snapcraft.io/frr

## Install on Debian/Ubuntu

### Add the FRR APT Repository

```bash
# Install prerequisites
sudo apt-get update
sudo apt-get install -y curl gnupg lsb-release

# Add the FRR GPG key
curl -s https://deb.frrouting.org/frr/keys.gpg | sudo tee /usr/share/keyrings/frrouting.gpg > /dev/null

# Determine FRR release to install (use "frr-stable" for latest stable)
FRR_RELEASE="frr-stable"

# Add the repository
echo "deb [signed-by=/usr/share/keyrings/frrouting.gpg] https://deb.frrouting.org/frr $(lsb_release -cs) $FRR_RELEASE" | \
  sudo tee /etc/apt/sources.list.d/frr.list
```

### Install FRR

```bash
sudo apt-get update
sudo apt-get install -y frr frr-pythontools
```

The `frr-pythontools` package includes `frr-reload.py`, which enables `systemctl reload frr` to apply configuration changes without a full restart.

### Install a Specific Version

To install a specific FRR release branch, change the repository line:

```bash
# Available release names: frr-stable, frr-10, frr-9, frr-8, etc.
FRR_RELEASE="frr-10"

echo "deb [signed-by=/usr/share/keyrings/frrouting.gpg] https://deb.frrouting.org/frr $(lsb_release -cs) $FRR_RELEASE" | \
  sudo tee /etc/apt/sources.list.d/frr.list

sudo apt-get update
sudo apt-get install -y frr frr-pythontools
```

To pin to a specific patch version:

```bash
# List available versions
apt-cache policy frr

# Install a specific version
sudo apt-get install -y frr=10.2.1-0~ubuntu24.04.1 frr-pythontools=10.2.1-0~ubuntu24.04.1
```

Pin the version to prevent unintended upgrades:

```bash
cat <<EOF | sudo tee /etc/apt/preferences.d/frr
Package: frr
Pin: version 10.2.1*
Pin-Priority: 1001

Package: frr-pythontools
Pin: version 10.2.1*
Pin-Priority: 1001
EOF
```

## Install on RHEL/Rocky/CentOS

### Add the FRR YUM Repository

```bash
# For RHEL/Rocky/CentOS 8+
FRR_RELEASE="frr-stable"

curl -s https://rpm.frrouting.org/repo/$FRR_RELEASE-repo-1-0.el$(rpm -E %{rhel}).noarch.rpm -o /tmp/frr-repo.rpm
sudo dnf install -y /tmp/frr-repo.rpm
```

### Install FRR

```bash
sudo dnf install -y frr frr-pythontools
```

### Install a Specific Version

```bash
# List available versions
dnf list --showduplicates frr

# Install a specific version
sudo dnf install -y frr-10.2.1-01.el9 frr-pythontools-10.2.1-01.el9

# Lock the version to prevent upgrades
sudo dnf install -y dnf-plugin-versionlock
sudo dnf versionlock add frr frr-pythontools
```

## Install on Fedora

### Add the FRR Repository

```bash
FRR_RELEASE="frr-stable"

curl -s https://rpm.frrouting.org/repo/$FRR_RELEASE-repo-1-0.el$(rpm -E %{rhel}).noarch.rpm -o /tmp/frr-repo.rpm
sudo dnf install -y /tmp/frr-repo.rpm
```

### Install FRR

```bash
sudo dnf install -y frr frr-pythontools
```

## Install from Snap

```bash
sudo snap install frr
```

> **Note:** Snap installs use different configuration paths. Configuration lives under `/var/snap/frr/` rather than `/etc/frr/`.

## Verify Installation

### Check the Version

```bash
vtysh --version
```

Expected output:

```
vtysh version 10.2.1
Copyright 1997-2024 Kunihiro Ishiguro, et al.
```

### Check Daemon Binaries

```bash
ls /usr/lib/frr/
```

Key binaries:

| Binary | Purpose |
|---|---|
| `zebra` | Kernel route manager |
| `bgpd` | BGP daemon |
| `ospfd` | OSPFv2 daemon |
| `ospf6d` | OSPFv3 daemon |
| `isisd` | IS-IS daemon |
| `ripd` | RIPv2 daemon |
| `ripngd` | RIPng daemon |
| `pimd` | PIM multicast daemon |
| `ldpd` | LDP/MPLS daemon |
| `bfdd` | BFD daemon |
| `staticd` | Static route daemon |
| `watchfrr` | Daemon monitor/supervisor |
| `vrrpd` | VRRP daemon |
| `nhrpd` | NHRP daemon |
| `eigrpd` | EIGRP daemon |
| `babeld` | Babel daemon |
| `pbrd` | Policy-based routing daemon |
| `fabricd` | OpenFabric daemon |
| `pathd` | PCEP/Segment Routing daemon |

### Check the Service

```bash
systemctl status frr
```

## Package Contents

### File Locations

| Path | Contents |
|---|---|
| `/etc/frr/` | Configuration directory |
| `/etc/frr/daemons` | Daemon enablement file -- controls which daemons start |
| `/etc/frr/frr.conf` | Unified FRR configuration (all protocols in one file) |
| `/etc/frr/vtysh.conf` | vtysh configuration |
| `/etc/frr/support_bundle_commands.conf` | Support bundle command definitions |
| `/usr/lib/frr/` | Daemon binaries |
| `/usr/lib/frr/modules/` | Loadable modules (BMP, gRPC, SNMP, etc.) |
| `/usr/bin/vtysh` | The vtysh CLI binary |
| `/usr/lib/frr/frr-reload.py` | Configuration reload script |
| `/usr/lib/frr/frrinit.sh` | Init helper script |
| `/var/run/frr/` | PID files and Unix sockets (runtime) |
| `/var/log/frr/` | Log files |
| `/var/tmp/frr/` | Crash logs and backtraces |

### File Ownership

All files under `/etc/frr/` are owned by `frr:frr` with mode `0640`. The `vtysh.conf` file is owned by `frr:frrvty` so that members of the `frrvty` group can use vtysh without root.

```bash
ls -la /etc/frr/
```

Expected output:

```
-rw-r----- 1 frr frr     2691 ... daemons
-rw-r----- 1 frr frr      566 ... frr.conf
-rw-r----- 1 frr frrvty    35 ... vtysh.conf
-rw-r----- 1 frr frr     1133 ... support_bundle_commands.conf
```

## Uninstall FRR

### Debian/Ubuntu

```bash
sudo apt-get remove --purge frr frr-pythontools
sudo rm /etc/apt/sources.list.d/frr.list
sudo rm /usr/share/keyrings/frrouting.gpg
```

### RHEL/CentOS/Fedora

```bash
sudo dnf remove frr frr-pythontools
sudo rm /etc/yum.repos.d/frr*.repo
```

## Next Step

After installing FRR, proceed to `skills/deploy/initial-setup` to enable daemons and configure kernel parameters.
