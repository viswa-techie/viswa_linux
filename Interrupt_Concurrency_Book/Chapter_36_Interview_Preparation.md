# Chapter 36: Interview Preparation

## Learning Goals
- Master the most-asked interview questions on interrupts and concurrency
- Practice structured answers with depth
- Know what interviewers expect at different experience levels
- Cover both conceptual understanding and practical implementation

---

## 36.1 Interview Question Categories

```
Category                │ Weight     │ Focus
────────────────────────┼────────────┼────────────────────────
Interrupt Basics        │ 20%        │ What, why, how
Interrupt Handling      │ 25%        │ Top/bottom half, threading
Concurrency Primitives  │ 25%        │ Locks, RCU, atomics
Driver Design           │ 15%        │ Real-world locking patterns
Debugging/Performance   │ 10%        │ Tools, optimization
Architecture (ARM/x86)  │ 5%         │ GIC, APIC, assembly entry
```

---

## 36.2 Top 50 Interview Questions with Model Answers

### INTERRUPTS — Fundamentals

**Q1: What is an interrupt? Why do we need them?**
```
An interrupt is an asynchronous hardware signal that causes the CPU to stop
executing the current code and jump to a handler routine.

Without interrupts: CPU must poll every device → wastes cycles, high latency.
With interrupts: CPU works on useful tasks, gets notified only when device
needs attention → efficient, low latency.

Types: hardware IRQs, software interrupts (syscalls), exceptions.
```

**Q2: Explain the difference between an interrupt and an exception.**
```
Interrupt: Asynchronous, caused by external hardware (device ready, timer).
  - Can happen between ANY two instructions
  - Handled via IRQ number → vector → handler

Exception: Synchronous, caused by CPU itself (page fault, divide-by-zero).
  - Triggered BY the executing instruction
  - Handled via exception type → vector → handler

Both use the same vector/IDT mechanism on x86.
```

**Q3: What are NMIs? Give examples of their use.**
```
NMI = Non-Maskable Interrupt. Cannot be disabled by local_irq_disable().

Uses:
  1. Hardware watchdog (detect hard lockups)
  2. Performance profiling (NMI-based sampling)
  3. Machine check exceptions (hardware failure)
  4. System crash dump (sysrq, kdump)
  5. Kernel debugging (break into debugger)

Restrictions: Cannot use spinlocks (may deadlock with interrupted code),
only trylock, atomics, per-CPU data.
```

**Q4: What is the difference between edge-triggered and level-triggered interrupts?**
```
Edge-triggered: Fires on signal transition (rising/falling edge).
  ─────┐       ┌─────
       │       │        ← Interrupt fires at this edge
       └───────┘
  - One shot: if missed, not re-delivered
  - Faster handling (no need to clear source)
  - Risk: lost interrupts if not handled quickly

Level-triggered: Fires as long as signal is asserted.
  ─────┐       ┌─────
       │       │        ← Interrupt fires continuously while LOW
       └───────┘
  - Self-correcting: re-fires until source cleared
  - Must clear interrupt source to de-assert
  - Safer: cannot miss an interrupt
  - Used in most embedded designs
```

### INTERRUPTS — Handling

**Q5: Explain top half and bottom half. Why split?**
```
Top half: Runs in hardirq context (IRQs disabled on local CPU).
  - Must be fast (<100µs)
  - Cannot sleep, cannot allocate with GFP_KERNEL
  - Job: ACK interrupt, save minimal data, schedule bottom half

Bottom half: Runs later in a less restricted context.
  - Mechanisms: softirq, tasklet, workqueue, threaded IRQ
  - Can do heavier processing

Why split:
  - IRQs are disabled during top half → long handlers delay OTHER interrupts
  - Splitting minimizes time with IRQs off → better latency for all devices
```

**Q6: Compare softirq, tasklet, workqueue, and threaded IRQ.**
```
Feature         │ SoftIRQ      │ Tasklet     │ Workqueue    │ Threaded IRQ
────────────────┼──────────────┼─────────────┼──────────────┼──────────
Context         │ Softirq      │ Softirq     │ Process      │ Process
Can sleep?      │ No           │ No          │ Yes          │ Yes
Concurrency     │ Same type on │ Serialized  │ Concurrent   │ Per-IRQ
                │ multiple CPUs│ per-tasklet │              │ thread
Typical use     │ Networking,  │ Simple      │ Complex work,│ I2C/SPI
                │ block I/O    │ driver BH   │ firmware load│ sensors
New code?       │ Rarely       │ Deprecated  │ Yes          │ Yes (preferred)
```

