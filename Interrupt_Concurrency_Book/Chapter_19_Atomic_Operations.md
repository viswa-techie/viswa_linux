# Chapter 19: Atomic Operations

## Learning Goals
- Understand how atomic operations avoid locks for simple counters/flags
- Use atomic_t, atomic64_t, and atomic_long_t correctly
- Know the full atomic API including bitwise and test-and-set
- Understand refcount_t and why it replaced atomic_t for reference counting
- Use atomic_t with memory ordering awareness

---

## 19.1 What Are Atomic Operations?

An **atomic operation** completes in a single indivisible step — no other CPU can observe it half-done. This eliminates the need for locks when manipulating simple integers or bits.

```
Without atomic:                    With atomic:
  CPU0: load counter → 5            CPU0: atomic_inc(&counter) → 6
  CPU1: load counter → 5            CPU1: atomic_inc(&counter) → 7
  CPU0: increment → 6               Result: correct (7)
  CPU1: increment → 6
  CPU0: store → 6
  CPU1: store → 6
  Result: 6 (LOST UPDATE!)
```

### Hardware Implementation

```
x86:
  LOCK prefix + ADD/INC/CMPXCHG → Bus lock or cache-line lock
  Example: lock incl (%rdi)

ARM64:
  LDXR/STXR (load-exclusive / store-exclusive) loops
  LSE atomics (ARMv8.1+): LDADD, STADD, CAS → single instruction
  Example: ldadd w1, w0, [x2]
```

---

## 19.2 atomic_t — 32-bit Atomic Integer

### Declaration

```c
#include <linux/atomic.h>

atomic_t counter = ATOMIC_INIT(0);    /* Static initialization */

/* Or runtime: */
atomic_t counter;
atomic_set(&counter, 0);
```

### Complete API

```c
/* Read / Write: */
int val = atomic_read(&counter);      /* Read current value */
atomic_set(&counter, 42);             /* Set value */

/* Arithmetic: */
atomic_inc(&counter);                 /* counter++ */
atomic_dec(&counter);                 /* counter-- */
atomic_add(5, &counter);             /* counter += 5 */
atomic_sub(3, &counter);             /* counter -= 3 */

/* Arithmetic + Return new value: */
int new = atomic_inc_return(&counter);
int new = atomic_dec_return(&counter);
int new = atomic_add_return(5, &counter);
int new = atomic_sub_return(3, &counter);

/* Test + Operate: */
bool zero = atomic_dec_and_test(&counter);   /* --counter == 0? */
bool zero = atomic_inc_and_test(&counter);   /* ++counter == 0? */
bool zero = atomic_sub_and_test(n, &counter);/* (counter -= n) == 0? */
bool neg  = atomic_add_negative(n, &counter);/* (counter += n) < 0? */

/* Compare-and-Swap (CAS): */
int old = atomic_cmpxchg(&counter, expected, new_val);
/* If counter == expected → set to new_val, return old value */

/* Exchange: */
int old = atomic_xchg(&counter, new_val);
/* Set to new_val, return old value unconditionally */

/* Fetch-and-Op (returns OLD value): */
int old = atomic_fetch_add(5, &counter);
int old = atomic_fetch_sub(3, &counter);
int old = atomic_fetch_and(mask, &counter);
int old = atomic_fetch_or(mask, &counter);
int old = atomic_fetch_xor(mask, &counter);
```

---

## 19.3 atomic64_t — 64-bit Atomic Integer

```c
#include <linux/atomic.h>

atomic64_t counter = ATOMIC64_INIT(0);
atomic64_set(&counter, 0);

/* Same API pattern as atomic_t: */
s64 val = atomic64_read(&counter);
atomic64_inc(&counter);
atomic64_add(val, &counter);
s64 old = atomic64_cmpxchg(&counter, expected, new_val);
/* ... all variants available ... */
```

On 32-bit architectures, atomic64_t uses spinlock fallback.

---

## 19.4 Atomic Bitwise Operations

```c
#include <linux/bitops.h>

unsigned long flags = 0;

set_bit(3, &flags);           /* Set bit 3 atomically */
clear_bit(3, &flags);         /* Clear bit 3 atomically */
change_bit(3, &flags);        /* Toggle bit 3 atomically */

/* Test + Set (return OLD value of bit): */
bool was_set = test_and_set_bit(3, &flags);
bool was_set = test_and_clear_bit(3, &flags);
bool was_set = test_and_change_bit(3, &flags);

/* Non-atomic versions (faster, for single-CPU scenarios): */
__set_bit(3, &flags);
__clear_bit(3, &flags);
__test_and_set_bit(3, &flags);

/* Test (non-modifying): */
bool is_set = test_bit(3, &flags);
```

