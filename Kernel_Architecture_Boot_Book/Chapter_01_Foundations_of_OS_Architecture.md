# Chapter 1: Foundations of Operating System Architecture

## Learning Goals
- Understand what an operating system is and why it exists
- Grasp the role of the kernel as the core of the OS
- Distinguish kernel space from user space
- Compare monolithic, microkernel, and hybrid architectures
- Understand the Linux kernel's architectural position

---

## 1.1 What Is an Operating System?

An **operating system (OS)** is system software that manages hardware resources and provides services for application programs. It acts as an intermediary between user applications and the physical hardware.

```
┌─────────────────────────────────────────────────────────┐
│                    USER APPLICATIONS                     │
│          (browsers, editors, servers, games)             │
├─────────────────────────────────────────────────────────┤
│                   SYSTEM LIBRARIES                       │
│               (glibc, Bionic, musl libc)                │
├─────────────────────────────────────────────────────────┤
│                  OPERATING SYSTEM                        │
│  ┌───────────────────────────────────────────────────┐  │
│  │                    KERNEL                          │  │
│  │  Process Mgmt │ Memory Mgmt │ Device Drivers      │  │
│  │  File Systems │ Networking  │ Security             │  │
│  └───────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────┤
│                      HARDWARE                            │
│           CPU   RAM   Disk   NIC   GPU   Sensors        │
└─────────────────────────────────────────────────────────┘
```

An OS provides five fundamental services:

| Service | Description | Example |
|---------|-------------|---------|
| **Process management** | Create, schedule, terminate processes | `fork()`, `exec()`, `exit()` |
| **Memory management** | Virtual memory, paging, allocation | `mmap()`, `brk()`, page fault handling |
| **File system management** | File I/O, directory structure | `open()`, `read()`, `write()` |
| **Device management** | Hardware abstraction via drivers | `/dev/sda`, `/dev/ttyS0` |
| **Security & protection** | Access control, isolation | DAC, MAC, capabilities, namespaces |

---

## 1.2 Role of the Kernel in an Operating System

The **kernel** is the core component of the OS. It runs with full hardware privileges and provides the fundamental services that all other software depends on.

```
   Application (user space)
        │
        │  system call (e.g., read())
        ▼
   ┌─────────────────────────────┐
   │         KERNEL              │
   │                             │
   │  1. Validate parameters     │
   │  2. Check permissions       │
   │  3. Perform operation       │
   │  4. Return result           │
   │                             │
   └─────────┬───────────────────┘
             │
             ▼
        Hardware (disk, memory, device)
```

The kernel's responsibilities:

| Responsibility | Details |
|---------------|---------|
| **Hardware abstraction** | Hides hardware differences behind uniform APIs |
| **Resource multiplexing** | Shares CPU, memory, I/O among processes |
| **Protection** | Prevents processes from interfering with each other |
| **Privilege enforcement** | Only kernel code can execute privileged instructions |
| **System call interface** | Provides controlled entry points for user code |

---

## 1.3 Kernel vs User Space

Modern processors provide **privilege levels** (also called **rings** on x86 or **exception levels** on ARM64). The OS uses these to separate trusted kernel code from untrusted user code.

```
x86 Architecture:                ARM64 Architecture:

Ring 0 ──── Kernel               EL3 ──── Secure Monitor (ATF)
Ring 1 ──── (unused in Linux)    EL2 ──── Hypervisor (KVM)
Ring 2 ──── (unused in Linux)    EL1 ──── Kernel (Linux)
Ring 3 ──── User applications    EL0 ──── User applications

     Most privileged                  Most privileged
          ↑                                ↑
          │                                │
     Least privileged                 Least privileged
```

| Aspect | Kernel Space | User Space |
|--------|-------------|------------|
| Privilege | Full hardware access (Ring 0 / EL1) | Restricted access (Ring 3 / EL0) |
| Memory | Can access all physical + virtual memory | Can only access own virtual address space |
| Instructions | Can execute privileged instructions | Privileged instructions cause fault |
| Crash impact | Kernel panic — entire system down | Only the process crashes |
| Address range (64-bit) | `0xFFFF000000000000` — top | `0x0000000000000000` — bottom |
| Entry mechanism | System call, interrupt, exception | Return from system call |