**Q7: What is request_threaded_irq()? When do you use it?**
```c
int request_threaded_irq(unsigned int irq,
                          irq_handler_t handler,      /* Hardirq */
                          irq_handler_t thread_fn,    /* Thread */
                          unsigned long flags,
                          const char *name,
                          void *dev);

Use when:
  - Handler needs to sleep (I2C/SPI communication)
  - Handler needs mutex, GFP_KERNEL allocation
  - Want PREEMPT_RT compatibility
  - Slow bus devices (regmap over I2C/SPI)

The hardirq handler returns IRQ_WAKE_THREAD to wake thread_fn.
IRQF_ONESHOT keeps IRQ masked until thread completes.
```

**Q8: What happens if you return IRQ_NONE vs IRQ_HANDLED?**
```
IRQ_HANDLED: Tells kernel this device generated the interrupt.
  - Kernel proceeds normally.

IRQ_NONE: Tells kernel this handler did NOT handle the interrupt.
  - For shared IRQs: kernel tries next handler in chain.
  - If ALL handlers return IRQ_NONE → "nobody cared" warning.
  - After 99,900 unhandled out of 100,000 → kernel disables the IRQ.

Rule: ALWAYS check your device's status register.
      Return IRQ_NONE if the interrupt isn't from your device.
```

### CONCURRENCY — Locking

**Q9: When do you use spinlock vs mutex?**
```
Spinlock:
  - Cannot sleep while held
  - Use in atomic context (IRQ handler, softirq)
  - Short critical sections (microseconds)
  - spin_lock(), spin_lock_irqsave()

Mutex:
  - CAN sleep while held (and the holder can sleep)
  - Use in process context ONLY
  - Longer critical sections (can do I/O, allocations)
  - Has lockdep support, priority inheritance (rt_mutex)

Decision: Can the code path sleep? → mutex. Otherwise → spinlock.
          Shared with IRQ? → spinlock_irqsave.
```

**Q10: Why can't you sleep while holding a spinlock?**
```
1. Spinlock disables preemption (or IRQs with irqsave variant).
2. If you sleep → schedule() → context switch to another task.
3. The new task may try to acquire the same spinlock.
4. But the sleeping task holds it and can't be scheduled back
   (it's on a different CPU's run queue or waiting).
5. Result: DEADLOCK or system hang.

Also: On PREEMPT_RT, spin_lock is converted to rt_mutex (sleeping),
so this rule is enforced differently, but the principle remains.
```

**Q11: Explain spin_lock_irqsave(). When is it needed?**
```c
unsigned long flags;
spin_lock_irqsave(&lock, flags);    /* Saves IRQ state + disables IRQs + acquires lock */
/* critical section */
spin_unlock_irqrestore(&lock, flags); /* Restores IRQ state + releases lock */

Needed when: Lock is shared between process context and hard IRQ handler.

Why: If process holds spin_lock() (IRQs on) and IRQ fires on same CPU,
IRQ handler tries spin_lock() → DEADLOCK (spinning on lock held by
preempted process).

irqsave (not irq): Saves and restores the previous IRQ state.
If IRQs were already disabled, irqrestore won't incorrectly enable them.
```

**Q12: What is a deadlock? How do you prevent it?**
```
Deadlock: Two or more tasks each hold a lock the other needs.

ABBA pattern:
  Task A: lock(A) → lock(B)     (blocks — B held by Task B)
  Task B: lock(B) → lock(A)     (blocks — A held by Task A)
  → Both stuck forever.

Prevention rules:
  1. Lock ordering: Always acquire locks in the same global order
  2. Avoid nested locks when possible
  3. Use lockdep (CONFIG_PROVE_LOCKING) to detect at runtime
  4. Never hold spinlock and then acquire mutex
  5. Use trylock where appropriate (with fallback)
```

**Q13: What is RCU? Explain in simple terms.**
```
RCU = Read-Copy-Update.

Simple explanation:
  - Multiple readers can access data simultaneously WITHOUT any lock
  - Writer creates a NEW copy of data, atomically swaps pointer
  - Old data freed AFTER all readers are done (grace period)

API:
  Reader: rcu_read_lock(); p = rcu_dereference(ptr); use(p); rcu_read_unlock();
  Writer: new = copy(old); modify(new); rcu_assign_pointer(ptr, new);
          synchronize_rcu(); kfree(old);

Cost: ~0 for readers (just preempt_disable). Heavy for writers (grace period wait).
Best for: Read-mostly data (routing tables, config, module lists).
```

