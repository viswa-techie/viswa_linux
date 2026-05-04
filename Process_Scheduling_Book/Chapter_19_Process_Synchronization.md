# Chapter 19: Process Synchronization

## Learning Goals
- Understand the critical section problem and race conditions
- Master kernel synchronization primitives (spinlocks, mutexes, semaphores, RCU)
- Learn when to use each primitive
- Understand memory barriers and atomic operations

---

## 19.1 Why Synchronization?

```
Race Condition Example:

  Thread A (CPU 0)              Thread B (CPU 1)
  ─────────────────            ─────────────────
  read counter (=5)
                               read counter (=5)
  counter = 5 + 1
                               counter = 5 + 1
  write counter (=6)
                               write counter (=6)

  Expected: 7    Actual: 6    ← RACE CONDITION

The problem: non-atomic read-modify-write on shared data

Sources of concurrency in Linux kernel:
  1. SMP — multiple CPUs executing simultaneously
  2. Preemption — kernel preemption switches tasks
  3. Interrupts — hardware interrupts at any point
  4. Softirqs/tasklets — deferred work
  5. Workqueues — kernel worker threads
```

---

## 19.2 Atomic Operations

Simplest synchronization — hardware-guaranteed atomic instructions:

```c
#include <linux/atomic.h>

atomic_t counter = ATOMIC_INIT(0);

atomic_inc(&counter);              /* counter++ (atomic) */
atomic_dec(&counter);              /* counter-- */
atomic_add(5, &counter);          /* counter += 5 */
int val = atomic_read(&counter);  /* read value */
atomic_set(&counter, 10);         /* set value */

/* Test-and-modify (returns old value): */
int old = atomic_fetch_add(1, &counter);

/* Compare-and-swap: */
atomic_cmpxchg(&counter, old_val, new_val);

/* Bit operations: */
set_bit(3, &flags);               /* atomic set bit 3 */
clear_bit(3, &flags);
int was_set = test_and_set_bit(3, &flags);

/* ARM64 implementation uses LDXR/STXR (load/store exclusive): */
/* x86_64 uses LOCK prefix (LOCK XADD, LOCK CMPXCHG) */
```

---

## 19.3 Spinlocks

**Busy-wait lock** — CPU spins until lock available. Used for short critical sections.

```c
#include <linux/spinlock.h>

static DEFINE_SPINLOCK(my_lock);

void critical_work(void)
{
    unsigned long flags;

    spin_lock_irqsave(&my_lock, flags);  /* Disable IRQs + acquire lock */
    /* Critical section — very short! */
    shared_data++;
    spin_unlock_irqrestore(&my_lock, flags);  /* Release + restore IRQs */
}
```

### Spinlock Variants

```
┌──────────────────────────┬──────────────┬────────────┬───────────────┐
│ Function                  │ Disables IRQ │ Disables   │ Use when      │
│                          │              │ preempt    │               │
├──────────────────────────┼──────────────┼────────────┼───────────────┤
│ spin_lock()              │ No           │ Yes        │ Process ctx   │
│                          │              │            │ only          │
├──────────────────────────┼──────────────┼────────────┼───────────────┤
│ spin_lock_bh()           │ Softirqs     │ Yes        │ Process +     │
│                          │              │            │ softirq       │
├──────────────────────────┼──────────────┼────────────┼───────────────┤
│ spin_lock_irq()          │ Yes          │ Yes        │ Process +     │
│                          │              │            │ hardirq       │
├──────────────────────────┼──────────────┼────────────┼───────────────┤
│ spin_lock_irqsave()      │ Yes (save)   │ Yes        │ Unknown IRQ   │
│                          │              │            │ state (safest)│
└──────────────────────────┴──────────────┴────────────┴───────────────┘

Rule: If data shared with interrupt handler → spin_lock_irqsave()
      If data shared with softirq → spin_lock_bh()
      If data shared only between process contexts → spin_lock()
```

### Deadlock Prevention

```
ABBA Deadlock:
  CPU 0: lock(A) → lock(B)
  CPU 1: lock(B) → lock(A)    ← DEADLOCK!

Prevention: Always acquire locks in consistent global order.

Self-deadlock:
  CPU 0: spin_lock(A) → (IRQ fires) → IRQ handler → spin_lock(A) ← DEAD!
  
Prevention: Use spin_lock_irqsave() if shared with IRQ handler.

Linux lockdep (CONFIG_LOCKDEP=y):
  Detects lock ordering violations at runtime
  Reports potential deadlocks before they happen
```

