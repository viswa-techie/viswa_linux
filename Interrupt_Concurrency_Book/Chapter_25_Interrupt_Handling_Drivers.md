# Chapter 25: Interrupt Handling in Drivers

## Learning Goals
- Write complete interrupt handlers for real hardware
- Choose between top-half only, threaded, and NAPI approaches
- Handle shared interrupts correctly
- Implement DMA-based interrupt-driven I/O
- Use managed (devm) IRQ APIs

---

## 25.1 IRQ Registration Patterns

### Pattern 1: Simple Top-Half Only

```c
/* For fast, minimal-work interrupts (e.g., simple status read) */
static irqreturn_t simple_isr(int irq, void *data)
{
    struct my_dev *dev = data;
    u32 status = readl(dev->regs + IRQ_STATUS);

    if (!(status & MY_DEV_IRQ))
        return IRQ_NONE;        /* Not our interrupt */

    writel(MY_DEV_IRQ, dev->regs + IRQ_ACK);
    atomic_inc(&dev->event_count);
    wake_up(&dev->wait_queue);

    return IRQ_HANDLED;
}

/* Registration: */
ret = devm_request_irq(&pdev->dev, irq, simple_isr,
                        IRQF_SHARED, "mydev", dev);
```

### Pattern 2: Threaded IRQ (Most Common for I2C/SPI)

```c
/* Hard IRQ: minimal — just check and ack */
static irqreturn_t my_hardirq(int irq, void *data)
{
    struct my_dev *dev = data;
    if (!readl(dev->regs + IRQ_STATUS))
        return IRQ_NONE;
    return IRQ_WAKE_THREAD;     /* Wake the thread handler */
}

/* Thread handler: runs in process context, can sleep */
static irqreturn_t my_thread_fn(int irq, void *data)
{
    struct my_dev *dev = data;

    /* Can sleep: I2C/SPI transfers, mutex, kmalloc(GFP_KERNEL) */
    mutex_lock(&dev->lock);
    regmap_read(dev->regmap, STATUS_REG, &status);
    process_event(dev, status);
    regmap_write(dev->regmap, ACK_REG, status);
    mutex_unlock(&dev->lock);

    return IRQ_HANDLED;
}

/* Registration: */
ret = devm_request_threaded_irq(&pdev->dev, irq,
                                 my_hardirq, my_thread_fn,
                                 IRQF_ONESHOT | IRQF_SHARED,
                                 "mydev", dev);
```

### Pattern 3: Thread-Only (No Hard IRQ)

```c
/* NULL primary handler → kernel provides default_primary_handler
   which just returns IRQ_WAKE_THREAD */
ret = devm_request_threaded_irq(&pdev->dev, irq,
                                 NULL, my_thread_fn,
                                 IRQF_ONESHOT,
                                 "mydev", dev);
```

### Pattern 4: NAPI (Network Drivers)

```c
/* IRQ handler: disable IRQ + schedule NAPI */
static irqreturn_t my_net_isr(int irq, void *data)
{
    struct my_netdev *priv = data;

    if (!readl(priv->regs + IRQ_STATUS))
        return IRQ_NONE;

    /* Disable HW interrupt: */
    writel(0, priv->regs + IRQ_ENABLE);
    napi_schedule(&priv->napi);

    return IRQ_HANDLED;
}

/* NAPI poll: process packets in softirq context */
static int my_napi_poll(struct napi_struct *napi, int budget)
{
    struct my_netdev *priv = container_of(napi, struct my_netdev, napi);
    int done = 0;

    while (done < budget) {
        struct sk_buff *skb = fetch_rx_packet(priv);
        if (!skb)
            break;
        napi_gro_receive(napi, skb);
        done++;
    }

    if (done < budget) {
        napi_complete_done(napi, done);
        writel(IRQ_RX, priv->regs + IRQ_ENABLE);  /* Re-enable */
    }
    return done;
}
```

---

## 25.2 IRQF Flags Reference

```
Flag                │ Meaning
────────────────────┼──────────────────────────────────────
IRQF_SHARED         │ IRQ line shared with other devices
IRQF_ONESHOT        │ Keep IRQ disabled until thread completes
IRQF_TRIGGER_RISING │ Rising edge trigger
IRQF_TRIGGER_FALLING│ Falling edge trigger
IRQF_TRIGGER_HIGH   │ Level high trigger
IRQF_TRIGGER_LOW    │ Level low trigger
IRQF_NO_THREAD      │ Never thread this IRQ (even on PREEMPT_RT)
IRQF_NO_SUSPEND     │ Do not disable during suspend
IRQF_TIMER          │ Timer interrupt
IRQF_NOBALANCING    │ Exclude from IRQ balancing
```

### IRQF_ONESHOT Details

