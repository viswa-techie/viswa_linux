# Chapter 30: Important Diagrams

## Learning Goals
- Consolidate key visual diagrams for process management and scheduling
- Use these as quick reference for interviews and debugging
- Understand system architecture through visual representation

---

## 30.1 Process State Transition Diagram

```
                          fork()
                            │
                            ▼
                     ┌─────────────┐
                     │   CREATED    │
                     │  (NEW TASK)  │
                     └──────┬──────┘
                            │ wake_up_new_task()
                            ▼
            ┌──────────────────────────────┐
     ┌─────→│    TASK_RUNNING (RUNNABLE)    │◄────────────┐
     │      │    On run queue, waiting      │             │
     │      └──────┬───────────────────────┘             │
     │             │ schedule() picks this task           │
     │             ▼                                      │
     │      ┌──────────────────────────────┐             │
     │      │    TASK_RUNNING (ON CPU)      │             │
     │      │    Actually executing          │             │
     │      └─┬──────────┬──────────┬──────┘             │
     │        │          │          │                      │
     │   sleep/wait  SIGSTOP    exit()                    │
     │        │      /ptrace      │                       │
     │        ▼          ▼        ▼                       │
     │  ┌──────────┐ ┌────────┐ ┌──────────┐             │
     │  │SLEEPING  │ │STOPPED │ │EXIT_ZOMBIE│             │
     │  │(S or D)  │ │  (T)   │ │   (Z)    │             │
     │  └────┬─────┘ └───┬────┘ └────┬─────┘             │
     │       │           │           │                     │
     │  wake_up()   SIGCONT      wait() by parent         │
     │       │           │           │                     │
     │       └───────────┘           ▼                     │
     │            │           ┌──────────┐                │
     └────────────┘           │EXIT_DEAD │                │
                              │ (freed)  │                │
                              └──────────┘

  S = TASK_INTERRUPTIBLE  (woken by signal + event)
  D = TASK_UNINTERRUPTIBLE (woken by event only)
  T = __TASK_STOPPED (SIGSTOP) or __TASK_TRACED (ptrace)
  Z = EXIT_ZOMBIE (exited, waiting for parent wait())
```

---

## 30.2 Scheduling Class Hierarchy

```
  ┌─────────────────────────────────────────────────────────┐
  │                    pick_next_task()                       │
  │  Walks class chain from highest to lowest priority       │
  └─────────────────────────────────────────────────────────┘
                            │
         ┌──────────────────┼──────────────────┐
         ▼                  ▼                  ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  STOP CLASS  │  │   DL CLASS   │  │   RT CLASS   │
  │  (internal)  │  │ SCHED_DEADLINE│ │ FIFO / RR    │
  │  migration/  │  │ EDF + CBS    │  │ prio 1-99    │
  │  stopper     │  │ RB tree by   │  │ O(1) bitmap  │
  │              │  │ abs. deadline│  │ + FIFO lists │
  │  prio: max   │  │  prio: -1    │  │  prio: 0-98  │
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         │                  │                  │
         │     If empty     │    If empty      │    If empty
         └────────┐  ┌──────┘  ┌───────────────┘
                  ▼  ▼         ▼
           ┌──────────────┐  ┌──────────────┐
           │  FAIR CLASS  │  │  IDLE CLASS  │
           │ SCHED_NORMAL │  │ (swapper/N)  │
           │ SCHED_BATCH  │  │ Runs when    │
           │ SCHED_IDLE   │  │ nothing else │
           │              │  │ to do        │
           │ CFS RB tree  │  │              │
           │ by vruntime  │  │              │
           │ prio: 100-139│  │ prio: MAX    │
           └──────────────┘  └──────────────┘
```

---

## 30.3 Per-CPU Run Queue Architecture

