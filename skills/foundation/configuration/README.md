# FRR Configuration

## vtysh -- The Integrated Shell

vtysh is a unified CLI that connects to all running FRR daemons simultaneously through Unix domain sockets. It provides a single Cisco IOS-like interface for configuring every protocol.

### How vtysh Works

- vtysh connects to each daemon's VTY Unix socket in `/var/run/frr/`
- Commands are routed to the appropriate daemon(s) automatically
- `show` commands may query multiple daemons and merge output
- Configuration saves can write a single integrated file or per-daemon files

### Access Requirements

- Run as root, or be a member of the `frrvty` group
- Daemons must be running for vtysh to reach them
- **WARNING:** If a daemon is stopped, vtysh cannot retrieve its configuration. Saving config while a daemon is down permanently loses that daemon's configuration from the saved file.

### Launch vtysh

```
sudo vtysh
```

Or run a single command without entering interactive mode:

```
sudo vtysh -c "show ip route"
```

Chain multiple commands:

```
sudo vtysh -c "show ip bgp summary" -c "show ip ospf neighbor"
```

### Apply a Config File

Load a configuration file into running daemons as if each line were typed:

```
sudo vtysh
copy /path/to/config-snippet.conf running-config
```

## Configuration Modes

vtysh uses a hierarchical mode system. Each mode has a distinct prompt and available command set.

```
View mode:         router>
Enable mode:       router#
Config mode:       router(config)#
Interface mode:    router(config-if)#
Router mode:       router(config-router)#
Route-map mode:    router(config-route-map)#
```

### Enter Each Mode

```
! Start in view mode (read-only)
router> enable

! Enter enable mode (read-write, show commands)
router# configure terminal

! Enter global configuration mode
router(config)# interface eth0

! Enter interface configuration mode
router(config-if)# exit

! Back to global config
router(config)# router bgp 65001

! Enter BGP router configuration mode
router(config-router)# end

! Return to enable mode from any depth
router#
```

### Mode Quick Reference

| Mode | Enter With | Exit With | Purpose |
|---|---|---|---|
| View | (default on login) | `quit` | Read-only show commands |
| Enable | `enable` | `disable` or `quit` | Privileged show commands, operational commands |
| Config | `configure terminal` | `end` or `exit` | Global configuration |
| Interface | `interface <name>` | `exit` | Per-interface settings |
| Router | `router <protocol> [args]` | `exit` | Protocol-specific configuration |
| Route-map | `route-map <name> <action> <seq>` | `exit` | Route policy configuration |

## Integrated vs Per-Daemon Configuration

FRR supports two configuration file strategies.

### Integrated Configuration (Recommended)

A single file `/etc/frr/frr.conf` holds configuration for all daemons.

- Daemons check for `frr.conf` at startup; if present, per-daemon files are ignored
- vtysh processes the file and distributes sections to the appropriate daemons via `vtysh -b`
- `write memory` saves all daemons' config back to `frr.conf`

Enable integrated mode in `/etc/frr/vtysh.conf`:

```
service integrated-vtysh-config
```

### Per-Daemon Configuration

Each daemon reads its own file: `zebra.conf`, `bgpd.conf`, `ospfd.conf`, etc.

- Used when `no service integrated-vtysh-config` is set in `vtysh.conf`
- Each daemon saves only its own config with `write memory`
- Files are located in `/etc/frr/`

### Decision Guide

| Scenario | Use |
|---|---|
| Standard deployments | Integrated (`frr.conf`) |
| Need per-daemon config management or version control | Per-daemon files |
| Automation tools manage individual protocol configs | Per-daemon files |
| Simple single-router setup | Integrated (`frr.conf`) |

### Detection Logic

If `/etc/frr/frr.conf` exists, FRR uses integrated mode regardless of `vtysh.conf` settings (auto-detect behavior). To force per-daemon mode, remove `frr.conf` and set `no service integrated-vtysh-config`.

## The daemons File

`/etc/frr/daemons` controls which daemons start when the FRR service launches. All daemons are disabled by default.

### Enable a Daemon

Edit `/etc/frr/daemons` and change the daemon entry from `no` to `yes`:

```
# /etc/frr/daemons
zebra=yes
bgpd=yes
ospfd=yes
staticd=yes

# These remain disabled
ripd=no
ripngd=no
isisd=no
pimd=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=no
fabricd=no
```

Then reload the FRR service to start the newly enabled daemons:

```
sudo systemctl reload frr
```

### Daemon-Specific Options

Each daemon has a corresponding `_options` variable for startup flags:

```
# /etc/frr/daemons
zebra_options=" -s 90000000 --daemon -A 127.0.0.1"
bgpd_options="   --daemon -A 127.0.0.1"
ospfd_options="  --daemon -A 127.0.0.1"
staticd_options="--daemon -A 127.0.0.1"
```

Common options:

| Flag | Purpose |
|---|---|
| `--daemon` | Fork into background |
| `-A 127.0.0.1` | Bind VTY to localhost only (security) |
| `-s <size>` | Netlink buffer size (zebra only) |
| `-P <port>` | VTY TCP port (0 disables TCP VTY) |

### Global Options

Apply flags to all daemons at once:

```
# /etc/frr/daemons
frr_global_options="-w"
```

The `-w` flag enables VRF-backed network namespaces for all daemons.

### Other Settings

```
# /etc/frr/daemons
vtysh_enable=yes          # Run "vtysh -b" at startup to load config
MAX_FDS=1024              # File descriptor limit per daemon (increase for large BGP tables)
FRR_NO_ROOT="yes"        # Run without root (use with caution)
watchfrr_options=""       # Additional watchfrr flags
```

