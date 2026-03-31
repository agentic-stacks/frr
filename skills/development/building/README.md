# Building FRR for Development

## Prerequisites

### System Dependencies (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install -y \
  git autoconf automake libtool make libreadline-dev texinfo \
  pkg-config libpam0g-dev libjson-c-dev bison flex libc-ares-dev \
  python3-dev python3-sphinx install-info build-essential libsnmp-dev \
  perl libcap-dev libelf-dev libunwind-dev \
  protobuf-c-compiler libprotobuf-c-dev
```

### Optional Dependencies

| Package | Enables |
|---|---|
| `libgrpc++-dev protobuf-compiler-grpc` | gRPC northbound interface |
| `libsqlite3-dev` | Configuration rollbacks |
| `libzmq5 libzmq3-dev` | ZeroMQ message transport |
| `libyang2-dev` | YANG model support (required -- see below) |
| `gcov lcov` | Code coverage instrumentation |
| `valgrind` | Memory leak detection |

### libyang Dependency

FRR requires libyang >= 2.0. If your distribution does not package a suitable version, build from source:

```bash
git clone https://github.com/CESNET/libyang.git
cd libyang && mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
sudo make install
sudo ldconfig
```

## Developer Build Workflow

### Step 1: Bootstrap

Generate the configure script from `configure.ac`:

```bash
./bootstrap.sh
```

This runs `autoreconf -i` under the hood.

### Step 2: Configure

A minimal developer configure invocation:

```bash
./configure \
  --prefix=/usr \
  --includedir=\${prefix}/include \
  --bindir=\${prefix}/bin \
  --sbindir=\${prefix}/lib/frr \
  --libdir=\${prefix}/lib/frr \
  --libexecdir=\${prefix}/lib/frr \
  --sysconfdir=/etc/frr \
  --localstatedir=/var/run/frr \
  --with-moduledir=\${prefix}/lib/frr/modules \
  --enable-user=frr \
  --enable-group=frr \
  --enable-vty-group=frrvty \
  --with-pkg-git-version \
  --enable-multipath=64
```

### Developer-Specific Configure Flags

These flags are essential for development work:

| Flag | Purpose |
|---|---|
| `--enable-dev-build` | Enable all developer warnings and assertions; sets `-g -O0` for debug symbols without optimization |
| `--enable-address-sanitizer` | Enable AddressSanitizer (ASan) for memory error detection; catches buffer overflows, use-after-free, leaks |
| `--enable-thread-sanitizer` | Enable ThreadSanitizer (TSan) for data race detection |
| `--enable-memory-sanitizer` | Enable MemorySanitizer for uninitialized memory reads |
| `--enable-undefined-sanitizer` | Enable UndefinedBehaviorSanitizer (UBSan) |
| `--enable-gcov` | Instrument for code coverage collection with gcov/lcov |
| `--enable-fuzzing` | Build fuzzing harnesses (for AFL/libFuzzer) |
| `--enable-snmp` | Build with SNMP AgentX support |
| `--enable-grpc` | Build with gRPC northbound transport |
| `--enable-config-rollbacks` | Enable configuration rollback support (requires libsqlite3) |
| `--disable-doc` | Skip documentation build (faster builds) |

### Recommended Developer Configure

```bash
./configure \
  --prefix=/usr \
  --sbindir=\${prefix}/lib/frr \
  --libdir=\${prefix}/lib/frr \
  --sysconfdir=/etc/frr \
  --localstatedir=/var/run/frr \
  --with-moduledir=\${prefix}/lib/frr/modules \
  --enable-user=frr \
  --enable-group=frr \
  --enable-vty-group=frrvty \
  --with-pkg-git-version \
  --enable-multipath=64 \
  --enable-dev-build \
  --enable-address-sanitizer \
  --disable-doc
