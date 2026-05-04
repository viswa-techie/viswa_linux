# Chapter 32: Glossary of Important Terms

## Chapter Overview

Definitions of every important term in Linux device driver development, organized alphabetically. Each entry includes context and cross-references to relevant chapters.

---

## A

**ACPI** — Advanced Configuration and Power Interface. Firmware table standard (primarily x86) that describes hardware topology, power states, and device configuration. Alternative to Device Tree. (Ch. 13)

**Atomic Context** — Execution context where sleeping is not allowed: interrupt handlers, spinlock-held code, softirqs, tasklets. Use `GFP_ATOMIC` for allocations, spinlocks only. (Ch. 16)

---

## B

**BAR** — Base Address Register. PCI configuration space registers that define memory or I/O regions the device uses. Accessed via `pci_resource_start()`. (Ch. 11)

**blk-mq** — Multi-queue block I/O layer. Modern block layer using per-CPU software queues mapped to hardware dispatch queues for parallel I/O. (Ch. 9)

**Bottom Half** — Deferred work from an interrupt handler. Runs outside hard IRQ context. Mechanisms: softirq, tasklet, workqueue, threaded IRQ. (Ch. 15)

**Bus** — Abstraction that connects devices to drivers. Defines match rules, probe mechanism, and power management. Examples: platform, PCI, USB, I2C, SPI. (Ch. 11)

**Bus Mastering** — Capability of a device to initiate DMA transfers on the bus without CPU involvement. Enabled via `pci_set_master()` for PCI. (Ch. 18)

---

## C

**cdev** — Character device structure (`struct cdev`). Links major/minor numbers to `file_operations`. (Ch. 8)

**Coherent DMA** — DMA buffer mapped as non-cacheable. CPU and device always see consistent data. Allocated via `dma_alloc_coherent()`. (Ch. 18)

**Compatible String** — Device Tree property (`compatible = "vendor,device"`) used to match devices to drivers via `of_match_table`. (Ch. 13)

**container_of()** — Macro to recover a containing structure pointer from a pointer to its embedded member. Fundamental pattern in Linux kernel. (Ch. 23)

**copy_to_user() / copy_from_user()** — Safe functions for transferring data between kernel and user address spaces. Handle page faults, validate addresses. (Ch. 21, 27)

---

## D

**debugfs** — Debug filesystem mounted at `/sys/kernel/debug/`. For developer-only information. Not part of stable ABI. (Ch. 25)

**Deferred Probe** — When `probe()` returns `-EPROBE_DEFER`, the device is queued for retry later when dependencies become available. (Ch. 4, 31)

**Device Model** — Linux kernel framework (bus/device/driver/class) for uniform device management, sysfs representation, and power management. (Ch. 4)

**Device Node** — Special file in `/dev/` with major:minor numbers that maps to a character or block driver via VFS. (Ch. 8)

**Device Tree (DT/DTS/DTB)** — Data structure describing hardware topology. DTS (source) compiled to DTB (binary blob) passed to kernel at boot. Standard on ARM, RISC-V. (Ch. 13)

**devm_*** — Device-managed resource APIs. Resources are automatically freed when the driver unbinds. Prevents resource leaks. (Ch. 7, 24)

**DMA** — Direct Memory Access. Hardware transfers data between device and memory without CPU involvement. (Ch. 18)

**DRM/KMS** — Direct Rendering Manager / Kernel Mode Setting. Linux graphics subsystem for display controllers and GPUs. (Ch. 22)

**DT Overlay (DTBO)** — Partial Device Tree that modifies the base DTB at runtime. Used for add-on boards, runtime configuration. (Ch. 13)

---

## E

**EXPORT_SYMBOL** — Makes a kernel function available to loadable modules. `EXPORT_SYMBOL_GPL` limits to GPL-licensed modules. (Ch. 6)

**evdev** — Generic input event interface. Creates `/dev/input/eventN` for each input device. Delivers `struct input_event` to userspace. (Ch. 22)

---

## F

**file_operations** — Structure of function pointers (`open`, `read`, `write`, `unlocked_ioctl`, `release`, `mmap`, `poll`) that define a character device's behavior. (Ch. 8)

