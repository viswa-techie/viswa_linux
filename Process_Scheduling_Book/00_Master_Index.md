# Linux Process Management & CPU Scheduling — Master Index

> **A Complete Study Guide**: 35 chapters covering the entire domain of
> process management and CPU scheduling in the Linux kernel.
> Suitable as a university-level course, kernel developer handbook,
> and interview preparation guide.

---

## Book Information

```
Subject  : Linux Process Management & CPU Scheduling
Scope    : Linux Kernel 5.x — 6.x (x86_64, ARM64)
Chapters : 35
Audience : Kernel developers, embedded engineers, CS students,
           interview candidates
Format   : Markdown with ASCII diagrams, C code, kernel references
```

---

## Complete Chapter List

### Part I: Foundations (Chapters 1–6)

| #  | Chapter | Key Topics |
|----|---------|------------|
| 01 | [Introduction & Foundations](Chapter_01_Foundations.md) | OS role, process vs thread, kernel vs user, Linux history |
| 02 | [History of Process Scheduling](Chapter_02_History.md) | O(n) → O(1) → CFS → EEVDF evolution, design motivations |
| 03 | [Hardware Support for Scheduling](Chapter_03_Hardware.md) | Timer interrupts, TSC, APIC, ARM GIC, TLB, ASID/PCID |
| 04 | [Process Representation — task_struct](Chapter_04_Task_Struct.md) | task_struct fields, memory layout, allocation, key members |
| 05 | [Process States Deep Dive](Chapter_05_Process_States.md) | State machine, transitions, TASK_RUNNING through EXIT_DEAD |
| 06 | [Process Lifecycle](Chapter_06_Lifecycle.md) | Birth → ready → running → waiting → exit, complete flow |

### Part II: Process Creation & Management (Chapters 7–10)

| #  | Chapter | Key Topics |
|----|---------|------------|
| 07 | [Process Creation Mechanisms](Chapter_07_Process_Creation.md) | fork(), vfork(), clone(), clone3(), COW, kernel_clone() |
| 08 | [Threads in Linux](Chapter_08_Threads.md) | NPTL, 1:1 model, pthread API, CLONE flags, thread_group |
| 09 | [Kernel Threads](Chapter_09_Kernel_Threads.md) | kthreadd, kworker, ksoftirqd, migration, RCU threads |
| 10 | [Context Switching](Chapter_10_Context_Switching.md) | switch_mm, switch_to, cpu_switch_to, register save/restore |

### Part III: Scheduler Architecture (Chapters 11–15)

| #  | Chapter | Key Topics |
|----|---------|------------|
| 11 | [Scheduler Architecture Overview](Chapter_11_Scheduler_Architecture.md) | Scheduling classes, struct rq, class hierarchy, modular design |
| 12 | [Scheduling Algorithms](Chapter_12_Scheduling_Algorithms.md) | FCFS, Round Robin, Priority, MLQ, MLFQ — theory and Linux mapping |
| 13 | [Completely Fair Scheduler (CFS)](Chapter_13_CFS.md) | vruntime, RB tree, weight table, timeslice calculation, EEVDF |
| 14 | [Real-Time Scheduling](Chapter_14_Real_Time_Scheduling.md) | SCHED_FIFO, SCHED_RR, SCHED_DEADLINE, EDF+CBS, RT throttling |
| 15 | [Scheduler Data Structures](Chapter_15_Scheduler_Data_Structures.md) | struct rq, cfs_rq, rt_rq, dl_rq, sched_entity, PELT, domains |

### Part IV: CPU Management (Chapters 16–18)

| #  | Chapter | Key Topics |
|----|---------|------------|
| 16 | [CPU Scheduling & Load Balancing](Chapter_16_CPU_Scheduling.md) | Multi-core, periodic/idle/wake balancing, affinity, NUMA, EAS |
| 17 | [Priority and Nice Values](Chapter_17_Priority_Nice_Values.md) | 0-139 range, 4 prio fields, weight mapping, PI, autogroup |
| 18 | [Preemption Models](Chapter_18_Preemption.md) | NONE/VOLUNTARY/PREEMPT/PREEMPT_RT, TIF_NEED_RESCHED, preempt_count |

### Part V: Synchronization & Communication (Chapters 19–22)

| #  | Chapter | Key Topics |
|----|---------|------------|
| 19 | [Process Synchronization](Chapter_19_Process_Synchronization.md) | Atomics, spinlocks, mutexes, rwlocks, RCU, semaphores, barriers |
| 20 | [Inter-Process Communication](Chapter_20_IPC.md) | Pipes, message queues, shared memory, futex, sockets, Binder |
| 21 | [Signals](Chapter_21_Signals.md) | Signal delivery, handlers, kernel path, RT signals, scheduling impact |
| 22 | [Process Groups and Sessions](Chapter_22_Process_Groups_Sessions.md) | PGID, SID, job control, controlling terminal, orphan groups |

### Part VI: Policies & Accounting (Chapters 23–25)

