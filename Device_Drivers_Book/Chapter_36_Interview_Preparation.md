# Chapter 36: Interview Preparation — Comprehensive Question Bank

## Chapter Overview

This final chapter provides a comprehensive collection of interview questions spanning all topics covered in this book. Questions range from basic concepts to advanced debugging scenarios, suitable for positions from junior driver engineer to kernel architect.

---

## 36.1 Fundamentals (Chapters 1-4)

**Q1: What is a device driver and why is it needed?**
A: A device driver is a kernel module or built-in code that acts as a translator between the operating system and hardware. It's needed because: 1) Hardware speaks registers, interrupts, DMA — the OS needs a uniform interface. 2) Isolation — userspace shouldn't access hardware directly. 3) Resource sharing — multiple processes can use the same device.

**Q2: Explain the Linux device model (bus-device-driver).**
A: The model has three core components: `struct bus_type` defines the bus (matching rules, probe mechanism), `struct device` represents a hardware device (resources, DT node, driver data), and `struct device_driver` represents a driver (probe/remove, match tables). The bus matches devices to drivers. When a match is found, `probe()` is called. sysfs exposes the entire hierarchy under `/sys/`.

**Q3: What is the difference between monolithic and microkernel architectures?**
A: Monolithic (Linux): All drivers run in kernel space with full privilege. Fast (no IPC overhead) but one buggy driver can crash the system. Microkernel (QNX): Drivers run as user-space processes. Isolated (driver crash ≠ system crash) but slower (IPC per operation). Linux mitigates with modularity, IOMMU, and KASAN.

**Q4: What are initcall levels and in what order do they run?**
A: `pure_initcall(0)` → `core_initcall(1)` → `postcore_initcall(2)` → `arch_initcall(3)` → `subsys_initcall(4)` → `fs_initcall(5)` → `device_initcall(6)` → `late_initcall(7)`. `module_init` maps to `device_initcall(6)` when built-in. Lower numbers run earlier.

---

## 36.2 Modules and Character Devices (Chapters 5-8)

**Q5: What is the difference between a loadable module and a built-in driver?**
A: Module (.ko): Can be loaded/unloaded at runtime. Uses `module_init`/`module_exit`. Increases flexibility but slight overhead. Built-in: Linked into vmlinux. Always present. Uses `module_init` which maps to `device_initcall`. Cannot be removed. Use for critical boot drivers.

**Q6: Explain major and minor device numbers.**
A: Major number identifies the driver (e.g., 4 for tty). Minor number identifies the specific device instance (ttyS0, ttyS1). Together stored as `dev_t` (12-bit major, 20-bit minor). `alloc_chrdev_region()` allocates them dynamically. udev uses them to create `/dev/` entries.

**Q7: Walk through what happens when userspace calls `read()` on a character device.**
A: 1) `read()` syscall → `ksys_read()`. 2) Get `struct file` from fd. 3) `vfs_read()` checks permissions. 4) Calls `file->f_op->read()`. 5) Driver's read function runs. 6) `copy_to_user()` transfers data. 7) Return byte count to userspace. If no data available, driver may `wait_event_interruptible()`.

**Q8: How do you implement ioctl properly?**
A: 1) Define commands using `_IO/_IOR/_IOW/_IOWR` macros with unique magic number. 2) Implement `unlocked_ioctl` in `file_operations`. 3) Switch on command, validate all arguments. 4) `copy_from_user()`/`copy_to_user()` for data transfer. 5) Return 0 on success, `-ENOTTY` for unknown commands, `-EINVAL` for bad arguments, `-EFAULT` for copy failure.

---

## 36.3 Block and Network Drivers (Chapters 9-10)

**Q9: What is blk-mq and why did Linux switch to it?**
A: blk-mq (multi-queue) replaced the single-queue block layer. It provides per-CPU software queues mapped to hardware dispatch queues. Benefits: eliminates single-lock contention, scales with NVMe (64K queues), reduces latency. The old single-queue layer was a bottleneck with modern SSDs.

**Q10: Explain NAPI in network drivers.**
A: NAPI (New API) is a hybrid interrupt/polling mechanism. 1) First packet triggers IRQ. 2) IRQ handler disables further interrupts, schedules NAPI poll. 3) Poll function processes packets in a loop (up to budget). 4) If all processed (done < budget), re-enable IRQs. Benefit: at high packet rates, avoids per-packet interrupt overhead.

