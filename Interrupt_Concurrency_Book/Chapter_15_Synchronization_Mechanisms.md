# Chapter 15: Kernel Synchronization Mechanisms

## Learning Goals
- Survey all synchronization primitives available in the Linux kernel
- Understand the trade-offs: performance, fairness, sleep-ability, context
- Know which primitive to use in each situation
- Understand lock granularity and its impact on scalability

---

## 15.1 Synchronization Overview

```
Linux Kernel Synchronization Primitives:

  ┌─────────────────────────────────────────────────────┐
  │              Synchronization Toolkit                 │
  │                                                      │
  │  Spinning (IRQ-safe):    Sleeping (process only):   │
  │  ├── spinlock_t          ├── struct mutex            │
  │  ├── raw_spinlock_t      ├── struct semaphore        │
  │  ├── rwlock_t            ├── struct rw_semaphore     │
  │  └── bit_spinlock        └── struct completion       │
  │                                                      │
  │  Lock-free:              Other:                      │
  │  ├── atomic_t/atomic64_t ├── RCU                     │
  │  ├── atomic_long_t       ├── per-CPU variables       │
  │  ├── Memory barriers     ├── seqlock                 │
  │  ├── READ_ONCE/WRITE_ONCE├── local_irq_disable       │
  │  └── refcount_t          ├── local_bh_disable        │
  │                          └── preempt_disable          │
  └─────────────────────────────────────────────────────┘
```

---

## 15.2 Mutual Exclusion Concept

```
Mutual exclusion: Only ONE thread can be in the critical
section at a time. All others must wait.

  Without mutual exclusion:     With mutual exclusion:
  CPU 0: read → modify →       CPU 0: lock → read → modify →
  CPU 1: read → modify →              write → unlock
     INTERLEAVED! CORRUPT!      CPU 1: ...waits... → lock →
                                       read → modify → write →
                                       unlock
                                 SERIALIZED! CORRECT!
```

---

## 15.3 Locking Mechanisms in Kernel

### Quick Reference: Which Lock When?

```
Situation                           │ Use
────────────────────────────────────┼──────────────────────
Process context, can sleep          │ mutex
Process context, very short hold    │ spinlock 
Process vs IRQ handler              │ spin_lock_irqsave
Process vs softirq                  │ spin_lock_bh
IRQ handler vs IRQ handler (SMP)    │ spin_lock
Read-heavy, rarely written          │ rwlock or RCU
Reference counting                  │ refcount_t / kref
Wait for event                      │ completion
Protect per-CPU data                │ preempt_disable
Protect sequence data               │ seqlock
Complex data structure traversal    │ RCU
```

### Lock Comparison Table

```
Lock Type      │ Sleep │ IRQ-Safe │ Preempt │ SMP │ Overhead │ Fairness
───────────────┼───────┼──────────┼─────────┼─────┼──────────┼─────────
spinlock_t     │ NO    │ YES*     │ Disables│ YES │ Lowest   │ FIFO(q)
raw_spinlock_t │ NO    │ YES      │ Disables│ YES │ Lowest   │ FIFO(q)
mutex          │ YES   │ NO       │ YES     │ YES │ Medium   │ FIFO
semaphore      │ YES   │ NO       │ YES     │ YES │ Medium   │ FIFO
rw_semaphore   │ YES   │ NO       │ YES     │ YES │ Higher   │ Complex
rwlock_t       │ NO    │ YES*     │ Disables│ YES │ Low      │ Unfair**
RCU            │ NO    │ YES      │ Disables│ YES │ Near-0*** │ N/A
seqlock        │ NO    │ YES      │ Disables│ YES │ Low      │ Writer prio
completion     │ YES   │ NO       │ YES     │ YES │ Low      │ FIFO

* With _irqsave variant
** Readers can starve writers
*** ZERO overhead for readers
```

---

## 15.4 Lock Granularity

### Coarse-Grained vs Fine-Grained

