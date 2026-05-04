# Chapter 7: Top Half and Bottom Half Mechanisms

## Learning Goals
- Understand why interrupt handling is split into two halves
- Know what work belongs in the top half vs bottom half
- Design interrupt handlers that minimize latency
- Choose the right bottom-half mechanism for your use case

---

## 7.1 Concept of Top Half Interrupt Handler

### The Problem: Competing Goals

```
Goal 1: Handle the interrupt FAST (don't block other IRQs)
Goal 2: Process the interrupt COMPLETELY (data processing, I/O)

These goals conflict! The solution: split the work.

  ┌─────────────────────────────────────────┐
  │         TOP HALF (hardirq)              │
  │  ■ Runs with interrupts disabled*       │
  │  ■ Must execute quickly (< 10 µs ideal) │
  │  ■ Acknowledges the hardware            │
  │  ■ Reads urgent data from device        │
  │  ■ Schedules bottom half                │
  │  * on same IRQ line, others may nest    │
  └──────────────────┬──────────────────────┘
                     │ schedules
  ┌──────────────────▼──────────────────────┐
  │         BOTTOM HALF (deferred)          │
  │  ■ Runs with interrupts ENABLED          │
  │  ■ Can take more time                    │
  │  ■ Processes data, updates structures   │
  │  ■ May run on any CPU                   │
  │  ■ Multiple mechanisms available        │
  └─────────────────────────────────────────┘
```

### Top Half Responsibilities

```
The top half (hardirq handler) MUST:
  1. Acknowledge the interrupt at the hardware
     writel(INT_CLEAR, dev->regs + INT_STATUS);
  
  2. Read time-critical data from device
     data = readl(dev->regs + DATA_REG);
  
  3. Save data for bottom half processing
     dev->rx_buffer[dev->rx_head] = data;
  
  4. Schedule bottom half if more work needed
     tasklet_schedule(&dev->tasklet);
  
  5. Return quickly

The top half SHOULD NOT:
  ✗ Process complex data
  ✗ Allocate large memory
  ✗ Touch heavy data structures
  ✗ Do I/O to other devices
  ✗ Anything that takes > ~10 µs
```

---

## 7.2 Limitations of Top Half Handlers

```
Constraint             │ Reason                       │ Impact
───────────────────────┼──────────────────────────────┼────────────
Cannot sleep           │ No process context — no      │ No mutex,
                       │ task_struct to suspend        │ no GFP_KERNEL
IRQs may be disabled   │ Same-line IRQ disabled during │ Other devices
                       │ handler execution             │ delayed
Limited stack          │ Interrupt stack (8-16 KB)    │ No recursion,
                       │                              │ small locals
Time pressure          │ Long handler = high latency  │ Missed events,
                       │ for all other interrupts     │ poor RT
No user space access   │ Interrupt context, arbitrary │ No copy_to_user
                       │ process interrupted          │
Preempt disabled       │ preempt_count > 0            │ No schedule()
```

### What Happens If You Violate These Rules

```
Violation                │ Consequence
─────────────────────────┼──────────────────────────────
Sleep in hardirq         │ BUG! Kernel oops, "scheduling
                         │ while atomic" crash
Hold mutex in hardirq    │ mutex_lock sleeps → crash
Take spinlock with IRQs  │ Deadlock if same IRQ fires
  enabled on same CPU    │   while lock held
Heavy processing         │ Other IRQs delayed, possible
                         │   interrupt storm, watchdog
printk() excessively     │ Console output takes ms →
                         │   cascading latency
```

---

## 7.3 Bottom Half Processing Concept

### Available Bottom Half Mechanisms

```
Mechanism     │ Context      │ Can Sleep? │ Per-CPU? │ Latency
──────────────┼──────────────┼────────────┼──────────┼──────────
SoftIRQ       │ softirq      │ No         │ Yes      │ Lowest
Tasklet       │ softirq      │ No         │ No*      │ Low
Workqueue     │ process      │ YES        │ Optional │ Higher
Threaded IRQ  │ process      │ YES        │ No       │ Medium
Kernel Timer  │ softirq      │ No         │ N/A      │ Timer-bound

* Tasklet runs on one CPU at a time (serialized per tasklet,
  but NOT per-CPU — it can run on any CPU)
```

