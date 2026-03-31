# FRR Codebase

## Repository Structure

The FRR source tree is organized by daemon and library:

| Directory | Purpose |
|---|---|
| `lib/` | Core library (libfrr) -- shared data structures, event loop, CLI, logging, memory management |
| `zebra/` | Zebra daemon -- RIB, FIB, interface management, kernel integration |
| `bgpd/` | BGP daemon |
| `ospfd/` | OSPFv2 daemon |
| `ospf6d/` | OSPFv3 daemon |
| `isisd/` | IS-IS daemon |
| `ripd/` | RIPv2 daemon |
| `ripngd/` | RIPng daemon |
| `eigrpd/` | EIGRP daemon |
| `babeld/` | Babel daemon |
| `pimd/` | PIM daemon (also handles MLD/IGMP) |
| `ldpd/` | LDP daemon |
| `bfdd/` | BFD daemon |
| `pbrd/` | PBR daemon |
| `nhrpd/` | NHRP daemon |
| `vrrpd/` | VRRP daemon |
| `pathd/` | SRTE path daemon |
| `staticd/` | Static route daemon |
| `mgmtd/` | Management daemon (centralized YANG-based management) |
| `watchfrr/` | Daemon health monitor and restarter |
| `vtysh/` | Unified CLI shell -- connects to all daemons |
| `tools/` | Helper scripts (frr-reload.py, checkpatch, indent.py, etc.) |
| `yang/` | YANG model files |
| `tests/` | Unit tests and topotests |
| `doc/` | Documentation (reStructuredText) |
| `docker/` | Dockerfiles and container build scripts |
| `snapcraft/` | Snap package definitions |
| `redhat/`, `debianpkg/`, `alpine/` | Distribution packaging |

## Core Library (libfrr)

Every daemon links against `libfrr`. It provides the event loop, CLI framework, logging, memory tracking, data structures, northbound/YANG integration, and IPC mechanisms.

Key source files in `lib/`:

| File | Provides |
|---|---|
| `event.c` / `event.h` | Event loop (threadmaster) |
| `command.c` / `command.h` | CLI engine, DEFUN/DEFPY macros, command graph |
| `vty.c` / `vty.h` | VTY terminal handling |
| `prefix.c` / `prefix.h` | IP prefix data structures |
| `table.c` / `table.h` | Radix-tree route table |
| `if.c` / `if.h` | Interface data structures |
| `stream.c` / `stream.h` | Byte-stream buffer for protocol messages |
| `zclient.c` / `zclient.h` | Client side of the zapi protocol (daemon-to-zebra) |
| `typesafe.h` | Typesafe container macros |
| `memory.h` / `memory.c` | Memory type tracking system |
| `log.c` / `zlog.h` | Logging subsystem |
| `hook.h` | Hook (callback subscription) system |
| `northbound.c` / `northbound.h` | YANG northbound API |
| `frr_pthread.c` / `frr_pthread.h` | Pthread wrapper with per-thread event loops |
| `linklist.c` | Legacy linked list (prefer typesafe.h for new code) |

## Key Data Structures

### struct prefix

Represents an IP prefix (address + mask length). Used universally across FRR.

```c
#include "prefix.h"

struct prefix p;
str2prefix("10.0.0.0/24", &p);      /* parse string to prefix */
prefix2str(&p, buf, sizeof(buf));    /* prefix to string */

/* Typed variants */
struct prefix_ipv4 p4;
struct prefix_ipv6 p6;
struct prefix_evpn pevpn;

/* Common operations */
prefix_match(&parent, &child);       /* does parent contain child? */
prefix_same(&a, &b);                 /* exact match? */
prefix_cmp(&a, &b);                  /* compare for sorting */
apply_mask(&p);                      /* zero host bits */
```

### struct route_table / struct route_node

A Patricia trie (radix tree) indexed by `struct prefix`. This is the fundamental lookup structure in zebra's RIB and in protocol daemons.

```c
#include "table.h"

struct route_table *table = route_table_init();

/* Lookup or create a node for a prefix */
struct route_node *rn = route_node_get(table, &prefix);  /* creates if absent */
rn->info = my_data;                                       /* attach user data */
route_unlock_node(rn);                                    /* balance the lock */

/* Exact lookup -- returns NULL if not found */
struct route_node *rn = route_node_lookup(table, &prefix);

/* Longest prefix match */
struct route_node *rn = route_node_match(table, &prefix);

/* Iterate all nodes */
for (rn = route_top(table); rn; rn = route_next(rn)) {
    if (!rn->info)
        continue;
    /* process rn */
}

route_table_finish(table);           /* destroy table */
```

