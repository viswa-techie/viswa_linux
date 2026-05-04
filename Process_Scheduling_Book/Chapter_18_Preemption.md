# Chapter 18: Preemption

## Learning Goals
- Understand kernel preemption models and their trade-offs
- Learn how TIF_NEED_RESCHED triggers rescheduling
- Know voluntary vs involuntary preemption points
- Master PREEMPT_RT for deterministic latency

---

## 18.1 What is Preemption?

```
Preemption = OS forcibly taking CPU from a running task to give to another

User-space preemption (always enabled):
  Timer interrupt fires → kernel checks → returns to different task
  User code has NO say — it's preempted transparently

Kernel preemption (configurable):
  Can the kernel ITSELF be preempted while executing kernel code?
  This is what "preemption models" control.

Why kernel preemption matters:
  Without: syscall or IRQ handler runs to completion before scheduling
  With:    kernel code can be interrupted for higher-priority tasks
  
  Impact: Lower latency for interactive/RT, but more complexity
```

---

## 18.2 Preemption Models

```
┌───────────────────────────────────────────────────────────────┐
│ Model               │ Preempt kernel? │ Latency │ Throughput │
├───────────────────────────────────────────────────────────────┤
│ PREEMPT_NONE        │ Never (only at  │ Worst   │ Best       │
│ (Server)            │  explicit points│ ~10ms   │            │
├───────────────────────────────────────────────────────────────┤
│ PREEMPT_VOLUNTARY   │ At might_sleep()│ Better  │ Good       │
│ (Desktop default)   │  + cond_resched │ ~1-4ms  │            │
├───────────────────────────────────────────────────────────────┤
│ PREEMPT             │ Anywhere except │ Good    │ Slightly   │
│ (Low-latency)       │  spinlocks and  │ ~0.5ms  │ less       │
│                     │  preempt_disable│         │            │
├───────────────────────────────────────────────────────────────┤
│ PREEMPT_RT          │ Even in "spin-  │ Best    │ Lowest     │
│ (Hard real-time)    │  locks" (now RT │ ~50µs   │            │
│                     │  mutexes)       │         │            │
└───────────────────────────────────────────────────────────────┘
```

### Kernel Config

```
CONFIG_PREEMPT_NONE=y       # Throughput-optimized server
CONFIG_PREEMPT_VOLUNTARY=y  # Good balance (Ubuntu desktop default)
CONFIG_PREEMPT=y            # Low-latency (audio workstations)
CONFIG_PREEMPT_RT=y         # Hard real-time (industrial, automotive)
```

---

## 18.3 TIF_NEED_RESCHED — The Rescheduling Flag

The central mechanism: a single bit in the current task's `thread_info` flags:

```c
/* include/linux/thread_info.h */
#define TIF_NEED_RESCHED    3  /* bit position varies by arch */

static inline void set_tsk_need_resched(struct task_struct *tsk)
{
    set_tsk_thread_flag(tsk, TIF_NEED_RESCHED);
}

static inline int need_resched(void)
{
    return unlikely(test_thread_flag(TIF_NEED_RESCHED));
}
```

### Who Sets TIF_NEED_RESCHED?

```
1. scheduler_tick() — timer interrupt
   If current task has run too long → set flag

2. try_to_wake_up() — waking a higher-priority task
   If woken task should preempt current → set flag

3. sched_setscheduler() — priority change
   If task now has higher priority → set flag

4. signal delivery — pending signal for current task

5. RT push/pull — when RT task should migrate
```

### Who Checks TIF_NEED_RESCHED?

```
1. Return from interrupt/exception (ALWAYS checked):
   irq_exit() → checks flag → calls schedule()

   /* arch/arm64/kernel/entry.S */
   ret_to_user:
       ldr x1, [tsk, #TSK_TI_FLAGS]
       tbnz x1, #TIF_NEED_RESCHED, el0_svc_preempt

2. Return from syscall (ALWAYS checked):
   syscall_exit → same path as interrupt return

3. preempt_enable() (PREEMPT=y only):
   If flag is set when preempt_count drops to 0 → schedule()

4. Explicit cond_resched() (PREEMPT_VOLUNTARY):
   Voluntary preemption points in long kernel paths
```

