# BGP EVPN

## Overview

EVPN (Ethernet VPN) uses the BGP L2VPN EVPN address family to provide MAC/IP advertisement and VXLAN-based overlay networking. It is the standard control plane for VXLAN fabrics in data center environments. FRR implements EVPN for both Layer 2 (bridging) and Layer 3 (routing) overlays.

Key concepts:

- **VNI (VXLAN Network Identifier):** The overlay network ID, analogous to a VLAN
- **L2VNI:** Maps to a MAC-VRF (bridging domain)
- **L3VNI:** Maps to an IP-VRF (routing table) for inter-VXLAN routing
- **VTEP:** VXLAN Tunnel Endpoint -- the router/switch terminating VXLAN tunnels
- **Route types:** EVPN defines 5 route types for different functions

## EVPN Route Types

| Type | Name | Purpose |
|---|---|---|
| Type-1 | Ethernet Auto-Discovery | Multi-homing, fast convergence |
| Type-2 | MAC/IP Advertisement | MAC and MAC+IP reachability |
| Type-3 | Inclusive Multicast | BUM traffic handling, VTEP discovery |
| Type-4 | Ethernet Segment | Multi-homing designated forwarder election |
| Type-5 | IP Prefix | Inter-subnet routing (L3VNI) |

## Prerequisites

### Linux Kernel Configuration

EVPN requires specific Linux interface settings:

```bash
# Disable MAC learning on VXLAN interfaces (BGP handles this)
bridge link set dev vxlan100 learning off

# Enable neighbor suppression
bridge link set dev vxlan100 neigh_suppress on
```

### Enable bgpd

```
# /etc/frr/daemons
bgpd=yes
zebra=yes
```

## Configure Basic EVPN Fabric

### Step 1: Configure VXLAN Interfaces (Linux)

```bash
# Create VXLAN interface for L2VNI 100
ip link add vxlan100 type vxlan id 100 local 10.0.0.1 dstport 4789 nolearning

# Add to bridge
ip link set vxlan100 master br0
ip link set vxlan100 up

# Disable learning
bridge link set dev vxlan100 learning off
bridge link set dev vxlan100 neigh_suppress on
```

### Step 2: Configure BGP EVPN

```
router bgp 65001
 bgp router-id 10.0.0.1
 no bgp default ipv4-unicast

 ! Peer with spine (route reflector)
 neighbor 10.0.0.100 remote-as 65001
 neighbor 10.0.0.100 update-source lo

 address-family l2vpn evpn
  neighbor 10.0.0.100 activate
  neighbor 10.0.0.100 send-community both
  advertise-all-vni
 exit-address-family
```

Replace:
- `10.0.0.1` -- local loopback/VTEP IP
- `10.0.0.100` -- spine/RR loopback IP
- `65001` -- AS number

### Step 3: Configure Spine as Route Reflector

```
router bgp 65001
 bgp router-id 10.0.0.100
 no bgp default ipv4-unicast

 neighbor LEAF peer-group
 neighbor LEAF remote-as 65001
 neighbor LEAF update-source lo

 neighbor 10.0.0.1 peer-group LEAF
 neighbor 10.0.0.2 peer-group LEAF
 neighbor 10.0.0.3 peer-group LEAF

 address-family l2vpn evpn
  neighbor LEAF activate
  neighbor LEAF send-community both
  neighbor LEAF route-reflector-client
 exit-address-family
```

## Configure L3VNI (Inter-VXLAN Routing)

L3VNI enables routing between VNIs using a symmetric IRB model.

### Step 1: Create L3VNI VXLAN and VRF (Linux)

```bash
# Create VRF
ip link add TENANT-A type vrf table 100
ip link set TENANT-A up

# Create L3VNI VXLAN
ip link add vxlan5000 type vxlan id 5000 local 10.0.0.1 dstport 4789 nolearning
ip link set vxlan5000 master br0
ip link set vxlan5000 up
bridge link set dev vxlan5000 learning off

# Create SVI for L3VNI (no IP address needed)
ip link add link br0 name vlan5000 type vlan id 5000
ip link set vlan5000 master TENANT-A
ip link set vlan5000 up
ip link set vlan5000 addrgenmode none
```

### Step 2: Associate VRF with L3VNI

```
vrf TENANT-A
 vni 5000
exit-vrf
```

### Step 3: Configure BGP for the VRF

```
router bgp 65001 vrf TENANT-A
 address-family ipv4 unicast
  redistribute connected
  redistribute static
 exit-address-family

 address-family l2vpn evpn
  advertise ipv4 unicast
 exit-address-family
```

### Step 4: Advertise SVI IP (for Type-2 Routes)

```
router bgp 65001
 address-family l2vpn evpn
  advertise-svi-ip
 exit-address-family
```

## Configure eBGP EVPN Fabric

For leaf-spine fabrics using eBGP (each device has a unique AS):

### Leaf Configuration

