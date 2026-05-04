# Chapter 4: Linux Kernel Architecture Overview

## Learning Goals
- Understand the major components of the Linux kernel
- Map the core subsystems and their interactions
- Grasp the modular design and how subsystems communicate
- Understand kernel services and abstractions
- Know the kernel execution environment and constraints

---

## 4.1 Kernel Architecture Components

```
Complete Linux Kernel Architecture:

┌──────────────────────────────────────────────────────────────────┐
│                         USER SPACE                                │
│   Applications    Libraries    Frameworks    Daemons              │
├──────────────── System Call Interface (syscall table) ────────────┤
│                                                                  │
│                        KERNEL SPACE                               │
│                                                                  │
│  ┌─────────────────────── VFS Layer ──────────────────────────┐  │
│  │  open()  read()  write()  close()  ioctl()  mmap()        │  │
│  │  Unified interface for ALL file-like objects               │  │
│  └──────┬──────────┬───────────┬──────────┬──────────────────┘  │
│         │          │           │          │                      │
│   ┌─────▼────┐ ┌───▼────┐ ┌───▼────┐ ┌──▼──────┐              │
│   │  ext4    │ │  f2fs  │ │  proc  │ │  sysfs  │ ← File       │
│   │  xfs     │ │  btrfs │ │  devfs │ │  debugfs│   Systems    │
│   └──────────┘ └────────┘ └────────┘ └─────────┘              │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                 Process Management                         │  │
│  │  Scheduler (CFS/RT/DL) │ fork/clone │ Signals │ Threads   │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                 Memory Management                          │  │
│  │  Page Allocator │ Slab │ vmalloc │ mmap │ Page Cache      │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                 Network Stack                              │  │
│  │  Socket Layer │ TCP/IP │ Netfilter │ CAN │ Wireless       │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                 Device Driver Layer                         │  │
│  │  char │ block │ net │ platform │ i2c │ spi │ usb │ drm    │  │
│  └───────────────────────────┬────────────────────────────────┘  │
│                               │                                  │
│  ┌────────────────────────────▼───────────────────────────────┐  │
│  │            Architecture-Specific Code (arch/)              │  │
│  │  Boot │ MMU setup │ Interrupts │ Context switch │ SMP     │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│                        HARDWARE                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4.2 Core Kernel Subsystems

| Subsystem | Kernel Directory | Responsibility |
|-----------|-----------------|----------------|
| **Process management** | `kernel/sched/`, `kernel/fork.c` | Process creation, scheduling, termination, signals |
| **Memory management** | `mm/` | Virtual memory, paging, allocation, swap, OOM |
| **VFS** | `fs/` | Unified file system interface, inode/dentry caching |
| **Networking** | `net/` | Protocol stacks (TCP/IP, UDP, CAN, Bluetooth) |
| **Device drivers** | `drivers/` | Hardware abstraction for all devices |
| **IPC** | `ipc/` | System V IPC: message queues, semaphores, shared memory |
| **Security** | `security/` | LSM framework, SELinux, AppArmor, capabilities |
| **Architecture** | `arch/` | CPU-specific: boot, MMU, interrupts, context switch |
| **Crypto** | `crypto/` | Cryptographic algorithms, hardware acceleration |
| **Block layer** | `block/` | Block I/O scheduling, bio processing |

```
Subsystem Interaction Map:

                 User Application
                      │
                      │ read("/dev/sda", buf, 4096)
                      ▼
              ┌───────────────┐
              │  System Call   │  sys_read()
              └───────┬───────┘
                      │
              ┌───────▼───────┐
              │     VFS       │  Resolves path, gets inode
              └───────┬───────┘
                      │
              ┌───────▼───────┐
              │   File System  │  ext4_read_iter()
              │   (ext4)      │
              └───────┬───────┘
                      │
              ┌───────▼───────┐
              │  Block Layer   │  Creates bio, I/O scheduling
              └───────┬───────┘
                      │
              ┌───────▼───────┐
              │  Block Driver  │  SCSI / NVMe driver
              └───────┬───────┘
                      │
              ┌───────▼───────┐
              │   Hardware     │  DMA transfer to disk
              └───────────────┘