### Decision Flowchart

```
Need to defer work from IRQ handler?
          │
          ▼
    Need to sleep?
    ┌─────┴─────┐
   YES         NO
    │           │
    ▼           ▼
 Workqueue   Need lowest latency?
 or         ┌─────┴─────┐
 Threaded  YES         NO
 IRQ        │           │
            ▼           ▼
         SoftIRQ     Tasklet
         (if you     (easier API,
          have a      serialized)
          predefined
          type)

Typical choices:
  Network driver:    SoftIRQ (NET_RX_SOFTIRQ, NET_TX_SOFTIRQ)
  Simple device:     Tasklet (easy, serialize per-device)
  Complex processing: Workqueue (can sleep, use kernel APIs)
  Modern best practice: Threaded IRQs (clean, RT-friendly)
```

---

## 7.4 Deferred Interrupt Handling

### Timeline: Top Half → Bottom Half

```
Time →
────────────────────────────────────────────────────────────

IRQ fires
  │
  ▼
┌──────────────────┐
│   TOP HALF       │ IRQs disabled for this line
│   ~1-5 µs        │
│   ack hw         │
│   save data      │
│   schedule BH    │←── tasklet_schedule(&dev->tl)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   irq_exit()     │ Check softirq pending
│   invoke_softirq │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   BOTTOM HALF    │ IRQs ENABLED, preemption off (softirq)
│   ~10-1000 µs    │ or IRQs enabled, preemptible (workqueue)
│   process data   │
│   update structs │
│   notify user    │
└──────────────────┘
```

### Example: Network Driver Split

```c
/* Top half: minimal — just acknowledge and schedule NAPI */
static irqreturn_t eth_irq_handler(int irq, void *dev_id)
{
    struct net_device *ndev = dev_id;
    struct my_priv *priv = netdev_priv(ndev);
    
    u32 status = readl(priv->regs + INT_STATUS);
    if (!(status & RX_INT))
        return IRQ_NONE;
    
    /* Disable further RX interrupts (NAPI will poll) */
    writel(0, priv->regs + INT_ENABLE);
    
    /* Schedule NAPI poll (bottom half via softirq) */
    napi_schedule(&priv->napi);
    
    return IRQ_HANDLED;
}

/* Bottom half: heavy processing in softirq context */
static int eth_napi_poll(struct napi_struct *napi, int budget)
{
    struct my_priv *priv = container_of(napi, struct my_priv, napi);
    int processed = 0;
    
    while (processed < budget) {
        struct sk_buff *skb;
        /* Read packet from DMA ring buffer */
        skb = eth_rx_packet(priv);
        if (!skb)
            break;
        /* Process and hand to network stack */
        netif_receive_skb(skb);
        processed++;
    }
    
    if (processed < budget) {
        napi_complete(napi);
        /* Re-enable RX interrupt */
        writel(RX_INT, priv->regs + INT_ENABLE);
    }
    return processed;
}
```

### Example: Simple Device with Tasklet

