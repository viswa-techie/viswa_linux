# Chapter 2: History and Evolution of Device Drivers

## Chapter Overview

Understanding the history of device drivers explains *why* the Linux driver architecture looks the way it does today. This chapter traces the journey from bare-metal I/O to the modern Linux Device Model.

---

## 2.1 Early Computer I/O Systems

### 1950s–1960s: Direct Hardware Manipulation
```
┌────────────┐     Direct I/O Instructions     ┌──────────┐
│ Application ├──────────────────────────────────►│ Hardware │
│ (Assembly)  │     IN/OUT, memory-mapped        │ (Printer)│
└────────────┘                                    └──────────┘
```

- No operating system — programs directly poked hardware registers
- Each program had to know the exact hardware addresses
- No abstraction, no sharing, no protection
- IBM 360 introduced **channels** — early DMA-like I/O processors

### 1960s–1970s: I/O Abstraction Emerges
- Multics (1964): First OS with device-independent I/O
- "Everything is a file" concept begins to form
- Device tables map logical names to physical hardware

---

## 2.2 Device Drivers in Early Unix Systems

### Unix V6 (1975): The Foundation
```
Unix V6 Device Table (cdevsw[], bdevsw[])
┌────────┬──────┬───────┬───────┬───────┐
│ Major# │ open │ close │ read  │ write │
├────────┼──────┼───────┼───────┼───────┤
│   0    │ tty  │ tty   │ tty   │ tty   │  ← Console
│   1    │ lp   │ lp    │ lp    │ lp    │  ← Line printer
│   2    │ mem  │ mem   │ mem   │ mem   │  ← /dev/mem
└────────┼──────┼───────┼───────┼───────┘
         Function pointer tables
```

- Dennis Ritchie & Ken Thompson: **cdevsw** (character) and **bdevsw** (block) switch tables
- Major number indexes into the table → selects the driver
- Minor number passed to the driver → selects sub-device
- This design persisted for decades

### BSD (1977–1990s): Enhancements
- Added network device drivers (not file-based — socket interface)
- Introduced `ioctl()` for device control beyond read/write
- `autoconfiguration` for device probing at boot

### System V: Streams Framework
- AT&T introduced STREAMS for network and terminal drivers
- Layered, modular I/O processing — influenced later designs
- Complex but powerful — not adopted by Linux

---

## 2.3 Evolution of Linux Driver Architecture

### Timeline

| Period | Kernel | Key Driver Changes |
|--------|--------|--------------------|
| 1991–1994 | 0.01–1.0 | Monolithic, all drivers compiled in, basic cdevsw-style |
| 1995–1999 | 1.2–2.2 | Loadable modules, /proc for device info |
| 2000–2003 | 2.4 | devfs (device filesystem), USB support |
| 2003–2005 | 2.6 (*) | **Linux Device Model**, sysfs, udev, kobject |
| 2006–2011 | 2.6.x | Device Tree support, regmap, devm_* APIs |
| 2012–2019 | 3.x–5.x | DT overlays, component framework, VFIO |
| 2020–now | 5.x–6.x | Rust drivers, io_uring, cleanup improvements |

> (*) Linux 2.6 (December 2003) was the **watershed moment** for the driver subsystem.

### The 2.6 Revolution: Linux Device Model

Before 2.6:
```
┌──────────────────┐
│   Flat driver     │
│   registration    │     No hierarchy
│   (major/minor)   │     No power management
│   No sysfs        │     No hot-plug framework
└──────────────────┘
```

After 2.6:
```
┌──────────────────────────────────────────┐
│              Linux Device Model           │
│                                          │
│  struct bus_type ─── struct device        │
│       │                  │               │
│       │              struct device_driver │
│       │                  │               │
│  sysfs (/sys)       kobject hierarchy    │
│       │                  │               │
│  udev (userspace)   power management     │
│       │                  │               │
│  hot-plug            device binding      │
└──────────────────────────────────────────┘
```

---

## 2.4 Monolithic vs Microkernel Driver Models

