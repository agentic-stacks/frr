# Run FRR in Containers

## Overview

FRR publishes official container images at `quay.io/frrouting/frr`. Running FRR in containers is useful for lab environments, CI/CD testing, multi-router topologies, and isolated routing instances. FRR containers require elevated privileges (NET_ADMIN capability at minimum) because routing daemons must modify the kernel routing table.

## Official Images

```bash
# Pull the latest stable release
docker pull quay.io/frrouting/frr:latest

# Pull a specific version
docker pull quay.io/frrouting/frr:10.2.1

# List available tags
skopeo list-tags docker://quay.io/frrouting/frr
```

Image tags follow the pattern: `<major>.<minor>.<patch>` (e.g., `10.2.1`). The `latest` tag tracks the most recent stable release.

## Run FRR in Docker

### Minimal Run

```bash
docker run -d --name frr1 \
  --privileged \
  quay.io/frrouting/frr:latest
```

> **Warning:** `--privileged` grants full host capabilities. For production or shared systems, use explicit capabilities instead (see below).

### Production Run with Volume Mounts

```bash
# Create a local config directory
mkdir -p ./frr-config
cp /etc/frr/daemons ./frr-config/daemons
cp /etc/frr/frr.conf ./frr-config/frr.conf
cp /etc/frr/vtysh.conf ./frr-config/vtysh.conf

# Run with explicit capabilities and config mount
docker run -d --name frr1 \
  --cap-add=NET_ADMIN \
  --cap-add=NET_RAW \
  --cap-add=SYS_ADMIN \
  --cap-add=NET_BIND_SERVICE \
  --network host \
  -v $(pwd)/frr-config:/etc/frr:rw \
  quay.io/frrouting/frr:latest
```

### Required Capabilities

| Capability | Reason |
|---|---|
| `NET_ADMIN` | Modify kernel routing table, set interface parameters |
| `NET_RAW` | Send/receive raw packets (OSPF, IS-IS, BFD) |
| `SYS_ADMIN` | Required for VRF and MPLS operations |
| `NET_BIND_SERVICE` | Bind to privileged ports (BGP port 179) |

### Network Mode Options

| Mode | Use Case | Command |
|---|---|---|
| `--network host` | Route real traffic on the host's interfaces | `docker run --network host ...` |
| `--network bridge` | Isolated lab with Docker networks | `docker run --network my-lab-net ...` |
| `--network none` | Attach networks manually after creation | `docker run --network none ...` |

For production routing, always use `--network host` so FRR can see and manage the host's physical interfaces.

## Docker Compose Example

Single FRR instance with persistent configuration:

```yaml
# docker-compose.yml
services:
  frr:
    image: quay.io/frrouting/frr:10.2.1
    container_name: frr1
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - NET_RAW
      - SYS_ADMIN
      - NET_BIND_SERVICE
    network_mode: host
    volumes:
      - ./frr-config:/etc/frr:rw
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv6.conf.all.forwarding=1
```

Start and manage:

```bash
docker compose up -d
docker compose logs -f frr
docker compose restart frr
docker compose down
```

## Run FRR in Podman

Podman commands are nearly identical to Docker. Key differences: Podman runs rootless by default, so you must run as root or use `--privileged` for routing operations.

```bash
# Pull the image
sudo podman pull quay.io/frrouting/frr:latest

# Run with host networking
sudo podman run -d --name frr1 \
  --cap-add=NET_ADMIN \
  --cap-add=NET_RAW \
  --cap-add=SYS_ADMIN \
  --cap-add=NET_BIND_SERVICE \
  --network host \
  -v ./frr-config:/etc/frr:Z \
  quay.io/frrouting/frr:latest
```

> **Note:** The `:Z` volume suffix on Podman applies the correct SELinux label for the container to access the mounted directory.

## Access vtysh Inside the Container

```bash
# Docker
docker exec -it frr1 vtysh

# Podman
sudo podman exec -it frr1 vtysh
```

Once inside vtysh:

```
frr1# show daemons
frr1# show ip route
frr1# show running-config
```

To run a single command without entering the shell:

```bash
docker exec frr1 vtysh -c "show ip route"
docker exec frr1 vtysh -c "show bgp summary"
```

