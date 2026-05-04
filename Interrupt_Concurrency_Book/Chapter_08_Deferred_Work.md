# Chapter 8: Deferred Work Mechanisms

## Learning Goals
- Master SoftIRQs, Tasklets, and Workqueues — the three pillars of deferred work
- Know the internal implementation of each mechanism
- Choose the right mechanism based on requirements
- Write correct code using each mechanism
- Understand kernel threads for deferred work (kworker, ksoftirqd)

---

## 8.1 SoftIRQs

### Concept

SoftIRQs are the highest-priority deferred work mechanism. They run in softirq context (interrupts enabled, preemption disabled). They are **statically defined** at compile time — drivers cannot create new softirq types.

```
Predefined SoftIRQ Types (kernel/softirq.c):

  Index │ Name              │ Used By
  ──────┼───────────────────┼──────────────────────
  0     │ HI_SOFTIRQ        │ High-priority tasklets
  1     │ TIMER_SOFTIRQ     │ Timer subsystem
  2     │ NET_TX_SOFTIRQ    │ Network transmit
  3     │ NET_RX_SOFTIRQ    │ Network receive
  4     │ BLOCK_SOFTIRQ     │ Block layer
  5     │ IRQ_POLL_SOFTIRQ  │ IRQ polling
  6     │ TASKLET_SOFTIRQ   │ Normal tasklets
  7     │ SCHED_SOFTIRQ     │ Scheduler load balance
  8     │ HRTIMER_SOFTIRQ   │ High-res timers
  9     │ RCU_SOFTIRQ       │ RCU callbacks
```

### SoftIRQ API

```c
/* Registration (done once, usually at boot): */
open_softirq(NET_RX_SOFTIRQ, net_rx_action);

/* Raise (trigger processing): */
raise_softirq(NET_RX_SOFTIRQ);
raise_softirq_irqoff(NET_RX_SOFTIRQ);  /* when IRQs already off */

/* These are NOT for driver use! Only kernel subsystems. */
```

### SoftIRQ Execution Path

```
When do softirqs run?

  1. After every hardirq handler (irq_exit → invoke_softirq)
  2. In ksoftirqd kernel thread (when too many pending)
  3. After local_bh_enable() re-enables bottom halves

  irq_exit()
      │
      ▼
  local_softirq_pending()?  ────NO────→ return
      │
     YES
      │
      ▼
  invoke_softirq()
      │
      ├── __do_softirq() (inline if short)
      │   │
      │   ├── Process all pending softirq types
      │   │   for each set bit in pending:
      │   │       h->action(h)  ← softirq handler
      │   │
      │   ├── Re-check pending (new ones raised during processing)
      │   │   If still pending AND time < MAX_SOFTIRQ_TIME (2ms)
      │   │   AND iterations < MAX_SOFTIRQ_RESTART (10):
      │   │       → loop again
      │   │
      │   └── If STILL pending after max iterations:
      │       → wakeup_softirqd() (defer to ksoftirqd thread)
      │
      └── ksoftirqd/<cpu> runs remaining softirqs
          at normal thread priority (nice 19? → actually SCHED_NORMAL)
```

### SoftIRQ Properties

```
Property              │ Value
──────────────────────┼────────────────────────────────
Context               │ Softirq (interrupts ON, preempt OFF)
Can sleep?            │ NO
Can be preempted?     │ NO (unless PREEMPT_RT)
Runs on which CPU?    │ Same CPU that raised it
Can run concurrently? │ YES! Same softirq on different CPUs
Re-entrancy           │ Must handle concurrent execution
Nesting               │ Hardirqs can interrupt softirqs
Stack                 │ Interrupted task's kernel stack
```

---

## 8.2 Tasklets

### Concept

Tasklets are built on top of SoftIRQs (TASKLET_SOFTIRQ and HI_SOFTIRQ). They provide a simpler API for drivers and guarantee that a **specific tasklet instance runs on only one CPU at a time**.

