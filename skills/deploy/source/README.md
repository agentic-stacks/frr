# Build FRR from Source

## Overview

Building FRR from source provides access to the latest features, bug fixes, and protocol support before packages are released. The process involves installing build dependencies, building the libyang library, compiling FRR with your chosen configure options, and performing post-install setup.

FRR source is available at https://github.com/FRRouting/frr.

## Clone the Repository

```bash
git clone https://github.com/FRRouting/frr.git frr
cd frr
```

### Select a Version

```bash
# Latest development (unstable)
git checkout master

# Specific stable branch
git checkout stable/10.2

# Specific release tag
git tag -l
git checkout frr-10.2.1
```

> **Warning:** The `master` branch is the primary development branch. It should be considered unstable. For production builds, always use a `stable/*` branch or a release tag.

Release tarballs are also available at https://github.com/FRRouting/frr/releases.

## Install Build Dependencies

### Debian 12 / Bookworm

```bash
sudo apt-get update
sudo apt-get install -y \
  git autoconf automake libtool make \
  libprotobuf-c-dev protobuf-c-compiler build-essential \
  python3-dev python3-pytest python3-sphinx libjson-c-dev \
  libelf-dev libreadline-dev cmake libcap-dev bison flex \
  pkg-config texinfo gdb libgrpc-dev python3-grpc-tools
```

### Ubuntu 24.04 / Noble

```bash
sudo apt-get update
sudo apt-get install -y \
  git autoconf automake libtool make libreadline-dev texinfo \
  pkg-config libpam0g-dev libjson-c-dev bison flex \
  libc-ares-dev python3-dev python3-sphinx \
  install-info build-essential libsnmp-dev perl \
  libcap-dev libelf-dev libunwind-dev \
  protobuf-c-compiler libprotobuf-c-dev
```

Optional packages for Ubuntu:

```bash
# gRPC support
sudo apt-get install -y libgrpc++-dev protobuf-compiler-grpc

# Configuration rollbacks (requires SQLite3)
sudo apt-get install -y libsqlite3-dev

# ZeroMQ support
sudo apt-get install -y libzmq5 libzmq3-dev
```

### CentOS 8 / Rocky 8

```bash
sudo dnf install --enablerepo=powertools -y \
  git autoconf pcre-devel automake libtool make \
  readline-devel texinfo net-snmp-devel pkgconfig \
  groff pkgconfig json-c-devel pam-devel bison flex \
  python2-pytest c-ares-devel python2-devel libcap-devel \
  elfutils-libelf-devel libunwind-devel protobuf-c-devel
```

### Fedora

```bash
sudo dnf install -y \
  git autoconf automake libtool make \
  readline-devel texinfo net-snmp-devel groff pkgconfig \
  json-c-devel pam-devel python3-pytest bison flex \
  c-ares-devel python3-devel python3-sphinx perl-core \
  patch libcap-devel elfutils-libelf-devel \
  libunwind-devel protobuf-c-devel
```

## Build and Install libyang

FRR requires libyang version 2.1.128 or newer (version 3 recommended).

### Option 1: Install from FRR Repositories

Pre-built packages are available at https://deb.frrouting.org and https://rpm.frrouting.org. Install both the library and development packages, plus PCRE2 development files:

```bash
# Debian/Ubuntu
sudo apt-get install -y libyang2-dev libpcre2-dev

# RHEL/CentOS/Fedora
sudo dnf install -y libyang-devel pcre2-devel
```

### Option 2: Build from Source

```bash
git clone https://github.com/CESNET/libyang.git
cd libyang
git checkout v3.13.6
mkdir build; cd build
cmake --install-prefix /usr \
      -D CMAKE_BUILD_TYPE:String="Release" ..
make
sudo make install
cd ../..
```

## Configure FRR

### Bootstrap the Build System

```bash
cd frr
./bootstrap.sh
```

### Production Configure