---

## 36.4 Platform Drivers and Device Tree (Chapters 11-13)

**Q11: What is the difference between platform devices and PCI devices?**
A: Platform devices are non-enumerable (SoC peripherals described by DT or ACPI). PCI devices are enumerable — the PCI bus discovers them by scanning. Platform: needs DT/ACPI to know they exist. PCI: hardware scan reveals vendor:device ID. Both use the same bus-device-driver model.

**Q12: How does Device Tree work end-to-end?**
A: 1) Developer writes `.dts` (source). 2) `dtc` compiles to `.dtb` (binary). 3) Bootloader passes DTB address to kernel. 4) Kernel unflattens to `device_node` tree. 5) `of_platform_populate()` creates `platform_device` for each compatible node. 6) Resources (reg → IORESOURCE_MEM, interrupts → IORESOURCE_IRQ) extracted from DT. 7) Bus `match()` compares device compatible with driver `of_match_table`. 8) On match, `probe()` called.

**Q13: What is deferred probe and when does it happen?**
A: When `probe()` returns `-EPROBE_DEFER`, the device is added to a deferred list and retried later. This happens when a dependency (clock, regulator, GPIO) isn't available yet because its provider driver hasn't probed. The deferred list is retried whenever any new driver registers or any probe succeeds.

---

## 36.5 Hardware Access (Chapter 14)

**Q14: Why must you use `readl()`/`writel()` instead of direct pointer dereference for MMIO?**
A: 1) They include memory barriers ensuring correct ordering on weakly-ordered architectures (ARM). 2) They prevent compiler reordering. 3) They handle endian conversion. 4) Sparse annotation checking (`__iomem`) catches direct dereferences. Direct dereference may work on x86 but is incorrect and non-portable.

**Q15: What is regmap and when should you use it?**
A: Regmap provides a unified register access API across MMIO, I2C, SPI. Use it when: 1) Register access pattern is uniform. 2) You need caching (I2C/SPI are slow). 3) Same driver supports multiple bus types. 4) You want debugfs register dump for free. Initialized via `devm_regmap_init_mmio/i2c/spi()`.

---

## 36.6 Interrupts (Chapter 15)

**Q16: Explain the difference between top half and bottom half.**
A: Top half (hard IRQ handler): runs in interrupt context, interrupts disabled on that line, must be fast. Should only: acknowledge hardware, save minimal state, schedule bottom half. Bottom half (threaded IRQ / workqueue / tasklet): runs later in process or softirq context. Can do heavy processing, can sleep (threaded IRQ, workqueue). The split avoids long IRQ-disabled periods.

**Q17: What is a threaded IRQ and when should you use it?**
A: Threaded IRQ has a hard IRQ handler (fast, ACK only) and a threaded handler (kernel thread, can sleep). Use `devm_request_threaded_irq()` with `IRQF_ONESHOT`. Use when: 1) Handler needs to do I2C/SPI transfers (sleep). 2) Handler is complex (>50µs). 3) You need mutex protection. The `IRQF_ONESHOT` flag keeps the IRQ masked until the thread completes.

**Q18: How do you debug an interrupt that's not firing?**
A: 1) `cat /proc/interrupts` — check if IRQ is registered and count is 0. 2) Verify DT interrupt specifier. 3) Check GIC configuration (SPI number, trigger type). 4) Read hardware interrupt status register. 5) Enable IRQ trace events. 6) Check if IRQ is masked. 7) Verify physical wiring.

---

## 36.7 Concurrency (Chapter 16)

**Q19: When do you use a spinlock vs a mutex?**
A: Spinlock: short critical sections, usable in any context (including IRQ). Busy-waits. Mutex: longer critical sections, process context only (can sleep). Blocks and schedules. Rule: if you might sleep inside the lock (I2C transfer, kmalloc with GFP_KERNEL), use mutex. If in IRQ context or very short section, use spinlock.

