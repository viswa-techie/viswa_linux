# Chapter 21: Locking in IRQ Context

## Learning Goals
- Understand why locking in interrupt context requires special care
- Know the correct spinlock variants for each context combination
- Avoid deadlocks caused by interrupts preempting lock holders
- Apply the lock selection matrix in real driver code

---

## 21.1 The Interrupt Context Problem

When an interrupt fires, it preempts whatever code is running — including code holding a lock. If the interrupt handler tries to acquire the same lock, **deadlock**.

```
Classic Deadlock Scenario:

CPU 0 (process context):
  spin_lock(&my_lock);        ← Acquired
  /* ... critical section ... */
       ↓ IRQ fires on same CPU!
  ─────────────────────────────
  IRQ handler (hardirq context):
    spin_lock(&my_lock);      ← DEADLOCK! (spinning on lock held by
                                  preempted process context)
    /* Never reached */
```

### Solution: Disable IRQs While Holding the Lock

```c
/* Process context code: */
spin_lock_irqsave(&my_lock, flags);   /* Disable local IRQs + acquire */
/* ... safe from IRQ preemption on this CPU ... */
spin_unlock_irqrestore(&my_lock, flags); /* Restore IRQ state + release */

/* IRQ handler: */
spin_lock(&my_lock);                   /* IRQs already disabled in hardirq */
/* ... */
spin_unlock(&my_lock);
```

---

## 21.2 The Context Hierarchy

```
Priority (highest first):
  1. NMI        ← Cannot be masked, extremely restricted
  2. Hard IRQ   ← Preempts everything below
  3. Soft IRQ   ← Preempts process context
  4. Process    ← Lowest priority, fully preemptible

Preemption arrows:
  NMI → can preempt Hard IRQ, Soft IRQ, Process
  Hard IRQ → can preempt Soft IRQ, Process
  Soft IRQ → can preempt Process
  Process → preemptible by everything
```

---

## 21.3 Lock Variant Selection Matrix

### The Golden Rule

**If a lock is shared with a higher-priority context, the lower-priority side must disable that context.**

```
Lock shared between    │ Lower-priority side must use
───────────────────────┼────────────────────────────────────
Process ↔ Hard IRQ     │ spin_lock_irqsave()    (disable IRQs)
Process ↔ Soft IRQ     │ spin_lock_bh()         (disable softirqs)
Process ↔ Process      │ spin_lock() or mutex_lock()
Soft IRQ ↔ Hard IRQ    │ spin_lock_irqsave()    (disable IRQs)
Soft IRQ ↔ Soft IRQ    │ spin_lock()            (softirqs serialized per-CPU)
Hard IRQ ↔ Hard IRQ    │ spin_lock()            (IRQs already disabled)
```

### Complete Decision Flowchart

```
Start: Need to protect shared data
  │
  ├─ Is it shared with hardirq context?
  │    YES → spin_lock_irqsave() / spin_unlock_irqrestore()
  │          (in process context and softirq context)
  │          spin_lock() / spin_unlock()
  │          (in hardirq handler — IRQs already off)
  │
  ├─ Is it shared with softirq/tasklet context?
  │    YES → spin_lock_bh() / spin_unlock_bh()
  │          (in process context)
  │          spin_lock() / spin_unlock()
  │          (in softirq/tasklet handler)
  │
  ├─ Is it shared between process contexts only?
  │    YES → Can the critical section sleep?
  │          YES → mutex_lock() / mutex_unlock()
  │          NO  → spin_lock() / spin_unlock()
  │
  └─ Is it per-CPU data?
       YES → Use per-CPU APIs with preempt_disable()
             (see Chapter 22)
```

---

## 21.4 Detailed API Usage

### spin_lock_irqsave / spin_unlock_irqrestore

```c
unsigned long flags;

spin_lock_irqsave(&lock, flags);
/* IRQs disabled on local CPU, lock held */
/* Safe from hardirq preemption */
spin_unlock_irqrestore(&lock, flags);
/* IRQ state restored to what it was before */
```

Why `irqsave` instead of `irq`?

```
spin_lock_irq(&lock);      // Unconditionally disables IRQs
spin_unlock_irq(&lock);    // Unconditionally enables IRQs ← DANGER!

Problem: if IRQs were already disabled before spin_lock_irq(),
         spin_unlock_irq() ENABLES them — breaking caller's assumption.

spin_lock_irqsave(&lock, flags);     // Saves current IRQ state
spin_unlock_irqrestore(&lock, flags); // Restores to saved state ← SAFE

Rule: ALWAYS prefer irqsave/irqrestore unless you are 100% certain
      that IRQs are enabled when the lock is acquired.
```

