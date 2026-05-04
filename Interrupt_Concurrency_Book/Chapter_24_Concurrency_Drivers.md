# Chapter 24: Concurrency in Drivers

## Learning Goals
- Apply all synchronization primitives in real driver scenarios
- Design correct locking for char, block, and network drivers
- Handle shared state between file operations, IRQs, and workqueues
- Avoid common concurrency bugs in driver code

---

## 24.1 Driver Concurrency Sources

A driver's data is subject to concurrent access from multiple paths:

```
                     ┌──────────────┐
     User space      │  open/read/  │  Multiple processes/threads
     system calls    │  write/ioctl │  call simultaneously
                     └──────┬───────┘
                            │
           ┌────────────────┼────────────────┐
           │                │                │
     ┌─────▼─────┐   ┌─────▼─────┐   ┌──────▼──────┐
     │ Process    │   │ Bottom    │   │ Hard IRQ    │
     │ context    │   │ half      │   │ handler     │
     │ (fops)     │   │(workqueue)│   │             │
     └─────┬──────┘   └─────┬─────┘   └──────┬──────┘
           │                │                 │
           └────────────────┼─────────────────┘
                            │
                     ┌──────▼───────┐
                     │  Device      │
                     │  struct data │  ← Needs protection!
                     └──────────────┘
```

### Concurrency Scenarios in Drivers

```
Scenario                          │ Example
──────────────────────────────────┼──────────────────────────────
Multiple opens                    │ Two processes open /dev/foo
Concurrent read + write           │ Thread A reads, thread B writes
ioctl during I/O                  │ Configuration change mid-transfer
IRQ during file operation         │ IRQ fires while read() in progress
Workqueue + ioctl                 │ Deferred work + control path
probe + user access               │ Device used before probe completes
remove during active I/O          │ Unplug while device is open
SMP: same fop on different CPUs   │ Two CPUs execute read() concurrently
```

---

## 24.2 Locking Design Patterns

### Pattern 1: Single Device Lock

```c
struct my_device {
    struct mutex    lock;           /* Protects all fields below */
    struct cdev     cdev;
    void __iomem   *regs;
    u8             *buffer;
    size_t          buf_len;
    bool            running;
};

static ssize_t my_read(struct file *file, char __user *buf,
                        size_t count, loff_t *ppos)
{
    struct my_device *dev = file->private_data;

    mutex_lock(&dev->lock);
    if (!dev->running) {
        mutex_unlock(&dev->lock);
        return -EIO;
    }
    /* ... copy data to user ... */
    mutex_unlock(&dev->lock);
    return count;
}
```

**Pros**: Simple, easy to reason about.
**Cons**: Serializes all access — poor performance if reads don't conflict.

### Pattern 2: Fine-Grained Locking

```c
struct my_device {
    spinlock_t      irq_lock;      /* Protects hw_status, IRQ shared */
    struct mutex    io_lock;        /* Protects buffer, process only */
    struct mutex    config_lock;    /* Protects configuration */

    /* Protected by irq_lock: */
    u32             hw_status;
    bool            irq_pending;

    /* Protected by io_lock: */
    u8             *buffer;
    size_t          buf_pos;

    /* Protected by config_lock: */
    u32             baudrate;
    u32             mode;
};
```

