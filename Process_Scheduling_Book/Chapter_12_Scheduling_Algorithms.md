# Chapter 12: Scheduling Algorithms — Theory

## Learning Goals
- Understand classic scheduling algorithms used in OS theory
- Compare FCFS, Round Robin, Priority, Multilevel Queue, and MLFQ
- See how these relate to Linux's actual implementation

---

## 12.1 First Come First Serve (FCFS)

```
Arrival:  P1(24ms)  P2(3ms)  P3(3ms)

Timeline:
  |----P1(24ms)----|--P2(3ms)--|--P3(3ms)--|
  0               24          27          30

Average wait: (0 + 24 + 27) / 3 = 17ms

Problem: "Convoy effect" — short jobs stuck behind long ones.
```

**Linux equivalent**: `SCHED_FIFO` without preemption — highest priority runs until it yields or blocks. But SCHED_FIFO is preemptible by higher-priority RT tasks.

---

## 12.2 Round Robin Scheduling

```
Time quantum = 4ms

Timeline (P1=24ms, P2=3ms, P3=3ms):
  |P1|P2|P3|P1|P1|P1|P1|P1|
  0  4  7 10 14 18 22 26 30

P2 and P3 finish quickly. P1 gets remaining time in slices.
Average wait: much lower than FCFS for short jobs.

Key parameter: quantum size
  Too small → excessive context switching
  Too large → degrades to FCFS
  Typical:   1-10 ms
```

**Linux equivalent**: `SCHED_RR` — round-robin among tasks at the same RT priority level. Default timeslice: 100ms (adjustable via `/proc/sys/kernel/sched_rr_timeslice_ms`).

---

## 12.3 Priority Scheduling

```
Priority (lower = higher priority):
  P1(prio=3), P2(prio=1), P3(prio=2)

Execution order: P2 → P3 → P1

  |---P2---|---P3---|---P1---|

Problem: STARVATION — low-priority tasks may never run.
Solution: AGING — increase priority of waiting tasks over time.
```

**Linux**: Priority ranges 0-139. RT: 0-99 (higher value = higher priority). Normal: 100-139 (nice -20 to +19). CFS prevents starvation via proportional fair sharing.

---

## 12.4 Multilevel Queue Scheduling

```
┌─────────────────────────────┐  Highest priority
│  Queue 1: Real-time tasks   │  (SCHED_FIFO, SCHED_RR)
│  Algorithm: Priority FIFO   │
├─────────────────────────────┤
│  Queue 2: Interactive tasks │  (foreground)
│  Algorithm: Round Robin     │
├─────────────────────────────┤
│  Queue 3: Batch tasks       │  (background)
│  Algorithm: FCFS            │
└─────────────────────────────┘  Lowest priority

Rules:
  - Higher queue always preempts lower
  - Tasks don't move between queues
  - Each queue has its own algorithm
```

**Linux**: Scheduling classes are essentially a multilevel queue: stop > deadline > RT > fair > idle. Higher class always preempts lower.

---

## 12.5 Multilevel Feedback Queue Scheduling (MLFQ)

```
┌──────────────────────────────┐
│  Queue 0 (highest): RR, q=8ms│ ← New tasks start here
├──────────────────────────────┤
│  Queue 1: RR, q=16ms        │ ← Demoted after using quantum
├──────────────────────────────┤
│  Queue 2 (lowest): FCFS     │ ← CPU-bound tasks end up here
└──────────────────────────────┘

Rules:
  1. New task enters highest queue
  2. If task uses full quantum → demote to lower queue
  3. If task blocks before quantum → stay (likely interactive)
  4. Periodically boost all tasks (prevent starvation)
```

**Linux equivalent**: The O(1) scheduler (2.6.0-2.6.22) used a variant of MLFQ with interactivity heuristics (bonus/penalty for sleep time). CFS replaced this with a mathematical model (vruntime) that achieves similar goals more reliably.

---

## Algorithm Comparison

| Algorithm | Preemptive | Starvation | Complexity | Linux Mapping |
|-----------|-----------|-----------|-----------|---------------|
| FCFS | No | No | O(1) | SCHED_FIFO (within same priority) |
| Round Robin | Yes | No | O(1) | SCHED_RR |
| Priority | Yes/No | **Yes** | O(n) or O(1) | RT scheduling (0-99) |
| MLQ | Yes | **Yes** (low queues) | O(1) | Scheduling classes |
| MLFQ | Yes | No (with boosting) | O(1) | Historical O(1) scheduler |
| **CFS** | Yes | No | O(log n) | SCHED_NORMAL (current) |

---

## Interview Questions

**Q1: What scheduling algorithm does Linux actually use?**
A: Linux uses a **class-based** system. For normal tasks: CFS (fair scheduling using vruntime and red-black tree). For RT tasks: priority-based FIFO or Round Robin. For deadline tasks: EDF (Earliest Deadline First). The classes form a multilevel queue with priority ordering.

**Q2: How does CFS achieve fairness without a fixed timeslice?**
A: CFS tracks `vruntime` for each task — the weighted CPU time consumed. The task with the smallest `vruntime` runs next. Higher-weight tasks (lower nice) accumulate vruntime slower, so they get more CPU. This naturally converges to proportional fair sharing without fixed quanta.

**Q3: What is the convoy effect and how does CFS mitigate it?**
A: Convoy effect: short jobs wait behind a long CPU-bound job (FCFS problem). CFS mitigates it because: 1) the long job's vruntime increases, making it less eligible. 2) Short jobs with less vruntime get picked first. 3) Preemption ensures the running task is replaced if a more deserving one arrives.

---

*Next: [Chapter 13 — Completely Fair Scheduler (CFS)](Chapter_13_CFS.md)*
