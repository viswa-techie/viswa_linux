# Chapter 16: Spinlocks

## Learning Goals
- Master spinlock internals: ticket locks, qspinlock, MCS
- Know all spinlock API variants and when to use each
- Understand raw_spinlock_t vs spinlock_t on PREEMPT_RT
- Write correct spinlock code in IRQ, softirq, and process contexts
- Avoid common spinlock pitfalls

---

## 16.1 Spinlock Concept

A spinlock is a busy-wait lock: the thread spins (loops) checking the lock until it becomes available. Suitable for short critical sections where sleeping is not allowed.

```
  CPU 0                    CPU 1
  ─────                    ─────
  spin_lock(&lock)         spin_lock(&lock)
  ← acquired! →            ← spins... →
  critical section         ← spins... →
  spin_unlock(&lock)       ← acquired! →
                           critical section
                           spin_unlock(&lock)
```

---

## 16.2 Spinlock Implementation

### Evolution: Test-and-Set → Ticket → qspinlock

```
1. Test-and-Set (ancient, unfair):
   While (test_and_set(&lock)) { /* spin */ }
   Problem: Not FIFO — starvation possible.

2. Ticket Spinlock (fair, FIFO):
   Next ticket number + serving counter.
   Each CPU takes a ticket, waits for its number.
   Problem: All CPUs spin on same cache line (scalability).

3. qspinlock (current Linux default):
   Combines ticket lock efficiency with MCS queue.
   First waiter spins on lock word (fast path).
   Additional waiters queue and spin on LOCAL variable.
   Minimal cache-line bouncing on high contention.
```

### qspinlock Structure

```c
/* include/asm-generic/qspinlock_types.h */
typedef struct qspinlock {
    union {
        atomic_t val;        /* 32-bit lock word */
        struct {
            u8 locked;       /* 0: unlocked, 1: locked */
            u8 pending;      /* 1: one waiter pending */
            u16 tail;        /* Queue tail (MCS node index) */
        };
    };
} arch_spinlock_t;

/* Lock word encoding:
   Bits 0:    locked (lock held)
   Bits 1:    pending (first waiter)
   Bits 2-15: MCS queue tail
   
   Fast path: CAS 0 → locked (one atomic op)
   Slow path: Pending bit → MCS queue → spin locally
*/
```

---

## 16.3 Spinlock Usage in Interrupt Context

### The Fundamental Rule

```
If a spinlock is shared between process context and IRQ:
  Process context MUST use spin_lock_irqsave()
  
WHY:
  CPU 0: process context takes spin_lock(&L)
  CPU 0: IRQ fires → handler tries spin_lock(&L)
  CPU 0: DEADLOCK! (spinning on lock it holds, IRQ can't return)

  Timeline:
  Process ──lock──┐
  IRQ fires ──────── spin_lock ──→ DEADLOCK!
                  └── never reaches unlock
```

### All Spinlock Variants

```c
/* Basic (process context, no IRQ interaction): */
spin_lock(&lock);
spin_unlock(&lock);

/* Disable IRQs on local CPU (share with hardirq): */
spin_lock_irq(&lock);    /* Assumes IRQs were enabled! */
spin_unlock_irq(&lock);

/* Save IRQ state (safe in any context): */
unsigned long flags;
spin_lock_irqsave(&lock, flags);
spin_unlock_irqrestore(&lock, flags);

/* Disable bottom halves (share with softirq): */
spin_lock_bh(&lock);
spin_unlock_bh(&lock);

/* Try lock (non-blocking): */
if (spin_trylock(&lock)) {
    /* acquired */
    spin_unlock(&lock);
} else {
    /* lock busy — handle without lock */
}

/* Initialization: */
spinlock_t lock;
spin_lock_init(&lock);         /* Dynamic */
DEFINE_SPINLOCK(lock);          /* Static */
```

### When to Use Which Variant

```
Variant            │ Use When
───────────────────┼────────────────────────────────────
spin_lock()        │ Only process-vs-process (SMP) or
                   │ softirq-vs-softirq (diff CPUs)
spin_lock_bh()     │ Process context shares with softirq
spin_lock_irq()    │ Process shares with hardirq, and you
                   │ KNOW IRQs are currently enabled
spin_lock_irqsave()│ Safest: process shares with hardirq,
                   │ unknown if IRQs are already off
spin_trylock()     │ NMI context, or when you can't wait
```

---

## 16.4 Raw Spinlocks

### raw_spinlock_t vs spinlock_t on PREEMPT_RT

```
Standard kernel:
  spinlock_t = raw_spinlock_t (identical)
  Both are true spinning locks.

PREEMPT_RT kernel:
  spinlock_t → converted to rt_mutex (sleeping lock!)
    - Can sleep! (preemptible)
    - Supports priority inheritance
    - Context switch possible while "spinning"
  
  raw_spinlock_t → stays as true spinning lock
    - Still disables preemption
    - Use only where sleeping is truly impossible
    - Examples: scheduler code, interrupt entry, low-level arch

Rule on PREEMPT_RT:
  Use spinlock_t by DEFAULT (becomes RT-safe mutex)
  Use raw_spinlock_t ONLY for truly non-preemptible sections
  (scheduler, IRQ entry, hardirq handlers that can't sleep)
```