```
Virtual Address Space Layout (64-bit Linux):

0xFFFFFFFFFFFFFFFF ┌───────────────────────┐
                   │                       │
                   │    KERNEL SPACE        │  ← Mapped in every process
                   │    (shared across      │     but not accessible from
                   │     all processes)     │     user mode
                   │                       │
0xFFFF800000000000 ├───────────────────────┤
                   │  Non-canonical hole   │  ← Access causes fault
0x0000800000000000 ├───────────────────────┤
                   │                       │
                   │    USER SPACE          │  ← Per-process mapping
                   │    Stack ↓            │
                   │    ...                │
                   │    mmap region        │
                   │    Heap  ↑            │
                   │    BSS                │
                   │    Data               │
                   │    Text (code)        │
0x0000000000000000 └───────────────────────┘
```

---

## 1.4 System Software Architecture Overview

A modern computing system is a layered stack where each layer provides abstractions for the layer above:

```
Layer 5: Applications
         ├── Web browsers, databases, Android apps
         │
Layer 4: System Libraries & Frameworks
         ├── glibc/Bionic, Android Framework, Qt
         │
Layer 3: Operating System Kernel
         ├── Linux kernel, Windows NT kernel, XNU
         │
Layer 2: Firmware
         ├── UEFI, U-Boot SPL, TrustZone firmware
         │
Layer 1: Hardware
         ├── CPU, memory controller, peripherals, buses
         │
Layer 0: Silicon / Transistors
         └── Physical gates, flip-flops, interconnects
```

Each layer only communicates with adjacent layers through well-defined interfaces:
- **Applications → Libraries**: Function calls (POSIX API)
- **Libraries → Kernel**: System calls (`syscall` instruction)
- **Kernel → Hardware**: MMIO reads/writes, special instructions
- **Hardware → Kernel**: Interrupts, exceptions, DMA completion

---

## 1.5 Monolithic Kernels

A **monolithic kernel** runs all OS services (process management, memory management, file systems, device drivers, networking) in a single address space in kernel mode.

```
┌─────────────────────────────────────────────────────────┐
│                    USER SPACE                            │
│              Applications  Libraries                     │
├─────────────────── System Call Interface ────────────────┤
│                                                         │
│                  KERNEL SPACE                            │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────┐  │
│  │ Process  │  │ Memory  │  │  File   │  │ Network  │  │
│  │Scheduler │  │ Manager │  │ Systems │  │  Stack   │  │
│  └────┬─────┘  └────┬────┘  └────┬────┘  └────┬─────┘  │
│       │             │            │             │        │
│  ┌────▼─────────────▼────────────▼─────────────▼─────┐  │
│  │              Device Drivers                        │  │
│  │     (all drivers run in kernel mode)               │  │
│  └────────────────────────────────────────────────────┘  │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                      HARDWARE                            │
└─────────────────────────────────────────────────────────┘

                 ALL in ONE address space
```

| Advantage | Disadvantage |
|-----------|-------------|
| Fast — no context switch between components | Large codebase (30M+ lines for Linux) |
| Direct function calls between subsystems | A bug in any driver can crash the entire kernel |
| Efficient shared data structures | Harder to formally verify |
| Proven performance (Linux, FreeBSD) | Adding new features increases complexity |

**Examples**: Linux, FreeBSD, OpenBSD, classic Unix

---

## 1.6 Microkernels

A **microkernel** runs only the absolute minimum in kernel mode: IPC, basic scheduling, and address space management. Everything else (drivers, file systems, networking) runs in user-space servers.

```
┌──────────────────────────────────────────────────────────┐
│                       USER SPACE                          │
│                                                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────┐   │
│  │  File   │  │ Network │  │ Device  │  │ Display  │   │
│  │ System  │  │  Stack  │  │ Driver  │  │ Server   │   │
│  │ Server  │  │ Server  │  │ Server  │  │          │   │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬─────┘   │
│       │            │            │             │          │
│       └────────────┼────────────┼─────────────┘          │
│                    │    IPC     │                         │
├────────────────────▼────────────▼─────────────────────────┤
│                   MICROKERNEL                             │
│          ┌──────────────────────────┐                     │
│          │  IPC  │ Scheduler │  MM  │                     │
│          │       │ (basic)   │(basic)│                    │
│          └──────────────────────────┘                     │
│              (minimal code in kernel mode)                │
├──────────────────────────────────────────────────────────┤
│                        HARDWARE                           │
└──────────────────────────────────────────────────────────┘
```

