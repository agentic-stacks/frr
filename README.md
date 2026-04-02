# FRR — Agentic Stack

An [agentic stack](https://github.com/agentic-stacks/agentic-stacks) that teaches AI agents to deploy, configure, operate, troubleshoot, and develop [FRR (Free Range Routing)](https://frrouting.org).

## What This Stack Covers

- **All routing protocols:** BGP, OSPF, IS-IS, RIP, PIM, LDP, BFD, Babel, PBR, OpenFabric, VRRP, EIGRP, NHRP
- **Multiple deployment models:** Package-based (apt/yum), container (Docker/Podman), and source builds
- **Full operational lifecycle:** Installation, configuration, health checks, upgrades, backup/restore, monitoring, performance tuning
- **Troubleshooting:** Symptom-based decision trees and FRR debug tooling
- **Development:** FRR codebase navigation, building, testing, and contributing

## Target Versions

- FRR 10.x (current stable)
- FRR 9.x (previous major)

## Usage

### New Project

```bash
agentic-stacks init agentic-stacks/frr my-frr-project
cd my-frr-project
```

### Add to Existing Project

```bash
cd my-project
agentic-stacks pull agentic-stacks/frr
```

### Composability

This stack focuses on standalone FRR on Linux. It composes well with infrastructure stacks for platform-specific deployments.

## Authoring

This stack follows the [Authoring a Stack](https://github.com/agentic-stacks/agentic-stacks/blob/main/docs/guides/authoring-a-stack.md) guide.
