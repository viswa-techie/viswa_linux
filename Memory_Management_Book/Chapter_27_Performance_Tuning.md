# Chapter 27: Performance Tuning and Optimization

## Chapter Overview

Memory performance tuning involves adjusting kernel parameters, allocation strategies, and application behavior to minimize latency, maximize throughput, and reduce memory waste. This chapter covers tuning `/proc/sys/vm/*`, huge page strategies, NUMA optimization, cache-aware programming, and memory-efficient design patterns.

---

## 27.1 Key Tunable Parameters (/proc/sys/vm/)

```bash
# ── Swap and Reclaim Behavior ──

vm.swappiness = 60         # Default: 60 (range: 0-200)
# Controls preference for swapping vs dropping file cache.
# 0   = Avoid swapping (prefer dropping file cache)
# 60  = Balanced (default)
# 100 = Treat anonymous and file-backed equally
# 200 = Aggressively swap anonymous pages
# Database servers: 10-30 (minimize swap for low latency)
# Desktop: 60 (balanced)
# RAM-constrained: 80-100 (use swap freely)

vm.dirty_ratio = 20        # Max dirty pages as % of total RAM
vm.dirty_background_ratio = 10  # Trigger background writeback at %
# Lower values → more frequent but smaller writebacks → less I/O spikes
# Higher values → less frequent but larger writebacks → potential stalls
# Fast SSD: dirty_ratio=40, dirty_background_ratio=10
# Slow HDD: dirty_ratio=10, dirty_background_ratio=5

vm.dirty_writeback_centisecs = 500   # Writeback thread wakes every 5s
vm.dirty_expire_centisecs = 3000     # Pages dirty > 30s get written

vm.vfs_cache_pressure = 100  # Default: 100
# Controls tendency to reclaim dentry/inode cache vs page cache.
# 50  = Keep VFS caches longer (lots of small files)
# 100 = Balanced
# 200 = Aggressively reclaim VFS caches

vm.min_free_kbytes = 65536   # Minimum free memory for emergency allocs
# Higher → more reserved for interrupts/DMA
# Too high → wastes memory. Too low → allocation failures under pressure.
# Typical: 64MB-256MB for servers

vm.zone_reclaim_mode = 0     # NUMA zone reclaim policy
# 0 = Allocate from remote node rather than reclaim local
# 1 = Reclaim local zone before going remote
# 0 recommended for most workloads (bandwidth > latency)

vm.overcommit_memory = 0     # Memory overcommit policy
# 0 = Heuristic (allow reasonable overcommit)
# 1 = Always allow (never fail malloc, may OOM later)
# 2 = Strict (limit: swap + CommitLimit% of RAM)
vm.overcommit_ratio = 50     # With mode=2: % of RAM for commit limit
```

---

## 27.2 Huge Pages Tuning

```bash
# ── Transparent Huge Pages (THP) ──
$ cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never
# always  = THP for all allocations
# madvise = THP only for madvise(MADV_HUGEPAGE) regions
# never   = Disable THP

# For databases (e.g., PostgreSQL):
$ echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
# THP "always" can cause latency spikes from compaction.
# "madvise" gives control to application.

$ cat /sys/kernel/mm/transparent_hugepage/defrag
[always] defer defer+madvise madvise never
# Controls compaction aggressiveness for THP:
# always         = compact immediately (may stall)
# defer          = compact in background
# defer+madvise  = defer for most, immediate for madvise
# madvise        = compact only for madvise'd regions
# never          = no compaction, use only if pages available

# khugepaged: background THP collapse daemon
$ cat /sys/kernel/mm/transparent_hugepage/khugepaged/scan_sleep_millisecs
10000  # Scan interval (10s default)
$ cat /sys/kernel/mm/transparent_hugepage/khugepaged/pages_to_scan
4096   # Pages to scan per interval

# ── Explicit Huge Pages (hugetlbfs) ──
# Reserve at boot for guaranteed availability:
# Boot param: hugepages=1024       ← 1024 × 2MB = 2GB reserved
# Or at runtime:
$ echo 1024 > /proc/sys/vm/nr_hugepages
$ cat /proc/meminfo | grep Huge
HugePages_Total:    1024
HugePages_Free:     1024
HugePages_Surp:        0
Hugepagesize:       2048 kB

# 1GB huge pages (x86_64, boot-time only):
# Boot param: hugepagesz=1G hugepages=4  ← 4 × 1GB
```