```c
/* raw_spinlock_t API (same naming, different prefix): */
raw_spinlock_t raw_lock;
raw_spin_lock_init(&raw_lock);
raw_spin_lock(&raw_lock);
raw_spin_unlock(&raw_lock);
raw_spin_lock_irqsave(&raw_lock, flags);
raw_spin_unlock_irqrestore(&raw_lock, flags);
```

---

## 16.5 Read-Write Spinlocks

```c
/* Multiple readers can hold lock simultaneously.
   Writers get exclusive access. */

rwlock_t my_rwlock;
rwlock_init(&my_rwlock);

/* Reader: */
read_lock(&my_rwlock);
/* Multiple CPUs can read concurrently */
read_unlock(&my_rwlock);

/* Writer: */
write_lock(&my_rwlock);
/* Exclusive — no readers or other writers */
write_unlock(&my_rwlock);

/* IRQ variants exist: */
read_lock_irqsave(&my_rwlock, flags);
write_lock_irqsave(&my_rwlock, flags);
```

### Problems with rwlock_t

```
Issues:
  1. Writers can be starved by continuous readers
  2. Not as fast as expected (cache-line bouncing on read)
  3. No priority inheritance on PREEMPT_RT

Modern alternatives:
  - RCU for read-mostly data (zero reader overhead)
  - seqlock for data with one writer, many readers
  - rw_semaphore for sleeping read-write locks
  
rwlock_t is considered legacy. Prefer RCU or seqlock.
```

---

## Common Spinlock Pitfalls

```
1. SLEEPING UNDER SPINLOCK:
   spin_lock(&lock);
   kmalloc(size, GFP_KERNEL);   /* BUG! May sleep */
   spin_unlock(&lock);
   Fix: Use GFP_ATOMIC or move allocation outside lock.

2. WRONG VARIANT (IRQ deadlock):
   spin_lock(&lock);             /* Process context */
   /* IRQ fires, handler: */
   spin_lock(&lock);             /* DEADLOCK! */
   Fix: Use spin_lock_irqsave() in process context.

3. HOLDING LOCK TOO LONG:
   spin_lock(&lock);
   /* ... heavy computation ... */  /* Bad! Other CPUs spin */
   spin_unlock(&lock);
   Fix: Do computation outside lock, only protect data access.

4. NESTED LOCKS IN WRONG ORDER:
   CPU 0: lock(A) → lock(B)
   CPU 1: lock(B) → lock(A)  /* DEADLOCK! */
   Fix: Always acquire in same order. Document ordering.

5. RETURNING WITH LOCK HELD:
   spin_lock(&lock);
   if (error)
       return -EINVAL;           /* BUG! Lock not released */
   spin_unlock(&lock);
   Fix: goto out; out: spin_unlock(&lock); return ret;
```

---

## Kernel Source References

```
Spinlock implementation:
  kernel/locking/spinlock.c     ← Wrapper implementation
  include/linux/spinlock.h      ← API definitions
  include/asm-generic/qspinlock.h ← qspinlock
  kernel/locking/qspinlock.c    ← qspinlock slow path

PREEMPT_RT:
  kernel/locking/spinlock_rt.c  ← RT spinlock (rt_mutex based)
  include/linux/spinlock_rt.h   ← RT spinlock API

Lock debugging:
  kernel/locking/lockdep.c      ← Dependency validator
  lib/debug_locks.c             ← Generic lock debugging
```

---

## Interview Questions

1. **How does a spinlock work internally? What is qspinlock?**
2. **Why must you use spin_lock_irqsave() when sharing data with an IRQ handler?**
3. **What is the difference between spin_lock_irq and spin_lock_irqsave?**
4. **What happens to spinlock_t on a PREEMPT_RT kernel?**
5. **When would you use raw_spinlock_t?**
6. **What happens if you call schedule() while holding a spinlock?**
7. **Explain the evolution from test-and-set to ticket to qspinlock.**
8. **What are the problems with rwlock_t? What should you use instead?**
9. **What is the advantage of qspinlock over ticket spinlock for high contention?**
10. **List five common spinlock pitfalls and their fixes.**
11. **Can you nest spinlocks? What precautions are needed?**
12. **How does lockdep detect spinlock deadlocks?**

---

## Summary

- Spinlocks busy-wait — appropriate for short critical sections and interrupt context
- Use the right variant: plain (process), _bh (softirq), _irqsave (hardirq)
- qspinlock is the modern Linux implementation: fair, scalable, MCS-queued
- On PREEMPT_RT, spinlock_t becomes a sleeping rt_mutex; raw_spinlock_t stays spinning
- Read-write spinlocks are legacy; prefer RCU for read-mostly data
- Always follow lock ordering, never sleep under spinlock, keep critical sections short

---

*Next: [Chapter 17 — Mutexes](Chapter_17_Mutexes.md)*