```c
/* include/linux/interrupt.h */
struct tasklet_struct {
    struct tasklet_struct *next;  /* linked list */
    unsigned long state;          /* TASKLET_STATE_SCHED, _RUN */
    atomic_t count;               /* 0 = enabled, >0 = disabled */
    void (*func)(unsigned long);  /* handler function */
    unsigned long data;           /* argument to handler */
};
```

### Tasklet API

```c
/* Static declaration: */
DECLARE_TASKLET(name, func, data);
DECLARE_TASKLET_DISABLED(name, func, data);

/* Dynamic initialization: */
tasklet_init(&my_tasklet, my_function, (unsigned long)my_data);

/* Schedule for execution: */
tasklet_schedule(&my_tasklet);     /* normal priority */
tasklet_hi_schedule(&my_tasklet);  /* high priority */

/* Disable/Enable: */
tasklet_disable(&my_tasklet);  /* Wait until no longer running, then disable */
tasklet_enable(&my_tasklet);   /* Re-enable */

/* Destroy (must not be scheduled): */
tasklet_kill(&my_tasklet);     /* Wait for completion, then remove */
```

### Complete Tasklet Driver Example

```c
#include <linux/module.h>
#include <linux/interrupt.h>
#include <linux/platform_device.h>

struct my_dev {
    void __iomem *base;
    int irq;
    struct tasklet_struct tasklet;
    u32 rx_data;
};

/* Bottom half: tasklet handler */
static void my_tasklet_handler(unsigned long data)
{
    struct my_dev *dev = (struct my_dev *)data;
    
    /* Heavy processing — IRQs are enabled */
    pr_info("Processing data: 0x%08x\n", dev->rx_data);
    
    /* Can do more work here: update stats, signal user, etc. */
    /* But still cannot sleep! (softirq context) */
}

/* Top half: hardirq handler */
static irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    struct my_dev *dev = dev_id;
    u32 status = readl(dev->base + 0x00);  /* INT_STATUS */
    
    if (!(status & BIT(0)))
        return IRQ_NONE;
    
    /* Read data and acknowledge */
    dev->rx_data = readl(dev->base + 0x04);
    writel(BIT(0), dev->base + 0x08);  /* INT_CLEAR */
    
    /* Schedule tasklet */
    tasklet_schedule(&dev->tasklet);
    return IRQ_HANDLED;
}

static int my_probe(struct platform_device *pdev)
{
    struct my_dev *dev;
    
    dev = devm_kzalloc(&pdev->dev, sizeof(*dev), GFP_KERNEL);
    dev->base = devm_ioremap_resource(&pdev->dev,
                    platform_get_resource(pdev, IORESOURCE_MEM, 0));
    dev->irq = platform_get_irq(pdev, 0);
    
    tasklet_init(&dev->tasklet, my_tasklet_handler, (unsigned long)dev);
    
    return devm_request_irq(&pdev->dev, dev->irq, my_irq_handler,
                            0, "my-dev", dev);
}

static int my_remove(struct platform_device *pdev)
{
    struct my_dev *dev = platform_get_drvdata(pdev);
    tasklet_kill(&dev->tasklet);
    return 0;
}
```

### Tasklet State Machine

```
           tasklet_schedule()
IDLE ─────────────────────────→ SCHEDULED
  ▲                                │
  │                     __do_softirq runs it
  │                                │
  │                                ▼
  │              ┌─── RUNNING ────┐
  │              │   func() executes
  │              │   on ONE CPU only
  │              └────────────────┘
  │                      │
  │         func returns  │
  └───────────────────────┘

Key guarantee:
  TASKLET_STATE_RUN bit ensures only ONE CPU runs
  this tasklet at any time. Other CPUs that see RUN
  set will skip and let the running CPU handle it.
```

### Note: Tasklets Are Deprecated for New Code

