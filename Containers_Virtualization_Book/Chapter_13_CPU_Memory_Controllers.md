# Chapter 13: CPU and Memory Controllers Deep Dive

## Learning Goals
- Understand CFS bandwidth control kernel implementation
- Learn memory controller page accounting and reclaim
- Master NUMA-aware cgroup memory allocation
- Know cgroup-aware OOM killer internals

---

## 1. CFS Bandwidth Control Internals

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  CFS Bandwidth Control (kernel/sched/fair.c):           │
  │                                                           │
  │  struct cfs_bandwidth {                                  │
  │      ktime_t period;          /* period length */        │
  │      u64     quota;           /* quota per period */     │
  │      u64     runtime;         /* remaining runtime */    │
  │      int     nr_periods;      /* periods elapsed */      │
  │      int     nr_throttled;    /* times throttled */      │
  │      u64     throttled_time;  /* total throttle time */  │
  │      struct hrtimer period_timer; /* period refresh */   │
  │      struct hrtimer slack_timer;  /* burst slack */      │
  │  };                                                       │
  │                                                           │
  │  Timeline:                                               │
  │  ┌────────┬────────┬────────┬────────┐                  │
  │  │Period 1│Period 2│Period 3│Period 4│                  │
  │  ├────────┼────────┼────────┼────────┤                  │
  │  │████    │████████│████    │████████│                  │
  │  │used 50%│THROTTLE│used 50%│THROTTLE│                  │
  │  │idle    │waiting │idle    │waiting │                  │
  │  └────────┴────────┴────────┴────────┘                  │
  │                                                           │
  │  quota=50000, period=100000 → 0.5 CPUs                  │
  │  Period 1: uses 50ms, idles rest                         │
  │  Period 2: uses 50ms (quota), throttled remaining 50ms  │
  │                                                           │
  │  Burst (kernel 5.14+):                                   │
  │  cpu.max.burst = 100000                                  │
  │  → Can "bank" unused quota and burst above limit         │
  │  → Helps latency-sensitive apps with bursty CPU usage   │
  │  → Banked: up to burst value of accumulated unused quota│
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Per-CPU Runtime Distribution

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  On SMP systems, quota must be distributed across CPUs: │
  │                                                           │
  │  ┌─────────────────────────────────────────┐             │
  │  │ Global quota pool: 200ms per 100ms period            │
  │  │                                                       │
  │  │ CPU 0 runtime_remaining: 50ms                        │
  │  │ CPU 1 runtime_remaining: 50ms                        │
  │  │ CPU 2 runtime_remaining: 50ms                        │
  │  │ CPU 3 runtime_remaining: 50ms                        │
  │  │ Unassigned: 0ms                                      │
  │  └─────────────────────────────────────────┘             │
  │                                                           │
  │  CPU drains slice:                                       │
  │  1. Task runs on CPU 0, consuming runtime               │
  │  2. CPU 0 slice exhausted → request more from global    │
  │  3. If global pool has runtime → refill local slice     │
  │  4. If global pool empty → CPU 0 throttles its tasks    │
  │                                                           │
  │  Throttling:                                             │
  │  - throttle_cfs_rq() removes cfs_rq from scheduling    │
  │  - Tasks in throttled rq cannot run                     │
  │  - Period timer fires → refills quota → unthrottle      │
  │  - unthrottle_cfs_rq() re-adds to scheduling           │
  │                                                           │
  │  Slack timer:                                            │
  │  - When task sleeps, unused slice is "slack"            │
  │  - Slack timer returns unused runtime to global pool    │
  │  - Prevents wasting quota when tasks block              │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Memory Controller Internals

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Memory accounting per-cgroup (v2):                      │
  │                                                           │
  │  struct mem_cgroup {                                     │
  │      struct page_counter memory;   /* main counter */    │
  │      struct page_counter swap;     /* swap counter */    │
  │      struct page_counter kmem;     /* kernel mem */      │
  │      struct page_counter tcpmem;   /* TCP buffers */     │
  │                                                           │
  │      unsigned long high;           /* memory.high */     │
  │      struct page_counter memsw;    /* memory+swap */     │
  │      unsigned long soft_limit;     /* v1 soft limit */   │
  │                                                           │
  │      struct mem_cgroup_threshold_ary *thresholds;        │
  │      struct list_head oom_notify;                        │
  │      ...                                                 │
  │  };                                                       │
  │                                                           │
  │  What's charged to a cgroup:                             │
  │  ┌────────────────────────────────┬──────────────────┐   │
  │  │ Charged                       │ NOT Charged      │   │
  │  ├────────────────────────────────┼──────────────────┤   │
  │  │ Anonymous pages (heap, stack) │ Kernel stack      │   │
  │  │ Page cache (file-backed)      │ Page tables*      │   │
  │  │ tmpfs / shmem pages           │ (charged in v2)   │   │
  │  │ Kernel slab (kmalloc, etc.)   │ Kernel percpu     │   │
  │  │ TCP/UDP socket buffers        │ allocations       │   │
  │  │ Swap entries                  │                    │   │
  │  │ Huge pages                    │                    │   │
  │  └────────────────────────────────┴──────────────────┘   │
  │                                                           │
  │  Charge path (page fault → allocation):                  │
  │  page_fault → handle_mm_fault → allocate page           │
  │    → charge_memcg(page, memcg)                          │
  │    → page_counter_try_charge(&memcg->memory, nr_pages)  │
  │    → if exceeds memory.max:                              │
  │       try_charge_memcg() → reclaim → OOM if fails      │
  │    → if exceeds memory.high:                             │
  │       schedule reclaim work (throttle allocations)      │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Cgroup-Aware OOM Killer

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Cgroup OOM flow:                                        │
  │                                                           │
  │  1. Process in cgroup allocates memory                   │
  │  2. memory.current exceeds memory.max                    │
  │  3. Kernel tries:                                        │
  │     a. Direct reclaim (scan cgroup's LRU lists)          │
  │     b. Swap out (if swap available and within limit)    │
  │     c. Writeback (flush dirty pages)                    │
  │                                                           │
  │  4. If all reclaim fails → mem_cgroup_oom()              │
  │     ┌────────────────────────────────────────────┐      │
  │     │ Select victim process:                     │      │
  │     │                                            │      │
  │     │ For each process in cgroup:                │      │
  │     │   score = process_pages + swap_pages       │      │
  │     │   score += oom_score_adj (user tunable)    │      │
  │     │   score *= (process is root ? 97/100 : 1)  │      │
  │     │                                            │      │
  │     │ Kill process with highest score            │      │
  │     │ Send SIGKILL to entire thread group        │      │
  │     │                                            │      │
  │     │ Only kills within THIS cgroup!             │      │
  │     │ Other containers and host are safe         │      │
  │     └────────────────────────────────────────────┘      │
  │                                                           │
  │  5. memory.events updated:                              │
  │     oom += 1                                             │
  │     oom_kill += <number of killed processes>             │
  │                                                           │
  │  Monitoring:                                             │
  │  cat /sys/fs/cgroup/docker/ct_a/memory.events            │
  │  low 0           ← exceeded memory.low                  │
  │  high 12         ← exceeded memory.high                 │
  │  max 3           ← hit memory.max (triggered reclaim)   │
  │  oom 1           ← OOM event                            │
  │  oom_kill 1      ← processes actually killed            │
  │  oom_group_kill 0 ← memory.oom.group kills              │
  │                                                           │
  │  memory.oom.group = 1:                                   │
  │  → Kill ALL processes in cgroup (not just heaviest)     │
  │  → Useful for containers: kill entire container on OOM  │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Memory Reclaim Hierarchy

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  v2 memory protection hierarchy:                         │
  │                                                           │
  │  /cgroup/                                                │
  │  └── parent/ (memory.max = 1G)                          │
  │      ├── child_a/ (memory.min=200M, memory.low=400M)   │
  │      │   memory.current = 350M                          │
  │      └── child_b/ (memory.min=100M, memory.low=200M)   │
  │          memory.current = 500M                          │
  │                                                           │
  │  When parent hits 1G limit:                              │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Reclaim priority order:                   │            │
  │  │                                           │            │
  │  │ 1. child_b first (500M > 200M = over low)│            │
  │  │    Reclaim from child_b down to 200M      │            │
  │  │    (but not below 100M = min)             │            │
  │  │                                           │            │
  │  │ 2. If still need more:                    │            │
  │  │    child_a is below low (350M < 400M)     │            │
  │  │    → protected from reclaim               │            │
  │  │                                           │            │
  │  │ 3. If desperate (global pressure):        │            │
  │  │    Reclaim from child_a too                │            │
  │  │    But NOT below min (200M)               │            │
  │  │    memory.min is HARD guarantee            │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Effective protections:                                  │
  │  - memory.min: absolute minimum, never reclaim below    │
  │  - memory.low: best-effort, avoid reclaiming below      │
  │  - memory.high: throttle point (slow, don't kill)       │
  │  - memory.max: hard limit (OOM kill)                    │
  │                                                           │
  │  min < low < high < max                                  │
  │  ────────────┬──────┬──────┬──────────                   │
  │  protected   │prefer│throt │  OOM                        │
  │              │not   │tle   │  kill                        │
  │              │recl  │      │                              │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Explain CPU CFS bandwidth throttling and the burst mechanism.**
**A:** CFS bandwidth control enforces a CPU time limit per period (e.g., 50ms per 100ms = 0.5 CPUs). Kernel implementation: `struct cfs_bandwidth` tracks remaining runtime per period. On SMP systems, the global quota pool distributes slices to per-CPU run queues. When a CPU exhausts its slice, it requests more from the global pool. When the global pool is empty, `throttle_cfs_rq()` removes the cgroup's CFS run queue from scheduling — tasks cannot run until the next period timer refills the quota. The burst mechanism (kernel 5.14+, `cpu.max.burst`) addresses a latency problem: without burst, a web server handling bursty requests gets throttled mid-request, causing tail latency spikes. With burst, unused quota from idle periods is "banked" (up to `cpu.max.burst` microseconds). During a burst, the cgroup can exceed its normal quota using the banked runtime. Example: quota=50ms/100ms with burst=100ms → normally 0.5 CPUs, but can burst to 1.5 CPUs temporarily (50ms + 100ms banked = 150ms in one period). The bank depletes quickly during sustained load, falling back to the normal quota.

**Q2: How does memory protection work in cgroups v2 with memory.min, memory.low, memory.high, and memory.max?**
**A:** These four knobs create a graduated memory management model: (1) `memory.min` — hard protection. The kernel will NEVER reclaim pages from this cgroup if usage is below this value, even under extreme memory pressure. This guarantees a minimum working set (e.g., for critical services). (2) `memory.low` — best-effort protection. The kernel avoids reclaiming from this cgroup when below this value, preferring to reclaim from unprotected cgroups first. Under extreme pressure, it can still be reclaimed (down to memory.min). (3) `memory.high` — throttle boundary. When usage exceeds this, the kernel applies reclaim pressure: allocations are slowed (process enters direct reclaim), but not killed. Creates back-pressure without OOM. No v1 equivalent — this is the recommended mechanism for normal memory control. (4) `memory.max` — hard limit. Exceeding this triggers the cgroup OOM killer (kills process within the cgroup with highest memory usage + oom_score_adj). Safety net for catastrophic memory leaks. Best practice for containers: set `memory.min`=critical working set, `memory.low`=expected usage, `memory.high`=soft limit (throttle), `memory.max`=hard ceiling.

---

## Summary

- CFS bandwidth: global quota pool distributed to per-CPU slices; throttle when exhausted
- Burst (5.14+): bank unused quota for latency-sensitive bursty workloads
- Memory charges: anonymous pages, page cache, slab, shmem, TCP buffers
- Charge path: page fault → try_charge → exceeds max → reclaim → OOM if fails
- Cgroup OOM: kills highest scorer WITHIN cgroup only (containers are isolated)
- memory.oom.group=1: kill all processes in cgroup on OOM (container-style)
- Protection hierarchy: memory.min (hard) < memory.low (soft) < memory.high (throttle) < memory.max (OOM)
- PSI + memory.events: monitoring pressure and OOM events per-cgroup

---

[Previous: Cgroups v2 Unified ←](Chapter_12_Cgroups_v2.md) | [Next: IO Controller and systemd Integration →](Chapter_14_IO_Systemd.md)
