# Chapter 12: Priority Inversion

## Learning Goals
- Understand priority inversion and why it threatens real-time guarantees
- Master Priority Inheritance Protocol (PIP)
- Learn Priority Ceiling Protocol (PCP) and Immediate PCP
- Analyze the Mars Pathfinder incident in detail
- Know how different RTOS implement inversion prevention
- Perform priority inversion analysis in system design

---

## 1. Priority Inversion Explained

```
  Unbounded Priority Inversion
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Three tasks: H (high), M (medium), L (low)             │
  │  One shared mutex between H and L                        │
  │                                                           │
  │  Time ─────────────────────────────────────────────────► │
  │                                                           │
  │  H (prio 3):          ▓▓BLOCKED▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▒▒▒▒  │
  │  M (prio 2):             ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓        │
  │  L (prio 1):  ▓▓▓L──────┤                    ├──▓▓▓     │
  │                  ▲ lock  │    M preempts L    │unlock    │
  │                  │       │    L can't finish  │  ▲       │
  │                  │       │    H can't run!    │  │       │
  │                  │       └────────────────────┘  │       │
  │                  │                                │       │
  │  Problem:                                                │
  │  1. L acquires mutex                                     │
  │  2. H becomes ready, tries mutex → blocks               │
  │  3. M becomes ready, preempts L (M > L priority)        │
  │  4. M runs for a LONG time (unbounded!)                  │
  │  5. L cannot finish and release mutex                    │
  │  6. H is blocked by M, even though H > M priority!      │
  │                                                           │
  │  The "inversion": H is effectively running at priority   │
  │  LOWER than M, even though H has highest priority.       │
  │  Duration is UNBOUNDED — any medium-priority task can    │
  │  extend the inversion.                                   │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Priority Inheritance Protocol (PIP)

```
  Priority Inheritance — prevents unbounded inversion
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Time ─────────────────────────────────────────────────► │
  │                                                           │
  │  H (prio 3):          ▓▓BLOCKED▓▓▓▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒     │
  │  M (prio 2):             wait───────▓▓▓▓▓▓▓▓▓▓▓▓        │
  │  L (prio 1→3):▓▓▓L──────▓▓▓▓▓▓L────┤                    │
  │                  ▲ lock  ▲ inherit  │unlock              │
  │                  │       │L→prio 3  │L→prio 1            │
  │                  │       │M can't   │H runs              │
  │                  │       │preempt!  │M runs after         │
  │                                                           │
  │  How it works:                                           │
  │  1. L acquires mutex                                     │
  │  2. H becomes ready, tries mutex → blocks               │
  │  3. L's priority RAISED to H's priority (3)             │
  │  4. M becomes ready, but L is now prio 3 → M waits     │
  │  5. L finishes critical section, releases mutex          │
  │  6. L's priority REVERTS to original (1)                 │
  │  7. H runs (highest ready priority)                      │
  │  8. H completes, then M runs                             │
  │                                                           │
  │  Inversion duration = L's critical section time only     │
  │  BOUNDED, but not minimal (depends on L's CS length)     │
  │                                                           │
  │  Properties:                                             │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ + Simple to implement                           │      │
  │  │ + No prior knowledge of resource/task map needed│      │
  │  │ + FreeRTOS mutex implements this by default      │      │
  │  │ - Can cause chain blocking (A→B→C→...)          │      │
  │  │ - Does not prevent deadlock                      │      │
  │  │ - Inheritance may be transitive (complex)       │      │
  │  └────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Priority Ceiling Protocol (PCP)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Priority Ceiling Protocol (Sha, Rajkumar, 1990):        │
  │                                                           │
  │  Ceiling(mutex) = highest priority of ANY task that      │
  │                   can lock that mutex                     │
  │                                                           │
  │  Rule: task can only lock a mutex if its priority is     │
  │  HIGHER than the ceiling of any mutex currently locked   │
  │  by OTHER tasks.                                         │
  │                                                           │
  │  Example:                                                │
  │  ┌─────────┬─────────┬───────────────────┐              │
  │  │ Mutex   │ Ceiling │ Tasks that use it  │              │
  │  ├─────────┼─────────┼───────────────────┤              │
  │  │ Mutex A │ 5       │ TaskH(5), TaskL(1) │              │
  │  │ Mutex B │ 3       │ TaskM(3), TaskL(1) │              │
  │  └─────────┴─────────┴───────────────────┘              │
  │                                                           │
  │  Properties:                                             │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ + Prevents DEADLOCK (proven)                    │      │
  │  │ + At most ONE blocking per task (optimal)       │      │
  │  │ + No chain blocking                             │      │
  │  │ - Requires static analysis: must know all       │      │
  │  │   task-resource mappings at design time          │      │
  │  │ - Conservative: may block unnecessarily          │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Immediate Priority Ceiling Protocol (IPCP):             │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ Also called "Priority Ceiling Emulation"        │      │
  │  │                                                  │      │
  │  │ When task ACQUIRES mutex:                        │      │
  │  │   Immediately raise task to mutex's ceiling      │      │
  │  │   (even if no higher-priority task is waiting)   │      │
  │  │                                                  │      │
  │  │ When task RELEASES mutex:                        │      │
  │  │   Revert to previous priority                    │      │
  │  │                                                  │      │
  │  │ Simpler than PCP, same deadlock prevention       │      │
  │  │ VxWorks uses this (SEM_INVERSION_SAFE + ceiling) │      │
  │  │ POSIX: PTHREAD_PRIO_PROTECT                      │      │
  │  └────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Mars Pathfinder — Detailed Analysis

```
  Mars Pathfinder Priority Inversion Incident (July 1997)
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  System: VxWorks RTOS on RAD6000 processor               │
  │                                                           │
  │  Tasks involved:                                         │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │ BC_SCHED (High, prio ~high):                    │     │
  │  │   Bus scheduler — manages 1553 data bus         │     │
  │  │   Runs periodically, must meet deadline         │     │
  │  │   Watchdog: system reset if BC_SCHED misses     │     │
  │  │                                                  │     │
  │  │ COMM_TASK (Medium, prio ~medium):               │     │
  │  │   Communication task — long-running data pipe   │     │
  │  │   Processes and distributes data                │     │
  │  │                                                  │     │
  │  │ MET_TASK (Low, prio ~low):                      │     │
  │  │   Meteorological data collection                │     │
  │  │   Publishes weather data to shared buffer       │     │
  │  └─────────────────────────────────────────────────┘     │
  │                                                           │
  │  Shared resource: information bus pipe (protected by     │
  │  a VxWorks mutex — but priority inheritance was          │
  │  DISABLED by default in the VxWorks configuration)       │
  │                                                           │
  │  Failure sequence:                                       │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │ 1. MET_TASK acquires bus pipe mutex               │     │
  │  │ 2. MET_TASK preempted by COMM_TASK (medium prio) │     │
  │  │ 3. BC_SCHED tries to acquire mutex → BLOCKS       │     │
  │  │ 4. COMM_TASK runs for extended period             │     │
  │  │    (data distribution takes a long time)          │     │
  │  │ 5. MET_TASK can't run → can't release mutex       │     │
  │  │ 6. BC_SCHED can't run → misses watchdog deadline  │     │
  │  │ 7. WATCHDOG FIRES → SYSTEM RESET                  │     │
  │  │ 8. System reboots, data & context lost            │     │
  │  │ 9. This happened repeatedly on Mars               │     │
  │  └─────────────────────────────────────────────────┘     │
  │                                                           │
  │  The fix (applied from Earth, 100+ million miles away):  │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │ 1. Enable VxWorks priority inheritance on the    │     │
  │  │    mutex protecting the bus pipe                  │     │
  │  │    (mutexOptionsSet: SEM_INVERSION_SAFE)          │     │
  │  │                                                   │     │
  │  │ 2. When BC_SCHED blocks on mutex, MET_TASK's     │     │
  │  │    priority is raised to BC_SCHED's level         │     │
  │  │                                                   │     │
  │  │ 3. COMM_TASK can no longer preempt MET_TASK       │     │
  │  │                                                   │     │
  │  │ 4. MET_TASK quickly finishes and releases mutex   │     │
  │  │                                                   │     │
  │  │ 5. BC_SCHED runs, meets watchdog deadline         │     │
  │  │                                                   │     │
  │  │ The fix was a C global variable change uploaded   │     │
  │  │ via command, toggling a VxWorks mutex option.     │     │
  │  └─────────────────────────────────────────────────┘     │
  │                                                           │
  │  Lessons learned:                                        │
  │  1. ALWAYS enable priority inheritance on shared mutexes │
  │  2. Test under realistic load (timing bugs are elusive)  │
  │  3. VxWorks should have had inheritance ON by default    │
  │  4. Watchdog alone doesn't fix design issues             │
  │  5. Remote debugging capability is critical              │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Implementation Across RTOS

```c
/* FreeRTOS: Priority inheritance is automatic for mutexes */
SemaphoreHandle_t xMutex = xSemaphoreCreateMutex();
/* PI is enabled by default — no additional configuration */
/* Stored in TCB: uxBasePriority (original), uxPriority (current) */
/* On mutex take: if holder->uxPriority < requester->uxPriority,
 *   holder->uxPriority = requester->uxPriority (inherit)
 * On mutex give: holder->uxPriority = holder->uxBasePriority (revert)
 */

/* VxWorks: Must explicitly enable */
SEM_ID mutex = semMCreate(
    SEM_Q_PRIORITY |          /* Wait queue by priority */
    SEM_INVERSION_SAFE |      /* Enable priority inheritance */
    SEM_DELETE_SAFE            /* Prevent task delete while holding */
);
/* SEM_INVERSION_SAFE options:
 * Default: basic priority inheritance
 * With VX_MUTEX_PRIO_CEILING: immediate priority ceiling
 */

/* QNX Neutrino (POSIX): */
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
/* or: PTHREAD_PRIO_PROTECT for priority ceiling */
pthread_mutex_init(&mutex, &attr);

/* If using PTHREAD_PRIO_PROTECT, set ceiling: */
pthread_mutexattr_setprioceiling(&attr, HIGHEST_USER_PRIORITY);

/* Zephyr: Priority inheritance built into k_mutex */
struct k_mutex my_mutex;
k_mutex_init(&my_mutex);    /* PI enabled by default */
k_mutex_lock(&my_mutex, K_FOREVER);
/* Thread gets priority boost if higher-priority thread waits */
k_mutex_unlock(&my_mutex);
```

---

## 6. Analysis and Prevention

```
  System Design with Priority Inversion Analysis
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Step 1: Map all task-resource interactions               │
  │  ┌────────┬───────────────┬─────────────────────────┐   │
  │  │ Task   │ Priority      │ Resources Used           │   │
  │  ├────────┼───────────────┼─────────────────────────┤   │
  │  │ Motor  │ 5 (highest)   │ MutexSPI, MutexState    │   │
  │  │ CAN    │ 4             │ MutexCAN, MutexState    │   │
  │  │ Sensor │ 3             │ MutexSPI                │   │
  │  │ Log    │ 1 (lowest)    │ MutexState              │   │
  │  └────────┴───────────────┴─────────────────────────┘   │
  │                                                           │
  │  Step 2: Calculate priority ceilings                     │
  │  ┌──────────┬─────────┬───────────────────────────┐     │
  │  │ Resource │ Ceiling │ Reason                     │     │
  │  ├──────────┼─────────┼───────────────────────────┤     │
  │  │ MutexSPI │ 5       │ Motor(5) and Sensor(3)    │     │
  │  │ MutexCAN │ 4       │ CAN(4) only               │     │
  │  │ MutexState│ 5      │ Motor(5), CAN(4), Log(1)  │     │
  │  └──────────┴─────────┴───────────────────────────┘     │
  │                                                           │
  │  Step 3: Analyze worst-case blocking                     │
  │  With PIP: blocking time ≤ longest CS of lower-prio task│
  │  With PCP: blocking time ≤ longest CS of ANY lower task │
  │            (but only ONE blocking per task)               │
  │                                                           │
  │  Step 4: Include blocking in schedulability analysis     │
  │                                                           │
  │  Response time with blocking:                            │
  │  Ri = Ci + Bi + Σ ⌈Ri/Tj⌉ × Cj                        │
  │                                                           │
  │  where Bi = worst-case blocking time for task i          │
  │  (maximum CS length of any lower-priority task using     │
  │   a resource that task i also uses)                      │
  │                                                           │
  │  Check: Ri ≤ Di for all tasks                           │
  └──────────────────────────────────────────────────────────┘
```

---

## 7. Priority Inversion in Multi-Core (SMP)

```
  SMP Priority Inversion Scenarios
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Core 0              Core 1                              │
  │  ┌──────────┐       ┌──────────┐                        │
  │  │ Task H   │       │ Task M   │                        │
  │  │ (prio 5) │       │ (prio 3) │                        │
  │  │ wants    │       │ running  │                        │
  │  │ mutex    │       │ freely   │                        │
  │  └──────────┘       └──────────┘                        │
  │                                                           │
  │  Mutex held by Task L (prio 1) — NOT running on either  │
  │  core because M (prio 3) is running!                     │
  │                                                           │
  │  SMP complications:                                      │
  │  1. Priority inheritance must cross cores               │
  │  2. Task L must be scheduled on SOME core to finish     │
  │  3. Simple UP inheritance doesn't work — L needs CPU    │
  │  4. Solution: preempt M on Core 1 to run L             │
  │     (cross-core preemption via IPI)                     │
  │                                                           │
  │  SMP RTOS solutions:                                     │
  │  - Cross-core IPI for preemption notification           │
  │  - Global ready queue with global scheduling decision   │
  │  - FreeRTOS SMP (v11+): handles this via IPI            │
  │  - VxWorks SMP: deterministic cross-core scheduling     │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Explain unbounded priority inversion with a concrete example.**
**A:** Three tasks: H (high), M (medium), L (low) sharing a mutex between H and L. Sequence: (1) L acquires mutex, (2) L is preempted by H, which tries the mutex and blocks, (3) M becomes ready and preempts L (M > L priority), (4) M runs for an arbitrarily long time, (5) L can't run (M keeps it off CPU), so L can't release mutex, (6) Therefore H is blocked for M's entire execution time — even though H has higher priority than M. This is "unbounded" because any number of medium-priority tasks can run and extend the blocking. The inversion: H effectively has lower priority than M.

**Q2: Compare Priority Inheritance and Priority Ceiling protocols.**
**A:** Priority Inheritance (PIP): When high-priority task blocks on mutex, holder's priority is temporarily raised to waiter's level. Reactive — only triggers when conflict occurs. Does not prevent deadlock. May cause chain blocking (A blocks on B's mutex, B blocks on C's mutex). Simpler to implement; no prior knowledge needed. Priority Ceiling (PCP): Each mutex has a "ceiling" = highest priority of any task that uses it. Task can only acquire mutex if its priority > ceiling of all mutexes currently held by other tasks. Prevents deadlock. At most one priority inversion per task. But requires complete static analysis of all task-resource mappings at design time. IPCP (Immediate): raises priority on acquisition, simpler variant used in practice.

**Q3: Describe the Mars Pathfinder bug. What was the root cause and fix?**
**A:** Root cause: priority inversion between bus scheduler (high), communication task (medium), and meteorological task (low) sharing a mutex on VxWorks. The mutex's priority inheritance was disabled by default in VxWorks configuration. When the low-priority met task held the mutex and was preempted by the medium-priority comm task, the high-priority bus scheduler couldn't run. It missed its watchdog deadline, triggering a system reset. Fix: upload a command from Earth to enable VxWorks `SEM_INVERSION_SAFE` flag on the shared mutex. This was a single configuration change. Lesson: always enable priority inheritance on any mutex shared between tasks of different priorities. The bug was hard to reproduce on ground — it only manifested under specific timing conditions with realistic data volumes.

**Q4: How does priority inversion analysis fit into RMS schedulability?**
**A:** The standard RMS response time analysis: Ri = Ci + Σ⌈Ri/Tj⌉×Cj. With blocking: Ri = Ci + Bi + Σ⌈Ri/Tj⌉×Cj, where Bi is the worst-case blocking time for task i. To compute Bi: identify all mutexes that task i shares with lower-priority tasks, find the longest critical section among those lower-priority tasks' uses of those mutexes. With PIP: Bi = max of all such critical sections (chain blocking possible). With PCP: Bi = single longest critical section. Then check Ri ≤ Di. If not schedulable, options: reduce critical section lengths, raise clock speed, or restructure task-resource mapping to eliminate sharing.

---

## Summary

- Unbounded priority inversion: medium-priority task delays high-priority via shared mutex
- Priority Inheritance (PIP): raise holder to waiter's priority — reactive, no deadlock prevention
- Priority Ceiling (PCP): restrict locking based on ceilings — prevents deadlock, one blocking max
- Immediate PCP: raise to ceiling on acquisition — simpler, used in VxWorks/POSIX
- Mars Pathfinder: real-world inversion bug, fixed by enabling VxWorks priority inheritance from Earth
- FreeRTOS/Zephyr: inheritance enabled by default on mutexes
- Include blocking time Bi in response time analysis: Ri = Ci + Bi + Σ⌈Ri/Tj⌉×Cj
- SMP complicates inversion: cross-core IPI needed for preemption

---

[Previous Chapter: Synchronization Primitives ←](Chapter_11_Synchronization.md) | [Next Chapter: Memory Management →](Chapter_13_Memory_Management.md)