```
As of Linux ~5.10+, tasklets are SOFT-DEPRECATED.
New drivers should prefer:
  - Threaded IRQs (request_threaded_irq)
  - Workqueues (for sleeping/complex work)

Reasons for deprecation:
  1. Cannot sleep (limits what bottom half can do)
  2. Runs in softirq context (delays other softirqs)
  3. Threaded IRQs provide better RT determinism
  4. Workqueues offer more flexibility

Existing drivers still use tasklets extensively.
```

---

## 8.3 Workqueues

### Concept

Workqueues run deferred work in **process context** — they use kernel threads (kworker). This means workqueue handlers can **sleep**, use mutexes, do I/O, and call any kernel API.

```
Workqueue Architecture:

  Driver code                 Kernel
  ──────────                  ──────
  INIT_WORK(&work, func)
  queue_work(wq, &work)  ──→  kworker thread wakes up
                               │
                               ▼
                           func(&work) runs in process context
                           (can sleep, take mutexes, etc.)
```

### Key Data Structures

```c
/* include/linux/workqueue.h */
struct work_struct {
    atomic_long_t data;        /* work state + pool info */
    struct list_head entry;    /* linked list in pool */
    work_func_t func;          /* handler function */
};

struct delayed_work {
    struct work_struct work;
    struct timer_list timer;   /* delay before queuing */
};
```

### Workqueue API

```c
/* Static declaration: */
DECLARE_WORK(name, func);
DECLARE_DELAYED_WORK(name, func);

/* Dynamic initialization: */
INIT_WORK(&my_work, my_work_handler);
INIT_DELAYED_WORK(&my_dwork, my_delayed_handler);

/* Queue for execution: */
schedule_work(&my_work);              /* on system_wq */
queue_work(my_wq, &my_work);         /* on specific workqueue */
schedule_delayed_work(&my_dwork, HZ); /* after 1 second */
queue_delayed_work(my_wq, &my_dwork, msecs_to_jiffies(100));

/* Cancel: */
cancel_work_sync(&my_work);          /* wait for completion */
cancel_delayed_work_sync(&my_dwork);

/* Flush (wait for all pending work): */
flush_work(&my_work);
flush_workqueue(my_wq);

/* Create custom workqueues: */
struct workqueue_struct *my_wq;
my_wq = alloc_workqueue("my-wq", WQ_UNBOUND | WQ_HIGHPRI, 0);
/* Or: create_singlethread_workqueue("my-wq"); */
destroy_workqueue(my_wq);
```

### Workqueue Types

```
Flag              │ Behavior
──────────────────┼──────────────────────────────────
(none)            │ Per-CPU, normal priority
WQ_UNBOUND        │ Not bound to specific CPU
WQ_HIGHPRI        │ High priority kworker threads
WQ_FREEZABLE      │ Can be frozen during suspend
WQ_MEM_RECLAIM    │ Guaranteed execution even under
                  │   memory pressure (rescue worker)
WQ_CPU_INTENSIVE  │ Won't delay other work items
                  │   on same CPU

System workqueues (pre-created):
  system_wq             ← default (schedule_work uses this)
  system_highpri_wq     ← high priority
  system_long_wq        ← for long-running work
  system_unbound_wq     ← not CPU-bound
  system_freezable_wq   ← freezable
```

### Complete Workqueue Driver Example