**Firmware Node** — Abstract handle (`struct fwnode_handle`) representing device description from either Device Tree or ACPI. Enables firmware-agnostic drivers. (Ch. 13)

**ftrace** — Function tracer. Kernel tracing framework for function timing, call graphs, and event tracing. (Ch. 25)

---

## G

**GFP Flags** — Get Free Page flags controlling memory allocation behavior: `GFP_KERNEL` (can sleep), `GFP_ATOMIC` (cannot sleep), `GFP_DMA` (DMA zone). (Ch. 17)

**GIC** — Generic Interrupt Controller. ARM interrupt controller: GICv2 (32-bit), GICv3/v4 (64-bit, virtualization). (Ch. 15)

**GPIO** — General Purpose Input/Output. Digital pin controllable by software. Managed via `gpiod_*` descriptor API. (Ch. 24)

---

## I

**IIO** — Industrial I/O subsystem. Framework for ADCs, DACs, accelerometers, gyroscopes, and other sensors. (Ch. 22)

**initcall** — Boot-time initialization mechanism. Levels: `pure_initcall` → `core_initcall` → ... → `late_initcall`. `module_init` maps to `device_initcall` when built-in. (Ch. 3)

**IOMMU** — I/O Memory Management Unit. Translates device DMA addresses (IOVAs) to physical addresses. Provides memory isolation and address space management. (Ch. 19)

**IOVA** — I/O Virtual Address. Address in the IOMMU's virtual address space, as seen by the device. (Ch. 19)

**ioctl** — I/O Control. Mechanism for device-specific commands via `unlocked_ioctl` file operation. Commands defined with `_IO/_IOR/_IOW/_IOWR` macros. (Ch. 8, 21)

**ioremap** — Map physical MMIO address to kernel virtual address. Required before `readl()`/`writel()` access. Use `devm_ioremap()` variant. (Ch. 14)

**IRQ** — Interrupt Request. Hardware signal from device to CPU requesting attention. (Ch. 15)

---

## K

**KASAN** — Kernel Address Sanitizer. Runtime memory error detector for use-after-free and out-of-bounds access. (Ch. 25, 27)

**KASLR** — Kernel Address Space Layout Randomization. Randomizes kernel base address to mitigate exploits. (Ch. 27)

**kobject** — Kernel object. Base structure for reference counting and sysfs representation. Embedded in `struct device`, `struct device_driver`. (Ch. 4)

**KUnit** — In-kernel unit test framework. Runs tests in the kernel, ideal for testing driver logic without hardware. (Ch. 26)

---

## L

**lockdep** — Lock dependency validator. Detects potential deadlocks by tracking lock ordering at runtime. (Ch. 16)

---

## M

**Major/Minor Number** — Device identification in `/dev/`. Major identifies the driver; minor identifies the specific device. Stored as `dev_t`. (Ch. 8)

**MMIO** — Memory-Mapped I/O. Hardware registers accessed as memory addresses via `readl()`/`writel()` after `ioremap()`. (Ch. 14)

**Module** — Loadable Kernel Module (.ko). Compiled driver that can be loaded/unloaded at runtime via `insmod`/`rmmod`. (Ch. 6)

**MODULE_DEVICE_TABLE** — Macro that creates an alias table in the module, enabling automatic loading by udev when matching hardware appears. (Ch. 6)

**Mutex** — Sleeping lock for process context. Only one thread can hold it. Use `mutex_lock()`/`mutex_unlock()`. Cannot be used in atomic context. (Ch. 16)

---

## N

**NAPI** — New API. Network driver polling mechanism. IRQ starts polling, packets processed in batches. Reduces IRQ overhead at high packet rates. (Ch. 10, 28)

**Netlink** — Socket-based IPC between kernel and userspace. Generic netlink used for driver events and configuration. (Ch. 21)

---

## O

**of_match_table** — Array of `struct of_device_id` in a driver, listing compatible strings this driver handles. Used by Device Tree matching. (Ch. 12, 13)

---

## P