---

## 18.4 Preemption in Detail

### PREEMPT_NONE: No Kernel Preemption

```
Syscall or IRQ handler path:

  User code → syscall_enter → kernel work → syscall_exit → CHECK flag
                                                              │
                                                         if TIF_NEED_RESCHED
                                                              → schedule()
                                                         else
                                                              → return to user

  No preemption during kernel code!
  Long kernel paths (e.g., file copy) → high latency

  Only preemption points:
    1. Return to user space (from syscall/interrupt)
    2. Explicit schedule() calls
    3. sleep/wait functions
```

### PREEMPT_VOLUNTARY: Explicit Preemption Points

```c
/* Added throughout long kernel paths: */
cond_resched();
/* Expands to: */
if (unlikely(need_resched())) {
    preempt_schedule_common();
}

/* Also: might_sleep() macro checks (in debug builds) */

/* ~400 cond_resched() calls scattered through kernel:
 *   mm/filemap.c       — page cache operations
 *   fs/namei.c         — path resolution
 *   mm/mmap.c          — memory mapping
 *   kernel/softirq.c   — softirq processing
 *   net/core/dev.c     — packet processing
 */
```

### PREEMPT: Full Kernel Preemption

```c
/* preempt_count — nesting counter */
/* include/linux/preempt.h */

preempt_disable();   /* preempt_count++ */
  /* critical section — NOT preemptible */
  spin_lock(&lock);    /* preempt_count++ (implicit) */
    /* locked region */
  spin_unlock(&lock);  /* preempt_count-- */
preempt_enable();    /* preempt_count-- → if 0, check TIF_NEED_RESCHED */

/* Preemption only blocked when preempt_count > 0 */
/* preempt_count encodes: preempt nesting + softirq + hardirq + NMI */
```

```
preempt_count bit layout (32-bit):

  Bits 0-7:    Preemption disable nesting (0-255)
  Bits 8-15:   Softirq disable nesting
  Bits 16-19:  Hardirq nesting
  Bit 20:      NMI flag
  Bit 21:      PREEMPT_NEED_RESCHED (optimization)

  Preemptible when: preempt_count == 0 && !in_irq() && !in_softirq()
```

### Preemption Flow (CONFIG_PREEMPT=y)

```
Interrupt fires while in kernel code:

  ┌─── Kernel code executing ─────────────┐
  │                                         │
  │  ← IRQ fires here                      │
  │  irq_enter() → handle IRQ              │
  │  irq_exit():                           │
  │    if (preempt_count == 0 &&           │
  │        TIF_NEED_RESCHED set)           │
  │      → preempt_schedule_irq()          │
  │        → __schedule()                  │
  │          → pick_next_task()            │
  │          → context_switch()            │
  │                                         │
  │  OR: kernel code calls preempt_enable()│
  │    if (preempt_count drops to 0 &&     │
  │        TIF_NEED_RESCHED set)           │
  │      → preempt_schedule()              │
  └─────────────────────────────────────────┘
```

---

## 18.5 PREEMPT_RT Deep Dive

PREEMPT_RT converts almost all non-preemptible sections to preemptible ones:

```
Key changes:

1. spinlock_t → rt_mutex (sleepable, priority-inheriting)
   raw_spinlock_t → actual spinning lock (only for truly atomic paths)

2. Threaded interrupts:
   All IRQ handlers run as kernel threads with configurable priority
   
   $ ps aux | grep irq/
   root  [irq/18-i2c]       SCHED_FIFO priority 50
   root  [irq/44-eth0]      SCHED_FIFO priority 50
   root  [irq/130-xhci]     SCHED_FIFO priority 50

3. Softirq in thread context:
   ksoftirqd runs softirqs — preemptible and schedulable

4. local_bh_disable → preempt_disable (still preemptible in RT)

Remaining NON-preemptible sections (raw_spinlock):
  - Scheduler locks (rq->lock)
  - Timer handling
  - Very short critical paths in interrupt entry/exit
```

### Latency Comparison

