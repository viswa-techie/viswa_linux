# Chapter 22: Kernel Scheduling Initialization

## Learning Goals
- Understand how the scheduler is initialized during boot
- Know per-CPU run queue setup
- Grasp scheduling class registration
- Understand the transition from boot to scheduled execution

---

## 22.1 Scheduler Initialization Overview

```
Scheduler Init During Boot:

start_kernel()
  │
  ├── sched_init()              ← Main scheduler initialization
  │   ├── Initialize root task group
  │   ├── Initialize per-CPU run queues
  │   ├── Initialize scheduling classes
  │   ├── Set up load balancing data
  │   └── Set up idle thread for boot CPU
  │
  ├── ... (other subsystems) ...
  │
  ├── rest_init()
  │   ├── kernel_thread(kernel_init)  ← PID 1
  │   ├── kernel_thread(kthreadd)     ← PID 2
  │   └── cpu_startup_entry()
  │       └── do_idle()               ← Boot CPU enters idle
  │
  └── smp_init() (from kernel_init)
      └── For each secondary CPU:
          └── cpu_startup_entry()
              └── do_idle()           ← Secondary enters idle
```

---

## 22.2 sched_init() Deep Dive

```c
/* kernel/sched/core.c — sched_init() simplified */
void __init sched_init(void)
{
    int i;

    /* Initialize wait queues for autogroup */
    autogroup_init(&init_task);

    /* Per-CPU run queue initialization */
    for_each_possible_cpu(i) {
        struct rq *rq = cpu_rq(i);

        raw_spin_lock_init(&rq->__lock);
        rq->nr_running = 0;
        rq->calc_load_active = 0;
        rq->calc_load_update = jiffies + LOAD_FREQ;

        /* Initialize CFS run queue */
        init_cfs_rq(&rq->cfs);
        /* Initialize RT run queue */
        init_rt_rq(&rq->rt);
        /* Initialize Deadline run queue */
        init_dl_rq(&rq->dl);

        rq->cpu = i;
        rq->online = 0;

        /* NUMA balancing */
        rq_attach_root(rq, &def_root_domain);

        /* Idle thread setup */
        init_idle(current, i);
    }

    /* Set current task scheduling class */
    init_task.sched_class = &fair_sched_class;

    scheduler_running = 1;
}
```

---

## 22.3 Run Queue Architecture

```
Per-CPU Run Queue Structure:

CPU 0 Run Queue (struct rq)              CPU 1 Run Queue (struct rq)
┌──────────────────────────┐             ┌──────────────────────────┐
│ nr_running: 5            │             │ nr_running: 3            │
│ cpu: 0                   │             │ cpu: 1                   │
│ online: 1                │             │ online: 1                │
│                          │             │                          │
│ ┌──────────────────────┐ │             │ ┌──────────────────────┐ │
│ │ CFS Run Queue        │ │             │ │ CFS Run Queue        │ │
│ │ ├── Red-black tree   │ │             │ │ ├── Red-black tree   │ │
│ │ │   of sched_entity  │ │             │ │ │   of sched_entity  │ │
│ │ ├── load: 4096       │ │             │ │ ├── load: 2048       │ │
│ │ └── min_vruntime     │ │             │ │ └── min_vruntime     │ │
│ └──────────────────────┘ │             │ └──────────────────────┘ │
│                          │             │                          │
│ ┌──────────────────────┐ │             │ ┌──────────────────────┐ │
│ │ RT Run Queue         │ │             │ │ RT Run Queue         │ │
│ │ ├── Priority bitmap  │ │             │ │ ├── Priority bitmap  │ │
│ │ └── Per-prio lists   │ │             │ │ └── Per-prio lists   │ │
│ └──────────────────────┘ │             │ └──────────────────────┘ │
│                          │             │                          │
│ ┌──────────────────────┐ │             │ ┌──────────────────────┐ │
│ │ DL Run Queue         │ │             │ │ DL Run Queue         │ │
│ │ └── Red-black tree   │ │             │ │ └── Red-black tree   │ │
│ │     (by deadline)    │ │             │ │     (by deadline)    │ │
│ └──────────────────────┘ │             │ └──────────────────────┘ │
│                          │             │                          │
│ idle: → idle_thread_0    │             │ idle: → idle_thread_1    │
│ curr: → highest_prio_task│             │ curr: → highest_prio_task│
└──────────────────────────┘             └──────────────────────────┘
```

---

## 22.4 Scheduling Classes

```
Scheduling Class Priority (checked in order):

┌─────────────────────────────────────────────────────┐
│  1. stop_sched_class        (highest priority)      │
│     - Migration/stop threads                        │
│     - Cannot be preempted                           │
│                                                     │
│  2. dl_sched_class          (deadline)              │
│     - SCHED_DEADLINE tasks                          │
│     - EDF (Earliest Deadline First)                 │
│                                                     │
│  3. rt_sched_class          (real-time)             │
│     - SCHED_FIFO, SCHED_RR                          │
│     - Fixed priority (1-99)                         │
│                                                     │
│  4. fair_sched_class        (CFS - normal)          │
│     - SCHED_NORMAL, SCHED_BATCH                     │
│     - Virtual runtime based                         │
│                                                     │
│  5. idle_sched_class        (lowest priority)       │
│     - SCHED_IDLE tasks                              │
│     - Only runs when nothing else can               │
└─────────────────────────────────────────────────────┘

Selection: pick_next_task() iterates from highest to lowest class
           until a class says "I have a runnable task"
```

