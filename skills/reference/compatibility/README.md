# Compatibility Reference

Kernel requirements, library dependencies, distribution support, and platform notes for FRR.

---

## Kernel Version Requirements

| Feature | Minimum Kernel | Notes |
|---|---|---|
| Basic routing (Netlink) | 2.6.32+ | Netlink interface available since 2.0; practical minimum is 2.6.32 |
| VRF (l3mdev) | 4.3+ | VRF device support; 4.8+ for full FRR VRF support |
| VRF route leaking | 4.14+ | Required for `import vrf` functionality |
| MPLS forwarding | 4.1+ | Basic MPLS label switching |
| MPLS LDP | 4.5+ | Full MPLS LDP support with label allocation |
| MPLS SR (Segment Routing) | 4.10+ | Segment routing label operations |
| SRv6 | 4.14+ | Basic SRv6; 5.14+ recommended for full functionality |
| VXLAN | 3.12+ | Basic VXLAN; 4.11+ for EVPN integration |
| PBR (Policy-Based Routing) | 4.14+ | FIB rules and ip-rule support |
| BPF dataplane | 5.2+ | eBPF-based dataplane acceleration |
| Nexthop groups | 5.3+ | Kernel nexthop objects for ECMP |
| Nexthop group resilient hashing | 5.19+ | Resilient ECMP hashing |
| 16-bit nexthop weights | 6.12+ | Extended ECMP weight precision |
| SRv6 End.DT46 | 5.14+ | Dual-stack decapsulation |

### Verify Kernel Version

```bash
uname -r
```

---

## Required Kernel Modules

Load these modules for the corresponding features. Add to `/etc/modules-load.d/frr.conf` for persistence.

### MPLS

```bash
modprobe mpls_router
modprobe mpls_iptunnel
modprobe mpls_gso

# Enable MPLS on interfaces
sysctl -w net.mpls.conf.eth0.input=1
sysctl -w net.mpls.platform_labels=1048575
```

| Module | Purpose |
|---|---|
| `mpls_router` | Core MPLS forwarding |
| `mpls_iptunnel` | MPLS tunnel encapsulation |
| `mpls_gso` | MPLS generic segmentation offload |

### VRF

```bash
modprobe vrf

# Enable VRF strict mode (recommended)
sysctl -w net.vrf.strict_mode=1
```

| Module | Purpose |
|---|---|
| `vrf` | L3 VRF device support |

### Netfilter (for PBR)

```bash
modprobe nf_tables
modprobe nft_fib_inet
```

### VXLAN

```bash
modprobe vxlan
modprobe udp_tunnel
```

### Persistent Module Loading

```bash
cat > /etc/modules-load.d/frr.conf << 'EOF'
# FRR required kernel modules
mpls_router
mpls_iptunnel
vrf
vxlan
EOF
```

### Persistent sysctl Settings

```bash
cat > /etc/sysctl.d/90-frr.conf << 'EOF'
# MPLS
net.mpls.platform_labels=1048575

# VRF
net.vrf.strict_mode=1

# IPv4/IPv6 forwarding
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1

# Disable RP filter for MPLS/VRF
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
EOF

sysctl -p /etc/sysctl.d/90-frr.conf
```

---

## libyang Compatibility

FRR uses libyang for YANG data model processing. Version compatibility is critical.

| FRR Version | libyang Minimum | libyang Recommended | Notes |
|---|---|---|---|
| 8.x | 2.0.0 | 2.1.80 | |
| 9.0 | 2.0.0 | 2.1.80 | 2.1.111 breaks prefix-list matching |
| 9.1 | 2.0.0 | 2.1.80 | 2.1.111 breaks prefix-list matching |
| 10.0 | 2.1.128 | 2.1.128+ | Minimum bumped; older versions rejected |
| 10.1–10.5 | 2.1.128 | 2.1.148+ | |
| 10.6 | 2.1.128 / 3.0+ | 3.x | Binary packages now built against libyang3 |

### Known libyang Issues

| libyang Version | Issue | Impact |
|---|---|---|
| 2.1.111 | Prefix-list matching broken | Route-map prefix-list matches silently fail |
| 2.1.112–2.1.127 | Various regressions | Not recommended |

### Check libyang Version

```bash
# Installed version
pkg-config --modversion libyang
# or
dpkg -l libyang2 2>/dev/null || rpm -q libyang

# Version FRR was compiled against
vtysh -c "show version" | grep libyang
```

---

## Distribution Support Matrix

