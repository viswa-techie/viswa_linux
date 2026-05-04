# Chapter 23: Interrupt Initialization

## Learning Goals
- Understand IRQ subsystem initialization during boot
- Know GIC (ARM) and APIC (x86) interrupt controller setup
- Grasp interrupt vector table configuration
- Understand IRQ domain and descriptor allocation

---

## 23.1 Interrupt Architecture Overview

```
Interrupt Flow (Hardware → Handler):

Hardware Device
     │
     │ IRQ signal (electrical)
     ▼
┌──────────────────────────────────┐
│  Interrupt Controller            │
│  (GIC on ARM / APIC on x86)     │
│  - Priority arbitration          │
│  - CPU targeting                 │
│  - IRQ number → vector mapping   │
└──────────┬───────────────────────┘
           │
           │ Interrupt delivered to CPU
           ▼
┌──────────────────────────────────┐
│  CPU                             │
│  1. Saves context               │
│  2. Looks up vector table        │
│  3. Jumps to vector handler      │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  Linux IRQ handling              │
│  1. irq_enter()                  │
│  2. Look up irq_desc for hwirq   │
│  3. Call registered handler      │
│  4. irq_exit() → softirq check   │
└──────────────────────────────────┘
```

---

## 23.2 Interrupt Init During Boot

```
start_kernel() Interrupt Initialization:

start_kernel()
  │
  ├── trap_init()                ← Install exception handlers
  │   └── Set up exception vectors (div-by-zero, page fault, etc.)
  │
  ├── early_irq_init()           ← Pre-allocate irq_desc structures
  │   └── Allocate NR_IRQS descriptors (or use radix tree)
  │
  ├── init_IRQ()                 ← Architecture-specific IRQ init
  │   │
  │   ├── x86: native_init_IRQ()
  │   │   ├── Set up IDT entries for hardware IRQs
  │   │   ├── Init APIC
  │   │   └── Configure I/O APIC from ACPI/MP tables
  │   │
  │   └── ARM64: irqchip_init()
  │       ├── Parse device tree /interrupt-controller nodes
  │       ├── Initialize GIC (gic_of_init)
  │       └── Set up IRQ domain hierarchy
  │
  ├── softirq_init()             ← Initialize softirq subsystem
  │   ├── Register TASKLET_SOFTIRQ
  │   └── Register HI_SOFTIRQ
  │
  └── local_irq_enable()         ← Enable interrupts on boot CPU
```

---

## 23.3 ARM GIC Initialization

```
GIC (Generic Interrupt Controller) Architecture:

┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   Device A   │   │   Device B   │   │   Device C   │
│   SPI 32     │   │   SPI 45     │   │   PPI 30     │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                  │                  │
       ▼                  ▼                  ▼
┌────────────────────────────────────────────────────┐
│                   GIC Distributor                   │
│  ┌──────────────────────────────────────────────┐  │
│  │ - Enable/disable each interrupt              │  │
│  │ - Set priority for each interrupt            │  │
│  │ - Route interrupt to target CPU(s)           │  │
│  │ - Record pending/active state                │  │
│  └──────────────────────────────────────────────┘  │
└────────────┬────────────────────┬──────────────────┘
             │                    │
             ▼                    ▼
    ┌─────────────────┐  ┌─────────────────┐
    │ CPU Interface 0 │  │ CPU Interface 1 │
    │ - Priority mask  │  │ - Priority mask  │
    │ - Preemption     │  │ - Preemption     │
    │ - ACK/EOI        │  │ - ACK/EOI        │
    └────────┬────────┘  └────────┬────────┘
             │                    │
             ▼                    ▼
          CPU 0                CPU 1

Interrupt types:
  SGI (Software Generated): 0-15   (inter-CPU signaling)
  PPI (Private Per-processor): 16-31 (per-CPU timers, PMU)
  SPI (Shared Peripheral): 32-1019  (devices)
  LPI (Locality-specific): 8192+    (GICv3+, MSI)
```

