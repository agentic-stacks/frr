# OSPFv3 Configuration

## Key Differences from OSPFv2

| Aspect | OSPFv2 | OSPFv3 |
|---|---|---|
| Protocol | IPv4 only | IPv6 (can carry IPv4 via AF) |
| Daemon | `ospfd` | `ospf6d` |
| Interface config | `network` statement or `ip ospf area` | `ipv6 ospf6 area` on interface (required) |
| Authentication | Built-in MD5/plaintext | IPsec or authentication trailer (RFC 7166) |
| Multicast addresses | 224.0.0.5, 224.0.0.6 | ff02::5, ff02::6 |
| Instance support | Via `router ospf <N>` | Via instance IDs per interface |
| LSA types | Types 1-5, 7 | Types 1-5, 7, plus 8 (Link) and 9 (Intra-Area-Prefix) |
| Network statement | `network A.B.C.D/M area X` | Not used; assign area on interface |
| Neighbor identification | IP address | Router-id |
| Link-local addresses | Not used | Used for next-hop and neighbor communication |

OSPFv3 runs directly on IPv6 link-local addresses. Global or ULA addresses are not required for adjacency formation, only for reachability of the advertised prefixes.

## Enable OSPFv3

Edit `/etc/frr/daemons`:

```
ospf6d=yes
```

Reload:

```
sudo systemctl reload frr
```

## Set the Router ID

OSPFv3 still uses a 32-bit router-id in dotted-decimal notation, even though it runs over IPv6:

```
router ospf6
 ospf6 router-id 10.0.0.1
```

### VRF-Aware OSPFv3

```
router ospf6 vrf CUSTOMER1
 ospf6 router-id 10.100.0.1
```

## Enable OSPFv3 on Interfaces

OSPFv3 requires per-interface area assignment. There is no `network` statement equivalent.

```
interface eth0
 ipv6 ospf6 area 0

interface eth1
 ipv6 ospf6 area 1

interface lo
 ipv6 ospf6 area 0
```

Each interface must have at least one IPv6 address (link-local is sufficient for adjacency, but global/ULA is needed for routable prefixes).

## Configure Instance IDs

Instance IDs allow multiple OSPFv3 instances on the same interface. Each instance ID creates a separate adjacency and LSDB. Default instance ID is 0.

```
interface eth0
 ipv6 ospf6 instance-id 0

interface eth0
 ipv6 ospf6 instance-id 1
```

Use cases:
- Running separate OSPFv3 topologies on shared infrastructure
- Isolating IPv6 unicast from IPv6 multicast routing

Both ends of a link must use the same instance ID to form an adjacency.

## Configure Interface Parameters

### Cost and Priority

```
interface eth0
 ipv6 ospf6 cost 10
 ipv6 ospf6 priority 200
```

### Timers

```
interface eth0
 ipv6 ospf6 hello-interval 5
 ipv6 ospf6 dead-interval 20
 ipv6 ospf6 retransmit-interval 3
```

Timer defaults are the same as OSPFv2: hello 10s, dead 40s, retransmit 5s.

**WARNING:** Hello and dead intervals must match on all routers sharing a link, just like OSPFv2.

### Network Type

```
interface eth0
 ipv6 ospf6 network point-to-point

interface eth1
 ipv6 ospf6 network broadcast

interface eth2
 ipv6 ospf6 network point-to-multipoint
```

### Passive Interface

```
interface eth1
 ipv6 ospf6 passive
```

### Explicit Neighbors (Point-to-Multipoint)

```
interface eth0
 ipv6 ospf6 neighbor 2001:db8::2
 ipv6 ospf6 neighbor 2001:db8::3 poll-interval 120
```

## Configure Authentication

OSPFv3 does not have built-in MD5 authentication like OSPFv2. FRR supports the authentication trailer mechanism defined in RFC 7166.

### Per-Interface Manual Key

```
interface eth0
 ipv6 ospf6 authentication key-id 1 hash-algo hmac-sha-256 key MySecretKey123
```

Supported hash algorithms: `md5`, `hmac-sha-1`, `hmac-sha-256`, `hmac-sha-384`, `hmac-sha-512`.

### Per-Interface Keychain

```
key chain OSPF6-KEYS
 key 1
  key-string MyKeyString1
  cryptographic-algorithm hmac-sha-256
 key 2
  key-string MyKeyString2
  cryptographic-algorithm hmac-sha-256

interface eth0
 ipv6 ospf6 authentication keychain OSPF6-KEYS
```

