# Known Issues Index

Documented known bugs, regressions, and workarounds for FRR releases, organized by major version.

---

## Entry Format

Each issue follows this standard template:

```
### ISSUE-TITLE

- **Symptom**: What the operator observes (crash, wrong routes, memory leak, etc.)
- **Cause**: Root cause or triggering condition
- **Workaround**: Configuration change, avoidance, or operational mitigation
- **Affected Versions**: First affected version through last affected version
- **Fixed In**: Version containing the fix, or "unresolved"
- **CVE**: CVE identifier if applicable
- **References**: GitHub issue/PR links
```

## Version Files

| File | Coverage |
|---|---|
| [frr-9.x.md](frr-9.x.md) | FRR 9.0 through 9.1.3 |
| [frr-10.x.md](frr-10.x.md) | FRR 10.0 through 10.6.0 |

## Severity Legend

Issues are tagged with severity indicators:

| Tag | Meaning |
|---|---|
| **CRASH** | Daemon crash or segfault |
| **DATA-LOSS** | Route loss, black-holing, or incorrect forwarding |
| **SECURITY** | CVE or exploitable vulnerability |
| **MEMORY** | Memory leak leading to eventual OOM |
| **OPERATIONAL** | Incorrect output, CLI bug, or non-critical regression |

## General Guidance

1. Always run the latest maintenance release for your major version (e.g., 9.1.3 not 9.1.0)
2. Subscribe to the [FRR security advisories](https://frrouting.org/security/) for CVE notifications
3. Check release notes before upgrading: `https://github.com/FRRouting/frr/releases`
4. Test upgrades in a lab environment matching your production topology
