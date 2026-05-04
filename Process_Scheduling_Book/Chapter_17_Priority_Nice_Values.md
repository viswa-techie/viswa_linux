# Chapter 17: Priority and Nice Values

## Learning Goals
- Understand Linux's priority system (static, dynamic, normal, effective)
- Master nice values and their mapping to weights
- Learn priority inheritance for RT tasks
- Know how to view and modify priorities

---

## 17.1 Priority Landscape

```
Linux Priority Map:

  Kernel prio    User-visible         Policy              Scheduling class
  ─────────────────────────────────────────────────────────────────────
  0              (internal)           SCHED_STOP          stop_sched_class
  1-99           RT priority 99→1    SCHED_FIFO/RR       rt_sched_class
  0 (DL)         N/A                 SCHED_DEADLINE       dl_sched_class
  100-139        nice -20 to +19     SCHED_NORMAL/BATCH   fair_sched_class
  140            (internal)           SCHED_IDLE           idle_sched_class

  LOWER kernel prio number = HIGHER scheduling priority

  Confusingly:
    RT "priority" 99 → kernel prio 0 (highest)
    RT "priority" 1  → kernel prio 98 (lowest RT)
    nice -20         → kernel prio 100
    nice  0          → kernel prio 120
    nice +19         → kernel prio 139
```

### Conversion Formulas

```c
/* include/linux/sched/prio.h */
#define MAX_RT_PRIO     100
#define MAX_PRIO        140

#define MAX_USER_RT_PRIO    99
#define DEFAULT_PRIO        (MAX_RT_PRIO + 20)  /* 120 = nice 0 */

/* RT: user_prio → kernel_prio */
#define MAX_RT_PRIO         100
static inline int rt_prio(int prio)      { return prio < MAX_RT_PRIO; }
/* kernel_prio = 99 - rt_priority (user) */

/* Nice: nice → kernel_prio */
#define NICE_TO_PRIO(nice)  ((nice) + DEFAULT_PRIO)  /* nice + 120 */
#define PRIO_TO_NICE(prio)  ((prio) - DEFAULT_PRIO)
#define TASK_NICE(p)        PRIO_TO_NICE((p)->static_prio)
```

---

## 17.2 Priority Fields in task_struct

```c
struct task_struct {
    int     prio;           /* effective (scheduling) priority */
    int     static_prio;    /* set by nice/setpriority (100-139) */
    int     normal_prio;    /* based on static_prio OR RT priority */
    unsigned int rt_priority; /* RT priority (1-99, user-space value) */
    unsigned int policy;     /* SCHED_NORMAL, SCHED_FIFO, etc. */
};
```

### How They Relate

```
For NORMAL tasks (SCHED_NORMAL, SCHED_BATCH):
  static_prio  = 120 + nice       (set by nice()/setpriority())
  normal_prio  = static_prio      (no RT adjustment)
  prio         = normal_prio      (unless temporarily boosted by PI)

For RT tasks (SCHED_FIFO, SCHED_RR):
  rt_priority  = 1-99             (set by sched_setscheduler())
  static_prio  = 120              (nice meaningless for RT)
  normal_prio  = 99 - rt_priority (maps to kernel 0-98)
  prio         = normal_prio      (unless PI-boosted)

Priority Inheritance:
  If task holds rt_mutex and higher-prio task waits:
    prio = waiter's prio    (temporarily boosted)
    After unlock: prio = normal_prio (restored)

  p->prio MAY differ from p->normal_prio during PI!
```

```c
/* kernel/sched/core.c */
static int effective_prio(struct task_struct *p)
{
    p->normal_prio = normal_prio(p);
    if (!rt_prio(p->prio))
        return p->normal_prio;  /* Normal task: effective = normal */
    /* PI-boosted: keep elevated prio */
    return p->prio;
}

static int normal_prio(struct task_struct *p)
{
    if (task_has_dl_policy(p))
        return MAX_DL_PRIO - 1;    /* -1 (internal) */
    if (task_has_rt_policy(p))
        return MAX_RT_PRIO - 1 - p->rt_priority;  /* 0-98 */
    return __normal_prio(p);        /* static_prio (100-139) */
}
```

---

## 17.3 Nice Values Deep Dive

### Setting Nice Values

```c
/* User space API */
#include <unistd.h>
#include <sys/resource.h>

/* Set nice value (relative change) */
nice(10);  /* Lower priority by 10 */

/* Set nice value (absolute) */
setpriority(PRIO_PROCESS, pid, 5);  /* nice = 5 */

/* Get nice value */
int nice_val = getpriority(PRIO_PROCESS, pid);
```

```bash
# Command line
nice -n 10 ./my_app           # Start with nice 10
renice -n 5 -p <pid>          # Change running process
renice -n -5 -u username      # Change all processes of user

# Only root can set negative nice (increase priority)
# CAP_SYS_NICE capability also allows it
```

### Nice to Weight Mapping

```
Weight relationship: each nice level = ~10% CPU difference

  nice -20: weight 88761  ─── 5917x more than nice +19
  nice -10: weight  9548  ───  637x more
  nice  -5: weight  3121  ───  208x more
  nice   0: weight  1024  ───   68x more (reference)
  nice  +5: weight   335  ───   22x more
  nice +10: weight   110  ───    7x more
  nice +19: weight    15  ───    1x (minimum)

CPU allocation with 2 tasks:
  Both nice 0:   50% / 50%
  nice 0 vs +5:  75% / 25%  (1024:335 ≈ 3:1)
  nice 0 vs +10: 90% /  10%  (1024:110 ≈ 9:1)
  nice -5 vs +5: 90% /  10%  (3121:335 ≈ 9.3:1)
```

### Impact on CFS vruntime

