# BGP Policies

## Overview

BGP policy controls which routes are accepted, advertised, and how their attributes are modified. FRR implements policy through route maps, prefix lists, AS path access lists, and distribute lists. Route maps are the primary mechanism -- they reference the other filter types as match conditions.

## Apply a Route Map to a Neighbor

### Inbound Policy (Filter/Modify Received Routes)

```
router bgp 65001
 address-family ipv4 unicast
  neighbor 10.0.0.2 route-map IMPORT-POLICY in
 exit-address-family
```

### Outbound Policy (Filter/Modify Advertised Routes)

```
router bgp 65001
 address-family ipv4 unicast
  neighbor 10.0.0.2 route-map EXPORT-POLICY out
 exit-address-family
```

## Define a Route Map

Route maps are ordered sequences of permit/deny clauses. Processing stops at the first match.

```
route-map IMPORT-POLICY permit 10
 match ip address prefix-list ALLOWED-PREFIXES
 set local-preference 200

route-map IMPORT-POLICY permit 20
 match community CUSTOMER-ROUTES
 set local-preference 150

route-map IMPORT-POLICY deny 100
 ! Implicit deny -- drop everything else
```

**Rules:**
- Lower sequence numbers are evaluated first
- `permit` with no match conditions matches everything
- An implicit `deny` exists at the end of every route map
- If no route map is attached, all routes are accepted/advertised (unless `bgp ebgp-requires-policy` is set)

### Route Map Actions Reference

| Set Action | Purpose |
|---|---|
| `set local-preference <0-4294967295>` | Override local preference |
| `set weight <0-4294967295>` | Set BGP weight (local only) |
| `set metric <0-4294967295>` | Set MED |
| `set as-path prepend <ASN> [ASN...]` | Prepend AS numbers to AS path |
| `set as-path prepend last-as <1-10>` | Prepend own AS N times |
| `set community <community> [additive]` | Set or add communities |
| `set large-community <community> [additive]` | Set or add large communities |
| `set extcommunity rt <RT>` | Set route target |
| `set origin <igp\|egp\|incomplete>` | Set origin attribute |
| `set ip next-hop <A.B.C.D>` | Override next-hop |
| `set ip next-hop peer-address` | Set next-hop to peer's address |
| `set ipv6 next-hop global <X:X::X:X>` | Override IPv6 global next-hop |
| `set tag <1-4294967295>` | Set route tag |

### Route Map Match Conditions

| Match Condition | Purpose |
|---|---|
| `match ip address prefix-list NAME` | Match destination prefix against prefix list |
| `match ip next-hop prefix-list NAME` | Match next-hop against prefix list |
| `match as-path WORD` | Match AS path against AS path access list |
| `match community WORD [exact-match]` | Match community against community list |
| `match large-community WORD [exact-match]` | Match large community |
| `match extcommunity WORD` | Match extended community |
| `match metric <0-4294967295>` | Match MED value |
| `match local-preference <0-4294967295>` | Match local preference |
| `match origin <igp\|egp\|incomplete>` | Match origin attribute |
| `match peer <A.B.C.D\|X:X::X:X>` | Match by peer address |
| `match tag <1-4294967295>` | Match route tag |
| `match rpki valid\|invalid\|notfound` | Match RPKI validation state |
| `match as-path-count <count>` | Match AS path length |

## Configure Prefix Lists

Prefix lists filter routes by prefix and prefix length.

### Create a Prefix List

```
ip prefix-list ALLOWED-PREFIXES seq 10 permit 10.0.0.0/8 le 24
ip prefix-list ALLOWED-PREFIXES seq 20 permit 172.16.0.0/12 le 24
ip prefix-list ALLOWED-PREFIXES seq 30 permit 192.168.0.0/16 le 24
ip prefix-list ALLOWED-PREFIXES seq 100 deny any
```

### Prefix List Syntax

