# NHRP Protocol Skill

## Overview

NHRP (Next Hop Resolution Protocol) enables dynamic spoke-to-spoke tunnels over NBMA (Non-Broadcast Multi-Access) networks in FRR, implemented by the `nhrpd` daemon. NHRP is the foundation of Cisco DMVPN (Dynamic Multipoint VPN) and is defined in RFC 2332. Spokes register with a hub, and when spoke-to-spoke traffic is detected, NHRP resolves the NBMA address to establish direct tunnels.

Key facts:

- Daemon: `nhrpd` (enable in `/etc/frr/daemons`)
- Status: **alpha quality** -- use with caution in production
- Architecture: hub-spoke with dynamic spoke-to-spoke shortcuts
- Tunnel type: GRE/mGRE over IPsec (typically)
- Addressing: per-host /32 prefixes (similar to Cisco FlexVPN)
- Requires: GRE tunnel interfaces, routing protocol (BGP recommended), IPsec (strongSwan)
- Does NOT handle prefix routing -- deploy BGP or another IGP alongside

> **Warning:** The nhrpd implementation is alpha quality in FRR. Test thoroughly in a lab before any production deployment. Functionality and CLI may change between FRR releases.

## Architecture: DMVPN with NHRP

```
                    ┌─────────┐
                    │   Hub   │ NHS (Next Hop Server)
                    │ 10.0.0.1│
                    └────┬────┘
                    GRE/IPsec
              ┌──────────┼──────────┐
              │          │          │
         ┌────┴────┐┌───┴────┐┌───┴────┐
         │ Spoke A ││Spoke B ││Spoke C │ NHC (Next Hop Client)
         │10.0.0.2 ││10.0.0.3││10.0.0.4│
         └─────────┘└────────┘└────────┘
              │                    │
              └────── shortcut ────┘  (dynamic spoke-to-spoke)
```

1. Spokes register their NBMA (underlay) addresses with the hub
2. All traffic initially flows through the hub
3. Hub sends NHRP Traffic Indication when it forwards spoke-to-spoke traffic
4. Spokes resolve each other's NBMA addresses via NHRP
5. Direct GRE/IPsec tunnel is established between spokes (shortcut)
6. Shortcut expires when traffic stops

## Prerequisites

### GRE Tunnel Setup

Create the GRE tunnel interface on each router using Linux commands:

```bash
# Create GRE tunnel with a shared key
ip tunnel add gre1 mode gre key 42 ttl 64
ip addr add 10.255.255.1/32 dev gre1
ip link set gre1 up
```

- Use a `/32` address on the GRE interface; nhrpd creates additional host routes dynamically
- The `key` must match on all DMVPN routers
- Set `ttl 64` (or appropriate value) to prevent TTL expiry in nested tunnels

> **Warning:** Do not assign a subnet mask (e.g., /24) to the GRE interface. NHRP manages peer reachability via dynamic /32 host routes. A subnet mask causes routing conflicts.

### IPsec / strongSwan

NHRP typically runs over IPsec. nhrpd integrates with strongSwan via the VICI protocol (UNIX socket at `/var/run/charon.vici`).

strongSwan configuration example (`/etc/ipsec.conf`):

```
conn dmvpn
  authby=secret
  auto=add
  keyexchange=ikev2
  ike=aes256-sha256-modp2048
  esp=aes256-sha256
  type=transport
  left=%defaultroute
  leftprotoport=gre
  right=%any
  rightprotoport=gre
```

`/etc/ipsec.secrets`:

```
%any : PSK "DMVPN-SECRET-KEY"
```

> **Note:** strongSwan requires custom patches for full NHRP/DMVPN integration. Alpine Linux packages include these patches. On other distributions, check strongSwan compatibility.

## Enable the NHRP Daemon

Edit `/etc/frr/daemons`:

```
nhrpd=yes
```

Reload FRR:

```
sudo systemctl reload frr
```

## Core NHRP Commands

### Interface-Level Commands

| Command | Description |
|---|---|
| `ip nhrp network-id (1-4294967295)` | Enable NHRP on interface; local domain identifier |
| `ip nhrp holdtime (1-65000)` | Registration interval and address validity (seconds) |
| `ip nhrp authentication PASSWORD` | Cisco-compatible plaintext auth (max 8 chars) |
| `ip nhrp map A.B.C.D A.B.C.D` | Static NHRP mapping (overlay IP to NBMA IP) |
| `ip nhrp map A.B.C.D local` | Map overlay IP to local NBMA address |
| `ip nhrp nhs A.B.C.D nbma A.B.C.D` | Configure static NHS (hub) address |
| `ip nhrp nhs dynamic nbma A.B.C.D` | Configure dynamic NHS address |
| `ip nhrp shortcut` | Enable spoke-to-spoke shortcut tunnels |
| `ip nhrp redirect` | Send redirect replies for shortcut establishment |
| `ip nhrp registration no-unique` | Allow dynamic IP clients to register |
| `ip nhrp mtu VALUE` | Advertise MTU to peers |

