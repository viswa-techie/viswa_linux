# Chapter 13: Completely Fair Scheduler (CFS)

## Learning Goals
- Understand CFS design philosophy and goals
- Master virtual runtime (vruntime) calculation
- Learn red-black tree management
- Know CFS tunables and their effects

---

## 13.1 CFS Design Philosophy

CFS, introduced in Linux 2.6.23 by Ingo Molnár, models an **ideal multitasking CPU** where each task gets exactly `1/N` of CPU time simultaneously. Since real hardware can only run one task at a time, CFS approximates this ideal by tracking how much CPU time each task *should* have received versus how much it *actually* got.

```
Ideal "fair" CPU (4 tasks):
  Each gets 25% CPU at every instant — impossible on real hardware

CFS approximation:
  Track virtual runtime (vruntime) per task
  Always run the task with SMALLEST vruntime
  Tasks that got less CPU → lower vruntime → run next
  Result: over time, all tasks converge to equal vruntime
```

Key insight: CFS replaces fixed timeslices with **proportional fair sharing** using a continuous-time model.

---

## 13.2 Virtual Runtime (vruntime)

### Formula

```
                  actual_runtime × NICE_0_LOAD
vruntime_delta = ──────────────────────────────
                       task_weight

Where:
  NICE_0_LOAD = 1024 (weight at nice 0)
  task_weight = sched_prio_to_weight[nice + 20]
```

### Weight Table (excerpt)

```c
/* kernel/sched/core.c — actual kernel weight table */
const int sched_prio_to_weight[40] = {
 /* -20 */  88761, 71755, 56483, 46273, 36291,
 /* -15 */  29154, 23254, 18705, 14949, 11916,
 /* -10 */   9548,  7620,  6100,  4904,  3906,
 /*  -5 */   3121,  2501,  1991,  1586,  1277,
 /*   0 */   1024,   820,   655,   526,   423,
 /*   5 */    335,   272,   215,   172,   137,
 /*  10 */    110,    87,    70,    56,    45,
 /*  15 */     36,    29,    23,    18,    15,
};
/* Each nice level ≈ 1.25x the weight of the next */
```

### Example

```
Task A: nice 0  → weight 1024
Task B: nice 5  → weight 335

Both run for 10ms actual time:
  A's vruntime += 10ms × 1024/1024 = 10ms
  B's vruntime += 10ms × 1024/335  ≈ 30.6ms

A "uses up" vruntime slower → gets more CPU time
Ratio of CPU: A gets ~3x more than B
```

### Kernel Implementation

```c
/* kernel/sched/fair.c */
static void update_curr(struct cfs_rq *cfs_rq)
{
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_clock_task(rq_of(cfs_rq));
    u64 delta_exec;

    delta_exec = now - curr->exec_start;
    if (unlikely((s64)delta_exec <= 0))
        return;

    curr->exec_start = now;
    curr->sum_exec_runtime += delta_exec;

    /* weighted vruntime update */
    curr->vruntime += calc_delta_fair(delta_exec, curr);

    update_min_vruntime(cfs_rq);
}

static u64 calc_delta_fair(u64 delta, struct sched_entity *se)
{
    if (unlikely(se->load.weight != NICE_0_LOAD))
        delta = __calc_delta(delta, NICE_0_LOAD, &se->load);
    return delta;
}
```

---

## 13.3 Red-Black Tree (Timeline)

CFS organizes all runnable tasks in a **red-black tree** keyed by `vruntime`.

```
Red-Black Tree (ordered by vruntime):

              [vruntime=50]  (black)
              /             \
       [vruntime=30]     [vruntime=70]  (red nodes)
       /          \       /          \
  [vruntime=20] [40]   [60]      [vruntime=80]

  Leftmost node = smallest vruntime = NEXT TO RUN
  ↑
  Cached as: cfs_rq->rb_leftmost   (O(1) access)

Properties:
  - Insert: O(log n)
  - Remove: O(log n)
  - Pick next (leftmost): O(1) — cached
  - Self-balancing: max depth = 2×log₂(n)
```

