# Chapter 30: Flow Diagrams — Critical Driver Paths

## Chapter Overview

This chapter provides detailed flow diagrams for the most important driver operations: loading, binding, I/O request processing, interrupt handling, and DMA transfer. These are the flows you will trace during debugging and optimization.

---

## 30.1 Driver Loading Flow

```
insmod my_driver.ko
    │
    ▼
sys_init_module()                    [kernel/module/main.c]
    │
    ├── Load ELF sections (.text, .data, .init, .bss)
    ├── Resolve symbols (EXPORT_SYMBOL references)
    ├── Apply relocations
    │
    ▼
do_init_module()
    │
    ▼
module_init → my_driver_init()       [driver code]
    │
    ▼
platform_driver_register(&my_driver)
    │
    ├── driver_register(&my_driver.driver)       [drivers/base/driver.c]
    │       │
    │       ├── bus_add_driver(drv)               [drivers/base/bus.c]
    │       │       │
    │       │       ├── Add to bus's driver list
    │       │       └── driver_attach(drv)
    │       │               │
    │       │               ▼ (for each unbound device on bus)
    │       │           __driver_attach(dev, drv)
    │       │               │
    │       │               ├── bus->match(dev, drv)     ← DT compatible check
    │       │               │       │
    │       │               │       ├── Match? → continue
    │       │               │       └── No match? → skip
    │       │               │
    │       │               ▼ (match found)
    │       │           driver_probe_device(drv, dev)
    │       │               │
    │       │               ▼
    │       │           really_probe(dev, drv)    [drivers/base/dd.c]
    │       │               │
    │       │               ├── dev->driver = drv
    │       │               ├── pinctrl_bind_pins(dev)
    │       │               ├── dma_configure(dev)
    │       │               │
    │       │               ▼
    │       │           drv->probe(dev)  →  my_probe(pdev)
    │       │               │
    │       │               ├── 0 (success) → device bound
    │       │               ├── -EPROBE_DEFER → add to deferred list
    │       │               └── error → dev->driver = NULL
    │       │
    │       └── kobject_uevent → udev notification
    │
    └── Return 0 (module loaded)
```

---

## 30.2 Device-Driver Binding Flow (Device Tree)

```
Bootloader passes DTB to kernel
    │
    ▼
unflatten_device_tree()              [drivers/of/fdt.c]
    │ Parse flat DTB → tree of device_node
    ▼
of_platform_default_populate()       [drivers/of/platform.c]
    │
    ▼ (for each DT node with "compatible")
of_platform_bus_create()
    │
    ▼
of_platform_device_create_pdata()
    │
    ├── platform_device_alloc(name, id)
    ├── dev->dev.of_node = np             ← Link to DT node
    ├── of_address_to_resource(np)        ← reg → IORESOURCE_MEM
    ├── of_irq_to_resource_table(np)      ← interrupts → IORESOURCE_IRQ
    │
    ▼
platform_device_add(pdev)
    │
    ├── device_add(&pdev->dev)
    │       │
    │       ├── bus_add_device(dev)        ← Add to platform bus device list
    │       └── bus_probe_device(dev)
    │               │
    │               ▼
    │           device_initial_probe(dev)
    │               │
    │               ▼ (try all registered drivers)
    │           __device_attach(dev)
    │               │
    │               ├── bus->match(dev, drv)  ← of_match_device()
    │               │       │
    │               │       ▼
    │               │   Compare dev->of_node->compatible
    │               │   with drv->of_match_table entries
    │               │
    │               └── If match → really_probe() → probe()
    │
    └── kobject_uevent(&pdev->dev.kobj, KOBJ_ADD)
```

---

## 30.3 I/O Request Flow (Character Device Read)

```
User: read(fd, buf, 4096)
    │
    ▼
SYSCALL_DEFINE3(read, ...)            [fs/read_write.c]
    │
    ▼
ksys_read(fd, buf, count)
    │
    ├── fdget_pos(fd) → struct fd
    │       └── fd.file → struct file
    │               └── file->f_op → struct file_operations
    │
    ▼
vfs_read(file, buf, count, &pos)     [fs/read_write.c]
    │
    ├── Check file mode (FMODE_READ)
    ├── rw_verify_area(READ, file, pos, count)
    │
    ▼
file->f_op->read(file, buf, count, pos)
    │
    ▼
my_read(file, buf, count, ppos)      [driver code]
    │
    ├── priv = file->private_data    ← Set in my_open()
    │
    ├── wait_event_interruptible(priv->wq, data_available)
    │       │
    │       └── (blocked until IRQ handler wakes us)
    │
    ├── copy_to_user(buf, priv->kbuf, len)
    │       │
    │       └── Returns 0 on success, >0 on partial fault
    │
    └── return len (bytes read)
         │
         ▼
    Returns to userspace → read() returns len
```

---

## 30.4 Interrupt Handling Flow

