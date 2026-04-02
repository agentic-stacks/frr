# BGP Confederations

## Overview

A BGP confederation splits a single AS into multiple sub-ASes internally while appearing as a single AS to external peers. Like route reflectors, confederations solve the iBGP full-mesh scaling problem, but through a different mechanism.

Key concepts:

- **Confederation identifier:** The AS number visible to the outside world
- **Sub-AS (member AS):** Internal AS numbers used between confederation members
- **Confederation eBGP (confed-eBGP):** Peering between different sub-ASes (uses eBGP mechanics but preserves iBGP attributes)
- **Confederation iBGP (confed-iBGP):** Peering within a sub-AS (standard iBGP rules)

## Configure a Confederation

### On Each Router

```
router bgp 65010
 bgp confederation identifier 65001
 bgp confederation peers 65020 65030
```

Replace:
- `65010` -- this router's sub-AS number
- `65001` -- the confederation identifier (what external peers see)
- `65020 65030` -- the other sub-AS numbers in the confederation

### Full Example: Three Sub-ASes

**Router R1 (sub-AS 65010):**

```
router bgp 65010
 bgp router-id 10.0.0.1
 bgp confederation identifier 65001
 bgp confederation peers 65020 65030

 ! iBGP within sub-AS 65010
 neighbor 10.0.0.2 remote-as 65010
 neighbor 10.0.0.2 update-source lo

 ! Confed-eBGP to sub-AS 65020
 neighbor 10.0.1.1 remote-as 65020

 address-family ipv4 unicast
  neighbor 10.0.0.2 activate
  neighbor 10.0.0.2 next-hop-self
  neighbor 10.0.1.1 activate
  neighbor 10.0.1.1 next-hop-self
 exit-address-family
```

**Router R3 (sub-AS 65020):**

```
router bgp 65020
 bgp router-id 10.0.1.1
 bgp confederation identifier 65001
 bgp confederation peers 65010 65030

 ! Confed-eBGP to sub-AS 65010
 neighbor 10.0.0.1 remote-as 65010

 address-family ipv4 unicast
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.1 next-hop-self
 exit-address-family
```

## Confederation Behavior

### AS Path Handling

- Sub-AS numbers appear in the AS path internally but are stripped when routes are advertised to true eBGP peers
- External peers see only the confederation identifier in the AS path
- Within the confederation, the sub-AS path is used for loop prevention

### Attribute Preservation

Confed-eBGP sessions preserve these iBGP attributes (unlike true eBGP):

| Attribute | Confed-eBGP Behavior |
|---|---|
| Local preference | Preserved across sub-ASes |
| MED | Preserved across sub-ASes |
| Next-hop | Not changed automatically (use `next-hop-self`) |

### Best Path Considerations

```
router bgp 65010
 ! Treat confed-eBGP AS path segments like iBGP for path length comparison
 bgp bestpath as-path confed
```

Without this, confederation AS path segments are ignored in best-path calculation. Enable it when you want path length to influence selection across sub-ASes.

## When to Use Confederations vs Route Reflectors

| Factor | Route Reflectors | Confederations |
|---|---|---|
| Configuration complexity | Lower -- only RR needs config | Higher -- every router needs confed config |
| Operational simplicity | Simpler -- clients need no special config | More complex -- sub-AS assignment planning required |
| Optimal routing | Can cause suboptimal paths (RR selects one best) | Better path diversity between sub-ASes |
| Scalability | Excellent with hierarchical design | Good, but limited by sub-AS topology |
| Migration effort | Easy to add incrementally | Requires coordinated deployment |
| Common use case | Most deployments, data centers | Large ISPs with natural geographic/administrative divisions |

**Recommendation:** Use route reflectors unless you have a specific reason to prefer confederations (e.g., multiple administrative domains within one AS, or strict need for optimal path selection across regions).

## Confederations with External Peers

External peers see the confederation identifier, not sub-AS numbers:

```
! On an external router peering with the confederation
router bgp 65002
 neighbor 10.0.0.1 remote-as 65001
```

The external peer uses `65001` (the confederation identifier), not the sub-AS number.

## Verify Confederation

```
! Check confederation settings
show bgp summary

! Look for confed identifier and peers in output
show bgp neighbors 10.0.1.1

! Check AS path -- confed segments appear in parentheses
show bgp ipv4 unicast 192.168.1.0/24
```

In `show bgp` output, confederation AS path segments appear in parentheses:

```
*> 192.168.1.0/24   10.0.1.1   0   0 (65020 65030) i
```

The `(65020 65030)` indicates the route traversed sub-ASes 65020 and 65030 within the confederation.

## Combine Confederations with Route Reflectors

Within each sub-AS, you can use route reflectors to avoid the iBGP full mesh:

```
! RR within sub-AS 65010
router bgp 65010
 bgp confederation identifier 65001
 bgp confederation peers 65020

 bgp cluster-id 10.0.0.100

 neighbor 10.0.0.2 remote-as 65010
 neighbor 10.0.0.2 update-source lo
 neighbor 10.0.0.3 remote-as 65010
 neighbor 10.0.0.3 update-source lo

 address-family ipv4 unicast
  neighbor 10.0.0.2 route-reflector-client
  neighbor 10.0.0.3 route-reflector-client
 exit-address-family
```

This is the most scalable design -- confederations between regions, route reflectors within each region.

## Troubleshoot Confederations

| Symptom | Likely Cause | Fix |
|---|---|---|
| No routes exchanged between sub-ASes | Missing `bgp confederation peers` on one side | Add the remote sub-AS to `bgp confederation peers` |
| External peer sees sub-AS in path | Confederation identifier mismatch | Ensure all routers use the same `bgp confederation identifier` |
| Local preference lost between sub-ASes | Peering configured as true eBGP instead of confed-eBGP | Verify both sides have matching `bgp confederation identifier` |
| Routing loops within confederation | AS path confed not enabled | Enable `bgp bestpath as-path confed` |