### Key Data Structures

```c
/* include/linux/sched.h */
struct sched_entity {
    struct load_weight      load;           /* task weight */
    struct rb_node          run_node;       /* RB tree node */
    unsigned int            on_rq;          /* on runqueue? */
    u64                     exec_start;     /* last schedule timestamp */
    u64                     sum_exec_runtime;
    u64                     vruntime;       /* virtual runtime */
    u64                     prev_sum_exec_runtime;
    /* ... group scheduling fields ... */
};

/* kernel/sched/sched.h */
struct cfs_rq {
    struct load_weight      load;           /* total load */
    unsigned int            nr_running;     /* number runnable */
    u64                     min_vruntime;   /* floor for new tasks */
    struct rb_root_cached   tasks_timeline; /* RB tree */
    struct sched_entity     *curr;          /* currently running */
    /* PELT load tracking */
    struct sched_avg        avg;
    /* ... */
};
```

### Pick Next Task

```c
/* kernel/sched/fair.c */
static struct sched_entity *
pick_next_entity(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
    struct sched_entity *left = __pick_first_entity(cfs_rq);

    /* Leftmost entity has smallest vruntime — most deserving */
    if (!left || (curr && entity_before(curr, left)))
        left = curr;

    /* Skip-buddy, last-buddy optimizations for cache warmth */
    struct sched_entity *se = left;
    /* ... buddy logic ... */
    return se;
}

static struct sched_entity *__pick_first_entity(struct cfs_rq *cfs_rq)
{
    struct rb_node *left = rb_first_cached(&cfs_rq->tasks_timeline);
    if (!left)
        return NULL;
    return rb_entry(left, struct sched_entity, run_node);
}
```

---

## 13.4 CFS Timeslice Calculation

CFS has no fixed timeslice. Instead, it calculates a **scheduling period** and divides it proportionally:

```
sched_latency = 6ms (default, for up to 8 tasks)
sched_min_granularity = 0.75ms

If nr_running > sched_nr_latency:
  period = nr_running × sched_min_granularity
Else:
  period = sched_latency

Each task's ideal runtime = period × (task_weight / total_weight)
```

### Example

```
3 tasks with nice 0, 0, 5 (weights: 1024, 1024, 335):
  total_weight = 1024 + 1024 + 335 = 2383
  period = 6ms (< 8 tasks)

  Task A (nice 0): 6ms × 1024/2383 = 2.58ms
  Task B (nice 0): 6ms × 1024/2383 = 2.58ms
  Task C (nice 5): 6ms × 335/2383  = 0.84ms
                                      ──────
                                      6.00ms ✓
```

### Tunables (/proc/sys/kernel/)

| Tunable | Default | Effect |
|---------|---------|--------|
| `sched_latency_ns` | 6000000 (6ms) | Target scheduling period |
| `sched_min_granularity_ns` | 750000 (0.75ms) | Min time per task |
| `sched_wakeup_granularity_ns` | 1000000 (1ms) | Preemption threshold |
| `sched_nr_latency` | 8 | sched_latency / sched_min_granularity |
| `sched_child_runs_first` | 0 | Child preempts parent after fork |
| `sched_tunable_scaling` | 1 | 0=none, 1=log, 2=linear scaling |

---

## 13.5 New Task Placement

When a new task (fork/wake) enters CFS, where does its vruntime start?

```c
static void place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se,
                          int initial)
{
    u64 vruntime = cfs_rq->min_vruntime;

    if (initial) {
        /* New fork: penalize slightly to prevent fork bombs */
        vruntime += sched_vslice(cfs_rq, se);
    }

    /* Don't go backwards — prevent unfair CPU stealing */
    se->vruntime = max_vruntime(se->vruntime, vruntime);
}
```

```
New task placement:

  min_vruntime ────┐
                   ↓
  |---existing-tasks---|
  [20][30][40][50][60][70]
                    ↑
  New task placed near min_vruntime + penalty
  (not at 0, which would steal all CPU)
```

---

## 13.6 Sleeper Fairness

