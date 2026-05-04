# Chapter 14: Concurrency Problems

## Learning Goals
- Identify race conditions, deadlocks, livelocks, and starvation
- Understand critical sections and why they must be protected
- Recognize concurrency bugs in real kernel code patterns
- Debug and prevent each class of concurrency problem

---

## 14.1 Race Conditions

A race condition occurs when the outcome depends on the relative timing of two or more concurrent operations on shared data.

```c
/* Classic race: lost update */
/* Thread A (CPU 0) */           /* Thread B (CPU 1) */
counter = shared->count;         counter = shared->count;
  /* reads 5 */                    /* reads 5 */
counter++;                       counter++;
  /* computes 6 */                 /* computes 6 */
shared->count = counter;         shared->count = counter;
  /* writes 6 */                   /* writes 6 */

/* Expected: shared->count = 7.  Actual: 6!  Lost update! */
```

### TOCTOU (Time-of-Check to Time-of-Use)

```c
/* Dangerous: check and use are not atomic */
if (device_is_ready(dev)) {      /* CHECK */
    /* Another CPU or IRQ could change state here! */
    read_device_data(dev);        /* USE — may no longer be ready! */
}

/* Fix: hold lock across check AND use */
spin_lock(&dev->lock);
if (device_is_ready(dev))
    read_device_data(dev);
spin_unlock(&dev->lock);
```

---

## 14.2 Critical Sections

```
Critical section = code that accesses shared resources.
Must be executed atomically (mutual exclusion).

  spin_lock(&lock);        ← Enter critical section
  shared_data->field = val; ← Access shared resource
  list_add(&item, &list);  ← Modify shared structure  
  spin_unlock(&lock);      ← Exit critical section

Rules:
  1. Keep critical sections SHORT (hold locks briefly)
  2. Don't sleep while holding spinlocks
  3. Avoid nested locks when possible (deadlock risk)
  4. Protect ALL accesses to shared data (not just writes)
  5. Use the NARROWEST lock scope (fine-grained > coarse-grained)
```

---

## 14.3 Deadlocks

### Definition

A deadlock occurs when two or more execution contexts are each waiting for the other to release a resource, and none can proceed.

```
Classic deadlock (ABBA):

  CPU 0                         CPU 1
  ─────                         ─────
  spin_lock(&A);                spin_lock(&B);
       │ holds A                     │ holds B
       ▼                             ▼
  spin_lock(&B);  ← BLOCKED!   spin_lock(&A);  ← BLOCKED!
  (B held by CPU 1)            (A held by CPU 0)
       │                             │
       └──── Both wait forever! ─────┘
             DEADLOCK!
```

### Lock Ordering Rule

```
ALWAYS acquire locks in the SAME ORDER everywhere:

  /* Correct: always A before B */
  CPU 0:  lock(A) → lock(B) → unlock(B) → unlock(A)
  CPU 1:  lock(A) → lock(B) → unlock(B) → unlock(A)

  /* WRONG: different order = deadlock! */
  CPU 0:  lock(A) → lock(B)
  CPU 1:  lock(B) → lock(A)  ← NEVER do this!
```

### Self-Deadlock: IRQ Context

```c
/* Deadlock scenario: process context + IRQ */
spin_lock(&dev->lock);        /* Process context takes lock */
/* ... interrupt fires on SAME CPU ... */
  /* IRQ handler: */
  spin_lock(&dev->lock);      /* DEADLOCK! Same CPU, same lock */
  /* CPU spins forever — IRQ can't be preempted */

/* Fix: disable IRQs when process context shares lock with IRQ */
spin_lock_irqsave(&dev->lock, flags);
/* ... safe — IRQ cannot fire on this CPU while lock held ... */
spin_unlock_irqrestore(&dev->lock, flags);
```

### Deadlock Detection: lockdep

```
Linux has a built-in lock dependency validator: CONFIG_LOCKDEP

Detects:
  - ABBA deadlocks (inconsistent lock ordering)
  - Self-deadlocks (same lock in nested contexts)  
  - IRQ-unsafe locks (missing irqsave variant)

  [ BUG: possible circular locking dependency detected ]
  [ INFO: possible irq lock inversion dependency detected ]

Enable in kernel config:
  CONFIG_PROVE_LOCKING=y
  CONFIG_LOCKDEP=y
  CONFIG_DEBUG_LOCK_ALLOC=y
```