| Advantage | Disadvantage |
|-----------|-------------|
| Small, verifiable kernel (< 10K lines) | IPC overhead for every operation |
| Driver crash doesn't crash the kernel | Slower than monolithic for I/O-heavy workloads |
| Easier formal verification (seL4) | Complex system architecture |
| Better fault isolation | Fewer developers, smaller ecosystem |

**Examples**: QNX (automotive/safety-critical), MINIX, seL4, L4, Fuchsia (Zircon)

---

## 1.7 Hybrid Kernels

A **hybrid kernel** combines aspects of both: a monolithic core with some microkernel concepts. Some components run in user space, but performance-critical parts remain in kernel space.

```
┌──────────────────────────────────────────────────────────┐
│                     USER SPACE                            │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  Applications    User-mode Drivers (some)           │ │
│  └─────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────┤
│                    KERNEL SPACE                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  Microkernel-like core          Monolithic services │ │
│  │  (IPC, basic MM)               (FS, Net, Drivers)  │ │
│  │  ┌──────────┐  ┌──────────────────────────────────┐│ │
│  │  │ Message  │  │ File Systems, Network Stack,     ││ │
│  │  │ Passing  │  │ Device Drivers (in-kernel)       ││ │
│  │  └──────────┘  └──────────────────────────────────┘│ │
│  └─────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────┤
│                      HARDWARE                             │
└──────────────────────────────────────────────────────────┘
```

| Advantage | Disadvantage |
|-----------|-------------|
| Flexible — can move components between kernel/user | Neither purely simple nor purely efficient |
| Practical compromise | Classification is debatable |
| Good for commercial OS | Can inherit disadvantages of both |

**Examples**: Windows NT/10/11, macOS (XNU = Mach + BSD), BeOS/Haiku

---

## 1.8 Linux Kernel Architecture Overview

Linux is a **monolithic kernel with loadable module support**. This gives it the performance of a monolithic design while allowing runtime extensibility.

```
┌──────────────────────────────────────────────────────────────┐
│                        USER SPACE                             │
│    App    App    App    Android Framework    HAL              │
├────────────────── System Call Interface ──────────────────────┤
│                                                              │
│                       KERNEL SPACE                            │
│                                                              │
│   ┌──────────────────────────────────────────────────────┐   │
│   │              CORE KERNEL (always loaded)              │   │
│   │                                                      │   │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │   │
│   │  │ Process  │  │  Memory  │  │   VFS (Virtual   │   │   │
│   │  │ Scheduler│  │  Manager │  │   File System)   │   │   │
│   │  └──────────┘  └──────────┘  └──────────────────┘   │   │
│   │                                                      │   │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │   │
│   │  │Interrupt │  │ Network  │  │    Security       │   │   │
│   │  │Subsystem │  │  Stack   │  │ (LSM, SELinux)   │   │   │
│   │  └──────────┘  └──────────┘  └──────────────────┘   │   │
│   └──────────────────────────────────────────────────────┘   │
│                                                              │
│   ┌──────────────────────────────────────────────────────┐   │
│   │           LOADABLE KERNEL MODULES (.ko)               │   │
│   │                                                      │   │
│   │  ┌────────┐  ┌────────┐  ┌────────┐  ┌──────────┐   │   │
│   │  │  ext4  │  │  NFS   │  │  USB   │  │  GPU/DRM │   │   │
│   │  │  .ko   │  │  .ko   │  │  .ko   │  │   .ko    │   │   │
│   │  └────────┘  └────────┘  └────────┘  └──────────┘   │   │
│   │                                                      │   │
│   │  ┌────────┐  ┌────────┐  ┌────────┐  ┌──────────┐   │   │
│   │  │ Wi-Fi  │  │  i2c   │  │  V4L2  │  │  Sound   │   │   │
│   │  │  .ko   │  │  .ko   │  │  .ko   │  │   .ko    │   │   │
│   │  └────────┘  └────────┘  └────────┘  └──────────┘   │   │
│   └──────────────────────────────────────────────────────┘   │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                          HARDWARE                             │
│       CPU (x86/ARM/RISC-V)  Memory  Peripherals  Buses      │
└──────────────────────────────────────────────────────────────┘
```

