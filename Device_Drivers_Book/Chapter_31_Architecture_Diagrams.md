# Chapter 31: Architecture Diagrams

## Chapter Overview

Visual diagrams of the Linux driver architecture at various levels of abstraction, from the full software stack to individual subsystem internals.

---

## 31.1 Linux Device Model — Complete Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         sysfs (/sys/)                           │
│  /sys/bus/  /sys/class/  /sys/devices/  /sys/module/            │
└────────┬──────────┬────────────┬──────────────┬─────────────────┘
         │          │            │              │
    ┌────▼────┐ ┌───▼───┐  ┌────▼─────┐  ┌────▼────┐
    │  Buses  │ │Classes│  │ Devices  │  │ Modules │
    └────┬────┘ └───┬───┘  └────┬─────┘  └─────────┘
         │          │           │
    ┌────▼──────────▼───────────▼────────────────────┐
    │              kobject / kset / ktype             │
    │         (Reference counting, sysfs entries)     │
    └────────────────────┬───────────────────────────┘
                         │
    ┌────────────────────▼───────────────────────────┐
    │                 kernfs / VFS                    │
    └────────────────────────────────────────────────┘
```

---

## 31.2 Driver Stack — Full Software Layers

```
┌───────────────────────────────────────────────────────┐
│                    Application                        │
│            (open, read, write, ioctl, mmap)           │
├───────────────────────────────────────────────────────┤
│                    C Library (glibc)                   │
│                    (syscall wrappers)                  │
├───────────────────────────────────────────────────────┤
│               System Call Interface                    │
│           (SYSCALL_DEFINE, sys_read, etc.)             │
├───────────────────────────────────────────────────────┤
│             Virtual File System (VFS)                  │
│     (struct file, struct inode, file_operations)       │
├────────┬──────────┬───────────┬───────────────────────┤
│  Char  │  Block   │  Network  │  Subsystem Frameworks │
│  Layer │  Layer   │  Layer    │  (Input, V4L2, DRM,   │
│        │ (blk-mq) │ (net_dev) │   IIO, Sound)         │
├────────┴──────────┴───────────┴───────────────────────┤
│              Bus Abstraction Layer                      │
│  (platform, PCI, USB, I2C, SPI, SDIO)                 │
├───────────────────────────────────────────────────────┤
│            Individual Device Drivers                   │
│      (probe, remove, suspend, resume, IRQ)             │
├───────────────────────────────────────────────────────┤
│          Hardware Abstraction                          │
│   (ioremap, readl/writel, regmap, DMA, clocks)         │
├───────────────────────────────────────────────────────┤
│               Physical Hardware                        │
│    (SoC peripherals, PCIe devices, USB devices)        │
└───────────────────────────────────────────────────────┘
```

---

## 31.3 Bus-Device-Driver Relationship

```
                    struct bus_type
                   (platform_bus_type)
                   ┌──────────────┐
                   │  name: "platform"
                   │  match()     │
                   │  probe()     │
                   └──────┬───────┘
                  ┌───────┴────────┐
          Device List          Driver List
          ┌──────┐             ┌──────┐
          │Dev A │             │Drv X │
          │ .compatible=       │ .of_match_table=
          │  "vendor,foo"      │  "vendor,foo"
          │ .driver ──────────►│              │
          └──────┘             └──────┘
          ┌──────┐             ┌──────┐
          │Dev B │             │Drv Y │
          │ .compatible=       │ .of_match_table=
          │  "vendor,bar"      │  "vendor,bar"
          │ .driver ──────────►│              │
          └──────┘             └──────┘
          ┌──────┐
          │Dev C │  (unbound — no matching driver loaded)
          │ .compatible=
          │  "vendor,baz"
          │ .driver = NULL
          └──────┘

  Match: bus->match() compares Dev.compatible ↔ Drv.of_match_table
  Bind:  bus->probe() → drv->probe(dev)
```

---

## 31.4 DMA Data Path Diagram

```
                CPU                          Device
                 │                             │
                 │     ┌───────────────┐       │
                 ├────►│  CPU Cache    │       │
                 │     │  (L1/L2/LLC)  │       │
                 │     └───────┬───────┘       │
                 │             │               │
                 │     ┌───────▼───────┐       │
                 │     │  System Bus   │       │
                 │     │  (AXI/CCI)    │       │
                 │     └──┬────────┬───┘       │
                 │        │        │           │
          ┌──────▼────┐   │   ┌────▼──────┐    │
          │  Memory   │   │   │  IOMMU    │    │
          │Controller │   │   │(SMMU/VT-d)│    │
          └──────┬────┘   │   └────┬──────┘    │
                 │        │        │           │
          ┌──────▼────────▼────────▼───────────▼──┐
          │              DRAM (Physical Memory)    │
          │                                        │
          │  ┌──────────┐       ┌──────────┐      │
          │  │ Kernel   │       │  DMA     │      │
          │  │ Buffer   │       │  Buffer  │      │
          │  └──────────┘       └──────────┘      │
          └────────────────────────────────────────┘

  Coherent DMA:  CPU cache bypassed (non-cacheable mapping)
  Streaming DMA: Explicit cache flush/invalidate via dma_sync_*()
