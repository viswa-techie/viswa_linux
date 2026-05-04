# Chapter 11: Scheduler Architecture in Linux

## Learning Goals
- Understand the modular scheduling class design
- Map the scheduler core, run queues, and scheduling domains
- See how `schedule()` dispatches to the right class

---

## 11.1 Linux Scheduler Overview

The Linux scheduler is **modular**: a core dispatches to scheduling classes, each implementing a different policy.

```
schedule()
    │
    ▼
┌───────────────────────────────────────────────────────────┐
│                    Scheduler Core                         │
│                  (kernel/sched/core.c)                    │
│                                                           │
│  for_each_class(class) {                                  │
│      next = class->pick_next_task(rq);                    │
│      if (next)                                            │
│          return next;  ← First class with a task wins     │
│  }                                                        │
└───────┬──────────┬──────────┬──────────┬─────────────────┘
        │          │          │          │
   ┌────▼───┐ ┌───▼────┐ ┌───▼────┐ ┌───▼────┐ ┌─────────┐
   │ stop   │ │deadline│ │  RT    │ │ fair   │ │  idle   │
   │ class  │ │ class  │ │ class  │ │ class  │ │  class  │
   │(highest│ │SCHED_  │ │SCHED_  │ │SCHED_  │ │(lowest  │
   │  prio) │ │DEADLINE│ │FIFO/RR │ │NORMAL  │ │  prio)  │
   └────────┘ └────────┘ └────────┘ └────────┘ └─────────┘
   stop_task   kernel/    kernel/    kernel/    idle tasks
   (migration) sched/     sched/     sched/
               deadline.c rt.c       fair.c
```

**Priority order**: stop > deadline > RT > fair > idle. The first class with a runnable task provides the next task.

---

## 11.2 Scheduling Classes

```c
/* include/linux/sched.h */
struct sched_class {
    void (*enqueue_task)(struct rq *rq, struct task_struct *p, int flags);
    void (*dequeue_task)(struct rq *rq, struct task_struct *p, int flags);
    struct task_struct *(*pick_next_task)(struct rq *rq);
    void (*put_prev_task)(struct rq *rq, struct task_struct *p);
    void (*task_tick)(struct rq *rq, struct task_struct *p, int queued);
    void (*check_preempt_curr)(struct rq *rq, struct task_struct *p, int flags);
    void (*set_next_task)(struct rq *rq, struct task_struct *p, bool first);
    /* ... */
};
```

| Class | Policy | Priority Range | Use Case |
|-------|--------|---------------|----------|
| **stop** | — | Highest | CPU migration, hotplug |
| **deadline** | SCHED_DEADLINE | Above RT | Hard real-time with deadlines |
| **rt** | SCHED_FIFO, SCHED_RR | 0-99 (RT) | Soft real-time |
| **fair** | SCHED_NORMAL, SCHED_BATCH | 100-139 (nice -20..19) | Normal tasks (99% of tasks) |
| **idle** | SCHED_IDLE | Lowest | Only when nothing else to run |

### Class Chain

```c
/* Classes linked in priority order */
const struct sched_class stop_sched_class;     /* Highest */
const struct sched_class dl_sched_class;
const struct sched_class rt_sched_class;
const struct sched_class fair_sched_class;
const struct sched_class idle_sched_class;     /* Lowest */

/* Traversal macro */
#define for_each_class(class)  \
    for (class = &stop_sched_class; class; class = class->next)
```

---

## 11.3 Scheduler Core Architecture

### The Run Queue (struct rq)

Each CPU has one run queue — the central data structure:

```c
/* kernel/sched/sched.h */
struct rq {
    raw_spinlock_t      __lock;         /* Protects this rq */
    unsigned int        nr_running;     /* Total runnable tasks */
    struct task_struct  *curr;          /* Currently running task */
    struct task_struct  *idle;          /* Idle task for this CPU */
    u64                 clock;          /* rq clock (ns) */

    struct cfs_rq       cfs;            /* CFS run queue */
    struct rt_rq        rt;             /* RT run queue */
    struct dl_rq        dl;             /* Deadline run queue */

    int                 cpu;            /* CPU number */
    unsigned long       nr_switches;    /* Context switch counter */

    /* Load tracking */
    unsigned long       cpu_load;
    struct sched_avg    avg;            /* PELT: Per-Entity Load Tracking */
};
```