---

## 19.4 Mutexes (Sleeping Locks)

```c
#include <linux/mutex.h>

static DEFINE_MUTEX(my_mutex);

void workfn(void)
{
    mutex_lock(&my_mutex);        /* Sleep if contended */
    /* Critical section — can be longer */
    do_work_that_takes_time();
    mutex_unlock(&my_mutex);
}

/* Try without sleeping: */
if (mutex_trylock(&my_mutex)) {
    /* Got it */
    mutex_unlock(&my_mutex);
} else {
    /* Contended — handle gracefully */
}

/* Interruptible (can be interrupted by signals): */
if (mutex_lock_interruptible(&my_mutex) == -EINTR) {
    return -ERESTARTSYS;
}
```

### Spinlock vs Mutex

```
┌─────────────────┬────────────────────┬────────────────────┐
│ Property         │ Spinlock            │ Mutex              │
├─────────────────┼────────────────────┼────────────────────┤
│ Blocking         │ Busy-wait (spin)    │ Sleep              │
│ Context          │ Any (inc. IRQ)      │ Process only       │
│ Hold time        │ Very short (<1µs)   │ Can be long        │
│ Preemptible      │ No (disables)       │ Yes (sleeps)       │
│ Recursive        │ No (deadlock!)      │ No (use nested)    │
│ IRQ context      │ Yes (irqsave)       │ NO!                │
│ Memory alloc     │ Only GFP_ATOMIC     │ GFP_KERNEL OK      │
│ PI support       │ No                  │ Yes (rt_mutex)     │
│ In PREEMPT_RT    │ → rt_mutex          │ → rt_mutex         │
└─────────────────┴────────────────────┴────────────────────┘

Decision flow:
  In IRQ context?          → spinlock
  Hold < 1µs?             → spinlock  
  Need to sleep inside?   → mutex
  Shared with IRQ handler? → spinlock_irqsave
  Otherwise?              → mutex (preferred for process context)
```

---

## 19.5 Read-Write Locks

For data read often, written rarely:

```c
#include <linux/rwlock.h>

static DEFINE_RWLOCK(my_rwlock);

/* Multiple readers concurrently: */
void read_data(void)
{
    read_lock(&my_rwlock);
    /* Read shared_data — many readers OK */
    read_unlock(&my_rwlock);
}

/* Exclusive writer: */
void write_data(void)
{
    write_lock(&my_rwlock);
    /* Modify shared_data — exclusive access */
    write_unlock(&my_rwlock);
}

/* Mutex version (sleeping): */
#include <linux/rwsem.h>
static DECLARE_RWSEM(my_rwsem);

down_read(&my_rwsem);    /* shared read */
up_read(&my_rwsem);

down_write(&my_rwsem);   /* exclusive write */
up_write(&my_rwsem);
```

---

## 19.6 RCU (Read-Copy-Update)

The most scalable read-side synchronization — zero overhead for readers:

```
RCU Concept:

  Readers:  No locks at all! Just disable preemption (rcu_read_lock)
  Writers:  Copy data, modify copy, atomically switch pointer, wait
            for all readers to finish, then free old copy

  Timeline:
    Old data: [A=1, B=2]
    Writer: copy → [A=1, B=2]' → modify → [A=3, B=4]'
    Writer: rcu_assign_pointer(ptr, new_copy)
    Writer: synchronize_rcu() ← waits for all readers
    Writer: kfree(old_data)

    Readers during transition:
      Some see old [A=1, B=2]  ← consistent old view
      Some see new [A=3, B=4]  ← consistent new view
      NONE see [A=3, B=2]     ← no torn reads!
```

```c
#include <linux/rcupdate.h>

struct my_data {
    int value;
    struct rcu_head rcu;
};

static struct my_data __rcu *global_ptr;

/* Reader: */
void read_side(void)
{
    struct my_data *p;
    rcu_read_lock();              /* Disable preemption (almost free) */
    p = rcu_dereference(global_ptr);
    printk("value = %d\n", p->value);
    rcu_read_unlock();
}

/* Writer: */
void write_side(int new_val)
{
    struct my_data *old, *new;
    new = kmalloc(sizeof(*new), GFP_KERNEL);
    new->value = new_val;

    old = rcu_dereference_protected(global_ptr, lockdep_is_held(&update_lock));
    rcu_assign_pointer(global_ptr, new);  /* Atomic pointer update */

    synchronize_rcu();                    /* Wait for readers */
    kfree(old);                           /* Safe to free now */
    /* OR: call_rcu(&old->rcu, my_free_callback); for deferred free */
}
```

