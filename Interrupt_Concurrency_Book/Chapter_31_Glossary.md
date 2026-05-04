# Chapter 31: Definitions and Glossary

## Learning Goals
- Quick reference for all interrupt and concurrency terminology
- Understand each term with one-line definition and context
- Use as a study aid for interviews and kernel development

---

## A

**Acquire Semantics** — Memory ordering where a load prevents all subsequent memory operations from being reordered before it. Used in lock acquisition. `smp_load_acquire()`.

**APIC (Advanced Programmable Interrupt Controller)** — x86 interrupt controller. Local APIC per CPU + I/O APIC for routing external IRQs.

**Atomic Context** — Execution state where sleeping is forbidden. Includes hardirq, softirq, and code under spin_lock. Check with `in_atomic()`.

**Atomic Operation** — CPU operation that completes indivisibly (no partial state visible). `atomic_inc()`, `test_and_set_bit()`.

**atomic_t** — Kernel type for atomic 32-bit integer operations. `include/linux/atomic.h`.

---

## B

**Barrier (Memory)** — CPU instruction (or compiler directive) that prevents reordering of memory operations across it. `smp_mb()`, `smp_wmb()`, `smp_rmb()`.

**Barrier (Compiler)** — `barrier()` — prevents the compiler from moving loads/stores across it, but has no CPU effect.

**BH (Bottom Half)** — Historical term for deferred interrupt work. Originally `struct bh_struct`, now replaced by softirq/tasklet/workqueue.

**Busy Waiting (Spinning)** — Repeatedly checking a condition in a loop. Spinlocks use this. Wastes CPU but avoids context switch overhead.

---

## C

**Cache Bouncing** — When multiple CPUs modify data on the same cache line, causing constant invalidation traffic. Per-CPU data avoids this.

**CAS (Compare-And-Swap)** — Atomic operation: if variable equals expected, set to new value. `atomic_cmpxchg()`. Foundation of lock-free algorithms.

**Completion** — Kernel synchronization primitive for waiting for an event. `wait_for_completion()`, `complete()`.

**Concurrency** — Multiple execution contexts making progress overlapping in time. Can occur on single core (preemption) or multiple cores (parallelism).

**Context Switch** — Saving one task's state and restoring another's. RCU quiescent state.

**Critical Section** — Code region that accesses shared data and must not be executed concurrently by multiple threads.

---

## D

**Deadlock** — Circular dependency where two or more tasks each hold a lock the other needs. Detected by lockdep.

**Deferred Work** — Processing postponed from interrupt time to a less critical context. Softirq, tasklet, workqueue, threaded IRQ.

**DMA (Direct Memory Access)** — Hardware transfers data to/from memory without CPU involvement. Completion signaled by interrupt.

**do_IRQ()** — x86 C entry point for hardware interrupts (older kernels).

---

## E

**Edge-Triggered** — Interrupt fires on signal transition (rising or falling edge). One shot — must be handled immediately.

**EOI (End of Interrupt)** — Signal to interrupt controller that handling is complete. `irq_chip->irq_eoi()`.

**Exception** — Synchronous event caused by CPU (division by zero, page fault). Uses same vector mechanism as interrupts.

---

## F

**False Sharing** — Two unrelated variables on the same cache line, causing bouncing when modified by different CPUs. Fix: `____cacheline_aligned`.

**force_irqthreads** — PREEMPT_RT mechanism to thread all IRQ handlers. Boot param: `threadirqs`.

**FTRACE** — Kernel's built-in function tracer. `/sys/kernel/debug/tracing/`.

---

## G

**GIC (Generic Interrupt Controller)** — ARM interrupt controller. Versions: GICv1, GICv2, GICv3, GICv4. Distributor + CPU interface.

**Grace Period (RCU)** — Time window during which all pre-existing RCU read-side critical sections complete. After grace period, old data can be freed.

---

## H

**Hardirq** — Hard interrupt context. IRQs disabled on local CPU. Check: `in_irq()`.

**hrtimer** — High-resolution timer using red-black tree. Nanosecond precision. `include/linux/hrtimer.h`.

---

## I

**IDT (Interrupt Descriptor Table)** — x86 table mapping interrupt vectors (0-255) to handlers. Set up at boot.

