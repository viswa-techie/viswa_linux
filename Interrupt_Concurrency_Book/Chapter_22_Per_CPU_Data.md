# Chapter 22: Per-CPU Data

## Learning Goals
- Understand per-CPU variables and why they eliminate locking
- Use static and dynamic per-CPU allocation APIs
- Know the preemption rules for per-CPU access
- See real kernel examples of per-CPU usage

---

## 22.1 Concept: Why Per-CPU?

If each CPU has its own private copy of data, no locking is needed — there's no sharing.

```
Traditional shared variable:          Per-CPU variable:
  ┌─────────────────────┐              CPU0: counter_0 = 5
  │  counter = 15       │              CPU1: counter_1 = 4
  │  (shared, needs lock)│              CPU2: counter_2 = 3
  └─────────────────────┘              CPU3: counter_3 = 3
  spin_lock needed for                 No lock needed!
  every increment                      Total = sum of all = 15
```

### Benefits

```
1. No cache-line bouncing: Each CPU writes to its own cache line
2. No locking overhead:    No spinlock acquisition
3. No contention:          Perfect scaling to N CPUs
4. Cache-friendly:         Data stays in L1/L2 of owning CPU
```

### Per-CPU Data Architecture

```
                Memory Layout:
  ┌──────────────────────────────────────────┐
  │  .data..percpu section (template)        │
  │  ┌──────────────────────────────────────┐│
  │  │ var_a │ var_b │ var_c │ ...         ││
  │  └──────────────────────────────────────┘│
  └──────────────────────────────────────────┘
             │  Duplicated at boot
    ┌────────┼────────┬────────┐
    ▼        ▼        ▼        ▼
  CPU 0    CPU 1    CPU 2    CPU 3
  copy     copy     copy     copy
  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
  │var_a│  │var_a│  │var_a│  │var_a│
  │var_b│  │var_b│  │var_b│  │var_b│
  │var_c│  │var_c│  │var_c│  │var_c│
  └─────┘  └─────┘  └─────┘  └─────┘
```

---

## 22.2 Static Per-CPU Variables

### Declaration

```c
#include <linux/percpu.h>

/* Define a per-CPU variable: */
DEFINE_PER_CPU(int, my_counter);                    /* Uninitialized */
DEFINE_PER_CPU(struct stats, cpu_stats);            /* Struct */
DEFINE_PER_CPU(int, my_counter) = 0;                /* Initialized */

/* External declaration (in header): */
DECLARE_PER_CPU(int, my_counter);

/* Read-mostly (placed in separate cache line from write-heavy data): */
DEFINE_PER_CPU_READ_MOSTLY(int, cpu_id);

/* First (aligned to cache line start — avoids false sharing): */
DEFINE_PER_CPU_FIRST(int, critical_counter);

/* Shared with user space (special section): */
DEFINE_PER_CPU_SHARED_ALIGNED(struct data, shared_data);
```

### Access API

```c
/* MUST disable preemption before accessing: */
int val;

preempt_disable();                              /* Prevent migration */
val = __this_cpu_read(my_counter);              /* Read this CPU's copy */
__this_cpu_write(my_counter, val + 1);          /* Write */
__this_cpu_inc(my_counter);                     /* Increment */
__this_cpu_dec(my_counter);                     /* Decrement */
__this_cpu_add(my_counter, 5);                  /* Add */
preempt_enable();

/* Or use get_cpu_var / put_cpu_var (auto preempt disable): */
get_cpu_var(my_counter)++;                      /* Disables preemption */
put_cpu_var(my_counter);                        /* Re-enables preemption */

/* Access another CPU's copy: */
val = per_cpu(my_counter, cpu_id);              /* Read CPU N's copy */

/* this_cpu_ptr — get pointer to this CPU's copy: */
preempt_disable();
struct stats *s = this_cpu_ptr(&cpu_stats);
s->hits++;
preempt_enable();
```

### Atomic Per-CPU Operations

```c
/* These are preemption-safe (single instruction on most archs): */
this_cpu_inc(my_counter);           /* Atomic inc, no explicit preempt_disable */
this_cpu_dec(my_counter);
this_cpu_add(my_counter, val);
this_cpu_read(my_counter);          /* Single safe read */

/* __this_cpu_* variants: NOT safe without explicit preempt_disable.
   Slightly faster (no preemption check). */
```