When a task wakes from sleep, its vruntime may be far behind running tasks. CFS limits the "credit":

```c
/* On wakeup: */
vruntime = max(vruntime, min_vruntime - sched_latency)
```

```
Before wake:
  Running tasks: vruntime 100, 105, 110
  min_vruntime = 100
  Sleeping task: vruntime = 20 (slept long)

After adjustment:
  Sleeping task: vruntime = max(20, 100 - 6) = 94
  Gets a small "credit" (runs soon) but NOT unlimited
```

---

## 13.7 EEVDF — The Future (Linux 6.6+)

Linux 6.6 replaced CFS's pick-next logic with **EEVDF** (Earliest Eligible Virtual Deadline First):

```
CFS problem: all tasks with same vruntime are treated equally,
  but some tasks have shorter "requests" (time slices) than others.

EEVDF adds:
  - Virtual deadline = vruntime + (request_size / weight)
  - Pick task with earliest virtual deadline among eligible tasks
  - "Eligible" = vruntime ≤ fair_share (not overserved)

Result: Short-running (interactive) tasks get lower latency
  while maintaining overall fairness.
```

```
EEVDF vs CFS pick-next:

CFS:   leftmost in RB tree (smallest vruntime)
EEVDF: eligible task with smallest virtual deadline

Effect: Interactive tasks (short bursts) get picked earlier
```

---

## 13.8 CFS Flow Diagram

```
Task becomes runnable (fork/wake)
            │
            ▼
    place_entity()
    Set vruntime near min_vruntime
            │
            ▼
    enqueue_entity()
    Insert into RB tree by vruntime
            │
            ▼
    check_preempt_curr()
    If new task's vruntime < curr - granularity
      → Set TIF_NEED_RESCHED
            │
            ▼
    ── schedule() called ──
            │
            ▼
    pick_next_entity()
    Get leftmost RB node (smallest vruntime)
            │
            ▼
    set_next_entity()
    Remove from tree, set as curr
            │
            ▼
    ── Task runs ──
            │
            ▼
    scheduler_tick() / schedule()
            │
            ▼
    update_curr()
    vruntime += calc_delta_fair(delta_exec)
            │
            ▼
    If vruntime > leftmost + granularity
      → Set TIF_NEED_RESCHED → loop back to schedule()
```

---

## Kernel Source References

| Source File | Key Functions |
|-------------|---------------|
| `kernel/sched/fair.c` | update_curr(), pick_next_entity(), enqueue_entity(), dequeue_entity(), place_entity() |
| `kernel/sched/sched.h` | struct cfs_rq, struct sched_entity |
| `include/linux/sched.h` | struct sched_entity definition |
| `kernel/sched/core.c` | sched_prio_to_weight[] table |

---

## Interview Questions

**Q1: Why does CFS use a red-black tree instead of a simple sorted list?**
A: A red-black tree provides O(log n) insert/remove while keeping O(1) access to the minimum via caching the leftmost node. A sorted list would be O(n) for insertion. A heap gives O(1) min but O(log n) remove of arbitrary elements. The RB tree balances all operations well and supports efficient in-order traversal.

**Q2: How does CFS prevent a nice -20 task from starving nice +19 tasks?**
A: CFS guarantees every runnable task gets CPU within the scheduling period. A nice -20 task has weight 88761 vs nice +19's weight 15 — a ratio of ~5917:1. With sched_latency=6ms and two such tasks, nice -20 gets 5.999ms and nice +19 gets 0.001ms per period. So it *nearly* starves but always gets at least `sched_min_granularity` (0.75ms). In practice, such extreme nice differences are rare.

**Q3: What problem does EEVDF solve that CFS doesn't?**
A: CFS picks purely by smallest vruntime — it doesn't distinguish between a task needing 0.1ms and one needing 100ms. EEVDF adds virtual deadlines so tasks requesting short slices get lower latency, improving interactive performance while maintaining overall fairness. It also eliminates the need for heuristic-based wakeup preemption tuning.

---

*Next: [Chapter 14 — Real-Time Scheduling](Chapter_14_Real_Time_Scheduling.md)*
