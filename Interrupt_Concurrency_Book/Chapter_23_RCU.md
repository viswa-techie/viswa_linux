# Chapter 23: RCU (Read-Copy-Update)

## Learning Goals
- Understand the RCU paradigm: read-side lock-free, write-side deferred free
- Use rcu_read_lock/rcu_read_unlock and rcu_dereference
- Know synchronize_rcu, call_rcu, and grace period concepts
- Apply RCU to linked lists and pointer-published data
- Understand SRCU and RCU flavors

---

## 23.1 What Is RCU?

RCU is a synchronization mechanism optimized for **read-mostly** data. Readers pay almost zero cost; writers do the heavy lifting.

```
Key Insight:
  - Readers access data with NO locking (just rcu_read_lock)
  - Writers update by publishing a NEW version of data
  - Writers wait for all existing readers to finish (grace period)
  - Old data is freed AFTER all readers are done

                    Writer publishes new data
                    ┌───────────────────────┐
   OLD data ──────►│ NEW data replaces OLD  │
                    └──────────┬────────────┘
                               │
              ┌────────────────┼────────────────┐
              │   Grace Period │                 │
              │   (wait for    │                 │
              │    existing    │                 │
              │    readers)    │                 │
              ▼                ▼                 ▼
   Readers using OLD      Last reader       kfree(OLD)
   finish their            exits             safely
   rcu_read_unlock()
```

### Why RCU?

```
rwlock_t (reader-writer lock):
  - Readers acquire lock → cache-line bouncing on lock variable
  - Readers block writers → writer starvation possible
  - Scaling: O(N) for N reader CPUs (all touch same lock)

RCU:
  - Readers: rcu_read_lock() = preempt_disable() (nearly free)
  - No cache-line bouncing for readers
  - Scaling: O(1) for readers regardless of CPU count
  - Cost shifted to writers (grace period wait)
```

---

## 23.2 Core Concepts

### Grace Period

```
  CPU 0          CPU 1          CPU 2
  ──────         ──────         ──────
  rcu_read_lock()
  read old_ptr                  rcu_read_lock()
  ...                           read old_ptr
  rcu_read_unlock()             ...
                 ←── synchronize_rcu() begins ──→
                 Writer waits for ALL pre-existing readers
                                rcu_read_unlock()
                 ←── Grace period ends ──→
                 kfree(old_data)  ← Safe now!
```

```
Grace period guarantee:
  After synchronize_rcu() returns, ALL rcu_read_lock() sections
  that started BEFORE the grace period have completed.
  
  New rcu_read_lock() sections may see the new data.
```

### Publish-Subscribe Pattern

```c
/* Writer publishes new data: */
struct data *new = kmalloc(sizeof(*new), GFP_KERNEL);
new->value = 42;
rcu_assign_pointer(global_ptr, new);  /* Publish with release barrier */

/* Reader subscribes: */
rcu_read_lock();
struct data *p = rcu_dereference(global_ptr);  /* Load with acquire barrier */
if (p)
    use(p->value);                              /* Guaranteed to see 42 */
rcu_read_unlock();
```

---

## 23.3 Reader API

```c
#include <linux/rcupdate.h>

/* Enter RCU read-side critical section: */
rcu_read_lock();
/* On non-RT: preempt_disable() — almost free */
/* On RT: per-CPU reader counter increment */

/* Access RCU-protected pointer: */
struct data *p = rcu_dereference(ptr);
/* Compiler barrier + optional data-dependency barrier (Alpha) */
/* Required: cannot use raw pointer load */

/* Exit RCU read-side critical section: */
rcu_read_unlock();

/* Rules inside rcu_read_lock() section:
   - CANNOT sleep (non-PREEMPT_RT)
   - CANNOT call schedule()
   - CAN be preempted by IRQs
   - CAN nest rcu_read_lock()
*/
```

### What rcu_dereference() Does

