# Chapter 23: Scheduling Policies

## Learning Goals
- Understand all Linux scheduling policies and their behavior
- Learn how to set and query scheduling policies
- Know the relationship between policies and scheduling classes
- Master policy selection for different workload types

---

## 23.1 Complete Policy Overview

```
┌─────────────────┬────────────┬───────────────┬───────────────────────┐
│ Policy           │ Class      │ Priority range│ Behavior              │
├─────────────────┼────────────┼───────────────┼───────────────────────┤
│ SCHED_DEADLINE   │ dl_class   │ N/A (params)  │ EDF + CBS bandwidth   │
│ SCHED_FIFO       │ rt_class   │ 1-99          │ Priority FIFO, no     │
│                  │            │               │ timeslice             │
│ SCHED_RR         │ rt_class   │ 1-99          │ Round-robin at same   │
│                  │            │               │ priority              │
│ SCHED_NORMAL     │ fair_class │ nice -20..+19 │ CFS proportional fair │
│ (SCHED_OTHER)    │            │               │ sharing               │
│ SCHED_BATCH      │ fair_class │ nice -20..+19 │ CFS, no wakeup        │
│                  │            │               │ preemption bonus      │
│ SCHED_IDLE       │ fair_class │ nice (ignored)│ CFS, very low weight  │
│                  │            │  *(see note)* │ runs only when idle   │
└─────────────────┴────────────┴───────────────┴───────────────────────┘

* SCHED_IDLE uses fair_class but with extremely low weight,
  effectively runs only when no other normal/batch tasks want CPU
```

---

## 23.2 SCHED_NORMAL (SCHED_OTHER)

```
Default policy for all regular processes.

Managed by CFS (Completely Fair Scheduler):
  - Proportional CPU sharing based on nice/weight
  - No fixed timeslice — dynamic based on load
  - Interactive tasks get wakeup preemption bonus
  - vruntime-based fairness

Characteristics:
  - Interactive tasks (GUI, terminal): responsive due to sleep bonus
  - CPU-bound tasks: fair sharing based on weight
  - Default nice = 0, weight = 1024

Best for:
  Most user-space applications, desktops, general servers
```

```c
/* Set SCHED_NORMAL (usually not needed — it's the default) */
struct sched_param param = { .sched_priority = 0 };
sched_setscheduler(pid, SCHED_OTHER, &param);
/* Note: SCHED_OTHER = SCHED_NORMAL (same constant) */
```

---

## 23.3 SCHED_BATCH

```
Same CFS algorithm as SCHED_NORMAL, with one key difference:
  NO wakeup preemption bonus

  SCHED_NORMAL: task wakes → may preempt current if vruntime is lower
                (sched_wakeup_granularity check)
  SCHED_BATCH:  task wakes → never causes immediate preemption
                Only runs when scheduler naturally picks it

Use case:
  CPU-intensive batch jobs (compilation, encoding, scientific compute)
  that don't need low latency and shouldn't disturb interactive tasks

Effect:
  Throughput slightly better (fewer context switches)
  Latency slightly worse (no preemption on wake)
```

```c
struct sched_param param = { .sched_priority = 0 };
sched_setscheduler(pid, SCHED_BATCH, &param);

/* Or from command line: */
/* chrt -b 0 ./my_batch_job */
```

---

## 23.4 SCHED_IDLE

```
Lowest priority normal scheduling policy.

Behavior:
  - Runs under CFS with extremely low weight
  - Only gets CPU when no SCHED_NORMAL or SCHED_BATCH tasks want it
  - NOT the same as idle CPU (CPU idle loop)
  - Still gets SOME CPU via min_granularity guarantee

Use case:
  Background maintenance: updatedb, log rotation, backup
  Tasks that should NEVER interfere with other work

Comparison with nice +19:
  nice +19 (SCHED_NORMAL): weight=15, still competes fairly
  SCHED_IDLE:              much lower effective weight
  
  With 1 SCHED_NORMAL nice 0 and 1 SCHED_IDLE:
    Normal: ~99% CPU
    Idle:   ~1% CPU
```