**Q20: What is an ABBA deadlock?**
A: Thread 1 holds lock A, waits for lock B. Thread 2 holds lock B, waits for lock A. Neither can proceed. Fix: always acquire locks in the same order. Lockdep detects this at runtime by tracking lock ordering.

---

## 36.8 DMA (Chapter 18)

**Q21: Explain the difference between coherent and streaming DMA.**
A: Coherent (`dma_alloc_coherent`): Non-cacheable mapping. CPU and device always see consistent data. No manual sync needed. Used for descriptors, control structures. Streaming (`dma_map_single`): Cacheable, requires explicit `dma_sync_*` calls. Better performance for bulk data because cache is used. Used for data buffers.

**Q22: What happens if you don't call `dma_unmap_single()` after a streaming DMA transfer?**
A: IOMMU mapping leak (IOVA exhaustion), potential cache coherency issues (stale cache lines), DMA API debug warnings. On x86 without IOMMU it might appear to work, masking a real bug that manifests on ARM or with IOMMU enabled.

**Q23: What is the SWIOTLB bounce buffer?**
A: When a device can only DMA to a limited address range (e.g., 32-bit) but the buffer is in high memory, SWIOTLB copies data to a low-memory bounce buffer, DMAs from there, then copies back. Transparent to the driver. Performance cost due to extra copy.

---

## 36.9 Power Management (Chapter 20)

**Q24: Explain the difference between system PM and runtime PM.**
A: System PM: entire system suspends/resumes (triggered by `echo mem > /sys/power/state`). All devices suspend in leaf-to-root order. Runtime PM: individual devices suspend when idle, independent of system state. Controlled by `pm_runtime_get/put`. A device can be runtime-suspended while the system is fully awake.

**Q25: What is autosuspend and why is it useful?**
A: Autosuspend delays the runtime suspend by a configurable time after the last `pm_runtime_put_autosuspend()`. Avoids rapid suspend/resume cycles for bursty I/O. Example: delay 50ms — if another I/O arrives within 50ms, no suspend/resume overhead.

---

## 36.10 Debugging Scenarios (Chapter 25)

**Q26: Your driver crashes with "Unable to handle kernel NULL pointer dereference at 0x48". How do you debug?**
A: 1) The offset 0x48 suggests a struct member access on a NULL pointer. 2) Look at `pc` in the oops — use `addr2line` or `objdump` to find the exact line. 3) Identify which struct has a member at offset 0x48 using `pahole` or manual calculation. 4) Trace back to where that pointer should have been set (likely a failed `devm_kzalloc` or unset `driver_data`).

**Q27: How do you debug a driver that causes the system to hang?**
A: 1) Enable serial console (hardcoded, not USB). 2) Use SysRq keys (Alt+SysRq+T for task dump). 3) Check if hard lockup (NMI watchdog) or soft lockup. 4) Enable ftrace before reproducing. 5) If spin_lock deadlock, lockdep would have warned (if enabled). 6) Use KGDB over serial for interactive debugging.

**Q28: Driver works on boot but fails after suspend/resume. How do you debug?**
A: 1) Check resume callback: is it restoring all register state? 2) Check clocks: are they re-enabled? 3) Check reset: does hardware need re-initialization? 4) Compare register dumps before suspend and after resume. 5) Check if runtime PM state is correct after system resume. 6) Verify DMA mappings are still valid.

---

## 36.11 Architecture and Design (Chapters 31, 33)

**Q29: Draw the complete Linux driver stack from application to hardware.**
A: Application → glibc → syscall → VFS → subsystem framework (char/block/net/input/V4L2) → bus layer (platform/PCI/I2C/SPI) → driver (probe/file_ops/IRQ handler) → HAL (ioremap/readl/DMA) → hardware. See Chapter 31 for detailed diagram.

**Q30: How would you port a Linux driver to QNX?**
A: 1) Replace `probe` with resource manager initialization (`resmgr_attach`). 2) Replace `readl/writel` with `mmap_device_io/in32/out32`. 3) Replace `request_irq` with `InterruptAttach`. 4) Replace file_operations with `resmgr_io_funcs`. 5) Replace kernel threading with POSIX threads. 6) No `devm_*` — manual cleanup needed. 7) Test extensively — different memory model.

---

## 36.12 Advanced Topics