```
ip prefix-list NAME seq SEQ permit|deny PREFIX [ge MIN] [le MAX]
```

| Parameter | Meaning |
|---|---|
| `PREFIX` | The network prefix to match (e.g., `10.0.0.0/8`) |
| `ge MIN` | Minimum prefix length (greater than or equal) |
| `le MAX` | Maximum prefix length (less than or equal) |
| `any` | Match any prefix |

### Examples

```
! Match exactly 10.0.0.0/24
ip prefix-list EXACT seq 10 permit 10.0.0.0/24

! Match 10.0.0.0/8 and all longer prefixes up to /24
ip prefix-list RANGE seq 10 permit 10.0.0.0/8 le 24

! Match any prefix with length /25 to /32 (block small allocations)
ip prefix-list SMALL-PREFIXES seq 10 deny 0.0.0.0/0 ge 25 le 32

! Default route only
ip prefix-list DEFAULT-ONLY seq 10 permit 0.0.0.0/0
```

### IPv6 Prefix Lists

```
ipv6 prefix-list V6-ALLOWED seq 10 permit 2001:db8::/32 le 48
ipv6 prefix-list V6-ALLOWED seq 100 deny any
```

### Show Prefix Lists

```
show ip prefix-list
show ip prefix-list ALLOWED-PREFIXES
show ip prefix-list ALLOWED-PREFIXES seq 10
show ipv6 prefix-list
```

## Configure AS Path Access Lists

AS path access lists filter routes based on the BGP AS path attribute using regular expressions.

### Create an AS Path Access List

```
bgp as-path access-list UPSTREAM-ONLY permit ^65002_
bgp as-path access-list UPSTREAM-ONLY deny .*
```

### AS Path Regex Reference

| Pattern | Matches |
|---|---|
| `^$` | Routes originated locally (empty AS path) |
| `^65002$` | Routes originated by AS 65002 only |
| `^65002_` | Routes that entered via AS 65002 |
| `_65002$` | Routes originated by AS 65002 (may have prepends) |
| `_65002_` | Routes that transited AS 65002 |
| `^65002_65003_` | Routes via AS 65002 then AS 65003 |
| `.*` | Any AS path (match all) |
| `^[0-9]+$` | Single-AS paths (direct customers) |

### Use in Route Map

```
route-map FILTER-TRANSIT deny 10
 match as-path TRANSIT-PATHS

route-map FILTER-TRANSIT permit 20

bgp as-path access-list TRANSIT-PATHS permit _65099_
```

### Show AS Path Access Lists

```
show bgp as-path-access-list
show bgp as-path-access-list UPSTREAM-ONLY
```

## Configure Distribute Lists

Distribute lists use standard or extended ACLs to filter routes. Prefix lists are preferred for new configurations, but distribute lists still work.

```
access-list 10 permit 10.0.0.0/8
access-list 10 deny any

router bgp 65001
 address-family ipv4 unicast
  neighbor 10.0.0.2 distribute-list 10 in
 exit-address-family
```

## Configure Filter Lists

Filter lists apply AS path access lists directly to a neighbor:

```
router bgp 65001
 address-family ipv4 unicast
  neighbor 10.0.0.2 filter-list CUSTOMER-PATHS in
 exit-address-family
```

## Configure Soft Reconfiguration

Soft reconfiguration stores received routes before policy is applied, allowing you to change inbound policy without resetting the session.

### Enable Soft Reconfiguration Inbound

```
router bgp 65001
 address-family ipv4 unicast
  neighbor 10.0.0.2 soft-reconfiguration inbound
 exit-address-family
```

**Note:** This uses additional memory because it stores a copy of all received routes before filtering.

### Trigger Soft Reconfiguration

After changing an inbound route map or prefix list:

```
! Re-apply inbound policy (safe -- no session reset)
clear bgp ipv4 unicast 10.0.0.2 soft in

! Re-apply outbound policy (safe -- no session reset)
clear bgp ipv4 unicast 10.0.0.2 soft out

! Re-apply both directions
clear bgp ipv4 unicast 10.0.0.2 soft
```