### When to Use Each Huge Page Strategy

```
┌──────────────────────┬──────────────────────┬─────────────────────────┐
│ Strategy             │ Best For             │ Considerations          │
├──────────────────────┼──────────────────────┼─────────────────────────┤
│ THP always           │ General workloads    │ Compaction latency      │
│ THP madvise          │ Databases, JVMs      │ App must opt-in         │
│ THP never            │ Latency-sensitive    │ Higher TLB pressure     │
│ hugetlbfs 2MB        │ Database shared bufs │ Reserved, not flexible  │
│ hugetlbfs 1GB        │ VMs, huge datasets   │ Boot-time only, wasted  │
│                      │                      │ if unused               │
└──────────────────────┴──────────────────────┴─────────────────────────┘

TLB Coverage Impact:
4KB pages:  512 TLB entries × 4KB  = 2MB coverage
2MB pages:  512 TLB entries × 2MB  = 1GB coverage  (512× improvement!)
1GB pages:  512 TLB entries × 1GB  = 512GB coverage
```

---

## 27.3 NUMA Optimization

```bash
# ── NUMA-Aware Process Placement ──

# Pin process to specific node:
$ numactl --cpunodebind=0 --membind=0 ./myapp
# Run on Node 0 CPUs, allocate from Node 0 memory

# Interleave memory across all nodes (good for shared data):
$ numactl --interleave=all ./myapp

# NUMA balancing (automatic migration):
$ cat /proc/sys/kernel/numa_balancing
1  # 1=enabled (default), 0=disabled
# Kernel inserts NUMA hint faults → detects access patterns
# → migrates pages to node where they're accessed most
# Good for dynamic workloads; disable for hand-tuned placement

# Memory policy per-process:
$ numactl --preferred=1 ./myapp   # Prefer Node 1 but fallback ok
$ numactl --localalloc ./myapp     # Always allocate local

# Check NUMA statistics:
$ numastat -p $(pidof myapp)
Per-node process memory usage (in MBs) for PID 1234 (myapp)
                   Node 0     Node 1     Total
                ---------- ---------- ----------
Huge              0.00       0.00       0.00
Heap             512.00      64.00     576.00
Stack              8.00       0.00       8.00
Private         1024.00     256.00    1280.00
--- mostly on Node 0 → good if running on Node 0 CPUs

# NUMA distance: memory access latency between nodes
$ numactl --hardware
node distances:
node   0   1
  0:  10  21    # Local=10, Remote=21 (2.1× latency penalty!)
  1:  21  10
```

---

## 27.4 Cache-Aware Programming

```c
/*
 * CPU cache performance patterns for systems programming:
 */

/* 1. Cache Line Alignment */
/* x86_64 cache line = 64 bytes */
/* Ensure hot data fits in same cache line */

struct hot_data {
    uint64_t counter;      /* offset 0  */
    uint64_t timestamp;    /* offset 8  */
    uint32_t flags;        /* offset 16 */
    uint32_t state;        /* offset 20 */
    /* All in one 64-byte cache line! */
} __attribute__((aligned(64)));

/* 2. False Sharing Prevention */
/* Two CPUs writing to same cache line = cache line bouncing */
/* (MESI protocol: invalidate → fetch → modify → invalidate...) */

/* BAD: False sharing */
struct shared_counters {
    atomic_t cpu0_count;   /* Cache line 0 */
    atomic_t cpu1_count;   /* SAME cache line! → bouncing */
};

/* GOOD: Padded to avoid false sharing */
struct padded_counters {
    atomic_t cpu0_count;
    char pad0[60];         /* Pad to fill 64-byte line */
    atomic_t cpu1_count;   /* Separate cache line */
    char pad1[60];
} __attribute__((aligned(64)));

/* Kernel helper: */
struct per_cpu_data {
    atomic_t count;
} ____cacheline_aligned_in_smp;  /* Aligned on SMP builds */

/* 3. Sequential Access Pattern */
/* CPU prefetcher works best with sequential/strided access */
/* Random access → cache misses → 100× slower */

/* BAD: Random access pattern */
for (int i = 0; i < N; i++)
    sum += array[random_indices[i]];  /* Cache miss per access */

/* GOOD: Sequential access */
for (int i = 0; i < N; i++)
    sum += array[i];  /* Prefetcher predicts next access */

/* 4. Structure of Arrays (SoA) vs Array of Structures (AoS) */
/* When iterating over one field of many objects: */

/* BAD for cache: AoS — touches many cache lines */
struct particle { float x, y, z, mass, charge, velocity; };
struct particle particles[N];
for (int i = 0; i < N; i++)
    total_mass += particles[i].mass;  /* 24 bytes between accesses */

/* GOOD for cache: SoA — sequential access to one array */
struct particles_soa {
    float x[N], y[N], z[N], mass[N], charge[N], velocity[N];
};
for (int i = 0; i < N; i++)
    total_mass += particles.mass[i];  /* 4 bytes between accesses */
```