### Summing Across All CPUs

```c
int total = 0;
int cpu;

for_each_possible_cpu(cpu)
    total += per_cpu(my_counter, cpu);

/* Or for online CPUs: */
for_each_online_cpu(cpu)
    total += per_cpu(my_counter, cpu);
```

---

## 22.3 Dynamic Per-CPU Allocation

For per-CPU data allocated at runtime:

```c
#include <linux/percpu.h>

/* Allocate: */
int __percpu *counters = alloc_percpu(int);
struct stats __percpu *stats = alloc_percpu(struct stats);

/* Allocate with alignment: */
void __percpu *p = __alloc_percpu(size, align);

/* Access: */
preempt_disable();
int *local = this_cpu_ptr(counters);
(*local)++;
preempt_enable();

/* Or: */
int *local = per_cpu_ptr(counters, cpu_id);

/* Free: */
free_percpu(counters);
```

---

## 22.4 Why Preemption Must Be Disabled

```
Problem without preempt_disable:

  CPU 0:                          CPU 1:
  val = per_cpu(counter, 0);     // val = 5
  ─── preempted, migrated to CPU 1 ───
  per_cpu(counter, 0) = val + 1; // WRONG CPU's copy!
                                  // Writes to CPU 0 from CPU 1

With preempt_disable:
  preempt_disable();
  val = __this_cpu_read(counter); // Guaranteed on current CPU
  __this_cpu_write(counter, val + 1);
  preempt_enable();               // Safe to migrate now
```

```
Contexts where preemption is already disabled:
  - Hard IRQ handler         → safe to use __this_cpu_*
  - Soft IRQ / tasklet       → safe
  - Under spin_lock          → safe
  - Under preempt_disable()  → safe
```

---

## 22.5 Per-CPU Usage in the Kernel

### Statistics Counters (Most Common Use)

```c
/* Network statistics: include/linux/netdevice.h */
struct pcpu_sw_netstats {
    u64_stats_sync syncp;     /* For 32-bit atomicity of u64 */
    u64 rx_packets;
    u64 rx_bytes;
    u64 tx_packets;
    u64 tx_bytes;
};

/* Allocated per-CPU: */
dev->tstats = netdev_alloc_pcpu_stats(struct pcpu_sw_netstats);

/* Update in fast path (per-CPU, no lock): */
struct pcpu_sw_netstats *stats = this_cpu_ptr(dev->tstats);
u64_stats_update_begin(&stats->syncp);
stats->rx_packets++;
stats->rx_bytes += skb->len;
u64_stats_update_end(&stats->syncp);

/* Read from all CPUs (for /proc/net/dev): */
for_each_possible_cpu(cpu) {
    struct pcpu_sw_netstats *s = per_cpu_ptr(dev->tstats, cpu);
    do {
        start = u64_stats_fetch_begin(&s->syncp);
        rx_packets = s->rx_packets;
    } while (u64_stats_fetch_retry(&s->syncp, start));
    total_rx += rx_packets;
}
```

### Per-CPU run queues (Scheduler)

```c
/* kernel/sched/core.c */
DEFINE_PER_CPU_SHARED_ALIGNED(struct rq, runqueues);

/* Each CPU's scheduler runqueue is per-CPU — no lock for local access */
struct rq *rq = this_rq();   /* Get current CPU's run queue */
```

### IRQ Counters

```c
/* kernel/softirq.c */
DEFINE_PER_CPU(struct task_struct *, ksoftirqd);

/* arch/x86/kernel/irq.c */
DEFINE_PER_CPU(irq_cpustat_t, irq_stat);
```

---

## 22.6 Per-CPU and IRQ Context

```c
/* Per-CPU data accessed from both process and IRQ contexts: */

DEFINE_PER_CPU(struct stats, my_stats);

/* Process context: must disable IRQs if IRQ handler also accesses */
unsigned long flags;
local_irq_save(flags);
this_cpu_ptr(&my_stats)->hits++;
local_irq_restore(flags);

/* Hard IRQ handler: already IRQ-disabled */
this_cpu_ptr(&my_stats)->irq_count++;

/* If ONLY softirq + process share it: */
local_bh_disable();
this_cpu_ptr(&my_stats)->soft_hits++;
local_bh_enable();
```

