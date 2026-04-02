# BGP Communities

## Overview

Communities are transitive BGP attributes that tag routes with metadata. They enable scalable policy: set a community on ingress, match it elsewhere for filtering, preference, or action. FRR supports three community types.

| Type | Format | Size | RFC |
|---|---|---|---|
| Standard | `ASN:VALUE` (e.g., `65001:100`) | 32-bit | RFC 1997 |
| Extended | `type:ASN:VALUE` (e.g., `rt 65001:100`) | 64-bit | RFC 4360 |
| Large | `ASN:FUNC:VALUE` (e.g., `65001:1:100`) | 96-bit | RFC 8092 |

## Well-Known Communities

| Community | Value | Purpose |
|---|---|---|
| `no-export` | 65535:65281 | Do not advertise outside the local AS |
| `no-advertise` | 65535:65282 | Do not advertise to any peer |
| `local-AS` | 65535:65283 | Do not advertise outside the local confederation sub-AS |
| `no-peer` | 65535:65284 | Do not advertise to eBGP peers (advisory) |
| `blackhole` | 65535:666 | Remotely-triggered blackhole (RTBH) |
| `graceful-shutdown` | 65535:0 | Signal planned maintenance |

## Send Communities to Neighbors

By default, FRR does not send communities. Enable per neighbor:

```
router bgp 65001
 address-family ipv4 unicast
  ! Send standard communities
  neighbor 10.0.0.2 send-community standard

  ! Send extended communities
  neighbor 10.0.0.2 send-community extended

  ! Send large communities
  neighbor 10.0.0.2 send-community large

  ! Send all types (recommended)
  neighbor 10.0.0.2 send-community all
 exit-address-family
```

**Note:** If you do not enable `send-community`, the peer will never see community attributes on routes you advertise. This is a common misconfiguration.

## Set Communities in Route Maps

### Set Standard Communities

```
route-map TAG-CUSTOMER permit 10
 set community 65001:100

route-map TAG-CUSTOMER permit 20
 set community 65001:200 additive
```

The `additive` keyword appends to existing communities instead of replacing them.

### Set Multiple Communities

```
route-map TAG-ROUTES permit 10
 set community 65001:100 65001:200 no-export
```

### Set Extended Communities

```
route-map SET-RT permit 10
 set extcommunity rt 65001:100

route-map SET-SOO permit 10
 set extcommunity soo 65001:1
```

Extended community types:

| Type | Keyword | Purpose |
|---|---|---|
| Route Target | `rt` | VPN import/export control |
| Site of Origin | `soo` | Loop prevention in multi-homed sites |
| Bandwidth | `bandwidth` | Link bandwidth signaling |

### Set Large Communities

```
route-map TAG-LC permit 10
 set large-community 65001:1:100

route-map TAG-LC permit 20
 set large-community 65001:2:200 additive
```

Large communities are useful for 4-byte ASNs where standard communities cannot encode the full AS number.

### Delete Communities

```
route-map STRIP-COMMUNITIES permit 10
 set community none

route-map STRIP-SPECIFIC permit 10
 set comm-list INTERNAL-COMMS delete
```

## Define Community Lists

Community lists group communities for use as match conditions in route maps.

### Standard Community List

```
bgp community-list standard CUSTOMER permit 65001:100
bgp community-list standard CUSTOMER permit 65001:200
bgp community-list standard CUSTOMER deny
```

A standard community list matches exact community values.

### Expanded Community List

```
bgp community-list expanded INTERNAL permit 65001:1[0-9][0-9]
bgp community-list expanded INTERNAL deny .*
```

Expanded community lists use regular expressions.

### Large Community List

```
bgp large-community-list standard LC-CUSTOMER permit 65001:1:100
bgp large-community-list standard LC-CUSTOMER permit 65001:1:200
```

### Extended Community List

```
bgp extcommunity-list standard RT-CUST-A permit rt 65001:100
bgp extcommunity-list standard RT-CUST-A permit rt 65001:200
```

## Match Communities in Route Maps

### Match Standard Community

```
route-map PREFER-CUSTOMER permit 10
 match community CUSTOMER
 set local-preference 200

route-map PREFER-CUSTOMER permit 20
```

### Match with Exact-Match

```
route-map EXACT-MATCH permit 10
 match community CUSTOMER exact-match
```

With `exact-match`, the route must have exactly the communities in the list and no others.

### Match Large Community

```
route-map MATCH-LC permit 10
 match large-community LC-CUSTOMER
 set local-preference 200
```

### Match Extended Community

```
route-map MATCH-RT permit 10
 match extcommunity RT-CUST-A
```

## Community-Based Policy Design

### Tagging Pattern

A common design uses communities to implement a layered policy:

```
! Step 1: Tag routes on ingress based on peer type
route-map CUSTOMER-IN permit 10
 set community 65001:100 additive

route-map PEER-IN permit 10
 set community 65001:200 additive

route-map TRANSIT-IN permit 10
 set community 65001:300 additive

! Step 2: Use communities to control egress
bgp community-list standard CUSTOMER-ROUTES permit 65001:100
bgp community-list standard PEER-ROUTES permit 65001:200

! To customers: send everything
route-map TO-CUSTOMER permit 10

! To peers: send only customer routes
route-map TO-PEER permit 10
 match community CUSTOMER-ROUTES

route-map TO-PEER deny 20

! To transit: send only customer routes
route-map TO-TRANSIT permit 10
 match community CUSTOMER-ROUTES

route-map TO-TRANSIT deny 20
```

### Blackhole Community

Implement remotely-triggered blackhole (RTBH) filtering:

```
! On the edge router: tag blackhole routes
route-map BLACKHOLE permit 10
 match community BLACKHOLE-COMM
 set ip next-hop 192.0.2.1
 set local-preference 999

bgp community-list standard BLACKHOLE-COMM permit 65535:666

! Customer sends a /32 with community 65535:666 to trigger blackholing
```

### No-Export Community

Prevent a route from being advertised outside the local AS:

```
route-map KEEP-LOCAL permit 10
 set community no-export additive
```

## Show Community Information

```
! Show routes with a specific community
show bgp ipv4 unicast community 65001:100

! Show routes with multiple communities (AND logic)
show bgp ipv4 unicast community 65001:100 65001:200

! Show routes with community using exact match
show bgp ipv4 unicast community 65001:100 exact-match

! Show routes with large community
show bgp ipv4 unicast large-community 65001:1:100

! Show community lists
show bgp community-list
show bgp community-list CUSTOMER

! Show large community lists
show bgp large-community-list

! Show extended community lists
show bgp extcommunity-list
```

## Troubleshoot Communities

```
! Verify communities are being sent
show bgp ipv4 unicast neighbors 10.0.0.2 advertised-routes

! Check a specific route for community attributes
show bgp ipv4 unicast 192.168.1.0/24

! Verify send-community is enabled
show bgp neighbors 10.0.0.2 json

! Debug community processing
debug bgp updates in
debug bgp updates out
```

Common issues:
- **Communities not visible on remote peer:** `send-community` not enabled
- **Route map not matching:** Community list name misspelled or wrong type (standard vs expanded)
- **Additive keyword missing:** Communities being replaced instead of appended