```
  ┌─────────── CPU 0 ──────────┐    ┌─────────── CPU 1 ──────────┐
  │         struct rq            │    │         struct rq            │
  │                              │    │                              │
  │  ┌── dl_rq ────────────┐   │    │  ┌── dl_rq ────────────┐   │
  │  │  RB tree (deadline)  │   │    │  │  RB tree (deadline)  │   │
  │  └─────────────────────┘   │    │  └─────────────────────┘   │
  │                              │    │                              │
  │  ┌── rt_rq ────────────┐   │    │  ┌── rt_rq ────────────┐   │
  │  │  bitmap: [0110...0]  │   │    │  │  bitmap: [0001...0]  │   │
  │  │  queue[1]: T1→T2     │   │    │  │  queue[3]: T5        │   │
  │  │  queue[2]: T3        │   │    │  │                      │   │
  │  └─────────────────────┘   │    │  └─────────────────────┘   │
  │                              │    │                              │
  │  ┌── cfs_rq ───────────┐   │    │  ┌── cfs_rq ───────────┐   │
  │  │       [vr=50]        │   │    │  │       [vr=45]        │   │
  │  │      /      \        │   │    │  │      /      \        │   │
  │  │   [30]     [70]     │   │    │  │   [35]     [60]     │   │
  │  │   / \       / \     │   │    │  │   / \                │   │
  │  │ [20] [40] [60] [80] │   │    │  │ [25] [40]           │   │
  │  │  ↑                   │   │    │  │  ↑                   │   │
  │  │  leftmost (next)     │   │    │  │  leftmost (next)     │   │
  │  └─────────────────────┘   │    │  └─────────────────────┘   │
  │                              │    │                              │
  │  curr: Task_A               │    │  curr: Task_D               │
  │  idle: swapper/0             │    │  idle: swapper/1             │
  │  nr_running: 9               │    │  nr_running: 5               │
  └──────────────────────────────┘    └──────────────────────────────┘
             │                                     │
             └──────── Load Balancing ─────────────┘
                    (periodic / idle / wake)
```

---

## 30.4 task_struct Key Fields Map

```
  struct task_struct (~6-8 KB)
  ┌─────────────────────────────────────────────────────────┐
  │ IDENTITY & STATE                                         │
  │   pid, tgid, comm[16], __state, flags, exit_state       │
  │   exit_code, exit_signal                                 │
  ├─────────────────────────────────────────────────────────┤
  │ SCHEDULING                                               │
  │   prio, static_prio, normal_prio, rt_priority           │
  │   policy (SCHED_NORMAL/FIFO/RR/DEADLINE/BATCH/IDLE)     │
  │   sched_class → fair/rt/dl/stop/idle_sched_class        │
  │   se (sched_entity): vruntime, load, run_node, avg      │
  │   rt (sched_rt_entity): run_list, time_slice            │
  │   dl (sched_dl_entity): runtime, deadline, period       │
  │   cpus_mask, nr_cpus_allowed                             │
  │   on_cpu, on_rq, wake_cpu                                │
  ├─────────────────────────────────────────────────────────┤
  │ MEMORY                                                   │
  │   mm → mm_struct (pgd, mmap, total_vm, rss)             │
  │   active_mm (for kernel threads)                         │
  ├─────────────────────────────────────────────────────────┤
  │ FILESYSTEM                                               │
  │   fs → fs_struct (root, pwd)                             │
  │   files → files_struct (fd_array[], fdt)                │
  ├─────────────────────────────────────────────────────────┤
  │ SIGNALS                                                  │
  │   signal → signal_struct (shared: pending, leaders)      │
  │   sighand → sighand_struct (action[64])                 │
  │   blocked, real_blocked (signal masks)                   │
  │   pending (per-thread pending signals)                   │
  ├─────────────────────────────────────────────────────────┤
  │ PROCESS HIERARCHY                                        │
  │   real_parent, parent                                    │
  │   children (list_head), sibling (list_head)              │
  │   group_leader                                           │
  ├─────────────────────────────────────────────────────────┤
  │ TIMING & ACCOUNTING                                      │
  │   utime, stime, start_time                               │
  │   nvcsw, nivcsw                                          │
  │   ioac (task_io_accounting)                              │
  ├─────────────────────────────────────────────────────────┤
  │ ARCHITECTURE SPECIFIC                                    │
  │   thread (thread_struct): sp, pc, regs, FPU state       │
  │   stack (kernel stack pointer)                           │
  │   thread_info (flags: TIF_NEED_RESCHED, TIF_SIGPENDING) │
  └─────────────────────────────────────────────────────────┘
```

