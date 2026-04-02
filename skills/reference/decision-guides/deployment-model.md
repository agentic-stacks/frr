# Deployment Model Guide

Choose between packages, containers, and source builds for FRR.

---

## Deployment Model Comparison

| Criterion | Packages (deb/rpm) | Containers (Docker/Podman) | Source Build | Snap |
|---|---|---|---|---|
| **Ease of install** | High | High | Low | High |
| **Ease of upgrade** | High (apt/dnf) | High (pull new image) | Low (rebuild) | High (snap refresh) |
| **Rollback** | Medium (pin versions) | High (tag previous image) | Low (rebuild old) | High (snap revert) |
| **Isolation** | None (system-wide) | Full (namespace) | None (system-wide) | Partial (snap confinement) |
| **Customization** | Low (prebuilt options) | Medium (custom Dockerfile) | Full (compile flags) | Low |
| **Production readiness** | High | High | High (if maintained) | Medium |
| **Debug symbols** | Optional (-dbgsym) | Must build custom | Full control | No |
| **Startup integration** | systemd (native) | systemd + docker | systemd (manual) | systemd (snap) |
| **Multi-version coexist** | No | Yes (different containers) | Yes (different prefix) | No |
| **Kernel module access** | Direct | Requires --privileged/caps | Direct | Limited |
| **FPM support** | Yes | Yes | Yes (if enabled) | Yes |
| **gRPC support** | Some packages | Must build custom | If --enable-grpc | No |

---

## Recommendations by Use Case

### Production Network

**Recommended: Packages (deb/rpm) from deb.frrouting.org / rpm.frrouting.org**

| Factor | Details |
|---|---|
| Why | Tested, signed, systemd integration, easy patching |
| Upgrade | `apt upgrade frr` / `dnf upgrade frr` |
| Rollback | Pin to previous version; keep .deb/.rpm files |
| Monitoring | Standard systemd journal, logrotate included |

```bash
# Install specific version (Debian/Ubuntu)
apt install frr=10.5.3-0~ubuntu22.04.1

# Pin version to prevent unintended upgrades
apt-mark hold frr

# Upgrade when ready
apt-mark unhold frr
apt install frr=10.6.0-0~ubuntu22.04.1
```

**When to choose source instead**: You need gRPC, LTTng, or custom compile flags not in the package.

---

### Development and Testing

**Recommended: Containers (Docker)**

| Factor | Details |
|---|---|
| Why | Instant setup/teardown, version switching, no host contamination |
| Image | `quay.io/frrouting/frr:10.6.0` |
| Networking | Use `--privileged --net=host` for full functionality |
| Multi-node | Combine with containerlab or docker-compose |

```bash
# Quick single-node test
docker run -d --name frr-test --privileged --net=host \
  -v /path/to/frr.conf:/etc/frr/frr.conf \
  quay.io/frrouting/frr:10.6.0

# Enter vtysh
docker exec -it frr-test vtysh

# Switch versions
docker stop frr-test && docker rm frr-test
docker run -d --name frr-test --privileged --net=host \
  -v /path/to/frr.conf:/etc/frr/frr.conf \
  quay.io/frrouting/frr:10.5.3
```

---

### Lab / Topology Testing

**Recommended: Containers with containerlab**

| Factor | Details |
|---|---|
| Why | Declarative topology, point-to-point links, easy teardown |
| Tool | containerlab (https://containerlab.dev) |
| Scale | 50+ nodes on a single host |

```yaml
# containerlab topology file
name: frr-lab
topology:
  nodes:
    spine1:
      kind: linux
      image: quay.io/frrouting/frr:10.6.0
      binds:
        - spine1.conf:/etc/frr/frr.conf
    spine2:
      kind: linux
      image: quay.io/frrouting/frr:10.6.0
      binds:
        - spine2.conf:/etc/frr/frr.conf
    leaf1:
      kind: linux
      image: quay.io/frrouting/frr:10.6.0
      binds:
        - leaf1.conf:/etc/frr/frr.conf
  links:
    - endpoints: ["spine1:eth1", "leaf1:eth1"]
    - endpoints: ["spine2:eth1", "leaf1:eth2"]
```

```bash
containerlab deploy -t topology.yml
containerlab destroy -t topology.yml
```

**Alternative**: Use topotests (FRR's own test framework) for automated regression testing.

---

### CI/CD Pipeline

**Recommended: Containers with pinned image tags**

| Factor | Details |
|---|---|
| Why | Reproducible, fast spin-up, version-pinned |
| Strategy | Pin exact image digest, not just tag |
| Testing | Run topotests in containers |

```yaml
# GitHub Actions example
jobs:
  frr-test:
    runs-on: ubuntu-latest
    services:
      frr:
        image: quay.io/frrouting/frr:10.6.0
        options: --privileged
    steps:
      - name: Verify FRR
        run: docker exec frr vtysh -c "show version"
```

---

### Custom Feature Build

**Recommended: Source build**

| Factor | Details |
|---|---|
| Why | Full control over compile flags, enable/disable daemons |
| When | Need gRPC, custom patches, LTTng, sanitizers |
| Install prefix | Use `/usr/local/frr` to avoid conflicts with packages |

```bash
git clone https://github.com/FRRouting/frr.git
cd frr
git checkout frr-10.6.0

./bootstrap.sh
./configure \
  --prefix=/usr \
  --sysconfdir=/etc/frr \
  --localstatedir=/var/run/frr \
  --sbindir=/usr/lib/frr \
  --enable-user=frr \
  --enable-group=frr \
  --enable-vty-group=frrvty \
  --enable-multipath=64 \
  --enable-grpc \
  --enable-fpm \
  --disable-doc

make -j$(nproc)
sudo make install
```

### Build with Sanitizers (Development)

```bash
./configure \
  --prefix=/usr \
  --enable-address-sanitizer \
  --enable-undefined-sanitizer \
  --enable-dev-build
```

---

## Comparison: Upgrade Workflows

### Package Upgrade

```bash
# Backup config
cp -r /etc/frr /etc/frr.bak.$(date +%Y%m%d)

# Upgrade
apt update && apt upgrade frr

# Verify
systemctl status frr
vtysh -c "show version"
```

### Container Upgrade

```bash
# Pull new version
docker pull quay.io/frrouting/frr:10.6.0

# Stop old, start new (config mounted from host)
docker stop frr && docker rm frr
docker run -d --name frr --privileged --net=host \
  -v /etc/frr:/etc/frr \
  quay.io/frrouting/frr:10.6.0

# Rollback if needed
docker stop frr && docker rm frr
docker run -d --name frr --privileged --net=host \
  -v /etc/frr:/etc/frr \
  quay.io/frrouting/frr:10.5.3
```

### Source Upgrade

```bash
cd /path/to/frr
git fetch && git checkout frr-10.6.0

make clean
./configure [same flags as before]
make -j$(nproc)

systemctl stop frr
sudo make install
systemctl start frr
```

---

## Decision Flowchart

```
Start
  |
  v
Production environment? ──Yes──> Packages (deb/rpm)
  |                                |
  No                               Need custom compile flags? ──Yes──> Source build
  |
  v
Need multiple FRR versions simultaneously? ──Yes──> Containers
  |
  No
  |
  v
Lab or topology testing? ──Yes──> Containers + containerlab
  |
  No
  |
  v
CI/CD pipeline? ──Yes──> Containers (pinned tags)
  |
  No
  |
  v
Quick evaluation? ──Yes──> Snap or Docker
  |
  No
  |
  v
Default ──> Packages
```
