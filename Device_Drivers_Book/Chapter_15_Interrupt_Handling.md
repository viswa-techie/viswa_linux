# Chapter 15: Interrupt Handling

## Chapter Overview

Interrupts are the primary mechanism for hardware to notify the CPU of events. This chapter covers the complete interrupt architecture, from hardware signals through top-half/bottom-half split to threaded IRQs.

---

## 15.1 Interrupt Architecture Overview

```
Hardware Device                    CPU
┌──────────┐                   ┌──────┐
│  Status   │──IRQ line──────→│ IRQ  │
│  changes  │                  │ pin  │
│  (data    │                  │      │
│   ready,  │     via          │      │
│   error)  │     GIC/APIC    │      │
└──────────┘                   └──┬───┘
                                  │
                          CPU suspends current task
                          Jumps to interrupt vector
                                  │
                                  ▼
                          Kernel IRQ handler runs
                          Driver's handler called
                          Acknowledges IRQ
                          Returns to interrupted code
```

---

## 15.2 Interrupt Controller

```
ARM GIC (Generic Interrupt Controller):

  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Device A │  │ Device B │  │ Device C │
  │ (SPI 42) │  │ (SPI 78) │  │ (PPI 14) │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │              │              │
  ┌────▼──────────────▼──────────────▼────┐
  │           GIC Distributor              │
  │  (routes IRQs to CPU interfaces)      │
  └───────────────┬───────────────────────┘
                  │
  ┌───────────────▼───────────────────────┐
  │        GIC CPU Interface              │
  │  (signals interrupt to core)          │
  └───────────────┬───────────────────────┘
                  │
                  ▼
              CPU Core
              (takes IRQ exception)
```

### Interrupt Types

| Type | Abbrev | Range (GIC) | Description |
|------|--------|-------------|-------------|
| Software Generated | SGI | 0-15 | IPI between cores |
| Private Peripheral | PPI | 16-31 | Per-core (timer, PMU) |
| Shared Peripheral | SPI | 32-1020 | Shared device IRQs |
| Message Signaled | MSI/MSI-X | — | PCIe write-to-address |

---

## 15.3 Interrupt Registration in Drivers

### Basic Registration

```c
#include <linux/interrupt.h>

static irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    struct my_priv *priv = dev_id;
    u32 status = readl(priv->base + REG_IRQ_STATUS);

    if (!(status & MY_IRQ_MASK))
        return IRQ_NONE;    /* Not our interrupt (shared line) */

    /* Handle the interrupt */
    writel(status, priv->base + REG_IRQ_CLEAR);  /* Acknowledge */
    return IRQ_HANDLED;
}

static int my_probe(struct platform_device *pdev)
{
    int irq = platform_get_irq(pdev, 0);
    if (irq < 0)
        return irq;

    int ret = devm_request_irq(&pdev->dev, irq, my_irq_handler,
                                0,              /* flags */
                                dev_name(&pdev->dev),  /* name */
                                priv);          /* dev_id (cookie) */
    if (ret)
        return ret;
}
```

### IRQ Flags

| Flag | Meaning |
|------|---------|
| `0` | Default (from DT: level/edge, trigger) |
| `IRQF_SHARED` | IRQ line shared with other devices |
| `IRQF_ONESHOT` | Keep IRQ masked until threaded handler completes |
| `IRQF_NO_SUSPEND` | Don't disable during system suspend |
| `IRQF_TRIGGER_RISING` | Rising edge triggered |
| `IRQF_TRIGGER_FALLING` | Falling edge triggered |
| `IRQF_TRIGGER_HIGH` | Level high triggered |
| `IRQF_TRIGGER_LOW` | Level low triggered |

---

## 15.4 Interrupt Handler Functions

### Return Values

```c
irqreturn_t handler(int irq, void *dev_id)
{
    /* IRQ_NONE    — Not our interrupt (shared line) */
    /* IRQ_HANDLED — We handled it */
    /* IRQ_WAKE_THREAD — Wake the threaded handler */
}
```

### Complete Handler Example

```c
static irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    struct my_priv *priv = dev_id;
    u32 status;

    status = readl(priv->base + REG_IRQ_STATUS);
    if (!status)
        return IRQ_NONE;

    /* Process each IRQ source */
    if (status & IRQ_RX_DONE) {
        /* Short work: read data from FIFO */
        priv->rx_data = readl(priv->base + REG_DATA);
        priv->data_ready = true;
        wake_up_interruptible(&priv->wait_queue);
    }

    if (status & IRQ_ERROR) {
        priv->errors++;
        /* Schedule heavy error recovery for bottom half */
        schedule_work(&priv->error_work);
    }

    /* Clear all handled interrupts */
    writel(status, priv->base + REG_IRQ_CLEAR);
    return IRQ_HANDLED;
}
```

