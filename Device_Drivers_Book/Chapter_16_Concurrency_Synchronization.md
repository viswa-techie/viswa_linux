# Chapter 16: Concurrency and Synchronization

## Chapter Overview

Drivers run in a multi-core, preemptible, interrupt-driven environment. Without proper synchronization, data corruption, deadlocks, and race conditions are inevitable. This chapter covers every locking mechanism available.

---

## 16.1 Concurrency Issues in Drivers

### Sources of Concurrency

```
┌────────────────────────────────────────────────┐
│ Why driver code can be interrupted:             │
│                                                 │
│ 1. SMP: Multiple CPUs run driver code in       │
│    parallel                                     │
│ 2. Preemption: CONFIG_PREEMPT allows kernel     │
│    preemption at almost any point               │
│ 3. Interrupts: Hardware IRQ interrupts         │
│    currently running driver code               │
│ 4. Softirqs/Tasklets: Deferred work may        │
│    run on any CPU                              │
│ 5. Workqueues: Multiple work items may run     │
│    concurrently                                │
│ 6. User processes: Multiple threads doing      │
│    ioctl/read/write simultaneously             │
└────────────────────────────────────────────────┘
```

---

## 16.2 Race Conditions

### Classic Race Condition

```c
/* BUGGY: race condition on shared counter */
static int open_count = 0;

static int my_open(struct inode *inode, struct file *filp)
{
    if (open_count >= MAX_OPENS)   /* Thread A reads: 4 */
        return -EBUSY;             /* Thread B reads: 4 */
    open_count++;                  /* A writes: 5, B writes: 5 (should be 6!) */
    return 0;
}
```

---

## 16.3-16.4 Spinlocks

Fast lock for short critical sections. **Cannot sleep while holding.**

```c
#include <linux/spinlock.h>

spinlock_t lock;
spin_lock_init(&lock);

/* Basic spinlock */
spin_lock(&lock);
/* ... critical section ... */
spin_unlock(&lock);

/* With IRQ disable (if shared with IRQ handler) */
unsigned long flags;
spin_lock_irqsave(&lock, flags);
/* ... safe from both SMP + IRQ ... */
spin_unlock_irqrestore(&lock, flags);

/* With softirq disable */
spin_lock_bh(&lock);
/* ... safe from softirqs (tasklets, timers) ... */
spin_unlock_bh(&lock);
```

### When to Use Which Spinlock Variant

| Contexts Sharing Data | Lock Type |
|-----------------------|-----------|
| Process ↔ Process (same context) | `spin_lock()` |
| Process ↔ Hardirq | `spin_lock_irqsave()` |
| Process ↔ Softirq/Tasklet | `spin_lock_bh()` |
| Hardirq ↔ Hardirq (different IRQs) | `spin_lock()` inside handler |
| Softirq ↔ Softirq | `spin_lock()` (softirqs of same type serialized per-CPU) |

---

## 16.5 Mutexes

Sleeping lock for longer critical sections in process context.

```c
#include <linux/mutex.h>

struct mutex lock;
mutex_init(&lock);

/* Standard lock */
mutex_lock(&lock);       /* Sleeps if contended */
/* ... critical section (can sleep here!) ... */
mutex_unlock(&lock);

/* Interruptible (returns -EINTR if signal received) */
if (mutex_lock_interruptible(&lock))
    return -ERESTARTSYS;
/* ... */
mutex_unlock(&lock);

/* Trylock (non-blocking) */
if (!mutex_trylock(&lock))
    return -EBUSY;   /* Couldn't acquire */
/* ... */
mutex_unlock(&lock);
```

### Mutex Rules
- **Only process context** — NEVER in IRQ/softirq/atomic
- **Owner must unlock** — cannot unlock from different context
- **Non-recursive** — locking twice deadlocks
- **Can sleep** while holding

---

## 16.6 Semaphores

Counting lock — allows N concurrent holders. Rarely used; prefer mutex.