```

---

## 31.5 IRQ Pipeline Diagram

```
  ┌──────────┐
  │ Hardware │
  │ Device   │ ──IRQ line──►┌──────────────────────┐
  └──────────┘              │  Interrupt Controller │
                            │  (GIC / APIC / PLIC) │
  ┌──────────┐              │                      │
  │ Hardware │ ──IRQ line──►│  ┌───────────────┐   │
  │ Device   │              │  │ Priority      │   │
  └──────────┘              │  │ Routing       │   │
                            │  │ Masking       │   │
  ┌──────────┐              │  └───────┬───────┘   │
  │ Timer    │ ──IRQ line──►│          │           │
  └──────────┘              └──────────┼───────────┘
                                       │
                               ┌───────▼───────┐
                               │   CPU Core    │
                               │  (IRQ exception)
                               └───────┬───────┘
                                       │
                            ┌──────────▼──────────┐
                            │   Linux IRQ Core    │
                            │  (irq_desc, irqchip)│
                            └──────────┬──────────┘
                                       │
                    ┌──────────────────┬┴──────────────────┐
                    │                  │                    │
              ┌─────▼─────┐    ┌──────▼─────┐    ┌────────▼────────┐
              │ Hard IRQ  │    │ Threaded   │    │  Softirq /      │
              │ Handler   │    │ IRQ Handler│    │  Tasklet        │
              │(top half) │    │(can sleep) │    │ (bottom half)   │
              └───────────┘    └────────────┘    └─────────────────┘
```

---

## 31.6 Platform Driver Probe Sequence

```
┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐
│ DT blob │───►│ DT Core  │───►│ Platform │───►│  Bus Match   │
│ (DTB)   │    │ (unflatten)   │ Bus      │    │ (compatible) │
└─────────┘    └──────────┘    └──────────┘    └──────┬───────┘
                                                      │
                                                      ▼
┌────────────────────────────────────────────────────────────────┐
│                    my_probe(pdev)                               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 1. devm_platform_ioremap_resource(pdev, 0) → base_addr  │  │
│  │ 2. devm_clk_get(dev, NULL) → get clock                  │  │
│  │ 3. clk_prepare_enable(clk) → start clock                │  │
│  │ 4. platform_get_irq(pdev, 0) → get IRQ number           │  │
│  │ 5. devm_request_threaded_irq() → install handler        │  │
│  │ 6. of_property_read_*() → read DT properties            │  │
│  │ 7. Initialize hardware registers                         │  │
│  │ 8. Register with subsystem (input, V4L2, etc.)          │  │
│  │ 9. pm_runtime_enable(dev) → enable runtime PM           │  │
│  └──────────────────────────────────────────────────────────┘  │
│  return 0; (success) or negative error code                    │
└────────────────────────────────────────────────────────────────┘
```

---

## 31.7 Deferred Probe Mechanism

```
                Driver A probe()
                    │
                    ├── devm_clk_get() → returns -EPROBE_DEFER
                    │   (clock provider driver not loaded yet)
                    │
                    ▼
              really_probe() sees -EPROBE_DEFER
                    │
                    ├── Add device to deferred_probe_list
                    ├── dev->driver = NULL
                    │
                    ▼
              (later... clock driver loads and probes successfully)
                    │
                    ▼
              driver_deferred_probe_trigger()
                    │
                    ▼
              Retry all devices on deferred_probe_list
                    │
                    ▼
              Driver A probe() called again
                    │
                    ├── devm_clk_get() → success (clock now available)
                    ├── Continue probe...
                    └── return 0 ✓
```

---

## Interview Questions

**Q1: Draw the complete path from userspace `read()` to driver code.**
A: User `read()` → glibc → syscall → `ksys_read()` → `vfs_read()` → `file->f_op->read()` → `my_read()` → `copy_to_user()` → return to userspace. (See Section 30.3 for complete flow.)

**Q2: Explain the deferred probe mechanism with a diagram.**
A: When probe returns `-EPROBE_DEFER`, the device is added to a deferred list. When any new driver registers or any probe succeeds, the deferred list is retried. This handles dependency ordering (e.g., clock provider must probe before clock consumer).

**Q3: How does the IOMMU fit into the DMA data path?**
A: IOMMU sits between the device bus and physical memory. Device DMAs to an IOVA (I/O Virtual Address) → IOMMU translates IOVA to physical address using its page table → memory controller serves the data. Without IOMMU, device DMAs directly to physical addresses.

---

*Next: [Chapter 32 — Glossary](Chapter_32_Glossary.md)*