```
Coarse-grained (BKL — Big Kernel Lock, historical):
  One lock protects everything.
  
  ┌───────────────────────────────────────┐
  │  ONE LOCK for entire subsystem        │
  │                                        │
  │  Pro: Simple                           │
  │  Con: No parallelism — everything     │
  │       serialized                       │
  └───────────────────────────────────────┘
  
  Example: Early Linux had BKL protecting most kernel code.
  Removed over many kernel versions (finally gone in 2.6.39).

Fine-grained:
  Each data structure has its own lock.
  
  ┌────────┐ ┌────────┐ ┌────────┐
  │ Lock A │ │ Lock B │ │ Lock C │
  │ Data A │ │ Data B │ │ Data C │
  └────────┘ └────────┘ └────────┘
  
  Pro: Maximum parallelism (independent data accessed concurrently)
  Con: More complex, deadlock risk with multiple locks

Linux approach: Start medium, refine based on profiling.
  Example: Per-inode locks, per-page locks, per-CPU runqueues.
```

### Granularity Examples

```
Subsystem      │ Lock Strategy
───────────────┼──────────────────────────────
File system    │ Per-inode mutex (i_mutex)
Network        │ Per-socket lock (sk_lock)
Memory         │ Per-zone lock, per-page lock
Scheduler      │ Per-CPU runqueue lock (rq->lock)
Block I/O      │ Per-device queue lock
IRQ framework  │ Per-irq_desc lock (desc->lock)

General principle:
  Contention = threads waiting for lock = bad scaling
  Solution: Make locks more fine-grained or use lock-free (RCU)
```

---

## Decision Tree for Choosing a Synchronization Primitive

```
Start: Need to protect shared data?
  │
  ▼
Can you avoid sharing? (per-CPU data)
├── YES → Use per-CPU variables (Chapter 22)
│         preempt_disable() to access
└── NO
    │
    ▼
  Read-heavy, rarely modified?
  ├── YES → RCU (Chapter 23)
  │         Zero-cost reads, deferred frees
  └── NO
      │
      ▼
    Can the holder sleep?
    ├── YES (process context only)
    │   │
    │   ▼
    │ Need counting > 1?
    │ ├── YES → semaphore
    │ └── NO  → mutex (PREFERRED for binary)
    │
    └── NO (IRQ context or very short hold)
        │
        ▼
      Shared with hardirq handler?
      ├── YES → spin_lock_irqsave()
      │
      └── NO
          │
          ▼
        Shared with softirq/tasklet?
        ├── YES → spin_lock_bh()
        │
        └── NO → spin_lock()
```

---

## Kernel Source References

```
Synchronization primitives:
  include/linux/spinlock.h      ← spinlock API
  include/linux/mutex.h         ← mutex API
  include/linux/semaphore.h     ← semaphore API
  include/linux/rwsem.h         ← Reader-writer semaphore
  include/linux/completion.h    ← Completion variable
  include/linux/rcupdate.h      ← RCU API
  include/linux/seqlock.h       ← Sequence lock
  include/linux/atomic.h        ← Atomic operations
  include/linux/refcount.h      ← Reference counting
  include/linux/percpu.h        ← Per-CPU variables

Lock debugging:
  kernel/locking/lockdep.c      ← Lock dependency checker
  kernel/locking/mutex.c        ← Mutex implementation
  kernel/locking/spinlock.c     ← Spinlock implementation
```

---

## Interview Questions

1. **List all synchronization primitives available in the Linux kernel.**
2. **When would you use a mutex vs a spinlock?**
3. **What is lock granularity and how does it affect scalability?**
4. **What was the BKL and why was it removed?**
5. **Draw the decision tree for choosing the right lock.**
6. **What is a seqlock and when is it useful?**
7. **Why is RCU called "zero-overhead" for readers?**
8. **What is the difference between spinlock_t and raw_spinlock_t?**
9. **When do you need spin_lock_irqsave vs spin_lock_bh vs spin_lock?**
10. **What is a completion variable and when would you use it?**

---

## Summary

- Linux provides a rich toolkit: spinlocks, mutexes, semaphores, RCU, atomics, barriers, per-CPU, seqlocks
- Choose based on: context (IRQ vs process), duration (short vs long), pattern (read vs write)
- Lock granularity directly impacts SMP scalability
- The decision tree (context → sleep ability → contender type) guides correct choice
- Modern kernel favors: mutex (sleeping), RCU (read-heavy), per-CPU (avoid sharing)

---

*Next: [Chapter 16 — Spinlocks](Chapter_16_Spinlocks.md)*
