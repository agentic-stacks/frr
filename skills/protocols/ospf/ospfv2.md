# OSPFv2 Configuration

## Set the Router ID

Every OSPF router needs a unique router-id. FRR auto-selects the highest loopback IP, but explicit configuration is strongly recommended.

```
router ospf
 ospf router-id 10.0.0.1
```

If no router-id is set and no loopback exists, OSPF uses the highest IP on any active interface. This can change unexpectedly if interfaces go down.

### Multiple OSPF Instances

FRR supports multiple OSPF instances, each identified by a number:

```
router ospf 1
 ospf router-id 10.0.0.1

router ospf 2
 ospf router-id 10.0.0.2
```

### VRF-Aware OSPF

```
router ospf vrf CUSTOMER1
 ospf router-id 10.100.0.1
 network 10.100.0.0/24 area 0
```

## Enable OSPF on Interfaces

Two methods exist: the `network` statement (traditional) and per-interface configuration (modern, preferred).

### Method 1: Network Statement

```
router ospf
 network 10.0.0.0/24 area 0
 network 172.16.0.0/16 area 1
 network 192.168.10.0/24 area 2
```

The `network` statement matches interface IPs against the prefix. Any interface whose IP falls within the range joins the specified area.

### Method 2: Per-Interface Configuration (Preferred)

```
interface eth0
 ip ospf area 0

interface eth1
 ip ospf area 1
```

Per-interface configuration is explicit and avoids ambiguity from overlapping network statements. Use this method for new deployments.

### Combine Both Methods

If both `network` and `ip ospf area` are configured on the same interface, the per-interface command takes precedence.

## Configure Area Types

### Stub Area

Blocks external LSAs (Type 5). The ABR injects a default route instead.

```
router ospf
 network 10.1.0.0/24 area 1
 area 1 stub
```

**WARNING:** All routers in the area must agree on the stub flag. A mismatch prevents adjacency formation.

### Totally Stubby Area

Blocks both external and inter-area summary LSAs. Only a default route is injected.

```
router ospf
 area 1 stub no-summary
```

Configure `no-summary` only on the ABR. Internal routers just need `area 1 stub`.

### NSSA (Not-So-Stubby Area)

Allows local redistribution via Type-7 LSAs while blocking external Type-5 LSAs from other areas.

```
router ospf
 area 2 nssa
```

### NSSA with Default Route

```
router ospf
 area 2 nssa default-information-originate metric-type 1 metric 100
```

### Totally NSSA

```
router ospf
 area 2 nssa no-summary
```

### NSSA Options

```
router ospf
 ! Suppress forwarding address in translated Type-5 LSAs
 area 2 nssa suppress-fa

 ! Summarize NSSA external routes at the ABR
 area 2 nssa range 10.99.0.0/16
 area 2 nssa range 10.99.0.0/16 not-advertise
 area 2 nssa range 10.99.0.0/16 cost 50
```

## Configure Area Summarization

ABRs can summarize routes when advertising between areas:

```
router ospf
 ! Summarize and advertise
 area 1 range 172.16.0.0/16

 ! Summarize with explicit cost
 area 1 range 172.16.0.0/16 advertise cost 100

 ! Suppress the summary (blackhole)
 area 1 range 172.16.0.0/16 not-advertise

 ! Substitute a different prefix
 area 1 range 172.16.0.0/16 substitute 10.0.0.0/8
```

## Configure Area Filtering

```
router ospf
 ! Filter with access-list
 area 1 export-list MY-EXPORT-LIST
 area 1 import-list MY-IMPORT-LIST

 ! Filter with prefix-list
 area 1 filter-list prefix FILTER-TO-AREA1 in
 area 1 filter-list prefix FILTER-FROM-AREA1 out
```

## Configure Virtual Links

Virtual links extend area 0 connectivity through a transit area when physical backbone contiguity is not possible.

```
router ospf
 area 1 virtual-link 10.0.0.2
```

Replace `10.0.0.2` with the router-id of the ABR at the other end of the virtual link. Area 1 is the transit area.

**WARNING:** Virtual links are a temporary design workaround. They add complexity and fragility. Redesign the topology to achieve physical backbone contiguity when possible.

### Virtual Link with Authentication