This is the recommended configure command for a production installation:

```bash
./configure \
    --prefix=/usr \
    --includedir=\${prefix}/include \
    --bindir=\${prefix}/bin \
    --sbindir=\${prefix}/lib/frr \
    --libdir=\${prefix}/lib/frr \
    --libexecdir=\${prefix}/lib/frr \
    --sysconfdir=/etc \
    --localstatedir=/var \
    --with-moduledir=\${prefix}/lib/frr/modules \
    --enable-configfile-mask=0640 \
    --enable-logfile-mask=0640 \
    --enable-snmp \
    --enable-multipath=64 \
    --enable-user=frr \
    --enable-group=frr \
    --enable-vty-group=frrvty \
    --enable-fpm \
    --with-pkg-git-version \
    --with-pkg-extra-version=-custom
```

### Developer Configure

For development builds with debug symbols and address sanitizer:

```bash
./configure \
    --prefix=/usr \
    --sbindir=\${prefix}/lib/frr \
    --libdir=\${prefix}/lib/frr \
    --sysconfdir=/etc \
    --localstatedir=/var \
    --enable-user=frr \
    --enable-group=frr \
    --enable-vty-group=frrvty \
    --enable-dev-build \
    --enable-sharpd \
    --enable-werror \
    --with-pkg-git-version
```

The `--enable-dev-build` flag sets `-g3 -O0` for full debug symbols and includes the grammar sandbox.

### Configure Options Reference

| Option | Effect |
|---|---|
| `--prefix=/usr` | Install prefix |
| `--sysconfdir=/etc` | Configuration directory (creates `/etc/frr/`) |
| `--localstatedir=/var` | State directory (creates `/var/run/frr/`) |
| `--enable-user=frr` | Run daemons as this user |
| `--enable-group=frr` | Run daemons as this group |
| `--enable-vty-group=frrvty` | VTY socket group (for vtysh access) |
| `--enable-multipath=64` | ECMP paths (0-999; default 16) |
| `--enable-snmp` | Enable SNMP support |
| `--enable-fpm` | Enable Forwarding Plane Manager module |
| `--enable-grpc` | Enable gRPC northbound plugin |
| `--enable-config-rollbacks` | Enable config rollbacks (requires SQLite3) |
| `--enable-scripting` | Enable Lua scripting |
| `--enable-dev-build` | Development build (`-g3 -O0`, grammar sandbox) |
| `--enable-sharpd` | Build sharpd (testing/route generation daemon) |
| `--enable-werror` | Treat warnings as errors (development only) |
| `--disable-bgpd` | Do not build bgpd |
| `--disable-ospfd` | Do not build ospfd |
| `--disable-ospf6d` | Do not build ospf6d |
| `--disable-ripd` | Do not build ripd |
| `--disable-isisd` | Do not build isisd |
| `--disable-pimd` | Do not build pimd |
| `--disable-ldpd` | Do not build ldpd |
| `--disable-nhrpd` | Do not build nhrpd |
| `--disable-eigrpd` | Do not build eigrpd |
| `--disable-babeld` | Do not build babeld |
| `--disable-bfdd` | Do not build bfdd |
| `--disable-pbrd` | Do not build pbrd |
| `--disable-vrrpd` | Do not build vrrpd |
| `--disable-fabricd` | Do not build fabricd |
| `--disable-staticd` | Do not build staticd |
| `--disable-vtysh` | Do not build vtysh |
| `--disable-doc` | Do not build documentation |
| `--disable-zebra` | Do not build zebra (BGP-only standalone) |
| `--disable-watchfrr` | Do not build watchfrr (breaks systemd integration) |
| `--enable-configfile-mask=0640` | Default permission mask for config files |
| `--enable-logfile-mask=0640` | Default permission mask for log files |
| `--with-pkg-git-version` | Include git hash in version string |
| `--with-pkg-extra-version=VER` | Append custom string to version |

## Build and Install