```c
#include <linux/workqueue.h>
#include <linux/interrupt.h>

struct my_dev {
    void __iomem *base;
    int irq;
    struct work_struct rx_work;
    struct mutex data_lock;  /* Can use mutex — process context! */
    u32 rx_data;
};

/* Bottom half: workqueue handler (process context) */
static void my_work_handler(struct work_struct *work)
{
    struct my_dev *dev = container_of(work, struct my_dev, rx_work);
    
    /* CAN SLEEP HERE! Full kernel API available */
    mutex_lock(&dev->data_lock);
    
    /* Process data */
    pr_info("Processing: 0x%08x\n", dev->rx_data);
    
    /* Could do: file I/O, memory allocation, hardware access */
    void *buf = kmalloc(4096, GFP_KERNEL);  /* Can use GFP_KERNEL! */
    if (buf) {
        /* ... heavy processing ... */
        kfree(buf);
    }
    
    mutex_unlock(&dev->data_lock);
}

/* Top half */
static irqreturn_t my_handler(int irq, void *dev_id)
{
    struct my_dev *dev = dev_id;
    u32 status = readl(dev->base + STATUS);
    
    if (!(status & DATA_READY))
        return IRQ_NONE;
    
    dev->rx_data = readl(dev->base + DATA);
    writel(DATA_READY, dev->base + INT_CLEAR);
    
    schedule_work(&dev->rx_work);  /* Queue on system_wq */
    return IRQ_HANDLED;
}

/* Init */
INIT_WORK(&dev->rx_work, my_work_handler);
mutex_init(&dev->data_lock);

/* Cleanup */
cancel_work_sync(&dev->rx_work);
```

---

## 8.4 Kernel Threads for Deferred Work

### ksoftirqd

```
Per-CPU kernel thread that runs softirqs when:
  - __do_softirq() loops too many times (>10)
  - __do_softirq() runs too long (>2ms)
  - Explicit wakeup needed

  ps aux | grep ksoftirqd
  root   3  ... [ksoftirqd/0]
  root   9  ... [ksoftirqd/1]
  root  15  ... [ksoftirqd/2]
  root  21  ... [ksoftirqd/3]

ksoftirqd runs at SCHED_NORMAL (can be preempted).
This prevents softirq processing from monopolizing the CPU.
```

### kworker

```
Kernel worker threads that execute workqueue items:

  ps aux | grep kworker
  root   5  ... [kworker/0:0]     ← per-CPU worker, CPU 0
  root   6  ... [kworker/0:0H]    ← high-priority, CPU 0
  root  11  ... [kworker/1:0]     ← per-CPU worker, CPU 1
  root  25  ... [kworker/u8:0]    ← unbound worker

Naming: kworker/<cpu>:<id>   or  kworker/u<cpus>:<id>
  <cpu>   = CPU bound to (or 'u' for unbound)
  <cpus>  = number of CPUs (for unbound)
  <id>    = worker ID
  H       = high priority
```

### IRQ Threads (from request_threaded_irq)

```
Each threaded IRQ creates a dedicated kernel thread:

  ps aux | grep irq/
  root  65  ... [irq/28-my-device]
  root  66  ... [irq/33-eth0]

These run at SCHED_FIFO priority 50 by default.
Higher priority than normal threads, lower than top-priority RT.
```

---

## 8.5 Differences Between Deferred Mechanisms

### Comprehensive Comparison

```
Feature          │ SoftIRQ      │ Tasklet     │ Workqueue   │ Threaded IRQ
─────────────────┼──────────────┼─────────────┼─────────────┼─────────────
Context          │ Softirq      │ Softirq     │ Process     │ Process
Can sleep?       │ NO           │ NO          │ YES         │ YES
Can use mutex?   │ NO           │ NO          │ YES         │ YES
GFP_KERNEL?      │ NO           │ NO          │ YES         │ YES
Run on any CPU?  │ Per-CPU      │ Any*        │ Any/Pinned  │ Any
Concurrent runs? │ YES (diff    │ NO (same    │ YES         │ NO (per IRQ)
                 │  CPUs)       │  instance)  │             │
Priority         │ Above process│ Above proc  │ Normal      │ SCHED_FIFO
Latency          │ Lowest       │ Low         │ Higher      │ Medium
Creation         │ Compile-time │ Runtime     │ Runtime     │ Runtime
Driver use?      │ No (subsys)  │ Yes (legacy)│ Yes         │ Yes (modern)
RT-friendly?     │ No           │ No          │ Yes         │ YES (best)
Deprecated?      │ No           │ Soft-yes    │ No          │ No

* Tasklet: runs on any CPU, but serialized per instance
```

### When to Use What (Modern Guidelines)

