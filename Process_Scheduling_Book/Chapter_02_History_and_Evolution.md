# Chapter 2: History and Evolution of Process Management

## Learning Goals
- Trace the evolution from batch processing to modern preemptive multitasking
- Understand how Linux scheduling evolved from O(n) to CFS to EEVDF
- Appreciate the design decisions behind each scheduler generation

---

## 2.1 Early Batch Processing Systems (1950s–1960s)

```
┌─────────────────────────────────────────────┐
│           Batch Processing Model            │
│                                             │
│  Job1 ──► [CPU] ──► Done                    │
│  Job2 ──► [CPU] ──► Done                    │
│  Job3 ──► [CPU] ──► Done                    │
│                                             │
│  One job at a time. CPU idle during I/O.    │
│  No multitasking. No interactivity.         │
└─────────────────────────────────────────────┘
```

- Jobs submitted on punch cards, executed sequentially
- CPU utilization was very low — idle during I/O waits
- No concept of "process" — just a single running job

---

## 2.2 Introduction of Multiprogramming (1960s)

Key insight: **Keep multiple jobs in memory; when one waits for I/O, switch to another.**

```
Memory:  [OS] [Job A] [Job B] [Job C]

Time:    Job A runs → I/O wait → Job B runs → I/O wait → Job C runs
         CPU never idle (as long as jobs available)
```

- IBM OS/360 (1964): First major multiprogramming OS
- Job scheduling was non-preemptive — jobs ran until they blocked
- Memory protection introduced to isolate jobs

---

## 2.3 Time-Sharing Systems (1960s–1970s)

**Revolution**: Give each user a **time slice** of the CPU — interactive computing.

| System | Year | Key Innovation |
|--------|------|---------------|
| CTSS (MIT) | 1961 | First time-sharing system |
| Multics | 1965 | Process model, hierarchical file system |
| Unix | 1969 | Simple, elegant process model (fork/exec) |

```
Time-sharing: CPU rapidly switches between users
  
  Time ──────────────────────────────►
  CPU:  [User1][User2][User3][User1][User2]...
        Each user thinks they have the whole machine
```

---

## 2.4 Process Management in Unix (1969–1990s)

Unix introduced the fundamental concepts that Linux inherits:

| Concept | Unix Innovation |
|---------|----------------|
| `fork()` | Create process by duplicating parent |
| `exec()` | Replace process image with new program |
| `wait()` | Parent waits for child termination |
| Pipes | `\|` — process-to-process data flow |
| Signals | Asynchronous process notification |
| Process hierarchy | Parent-child tree rooted at PID 1 (init) |

```
Unix Process Creation Model:
  
  parent ──fork()──► child (copy of parent)
                        │
                     exec()
                        │
                        ▼
                   new program running
```

Unix scheduling (V7, 4.3BSD): multi-level priority queues with aging.

---

## 2.5 Evolution of Process Management in Linux

| Era | Kernel | Change |
|-----|--------|--------|
| 1991 | 0.01 | Simple round-robin scheduler |
| 1995 | 1.2 | Basic priority scheduler |
| 2001 | 2.4 | O(n) scheduler — scanned all tasks every tick |
| 2003 | 2.6.0 | **O(1) scheduler** (Ingo Molnár) — constant-time scheduling |
| 2007 | 2.6.23 | **CFS** (Completely Fair Scheduler, Ingo Molnár) — red-black tree |
| 2009 | 2.6.31 | SCHED_DEADLINE added (EDF — Earliest Deadline First) |
| 2014 | 3.14+ | deadline scheduling refined, CPU bandwidth throttling |
| 2023 | 6.6 | **EEVDF** (Earliest Eligible Virtual Deadline First) replaces CFS |

---

## 2.6 Evolution of Linux Scheduling Algorithms

### The O(n) Scheduler (Linux 2.4)

```c
/* Simplified: scan ALL tasks to find best */
struct task_struct *pick_next_task(void)
{
    struct task_struct *p, *best = NULL;
    int best_goodness = -1000;

    for_each_task(p) {
        int g = goodness(p, prev);  /* Priority + bonus */
        if (g > best_goodness) {
            best_goodness = g;
            best = p;
        }
    }
    return best;
}
/* Problem: with 1000 tasks, scans 1000 entries every schedule() */
```