### Global Commands

| Command | Description |
|---|---|
| `nhrp nflog-group (1-65535)` | Kernel NFLOG group for traffic indication |
| `nhrp multicast-nflog-group (1-65535)` | NFLOG group for multicast forwarding |
| `nhrp event socket PATH` | Unix socket path for external event listeners |

## Hub Configuration

The hub (NHS) serves as the registration point for all spokes and forwards initial spoke-to-spoke traffic.

### Hub FRR Configuration

```
interface gre1
 ip address 10.255.255.1/32
 ip nhrp network-id 1
 ip nhrp holdtime 300
 ip nhrp authentication DMVPN1
 ip nhrp redirect
 ip nhrp map multicast dynamic
 tunnel source eth0
```

### Hub iptables Rules (Traffic Indication)

The hub must use iptables NFLOG to detect spoke-to-spoke traffic and trigger NHRP Traffic Indication messages:

```bash
iptables -A FORWARD -i gre1 -o gre1 \
  -m hashlimit --hashlimit-upto 4/minute --hashlimit-burst 1 \
  --hashlimit-mode srcip,dstip --hashlimit-srcmask 24 --hashlimit-dstmask 24 \
  --hashlimit-name loglimit-0 \
  -j NFLOG --nflog-group 1 --nflog-range 128
```

Configure FRR to read from the NFLOG group:

```
nhrp nflog-group 1
```

> **Warning:** Without the NFLOG iptables rules, the hub cannot detect spoke-to-spoke traffic and will not send Traffic Indication messages. Shortcut tunnels will never form.

### Hub BGP Configuration

The hub must redistribute NHRP routes so spokes learn each other's prefixes:

```
router bgp 65000
 bgp router-id 10.255.255.1
 address-family ipv4 unicast
  redistribute nhrp
  network 172.16.0.0/16
 exit-address-family
```

## Spoke Configuration

Spokes (NHC) register with the hub and optionally enable shortcut switching for direct spoke-to-spoke tunnels.

### Spoke FRR Configuration

```
interface gre1
 ip address 10.255.255.2/32
 ip nhrp network-id 1
 ip nhrp holdtime 300
 ip nhrp authentication DMVPN1
 ip nhrp nhs 10.255.255.1 nbma 203.0.113.1
 ip nhrp shortcut
 ip nhrp map multicast 203.0.113.1
 tunnel source eth0
```

Replace:
- `10.255.255.2` -- this spoke's overlay IP
- `10.255.255.1` -- the hub's overlay IP
- `203.0.113.1` -- the hub's NBMA (underlay/public) IP

### Spoke BGP Configuration

```
router bgp 65001
 bgp router-id 10.255.255.2
 neighbor 10.255.255.1 remote-as 65000
 address-family ipv4 unicast
  network 192.168.1.0/24
 exit-address-family
```

## Shortcut Switching

When both `ip nhrp redirect` (on hub) and `ip nhrp shortcut` (on spokes) are enabled:

1. Spoke A sends traffic to Spoke B through the hub
2. Hub detects this via NFLOG and sends NHRP Traffic Indication to Spoke A
3. Spoke A sends NHRP Resolution Request to Spoke B (via hub)
4. Spoke B replies with its NBMA address
5. Spoke A installs a /32 shortcut route directly to Spoke B
6. Subsequent traffic flows directly between spokes

The shortcut route expires based on the `holdtime` value. It is refreshed while traffic flows.

## Multicast Support

For routing protocols that use multicast (OSPF, RIP), configure multicast forwarding:

### Hub Multicast

```
interface gre1
 ip nhrp map multicast dynamic
```

This dynamically maps multicast destinations to registered spoke NBMA addresses.

### Spoke Multicast

```
interface gre1
 ip nhrp map multicast 203.0.113.1
```

Statically maps multicast to the hub's NBMA address.

### Multicast NFLOG (Hub)

```bash
iptables -A OUTPUT -d 224.0.0.0/24 -o gre1 -j NFLOG --nflog-group 2
iptables -A OUTPUT -d 224.0.0.0/24 -o gre1 -j DROP
```

