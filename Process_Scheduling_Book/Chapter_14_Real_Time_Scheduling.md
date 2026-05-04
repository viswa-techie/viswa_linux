# Chapter 14: Real-Time Scheduling

## Learning Goals
- Understand Linux RT scheduling policies (SCHED_FIFO, SCHED_RR, SCHED_DEADLINE)
- Learn RT priority system and its interaction with normal scheduling
- Master deadline scheduling (EDF/CBS) introduced in Linux 3.14
- Understand PREEMPT_RT for hard real-time requirements

---

## 14.1 Real-Time Scheduling Overview

```
Linux Scheduling Priority Spectrum:

  Priority 0-99:   RT tasks (SCHED_FIFO, SCHED_RR, SCHED_DEADLINE)
  Priority 100-139: Normal tasks (SCHED_NORMAL/OTHER, SCHED_BATCH, SCHED_IDLE)

  ┌────────────────────────────────────────────────┐
  │  STOP class (internal, migration threads)      │  Highest
  ├────────────────────────────────────────────────┤
  │  DEADLINE class (SCHED_DEADLINE)               │
  ├────────────────────────────────────────────────┤
  │  RT class (SCHED_FIFO, SCHED_RR)              │
  │    Priority 99 (highest RT)                    │
  │    Priority 98                                 │
  │    ...                                         │
  │    Priority 1 (lowest RT)                      │
  ├────────────────────────────────────────────────┤
  │  FAIR class (SCHED_NORMAL, SCHED_BATCH)        │
  ├────────────────────────────────────────────────┤
  │  IDLE class (SCHED_IDLE)                       │  Lowest
  └────────────────────────────────────────────────┘

  ANY runnable RT task preempts ALL normal tasks.
```

---

## 14.2 SCHED_FIFO

**First-In, First-Out** real-time policy:

```
Rules:
  1. Highest priority RT task runs immediately
  2. Runs until it: blocks, yields, or is preempted by higher priority
  3. If same priority: FIFO order — first task runs until done
  4. NO timeslice — task can run forever

Use case: Audio processing, control loops, interrupt handling threads
```

### Setting SCHED_FIFO

```c
#include <sched.h>
#include <stdio.h>

int main(void)
{
    struct sched_param param;
    param.sched_priority = 50;  /* 1-99, higher = more important */

    if (sched_setscheduler(0, SCHED_FIFO, &param) < 0) {
        perror("sched_setscheduler");
        return 1;
    }

    printf("Running as SCHED_FIFO priority %d\n", param.sched_priority);

    /* Critical real-time work */
    while (1) {
        do_realtime_work();
        /* Must voluntarily yield or block — no timeslice! */
    }
}
```

### From Command Line

```bash
# Run a command with SCHED_FIFO priority 80
chrt -f 80 ./my_rt_app

# Check scheduling policy of a process
chrt -p <pid>

# Change policy of running process
chrt -f -p 80 <pid>
```

---

## 14.3 SCHED_RR

**Round Robin** real-time policy — SCHED_FIFO with timeslice:

```
Rules:
  1. Same as SCHED_FIFO EXCEPT:
  2. Tasks at SAME priority get time-sliced (round-robin)
  3. Default timeslice: 100ms
  4. Higher priority still preempts immediately

  SCHED_RR vs SCHED_FIFO:
    Within same priority level:
      FIFO: first task runs until it blocks/yields
      RR:   tasks rotate with timeslice

  Between different priority levels:
    Both behave identically — higher preempts lower
```

### Timeslice Control

```bash
# View default RR timeslice (milliseconds)
cat /proc/sys/kernel/sched_rr_timeslice_ms
100

# Change (requires root)
echo 50 > /proc/sys/kernel/sched_rr_timeslice_ms
```

### Query Timeslice Programmatically

```c
struct timespec ts;
sched_rr_get_interval(pid, &ts);
printf("RR timeslice: %ld.%09ld sec\n", ts.tv_sec, ts.tv_nsec);
```

---

## 14.4 SCHED_DEADLINE — Earliest Deadline First

Added in Linux 3.14. Uses **CBS (Constant Bandwidth Server)** + **EDF (Earliest Deadline First)**.