```
Without IRQF_ONESHOT:
  IRQ fires → hardirq ack → IRQ re-enabled → thread runs
  Problem: If hardware re-asserts before thread handles it → IRQ storm!

With IRQF_ONESHOT:
  IRQ fires → hardirq ack → IRQ stays MASKED → thread runs → complete →
  IRQ unmasked
  Safe: Hardware cannot re-interrupt until thread is done.

Required when:
  - Thread handler accesses slow bus (I2C/SPI)
  - Thread handler does the actual ACK
  - NULL primary handler (thread-only)
```

---

## 25.3 Shared Interrupts

Multiple devices on the same IRQ line:

```c
/* Each driver must:
   1. Use IRQF_SHARED
   2. Provide unique dev_id (non-NULL)
   3. Check if the interrupt is from its device
   4. Return IRQ_NONE if not its interrupt
*/

static irqreturn_t device_a_isr(int irq, void *data)
{
    struct dev_a *dev = data;
    if (!(readl(dev->regs + STATUS) & IRQ_PENDING))
        return IRQ_NONE;        /* Not mine! */

    writel(IRQ_ACK, dev->regs + STATUS);
    /* ... handle ... */
    return IRQ_HANDLED;
}

/* Both devices share IRQ 42: */
request_irq(42, device_a_isr, IRQF_SHARED, "dev_a", dev_a);
request_irq(42, device_b_isr, IRQF_SHARED, "dev_b", dev_b);

/* Kernel calls BOTH handlers in chain:
   device_a_isr() → IRQ_NONE (not mine)
   device_b_isr() → IRQ_HANDLED (mine) */
```

---

## 25.4 DMA + Interrupt Pattern

```c
struct my_dma_dev {
    struct mutex        io_lock;
    spinlock_t          hw_lock;
    struct completion    dma_done;
    dma_addr_t          dma_handle;
    void               *dma_buf;
    size_t              dma_len;
    int                 dma_error;
};

/* === Start DMA transfer === */
static ssize_t my_read(struct file *file, char __user *buf,
                        size_t count, loff_t *ppos)
{
    struct my_dma_dev *dev = file->private_data;
    unsigned long flags;
    ssize_t ret;

    mutex_lock(&dev->io_lock);

    count = min(count, (size_t)BUF_SIZE);

    /* Allocate DMA buffer: */
    dev->dma_buf = dma_alloc_coherent(dev->dev, count,
                                       &dev->dma_handle, GFP_KERNEL);
    dev->dma_len = count;

    reinit_completion(&dev->dma_done);

    /* Start DMA: */
    spin_lock_irqsave(&dev->hw_lock, flags);
    writel(lower_32_bits(dev->dma_handle), dev->regs + DMA_ADDR_LO);
    writel(upper_32_bits(dev->dma_handle), dev->regs + DMA_ADDR_HI);
    writel(count, dev->regs + DMA_LEN);
    writel(DMA_START | DMA_IRQ_EN, dev->regs + DMA_CTRL);
    spin_unlock_irqrestore(&dev->hw_lock, flags);

    /* Wait for DMA completion: */
    ret = wait_for_completion_interruptible_timeout(&dev->dma_done,
                                                     msecs_to_jiffies(5000));
    if (ret == 0)
        ret = -ETIMEDOUT;
    else if (ret > 0 && !dev->dma_error)
        ret = copy_to_user(buf, dev->dma_buf, count) ? -EFAULT : count;

    dma_free_coherent(dev->dev, count, dev->dma_buf, dev->dma_handle);

    mutex_unlock(&dev->io_lock);
    return ret;
}

/* === DMA completion IRQ === */
static irqreturn_t dma_isr(int irq, void *data)
{
    struct my_dma_dev *dev = data;
    u32 status;

    spin_lock(&dev->hw_lock);
    status = readl(dev->regs + DMA_STATUS);
    if (!(status & DMA_DONE_BIT)) {
        spin_unlock(&dev->hw_lock);
        return IRQ_NONE;
    }

    writel(DMA_DONE_BIT, dev->regs + DMA_STATUS);  /* ACK */
    dev->dma_error = (status & DMA_ERROR_BIT) ? -EIO : 0;
    spin_unlock(&dev->hw_lock);

    complete(&dev->dma_done);
    return IRQ_HANDLED;
}
```

---

## 25.5 Managed IRQ (devm)

```c
/* devm_request_irq: automatically freed on driver detach */
ret = devm_request_irq(&pdev->dev, irq, my_isr,
                        IRQF_SHARED, dev_name(&pdev->dev), dev);
if (ret) {
    dev_err(&pdev->dev, "Failed to request IRQ %d: %d\n", irq, ret);
    return ret;
}

/* No need to call free_irq() in remove — devm handles it */

/* devm_request_threaded_irq: same for threaded */
ret = devm_request_threaded_irq(&pdev->dev, irq,
                                 my_hardirq, my_thread_fn,
                                 IRQF_ONESHOT, "mydev", dev);
```

