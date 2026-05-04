# Chapter 35: Documentation and References

## Learning Goals
- Know the essential kernel documentation for interrupts and concurrency
- Find authoritative references for each subsystem
- Use books, papers, and online resources effectively
- Build a personal reference library for kernel development

---

## 35.1 Kernel In-Tree Documentation

### Primary References

```
Documentation/                          Location in kernel tree
──────────────────────────────────┬─────────────────────────────────
Interrupts:                       │
  core-api/genericirq.rst         │ IRQ subsystem architecture
  core-api/irq/                   │ IRQ domain, chip concepts
  driver-api/basics.rst           │ Driver IRQ registration
                                  │
Memory Barriers:                  │
  memory-barriers.txt             │ THE definitive reference (1000+ lines)
                                  │ Must-read for any kernel developer
                                  │
Locking:                          │
  locking/                        │
  locking/locktypes.rst           │ spinlock_t vs raw_spinlock_t
  locking/lockdep-design.rst      │ Lock validator design
  locking/mutex-design.rst        │ Mutex implementation details
  locking/rt-mutex-design.rst     │ RT mutex and priority inheritance
  locking/spinlocks.rst           │ Spinlock rules and variants
                                  │
RCU:                              │
  RCU/                            │ Extensive RCU documentation
  RCU/whatisRCU.rst               │ RCU overview
  RCU/rcu_dereference.rst         │ Pointer access rules
  RCU/listRCU.rst                 │ RCU-protected lists
  RCU/Design/                     │ Detailed design documents
  RCU/rcubarrier.rst              │ rcu_barrier usage
                                  │
Real-Time:                        │
  locking/locktypes.rst           │ RT lock type conversions
  admin-guide/kernel-parameters.txt│ isolcpus, nohz_full, etc.
                                  │
Workqueues:                       │
  core-api/workqueue.rst          │ Workqueue API and design
                                  │
Tracing:                          │
  trace/ftrace.rst                │ Ftrace usage guide
  trace/events.rst                │ Trace event subsystem
  trace/kprobes.rst               │ Kprobes/kretprobes
                                  │
Architecture:                     │
  arm64/                          │ ARM64 specifics
  x86/                            │ x86 specifics
```

### How to Read Kernel Docs

```bash
# Build HTML documentation:
cd linux/
make htmldocs
# Open Documentation/output/index.html

# Or read raw .rst files:
less Documentation/memory-barriers.txt
less Documentation/RCU/whatisRCU.rst
less Documentation/locking/spinlocks.rst

# Online at kernel.org:
# https://docs.kernel.org/
# https://docs.kernel.org/locking/
# https://docs.kernel.org/RCU/
```

---

## 35.2 Essential Books

### Kernel Development

```
1. "Linux Kernel Development" — Robert Love (3rd ed., 2010)
   - Best introduction to kernel internals
   - Chapters on interrupts, bottom halves, synchronization
   - Accessible but thorough

2. "Linux Device Drivers" — Corbet, Rubini, Kroah-Hartman (3rd ed., 2005)
   - Definitive driver development guide (free online: lwn.net/Kernel/LDD3/)
   - Chapters 10-11: interrupts and concurrency
   - Still relevant despite age

3. "Understanding the Linux Kernel" — Bovet, Cesati (3rd ed., 2005)
   - Deep dive into kernel internals
   - Detailed interrupt handling, timing, synchronization
   - Best for understanding implementation

4. "Linux Kernel in a Nutshell" — Greg Kroah-Hartman (2006)
   - Building, configuring, installing kernels
   - Free online

5. "Professional Linux Kernel Architecture" — Mauerer (2008)
   - Comprehensive kernel architecture
   - Excellent diagrams and explanations
```

### Concurrency and Synchronization