---

## 27.5 Memory-Efficient Design Patterns

```c
/* 1. Object Pooling (avoid alloc/free overhead) */
struct kmem_cache *my_cache;

/* Init: Create dedicated slab cache */
my_cache = kmem_cache_create("my_objects", sizeof(struct my_obj),
                              0, SLAB_HWCACHE_ALIGN, NULL);

/* Alloc: Fast per-CPU allocation */
struct my_obj *obj = kmem_cache_alloc(my_cache, GFP_KERNEL);

/* Free: Returns to per-CPU freelist */
kmem_cache_free(my_cache, obj);

/* Benefit: No fragmentation, no size-class waste, fast alloc/free */

/* 2. Memory Mapping Instead of Read/Write */
/* For large files: mmap instead of read() */
void *data = mmap(NULL, file_size, PROT_READ, MAP_PRIVATE, fd, 0);
/* Pages loaded on demand, shared across processes */
/* No copy: kernel page cache maps directly into user VA */

/* 3. Preallocate + reuse instead of alloc/free per request */
/* Connection pools, buffer pools, thread-local caches */

/* 4. Compact data structures */
/* Use bitfields, enums, __packed where appropriate */
/* Every byte saved × millions of objects = GBs saved */
struct compact_node {
    uint32_t key;
    uint16_t value;
    uint8_t  flags;
    uint8_t  type;     /* 8 bytes total, fits 8 per cache line */
} __attribute__((packed));
```

---

## 27.6 Memory Overcommit Tuning

```
Overcommit policy (vm.overcommit_memory):

┌──────┬──────────────────────────────────────────────────────────┐
│ Mode │ Behavior                                                │
├──────┼──────────────────────────────────────────────────────────┤
│ 0    │ Heuristic: Allow overcommit unless clearly excessive.   │
│      │ malloc() rarely fails. OOM killer handles excess.       │
│      │ Default. Good for most workloads.                       │
├──────┼──────────────────────────────────────────────────────────┤
│ 1    │ Always allow. malloc() NEVER fails (until OOM).         │
│      │ Use for: applications that check malloc carefully.      │
│      │ Risk: OOM with no warning.                              │
├──────┼──────────────────────────────────────────────────────────┤
│ 2    │ Strict accounting. CommitLimit = Swap + RAM × ratio.    │
│      │ malloc() fails if total committed > CommitLimit.        │
│      │ Use for: accounting apps, databases needing guarantees. │
│      │ Downside: May reject allocations that would succeed.    │
└──────┴──────────────────────────────────────────────────────────┘

$ cat /proc/meminfo | grep Commit
CommitLimit:    12288000 kB   # Maximum total committed memory
Committed_AS:    8192000 kB   # Currently committed (allocated, maybe not used)

# Committed_AS > MemTotal is normal with overcommit!
# Processes malloc more than they touch.
# Demand paging means untouched pages cost zero physical RAM.
```

---

## 27.7 Performance Monitoring Checklist

```bash
# ── Quick Health Check ──
# 1. Overall memory state:
$ free -h
              total    used    free  shared  buff/cache available
Mem:           16G     6.0G    2.0G    256M      8.0G      9.5G
Swap:          8.0G    100M    7.9G
# Available >> 0 → healthy. Swap used > 1GB → investigate.

# 2. Memory pressure (PSI):
$ cat /proc/pressure/memory
some avg10=0.00 avg60=0.00 avg300=0.00 total=0
# All zeros = no memory pressure. some > 5% = investigate.

# 3. Page faults:
$ perf stat -e page-faults,major-faults ./myapp
# major-faults > 0 means I/O → slow. Minimize these.

# 4. TLB misses:
$ perf stat -e dTLB-load-misses ./myapp  
# High TLB misses → benefit from huge pages.

# 5. OOM events:
$ dmesg | grep -c "Out of memory"
0    # Should be 0!

# 6. Memory fragmentation:
$ cat /proc/buddyinfo
# High-order (8+) counts near 0 → fragmented
# Fix: echo 1 > /proc/sys/vm/compact_memory

# ── Performance Tuning Workflow ──
# Step 1: Measure baseline (perf stat)
# Step 2: Identify bottleneck (TLB? Cache? Swap? Fragmentation?)
# Step 3: Apply ONE change
# Step 4: Measure again, compare
# Step 5: Iterate
```