```c
/* GIC initialization from device tree */
static int __init gic_of_init(struct device_node *node,
                              struct device_node *parent)
{
    void __iomem *dist_base, *cpu_base;

    /* Map distributor registers */
    dist_base = of_iomap(node, 0);
    /* Map CPU interface registers */
    cpu_base = of_iomap(node, 1);

    /* Initialize the GIC */
    gic_init_bases(gic_cnt, dist_base, cpu_base, node);

    /* Set up IRQ domain for this GIC */
    gic->domain = irq_domain_create_linear(
        of_node_to_fwnode(node),
        gic_irqs,              /* number of IRQs */
        &gic_irq_domain_ops,   /* domain callbacks */
        gic
    );

    /* Set this GIC as the root interrupt controller */
    set_handle_irq(gic_handle_irq);

    return 0;
}
```

---

## 23.4 x86 APIC Initialization

```
x86 APIC Architecture:

┌────────────────────────────────────────────────┐
│   External Devices (PCI, USB, etc.)            │
│   Connected via I/O APIC redirection entries   │
└────────────┬───────────────────────────────────┘
             │
             ▼
┌────────────────────────────┐
│   I/O APIC                 │
│   ├── 24 redirection       │
│   │   entries              │
│   ├── Routes device IRQs   │
│   │   to Local APICs       │
│   └── Delivery mode:       │
│       fixed/lowest/NMI     │
└────────────┬───────────────┘
             │
     ┌───────┴────────┐
     ▼                ▼
┌──────────┐   ┌──────────┐
│Local APIC│   │Local APIC│
│  CPU 0   │   │  CPU 1   │
│ - Timer  │   │ - Timer  │
│ - IPI    │   │ - IPI    │
│ - Vectors│   │ - Vectors│
└──────────┘   └──────────┘

IDT (Interrupt Descriptor Table):
  Vector 0-31:   CPU exceptions (page fault, GPF, etc.)
  Vector 32-127: Device interrupts
  Vector 128:    System call (int 0x80)
  Vector 239:    Local APIC timer
  Vector 240-255: IPI and special vectors
```

---

## 23.5 IRQ Domain Framework

```
IRQ Domain — Maps Hardware IRQs to Linux IRQ Numbers:

Device Tree describes interrupts:
  device@1234 {
      interrupts = <0 45 IRQ_TYPE_LEVEL_HIGH>;
      /*            ^  ^  ^                    */
      /*            |  |  └── trigger type     */
      /*            |  └── hwirq number        */
      /*            └── type (0=SPI,1=PPI)     */
  };

Mapping flow:
┌──────────────────────────────────────────────┐
│        Device Tree hwirq = 45                │
│                    │                         │
│                    ▼                         │
│  ┌──────────────────────────────────┐        │
│  │  IRQ Domain                      │        │
│  │  irq_domain_ops.xlate()          │        │
│  │  → Parse intspec, get hwirq=45   │        │
│  │  irq_domain_ops.map()            │        │
│  │  → Allocate linux_irq, set chip  │        │
│  └──────────────────────────────────┘        │
│                    │                         │
│                    ▼                         │
│  linux_irq = 67 (arbitrary kernel number)    │
│                    │                         │
│                    ▼                         │
│  ┌──────────────────────────────────┐        │
│  │  struct irq_desc #67             │        │
│  │  ├── irq_data.hwirq = 45        │        │
│  │  ├── irq_data.chip = &gic_chip  │        │
│  │  ├── action → handler_fn        │        │
│  │  └── status flags               │        │
│  └──────────────────────────────────┘        │
└──────────────────────────────────────────────┘
```

---

## 23.6 Interrupt Handler Registration

```c
/* Driver requesting an interrupt */
static irqreturn_t my_isr(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;

    /* Read status register to determine interrupt cause */
    u32 status = readl(dev->regs + STATUS_REG);

    if (!(status & MY_IRQ_PENDING))
        return IRQ_NONE;  /* Not our interrupt */

    /* Acknowledge the interrupt in hardware */
    writel(status, dev->regs + STATUS_REG);

    /* Schedule bottom half for heavy processing */
    tasklet_schedule(&dev->tasklet);

    return IRQ_HANDLED;
}

static int my_probe(struct platform_device *pdev)
{
    int irq, ret;

    /* Get IRQ from device tree (resolved through IRQ domain) */
    irq = platform_get_irq(pdev, 0);

    /* Register the handler */
    ret = devm_request_irq(&pdev->dev, irq, my_isr,
                           IRQF_SHARED,      /* flags */
                           "my-device",       /* name */
                           my_dev);           /* dev_id */
    return ret;
}
```