### spin_lock_bh / spin_unlock_bh

```c
spin_lock_bh(&lock);
/* Softirqs disabled on local CPU, lock held */
/* Safe from tasklet/softirq preemption */
spin_unlock_bh(&lock);
```

### In Interrupt Handlers

```c
/* Hard IRQ handler: IRQs already disabled by entry code */
irqreturn_t my_irq_handler(int irq, void *data)
{
    struct my_dev *dev = data;

    spin_lock(&dev->lock);        /* Plain spin_lock is correct */
    /* ... update shared state ... */
    spin_unlock(&dev->lock);

    return IRQ_HANDLED;
}

/* Softirq / tasklet: softirqs serialized per-CPU */
void my_tasklet_fn(struct tasklet_struct *t)
{
    struct my_dev *dev = from_tasklet(dev, t, tasklet);

    spin_lock(&dev->lock);        /* Plain spin_lock if no hardirq sharing */
    spin_lock_irqsave(&dev->lock, flags); /* If also shared with hardirq */
    /* ... */
}
```

---

## 21.5 Complete Driver Example

```c
#include <linux/module.h>
#include <linux/interrupt.h>
#include <linux/spinlock.h>

struct my_device {
    spinlock_t      lock;
    u32             hw_status;
    struct list_head pending_buffers;
    int             irq;
};

/* === Hard IRQ Handler === */
static irqreturn_t my_irq_handler(int irq, void *data)
{
    struct my_device *dev = data;

    spin_lock(&dev->lock);          /* IRQs already off in hardirq */
    dev->hw_status = readl(dev->regs + STATUS_REG);
    if (dev->hw_status & IRQ_PENDING) {
        writel(IRQ_ACK, dev->regs + ACK_REG);
        /* Queue work for bottom half */
        tasklet_schedule(&dev->tasklet);
    }
    spin_unlock(&dev->lock);
    return IRQ_HANDLED;
}

/* === Tasklet (Soft IRQ Context) === */
static void my_tasklet_fn(struct tasklet_struct *t)
{
    struct my_device *dev = from_tasklet(dev, t, tasklet);
    unsigned long flags;

    spin_lock_irqsave(&dev->lock, flags);  /* Shared with hardirq! */
    process_completed_buffers(dev);
    spin_unlock_irqrestore(&dev->lock, flags);
}

/* === Process Context (ioctl) === */
static long my_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    struct my_device *dev = file->private_data;
    unsigned long flags;

    spin_lock_irqsave(&dev->lock, flags);  /* Shared with hardirq! */
    submit_buffer_to_hw(dev, arg);
    spin_unlock_irqrestore(&dev->lock, flags);

    return 0;
}

/* === Workqueue (Process Context, Can Sleep) === */
static void my_work_fn(struct work_struct *work)
{
    struct my_device *dev = container_of(work, struct my_device, work);
    unsigned long flags;

    /* Part 1: Quick lock access */
    spin_lock_irqsave(&dev->lock, flags);
    u32 status = dev->hw_status;
    spin_unlock_irqrestore(&dev->lock, flags);

    /* Part 2: Sleeping operations (outside spinlock) */
    if (status & NEED_FIRMWARE) {
        request_firmware(&fw, "my_fw.bin", dev->dev);  /* May sleep */
        /* ... */
    }
}
```

---

## 21.6 Nested Locking and Lock Ordering

When multiple locks are needed across contexts:

```
Rule: Always acquire locks in the SAME order everywhere.

Example: device has two locks
  lock_a: shared between process and hardirq
  lock_b: shared between process contexts only

/* Correct ordering: lock_a before lock_b */

Process context:
  spin_lock_irqsave(&lock_a, flags);
  spin_lock(&lock_b);                   /* IRQs already off via lock_a */
  /* ... */
  spin_unlock(&lock_b);
  spin_unlock_irqrestore(&lock_a, flags);

IRQ handler:
  spin_lock(&lock_a);                   /* Only needs lock_a */
  /* ... */
  spin_unlock(&lock_a);

WRONG (deadlock risk):
  spin_lock(&lock_b);                   /* IRQs still on! */
  spin_lock_irqsave(&lock_a, flags);   /* IRQ between these two
                                           locks → lock_a deadlock */
```

### lockdep Annotations

```c
/* Tell lockdep about IRQ context usage: */
spin_lock_irqsave(&dev->lock, flags);
/* lockdep marks this lock as "IRQ-safe" */

/* If you later do: */
spin_lock(&dev->lock);
/* lockdep WARNS: "inconsistent lock state"
   because the lock is used with IRQs both on and off */
```

---