| Distribution | FRR Packages | Supported Versions | Package Source |
|---|---|---|---|
| Debian | Official + FRR repo | Bullseye (11), Bookworm (12), Trixie (13) | deb.frrouting.org |
| Ubuntu | Official + FRR repo | 20.04, 22.04, 24.04 | deb.frrouting.org |
| RHEL / Rocky / Alma | FRR repo | 8, 9 | rpm.frrouting.org |
| Fedora | Official + FRR repo | 38, 39, 40 | rpm.frrouting.org |
| Alpine Linux | Docker only | 3.18, 3.19, 3.20 | quay.io/frrouting/frr |
| openSUSE | Community | Leap 15.5+ | OBS |
| Snap | snapcraft.io | Any snapd-capable | snapcraft.io/frr |
| Docker | Official | Any Docker host | quay.io/frrouting/frr |

### Package Sources

```bash
# Debian/Ubuntu — add FRR repository
curl -s https://deb.frrouting.org/frr/keys.gpg | sudo tee /usr/share/keyrings/frrouting.gpg > /dev/null
FRRVER="frr-stable"
echo "deb [signed-by=/usr/share/keyrings/frrouting.gpg] https://deb.frrouting.org/frr $(lsb_release -s -c) $FRRVER" | \
  sudo tee /etc/apt/sources.list.d/frr.list

# RHEL/Rocky — add FRR repository
FRRVER="frr-stable"
curl -O https://rpm.frrouting.org/repo/$FRRVER-repo-1-0.el$(rpm -E %{rhel}).noarch.rpm
sudo dnf install ./$FRRVER-repo-1-0.el$(rpm -E %{rhel}).noarch.rpm
```

---

## Build Dependencies

| Dependency | Purpose | Minimum Version |
|---|---|---|
| libyang | YANG models | See table above |
| protobuf-c | FPM protobuf | 1.0+ |
| libcap | Capabilities | 2.0+ |
| libjson-c | JSON output | 0.13+ |
| libreadline | CLI | 6.0+ |
| c-ares | DNS resolution | 1.10+ |
| libpcre2 | Regex | 10.0+ |
| libelf | BPF support | 0.170+ |
| grpc | gRPC Northbound | 1.16+ (optional) |
| libsnmp | SNMP AgentX | 5.8+ (optional) |
| liblttng-ust | LTTng tracing | 2.0+ (optional) |

### Install Build Dependencies

```bash
# Debian/Ubuntu
sudo apt install -y git autoconf automake libtool make \
  libreadline-dev texinfo libjson-c-dev pkg-config bison flex \
  libyang2-dev libcap-dev libelf-dev libprotobuf-c-dev protobuf-c-compiler \
  python3-dev python3-sphinx libpcre2-dev libsqlite3-dev

# RHEL/Rocky
sudo dnf install -y git autoconf automake libtool make \
  readline-devel texinfo json-c-devel pkgconfig bison flex \
  libyang2-devel libcap-devel elfutils-libelf-devel protobuf-c-devel \
  python3-devel python3-sphinx pcre2-devel sqlite-devel
```

---

## Hardware and Platform Notes

### CPU Architecture Support

| Architecture | Status | Notes |
|---|---|---|
| x86_64 / amd64 | Full support | Primary development/test platform |
| aarch64 / arm64 | Full support | Tested on AWS Graviton, Ampere |
| armhf / armv7 | Community | Works but not primary test target |
| ppc64le | Community | Basic testing only |
| s390x | Community | Basic testing only |
| MIPS | Unsupported | May work from source |

### Virtualization

| Platform | Status | Notes |
|---|---|---|
| KVM/QEMU | Full support | Recommended for labs and production |
| VMware ESXi | Full support | Use VMXNET3 for best performance |
| Hyper-V | Works | Verify synthetic NIC driver support |
| Docker/Podman | Full support | Official images available |
| LXC/LXD | Full support | Requires `--privileged` or specific capabilities |
| VirtualBox | Lab only | Performance limitations |
| WSL2 | Lab only | Limited kernel module support |

### Network Hardware Offload

| Feature | Requirement |
|---|---|
| ECMP hashing | NIC RSS support recommended |
| MPLS | No hardware offload; CPU-based forwarding |
| VXLAN offload | Mellanox ConnectX-5+, Intel E810 |
| PBR | Uses kernel FIB rules; no special hardware |

### Memory Guidelines

| Deployment Scale | Recommended RAM |
|---|---|
| Small office (< 1K routes) | 512 MB |
| Enterprise (10K–100K routes) | 2–4 GB |
| ISP/transit (full table, ~1M routes) | 8–16 GB |
| Route reflector (multiple full tables) | 16–32 GB |
| EVPN at scale (100K+ MAC/IP) | 8–16 GB |

### File Descriptor Limits

For large-scale deployments, increase file descriptor limits:

```bash
cat > /etc/security/limits.d/frr.conf << 'EOF'
frr soft nofile 65536
frr hard nofile 65536
EOF

# Or in systemd override
sudo systemctl edit frr
# Add:
# [Service]
# LimitNOFILE=65536
```