```
router ospf
 area 1 virtual-link 10.0.0.2 authentication message-digest
 area 1 virtual-link 10.0.0.2 message-digest-key 1 md5 MySecret123
```

## Configure Authentication

### Area-Wide Simple Password

```
router ospf
 area 0 authentication

interface eth0
 ip ospf authentication-key MyPlainPassword
```

### Area-Wide MD5 Authentication

```
router ospf
 area 0 authentication message-digest

interface eth0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 MyMD5Secret
```

### Per-Interface Authentication (No Area Statement Needed)

```
interface eth0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 MyMD5Secret
```

### Keychain Authentication

```
key chain OSPF-KEYS
 key 1
  key-string MyKeyString1
  cryptographic-algorithm hmac-sha-256
 key 2
  key-string MyKeyString2
  cryptographic-algorithm hmac-sha-256

interface eth0
 ip ospf authentication key-chain OSPF-KEYS
```

**WARNING:** Authentication must match exactly on both sides of a link. A mismatch silently prevents adjacency formation with no error in the neighbor table -- the neighbor simply never appears.

### Key Rotation

When rotating MD5 keys, add the new key before removing the old one. OSPF accepts packets authenticated with any configured key but sends with the highest key ID.

```
! Step 1: Add new key on all routers
interface eth0
 ip ospf message-digest-key 2 md5 NewSecret456

! Step 2: Verify adjacencies are stable
show ip ospf neighbor

! Step 3: Remove old key on all routers
interface eth0
 no ip ospf message-digest-key 1
```

## Configure Timers

### Interface Timers

```
interface eth0
 ip ospf hello-interval 5
 ip ospf dead-interval 20
 ip ospf retransmit-interval 3
 ip ospf transmit-delay 1
```

**WARNING:** Hello and dead intervals must match on all routers sharing a link. Mismatched timers prevent adjacency formation.

The dead interval should be at least 4x the hello interval. Common fast-convergence pairs: hello 1s / dead 4s, or use BFD instead.

### Fast Hellos (Sub-Second Dead Detection)

```
interface eth0
 ip ospf dead-interval minimal hello-multiplier 5
```

This sets dead interval to 1 second and sends hellos every 200ms (1s / 5). All routers on the link must use the same configuration.

### SPF Throttle Timers

Control how aggressively OSPF recalculates routes after topology changes:

```
router ospf
 timers throttle spf 50 200 5000
```

Parameters (in milliseconds):
- `50` -- initial delay before first SPF run after a change
- `200` -- minimum hold time between consecutive SPF runs
- `5000` -- maximum hold time (back-off ceiling)

### LSA Throttle Timers

```
router ospf
 timers throttle lsa all 200
 timers lsa min-arrival 500
```

## Configure Cost and Metrics

### Per-Interface Cost

```
interface eth0
 ip ospf cost 10

interface eth1
 ip ospf cost 100
```

Lower cost is preferred. Set costs explicitly when auto-cost does not reflect your design intent.

### Auto-Cost Reference Bandwidth

The formula is: cost = reference-bandwidth / interface-bandwidth (in Mbps).

```
router ospf
 auto-cost reference-bandwidth 10000
```

| Interface Speed | Ref 100 (default) | Ref 1000 | Ref 10000 | Ref 100000 |
|---|---|---|---|---|
| 10 Mbps | 10 | 100 | 1000 | 10000 |
| 100 Mbps | 1 | 10 | 100 | 1000 |
| 1 Gbps | 1 | 1 | 10 | 100 |
| 10 Gbps | 1 | 1 | 1 | 10 |
| 100 Gbps | 1 | 1 | 1 | 1 |

**WARNING:** With the default reference bandwidth of 100 Mbps, all interfaces 100 Mbps and faster have cost 1. This makes OSPF unable to distinguish between Fast Ethernet, Gigabit, and 10G links. Set reference-bandwidth to match your fastest link speed across all routers.

### Administrative Distance

```
router ospf
 distance 110
 distance ospf intra-area 90
 distance ospf inter-area 100
 distance ospf external 120
```

### Default Metric for Redistributed Routes

```
router ospf
 default-metric 100
```

## Configure Passive Interfaces