Route nodes use reference counting. Every `route_node_get()`, `route_node_lookup()`, and `route_top()`/`route_next()` call that returns a non-NULL node increments the lock. Call `route_unlock_node()` when done. Breaking out of an iteration early requires calling `route_unlock_node()` on the current node.

### struct interface

Represents a network interface. Managed by zebra and distributed to protocol daemons via zapi.

```c
#include "if.h"

struct interface *ifp = if_lookup_by_name("eth0", VRF_DEFAULT);
if (ifp) {
    ifp->ifindex;       /* kernel interface index */
    ifp->flags;         /* IFF_UP, IFF_RUNNING, etc. */
    ifp->mtu;           /* interface MTU */
    ifp->connected;     /* list of addresses on this interface */
    ifp->info;          /* daemon-specific data pointer */
}
```

### struct stream

A byte-stream buffer used for serializing and deserializing protocol messages (zapi, BGP OPEN/UPDATE, OSPF packets, etc.).

```c
#include "stream.h"

struct stream *s = stream_new(1024);   /* allocate 1024-byte buffer */

/* Write operations (append to end) */
stream_putc(s, 0xFF);                 /* 1 byte */
stream_putw(s, 1234);                 /* 2 bytes (network order) */
stream_putl(s, 100000);               /* 4 bytes (network order) */
stream_putq(s, val64);                /* 8 bytes (network order) */
stream_put(s, data, len);             /* raw bytes */
stream_put_prefix(s, &prefix);        /* serialized prefix */

/* Read operations (consume from current position) */
uint8_t  c = stream_getc(s);
uint16_t w = stream_getw(s);
uint32_t l = stream_getl(s);

/* Position management */
stream_get_endp(s);                   /* number of bytes written */
stream_get_getp(s);                   /* current read position */
stream_set_getp(s, 0);               /* rewind read pointer */
stream_reset(s);                      /* reset both pointers */

stream_free(s);
```

## Event / Thread Model

FRR daemons are event-driven. Each daemon has a `struct event_loop` (historically called "threadmaster") that dispatches tasks:

```c
#include "event.h"

/* In daemon's main() */
struct event_loop *master = event_loop_new();

/* Schedule tasks */
event_add_read(master, callback, arg, fd, &event_ref);     /* when fd is readable */
event_add_write(master, callback, arg, fd, &event_ref);    /* when fd is writable */
event_add_timer(master, callback, arg, seconds, &event_ref); /* after delay */
event_add_timer_msec(master, callback, arg, ms, &event_ref);
event_add_event(master, callback, arg, val, &event_ref);   /* high-priority immediate */

/* Cancel a scheduled task */
event_cancel(&event_ref);

/* Main loop */
struct event event;
while (event_fetch(master, &event))
    event_call(&event);
```

Task callback signature:

```c
void my_callback(struct event *event)
{
    void *arg = EVENT_ARG(event);    /* retrieve the arg pointer */
    int fd  = EVENT_FD(event);       /* retrieve the file descriptor */
    int val = EVENT_VAL(event);      /* retrieve the integer value */
}
```

Fetch priority order: events > timers > I/O.

### Kernel Threads

Some daemons use pthreads for performance. Each pthread runs its own event loop:

```c
#include "frr_pthread.h"

struct frr_pthread_attr attr = { .start = my_start, .stop = my_stop };
struct frr_pthread *fpt = frr_pthread_new(&attr, "IO-thread", "io");
frr_pthread_run(fpt, NULL);

/* Schedule work on another thread's event loop */
event_add_event(fpt->master, callback, arg, 0, NULL);

/* Cancel tasks across threads (thread-safe) */
event_cancel_async(fpt->master, &event_ref, NULL);
```

## CLI Infrastructure

FRR provides a modal CLI built on a directed-graph parser. Commands are defined with macros and installed into command nodes.

### DEFUN and DEFPY