### Getting IRQ Numbers

```c
/* Platform device: */
int irq = platform_get_irq(pdev, 0);       /* First IRQ */
int irq = platform_get_irq_byname(pdev, "rx");  /* Named IRQ */

/* PCI device: */
int irq = pci_irq_vector(pdev, 0);         /* MSI/MSI-X */

/* Device Tree: */
int irq = of_irq_get(np, index);

/* GPIO as IRQ: */
int irq = gpiod_to_irq(desc);
```

---

## 25.6 IRQ Enabling / Disabling

```c
/* Disable specific IRQ (all CPUs): */
disable_irq(irq);          /* Waits for running handler to complete */
disable_irq_nosync(irq);   /* Returns immediately (use in ISR) */

/* Re-enable: */
enable_irq(irq);

/* These nest: must enable same number of times as disabled */

/* Disable all IRQs on local CPU: */
local_irq_disable();
local_irq_enable();

/* Save and restore: */
unsigned long flags;
local_irq_save(flags);
/* ... */
local_irq_restore(flags);
```

### When to Use

```
disable_irq:
  - During hardware reconfiguration
  - Synchronization point (ensure handler not running)
  - Before free_irq() in legacy code

disable_irq_nosync:
  - Inside the IRQ handler itself (already on this CPU)
  - Quick disable without waiting
```

---

## 25.7 IRQ Affinity From Driver

```c
/* Set IRQ affinity to specific CPU mask: */
cpumask_var_t mask;
alloc_cpumask_var(&mask, GFP_KERNEL);
cpumask_set_cpu(target_cpu, mask);
irq_set_affinity_hint(irq, mask);

/* For MSI-X: assign each queue to a CPU */
for (i = 0; i < num_queues; i++) {
    irq = pci_irq_vector(pdev, i);
    cpumask_set_cpu(i % num_online_cpus(), mask);
    irq_set_affinity_hint(irq, mask);
}
```

---

## 25.8 Error Handling and Edge Cases

```c
/* Common failure patterns: */

/* 1. IRQ request fails — check return: */
ret = devm_request_irq(...);
if (ret) {
    dev_err(dev, "IRQ request failed: %d\n", ret);
    return ret;
}

/* 2. Spurious interrupts — always check status register: */
static irqreturn_t my_isr(int irq, void *data)
{
    u32 status = readl(regs + STATUS);
    if (!status)
        return IRQ_NONE;  /* Spurious — MUST return IRQ_NONE */
    /* ... */
}

/* 3. IRQ storm prevention: */
/* If device keeps asserting, kernel will disable:
   "irq N: nobody cared (try booting with irqpoll)" */

/* 4. Timeout on wait: */
ret = wait_for_completion_timeout(&dev->done, HZ * 5);
if (!ret) {
    dev_err(dev->dev, "IRQ timeout — resetting hardware\n");
    hardware_reset(dev);
    return -ETIMEDOUT;
}
```

---

## Kernel Source References

```
IRQ API:
  include/linux/interrupt.h            ← request_irq, free_irq
  kernel/irq/manage.c                  ← IRQ management
  kernel/irq/chip.c                    ← IRQ chip ops

Platform IRQ:
  drivers/base/platform.c             ← platform_get_irq
  include/linux/platform_device.h

PCI MSI:
  drivers/pci/msi/                     ← MSI/MSI-X support
  include/linux/pci.h

Example drivers:
  drivers/iio/adc/                     ← Clean threaded IRQ usage
  drivers/net/ethernet/intel/          ← NAPI + MSI-X examples
  drivers/input/                       ← Simple IRQ patterns
```

---

## Interview Questions

1. **Walk through implementing a threaded IRQ handler for an I2C sensor.**
2. **What is IRQF_ONESHOT and why is it required for threaded handlers?**
3. **How do shared interrupts work? What must each handler do?**
4. **Explain the DMA + interrupt pattern for a read operation.**
5. **What happens when you return IRQ_NONE? When is it required?**
6. **What is the difference between disable_irq() and disable_irq_nosync()?**
7. **Why use devm_request_irq instead of request_irq?**
8. **How do you handle IRQ timeouts gracefully?**
9. **Compare top-half only, threaded IRQ, and NAPI approaches.**
10. **How do you assign IRQ affinity from driver code?**

---

## Summary

- Choose handler type based on work: top-half only (fast), threaded (slow bus), NAPI (network)
- Always check status register in shared IRQs; return IRQ_NONE if not yours
- Use IRQF_ONESHOT for threaded handlers to prevent IRQ storms
- DMA pattern: start transfer → wait for completion IRQ → copy data to user
- Use devm_request_irq for automatic cleanup on driver unbind
- Handle timeouts (wait_for_completion_timeout) and hardware errors gracefully

---

*Next: [Chapter 26 — Interrupt Debugging](Chapter_26_Interrupt_Debugging.md)*
