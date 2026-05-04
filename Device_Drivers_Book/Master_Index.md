# Linux Device Drivers — Master Index

## Complete Study Guide: 36 Chapters

**Scope:** Linux Kernel 5.x–6.x | Architectures: x86_64, ARM64, RISC-V
**Audience:** Embedded engineers, kernel developers, driver architects, interview candidates

---

## Part I: Foundations (Chapters 1–4)

| # | Chapter | Key Topics |
|---|---------|------------|
| 1 | [Foundations of Device Drivers](Chapter_01_Foundations_of_Device_Drivers.md) | What is a driver, user/kernel boundary, device types, hardware abstraction |
| 2 | [History and Evolution](Chapter_02_History_and_Evolution.md) | Unix V6, BSD, Linux evolution, DT introduction, devm_*, Rust drivers |
| 3 | [Kernel Architecture](Chapter_03_Kernel_Architecture.md) | Subsystems, VFS→driver path, modules vs built-in, initcall levels, execution contexts |
| 4 | [Linux Device Model](Chapter_04_Linux_Device_Model.md) | bus/device/driver/class, kobject, sysfs, device-driver binding, deferred probe |

---

## Part II: Driver Types & Modules (Chapters 5–8)

| # | Chapter | Key Topics |
|---|---------|------------|
| 5 | [Types of Device Drivers](Chapter_05_Types_of_Device_Drivers.md) | Char, block, network, platform, USB, PCI, I2C, SPI with code examples |
| 6 | [Kernel Modules](Chapter_06_Kernel_Modules.md) | .ko anatomy, lifecycle, dependencies, Kbuild, module_param, auto-loading |
| 7 | [Writing a Basic Driver](Chapter_07_Writing_Basic_Driver.md) | Skeleton, headers, probe pattern, error handling, logging |
| 8 | [Character Device Drivers](Chapter_08_Character_Device_Drivers.md) | Major/minor, cdev, file_operations, ioctl, complete multi-device example |

---

## Part III: Block, Network & Bus (Chapters 9–12)

| # | Chapter | Key Topics |
|---|---------|------------|
| 9 | [Block Device Drivers](Chapter_09_Block_Device_Drivers.md) | blk-mq, gendisk, request queue, RAM disk example |
| 10 | [Network Device Drivers](Chapter_10_Network_Device_Drivers.md) | net_device_ops, NAPI, TX/RX paths, sk_buff |
| 11 | [Bus Architecture](Chapter_11_Bus_Architecture.md) | bus_type, enumerable vs non-enumerable, match mechanisms |
| 12 | [Platform Device Drivers](Chapter_12_Platform_Device_Drivers.md) | DT-based probe, resource APIs, dev_err_probe, production probe pattern |

---

## Part IV: Device Tree, Hardware & Interrupts (Chapters 13–16)

| # | Chapter | Key Topics |
|---|---------|------------|
| 13 | [Device Tree](Chapter_13_Device_Tree.md) | DTS syntax, nodes/properties, overlays, binding documents, DT→driver flow |
| 14 | [Hardware Register Access](Chapter_14_Hardware_Register_Access.md) | MMIO vs PIO, ioremap, readl/writel, regmap, FIELD_PREP/FIELD_GET |
| 15 | [Interrupt Handling](Chapter_15_Interrupt_Handling.md) | GIC, request_irq, top/bottom half, threaded IRQs, workqueue |
| 16 | [Concurrency and Synchronization](Chapter_16_Concurrency_Synchronization.md) | Spinlocks, mutexes, atomic, completions, lockdep, deadlock patterns |

---

## Part V: Memory, DMA & Power (Chapters 17–20)

| # | Chapter | Key Topics |
|---|---------|------------|
| 17 | [Memory Management for Drivers](Chapter_17_Memory_Management.md) | kmalloc, vmalloc, GFP flags, memory pools, scatter-gather |
| 18 | [DMA](Chapter_18_DMA.md) | Coherent vs streaming, dma_alloc_coherent, dma_map_single, SG-DMA |
| 19 | [IOMMU](Chapter_19_IOMMU.md) | IOMMU architecture, IOVA, DMA remapping, IOMMU groups, VFIO |
| 20 | [Power Management](Chapter_20_Power_Management.md) | System PM, runtime PM, autosuspend, wakeup sources, PM callbacks |

---

## Part VI: Interfaces & APIs (Chapters 21–24)