**Q14: What is priority inversion? How does Linux solve it?**
```
Priority inversion: High-priority task H blocked by low-priority task L
holding a lock, while medium-priority task M runs (preempting L).

H waits for L → L preempted by M → H effectively blocked by M.
Unbounded if many medium-priority tasks exist.

Linux solution: Priority Inheritance (PI) via rt_mutex.
  When H blocks on lock held by L:
    L's priority temporarily boosted to H's priority
    L cannot be preempted by M
    L finishes quickly, releases lock
    L's priority restored

rt_mutex_lock() implements PI. On PREEMPT_RT, all spin_locks
become rt_mutexes → automatic PI.
```

**Q15: Explain memory barriers. Why are they needed?**
```
CPUs reorder memory operations for performance.
What you write in C isn't the order the CPU executes.

Without barriers:
  CPU 0: x=1; flag=1;  → CPU may do: flag=1; x=1; (reordered!)
  CPU 1: if(flag) use(x); → May see flag=1 but x=0!

Barriers:
  smp_wmb() — write barrier: stores before it complete before stores after
  smp_rmb() — read barrier: loads before it complete before loads after
  smp_mb()  — full barrier: all before complete before all after
  smp_store_release(&flag,1) — all stores before visible before this store
  smp_load_acquire(&flag)    — this load completes before all subsequent ops

Must be PAIRED: wmb on writer ↔ rmb on reader.
```

### DRIVER DESIGN

**Q16: Design locking for a char driver with read(), ioctl(), and IRQ handler.**
```c
struct my_driver {
    spinlock_t      hw_lock;    /* Shared with IRQ handler */
    struct mutex    io_lock;    /* Process context only (read/ioctl) */
    wait_queue_head_t wait;     /* Sleep until IRQ data ready */
    bool            data_ready; /* Protected by hw_lock */
    u8              buffer[N];  /* Protected by io_lock */
};

IRQ handler (hardirq):
    spin_lock(&hw_lock);       /* Plain — IRQs already off */
    read HW data → buffer
    data_ready = true;
    spin_unlock(&hw_lock);
    wake_up(&wait);

read() (process ctx):
    wait_event_interruptible(wait, data_ready);
    mutex_lock(&io_lock);
    spin_lock_irqsave(&hw_lock, flags);  /* Quick access to shared flag */
    data_ready = false;
    spin_unlock_irqrestore(&hw_lock, flags);
    copy_to_user(buf, buffer, count);    /* Outside spinlock (can sleep) */
    mutex_unlock(&io_lock);

ioctl() (process ctx):
    mutex_lock(&io_lock);
    /* Configuration changes */
    mutex_unlock(&io_lock);
```

**Q17: How do you handle device removal while it's in use?**
```
Problem: remove() called while threads are in read()/write().

Pattern:
  1. Set "shutting down" flag
  2. Remove user interface (cdev_del, netdev_unregister)
  3. Wait for active users (synchronize, cancel work)
  4. Free resources

    mutex_lock(&dev->lock);
    dev->shutting_down = true;
    mutex_unlock(&dev->lock);

    cdev_del(&dev->cdev);           /* No new opens */
    cancel_work_sync(&dev->work);    /* Wait for pending work */
    free_irq(dev->irq, dev);        /* Wait for running handler */
    /* Now safe to free dev */
```

### DEBUGGING

**Q18: How do you debug a deadlock?**
```
1. Enable lockdep: CONFIG_PROVE_LOCKING=y
   - Detects potential deadlocks at first occurrence (before actual deadlock)
   - Shows lock dependency chain in dmesg

2. If system is hung:
   - SysRq+t (show all tasks): echo t > /proc/sysrq-trigger
   - Look for tasks in "D" state (uninterruptible sleep)
   - Check their wait channel (wchan) — shows which lock they're waiting on

3. Read lockdep output:
   - "possible circular locking dependency detected"
   - Shows the chain: Lock A → Lock B → Lock A (cycle)

4. Use ftrace to trace lock acquisition order:
   - Enable lock tracepoints

5. Check your code for:
   - ABBA lock ordering violations
   - spin_lock (not irqsave) shared with IRQ handler
   - Nested locks acquired in different orders
```

**Q19: What is /proc/interrupts? How do you use it?**
```bash
cat /proc/interrupts
# Shows: IRQ#, per-CPU counts, controller, trigger, device name

Use cases:
  1. Verify IRQ is registered and firing
  2. Check balance across CPUs (affinity issues)
  3. Detect IRQ storms (rapidly increasing counts)
  4. Identify shared IRQs (multiple names on one line)

Watch rate:
  watch -n1 -d cat /proc/interrupts
```

**Q20: How do you measure interrupt latency?**
```
1. ftrace irqsoff tracer:
   echo irqsoff > /sys/kernel/debug/tracing/current_tracer
   → Shows longest time IRQs were disabled (worst-case IRQ latency)

2. cyclictest (for RT):
   cyclictest -p 80 -t 4 -n -m
   → Measures scheduling + interrupt response latency

3. GPIO + oscilloscope (embedded):
   Toggle GPIO at ISR entry/exit, measure with scope
   → Most accurate hardware timing

4. bpftrace:
   Histogram IRQ handler duration using tracepoints

5. perf:
   perf stat -e irq:irq_handler_entry -a sleep 5
   → Count interrupt rate
```