**WARNING:** `clear bgp ipv4 unicast 10.0.0.2` without the `soft` keyword performs a hard reset, tearing down the TCP session and causing route withdrawal.

### View Pre-Policy Routes

When soft-reconfiguration is enabled:

```
! Routes as received (before inbound policy)
show bgp ipv4 unicast neighbors 10.0.0.2 received-routes

! Routes after inbound policy
show bgp ipv4 unicast neighbors 10.0.0.2 routes

! Routes sent to the neighbor (after outbound policy)
show bgp ipv4 unicast neighbors 10.0.0.2 advertised-routes
```

## eBGP Policy Requirement

FRR enforces `bgp ebgp-requires-policy` by default. With this enabled, eBGP neighbors exchange zero routes unless a route map is attached in both directions.

### Disable (Testing Only)

```
router bgp 65001
 no bgp ebgp-requires-policy
```

### Recommended: Define Explicit Policies

```
! Permit all -- use as a starting point, then tighten
route-map ALLOW-ALL permit 10

router bgp 65001
 address-family ipv4 unicast
  neighbor 10.0.0.2 route-map ALLOW-ALL in
  neighbor 10.0.0.2 route-map ALLOW-ALL out
 exit-address-family
```

## RPKI Route Validation

Filter routes based on RPKI origin validation (requires RPKI cache server):

### Enable RPKI

```
rpki
 rpki cache tcp 10.0.0.100 3323 preference 1
 rpki polling_period 300
exit
```

### Match RPKI State in Route Map

```
route-map RPKI-FILTER deny 10
 match rpki invalid

route-map RPKI-FILTER permit 20
 match rpki valid
 set local-preference 200

route-map RPKI-FILTER permit 30
 match rpki notfound
```

### Verify RPKI

```
show rpki prefix-table
show rpki cache-connection
show bgp ipv4 unicast rpki valid
show bgp ipv4 unicast rpki invalid
show bgp ipv4 unicast rpki notfound
```

## Complete Policy Example

Full policy for a transit customer:

```
! Define allowed customer prefixes
ip prefix-list CUST-A-PREFIXES seq 10 permit 203.0.113.0/24
ip prefix-list CUST-A-PREFIXES seq 20 permit 198.51.100.0/24

! Define AS path filter (only customer's AS)
bgp as-path access-list CUST-A-AS permit ^65100$

! Inbound policy: accept only customer prefixes from correct AS
route-map CUST-A-IN permit 10
 match ip address prefix-list CUST-A-PREFIXES
 match as-path CUST-A-AS
 set local-preference 150
 set community 65001:1000

route-map CUST-A-IN deny 20

! Outbound policy: send full table
route-map CUST-A-OUT permit 10

! Apply to neighbor
router bgp 65001
 neighbor 10.0.0.10 remote-as 65100
 address-family ipv4 unicast
  neighbor 10.0.0.10 route-map CUST-A-IN in
  neighbor 10.0.0.10 route-map CUST-A-OUT out
  neighbor 10.0.0.10 prefix-list CUST-A-PREFIXES in
  neighbor 10.0.0.10 maximum-prefix 10 warning-only
  neighbor 10.0.0.10 send-community both
 exit-address-family
```

## Troubleshoot Policies

```
! Check if a route map exists and its contents
show route-map IMPORT-POLICY

! Check prefix list contents
show ip prefix-list ALLOWED-PREFIXES

! Check which routes match a route map (debug)
debug bgp updates in
debug bgp updates out

! See policy-rejected routes (requires soft-reconfig inbound)
show bgp ipv4 unicast neighbors 10.0.0.2 received-routes
show bgp ipv4 unicast neighbors 10.0.0.2 routes
! Compare the two -- routes in received but not in routes were filtered
```
