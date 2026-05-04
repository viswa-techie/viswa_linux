# Chapter 33: Real-Time Linux (PREEMPT_RT)

## Learning Goals
- Understand the PREEMPT_RT patch set and its mainlining status
- Know how PREEMPT_RT transforms interrupt handling and locking
- Configure and tune an RT Linux system for determinism
- Measure and validate real-time performance
- Apply RT-aware driver design principles

---

## 33.1 What Is PREEMPT_RT?

PREEMPT_RT transforms Linux from a general-purpose OS into a real-time OS by making (almost) everything preemptible.

```
Standard Linux:                    PREEMPT_RT Linux:
  Hard IRQ handlers: not preempt    Hard IRQ: threaded (preemptible)
  Softirqs: not preempt             Softirqs: threaded (preemptible)
  Spinlocks: disable preempt        Spinlocks: sleeping (rt_mutex)
  local_irq_disable: real HW off    local_irq_disable: no real HW off*

  * Unless using raw_spin_lock / raw_local_irq_disable
```

### Mainlining Status

```
Timeline:
  2004: PREEMPT_RT started by Ingo Molnar, Thomas Gleixner
  2004-2023: Out-of-tree patch set, maintained separately
  2023-2024: Major components merged into mainline
  Linux 6.12: PREEMPT_RT fully merged into mainline
  
Config: CONFIG_PREEMPT_RT=y (selectable at build time)
```

---

## 33.2 Key PREEMPT_RT Transformations

### 1. Forced Interrupt Threading

```
All IRQ handlers become kernel threads (SCHED_FIFO 50):

Before (standard):
  IRQ → hardirq handler → softirq

After (PREEMPT_RT):
  IRQ → minimal hardirq (just wake thread) → IRQ thread (schedulable)
  
  ps aux | grep irq
  root  [irq/25-nvme0q1]   SCHED_FIFO priority 50
  root  [irq/26-eth0]      SCHED_FIFO priority 50

Exceptions (NOT threaded):
  - IRQF_NO_THREAD handlers
  - Timer interrupt (needs to run in hardirq)
  - IPI handlers
  - NMI
```

### 2. Spinlock → Sleeping Lock

```c
/* On PREEMPT_RT: */
spin_lock(&lock)          → rt_mutex_lock(&lock.rtmutex)
                            (sleeping, preemptible, priority inheritance)

spin_lock_irqsave(&lock)  → rt_mutex_lock(&lock.rtmutex)
                            (NO real IRQ disable!)

/* Only raw_ variants remain true spinlocks: */
raw_spin_lock(&lock)       → actual spin (non-preemptible)
raw_spin_lock_irqsave()    → actual IRQ disable + spin
```

### 3. Softirq Processing in Thread Context

```
Standard:
  __do_softirq() runs in softirq context (cannot be preempted)

PREEMPT_RT:
  Softirqs run in ksoftirqd or IRQ thread context
  Fully preemptible by higher-priority threads
  
  This means: High-priority RT task is NOT delayed by networking
  softirqs (NET_RX, NET_TX) as it would be on standard Linux.
```

---

## 33.3 The Preemption Model Hierarchy

```
Config Option              │ Preemption Level    │ Use Case
───────────────────────────┼─────────────────────┼──────────────
CONFIG_PREEMPT_NONE        │ No kernel preempt   │ Servers, throughput
                           │ (only at syscall    │
                           │  return)            │
CONFIG_PREEMPT_VOLUNTARY   │ Explicit preempt    │ Desktop
                           │ points (might_sleep)│
CONFIG_PREEMPT             │ Full preempt        │ Low-latency desktop
                           │ (except spin_lock   │ interactive
                           │  and IRQ context)   │
CONFIG_PREEMPT_RT          │ Fully preemptible   │ Real-time, automotive
                           │ (spinlocks sleep,   │ industrial control
                           │  IRQs threaded)     │
```

---

## 33.4 RT-Aware Driver Design

### Using raw_spinlock_t