Key characteristics of the Linux kernel:
- **Monolithic**: All core services run in kernel space for performance
- **Modular**: Drivers and file systems can be loaded/unloaded at runtime (`.ko` files)
- **Preemptive**: Supports voluntary, full, and real-time preemption models
- **SMP-capable**: Scales from single-core embedded to 1000+ core servers
- **Portable**: Runs on 30+ architectures (x86, ARM, ARM64, RISC-V, MIPS, etc.)
- **Open source**: GPL v2 licensed, community-developed

```c
/* Linux's modular monolithic approach */

/* Built-in (always in kernel) — configured as y */
CONFIG_SCHED_CFS=y          /* Can't remove the scheduler */
CONFIG_MMU=y                /* Can't remove memory management */

/* Loadable module (optional) — configured as m */
CONFIG_EXT4_FS=m            /* Can load/unload ext4 at runtime */
CONFIG_USB_STORAGE=m        /* Can load/unload USB storage */
CONFIG_SND_SOC=m            /* Can load/unload audio */

/* Disabled — not compiled at all */
# CONFIG_NTFS_FS is not set /* Not needed, not compiled */
```

---

## Kernel Source References

| File | Content |
|------|---------|
| `init/main.c` | Kernel main entry — `start_kernel()` |
| `arch/x86/entry/syscalls/` | x86 system call table |
| `arch/arm64/kernel/entry.S` | ARM64 exception/syscall entry |
| `kernel/module/main.c` | Module loading infrastructure |
| `include/linux/syscalls.h` | System call declarations |

---

## OS Comparison

| Concept | Linux | Windows NT | macOS (XNU) | QNX |
|---------|-------|-----------|-------------|-----|
| Architecture | Monolithic + modules | Hybrid | Hybrid (Mach + BSD) | Microkernel |
| Kernel code size | ~30M lines | ~50M lines (est.) | ~7M lines | ~150K lines |
| Driver location | Kernel space | Kernel + some user | Kernel (kexts) + user (dexts) | User space |
| Module system | `.ko` (insmod/modprobe) | `.sys` (services) | `.kext` / DriverKit | User processes |
| Preemption | Configurable (NONE/FULL/RT) | Full preemption | Full preemption | Full preemption |
| Real-time | PREEMPT_RT patch | Not designed for RT | Not designed for RT | Native RTOS |

---

## Interview Questions

**Q1: What are the differences between monolithic and microkernel architectures?**
A: In a monolithic kernel (Linux), all OS services run in kernel mode in a single address space — fast but a driver bug can crash the system. In a microkernel (QNX), only IPC, basic scheduling, and MM are in kernel mode — drivers run in user space with fault isolation but IPC overhead.

**Q2: Why did Linux choose a monolithic design?**
A: Performance — direct function calls between subsystems are faster than IPC messages. Linus Torvalds pragmatically chose monolithic for efficiency, adding loadable modules to get some modularity benefits without the IPC cost.

**Q3: How does the kernel enforce the boundary between kernel and user space?**
A: The CPU hardware enforces it via privilege levels (Ring 0 vs Ring 3 on x86, EL1 vs EL0 on ARM64). User code cannot execute privileged instructions or access kernel memory. The only way into kernel mode is through controlled entry points: system calls, interrupts, and exceptions.

**Q4: What is the advantage of loadable kernel modules?**
A: Modules let you add/remove functionality (drivers, file systems) without recompiling or rebooting the kernel. This keeps the base kernel small while supporting vast hardware. Only needed modules are loaded, saving memory on embedded systems.

---

## Summary

- An OS manages hardware and provides services; the kernel is its privileged core
- Kernel space has full hardware access; user space is restricted and must use system calls
- Monolithic kernels (Linux) are fast but a bug anywhere in kernel code can crash the system
- Microkernels (QNX) are safer but slower due to IPC overhead
- Linux is a monolithic kernel with loadable module support — combining performance with extensibility
- CPU hardware (privilege levels) enforces the kernel/user boundary

---

*Next: [Chapter 2 — History and Evolution of the Linux Kernel](Chapter_02_History_and_Evolution.md)*