| #  | Chapter | Key Topics |
|----|---------|------------|
| 23 | [Scheduling Policies](Chapter_23_Scheduling_Policies.md) | All 6 policies, API (chrt, sched_setattr), policy selection guide |
| 24 | [Process Accounting](Chapter_24_Process_Accounting.md) | CPU time, ctx switches, I/O, /proc, cgroup accounting, schedstat |
| 25 | [Scheduler Performance Optimization](Chapter_25_Scheduler_Performance.md) | Context switch cost, cache locality, tuning, NUMA, tickless |

### Part VII: Source Code & Tools (Chapters 26–28)

| #  | Chapter | Key Topics |
|----|---------|------------|
| 26 | [Kernel Source Code Walkthrough](Chapter_26_Kernel_Source_Code.md) | core.c, fair.c, rt.c walkthrough, file map, code reading tips |
| 27 | [Process Debugging Tools](Chapter_27_Process_Debugging_Tools.md) | ps, top, perf, strace, pidstat, /proc, diagnosing issues |
| 28 | [Kernel Tracing for Scheduling](Chapter_28_Kernel_Tracing.md) | ftrace, trace-cmd, perf sched, perfetto, bpftrace |

### Part VIII: Visual Reference (Chapters 29–30)

| #  | Chapter | Key Topics |
|----|---------|------------|
| 29 | [Flow Diagrams](Chapter_29_Flow_Diagrams.md) | fork, exec, exit, __schedule, CFS pick, wakeup, signal flows |
| 30 | [Important Diagrams](Chapter_30_Important_Diagrams.md) | State transitions, class hierarchy, rq layout, topology, priority map |

### Part IX: Reference & Advanced (Chapters 31–35)

| #  | Chapter | Key Topics |
|----|---------|------------|
| 31 | [Glossary](Chapter_31_Glossary.md) | A-Z definitions with kernel context and chapter cross-references |
| 32 | [OS Comparison](Chapter_32_OS_Comparison.md) | Linux vs Windows vs macOS vs QNX vs FreeRTOS vs Zephyr |
| 33 | [Embedded System Scheduling](Chapter_33_Embedded_Scheduling.md) | Automotive, Android, audio, PREEMPT_RT, power-aware, industrial |
| 34 | [Documentation and References](Chapter_34_References.md) | Books, papers, kernel docs, LWN, tools, learning path |
| 35 | [Interview Preparation](Chapter_35_Interview_Preparation.md) | Basic/intermediate/advanced Q&A, scenarios, debugging, design |

---

## Reading Paths

### Path 1: Quick Start (Core Scheduler Knowledge)
```
Chapter 01 → 04 → 05 → 07 → 10 → 11 → 13 → 14 → 18 → 35
  (Foundations → task_struct → States → fork → Context Switch →
   Architecture → CFS → RT → Preemption → Interview Prep)
```

### Path 2: Complete Course (University Level)
```
Chapters 01 → 35 sequentially
  (~35 hours reading + practice time)
```

### Path 3: Embedded/Automotive Engineer
```
Chapter 03 → 10 → 14 → 16 → 18 → 19 → 25 → 28 → 33
  (Hardware → Context Switch → RT → Load Balance → Preemption →
   Sync → Performance → Tracing → Embedded)
```

### Path 4: Interview Preparation (1-Week Sprint)
```
Day 1: Chapter 04 (task_struct), 05 (States), 07 (fork)
Day 2: Chapter 13 (CFS), 14 (RT), 17 (Priority)
Day 3: Chapter 10 (Context Switch), 18 (Preemption)
Day 4: Chapter 19 (Synchronization), 20 (IPC)
Day 5: Chapter 16 (Load Balancing), 15 (Data Structures)
Day 6: Chapter 29 (Flow Diagrams), 30 (Diagrams)
Day 7: Chapter 35 (Interview Q&A) — full review
```

### Path 5: Debugging & Performance
```
Chapter 24 → 25 → 27 → 28 → 29
  (Accounting → Optimization → Debug Tools → Tracing → Flows)
```

---

## Key Kernel Functions Reference

```
Function                │ File                    │ Chapter
────────────────────────┼─────────────────────────┼────────
kernel_clone()          │ kernel/fork.c           │ 07
copy_process()          │ kernel/fork.c           │ 07
__schedule()            │ kernel/sched/core.c     │ 11, 26
pick_next_task()        │ kernel/sched/core.c     │ 11, 26
context_switch()        │ kernel/sched/core.c     │ 10, 26
try_to_wake_up()        │ kernel/sched/core.c     │ 16, 26
scheduler_tick()        │ kernel/sched/core.c     │ 11, 26
update_curr()           │ kernel/sched/fair.c     │ 13, 26
pick_next_entity()      │ kernel/sched/fair.c     │ 13, 26
load_balance()          │ kernel/sched/fair.c     │ 16, 26
select_task_rq_fair()   │ kernel/sched/fair.c     │ 16
pick_next_task_rt()     │ kernel/sched/rt.c       │ 14, 26
do_exit()               │ kernel/exit.c           │ 06
do_execveat_common()    │ fs/exec.c               │ 07
do_signal()             │ kernel/signal.c         │ 21
cpu_switch_to           │ arch/arm64/kernel/entry.S│ 10
switch_mm()             │ arch/arm64/mm/context.c │ 10
```