---

## 14.4 Livelocks

```
Livelock: Threads keep executing but make no progress
(constantly yielding or retrying without completion).

Example: Two CPUs try CAS (Compare-And-Swap) in a loop,
each invalidating the other's cache line:

  CPU 0: CAS → succeeds → CPU 1 invalidates → retry
  CPU 1: CAS → succeeds → CPU 0 invalidates → retry
  Both CPUs busy but neither completes the operation!

Solutions:
  - Exponential backoff between retries
  - Use proper locks instead of lock-free retry loops
  - Limit retry count
```

---

## 14.5 Starvation

```
Starvation: A thread never gets the resource it needs
because other threads keep taking it first.

Example with spinlock:
  CPU 0, 1, 2 all contending for same spinlock.
  CPUs 0 and 1 are fast — they release and re-acquire quickly.
  CPU 2 always loses the race → never acquires the lock.

Example with priority:
  Low-priority task holds a mutex.
  High-priority tasks keep preempting.
  Low-priority task never completes → mutex never released.
  Other tasks waiting for mutex → starved!

Solutions:
  - Ticket spinlocks (FIFO ordering) — Linux default
  - qspinlock (queued) — fair, scalable
  - Priority inheritance (rt_mutex)
  - Reader-writer lock fairness policies
```

### Priority Inversion (Related)

```
  High-priority task H needs lock held by low-priority L.
  Medium-priority M preempts L.
  H effectively blocked by M — priority inverted!

  Timeline:
  L ──runs──lock──┐
  M ────────────── preempts L ──runs────────
  H ──────────────────────── blocked on lock ──→ STARVED!

  Fix: Priority Inheritance (PI)
    When H blocks on lock held by L:
      L's priority boosted to H's level
      L can't be preempted by M
      L finishes → releases lock → H gets lock immediately
    
  Linux: rt_mutex implements PI automatically
```

---

## Common Kernel Concurrency Bugs

```
Bug Type                    │ Symptom                  │ Fix
────────────────────────────┼──────────────────────────┼────────────
Missing lock                │ Corrupted data, oops     │ Add lock
Wrong lock variant          │ Deadlock, hang           │ Use _irqsave
Lock ordering violation     │ Rare deadlock            │ lockdep, order
Use-after-free (race)       │ Oops, memory corruption  │ RCU, refcount
Double free (race)          │ Panic, slab error        │ Proper sync
Sleep under spinlock        │ BUG: scheduling atomic   │ Use mutex
Read without lock           │ Stale/inconsistent data  │ Lock reads too
Forgot local_bh_disable     │ Softirq races            │ spin_lock_bh
Forgot preempt_disable      │ Per-CPU data corruption  │ get_cpu/put_cpu
```

---

## Interview Questions

1. **What is a race condition? Give a kernel-level example.**
2. **What is TOCTOU and how do you prevent it?**
3. **Explain the ABBA deadlock pattern.**
4. **Why does taking a spinlock in process context that's also used in an IRQ handler cause deadlock?**
5. **What is lockdep and what types of problems does it detect?**
6. **What is the difference between a deadlock and a livelock?**
7. **How do ticket spinlocks prevent starvation?**
8. **What is priority inversion? How does Linux solve it?**
9. **List five common concurrency bugs in kernel drivers.**
10. **What is a critical section? What are the rules for designing one?**
11. **Can use-after-free be caused by a concurrency bug? Explain.**
12. **How would you debug a rare deadlock that only happens under load?**

---

## Summary

- Race conditions: outcome depends on timing — fix with proper synchronization
- Critical sections: code accessing shared data — must be mutually exclusive
- Deadlocks: circular wait for resources — prevent with consistent lock ordering
- Livelocks: busy but no progress — add backoff or use different synchronization
- Starvation: unfair resource access — use fair locks (qspinlock) and PI (rt_mutex)
- lockdep is an invaluable tool for catching deadlocks and lock ordering violations at runtime

---

*Next: [Chapter 15 — Kernel Synchronization Mechanisms](Chapter_15_Synchronization_Mechanisms.md)*
