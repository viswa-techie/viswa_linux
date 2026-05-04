# Chapter 16: CPU Scheduling & Load Balancing

## Learning Goals
- Understand how Linux allocates CPU time across multiple cores
- Learn load balancing mechanisms (periodic, idle, migration)
- Master CPU affinity and its impact on scheduling
- Understand Energy Aware Scheduling (EAS), NUMA balancing

---

## 16.1 Multi-Core CPU Scheduling Overview

```
Single-core: scheduler picks ONE task to run — simple
Multi-core:  scheduler must decide WHICH task runs on WHICH CPU

Challenges:
  1. Load balancing  — distribute tasks evenly across CPUs
  2. Cache affinity  — keep tasks on same CPU for cache warmth
  3. NUMA awareness  — prefer local memory
  4. Power efficiency — consolidate on fewer CPUs when idle (EAS)
  5. RT guarantees   — push/pull RT tasks immediately

These goals often CONFLICT:
  Load balancing wants to spread tasks
  Cache affinity wants to keep tasks in place
  Power saving wants to pack tasks together
```

---

## 16.2 Load Balancing Architecture

```
Load balancing happens at each scheduling domain level:

  NUMA domain (balance every 64-128 ms)
    ├── MC domain — Package 0 (balance every 4-8 ms)
    │     ├── SMT group: CPU 0,1
    │     └── SMT group: CPU 2,3
    └── MC domain — Package 1 (balance every 4-8 ms)
          ├── SMT group: CPU 4,5
          └── SMT group: CPU 6,7

Balance triggers:
  1. Periodic (scheduler_tick → trigger_load_balance)
  2. Idle (CPU has no tasks → actively steal from busiest)
  3. Fork/exec (place new task on least-loaded CPU)
  4. Wake-up (select_task_rq → choose best CPU for waking task)
```

---

## 16.3 Periodic Load Balancing

```c
/* kernel/sched/fair.c — simplified flow */

/* Called from scheduler_tick() */
void trigger_load_balance(struct rq *rq)
{
    if (time_after_eq(jiffies, rq->next_balance))
        raise_softirq(SCHED_SOFTIRQ);
}

/* SCHED_SOFTIRQ handler */
static void run_rebalance_domains(struct softirq_action *h)
{
    for_each_domain(cpu, sd) {
        if (time_for_balance(sd)) {
            load_balance(cpu, rq, sd, idle_type);
        }
    }
}
```

### Load Balance Decision

```
load_balance() algorithm:

  1. Find busiest sched_group in current domain
     - Calculate group load (sum of PELT load_avg)
     - Compare with local group load
     
  2. Find busiest rq in that group
     - Pick CPU with highest nr_running or load

  3. Detach tasks from busiest rq
     - Select tasks that won't violate affinity
     - Prefer cache-cold tasks (less loss from migration)

  4. Attach tasks to local rq
     - Enqueue on current CPU

  Busiest CPU                    Idle/Less-loaded CPU
  ┌──────────┐                   ┌──────────┐
  │ T1 T2 T3 │ ── migrate T3 →  │ T4  T3   │
  │ T4       │                   │          │
  └──────────┘                   └──────────┘
```

---

## 16.4 Idle Balancing

When a CPU goes idle (no runnable tasks), it aggressively tries to find work:

```c
/* kernel/sched/fair.c */
static int newidle_balance(struct rq *this_rq, struct rq_flags *rf)
{
    for_each_domain(this_cpu, sd) {
        pulled = load_balance(this_cpu, this_rq, sd, CPU_NEWLY_IDLE);
        if (pulled)
            break;  /* Found work, stop searching */
    }
    /* If still idle → enter idle loop */
}
```

```
Idle balancing progression:

  CPU 3 goes idle:
    1. Check SMT siblings (CPU 2) — nearest, cheapest migration
    2. Check MC group (CPUs 0-3) — same package, shared L3
    3. Check NUMA node (CPUs 0-7) — cross-package, higher cost
    4. Check remote NUMA — most expensive, last resort
    
  Stop at first level where work is found.
```

---

## 16.5 Wake-Up Path: select_task_rq

When a task wakes up, the scheduler selects the best CPU:

```c
/* kernel/sched/fair.c */
static int select_task_rq_fair(struct task_struct *p, int prev_cpu,
                                int wake_flags)
{
    /* Fast path: if prev_cpu is idle and in same domain → use it */
    if (want_affine) {
        new_cpu = select_idle_sibling(p, prev_cpu, target);
    }

    /* Slow path: walk domains for best CPU */
    for_each_domain(cpu, sd) {
        /* find_idlest_group → find_idlest_cpu */
    }

    return new_cpu;
}

/*
 * select_idle_sibling: prefer idle CPUs close to prev_cpu
 *   1. prev_cpu itself (best cache warmth)
 *   2. target (last CPU to run this task)
 *   3. idle CPU in same LLC (shared L3 cache)
 *   4. idle CPU in same core (SMT sibling)
 */
```

### Wake Affine Heuristic

```
Should we wake task on waker's CPU or its previous CPU?

  Task T sleeps, Task W wakes T:
    T previously ran on CPU 2
    W is running on CPU 5

  Decision:
    If W and T share data → wake T near W (CPU 5/sibling)
    If T has warm cache on CPU 2 → wake T on CPU 2

  Heuristic: track "wake affinity" — if W frequently wakes T,
  they likely share data → migrate T closer to W.
```

---

## 16.6 CPU Affinity

### Setting Affinity

```c
#define _GNU_SOURCE
#include <sched.h>

int main(void)
{
    cpu_set_t mask;
    CPU_ZERO(&mask);
    CPU_SET(0, &mask);  /* Pin to CPU 0 */
    CPU_SET(1, &mask);  /* Also allow CPU 1 */

    if (sched_setaffinity(0, sizeof(mask), &mask) < 0)
        perror("sched_setaffinity");

    /* From command line: */
    /* taskset -c 0,1 ./my_app */
    /* taskset -p 0x3 <pid> */
}
```

### Kernel Perspective

```c
/* include/linux/sched.h */
struct task_struct {
    cpumask_t   cpus_mask;          /* allowed CPUs (set by affinity) */
    const struct cpumask *cpus_ptr; /* points to cpus_mask or temp */

    int         nr_cpus_allowed;    /* popcount of cpus_mask */
    int         recent_used_cpu;    /* last CPU used (hint) */
    int         wake_cpu;           /* preferred wake CPU */
};

/* Load balancing respects cpus_mask:
 * Tasks CANNOT be migrated to CPUs not in their mask
 * Reduces balancing flexibility but ensures affinity constraints
 */
```

### CPU Isolation

```bash
# Boot parameter: isolate CPUs 2,3 from general scheduling
# and timer ticks
isolcpus=2,3 nohz_full=2,3

# Only explicitly affined tasks will run on CPUs 2,3
# Ideal for RT/latency-sensitive workloads

# cgroup cpuset: dynamic isolation
echo 0-1 > /sys/fs/cgroup/cpuset/general/cpuset.cpus
echo 2-3 > /sys/fs/cgroup/cpuset/realtime/cpuset.cpus
```

---

## 16.7 NUMA Scheduling

```
NUMA (Non-Uniform Memory Access):

  Node 0 [CPU 0-7, RAM 0-64GB]  ←──── interconnect ────→  Node 1 [CPU 8-15, RAM 64-128GB]
        ↑                                                       ↑
        Local access: ~70ns                                    Local: ~70ns
        Remote access: ~150ns                                  Remote: ~150ns

Goal: Keep tasks close to their memory → better performance
```

### Automatic NUMA Balancing

```
Linux's NUMA balancing (enabled by default):

  1. NUMA fault scanning:
     - Periodically unmap pages (set PTE to PROT_NONE)
     - When task accesses page → NUMA fault
     - Record: which task, which node, which page

  2. Task migration:
     - If task mostly accesses remote memory → migrate task to that node
     - Heavy heuristics to avoid thrashing

  3. Page migration:
     - If page mostly accessed by tasks on another node → migrate page
     - Uses move_pages() / migrate_pages() internally

  /proc/sys/kernel/numa_balancing = 1 (enabled)
```

```c
/* kernel/sched/fair.c */
void task_numa_work(struct callback_head *work)
{
    /* Scan task's address space, set PROT_NONE on sampled pages */
    /* On subsequent access → numa_fault → record access pattern */
}

void task_numa_placement(struct task_struct *p)
{
    /* Analyze NUMA fault statistics */
    /* Decide: should this task move to a different node? */
    /* Consider: task groups, memory placement, bandwidth */
}
```