| # | Chapter | Key Topics |
|---|---------|------------|
| 21 | [Userspace Communication](Chapter_21_Userspace_Communication.md) | Device files, ioctl, sysfs, procfs, netlink, debugfs |
| 22 | [Kernel I/O Subsystems](Chapter_22_IO_Subsystems.md) | Input subsystem, V4L2, DRM/KMS, storage/SCSI |
| 23 | [Important Kernel Data Structures](Chapter_23_Data_Structures.md) | struct device/driver/bus_type/platform_device/platform_driver, container_of |
| 24 | [Kernel Driver APIs](Chapter_24_Driver_APIs.md) | Registration, MMIO, clock, reset, GPIO, IRQ, DMA, PM, sync, DT, logging APIs |

---

## Part VII: Quality & Optimization (Chapters 25–28)

| # | Chapter | Key Topics |
|---|---------|------------|
| 25 | [Debugging Techniques](Chapter_25_Debugging.md) | printk, dynamic debug, ftrace, debugfs, devcoredump, KASAN |
| 26 | [Driver Testing](Chapter_26_Testing.md) | Static analysis, KUnit, QEMU, sysfs testing, stress testing, fault injection |
| 27 | [Security](Chapter_27_Security.md) | Vulnerabilities, copy_to/from_user, capabilities, IOMMU, secure design |
| 28 | [Performance Optimization](Chapter_28_Performance.md) | Latency, throughput, NAPI, cache optimization, profiling with perf |

---

## Part VIII: Reference Material (Chapters 29–32)

| # | Chapter | Key Topics |
|---|---------|------------|
| 29 | [Source Code Locations](Chapter_29_Source_Code.md) | Kernel source tree map, drivers/ directory, key files to study |
| 30 | [Flow Diagrams](Chapter_30_Flow_Diagrams.md) | Driver loading, DT binding, I/O request, IRQ handling, DMA transfer, PM flows |
| 31 | [Architecture Diagrams](Chapter_31_Architecture_Diagrams.md) | Device model, driver stack, bus-device-driver, DMA path, IRQ pipeline |
| 32 | [Glossary](Chapter_32_Glossary.md) | Alphabetical definitions of all important terms |

---

## Part IX: Professional Development (Chapters 33–36)

| # | Chapter | Key Topics |
|---|---------|------------|
| 33 | [OS Comparison](Chapter_33_OS_Comparison.md) | Linux vs Windows/macOS/QNX/Android driver models |
| 34 | [Embedded & Automotive Development](Chapter_34_Embedded_Development.md) | BSP, SoC drivers, CAN bus, automotive requirements, cross-compilation |
| 35 | [References and Documentation](Chapter_35_References.md) | Books, kernel docs, online resources, tools, standards |
| 36 | [Interview Preparation](Chapter_36_Interview_Preparation.md) | 34+ questions with answers, debugging scenarios, coding exercises |

---

## Suggested Reading Paths

### Path 1: Complete Beginner (2-3 weeks)
Ch 1 → 3 → 4 → 6 → 7 → 8 → 12 → 13 → 14 → 15 → 25 → 32

### Path 2: Experienced Developer, New to Kernel (1-2 weeks)
Ch 4 → 12 → 13 → 14 → 15 → 16 → 18 → 20 → 23 → 24 → 25

### Path 3: Interview Preparation (3-5 days)
Ch 4 → 12 → 15 → 18 → 20 → 23 → 30 → 31 → 36

### Path 4: Automotive/Embedded Focus
Ch 4 → 12 → 13 → 15 → 18 → 19 → 20 → 27 → 28 → 34

### Path 5: Deep Reference (ongoing)
Ch 24 → 29 → 30 → 31 → 32 → 35

---

## Quick API Lookup

| Need to... | Go to |
|-----------|-------|
| Register a driver | Ch 24 §24.1 |
| Access hardware registers | Ch 14, Ch 24 §24.2 |
| Handle interrupts | Ch 15, Ch 24 §24.3 |
| Allocate memory | Ch 17, Ch 24 §24.4 |
| Set up DMA | Ch 18, Ch 24 §24.5 |
| Manage power | Ch 20, Ch 24 §24.6 |
| Use Device Tree | Ch 13, Ch 24 §24.8 |
| Debug a problem | Ch 25 |
| Understand data structures | Ch 23 |
| Trace a flow | Ch 30 |
| Find source code | Ch 29 |
| Look up a term | Ch 32 |

---

*36 chapters | ~5000+ lines of content | Complete Linux Device Driver reference*