## vtysh.conf

`/etc/frr/vtysh.conf` contains settings specific to the vtysh CLI itself. This file is NOT overwritten by `write memory` -- you must edit it manually.

### Minimal vtysh.conf

```
! /etc/frr/vtysh.conf
service integrated-vtysh-config
hostname router1
```

### Common Settings

```
! /etc/frr/vtysh.conf
service integrated-vtysh-config    ! Use single frr.conf
hostname router1                    ! Prompt hostname
domainname example.com              ! Domain name for prompt
banner motd line FRR Router 1       ! Login banner
username admin nopassword           ! Skip PAM auth for this user
terminal paginate                   ! Enable paged output by default
```

### Pagination Control

Control the pager program via environment variable:

```
export VTYSH_PAGER=less
sudo -E vtysh
```

Set `VTYSH_PAGER` to `cat` to disable pagination entirely:

```
export VTYSH_PAGER=cat
```

## Applying Configuration Changes

Three methods exist for applying configuration changes, each with different trade-offs.

### write memory

Saves the running configuration to disk. Does NOT change the running config -- it persists what is already active.

```
router# write memory
Building Configuration...
Configuration saved to /etc/frr/frr.conf
[OK]
```

Equivalent commands (all do the same thing):

```
write memory
write file
copy running-config startup-config
```

**WARNING:** If any daemon is stopped when you run `write memory`, that daemon's configuration is lost from the saved file. Always verify all daemons are running first:

```
router# show watchfrr
```

### frr-reload.py

Apply configuration changes without restarting daemons. This is the safest way to modify a running router.

**How it works:**

1. Reads the target config file (e.g., `/etc/frr/frr.conf`)
2. Retrieves the current running config via `show running-config`
3. Computes the diff
4. Applies only the delta commands via vtysh

**Test changes first (dry run):**

```
sudo /usr/lib/frr/frr-reload.py --test /etc/frr/frr.conf
```

This shows what commands would be applied without executing them.

**Apply changes:**

```
sudo /usr/lib/frr/frr-reload.py --reload /etc/frr/frr.conf
```

**Per-daemon reload (when using per-daemon config files):**

```
sudo /usr/lib/frr/frr-reload.py --daemon bgpd --reload /etc/frr/bgpd.conf
```

**Additional options:**

| Flag | Purpose |
|---|---|
| `--test` | Show diff only, do not apply |
| `--reload` | Apply configuration delta |
| `--daemon <name>` | Target a specific daemon |
| `--debug` | Verbose output |
| `--overwrite` | Update config file after applying |
| `--bindir <path>` | Path to vtysh binary |
| `--confdir <path>` | Config directory |
| `--vty_socket <path>` | VTY socket path |
| `--stdout` | Output to stdout |
| `--logfile <path>` | Log file (default: `/var/log/frr/frr-reload.log`) |

**Via systemd:**

```
sudo systemctl reload frr
```

This invokes `frr-reload.py` under the hood.

### Daemon Restart

Full restart kills all daemons and reinitializes them.

**WARNING:** Restarting FRR drops ALL protocol sessions (BGP peers, OSPF adjacencies, etc.) and causes network reconvergence. Only restart when reload is insufficient.

```
sudo systemctl restart frr
```

### Decision Guide: Reload vs Restart

| Situation | Use |
|---|---|
| Added/changed route-maps, prefix-lists, ACLs | `frr-reload.py --reload` |
| Changed BGP neighbor config | `frr-reload.py --reload` |
| Added a new protocol daemon | Edit `daemons` file, then `systemctl reload frr` |
| Changed daemon startup flags | `systemctl restart frr` |
| Daemon is crashed or hung | `systemctl restart frr` |
| Upgraded FRR binary | `systemctl restart frr` |
| Changed kernel parameters | `systemctl restart frr` |

## Config Include Files

FRR supports including external configuration snippets in `frr.conf`:

```
! /etc/frr/frr.conf
frr defaults datacenter
hostname router1
!
! Include prefix lists from a separate file
include /etc/frr/prefix-lists.conf
!
router bgp 65001
  ! Include neighbor definitions
  include /etc/frr/bgp-neighbors.conf
```

This enables modular configuration management and templating.

## Essential vtysh Commands

### View Running State

```
! Show full running configuration
show running-config

! Show running config for a specific daemon
show running-config daemon bgpd

! Show saved (startup) configuration
show startup-config

! Display running config to terminal (same as show running-config)
write terminal
```

### Save and Load Configuration

```
! Save running config to disk
write memory

! Write integrated config via watchfrr
write integrated

! Load a config file into running state
copy /path/to/file.conf running-config
```

### System Information

```
! Show FRR version and build info
show version

! Show memory usage per daemon
show memory

! Show event loop CPU statistics
show event cpu

! Show logging configuration
show logging

! Show command history
show history
```

### Search and Filter

```
! Search all available commands by regex
find bgp neighbor

! Filter command output with regex
show ip route | include 10.0.0

! Show all commands containing a pattern
find redistribute
```

### Daemon and Process Status

```
! Show watchfrr daemon status
show watchfrr

! Show all connected daemons (from zebra perspective)
show zebra client summary
```

### Common Operational Commands

```
! Show IP routing table
show ip route
show ip route 10.0.0.0/24
show ip route summary
show ipv6 route

! Show interface status
show interface
show interface eth0
show interface brief

! Network diagnostics from the router
ping 10.0.0.1
traceroute 10.0.0.1
```