```c
/* rcu_dereference(p) expands to roughly: */
({
    typeof(p) _p = READ_ONCE(p);    /* Prevent load tearing */
    smp_read_barrier_depends();      /* Alpha only (removed in 6.x+) */
    _p;
})

/* This ensures:
   1. Pointer is loaded exactly once
   2. Data-dependent loads after the pointer are ordered
   3. Compiler doesn't optimize away the load
*/
```

---

## 23.4 Writer API

```c
/* Publish new pointer (replaces old): */
rcu_assign_pointer(ptr, new);
/* Includes smp_store_release() — all writes to *new visible before ptr update */

/* Option 1: Synchronous wait (blocks until grace period ends): */
synchronize_rcu();
/* After return: safe to free old data */
kfree(old);

/* Option 2: Asynchronous (callback after grace period): */
call_rcu(&old->rcu_head, my_rcu_callback);
/* my_rcu_callback called after grace period — use for fast path */

void my_rcu_callback(struct rcu_head *head)
{
    struct data *old = container_of(head, struct data, rcu_head);
    kfree(old);
}

/* Convenience: kfree_rcu (no custom callback needed): */
kfree_rcu(old, rcu_head);
/* Or since 5.x, no rcu_head needed: */
kvfree_rcu(old);
```

### synchronize_rcu vs call_rcu

```
synchronize_rcu():
  - Blocks calling context until grace period ends
  - Simple but slow (milliseconds)
  - Use in module cleanup, rare updates
  - CANNOT be called from atomic context

call_rcu():
  - Non-blocking: queues callback for later
  - Fast return
  - Use in fast paths, frequent updates
  - Callback runs in softirq context
  - Requires struct rcu_head in data structure
```

---

## 23.5 RCU-Protected Linked Lists

RCU is most commonly used with linked lists.

```c
#include <linux/rculist.h>

struct my_entry {
    struct list_head list;
    struct rcu_head  rcu;
    int key;
    int value;
};

static LIST_HEAD(my_list);
static DEFINE_SPINLOCK(my_list_lock);  /* Protects writes only */

/* === Reader (lock-free): === */
void lookup(int key)
{
    struct my_entry *e;

    rcu_read_lock();
    list_for_each_entry_rcu(e, &my_list, list) {
        if (e->key == key) {
            pr_info("Found: %d\n", e->value);
            break;
        }
    }
    rcu_read_unlock();
}

/* === Writer (add): === */
void add_entry(int key, int value)
{
    struct my_entry *new = kmalloc(sizeof(*new), GFP_KERNEL);
    new->key = key;
    new->value = value;

    spin_lock(&my_list_lock);
    list_add_rcu(&new->list, &my_list);    /* Publish */
    spin_unlock(&my_list_lock);
}

/* === Writer (delete): === */
void delete_entry(int key)
{
    struct my_entry *e;

    spin_lock(&my_list_lock);
    list_for_each_entry(e, &my_list, list) {
        if (e->key == key) {
            list_del_rcu(&e->list);        /* Remove from list */
            spin_unlock(&my_list_lock);
            kfree_rcu(e, rcu);             /* Free after grace period */
            return;
        }
    }
    spin_unlock(&my_list_lock);
}

/* === Writer (replace): === */
void update_entry(int key, int new_value)
{
    struct my_entry *old, *new;

    new = kmalloc(sizeof(*new), GFP_KERNEL);

    spin_lock(&my_list_lock);
    list_for_each_entry(old, &my_list, list) {
        if (old->key == key) {
            new->key = old->key;
            new->value = new_value;
            list_replace_rcu(&old->list, &new->list);
            spin_unlock(&my_list_lock);
            kfree_rcu(old, rcu);
            return;
        }
    }
    spin_unlock(&my_list_lock);
    kfree(new);
}
```

### RCU List API

```
Standard list          │ RCU variant
───────────────────────┼───────────────────────
list_add()             │ list_add_rcu()
list_del()             │ list_del_rcu()
list_replace()         │ list_replace_rcu()
list_for_each_entry()  │ list_for_each_entry_rcu()
hlist_add_head()       │ hlist_add_head_rcu()
hlist_del()            │ hlist_del_rcu()
hlist_for_each_entry() │ hlist_for_each_entry_rcu()
```

