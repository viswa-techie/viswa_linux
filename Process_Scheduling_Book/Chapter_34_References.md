# Chapter 34: Documentation and References

## Learning Goals
- Know where to find authoritative information on Linux scheduling
- Access kernel documentation, design papers, and community resources
- Build a continuous learning path for kernel scheduling mastery

---

## 34.1 Official Kernel Documentation

### In-Tree Documentation (Documentation/)

```
Key scheduling documentation in the kernel source tree:

Documentation/scheduler/
├── sched-design-CFS.rst      ← CFS design document by Ingo Molnár
├── sched-domains.rst          ← Scheduling domains explained
├── sched-nice-design.rst      ← Nice levels and weight ratios
├── sched-rt-group.rst         ← RT group scheduling
├── sched-deadline.rst         ← SCHED_DEADLINE configuration
├── sched-energy.rst           ← Energy Aware Scheduling (EAS)
├── sched-capacity.rst         ← CPU capacity and asymmetric systems
├── sched-bwc.rst              ← CFS bandwidth control
├── sched-stats.rst            ← Scheduler statistics
├── sched-arch.rst             ← Architecture integration
├── sched-pelt.rst             ← PELT load tracking
└── completion.rst             ← Completion variable API

Other relevant docs:
Documentation/admin-guide/
├── cgroup-v2.rst              ← cgroup v2 CPU controller
├── kernel-parameters.txt      ← Boot parameters (isolcpus, etc.)
└── pm/cpufreq.rst             ← CPU frequency scaling

Documentation/locking/
├── spinlocks.rst
├── mutex-design.rst
├── rt-mutex-design.rst        ← Priority inheritance mutexes
└── locktypes.rst              ← Lock type selection guide
```

### Reading Kernel Docs

```bash
# Build HTML documentation
cd linux-source/
make htmldocs
# Open Documentation/output/scheduler/sched-design-CFS.html

# Or read RST directly:
less Documentation/scheduler/sched-design-CFS.rst

# Online: https://www.kernel.org/doc/html/latest/scheduler/
```

---

## 34.2 Key Research Papers and Publications

### CFS and Scheduling Theory

```
1. "Completely Fair Scheduler" — Ingo Molnár (2007)
   Kernel commit: "sched: cfs-v2 scheduler code"
   Original design document in sched-design-CFS.rst
   Key insight: vruntime + red-black tree = O(log n) fairness

2. "An EEVDF CPU Scheduler for Linux" — Peter Zijlstra (2023)
   Replaces CFS pick-next with virtual-deadline-first
   Kernel commit series: "sched/eevdf"
   Improves latency fairness for mixed workloads

3. "Earliest Eligible Virtual Deadline First" — 
   Stoica & Abdel-Wahab (1995)
   Original EEVDF paper — theoretical foundation
   IEEE Real-Time Systems Symposium

4. "Inside the Linux 2.6 Completely Fair Scheduler" — 
   IBM DeveloperWorks, M. Tim Jones
   Practical CFS walkthrough with code examples
```

### Real-Time Scheduling

```
5. "SCHED_DEADLINE: Real-Time Scheduling in Linux"
   Lelli, Lipari, Scordino, Abeni
   Linux Plumbers Conference 2014
   EDF + CBS implementation details

6. "The Linux PREEMPT_RT Project" — 
   Thomas Gleixner, Steven Rostedt
   Multiple LPC/KS presentations (2004-present)
   History of making Linux fully preemptible

7. "Constant Bandwidth Server Revisited" — 
   Abeni & Buttazzo (2004)
   Foundation for SCHED_DEADLINE's bandwidth control

8. "A Practitioner's Guide to Real-Time Linux" — 
   John Googler, Julia Cartwright
   Embedded Linux Conference 2018
   Practical PREEMPT_RT deployment guide
```

### Load Balancing and Energy

```
9. "Energy Aware Scheduling" — ARM, Linaro
   https://www.kernel.org/doc/html/latest/scheduler/sched-energy.rst
   EAS design and implementation

10. "Per-Entity Load Tracking" (PELT)
    Morten Rasmussen, Ben Segall
    Kernel commits: "sched: add per-entity load tracking"
    Foundation for modern load balancing and EAS

11. "Scheduler Domains" — Nick Piggin
    Multi-level CPU topology awareness
    Documentation/scheduler/sched-domains.rst
```

---

## 34.3 Books

```
Essential reading for process management and scheduling:

1. "Understanding the Linux Kernel" — Bovet & Cesati
   O'Reilly, 3rd Edition (covers 2.6)
   Chapters 3 (Processes), 7 (Process Scheduling)
   Most detailed published reference on kernel internals

2. "Linux Kernel Development" — Robert Love
   3rd Edition, Addison-Wesley
   Chapters 3-7: Process management, scheduling, system calls
   Excellent balance of depth and readability

3. "Professional Linux Kernel Architecture" — Wolfgang Mauerer
   Wiley, covers 2.6.24
   Chapter 2 (Process Management), Chapter 14 (Kernel Arch)
   Deepest dive into scheduler internals

4. "Operating System Concepts" — Silberschatz, Galvin, Gagne
   "Dinosaur Book" — standard OS textbook
   Chapters 3-7: Processes, Threads, CPU Scheduling
   Essential theory foundation

5. "Operating Systems: Three Easy Pieces" — 
   Remzi & Andrea Arpaci-Dusseau (FREE online)
   http://pages.cs.wisc.edu/~remzi/OSTEP/
   Excellent OS fundamentals with Linux focus

6. "Linux Device Drivers" — Corbet, Rubini, Kroah-Hartman
   3rd Edition (FREE online: lwn.net/Kernel/LDD3/)
   Chapter 7 (Sleeping/Waking), concurrency chapters
```

