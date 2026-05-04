# Linux Interrupts & Concurrency — Master Index

## Complete Study Guide: 36 Chapters

> **Scope**: From hardware interrupt signals to kernel synchronization internals.
> **Target**: Linux kernel 5.x-6.x, x86_64 and ARM64 architectures.
> **Level**: University course + kernel developer handbook.

---

## Part I: Interrupt Foundations (Chapters 1-4)

| # | Chapter | Key Topics |
|---|---------|------------|
| 1 | [Foundations](Chapter_01_Foundations.md) | What is an interrupt, polling vs interrupt-driven, NAPI hybrid, concurrency vs parallelism, terminology |
| 2 | [History and Evolution](Chapter_02_History_and_Evolution.md) | UNIVAC to modern Linux, PIC→APIC→GIC, BH→SoftIRQ→Tasklet→Workqueue→Threaded IRQ |
| 3 | [Hardware Architecture](Chapter_03_Hardware_Architecture.md) | x86 IDT/APIC, ARM64 GIC (v1-v4), interrupt signals, MSI/MSI-X, masking hierarchy |
| 4 | [Types of Interrupts](Chapter_04_Types_of_Interrupts.md) | Hardware IRQs, software interrupts, exceptions, NMI, IPI, timer interrupts |

## Part II: Interrupt Handling (Chapters 5-8)

| # | Chapter | Key Topics |
|---|---------|------------|
| 5 | [Interrupt Handling Flow](Chapter_05_Interrupt_Handling_Flow.md) | Hardware→kernel complete flow, x86 and ARM64 paths, timing breakdown |
| 6 | [Linux Interrupt Handling](Chapter_06_Linux_Interrupt_Handling.md) | irq_desc, irq_chip, irqaction, request_irq() internals, /proc/interrupts |
| 7 | [Top Half and Bottom Half](Chapter_07_Top_Bottom_Half.md) | Why split, top half rules, bottom half mechanisms, decision flowchart |
| 8 | [Deferred Work Mechanisms](Chapter_08_Deferred_Work.md) | SoftIRQs (10 types), Tasklets, Workqueues, ksoftirqd/kworker |

## Part III: Advanced Interrupt Topics (Chapters 9-12)

| # | Chapter | Key Topics |
|---|---------|------------|
| 9 | [Interrupt Threading](Chapter_09_Interrupt_Threading.md) | request_threaded_irq(), IRQF_ONESHOT, PREEMPT_RT force-threading, design patterns |
| 10 | [Interrupt Affinity](Chapter_10_Interrupt_Affinity.md) | smp_affinity, irqbalance, RSS/RPS, CPU isolation for RT |
| 11 | [Inter-Processor Interrupts](Chapter_11_IPI.md) | IPI types, TLB shootdown, smp_call_function, PREEMPT_RT impact |
| 12 | [Timer Interrupts](Chapter_12_Timer_Interrupts.md) | Timer hardware, CONFIG_HZ, timer_list, hrtimer, NO_HZ modes |

## Part IV: Concurrency Foundations (Chapters 13-16)

| # | Chapter | Key Topics |
|---|---------|------------|
| 13 | [Concurrency Concepts](Chapter_13_Concurrency_Concepts.md) | Four execution contexts, concurrency scenarios, lock selection matrix |
| 14 | [Concurrency Problems](Chapter_14_Concurrency_Problems.md) | Race conditions, deadlocks, livelocks, starvation, priority inversion |
| 15 | [Synchronization Mechanisms Overview](Chapter_15_Synchronization_Mechanisms.md) | All primitives survey, comparison table, decision tree |
| 16 | [Spinlocks](Chapter_16_Spinlocks.md) | qspinlock internals, API variants, raw_spinlock_t vs spinlock_t, PREEMPT_RT |

## Part V: Synchronization Primitives (Chapters 17-20)

| # | Chapter | Key Topics |
|---|---------|------------|
| 17 | [Mutexes](Chapter_17_Mutexes.md) | Three-path acquisition, optimistic spinning, rt_mutex, priority inheritance |
| 18 | [Semaphores](Chapter_18_Semaphores.md) | Counting/binary semaphores, rw_semaphore, mutex vs semaphore |
| 19 | [Atomic Operations](Chapter_19_Atomic_Operations.md) | atomic_t, atomic64_t, bitwise ops, refcount_t, lock-free patterns |
| 20 | [Memory Barriers](Chapter_20_Memory_Barriers.md) | CPU reordering, smp_mb/wmb/rmb, READ_ONCE/WRITE_ONCE, acquire/release |

## Part VI: Advanced Synchronization (Chapters 21-24)

