# BGP FlowSpec

## Overview

FlowSpec (RFC 5575) uses BGP to distribute traffic filtering and policing rules. FRR implements FlowSpec as a **client only** -- it receives flow specification rules from BGP peers and programs them into the Linux kernel using netfilter (iptables/ipset) and policy-based routing.

Key facts:

- FRR cannot originate FlowSpec rules -- it only receives and applies them
- Rules are applied via Linux `iptables`, `ipset`, and PBR (`ip rule`/`ip route table`)
- Supports IPv4 and IPv6 FlowSpec address families
- Actions: redirect to VRF, redirect to IP, discard, rate-limit (via redirect)

## Prerequisites

The following Linux packages must be installed for FlowSpec to function:

- `iptables`
- `ipset`

Verify:

```bash
which iptables && which ipset
```

## Configure FlowSpec

### Enable the FlowSpec Address Family

```
router bgp 65001
 neighbor 10.0.0.2 remote-as 65002

 address-family ipv4 flowspec
  neighbor 10.0.0.2 activate
 exit-address-family
```

### IPv6 FlowSpec

```
router bgp 65001
 neighbor 2001:db8::2 remote-as 65002

 address-family ipv6 flowspec
  neighbor 2001:db8::2 activate
 exit-address-family
```

### Limit FlowSpec to Specific Interfaces

By default, FlowSpec rules apply to all interfaces. Restrict to specific interfaces:

```
router bgp 65001
 address-family ipv4 flowspec
  local-install eth0
 exit-address-family
```

Or allow all interfaces explicitly:

```
  local-install any
```

## FlowSpec Match Criteria

FlowSpec rules can match on combinations of these Layer 3/4 fields:

| Criterion | Description |
|---|---|
| Destination prefix | Destination IP address/prefix |
| Source prefix | Source IP address/prefix |
| IP protocol | Protocol number (TCP=6, UDP=17, ICMP=1, etc.) |
| Destination port | TCP/UDP destination port |
| Source port | TCP/UDP source port |
| ICMP type | ICMP type value |
| ICMP code | ICMP code value |
| TCP flags | TCP flag combinations (SYN, ACK, FIN, etc.) |
| Packet length | IP packet length |
| DSCP | Differentiated Services Code Point |
| Fragment | Fragmentation flags |

## FlowSpec Actions

| Action | Effect | Implementation |
|---|---|---|
| Redirect to VRF | Forward matching traffic to a different VRF | PBR with `ip rule` |
| Redirect to IP | Forward matching traffic to a specific nexthop | PBR with `ip route table` |
| Discard (rate=0) | Drop matching traffic | iptables DROP rule |
| Rate limit | Rate-limit matching traffic (limited support) | iptables with mark + tc |

## Configure VRF Redirect

To enable VRF redirection, import route targets under the target VRF:

```
router bgp 65001
 address-family ipv4 unicast
  rt redirect import 65002:100
 exit-address-family
```

Replace `65002:100` with the route target carried in the FlowSpec redirect extended community.

### IPv6 VRF Redirect

```
router bgp 65001
 address-family ipv6 unicast
  rt6 redirect import 2001:db8::1:100
 exit-address-family
```

## Full FlowSpec Configuration Example

### FlowSpec Client (Receiving Rules)

```
router bgp 65001
 bgp router-id 10.0.0.1
 neighbor 10.0.0.2 remote-as 65002
 neighbor 10.0.0.2 description "FlowSpec Controller"

 address-family ipv4 flowspec
  neighbor 10.0.0.2 activate
  local-install any
 exit-address-family

 address-family ipv4 unicast
  rt redirect import 65002:100
 exit-address-family
```

### What the Controller Sends

The FlowSpec controller (an external device or software) sends BGP UPDATE messages containing NLRI that encodes match/action rules. Example rules the controller might send:

- Discard all traffic to 203.0.113.1/32 from any source (DDoS blackhole)
- Rate-limit UDP traffic to 198.51.100.0/24 destination port 53 (DNS amplification)
- Redirect TCP traffic to 10.0.0.0/24 port 80 to VRF "scrubbing"

## Show FlowSpec Information

```
! View received FlowSpec rules
show bgp ipv4 flowspec
show bgp ipv4 flowspec detail

! View a specific flow rule
show bgp ipv4 flowspec 203.0.113.1/32

! IPv6 FlowSpec
show bgp ipv6 flowspec
show bgp ipv6 flowspec detail

! View installed PBR rules
show pbr ipset IPSETNAME
show pbr iptable

! View policy routing tables
show ip route table 100

! BGP FlowSpec summary
show bgp ipv4 flowspec summary
```

## Debug FlowSpec

```
! Enable FlowSpec debugging
debug bgp flowspec

! Debug PBR rule installation
debug bgp pbr
debug bgp pbr error

! Check iptables rules installed by FlowSpec
! (run from shell, not vtysh)
sudo iptables -L -n -v
sudo ipset list
sudo ip rule show
```

## Troubleshoot FlowSpec

| Symptom | Likely Cause | Fix |
|---|---|---|
| No FlowSpec rules received | Address family not activated | Add `neighbor X activate` under `address-family ipv4 flowspec` |
| Rules received but not applied | `iptables`/`ipset` not installed | Install packages and restart FRR |
| VRF redirect not working | Route target not imported | Add `rt redirect import` under ipv4 unicast |
| Rules applied to wrong interfaces | `local-install` not configured | Specify target interface with `local-install <ifname>` |
| Rules not matching traffic | Match criteria too specific | Check `show bgp ipv4 flowspec detail` for exact match fields |

## Limitations

- FRR is a FlowSpec **client only** -- it cannot originate flow rules
- Rate limiting has limited support (implemented as redirect without actual shaping in some cases)
- FlowSpec rules depend on Linux netfilter -- kernel must support iptables and ipset
- Performance depends on the number of rules and the Linux netfilter subsystem
- No nftables support -- requires legacy iptables