```
Worst-case interrupt-to-handler latency:

  PREEMPT_NONE:      1,000 - 100,000 µs (1-100ms)
  PREEMPT_VOLUNTARY: 500 - 10,000 µs
  PREEMPT:           100 - 1,000 µs
  PREEMPT_RT:        10 - 100 µs

  ┌──────────────────────────────────────────────┐
  │  Latency Distribution (log scale)             │
  │                                                │
  │  PREEMPT_NONE:        ████████████████░░░░░░░ │
  │  PREEMPT_VOLUNTARY:   ██████████░░            │
  │  PREEMPT:             █████░                   │
  │  PREEMPT_RT:          ██░                      │
  │                       ↑                        │
  │                    10µs   100µs  1ms  10ms     │
  └──────────────────────────────────────────────┘
```

---

## 18.6 Voluntary Preemption (cond_resched)

```c
/* Places where kernel MUST check for preemption in long paths: */

/* Example: memory allocation loop */
static int scan_pages(struct zone *zone)
{
    struct page *page;
    unsigned long count = 0;

    list_for_each_entry(page, &zone->lru, lru) {
        process_page(page);
        if (++count % 32 == 0)
            cond_resched();   /* <── voluntary preemption point */
    }
}

/* Example: file system traversal */
int iterate_dir(struct file *file, struct dir_context *ctx)
{
    /* ... */
    while (has_more_entries) {
        emit_entry(ctx);
        cond_resched();       /* <── voluntary preemption point */
    }
}
```

---

## 18.7 Preemption and Locking

```
Lock type behavior across preemption models:

                    PREEMPT_NONE    PREEMPT     PREEMPT_RT
  ─────────────────────────────────────────────────────────
  spin_lock()       Disable preempt  Disable     rt_mutex (sleep OK)
  raw_spin_lock()   Disable preempt  Disable     Disable preempt
  mutex_lock()      May sleep        May sleep   May sleep
  rw_lock()         Disable preempt  Disable     rt_rwlock (sleep OK)
  local_irq_disable Disable IRQs    Disable IRQs Disable IRQs

  In PREEMPT_RT: spin_lock() becomes sleepable!
  → Cannot use spin_lock() in hard IRQ context
  → Use raw_spin_lock() for actual critical atomic sections
```

---

## 18.8 preempt_lazy — Emerging Approach

Linux 6.x introduces lazy preemption concept to unify models:

```
Traditional: TIF_NEED_RESCHED checked at many points
Lazy:        TIF_NEED_RESCHED_LAZY checked only on return to user

  Normal task wakeup → set TIF_NEED_RESCHED_LAZY
    → Preemption happens on return to user (lower overhead)
  
  RT task wakeup → set TIF_NEED_RESCHED (immediate)
    → Preemption happens at next preemption point

  Benefits:
    - Throughput tasks preempted less often
    - RT tasks still get immediate preemption
    - Single kernel binary serves both use cases
```

---

## Interview Questions

**Q1: On an Android phone, which preemption model is typically used?**
A: Most Android kernels use **PREEMPT** (full preemption) for good UI responsiveness. Some high-end devices targeting audio/automotive use cases may use **PREEMPT_RT**. Android requires low-latency scheduling for 60/120 fps rendering and audio processing (~5ms buffer), making PREEMPT_VOLUNTARY insufficient.

**Q2: Why can't you call `schedule()` while holding a spinlock?**
A: With `CONFIG_PREEMPT`, spinlocks disable preemption. `schedule()` switches tasks — if the new task tries to take the same spinlock on the same CPU, it deadlocks (the lock holder is sleeping, and spinlocks don't sleep). Even with PREEMPT_RT where spinlocks become RT mutexes, the raw_spinlock rules still apply for scheduler-internal locks. The general rule: don't voluntarily sleep while holding non-sleepable locks.

**Q3: How does the kernel know it's safe to preempt when returning from an interrupt?**
A: It checks two things: 1) `preempt_count == 0` (not in atomic context, no locks held), and 2) `TIF_NEED_RESCHED` is set. If both are true, the return-from-interrupt path calls `preempt_schedule_irq()` which invokes `__schedule()`. If preempt_count > 0 (in spinlock or IRQ nesting), preemption is deferred until the count drops to zero.

---

*Next: [Chapter 19 — Process Synchronization](Chapter_19_Process_Synchronization.md)*