---

## 23.7 Softirq and Bottom Half Init

```
Softirq Initialization:

softirq_init() in start_kernel():
  ├── Register HI_SOFTIRQ → tasklet_hi_action
  └── Register TASKLET_SOFTIRQ → tasklet_action

open_softirq() — used by subsystems:
  ├── TIMER_SOFTIRQ      → run_timer_softirq()
  ├── NET_TX_SOFTIRQ     → net_tx_action()
  ├── NET_RX_SOFTIRQ     → net_rx_action()
  ├── BLOCK_SOFTIRQ      → blk_done_softirq()
  ├── IRQ_POLL_SOFTIRQ   → irq_poll_softirq()
  ├── SCHED_SOFTIRQ      → run_rebalance_domains()
  ├── HRTIMER_SOFTIRQ    → hrtimer_run_softirq()
  └── RCU_SOFTIRQ        → rcu_core_si()

Processing:
  irq_exit() checks if softirqs are pending:
    if (local_softirq_pending())
        invoke_softirq() → __do_softirq()

  If too many iterations → wake ksoftirqd thread
```

---

## Kernel Source References

| Function/File | Path | Purpose |
|-------|------|---------|
| early_irq_init() | kernel/irq/irqdesc.c | Pre-allocate IRQ descriptors |
| init_IRQ() | arch/*/kernel/irq.c | Architecture IRQ init |
| trap_init() | arch/*/kernel/traps.c | Exception vector setup |
| gic_of_init() | drivers/irqchip/irq-gic.c | GICv2 initialization |
| gic_of_init() | drivers/irqchip/irq-gic-v3.c | GICv3 initialization |
| softirq_init() | kernel/softirq.c | Softirq subsystem init |
| irq_domain_create_linear() | kernel/irq/irqdomain.c | Create IRQ domain |
| request_irq() | kernel/irq/manage.c | Register interrupt handler |

---

## Interview Questions

**Q1: What is the difference between GIC and APIC?**
A: GIC (Generic Interrupt Controller) is the standard interrupt controller for ARM — it has a Distributor (routes IRQs to CPUs) and per-CPU interfaces. APIC (Advanced Programmable Interrupt Controller) is for x86 — it has I/O APICs for device routing and Local APICs per CPU. Both serve the same purpose: prioritize, route, and deliver hardware interrupts to CPUs.

**Q2: What is an IRQ domain and why is it needed?**
A: IRQ domains map hardware interrupt numbers (hwirq) to Linux virtual IRQ numbers. They're needed because different interrupt controllers have overlapping hwirq numbering (both GIC and GPIO controller might have IRQ #5). The domain framework provides a per-controller namespace, allowing hierarchical interrupt routing and clean separation between controllers.

**Q3: What are softirqs and when are they initialized?**
A: Softirqs are deferred interrupt processing — they run after hardware IRQ handlers complete, still in interrupt context but with interrupts enabled. `softirq_init()` registers HI_SOFTIRQ and TASKLET_SOFTIRQ during boot. Other softirqs (NET, TIMER, SCHED, RCU) are registered by their respective subsystems. If softirqs take too long, processing moves to `ksoftirqd` kernel threads.

---

## Summary

- `trap_init()` sets up exception vectors, `init_IRQ()` initializes interrupt controllers
- ARM uses GIC (Distributor + CPU interfaces), x86 uses APIC (I/O APIC + Local APIC)
- IRQ domains map hardware IRQ numbers to Linux virtual IRQ numbers per controller
- Softirqs provide deferred interrupt processing — initialized in `softirq_init()`
- Interrupts are enabled on the boot CPU with `local_irq_enable()` after init

---

*Next: [Chapter 24 — Memory Initialization](Chapter_24_Memory_Initialization.md)*