```
Task parameters:
  runtime  — worst-case execution time per period
  deadline — relative deadline for each job
  period   — how often the task runs

Guarantee: task gets 'runtime' CPU within each 'period'

Example (audio processing):
  runtime  = 2ms   (needs 2ms of CPU)
  deadline = 10ms  (must finish within 10ms)
  period   = 10ms  (runs every 10ms)
  
  Bandwidth = runtime/period = 2/10 = 20% CPU

  |--2ms work--|------8ms slack------|--2ms work--|...
  0           2                     10           12    20
  |<────── deadline (10ms) ────────>|
```

### Setting SCHED_DEADLINE

```c
#define _GNU_SOURCE
#include <sched.h>
#include <linux/sched.h>
#include <sys/syscall.h>

struct sched_attr {
    uint32_t size;
    uint32_t sched_policy;
    uint64_t sched_flags;
    int32_t  sched_nice;
    uint32_t sched_priority;
    uint64_t sched_runtime;   /* nanoseconds */
    uint64_t sched_deadline;  /* nanoseconds */
    uint64_t sched_period;    /* nanoseconds */
};

int main(void)
{
    struct sched_attr attr = {
        .size = sizeof(attr),
        .sched_policy   = SCHED_DEADLINE,
        .sched_runtime  =  2000000,  /*  2ms */
        .sched_deadline = 10000000,  /* 10ms */
        .sched_period   = 10000000,  /* 10ms */
    };

    if (syscall(SYS_sched_setattr, 0, &attr, 0) < 0) {
        perror("sched_setattr");
        return 1;
    }

    while (1) {
        do_periodic_work();
        sched_yield();  /* Signal completion of this period's work */
    }
}
```

### Admission Control

```
SCHED_DEADLINE enforces admission control:

  Sum of all deadline tasks' bandwidths ≤ total CPU capacity

  For M CPUs:
    Σ (runtime_i / period_i) ≤ M × rt_bandwidth

  Default rt_bandwidth: 95% (950000/1000000)
  
  /proc/sys/kernel/sched_rt_runtime_us  = 950000  (950ms)
  /proc/sys/kernel/sched_rt_period_us   = 1000000 (1000ms)

  If adding a new deadline task would exceed capacity → EBUSY
```

---

## 14.5 RT Throttling Safety Net

Without limits, a buggy RT task could lock up the system. Linux provides **RT throttling**:

```
Default: RT tasks get 950ms out of every 1000ms
  → 50ms reserved for normal tasks (5%)

sched_rt_runtime_us = 950000   (how much RT gets)
sched_rt_period_us  = 1000000  (measurement window)

Timeline:
  |==================RT=================|==NORMAL==|
  0                                   950ms      1000ms

To disable (dangerous!):
  echo -1 > /proc/sys/kernel/sched_rt_runtime_us
```

---

## 14.6 RT Scheduling Data Structures

```c
/* kernel/sched/sched.h */
struct rt_rq {
    struct rt_prio_array    active;
    unsigned int            rt_nr_running;      /* total RT tasks */
    unsigned int            rr_nr_running;      /* SCHED_RR tasks */
    int                     highest_prio;       /* cache */
    int                     overloaded;         /* for push/pull */
    /* RT bandwidth enforcement */
    struct rt_bandwidth     rt_runtime;
};

/* Priority array — O(1) priority lookup */
struct rt_prio_array {
    DECLARE_BITMAP(bitmap, MAX_RT_PRIO + 1);  /* 100 bits */
    struct list_head queue[MAX_RT_PRIO];       /* 100 lists */
};
```

```
RT Priority Array:

bitmap: [0][1][1][0]...[1][0]
         ↑      ↑        ↑
         |      |        Priority 50 has tasks
         |      Priority 2 has tasks
         Priority 1 has tasks

queue[1]: Task_A → Task_B → (FIFO order)
queue[2]: Task_C → Task_D
queue[50]: Task_E

Pick next = find_first_bit(bitmap) → O(1)
           Get first task from that queue → O(1)
```

---

## 14.7 RT Load Balancing (Push/Pull)

