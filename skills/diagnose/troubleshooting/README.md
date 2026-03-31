# Troubleshooting

Symptom-based decision trees for diagnosing common FRR problems. Follow each tree top-to-bottom; stop at the first matching resolution.

## BGP Neighbor Not Established

```
show bgp summary
```

If neighbor is missing entirely:
  -> Verify `neighbor <IP> remote-as <ASN>` exists in `show running-config`
  -> If missing, add the neighbor configuration

If neighbor shows state `Idle`:
  -> Check `show bgp neighbor <IP>` for "Last reset reason"
  -> If "No route to host" -> verify reachability: `ping <IP>`, check `show ip route <IP>`
  -> If "Connection refused" -> verify bgpd is running on remote peer, TCP port 179 is open
  -> If "Peer-group member not configured" -> check peer-group assignment

If neighbor shows state `Connect` or `Active`:
  -> TCP session cannot establish
  -> Check firewall rules: `iptables -L -n | grep 179`
  -> Check update-source: does the source IP match what the remote peer expects?
    ```
    show bgp neighbor <IP> | include update-source
    ```
  -> If multihop eBGP: verify `neighbor <IP> ebgp-multihop <N>` is set on both sides
  -> If using TTL security: verify `neighbor <IP> ttl-security hops <N>` matches

If neighbor shows state `OpenSent`:
  -> BGP OPEN sent but no reply received
  -> Likely firewall blocking return traffic or remote peer not configured
  -> Check remote peer configuration

If neighbor shows state `OpenConfirm`:
  -> OPEN received, waiting for KEEPALIVE
  -> Check for AS number mismatch: `show bgp neighbor <IP> | include remote AS`
  -> Check for capability mismatch (4-byte ASN, multiprotocol)
  -> If password authentication: verify MD5 password matches on both sides
    ```
    show bgp neighbor <IP> | include password
    ```

If neighbor flaps (repeatedly goes up/down):
  -> Check hold timer: `show bgp neighbor <IP> | include timer`
  -> If hold time is too low (< 9s) with lossy link, increase it
  -> Check for route policy rejecting all routes causing session reset
  -> Check system resources: CPU, memory (high utilization causes missed keepalives)
  -> Check for MTU mismatch on path (causes TCP retransmits)

## OSPF Neighbor Stuck

```
show ip ospf neighbor
```

If neighbor is stuck in `Init`:
  -> One side sees the other but not vice versa
  -> Check ACLs or firewall blocking OSPF (IP protocol 89)
  -> Verify multicast: `show ip ospf interface <IFACE>` -- check network type
  -> On broadcast/NBMA networks, verify multicast group 224.0.0.5 membership

If neighbor is stuck in `2-Way`:
  -> Normal on broadcast/NBMA if neither side is DR/BDR
  -> If adjacency is expected: check OSPF network type
    ```
    show ip ospf interface <IFACE> | include Network Type
    ```
  -> Mismatched network types (e.g., one side broadcast, other point-to-point) prevent full adjacency

If neighbor is stuck in `ExStart` or `Exchange`:
  -> MTU mismatch is the most common cause
  -> Check MTU on both sides: `show interface <IFACE>` -- compare MTU values
  -> Fix: set matching MTU, or use `ip ospf mtu-ignore` (workaround, not recommended long-term)
  -> Check for duplicate Router IDs: `show ip ospf` on both routers

If neighbor is stuck in `Loading`:
  -> Database exchange incomplete
  -> Check for very large LSA databases: `show ip ospf database | include Total`
  -> Check for packet loss on the link (retransmit counters in `show ip ospf neighbor detail`)
  -> Check MTU if large LSAs are being fragmented and dropped

If neighbor never appears at all:
  -> Verify OSPF is enabled on the interface:
    ```
    show ip ospf interface <IFACE>
    ```
  -> If "No such interface": ensure `network <SUBNET> area <AREA>` covers the interface, or `ip ospf area <AREA>` is configured on the interface
  -> Verify area ID matches on both sides
  -> Verify hello/dead timers match:
    ```
    show ip ospf interface <IFACE> | include Timer
    ```
  -> Check authentication: type and key must match (plaintext, MD5, or HMAC-SHA)

## Routes Not Appearing

```
show ip route
show ip route <PREFIX>
```

If route exists in protocol table but not in RIB:
  -> Check administrative distance: another protocol with lower AD may be winning
    ```
    show ip route <PREFIX> longer-prefixes
    ```
  -> Check for route filtering: `show route-map`, `show ip prefix-list`

If route exists in RIB but not in FIB (kernel):
  -> Check zebra:
    ```
    show ip route <PREFIX>
    show zebra fpm stats
    ```
  -> Look for "not installed" or "queued" status
  -> Check if nexthop is resolvable: `show ip route <NEXTHOP_IP>`
  -> Verify zebra is running: `show zebra client summary`

If BGP route not appearing:
  -> Check if route is received: `show bgp ipv4 unicast neighbors <IP> received-routes`
  -> If received but not accepted: check inbound route-map/prefix-list/filter-list
    ```
    show bgp ipv4 unicast neighbors <IP> routes
    ```
  -> If "Filtered" appears: identify which filter and adjust
  -> If not received at all: check remote peer's outbound policy
  -> Verify address family is activated: `show bgp ipv4 unicast summary`