### Common Pattern: State Machine Flags

```c
/* Device states */
#define DEV_STATE_OPEN     0
#define DEV_STATE_RUNNING  1
#define DEV_STATE_ERROR    2

struct my_device {
    unsigned long state;
};

/* Only one thread wins the open race: */
if (test_and_set_bit(DEV_STATE_OPEN, &dev->state))
    return -EBUSY;  /* Already open */
```

---

## 19.5 Memory Ordering with Atomics

Atomic operations have **relaxed ordering** by default on some architectures (ARM64). The kernel provides explicit ordering variants:

```
Ordering Suffixes:
  No suffix     → relaxed (no barriers)
  _acquire      → load-acquire: operations after cannot move before
  _release      → store-release: operations before cannot move after

Fully-ordered Macros (implicit full barrier):
  atomic_add_return()     ← full barrier
  atomic_cmpxchg()        ← full barrier
  atomic_dec_and_test()   ← full barrier
  test_and_set_bit()      ← full barrier
```

### Ordering Example

```c
/* Producer-consumer with atomic flag: */

/* Producer: */
data = compute_result();                    /* Must complete first */
smp_store_release(&result_ready, 1);        /* Release: data visible before flag */

/* Consumer: */
while (!smp_load_acquire(&result_ready))    /* Acquire: flag read before data */
    cpu_relax();
use(data);                                  /* Guaranteed to see producer's data */
```

---

## 19.6 refcount_t — Safe Reference Counting

`refcount_t` replaced raw `atomic_t` for reference counting to prevent use-after-free exploits caused by counter overflow/underflow.

```c
#include <linux/refcount.h>

struct my_object {
    refcount_t refcount;
    /* ... */
};

/* Initialize: */
refcount_set(&obj->refcount, 1);

/* Increment (checked): */
refcount_inc(&obj->refcount);              /* Saturates at REFCOUNT_SATURATED */
bool ok = refcount_inc_not_zero(&obj->refcount);  /* Returns false if already 0 */

/* Decrement + test: */
if (refcount_dec_and_test(&obj->refcount))
    free_object(obj);                      /* Last reference dropped */

/* Read: */
unsigned int val = refcount_read(&obj->refcount);
```

### Why Not atomic_t for Refcounting?

```
Problem: atomic_t allows wrap-around (INT_MAX+1 = negative)

Attack scenario:
  Thread A: atomic_dec_and_test() → 0 → free object
  Thread B: (raced) atomic_inc() → 1 on freed memory → use-after-free

refcount_t protections:
  1. Saturates instead of overflowing
  2. WARN on inc from 0 (object already freed)
  3. WARN on underflow below 0
  4. CONFIG_REFCOUNT_FULL: additional checks (older kernels)
  5. Since 5.x: always-on checks (no config needed)
```

ASCII diagram of refcount lifecycle:

```
  refcount_set(1)
       │
       ▼
  ┌─────────┐
  │ Active   │ refcount >= 1
  │          │◄──── refcount_inc() / refcount_inc_not_zero()
  └────┬─────┘
       │ refcount_dec_and_test() → true (count reaches 0)
       ▼
  ┌─────────┐
  │ Dead     │ refcount == 0
  │          │ → Free object
  └─────────┘
       │
       ▼ refcount_inc() on dead object
  ┌─────────┐
  │ WARN!   │ Bug detected — potential use-after-free
  └─────────┘
```

---

## 19.7 Lock-Free Programming Patterns

### Pattern 1: Atomic Flag

```c
static atomic_t initialization_done = ATOMIC_INIT(0);

void ensure_init(void)
{
    if (atomic_cmpxchg(&initialization_done, 0, 1) == 0) {
        /* We won the race — do initialization */
        do_init();
        smp_wmb();  /* Ensure init visible before flag */
        atomic_set(&initialization_done, 2);  /* Signal completion */
    } else {
        /* Wait for init to complete */
        while (atomic_read(&initialization_done) != 2)
            cpu_relax();
        smp_rmb();  /* Ensure we see the initialized data */
    }
}
```

### Pattern 2: Atomic Statistics Counter

