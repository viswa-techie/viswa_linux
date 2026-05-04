# Chapter 6: Interrupt Handling in Linux Kernel

## Learning Goals
- Master Linux's interrupt subsystem architecture (kernel/irq/)
- Understand irq_desc, irq_chip, irqaction — the core data structures
- Know how request_irq() works start to finish
- Write correct interrupt handlers and know the rules
- Understand shared interrupt handling and the action chain

---

## 6.1 Linux Interrupt Subsystem Overview

```
Linux Interrupt Subsystem Architecture:

  ┌─────────────────────────────────────────────────────────┐
  │                  Driver Layer                            │
  │  request_irq()  free_irq()  enable_irq()  disable_irq()│
  └──────────────────────┬──────────────────────────────────┘
                         │
  ┌──────────────────────▼──────────────────────────────────┐
  │              Generic IRQ Layer (kernel/irq/)            │
  │                                                         │
  │  ┌──────────────┐  ┌────────────┐  ┌────────────────┐ │
  │  │  irq_desc[]  │  │ Flow       │  │ IRQ Domain     │ │
  │  │  (per-IRQ    │  │ Handlers   │  │ (HW→Linux      │ │
  │  │   descriptor)│  │ (level,    │  │  IRQ mapping)  │ │
  │  │              │  │  edge,     │  │                │ │
  │  │  → irqaction │  │  fasteoi)  │  │                │ │
  │  │  → irq_data  │  │            │  │                │ │
  │  └──────────────┘  └────────────┘  └────────────────┘ │
  │                                                         │
  │  ┌──────────────┐  ┌────────────┐                      │
  │  │  irq_chip    │  │ Threaded   │                      │
  │  │  (controller │  │ IRQ        │                      │
  │  │   operations)│  │ Support    │                      │
  │  └──────────────┘  └────────────┘                      │
  └──────────────────────┬──────────────────────────────────┘
                         │
  ┌──────────────────────▼──────────────────────────────────┐
  │           Hardware Abstraction (arch/*/kernel/)          │
  │  APIC driver, GIC driver, GPIO-IRQ driver               │
  └─────────────────────────────────────────────────────────┘
```

---

## 6.2 Interrupt Descriptor Structures

### struct irq_desc — The Per-IRQ Descriptor

```c
/* include/linux/irqdesc.h (simplified) */
struct irq_desc {
    struct irq_data         irq_data;       /* chip, hwirq, domain */
    irq_flow_handler_t      handle_irq;     /* flow handler */
    struct irqaction        *action;         /* action chain (driver handlers) */
    unsigned int            status_use_accessors;
    unsigned int            depth;          /* disable nesting depth */
    unsigned int            irq_count;      /* IRQ occurrence count */
    unsigned long           last_unhandled; /* aging for spurious */
    unsigned int            irqs_unhandled; /* spurious IRQ counter */
    raw_spinlock_t          lock;           /* protects this descriptor */
    const char              *name;          /* /proc/interrupts name */
    int                     parent_irq;     /* for chained controllers */
    struct cpumask          *percpu_enabled; /* per-CPU enable mask */
    ktime_t                 tot_count;      /* time accounting */
    /* ... more fields ... */
};
```

### struct irq_data — Hardware-Side Information

```c
/* include/linux/irq.h (simplified) */
struct irq_data {
    unsigned int            irq;        /* Linux IRQ number */
    unsigned long           hwirq;      /* Hardware IRQ number */
    struct irq_chip         *chip;      /* Controller operations */
    struct irq_domain       *domain;    /* IRQ domain */
    void                    *chip_data; /* Controller private data */
    struct irq_data         *parent_data; /* For hierarchical domains */
    /* ... */
};
```

### struct irq_chip — Controller Operations

```c
/* include/linux/irq.h (simplified) */
struct irq_chip {
    const char  *name;
    
    void        (*irq_startup)(struct irq_data *data);
    void        (*irq_shutdown)(struct irq_data *data);
    void        (*irq_enable)(struct irq_data *data);
    void        (*irq_disable)(struct irq_data *data);
    
    void        (*irq_ack)(struct irq_data *data);      /* acknowledge */
    void        (*irq_mask)(struct irq_data *data);      /* mask (disable) */
    void        (*irq_unmask)(struct irq_data *data);    /* unmask (enable) */
    void        (*irq_eoi)(struct irq_data *data);       /* end of interrupt */
    
    int         (*irq_set_affinity)(struct irq_data *data,
                    const struct cpumask *dest, bool force);
    int         (*irq_set_type)(struct irq_data *data,
                    unsigned int type);  /* edge/level */
    /* ... */
};

/* Example: GIC chip operations */
static struct irq_chip gic_chip = {
    .name           = "GICv3",
    .irq_mask       = gic_mask_irq,
    .irq_unmask     = gic_unmask_irq,
    .irq_eoi        = gic_eoi_irq,
    .irq_set_type   = gic_set_type,
    .irq_set_affinity = gic_set_affinity,
};
```