**PIO** — Programmed I/O. CPU reads/writes each byte manually (vs. DMA where hardware transfers data). Slow for bulk data. (Ch. 14, 18)

**Platform Device** — Non-enumerable device (SoC peripheral) described by Device Tree or board file. `struct platform_device`. (Ch. 12, 23)

**Platform Driver** — Driver that binds to platform devices. `struct platform_driver` with `probe()`/`remove()`. (Ch. 12, 23)

**printk** — Kernel print function. Writes to kernel log buffer (dmesg). Prefer `dev_err()`/`dev_info()` etc. in drivers. (Ch. 25)

**probe()** — Driver callback called when a matching device is found. Initializes hardware, claims resources, registers with subsystems. (Ch. 7, 12)

**procfs** — Process filesystem (`/proc/`). Originally for process info, some legacy kernel/driver info. New drivers should use sysfs. (Ch. 21)

---

## R

**RCU** — Read-Copy-Update. Lock-free synchronization mechanism for read-heavy data. Readers proceed without locks; updates create new copies. (Ch. 16)

**readl() / writel()** — 32-bit MMIO read/write with memory barriers. `readl_relaxed()`/`writel_relaxed()` omit barriers (ARM). (Ch. 14)

**regmap** — Register map abstraction. Unified API for register access across MMIO, I2C, SPI with optional caching and endian handling. (Ch. 14)

**remove()** — Driver callback called when device is unbound. Releases resources (or devm_ handles it automatically). (Ch. 7)

**Runtime PM** — Framework for per-device power management independent of system state. Devices suspended when idle via `pm_runtime_put()`. (Ch. 20)

---

## S

**Scatter-Gather (SG)** — DMA technique to transfer data from/to non-contiguous physical pages without intermediate copying. Uses `struct scatterlist`. (Ch. 18)

**sk_buff** — Socket buffer. Network packet container, central to Linux networking stack. (Ch. 10)

**Softirq** — Fixed set of bottom-half handlers for high-frequency events (NET_RX, NET_TX, TIMER, etc.). Per-CPU, cannot sleep. (Ch. 15)

**Spinlock** — Busy-wait lock suitable for short critical sections in any context (including IRQ). `spin_lock_irqsave()` for IRQ safety. (Ch. 16)

**Streaming DMA** — DMA mapping for temporary, single-use transfers. Cacheable memory with explicit sync. `dma_map_single()`/`dma_unmap_single()`. (Ch. 18)

**SWIOTLB** — Software I/O TLB. Bounce buffer mechanism for devices that cannot DMA to all physical memory addresses. (Ch. 18)

**sysfs** — Kernel object filesystem mounted at `/sys/`. Exposes device model hierarchy. One value per file. (Ch. 4, 21)

---

## T

**Tasklet** — Softirq-based bottom-half mechanism for single-instance deferred work. Being deprecated in favor of threaded IRQs. (Ch. 15)

**Threaded IRQ** — IRQ handler that runs in a kernel thread (process context, can sleep). Requested via `devm_request_threaded_irq()` with `IRQF_ONESHOT`. (Ch. 15)

**Top Half** — The hard IRQ handler. Runs in interrupt context, must be fast. Should only acknowledge and schedule bottom half. (Ch. 15)

---

## U

**udev** — Userspace device manager. Creates `/dev/` entries, loads modules, and sets permissions based on kernel uevents. (Ch. 6, 8)

**uevent** — Kernel event notification to userspace (via netlink). Triggers udev rules for device creation/removal. (Ch. 4)

---

## V

**V4L2** — Video4Linux2. Multimedia subsystem for cameras, video codecs, TV tuners. Userspace via `/dev/videoN`. (Ch. 22)

**VFS** — Virtual File System. Abstraction layer providing uniform file operations. Routes `read()`/`write()` to specific filesystem or device driver. (Ch. 3)

---

## W

**Workqueue** — Bottom-half mechanism using kernel worker threads. Can sleep. `schedule_work()` or `queue_work()`. (Ch. 15)

---

*Next: [Chapter 33 — OS Comparison](Chapter_33_OS_Comparison.md)*
