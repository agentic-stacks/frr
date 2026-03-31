# BGP Sessions

## Configure an eBGP Neighbor

```
router bgp 65001
 neighbor 10.0.0.2 remote-as 65002
 address-family ipv4 unicast
  neighbor 10.0.0.2 activate
 exit-address-family
```

Replace `10.0.0.2` with the neighbor's IP and `65002` with their AS number. The `activate` command enables route exchange in that address family.

### Use Auto-Detection for AS Type

Instead of specifying the exact AS number:

```
router bgp 65001
 ! Peer is in a different AS (eBGP)
 neighbor 10.0.0.2 remote-as external

 ! Peer is in the same AS (iBGP)
 neighbor 10.0.0.3 remote-as internal
```

## Configure an iBGP Neighbor

```
router bgp 65001
 neighbor 10.0.0.3 remote-as 65001
 neighbor 10.0.0.3 update-source lo
 address-family ipv4 unicast
  neighbor 10.0.0.3 activate
  neighbor 10.0.0.3 next-hop-self
 exit-address-family
```

Key differences from eBGP:

- Same AS number on both sides
- `update-source lo` -- use loopback for the TCP session (survives link failures)
- `next-hop-self` -- rewrite the next-hop to your own address (required when the eBGP next-hop is not reachable by iBGP peers)

## Configure Peer Groups

Peer groups apply common settings to multiple neighbors, reducing configuration and improving update efficiency.

```
router bgp 65001
 neighbor UPSTREAM peer-group
 neighbor UPSTREAM remote-as external
 neighbor UPSTREAM timers 10 30
 neighbor UPSTREAM route-map IMPORT-UPSTREAM in
 neighbor UPSTREAM route-map EXPORT-UPSTREAM out
 neighbor UPSTREAM send-community both

 neighbor 10.0.0.2 peer-group UPSTREAM
 neighbor 10.0.0.4 peer-group UPSTREAM
 neighbor 10.0.0.6 peer-group UPSTREAM
```

Replace `UPSTREAM` with a descriptive group name. Individual neighbor settings override the peer group.

### Show Peer Groups

```
show bgp peer-group
show bgp peer-group UPSTREAM
```

## Configure Dynamic Neighbors

Accept BGP sessions from any IP within a range, automatically assigning them to a peer group:

```
router bgp 65001
 neighbor LEAF peer-group
 neighbor LEAF remote-as external

 bgp listen range 10.0.0.0/24 peer-group LEAF
 bgp listen limit 100
```

Replace `10.0.0.0/24` with the subnet range of expected peers. The `limit` caps the number of dynamic sessions (default: 100).

## Configure Timers

### Per-Neighbor Timers

```
router bgp 65001
 ! keepalive 10s, hold 30s
 neighbor 10.0.0.2 timers 10 30

 ! Connect retry timer (seconds between reconnection attempts)
 neighbor 10.0.0.2 timers connect 30

 ! DelayOpen -- wait before sending OPEN (useful for passive peers)
 neighbor 10.0.0.2 timers delayopen 5

 ! Minimum interval between route advertisements
 neighbor 10.0.0.2 advertisement-interval 5
```

### Global Timers

```
router bgp 65001
 timers bgp 10 30
```

Sets the default keepalive (10s) and hold time (30s) for all peers. Per-neighbor timers override this.

### Timer Reference

| Timer | Default | Range | Purpose |
|---|---|---|---|
| Hold time | 180s | 0, 3-65535 | Peer declared dead if no message received |
| Keepalive | 60s | 0-65535 | Interval between keepalive messages |
| Connect retry | 120s | 1-65535 | Delay between TCP connection attempts |
| DelayOpen | disabled | 1-240 | Wait before sending OPEN message |
| Advertisement interval | 0s | 0-600 | Min time between UPDATE messages to a peer |

**Note:** Setting hold time to 0 disables keepalive/hold checking entirely. Both sides must agree on this.

## Configure Authentication

### TCP MD5 Authentication

```
router bgp 65001
 neighbor 10.0.0.2 password MySecretKey123
```

Both sides must configure the same password. This enables TCP MD5 signature (RFC 2385) which protects against spoofed TCP resets.

**WARNING:** Changing or removing the password on one side while the other side still has it configured will break the session.

## Configure eBGP Multihop

By default, eBGP peers must be directly connected (TTL=1). For loopback-to-loopback peering:

```
router bgp 65001
 neighbor 10.0.0.2 remote-as 65002
 neighbor 10.0.0.2 ebgp-multihop 2
 neighbor 10.0.0.2 update-source lo
```

Replace `2` with the maximum number of hops to the peer.

### Alternative: TTL Security

TTL security is more secure than multihop -- it sets minimum TTL instead of maximum:

```
router bgp 65001
 neighbor 10.0.0.2 ttl-security hops 1
```

This protects against remote attacks by requiring packets to have TTL >= 254 (i.e., the peer is 1 hop away).

**Note:** `ttl-security` and `ebgp-multihop` are mutually exclusive.

## Configure BFD Integration

BFD (Bidirectional Forwarding Detection) provides sub-second failure detection for BGP sessions.

### Prerequisite

Enable `bfdd` in `/etc/frr/daemons`:

```
bfdd=yes
```

### Enable BFD on a Neighbor

```
router bgp 65001
 neighbor 10.0.0.2 bfd
```

### Enable BFD on a Peer Group

```
router bgp 65001
 neighbor UPSTREAM peer-group
 neighbor UPSTREAM bfd
```