### struct irqaction — Driver Handler Registration

```c
/* include/linux/interrupt.h (simplified) */
struct irqaction {
    irq_handler_t       handler;    /* hardirq handler function */
    void                *dev_id;    /* device identification */
    unsigned int        irq;        /* IRQ number */
    unsigned int        flags;      /* IRQF_* flags */
    const char          *name;      /* /proc/interrupts name */
    
    irq_handler_t       thread_fn;  /* threaded handler function */
    struct task_struct  *thread;    /* IRQ thread (if threaded) */
    
    struct irqaction    *next;      /* next action in shared chain */
    /* ... */
};
```

### Data Structure Relationships

```
  irq_desc[N]
  ┌────────────────────────────┐
  │ .irq_data                  │
  │   ├── .irq = N (Linux IRQ) │
  │   ├── .hwirq = H (HW IRQ) │
  │   ├── .chip → irq_chip     │──→ GIC/APIC ops
  │   └── .domain → irq_domain │──→ HW←→Linux mapping
  │                             │
  │ .handle_irq                 │──→ handle_fasteoi_irq()
  │                             │
  │ .action                     │──→ irqaction (driver 1)
  │                             │      │ .handler = handler_A
  │                             │      │ .dev_id = dev_A
  │                             │      │ .next
  │                             │      └──→ irqaction (driver 2)
  │                             │            .handler = handler_B
  │                             │            .dev_id = dev_B
  │                             │            .next = NULL
  └────────────────────────────┘
```

---

## 6.3 Interrupt Vector Table in Linux

### x86: Interrupt Descriptor Table (IDT)

```c
/* Populated during boot: arch/x86/kernel/idt.c */
/* Vectors 0-31: CPU exceptions (fixed) */
/* Vectors 32-255: programmable (devices + special) */

/* Common entry point for device IRQs: */
/* Each vector → stub → common_interrupt → do_IRQ */

/* Key: common_interrupt calls:
   handle_irq(desc) which dispatches to handle_fasteoi_irq etc. */
```

### ARM64: Exception Vector Table

```asm
/* arch/arm64/kernel/entry.S */
SYM_CODE_START(vectors)
    /* Current EL with SP_EL0 */
    kernel_ventry  el1t_64_sync    /* offset 0x000 */
    kernel_ventry  el1t_64_irq     /* offset 0x080 */
    kernel_ventry  el1t_64_fiq     /* offset 0x100 */
    kernel_ventry  el1t_64_error   /* offset 0x180 */
    
    /* Current EL with SP_ELx (normal kernel mode) */
    kernel_ventry  el1h_64_sync    /* offset 0x200 */
    kernel_ventry  el1h_64_irq     /* offset 0x280 ← most IRQs */
    kernel_ventry  el1h_64_fiq     /* offset 0x300 */
    kernel_ventry  el1h_64_error   /* offset 0x380 */
    /* ... lower EL entries ... */
SYM_CODE_END(vectors)

/* el1h_64_irq_handler → gic_handle_irq → generic_handle_domain_irq */
```

---

## 6.4 Interrupt Registration in Drivers

### request_irq() API

```c
/* include/linux/interrupt.h */
static inline int request_irq(
    unsigned int irq,           /* Linux IRQ number */
    irq_handler_t handler,      /* hardirq handler function */
    unsigned long flags,        /* IRQF_* flags */
    const char *name,           /* identifier for /proc/interrupts */
    void *dev_id               /* passed to handler; for shared: MUST be non-NULL */
);

/* Returns: 0 on success, negative errno on failure */
```

### Common IRQF Flags

```c
IRQF_SHARED        /* IRQ shared between multiple devices */
IRQF_TRIGGER_RISING /* Edge: low-to-high */
IRQF_TRIGGER_FALLING/* Edge: high-to-low */
IRQF_TRIGGER_HIGH   /* Level: active high */
IRQF_TRIGGER_LOW    /* Level: active low */
IRQF_ONESHOT        /* Keep masked until threaded handler completes */
IRQF_NO_SUSPEND     /* Keep enabled during system suspend */
IRQF_NO_THREAD      /* Never force-thread (even on PREEMPT_RT) */
```