```c
/* Legacy macro -- manual argument parsing */
DEFUN(show_ip_route,
      show_ip_route_cmd,
      "show ip route [json]",
      SHOW_STR
      IP_STR
      "Routing table\n"
      JSON_STR)
{
    bool uj = use_json(argc, argv);
    /* ... */
    return CMD_SUCCESS;
}

/* Modern macro -- automatic typed argument parsing */
DEFPY(show_ip_bgp_neighbor,
      show_ip_bgp_neighbor_cmd,
      "show ip bgp neighbor A.B.C.D$addr [detail$detail]",
      SHOW_STR IP_STR BGP_STR
      "Neighbor\n"
      "Neighbor address\n"
      "Detail\n")
{
    /* addr is struct in_addr, detail is const char* (NULL if omitted) */
    if (detail)
        show_neighbor_detail(vty, addr);
    return CMD_SUCCESS;
}
```

Use `DEFPY` for all new commands. It preprocesses the definition string with `python/clidef.py` to generate typed C parameters.

### DEFPY Type Mappings

| Token | C type | Default if omitted |
|---|---|---|
| `A.B.C.D` | `struct in_addr` | `0.0.0.0` |
| `X:X::X:X` | `struct in6_addr` | `::` |
| `A.B.C.D/M` | `const struct prefix_ipv4 *` | all-zeros struct |
| `X:X::X:X/M` | `const struct prefix_ipv6 *` | all-zeros struct |
| `(0-4294967295)` | `long` | `0` |
| `VARIABLE` | `const char *` | `NULL` |
| `$name` after keyword | `const char *` | `NULL` (absent) / string (present) |

Each non-string type also gets a `_str` variant with the raw input string.

### Grammar Syntax

| Pattern | Meaning |
|---|---|
| `WORD` | Literal keyword match |
| `<A\|B>` | Mutually exclusive alternatives |
| `[optional]` | Optional token |
| `{A\|B\|C}` | One or more in any order |
| `(1-100)` | Numeric range |
| `TOKEN...` | Repeatable (variadic) |
| `$varname` | Name the preceding token |

### Install Commands into Nodes

```c
void my_daemon_vty_init(void)
{
    install_element(VIEW_NODE, &show_ip_route_cmd);
    install_element(BGP_NODE, &neighbor_remote_as_cmd);
}
```

Common nodes: `VIEW_NODE`, `ENABLE_NODE`, `CONFIG_NODE`, `INTERFACE_NODE`, `BGP_NODE`, `OSPF_NODE`, `ISIS_NODE`, `RIP_NODE`.

## Northbound / YANG (mgmtd)

FRR is transitioning to a YANG-model-driven management plane through `mgmtd`.

### Architecture

```
CLI / RESTCONF client
       |
    mgmtd (frontend)         <-- parses commands, manipulates YANG datastore
       |
  BE client library           <-- mgmt_be_client_create()
       |
  backend daemons (zebra, staticd, ripd, ...)
```

`mgmtd` holds the running/candidate configuration datastores. Frontend code (CLI handlers) translate user input into YANG data. Backend daemons receive configuration via XPATH-mapped callbacks.

### Conversion Status

| Status | Daemons |
|---|---|
| Fully converted | staticd, ripd, ripngd, zebra |
| Northbound converted | bfdd, pathd, pbrd, pimd |
| Partially converted | eigrpd, isisd |
| Not yet converted | bgpd, ospfd, ospf6d, ldpd, babeld, nhrpd |

### YANG Module Organization

YANG models live in the `yang/` directory. Each module follows the naming convention `frr-<daemon>.yang` (e.g., `frr-bgp.yang`, `frr-ripd.yang`). Validate models with:

```bash
yanglint -f yang yang/frr-ripd.yang
```

### Northbound Callbacks

Backend daemons define `struct frr_yang_module_info` arrays mapping XPATH paths to callbacks:

```c
const struct frr_yang_module_info frr_ripd_info = {
    .name = "frr-ripd",
    .nodes = {
        { .xpath = "/frr-ripd:ripd/instance",
          .cbs = { .create = ripd_instance_create,
                   .destroy = ripd_instance_destroy } },
        /* ... */
    }
};
```

## Logging Subsystem

FRR uses the `zlog` logging system with thread-safe, RCU-based target management.

### Log Levels

| Level | Use |
|---|---|
| Error | Requires user intervention; significant operational impact |
| Warning | Recoverable issue; operation proceeds |
| Informational | State transitions (peer up/down, interface changes) |
| Debug | Developer diagnostics; gated by `debug <category>` CLI commands |

### Usage

```c
#include "log.h"

zlog_err("Failed to connect to peer %pI4", &addr);
zlog_warn("Received malformed update from %s", peer->host);
zlog_info("BGP peer %s Up", peer->host);
zlog_debug("Processing UPDATE with %d paths", count);
```