### BFD with Custom Profile

```
router bgp 65001
 neighbor 10.0.0.2 bfd profile FAST

bfd
 profile FAST
  detect-multiplier 3
  receive-interval 100
  transmit-interval 100
 exit
exit
```

### Verify BFD

```
show bfd peers
show bfd peers brief
```

## Configure Graceful Restart

Graceful restart allows a router to maintain forwarding during a BGP daemon restart, preventing traffic loss.

### Enable Globally

```
router bgp 65001
 bgp graceful-restart
 bgp graceful-restart preserve-fw-state
 bgp graceful-restart restart-time 120
 bgp graceful-restart stalepath-time 360
 bgp graceful-restart select-defer-time 360
 bgp graceful-restart rib-stale-time 500
```

| Parameter | Default | Purpose |
|---|---|---|
| `restart-time` | 120s | Time the restarting router expects to complete restart |
| `stalepath-time` | 360s | How long peers keep stale routes from the restarting router |
| `select-defer-time` | 360s | Delay best-path selection after restart |
| `rib-stale-time` | 500s | How long stale routes remain in RIB |
| `preserve-fw-state` | disabled | Tell zebra to preserve forwarding entries during restart |

### Per-Neighbor Graceful Restart

```
router bgp 65001
 ! This neighbor participates in graceful restart
 neighbor 10.0.0.2 graceful-restart

 ! This neighbor only helps others restart (does not restart itself)
 neighbor 10.0.0.3 graceful-restart-helper

 ! Disable graceful restart for this neighbor
 neighbor 10.0.0.4 graceful-restart-disable
```

### Long-Lived Graceful Restart

For peers that may be down for extended periods (e.g., remote sites):

```
router bgp 65001
 bgp long-lived-graceful-restart stale-time 86400
```

### Verify Graceful Restart

```
show bgp neighbors 10.0.0.2 graceful-restart
show bgp ipv4 unicast neighbors 10.0.0.2 graceful-restart json
```

## Configure Graceful Shutdown

Graceful shutdown signals to all peers that this router is about to go offline, allowing traffic to be rerouted before the session drops.

### Shut Down the Entire Router

```
router bgp 65001
 bgp shutdown message "Planned maintenance window"
```

This sets the GRACEFUL_SHUTDOWN well-known community on all advertised routes, causing peers to de-prefer them. Remove with `no bgp shutdown`.

### Shut Down a Single Neighbor

```
router bgp 65001
 neighbor 10.0.0.2 shutdown message "Link maintenance"
```

Remove with `no neighbor 10.0.0.2 shutdown`.

### Graceful Shutdown with Route Map (Peer Side)

Peers should honor the GRACEFUL_SHUTDOWN community by lowering local-pref:

```
route-map GRACEFUL-SHUTDOWN permit 10
 match community GSHUT
 set local-preference 0

route-map GRACEFUL-SHUTDOWN permit 20

bgp community-list standard GSHUT permit graceful-shutdown

router bgp 65002
 address-family ipv4 unicast
  neighbor 10.0.0.1 route-map GRACEFUL-SHUTDOWN in
 exit-address-family
```

## Configure Local-AS (AS Migration)

When migrating between AS numbers, `local-as` lets you appear as the old AS to specific peers:

```
router bgp 65001
 neighbor 10.0.0.2 local-as 65099 no-prepend replace-as
```

| Option | Effect |
|---|---|
| (none) | Prepend local-as AND real AS to outbound updates |
| `no-prepend` | Do not prepend local-as to inbound AS path |
| `no-prepend replace-as` | Replace real AS with local-as in outbound updates |
| `no-prepend replace-as dual-as` | Accept connections from peers configured with either AS |

## Configure Passive Mode

Make a neighbor wait for the remote side to initiate the TCP connection:

```
router bgp 65001
 neighbor 10.0.0.2 passive
```

Useful when only one side should initiate (e.g., dynamic neighbors, hub-spoke topologies).

## Configure Neighbor Description

```
router bgp 65001
 neighbor 10.0.0.2 description "Transit: Provider-A circuit-12345"
```

Descriptions appear in `show bgp neighbors` output and help with operational clarity.

## Verify Sessions

```
! All peers with state and prefix counts
show bgp summary

! Detailed single-peer info (state, timers, capabilities, counters)
show bgp neighbors 10.0.0.2

! Check specific capabilities exchanged
show bgp neighbors 10.0.0.2 json | grep -A5 capability

! Quick one-liner per peer
show bgp neighbors brief
```

### Common Session States

| State | Meaning | Action |
|---|---|---|
| Idle | No connection attempt | Check remote-as, IP reachability, password |
| Connect | TCP SYN sent, waiting for response | Check firewall, port 179 accessibility |
| Active | TCP connection failed, retrying | Check IP reachability, remote side config |
| OpenSent | OPEN message sent, waiting for peer's OPEN | Check AS number, router-id conflicts |
| OpenConfirm | OPEN received, waiting for KEEPALIVE | Usually transitions quickly to Established |
| Established | Session is up and exchanging routes | Normal operating state |

### Troubleshooting Session Problems

```
! Enable BGP neighbor debug
debug bgp neighbor-events
debug bgp updates in
debug bgp updates out
debug bgp keepalives

! Check logs
show logging

! Disable debug when done
no debug bgp neighbor-events
no debug bgp updates in
no debug bgp updates out
no debug bgp keepalives
```