---

## 22.7 Complete Driver Example

```c
#include <linux/module.h>
#include <linux/percpu.h>

struct driver_stats {
    u64 reads;
    u64 writes;
    u64 errors;
};

static struct driver_stats __percpu *stats;

static int __init my_init(void)
{
    stats = alloc_percpu(struct driver_stats);
    if (!stats)
        return -ENOMEM;
    return 0;
}

/* Fast path — per-CPU, no locking: */
static ssize_t my_read(struct file *file, char __user *buf,
                        size_t count, loff_t *ppos)
{
    this_cpu_ptr(stats)->reads++;
    /* ... actual read ... */
    return count;
}

/* Slow path — aggregate for sysfs: */
static ssize_t stats_show(struct device *dev,
                           struct device_attribute *attr, char *buf)
{
    u64 total_reads = 0, total_writes = 0, total_errors = 0;
    int cpu;

    for_each_online_cpu(cpu) {
        struct driver_stats *s = per_cpu_ptr(stats, cpu);
        total_reads  += READ_ONCE(s->reads);
        total_writes += READ_ONCE(s->writes);
        total_errors += READ_ONCE(s->errors);
    }

    return sysfs_emit(buf, "reads=%llu writes=%llu errors=%llu\n",
                       total_reads, total_writes, total_errors);
}

static void __exit my_exit(void)
{
    free_percpu(stats);
}
```

---

## 22.8 Per-CPU vs Atomic vs Spinlock

```
Feature           │ Per-CPU        │ Atomic         │ Spinlock
──────────────────┼────────────────┼────────────────┼────────────────
Cache bouncing    │ None           │ Yes (shared CL)│ Yes (shared CL)
Lock overhead     │ None           │ LOCK prefix    │ Spin on contention
Scalability       │ O(1) per CPU   │ O(N) contention│ O(N) contention
Read total value  │ O(N_cpus)      │ O(1)           │ O(1)
Data type         │ Any            │ Integer/bits   │ Any
Sleep allowed?    │ No (preempt off)│ Yes           │ No

Best for:
  Per-CPU:   Counters updated frequently, read rarely (statistics)
  Atomic:    Single integer, moderate contention, need exact value
  Spinlock:  Complex data structures, short critical sections
```

---

## Kernel Source References

```
Per-CPU implementation:
  include/linux/percpu.h               ← API
  include/linux/percpu-defs.h          ← DEFINE_PER_CPU macros
  mm/percpu.c                          ← Dynamic allocation
  include/asm-generic/percpu.h         ← Arch-generic
  arch/x86/include/asm/percpu.h        ← x86 optimized (gs: segment)
  arch/arm64/include/asm/percpu.h      ← ARM64 (TPIDR_EL1)

Kernel usage examples:
  kernel/sched/core.c                  ← runqueues
  net/core/dev.c                       ← network stats
  kernel/softirq.c                     ← ksoftirqd
```

---

## Interview Questions

1. **What is per-CPU data? Why does it eliminate locking?**
2. **Why must preemption be disabled when accessing per-CPU variables?**
3. **What is the difference between this_cpu_inc() and __this_cpu_inc()?**
4. **How do you read the total of a per-CPU counter across all CPUs?**
5. **When would you use per-CPU data vs atomic_t?**
6. **What is alloc_percpu()? When do you use dynamic vs static per-CPU?**
7. **How does the kernel implement per-CPU on x86? (hint: gs segment)**
8. **What happens if an IRQ handler accesses per-CPU data? What care is needed?**
9. **What is u64_stats_sync? Why is it needed for 64-bit per-CPU stats on 32-bit?**
10. **Give a real kernel subsystem that uses per-CPU data and explain why.**

---

## Summary

- Per-CPU data gives each CPU its own copy — no locking, no cache bouncing
- Must disable preemption during access (prevent CPU migration mid-access)
- this_cpu_inc/read/write: preempt-safe single operations
- For IRQ-shared per-CPU data: also disable IRQs (local_irq_save)
- Dynamic allocation: alloc_percpu(type) / free_percpu(ptr)
- Ideal for statistics, per-CPU caches, and frequently-updated counters
- Trade-off: reading total requires summing across all CPUs

---

*Next: [Chapter 23 — RCU (Read-Copy-Update)](Chapter_23_RCU.md)*