**Limitations**: O(n) complexity — scheduling latency grew linearly with task count. Unacceptable for servers with thousands of tasks.

### The O(1) Scheduler (Linux 2.6.0–2.6.22)

```
Active Array       Expired Array
┌───────────┐     ┌───────────┐
│ Prio 0    │     │ Prio 0    │
│ Prio 1    │     │ Prio 1    │
│ ...       │     │ ...       │
│ Prio 139  │     │ Prio 139  │
└───────────┘     └───────────┘
      │                 ▲
      │ task uses up    │
      │ timeslice       │
      └─────────────────┘
      When active empty: swap pointers
```

- Two priority arrays (active/expired), 140 priority levels
- `pick_next_task()` = find first set bit → O(1)
- **Problem**: Interactive detection heuristics were fragile; unfair to some workloads

### CFS — Completely Fair Scheduler (Linux 2.6.23–6.5)

```
Red-Black Tree (sorted by vruntime)
                    ┌─────┐
                    │vr=50│
                   ╱       ╲
              ┌─────┐     ┌─────┐
              │vr=30│     │vr=80│
             ╱       ╲         ╲
        ┌─────┐  ┌─────┐  ┌─────┐
        │vr=10│  │vr=40│  │vr=90│
        └─────┘  └─────┘  └─────┘
              ↑
         Leftmost = smallest vruntime = next to run
```

- **Virtual runtime (vruntime)**: tracks proportional CPU consumption
- Leftmost node = most deserving task → runs next
- O(log n) insert/remove — excellent scalability
- **Fair**: each task gets CPU proportional to its weight

### EEVDF — Earliest Eligible Virtual Deadline First (Linux 6.6+)

- Improvement over CFS: adds **virtual deadline** concept
- Task with earliest virtual deadline among **eligible** tasks runs next
- Better latency guarantees, less reliance on heuristics
- Same red-black tree structure, different selection criteria

---

## Timeline Diagram

```
1991        2001        2003        2007        2009        2023
 │           │           │           │           │           │
 ▼           ▼           ▼           ▼           ▼           ▼
Round     O(n)        O(1)        CFS        SCHED_      EEVDF
Robin     sched       sched       (vruntime)  DEADLINE    (replaces
                      (bitmap)    (rbtree)    (EDF)       CFS core)
 │           │           │           │           │           │
 └───────────┴───────────┴───────────┴───────────┴───────────┘
             Linux Scheduler Evolution
```

---

## Interview Questions

**Q1: Why did Linux move from O(1) to CFS?**
A: The O(1) scheduler used heuristics to detect interactive tasks, which were unreliable and led to unfairness. CFS replaced heuristics with a mathematical model: each task tracks `vruntime` (weighted CPU time consumed). The task with the least vruntime runs next — inherently fair without heuristics.

**Q2: What is the time complexity of CFS pick_next_task()?**
A: O(1) — the leftmost node of the red-black tree is cached (`rb_leftmost`). Insertion/deletion is O(log n). The overall schedule() path is effectively O(log n) due to enqueue/dequeue of sched entities.

**Q3: What is EEVDF and why was it introduced?**
A: EEVDF (Earliest Eligible Virtual Deadline First) was introduced in Linux 6.6 to replace CFS's core selection logic. It adds a virtual deadline to each task and selects the eligible task with the earliest deadline. This provides better latency guarantees for latency-sensitive tasks without the `sched_latency` tuning knobs.

**Q4: How did Unix's fork/exec model differ from VMS or Windows?**
A: Unix creates processes by duplicating (fork) then replacing (exec) — simple, composable. VMS (`$CREPRC`) and Windows (`CreateProcess()`) create processes from scratch, specifying the program upfront. Unix's approach enables powerful patterns like pipe chains, but can be less efficient without copy-on-write.

---

*Next: [Chapter 3 — Hardware Support for Process Management](Chapter_03_Hardware_Support.md)*