| # | Chapter | Key Topics |
|---|---------|------------|
| 21 | [Locking in IRQ Context](Chapter_21_Locking_IRQ_Context.md) | IRQ lock selection matrix, irqsave variants, NMI locking, PREEMPT_RT |
| 22 | [Per-CPU Data](Chapter_22_Per_CPU_Data.md) | Static/dynamic per-CPU, preemption rules, kernel usage examples |
| 23 | [RCU (Read-Copy-Update)](Chapter_23_RCU.md) | Grace periods, rcu_read_lock, call_rcu, RCU lists, SRCU |
| 24 | [Concurrency in Drivers](Chapter_24_Concurrency_Drivers.md) | Driver locking patterns, char/net driver examples, probe/remove races |

## Part VII: Practical Application (Chapters 25-28)

| # | Chapter | Key Topics |
|---|---------|------------|
| 25 | [Interrupt Handling in Drivers](Chapter_25_Interrupt_Handling_Drivers.md) | IRQ registration patterns, shared IRQs, DMA+interrupt, devm APIs |
| 26 | [Interrupt Debugging](Chapter_26_Interrupt_Debugging.md) | /proc/interrupts, lockdep, KASAN, KCSAN, IRQ storms, lockups |
| 27 | [Kernel Tracing](Chapter_27_Kernel_Tracing.md) | ftrace, trace-cmd, perf, bpftrace, cyclictest, irqsoff tracer |
| 28 | [Kernel Source Code](Chapter_28_Kernel_Source.md) | Source tree map, key files, code path walkthrough, navigation tools |

## Part VIII: Optimization and Reference (Chapters 29-32)

| # | Chapter | Key Topics |
|---|---------|------------|
| 29 | [Performance Optimization](Chapter_29_Performance_Optimization.md) | Interrupt coalescing, NAPI, lock optimization, NUMA affinity |
| 30 | [Important Diagrams](Chapter_30_Diagrams.md) | Complete IRQ flow, context hierarchy, lock decision tree, RCU timeline |
| 31 | [Definitions and Glossary](Chapter_31_Glossary.md) | A-Z terminology with definitions and API→header mapping |
| 32 | [OS Comparison](Chapter_32_OS_Comparison.md) | Linux vs Windows vs macOS vs QNX vs FreeRTOS |

## Part IX: Specialized Topics (Chapters 33-36)

| # | Chapter | Key Topics |
|---|---------|------------|
| 33 | [Real-Time Linux (PREEMPT_RT)](Chapter_33_Real_Time_Linux.md) | RT transformations, raw_spinlock, CPU isolation, cyclictest, tuning |
| 34 | [Embedded Interrupt Design](Chapter_34_Embedded_Interrupt_Design.md) | Device Tree IRQs, power-aware design, SoC architecture, latency budget |
| 35 | [Documentation and References](Chapter_35_Documentation_References.md) | Books, kernel docs, LWN articles, ARM/Intel manuals, study path |
| 36 | [Interview Preparation](Chapter_36_Interview_Preparation.md) | Top 50 questions with model answers, design questions, self-assessment |

---

## Reading Paths

### Path 1: Quick Overview (4 chapters)
Chapters 1 → 7 → 15 → 30

### Path 2: Driver Developer (12 chapters)
Chapters 1 → 5 → 6 → 7 → 8 → 9 → 21 → 24 → 25 → 26 → 27 → 29

### Path 3: Concurrency Deep Dive (10 chapters)
Chapters 13 → 14 → 15 → 16 → 17 → 19 → 20 → 22 → 23 → 29

### Path 4: Real-Time / Embedded (8 chapters)
Chapters 9 → 10 → 12 → 21 → 27 → 33 → 34 → 29

### Path 5: Interview Prep (8 chapters)
Chapters 7 → 15 → 20 → 21 → 23 → 30 → 32 → 36

### Path 6: Full Course (all 36 chapters)
Read sequentially: 1 → 36

---

## Quick API Reference

| API | Header | Chapter |
|-----|--------|---------|
| `request_irq()` | `linux/interrupt.h` | 6, 25 |
| `request_threaded_irq()` | `linux/interrupt.h` | 9, 25 |
| `spin_lock_irqsave()` | `linux/spinlock.h` | 16, 21 |
| `mutex_lock()` | `linux/mutex.h` | 17 |
| `atomic_inc()` | `linux/atomic.h` | 19 |
| `smp_mb()` / `smp_wmb()` / `smp_rmb()` | `asm/barrier.h` | 20 |
| `rcu_read_lock()` | `linux/rcupdate.h` | 23 |
| `DEFINE_PER_CPU()` | `linux/percpu.h` | 22 |
| `schedule_work()` | `linux/workqueue.h` | 8 |
| `hrtimer_start()` | `linux/hrtimer.h` | 12 |
| `refcount_inc()` | `linux/refcount.h` | 19 |
| `complete()` | `linux/completion.h` | 25 |

---

## Key Decision Flowcharts

**Choosing a Synchronization Primitive** → Chapter 15, 30
**Choosing a Bottom Half Mechanism** → Chapter 7, 8
**Choosing the Right spin_lock Variant** → Chapter 21
**Driver Locking Design** → Chapter 24

---

*Total: 36 chapters covering the complete Linux interrupts and concurrency domain.*
