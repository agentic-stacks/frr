# Known Issues — FRR 10.x

Covers FRR 10.0.0 through 10.6.0. Always run the latest maintenance release for your minor version.

---

## Security Vulnerabilities

### CVE-2024-27913 — Memory Leak via Malformed BGP UPDATE

- **Symptom**: bgpd memory grows when receiving repeated malformed BGP UPDATEs
- **Cause**: Error path did not free allocated memory for malformed attributes
- **Workaround**: Apply strict prefix filtering on all eBGP sessions; upgrade
- **Affected Versions**: 10.0.0
- **Fixed In**: 10.0.1
- **CVE**: CVE-2024-27913

### CVE-2024-34088 — bgpd Crash on Crafted BGP Message

- **Symptom**: bgpd crash processing specially crafted BGP messages
- **Cause**: Missing validation in BGP message handler
- **Workaround**: Upgrade immediately
- **Affected Versions**: 10.0.0
- **Fixed In**: 10.0.1
- **CVE**: CVE-2024-34088

### CVE-2024-31948 through CVE-2024-31951 — Multiple BGP Crashes

- **Symptom**: Various bgpd crashes via crafted BGP UPDATE messages
- **Cause**: Heap use-after-free, buffer overflow, tunnel encapsulation parsing errors
- **Workaround**: Upgrade immediately
- **Affected Versions**: 10.0.0
- **Fixed In**: 10.0.1
- **CVE**: CVE-2024-31948, CVE-2024-31949, CVE-2024-31950, CVE-2024-31951
- **References**: https://frrouting.org/security/

### CVE-2024-55553 — BGP Vulnerability (Fixed in 10.3)

- **Symptom**: bgpd vulnerability (details in security advisory)
- **Cause**: BGP protocol handling flaw
- **Workaround**: Upgrade to 10.3 or later
- **Affected Versions**: 10.0.x through 10.2.x
- **Fixed In**: 10.3
- **CVE**: CVE-2024-55553

---

## Breaking Changes by Version

### 10.0 — libyang Minimum Version Bumped to 2.1.128

- **Symptom**: FRR 10.0 fails to compile or start with older libyang
- **Cause**: API changes require libyang >= 2.1.128
- **Workaround**: Upgrade libyang before upgrading FRR
- **Affected Versions**: 10.0.0+

### 10.2 — PIM Config Node Restructured

- **Symptom**: Existing PIM global commands deprecated; `router pim` node introduced
- **Cause**: PIM configuration moved to per-instance model matching other protocols
- **Workaround**: Migrate config to new `router pim` node; old global commands still work but are deprecated
- **Affected Versions**: 10.2+
- **References**: PR #16269

### 10.5 — BGP Rejects AS_SET by Default

- **Symptom**: BGP sessions that send AS_SET may have routes rejected
- **Cause**: RFC 9774 compliance; `set as-path` with AS_SET now rejected by default
- **Workaround**: Explicitly allow AS_SET if needed: `bgp reject-as-sets disable`
- **Affected Versions**: 10.5.0+

### 10.6 — libyang3 Default for Binary Packages

- **Symptom**: Binary packages now built against libyang3; libyang2 modules incompatible
- **Cause**: Upstream migration to libyang3
- **Workaround**: Recompile from source with libyang2 if needed; or migrate to libyang3
- **Affected Versions**: 10.6.0+

---

## BGP Issues

### DATA-LOSS — Route Leaking from Default L3VRF Broken

- **Symptom**: Routes leaked from the default VRF to other VRFs are missing or incorrect
- **Cause**: Bug in VRF route export logic for the default instance
- **Workaround**: Use static routes or redistribution as alternative; upgrade
- **Affected Versions**: 10.0.0
- **Fixed In**: 10.0.1, 10.1.0

### CRASH — bgpd Crash When Deleting SRv6 Locator

- **Symptom**: bgpd segfault when removing SRv6 locator configuration
- **Cause**: Null pointer dereference in SRv6 cleanup path
- **Workaround**: Restart bgpd after SRv6 locator removal; upgrade
- **Affected Versions**: 10.0.0
- **Fixed In**: 10.0.1

### MEMORY — bgpd SRv6 Memory Leaks

- **Symptom**: Gradual memory growth in bgpd when using SRv6
- **Cause**: SRv6 functions and locator chunks not freed on cleanup
- **Workaround**: Periodic bgpd restart; upgrade
- **Affected Versions**: 10.0.0
- **Fixed In**: 10.0.1

### OPERATIONAL — BGP Dynamic Capability Default Change (10.1)

- **Symptom**: Unexpected BGP session behavior after upgrading to 10.1 with datacenter profile
- **Cause**: Dynamic capability enabled by default for datacenter profile
- **Workaround**: Explicitly disable if not desired: `no bgp default dynamic-capability`
- **Affected Versions**: 10.1.0+

### OPERATIONAL — RPKI Cache Command Syntax Broken (pre-10.1)

- **Symptom**: TCP RPKI cache with `source` option treated as SSH session
- **Cause**: Ambiguous CLI parsing for `rpki cache` command
- **Workaround**: Upgrade to 10.1+ which splits into `rpki cache tcp` and `rpki cache ssh`
- **Affected Versions**: 10.0.x
- **Fixed In**: 10.1.0