**IPI (Inter-Processor Interrupt)** — Interrupt sent from one CPU to another. Used for TLB shootdown, rescheduling, function calls.

**IRQ** — Interrupt Request. Hardware signal requesting CPU attention.

**irq_chip** — Kernel abstraction for interrupt controller hardware operations (mask, unmask, ack, eoi).

**irq_desc** — Per-IRQ descriptor containing handler chain, controller reference, and state.

**irq_domain** — Maps hardware IRQ numbers to Linux virtual IRQ numbers.

**IRQF_ONESHOT** — Keep IRQ masked until thread handler completes. Required for threaded handlers.

**IRQF_SHARED** — Multiple devices share one IRQ line. Each handler must check if its device generated the interrupt.

**IRQ Storm** — Rapid uncontrolled interrupt firing. Causes system hang. Usually due to uncleared interrupt condition.

---

## K

**KASAN** — Kernel Address Sanitizer. Detects use-after-free and out-of-bounds memory access.

**KCSAN** — Kernel Concurrency Sanitizer. Detects data races (missing locks/barriers).

**kfree_rcu()** — Free RCU-protected memory after grace period. Shorthand for `call_rcu()` + `kfree()`.

**ksoftirqd** — Per-CPU kernel thread that handles softirq overflow (when `__do_softirq` exceeds time/iteration limits).

**kworker** — Kernel worker thread for workqueue execution. Named `kworker/CPU:ID`.

---

## L

**Level-Triggered** — Interrupt asserts as long as signal is active. Must clear the source to de-assert.

**Livelock** — System is active but makes no useful progress (e.g., two tasks constantly yielding to each other).

**Lockdep** — Kernel lock dependency validator. Detects potential deadlocks at runtime. `CONFIG_PROVE_LOCKING`.

**Lock Ordering** — Rule that locks must always be acquired in the same sequence to prevent deadlocks.

---

## M

**Maskable Interrupt** — IRQ that can be disabled (masked) by software. Most hardware IRQs are maskable.

**MCS Lock** — Queue-based spinlock where each waiter spins on a local variable. Basis of qspinlock.

**Memory Model** — Rules defining when stores by one CPU become visible to other CPUs. `Documentation/memory-barriers.txt`.

**MSI (Message Signaled Interrupt)** — PCIe interrupt mechanism using memory writes instead of dedicated IRQ lines. Avoids sharing.

**Mutex** — Sleeping mutual exclusion lock. Owner can sleep. Has priority inheritance via `rt_mutex`. `include/linux/mutex.h`.

---

## N

**NAPI (New API)** — Network driver technique: interrupt → disable IRQ → poll packets in softirq → re-enable IRQ.

**NMI (Non-Maskable Interrupt)** — Highest priority interrupt. Cannot be disabled. Used for watchdog, profiling, crash dump.

**NO_HZ** — Tickless kernel mode. `CONFIG_NO_HZ_IDLE` (stop ticks on idle) or `CONFIG_NO_HZ_FULL` (stop ticks on busy CPUs too).

---

## P

**Per-CPU Data** — Data replicated per CPU, eliminating need for locking. `DEFINE_PER_CPU()`, `this_cpu_ptr()`.

**Preemption** — Involuntary context switch. `preempt_disable()` prevents it. Disabled under spinlock and in interrupt context.

**PREEMPT_RT** — Real-time Linux patch set (mainline since 6.12). Converts spinlocks to sleeping mutexes, threads all IRQs.

**Priority Inheritance** — When high-priority task blocks on lock held by low-priority task, low-priority temporarily gets high priority.

**Priority Inversion** — High-priority task blocked by low-priority task holding a needed lock. Fixed by priority inheritance.

---

## Q

**qspinlock (Queued Spinlock)** — Default Linux spinlock. 32-bit word with locked/pending/tail fields. MCS queue for scalability.

**Quiescent State (RCU)** — Point where a CPU is guaranteed not to be in an RCU read-side critical section (context switch, idle, user space).

---

## R

**Race Condition** — Bug where outcome depends on unpredictable timing of concurrent operations.

**Raw Spinlock** — `raw_spinlock_t`. Real hardware spinlock even on PREEMPT_RT. Used for scheduler, interrupt controller.