```
vruntime accumulation rate:

  nice 0 task:  runs 10ms → vruntime += 10ms
  nice 5 task:  runs 10ms → vruntime += 10 × 1024/335 ≈ 30.6ms
  nice -5 task: runs 10ms → vruntime += 10 × 1024/3121 ≈ 3.3ms

  Lower vruntime → runs more (leftmost in RB tree)
  Higher nice → vruntime grows faster → runs less
```

---

## 17.4 Priority Inheritance in Detail

```
Priority Inversion Scenario:

  Time →  0    5    10   15   20   25   30
  
  Task H (prio 10): ──LOCK──────BLOCKED─────────────RUN──
  Task M (prio 50): ─────────────RUN──RUN──RUN──────────
  Task L (prio 90): RUN──RUN──SLEEP─────────────────────
                     ↑    ↑
                  L holds  H tries to lock
                  mutex    → blocked!
                           M runs because L can't → INVERSION

With Priority Inheritance:

  Time →  0    5    10   15   20   25
  
  Task H (prio 10): ──LOCK──────BLOCKED──────RUN──
  Task M (prio 50): ─────────────────────────────RUN──
  Task L (prio 90→10): RUN──RUN──BOOSTED──UNLOCK──(back to 90)
                         ↑         ↑
                      L holds    H waits → L gets H's priority
                      mutex      L runs at prio 10, finishes fast
```

### rt_mutex Implementation

```c
/* kernel/locking/rtmutex.c */
static int rt_mutex_adjust_prio_chain(struct task_struct *task,
                                       int deadlock_detect,
                                       struct rt_mutex *orig_lock,
                                       struct rt_mutex_waiter *waiter,
                                       struct task_struct *top_task)
{
    /* Walk the PI chain:
     * task → lock it's waiting for → owner → lock that owner waits for → ...
     * Propagate priority boost through the entire chain
     *
     * Also detects deadlocks (circular chains)
     */
}

/* When task blocks on rt_mutex held by lower-priority owner:
 * 1. Create waiter entry with task's priority
 * 2. Insert into lock's waiters RB tree (ordered by priority)
 * 3. If waiter is top priority → boost owner's prio
 * 4. Propagate boost up the chain if needed
 */
```

---

## 17.5 Autogroup (Per-TTY Grouping)

Linux automatically groups tasks by terminal session for better desktop responsiveness:

```
Without autogroup:
  Terminal 1: make -j8 (8 compile tasks)
  Terminal 2: interactive shell
  
  CFS distributes: 8 compile + 1 shell = 9 tasks
  Shell gets 1/9 = 11% CPU — feels sluggish!

With autogroup (CONFIG_SCHED_AUTOGROUP=y):
  Group /autogroup-42 (Terminal 1): 8 compile tasks
  Group /autogroup-43 (Terminal 2): 1 shell task
  
  CFS distributes: 2 groups, each gets ~50%
  Shell gets 50% of CPU — responsive!
```

```bash
# View autogroups
cat /proc/<pid>/autogroup
# /autogroup-42 nice 0

# Adjust autogroup nice (per-group)
echo 10 > /proc/<pid>/autogroup   # Lower group priority

# Disable autogroup
echo 0 > /proc/sys/kernel/sched_autogroup_enabled
```

---

## 17.6 Priority Viewing Tools

```bash
# ps with priority info
ps -eo pid,ni,pri,rtprio,cls,comm
  PID  NI PRI RTPRIO CLS COMMAND
    1   0  19      - TS  systemd
   42  -5  24      - TS  special_task
  100   -  41     99 FF  irq/18-i2c
  200   0  19      - TS  bash

# Columns:
#   NI:     nice value (-20 to +19)
#   PRI:    display priority (higher = more priority)
#   RTPRIO: RT priority (1-99, '-' for non-RT)
#   CLS:    TS=SCHED_NORMAL, FF=SCHED_FIFO, RR=SCHED_RR

# top: press 'f' to add NI and PR columns

# chrt: view/set RT scheduling
chrt -p <pid>
# pid 100's current scheduling policy: SCHED_FIFO
# pid 100's current scheduling priority: 99

# /proc/<pid>/sched — detailed scheduler stats
cat /proc/<pid>/sched
# se.vruntime   : 123456789.012345
# policy        : 0 (SCHED_NORMAL)
# prio          : 120
```

---

## Interview Questions

**Q1: What's the difference between `prio`, `static_prio`, `normal_prio`, and `rt_priority`?**
A: `static_prio` = set by user via nice (100-139). `rt_priority` = set by user for RT tasks (1-99). `normal_prio` = computed from static_prio (normal) or rt_priority (RT) — the "base" effective priority. `prio` = the actual scheduling priority, usually equals normal_prio but can be temporarily boosted by priority inheritance. Only `prio` is checked during scheduling decisions.

**Q2: If I set a task to nice -20, can it starve nice +19 tasks?**
A: Nearly, but not completely. Weight ratio is 88761:15 (~5917:1). With CFS sched_min_granularity, the +19 task always gets at least 0.75ms per scheduling period. In practice, the nice -20 task gets 99.98% of CPU. However, RT tasks truly preempt all normal tasks regardless of nice.

**Q3: How does priority inheritance prevent the Mars Pathfinder problem?**
A: Mars Pathfinder (1997) suffered from priority inversion where a meteorological task (medium priority) preempted a bus management task (low priority) that held a mutex needed by the critical task (high priority). The fix was enabling priority inheritance in the VxWorks RTOS mutex, which is exactly what Linux's `rt_mutex` does — temporarily boosting the lock owner's priority to the highest waiter's level, preventing medium-priority tasks from causing unbounded delay.

---

*Next: [Chapter 18 — Preemption](Chapter_18_Preemption.md)*