---

## 30.5 CPU Topology and Scheduling Domains

```
  ┌───────────────── NUMA Node 0 ─────────────────┐
  │                                                  │
  │  ┌──── MC Domain (Package/Die) ────┐            │
  │  │                                  │            │
  │  │  ┌─ SMT ─┐    ┌─ SMT ─┐        │            │
  │  │  │ C0  C1│    │ C2  C3│        │            │
  │  │  │HT pair│    │HT pair│        │            │
  │  │  └───────┘    └───────┘        │            │
  │  │      ↕ shared L1/L2            │            │
  │  │      ↕ shared L3 cache          │            │
  │  └──────────────────────────────────┘            │
  │           ↕ shared local DRAM                     │
  └──────────────────────────────────────────────────┘
          ↕                                    ↕
    QPI / UPI interconnect (~150ns remote access)
          ↕                                    ↕
  ┌───────────────── NUMA Node 1 ─────────────────┐
  │  ┌──── MC Domain ─────────────────┐            │
  │  │  ┌─ SMT ─┐    ┌─ SMT ─┐      │            │
  │  │  │ C4  C5│    │ C6  C7│      │            │
  │  │  └───────┘    └───────┘      │            │
  │  └──────────────────────────────┘            │
  └──────────────────────────────────────────────┘

  Scheduling Domain Hierarchy:
  
    SD_SMT (level 0):  {C0,C1}  {C2,C3}  {C4,C5}  {C6,C7}
      Balance interval: 1ms, share CPU capacity
      
    SD_MC (level 1):   {C0,C1,C2,C3}  {C4,C5,C6,C7}
      Balance interval: 4ms, share L3 cache
      
    SD_NUMA (level 2):  {C0-C3, C4-C7}
      Balance interval: 64ms, different memory speeds
```

---

## 30.6 CFS Virtual Runtime Model

```
  vruntime accumulation for different nice values:
  
  Time →   0    5ms   10ms  15ms  20ms  25ms  30ms
  
  nice -5 (weight 3121):
  vruntime: 0   1.6   3.3   4.9   6.6   8.2   9.9
  ─────────────────────────────────────────────────── slow growth → more CPU
  
  nice 0 (weight 1024):
  vruntime: 0   5.0  10.0  15.0  20.0  25.0  30.0
  ─────────────────────────────────────────────────── normal growth
  
  nice +5 (weight 335):
  vruntime: 0  15.3  30.6  45.9  61.2  76.5  91.8
  ─────────────────────────────────────────────────── fast growth → less CPU
  
  
  RB Tree picks leftmost (smallest vruntime):
  nice -5 task always has smallest → runs most
  
  Over 30ms period with all 3 tasks:
    nice -5: ~20ms CPU (67%)
    nice  0: ~7ms CPU  (23%)
    nice +5: ~3ms CPU  (10%)
```

---

## 30.7 Context Switch Hardware View

