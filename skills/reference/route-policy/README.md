# Route Policy Building Blocks

Cross-protocol policy primitives in FRR: route maps, prefix lists, community lists, AS path access lists, and distribute lists.

---

## Route Maps

Route maps are ordered sequences of permit/deny entries that match routes and optionally modify their attributes. They are the primary policy tool in FRR.

### Create a Route Map Entry

```
route-map NAME (permit|deny) SEQUENCE
```

- **NAME** — arbitrary identifier
- **permit|deny** — action when all match clauses succeed
- **SEQUENCE** — order number (lower = evaluated first); entries processed sequentially

### Processing Logic

| Entry Policy | All Match Clauses True | Match Clauses False |
|---|---|---|
| permit | Execute set actions, permit route (unless exit policy overrides) | Skip to next entry |
| deny | Deny route immediately | Skip to next entry |

If no entry matches, the route is **denied** (implicit deny at end).

Add an empty permit entry at the end to change this:

```
route-map ALLOW-ALL permit 65535
!
```

### Match Clauses

```
! IPv4
match ip address ACCESS_LIST
match ip address prefix-list PREFIX_LIST
match ip address prefix-len 0-32
match ip next-hop ACCESS_LIST
match ip next-hop IPV4_ADDR
match ip next-hop prefix-list PREFIX_LIST

! IPv6
match ipv6 address ACCESS_LIST
match ipv6 address prefix-list PREFIX_LIST
match ipv6 address prefix-len 0-128
match ipv6 next-hop ACCESS_LIST
match ipv6 next-hop IPV6_ADDR
match ipv6 next-hop prefix-list PREFIX_LIST

! BGP attributes
match as-path AS_PATH_LIST
match community COMMUNITY_LIST [exact-match|any]
match local-preference METRIC
match metric METRIC
match origin (egp|igp|incomplete)
match peer [IPV4|IPV6|INTERFACE_NAME|PEER_GROUP]
match src-peer [IPV4|IPV6|INTERFACE|PEER_GROUP]
match tag <untagged|(1-4294967295)>

! Protocol / source
match source-protocol PROTOCOL_NAME
match source-instance NUMBER (0-255)

! EVPN
match evpn route-type ROUTE_TYPE (1-5)
match evpn vni NUMBER (1-16777215)

! VPN
match vpn dataplane <mpls|srv6|vxlan>
```

### Set Actions

```
! Next-hop manipulation
set ip next-hop IPV4_ADDRESS
set ip next-hop peer-address
set ip next-hop unchanged
set ipv6 next-hop global IPV6_ADDRESS
set ipv6 next-hop peer-address
set ipv6 next-hop prefer-global
set ipv6 next-hop local IPV6_ADDRESS

! BGP attributes
set local-preference LOCAL_PREF
set local-preference +LOCAL_PREF          ! increment
set local-preference -LOCAL_PREF          ! decrement
set weight WEIGHT
set metric <[+|-](1-4294967295)|rtt|+rtt|-rtt|igp|aigp>
set min-metric (0-4294967295)
set max-metric (0-4294967295)
set aigp-metric <igp-metric|(0-4294967295)>
set origin <egp|igp|incomplete>
set distance (1-255)

! AS path manipulation
set as-path prepend AS_PATH
set as-path exclude AS-NUMBER...
set as-path exclude all
set as-path exclude as-path-access-list WORD
set as-path replace <any|ASN> [ASN]
set as-path replace as-path-access-list WORD [ASN]

! Community manipulation
set community COMMUNITY
set community additive COMMUNITY
set community none
set extended-comm-list EXTCOMMUNITY_LIST_NAME delete

! Tagging / table
set tag <untagged|(1-4294967295)>
set table (1-4294967295)

! Segment routing
set sr-te color (1-4294967295)
set l3vpn next-hop encapsulation gre
```

### Call (Sub-Route-Map)

Invoke another route map from within an entry. If the called map returns deny, processing terminates and the route is denied.

```
route-map PARENT permit 10
 match ip address prefix-list CUSTOMER
 call CHILD-MAP
!
```

### Exit / Continue Policy

Override the default "stop on first permit match" behavior:

```
on-match next              ! continue to next sequence entry
on-match goto N            ! jump forward to sequence >= N
continue [N]               ! like goto; continue processing at N
```