```
6. "Is Parallel Programming Hard?" — Paul E. McKenney (free online)
   - By the RCU maintainer
   - Best resource for RCU, memory barriers, concurrency
   - https://mirrors.edge.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html

7. "The Art of Multiprocessor Programming" — Herlihy, Shavit
   - Theory of concurrent data structures
   - Lock-free algorithms, linearizability

8. "C++ Concurrency in Action" — Anthony Williams
   - Memory model, atomics (relevant to kernel concepts)
```

### Real-Time and Embedded

```
9. "Building Embedded Linux Systems" — Yaghmour et al.
   - Practical embedded Linux guide

10. "Embedded Linux Primer" — Christopher Hallinan
    - Hardware, bootloaders, kernel, drivers

11. "Real-Time Linux with PREEMPT_RT" — various LWN.net articles
    - See LWN references below
```

---

## 35.3 Key Online Resources

### LWN.net Articles (Linux Weekly News)

```
LWN.net — the most authoritative ongoing Linux kernel coverage.

Essential articles:
  "A guide to kernel locking"              lwn.net/Articles/10318/
  "What is RCU, Fundamentally?"            lwn.net/Articles/262464/
  "The PREEMPT_RT patchset"                lwn.net/Articles/146861/
  "Threaded interrupts"                    lwn.net/Articles/302043/
  "Interrupt threads"                       lwn.net/Articles/380937/
  "Softirqs, tasklets, bottom halves"      lwn.net/Articles/17802/
  "Memory barriers"                         lwn.net/Articles/573436/
  "The qspinlock"                          lwn.net/Articles/590243/
  "Workqueue redesign"                      lwn.net/Articles/403891/
  "Per-CPU variables"                       lwn.net/Articles/258238/
  "Lock ordering"                           lwn.net/Articles/185666/
  "Priority inheritance in the kernel"      lwn.net/Articles/178253/
  "Deadline scheduling"                     lwn.net/Articles/743740/

Subscription recommended for kernel developers.
```

### Source Code Browsers

```
Elixir (Bootlin):   https://elixir.bootlin.com/linux/latest/source
  - Cross-referenced source code
  - Click any symbol to see definition + references
  - Multiple kernel versions

Linux Cross Reference: https://lxr.sourceforge.io/
  - Alternative source browser

cregit:             https://cregit.linuxsources.org/
  - Shows authorship per line (git blame at scale)
```

### Mailing Lists and Community

```
LKML:               https://lkml.org/
  - Linux Kernel Mailing List (primary development)

linux-rt-users:     Subscribe via vger.kernel.org
  - PREEMPT_RT discussion and support

kernelnewbies:      https://kernelnewbies.org/
  - Beginner-friendly kernel development info
  - Excellent changelog summaries per release

kernel.org:         https://www.kernel.org/
  - Official kernel releases and documentation
```

---

## 35.4 Conference Talks and Videos

```
Essential presentations:

Linux Foundation / Open Source Summit:
  - "Understanding the Real-Time Linux Kernel" — Steven Rostedt
  - "IRQ Subsystem Overview" — Thomas Gleixner
  - "RCU Usage In the Linux Kernel" — Paul McKenney

Linux Plumbers Conference:
  - Deep technical discussions on subsystem design
  - Archives: linuxplumbersconf.org

Kernel Recipes:
  - Annual European kernel conference
  - Excellent technical depth
  - Archives: kernel-recipes.org

YouTube channels:
  - The Linux Foundation
  - LWN.net presentations
  - FOSDEM (embedded/kernel tracks)
```

---

## 35.5 ARM and x86 Architecture References

```
ARM:
  "ARM Architecture Reference Manual" (ARMv8-A)
    - Chapter D1: AArch64 Exception Model
    - Chapter D8: Generic Interrupt Controller
    - Available from developer.arm.com

  "ARM GIC Architecture Specification" (v3/v4)
    - IHI 0069: GICv3/v4 architecture
    - Detailed distributor, redistributor, CPU interface
    - developer.arm.com/documentation/ihi0069

  "ARM Cortex-A Series Programmer's Guide"
    - Practical guide for A-series interrupt handling

x86:
  "Intel 64 and IA-32 Architectures Software Developer's Manual"
    - Volume 3A: System Programming Guide
    - Chapter 6: Interrupt and Exception Handling
    - Chapter 10: APIC

  "AMD64 Architecture Programmer's Manual"
    - Volume 2: System Programming
    - Chapter 8: Exceptions and Interrupts
```

