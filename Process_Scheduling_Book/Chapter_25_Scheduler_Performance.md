# Chapter 25: Scheduler Performance Optimization

## Learning Goals
- Understand context switch overhead and how to minimize it
- Learn cache locality optimization techniques
- Master load balancing tuning for different workloads
- Know NUMA-aware scheduling optimization

---

## 25.1 Context Switch Overhead Analysis

```
Context switch cost breakdown:

  Component                    │ Cost (approx)  │ Reducible?
  ─────────────────────────────┼────────────────┼───────────
  Direct cost:                 │                │
    Save/restore registers     │ 50-200 ns      │ Partial (FPU lazy)
    Switch page table (CR3/    │ 100-500 ns     │ PCID/ASID
      TTBR0)                   │                │
    TLB invalidation           │ 0-2000 ns      │ PCID/ASID avoid
    Flush pipeline             │ 20-50 ns       │ No
  ─────────────────────────────┼────────────────┼───────────
  Indirect cost:               │                │
    L1 cache misses            │ 1-10 µs        │ CPU affinity
    L2/L3 cache misses         │ 5-50 µs        │ CPU affinity
    TLB misses (re-walk)       │ 1-5 µs         │ Huge pages
    Branch predictor cold      │ 0.5-2 µs       │ CPU affinity
  ─────────────────────────────┼────────────────┼───────────
  TOTAL                        │ 2-60 µs        │
  Thread switch (same process) │ 1-5 µs         │ Threads > procs
  Process switch               │ 5-60 µs        │
```

### Measuring Context Switch Cost

```bash
# perf bench: measure context switch latency
perf bench sched pipe
# (executing 1,000,000 pipe operations between two threads)
# Total time: 5.123 seconds (5.123 µs/op)

# perf stat: count context switches
perf stat -e context-switches,cpu-migrations sleep 10

# vmstat: system-wide context switches per second
vmstat 1
# procs ----memory----- ---swap-- -----io---- --system-- ----cpu----
#  r  b   free   buff    si   so    bi    bo   in    cs  us sy id wa
#  2  0 500000  50000     0    0     0     0  1500  8000  25 10 65  0
#                                               ↑    ↑
#                                         interrupts  ctx switches/sec

# Per-process: /proc/<pid>/status
grep ctxt /proc/<pid>/status
# voluntary_ctxt_switches:    1500
# nonvoluntary_ctxt_switches: 200
```

---

## 25.2 Cache Locality Optimization

### CPU Affinity for Cache Warmth

```c
/* Pin task to specific CPU for cache reuse */
cpu_set_t mask;
CPU_ZERO(&mask);
CPU_SET(2, &mask);
sched_setaffinity(0, sizeof(mask), &mask);

/* Pin to same LLC (Last-Level Cache) domain */
/* CPUs 0-3 share L3 cache on this system */
CPU_SET(0, &mask); CPU_SET(1, &mask);
CPU_SET(2, &mask); CPU_SET(3, &mask);
sched_setaffinity(0, sizeof(mask), &mask);
```

### Scheduler Tuning for Cache Affinity

```bash
# Increase migration cost (keep tasks on same CPU longer)
# Higher value = less migration = better cache reuse
echo 500000 > /proc/sys/kernel/sched_migration_cost_ns
# Default: 500000 (500µs) — consider task "cache hot" for 500µs
# Increase for cache-sensitive workloads

# Reduce load balancing frequency for more stability
# Higher value = less frequent balancing
# Per scheduling domain:
cat /proc/sys/kernel/sched_domain/cpu0/domain0/min_interval
cat /proc/sys/kernel/sched_domain/cpu0/domain0/max_interval
```

### ASID/PCID for TLB Optimization

```
Without ASID/PCID:
  Context switch → flush entire TLB → all entries rebuilt → SLOW

With ASID (ARM64) / PCID (x86):
  Each address space gets an ID tag
  TLB entries tagged with ASID/PCID
  Context switch → change ASID/PCID → old entries REMAIN valid
  No TLB flush needed!

  ARM64: ASID in TTBR0_EL1 (8 or 16 bits = 256 or 65536 ASIDs)
  x86:   PCID in CR3 (12 bits = 4096 PCIDs)
  
  Impact: reduces context switch cost by 1-5µs on TLB-heavy workloads
```

