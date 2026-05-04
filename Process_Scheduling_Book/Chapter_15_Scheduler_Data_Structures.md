# Chapter 15: Scheduler Data Structures

## Learning Goals
- Master the key data structures powering the Linux scheduler
- Understand run queues, scheduling entities, and scheduling groups
- Learn how cgroup bandwidth control works
- See how PELT (Per-Entity Load Tracking) works

---

## 15.1 Per-CPU Run Queue (struct rq)

Every CPU has one `struct rq` — the master scheduling data structure:

```c
/* kernel/sched/sched.h (simplified) */
struct rq {
    raw_spinlock_t      __lock;         /* protects this rq */

    unsigned int        nr_running;     /* total runnable tasks */
    unsigned int        nr_switches;    /* context switch counter */

    /* Per-class sub-runqueues */
    struct cfs_rq       cfs;            /* CFS runqueue */
    struct rt_rq        rt;             /* RT runqueue */
    struct dl_rq        dl;             /* Deadline runqueue */

    struct task_struct  *curr;          /* currently running task */
    struct task_struct  *idle;          /* this CPU's idle task */
    struct task_struct  *stop;          /* stop-class task */

    u64                 clock;          /* rq clock (ns) */
    u64                 clock_task;     /* clock excluding irq time */

    int                 cpu;            /* this CPU's ID */
    int                 online;         /* CPU is online */

    /* Load and capacity */
    unsigned long       cpu_capacity;
    struct sched_avg    avg;            /* PELT average */

    /* Push/pull for RT and DL */
    struct root_domain  *rd;
    struct sched_domain *sd;

    /* Scheduler statistics */
    unsigned int        yld_count;      /* sched_yield count */
    unsigned int        sched_count;    /* schedule() calls */
    unsigned int        ttwu_count;     /* try_to_wake_up calls */
};
```

```
Per-CPU Run Queue Layout:

  CPU 0 rq                    CPU 1 rq
  ┌──────────────┐            ┌──────────────┐
  │ nr_running: 5│            │ nr_running: 3│
  │ curr: TaskA  │            │ curr: TaskD  │
  ├──────────────┤            ├──────────────┤
  │ dl_rq:       │            │ dl_rq:       │
  │  [DL tasks]  │            │  [DL tasks]  │
  ├──────────────┤            ├──────────────┤
  │ rt_rq:       │            │ rt_rq:       │
  │  prio_array  │            │  prio_array  │
  │  [RT tasks]  │            │  [RT tasks]  │
  ├──────────────┤            ├──────────────┤
  │ cfs_rq:      │            │ cfs_rq:      │
  │  rb_tree     │            │  rb_tree     │
  │  [Fair tasks]│            │  [Fair tasks]│
  └──────────────┘            └──────────────┘
```

---

## 15.2 CFS Run Queue (struct cfs_rq)

```c
/* kernel/sched/sched.h */
struct cfs_rq {
    struct load_weight      load;           /* sum of task weights */
    unsigned int            nr_running;     /* runnable entities */

    u64                     exec_clock;     /* total execution time */
    u64                     min_vruntime;   /* monotonic floor */

    struct rb_root_cached   tasks_timeline; /* RB tree of sched_entity */

    struct sched_entity     *curr;          /* currently running entity */
    struct sched_entity     *next;          /* hint: next to schedule */
    struct sched_entity     *last;          /* hint: last ran */
    struct sched_entity     *skip;          /* hint: skip this one */

    /* Group scheduling */
    struct task_group       *tg;            /* owner task group */
    struct sched_entity     *my_q;          /* parent entity */

    /* PELT (Per-Entity Load Tracking) */
    struct sched_avg        avg;
    unsigned long           runnable_weight;

    /* Bandwidth control */
    int                     throttled;
    u64                     throttled_clock;
    struct cfs_bandwidth    *bandwidth;
};
```

### min_vruntime

```
min_vruntime is a monotonically increasing floor:

  Purpose:
    1. Anchor point for new task placement
    2. Prevents vruntime underflow when tasks leave/join
    3. Normalizes vruntime across migrations

  Update: max(min_vruntime, min(curr->vruntime, leftmost->vruntime))
          Never goes backward

  Task A vruntime: 100
  Task B vruntime: 110
  Task C wakes from sleep, old vruntime: 50
    → Adjusted to max(50, min_vruntime - sched_latency)
    → e.g., max(50, 100 - 6) = 94  (gets small boost, not unfair)
```