---

## Key Data Structures Reference

```
Structure           │ Header/File                    │ Chapter
────────────────────┼────────────────────────────────┼────────
task_struct          │ include/linux/sched.h          │ 04
sched_entity         │ include/linux/sched.h          │ 15
sched_rt_entity      │ include/linux/sched.h          │ 15
sched_dl_entity      │ include/linux/sched.h          │ 15
rq (struct rq)       │ kernel/sched/sched.h           │ 15
cfs_rq               │ kernel/sched/sched.h           │ 13, 15
rt_rq                │ kernel/sched/sched.h           │ 14, 15
dl_rq                │ kernel/sched/sched.h           │ 14, 15
sched_class          │ kernel/sched/sched.h           │ 11
sched_domain         │ include/linux/sched/topology.h │ 16
sched_group          │ include/linux/sched/topology.h │ 16
signal_struct        │ include/linux/sched/signal.h   │ 21
pid / struct pid     │ include/linux/pid.h            │ 07
mm_struct             │ include/linux/mm_types.h       │ 10
thread_struct         │ arch/*/include/asm/processor.h │ 10
pt_regs               │ arch/*/include/asm/ptrace.h    │ 10
```

---

## Scheduling Policies Quick Reference

```
Policy          │ Class  │ Priority  │ Timeslice   │ Use Case
────────────────┼────────┼───────────┼─────────────┼──────────────
SCHED_DEADLINE  │ dl     │ Highest   │ Runtime     │ Hard real-time
SCHED_FIFO      │ rt     │ 1-99      │ None        │ RT no preempt
SCHED_RR        │ rt     │ 1-99      │ 100ms       │ RT with sharing
SCHED_NORMAL    │ fair   │ nice±20   │ Proportional│ General purpose
SCHED_BATCH     │ fair   │ nice±20   │ Proportional│ Throughput
SCHED_IDLE      │ idle   │ Lowest    │ Proportional│ Background
```

---

## /proc Filesystem — Scheduling Information

```
Path                          │ Information              │ Chapter
──────────────────────────────┼──────────────────────────┼────────
/proc/<pid>/stat              │ State, priority, nice    │ 24, 27
/proc/<pid>/status            │ Context switches, state  │ 24, 27
/proc/<pid>/sched             │ CFS stats, vruntime      │ 24, 27
/proc/<pid>/schedstat         │ Run/wait time, switches  │ 24
/proc/<pid>/stack             │ Kernel stack trace       │ 27
/proc/<pid>/wchan             │ Wait channel (function)  │ 27
/proc/<pid>/io                │ I/O accounting           │ 24
/proc/<pid>/maps              │ Memory mappings          │ 04
/proc/sched_debug             │ All runqueues, all tasks │ 27
/proc/schedstat               │ Per-CPU scheduling stats │ 24
/proc/sys/kernel/sched_*      │ Scheduler tunables       │ 13, 25
```

---

## Tool Quick Reference

```
Tool          │ Purpose                       │ Chapter
──────────────┼───────────────────────────────┼────────
ps            │ Process listing               │ 27
top/htop      │ Real-time monitoring          │ 27
perf sched    │ Scheduler analysis            │ 28
perf stat     │ CPU/scheduling counters       │ 27
ftrace        │ Kernel function/event tracing │ 28
trace-cmd     │ ftrace frontend               │ 28
bpftrace      │ Custom tracing scripts        │ 28
strace        │ System call tracing           │ 27
pidstat       │ Per-process scheduler stats   │ 27
cyclictest    │ RT latency measurement        │ 33
chrt          │ Set/get scheduling policy     │ 23
taskset       │ Set CPU affinity              │ 16
numactl       │ NUMA policy control           │ 16
vmstat        │ Context switch rate           │ 25
```

---

## Related Study Materials

```
In the same parent directory:

ZZZ_LEARN_VISWA_DOCS/linux_kernel_drivers/
├── Device_Drivers_Book/      ← 36-chapter Linux Device Drivers guide
├── Memory_Management_Book/   ← 36-chapter Linux Memory Management guide
└── Process_Scheduling_Book/  ← THIS BOOK (35 chapters)

Together these three books provide comprehensive coverage of:
  - Process Management & Scheduling (this book)
  - Memory Management (virtual memory, page allocation, SLAB, etc.)
  - Device Drivers (char/block/network drivers, DMA, interrupts, etc.)
```

---

*Generated as a comprehensive Linux kernel study resource.*
*Covers Linux kernel versions 5.x through 6.x on x86_64 and ARM64.*