```
nhrp multicast-nflog-group 2
```

## Show Commands

### Show NHRP Cache

```
show ip nhrp cache
show ipv6 nhrp cache
```

Displays registered peers, their overlay and NBMA addresses, and cache entry state.

### Show NHRP in OpenNHRP Format

```
show ip nhrp opennhrp
```

### Show NHS Status

```
show ip nhrp nhs
```

Shows configured NHS servers and registration status.

### Show DMVPN Status

```
show dmvpn
```

Displays IPsec/IKE security associations and tunnel status.

### JSON Output

All show commands support `json` suffix:

```
show ip nhrp cache json
show ip nhrp nhs json
show dmvpn json
```

## Debug Commands

| Command | What It Shows |
|---|---|
| `debug nhrp all` | All NHRP events |
| `debug nhrp common` | Common protocol events |
| `debug nhrp kernel` | Kernel/netlink interaction |
| `debug nhrp route` | Route installation/removal |
| `debug nhrp vici` | strongSwan VICI protocol interaction |

## Complete Configuration Example

Three-node DMVPN: one hub, two spokes.

### Linux Setup (All Nodes)

```bash
# Hub (NBMA: 203.0.113.1)
ip tunnel add gre1 mode gre key 42 ttl 64
ip addr add 10.255.255.1/32 dev gre1
ip link set gre1 up

# Spoke A (NBMA: 198.51.100.2)
ip tunnel add gre1 mode gre key 42 ttl 64
ip addr add 10.255.255.2/32 dev gre1
ip link set gre1 up

# Spoke B (NBMA: 198.51.100.3)
ip tunnel add gre1 mode gre key 42 ttl 64
ip addr add 10.255.255.3/32 dev gre1
ip link set gre1 up
```

### Hub FRR Configuration

```
! /etc/frr/frr.conf -- Hub
!
nhrp nflog-group 1
!
interface gre1
 ip address 10.255.255.1/32
 ip nhrp network-id 1
 ip nhrp holdtime 300
 ip nhrp authentication DMVPN1
 ip nhrp redirect
 ip nhrp map multicast dynamic
!
router bgp 65000
 bgp router-id 10.255.255.1
 neighbor SPOKES peer-group
 neighbor SPOKES remote-as external
 bgp listen range 10.255.255.0/24 peer-group SPOKES
 address-family ipv4 unicast
  redistribute nhrp
  network 172.16.0.0/16
 exit-address-family
```

Hub iptables:

```bash
iptables -A FORWARD -i gre1 -o gre1 \
  -m hashlimit --hashlimit-upto 4/minute --hashlimit-burst 1 \
  --hashlimit-mode srcip,dstip --hashlimit-srcmask 24 --hashlimit-dstmask 24 \
  --hashlimit-name loglimit-0 \
  -j NFLOG --nflog-group 1 --nflog-range 128
```

### Spoke A FRR Configuration

```
! /etc/frr/frr.conf -- Spoke A
!
interface gre1
 ip address 10.255.255.2/32
 ip nhrp network-id 1
 ip nhrp holdtime 300
 ip nhrp authentication DMVPN1
 ip nhrp nhs 10.255.255.1 nbma 203.0.113.1
 ip nhrp shortcut
 ip nhrp map multicast 203.0.113.1
!
router bgp 65001
 bgp router-id 10.255.255.2
 neighbor 10.255.255.1 remote-as 65000
 address-family ipv4 unicast
  network 192.168.1.0/24
 exit-address-family
```

### Spoke B FRR Configuration

```
! /etc/frr/frr.conf -- Spoke B
!
interface gre1
 ip address 10.255.255.3/32
 ip nhrp network-id 1
 ip nhrp holdtime 300
 ip nhrp authentication DMVPN1
 ip nhrp nhs 10.255.255.1 nbma 203.0.113.1
 ip nhrp shortcut
 ip nhrp map multicast 203.0.113.1
!
router bgp 65002
 bgp router-id 10.255.255.3
 neighbor 10.255.255.1 remote-as 65000
 address-family ipv4 unicast
  network 192.168.2.0/24
 exit-address-family
```

## Verify DMVPN Operation

```
! Check NHRP registrations on hub
show ip nhrp cache

! Check NHS status on spokes
show ip nhrp nhs

! Check BGP learned routes
show ip bgp

! Check IPsec tunnels
show dmvpn

! Verify shortcut (after spoke-to-spoke traffic)
show ip nhrp cache
! Look for entries with type "dynamic" between spoke addresses
```
