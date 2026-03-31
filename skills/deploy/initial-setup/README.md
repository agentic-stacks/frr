# Initial Setup After Installing FRR

## Overview

A fresh FRR installation starts no routing daemons by default. You must enable the daemons you need, configure kernel parameters for IP forwarding, start the FRR service, and verify everything is running. This skill covers all post-install steps regardless of whether FRR was installed from packages, containers, or source.

## Enable Daemons

Edit `/etc/frr/daemons` and change each daemon you need from `no` to `yes`:

```bash
sudo vi /etc/frr/daemons
```

The daemons file controls which processes start when FRR is launched:

```
zebra=yes
bgpd=yes
ospfd=yes
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
staticd=yes
pbrd=no
bfdd=yes
fabricd=no
vrrpd=no
pathd=no
```

> **Important:** `zebra` must always be `yes`. It manages the kernel routing table and is required by all other daemons. `staticd` should be `yes` if you use any static routes. `bfdd` is recommended for fast failure detection.

The file also contains daemon startup options:

```
vtysh_enable=yes
zebra_options=" -s 90000000 --daemon -A 127.0.0.1"
bgpd_options="   --daemon -A 127.0.0.1"
ospfd_options="  --daemon -A 127.0.0.1"
ospf6d_options=" --daemon -A ::1"
ripd_options="   --daemon -A 127.0.0.1"
ripngd_options=" --daemon -A ::1"
isisd_options="  --daemon -A 127.0.0.1"
pimd_options="  --daemon -A 127.0.0.1"
ldpd_options="  --daemon -A 127.0.0.1"
nhrpd_options="  --daemon -A 127.0.0.1"
eigrpd_options="  --daemon -A 127.0.0.1"
babeld_options="  --daemon -A 127.0.0.1"
sharpd_options="  --daemon -A 127.0.0.1"
staticd_options="  --daemon -A 127.0.0.1"
pbrd_options="  --daemon -A 127.0.0.1"
bfdd_options="  --daemon -A 127.0.0.1"
fabricd_options="  --daemon -A 127.0.0.1"
```

Key settings:

| Setting | Purpose |
|---|---|
| `vtysh_enable=yes` | Load config via `vtysh -b` on startup |
| `-s 90000000` | Zebra netlink buffer size (increase for large routing tables) |
| `--daemon` | Fork into background |
| `-A 127.0.0.1` | Bind VTY listener to localhost only |
| `MAX_FDS=1024` | Max open file descriptors per daemon (increase for large BGP deployments) |
| `FRR_NO_ROOT="yes"` | Run as non-root (container use only, most daemons need root capabilities) |

To apply the same option to every daemon, use `frr_global_options`:

```
frr_global_options="-w"
```

This example enables VRF-backed network namespaces on all daemons.

## Configure Kernel Parameters

### IP Forwarding (Required)

A router must forward packets. Enable IPv4 and IPv6 forwarding:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/90-routing-sysctl.conf
# Enable IP forwarding
net.ipv4.ip_forward=1
net.ipv4.conf.all.forwarding=1
net.ipv6.conf.all.forwarding=1
EOF

sudo sysctl -p /etc/sysctl.d/90-routing-sysctl.conf
```

### Reverse Path Filtering

Disable strict reverse-path filtering. Strict mode (rp_filter=1) drops packets arriving on unexpected interfaces, which breaks asymmetric routing and BGP unnumbered:

```bash
cat <<EOF | sudo tee -a /etc/sysctl.d/90-routing-sysctl.conf

# Disable strict reverse-path filtering
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.lo.rp_filter=0
EOF

sudo sysctl -p /etc/sysctl.d/90-routing-sysctl.conf
```

### MPLS Support

Load MPLS kernel modules and enable MPLS on interfaces:

```bash
# Load MPLS modules
cat <<EOF | sudo tee /etc/modules-load.d/mpls.conf
mpls_router
mpls_iptunnel
EOF

sudo modprobe mpls_router mpls_iptunnel

# Enable MPLS on specific interfaces
cat <<EOF | sudo tee -a /etc/sysctl.d/90-routing-sysctl.conf

# MPLS label processing
net.mpls.conf.eth0.input=1
net.mpls.conf.eth1.input=1
net.mpls.platform_labels=100000
EOF