```c
/* For truly non-preemptible sections (rare): */
raw_spinlock_t hw_lock;
raw_spin_lock_init(&hw_lock);

raw_spin_lock_irqsave(&hw_lock, flags);
/* Real IRQ disable, real spinning — use ONLY when necessary:
   - Interrupt controller code
   - Timer/scheduler internals
   - Very short hardware register sequences
*/
raw_spin_unlock_irqrestore(&hw_lock, flags);
```

### Driver Pattern for RT Compatibility

```c
struct my_rt_driver {
    /* For hardware register access (short, non-sleeping): */
    raw_spinlock_t      hw_lock;

    /* For driver state (can be preempted on RT): */
    spinlock_t          state_lock;   /* → rt_mutex on PREEMPT_RT */

    /* For long operations: */
    struct mutex        io_lock;
};

static int __init my_probe(struct platform_device *pdev)
{
    raw_spin_lock_init(&dev->hw_lock);
    spin_lock_init(&dev->state_lock);
    mutex_init(&dev->io_lock);

    /* Use threaded IRQ (RT-friendly): */
    devm_request_threaded_irq(&pdev->dev, irq,
                               my_hardirq, my_thread_fn,
                               IRQF_ONESHOT, "mydev", dev);
    return 0;
}

/* Hard IRQ: minimal (runs in real hardirq even on RT): */
static irqreturn_t my_hardirq(int irq, void *data)
{
    struct my_rt_driver *dev = data;
    u32 status;

    raw_spin_lock(&dev->hw_lock);
    status = readl(dev->regs + STATUS);
    writel(status, dev->regs + ACK);
    raw_spin_unlock(&dev->hw_lock);

    return (status & MY_IRQ) ? IRQ_WAKE_THREAD : IRQ_NONE;
}

/* Thread handler: process context, preemptible: */
static irqreturn_t my_thread_fn(int irq, void *data)
{
    struct my_rt_driver *dev = data;

    spin_lock(&dev->state_lock);    /* rt_mutex on RT — can be preempted */
    process_data(dev);
    spin_unlock(&dev->state_lock);

    return IRQ_HANDLED;
}
```

---

## 33.5 Configuring an RT System

### Kernel Configuration

```bash
# Enable RT:
CONFIG_PREEMPT_RT=y
CONFIG_HIGH_RES_TIMERS=y
CONFIG_NO_HZ_FULL=y

# Debug options for RT development:
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_PREEMPT=y
CONFIG_LATENCYTOP=y
CONFIG_FTRACE=y
CONFIG_IRQSOFF_TRACER=y
```

### Boot Parameters

```bash
# Isolate CPUs for RT tasks (CPUs 2-3 dedicated):
isolcpus=2,3
nohz_full=2,3
rcu_nocbs=2,3

# Disable kernel features that cause latency spikes:
nosoftlockup
nowatchdog
tsc=reliable
idle=poll               # Avoid deep C-states (latency on wake)
processor.max_cstate=0  # No CPU power saving
intel_idle.max_cstate=0
```

### Runtime Tuning

```bash
# Set RT task affinity to isolated CPUs:
taskset -c 2,3 ./my_rt_app

# Set RT scheduling:
chrt -f 80 ./my_rt_app   # SCHED_FIFO priority 80

# Set IRQ thread priority:
chrt -f -p 90 $(pgrep irq/25)  # Higher than app for fast response

# Disable IRQ balancing on RT CPUs:
echo 2,3 > /proc/irq/25/smp_affinity_list    # Bind to RT CPU
service irqbalance stop                        # Stop irqbalance

# Lock memory (prevent page faults):
mlockall(MCL_CURRENT | MCL_FUTURE)  # In application code

# Set memory limits:
echo 0 > /proc/sys/vm/overcommit_memory
```

---

## 33.6 Priority Configuration

```
Typical automotive RT priority scheme:

Priority │ SCHED  │ Component              │ CPU
─────────┼────────┼────────────────────────┼─────
99       │ FIFO   │ Watchdog               │ 2
98       │ FIFO   │ Safety-critical ISR     │ 2
95       │ FIFO   │ Motor control loop      │ 2
90       │ FIFO   │ Sensor IRQ threads      │ 2-3
85       │ FIFO   │ CAN bus processing      │ 3
80       │ FIFO   │ Application RT thread   │ 3
50       │ FIFO   │ Default IRQ threads     │ 0-1
20       │ RR     │ Logging/diagnostics     │ 0-1
0        │ OTHER  │ Non-RT processes        │ 0-1

Rule: Higher priority = lower number in standard Linux,
      but with chrt/SCHED_FIFO: higher number = higher priority.
```

