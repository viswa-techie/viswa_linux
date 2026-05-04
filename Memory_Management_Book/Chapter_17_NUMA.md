# Chapter 17: NUMA Memory Management

## Chapter Overview

In NUMA (Non-Uniform Memory Access) systems, memory access time depends on which CPU accesses which memory node. Linux has sophisticated NUMA-aware allocation and page migration. This chapter covers NUMA topology, allocation policies, and automatic NUMA balancing.

---

## 17.1 NUMA Architecture Overview

```
NUMA System (2-socket server):

     Node 0                              Node 1
┌─────────────────┐                ┌─────────────────┐
│  CPU 0-7        │  UPI/QPI Link  │  CPU 8-15       │
│  128MB L3 Cache │◄──────────────►│  128MB L3 Cache │
│                 │   (~100ns      │                 │
│  Memory Ctrl    │    remote      │  Memory Ctrl    │
│       │         │    latency)    │       │         │
│  ┌────┴──────┐  │                │  ┌────┴──────┐  │
│  │ 64GB DDR5 │  │                │  │ 64GB DDR5 │  │
│  │ (~80ns    │  │                │  │ (~80ns    │  │
│  │  local)   │  │                │  │  local)   │  │
│  └───────────┘  │                │  └───────────┘  │
└─────────────────┘                └─────────────────┘

Latency:
  Local memory access:  ~80ns
  Remote memory access: ~130ns  (1.6x slower!)
```

```bash
# View NUMA topology
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7
node 0 size: 65536 MB
node 1 cpus: 8 9 10 11 12 13 14 15
node 1 size: 65536 MB
node distances:
node   0   1
  0:  10  21
  1:  21  10
```

---

## 17.2 NUMA Memory Policies

```c
/* include/linux/mempolicy.h */

/* Per-process or per-VMA NUMA policies */
#define MPOL_DEFAULT    0  /* Use system default (local allocation) */
#define MPOL_PREFERRED  1  /* Prefer specified node, fallback allowed */
#define MPOL_BIND       2  /* Strictly allocate from specified nodes */
#define MPOL_INTERLEAVE 3  /* Round-robin across specified nodes */
#define MPOL_LOCAL      4  /* Allocate from node of triggering CPU */
#define MPOL_PREFERRED_MANY 5  /* Prefer set of nodes (5.12+) */

/* User space API */
#include <numa.h>

/* Set default policy */
set_mempolicy(MPOL_INTERLEAVE, &nodemask, maxnodes);

/* Per-range policy */
mbind(addr, len, MPOL_BIND, &nodemask, maxnodes, flags);

/* Command-line tool */
/* Bind process to node 0 memory */
$ numactl --membind=0 ./my_app

/* Interleave across all nodes */
$ numactl --interleave=all ./my_app
```

---

## 17.3 NUMA Node Allocation

```
Allocation order for a CPU on Node 0:

1. Try Node 0, ZONE_NORMAL  (local, preferred)
2. Try Node 0, ZONE_DMA32   (local, fallback zone)
3. Try Node 1, ZONE_NORMAL  (remote, least preferred)
4. Try Node 0, ZONE_DMA     (local, legacy zone)

This ordering is in the "zonelist" built at boot time.
```

---

## 17.4 NUMA Balancing

Linux automatically migrates pages to the node where they're most accessed.

```
NUMA Balancing Flow:
1. Periodically scan page tables, mark pages PROT_NONE (not accessible)
2. When page is accessed → NUMA hint page fault
3. Fault handler: do_numa_page()
4. Record which CPU/node accessed the page
5. If page is on wrong node → migrate to accessing node

┌───────┐    Access to    ┌────────────┐   Migrate   ┌───────┐
│Node 0 │───────page on──→│NUMA fault  │────page to──→│Node 0 │
│CPU    │    Node 1       │handler     │   Node 0    │memory │
└───────┘                 └────────────┘              └───────┘
```

```bash
# Enable/disable NUMA balancing
$ echo 1 > /proc/sys/kernel/numa_balancing  # Enable
$ echo 0 > /proc/sys/kernel/numa_balancing  # Disable

# Monitor NUMA stats
$ numastat
                           node0           node1
numa_hit                12456789         9876543
numa_miss                  54321           67890
numa_foreign               67890           54321
interleave_hit             12345           12345
local_node              12400000         9800000
other_node                 56789           76543
```

---

## Interview Questions

1. **Q: What is NUMA and why does it matter?**
   A: NUMA (Non-Uniform Memory Access) means memory access latency depends on which CPU accesses which node's memory. Local access is 1.5-2x faster than remote. Linux must allocate memory on the correct node for performance.

2. **Q: What NUMA policies does Linux support?**
   A: Default (local), Preferred (prefer specific node), Bind (strict), Interleave (round-robin), Local (triggering CPU's node). Set via `set_mempolicy()` or `numactl`.

3. **Q: How does automatic NUMA balancing work?**
   A: Kernel periodically marks pages inaccessible. Access causes NUMA hint faults. The handler records which node accessed the page and migrates it if it's on the wrong node.

---

## Summary & Key Takeaways

1. NUMA systems have non-uniform memory access latency — local is faster than remote.
2. Linux allocates from the local node by default.
3. NUMA policies (Bind, Interleave, Preferred) control allocation behavior.
4. Automatic NUMA balancing migrates pages to the most-accessing node.
5. Use `numactl` and `numastat` for monitoring and control.

---

*Next: [Chapter 18 — Contiguous Memory Allocation](Chapter_18_CMA.md)*