sudo sysctl -p /etc/sysctl.d/90-routing-sysctl.conf
```

> **Important:** Add a `net.mpls.conf.<interface>.input=1` line for every interface that will process MPLS labels. MPLS requires kernel 4.1+ (basic) / 4.5+ (full support).

### VRF Support

VRF requires kernel 4.15+ (IPv4) / 5.0+ (IPv6). No special sysctl is needed -- VRF support is built into the kernel. Enable VRF-backed namespaces in FRR by adding `-w` to daemon options in `/etc/frr/daemons`:

```
frr_global_options="-w"
```

### Full Recommended Sysctl Configuration

This is the complete recommended sysctl file for an FRR router, drawn from the official FRR documentation:

```bash
cat <<'EOF' | sudo tee /etc/sysctl.d/99frr_defaults.conf
# /etc/sysctl.d/99frr_defaults.conf
# FRR recommended kernel parameters

# Enable IPv4/IPv6 forwarding
net.ipv4.ip_forward=1
net.ipv4.conf.all.forwarding=1
net.ipv4.conf.default.forwarding=1
net.ipv6.conf.all.forwarding=1

# Routing table sizes
net.ipv6.route.max_size=131072

# Ignore routes on down links
net.ipv4.conf.all.ignore_routes_with_linkdown=1
net.ipv6.conf.all.ignore_routes_with_linkdown=1

# Reverse-path filtering (disabled for routing)
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.lo.rp_filter=0

# ARP settings for BGP unnumbered / OSPF
net.ipv4.conf.default.arp_announce=2
net.ipv4.conf.default.arp_notify=1
net.ipv4.conf.default.arp_ignore=1
net.ipv4.conf.all.arp_announce=2
net.ipv4.conf.all.arp_notify=1
net.ipv4.conf.all.arp_ignore=1
net.ipv4.icmp_errors_use_inbound_ifaddr=1

# Keep IPv6 addresses on admin down
net.ipv6.conf.all.keep_addr_on_down=1
net.ipv6.route.skip_notify_on_dev_down=1

# IGMP / MLD
net.ipv4.igmp_max_memberships=1000
net.ipv4.neigh.default.mcast_solicit=10
net.ipv6.mld_max_msf=512

# Neighbor table garbage collection (scale for large networks)
net.ipv4.neigh.default.gc_thresh2=7168
net.ipv4.neigh.default.gc_thresh3=8192
net.ipv4.neigh.default.base_reachable_time_ms=14400000
net.ipv6.neigh.default.gc_thresh2=3584
net.ipv6.neigh.default.gc_thresh3=4096
net.ipv6.neigh.default.base_reachable_time_ms=14400000

# Use neighbor info for multipath nexthop selection
net.ipv4.fib_multipath_use_neigh=1
EOF

sudo sysctl -p /etc/sysctl.d/99frr_defaults.conf
```

## Start FRR

```bash
sudo systemctl start frr
sudo systemctl enable frr
```

## Verify Daemons Are Running

### Check systemd Status

```bash
systemctl status frr
```

Expected output shows `active (running)` with the list of started daemons.

### Check Running Daemons via vtysh

```bash
vtysh -c "show daemons"
```

Expected output:

```
 watchfrr zebra bgpd staticd bfdd
```

Only the daemons you enabled in `/etc/frr/daemons` should appear.

### Check Individual Daemon Status

```bash
# Zebra
vtysh -c "show zebra"

# BGP
vtysh -c "show bgp summary"

# OSPF
vtysh -c "show ip ospf"

# Check all interfaces
vtysh -c "show interface brief"

# Check the routing table
vtysh -c "show ip route"
```

## Basic Zebra Configuration

After FRR is running, configure interfaces and basic routing in vtysh:

```bash
vtysh
```

```
configure terminal
!
! Set the hostname
hostname router1
!
! Configure a loopback address (used as router-id)
interface lo
 ip address 10.255.0.1/32
exit
!
! Configure a physical interface
interface eth0
 ip address 192.168.1.1/24
 no shutdown
