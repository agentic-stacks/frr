# BGP Route Reflectors

## Overview

In a full-mesh iBGP topology, every router must peer with every other router. This does not scale: N routers require N*(N-1)/2 sessions. Route reflectors (RR) solve this by allowing a single router to reflect iBGP-learned routes to other iBGP peers, breaking the full-mesh requirement.

Key concepts:

- **Route reflector (RR):** An iBGP router configured to reflect routes to its clients
- **RR client:** An iBGP peer that receives reflected routes from the RR
- **Non-client:** An iBGP peer that is not a client (standard iBGP peering rules apply)
- **Cluster:** The RR and all its clients form a cluster, identified by a cluster-id

## Configure a Route Reflector

### Designate Clients

On the RR, mark each client neighbor:

```
router bgp 65001
 bgp router-id 10.0.0.1
 bgp cluster-id 10.0.0.1

 neighbor 10.0.0.2 remote-as 65001
 neighbor 10.0.0.2 update-source lo
 neighbor 10.0.0.3 remote-as 65001
 neighbor 10.0.0.3 update-source lo

 address-family ipv4 unicast
  neighbor 10.0.0.2 route-reflector-client
  neighbor 10.0.0.3 route-reflector-client
 exit-address-family
```

Replace `10.0.0.1` with the RR's loopback/router-id, and `10.0.0.2`/`10.0.0.3` with the client loopback addresses.

### Client Configuration

RR clients require **no special configuration** -- they peer with the RR as a normal iBGP neighbor:

```
router bgp 65001
 bgp router-id 10.0.0.2
 neighbor 10.0.0.1 remote-as 65001
 neighbor 10.0.0.1 update-source lo

 address-family ipv4 unicast
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.1 next-hop-self
 exit-address-family
```

## Configure Cluster ID

The cluster-id identifies the RR cluster for loop prevention. By default, the router-id is used.

```
router bgp 65001
 bgp cluster-id 1.1.1.1
```

Set a custom cluster-id when:

- Multiple RRs serve the same cluster (they must share the same cluster-id)
- You want to distinguish between clusters in a hierarchical design

## Route Reflection Rules

The RR follows these reflection rules:

| Route Learned From | Reflected To |
|---|---|
| eBGP peer | All clients and non-clients |
| RR client | All other clients and all non-clients |
| Non-client (iBGP) | All clients only (not to other non-clients) |

### Loop Prevention

BGP uses two mechanisms to prevent loops in RR topologies:

1. **Originator-ID:** Set to the router-id of the route originator. If a router receives a route with its own router-id as originator-id, it discards the route.
2. **Cluster-list:** Each RR prepends its cluster-id to the cluster-list. If a router sees its own cluster-id in the cluster-list, it discards the route.

## Design Patterns

### Single RR (Small Network)

Suitable for up to ~20 iBGP routers.

```
          +--------+
          |  RR    |
          | 10.0.0.1|
          +---+----+
         /    |    \
        /     |     \
  +----+  +---+--+ +----+
  |R2  |  |R3    | |R4  |
  +----+  +------+ +----+
  (client) (client) (client)
```

All routers peer only with the RR. The RR reflects all client routes to all other clients.

### Redundant RRs (Recommended)

Deploy two RRs with the same cluster-id for redundancy:

```
router bgp 65001
 ! On RR-1 (10.0.0.1)
 bgp cluster-id 10.0.0.100

 ! On RR-2 (10.0.0.5)
 bgp cluster-id 10.0.0.100
```

Both RRs must share the same cluster-id. Clients peer with both:

```
! Client config
router bgp 65001
 neighbor 10.0.0.1 remote-as 65001
 neighbor 10.0.0.1 update-source lo
 neighbor 10.0.0.5 remote-as 65001
 neighbor 10.0.0.5 update-source lo

 address-family ipv4 unicast
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.5 activate
 exit-address-family
```

### Hierarchical RRs (Large Network)

For very large networks, use a two-tier RR hierarchy:

```
         +--------+  +--------+
         | Top-RR1|  | Top-RR2|
         +---+----+  +---+----+
        /    |            |    \
       /     |            |     \
  +---+--+ +-+----+  +---+--+ +-+----+
  |Mid-RR1| |Mid-RR2| |Mid-RR3| |Mid-RR4|
  +---+---+ +---+---+ +---+---+ +---+---+
     / \       / \        / \       / \
   (clients) (clients) (clients) (clients)
```

