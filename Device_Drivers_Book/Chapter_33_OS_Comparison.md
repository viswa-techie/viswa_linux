# Chapter 33: OS Comparison — Driver Models Across Operating Systems

## Chapter Overview

Comparing Linux driver architecture with Windows, macOS, QNX, and Android HAL. Understanding these differences is valuable for porting drivers, interviews, and architectural decisions.

---

## 33.1 Architecture Comparison

| Aspect | Linux | Windows | macOS | QNX | Android |
|--------|-------|---------|-------|-----|---------|
| **Kernel type** | Monolithic (modular) | Hybrid | Hybrid (XNU = Mach + BSD) | Microkernel | Linux + HAL |
| **Driver runs in** | Kernel space | Kernel space | Kernel space (kext → dext) | User space* | Kernel + HAL user space |
| **Driver framework** | Bus/device/driver model | WDM / WDF (KMDF/UMDF) | IOKit / DriverKit | Resource managers | Linux + HIDL/AIDL |
| **Device description** | Device Tree / ACPI | ACPI / INF files | IOKit matching | Device tree / custom | Device Tree |
| **Source available** | Yes (GPL) | No (DDK docs) | Partial (XNU open) | No | Kernel yes, HAL partial |

*QNX drivers run in user space by default (microkernel), gaining isolation at the cost of IPC overhead.

---

## 33.2 Driver Model Comparison

### Linux: Bus-Device-Driver

```
struct bus_type → match(dev, drv) → probe(dev)
     ├── struct device (sysfs)
     └── struct device_driver (module)

Matching: compatible string (DT), vendor:device (PCI), etc.
Registration: platform_driver_register(), module_pci_driver(), etc.
```

### Windows: WDF (Windows Driver Framework)

```
WDF Object Model:
  WDFDRIVER → WDFDEVICE → WDFQUEUE → WDFREQUEST

Driver Entry → EvtDriverDeviceAdd() (similar to probe)
I/O: Request queue model, not file_operations
PM: Built into WDF framework
PnP: Automatic via INF files

Key differences:
  - Object model with parent-child ownership (auto-cleanup)
  - KMDF (kernel) / UMDF (user-space) same programming model
  - IRP (I/O Request Packet) instead of syscall → VFS → driver
```

### macOS: IOKit → DriverKit

```
IOKit (legacy, kernel space):
  IOService (base class) → matching dictionary → probe/start/stop
  C++ object-oriented driver model
  IORegistryExplorer for device tree visualization

DriverKit (modern, user space):
  System extensions (dext) run in user space
  IOService subclass but sandboxed
  Apple pushing all new drivers here

Key differences:
  - C++ based (Linux is C)
  - Object inheritance for driver families
  - Automatic reference counting
```

### QNX: Resource Managers

```
QNX (microkernel):
  Drivers are user-space processes (resource managers)
  Register pathname (/dev/mydev) via resmgr_attach()
  Receive messages via MsgReceive()

  Client: open("/dev/mydev") → QNX routes to resource manager
  
  Advantages: crash isolation (driver crash ≠ kernel crash)
  Disadvantages: IPC overhead, context switches per I/O

Key differences:
  - User-space by default (Linux: kernel space)
  - Message-passing IPC (Linux: function calls)
  - No kernel modules concept
```

---

## 33.3 Interrupt Handling Comparison

| Feature | Linux | Windows | QNX | macOS |
|---------|-------|---------|-----|-------|
| Register handler | `request_irq()` | `IoConnectInterrupt()` | `InterruptAttach()` | `registerInterrupt()` |
| Top/bottom half | Hard IRQ + threaded/workqueue | ISR + DPC | ISR + pulse to thread | Primary + work loop |
| Threaded IRQ | Yes (`IRQF_ONESHOT`) | UMDF auto | Default (user space) | DriverKit event queue |
| Shared IRQ | Yes (`IRQF_SHARED`) | Yes | No (dedicated) | Yes |

---

## 33.4 Memory/DMA Comparison

