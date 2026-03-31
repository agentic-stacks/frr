# Known Issues — FRR 9.x

Covers FRR 9.0.0 through 9.1.3. Always run the latest maintenance release (9.1.3).

---

## Security Vulnerabilities

### CVE-2023-47235 — BGP Malformed Attribute Crash

- **Symptom**: bgpd crashes when receiving crafted BGP UPDATE messages with malformed attributes
- **Cause**: Insufficient validation of BGP UPDATE attribute lengths
- **Workaround**: Apply prefix-list and AS path filters on all eBGP sessions; upgrade immediately
- **Affected Versions**: 9.0.0, 9.0.1
- **Fixed In**: 9.0.2
- **CVE**: CVE-2023-47235
- **References**: https://frrouting.org/security/cve-2023-47235

### CVE-2024-31948 — BGP Tunnel Encapsulation Crash

- **Symptom**: bgpd crash when processing BGP UPDATE with tunnel encapsulation attribute
- **Cause**: Missing bounds check on tunnel encapsulation attribute parsing
- **Workaround**: Upgrade to patched release
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.0.3, 9.1.1
- **CVE**: CVE-2024-31948
- **References**: https://frrouting.org/security/cve-2024-31948

### CVE-2024-31949 — BGP Crash on Malformed Packet

- **Symptom**: bgpd crash on specially crafted BGP message
- **Cause**: Improper error handling in BGP message processing
- **Workaround**: Upgrade to patched release
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.0.3, 9.1.1
- **CVE**: CVE-2024-31949
- **References**: https://frrouting.org/security/cve-2024-31949

### CVE-2024-31950 — BGP Heap Use-After-Free

- **Symptom**: bgpd heap use-after-free during best path selection
- **Cause**: Race condition in `bgp_best_selection()` freeing memory prematurely
- **Workaround**: Upgrade to patched release
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.0.3, 9.1.1
- **CVE**: CVE-2024-31950
- **References**: https://frrouting.org/security/cve-2024-31950

### CVE-2024-31951 — BGP ecommunity Buffer Overflow

- **Symptom**: bgpd heap-buffer-overflow in `ecommunity_fill_pbr_action`
- **Cause**: Missing length validation for extended community PBR actions
- **Workaround**: Upgrade to patched release
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.0.3, 9.1.1
- **CVE**: CVE-2024-31951
- **References**: https://frrouting.org/security/cve-2024-31951

---

## BGP Issues

### CRASH — bgpd Crash with default-originate Withdrawing Non-Default Routes

- **Symptom**: Neighbor loses routes other than the default when `default-originate` is configured
- **Cause**: default-originate logic incorrectly withdrew non-default routes
- **Workaround**: Remove `default-originate` or upgrade
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.1.1

### DATA-LOSS — Aggregate-Address summary-only Suppressed Export to EVPN

- **Symptom**: Routes suppressed by `aggregate-address summary-only` were not properly exported to EVPN
- **Cause**: Aggregation suppression incorrectly applied to EVPN address family
- **Workaround**: Avoid `summary-only` with EVPN or upgrade
- **Affected Versions**: 9.0.0, 9.0.1
- **Fixed In**: 9.0.2, 9.1.1

### CRASH — bgpd Heap Use-After-Free in Best Path Selection

- **Symptom**: bgpd segfault during convergence under load
- **Cause**: `bgp_best_selection()` accessed freed memory when paths changed during evaluation
- **Workaround**: Upgrade; reduce churn during maintenance windows
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.0.2, 9.1.1

### DATA-LOSS — VRF Leaking Broken with `no bgp network import-check`

- **Symptom**: Leaked routes between VRFs disappear
- **Cause**: Incorrect interaction between import-check disable and VRF route leaking
- **Workaround**: Keep `bgp network import-check` enabled when using VRF leaking, or upgrade
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.1.1

### OPERATIONAL — Dynamic Peer Graceful Restart Race Condition

- **Symptom**: Dynamic BGP peers fail graceful restart intermittently
- **Cause**: Race condition in peer state machine during GR negotiation with dynamic peers
- **Workaround**: Use static peer definitions for sessions requiring GR
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.1.1

