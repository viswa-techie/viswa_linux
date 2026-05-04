# Linux Kernel — Interrupt Subsystem & DMA Framework

## Interrupt Handling — Two Halves Model

```
Hardware IRQ fires
        │
        ▼
┌──────────────────────┐
│   Top Half (ISR)     │  ← runs immediately, with IRQs disabled (on that CPU)
│   - READ the hardware│     MUST be FAST (microseconds)
│   - Acknowledge IRQ  │     CAN NOT sleep, CAN NOT call blocking functions
│   - Schedule bottom  │
│     half work        │
└──────────┬───────────┘
           │  schedule workqueue / tasklet / threaded IRQ
           ▼
┌──────────────────────┐
│  Bottom Half         │  ← runs in process context (threaded IRQ)
│  - Heavy processing  │     CAN sleep, CAN acquire mutex
│  - Copy DMA data     │     runs in kthread context
│  - Wake up waiters   │
└──────────────────────┘
```

---

## Request an IRQ

```c
#include <linux/interrupt.h>

/* Top half handler */
static irqreturn_t my_isr(int irq, void *dev_id) {
    struct my_dev *dev = dev_id;

    /* Read status register to clear interrupt & determine cause */
    u32 status = readl(dev->base + IRQ_STATUS);
    if (!(status & IRQ_MY_BIT))
        return IRQ_NONE;  // not our interrupt (shared IRQs)

    writel(IRQ_MY_BIT, dev->base + IRQ_STATUS);  // clear

    /* Schedule work for bottom half */
    tasklet_schedule(&dev->tasklet);
    // OR: queue_work(dev->wq, &dev->work);

    return IRQ_HANDLED;
}

/* In probe() */
ret = devm_request_irq(dev, irq, my_isr,
                       IRQF_SHARED,   // shared IRQ line
                       "my-device", my_dev_priv);
if (ret) {
    dev_err(dev, "failed to request IRQ %d: %d\n", irq, ret);
    return ret;
}

/* Threaded IRQ — top half + sleeping bottom half in one call */
ret = devm_request_threaded_irq(dev, irq,
    my_top_half,   // runs with IRQ disabled, fast
    my_bottom_half, // runs in kthread, can sleep
    IRQF_SHARED | IRQF_ONESHOT, "my-irq", priv);
```

---

## Tasklets vs Workqueues vs Threaded IRQ

| Mechanism | Context | Can sleep? | When to use |
|---|---|---|---|
| Tasklet | Softirq (atomic) | **NO** | Fast deferred work, simple |
| Workqueue | Process (kthread) | **YES** | I/O, delay, mutex needed |
| Threaded IRQ | kthread per IRQ | **YES** | Modern preferred approach |

```c
/* Tasklet */
static void my_tasklet_fn(unsigned long data) {
    /* process queued data, no sleeping */
}
DECLARE_TASKLET(my_tasklet, my_tasklet_fn, 0);

/* In ISR: */
tasklet_schedule(&my_tasklet);

/* Workqueue (kernel global wq, simple case) */
static void my_work_fn(struct work_struct *work) {
    struct my_dev *d = container_of(work, struct my_dev, work);
    /* can sleep here */
}
INIT_WORK(&dev->work, my_work_fn);

/* In ISR or probe: */
schedule_work(&dev->work);
/* Or for delay: */
schedule_delayed_work(&dev->dwork, msecs_to_jiffies(50));
```

---

## Spinlock vs Mutex — Context Rules

```c
#include <linux/spinlock.h>
#include <linux/mutex.h>

/* SPINLOCK — interrupt-safe, non-sleeping, short critical sections */
DEFINE_SPINLOCK(my_lock);

// In normal process context:
spin_lock(&my_lock);
/* critical section */
spin_unlock(&my_lock);

// If ISR may also take this lock:
unsigned long flags;
spin_lock_irqsave(&my_lock, flags);
/* critical section */
spin_unlock_irqrestore(&my_lock, flags);

/* MUTEX — process context only, CAN sleep */
DEFINE_MUTEX(my_mutex);

mutex_lock(&my_mutex);   // blocking
/* critical section */
mutex_unlock(&my_mutex);

/* Rule of thumb:
   - Interrupt context?  → ONLY spinlock (no sleep allowed)
   - Process context, need to sleep? → mutex
   - Process context, short section? → spinlock or mutex
*/
```

---

## DMA Framework (DMA Engine)

```c
/* DMA transfer mental model:
   CPU sets up DMA descriptor → DMA controller moves data → IRQ on completion
   CPU is FREE during transfer — that's the point.
*/

#include <linux/dmaengine.h>
#include <linux/dma-mapping.h>

/* 1. Get a DMA channel (from device tree: dmas = <&dma 5>) */
struct dma_chan *chan = dma_request_chan(dev, "rx");

/* 2. Allocate coherent DMA buffer (CPU & device share same physical memory) */
void    *cpu_addr;
dma_addr_t dma_addr;
cpu_addr = dma_alloc_coherent(dev, BUFFER_SIZE, &dma_addr, GFP_KERNEL);
//                                              ↑ physical address for DMA controller

/* 3. Prepare a DMA descriptor */
struct dma_async_tx_descriptor *desc;
desc = dmaengine_prep_slave_single(
    chan,
    dma_addr,         // physical address in RAM
    BUFFER_SIZE,
    DMA_DEV_TO_MEM,   // direction: device→memory (RX)
    DMA_PREP_INTERRUPT | DMA_CTRL_ACK
);

/* 4. Set completion callback */
desc->callback       = my_dma_complete;
desc->callback_param = dev_priv;

/* 5. Submit and fire */
dmaengine_submit(desc);
dma_async_issue_pending(chan);

/* Completion callback (called in softirq context) */
static void my_dma_complete(void *param) {
    struct my_dev *d = param;
    complete(&d->dma_done);  // wake up waiter
}

/* 6. Wait for completion */
wait_for_completion(&dev->dma_done);

/* 7. Cleanup */
dma_free_coherent(dev, BUFFER_SIZE, cpu_addr, dma_addr);
dma_release_channel(chan);
```

---

## DMA Mapping Types

```c
/* Coherent (consistent) mapping — no explicit sync needed, expensive */
dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);

/* Streaming (single-use) — faster, you must sync manually */
dma_addr_t dma_handle;
// Map before DMA:
dma_handle = dma_map_single(dev, cpu_buf, size, DMA_TO_DEVICE);
// Check for mapping error:
if (dma_mapping_error(dev, dma_handle)) { /* handle error */ }
// After DMA completes (before CPU reads):
dma_sync_single_for_cpu(dev, dma_handle, size, DMA_FROM_DEVICE);
// Unmap:
dma_unmap_single(dev, dma_handle, size, DMA_FROM_DEVICE);
```

---

## ION / dma-buf in Android (camera/display)

```
┌──────────────────────────────┐
│   Camera HAL allocates ION   │→ dma_buf fd → shares with Display/GPU
│   buffer via /dev/ion or     │
│   DMA Heap (/dev/dma_heap/)  │
└──────────────────────────────┘
  CPU maps with mmap()
  DMA uses physical address
  GPU uses imported CL/GL image
```

---

## Interview Questions

- Q: What is the difference between top half and bottom half of an interrupt handler?
- Q: When would you use a workqueue vs a tasklet?
- Q: Why can't you sleep inside an interrupt handler?
- Q: What is `irqsave`/`irqrestore` and when do you need it?
- Q: What is coherent DMA vs streaming DMA? When do you use each?
- Q: What is a dma-buf and why is it used in Android camera/display?