### Complete Registration Example

```c
#include <linux/interrupt.h>

struct my_device {
    void __iomem *regs;
    int irq;
    spinlock_t lock;
    struct tasklet_struct tasklet;
};

static irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;
    u32 status;
    
    /* Read device interrupt status */
    status = readl(dev->regs + INT_STATUS_REG);
    
    /* Check if this interrupt is for us (CRITICAL for shared) */
    if (!(status & OUR_DEVICE_INT_MASK))
        return IRQ_NONE;  /* Not ours */
    
    /* Acknowledge interrupt at hw (clear status bits) */
    writel(status, dev->regs + INT_CLEAR_REG);
    
    /* Quick work in hardirq context */
    spin_lock(&dev->lock);
    /* ... minimal processing ... */
    spin_unlock(&dev->lock);
    
    /* Schedule heavy work for bottom half */
    tasklet_schedule(&dev->tasklet);
    
    return IRQ_HANDLED;
}

static int my_probe(struct platform_device *pdev)
{
    struct my_device *dev;
    int ret;
    
    dev = devm_kzalloc(&pdev->dev, sizeof(*dev), GFP_KERNEL);
    dev->irq = platform_get_irq(pdev, 0);
    dev->regs = devm_ioremap_resource(&pdev->dev, ...);
    
    ret = request_irq(dev->irq, my_irq_handler,
                      IRQF_SHARED, "my-device", dev);
    if (ret) {
        dev_err(&pdev->dev, "Failed to request IRQ %d: %d\n",
                dev->irq, ret);
        return ret;
    }
    
    return 0;
}

static int my_remove(struct platform_device *pdev)
{
    struct my_device *dev = platform_get_drvdata(pdev);
    free_irq(dev->irq, dev);  /* dev_id must match! */
    return 0;
}
```

### Managed (devm) Version

```c
/* Automatically freed when device is removed */
ret = devm_request_irq(&pdev->dev, dev->irq, my_irq_handler,
                       IRQF_SHARED, "my-device", dev);
/* No need for free_irq() in remove function! */
```

---

## 6.5 request_irq() Mechanism Internalized

```
request_irq(irq, handler, flags, name, dev_id)
    │
    ▼
__setup_irq(irq_desc, irqaction)
    │
    ├── Allocate and fill struct irqaction
    │   .handler = handler
    │   .flags = flags
    │   .name = name
    │   .dev_id = dev_id
    │
    ├── Validate:
    │   - For IRQF_SHARED: dev_id must be non-NULL
    │   - For IRQF_SHARED: trigger type must match existing
    │   - Handler must not be NULL
    │
    ├── If first handler on this IRQ:
    │   - irq_startup() → irq_chip->irq_enable() → unmask
    │
    ├── If IRQF_SHARED and existing handlers:
    │   - Append to irq_desc->action linked list
    │   - Check trigger type compatibility
    │
    ├── If threaded (request_threaded_irq):
    │   - Create kernel thread: irq/<irq>-<name>
    │   - action->thread = kthread_create(irq_thread, ...)
    │
    └── Register in /proc/interrupts
```

---

## 6.6 Interrupt Handler Execution

### handler return values

```c
/* include/linux/irqreturn.h */
enum irqreturn {
    IRQ_NONE        = 0,  /* Not our interrupt */
    IRQ_HANDLED     = 1,  /* Successfully handled */
    IRQ_WAKE_THREAD = 2,  /* Wake threaded handler */
};
typedef enum irqreturn irqreturn_t;
```

### Rules for Interrupt Handlers

```
What you CAN do in hardirq context:
  ✓ Read/write device registers (readl/writel)
  ✓ Use spin_lock() / spin_unlock() (NOT spin_lock_irq!)
  ✓ Use atomic operations
  ✓ Allocate with GFP_ATOMIC (may fail!)
  ✓ Schedule bottom halves (tasklet_schedule, queue_work)
  ✓ Wake up tasks (wake_up_interruptible)

What you CANNOT do:
  ✗ Sleep or block (schedule(), msleep(), wait_event)
  ✗ Allocate with GFP_KERNEL (might sleep)
  ✗ Use mutex_lock() (sleeps!)
  ✗ Copy to/from user space (copy_to_user)
  ✗ Call printk() extensively (use trace instead)
  ✗ Do heavy computation (keep under ~10 µs ideally)
```

### Shared Interrupt Handler Chain