## Multi-Router Lab Setup

Build a topology with multiple FRR containers connected via Docker networks.

### Create Lab Networks

```bash
# Create point-to-point networks between routers
docker network create --subnet=10.0.12.0/24 link-r1-r2
docker network create --subnet=10.0.13.0/24 link-r1-r3
docker network create --subnet=10.0.23.0/24 link-r2-r3
```

### Create Router Configs

```bash
# Create per-router config directories
for r in r1 r2 r3; do
  mkdir -p ./lab/$r
done
```

Create `./lab/r1/daemons`:

```
zebra=yes
bgpd=yes
ospfd=yes
staticd=yes
```

Create `./lab/r1/frr.conf`:

```
frr defaults traditional
hostname r1
!
interface eth1
 ip address 10.0.12.1/24
!
interface eth2
 ip address 10.0.13.1/24
!
router ospf
 network 10.0.12.0/24 area 0
 network 10.0.13.0/24 area 0
!
```

Repeat for r2 and r3 with appropriate addresses.

### Launch Lab with Docker Compose

```yaml
# docker-compose.lab.yml
services:
  r1:
    image: quay.io/frrouting/frr:10.2.1
    container_name: r1
    hostname: r1
    privileged: true
    volumes:
      - ./lab/r1:/etc/frr:rw
    networks:
      link-r1-r2:
        ipv4_address: 10.0.12.1
      link-r1-r3:
        ipv4_address: 10.0.13.1

  r2:
    image: quay.io/frrouting/frr:10.2.1
    container_name: r2
    hostname: r2
    privileged: true
    volumes:
      - ./lab/r2:/etc/frr:rw
    networks:
      link-r1-r2:
        ipv4_address: 10.0.12.2
      link-r2-r3:
        ipv4_address: 10.0.23.2

  r3:
    image: quay.io/frrouting/frr:10.2.1
    container_name: r3
    hostname: r3
    privileged: true
    volumes:
      - ./lab/r3:/etc/frr:rw
    networks:
      link-r1-r3:
        ipv4_address: 10.0.13.3
      link-r2-r3:
        ipv4_address: 10.0.23.3

networks:
  link-r1-r2:
    driver: bridge
    ipam:
      config:
        - subnet: 10.0.12.0/24
  link-r1-r3:
    driver: bridge
    ipam:
      config:
        - subnet: 10.0.13.0/24
  link-r2-r3:
    driver: bridge
    ipam:
      config:
        - subnet: 10.0.23.0/24
```

```bash
docker compose -f docker-compose.lab.yml up -d

# Verify OSPF adjacencies
docker exec r1 vtysh -c "show ip ospf neighbor"

# Check routes learned via OSPF
docker exec r1 vtysh -c "show ip route ospf"

# Tear down the lab
docker compose -f docker-compose.lab.yml down
```

## Build FRR Docker Image from Source

FRR provides Dockerfiles for building custom images:

```bash
git clone https://github.com/FRRouting/frr.git
cd frr

# Build Alpine-based image
docker build -f docker/alpine/Dockerfile -t frr:custom .

# Build with multi-architecture support
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  -f docker/alpine/Dockerfile \
  -t frr:custom .
```

## Container-Specific Considerations

1. **Kernel parameters** -- Sysctl settings (IP forwarding, MPLS) apply to the host kernel, not the container. Set them on the host or use the `sysctls` directive in Docker Compose (only works for namespaced sysctls).
2. **MPLS in containers** -- MPLS kernel modules must be loaded on the host: `sudo modprobe mpls_router mpls_iptunnel`. The container cannot load kernel modules.
3. **VRF support** -- Requires `SYS_ADMIN` capability and kernel >= 4.15 (IPv4) / 5.0 (IPv6).
4. **Persistent config** -- Always mount `/etc/frr` as a volume. Without a mount, configuration is lost when the container is removed.
5. **Logging** -- FRR logs to syslog by default inside the container. Add `log file /var/log/frr/frr.log` to `frr.conf` or mount `/var/log/frr` to persist logs.

## Next Step

After running FRR in a container, see `skills/deploy/initial-setup` for daemon enablement and kernel parameter tuning on the host.