```
1. NETWORK DRIVER → NAPI (uses NET_RX_SOFTIRQ internally)
   Highest throughput, batch processing

2. SIMPLE DEVICE DRIVER → Threaded IRQ
   request_threaded_irq(irq, top_half, bottom_half, ...)
   Clean, RT-compatible, modern best practice

3. NEED TO SLEEP IN BH → Workqueue
   Mutex, file I/O, significant allocation

4. LEGACY DEVICE → Tasklet
   Still works, simple, but don't use for new code

5. HIGH-THROUGHPUT SUBSYSTEM → SoftIRQ
   Only if you're writing core kernel subsystem code
   (networking, block layer, timer)
```

---

## Execution Priority Stack

```
  Highest priority
    │
    ▼
┌──────────────────────┐
│  NMI                 │ Cannot be masked
├──────────────────────┤
│  Hard IRQ (top half) │ Interrupts device-level
├──────────────────────┤
│  SoftIRQ / Tasklet   │ Deferred from hardirq
├──────────────────────┤
│  IRQ Thread           │ SCHED_FIFO 50
├──────────────────────┤
│  kworker (highpri)   │ SCHED_NORMAL (high nice)
├──────────────────────┤
│  kworker (normal)    │ SCHED_NORMAL
├──────────────────────┤
│  User processes      │ SCHED_NORMAL
├──────────────────────┤
│  Idle                │ Lowest
└──────────────────────┘
    │
    ▼
  Lowest priority
```

---

## Kernel Source References

```
SoftIRQ:
  kernel/softirq.c              ← __do_softirq, raise_softirq
  include/linux/interrupt.h     ← softirq types, open_softirq

Tasklet:
  kernel/softirq.c              ← tasklet_action, tasklet_hi_action
  include/linux/interrupt.h     ← tasklet_struct, tasklet_schedule

Workqueue:
  kernel/workqueue.c            ← Core workqueue implementation
  include/linux/workqueue.h     ← work_struct, INIT_WORK, queue_work

Threaded IRQ:
  kernel/irq/manage.c           ← irq_thread, setup_irq_thread
  include/linux/interrupt.h     ← request_threaded_irq
```

---

## Interview Questions

1. **Compare SoftIRQ, Tasklet, and Workqueue. When would you use each?**
2. **Why can't you sleep in a tasklet but you can in a workqueue?**
3. **What is ksoftirqd and when does it run?**
4. **How many SoftIRQ types does Linux have? Can a driver add new ones?**
5. **What is the WQ_MEM_RECLAIM flag for workqueues?**
6. **Write code that uses INIT_WORK/schedule_work to defer work from an IRQ handler.**
7. **Why are tasklets being deprecated? What should new drivers use instead?**
8. **What is the maximum time __do_softirq runs before deferring to ksoftirqd?**
9. **Can the same softirq handler run simultaneously on multiple CPUs?**
10. **What happens if you call tasklet_schedule() while the tasklet is already running?**
11. **What is the difference between schedule_work() and queue_work()?**
12. **How does cancel_work_sync() ensure the work item has completed?**
13. **What scheduling policy do IRQ threads use by default?**
14. **Explain WQ_UNBOUND. When would you use it over a per-CPU workqueue?**
15. **What are the system workqueues (system_wq, system_highpri_wq, etc.)?**

---

## Summary

- **SoftIRQs**: Fastest deferred work, per-CPU, compile-time defined, for kernel subsystems only
- **Tasklets**: Built on softirqs, per-instance serialization, simple driver API, now deprecated for new code
- **Workqueues**: Process context (can sleep!), uses kworker threads, most flexible, recommended for complex drivers
- **Threaded IRQs**: Modern best practice — runs IRQ handler in a dedicated kernel thread, RT-compatible
- ksoftirqd handles softirq overflow; kworker threads execute workqueue items
- Modern driver recommendation: use threaded IRQs or workqueues, avoid creating new tasklet-based code

---

*Next: [Chapter 9 — Interrupt Threading](Chapter_09_Interrupt_Threading.md)*