---

## 35.6 Tools Documentation

```
ftrace:
  Documentation/trace/ftrace.rst           (in-tree)
  https://docs.kernel.org/trace/ftrace.html

perf:
  https://perf.wiki.kernel.org/
  man perf-record, perf-report, perf-lock

trace-cmd:
  https://trace-cmd.org/
  man trace-cmd-record

bpftrace:
  https://github.com/iovisor/bpftrace
  Reference guide and tutorials

cyclictest:
  Part of rt-tests package
  https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cyclictest

lockdep:
  Documentation/locking/lockdep-design.rst
  https://docs.kernel.org/locking/lockdep-design.html
```

---

## 35.7 Quick Reference Card

```
Task                           │ Where to Look First
───────────────────────────────┼────────────────────────────────────
Register an IRQ                │ include/linux/interrupt.h
Spinlock API                   │ include/linux/spinlock.h
Mutex API                      │ include/linux/mutex.h
RCU API                        │ include/linux/rcupdate.h
Workqueue API                  │ include/linux/workqueue.h
Per-CPU API                    │ include/linux/percpu.h
Atomic API                     │ include/linux/atomic.h
Memory barriers                │ Documentation/memory-barriers.txt
Lock types on RT               │ Documentation/locking/locktypes.rst
Device Tree interrupts         │ Documentation/devicetree/bindings/
IRQ domain                     │ Documentation/core-api/irq/
Driver examples                │ drivers/iio/ (clean, well-documented)
```

---

## 35.8 Recommended Study Path

```
Beginner → Expert progression:

Week 1-2: Foundations
  □ Read "Linux Kernel Development" (Love) Ch. 7-8
  □ Read Documentation/locking/spinlocks.rst
  □ Write a simple char driver with IRQ

Week 3-4: Interrupts Deep Dive
  □ Read LDD3 chapters 10-11 (free online)
  □ Study kernel/irq/manage.c source
  □ Write a driver with threaded IRQ

Week 5-6: Concurrency Mastery
  □ Read Documentation/memory-barriers.txt
  □ Read "Is Parallel Programming Hard?" (McKenney) Ch. 1-9
  □ Study RCU: Documentation/RCU/whatisRCU.rst

Week 7-8: Advanced Topics
  □ Study kernel/locking/qspinlock.c
  □ Read PREEMPT_RT documentation
  □ Run cyclictest, ftrace irqsoff

Week 9+: Kernel Contribution
  □ Subscribe to LKML / linux-rt-users
  □ Find a bug or improvement in drivers/
  □ Submit a patch following Documentation/process/
```

---

## Interview Questions

1. **What are the top 3 kernel documentation files every driver developer should read?**
2. **Where would you look to understand the RCU grace period implementation?**
3. **What book is recommended for understanding Linux concurrency primitives?**
4. **How do you build kernel HTML documentation?**
5. **What is LWN.net? Why is it valuable for kernel developers?**

---

## Summary

- kernel Documentation/: memory-barriers.txt, locking/, RCU/ are must-reads
- Books: Love (introduction), LDD3 (drivers), McKenney (concurrency + RCU)
- LWN.net: best ongoing source for kernel development articles
- elixir.bootlin.com: cross-referenced kernel source browser
- ARM/Intel architecture manuals: essential for interrupt controller details
- Study path: foundations → interrupts → concurrency → RT → contribution

---

*Next: [Chapter 36 — Interview Preparation](Chapter_36_Interview_Preparation.md)*