```c
struct sched_param param = { .sched_priority = 0 };
sched_setscheduler(pid, SCHED_IDLE, &param);

/* chrt -i 0 ./my_idle_job */
```

---

## 23.5 SCHED_FIFO

```
Real-time FIFO policy (covered in Chapter 14, summarized here):

  - Highest RT priority task runs indefinitely
  - No timeslice — runs until blocks, yields, or higher priority preempts
  - Same priority: strict FIFO (first queued runs first)
  - ALWAYS preempts SCHED_NORMAL/BATCH/IDLE
  
  Priority: 1 (lowest RT) to 99 (highest RT)
  
  Required privilege: root or CAP_SYS_NICE (or RLIMIT_RTPRIO)
```

---

## 23.6 SCHED_RR

```
Real-time Round-Robin:

  - Same as SCHED_FIFO except:
    At same priority level, tasks are time-sliced (round-robin)
  - Default timeslice: 100ms (/proc/sys/kernel/sched_rr_timeslice_ms)
  - Higher priority still preempts immediately
  
  Use when: Multiple RT tasks at same priority need fair sharing
```

---

## 23.7 SCHED_DEADLINE

```
Deadline scheduling (covered in Chapter 14, summarized here):

  Parameters: runtime, deadline, period (nanoseconds)
  Algorithm: EDF (Earliest Deadline First) + CBS (bandwidth server)
  
  Admission control: sum of runtime/period ≤ rt_bandwidth × nr_cpus
  Higher priority than SCHED_FIFO/RR

  Use when: Periodic tasks with known execution time and deadlines
  (audio, video, control loops, sensor sampling)
```

---

## 23.8 Policy Setting API

```c
#include <sched.h>

/* Get current policy */
int policy = sched_getscheduler(pid);

/* Set policy with sched_setscheduler */
struct sched_param param;
param.sched_priority = 50;  /* For RT; 0 for normal */
sched_setscheduler(pid, SCHED_FIFO, &param);

/* For SCHED_DEADLINE: must use sched_setattr */
struct sched_attr attr = {
    .size = sizeof(attr),
    .sched_policy = SCHED_DEADLINE,
    .sched_runtime  =  1000000,  /* 1ms */
    .sched_deadline = 10000000,  /* 10ms */
    .sched_period   = 10000000,  /* 10ms */
};
syscall(SYS_sched_setattr, 0, &attr, 0);

/* Query policy */
struct sched_attr attr;
syscall(SYS_sched_getattr, pid, &attr, sizeof(attr), 0);
```

### Command Line Tools

```bash
# View scheduling info
chrt -p <pid>
# pid 1234's current scheduling policy: SCHED_OTHER
# pid 1234's current scheduling priority: 0

# Set policies
chrt -f 80 ./app          # SCHED_FIFO, priority 80
chrt -r 50 ./app          # SCHED_RR, priority 50
chrt -b 0  ./app          # SCHED_BATCH
chrt -i 0  ./app          # SCHED_IDLE
chrt -o 0  ./app          # SCHED_OTHER (normal)
chrt -d --sched-runtime 1000000 --sched-deadline 10000000 \
        --sched-period 10000000 0 ./app  # SCHED_DEADLINE

# Change running process
chrt -f -p 80 <pid>
```

---

## 23.9 Policy Decision Guide

```
Workload type → Recommended policy:

  Interactive desktop (GUI, editors)
    → SCHED_NORMAL, nice 0 (default)
    
  Server (web, database)
    → SCHED_NORMAL, nice 0 to -5 for critical threads
    
  Batch processing (compile, render, encode)
    → SCHED_BATCH, or SCHED_NORMAL nice +5 to +15
    
  Background maintenance (indexing, backup)
    → SCHED_IDLE, or SCHED_NORMAL nice +19
    
  Audio processing (ALSA, PulseAudio thread)
    → SCHED_FIFO priority 50-70
    
  Hardware control (sensor, actuator loop)
    → SCHED_FIFO priority 80-90
    
  Periodic sampling (camera, audio frame)
    → SCHED_DEADLINE (runtime/deadline/period)
    
  Interrupt thread (threaded IRQ)
    → SCHED_FIFO priority 50 (kernel default)
    
  Watchdog, migration threads
    → SCHED_FIFO priority 99 (kernel internal)
```