```c
/* Scheduling class linkage — checked in priority order */
const struct sched_class stop_sched_class = {
    .next = &dl_sched_class,
    .enqueue_task = enqueue_task_stop,
    .pick_next_task = pick_next_task_stop,
    /* ... */
};

const struct sched_class dl_sched_class = {
    .next = &rt_sched_class,
    .enqueue_task = enqueue_task_dl,
    .pick_next_task = pick_next_task_dl,
    /* ... */
};
```

---

## 22.5 Idle Thread Creation

```
Idle Thread Setup:

Each CPU gets its own idle thread:

Boot CPU:
  - init_task (PID 0) becomes the idle thread for CPU 0
  - After creating PID 1 and PID 2, it calls:
    cpu_startup_entry(CPUHP_ONLINE) → do_idle()

Secondary CPUs:
  - idle_threads_init() creates idle threads for all CPUs
  - Each secondary CPU after init calls:
    cpu_startup_entry(CPUHP_ONLINE) → do_idle()

do_idle() loop:
  ┌─────────────────────────────────────────┐
  │  while (1) {                            │
  │      tick_nohz_idle_enter();            │
  │      while (!need_resched()) {          │
  │          cpuidle_idle_call();           │
  │          /* Enter C-state (low power) */│
  │      }                                  │
  │      tick_nohz_idle_exit();             │
  │      schedule_idle();     /* run next */│
  │  }                                      │
  └─────────────────────────────────────────┘
```

---

## 22.6 Load Balancing Initialization

```
Load Balancer Setup:

sched_init_smp() — called after SMP bring-up:
  │
  ├── init_sched_domains()
  │   ├── Build scheduling domain hierarchy
  │   │   ┌─────────────────────────────────┐
  │   │   │  NUMA Node Domain (SD_NUMA)     │
  │   │   │  ├── MC Domain (SD_MC)          │
  │   │   │  │   ├── CPU 0 (SMT siblings)   │
  │   │   │  │   ├── CPU 1                   │
  │   │   │  │   ├── CPU 2                   │
  │   │   │  │   └── CPU 3                   │
  │   │   │  └── MC Domain (SD_MC)          │
  │   │   │      ├── CPU 4                   │
  │   │   │      ├── CPU 5                   │
  │   │   │      ├── CPU 6                   │
  │   │   │      └── CPU 7                   │
  │   │   └─────────────────────────────────┘
  │   └── Set balance intervals per domain
  │
  └── open_softirq(SCHED_SOFTIRQ, run_rebalance_domains)
      └── Periodic load balance via softirq
```

---

## Kernel Source References

| Function/File | Path | Purpose |
|-------|------|---------|
| sched_init() | kernel/sched/core.c | Main scheduler initialization |
| init_idle() | kernel/sched/core.c | Set up idle thread for a CPU |
| sched_init_smp() | kernel/sched/core.c | SMP-specific scheduler setup |
| init_cfs_rq() | kernel/sched/fair.c | CFS run queue initialization |
| init_rt_rq() | kernel/sched/rt.c | RT run queue initialization |
| init_dl_rq() | kernel/sched/deadline.c | Deadline run queue initialization |
| struct rq | kernel/sched/sched.h | Per-CPU run queue structure |
| cpu_startup_entry() | kernel/sched/idle.c | Enter idle loop |
| do_idle() | kernel/sched/idle.c | Main idle loop |

---

## Interview Questions

**Q1: When is the scheduler initialized during Linux boot?**
A: `sched_init()` is called early in `start_kernel()`. It initializes per-CPU run queues, scheduling classes (CFS, RT, DL), and the idle thread for the boot CPU. SMP-specific scheduling (load balancing domains) is set up later by `sched_init_smp()` after all CPUs are online.

**Q2: How does the idle thread work?**
A: Each CPU has an idle thread that runs `do_idle()` — an infinite loop that enters low-power C-states when no tasks are runnable. When a task becomes runnable, the idle thread exits its power state and calls `schedule_idle()` to switch to the runnable task. On the boot CPU, PID 0 (init_task) becomes the idle thread.

**Q3: In what order does the scheduler check scheduling classes?**
A: stop → deadline → real-time → fair (CFS) → idle. `pick_next_task()` iterates from highest to lowest priority class. The first class that has a runnable task wins. This ensures deadline and RT tasks always preempt normal tasks.

---

## Summary

- `sched_init()` initializes per-CPU run queues with CFS, RT, and DL sub-queues
- Each CPU gets its own run queue — no global lock contention
- Scheduling classes are checked in priority order: stop > DL > RT > CFS > idle
- PID 0 becomes the idle thread — it runs when nothing else is runnable
- Load balancing domains are set up after SMP init for cross-CPU task migration

---

*Next: [Chapter 23 — Interrupt Initialization](Chapter_23_Interrupt_Initialization.md)*
