# Chapter 13: Concurrency in Operating Systems

## Learning Goals
- Understand concurrency from first principles in an OS context
- Distinguish processes, threads, and kernel execution contexts
- Know the four sources of concurrency in the Linux kernel
- See why concurrency bugs are the hardest class of kernel bugs

---

## 13.1 Definition of Concurrency

Concurrency means multiple execution flows make progress within overlapping time periods. They may or may not execute simultaneously — the key is that their execution interleaves in a way that creates the possibility of interaction.

```
Sequential:     A ████████████  B ████████████
Concurrent:     A ████░░██░░██  B ░░██░░██░░██  (interleaved)
Parallel:       A ████████████
                B ████████████  (simultaneous on different CPUs)
```

In the Linux kernel, concurrency arises even on a **single CPU** due to interrupts and preemption. On SMP, true parallelism adds another dimension.

---

## 13.2 Processes vs Threads

```
Process                              Thread
───────                              ──────
Own address space (mm_struct)        Shared address space
Own file descriptors                 Shared file descriptors
Own signal handlers                  Shared signal handlers
Heavy creation (copy page tables)    Light creation (no mm copy)
task_struct with unique mm           task_struct sharing mm
                                     
  Process A          Process B       Thread Group
  ┌──────────┐      ┌──────────┐    ┌──────────────────┐
  │ mm_struct │      │ mm_struct │    │   mm_struct      │
  │ ┌──────┐ │      │ ┌──────┐ │    │ ┌──────┐┌──────┐ │
  │ │ task │ │      │ │ task │ │    │ │task 1││task 2│ │
  │ └──────┘ │      │ └──────┘ │    │ └──────┘└──────┘ │
  └──────────┘      └──────────┘    └──────────────────┘
  Isolated memory    Isolated memory  Shared memory

Kernel view: Both processes and threads are task_struct.
A "thread" is just a task sharing mm, files, signals with a group.
```

---

## 13.3 Multicore Execution

```
SMP concurrency model:

  CPU 0              CPU 1              CPU 2
  ─────              ─────              ─────
  task A running     task B running     task C running
       │                  │                  │
       ▼                  ▼                  ▼
  Accesses shared    Accesses shared    Accesses shared
  data structure     data structure     data structure
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                    RACE CONDITION!
                    (without locking)
```

All three CPUs can literally execute at the **same nanosecond**, accessing the same data structure. This is the fundamental challenge.

---

## 13.4 Concurrent Execution Models in the Kernel

### The Four Contexts

```
Context            │ Can Sleep │ Preemptible │ IRQs │ Example
───────────────────┼───────────┼─────────────┼──────┼──────────────
Process context    │ YES       │ YES*        │ ON   │ System call
Softirq context   │ NO        │ NO**        │ ON   │ Tasklet, NET_RX
Hardirq context   │ NO        │ NO          │ OFF*** │ IRQ handler
NMI context        │ NO        │ NO          │ OFF  │ NMI handler

* If CONFIG_PREEMPT; ** preemptible on PREEMPT_RT
*** same-line off; other IRQs may nest on some configs
```

### Concurrency Scenarios in the Kernel

```
Scenario 1: Process vs Process (SMP)
  CPU 0: sys_write() → writes to file buffer
  CPU 1: sys_read()  → reads from same file buffer
  Protection: mutex (file->f_pos_lock)

Scenario 2: Process vs Softirq (same CPU)
  Process modifies network data → softirq fires → reads same data
  Protection: spin_lock_bh() / local_bh_disable()

Scenario 3: Process vs Hardirq (same CPU)
  Process accesses device state → device IRQ fires → handler accesses
  Protection: spin_lock_irqsave()

Scenario 4: Softirq vs Softirq (different CPUs)
  NET_RX_SOFTIRQ on CPU 0 and CPU 1 access shared socket
  Protection: spin_lock() (already in softirq, IRQs on)

Scenario 5: Hardirq vs Hardirq (different CPUs)
  Same IRQ on different CPUs (shared IRQ)
  Protection: spin_lock() within IRQ handler

Scenario 6: Preemption (same CPU)
  Task A running → preempted → task B accesses same data
  Protection: preempt_disable() or any lock
```

### Concurrency Rule Matrix

```
Which lock variant to use:

  Contender              │  Lock Type
  ───────────────────────┼────────────────────────
  Process vs Process     │  mutex (if can sleep)
  (different CPUs)       │  spin_lock() (if cannot)
                         │
  Process vs Softirq     │  spin_lock_bh()
  (same or diff CPU)     │
                         │
  Process vs Hardirq     │  spin_lock_irqsave()
  (same or diff CPU)     │
                         │
  Softirq vs Softirq    │  spin_lock()
  (different CPUs)       │
                         │
  Softirq vs Hardirq    │  spin_lock_irqsave()
  (same or diff CPU)     │
                         │
  Hardirq vs Hardirq    │  spin_lock()
  (different CPUs only)  │  (same CPU: cannot happen)
```

---

## Kernel Source References

```
Concurrency primitives:
  include/linux/spinlock.h      ← Spinlock API
  include/linux/mutex.h         ← Mutex API
  include/linux/preempt.h       ← preempt_disable/enable
  include/linux/bottom_half.h   ← local_bh_disable/enable
  include/linux/hardirq.h       ← in_irq(), in_softirq() checks
```

---

## Interview Questions

1. **What are the sources of concurrency in the Linux kernel?**
2. **Can concurrency bugs happen on a single-CPU system? Explain.**
3. **What is the difference between process context and interrupt context?**
4. **Why is it important to know WHICH context your code runs in?**
5. **Given process context code that shares data with a hardirq handler, which lock do you use?**
6. **What is preemption and how does it create concurrency on a single CPU?**
7. **Explain the concurrency difference between CONFIG_PREEMPT_NONE and CONFIG_PREEMPT.**
8. **What are the four execution contexts in the Linux kernel?**
9. **Why can you sleep in process context but not in interrupt context?**
10. **Draw the lock selection matrix for different context pairs.**

---

## Summary

- Concurrency in the kernel comes from SMP, preemption, interrupts, and softirqs
- Four execution contexts with different rules: process, softirq, hardirq, NMI
- Correct locking depends on which contexts contend for shared data
- Even single-CPU systems face concurrency from interrupts and preemption
- The lock type matrix (which lock for which context pair) is essential knowledge

---

*Next: [Chapter 14 — Concurrency Problems](Chapter_14_Concurrency_Problems.md)*