**RCU (Read-Copy-Update)** — Synchronization allowing lock-free readers. Writers defer freeing until grace period.

**READ_ONCE()** — Force compiler to read variable from memory (prevent load fusing/caching in register).

**refcount_t** — Hardened reference counter. Saturates on overflow, warns on underflow. Replaces `atomic_t` for refcounting.

**Release Semantics** — Memory ordering where a store ensures all preceding memory operations complete before it becomes visible. `smp_store_release()`.

**request_irq()** — Register an IRQ handler. `include/linux/interrupt.h`.

**request_threaded_irq()** — Register IRQ with both hardirq handler and thread handler.

**Rescheduling IPI** — IPI sent to wake a CPU to run a newly-queued task. `IPI_RESCHEDULE`.

**rwlock_t** — Reader-writer spinlock. Deprecated; prefer RCU or rw_semaphore.

**rw_semaphore** — Sleeping reader-writer lock with writer priority. `include/linux/rwsem.h`.

---

## S

**Semaphore** — Sleeping counter-based lock. Counting (N concurrent) or binary (1). Legacy; prefer mutex for binary.

**Shared IRQ** — Multiple devices on same IRQ line. Each handler must check device status.

**smp_mb()** — Full memory barrier preventing all reordering across it.

**smp_rmb()** — Read memory barrier preventing LoadLoad reordering.

**smp_wmb()** — Write memory barrier preventing StoreStore reordering.

**SoftIRQ** — Deferred interrupt processing mechanism. 10 predefined types (NET_TX, NET_RX, TIMER, etc.). Runs in softirq context.

**Spinlock** — Busy-wait lock. Holder cannot sleep. Fast for short critical sections. `spin_lock()`.

**Spurious Interrupt** — IRQ that fires without a real hardware event. Handler must return IRQ_NONE.

**SRCU (Sleepable RCU)** — RCU variant allowing readers to sleep. Per-domain grace periods.

**Starvation** — Thread perpetually denied access to a resource. qspinlock FIFO ordering prevents this.

**Store Buffer** — CPU-internal buffer holding stores before they're visible to other CPUs. Source of reordering.

---

## T

**Tasklet** — Softirq-based deferred work mechanism. Simpler than raw softirq. Serialized per-tasklet. Being deprecated in favor of threaded IRQ / workqueue.

**Threaded IRQ** — IRQ handler split: minimal hardirq + kernel thread for main processing. Can sleep in thread.

**Tick** — Periodic timer interrupt driving scheduler, timekeeping. Frequency: `CONFIG_HZ` (100-1000 Hz).

**TLB Shootdown** — IPI-based mechanism to invalidate TLB entries on remote CPUs after page table change.

**TOCTOU (Time of Check, Time of Use)** — Race condition where condition changes between checking and acting.

**Top Half** — The primary IRQ handler running in hardirq context. Fast, cannot sleep.

**TSO (Total Store Order)** — x86 memory model. Stores are seen in program order; only StoreLoad reordering allowed.

---

## V

**Vector** — Index into interrupt/exception table. x86: 0-31 exceptions, 32-255 interrupts. ARM: exception types (sync, IRQ, FIQ, SError).

---

## W

**Wait Queue** — Mechanism for tasks to sleep until a condition is true. `wait_event()`, `wake_up()`.

**Workqueue** — Kernel mechanism for deferring work to process context (kworker threads). Can sleep. `schedule_work()`.

**WRITE_ONCE()** — Force compiler to write variable to memory in a single store (prevent store tearing/fusing).

---

## Quick Reference: API → Header

```
API                     │ Header
────────────────────────┼──────────────────────────
request_irq             │ linux/interrupt.h
spin_lock               │ linux/spinlock.h
mutex_lock              │ linux/mutex.h
atomic_inc              │ linux/atomic.h
rcu_read_lock           │ linux/rcupdate.h
schedule_work           │ linux/workqueue.h
hrtimer_start           │ linux/hrtimer.h
DEFINE_PER_CPU          │ linux/percpu.h
complete                │ linux/completion.h
refcount_inc            │ linux/refcount.h
smp_mb                  │ asm/barrier.h
test_and_set_bit        │ linux/bitops.h
```

---

*Next: [Chapter 32 — OS Comparison](Chapter_32_OS_Comparison.md)*