---

## 25.3 Reducing Unnecessary Context Switches

```
Strategy 1: Use SCHED_BATCH for CPU-bound workloads
  → No wakeup preemption → fewer involuntary switches

Strategy 2: Increase sched_min_granularity
  echo 1500000 > /proc/sys/kernel/sched_min_granularity_ns
  # Default: 750000 (0.75ms), increase for throughput

Strategy 3: Increase sched_latency_ns (scheduling period)
  echo 12000000 > /proc/sys/kernel/sched_latency_ns
  # Default: 6ms, increase allows longer runs between switches
  # Trade-off: higher latency for interactive tasks

Strategy 4: Use threads instead of processes
  Thread switch cost much lower (no page table switch)
  Shared address space → shared TLB entries and caches

Strategy 5: Reduce CONFIG_HZ
  CONFIG_HZ=100  → 10ms ticks → fewer scheduler interrupts
  CONFIG_HZ=1000 → 1ms ticks  → more responsive but overhead
  CONFIG_NO_HZ_FULL → tickless on selected CPUs
```

---

## 25.4 Load Balancing Optimization

### For Throughput Workloads

```bash
# Reduce migration — keep tasks where they are
echo 5000000 > /proc/sys/kernel/sched_migration_cost_ns  # 5ms

# Isolate CPUs for dedicated workloads
# Boot parameter:
isolcpus=4,5,6,7 nohz_full=4,5,6,7

# Pin workload to isolated CPUs
taskset -c 4-7 ./compute_heavy_app

# No timer ticks on isolated CPUs (nohz_full)
# No load balancing interference
# Maximum throughput for dedicated workload
```

### For Latency-Sensitive Workloads

```bash
# More aggressive balancing (spread work quickly)
echo 100000 > /proc/sys/kernel/sched_migration_cost_ns  # 100µs

# Reduce wakeup granularity (preempt sooner)
echo 500000 > /proc/sys/kernel/sched_wakeup_granularity_ns

# Ensure idle balancing is aggressive (default is fine)

# Use SCHED_FIFO for critical threads
chrt -f 50 ./latency_critical_thread
```

### NUMA Load Balancing

```bash
# Enable/disable automatic NUMA balancing
echo 1 > /proc/sys/kernel/numa_balancing  # default: on

# For NUMA-sensitive workloads, use numactl:
numactl --membind=0 --cpunodebind=0 ./my_app
# Binds both memory and CPU to NUMA node 0

# Check NUMA statistics
numastat -p <pid>
# Shows per-node memory allocation

# Disable NUMA balancing for workloads that are already optimally placed
echo 0 > /proc/sys/kernel/numa_balancing
```

---

## 25.5 Huge Pages for TLB Efficiency

```
Regular pages: 4KB → TLB covers limited memory
Huge pages:    2MB (x86) or 1GB → TLB covers 512x or 262144x more

Impact on scheduling:
  Fewer TLB misses after context switch
  Fewer page table levels to walk on miss
  
# Transparent Huge Pages (automatic):
echo always > /sys/kernel/mm/transparent_hugepage/enabled

# Explicit huge pages:
echo 1024 > /proc/sys/vm/nr_hugepages  # Pre-allocate 1024×2MB pages
# Application uses mmap with MAP_HUGETLB

# Per-process TLB miss monitoring:
perf stat -e dTLB-load-misses,dTLB-store-misses ./my_app
```

---

## 25.6 Tickless Kernel (NO_HZ)

```
Timer interrupts cause unnecessary wakeups on idle/dedicated CPUs:

CONFIG_HZ=1000 → 1000 interrupts/sec per CPU (even if idle!)

NO_HZ modes:
  CONFIG_NO_HZ_IDLE:  Stop ticks when CPU is idle
    → Default on most systems
    → Idle CPU gets zero interrupts until work arrives

  CONFIG_NO_HZ_FULL:  Stop ticks even when ONE task runs
    → For dedicated real-time/HPC CPUs
    → nohz_full=4-7 boot parameter
    → No timer interrupts at all on those CPUs!
    → Scheduling decisions only on syscall entry/IRQ

Impact on scheduling:
  Without NO_HZ: scheduler_tick() runs 1000x/sec per CPU
  With NO_HZ_IDLE: scheduler_tick() only when CPU is active
  With NO_HZ_FULL: scheduler_tick() only when >1 task runnable
```

