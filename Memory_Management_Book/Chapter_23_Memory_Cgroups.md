# Chapter 23: Memory Control Groups (cgroups)

## Chapter Overview

Memory cgroups (memcg) allow the kernel to limit, account, and isolate memory usage of process groups. They are the foundation for container resource management (Docker, Kubernetes). This chapter covers cgroup v1 and v2 memory controllers, OOM handling, and memory isolation.

---

## 23.1 What Are Memory Cgroups?

```
cgroups (control groups): Kernel mechanism to organize processes into
hierarchical groups with resource limits.

Memory cgroup = memory controller = limits memory usage per group.

┌─── Root cgroup (/) ───────────────────────────────────┐
│   system total memory                                  │
│                                                        │
│  ┌─── /system.slice ──────┐  ┌─── /user.slice ─────┐ │
│  │ limit: 4GB             │  │ limit: 8GB           │ │
│  │                        │  │                       │ │
│  │ ┌── sshd ──┐          │  │ ┌── user1 ──┐        │ │
│  │ │ 50MB     │          │  │ │ 3GB       │        │ │
│  │ └──────────┘          │  │ └───────────┘        │ │
│  │ ┌── nginx ─┐          │  │ ┌── user2 ──┐        │ │
│  │ │ 500MB    │          │  │ │ 2GB       │        │ │
│  │ └──────────┘          │  │ └───────────┘        │ │
│  └────────────────────────┘  └───────────────────────┘ │
└────────────────────────────────────────────────────────┘

If nginx exceeds 4GB slice limit → OOM killer targets nginx's cgroup.
user1 and user2 isolated — one can't eat the other's memory.
```

---

## 23.2 Memory Cgroup v1 vs v2

```
┌──────────────────────┬──────────────────────┬──────────────────────┐
│ Feature              │ cgroup v1            │ cgroup v2            │
├──────────────────────┼──────────────────────┼──────────────────────┤
│ Hierarchy            │ Multiple hierarchies │ Single unified       │
│ Mount point          │ /sys/fs/cgroup/memory│ /sys/fs/cgroup       │
│ Memory limit file    │ memory.limit_in_bytes│ memory.max           │
│ Soft limit           │ memory.soft_limit    │ memory.high          │
│ Swap limit           │ memory.memsw.limit   │ memory.swap.max      │
│ Current usage        │ memory.usage_in_bytes│ memory.current       │
│ OOM control          │ memory.oom_control   │ memory.oom.group     │
│ Pressure info        │ N/A                  │ memory.pressure      │
│ Socket memory acct   │ Separate controller  │ Unified              │
│ Default in modern    │ Legacy               │ Preferred            │
│ Docker/Kubernetes    │ Older Docker         │ Modern runtimes      │
└──────────────────────┴──────────────────────┴──────────────────────┘
```

---

## 23.3 Memory Cgroup v2 Interface

```bash
# cgroup v2 is mounted at /sys/fs/cgroup (unified hierarchy)

# Create a new cgroup:
$ mkdir /sys/fs/cgroup/myapp

# Move a process into the cgroup:
$ echo $PID > /sys/fs/cgroup/myapp/cgroup.procs

# Set memory limit (hard limit — triggers OOM):
$ echo 512M > /sys/fs/cgroup/myapp/memory.max

# Set memory high watermark (soft limit — triggers throttling):
$ echo 256M > /sys/fs/cgroup/myapp/memory.high
# When usage > high: kernel aggressively reclaims from this cgroup
# Processes may be throttled but NOT killed

# Set swap limit:
$ echo 128M > /sys/fs/cgroup/myapp/memory.swap.max

# Read current usage:
$ cat /sys/fs/cgroup/myapp/memory.current
167936000  # bytes

# Detailed memory statistics:
$ cat /sys/fs/cgroup/myapp/memory.stat
anon 104857600         # Anonymous memory (heap, stack)
file 62914560          # Page cache
kernel 8388608         # Kernel memory (slab, page tables)
shmem 0                # Shared memory / tmpfs
sock 0                 # Socket buffers
anon_thp 0             # Transparent huge pages
file_mapped 41943040   # mmap'd file pages
inactive_anon 52428800 # Inactive anonymous pages (LRU)
active_anon 52428800   # Active anonymous pages (LRU)
inactive_file 31457280 # Inactive file pages (LRU)
active_file 31457280   # Active file pages (LRU)
pgfault 1234567        # Page faults
pgmajfault 42          # Major page faults (disk I/O)

# Memory pressure information (PSI):
$ cat /sys/fs/cgroup/myapp/memory.pressure
some avg10=0.00 avg60=0.00 avg300=0.00 total=12345
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
# "some" = at least one task stalled; "full" = ALL tasks stalled
```