---

## 23.10 Policy Interaction Diagram

```
Which task runs? (Multiple CPUs simplify this — one per CPU)

  Higher ┌──────────────────────────────────────────┐
  prio   │  SCHED_DEADLINE tasks                     │
  runs   │    → EDF pick (earliest absolute deadline) │
  first  ├──────────────────────────────────────────┤
         │  SCHED_FIFO / SCHED_RR tasks              │
         │    → Highest RT priority (99 first)        │
         │    → FIFO: first in queue at same prio     │
         │    → RR: round-robin at same prio          │
         ├──────────────────────────────────────────┤
         │  SCHED_NORMAL / SCHED_BATCH tasks          │
         │    → CFS: smallest vruntime                │
         │    → BATCH: same but no wakeup preemption  │
         ├──────────────────────────────────────────┤
         │  SCHED_IDLE tasks                          │
  Lower  │    → Only when nothing above wants CPU     │
         └──────────────────────────────────────────┘

  Between classes: ABSOLUTE priority (no sharing)
  Within CFS class: PROPORTIONAL sharing (weights/vruntime)
```

---

## 23.11 Kernel Implementation

```c
/* kernel/sched/core.c — policy to class mapping */
static const struct sched_class *__setscheduler_class(int policy)
{
    if (dl_policy(policy))
        return &dl_sched_class;
    if (rt_policy(policy))
        return &rt_sched_class;
    return &fair_sched_class;
}

/* Policy validation */
static bool valid_policy(int policy)
{
    return policy == SCHED_NORMAL || policy == SCHED_BATCH ||
           policy == SCHED_IDLE   || policy == SCHED_FIFO  ||
           policy == SCHED_RR     || policy == SCHED_DEADLINE;
}

/* Runtime policy checks */
#define rt_policy(policy)  ((policy) == SCHED_FIFO || (policy) == SCHED_RR)
#define dl_policy(policy)  ((policy) == SCHED_DEADLINE)
#define fair_policy(policy) ((policy) == SCHED_NORMAL || \
                             (policy) == SCHED_BATCH  || \
                             (policy) == SCHED_IDLE)
```

---

## Interview Questions

**Q1: What's the difference between SCHED_NORMAL and SCHED_BATCH?**
A: Both use CFS with the same vruntime/weight algorithm. The only difference: SCHED_BATCH tasks don't get the "wakeup preemption" bonus. When a SCHED_NORMAL task wakes, it checks if it should preempt the current task (if its vruntime is lower by more than `sched_wakeup_granularity`). SCHED_BATCH skips this check, reducing context switches at the cost of latency. Best for CPU-bound batch workloads.

**Q2: Can a SCHED_IDLE task starve?**
A: Not completely — CFS guarantees every task gets at least `sched_min_granularity` per scheduling period. But in practice, if many SCHED_NORMAL tasks run, SCHED_IDLE tasks may get negligible CPU. The key difference from `SCHED_OTHER` nice +19: SCHED_IDLE has an even lower effective weight. True starvation only happens with runaway RT tasks (mitigated by RT throttling).

**Q3: When should you use SCHED_DEADLINE vs SCHED_FIFO for real-time?**
A: Use SCHED_DEADLINE for **periodic tasks** where you know runtime, deadline, and period (e.g., 10ms audio buffer processed in 2ms every 10ms). Admission control guarantees bandwidth. Use SCHED_FIFO for **event-driven** tasks where execution is unpredictable (e.g., handling hardware events). SCHED_DEADLINE has higher kernel priority than SCHED_FIFO, provides bandwidth isolation, and prevents one RT task from monopolizing the CPU.

---

*Next: [Chapter 24 — Process Accounting](Chapter_24_Process_Accounting.md)*