**Q31: How does IOMMU improve security?**
A: Without IOMMU, any bus-master device can DMA to any physical address — a malicious PCIe device could read kernel memory. IOMMU interposes page tables: devices can only access explicitly mapped pages. This is critical for: PCIe hotplug security, VM device passthrough (VFIO), Thunderbolt DMA protection.

**Q32: What is `container_of()` and why is it fundamental in the kernel?**
A: `container_of(ptr, type, member)` recovers a pointer to the containing structure from a pointer to one of its members. It's fundamental because the kernel embeds structures (e.g., `struct device` inside `struct platform_device`). Given a `struct device *dev`, you can get the `platform_device` with `container_of(dev, struct platform_device, dev)` or the shorthand `to_platform_device(dev)`.

**Q33: Explain the devm_* resource management system.**
A: When you call `devm_kzalloc()`, `devm_ioremap()`, etc., the resource is registered in `dev->devres_head` (a linked list). When the driver unbinds (`remove()` or probe failure), all devm resources are freed in reverse order automatically. This prevents resource leaks, especially in error paths. Implementation: `drivers/base/devres.c`.

**Q34: How would you write a production-quality probe function?**
A: 1) Allocate private data with `devm_kzalloc`. 2) Get resources: `devm_platform_ioremap_resource`, `devm_clk_get`, `platform_get_irq`. 3) Enable clocks + reset deassert. 4) Read DT properties. 5) Initialize hardware. 6) Request IRQ (threaded if complex). 7) Register with subsystem. 8) Enable runtime PM. 9) Use `dev_err_probe()` for all error returns. 10) All failures handled by devm — no goto chains needed.

---

## 36.13 Quick-Fire Round (One-Line Answers)

| Question | Answer |
|----------|--------|
| What does `MODULE_LICENSE("GPL")` do? | Declares license, enables GPL-only symbol access |
| What is `__iomem`? | Sparse annotation for memory-mapped I/O pointers |
| What does `IRQF_SHARED` mean? | Multiple devices share the same IRQ line |
| What is `GFP_ATOMIC`? | Memory allocation flag for atomic/IRQ context (no sleep) |
| What does `unlikely()` do? | Branch prediction hint — tells CPU this path is rare |
| What is `sysfs_emit()`? | Bounds-safe print to sysfs buffer (replaces sprintf) |
| What is a `dev_t`? | Device number type (32-bit: 12-bit major + 20-bit minor) |
| What does `module_platform_driver()` expand to? | `module_init` + `module_exit` calling `platform_driver_[un]register` |
| What is `FIELD_PREP()`? | Shift a value into a register field position with mask |
| What is `PTR_ERR()`? | Extract error code from an error-encoded pointer |

---

## 36.14 Coding Exercise Questions

**Exercise 1:** Write a minimal platform driver that reads a "clock-frequency" property from Device Tree and prints it.

**Exercise 2:** Write a character driver with `read()` that blocks until data is available (using wait queues), and an IRQ handler that wakes it.

**Exercise 3:** Given a register at offset 0x10 with bits [7:4] as a mode field, write code using `FIELD_PREP()`/`FIELD_GET()` to set mode to 5 and read it back.

**Exercise 4:** Write error handling for a probe function that gets clock, ioremap, and IRQ — both with goto chains and with `devm_*` (show both).

---

## Summary

This 36-chapter book has covered the entire landscape of Linux device driver development:

- **Foundations** (Ch 1-4): What drivers are, kernel architecture, device model
- **Driver Types** (Ch 5-12): Char, block, network, platform, bus architecture
- **Hardware Interaction** (Ch 13-16): Device Tree, MMIO, interrupts, synchronization
- **Memory & Power** (Ch 17-20): Allocation, DMA, IOMMU, power management
- **Interfaces** (Ch 21-24): Userspace communication, subsystems, data structures, APIs
- **Quality** (Ch 25-28): Debugging, testing, security, performance
- **Reference** (Ch 29-32): Source locations, flow diagrams, architecture, glossary
- **Professional** (Ch 33-36): OS comparison, embedded development, references, interviews

The key to mastering driver development: **read real kernel code**, **write drivers for real hardware**, and **understand the data structures and their relationships**.

---

*Back to: [Master Index](Master_Index.md)*