```
Hardware asserts IRQ line
    │
    ▼
GIC (Generic Interrupt Controller)
    │
    ├── Prioritize, route to target CPU
    ├── Signal CPU via IRQ exception
    │
    ▼
CPU takes IRQ exception
    │
    ▼
el1_irq / do_IRQ                     [arch/arm64/kernel/entry.S]
    │
    ▼
generic_handle_domain_irq()           [kernel/irq/irqdesc.c]
    │
    ├── Find irq_desc for hardware IRQ number
    │
    ▼
handle_fasteoi_irq()                  [kernel/irq/chip.c]
    │
    ▼
handle_irq_event()
    │
    ▼
handle_irq_event_percpu()
    │
    ▼
__handle_irq_event_percpu()
    │
    ▼ (for each registered action)
action->handler(irq, action->dev_id)
    │
    ▼
my_hard_irq(irq, data)               [driver code]
    │
    ├── readl(REG_STATUS) → check if our interrupt
    ├── writel(status, REG_CLEAR) → acknowledge
    │
    ├── return IRQ_HANDLED            → done
    │   OR
    ├── return IRQ_WAKE_THREAD        → schedule threaded handler
    │       │
    │       ▼
    │   irq_thread()                  [kernel/irq/manage.c]
    │       │
    │       ▼
    │   action->thread_fn(irq, data)
    │       │
    │       ▼
    │   my_thread_fn(irq, data)       [driver code]
    │       │
    │       ├── Process data (can sleep)
    │       ├── wake_up(&priv->wq)    → wake blocked read()
    │       └── return IRQ_HANDLED
    │
    └── return IRQ_NONE → not our interrupt
```

---

## 30.5 DMA Transfer Flow

```
Driver initiates DMA transfer
    │
    ▼
dma_map_single(dev, buf, len, DMA_TO_DEVICE)   [kernel/dma/mapping.c]
    │
    ├── Without IOMMU:
    │   └── dma_addr = virt_to_phys(buf)    (or SWIOTLB bounce)
    │
    ├── With IOMMU:
    │   ├── Allocate IOVA
    │   ├── Map IOVA → physical in IOMMU page table
    │   └── dma_addr = IOVA
    │
    ├── Cache flush (ARM: clean/invalidate)
    │
    └── Return dma_addr_t
    │
    ▼
Program hardware DMA registers
    writel(dma_addr, REG_DMA_SRC);
    writel(len, REG_DMA_LEN);
    writel(DMA_START, REG_DMA_CTRL);
    │
    ▼
Hardware performs DMA (bus master)
    │
    ├── Device reads from dma_addr via bus
    ├── [IOMMU translates IOVA → physical] (if present)
    ├── Memory controller serves data
    │
    ▼
DMA complete → hardware asserts IRQ
    │
    ▼
IRQ handler:
    dma_unmap_single(dev, dma_addr, len, DMA_TO_DEVICE);
    │
    ├── Invalidate IOMMU mapping (if used)
    ├── Cache invalidate (for DMA_FROM_DEVICE)
    │
    └── Buffer safe for CPU access
```

---

## 30.6 Power Management Flow (System Suspend)

```
echo mem > /sys/power/state
    │
    ▼
pm_suspend(PM_SUSPEND_MEM)
    │
    ▼
suspend_prepare()
    ├── Freeze user processes
    ├── Freeze kernel threads
    │
    ▼
suspend_devices_and_enter()
    │
    ▼
dpm_suspend()  (for each device, leaf → root)
    │
    ▼
device_suspend(dev)
    │
    ├── dev->pm_domain->ops->suspend()    (if PM domain)
    ├── dev->type->pm->suspend()          (if type PM)
    ├── dev->class->pm->suspend()         (if class PM)
    ├── dev->bus->pm->suspend()           (if bus PM)
    └── dev->driver->pm->suspend()        ← Driver suspend callback
           │
           ▼
        my_suspend(dev)
           ├── Disable hardware
           ├── Save register state
           ├── Release clocks
           └── return 0
    │
    ▼
suspend_enter()
    └── CPU enters low-power state (WFI)

... (wake event) ...

dpm_resume()  (for each device, root → leaf)
    │
    └── dev->driver->pm->resume()  → my_resume(dev)
           ├── Restore clocks
           ├── Restore register state
           └── Re-enable hardware
```

---

## Interview Questions

**Q1: Trace the path from `insmod` to `probe()` being called.**
A: `insmod` → `sys_init_module()` → load ELF → `module_init()` → `platform_driver_register()` → `driver_register()` → `bus_add_driver()` → `driver_attach()` → for each device: `bus->match()` → if match: `really_probe()` → `drv->probe()`.

**Q2: What happens between a hardware interrupt and your handler running?**
A: 1) GIC prioritizes and routes IRQ to CPU. 2) CPU takes exception, saves state. 3) Kernel IRQ entry code runs. 4) `generic_handle_domain_irq()` looks up `irq_desc`. 5) Flow handler (`handle_fasteoi_irq`) runs. 6) Each registered `action->handler()` is called. 7) If `IRQ_WAKE_THREAD`, the kernel IRQ thread invokes `thread_fn()`.

**Q3: How does DMA work with and without IOMMU?**
A: Without IOMMU: `dma_map_single()` returns physical address directly (or bounces through SWIOTLB). Device accesses physical memory directly. With IOMMU: an IOVA is allocated, mapped in IOMMU page tables to the physical address, and the IOVA is returned. The IOMMU translates device bus addresses to physical addresses.

---

*Next: [Chapter 31 — Architecture Diagrams](Chapter_31_Architecture_Diagrams.md)*