```

---

## 4.3 Kernel Modular Design

Linux source tree organization:

```
linux/
├── arch/               ← Architecture-specific code
│   ├── arm64/          ← ARM64 implementation
│   │   ├── boot/       ← Boot code, decompressor
│   │   ├── kernel/     ← Entry, setup, SMP, signal
│   │   ├── mm/         ← MMU, page tables, TLB
│   │   └── include/    ← Headers (asm/)
│   ├── x86/            ← x86/x86_64 implementation
│   └── riscv/          ← RISC-V implementation
├── block/              ← Block I/O layer
├── crypto/             ← Cryptographic subsystem
├── drivers/            ← ALL device drivers (~60% of kernel code)
│   ├── char/           ← Character device drivers
│   ├── block/          ← Block device drivers
│   ├── net/            ← Network device drivers
│   ├── gpu/drm/        ← GPU / display drivers
│   ├── usb/            ← USB subsystem
│   ├── i2c/            ← I2C bus drivers
│   ├── spi/            ← SPI bus drivers
│   ├── of/             ← Device tree support
│   └── base/           ← Driver model core
├── fs/                 ← Filesystems + VFS
│   ├── ext4/           ← ext4 filesystem
│   ├── proc/           ← /proc pseudo-filesystem
│   └── sysfs/          ← /sys pseudo-filesystem
├── include/            ← Kernel headers
│   ├── linux/          ← Core kernel headers
│   └── uapi/           ← User-space API headers
├── init/               ← Kernel initialization
│   └── main.c          ← start_kernel() lives here
├── ipc/                ← Inter-process communication
├── kernel/             ← Core kernel
│   ├── sched/          ← Scheduler
│   ├── locking/        ← Locks, mutexes
│   ├── irq/            ← IRQ handling
│   └── time/           ← Timekeeping, timers
├── lib/                ← Kernel utility library
├── mm/                 ← Memory management
├── net/                ← Networking
│   ├── core/           ← Socket layer
│   ├── ipv4/           ← IPv4 stack
│   ├── ipv6/           ← IPv6 stack
│   └── can/            ← CAN bus protocol
├── scripts/            ← Build scripts, tools
├── security/           ← Security modules (SELinux, etc.)
├── sound/              ← Audio (ALSA/ASoC)
└── tools/              ← Userspace tools (perf, bpf)
```

Driver code dominates the kernel:

```
Code Distribution in Linux 6.x:

drivers/     ████████████████████████████████████████   ~60%
arch/        ████████████                               ~15%
fs/          ████████                                    ~8%
net/         ██████                                      ~6%
sound/       ████                                        ~4%
kernel/      ███                                         ~3%
mm/          ██                                          ~2%
others       ██                                          ~2%
```

---

## 4.4 Kernel Services and Abstractions

The kernel provides clean abstractions that hide hardware complexity:

```
Abstraction Layers:

User sees:          Kernel implements:           Hardware reality:
─────────────────────────────────────────────────────────────────
/dev/sda        →   Block device + VFS      →   SATA/NVMe controller
                    (struct block_device)        DMA, interrupts, sectors

/dev/ttyUSB0    →   TTY subsystem + USB     →   USB-to-serial chip
                    (struct tty_struct)          USB URBs, endpoints

/dev/video0     →   V4L2 framework          →   Camera sensor + ISP
                    (struct video_device)        MIPI CSI, DMA buffers

eth0            →   Network stack           →   Ethernet MAC + PHY
                    (struct net_device)          MDIO, DMA rings

/proc/cpuinfo   →   procfs pseudo-file      →   CPU registers, CPUID
                    (read handler function)      No real file on disk
```

Key kernel service APIs:

| Abstraction | Key Structures | API |
|-------------|---------------|-----|
| Process | `task_struct` | `fork()`, `exec()`, `exit()`, `wait()` |
| Virtual memory | `mm_struct`, `vm_area_struct` | `mmap()`, `brk()`, page fault handler |
| File | `file`, `inode`, `dentry` | `open()`, `read()`, `write()`, `close()` |
| Socket | `socket`, `sock` | `socket()`, `bind()`, `connect()`, `send()` |
| Device | `device`, `cdev`, `class` | `register_chrdev()`, `device_create()` |
| Timer | `timer_list`, `hrtimer` | `mod_timer()`, `hrtimer_start()` |
| Workqueue | `work_struct` | `schedule_work()`, `queue_work()` |

---

## 4.5 Kernel Execution Environment

Kernel code runs in a fundamentally different environment than user-space code. Understanding these constraints is critical.

```
Kernel Execution Contexts:

┌──────────────────────────────────────────────────────────┐
│                 PROCESS CONTEXT                           │
│                                                          │
│  When: System calls, workqueues, kernel threads          │
│  Current process: current → valid task_struct            │
│  Can sleep: YES (schedule(), mutex_lock(), kmalloc GFP)  │
│  Can access user memory: YES (copy_from_user())          │
│  Has user address space: YES (current->mm != NULL)       │
│  Preemptible: YES (unless preemption disabled)           │
│  Stack: kernel stack of current process (8-16 KB)        │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                 INTERRUPT CONTEXT                         │
│                                                          │
│  When: Hardware IRQ handlers, softirqs, tasklets         │
│  Current process: current is meaningless / stale         │
│  Can sleep: NO (fatal — will deadlock or crash)          │
│  Can access user memory: NO                              │
│  Has user address space: NO                              │
│  Preemptible: NO                                         │
│  Stack: Interrupt stack (per-CPU, separate)               │
└──────────────────────────────────────────────────────────┘
```

Kernel stack constraints:

```
Kernel Stack Layout (typical 16 KB):

High address  ┌──────────────────────┐
              │  thread_info /       │
              │  stack canary        │ ← Overflow detection
              ├──────────────────────┤
              │                      │
              │   Stack grows DOWN   │
              │         ↓            │
              │                      │
              │  Local variables     │
              │  Saved registers     │
              │  Function frames     │
              │                      │
              ├──────────────────────┤
              │  Guard page          │ ← CONFIG_VMAP_STACK
Low address   └──────────────────────┘

WARNING: Only 8-16 KB total!
- No large arrays on stack
- No deep recursion
- Use kmalloc() for large buffers
```

What you CANNOT do in the kernel:

| Forbidden | Reason | Alternative |
|-----------|--------|-------------|
| Use libc functions | No C library in kernel | Kernel provides own: `printk`, `kmalloc`, `memcpy` |
| Use floating point | FPU state not saved in kernel | Use integer math, `kernel_fpu_begin()` if must |
| Access user pointers directly | Different address space, security | `copy_from_user()`, `copy_to_user()` |
| Sleep in interrupt context | No process context to schedule | Use workqueues or threaded IRQs |
| Allocate large stack variables | Kernel stack is 8-16 KB | Use `kmalloc()`, `vzalloc()` |

---

## Kernel Source References

| File | Content |
|------|---------|
| `init/main.c` | Kernel initialization sequence |
| `include/linux/fs.h` | VFS structures: `file`, `inode`, `super_block` |
| `include/linux/sched.h` | `task_struct` definition |
| `include/linux/mm_types.h` | `mm_struct`, `vm_area_struct` |
| `include/linux/device.h` | Device model: `device`, `driver`, `bus_type` |
| `kernel/sched/core.c` | Scheduler core |

---

## Interview Questions

**Q1: What are the main subsystems of the Linux kernel?**
A: Process management (scheduling, creation, signals), memory management (virtual memory, paging, allocation), VFS (unified file operations), networking (protocol stacks), device drivers (hardware abstraction), IPC, and security (LSM/SELinux). These are in `kernel/`, `mm/`, `fs/`, `net/`, `drivers/`, `ipc/`, `security/`.

**Q2: What is the difference between process context and interrupt context?**
A: Process context has a valid `current` task, can sleep, accesses user memory, and runs in a schedulable thread. Interrupt context has no valid process, cannot sleep, cannot access user memory, and runs on a per-CPU interrupt stack. Using sleep-capable functions (like `mutex_lock` or `kmalloc(GFP_KERNEL)`) in interrupt context is a fatal bug.

**Q3: Why can't the kernel use the C standard library?**
A: The kernel runs directly on hardware with no runtime support. libc relies on system calls that the kernel itself provides — circular dependency. Instead, the kernel has its own implementations: `printk()` instead of `printf()`, `kmalloc()` instead of `malloc()`, etc.

**Q4: Why is the kernel stack so small?**
A: Every process (potentially thousands) needs its own kernel stack. A 16 KB stack × 10,000 processes = 160 MB just for stacks. Keeping stacks small saves memory. CONFIG_VMAP_STACK adds guard pages to detect overflows. Developers must use heap allocation for large buffers.

---

## Summary

- The Linux kernel has well-defined subsystems: process management, MM, VFS, networking, drivers, security
- VFS provides a unified interface — `open/read/write/close` works for files, devices, pipes, sockets
- Drivers comprise ~60% of kernel code — they bridge the abstraction gap between generic APIs and hardware
- The kernel source tree is organized by subsystem with `arch/` containing CPU-specific code
- Kernel code runs in process context (can sleep) or interrupt context (cannot sleep)
- The kernel stack is tiny (8-16 KB) — no large allocations on it
- No C library — the kernel provides its own utility functions

---

*Next: [Chapter 5 — Linux Kernel Subsystems](Chapter_05_Kernel_Subsystems.md)*