---

## 34.4 Online Resources

### Websites

```
1. LWN.net (Linux Weekly News)
   https://lwn.net/
   Best source for kernel development news and articles
   Key scheduler articles:
     - "A look at the CFS scheduler" (2007)
     - "BFS vs CFS" controversy series
     - "EEVDF scheduler" coverage (2023)
     - "The deadline scheduler" series

2. kernel.org documentation
   https://www.kernel.org/doc/html/latest/
   Official, always current

3. Brendan Gregg's blog
   https://www.brendangregg.com/
   Performance analysis, perf tools, BPF
   Scheduling-related posts on perf sched

4. The Linux Foundation wiki
   https://wiki.linuxfoundation.org/
   Real-time Linux wiki (PREEMPT_RT)
```

### Source Code Browsing

```
1. Elixir Cross-Referencing
   https://elixir.bootlin.com/linux/latest/source
   Browse kernel source with cross-references
   Navigate: kernel/sched/core.c, fair.c, rt.c

2. GitHub mirror
   https://github.com/torvalds/linux
   Search, blame, history

3. git log for scheduler history
   git log --oneline --graph kernel/sched/
   git log --author="Peter Zijlstra" kernel/sched/
   git log --grep="CFS" --oneline
```

### Mailing Lists and Conferences

```
1. LKML (Linux Kernel Mailing List)
   https://lkml.org/
   Primary kernel development discussion

2. linux-rt-users@vger.kernel.org
   PREEMPT_RT user community

3. Linux Plumbers Conference (annual)
   https://lpc.events/
   Scheduler/RT micro-conference presentations

4. Kernel Summit / Maintainers Summit
   High-level scheduling design discussions

5. Embedded Linux Conference (annual)
   https://events.linuxfoundation.org/
   Practical embedded scheduling talks
```

---

## 34.5 Tools Documentation

```
1. perf wiki: https://perf.wiki.kernel.org/
   Complete perf tool documentation
   perf sched, perf stat, perf record

2. ftrace documentation:
   Documentation/trace/ftrace.rst (in kernel tree)
   Steven Rostedt's LWN articles on ftrace

3. trace-cmd: https://trace-cmd.org/
   Convenient frontend for ftrace

4. bpftrace: https://github.com/iovisor/bpftrace
   Reference guide for scheduling tracepoints

5. cyclictest: https://wiki.linuxfoundation.org/realtime/
   RT latency benchmarking tool

6. Perfetto (Android): https://perfetto.dev/docs/
   Modern Android/Linux trace visualization
```

---

## 34.6 Kernel Source Files — Quick Reference

```
Start here when investigating scheduler behavior:

  What happens when...          │ Look at
  ──────────────────────────────┼─────────────────────────────
  Process is created?            │ kernel/fork.c → kernel_clone()
  Task is scheduled?             │ kernel/sched/core.c → __schedule()
  CFS picks next task?           │ kernel/sched/fair.c → pick_next_entity()
  RT task is selected?           │ kernel/sched/rt.c → pick_next_task_rt()
  Timer tick fires?              │ kernel/sched/core.c → scheduler_tick()
  Task wakes up?                 │ kernel/sched/core.c → try_to_wake_up()
  Load is balanced?              │ kernel/sched/fair.c → load_balance()
  Signal is delivered?           │ kernel/signal.c → do_signal()
  Process exits?                 │ kernel/exit.c → do_exit()
  Context switch happens?        │ kernel/sched/core.c → context_switch()
  Page table is switched?        │ arch/arm64/mm/context.c → switch_mm()
  Registers are saved?           │ arch/arm64/kernel/entry.S → cpu_switch_to
  Priority inheritance occurs?   │ kernel/locking/rtmutex.c
  cgroup bandwidth enforced?     │ kernel/sched/fair.c → account_cfs_rq_runtime()
```

---

## 34.7 Recommended Learning Path

```
Beginner → Intermediate → Advanced:

1. OSTEP (Three Easy Pieces) — free online
   → Understand OS scheduling theory

2. Robert Love's "Linux Kernel Development"
   → Practical Linux kernel overview

3. This book — Process Scheduling chapters
   → Deep Linux-specific knowledge

4. Read kernel source (kernel/sched/)
   → Start with __schedule() in core.c
   → Follow code paths from this book

5. Experiment with perf sched, ftrace, bpftrace
   → Hands-on understanding of scheduling behavior

6. Contribute to kernel (fix bugs, improve docs)
   → Subscribe to LKML, review patches in kernel/sched/

7. LWN.net articles on scheduling
   → Stay current with scheduler evolution
```

---

*Next: [Chapter 35 — Interview Preparation](Chapter_35_Interview_Preparation.md)*