```
         Monolithic (Linux)                    Microkernel (QNX/L4)
┌─────────────────────────────┐    ┌─────────────────────────────┐
│       KERNEL SPACE           │    │       KERNEL SPACE          │
│                              │    │  (minimal: IPC, scheduler)  │
│  Scheduler │ Memory │ VFS    │    └─────────────────────────────┘
│  ─────────────────────────── │    ┌──────┐ ┌──────┐ ┌──────┐
│  Driver A │ Driver B │ Net   │    │Drv A │ │Drv B │ │ FS   │
│  (all run in kernel space)   │    │(user)│ │(user)│ │(user)│
└─────────────────────────────┘    └──────┘ └──────┘ └──────┘
                                      ALL RUN IN USER SPACE
Crash in Driver A → kernel panic      Crash in Drv A → restart Drv A
Fast (no IPC overhead)                Slower (IPC between components)
```

| Aspect | Monolithic (Linux) | Microkernel (QNX) | Hybrid (Windows) |
|--------|-------------------|-------------------|-------------------|
| Driver runs in | Kernel space | User space | Kernel (WDM) or User (UMDF) |
| Driver crash | Kernel panic possible | Restart driver only | BSOD (kernel), recoverable (user) |
| Performance | Fast (direct calls) | Slower (IPC overhead) | Mixed |
| Complexity | Shared state, harder isolation | Clean interfaces, more IPC code | Complex layering |
| Linux approach | Modules + fault isolation efforts | N/A | N/A |

**Linux's pragmatic solution**: Monolithic for performance, but with loadable modules for flexibility, and ongoing efforts for driver isolation (eBPF, Rust drivers, VFIO).

---

## 2.5 Driver Model Evolution in Linux Kernel

### Phase 1: Simple Registration (1991–2002)
```c
/* Linux 1.x–2.4 style */
register_chrdev(MAJOR_NUM, "mydev", &fops);
/* That's it — no bus, no device model, no sysfs */
```

### Phase 2: kobject + sysfs (2.6, 2003)
```c
/* Greg KH & Pat Mochel introduced the unified device model */
struct bus_type {
    const char *name;
    int (*match)(struct device *, struct device_driver *);
    int (*probe)(struct device *);
};

struct device {
    struct kobject kobj;   /* sysfs representation */
    struct bus_type *bus;
    struct device *parent;
    /* ... */
};
```

### Phase 3: Device Tree (2011+)
```c
/* ARM switched from board files to Device Tree */
/* Before (board file): */
static struct platform_device my_uart = {
    .name = "my-uart",
    .resource = { ... },
};
platform_device_register(&my_uart);

/* After (Device Tree): */
/* my-uart described in .dts, kernel parses automatically */
```

### Phase 4: devm_* Managed APIs (2012+)
```c
/* Before: manual cleanup on every error path */
buf = kmalloc(size, GFP_KERNEL);
if (!buf) goto err_buf;
irq = request_irq(...);
if (irq < 0) goto err_irq;
/* Error handling: free in reverse order */

/* After: automatic cleanup on driver detach/error */
buf = devm_kmalloc(dev, size, GFP_KERNEL);
irq = devm_request_irq(dev, ...);
/* Freed automatically when device is removed */
```

### Phase 5: Rust in the Kernel (2022+)
```
Linux 6.1+: Rust as a second language for drivers
Goal: Memory safety guarantees at compile time
Status: Infrastructure merged, sample drivers available
Example: drivers/net/phy/ax88796b_rust.rs (experimental)
```

---

## 2.6 Introduction of Linux Device Model

### The Core Problem It Solves

Before the device model, the kernel had no unified way to:
- Enumerate all devices in the system
- Match devices to their drivers
- Manage power (no way to walk the device tree)
- Support hot-plug (no notifications)
- Expose device info to user space

### The Four Pillars

```
┌──────────────────────────────────────────────────────────┐
│                LINUX DEVICE MODEL                        │
│                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │   BUS    │    │  DEVICE  │    │  DRIVER  │          │
│  │ (bus_type)│    │ (device) │    │(device_  │          │
│  │          │    │          │    │ driver)  │          │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘          │
│       │               │               │                 │
│       └───── match() ─┴─── probe() ───┘                │
│                                                          │
│  ┌──────────┐                                           │
│  │  CLASS   │    Groups similar devices                 │
│  │  (class) │    e.g., "input", "tty", "net"           │
│  └──────────┘                                           │
│                                                          │
│  All exposed via /sys (sysfs)                           │
└──────────────────────────────────────────────────────────┘
```

