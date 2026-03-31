# Testing FRR

## Test Types

| Type | Location | Framework | Purpose |
|---|---|---|---|
| Unit tests | `tests/` | C + check/cmocka | Test individual functions and modules |
| Topotests | `tests/topotests/` | Python + pytest + micronet | End-to-end topology-based integration tests |
| Fuzzing | Various | AFL / libFuzzer | Find crashes from malformed input |

## Unit Tests

### Run All Unit Tests

```bash
make check
# or
make test
```

### Run a Specific Unit Test

```bash
make -C tests check TESTS=test_name
```

Unit tests live in the `tests/` directory, one C file per test. They link against libfrr and the daemon libraries.

## Topotests

Topotests are FRR's primary integration test suite. They create virtual network topologies using micronet (Linux network namespaces), start real FRR daemons, and verify protocol behavior.

### System Requirements

- Linux with network namespace support (Ubuntu 22.04+, Debian 12+)
- Python 3
- Root privileges (or sudo)
- FRR installed (binaries in `/usr/lib/frr`)

### Install Topotest Dependencies

```bash
sudo apt install -y gdb iproute2 net-tools python3-pip iputils-ping \
  iptables tshark valgrind

pip3 install --user \
  "pytest>=8.3.2" pytest-asyncio pytest-xdist \
  "scapy>=2.4.5" "libyang<4" \
  pyyaml xmltodict exabgp
```

### Configure sudo

Topotests require preserved environment variables:

```bash
# Add to /etc/sudoers (via visudo)
Defaults env_keep="TMUX"
Defaults env_keep+="TMUX_PANE"
```

Or always use `sudo -E` to preserve the environment.

### Run All Topotests

```bash
cd tests/topotests
sudo -E pytest -s -v -nauto --dist=loadfile
```

| Flag | Purpose |
|---|---|
| `-s` | Show stdout/stderr (required for debug output) |
| `-v` | Verbose -- show individual test names |
| `-nauto` | Distribute across all CPU cores (pytest-xdist) |
| `--dist=loadfile` | Keep tests from same file on same worker |

### Run a Single Topotest

```bash
cd tests/topotests
sudo -E pytest -s -v bgp_l3vpn_to_bgp_vrf/test_bgp_l3vpn_to_bgp_vrf.py
```

### Run a Single Test Function

```bash
sudo -E pytest -s -v bgp_l3vpn_to_bgp_vrf/test_bgp_l3vpn_to_bgp_vrf.py::test_convergence
```

### Run Tests Matching a Pattern

```bash
sudo -E pytest -s -v -k "bgp and not evpn"
```

## Writing New Topotests

### Directory Structure

```
tests/topotests/my_new_test/
  ├── __init__.py                 # Required (empty file)
  ├── test_my_new_test.py         # Topology definition + test functions
  ├── test_my_new_test.dot        # Graphviz topology diagram (optional)
  ├── r1/
  │   └── frr.conf               # Router 1 configuration
  ├── r2/
  │   └── frr.conf               # Router 2 configuration
  └── ...
```

### Minimal Topotest Template

```python
#!/usr/bin/env python3
"""
Test description here.
"""

import os
import sys
import pytest

CWD = os.path.dirname(os.path.realpath(__file__))
sys.path.append(os.path.join(CWD, "../"))

from lib.topogen import Topogen, TopoRouter, get_topogen
from lib.topolog import logger

pytestmark = [pytest.mark.bgpd]

# Simple topology: r1 --s1-- r2 --s2-- r3
topodef = {
    "s1": ("r1", "r2"),
    "s2": ("r2", "r3"),
}


@pytest.fixture(scope="module")
def tgen(request):
    """Setup/teardown the topology."""
    tgen = Topogen(topodef, request.module.__name__)
    tgen.start_topology()

    # Load router configs from per-router directories
    router_list = tgen.routers()
    for rname, router in router_list.items():
        router.load_frr_config(
            os.path.join(CWD, "{}/frr.conf".format(rname)),
            [(TopoRouter.RD_ZEBRA, "-s 90000000"),
             (TopoRouter.RD_MGMTD, None),
             (TopoRouter.RD_BGP, None)],
        )

    tgen.start_router()
    yield tgen
    tgen.stop_topology()


@pytest.fixture(scope="module")
def router_list(tgen):
    """Convenience fixture."""
    if tgen.routers_have_failure():
        pytest.skip(tgen.errors)
    return tgen.routers()


def test_convergence(tgen, router_list):
    """Verify BGP sessions establish."""
    if tgen.routers_have_failure():
        pytest.skip(tgen.errors)

    r1 = router_list["r1"]
    output = r1.vtysh_cmd("show bgp summary json")
    # Parse JSON, verify peers are established
    # ...
    assert "Established" in output


def test_routes_installed(tgen, router_list):
    """Verify expected routes are present."""
    if tgen.routers_have_failure():
        pytest.skip(tgen.errors)

    r3 = router_list["r3"]
    output = r3.vtysh_cmd("show ip route json")
    # Verify routes
    # ...
```