```c
#include <linux/semaphore.h>

struct semaphore sem;
sema_init(&sem, 3);   /* Allow 3 concurrent accesses */

down(&sem);            /* Acquire (decrement), sleep if 0 */
/* ... */
up(&sem);              /* Release (increment) */

/* Interruptible variant */
if (down_interruptible(&sem))
    return -ERESTARTSYS;
```

---

## 16.7 Atomic Operations

Lock-free operations for simple counters and flags.

```c
#include <linux/atomic.h>

atomic_t counter = ATOMIC_INIT(0);

atomic_inc(&counter);                    /* counter++ */
atomic_dec(&counter);                    /* counter-- */
int val = atomic_read(&counter);         /* Read value */
atomic_set(&counter, 5);                 /* Set value */
atomic_add(3, &counter);                 /* counter += 3 */

/* Test and modify */
if (atomic_dec_and_test(&counter))       /* Returns true if result == 0 */
    /* Last reference dropped */

/* Compare and swap */
int old = 5;
atomic_cmpxchg(&counter, old, 10);      /* If counter==5, set to 10 */

/* 64-bit variant */
atomic64_t big_counter = ATOMIC64_INIT(0);
atomic64_inc(&big_counter);

/* Bit operations (on unsigned long) */
unsigned long flags = 0;
set_bit(3, &flags);                      /* Set bit 3 */
clear_bit(3, &flags);                    /* Clear bit 3 */
if (test_and_set_bit(5, &flags))         /* Was already set? */
    /* bit 5 was previously set */
```

---

## 16.8 Read-Write Locks

For data that is read frequently but written rarely.

```c
/* RW Spinlock */
rwlock_t rwlock;
rwlock_init(&rwlock);

read_lock(&rwlock);       /* Multiple readers OK */
/* ... read shared data ... */
read_unlock(&rwlock);

write_lock(&rwlock);      /* Exclusive */
/* ... modify shared data ... */
write_unlock(&rwlock);

/* RW Semaphore (sleeping) */
struct rw_semaphore rwsem;
init_rwsem(&rwsem);

down_read(&rwsem);        /* Multiple readers OK, can sleep */
/* ... read ... */
up_read(&rwsem);

down_write(&rwsem);       /* Exclusive, can sleep */
/* ... write ... */
up_write(&rwsem);
```

---

## Complete Lock Selection Guide

```
Need to protect shared data?
       │
       ├── Is it just a simple counter?
       │       └── YES → atomic_t / atomic64_t
       │
       ├── Is it accessed from IRQ context?
       │       └── YES → spin_lock_irqsave()
       │              └── Also from process context?
       │                     └── spin_lock_irqsave() in both
       │
       ├── Is it accessed from softirq/tasklet?
       │       └── YES → spin_lock_bh()
       │
       ├── Is the critical section short (no sleeping)?
       │       └── YES → spinlock
       │
       ├── Does the critical section need to sleep?
       │       └── YES → mutex
       │
       ├── Mostly readers, rare writers?
       │       └── YES → rw_semaphore
       │
       └── Need N concurrent accesses?
               └── YES → semaphore (rare)
```

### Synchronization Comparison Table

| Mechanism | Context | Can Sleep | Recursive | Performance | Use Case |
|-----------|---------|-----------|-----------|-------------|----------|
| **Spinlock** | Any | No | No | Fastest | Short critical sections |
| **Mutex** | Process | Yes | No | Good | I/O, allocation, long sections |
| **Semaphore** | Process | Yes | N/A | Moderate | Count-limited access |
| **Atomic** | Any | N/A | N/A | Lock-free | Counters, flags |
| **RW lock** | Any/Process | Depends | No | Read-biased | Read-heavy data |
| **RCU** | Read: Any | Read: No | Read: Yes | Read: zero-cost | Read-dominated lists |
| **Completion** | Process | Yes | N/A | Event-based | Wait for event |

---

## Completion (Wait for Event)

```c
#include <linux/completion.h>

struct completion done;
init_completion(&done);

/* Waiter (e.g., in write/ioctl) */
wait_for_completion(&done);                    /* Sleep until complete */
wait_for_completion_timeout(&done, HZ * 5);    /* 5 second timeout */
wait_for_completion_interruptible(&done);      /* Interruptible */

/* Signaler (e.g., in IRQ handler) */
complete(&done);          /* Wake one waiter */
complete_all(&done);      /* Wake all waiters */
reinit_completion(&done);  /* Reset for reuse */
```