---

## 15.3 Scheduling Entity (struct sched_entity)

The schedulable unit — can be a **task** or a **group**:

```c
/* include/linux/sched.h */
struct sched_entity {
    struct load_weight      load;               /* weight from nice */
    struct rb_node          run_node;            /* RB tree linkage */
    unsigned int            on_rq;              /* enqueued? */

    u64                     exec_start;          /* last update time */
    u64                     sum_exec_runtime;    /* total CPU time */
    u64                     prev_sum_exec_runtime;
    u64                     vruntime;            /* virtual runtime */

    u64                     nr_migrations;       /* cross-CPU moves */

    /* Group scheduling hierarchy */
    int                     depth;
    struct sched_entity     *parent;            /* parent group entity */
    struct cfs_rq           *cfs_rq;            /* rq I'm enqueued on */
    struct cfs_rq           *my_q;              /* rq I own (if group) */

    /* PELT */
    struct sched_avg        avg;
};
```

### Task vs Group Entity

```
Without group scheduling (CONFIG_FAIR_GROUP_SCHED=n):
  Each sched_entity = one task
  
  cfs_rq (root)
    ├── se (Task A)
    ├── se (Task B)
    └── se (Task C)

With group scheduling (CONFIG_FAIR_GROUP_SCHED=y):
  sched_entity can represent a GROUP containing a sub-cfs_rq

  cfs_rq (root)
    ├── se (Task A)
    ├── se (Group X) ──→ cfs_rq (Group X)
    │                      ├── se (Task B)
    │                      └── se (Task C)
    └── se (Group Y) ──→ cfs_rq (Group Y)
                           ├── se (Task D)
                           └── se (Task E)

  Group X and Group Y compete for CPU as single entities
  Tasks within each group compete among themselves
```

---

## 15.4 RT Data Structures

```c
/* kernel/sched/sched.h */
struct rt_rq {
    struct rt_prio_array    active;             /* priority array */
    unsigned int            rt_nr_running;      /* total RT tasks */
    unsigned int            rr_nr_running;      /* SCHED_RR count */
    struct {
        int                 curr;               /* highest prio running */
        int                 next;               /* next highest */
    } highest_prio;
    int                     overloaded;         /* >1 RT task on CPU? */
    struct rt_bandwidth     tg_rt_bandwidth;    /* group bandwidth */
    int                     rt_throttled;
};

struct rt_prio_array {
    DECLARE_BITMAP(bitmap, MAX_RT_PRIO + 1);   /* 100 priority bits */
    struct list_head queue[MAX_RT_PRIO];        /* per-priority FIFO */
};
```

```
O(1) RT task selection:

Step 1: idx = sched_find_first_bit(bitmap)     → O(1) via CPU instruction
Step 2: task = list_first_entry(&queue[idx])   → O(1)

bitmap: [0 0 1 0 0 ... 1 0 0]
             ↑ priority 2         ↑ priority 50

Result: Always picks highest-priority RT task in O(1)
```

---

## 15.5 Deadline Data Structures

```c
/* kernel/sched/sched.h */
struct dl_rq {
    struct rb_root_cached   root;               /* RB tree by deadline */
    unsigned long           dl_nr_running;      /* DL tasks count */
    struct {
        u64                 curr;               /* earliest deadline */
        u64                 next;
    } earliest_dl;
    int                     overloaded;
    struct dl_bandwidth     dl_runtime;         /* total DL bandwidth */
};

/* include/linux/sched.h */
struct sched_dl_entity {
    struct rb_node          rb_node;
    u64                     dl_runtime;         /* budget per period */
    u64                     dl_deadline;        /* relative deadline */
    u64                     dl_period;          /* period */
    u64                     runtime;            /* remaining budget */
    u64                     deadline;           /* absolute deadline */
    unsigned int            dl_throttled : 1;
    unsigned int            dl_yielded   : 1;
    unsigned int            dl_non_contending : 1;
};
```