### Best Practices for Topotests

| Practice | Detail |
|---|---|
| Use `@retry` decorators | Instead of `time.sleep()` -- polls until condition is met or timeout |
| Generous BGP timeouts | At least 130 seconds for BGP convergence |
| Avoid unstable data | Do not assert on link-local addresses, ifindex values, or timestamps |
| Use JSON output | `show ... json` for reliable parsing vs. text scraping |
| Use RFC 5737 addresses | 192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24 for IPv4 |
| Use RFC 3849 addresses | 2001:db8::/32 for IPv6 |
| Directory naming | Underscores only, no hyphens (e.g., `bgp_new_feature`) |
| Format with black | Run `black` on all Python code before submitting |
| Reuse library functions | Check `tests/topotests/lib/` for existing helpers |

### Topology Definition Methods

**Simple dictionary:**

```python
topodef = {
    "s1": ("r1", "r2"),
    "s2": ("r2", "r3"),
}
```

**Build function (for complex topologies):**

```python
def build_topo(tgen):
    tgen.add_router("r1")
    tgen.add_router("r2")
    switch = tgen.add_switch("s1")
    switch.add_link(tgen.gears["r1"])
    switch.add_link(tgen.gears["r2"])
```

**JSON topology (alternative framework):**

```python
from lib import fixtures
tgen = pytest.fixture(fixtures.tgen_json, scope="module")
```

With a corresponding JSON file defining routers, links, and protocol configuration.

## Topotest Debugging

### Interactive CLI on Error

```bash
sudo -E pytest --cli-on-error bgp_test/test_bgp.py
```

When a test fails, drops into an interactive shell where you can run:

- `vtysh [hosts]` -- open vtysh terminals on routers
- `sh [hosts] <command>` -- run shell commands on routers
- `term [hosts]` -- open terminal windows on routers
- `help` -- list available commands

### Python Debugger

```bash
sudo -E pytest -s --pdb bgp_test/test_bgp.py
```

Drops into pdb on assertion failure. Add manual breakpoints:

```python
import pdb; pdb.set_trace()
```

### Live Daemon Log Viewing

```bash
# Watch ripd logs in real time
sudo -E pytest --logd=ripd rip_test/

# Watch multiple daemons on specific routers
sudo -E pytest --logd=r1:bgpd,r2:ospfd bgp_ospf_test/
```

### Packet Capture

```bash
# Capture on switch s1 and r2's eth0
sudo -E pytest --pcap=sw1,r2:r2-eth0 isis_topo1/
```

Saves `.pcap` files for analysis with Wireshark/tshark.

### GDB Debugging

```bash
sudo -E pytest \
  --gdb-routers=r1 \
  --gdb-daemons=bgpd,zebra \
  --gdb-breakpoints=nb_config_diff \
  all-protocol-startup/
```

For Emacs users: add `--gdb-use-emacs`.

### Valgrind Memory Leak Detection

```bash
sudo -E pytest --valgrind-memleaks all-protocol-startup/
sudo -E pytest --valgrind-leak-kinds=definite,possible bgp_test/
```

### Performance Profiling with perf

```bash
sudo -E pytest --perf=mgmtd,r1 config_timing/
```

Results saved as `perf.data` files in router-specific directories.

### Record/Replay with rr

```bash
sudo -E pytest --rr-routers=r1 --rr-daemons=mgmtd config_timing/
```

### Topology-Only Mode

Start the topology without running tests -- useful for interactive exploration:

```bash
sudo -E pytest --topology-only bgp_test/test_bgp.py
```

## Analyze Test Results

Topotest results are saved to `/tmp/topotests/`. Use the `analyze.py` tool:

```bash
cd tests/topotests

# Show failed and errored tests
./analyze.py -Ar run-save

# Enumerate all results
./analyze.py -Ar run-save -E

# Examine a specific test (by index N from enumeration)
./analyze.py -Ar run-save -T N --errmsg
./analyze.py -Ar run-save -T N --errtext

# Filter by status (e=error, f=failed, p=passed, s=skipped)
./analyze.py -Ar run-save -S f

# Search for patterns across results
./analyze.py -Ar run-save --search "REGEXP"
```

## Reproducing Intermittent Failures

```bash
# Create symlinks to run the same test multiple times in parallel
cd tests/topotests/flaky_test/
ln -s test_flaky.py test_a.py
ln -s test_flaky.py test_b.py
ln -s test_flaky.py test_c.py
sudo -E pytest -s -v -nauto --dist=loadfile .

# Increase parallelism beyond core count to alter timing
sudo -E pytest -n 16 --dist=loadfile .

# Saturate CPU to expose race conditions
stress -c 4 &
sudo -E pytest -s -v flaky_test/
```

## Docker-Based Testing

Run topotests inside a Docker container:

```bash
# Quick start -- pulls image, compiles, runs all tests
make topotests

# Custom options
TOPOTEST_CLEAN=1 ./tests/topotests/docker/frr-topotests.sh

# Run specific test
./tests/topotests/docker/frr-topotests.sh -vv -s \
  all-protocol-startup/test_all_protocol_startup.py
```

New test files must be `git add`-ed before container runs (the startup script runs `git-clean`).

## Code Coverage

### Collect Coverage from Topotests

```bash
# 1. Build with --enable-gcov
./configure --enable-gcov <other-flags>
make -j$(nproc) && sudo make install

# 2. Run topotests with coverage collection
cd tests/topotests
sudo -E pytest --cov-topotest --cov-frr-build-dir=/path/to/frr/build

# 3. Coverage data accumulates in /tmp/topotests/gcda/
# 4. lcov generates coverage.info automatically
genhtml /tmp/topotests/gcda/coverage.info --output-directory coverage-html
```

### Collect Coverage from Unit Tests

```bash
make check
lcov --capture --directory . --output-file coverage.info
genhtml coverage.info --output-directory coverage-html
```

## CI Pipeline

Every pull request triggers automated CI. The pipeline includes:

| Check | What It Verifies |
|---|---|
| Build | Compiles on multiple platforms (Ubuntu, Debian, CentOS, FreeBSD) |
| Unit tests | `make check` passes |
| Topotests | Full topotest suite passes |
| Coding style | `checkpatch.sh` and `clang-format` compliance |
| Documentation | `make html` and `make latexpdf` build without warnings |
| Package builds | Distribution packages build successfully |

### CI Results

- Results are emailed to the PR author within ~2 hours
- Check the GitHub PR status indicators if email is not received
- **Expect the community to ignore submissions until CI passes**
- Fix failures and update the PR -- responsibility rests with the author

## Fuzzing

### Build for Fuzzing

```bash
./configure --enable-fuzzing <other-flags>
make -j$(nproc)
```

Fuzzing harnesses live alongside their target code. They exercise protocol parsers with malformed input to find crashes, memory errors, and undefined behavior.

### Combine with Sanitizers

For maximum effectiveness, combine fuzzing with AddressSanitizer:

```bash
./configure --enable-fuzzing --enable-address-sanitizer
```

## Memory Leak Detection

### Via Topotests

Set the environment variable to collect leak reports:

```bash
export TOPOTESTS_CHECK_MEMLEAK="/home/user/memleak_"
sudo -E pytest --memleaks --cov-topotest all-protocol-startup/
```

Leak reports are written to files prefixed with the specified path.

### Via Valgrind

```bash
sudo -E pytest --valgrind-memleaks all-protocol-startup/
```

### Via AddressSanitizer

Build with `--enable-address-sanitizer`. ASan detects leaks at process exit and writes findings to `/tmp/AddressSanitizer.txt` (Markdown format) during topotest runs.