---

## 23.4 Memory Limit Hierarchy

```
Memory limits are hierarchical in cgroup v2:

┌── root (unlimited) ─────────────────────────────────┐
│                                                      │
│  ┌── parent (memory.max = 2G) ────────────────────┐ │
│  │                                                  │ │
│  │  ┌── child_A (memory.max = 1G) ──┐             │ │
│  │  │ Effective limit: min(1G, 2G)=1G│             │ │
│  │  └────────────────────────────────┘             │ │
│  │                                                  │ │
│  │  ┌── child_B (memory.max = 3G) ──┐             │ │
│  │  │ Effective limit: min(3G, 2G)=2G│             │ │
│  │  │ (capped by parent!)            │             │ │
│  │  └────────────────────────────────┘             │ │
│  │                                                  │ │
│  │  Total: child_A + child_B ≤ 2G (parent limit)  │ │
│  └──────────────────────────────────────────────────┘ │
│                                                      │
└──────────────────────────────────────────────────────┘

memory.min — guaranteed minimum (protected from reclaim)
memory.low — best-effort minimum (low-priority reclaim only)
memory.high — soft limit (throttle, no OOM)
memory.max — hard limit (OOM if exceeded)

Priority: min < low < high < max
```

```
Memory protection example:

Parent: memory.max = 4G
Child A: memory.min = 1G, memory.max = 2G
Child B: memory.low = 512M, memory.max = 3G

Under memory pressure:
1. Reclaim from Child B if it exceeds 512M (best-effort protect)
2. Child A's 1G is GUARANTEED — never reclaimed even under extreme pressure
3. Reclaim from whoever exceeds their 'low' first
4. If child reaches 'max' → OOM within that cgroup
```

---

## 23.5 Memory Accounting

```
What is charged to a memory cgroup:

┌────────────────────────┬──────────────┬──────────────────────────┐
│ Memory Type            │ Charged?     │ Notes                    │
├────────────────────────┼──────────────┼──────────────────────────┤
│ Anonymous pages (heap) │ Yes          │ Main memory consumer     │
│ Page cache (file)      │ Yes          │ Charged to first toucher │
│ Shared memory (shmem)  │ Yes          │ Charged to creator       │
│ Kernel stacks          │ Yes (v2)     │ Per-task stack pages     │
│ Page tables            │ Yes (v2)     │ Can be significant       │
│ Slab (kmem)            │ Yes (v2)     │ Kernel objects for cgroup│
│ Socket buffers         │ Yes (v2)     │ TCP/UDP buffers          │
│ tmpfs                  │ Yes          │ Backed by RAM            │
│ Huge pages (THP)       │ Yes          │ Charged as regular pages │
│ hugetlb pages          │ Separate     │ hugetlb cgroup controller│
│ Kernel text/data       │ No           │ Global, not per-cgroup   │
│ DMA buffers            │ No           │ Kernel/driver owned      │
└────────────────────────┴──────────────┴──────────────────────────┘

/* Kernel source: mm/memcontrol.c */
/* mem_cgroup_charge() — charge a page to a cgroup */
/* mem_cgroup_uncharge() — uncharge when page freed */
```

---