Passive interfaces advertise their subnet into OSPF but do not send or process hellos. Use for host-facing LANs, loopbacks, and management interfaces.

```
router ospf
 passive-interface lo
 passive-interface eth1
```

### Make All Interfaces Passive by Default

```
router ospf
 passive-interface default

interface eth0
 no ip ospf passive
```

This is the safest approach: start passive, then explicitly enable OSPF peering only on transit links.

### Per-Interface Passive

```
interface eth1
 ip ospf passive
```

## Configure Redistribution

### Redistribute into OSPF

```
router ospf
 redistribute connected metric-type 1 metric 20 route-map CONNECTED-TO-OSPF
 redistribute static metric-type 2 metric 100 route-map STATIC-TO-OSPF
 redistribute bgp metric-type 1 metric 50 route-map BGP-TO-OSPF
```

Supported sources: `babel`, `bgp`, `connected`, `eigrp`, `isis`, `kernel`, `openfabric`, `rip`, `sharp`, `static`, `table`.

### Metric Type 1 vs Type 2

| Type | Route Code | Cost Calculation | Use When |
|---|---|---|---|
| Type 1 (E1) | `O E1` | OSPF internal cost + external metric | Multiple exit points to external destinations |
| Type 2 (E2) | `O E2` | External metric only (ignores OSPF cost) | Single exit point, or internal cost is irrelevant |

Type 2 is the default. Use Type 1 when you have multiple ASBRs redistributing the same external routes, so OSPF can prefer the nearest exit.

### Default Route Origination

```
router ospf
 ! Originate default only when one exists in RIB
 default-information originate

 ! Always originate default, even without one in RIB
 default-information originate always

 ! With metric and type
 default-information originate always metric 10 metric-type 1

 ! Conditional on route-map match
 default-information originate always route-map DEFAULT-CHECK
```

### External Route Summarization

ASBRs can summarize redistributed external routes:

```
router ospf
 summary-address 10.99.0.0/16
 summary-address 10.99.0.0/16 tag 100
 summary-address 172.16.0.0/12 no-advertise
 aggregation timer 10
```

### Distribution Lists

```
router ospf
 distribute-list FILTER-EXTERNAL out bgp
 distribute-list FILTER-STATIC out static
```

## Configure ECMP

```
router ospf
 maximum-paths 4
```

Default is 64 equal-cost paths. Set to 1 to disable ECMP.

## Configure Max-Metric (Stub Router)

Advertise maximum metric to gracefully drain traffic before maintenance:

```
router ospf
 ! Immediate max-metric (manual drain)
 max-metric router-lsa administrative

 ! Max-metric on startup for 300 seconds (wait for convergence)
 max-metric router-lsa on-startup 300

 ! Max-metric on shutdown for 60 seconds (drain before stop)
 max-metric router-lsa on-shutdown 60
```

Remove administrative max-metric to resume normal operation:

```
router ospf
 no max-metric router-lsa administrative
```

## Configure Graceful Restart

Graceful restart allows OSPF to restart without disrupting forwarding.

### Enable as Restarter

```
router ospf
 graceful-restart grace-period 180
```

### Prepare for Graceful Restart (Before Planned Shutdown)

```
graceful-restart prepare ip ospf
```

### Enable as Helper

```
router ospf
 graceful-restart helper enable
 graceful-restart helper strict-lsa-checking
 graceful-restart helper supported-grace-time 300
 graceful-restart helper planned-only
```

### Per-Interface Hello Delay During Restart

```
interface eth0
 ip ospf graceful-restart hello-delay 10
```

### Verify Graceful Restart

```
show ip ospf graceful-restart helper detail
```

## Configure OSPF-TE (Traffic Engineering)

### Enable OSPF-TE

```
router ospf
 mpls-te on
 mpls-te router-address 10.0.0.1
```

### Enable Opaque LSA Capability

OSPF-TE requires opaque LSA support:

```
router ospf
 capability opaque
```

Per-interface opaque capability:

```
interface eth0
 ip ospf capability opaque
```

### Inter-AS TE

```
router ospf
 mpls-te inter-as area 0
 ! or
 mpls-te inter-as as
```

### Export TE Database

```
router ospf
 mpls-te export
```

### Verify OSPF-TE

