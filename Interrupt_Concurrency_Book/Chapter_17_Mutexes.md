# Chapter 17: Mutexes

## Learning Goals
- Understand mutex design, implementation, and optimistic spinning
- Know when mutex is preferred over spinlock
- Write correct mutex-based synchronization
- Understand mutex debugging features and rt_mutex for priority inheritance

---

## 17.1 Mutex Concept

A mutex (mutual exclusion) is a **sleeping lock**: when a thread cannot acquire it, the thread is put to sleep instead of spinning, freeing the CPU for other work.

```
Spinlock vs Mutex behavior when lock is held:

  Spinlock:                     Mutex:
  CPU spins in tight loop       CPU puts thread to sleep
  ┌──→ check lock               Thread added to wait queue
  │    locked? spin ──┐         CPU runs other tasks
  │   unlocked? take  │         When lock released:
  └───────────────────┘           wake up first waiter

  Spinlock: wastes CPU          Mutex: efficient if wait is long
  Spinlock: no context switch   Mutex: context switch overhead
```

---

## 17.2 Mutex Implementation in Kernel

```c
/* include/linux/mutex.h */
struct mutex {
    atomic_long_t       owner;      /* Lock owner (task_struct ptr) */
    raw_spinlock_t      wait_lock;  /* Protects wait_list */
    struct list_head    wait_list;  /* List of waiting tasks */
    /* Debug fields when CONFIG_DEBUG_MUTEXES */
};
```

### Three-Path Acquisition

```
mutex_lock() attempts three paths:

  Path 1: Fast path (atomic CAS)
    If mutex is unlocked → acquire with single atomic op
    Cost: ~few nanoseconds (cache-line uncontended)

  Path 2: Optimistic spinning (MCS-like)
    If owner is running on another CPU → spin briefly
    Rationale: owner will release soon (no context switch)
    Uses osq_lock (optimistic spin queue)
    
  Path 3: Slow path (sleep)
    If owner is sleeping or spinning is fruitless:
    Add self to wait_list → schedule() → sleep
    Woken when lock released

  mutex_lock()
      │
      ├── Fast path: CAS 0 → current ─→ SUCCESS
      │
      ├── Midpath: Owner running on another CPU?
      │   └── Spin on owner->on_cpu (optimistic spinning)
      │       └── Owner releases → CAS → SUCCESS
      │
      └── Slow path: 
          ├── Add to wait_list (FIFO)
          ├── schedule() → sleep
          └── Woken up → try_to_acquire → SUCCESS
```

---

## 17.3 Sleeping Locks

### Mutex API

```c
/* Declaration: */
struct mutex my_mutex;
mutex_init(&my_mutex);           /* Dynamic */
DEFINE_MUTEX(my_mutex);          /* Static */

/* Lock (sleeps if unavailable): */
mutex_lock(&my_mutex);           /* Uninterruptible */
mutex_lock_interruptible(&my_mutex); /* Returns -EINTR on signal */
mutex_lock_killable(&my_mutex);  /* Returns -EINTR on fatal signal */

/* Try lock (non-blocking): */
if (mutex_trylock(&my_mutex)) {
    /* acquired */
} else {
    /* busy — don't sleep, handle differently */
}

/* Unlock: */
mutex_unlock(&my_mutex);

/* Check: */
mutex_is_locked(&my_mutex);      /* For debugging only */
```

### Mutex Rules

```
1. Only the OWNER can unlock (not like semaphore)
2. NO recursive locking (same thread locks twice → deadlock)
3. CANNOT be used in interrupt context (sleeps!)
4. Cannot be used in softirq context (sleeps!)
5. Must be released in the same context as acquired
6. Cannot hold while doing copy_to_user (might fault → sleep under lock OK if mutex)
   Actually: mutex IS OK for this since it's a sleeping lock
```

---

## 17.4 Mutex vs Spinlock Comparison

```
Feature          │ Spinlock              │ Mutex
─────────────────┼───────────────────────┼─────────────────────
Wait behavior    │ Busy-wait (spin)      │ Sleep (yield CPU)
Interrupt safe?  │ YES (with _irqsave)   │ NO (cannot use in IRQ)
Context          │ Any (process, IRQ)    │ Process only
Hold time        │ Very short (<µs)      │ Can be long (ms+)
CPU waste        │ Yes (while spinning)  │ No (sleeping)
Context switch   │ None                  │ 2 switches (sleep+wake)
PREEMPT_RT       │ spinlock_t → rt_mutex │ Still mutex
Recursion        │ Not allowed           │ Not allowed
Priority inherit │ No (yes with RT)      │ No (yes with rt_mutex)
Optimistic spin  │ N/A (always spin)     │ Yes (if owner running)
```

### When to Use Mutex Over Spinlock

```
Use mutex when:
  ✓ Critical section might take > few µs
  ✓ Need to allocate memory (GFP_KERNEL)
  ✓ Need to do I/O (file, I2C, SPI, USB)
  ✓ Need to call sleeping functions
  ✓ Only process context code involved

Use spinlock when:
  ✓ Critical section is very short
  ✓ Data shared with interrupt handler
  ✓ Data shared with softirq
  ✓ Cannot afford context switch overhead
```

---

## rt_mutex: Priority Inheritance Mutex

```c
/* kernel/locking/rtmutex.c */
/* rt_mutex implements priority inheritance:
   When high-priority task blocks on mutex held by low-priority:
   → Low-priority task boosted to high-priority level
   → Prevents priority inversion */

/* PREEMPT_RT: spinlock_t is implemented as rt_mutex!
   This means ALL spinlocks get PI on RT kernels. */

/* Explicit rt_mutex usage (rare — usually via futex/PI-futex): */
struct rt_mutex my_rt_mutex;
rt_mutex_init(&my_rt_mutex);
rt_mutex_lock(&my_rt_mutex);
rt_mutex_unlock(&my_rt_mutex);
```

---

## Kernel Source References

```
Mutex:
  kernel/locking/mutex.c            ← Core implementation
  include/linux/mutex.h             ← API definitions
  kernel/locking/mutex-debug.c      ← Debug extensions

rt_mutex:
  kernel/locking/rtmutex.c          ← RT mutex + PI
  include/linux/rtmutex.h

Optimistic spinning:
  kernel/locking/osq_lock.c         ← MCS-based optimistic spin queue
```

---

## Interview Questions

1. **How does a mutex differ from a spinlock?**
2. **Explain the three-path acquisition of Linux mutex.**
3. **What is optimistic spinning in mutex? Why does it work?**
4. **Why can't you use a mutex in interrupt context?**
5. **What is rt_mutex and how does it implement priority inheritance?**
6. **Can you lock a mutex recursively? What happens if you try?**
7. **What is mutex_lock_interruptible vs mutex_lock?**
8. **Why is mutex preferred for protecting I2C/SPI transactions?**
9. **How does PREEMPT_RT relate spinlock_t to rt_mutex?**
10. **Design a driver that uses mutex to protect a shared data buffer.**

---

## Summary

- Mutex is a sleeping lock: thread yields CPU instead of spinning
- Three acquisition paths: fast (CAS), optimistic spin (owner running), slow (sleep)
- Use mutex for process-context-only locking, especially for longer critical sections
- Cannot use in IRQ or softirq context (would sleep in atomic context)
- rt_mutex adds priority inheritance — PREEMPT_RT builds spinlock_t on rt_mutex
- Mutex is the most commonly used lock in driver code (process-context operations)

---

*Next: [Chapter 18 — Semaphores](Chapter_18_Semaphores.md)*