```c
struct my_device {
    void __iomem *regs;
    int irq;
    spinlock_t lock;
    struct tasklet_struct tasklet;
    u32 pending_data[64];
    int data_count;
};

/* Top half */
static irqreturn_t my_top_half(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;
    u32 status = readl(dev->regs + STATUS);
    
    if (!(status & DATA_READY))
        return IRQ_NONE;
    
    /* Read urgent data */
    spin_lock(&dev->lock);
    dev->pending_data[dev->data_count++] = readl(dev->regs + DATA);
    spin_unlock(&dev->lock);
    
    /* Clear interrupt */
    writel(DATA_READY, dev->regs + INT_CLEAR);
    
    /* Schedule bottom half */
    tasklet_schedule(&dev->tasklet);
    return IRQ_HANDLED;
}

/* Bottom half (tasklet) */
static void my_bottom_half(unsigned long data)
{
    struct my_device *dev = (struct my_device *)data;
    int i, count;
    u32 buf[64];
    
    spin_lock_bh(&dev->lock);
    count = dev->data_count;
    memcpy(buf, dev->pending_data, count * sizeof(u32));
    dev->data_count = 0;
    spin_unlock_bh(&dev->lock);
    
    /* Heavy processing — IRQs are enabled here */
    for (i = 0; i < count; i++)
        process_data(buf[i]);
}

/* Setup */
tasklet_init(&dev->tasklet, my_bottom_half, (unsigned long)dev);
```

---

## OS Comparison: Top/Bottom Half Equivalents

```
Linux                    │ Windows              │ macOS/XNU
─────────────────────────┼──────────────────────┼──────────────
Top half (hardirq)       │ ISR (DIRQL)          │ Primary interrupt
                         │                      │   filter
Bottom half (softirq)    │ DPC (DISPATCH_LEVEL) │ Deferred work
Bottom half (workqueue)  │ System worker thread │ Work loop
                         │ (PASSIVE_LEVEL)      │
Threaded IRQ             │ ISR thread           │ Interrupt thread
                         │ (not standard)       │

Windows IRQL levels:
  PASSIVE_LEVEL  < APC_LEVEL < DISPATCH_LEVEL < DIRQL < HIGH
       ↑                          ↑                ↑
   Normal code               DPC/scheduler     ISR/interrupt

QNX:
  ISR (hardware) → pulse → driver thread (process context)
  Very similar to Linux's threaded IRQ model!
  Deterministic from the start.
```

---

## Kernel Source References

```
Top half execution:
  kernel/irq/handle.c         ← generic_handle_irq calls action->handler
  kernel/irq/chip.c           ← Flow handlers invoke action chain

Bottom half infrastructure:
  kernel/softirq.c            ← SoftIRQ + tasklet implementation
  include/linux/interrupt.h   ← Tasklet API, softirq types
  kernel/workqueue.c          ← Workqueue implementation

Threaded IRQ:
  kernel/irq/manage.c         ← irq_thread() function, setup_irq_thread
```

---

## Interview Questions

1. **Why is interrupt handling split into top half and bottom half?**
2. **What work should go in the top half vs bottom half?**
3. **What happens if your top half handler takes too long?**
4. **Name the three bottom half mechanisms and compare them.**
5. **When would you use a tasklet vs a workqueue?**
6. **Can a bottom half be interrupted by a top half? Explain for each BH type.**
7. **What is the NAPI mechanism and how does it relate to top/bottom halves?**
8. **Write a driver that uses top half for ack + data read, and tasklet for processing.**
9. **What is the modern recommended approach: tasklet or threaded IRQ?**
10. **Compare Linux's top/bottom half model with Windows ISR/DPC model.**
11. **What restrictions apply specifically to softirq context?**
12. **Can softirqs be interrupted by other softirqs on the same CPU?**
13. **What is ksoftirqd and when does it run?**
14. **Explain the latency trade-off between softirq and workqueue bottom halves.**
15. **How does PREEMPT_RT change the top/bottom half model?**

---

## Summary

- The top half (hardirq handler) runs with interrupts disabled: must be fast, ack hardware, schedule deferred work
- The bottom half processes the interrupt data at a lower priority with less restrictions
- Three main mechanisms: SoftIRQ (highest performance), Tasklet (easy, serialized), Workqueue (can sleep)
- Modern drivers increasingly use threaded IRQs (request_threaded_irq) which combine clean design with RT compatibility
- The split is the most fundamental design pattern in Linux interrupt handling

---

*Next: [Chapter 8 — Deferred Work Mechanisms](Chapter_08_Deferred_Work.md)*