## 23.6 OOM Killer and Cgroups

```
When a cgroup reaches memory.max:

1. Try to reclaim pages within the cgroup (cgroup-local reclaim)
2. If reclaim fails → cgroup-local OOM killer

OOM behavior (cgroup v2):
┌──────────────────────────────────────────────────────────────────┐
│ memory.oom.group = 0 (default):                                 │
│   OOM killer selects ONE process within the cgroup to kill.      │
│   Selection based on oom_score_adj and memory usage.             │
│                                                                  │
│ memory.oom.group = 1:                                            │
│   OOM killer kills ALL processes in the cgroup.                  │
│   Used for applications with dependent processes                 │
│   (e.g., database + helper processes — kill all or none).        │
└──────────────────────────────────────────────────────────────────┘

OOM score priority:
$ cat /proc/<pid>/oom_score_adj
# Range: -1000 to 1000
# -1000 = never kill (OOM protect)
# 0     = default
# 1000  = always kill first

# In container environments:
# Kubernetes sets oom_score_adj based on QoS class:
# Guaranteed pods: -998
# BestEffort pods: 1000
# Burstable pods: 2-999 (proportional to request/limit ratio)
```

### OOM Flow Diagram

```
Allocation fails in cgroup C:
│
├─→ try_charge() fails
│     │
│     ├─→ mem_cgroup_reclaim() — reclaim within cgroup
│     │     ├─→ scan LRU lists of cgroup C
│     │     ├─→ try to reclaim inactive pages
│     │     └─→ if enough freed → charge succeeds → done
│     │
│     ├─→ if reclaim insufficient:
│     │     └─→ mem_cgroup_oom() — invoke cgroup OOM
│     │           │
│     │           ├─→ oom.group=0: select victim process
│     │           │     oom_badness() ranks by memory usage
│     │           │     Kill selected process → free memory
│     │           │
│     │           └─→ oom.group=1: kill all in cgroup
│     │                 Send SIGKILL to all tasks
│     │
│     └─→ retry allocation
```

---

## 23.7 Memory Pressure Stall Information (PSI)

```bash
# PSI tracks how much time tasks spend waiting for memory:

$ cat /proc/pressure/memory
some avg10=0.50 avg60=1.20 avg300=0.80 total=853421
full avg10=0.00 avg60=0.10 avg300=0.05 total=12345

# "some": percentage of time at least ONE task stalled on memory
# "full": percentage of time ALL tasks stalled on memory

# Per-cgroup PSI:
$ cat /sys/fs/cgroup/myapp/memory.pressure
some avg10=5.00 avg60=2.50 avg300=1.00 total=9876543
full avg10=0.00 avg60=0.00 avg300=0.00 total=0

# PSI monitoring (event-driven):
# Trigger when "some" pressure exceeds 10% for 1 second:
$ echo "some 100000 1000000" > /proc/pressure/memory
# 100000 µs of stall in 1000000 µs window = 10%
# poll()/epoll() returns when threshold exceeded

# Use case: Container orchestrator detects memory pressure
# → scale out → add more pods → reduce pressure
```

---

## 23.8 Cgroups in Container Environments

```
Docker container memory configuration:

$ docker run --memory=512m --memory-swap=1g --memory-reservation=256m myapp

Translates to cgroup v2:
  memory.max = 512M              # Hard limit
  memory.swap.max = 512M         # 1G total - 512M RAM = 512M swap
  memory.low = 256M              # Soft reservation

Kubernetes Pod Resource Limits:
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: myapp
    resources:
      requests:
        memory: "256Mi"    # → memory.low (soft guarantee)
      limits:
        memory: "512Mi"    # → memory.max (hard limit)
        
# Kubernetes QoS classes based on requests/limits:
# Guaranteed: requests == limits  → OOM score -998
# Burstable:  requests < limits   → OOM score calculated
# BestEffort: no requests/limits  → OOM score 1000 (first to die)

┌───────────────────────────────────────────────────────────────┐
│ Namespace             │ Cgroup                                │
│ ─────────             │ ──────                                │
│ PID namespace:        │ Memory cgroup:                        │
│  Process isolation    │  Memory limit/accounting              │
│                       │                                       │
│ Network namespace:    │ CPU cgroup:                           │
│  Network isolation    │  CPU time limit                       │
│                       │                                       │
│ Mount namespace:      │ IO cgroup:                            │
│  Filesystem isolation │  I/O bandwidth limit                  │
│                       │                                       │
│ Namespaces = WHAT     │ Cgroups = HOW MUCH                    │
│ processes can see     │ resources they can use                │
└───────────────────────────────────────────────────────────────┘
```