---

## 16.8 Energy Aware Scheduling (EAS)

For heterogeneous systems (ARM big.LITTLE, DynamIQ):

```
big.LITTLE Architecture:

  ┌────────────────────────┬────────────────────────┐
  │     Little Cluster     │      Big Cluster       │
  │  (power efficient)     │  (high performance)    │
  │  CPU 0-3: Cortex-A55   │  CPU 4-7: Cortex-A78  │
  │  Max freq: 1.8GHz      │  Max freq: 2.84GHz    │
  │  Capacity: 400          │  Capacity: 1024       │
  └────────────────────────┴────────────────────────┘
```

### EAS Decision

```
EAS goal: minimize energy while meeting performance needs

  For each candidate CPU:
    energy = Σ (EM_power(cpu, util) for all cpus_in_performance_domain)

  Select CPU that minimizes total energy.

  Example:
    Light task (util_avg = 100):
      On little CPU: power = 50mW  ← EAS picks this
      On big CPU:    power = 200mW
    
    Heavy task (util_avg = 800):
      On little CPU: can't fit (capacity 400 < 800)
      On big CPU:    power = 500mW ← Only option

  EAS requires:
    - CONFIG_ENERGY_MODEL=y
    - Energy model populated (from DT or cpufreq driver)
    - Asymmetric CPU capacities
    - schedutil cpufreq governor
```

```c
/* kernel/sched/fair.c */
static int find_energy_efficient_cpu(struct task_struct *p, int prev_cpu)
{
    /* For each performance domain (cluster): */
    for_each_pd(pd) {
        /* Compute energy if task placed on best CPU in this domain */
        energy = compute_energy(p, cpu, pd);
        if (energy < best_energy) {
            best_energy = energy;
            best_cpu = cpu;
        }
    }
    /* Return CPU that minimizes total system energy */
}
```

---

## 16.9 Migration Thread

Each CPU has a `migration/<cpu>` kernel thread (SCHED_FIFO, priority 99):

```
Purpose: Handle forced migration requests
  - CPU hotplug (CPU going offline → move all tasks)
  - Active balancing (migrate specific task)
  - stop_machine callbacks

  migration/0, migration/1, migration/2, ...

Flow:
  cpu_stop_queue_work(cpu, &work)
    → Wake migration/<cpu>
    → migration thread runs the callback on that CPU
    → Typical: move task from this CPU to another
```

---

## Load Metrics Summary

| Metric | Source | Used For |
|--------|--------|----------|
| `nr_running` | rq | Simple load count |
| `load_avg` | PELT | Weighted load balancing |
| `util_avg` | PELT | CPU utilization, EAS, frequency scaling |
| `runnable_avg` | PELT | Queue pressure (how overloaded) |
| `cpu_capacity` | arch | Max compute capacity (big vs little) |
| `misfit` | EAS | Task too big for current CPU |

---

## Interview Questions

**Q1: What is the difference between load balancing and task placement?**
A: **Load balancing** is reactive — it periodically checks if CPUs are imbalanced and migrates existing tasks. **Task placement** is proactive — when a task wakes/forks, it selects the best CPU immediately (select_task_rq). Good placement reduces the need for balancing. Both consider cache affinity, NUMA topology, and load.

**Q2: Why doesn't Linux use a single global run queue?**
A: Global queue means every schedule() call contends on one lock — unacceptable for >4 CPUs. Per-CPU queues allow lock-free scheduling on each CPU. The cost is periodic load balancing, which runs at much lower frequency (ms) than scheduling decisions (µs). This trade-off greatly improves scalability.

**Q3: How does EAS differ from traditional load balancing?**
A: Traditional balancing optimizes for **throughput** (spread load evenly). EAS optimizes for **energy** (place tasks on the most efficient CPU that meets their needs). A light task on a big core wastes power. EAS considers the energy model of each CPU and computes the minimum-energy placement. EAS only activates on asymmetric systems (big.LITTLE) and requires the energy model.

---

*Next: [Chapter 17 — Priority and Nice Values](Chapter_17_Priority_Nice_Values.md)*