Keychains support timed key rotation. Both sides must have matching active keys.

**WARNING:** Authentication must match exactly on both endpoints. A mismatch silently prevents adjacency formation.

## Configure Area Types

### Stub Area

```
router ospf6
 area 1 stub
```

All routers in the area must agree on the stub flag.

### Totally Stubby Area

```
router ospf6
 area 1 stub no-summary
```

### NSSA

```
router ospf6
 area 2 nssa
 area 2 nssa default-information-originate
```

### Totally NSSA

```
router ospf6
 area 2 nssa no-summary
```

## Configure Area Summarization

```
router ospf6
 area 1 range 2001:db8:1::/48 advertise
 area 1 range 2001:db8:1::/48 advertise cost 100
 area 1 range 2001:db8:2::/48 not-advertise
```

## Configure Area Filtering

```
router ospf6
 area 1 export-list MY-EXPORT
 area 1 import-list MY-IMPORT
 area 1 filter-list prefix FILTER-IN in
 area 1 filter-list prefix FILTER-OUT out
```

## Configure Redistribution

```
router ospf6
 redistribute connected metric-type 1 metric 20 route-map CONN-TO-OSPF6
 redistribute static metric-type 2 metric 100
 redistribute bgp route-map BGP-TO-OSPF6
```

Supported sources: `babel`, `bgp`, `connected`, `isis`, `kernel`, `openfabric`, `ripng`, `sharp`, `static`, `table`.

**Note:** OSPFv3 redistributes from `ripng`, not `rip`.

### Default Route Origination

```
router ospf6
 default-information originate always metric 10 metric-type 1
 default-information originate always route-map DEFAULT-CHECK
```

### ASBR External Route Summarization

```
router ospf6
 summary-address 2001:db8::/32
 summary-address 2001:db8::/32 tag 100
 summary-address 2001:db8:ff::/48 no-advertise
 summary-address 2001:db8::/32 metric 50 metric-type 1
```

## Configure Graceful Restart

### Enable as Restarter

```
router ospf6
 graceful-restart grace-period 180
```

### Prepare for Planned Restart

```
graceful-restart prepare ipv6 ospf
```

### Enable as Helper

```
router ospf6
 graceful-restart helper enable
```

## Show Commands

```
! Process overview
show ipv6 ospf6

! Interface status
show ipv6 ospf6 interface
show ipv6 ospf6 interface eth0

! Neighbor state
show ipv6 ospf6 neighbor
show ipv6 ospf6 neighbor detail

! Link-state database
show ipv6 ospf6 database
show ipv6 ospf6 database detail
show ipv6 ospf6 database self-originate

! Routing table
show ipv6 ospf6 route

! Border routers
show ipv6 ospf6 border-routers

! Redistribution summary
show ipv6 ospf6 redistribute
```

## Operational Commands

**WARNING:** Clearing the OSPFv3 process drops all adjacencies and triggers full reconvergence.

```
clear ipv6 ospf6 process
clear ipv6 ospf6 neighbor
```

## Complete OSPFv3 Configuration Example

```
! Router R1 -- dual-stack ABR with OSPFv3 for IPv6
router ospf6
 ospf6 router-id 10.0.0.1
 area 1 stub
 area 1 range 2001:db8:1::/48
 redistribute connected route-map CONN-TO-OSPF6
 graceful-restart grace-period 180

interface lo
 ipv6 ospf6 area 0

interface eth0
 ipv6 ospf6 area 0
 ipv6 ospf6 network point-to-point
 ipv6 ospf6 cost 10
 ipv6 ospf6 authentication key-id 1 hash-algo hmac-sha-256 key BackboneKey

interface eth1
 ipv6 ospf6 area 1
 ipv6 ospf6 network point-to-point
 ipv6 ospf6 cost 10
 ipv6 ospf6 authentication keychain OSPF6-KEYS

interface eth2
 ipv6 ospf6 area 1
 ipv6 ospf6 passive
```

## Debugging

```
debug ospf6 message hello send
debug ospf6 message hello recv
debug ospf6 message dbdesc send
debug ospf6 message lsreq recv
debug ospf6 message lsupdate send
debug ospf6 message lsack recv

debug ospf6 neighbor state
debug ospf6 neighbor event

debug ospf6 lsa originate
debug ospf6 lsa examine

debug ospf6 interface
debug ospf6 route table
debug ospf6 flooding
debug ospf6 graceful-restart
```

**WARNING:** Debug output can be extremely verbose. Always specify message type and direction. Remove debug commands after troubleshooting.