---

## 36.3 Additional Practice Questions

### Quick-Fire (1-2 sentence answers expected)

```
21. What does local_irq_save() do?
22. Can you call kmalloc(GFP_KERNEL) in a hard IRQ handler?
23. What is a softirq? How many types exist in Linux?
24. What is IRQF_SHARED?
25. What is the difference between atomic_t and refcount_t?
26. What does rcu_dereference() do?
27. Why is rwlock_t deprecated?
28. What is per-CPU data? Why does it avoid locking?
29. What is PREEMPT_RT?
30. What does synchronize_rcu() wait for?
```

### Design Questions (5-10 minute answers expected)

```
31. Design the interrupt handling for a DMA-based storage driver.
32. How would you optimize a network driver doing 10M packets/sec?
33. Design locking for a shared ring buffer between IRQ and user space.
34. How would you port a Linux driver to PREEMPT_RT?
35. Design interrupt architecture for an automotive ADAS system.
```

### Scenario Questions

```
36. Your system shows 100% CPU in softirq. What do you check?
37. A driver works on x86 but crashes on ARM64. What could cause this?
38. cyclictest shows 500µs max latency on RT. How do you improve it?
39. lockdep warns about "inconsistent lock state." What does this mean?
40. /proc/interrupts shows IRQ stuck at 0. How do you debug?
```

### Advanced

```
41. Explain the qspinlock fast path, pending path, and slow path.
42. How does the mutex optimistic spinning work?
43. What is the RCU tree hierarchy and why is it needed?
44. How does TLB shootdown work? What IPIs are involved?
45. Explain the memory model differences between x86 TSO and ARM64 weak ordering.
46. What is KCSAN? What type of bugs does it detect?
47. How does the kernel detect soft lockups vs hard lockups?
48. What is the difference between smp_store_release and smp_wmb?
49. How does kfree_rcu work internally?
50. What changes in interrupt handling between Linux 5.x and 6.x?
```

---

## 36.4 Answer Templates

### Structure for Technical Answers

```
1. Definition (1-2 sentences)
2. Why it matters (1 sentence)
3. How it works (2-4 sentences with key details)
4. Example (code snippet or scenario)
5. Gotchas/edge cases (1-2 points)
```

### Example Template Applied

```
Q: "What is a completion?"

1. Definition: A completion is a synchronization primitive for waiting
   until an event occurs, implemented via wait_for_completion/complete().

2. Why: Used when one thread must wait for another thread or IRQ to
   finish a specific operation (e.g., DMA transfer done).

3. How: Thread calls wait_for_completion() → sleeps on wait queue.
   When event happens (often in IRQ), complete() is called → wakes waiter.
   Internally: counter-based, can be used for one-shot or repeating events.

4. Example:
   struct completion dma_done;
   init_completion(&dma_done);
   
   /* Start DMA, IRQ handler will call: complete(&dma_done); */
   ret = wait_for_completion_interruptible_timeout(&dma_done, HZ*5);

5. Gotchas:
   - Use reinit_completion() before reuse (not init_completion)
   - Timeout version prevents hanging if event never occurs
```

---

## 36.5 Self-Assessment Checklist

```
□ Can explain interrupt flow from hardware to handler (both x86 and ARM64)
□ Can write a threaded IRQ handler with correct locking
□ Know when to use spin_lock vs spin_lock_irqsave vs spin_lock_bh
□ Can explain RCU and write reader/writer code
□ Understand memory barriers and can write correct lock-free code
□ Know how to debug deadlocks with lockdep
□ Can design locking for a multi-context driver
□ Understand PREEMPT_RT impact on interrupts and locking
□ Can use ftrace, perf, and /proc/interrupts for debugging
□ Know the difference between at least 3 OS interrupt models
```

---

## Summary

- Interviews test both theory (what is X?) and practice (design locking for Y)
- Structure answers: definition → why → how → example → gotchas
- Most common topics: top/bottom half, spinlock vs mutex, RCU, deadlocks
- Practice drawing diagrams: interrupt flow, lock decision tree, context hierarchy
- Know your tools: /proc/interrupts, lockdep, ftrace, cyclictest
- For senior roles: expect architecture-level design questions
- For embedded roles: expect DT, power management, latency budget questions

---

*This concludes the 36-chapter Interrupts & Concurrency series.*
*Return to: [Master Index](00_Master_Index.md)*