| Feature | Linux | Windows | QNX | macOS |
|---------|-------|---------|-----|-------|
| Coherent DMA | `dma_alloc_coherent()` | `AllocCommonBuffer()` | `mmap_device_memory()` | `IOBufferMemoryDescriptor` |
| Streaming DMA | `dma_map_single()` | `MapTransfer()` | manual cache flush | `IODMACommand` |
| MMIO access | `ioremap() + readl()` | `MmMapIoSpace()` | `mmap_device_io()` | `IOMemoryMap` |
| IOMMU | Transparent via DMA API | Transparent via HAL | Manual | Transparent |

---

## 33.5 Power Management Comparison

| Feature | Linux | Windows | QNX |
|---------|-------|---------|-----|
| Runtime PM | `pm_runtime_*()` | D0/D3 via WDF | Manual |
| System suspend | `dev_pm_ops` callbacks | Dx power IRP | Custom |
| Auto-suspend | `pm_runtime_use_autosuspend()` | S0ix idle | N/A |
| Wake source | `enable_irq_wake()` | `WdfDeviceAssignWakeSettings()` | Custom |

---

## 33.6 Android HAL Architecture

Android adds hardware abstraction on top of Linux kernel drivers:

```
┌──────────────────────────┐
│    Android Application   │
├──────────────────────────┤
│    Android Framework     │
│    (Java/Kotlin)         │
├──────────────────────────┤
│    HAL Interface         │
│    (AIDL / HIDL)         │
├──────────────────────────┤
│    HAL Implementation    │
│    (C++ user-space)      │
├──────────────────────────┤
│    Linux Kernel Driver   │
│    (standard driver)     │
├──────────────────────────┤
│    Hardware              │
└──────────────────────────┘
```

### AIDL HAL Example (Modern Android)

```cpp
// HAL interface definition (AIDL)
interface IMyDevice {
    int readRegister(int offset);
    void writeRegister(int offset, int value);
}

// HAL implementation (C++)
class MyDevice : public BnMyDevice {
    int readRegister(int offset) override {
        int fd = open("/dev/my_device", O_RDWR);
        // ioctl to kernel driver
        struct reg_access ra = { .offset = offset };
        ioctl(fd, MY_READ_REG, &ra);
        close(fd);
        return ra.value;
    }
};
```

### Key Android-Specific Concepts

| Concept | Description |
|---------|-------------|
| **Treble** | Separation of Android framework from vendor HAL |
| **VNDK** | Vendor NDK — libraries available to vendor code |
| **SELinux** | Mandatory access control for all device access |
| **init.rc** | Service definitions for HAL processes |
| **VINTF** | Vendor Interface manifest (compatibility matrix) |

---

## 33.7 When to Choose Which Model

| Scenario | Best Choice | Why |
|----------|------------|-----|
| Safety-critical (automotive) | QNX / Linux + isolation | Driver crash containment |
| Consumer electronics | Linux | Open source, community |
| Desktop/laptop | Linux or Windows | Application ecosystem |
| Mobile | Android (Linux) | Google ecosystem |
| Embedded real-time | QNX or Linux + PREEMPT_RT | Deterministic latency |
| GPU/graphics | Linux (DRM) or Windows | Driver maturity |

---

## Interview Questions

**Q1: What is the fundamental difference between Linux and QNX driver architectures?**
A: Linux drivers run in kernel space (monolithic kernel) — a driver bug can crash the system. QNX is a microkernel: drivers run as user-space resource managers communicating via IPC. A QNX driver crash is isolated and can be restarted. The trade-off is IPC overhead in QNX.

**Q2: How does Android HAL relate to Linux kernel drivers?**
A: Android HAL is a user-space abstraction layer between Android framework and Linux kernel drivers. The kernel driver handles hardware directly; the HAL implementation (C++) communicates with it via `/dev/` nodes and ioctl. AIDL/HIDL defines the interface. Treble ensures HAL compatibility across Android versions.

**Q3: Why is Apple moving from IOKit to DriverKit?**
A: Security and stability. IOKit drivers (kexts) run in kernel space — a bug crashes the system and a vulnerability compromises it. DriverKit (dexts) run in user space with sandboxing, like QNX's model. Crash isolation + reduced attack surface.

---

*Next: [Chapter 34 — Embedded & Automotive Development](Chapter_34_Embedded_Development.md)*
