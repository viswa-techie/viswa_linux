# Chapter 18: Semaphores

## Learning Goals
- Understand counting and binary semaphores
- Know when to use semaphores vs mutexes
- Use the semaphore API correctly in driver code
- Understand rw_semaphore for reader-writer patterns

---

## 18.1 Semaphore Concept

A semaphore is a sleeping synchronization primitive with a counter. When the counter reaches 0, subsequent `down()` callers sleep until someone calls `up()`.

```
Semaphore with count=3:
  Thread A: down() → count=2, proceed
  Thread B: down() → count=1, proceed
  Thread C: down() → count=0, proceed
  Thread D: down() → count=0, SLEEP (wait)
  Thread A: up()   → count=1, wake Thread D
  Thread D: wakes  → count=0, proceed
```

---

## 18.2 Binary Semaphores

```
Binary semaphore: count = 1 (like a mutex, but different rules)

Difference from mutex:
  - Semaphore can be released by ANY thread (not just owner)
  - No ownership tracking
  - No priority inheritance
  - No optimistic spinning
  - No lockdep support

Rule: For new code, PREFER mutex over binary semaphore.
Semaphores are legacy in most use cases.
```

---

## 18.3 Counting Semaphores

```
Counting semaphore: Allows N concurrent accessors.
Useful for resource pools (DMA channels, buffer slots).

Example: 4 DMA channels available
  struct semaphore dma_sem;
  sema_init(&dma_sem, 4);  /* 4 resources */

  /* Acquire a DMA channel: */
  down(&dma_sem);           /* Decrements count, sleeps if 0 */
  use_dma_channel();
  up(&dma_sem);             /* Increments count, wakes waiter */
```

---

## 18.4 Semaphore Usage in Kernel

### API

```c
#include <linux/semaphore.h>

struct semaphore my_sem;
sema_init(&my_sem, count);       /* Initialize with count */
DEFINE_SEMAPHORE(my_sem);        /* Static, count=1 */

/* Acquire (decrement): */
down(&my_sem);                   /* Sleeps, uninterruptible */
down_interruptible(&my_sem);     /* Returns -EINTR on signal */
down_killable(&my_sem);          /* Returns -EINTR on fatal signal */
down_trylock(&my_sem);           /* Non-blocking: 0=acquired, 1=busy */
down_timeout(&my_sem, jiffies);  /* Timeout version */

/* Release (increment): */
up(&my_sem);                     /* Wakes one waiter */
```

### Semaphore vs Mutex

```
Feature              │ Semaphore      │ Mutex
─────────────────────┼────────────────┼──────────────────
Count                │ 0..N           │ 0 or 1
Ownership            │ None           │ Yes (owner only unlocks)
Priority inheritance │ No             │ No (rt_mutex: yes)
Optimistic spinning  │ No             │ Yes
Can up() from IRQ?   │ Yes            │ No (not recommended)
Lockdep support      │ No             │ Yes
Recommended?         │ Counting only  │ Yes (for binary)
```

---

## rw_semaphore (Reader-Writer Semaphore)

```c
#include <linux/rwsem.h>

struct rw_semaphore my_rwsem;
init_rwsem(&my_rwsem);
DECLARE_RWSEM(my_rwsem);

/* Reader (shared access): */
down_read(&my_rwsem);
/* ... read shared data ... */
up_read(&my_rwsem);

/* Writer (exclusive access): */
down_write(&my_rwsem);
/* ... modify shared data ... */
up_write(&my_rwsem);

/* Downgrade (writer → reader without releasing): */
downgrade_write(&my_rwsem);
/* Now holds read lock — others can also read */
up_read(&my_rwsem);
```

---

## Kernel Source References

```
Semaphore:
  kernel/locking/semaphore.c      ← Implementation
  include/linux/semaphore.h       ← API

rw_semaphore:
  kernel/locking/rwsem.c          ← Implementation
  include/linux/rwsem.h           ← API
```

---

## Interview Questions

1. **What is a counting semaphore? When would you use one?**
2. **Why is mutex preferred over binary semaphore for new code?**
3. **Can one thread down() and a different thread up() a semaphore?**
4. **What is rw_semaphore? How does it differ from rwlock_t?**
5. **What is downgrade_write() and when is it useful?**
6. **Give an example of counting semaphore usage in a driver (DMA channels).**
7. **Why don't semaphores support priority inheritance?**
8. **What return value does down_interruptible() give on signal?**

---

## Summary

- Semaphores are sleeping counters: counting (N accessors) or binary (1 accessor)
- For binary mutual exclusion, prefer mutex over semaphore (better features, lockdep)
- Counting semaphores useful for resource pools (DMA channels, buffer slots)
- rw_semaphore: sleeping reader-writer lock with writer priority and downgrade support
- Semaphore can be released by non-owner — flexibility but less safety

---

*Next: [Chapter 19 — Atomic Operations](Chapter_19_Atomic_Operations.md)*