**continue** is especially useful when you need multiple set actions across entries:

```
route-map MULTI-SET permit 10
 match community CUST-A
 set local-preference 200
 continue 20
!
route-map MULTI-SET permit 20
 match community CUST-A
 set community 65000:100 additive
!
```

### Route Map Optimization

Enabled by default. Uses prefix-tree lookup instead of sequential scan for prefix-list based matches.

```
route-map NAME optimization       ! enable (default)
no route-map NAME optimization    ! disable
```

### Display and Troubleshoot

```
show route-map [NAME] [json]
show route-map-unused [json]
clear route-map counter [NAME]
```

---

## Prefix Lists

Filter routes by prefix and prefix length. Used inside route maps (`match ip address prefix-list`) or directly in protocol configurations.

### IPv4 Prefix List

```
ip prefix-list NAME (permit|deny) PREFIX [ge GE_LEN] [le LE_LEN]
ip prefix-list NAME seq NUMBER (permit|deny) PREFIX [ge GE_LEN] [le LE_LEN]
ip prefix-list NAME description TEXT
```

### IPv6 Prefix List

```
ipv6 prefix-list NAME (permit|deny) PREFIX [ge GE_LEN] [le LE_LEN]
ipv6 prefix-list NAME seq NUMBER (permit|deny) PREFIX [ge GE_LEN] [le LE_LEN]
ipv6 prefix-list NAME description TEXT
```

### ge/le Semantics

| Parameter | Meaning |
|---|---|
| (none) | Exact match only |
| `ge G` | Prefix length >= G |
| `le L` | Prefix length <= L |
| `ge G le L` | G <= prefix length <= L |

The prefix itself must always match first; ge/le then filter on the mask length.

### Examples

```
! Match the default route only
ip prefix-list DEFAULT-ONLY permit 0.0.0.0/0

! Match all routes
ip prefix-list ANY permit 0.0.0.0/0 le 32

! Match /24 through /28 subnets within 10.0.0.0/8
ip prefix-list SPECIFIC permit 10.0.0.0/8 ge 24 le 28

! Block RFC 1918 and allow everything else
ip prefix-list NO-RFC1918 deny 10.0.0.0/8 le 32
ip prefix-list NO-RFC1918 deny 172.16.0.0/12 le 32
ip prefix-list NO-RFC1918 deny 192.168.0.0/16 le 32
ip prefix-list NO-RFC1918 permit 0.0.0.0/0 le 32

! IPv6 — allow /48 or shorter from 2001:db8::/32
ipv6 prefix-list V6-CUSTOMERS permit 2001:db8::/32 ge 32 le 48
```

### Display Commands

```
show ip prefix-list [NAME] [json]
show ip prefix-list NAME PREFIX [longer|first-match]
show ip prefix-list summary [NAME] [json]
show ip prefix-list detail [NAME] [json]
show ipv6 prefix-list [NAME] [json]
clear ip prefix-list [NAME [PREFIX]]
debug prefix-list NAME match PREFIX [address-mode]
```

---

## Access Lists

### Standard Access List

Match by destination prefix:

```
access-list NAME [seq (1-4294967295)] <permit|deny> <A.B.C.D/M [exact-match]|any>
```

### Extended Access List

Match by source and destination with wildcard masks:

```
access-list NAME [seq (1-4294967295)] <deny|permit> ip SOURCE DESTINATION
```