---

## 25.7 Scheduling Domain Tuning

```bash
# View domain hierarchy
ls /proc/sys/kernel/sched_domain/cpu0/
# domain0/  (SMT)
# domain1/  (MC - multi-core)
# domain2/  (NUMA)

# Per-domain tunables:
cat /proc/sys/kernel/sched_domain/cpu0/domain1/
# busy_factor       16      # How much busier before migrating
# busy_idx          2       # Load index for busy balancing
# cache_nice_tries  1       # Tries before migrating cache-hot task
# flags             4143    # Domain capability flags
# forkexec_idx      0       # Load index for fork balancing
# idle_idx          0       # Load index for idle balancing
# imbalance_pct     117     # % imbalance threshold (117 = 17% imbalance)
# max_interval      8       # Max balance interval (ticks)
# min_interval      4       # Min balance interval (ticks)
# wake_idx          0       # Load index for wake balancing

# For less migration (throughput):
echo 200 > /proc/sys/kernel/sched_domain/cpu0/domain1/imbalance_pct
echo 16  > /proc/sys/kernel/sched_domain/cpu0/domain1/max_interval
```

---

## 25.8 Performance Tuning Summary

```
┌─────────────────────┬──────────────────────────────────────┐
│ Workload Type        │ Tuning Strategy                     │
├─────────────────────┼──────────────────────────────────────┤
│ HPC / Throughput     │ isolcpus, nohz_full, SCHED_BATCH,  │
│                      │ high migration_cost, large latency  │
├─────────────────────┼──────────────────────────────────────┤
│ Interactive Desktop  │ CONFIG_PREEMPT, low wakeup_gran,   │
│                      │ autogroup, default tunables          │
├─────────────────────┼──────────────────────────────────────┤
│ Real-time / Audio    │ SCHED_FIFO, isolcpus, PREEMPT_RT, │
│                      │ nohz_full, CPU shielding            │
├─────────────────────┼──────────────────────────────────────┤
│ Database / Server    │ NUMA pinning (numactl), large TLB, │
│                      │ SCHED_NORMAL, IRQ affinity          │
├─────────────────────┼──────────────────────────────────────┤
│ Embedded / Android   │ EAS, cgroup bandwidth, SCHED_DL    │
│                      │ for periodic, power-aware tuning     │
└─────────────────────┴──────────────────────────────────────┘
```

---

## Interview Questions

**Q1: A system has 10,000 context switches per second. Is this too many?**
A: It depends. For a web server handling 5,000 req/s with blocking I/O: ~2 switches/req is normal and expected. For a CPU-bound HPC workload: >1000 cs/s indicates excessive preemption — increase sched_latency or use SCHED_BATCH. For an idle system: >1000 cs/s suggests unnecessary wakeups (check timer resolution, background daemons). Profile with `perf stat -e context-switches` and `pidstat -w` to find the cause.

**Q2: How does CPU isolation (isolcpus) improve real-time performance?**
A: `isolcpus` prevents the scheduler from placing ANY task on those CPUs unless explicitly affined. This eliminates: 1) Load balancing decisions on those CPUs. 2) Timer ticks (with nohz_full). 3) Cache pollution from unrelated tasks. 4) Scheduling latency from competing tasks. Only the pinned RT application and its explicitly bound interrupt handlers run on isolated CPUs, giving deterministic sub-100µs latency.

**Q3: Why might increasing CONFIG_HZ from 100 to 1000 hurt throughput?**
A: Higher HZ means more timer interrupts: 1000/sec vs 100/sec per CPU. Each interrupt costs ~1-5µs (enter IRQ, scheduler_tick, update accounting, exit IRQ). On 8 CPUs: 8000 interrupts/sec × 3µs = 24ms/sec overhead (2.4%). Additionally, more frequent tick → more preemption opportunities → more context switches → more cache thrashing. For throughput workloads, CONFIG_HZ=100 or NO_HZ_FULL is preferred.

---

*Next: [Chapter 26 — Kernel Source Code for Scheduling](Chapter_26_Kernel_Source_Code.md)*