### OPERATIONAL — `match peer` Broken When Switching Between IPv4/IPv6/Interface

- **Symptom**: Route map `match peer` clause stops matching after changing the peer type
- **Cause**: Internal data structure not updated on peer type change
- **Workaround**: Remove and re-add the route-map entry after changing peer type
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.1.1

---

## OSPF Issues

### CRASH — ospfd Infinite Loop When Listing Interfaces

- **Symptom**: ospfd hangs (100% CPU) when executing `show ip ospf interface`
- **Cause**: Infinite loop in interface listing code
- **Workaround**: Avoid the show command; upgrade
- **Affected Versions**: 9.0.0, 9.0.1
- **Fixed In**: 9.0.2

### CRASH — ospfd Crash in TE Parsing with Router Information

- **Symptom**: ospfd crash when TE and Router Information LSAs interact
- **Cause**: Null pointer dereference in `ri_parsing` with OSPF-TE enabled
- **Workaround**: Disable TE or Router Information, or upgrade
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.1.1

---

## IS-IS Issues

### CRASH — isisd Heap Use-After-Free in SPF Tree Deletion

- **Symptom**: isisd crash during SPF recalculation
- **Cause**: SPF tree freed while still referenced
- **Workaround**: Upgrade
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.1.1

### CRASH — isisd Heap-After-Free with Prefix SID

- **Symptom**: isisd crash when configuring or removing prefix SIDs
- **Cause**: Use-after-free in prefix SID handling
- **Workaround**: Avoid dynamic prefix SID changes, or upgrade
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.0.3, 9.1.1

---

## PIM Issues

### CRASH — pimd Crash When Unconfiguring RP Keepalive Timer

- **Symptom**: pimd segfault when removing RP keepalive timer configuration
- **Cause**: Null pointer dereference during timer cleanup
- **Workaround**: Restart pimd after configuration change, or upgrade
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.1.1

### CRASH — pimd Crash When Mixing SSM/Any-Source Joins

- **Symptom**: pimd crash when SSM and ASM joins coexist on an interface
- **Cause**: Incorrect handling of mixed multicast mode joins
- **Workaround**: Do not mix SSM and ASM on the same interface, or upgrade
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.1.1

---

## NHRP Issues

### CRASH — nhrpd Core Dump on Shutdown

- **Symptom**: nhrpd crashes during clean shutdown
- **Cause**: Race condition in cleanup path
- **Workaround**: Restart rather than graceful stop; upgrade
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.0.3, 9.1.1

---

## Zebra Issues

### CRASH — zebra Crash on macvlan Link in Another Netns

- **Symptom**: zebra segfaults when a macvlan interface exists in a different network namespace
- **Cause**: Null pointer dereference when macvlan parent is in another namespace
- **Workaround**: Keep macvlan and parent in the same namespace, or upgrade
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.1.1

### MEMORY — zebra FPM Netlink Return Path Memory Leak

- **Symptom**: Gradual memory growth in zebra when using FPM with Netlink
- **Cause**: Memory allocated for FPM return path not freed
- **Workaround**: Periodically restart zebra, or upgrade
- **Affected Versions**: 9.0.x, 9.1.0
- **Fixed In**: 9.1.1

---

## libyang Compatibility

### OPERATIONAL — Prefix-List Matching Broken with libyang 2.1.111

- **Symptom**: Route-map prefix-list matching fails silently; routes not matched
- **Cause**: Regression in libyang 2.1.111 breaks YANG-based prefix-list evaluation
- **Workaround**: Downgrade libyang to 2.1.80
- **Affected Versions**: All 9.x when used with libyang 2.1.111
- **Fixed In**: Use libyang 2.1.80 or >= 2.1.128
- **References**: https://github.com/CESNET/libyang/issues/2090

---

## Tools Issues

### OPERATIONAL — frr-reload.py Breaks Interface Descriptions

- **Symptom**: `frr-reload.py` incorrectly handles `no description` commands, may remove or duplicate them
- **Cause**: Parser does not properly track interface description state
- **Workaround**: Manually verify interface descriptions after reload, or upgrade
- **Affected Versions**: 9.0.0, 9.0.1
- **Fixed In**: 9.0.2, 9.1.1