```
Per-CPU Run Queue (struct rq)
┌─────────────────────────────────────────┐
│  CPU 0 Run Queue                        │
│                                         │
│  ┌──────────┐  ┌──────────┐  ┌────────┐│
│  │  dl_rq   │  │  rt_rq   │  │ cfs_rq ││
│  │(deadline)│  │(FIFO/RR) │  │ (fair) ││
│  │ rb-tree  │  │ prio list│  │ rb-tree││
│  └──────────┘  └──────────┘  └────────┘│
│                                         │
│  curr → currently running task_struct   │
│  idle → idle task (swapper/0)           │
│  nr_running = 15                        │
└─────────────────────────────────────────┘
```

### schedule() Simplified

```c
/* kernel/sched/core.c — the heart of scheduling */
static void __sched __schedule(unsigned int sched_mode)
{
    struct rq *rq = this_rq();
    struct task_struct *prev = rq->curr;
    struct task_struct *next;

    rq_lock(rq);

    /* If prev is going to sleep, dequeue it */
    if (!preempt && prev_state) {
        deactivate_task(rq, prev, DEQUEUE_SLEEP);
    }

    /* Pick next task (walks scheduling classes in priority order) */
    next = pick_next_task(rq, prev);

    if (likely(prev != next)) {
        rq->nr_switches++;
        rq->curr = next;

        /* Do the actual context switch */
        context_switch(rq, prev, next);
        /* Never reaches here for prev — returns as next */
    }

    rq_unlock(rq);
}
```

---

## 11.4 Scheduling Domains

Scheduling domains describe CPU topology for load balancing.

```
              ┌─────────────────────────────────────┐
              │  NUMA domain (cross-node balancing)  │
              │  Balance period: 64ms                │
              └──────────┬──────────┬───────────────┘
                         │          │
              ┌──────────▼──┐  ┌───▼────────────┐
              │  MC domain  │  │  MC domain      │
              │ (multi-core)│  │ (multi-core)    │
              │ Balance: 6ms│  │ Balance: 6ms    │
              └──┬──────┬──┘  └──┬──────┬───────┘
                 │      │        │      │
              ┌──▼┐  ┌──▼┐   ┌──▼┐  ┌──▼┐
              │CPU│  │CPU│   │CPU│  │CPU│
              │ 0 │  │ 1 │   │ 2 │  │ 3 │
              └───┘  └───┘   └───┘  └───┘
              NUMA node 0    NUMA node 1
```

```bash
# View scheduling domains
cat /proc/sys/kernel/sched_domain/cpu0/domain0/name   # SMT
cat /proc/sys/kernel/sched_domain/cpu0/domain1/name   # MC
cat /proc/sys/kernel/sched_domain/cpu0/domain2/name   # NUMA

# Balance interval
cat /proc/sys/kernel/sched_domain/cpu0/domain1/min_interval
```

---

## Interview Questions

**Q1: How does the Linux scheduler decide which task runs next?**
A: `pick_next_task()` iterates scheduling classes in priority order: stop → deadline → RT → fair → idle. The first class that has a runnable task provides the next task. For fair (CFS), it picks the task with the smallest vruntime from the red-black tree.

**Q2: Why does Linux use per-CPU run queues instead of a single global queue?**
A: A global queue would require a global lock — severe contention on 100+ CPU systems. Per-CPU queues let each CPU schedule independently with its own lock. Load balancing migrates tasks between queues periodically to maintain fairness.

**Q3: What is the relationship between scheduling classes and policies?**
A: Policies (`SCHED_NORMAL`, `SCHED_FIFO`, etc.) are user-facing names. Classes are the kernel implementation. `SCHED_NORMAL` and `SCHED_BATCH` → `fair_sched_class`. `SCHED_FIFO` and `SCHED_RR` → `rt_sched_class`. `SCHED_DEADLINE` → `dl_sched_class`. The class provides the `pick_next_task()`, `enqueue_task()`, etc. implementations.

---

*Next: [Chapter 12 — Scheduling Algorithms](Chapter_12_Scheduling_Algorithms.md)*