```

### Step 3: Compile

```bash
make -j$(nproc)
```

### Step 4: Install

```bash
sudo make install
```

## Incremental Builds

After modifying source files, run `make` again -- it recompiles only changed files:

```bash
make -j$(nproc)
sudo make install
```

If you changed `configure.ac` or any `Makefile.am`, re-run bootstrap and configure:

```bash
./bootstrap.sh
./configure <same-flags>
make -j$(nproc)
```

If you changed only YANG models or CLI definitions (`.yang`, DEFPY definitions), a simple `make` handles regeneration.

## Build Individual Daemons

Build only the daemon you are working on to save time:

```bash
# Build only bgpd
make -j$(nproc) -C bgpd

# Build only zebra
make -j$(nproc) -C zebra

# Build only the core library
make -j$(nproc) -C lib

# Build vtysh (requires all daemon libs)
make -j$(nproc) -C vtysh
```

Install a single daemon:

```bash
sudo make -C bgpd install
sudo systemctl restart frr
```

## Build Distribution Tarball

```bash
make dist
```

This creates `frr-<version>.tar.gz` for release verification.

## Build Checks

Run a quick build verification:

```bash
tools/buildtest.sh    # full build + basic checks
make test             # run unit tests
make check            # run all configured checks
```

## Debug Symbols

The `--enable-dev-build` flag includes `-g` (debug symbols) and `-O0` (no optimization). For production builds with debug symbols, add `CFLAGS` manually:

```bash
./configure CFLAGS="-g -O2" <other-flags>
```

Verify debug info is present:

```bash
file bgpd/.libs/bgpd
# Should show "with debug_info"
```

## AddressSanitizer Workflow

When building with `--enable-address-sanitizer`:

```bash
# ASan requires lower ASLR entropy on some kernels
sudo sysctl vm.mmap_rnd_bits=28

# Run daemons normally -- ASan prints errors to stderr
# In topotests, ASan output goes to /tmp/AddressSanitizer.txt
```

Common ASan findings:
- Buffer overflows (stack and heap)
- Use-after-free
- Double-free
- Memory leaks (at exit)

## Code Coverage Workflow

```bash
# 1. Configure with gcov
./configure --enable-gcov <other-flags>
make -j$(nproc) && sudo make install

# 2. Run tests (unit or topotests)
make test
# or: sudo -E pytest tests/topotests/...

# 3. Collect coverage data
lcov --capture --directory . --output-file coverage.info

# 4. Generate HTML report
genhtml coverage.info --output-directory coverage-html
# Open coverage-html/index.html in a browser
```

## IDE Integration Tips

### Compilation Database (compile_commands.json)

Most IDEs and language servers (clangd, ccls) need `compile_commands.json`:

```bash
# Option 1: Use bear to intercept build commands
bear -- make -j$(nproc)

# Option 2: Use compiledb (pip install compiledb)
compiledb make -j$(nproc)
```

The resulting `compile_commands.json` in the project root enables jump-to-definition, autocompletion, and diagnostics.

### VS Code Settings

Recommended `.vscode/settings.json`:

```json
{
  "C_Cpp.default.compileCommands": "${workspaceFolder}/compile_commands.json",
  "C_Cpp.default.cStandard": "gnu11",
  "files.associations": {
    "*.h": "c"
  }
}
```

### Vim/Neovim with clangd

Ensure clangd is installed and `compile_commands.json` is at the project root. Most LSP plugins auto-detect it.

### Ctags / Cscope

```bash
# Generate ctags for the entire codebase
ctags -R --languages=C,C++ --exclude='.git' .

# Generate cscope database
find . -name '*.[ch]' > cscope.files
cscope -b -q
```

## Common Build Problems

| Problem | Solution |
|---|---|
| `libyang` not found | Install libyang2-dev or build from source |
| `protobuf-c` version mismatch | Install matching protobuf-c-compiler and libprotobuf-c-dev |
| Stale `.o` files after branch switch | `make clean && make -j$(nproc)` |
| `bootstrap.sh` fails | Install autoconf, automake, libtool |
| `flex`/`bison` errors | Install latest versions; FRR requires bison >= 2.7 |
| Permission errors on install | Ensure frr user/group exist; use `sudo make install` |