### CRASH — bgpd Crash When Fetching Statistics for BGP Instance

- **Symptom**: bgpd crash when running statistics-related show commands
- **Cause**: Null pointer in statistics collection
- **Workaround**: Avoid per-instance stats commands; upgrade
- **Affected Versions**: 10.4.x
- **Fixed In**: 10.5.0

### OPERATIONAL — BGP IPv6 Link-Local Capability Breaks prefer-global

- **Symptom**: Routes with `set ipv6 next-hop prefer-global` fail when peer sends only link-local
- **Cause**: Link-Local capability was enabled by default in datacenter profile (10.4.0); receiver expecting global address gets only link-local
- **Workaround**: Disable link-local capability: `no neighbor PEER capability link-local`
- **Affected Versions**: 10.4.0, 10.4.1
- **Fixed In**: 10.4.2, 10.5.0 (disabled by default)

### DATA-LOSS — bgpd iBGP Malformed Packet Resets Session (pre-10.5)

- **Symptom**: iBGP session reset on malformed packet instead of treat-as-withdraw
- **Cause**: RFC 7606 error handling not applied to iBGP peers
- **Workaround**: Upgrade to 10.5+ for proper RFC 7606 handling on iBGP
- **Affected Versions**: 10.0 through 10.4.x
- **Fixed In**: 10.5.0

---

## OSPF Issues

### CRASH — ospfd Crash in TE Parsing

- **Symptom**: ospfd crash when TE and Router Information LSAs interact
- **Cause**: Null pointer dereference in TE opaque LSA parser
- **Workaround**: Disable TE, or upgrade
- **Affected Versions**: 10.0.0
- **Fixed In**: 10.0.1

### CRASH — ospf6d Summary LSA Removal Failure

- **Symptom**: ospf6d crash or incorrect routing after ABR summary LSA removal
- **Cause**: Bug in summary LSA cleanup logic
- **Workaround**: Clear OSPF process; upgrade
- **Affected Versions**: 10.0.x through 10.4.x
- **Fixed In**: 10.5.0

---

## IS-IS Issues

### CRASH — isisd Crash on SRv6 SID Structure JSON Display

- **Symptom**: isisd crash when running `show isis segment-routing srv6 ... json`
- **Cause**: Null pointer in JSON serialization of SRv6 SID structure
- **Workaround**: Avoid JSON output for SRv6 SID display; upgrade
- **Affected Versions**: 10.0.x through 10.1.x
- **Fixed In**: 10.2.0

### OPERATIONAL — IS-IS Neighbor Adjacency Fails After Type Switching

- **Symptom**: IS-IS neighbor adjacency cannot be established after switching IS type multiple times
- **Cause**: State machine not properly reset on repeated type changes
- **Workaround**: Clear IS-IS process after type changes; upgrade
- **Affected Versions**: 10.0.x through 10.1.x
- **Fixed In**: 10.2.0

---

## PIM Issues

### CRASH — pimd DR Election Race on Startup

- **Symptom**: pimd crash during DR election immediately after startup
- **Cause**: Race condition in DR election when multiple interfaces come up simultaneously
- **Workaround**: Stagger interface bring-up; upgrade
- **Affected Versions**: 10.0.x through 10.3.x
- **Fixed In**: 10.4.0

---

## NHRP Issues

### CRASH — nhrpd Crash Accessing Invalid Memory

- **Symptom**: nhrpd segfault during normal operation
- **Cause**: Invalid memory access in NHRP cache handling
- **Workaround**: Restart nhrpd; upgrade
- **Affected Versions**: 10.0.x through 10.4.x
- **Fixed In**: 10.5.0

---

## Zebra Issues

### CRASH — zebra Crash in PW (Pseudowire) Code

- **Symptom**: zebra crash when processing pseudowire operations
- **Cause**: Null pointer dereference in PW handling
- **Workaround**: Avoid PW configuration changes under load; upgrade
- **Affected Versions**: 10.2.x
- **Fixed In**: 10.3.0

### OPERATIONAL — zebra Starvation in Dataplane Thread Loop

- **Symptom**: Route installation delays; zebra dataplane thread unresponsive
- **Cause**: Tight loop in `dplane_thread_loop` preventing other work
- **Workaround**: Upgrade
- **Affected Versions**: 10.0.x through 10.1.x
- **Fixed In**: 10.2.0

---

## Tools Issues

### OPERATIONAL — frr-reload.py Missing mgmtd in logrotate/rsyslog

- **Symptom**: mgmtd logs not rotated; missing from rsyslog configuration
- **Cause**: mgmtd not included in logrotate/rsyslog tool templates
- **Workaround**: Manually add mgmtd to logrotate and rsyslog configs
- **Affected Versions**: 10.0.x through 10.2.x
- **Fixed In**: 10.3.0

### OPERATIONAL — IPv6 Hash Only Uses 4 Bytes

- **Symptom**: Poor ECMP distribution for IPv6 traffic
- **Cause**: Hash function only processed 4 bytes of IPv6 addresses instead of 16
- **Workaround**: Upgrade to 10.4+
- **Affected Versions**: All versions prior to 10.4.0
- **Fixed In**: 10.4.0
- **References**: PR #17901