---

## 33.7 Measuring RT Performance

### cyclictest

```bash
# Basic RT latency test:
cyclictest -p 80 -t 4 -n -m -l 100000

# With CPU isolation:
cyclictest -p 80 -t 1 -n -m -a 2 -l 1000000

# With stress:
stress-ng --cpu 4 --io 2 --vm 2 &
cyclictest -p 99 -t 1 -n -m -a 2 -l 1000000 -q

# Expected results:
# Standard Linux:  Max ~100-500µs
# PREEMPT_RT:      Max ~10-50µs
# Tuned RT:        Max ~5-15µs
```

### hwlatdetect

```bash
# Detect hardware-induced latency (SMI, power management):
hwlatdetect --duration=60 --threshold=10
# Reports latency spikes caused by hardware (not kernel)
```

### ftrace Latency

```bash
echo preemptirqsoff > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# Run workload
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
# Shows worst-case preemption/IRQ-disabled duration
```

---

## 33.8 Common RT Pitfalls

```
Pitfall                               │ Fix
──────────────────────────────────────┼───────────────────────────────
Using spin_lock for HW register       │ raw_spin_lock (minimal use)
  access on RT (becomes sleeping)     │
Page fault in RT path                 │ mlockall(), pre-fault pages
  (triggers memory allocation)        │
GFP_KERNEL in time-critical code      │ Pre-allocate, use GFP_ATOMIC
System management interrupt (SMI)     │ Disable SMI or detect with
  causes µs-ms latency               │ hwlatdetect
Console output (printk) in RT path    │ Use trace_printk(), async log
CPU frequency scaling causes jitter   │ Set performance governor
Timer coalescing groups timers        │ Use hrtimer for precise timing
Network softirq stalls RT tasks       │ Move network to non-RT CPU
RCU callbacks on RT CPU               │ rcu_nocbs= boot parameter
```

---

## Kernel Source References

```
PREEMPT_RT core:
  kernel/locking/rtmutex.c              ← RT mutex implementation
  include/linux/spinlock_rt.h           ← spin_lock → rt_mutex mapping
  kernel/sched/rt.c                     ← RT scheduler class
  kernel/irq/manage.c                   ← IRQ threading (setup_irq_thread)
  include/linux/preempt.h              ← Preemption model definitions

Configuration:
  kernel/Kconfig.preempt               ← Preemption model choices
  Documentation/admin-guide/kernel-parameters.txt ← Boot params
```

---

## Interview Questions

1. **What does PREEMPT_RT do? How does it achieve real-time behavior?**
2. **How does spin_lock() behave differently on PREEMPT_RT?**
3. **What is the difference between raw_spinlock_t and spinlock_t on RT?**
4. **How are interrupts handled on PREEMPT_RT? What changes?**
5. **What is priority inversion? How does RT Linux solve it?**
6. **How do you isolate CPUs for RT tasks?**
7. **What is cyclictest? What latency numbers should you expect?**
8. **Name 5 tuning steps for a deterministic RT Linux system.**
9. **Why might printk() cause latency problems in RT systems?**
10. **How do you design a driver that works correctly on both standard and RT Linux?**

---

## Summary

- PREEMPT_RT makes Linux fully preemptible: threaded IRQs, sleeping spinlocks
- spin_lock() → rt_mutex on RT; only raw_spin_lock() is a real spinlock
- All IRQ handlers become threads (SCHED_FIFO 50) — fully schedulable
- Isolate CPUs with isolcpus/nohz_full/rcu_nocbs for determinism
- Measure with cyclictest: expect <50µs max latency when properly tuned
- Drivers should use threaded IRQ + raw_spin_lock only for minimal HW access
- PREEMPT_RT merged into mainline Linux 6.12 — no longer an external patch
- Key trade-off: throughput decreases slightly, determinism improves greatly

---

*Next: [Chapter 34 — Embedded System Interrupt Design](Chapter_34_Embedded_Interrupt_Design.md)*