exit
!
! Add a static default route
ip route 0.0.0.0/0 192.168.1.254
!
end
!
! Save configuration
write memory
```

Verify:

```bash
vtysh -c "show ip route"
vtysh -c "show interface brief"
```

## Firewall Considerations

FRR routing protocols use specific ports and protocols. If a firewall is active, open these as needed:

| Protocol | Port/Proto | Direction | Purpose |
|---|---|---|---|
| BGP | TCP 179 | In + Out | BGP peering |
| OSPF | IP protocol 89 | In + Out | OSPF hello/LSA (multicast 224.0.0.5, 224.0.0.6) |
| OSPFv3 | IP protocol 89 | In + Out | OSPFv3 over IPv6 (multicast ff02::5, ff02::6) |
| IS-IS | Ethernet (no IP) | In + Out | IS-IS runs directly on L2, not IP |
| RIP | UDP 520 | In + Out | RIPv2 (multicast 224.0.0.9) |
| RIPng | UDP 521 | In + Out | RIPng over IPv6 |
| LDP | TCP 646, UDP 646 | In + Out | LDP discovery and sessions |
| BFD | UDP 3784, 3785 | In + Out | BFD control and echo |
| PIM | IP protocol 103 | In + Out | PIM multicast (224.0.0.13) |
| VRRP | IP protocol 112 | In + Out | VRRP advertisements (224.0.0.18) |
| NHRP | GRE (IP protocol 47) | In + Out | NHRP over GRE tunnels |
| EIGRP | IP protocol 88 | In + Out | EIGRP (multicast 224.0.0.10) |
| Babel | UDP 6696 | In + Out | Babel routing |

### Example: Allow BGP and OSPF with firewalld

```bash
sudo firewall-cmd --permanent --add-port=179/tcp
sudo firewall-cmd --permanent --add-protocol=89
sudo firewall-cmd --reload
```

### Example: Allow BGP and OSPF with iptables

```bash
sudo iptables -A INPUT -p tcp --dport 179 -j ACCEPT
sudo iptables -A INPUT -p 89 -j ACCEPT
sudo iptables -A OUTPUT -p tcp --dport 179 -j ACCEPT
sudo iptables -A OUTPUT -p 89 -j ACCEPT
```

> **Note:** On Fedora, `firewalld` is enabled by default and may block routing protocol traffic. Either configure firewalld rules or disable it if the host is a dedicated router in a trusted network.

## Services File

Add FRR daemon ports to `/etc/services` for reference (may already be present from package install):

```
zebrasrv      2600/tcp          # zebra service
zebra         2601/tcp          # zebra vty
ripd          2602/tcp          # RIPd vty
ripngd        2603/tcp          # RIPngd vty
ospfd         2604/tcp          # OSPFd vty
bgpd          2605/tcp          # BGPd vty
ospf6d        2606/tcp          # OSPF6d vty
ospfapi       2607/tcp          # ospfapi
isisd         2608/tcp          # ISISd vty
babeld        2609/tcp          # BABELd vty
nhrpd         2610/tcp          # nhrpd vty
pimd          2611/tcp          # PIMd vty
ldpd          2612/tcp          # LDPd vty
eigrpd        2613/tcp          # EIGRPd vty
bfdd          2617/tcp          # bfdd vty
fabricd       2618/tcp          # fabricd vty
vrrpd         2619/tcp          # vrrpd vty
```

These ports are used for direct VTY access to individual daemons (bypassing vtysh). They are not required for normal operation but useful for debugging.

## Troubleshooting First Start

| Symptom | Cause | Fix |
|---|---|---|
| `systemctl start frr` fails | No daemons enabled in `/etc/frr/daemons` | Enable at least `zebra=yes` |
| `vtysh: error while loading shared libraries` | Library path not configured | Run `sudo ldconfig` or add `/usr/local/lib` to `/etc/ld.so.conf` |
| `vtysh` hangs or times out | Daemon not running or socket permission issue | Check `systemctl status frr` and verify `/var/run/frr/` permissions |
| Daemons start but no routes | IP forwarding disabled | Set `net.ipv4.ip_forward=1` via sysctl |
| MPLS labels not working | Kernel modules not loaded | Run `sudo modprobe mpls_router mpls_iptunnel` |
| BGP peer connection refused | Firewall blocking TCP 179 | Open port 179 in firewalld/iptables |
| OSPF neighbors stuck in Init | Firewall blocking protocol 89 | Allow IP protocol 89 |
| Permission denied on `/etc/frr/` | Wrong ownership | Run `sudo chown -R frr:frr /etc/frr/` |

## Next Step

FRR is now running. Proceed to the protocol-specific skills under `skills/protocols/` to configure routing:

- `skills/protocols/bgp` -- BGP configuration
- `skills/protocols/ospf` -- OSPF configuration
- `skills/protocols/isis` -- IS-IS configuration