### sysfs: The Visible Output
```
/sys/
├── bus/
│   ├── platform/
│   │   ├── devices/      → links to actual devices
│   │   └── drivers/      → registered platform drivers
│   ├── pci/
│   ├── usb/
│   └── i2c/
├── class/
│   ├── input/
│   ├── tty/
│   └── net/
├── devices/
│   └── platform/
│       ├── serial8250/
│       └── my-device@10000000/
└── module/
    └── my_driver/
```

### Key Developers
- **Greg Kroah-Hartman**: Maintainer of driver core, USB, staging
- **Pat Mochel**: Original architect of the Linux Device Model
- **Jonathan Corbet**: Documentation, LWN.net editor

---

## Kernel Source References

| File | Purpose |
|------|---------|
| `drivers/base/core.c` | `device_register()`, `device_add()` |
| `drivers/base/bus.c` | `bus_register()`, device-driver matching |
| `drivers/base/driver.c` | `driver_register()`, probe/remove |
| `drivers/base/class.c` | Device class management |
| `include/linux/kobject.h` | kobject, kset — sysfs building blocks |
| `fs/sysfs/` | sysfs filesystem implementation |
| `Documentation/driver-api/driver-model/` | Official documentation |

---

## OS Comparison: Driver Evolution

| Era | Linux | Windows | macOS | RTOS |
|-----|-------|---------|-------|------|
| 1990s | Flat cdevsw tables | VxD (Win 3.1/95) | NuBus drivers | Bare-metal ISRs |
| 2000s | Device Model + sysfs | WDM + INF | IOKit C++ classes | Static config tables |
| 2010s | Device Tree + devm_* | WDF (KMDF/UMDF) | System Extensions | Zephyr device model |
| 2020s | Rust drivers, io_uring | Modern WDF 2.x | DriverKit (user) | Zephyr devicetree |

---

## Interview Questions

**Q1: Why was the Linux Device Model introduced in 2.6?**
A: To provide unified device enumeration, device-driver matching, power management (walk device tree for suspend/resume), hot-plug support, and user-space visibility via sysfs. Before 2.6, each subsystem had ad-hoc mechanisms.

**Q2: What was devfs and why was it replaced by udev?**
A: devfs was a kernel filesystem that automatically created /dev entries. It was replaced by udev (userspace device manager) because: policy in kernel is bad practice, naming rules should be configurable, and kernel shouldn't decide permissions. udev receives events from the kernel and creates /dev nodes based on rules.

**Q3: What are board files and why were they replaced by Device Tree?**
A: Board files were C source files (arch/arm/mach-xxx/board-xxx.c) that hard-coded platform device descriptions. Replacing them with Device Tree (.dts) separates hardware description from kernel code, enables one kernel binary to support multiple boards, and reduces arch/arm/ code bloat.

**Q4: What advantage do devm_* APIs provide?**
A: Automatic resource cleanup when the device is removed or probe fails. Eliminates complex error-path cleanup code and prevents resource leaks. Resources tied to the device lifetime, freed in reverse allocation order.

**Q5: What is a kobject and what role does it play?**
A: kobject is the base object type in the Linux device model. Every bus, device, driver has one. It provides: reference counting (kref), sysfs directory creation, parent-child hierarchy, and uevent generation for hot-plug.

---

## Summary

| Period | Key Change | Impact |
|--------|-----------|--------|
| 1975 (Unix V6) | cdevsw/bdevsw tables | First driver abstraction |
| 1991 (Linux 0.01) | Monolithic, compiled-in | Simple but inflexible |
| 1996 (Linux 2.0) | Loadable modules | Runtime driver loading |
| 2003 (Linux 2.6) | Device Model + sysfs | Unified hierarchy, power mgmt |
| 2011+ | Device Tree | Hardware description separated |
| 2012+ | devm_* APIs | Automatic resource management |
| 2022+ | Rust drivers | Memory safety for drivers |

The constant theme: **increasing abstraction, modularity, and safety** — while maintaining performance.

---

*Next: [Chapter 3 — Linux Kernel Architecture Overview](Chapter_03_Kernel_Architecture.md)*