---

## 15.6 PELT — Per-Entity Load Tracking

PELT (introduced ~3.8, refined through 5.x) tracks CPU utilization of each entity over time with geometric decay:

```
PELT computes a decaying average of CPU demand:

  load_avg   = runnable_time × weight (demand weighted by priority)
  util_avg   = running_time / elapsed_time (actual utilization 0-1024)
  runnable_avg = runnable_time / elapsed_time (demand including queue wait)

Decay formula (per 1024µs period):
  avg = avg × (y^periods) + new_contribution

  Where y ≈ 0.978 (half-life ≈ 32ms)

Example:
  Task runs continuously for 32ms then sleeps:
    util_avg rises to ~512 (50% of max 1024)
    After sleeping 32ms: decays to ~256
    After sleeping 64ms: decays to ~128
    
    ┌──────┐
    │ 1024 │     ╱──────╲
    │  512 │   ╱          ╲
    │  256 │ ╱              ╲────
    │    0 │────────────────────────
    └──────┘ running  sleeping
```

### PELT in Scheduling Decisions

```c
/* kernel/sched/fair.c — load tracking */
struct sched_avg {
    u64                 last_update_time;
    u64                 load_sum;
    u64                 runnable_sum;
    u32                 util_sum;
    u32                 period_contrib;
    unsigned long       load_avg;       /* weighted load demand */
    unsigned long       runnable_avg;   /* runnable demand */
    unsigned long       util_avg;       /* actual utilization */
};

/* Usage in load balancing:
 *   load_avg  → task weight in load balancing decisions
 *   util_avg  → CPU utilization for EAS (Energy Aware Scheduling)
 *              and OPP (frequency) selection
 */
```

---

## 15.7 Scheduling Domains & Groups

```
Scheduling domains represent CPU topology for load balancing:

  ┌─────────── NUMA Domain (cross-socket) ──────────┐
  │                                                   │
  │  ┌──── MC Domain (package) ────┐  ┌──── MC ────┐ │
  │  │                              │  │            │ │
  │  │ ┌─SMT─┐  ┌─SMT─┐  ┌─SMT─┐ │  │ ┌─SMT─┐   │ │
  │  │ │C0 C1│  │C2 C3│  │C4 C5│ │  │ │C6 C7│   │ │
  │  │ └─────┘  └─────┘  └─────┘ │  │ └─────┘   │ │
  │  └──────────────────────────────┘  └──────────┘ │
  └──────────────────────────────────────────────────┘

Each domain level has:
  - struct sched_domain (properties, flags)
  - struct sched_group (group of CPUs for balancing)
  - Balance interval (how often to check)
  - Imbalance threshold
```

```c
/* kernel/sched/sched.h */
struct sched_domain {
    struct sched_domain     *parent;
    struct sched_domain     *child;
    struct sched_group      *groups;        /* groups of CPUs */
    unsigned long           span[0];        /* CPU bitmask */
    unsigned int            span_weight;    /* # CPUs in domain */

    /* Balancing parameters */
    unsigned int            balance_interval;
    unsigned int            busy_factor;
    unsigned int            imbalance_pct;

    unsigned int            flags;          /* SD_SHARE_CPUCAPACITY, etc. */
    enum sched_domain_level level;          /* SMT, MC, NUMA */
};
```

---

## 15.8 Task Group and CFS Bandwidth

```c
/* kernel/sched/sched.h */
struct task_group {
    struct sched_entity     **se;           /* per-CPU sched_entity */
    struct cfs_rq           **cfs_rq;       /* per-CPU cfs_rq */
    struct rt_rq            **rt_rq;        /* per-CPU rt_rq */

    unsigned long           shares;         /* cpu.shares (weight) */

    /* CFS bandwidth control */
    struct cfs_bandwidth    cfs_bandwidth;
};

struct cfs_bandwidth {
    raw_spinlock_t          lock;
    ktime_t                 period;         /* measurement window */
    u64                     quota;          /* max runtime per period */
    u64                     runtime;        /* remaining in current period */
    int                     nr_throttled;   /* how many cfs_rqs throttled */
};
```

### cgroup CPU Control