```
When a shared IRQ fires:

  handle_level_irq(desc):
      irq_chip->irq_mask_ack()      ← Mask and acknowledge
      
      for each action in desc->action list:
          ret = action->handler(irq, action->dev_id)
          if (ret == IRQ_HANDLED)
              handled = true
      
      irq_chip->irq_unmask()         ← Unmask
      
      if (!handled)
          note_interrupt(desc) → spurious tracking
```

---

## 6.7 Interrupt Return Flow

```
After all handlers execute:

  1. Flow handler returns to generic_handle_irq()
  2. Returns to architecture IRQ dispatcher
  3. irq_exit():
     ├── preempt_count -= HARDIRQ_OFFSET
     ├── if (local_softirq_pending())
     │   └── invoke_softirq()
     │       └── __do_softirq() or wakeup_softirqd
     └── tick_irq_exit() (time accounting)
  4. Architecture return code:
     ├── If returning to user mode:
     │   ├── Check TIF_NEED_RESCHED → maybe schedule()
     │   ├── Check TIF_SIGPENDING → do_signal()
     │   └── Restore user registers → IRETQ/ERET
     └── If returning to kernel mode:
         ├── If CONFIG_PREEMPT && TIF_NEED_RESCHED:
         │   └── preempt_schedule_irq()
         └── Restore kernel registers → resume
```

---

## Debugging Tip: /proc/interrupts

```bash
$ cat /proc/interrupts
           CPU0       CPU1       CPU2       CPU3
  1:          9          0          0          0   IO-APIC   1-edge      i8042
  8:          0          0          0          0   IO-APIC   8-edge      rtc0
 16:     123456          0          0          0   IO-APIC  16-fasteoi   ehci_hcd
 33:          0     234567          0          0   PCI-MSI  524288-edge  nvme0q0
 34:          0          0     345678          0   PCI-MSI  524289-edge  nvme0q1
IPI:      12345      12346      12347      12348   Inter-processor interrupts
LOC:    9876543    9876544    9876545    9876546   Local timer interrupts
```

Each column = per-CPU count. This shows:
- Which IRQs are active
- How they're distributed across CPUs
- The controller type and trigger mode
- The driver name (registered via request_irq)

---

## Kernel Source References

```
Core IRQ management:
  kernel/irq/manage.c           ← request_irq, free_irq, __setup_irq
  kernel/irq/handle.c           ← generic_handle_irq dispatch
  kernel/irq/chip.c             ← handle_level_irq, handle_edge_irq
  kernel/irq/spurious.c         ← Spurious IRQ detection
  kernel/irq/irqdesc.c          ← irq_desc allocation and management
  kernel/irq/irqdomain.c        ← HW-to-Linux IRQ mapping

Key includes:
  include/linux/interrupt.h     ← request_irq API, irqreturn_t
  include/linux/irq.h           ← irq_chip, irq_data, flow types
  include/linux/irqdesc.h       ← struct irq_desc
```

---

## Interview Questions

1. **Describe the three core data structures of Linux's IRQ subsystem.**
2. **What happens inside request_irq()? Walk through the flow.**
3. **What is the irq_chip abstraction and why does it exist?**
4. **Write a correct shared interrupt handler. What rules must it follow?**
5. **What is the difference between IRQ_HANDLED and IRQ_NONE?**
6. **What is handle_fasteoi_irq and how does it differ from handle_level_irq?**
7. **Why can't you sleep in a hardirq handler?**
8. **What is devm_request_irq and why is it preferred?**
9. **How does Linux handle shared IRQs (multiple devices on the same line)?**
10. **What is a spurious interrupt and how does Linux detect them?**
11. **What is an IRQ domain and why was it introduced?**
12. **What happens if you forget to call free_irq() when your driver is unloaded?**
13. **Explain the irqaction linked list for shared IRQs.**
14. **What flags would you use for a GPIO interrupt that triggers on rising edge?**
15. **How does /proc/interrupts relate to irq_desc?**

---

## Summary

- Linux's IRQ subsystem is built around three core structures: irq_desc (per-IRQ state), irq_chip (controller ops), irqaction (driver handler)
- request_irq() creates an irqaction and links it into the irq_desc's action chain
- Flow handlers (level, edge, fasteoi) encapsulate controller-specific handling sequences
- Interrupt handlers run in hardirq context: no sleeping, minimal work, schedule bottom halves
- Shared interrupts walk the action list; each handler must check if the IRQ is really for its device
- irq_domain provides the mapping from hardware interrupt numbers to Linux IRQ numbers

---

*Next: [Chapter 7 — Top Half and Bottom Half Mechanisms](Chapter_07_Top_Bottom_Half.md)*