Wildcard masks use inverse logic (0 = must match, 1 = don't care):

```
access-list BLOCK-HOST seq 10 deny ip 10.0.0.1 0.0.0.0 any
access-list BLOCK-HOST seq 20 permit ip any any
```

### Display

```
show ip access-list [NAME] [json]
show ipv6 access-list [NAME] [json]
```

---

## Community Lists

Community lists match BGP communities for use in route-map `match community` clauses.

### Standard Community List

Match exact community values:

```
bgp community-list <1-99> (permit|deny) COMMUNITY
bgp community-list standard NAME (permit|deny) COMMUNITY
```

Community values: `AA:NN`, `internet`, `local-AS`, `no-advertise`, `no-export`, `graceful-shutdown`.

### Expanded Community List

Match communities using regular expressions:

```
bgp community-list <100-500> (permit|deny) REGEX
bgp community-list expanded NAME (permit|deny) REGEX
```

### Examples

```
! Standard — match exact community
bgp community-list standard CUSTOMERS permit 65000:100
bgp community-list standard CUSTOMERS permit 65000:200

! Expanded — match any community from AS 65000
bgp community-list expanded AS65000 permit 65000:.*

! Use in route map
route-map FILTER-COMMUNITY permit 10
 match community CUSTOMERS
 set local-preference 200
!
route-map FILTER-COMMUNITY permit 20
 match community CUSTOMERS exact-match
 set local-preference 300
!
```

### Large Community Lists

```
bgp large-community-list standard NAME (permit|deny) AA:BB:CC
bgp large-community-list expanded NAME (permit|deny) REGEX
```

### Extended Community Lists

```
bgp extcommunity-list standard NAME (permit|deny) EXTCOMMUNITY
bgp extcommunity-list expanded NAME (permit|deny) REGEX
```

---

## AS Path Access Lists

Filter routes by BGP AS path using regular expressions.

### Syntax

```
bgp as-path access-list WORD [seq (0-4294967295)] permit|deny LINE
```

### Common Regex Patterns

| Pattern | Matches |
|---|---|
| `^$` | Routes originated locally (empty AS path) |
| `_65000$` | Routes originated by AS 65000 |
| `_65000_` | Routes transiting AS 65000 |
| `^65000_` | Routes received directly from AS 65000 |
| `^65000 65001` | Routes from AS 65001 via AS 65000 |
| `_0_` | Bogon AS 0 in path |
| `_23456_` | AS_TRANS (4-byte AS transition) |
| `^[0-9]+$` | Single-hop routes (one ASN in path) |
| `.*` | Any AS path (match all) |

Underscore `_` matches beginning, end, or whitespace (AS boundary).

### Examples

```
! Accept only customer routes (originated by known ASes)
bgp as-path access-list CUSTOMERS seq 10 permit ^65001$
bgp as-path access-list CUSTOMERS seq 20 permit ^65002$
bgp as-path access-list CUSTOMERS seq 30 deny .*

! Filter bogon ASes
bgp as-path access-list NO-BOGONS seq 10 deny _0_
bgp as-path access-list NO-BOGONS seq 20 deny _23456_
bgp as-path access-list NO-BOGONS seq 30 deny _6449[6-9]_|_64[5-9][0-9][0-9]_|_65[0-4][0-9][0-9]_|_655[0-4][0-9]_|_6555[0-5]_
bgp as-path access-list NO-BOGONS seq 100 permit .*

! Use in route map
route-map BGP-IN permit 10
 match as-path CUSTOMERS
!

! Display
show bgp as-path-access-list [NAME] [json]
```

---

## Distribute Lists

Distribute lists apply access lists or prefix lists to filter routes for specific protocols. They are an older mechanism; prefer prefix-list-based route maps for new configurations.

```
! Under router ospf / router rip / etc.
distribute-list ACCESS_LIST (in|out) [INTERFACE]
distribute-list prefix PREFIX_LIST (in|out) [INTERFACE]
```

---

## Where Route Maps Are Applied

### BGP Per-Neighbor

```
router bgp 65000
 neighbor 10.0.0.1 route-map IMPORT in
 neighbor 10.0.0.1 route-map EXPORT out
 neighbor 10.0.0.1 unsuppress-map UNSUPPRESS
 neighbor 10.0.0.1 advertise-map ADV exist-map EXIST
 neighbor 10.0.0.1 default-originate route-map CONDITION
!
```

### Redistribution

```
router bgp 65000
 address-family ipv4 unicast
  redistribute ospf route-map OSPF-TO-BGP
  redistribute connected route-map CONNECTED-TO-BGP
  redistribute static route-map STATIC-TO-BGP
!

router ospf
 redistribute bgp route-map BGP-TO-OSPF
 redistribute connected route-map CONNECTED-TO-OSPF
!
```

### Zebra Protocol Filtering

```
ip protocol bgp route-map ZEBRA-BGP-FILTER
ip protocol ospf route-map ZEBRA-OSPF-FILTER
ipv6 protocol bgp route-map ZEBRA-BGP6-FILTER
```

### Table Map (BGP to RIB)

```
router bgp 65000
 address-family ipv4 unicast
  table-map TABLE-FILTER
!
```

### Default Route Origination

```
router bgp 65000
 neighbor 10.0.0.1 default-originate route-map CHECK-DEFAULT
!
```

---

## Full Real-World Example: ISP Peering Policy

```
! === Prefix lists ===
ip prefix-list BOGONS deny 0.0.0.0/8 le 32
ip prefix-list BOGONS deny 10.0.0.0/8 le 32
ip prefix-list BOGONS deny 100.64.0.0/10 le 32
ip prefix-list BOGONS deny 127.0.0.0/8 le 32
ip prefix-list BOGONS deny 169.254.0.0/16 le 32
ip prefix-list BOGONS deny 172.16.0.0/12 le 32
ip prefix-list BOGONS deny 192.0.2.0/24 le 32
ip prefix-list BOGONS deny 192.168.0.0/16 le 32
ip prefix-list BOGONS deny 198.18.0.0/15 le 32
ip prefix-list BOGONS deny 198.51.100.0/24 le 32
ip prefix-list BOGONS deny 203.0.113.0/24 le 32
ip prefix-list BOGONS deny 224.0.0.0/4 le 32
ip prefix-list BOGONS deny 240.0.0.0/4 le 32
ip prefix-list BOGONS permit 0.0.0.0/0 le 24

! Reject too-specific prefixes
ip prefix-list MAX-PREFIX-LEN permit 0.0.0.0/0 le 24

! === AS path filters ===
bgp as-path access-list NO-BOGON-ASNS seq 10 deny _0_
bgp as-path access-list NO-BOGON-ASNS seq 20 deny _23456_
bgp as-path access-list NO-BOGON-ASNS seq 30 deny _6553[0-5]_
bgp as-path access-list NO-BOGON-ASNS seq 100 permit .*

! === Community lists ===
bgp community-list standard BLACKHOLE permit 65000:666
bgp community-list standard NO-EXPORT permit 65000:999

! === Route maps ===
route-map PEER-IN permit 10
 description Drop bogon prefixes
 match ip address prefix-list BOGONS
 match as-path NO-BOGON-ASNS
 set local-preference 100
 continue 20
!
route-map PEER-IN permit 20
 description Tag blackhole routes
 match community BLACKHOLE
 set community no-export additive
 set ip next-hop 192.0.2.1
!
route-map PEER-IN deny 65535
 description Implicit deny (explicit for clarity)
!

route-map PEER-OUT permit 10
 description Announce only our prefixes
 match ip address prefix-list OUR-PREFIXES
 match as-path OUR-ORIGIN
!
route-map PEER-OUT permit 20
 description Apply no-export where requested
 match community NO-EXPORT
 set community no-export
!

! === Apply to BGP neighbor ===
router bgp 65000
 neighbor 198.51.100.1 remote-as 65001
 neighbor 198.51.100.1 description Peer-AS65001
 address-family ipv4 unicast
  neighbor 198.51.100.1 route-map PEER-IN in
  neighbor 198.51.100.1 route-map PEER-OUT out
  neighbor 198.51.100.1 prefix-list MAX-PREFIX-LEN in
 exit-address-family
!
```

## Full Real-World Example: DC Leaf-Spine BGP

```
! === Prefix list — only allow host routes and connected ===
ip prefix-list LEAF-EXPORT permit 10.255.0.0/16 ge 32 le 32
ip prefix-list LEAF-EXPORT permit 10.0.0.0/8 ge 24 le 24

! === Route map for spine import ===
route-map SPINE-IN permit 10
 set weight 0
!

route-map SPINE-OUT permit 10
 match ip address prefix-list LEAF-EXPORT
!

! === BGP unnumbered to spines ===
router bgp 65100
 bgp bestpath as-path multipath-relax
 neighbor SPINES peer-group
 neighbor SPINES remote-as external
 neighbor SPINES route-map SPINE-IN in
 neighbor SPINES route-map SPINE-OUT out
 neighbor swp1 interface peer-group SPINES
 neighbor swp2 interface peer-group SPINES
 address-family ipv4 unicast
  redistribute connected route-map REDISTRIBUTE-CONNECTED
 exit-address-family
!
```