```
Multi-CPU RT scheduling needs special balancing.
Normal load balancing is too slow for RT guarantees.

PUSH Migration:
  When RT task wakes on a CPU where higher-priority RT runs:
  → Push lower-priority RT to another CPU

PULL Migration:  
  When a CPU's RT task finishes:
  → Pull highest-priority waiting RT from another CPU

  CPU 0 [RT prio 90]     CPU 1 [RT prio 50]
         │                       │
    New RT prio 60 wakes         │
    Can't preempt 90!            │
         │                       │
         └── PUSH prio 60 ─────→│ Preempts 50, runs 60
```

---

## 14.8 Priority Inversion Problem

```
Classic priority inversion scenario:

  Task H (high priority)  ─── blocked on lock held by L
  Task M (medium priority) ─── running (preempts L!)
  Task L (low priority)   ─── holds lock, but can't run

  Result: H waits for M (even though H > M) — priority inversion!

Solution: Priority Inheritance (PI)
  When H blocks on L's lock:
    L temporarily inherits H's priority
    L preempts M, releases lock
    H runs immediately

Linux implementation: rt_mutex (PI-aware mutex)
```

```c
/* Using rt_mutex for priority inheritance */
#include <linux/rtmutex.h>

static DEFINE_RT_MUTEX(my_rt_lock);

void critical_section(void)
{
    rt_mutex_lock(&my_rt_lock);
    /* Protected work — lock owner inherits waiters' priority */
    rt_mutex_unlock(&my_rt_lock);
}
```

---

## 14.9 PREEMPT_RT Patch

For hard real-time requirements, the **PREEMPT_RT** patchset (being mainlined) makes Linux fully preemptible:

```
Preemption Models:
  PREEMPT_NONE    — Only at explicit schedule points
  PREEMPT_VOLUNTARY — + explicit preemption points
  PREEMPT          — Preempt anywhere except spinlock-held
  PREEMPT_RT       — Convert spinlocks to rt_mutex (preemptible!)

PREEMPT_RT changes:
  ✓ spinlock_t → rt_mutex (sleepable, PI-aware)
  ✓ Threaded interrupts (all IRQs become kernel threads)
  ✓ Preemptible critical sections
  ✓ High-resolution timers throughout
  
Latency comparison:
  Standard kernel: worst-case ~1-10ms
  PREEMPT_RT:      worst-case ~10-100µs
```

---

## SCHED_FIFO vs SCHED_RR vs SCHED_DEADLINE

| Feature | SCHED_FIFO | SCHED_RR | SCHED_DEADLINE |
|---------|-----------|---------|---------------|
| Timeslice | None | 100ms default | Runtime budget per period |
| Same-priority | FIFO order | Round-robin | N/A (each task unique) |
| Priority range | 1-99 | 1-99 | N/A (deadline-based) |
| Preempts normal | Always | Always | Always |
| Admission control | No | No | **Yes** |
| Starvation of normal | Possible (throttled) | Possible (throttled) | Bandwidth limited |
| Best for | Single critical task | Multiple same-priority RT | Periodic tasks |

---

## Interview Questions

**Q1: Can SCHED_DEADLINE tasks starve SCHED_FIFO tasks?**
A: Yes! SCHED_DEADLINE class has higher priority than SCHED_FIFO/RR. A deadline task will preempt any RT task. However, deadline tasks are bandwidth-limited (admission control), so they cannot consume more CPU than their declared runtime/period ratio.

**Q2: What happens if a SCHED_FIFO task enters an infinite loop?**
A: Without RT throttling, it would lock out all lower-priority tasks on that CPU. With default throttling (enabled), it gets 950ms/1000ms, leaving 50ms for normal tasks. The system remains partially responsive. With PREEMPT_RT and threaded IRQs, even interrupts can be preempted, so a misbehaving RT task at priority 99 can truly lock a CPU unless throttled.

**Q3: When would you choose SCHED_DEADLINE over SCHED_FIFO?**
A: SCHED_DEADLINE is ideal for **periodic tasks** where you know the execution time and period (audio/video processing, control loops, sensor sampling). It provides guaranteed bandwidth and admission control. SCHED_FIFO is better for event-driven tasks where periodicity is unknown or when simplicity is needed.

---

*Next: [Chapter 15 — Scheduler Data Structures](Chapter_15_Scheduler_Data_Structures.md)*