---

## 15.5 Top Half and Bottom Half Mechanisms

```
Interrupt fires!
       │
       ▼
┌──────────────────────────────────────┐
│ TOP HALF (hardirq context)           │
│ • Runs with IRQ disabled on this line│
│ • Must be FAST (< ~100 µs)          │
│ • Read status, acknowledge HW       │
│ • Copy urgent data                   │
│ • Schedule bottom half              │
│ • Cannot sleep!                      │
└──────────┬───────────────────────────┘
           │
           ▼
┌──────────────────────────────────────┐
│ BOTTOM HALF (deferred processing)    │
│ • Runs after interrupts re-enabled   │
│ • Does the heavy work               │
│ • May sleep (workqueue/thread)       │
│ • Process data, update state         │
│ • Wake up userspace waiters         │
└──────────────────────────────────────┘
```

### Bottom Half Mechanisms Comparison

| Mechanism | Context | Can Sleep | Priority | Use Case |
|-----------|---------|-----------|----------|----------|
| **Softirq** | Softirq | No | Highest | Network RX/TX, block |
| **Tasklet** | Softirq | No | High | Simple deferred work |
| **Workqueue** | Process | Yes | Normal | Complex processing |
| **Threaded IRQ** | kthread | Yes | RT-capable | Modern preferred |

---

## 15.6 SoftIRQ

Reserved for core subsystems (network, block, timer). Drivers rarely use directly.

```c
/* Defined in include/linux/interrupt.h */
enum {
    HI_SOFTIRQ = 0,        /* High-priority tasklets */
    TIMER_SOFTIRQ,          /* Timer callbacks */
    NET_TX_SOFTIRQ,         /* Network transmit */
    NET_RX_SOFTIRQ,         /* Network receive */
    BLOCK_SOFTIRQ,          /* Block layer */
    IRQ_POLL_SOFTIRQ,       /* IRQ polling */
    TASKLET_SOFTIRQ,        /* Normal tasklets */
    SCHED_SOFTIRQ,          /* Scheduler */
    HRTIMER_SOFTIRQ,        /* High-res timer */
    RCU_SOFTIRQ,            /* RCU */
};
```

---

## 15.7 Tasklets

Simple bottom-half mechanism (being phased out in favor of threaded IRQs):

```c
/* Declaration */
static void my_tasklet_func(struct tasklet_struct *t)
{
    struct my_priv *priv = from_tasklet(priv, t, tasklet);
    /* Process data — runs in softirq context, cannot sleep */
}

/* In probe: */
tasklet_setup(&priv->tasklet, my_tasklet_func);

/* In top-half IRQ handler: */
tasklet_schedule(&priv->tasklet);

/* In remove: */
tasklet_kill(&priv->tasklet);
```

---

## 15.8 Threaded Interrupts (Modern Preferred Approach)

```c
static irqreturn_t my_irq_top(int irq, void *dev_id)
{
    struct my_priv *priv = dev_id;
    u32 status = readl(priv->base + REG_IRQ_STATUS);

    if (!status)
        return IRQ_NONE;

    priv->irq_status = status;
    writel(status, priv->base + REG_IRQ_CLEAR);
    return IRQ_WAKE_THREAD;   /* Wake the threaded handler */
}

static irqreturn_t my_irq_thread(int irq, void *dev_id)
{
    struct my_priv *priv = dev_id;

    /* Runs in kernel thread context — CAN SLEEP */
    if (priv->irq_status & IRQ_DATA_READY) {
        /* Can do I2C reads, mutex_lock, kmalloc(GFP_KERNEL) */
        i2c_smbus_read_byte_data(priv->client, REG_DATA);
    }

    return IRQ_HANDLED;
}

/* Registration with both top and threaded handler */
devm_request_threaded_irq(dev, irq,
    my_irq_top,      /* Top half (hardirq) */
    my_irq_thread,   /* Threaded handler (process context) */
    IRQF_ONESHOT,    /* Keep IRQ masked until thread completes */
    dev_name(dev),
    priv);

/* Thread-only handler (top half = NULL → generic ack) */
devm_request_threaded_irq(dev, irq,
    NULL,             /* No top half — kernel provides default */
    my_irq_thread,    /* Threaded handler only */
    IRQF_ONESHOT | IRQF_TRIGGER_FALLING,
    dev_name(dev),
    priv);
```

---

## IRQ Handling Flow Diagram