```
  ┌─────────────── CPU Core ────────────────────┐
  │                                               │
  │  Task A Running          Task B Waiting       │
  │  ┌──────────────┐       (state saved in       │
  │  │ Registers:   │        kernel stack)         │
  │  │  PC, SP, LR  │                              │
  │  │  x0-x30      │                              │
  │  │  FPU q0-q31  │                              │
  │  └──────────────┘                              │
  │  ┌──────────────┐                              │
  │  │ Page table:  │                              │
  │  │  TTBR0 → A's│                              │
  │  │  pgd        │                              │
  │  └──────────────┘                              │
  │  ┌──────────────┐                              │
  │  │ TLB:        │                              │
  │  │  A's entries│                              │
  │  └──────────────┘                              │
  │                                               │
  │ ═══ schedule() → context_switch() ═══════════ │
  │                                               │
  │  Task B Running          Task A Saved          │
  │  ┌──────────────┐       ┌──────────────┐      │
  │  │ Registers:   │       │ Saved in     │      │
  │  │  B's values  │       │ A's kernel   │      │
  │  │  restored    │       │ stack:       │      │
  │  │  from stack  │       │ x19-x28,fp  │      │
  │  └──────────────┘       │ lr,sp        │      │
  │  ┌──────────────┐       └──────────────┘      │
  │  │ Page table:  │                              │
  │  │  TTBR0 → B's│                              │
  │  │  pgd        │                              │
  │  └──────────────┘                              │
  │  ┌──────────────┐                              │
  │  │ TLB:        │                              │
  │  │  B's ASID   │  (A's entries still valid    │
  │  │  active     │   with different ASID tag)    │
  │  └──────────────┘                              │
  └───────────────────────────────────────────────┘
```

---

## 30.8 Priority Map

```
  Kernel      User-visible        Policy          Effect
  priority    priority
  ─────────────────────────────────────────────────────────
  -1          (internal)          SCHED_DEADLINE   │ H
   0          RT priority 99     SCHED_FIFO/RR    │ I
   1          RT priority 98     SCHED_FIFO/RR    │ G
   ...        ...                ...               │ H
  98          RT priority 1      SCHED_FIFO/RR    │ E
  99          (unused)           (gap)             │ S
  ─────────────────────────────────────────────────│ T
  100         nice -20           SCHED_NORMAL     │
  101         nice -19           SCHED_NORMAL     │ P
  ...         ...                ...               │ R
  120         nice 0 (DEFAULT)   SCHED_NORMAL     │ I
  ...         ...                ...               │ O
  139         nice +19           SCHED_NORMAL     │ R
  ─────────────────────────────────────────────────│ I
  MAX         (internal)         SCHED_IDLE class  │ T
                                 (idle/swapper)    │ Y
  ─────────────────────────────────────────────────
              ← Lower number = higher priority →   L
                                                    O
                                                    W
```

---

## 30.9 Load Balancing Flow

```
  ┌──── CPU 0 ──────┐   ┌──── CPU 1 ──────┐   ┌──── CPU 2 ──────┐
  │ load: HIGH       │   │ load: LOW        │   │ load: IDLE       │
  │ T1 T2 T3 T4 T5  │   │ T6              │   │ (no tasks)       │
  └────────┬─────────┘   └────────┬─────────┘   └────────┬─────────┘
           │                      │                       │
           └──────────────────────┼───────────────────────┘
                                  │
                     ┌────────────┴────────────┐
                     │  trigger_load_balance() │
                     │  (SCHED_SOFTIRQ)        │
                     └────────────┬────────────┘
                                  │
                     ┌────────────┴────────────┐
                     │  find_busiest_group()   │
                     │  → CPU 0's group        │
                     └────────────┬────────────┘
                                  │
                     ┌────────────┴────────────┐
                     │  find_busiest_queue()    │
                     │  → CPU 0 (5 tasks)      │
                     └────────────┬────────────┘
                                  │
                     ┌────────────┴────────────┐
                     │  detach_tasks(CPU 0)     │
                     │  Select T4 (cache-cold)  │
                     │  Select T5 (cache-cold)  │
                     └────────────┬────────────┘
                                  │
                 ┌────────────────┼────────────────┐
                 ▼                                  ▼
   ┌──── CPU 1 ──────┐               ┌──── CPU 2 ──────┐
   │ T6, T4          │               │ T5              │
   │ load: BALANCED   │               │ load: BALANCED   │
   └─────────────────┘               └─────────────────┘
```

---

*Next: [Chapter 31 — Glossary and Definitions](Chapter_31_Glossary.md)*