---

## 23.6 RCU Internals

### Grace Period Detection

```
How does the kernel know ALL readers are done?

Classic RCU (Tree RCU):
  Every CPU must pass through a "quiescent state":
    - Context switch
    - Return to user space
    - Idle loop
  
  Once ALL CPUs have passed a QS → grace period complete.

  ┌──────────────────────────────────────────────────┐
  │    Tree RCU Hierarchy (4-CPU example):           │
  │                                                  │
  │              ┌─────────┐                         │
  │              │  root   │ All CPUs reported QS    │
  │              │  node   │ → grace period ends     │
  │              └────┬────┘                         │
  │           ┌───────┴────────┐                     │
  │      ┌────┴────┐     ┌────┴────┐                 │
  │      │ node 0  │     │ node 1  │                 │
  │      └────┬────┘     └────┬────┘                 │
  │      ┌────┴──┐       ┌────┴──┐                   │
  │     CPU0   CPU1    CPU2   CPU3                   │
  │      QS     QS      QS     QS   ← quiescent     │
  └──────────────────────────────────────────────────┘
```

### RCU Callbacks

```
  call_rcu(&obj->rcu_head, callback)
       │
       ▼
  Per-CPU callback list (segmented)
  ┌─────────┬─────────┬──────────┬──────────┐
  │  DONE   │  WAIT   │ NEXT_RDY │ NEXT     │
  │(execute)│(in GP)  │(next GP) │(newest)  │
  └─────────┴─────────┴──────────┴──────────┘
       │         │
       ▼         ▼
  Execute      Grace period
  callbacks    advances segments
```

---

## 23.7 SRCU (Sleepable RCU)

Standard RCU readers cannot sleep. SRCU allows sleeping in read-side critical sections.

```c
#include <linux/srcu.h>

DEFINE_SRCU(my_srcu);
/* Or dynamic: */
struct srcu_struct my_srcu;
init_srcu_struct(&my_srcu);

/* Reader (CAN sleep): */
int idx = srcu_read_lock(&my_srcu);
/* ... can sleep here ... */
/* ... can call kmalloc(GFP_KERNEL) ... */
srcu_read_unlock(&my_srcu, idx);

/* Writer: */
synchronize_srcu(&my_srcu);       /* Wait for SRCU grace period */
call_srcu(&my_srcu, &obj->rcu_head, callback);

/* Cleanup: */
cleanup_srcu_struct(&my_srcu);
```

### When to Use SRCU

```
Use SRCU when:
  - Read-side needs to sleep (I/O, allocation with GFP_KERNEL)
  - Domain-specific (not global) grace periods
  - Reader-writer pattern where readers need to sleep

Cost: Heavier than standard RCU (per-CPU counters per SRCU domain)
```

---

## 23.8 RCU Flavors Summary

```
Flavor              │ Reader API           │ Sleep OK? │ Grace Period
────────────────────┼──────────────────────┼───────────┼──────────────
RCU (Tree RCU)      │ rcu_read_lock()      │ No*       │ synchronize_rcu()
RCU-bh (deprecated) │ rcu_read_lock_bh()   │ No        │ synchronize_rcu()
RCU-sched (unified) │ rcu_read_lock_sched()│ No        │ synchronize_rcu()
SRCU                │ srcu_read_lock()     │ Yes       │ synchronize_srcu()
Tasks RCU           │ (voluntary sched)    │ Yes       │ synchronize_rcu_tasks()

* On PREEMPT_RT, rcu_read_lock readers CAN be preempted
  (but still cannot voluntarily sleep).

Since Linux 5.x: RCU-bh and RCU-sched unified into single RCU.
```

---

## 23.9 Common RCU Patterns

### Pattern 1: Module Parameter Protected by RCU