---

## 27.8 Application-Specific Tuning Profiles

```
┌──────────────────┬──────────────────────────────────────────────────┐
│ Workload         │ Recommended Tuning                               │
├──────────────────┼──────────────────────────────────────────────────┤
│ Database         │ swappiness=10, THP=madvise, hugetlbfs for       │
│ (PostgreSQL,     │ shared_buffers, dirty_ratio=40, NUMA bind to    │
│  MySQL)          │ local node, overcommit=2                        │
├──────────────────┼──────────────────────────────────────────────────┤
│ Web Server       │ swappiness=30, THP=always, vfs_cache_pressure=  │
│ (nginx, Apache)  │ 50 (keep dentry cache), min_free_kbytes=128M   │
├──────────────────┼──────────────────────────────────────────────────┤
│ JVM Application  │ swappiness=1, THP=always or madvise, pin heap  │
│ (Java, Kotlin)   │ with -XX:+UseHugeTLBFS, NUMA interleave        │
├──────────────────┼──────────────────────────────────────────────────┤
│ HPC / Scientific │ hugetlbfs 1GB pages, NUMA bind, disable swap,  │
│                  │ overcommit=1, THP=always                        │
├──────────────────┼──────────────────────────────────────────────────┤
│ Container Host   │ Per-container memory.max, swappiness per cgroup,│
│ (Kubernetes)     │ memory.high for soft limits, PSI monitoring     │
├──────────────────┼──────────────────────────────────────────────────┤
│ Embedded/Android │ swappiness=100 (with zram), THP=never (limited  │
│                  │ RAM), lowmemorykiller, vmpressure monitoring    │
└──────────────────┴──────────────────────────────────────────────────┘
```

---

## Interview Questions

1. **Q: What does `vm.swappiness` control and what value would you set for a database?**
   A: `vm.swappiness` (0-200) controls the kernel's tendency to swap anonymous pages vs dropping file cache. For databases, set to 10-30 to minimize swapping (database shared buffers are in page cache, and you don't want active data swapped out causing latency spikes). The actual reclaim ratio is scaled by swappiness relative to the value 100.

2. **Q: How do you decide between THP and hugetlbfs?**
   A: THP is transparent and requires no application changes, but can cause compaction latency spikes (set `defrag=madvise` to control this). hugetlbfs requires application awareness and pre-reservation but guarantees huge page availability. Databases often use hugetlbfs for predictability; general applications use THP=madvise for opt-in benefit.

3. **Q: What is false sharing and how do you prevent it?**
   A: False sharing occurs when two CPUs write to different variables that reside on the same cache line (64 bytes). The MESI protocol bounces the cache line between CPUs, severely degrading performance. Fix: pad structures to cache line boundaries (`____cacheline_aligned_in_smp`), or use per-CPU data structures.

4. **Q: How would you tune memory for a container running on Kubernetes?**
   A: Set `memory.max` (hard limit from resources.limits), `memory.high` (soft limit for throttling), `memory.min` (guaranteed reservation from resources.requests). Monitor per-cgroup PSI for pressure. Set `oom_score_adj` based on QoS class. Use per-cgroup `memory.swap.max` to control swap usage.

---

## Summary

1. `/proc/sys/vm/` knobs control swappiness, dirty pages, cache pressure, overcommit.
2. **Huge pages**: THP for transparent use, hugetlbfs for guaranteed allocation. Massive TLB improvement.
3. **NUMA tuning**: Bind processes to nodes, choose memory policy, monitor with `numastat`.
4. **Cache-aware design**: Align to cache lines, avoid false sharing, prefer sequential access.
5. **Memory-efficient patterns**: Object pools, slab caches, mmap, preallocate+reuse.
6. Always **measure before and after** tuning; use `perf stat`, PSI, `vmstat`.
7. Tuning profiles differ by workload: databases, web servers, HPC, containers, embedded.

---

*Next: [Chapter 28 — End-to-End Flow Diagrams](Chapter_28_Flow_Diagrams.md)*