---

## 23.9 Memory Cgroup Kernel Internals

```c
/* Key kernel structure: mm/memcontrol.c */

struct mem_cgroup {
    struct cgroup_subsys_state css;  /* cgroup infrastructure */
    
    /* Page counter for memory usage */
    struct page_counter memory;       /* memory.current / memory.max */
    struct page_counter swap;         /* swap.current / swap.max */
    struct page_counter kmem;         /* kernel memory usage */
    struct page_counter tcpmem;       /* TCP buffer memory */
    
    /* Watermarks */
    unsigned long high;               /* memory.high threshold */
    
    /* LRU lists (per-cgroup) */
    struct mem_cgroup_per_node *nodeinfo[];
    /* Each node has its own LRU lists for this cgroup */
    
    /* OOM settings */
    bool oom_group;                   /* Kill all or select one */
    
    /* Memory pressure (PSI) */
    struct psi_group *psi;
};

/* Page → cgroup association:
 * Each page/folio stores its memcg owner:
 * folio->memcg_data → mem_cgroup
 * This allows cgroup-aware reclaim (scan only pages belonging
 * to the target cgroup's LRU lists)
 */
```

---

## Interview Questions

1. **Q: What is the difference between `memory.high` and `memory.max` in cgroup v2?**
   A: `memory.max` is the hard limit — exceeding it triggers the OOM killer. `memory.high` is a soft limit — exceeding it causes aggressive reclaim and process throttling, but no OOM. Use `high` to slow down greedy processes; use `max` as a safety net.

2. **Q: How does the kernel track which pages belong to which cgroup?**
   A: Each page/folio stores a pointer to its owning `mem_cgroup` in `folio->memcg_data`. When a page is allocated and charged to a cgroup, this field is set. This enables cgroup-aware LRU scanning — the reclaimer can scan only pages belonging to a specific cgroup.

3. **Q: How does Kubernetes use memory cgroups?**
   A: K8s sets `memory.max` from the container's `limits.memory` (hard cap), `memory.low` from `requests.memory` (soft guarantee), and `oom_score_adj` based on QoS class (-998 for Guaranteed, 1000 for BestEffort). Pods exceeding limits are OOM-killed; BestEffort pods are killed first under node pressure.

4. **Q: What is PSI and how is it used for memory management?**
   A: Pressure Stall Information tracks the percentage of time tasks are stalled waiting for memory. "some" means at least one task is stalled; "full" means all tasks are stalled. Tools monitor PSI to detect memory pressure and trigger scaling or OOM before the kernel does, enabling proactive container management.

---

## Summary

1. **Memory cgroups** limit and account memory usage per process group.
2. **cgroup v2** (unified hierarchy) is the modern interface with `memory.max`, `memory.high`, `memory.min`, `memory.low`.
3. Limits are **hierarchical** — child can't exceed parent's limit.
4. **OOM killer** is cgroup-aware — kills within the offending cgroup.
5. **PSI** provides memory pressure monitoring for proactive management.
6. **Containers** (Docker/K8s) are built on cgroups + namespaces.
7. Kernel tracks per-cgroup page ownership via `folio->memcg_data`.

---

*Next: [Chapter 24 — Memory Debugging Tools](Chapter_24_Memory_Debugging_Tools.md)*