```c
static struct config __rcu *current_config;

void update_config(struct config *new)
{
    struct config *old;

    old = rcu_replace_pointer(current_config, new, true);
    kfree_rcu(old, rcu_head);
}

int read_config_value(void)
{
    struct config *cfg;
    int val;

    rcu_read_lock();
    cfg = rcu_dereference(current_config);
    val = cfg->value;
    rcu_read_unlock();
    return val;
}
```

### Pattern 2: RCU + Reference Counting

```c
struct object {
    struct rcu_head rcu;
    refcount_t      refcount;
    /* ... data ... */
};

/* Lookup without lock: */
struct object *find_object(int id)
{
    struct object *obj;

    rcu_read_lock();
    obj = rcu_dereference(global_table[id]);
    if (obj && !refcount_inc_not_zero(&obj->refcount))
        obj = NULL;  /* Object being freed */
    rcu_read_unlock();
    return obj;      /* Caller holds reference */
}

/* Release: */
void put_object(struct object *obj)
{
    if (refcount_dec_and_test(&obj->refcount))
        kfree_rcu(obj, rcu);
}
```

---

## 23.10 Common Mistakes

```
Mistake                                  │ Fix
─────────────────────────────────────────┼───────────────────────────
Accessing RCU pointer without            │ Use rcu_dereference()
  rcu_dereference()                      │
Freeing RCU-protected data immediately   │ Use kfree_rcu / call_rcu
Sleeping in rcu_read_lock section        │ Use SRCU
Forgetting write-side lock               │ Readers lock-free,
  (multiple writers race)                │ writers need spinlock
Using list_del instead of list_del_rcu   │ Use list_del_rcu
Not calling synchronize_rcu in           │ Call synchronize_rcu in
  module_exit                            │ module_exit before kfree
```

---

## Kernel Source References

```
RCU core:
  kernel/rcu/tree.c                     ← Tree RCU implementation
  kernel/rcu/update.c                   ← Core RCU APIs
  kernel/rcu/srcutree.c                 ← SRCU implementation
  include/linux/rcupdate.h              ← Main RCU API
  include/linux/rculist.h              ← RCU-protected list API
  include/linux/srcu.h                  ← SRCU API

Documentation:
  Documentation/RCU/                    ← Extensive RCU docs
  Documentation/RCU/whatisRCU.rst       ← Overview
  Documentation/RCU/rcu_dereference.rst ← Pointer access rules
  Documentation/RCU/listRCU.rst         ← RCU list usage
```

---

## Interview Questions

1. **What is RCU? How does it differ from reader-writer locks?**
2. **What is a grace period? How does the kernel detect it?**
3. **Explain rcu_read_lock(), rcu_dereference(), rcu_assign_pointer().**
4. **What is the difference between synchronize_rcu() and call_rcu()?**
5. **Why can't you sleep in an RCU read-side critical section?**
6. **What is SRCU and when would you use it?**
7. **How does RCU protect a linked list? Show the add/delete/lookup pattern.**
8. **What is kfree_rcu()? Why not just kfree() after list_del_rcu()?**
9. **How does RCU scale better than rwlock_t for readers?**
10. **What is the "read-copy-update" name referring to?**
11. **How does PREEMPT_RT affect RCU read-side?**
12. **What is Tree RCU? Why a tree hierarchy for grace period detection?**

---

## Summary

- RCU: readers are lock-free (near-zero cost), writers defer freeing until grace period
- rcu_read_lock/unlock: delimit read-side critical section (preempt_disable on non-RT)
- rcu_dereference: load RCU-protected pointer safely
- rcu_assign_pointer: publish new pointer with release semantics
- Grace period: time until all pre-existing readers complete; detected via quiescent states
- call_rcu / kfree_rcu: defer freeing; synchronize_rcu: block until safe
- RCU lists: list_add_rcu, list_del_rcu, list_for_each_entry_rcu
- SRCU: allows sleeping in read-side critical sections
- Writer still needs locking (spinlock) to serialize with other writers
- RCU is the most scalable read-side mechanism in the Linux kernel

---

*Next: [Chapter 24 — Concurrency in Drivers](Chapter_24_Concurrency_Drivers.md)*