```bash
# Create a cgroup with CPU limits
mkdir /sys/fs/cgroup/cpu/mygroup

# Limit: 50% CPU (50ms every 100ms)
echo 50000 > /sys/fs/cgroup/cpu/mygroup/cpu.cfs_quota_us
echo 100000 > /sys/fs/cgroup/cpu/mygroup/cpu.cfs_period_us

# Weight (shares) for proportional allocation
echo 512 > /sys/fs/cgroup/cpu/mygroup/cpu.shares  # half of default 1024

# cgroup v2 equivalent
echo "50000 100000" > /sys/fs/cgroup/mygroup/cpu.max
echo 100 > /sys/fs/cgroup/mygroup/cpu.weight  # default 100, range 1-10000
```

---

## 15.9 Data Structure Relationships

```
Complete data structure map:

  task_struct
    ├── sched_entity se          ─── embedded in task_struct
    │     ├── vruntime
    │     ├── load (weight)
    │     ├── run_node           ─── links into cfs_rq.tasks_timeline (RB tree)
    │     ├── cfs_rq *           ─── points to the cfs_rq I'm on
    │     └── parent *           ─── group hierarchy
    ├── sched_rt_entity rt       ─── for RT scheduling
    │     └── run_list           ─── links into rt_rq.active.queue[]
    ├── sched_dl_entity dl       ─── for deadline scheduling
    │     └── rb_node            ─── links into dl_rq.root
    ├── sched_class *            ─── points to fair/rt/dl/idle_sched_class
    ├── policy                   ─── SCHED_NORMAL, SCHED_FIFO, etc.
    ├── prio / static_prio / normal_prio
    └── cpus_mask                ─── allowed CPUs

  struct rq (per CPU)
    ├── cfs_rq
    │     ├── tasks_timeline     ─── RB tree of sched_entity
    │     ├── min_vruntime
    │     └── nr_running
    ├── rt_rq
    │     ├── active.bitmap      ─── priority bitfield
    │     └── active.queue[100]  ─── per-priority linked lists
    ├── dl_rq
    │     └── root               ─── RB tree by deadline
    └── curr, idle, stop         ─── task pointers
```

---

## Kernel Source References

| Source File | Key Structures |
|-------------|---------------|
| `kernel/sched/sched.h` | struct rq, cfs_rq, rt_rq, dl_rq, sched_domain, task_group |
| `include/linux/sched.h` | struct task_struct, sched_entity, sched_rt_entity, sched_dl_entity |
| `kernel/sched/fair.c` | CFS operations on cfs_rq and sched_entity |
| `kernel/sched/rt.c` | RT operations on rt_rq and rt_prio_array |
| `kernel/sched/deadline.c` | DL operations on dl_rq |
| `kernel/sched/pelt.c` | PELT load tracking implementation |

---

## Interview Questions

**Q1: Why does each CPU have its own run queue instead of a global queue?**
A: Per-CPU run queues: 1) Eliminate global lock contention (critical for scalability). 2) Maintain cache locality — a task running on the same CPU reuses warm caches. 3) Enable independent scheduling decisions without synchronization. The cost is complexity in load balancing across CPUs, handled by scheduling domains.

**Q2: What is the difference between `load_avg` and `util_avg` in PELT?**
A: `load_avg` = CPU demand weighted by task priority (a high-priority waiting task has high load_avg). Used for **load balancing** — distributing weighted demand. `util_avg` = actual CPU utilization (0-1024 scale), independent of priority. Used for **frequency scaling** (how busy is this CPU?) and **Energy Aware Scheduling**. A sleeping task has declining util_avg but may retain load_avg if recently active.

**Q3: How does CFS bandwidth control work in cgroups?**
A: Each task_group has a `cfs_bandwidth` with quota (max runtime) and period (measurement window). Each per-CPU cfs_rq requests runtime from the global pool. When a cfs_rq exhausts its allocation, it's throttled (all its tasks dequeued). A timer replenishes the global pool each period, unthrottling cfs_rqs. Example: quota=50ms, period=100ms limits the group to 50% CPU across all cores.

---

*Next: [Chapter 16 — CPU Scheduling & Load Balancing](Chapter_16_CPU_Scheduling.md)*