---

## Common Deadlock Patterns

```c
/* Deadlock 1: ABBA ordering */
/* Thread 1: lock(A) → lock(B) */
/* Thread 2: lock(B) → lock(A)   ← DEADLOCK */
/* Fix: Always lock in same order (A before B) */

/* Deadlock 2: mutex in IRQ */
/* IRQ handler: mutex_lock(&mtx)   ← DEADLOCK (can't sleep in IRQ!) */
/* Fix: Use spin_lock_irqsave() for IRQ-shared data */

/* Deadlock 3: recursive spinlock */
/* spin_lock(&lock); spin_lock(&lock); ← DEADLOCK (self) */
/* Fix: Don't re-enter, or restructure code */

/* Deadlock 4: sleeping inside spinlock */
/* spin_lock(&lock);
 * kmalloc(size, GFP_KERNEL);  ← May sleep → DEADLOCK
 * Fix: Use GFP_ATOMIC or move alloc outside spinlock */
```

### Lockdep: Deadlock Detector

```bash
# Enable in kernel config
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_LOCK_ALLOC=y
CONFIG_LOCKDEP=y

# Lockdep detects:
# - ABBA lock ordering violations
# - Sleeping inside atomic context
# - IRQ safety mismatches
# Check dmesg for lockdep warnings
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| `include/linux/spinlock.h` | Spinlock API |
| `include/linux/mutex.h` | Mutex API |
| `include/linux/rwsem.h` | RW semaphore |
| `include/linux/atomic.h` | Atomic operations |
| `include/linux/completion.h` | Completion API |
| `kernel/locking/mutex.c` | Mutex implementation |
| `kernel/locking/spinlock.c` | Spinlock implementation |
| `kernel/locking/lockdep.c` | Lock dependency validator |

---

## Interview Questions

**Q1: When should you use a spinlock vs a mutex?**
A: Spinlock: short critical sections, any context (including IRQ), cannot sleep while holding. Mutex: longer sections, process context only, can sleep while holding. Rule: if the critical section might sleep (I/O, memory allocation with GFP_KERNEL), use mutex.

**Q2: What happens if you call `mutex_lock()` in interrupt context?**
A: Kernel bug — mutex_lock may sleep (schedule out), but interrupt context has no process to schedule from. This causes a "scheduling while atomic" BUG, likely crashing the system. Lockdep catches this at runtime.

**Q3: Why use `spin_lock_irqsave()` instead of `spin_lock()`?**
A: If the same data is accessed from both process context and IRQ handler, `spin_lock()` alone won't prevent the IRQ from interrupting the lock holder (deadlock). `spin_lock_irqsave()` disables interrupts on the local CPU + acquires the spinlock.

**Q4: What is a completion and when should you use it?**
A: A completion is a lightweight synchronization mechanism for one thread waiting for another to finish something. Common pattern: submit DMA → `wait_for_completion_timeout()` → IRQ handler calls `complete()`. Simpler than cond_wait/signal.

**Q5: What is lockdep?**
A: Runtime lock dependency validator. Tracks lock acquisition order across the entire kernel. Detects potential deadlocks (ABBA) even if they haven't occurred yet. Essential for driver development — enable it always.

---

## Summary

| Lock | Process Ctx | IRQ Ctx | Can Sleep | Speed |
|------|------------|---------|-----------|-------|
| spinlock | Yes | Yes | No | Fastest |
| spin_lock_irqsave | Yes | Yes | No | Fast |
| mutex | Yes | **No** | Yes | Good |
| semaphore | Yes | **No** | Yes | Moderate |
| atomic_t | Yes | Yes | N/A | Lock-free |
| completion | Yes | Signal: Yes | Wait: Yes | Event |

---

*Next: [Chapter 17 — Memory Management for Drivers](Chapter_17_Memory_Management.md)*