- Top-tier RRs peer with each other (full mesh between top RRs)
- Mid-tier RRs are clients of top-tier RRs
- Leaf routers are clients of mid-tier RRs
- Each tier has its own cluster-id

Example for a mid-tier RR that is both a client (to the top) and an RR (to leaves):

```
router bgp 65001
 bgp cluster-id 10.0.1.100

 ! Upstream: peer with top-tier RRs as a normal iBGP peer
 neighbor 10.0.0.1 remote-as 65001
 neighbor 10.0.0.1 update-source lo
 neighbor 10.0.0.5 remote-as 65001
 neighbor 10.0.0.5 update-source lo

 ! Downstream: leaf routers as RR clients
 neighbor 10.0.1.1 remote-as 65001
 neighbor 10.0.1.1 update-source lo
 neighbor 10.0.1.2 remote-as 65001
 neighbor 10.0.1.2 update-source lo

 address-family ipv4 unicast
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.5 activate
  neighbor 10.0.1.1 route-reflector-client
  neighbor 10.0.1.2 route-reflector-client
 exit-address-family
```

## Route Reflector as Route Server (Non-Forwarding)

When the RR does not need to forward traffic (out-of-band RR), start bgpd with `-n` to skip kernel route installation:

```
# /etc/frr/daemons
bgpd_options="--daemon -A 127.0.0.1 -n"
```

This reduces memory and CPU usage on the RR since it only needs to process BGP control plane.

## Allow Outbound Policy on RR

By default, outbound route maps on RR clients are ignored for reflected routes. To enable policy on reflected routes:

```
router bgp 65001
 bgp route-reflector allow-outbound-policy
```

**WARNING:** This changes the default RR behavior. Use only when you need to filter or modify attributes on reflected routes (e.g., attaching communities to routes sent to specific clients).

## RR with Multiple Address Families

An RR can reflect routes for multiple address families. Configure `route-reflector-client` in each family:

```
router bgp 65001
 bgp cluster-id 10.0.0.100

 address-family ipv4 unicast
  neighbor 10.0.0.2 route-reflector-client
  neighbor 10.0.0.3 route-reflector-client
 exit-address-family

 address-family ipv6 unicast
  neighbor 10.0.0.2 route-reflector-client
  neighbor 10.0.0.3 route-reflector-client
 exit-address-family

 address-family l2vpn evpn
  neighbor 10.0.0.2 route-reflector-client
  neighbor 10.0.0.3 route-reflector-client
 exit-address-family
```

## Verify Route Reflector

```
! Check cluster-id in neighbor details
show bgp neighbors 10.0.0.2

! Look for "Route-Reflector Client" in output
show bgp neighbors 10.0.0.2 json

! Verify reflected routes have originator-id and cluster-list
show bgp ipv4 unicast 192.168.1.0/24

! Check all RR clients
show bgp peer-group
```

### Verify Loop Prevention

A route with cluster-list and originator-id attributes indicates it was reflected:

```
show bgp ipv4 unicast 192.168.1.0/24
```

Look for:
- `Originator: 10.0.0.2` -- the original advertiser
- `Cluster list: 10.0.0.100` -- the cluster(s) it passed through

## Troubleshoot Route Reflectors

| Symptom | Likely Cause | Fix |
|---|---|---|
| Client not receiving routes | `route-reflector-client` not configured in the address family | Add it under the correct `address-family` block |
| Route loop detected | Duplicate cluster-id across separate clusters | Ensure each cluster has a unique cluster-id |
| Suboptimal routing | RR selects a path the client cannot use | Enable `bgp bestpath compare-routerid` or add-path |
| Missing routes after RR failover | Clients only peered with one RR | Add redundant RR with same cluster-id |

### Add-Path to Fix Suboptimal Routing

The RR normally advertises only its best path. To send all paths to clients:

```
router bgp 65001
 address-family ipv4 unicast
  neighbor 10.0.0.2 addpath-tx-all-paths
 exit-address-family
```

Or send only the best path per originating AS:

```
  neighbor 10.0.0.2 addpath-tx-bestpath-per-AS
```

This allows clients to make independent best-path decisions, solving the "RR hidden path" problem.