**Pros**: Higher concurrency (config doesn't block I/O).
**Cons**: More complex, must document lock ordering.

### Pattern 3: Lock-Free Fast Path + Lock for Setup

```c
struct my_device {
    struct mutex        config_lock;    /* Setup/teardown */
    atomic_t            state;          /* Lock-free state check */
    atomic64_t          bytes_total;    /* Lock-free statistics */
    struct completion   io_done;        /* Wait for DMA */
};

static ssize_t my_read(struct file *file, char __user *buf,
                        size_t count, loff_t *ppos)
{
    struct my_device *dev = file->private_data;

    /* Lock-free fast-path check: */
    if (atomic_read(&dev->state) != STATE_RUNNING)
        return -EIO;

    /* Start DMA (atomic register write): */
    writel(count, dev->regs + DMA_LEN);
    writel(DMA_START, dev->regs + DMA_CTRL);

    /* Wait for completion (set by IRQ handler): */
    wait_for_completion(&dev->io_done);

    atomic64_add(count, &dev->bytes_total);
    return count;
}
```

---

## 24.3 Char Driver Complete Example

```c
#include <linux/module.h>
#include <linux/cdev.h>
#include <linux/fs.h>
#include <linux/mutex.h>
#include <linux/spinlock.h>
#include <linux/interrupt.h>
#include <linux/wait.h>

#define BUF_SIZE 4096

struct my_char_dev {
    struct cdev         cdev;
    struct device      *dev;
    void __iomem       *regs;
    int                 irq;

    /* Protects buffer (process context only): */
    struct mutex        buf_lock;

    /* Protects hw regs, shared with IRQ: */
    spinlock_t          hw_lock;

    /* Buffer: */
    u8                  buffer[BUF_SIZE];
    size_t              data_len;

    /* IRQ signaling: */
    wait_queue_head_t   read_wait;
    bool                data_ready;     /* Protected by hw_lock */

    /* Device state: */
    atomic_t            open_count;
};

/* === IRQ Handler (hardirq context) === */
static irqreturn_t my_irq(int irq, void *data)
{
    struct my_char_dev *dev = data;
    u32 status;

    spin_lock(&dev->hw_lock);
    status = readl(dev->regs + IRQ_STATUS);
    if (!(status & MY_IRQ_BIT)) {
        spin_unlock(&dev->hw_lock);
        return IRQ_NONE;
    }

    /* ACK interrupt: */
    writel(MY_IRQ_BIT, dev->regs + IRQ_ACK);

    /* Copy data from hardware FIFO: */
    dev->data_len = readl(dev->regs + FIFO_LEN);
    memcpy_fromio(dev->buffer, dev->regs + FIFO_DATA, dev->data_len);
    dev->data_ready = true;

    spin_unlock(&dev->hw_lock);

    wake_up_interruptible(&dev->read_wait);
    return IRQ_HANDLED;
}

/* === open (process context) === */
static int my_open(struct inode *inode, struct file *file)
{
    struct my_char_dev *dev = container_of(inode->i_cdev,
                                           struct my_char_dev, cdev);
    atomic_inc(&dev->open_count);
    file->private_data = dev;
    return 0;
}

/* === read (process context, may sleep) === */
static ssize_t my_read(struct file *file, char __user *buf,
                        size_t count, loff_t *ppos)
{
    struct my_char_dev *dev = file->private_data;
    unsigned long flags;
    ssize_t ret;

    /* Wait for data from IRQ: */
    ret = wait_event_interruptible(dev->read_wait, dev->data_ready);
    if (ret)
        return ret;

    /* Lock buffer (process context only): */
    mutex_lock(&dev->buf_lock);

    /* Access IRQ-shared flag with spinlock: */
    spin_lock_irqsave(&dev->hw_lock, flags);
    dev->data_ready = false;
    spin_unlock_irqrestore(&dev->hw_lock, flags);

    /* Copy to user (can sleep — outside spinlock): */
    count = min(count, dev->data_len);
    if (copy_to_user(buf, dev->buffer, count))
        ret = -EFAULT;
    else
        ret = count;

    mutex_unlock(&dev->buf_lock);
    return ret;
}

/* === release (process context) === */
static int my_release(struct inode *inode, struct file *file)
{
    struct my_char_dev *dev = file->private_data;
    atomic_dec(&dev->open_count);
    return 0;
}
```

---

## 24.4 Network Driver Concurrency

```c
struct my_netdev_priv {
    spinlock_t          tx_lock;    /* Protect TX ring */
    struct napi_struct  napi;       /* NAPI for RX */

    /* TX ring (shared between ndo_start_xmit and TX IRQ): */
    struct tx_desc     *tx_ring;
    u32                 tx_head;    /* Written by xmit */
    u32                 tx_tail;    /* Written by IRQ */
};

/* === TX path (process context or softirq) === */
static netdev_tx_t my_start_xmit(struct sk_buff *skb,
                                  struct net_device *ndev)
{
    struct my_netdev_priv *priv = netdev_priv(ndev);

    spin_lock(&priv->tx_lock);
    if (tx_ring_full(priv)) {
        spin_unlock(&priv->tx_lock);
        return NETDEV_TX_BUSY;
    }
    fill_tx_descriptor(priv, skb);
    priv->tx_head++;
    spin_unlock(&priv->tx_lock);

    /* Kick hardware: */
    writel(priv->tx_head, priv->regs + TX_DOORBELL);
    return NETDEV_TX_OK;
}

/* === RX path (NAPI poll, softirq context) === */
static int my_napi_poll(struct napi_struct *napi, int budget)
{
    struct my_netdev_priv *priv = container_of(napi,
                                    struct my_netdev_priv, napi);
    int processed = 0;

    /* No locking needed — NAPI guarantees single-threaded poll */
    while (processed < budget && rx_desc_available(priv)) {
        struct sk_buff *skb = receive_packet(priv);
        napi_gro_receive(napi, skb);
        processed++;
    }

    if (processed < budget) {
        napi_complete_done(napi, processed);
        enable_rx_irq(priv);
    }
    return processed;
}
```

---

## 24.5 Race Conditions at probe/remove

```c
static int my_probe(struct platform_device *pdev)
{
    struct my_device *dev;

    dev = devm_kzalloc(&pdev->dev, sizeof(*dev), GFP_KERNEL);
    mutex_init(&dev->lock);
    spin_lock_init(&dev->hw_lock);
    init_waitqueue_head(&dev->read_wait);

    /* Initialize hardware BEFORE exposing device to users: */
    dev->regs = devm_ioremap_resource(&pdev->dev, res);
    hardware_init(dev);

    /* Register IRQ: */
    devm_request_irq(&pdev->dev, dev->irq, my_irq, 0, "mydev", dev);

    /* === Expose device LAST (after everything initialized): === */
    cdev_add(&dev->cdev, devno, 1);  /* Now users can open */
    platform_set_drvdata(pdev, dev);

    return 0;
}

static int my_remove(struct platform_device *pdev)
{
    struct my_device *dev = platform_get_drvdata(pdev);

    /* === Remove user access FIRST: === */
    cdev_del(&dev->cdev);

    /* Then tear down (no new users can arrive): */
    mutex_lock(&dev->lock);
    dev->running = false;
    mutex_unlock(&dev->lock);

    /* Flush pending work: */
    cancel_work_sync(&dev->work);

    /* Hardware cleanup: */
    hardware_shutdown(dev);

    return 0;
}
```

### The "Open After Remove" Race

```
Problem:
  CPU 0: remove() starts         CPU 1: open() called
  ...                             dev = container_of(cdev)
  cdev_del()                      ... racing ...
  kfree(dev)                      dev->lock → USE AFTER FREE!

Solution: Use a device state flag or devm_* (managed resources):
  - Set state to "shutting down" before cdev_del
  - open() checks state
  - Or use misc_deregister() which waits for active users
```

---

## 24.6 Locking Rules Summary

```
Operation                    │ Lock to use
─────────────────────────────┼──────────────────────────────
File ops (read/write/ioctl)  │ mutex (can sleep)
File ops ↔ IRQ shared data   │ spin_lock_irqsave
File ops ↔ workqueue         │ mutex (both process context)
IRQ handler only             │ spin_lock (IRQs off)
Statistics counters          │ per-CPU or atomic_t
Device state (open/close)    │ atomic_t
Hardware register access     │ spin_lock_irqsave (if IRQ shares)
DMA descriptor ring          │ spinlock (short, no sleep)
Configuration update         │ mutex + RCU (readers lock-free)
```

---

## Kernel Source References

```
Driver concurrency patterns:
  drivers/char/                        ← Char driver examples
  drivers/net/ethernet/                ← Network driver patterns
  drivers/iio/                         ← IIO (clean locking patterns)
  Documentation/driver-api/            ← Driver API docs
```

---

## Interview Questions

1. **What concurrency challenges does a Linux device driver face?**
2. **Design the locking for a char driver with IRQ + ioctl + read.**
3. **Why use mutex for file operations and spinlock for hardware access?**
4. **How do you protect the probe/remove race?**
5. **What locking does NAPI guarantee for the poll function?**
6. **When should you use atomic_t vs mutex in a driver?**
7. **What is the danger of cdev_add before hardware initialization?**
8. **How do you handle device removal while users have it open?**
9. **Explain fine-grained vs coarse-grained locking trade-offs in drivers.**
10. **How would you add per-CPU statistics to a network driver?**

---

## Summary

- Drivers face concurrency from user fops, IRQs, bottom halves, and SMP
- Use mutex for process-context-only paths (can sleep)
- Use spin_lock_irqsave for data shared with IRQ handlers
- Initialize hardware BEFORE cdev_add; cdev_del BEFORE teardown
- NAPI poll is single-threaded per instance — no extra locking for RX ring
- Combine primitives: mutex for setup, spinlock for fast-path, atomic for state
- Always document which lock protects which data

---

*Next: [Chapter 25 — Interrupt Handling in Drivers](Chapter_25_Interrupt_Handling_Drivers.md)*