```
show ip ospf mpls-te interface
show ip ospf mpls-te interface eth0
show ip ospf mpls-te router
show ip ospf mpls-te database
show ip ospf mpls-te database verbose
show ip ospf mpls-te database json
```

## Configure Segment Routing

### Enable SR on OSPF

```
router ospf
 segment-routing on
 segment-routing global-block 16000 23999
 segment-routing node-msd 8
```

### Assign Prefix SIDs

```
router ospf
 segment-routing prefix 10.0.0.1/32 index 1
 segment-routing prefix 10.0.0.1/32 index 1 no-php-flag
 segment-routing prefix 10.0.0.1/32 index 1 explicit-null
```

The prefix SID index is added to the SRGB base to produce the MPLS label. With SRGB 16000-23999 and index 1, the label is 16001.

### Local Block

Reserve a separate label range for adjacency SIDs:

```
router ospf
 segment-routing global-block 16000 23999 local-block 15000 15999
```

## Configure TI-LFA (Fast Reroute)

```
interface eth0
 ip ospf fast-reroute ti-lfa
 ip ospf fast-reroute ti-lfa node-protection
```

## Configure ABR Type

```
router ospf
 ospf abr-type standard
```

Options: `cisco`, `ibm`, `shortcut`, `standard`. Default is `cisco`. Only change this if interoperating with specific vendors that implement ABR behavior differently.

## Configure Network Types

```
interface eth0
 ip ospf network point-to-point

interface eth1
 ip ospf network broadcast

interface eth2
 ip ospf network non-broadcast

interface eth3
 ip ospf network point-to-multipoint
```

### Point-to-Point on Ethernet

Modern best practice for /31 or /30 links between two routers on Ethernet:

```
interface eth0
 ip ospf network point-to-point
```

This eliminates DR/BDR election and speeds adjacency formation.

### NBMA Neighbors

On non-broadcast networks, explicitly configure neighbors:

```
router ospf
 neighbor 10.0.0.2
 neighbor 10.0.0.3 priority 0 poll-interval 120
```

## Configure Prefix Suppression

Hide transit link prefixes from the routing table while maintaining adjacencies:

```
interface eth0
 ip ospf prefix-suppression
```

## Configure DR/BDR Priority

```
interface eth0
 ip ospf priority 200
```

Priority 0 prevents the router from becoming DR or BDR. Higher values win the election. Default is 1.

## Configure Logging

```
router ospf
 log-adjacency-changes
 log-adjacency-changes detail
```

## Complete Multi-Area Configuration Example

```
! Router R1 -- ABR between area 0 and area 1
router ospf
 ospf router-id 10.0.0.1
 auto-cost reference-bandwidth 10000
 log-adjacency-changes
 passive-interface default
 !
 area 1 stub
 area 1 range 172.16.0.0/16
 !
 default-information originate always metric 10 metric-type 1
 redistribute connected route-map CONN-TO-OSPF
 !
 graceful-restart grace-period 180

interface lo
 ip ospf area 0

interface eth0
 ip ospf area 0
 ip ospf network point-to-point
 ip ospf cost 10
 no ip ospf passive

interface eth1
 ip ospf area 1
 ip ospf network point-to-point
 ip ospf cost 10
 no ip ospf passive

interface eth2
 ip ospf area 1
 ip ospf cost 100
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 Area1Secret
 no ip ospf passive
```

## Debugging

```
! Packet-level debugging
debug ospf packet hello send
debug ospf packet hello recv
debug ospf packet dd send detail
debug ospf packet ls-update recv detail

! State machine debugging
debug ospf ism status
debug ospf ism events
debug ospf nsm status
debug ospf nsm events

! LSA debugging
debug ospf lsa generate
debug ospf lsa install
debug ospf lsa flooding
debug ospf lsa refresh

! Feature-specific debugging
debug ospf event
debug ospf nssa
debug ospf sr
debug ospf te
debug ospf ti-lfa
debug ospf zebra interface
debug ospf zebra redistribute
debug ospf graceful-restart
debug ospf bfd
```

**WARNING:** OSPF debug output can be extremely verbose on production routers with many adjacencies. Always specify the packet type and direction. Remove debug commands after troubleshooting with `no debug ospf` or `undebug all`.