## 21.7 NMI Context

NMI (Non-Maskable Interrupt) cannot be disabled. Standard locking is impossible.

```
NMI-safe primitives:
  - atomic operations (atomic_inc, etc.)
  - Per-CPU variables with exclusive NMI access
  - Lock-free data structures
  - raw_spin_trylock() — ONLY trylock (never block)

CANNOT use in NMI:
  - spin_lock() (may deadlock — NMI could preempt lock holder)
  - mutex_lock()
  - sleeping functions
  - printk() (use nmi_backtrace, trace_printk instead)
```

```c
/* NMI handler pattern: */
static int my_nmi_handler(unsigned int type, struct pt_regs *regs)
{
    /* Only trylock — must not spin: */
    if (raw_spin_trylock(&nmi_lock)) {
        /* Got the lock — update stats */
        per_cpu(nmi_count, smp_processor_id())++;
        raw_spin_unlock(&nmi_lock);
    }
    /* If trylock fails, skip — NMI must be fast */
    return NMI_HANDLED;
}
```

---

## 21.8 PREEMPT_RT Considerations

On PREEMPT_RT:
- `spin_lock()` becomes a sleeping lock (rt_mutex)
- `spin_lock_irqsave()` does NOT disable hardware IRQs
- Only `raw_spin_lock_irqsave()` disables real IRQs

```
Standard Linux:                    PREEMPT_RT:
  spin_lock()        → real spin   spin_lock()       → rt_mutex (sleeping)
  spin_lock_irqsave  → disable IRQ spin_lock_irqsave → rt_mutex (NO real IRQ disable)
  raw_spin_lock()    → real spin   raw_spin_lock()   → real spin
  raw_spin_lock_irqsave → disable  raw_spin_lock_irqsave → disable IRQ

Rule on PREEMPT_RT:
  Use raw_spin_lock* ONLY when truly in hardirq context
  and cannot sleep (interrupt controller code, scheduler).
```

---

## 21.9 Common Pitfalls

```
Pitfall                                │ Fix
───────────────────────────────────────┼────────────────────────────────
Using spin_lock() with IRQ handler     │ Use spin_lock_irqsave()
Using spin_lock_irq() when IRQs may   │ Use spin_lock_irqsave()
  already be disabled                  │
Sleeping while holding spinlock_irqsave│ Release lock before sleep
Calling kmalloc(GFP_KERNEL) in IRQ     │ Use GFP_ATOMIC
Using mutex in atomic context          │ Use spinlock
Forgetting bh disable with softirq    │ Use spin_lock_bh()
Not pairing irqsave with irqrestore   │ Always use matching pair
```

---

## Kernel Source References

```
Locking primitives:
  include/linux/spinlock.h              ← spin_lock API
  include/linux/spinlock_types.h        ← spinlock_t definition
  kernel/locking/spinlock.c             ← Implementation
  include/linux/interrupt.h             ← local_irq_save/restore

PREEMPT_RT locking:
  include/linux/spinlock_rt.h           ← RT spin_lock → rt_mutex
  kernel/locking/rtmutex.c              ← rt_mutex implementation

Lockdep:
  kernel/locking/lockdep.c              ← Lock dependency validator
  include/linux/lockdep.h               ← Annotations
```

---

## Interview Questions

1. **Why can't you use spin_lock() for data shared between process context and hardirq?**
2. **What is the difference between spin_lock_irq() and spin_lock_irqsave()?**
3. **When should you use spin_lock_bh()?**
4. **In a hard IRQ handler, which spin_lock variant should you use? Why?**
5. **What locking is safe in NMI context?**
6. **How does lockdep detect inconsistent IRQ lock usage?**
7. **On PREEMPT_RT, what does spin_lock_irqsave() actually do?**
8. **Can you hold a mutex in softirq context? Process context? Why?**
9. **Write a driver with correct locking across process, tasklet, and IRQ contexts.**
10. **What is lock ordering? How do you handle nested locks with different IRQ properties?**

---

## Summary

- Interrupts preempt lock holders — if the handler needs the same lock, DEADLOCK
- Process ↔ hardirq: use spin_lock_irqsave / spin_unlock_irqrestore
- Process ↔ softirq: use spin_lock_bh / spin_unlock_bh
- Always prefer irqsave over irq (saves and restores interrupt state)
- NMI: only trylock, atomics, per-CPU data
- PREEMPT_RT changes spin_lock semantics — raw_spin_lock for real hardware disable
- Lock ordering must be consistent across all contexts to prevent deadlocks

---

*Next: [Chapter 22 — Per-CPU Data](Chapter_22_Per_CPU_Data.md)*