```c
struct driver_stats {
    atomic64_t bytes_read;
    atomic64_t bytes_written;
    atomic_t   errors;
};

/* In driver read path (no lock needed): */
atomic64_add(count, &stats->bytes_read);

/* In error path: */
atomic_inc(&stats->errors);

/* In sysfs show: */
seq_printf(s, "bytes_read: %lld\n", atomic64_read(&stats->bytes_read));
```

### Pattern 3: Sequence Generator

```c
static atomic_t sequence = ATOMIC_INIT(0);

int get_next_sequence(void)
{
    return atomic_inc_return(&sequence);  /* Unique, monotonic */
}
```

---

## 19.8 Complete Driver Example

```c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/atomic.h>

#define MAX_OPENS 3

struct my_device {
    atomic_t        open_count;
    atomic64_t      total_bytes;
    unsigned long   flags;
    refcount_t      refcount;
};

static struct my_device *g_dev;

static int my_open(struct inode *inode, struct file *file)
{
    /* Limit concurrent opens: */
    int old = atomic_read(&g_dev->open_count);
    do {
        if (old >= MAX_OPENS)
            return -EBUSY;
    } while (!atomic_try_cmpxchg(&g_dev->open_count, &old, old + 1));

    /* Take a reference: */
    refcount_inc(&g_dev->refcount);
    file->private_data = g_dev;
    return 0;
}

static int my_release(struct inode *inode, struct file *file)
{
    struct my_device *dev = file->private_data;

    atomic_dec(&dev->open_count);

    if (refcount_dec_and_test(&dev->refcount))
        kfree(dev);  /* Last reference */
    return 0;
}

static ssize_t my_read(struct file *file, char __user *buf,
                        size_t count, loff_t *ppos)
{
    struct my_device *dev = file->private_data;

    /* ... actual read ... */

    atomic64_add(count, &dev->total_bytes);
    return count;
}
```

---

## 19.9 Comparison Table

```
Feature            │ atomic_t       │ refcount_t     │ Bitwise ops
───────────────────┼────────────────┼────────────────┼────────────────
Purpose            │ General counter│ Ref counting   │ Flag management
Overflow check     │ No (wraps)     │ Yes (saturates)│ N/A
Underflow check    │ No             │ Yes (WARN)     │ N/A
Sleep allowed?     │ Yes (non-block)│ Yes            │ Yes
IRQ context?       │ Yes            │ Yes            │ Yes
Ordering           │ Relaxed*       │ Release+Acquire│ Full barrier**
Lock-free?         │ Yes            │ Yes            │ Yes

* _return variants have full barriers
** test_and_*_bit have full barriers; set_bit/clear_bit have release semantics
```

---

## Kernel Source References

```
Atomic operations:
  include/linux/atomic.h                  ← Core API
  include/linux/atomic/atomic-instrumented.h ← KASAN wrappers
  arch/x86/include/asm/atomic.h           ← x86 implementation
  arch/arm64/include/asm/atomic.h         ← ARM64 implementation

refcount_t:
  lib/refcount.c                          ← Implementation
  include/linux/refcount.h                ← API

Bitwise:
  include/linux/bitops.h                  ← API
  include/asm-generic/bitops/atomic.h     ← Generic
```

---

## Interview Questions

1. **What makes an operation "atomic"? How is it implemented on x86 vs ARM64?**
2. **When should you use atomic_t vs a spinlock?**
3. **What is the difference between atomic_add() and atomic_fetch_add()?**
4. **Why was refcount_t introduced? What bug does it prevent?**
5. **What happens if you call refcount_inc() on an object with refcount 0?**
6. **Explain test_and_set_bit(). How is it used for state machines?**
7. **What memory ordering do atomic_inc_return() and atomic_read() provide?**
8. **How does atomic_cmpxchg() work? Give a lock-free pattern using it.**
9. **Why are non-atomic __set_bit() variants available?**
10. **What is the performance cost of atomic operations on multi-core CPUs?**

---

## Summary

- Atomic ops provide lock-free manipulation of integers and bits
- atomic_t: general 32-bit counter; atomic64_t: 64-bit variant
- Test-and-set bit ops: perfect for state machines and flags
- refcount_t: hardened reference counter that detects overflow/underflow
- Memory ordering is relaxed by default; use _return/_acquire/_release for ordering
- For simple counters/flags, atomics are faster than locks; for complex multi-variable updates, use locks

---

*Next: [Chapter 20 — Memory Barriers and Ordering](Chapter_20_Memory_Barriers.md)*