```bash
make
make check    # optional: run unit tests
sudo make install
```

## Post-Build Setup

### Create the FRR User and Group

Skip this step if the frr user/group already exist (e.g., from a previous package install).

**Debian/Ubuntu:**

```bash
sudo addgroup --system --gid 92 frr
sudo addgroup --system --gid 85 frrvty
sudo adduser --system --ingroup frr --home /var/opt/frr/ \
  --gecos "FRR suite" --shell /bin/false frr
sudo usermod -a -G frrvty frr
```

**RHEL/CentOS/Fedora:**

```bash
sudo groupadd -g 92 frr
sudo groupadd -r -g 85 frrvty
sudo useradd -u 92 -g 92 -M -r -G frrvty -s /sbin/nologin \
  -c "FRR FRRouting suite" -d /var/run/frr frr
```

### Create Configuration Files

**Debian 12:**

```bash
sudo install -m 640 -o frr -g frr /dev/null /etc/frr/frr.conf
sudo install -m 640 -o frr -g frr tools/etc/frr/daemons /etc/frr/daemons
sudo install -m 640 -o frr -g frr tools/etc/frr/support_bundle_commands.conf /etc/frr/support_bundle_commands.conf
```

**Ubuntu / Fedora:**

```bash
sudo install -m 775 -o frr -g frr -d /var/log/frr
sudo install -m 775 -o frr -g frrvty -d /etc/frr
sudo install -m 640 -o frr -g frrvty tools/etc/frr/vtysh.conf /etc/frr/vtysh.conf
sudo install -m 640 -o frr -g frr tools/etc/frr/frr.conf /etc/frr/frr.conf
sudo install -m 640 -o frr -g frr tools/etc/frr/daemons.conf /etc/frr/daemons.conf
sudo install -m 640 -o frr -g frr tools/etc/frr/daemons /etc/frr/daemons
sudo install -m 640 -o frr -g frr tools/etc/frr/support_bundle_commands.conf /etc/frr/support_bundle_commands.conf
```

**CentOS/Rocky:**

```bash
sudo mkdir -p /var/log/frr /etc/frr
sudo touch /etc/frr/zebra.conf
sudo touch /etc/frr/bgpd.conf
sudo touch /etc/frr/ospfd.conf
sudo touch /etc/frr/ospf6d.conf
sudo touch /etc/frr/isisd.conf
sudo touch /etc/frr/ripd.conf
sudo touch /etc/frr/ripngd.conf
sudo touch /etc/frr/pimd.conf
sudo touch /etc/frr/nhrpd.conf
sudo touch /etc/frr/eigrpd.conf
sudo touch /etc/frr/babeld.conf
sudo chown -R frr:frr /etc/frr/
sudo touch /etc/frr/vtysh.conf
sudo chown frr:frrvty /etc/frr/vtysh.conf
sudo chmod 640 /etc/frr/*.conf

sudo install -p -m 644 tools/etc/frr/daemons /etc/frr/
sudo chown frr:frr /etc/frr/daemons
sudo install -m 640 -o frr -g frr tools/etc/frr/support_bundle_commands.conf /etc/frr/support_bundle_commands.conf
```

### Install the systemd Service

```bash
sudo install -p -m 644 tools/frr.service /etc/systemd/system/frr.service
sudo systemctl daemon-reload
sudo systemctl enable frr
```

### Fix Library Path (if needed)

If daemons fail with shared library errors after `make install`:

```bash
echo "include /usr/local/lib" | sudo tee -a /etc/ld.so.conf
sudo ldconfig
```

## Verify the Build

```bash
# Check version
/usr/lib/frr/zebra --version

# Check vtysh
vtysh --version

# Start FRR
sudo systemctl start frr

# Verify daemons
vtysh -c "show daemons"
```

## Next Step

After building and installing FRR, proceed to `skills/deploy/initial-setup` to enable daemons and configure kernel parameters.