If OSPF route not appearing:
  -> Check LSA database: `show ip ospf database`
  -> Verify the route type (intra-area, inter-area, external)
  -> For external routes: verify redistribution on the ASBR
    ```
    show ip ospf database external
    ```
  -> Check for stub/NSSA area filtering external LSAs
  -> Check for `distribute-list` or `filter-list` blocking routes

If redistributed route not appearing:
  -> Verify redistribution is configured:
    ```
    show running-config | include redistribute
    ```
  -> Check route-map applied to redistribution for deny rules
  -> Verify the source route exists: `show ip route <PREFIX>`
  -> Check metric assignment for the target protocol

## High CPU

```
show cpu thread
show memory
```

If a specific daemon is consuming CPU:
  -> Identify the thread: `show cpu thread` in that daemon's vtysh context
  -> Common causes by daemon:
    - **bgpd**: large update bursts, route-map with regex on every update, massive table scan
    - **zebra**: route churn (many installs/deletes), FPM sync backlog
    - **ospfd**: SPF recalculation storm (flapping link), large LSA database

If bgpd CPU is high:
  -> Check for route oscillation: `show bgp ipv4 unicast statistics`
  -> Check update groups: `show bgp update-group` -- large groups with complex policies are expensive
  -> Examine route-map complexity -- AS-path regex `.*` patterns are expensive
  -> Consider `bgp bestpath as-path multipath-relax` to reduce computation

If zebra CPU is high:
  -> Check route install rate: `show zebra work-queue`
  -> Check for recursive nexthop resolution loops
  -> Check FPM connection (if used): `show zebra fpm stats`

If ospfd CPU is high:
  -> Check SPF frequency: `show ip ospf` -- look at SPF run count and timing
  -> If SPF runs too frequently: identify flapping interface
    ```
    show ip ospf database router | include Link State ID
    show logging | include OSPF
    ```
  -> Apply SPF throttle timers: `timers throttle spf <delay> <initial-hold> <max-hold>`

General high CPU:
  -> Check if debug logging is enabled: `show debugging` in each daemon
  -> **Disable debug logging in production** -- it is a common cause of high CPU
  -> Check system-level with `top -H -p $(pidof bgpd)` to see per-thread usage

## Daemon Crashing

Check crash logs first:
```
ls -lt /var/tmp/frr.*.crashlog
cat /var/tmp/frr.<daemon>.crashlog
```

If crash log exists:
  -> Read the backtrace in the crash log
  -> Look for the faulting function and address
  -> Check FRR version: `show version` -- search known issues at https://github.com/FRRouting/frr/issues

If core dump is available:
  -> Generate backtrace:
    ```
    gdb /usr/lib/frr/<daemon> /var/tmp/<core_file>
    (gdb) bt full
    (gdb) info threads
    (gdb) thread apply all bt
    ```
  -> If no core dump: enable core dumps (see debugging skill)

If crash is reproducible:
  -> Collect the exact sequence of events/commands that trigger it
  -> Run daemon with full debug and capture logs leading up to crash
  -> Check if a specific command triggers the crash: try running the command after restarting the daemon

If crash happens on startup:
  -> Check configuration syntax:
    ```
    vtysh -f /etc/frr/frr.conf --dryrun
    ```
  -> Check for corrupt state files in `/var/run/frr/`
  -> Try starting with minimal configuration

If crash happens under load:
  -> Check memory: `show memory` -- look for exhaustion
  -> Check for known memory leaks in the version
  -> Check OOM killer: `dmesg | grep -i "out of memory"`
  -> Consider setting memory limits: `memory-limit <MB>` in daemon config

## Routing Loops and Split-Brain

Symptoms: packets with TTL expired, traceroute shows cycling, traffic blackhole.

```
traceroute <DESTINATION>
show ip route <DESTINATION>
```

If traceroute shows a loop between two or more routers:
  -> Check the route table on each router in the loop path
  -> Identify which router has an incorrect next-hop
  -> Common causes:
    - Asymmetric redistribution (protocol A -> B on one router, B -> A on another)
    - Mismatched administrative distance overrides
    - Stale routes from a failed graceful restart

If redistribution loop is suspected:
  -> Check mutual redistribution:
    ```
    show running-config | include redistribute
    ```
  -> If two protocols redistribute into each other, apply route tags to prevent feedback:
    ```
    route-map REDISTRIBUTE-INTO-OSPF permit 10
      set tag 100
    route-map ACCEPT-FROM-OSPF deny 10
      match tag 100
    route-map ACCEPT-FROM-OSPF permit 20
    ```
  -> Use `distribute-list` or `prefix-list` to explicitly block re-learned routes

If split-brain (two routers both think they are authoritative):
  -> Check for network partitions that have healed, leaving stale state
  -> OSPF: check for duplicate Router IDs
    ```
    show ip ospf database router
    ```
  -> BGP: check for AS-path loops disabled (`allowas-in`) causing route reflection
  -> Verify that graceful-restart stale timers are not holding old routes too long:
    ```
    show bgp neighbor <IP> graceful-restart
    ```

If blackhole route present:
  -> Check for a Null0 or unreachable route:
    ```
    show ip route <PREFIX>
    ```
  -> Common source: aggregate with `summary-only` in BGP after component routes withdrawn
  -> Check for stale static routes pointing to down interfaces

General prevention:
  -> Always use route tags when doing mutual redistribution
  -> Set unique Router IDs on every router
  -> Use BFD for fast failure detection to minimize convergence windows
  -> Audit administrative distance values to ensure deterministic route selection