### Log Targets

- **stderr** -- active during startup
- **syslog** -- single instance
- **file** -- multiple instances supported
- **crashlog** -- automatic writes to `/var/tmp/frr.<daemon>.crashlog`

Thread-local buffering is available for DEBUG/INFO messages. Higher-priority messages flush the buffer.

### Configuration

```
log syslog informational
log file /var/log/frr/frr.log debugging
log timestamp precision 3
debug bgp updates
debug ospf lsa
```

## Memory Type System

FRR tracks all dynamic memory allocations by type for leak detection and accounting.

### Declare and Define Memory Types

```c
/* In header (.h) */
DECLARE_MTYPE(MY_DATA);

/* In source (.c) -- use STATIC variant for file-local types (preferred in >80% of cases) */
DEFINE_MTYPE_STATIC(BGP, MY_DATA, "My custom data");
/* Or for cross-file types: */
DEFINE_MTYPE(BGP, MY_DATA, "My custom data");
```

### Allocation Functions

```c
struct my_struct *p = XCALLOC(MTYPE_MY_DATA, sizeof(*p));
char *s = XSTRDUP(MTYPE_MY_DATA, input);
p = XREALLOC(MTYPE_MY_DATA, p, new_size);
XFREE(MTYPE_MY_DATA, p);   /* sets p to NULL automatically; NULL-safe */
```

### Inspect at Runtime

```
show memory bgpd
```

## Typesafe Containers

New code must use typesafe containers from `lib/typesafe.h` instead of legacy `linklist.c` or `hash.c`.

### Available Types

| Macro | Structure | Sorted | Unique |
|---|---|---|---|
| `DECLARE_LIST` | Single/double-linked list | No | No |
| `DECLARE_DLIST` | Double-linked list | No | No |
| `DECLARE_HASH` | Hash table | By hash | Yes |
| `DECLARE_RBTREE_UNIQ` | Red-black tree | Yes | Yes |
| `DECLARE_RBTREE_NONUNIQ` | Red-black tree | Yes | No |
| `DECLARE_SKIPLIST_UNIQ` | Skiplist | Yes | Yes |
| `DECLARE_SKIPLIST_NONUNIQ` | Skiplist | Yes | No |
| `DECLARE_HEAP` | 8-ary heap | Partial | No |
| `DECLARE_ATOMLIST` | Atomic single-linked list | No | No |

### Usage Pattern

```c
/* 1. Pre-declare */
PREDECL_DLIST(my_items);

/* 2. Define the item struct with an embedded list member */
struct my_item {
    int value;
    struct my_items_item entry;   /* embedded list linkage */
};

/* 3. Declare the implementation (comparison function for sorted types) */
DECLARE_DLIST(my_items, struct my_item, entry);

/* 4. Use */
struct my_items_head head;
my_items_init(&head);

struct my_item *item = XCALLOC(MTYPE_MY_ITEM, sizeof(*item));
my_items_add_tail(&head, item);

frr_each (my_items, &head, item) {
    /* process item */
}

frr_each_safe (my_items, &head, item) {
    /* safe to remove item during iteration */
    my_items_del(&head, item);
    XFREE(MTYPE_MY_ITEM, item);
}

my_items_fini(&head);
```

Never cast when using typesafe APIs -- a cast indicates a design problem.

## Hook System

Hooks provide type-safe callback subscription points for modular extensibility.

### Declare and Define

```c
/* In header */
DECLARE_HOOK(if_real, (struct interface *ifp), (ifp));

/* In exactly one source file */
DEFINE_HOOK(if_real, (struct interface *ifp), (ifp));
```

A hook can only be called from the source file where it is defined.

### Register and Call

```c
/* Register a callback */
static int my_if_handler(struct interface *ifp)
{
    /* react to interface becoming real */
    return 0;   /* 0 = success by convention */
}
hook_register(if_real, my_if_handler);

/* Or with priority (default is 1000) */
hook_register_prio(if_real, 500, my_if_handler);

/* Invoke all registered callbacks */
hook_call(if_real, ifp);   /* returns sum of callback return values */
```

### Priority Ranges

| Range | Intended Use |
|---|---|
| -1999 to -1000 | Pre-daemon modules |
| -999 to 999 | Daemon code |
| 1000 to 1999 | Post-daemon modules |