```
Hardware IRQ fires
       │
       ▼
CPU takes exception → arch vector table
       │
       ▼
generic_handle_irq()                    ← kernel/irq/handle.c
       │
       ▼
irq_desc[irq]->handle_irq()            ← flow handler
       │                                   (handle_level_irq, handle_edge_irq)
       ▼
action->handler(irq, dev_id)            ← YOUR top-half handler
       │
       ├── Returns IRQ_HANDLED → done
       │
       └── Returns IRQ_WAKE_THREAD
              │
              ▼
       Wake irq/XX-name kthread
              │
              ▼
       action->thread_fn(irq, dev_id)   ← YOUR threaded handler
              │
              ▼
       Unmask IRQ (IRQF_ONESHOT)
```

---

## Workqueue Alternative

```c
/* For bottom-half work that needs process context */
static void my_work_handler(struct work_struct *work)
{
    struct my_priv *priv = container_of(work, struct my_priv, work);
    /* Can sleep, allocate memory, take mutexes */
}

/* In probe: */
INIT_WORK(&priv->work, my_work_handler);

/* In top-half: */
schedule_work(&priv->work);

/* In remove: */
cancel_work_sync(&priv->work);
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| `kernel/irq/manage.c` | `request_irq()`, `request_threaded_irq()` |
| `kernel/irq/handle.c` | Generic IRQ handling |
| `kernel/irq/chip.c` | IRQ chip (controller) framework |
| `include/linux/interrupt.h` | IRQ API, flags, bottom half |
| `drivers/irqchip/` | GIC, APIC, etc. drivers |
| `kernel/softirq.c` | Softirq processing |
| `kernel/workqueue.c` | Workqueue implementation |

---

## Debugging Interrupts

```bash
# View interrupt counts per CPU
cat /proc/interrupts

# View IRQ affinity
cat /proc/irq/42/smp_affinity

# Set IRQ affinity to CPU 0
echo 1 > /proc/irq/42/smp_affinity

# See spurious interrupts
cat /proc/irq/42/spurious

# ftrace IRQ events
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
cat /sys/kernel/debug/tracing/trace
```

---

## OS Comparison

| Aspect | Linux | Windows (WDF) | QNX | FreeRTOS |
|--------|-------|--------------|-----|----------|
| Top half | hardirq handler | ISR (DIRQL) | Pulse handler | ISR |
| Bottom half | threaded/workqueue/softirq | DPC (DISPATCH_LEVEL) | Pulse thread | Deferred interrupt handler |
| Registration | `request_irq()` | `WdfInterruptCreate()` | `InterruptAttach()` | `xTaskCreate()` or ISR |
| Can sleep in BH | Threaded/WQ: Yes | DPC: No | Thread: Yes | Task: Yes |
| Default model | Top+bottom split | ISR+DPC | ISR+thread | ISR+task notify |

---

## Interview Questions

**Q1: What is the difference between top half and bottom half?**
A: Top half runs in hardirq context immediately when IRQ fires — must be fast, can't sleep, just acknowledges HW and schedules deferred work. Bottom half runs later in softirq/tasklet (can't sleep) or workqueue/threaded IRQ (can sleep) — does heavy processing.

**Q2: Why use `IRQF_ONESHOT` with threaded IRQs?**
A: IRQF_ONESHOT keeps the IRQ line masked until the threaded handler completes. Without it, a level-triggered IRQ would fire again immediately (since the threaded handler hasn't cleared the condition yet), causing an interrupt storm.

**Q3: What happens if an IRQ handler takes too long?**
A: Other IRQs on the same CPU are blocked (hardirq context). Excessive latency causes missed interrupts, timer jitter, and potential watchdog resets. Keep hardirq handlers under ~100µs.

**Q4: When should you use `request_threaded_irq()` vs workqueue?**
A: Threaded IRQ: when the entire IRQ processing needs process context (I2C/SPI sensor reads in the handler). Workqueue: when only specific heavy operations need deferral and the top half does meaningful quick work.

**Q5: What is an interrupt storm and how do you handle it?**
A: Rapid, continuous interrupts overwhelming the CPU. Common causes: misconfigured level-triggered IRQ (not clearing condition), broken hardware. Linux detects this: after 99,900 unhandled interrupts, it disables the IRQ line and logs "nobody cared". Fix: ensure handler clears the IRQ source, use IRQF_ONESHOT, or fix hardware.

---

## Summary

| Concept | Key Point |
|---------|-----------|
| Top half | Fast, hardirq context, read status + ack |
| Threaded IRQ | Process context, can sleep, modern preferred |
| Tasklet | Softirq context, can't sleep, being phased out |
| Workqueue | Process context, can sleep, flexible |
| IRQF_ONESHOT | Keep IRQ masked during threaded handler |
| `devm_request_irq()` | Auto-freed on driver removal |
| `platform_get_irq()` | Get IRQ number from DT |

---

*Next: [Chapter 16 — Concurrency and Synchronization](Chapter_16_Concurrency_Synchronization.md)*