```
router bgp 65101
 bgp router-id 10.0.0.1
 no bgp default ipv4-unicast

 neighbor eth0 interface remote-as external
 neighbor eth1 interface remote-as external

 address-family l2vpn evpn
  neighbor eth0 activate
  neighbor eth1 activate
  advertise-all-vni
 exit-address-family
```

### Spine Configuration

```
router bgp 65100
 bgp router-id 10.0.0.100
 no bgp default ipv4-unicast

 neighbor eth0 interface remote-as external
 neighbor eth1 interface remote-as external
 neighbor eth2 interface remote-as external

 address-family l2vpn evpn
  neighbor eth0 activate
  neighbor eth1 activate
  neighbor eth2 activate
 exit-address-family
```

## Advertise IP Prefixes as Type-5 Routes

Export IPv4/IPv6 prefixes into EVPN as Type-5 routes:

```
router bgp 65001 vrf TENANT-A
 address-family l2vpn evpn
  advertise ipv4 unicast
  advertise ipv6 unicast
 exit-address-family
```

Optionally apply a route map to filter which prefixes are advertised:

```
router bgp 65001 vrf TENANT-A
 address-family l2vpn evpn
  advertise ipv4 unicast route-map EVPN-EXPORT
 exit-address-family
```

## Configure Route Target Import/Export

By default, FRR auto-derives route targets. For manual control:

```
router bgp 65001
 address-family l2vpn evpn
  vni 100
   rd 10.0.0.1:100
   route-target import 65001:100
   route-target export 65001:100
  exit-vni
 exit-address-family
```

## Configure Duplicate Address Detection

FRR detects duplicate MAC/IP addresses in EVPN:

```
router bgp 65001
 address-family l2vpn evpn
  dup-addr-detection
  dup-addr-detection max-moves 5 time 180
  dup-addr-detection freeze permanent
 exit-address-family
```

| Parameter | Default | Purpose |
|---|---|---|
| `max-moves` | 5 | Number of moves before declaring duplicate |
| `time` | 180s | Window for counting moves |
| `freeze permanent` | -- | Permanently freeze duplicate entries |
| `freeze <seconds>` | -- | Temporarily freeze duplicates |

## Single VXLAN Device (SVD)

FRR supports a single VXLAN device carrying multiple VNIs via VLAN-aware bridges:

```bash
# Create single VXLAN device
ip link add vxlan0 type vxlan external dstport 4789 nolearning

# Create VLAN-aware bridge
ip link add br0 type bridge vlan_filtering 1
ip link set vxlan0 master br0
bridge link set dev vxlan0 vlan_tunnel on
ip link set br0 up
ip link set vxlan0 up

# Map VLANs to VNIs
bridge vlan add vid 100 dev vxlan0 tunnel_info id 100
bridge vlan add vid 200 dev vxlan0 tunnel_info id 200
```

## Show EVPN Information

```
! EVPN summary
show bgp l2vpn evpn summary

! All EVPN routes
show bgp l2vpn evpn

! EVPN routes by type
show bgp l2vpn evpn route type macip
show bgp l2vpn evpn route type multicast
show bgp l2vpn evpn route type prefix

! Routes for a specific VNI
show bgp l2vpn evpn route vni 100

! MAC table per VNI
show evpn mac vni 100
show evpn mac vni 100 detail
show evpn mac vni all

! ARP/ND table per VNI
show evpn arp-cache vni 100
show evpn arp-cache vni all

! VNI information
show evpn vni
show evpn vni 100

! VRF to L3VNI mapping
show vrf vni

! VTEP peers
show evpn next-hops vni all
```

## Troubleshoot EVPN

### Common Issues

| Symptom | Likely Cause | Fix |
|---|---|---|
| No Type-2 routes | `advertise-all-vni` missing | Add under `address-family l2vpn evpn` |
| No Type-5 routes | `advertise ipv4 unicast` missing in VRF | Add under VRF's `address-family l2vpn evpn` |
| MAC not learned remotely | VXLAN learning not disabled | Set `learning off` on bridge port |
| L3VNI not working | VRF-VNI mapping missing | Add `vni <id>` under `vrf` block |
| Duplicate address detected | Host moving between VTEPs | Check for loops, verify dup-addr-detection settings |
| Routes not reflected | `send-community both` missing | Extended communities required for EVPN RT |

### Debug EVPN

```
debug bgp updates in
debug bgp updates out
debug bgp zebra
debug zebra vxlan
debug zebra evpn-mh
```

### Verify End-to-End

```
! 1. Check BGP EVPN session is up
show bgp l2vpn evpn summary

! 2. Verify VNIs are learned
show evpn vni

! 3. Check local MACs are advertised
show evpn mac vni 100

! 4. Verify remote MACs are learned
show evpn mac vni 100 | include remote

! 5. Check VTEP peers
show evpn next-hops vni 100

! 6. Verify routes in the VRF
show ip route vrf TENANT-A
```