### When to Use RCU

```
RCU is ideal for:
  ✓ Read-mostly data structures (routing tables, module list)
  ✓ Pointer-based data (linked lists, trees)
  ✓ Where readers vastly outnumber writers

RCU is NOT for:
  ✗ Protecting non-pointer data (use atomic/spinlock)
  ✗ Write-heavy workloads (each write copies data)
  ✗ When readers need up-to-the-instant consistency
```

---

## 19.7 Completion Variables

For one-time "wait until done" synchronization:

```c
#include <linux/completion.h>

static DECLARE_COMPLETION(work_done);

/* Waiting side: */
void waiter(void)
{
    wait_for_completion(&work_done);  /* Sleep until signaled */
    /* OR: wait_for_completion_timeout(&work_done, msecs_to_jiffies(5000)); */
}

/* Signaling side: */
void worker(void)
{
    do_heavy_work();
    complete(&work_done);     /* Wake one waiter */
    /* OR: complete_all(&work_done);  Wake all waiters */
}
```

---

## 19.8 Memory Barriers

Prevent CPU and compiler from reordering memory operations:

```c
/* Compiler barrier: prevents compiler reordering */
barrier();

/* Full memory barrier: prevents CPU + compiler reordering */
mb();       /* read + write barrier */
rmb();      /* read barrier only */
wmb();      /* write barrier only */

/* SMP barriers (no-op on UP): */
smp_mb();
smp_rmb();
smp_wmb();

/* Example: message passing */
/* CPU 0 (writer): */
data = 42;
smp_wmb();          /* Ensure data written before flag */
flag = 1;

/* CPU 1 (reader): */
while (!flag);
smp_rmb();          /* Ensure flag read before data */
use(data);          /* Guaranteed to see 42 */
```

---

## 19.9 Synchronization Decision Tree

```
Need to protect shared data?
│
├─ Atomic variable (counter/flag)?
│   → atomic_t / atomic_long_t
│
├─ In interrupt context?
│   ├─ Shared with process context?
│   │   → spin_lock_irqsave()
│   └─ IRQ-only data?
│       → spin_lock() (within same IRQ level)
│
├─ Very short hold time (<1µs)?
│   └─ Any context?
│       → spinlock
│
├─ Read-mostly data structure?
│   └─ Pointer-based, can tolerate stale reads?
│       → RCU
│
├─ Need exclusive long access?
│   └─ Process context only?
│       → mutex (preferred) or semaphore
│
├─ Multiple readers, rare writers?
│   └─ → rwsem (process ctx) or rwlock (any ctx)
│
└─ One-shot "done" signaling?
    → completion
```

---

## Interview Questions

**Q1: Why can't you use a mutex in interrupt context?**
A: Mutexes may sleep when contended. Interrupt context cannot sleep because: 1) There's no backing `task_struct` to put to sleep (interrupt borrows the interrupted task's stack). 2) Sleeping would block the interrupted task and prevent interrupt completion. 3) Other interrupts may be masked, causing missed events. Use `spin_lock_irqsave()` instead.

**Q2: How does RCU achieve zero-overhead reads?**
A: RCU readers only call `rcu_read_lock()` / `rcu_read_unlock()`, which under typical configs just increment/decrement a per-CPU preemption counter (or are even no-ops on non-preemptible kernels). No locks, no atomic operations, no cache-line bouncing between CPUs. Writers pay the cost instead by copying data and waiting for a grace period.

**Q3: In PREEMPT_RT, spinlocks become sleeping locks. How does this affect synchronization?**
A: PREEMPT_RT converts `spinlock_t` to `rt_mutex`, which is sleepable and supports priority inheritance. This means: 1) Spinlock regions are preemptible. 2) Priority inversion is avoided. 3) Code holding a spinlock CANNOT run in hard IRQ context (must use `raw_spinlock_t` for truly non-preemptible needs). 4) Overall latency improves because high-priority tasks preempt low-priority lock holders instead of spinning.

---

*Next: [Chapter 20 — Inter-Process Communication (IPC)](Chapter_20_IPC.md)*
