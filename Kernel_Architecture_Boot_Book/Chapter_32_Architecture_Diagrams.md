# Chapter 32: Architecture Diagrams

## Learning Goals
- Visualize the full Linux kernel architecture with detailed diagrams
- Understand subsystem relationships and data flow
- See hardware-software boundaries clearly
- Grasp the layered architecture of the kernel

---

## 32.1 Complete Kernel Architecture

```
╔══════════════════════════════════════════════════════════════════╗
║                      USER SPACE (EL0 / Ring 3)                  ║
║                                                                 ║
║  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       ║
║  │   App    │  │   App    │  │  Shell    │  │ System   │       ║
║  │ (Firefox)│  │ (Python) │  │  (bash)   │  │ Daemon   │       ║
║  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       ║
║       │              │              │              │             ║
║  ┌────┴──────────────┴──────────────┴──────────────┴─────┐      ║
║  │              C Library (glibc / musl / bionic)         │      ║
║  │              POSIX API: open, read, write, fork, exec  │      ║
║  └──────────────────────────┬────────────────────────────┘      ║
╠═════════════════════════════╪════════════════════════════════════╣
║                             │ syscall (svc/int 0x80/syscall)    ║
╠═════════════════════════════╪════════════════════════════════════╣
║                      KERNEL SPACE (EL1 / Ring 0)                ║
║                             │                                   ║
║  ┌──────────────────────────┴────────────────────────────────┐  ║
║  │                  System Call Interface                     │  ║
║  │   sys_read, sys_write, sys_open, sys_fork, sys_ioctl      │  ║
║  └──┬────────────┬────────────┬────────────┬────────────┬────┘  ║
║     │            │            │            │            │       ║
║  ┌──▼──────┐  ┌──▼──────┐  ┌─▼────────┐ ┌─▼────────┐ ┌▼─────┐ ║
║  │ Process │  │ Memory  │  │ Virtual  │ │ Network  │ │IPC   │ ║
║  │ Mgmt    │  │ Mgmt    │  │ File Sys │ │ Stack    │ │      │ ║
║  │         │  │         │  │          │ │          │ │      │ ║
║  │ fork()  │  │ mmap()  │  │ VFS      │ │ socket() │ │pipe  │ ║
║  │schedule │  │ brk()   │  │ ├─ext4   │ │ TCP/IP   │ │signal│ ║
║  │ wait()  │  │ pgfault │  │ ├─xfs    │ │ UDP      │ │shmem │ ║
║  │ signal  │  │ OOM     │  │ ├─tmpfs  │ │ netfilter│ │msgq  │ ║
║  │ cgroup  │  │ swap    │  │ ├─proc   │ │ drivers  │ │      │ ║
║  │         │  │ huge pg │  │ └─sysfs  │ │          │ │      │ ║
║  └──┬──────┘  └──┬──────┘  └──┬───────┘ └──┬───────┘ └┬─────┘ ║
║     │            │            │            │           │       ║
║  ┌──▼────────────▼────────────▼────────────▼───────────▼─────┐ ║
║  │                    Device Driver Layer                     │ ║
║  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐  │ ║
║  │  │ Block  │ │  Char  │ │Network │ │ Input  │ │ GPU/   │  │ ║
║  │  │Drivers │ │Drivers │ │Drivers │ │Drivers │ │Display │  │ ║
║  │  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘  │ ║
║  └──────┼──────────┼──────────┼──────────┼──────────┼────────┘ ║
║         │          │          │          │          │           ║
╠═════════╪══════════╪══════════╪══════════╪══════════╪═══════════╣
║         │          │          │          │          │           ║
║  ┌──────▼──────────▼──────────▼──────────▼──────────▼────────┐ ║
║  │                      HARDWARE                             │ ║
║  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌───────┐ │ ║
║  │  │ CPU  │ │ RAM  │ │ Disk │ │ NIC  │ │ GPU  │ │ Input │ │ ║
║  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └───────┘ │ ║
║  └───────────────────────────────────────────────────────────┘ ║
╚════════════════════════════════════════════════════════════════╝
```

---

## 32.2 Memory Management Architecture

```
Memory Management Subsystem:

┌─────────────────────────────────────────────────────────┐
│                    USER SPACE                           │
│  malloc()/free() → glibc → brk()/mmap() syscalls       │
└──────────────────────────┬──────────────────────────────┘
                           │ syscall
┌──────────────────────────▼──────────────────────────────┐
│                    VMA LAYER                             │
│  struct vm_area_struct (per-process virtual regions)     │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐              │
│  │.text│ │.data│ │heap │ │mmap │ │stack│              │
│  │ r-x │ │ rw- │ │ rw- │ │ r-x │ │ rw- │              │
│  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘              │
└─────┼───────┼───────┼───────┼───────┼──────────────────┘
      │       │       │       │       │
┌─────▼───────▼───────▼───────▼───────▼──────────────────┐
│              PAGE TABLE WALKER                           │
│  PGD → P4D → PUD → PMD → PTE → Physical Page           │
│                                                         │
│  Page Fault Handler:                                    │
│  ├── Minor fault → Map existing page                    │
│  ├── Major fault → Read from disk/swap                  │
│  ├── CoW fault → Copy page, update PTE                  │
│  └── SIGSEGV → Invalid access                          │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│              PAGE ALLOCATOR (Buddy System)               │
│  Per-zone free lists: order 0 (4KB) to order 10 (4MB)   │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ZONE_DMA  │  │ZONE_DMA32│  │ZONE_NORMAL│             │
│  │ <16MB    │  │ <4GB     │  │ >4GB      │             │
│  └────┬─────┘  └────┬─────┘  └────┬──────┘             │
└───────┼──────────────┼─────────────┼────────────────────┘
        │              │             │
┌───────▼──────────────▼─────────────▼────────────────────┐
│              SLAB ALLOCATOR (SLUB)                       │
│  kmalloc-8, kmalloc-16, ..., kmalloc-8k                 │
│  task_struct, inode_cache, dentry cache, ...             │
└─────────────────────────────────────────────────────────┘
```

---

## 32.3 Interrupt Handling Architecture

```
Interrupt Flow — Hardware to Handler:

Hardware          Interrupt          CPU              Kernel
  │               Controller          │                │
  │  IRQ signal   │                   │                │
  ├──────────────►│  GIC/APIC         │                │
  │               │                   │                │
  │               │  Priority check   │                │
  │               │  CPU targeting    │                │
  │               │──────────────────►│                │
  │               │                   │ Save context   │
  │               │                   │ (registers,    │
  │               │                   │  stack, PC)    │
  │               │                   │                │
  │               │                   │───────────────►│
  │               │                   │                │ irq_enter()
  │               │                   │                │
  │               │                   │                │ Read IRQ #
  │               │                   │                │ from GIC IAR
  │               │                   │                │
  │               │                   │                │ Lookup
  │               │                   │                │ irq_desc[N]
  │               │                   │                │
  │               │                   │                │ Call handler:
  │               │                   │                │ action->
  │               │                   │                │  handler()
  │               │                   │                │
  │               │                   │                │ ACK interrupt
  │               │                   │                │ (write EOI)
  │               │                   │                │
  │               │                   │                │ irq_exit()
  │               │                   │                │ Check softirq
  │               │                   │                │
  │               │                   │◄───────────────│
  │               │                   │ Restore context│
  │               │                   │ Return to      │
  │               │                   │ interrupted    │
  │               │                   │ code           │

Top Half (hardirq):  Fast, interrupts may be disabled
  └── Acknowledge HW, read status, schedule bottom half

Bottom Half (softirq/tasklet/workqueue):  Deferred processing
  └── Process data, send to upper layers, wake tasks
```

---

## 32.4 Driver Model Architecture

```
Linux Driver Model:

                    ┌─────────────┐
                    │  kobject     │ ← Foundation object
                    │  (sysfs node)│
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼────┐  ┌───▼────┐  ┌───▼──────┐
        │  struct   │  │ struct │  │  struct   │
        │  bus_type │  │ device │  │  driver   │
        │           │  │        │  │           │
        │ platform  │  │ pdev   │  │ pdrv      │
        │ pci       │  │ pci_dev│  │ pci_driver│
        │ i2c       │  │ i2c_cl │  │ i2c_driver│
        │ spi       │  │ spi_dev│  │ spi_driver│
        └─────┬─────┘  └───┬────┘  └───┬──────┘
              │            │            │
              │     match(device, driver)│
              │            │            │
              │     ┌──────▼──────┐     │
              └────►│   Binding   │◄────┘
                    │             │
                    │ driver.probe│
                    │   (device)  │
                    └─────────────┘

/sys/bus/platform/
├── devices/
│   ├── 9000000.uart → ../../devices/platform/9000000.uart
│   └── a000000.gpio → ../../devices/platform/a000000.gpio
└── drivers/
    ├── pl011 → ../../devices/platform/9000000.uart/driver
    └── pl061 → ../../devices/platform/a000000.gpio/driver

Device-Driver matching (platform bus):
  1. of_match_table: Compare DT compatible string
  2. acpi_match_table: Compare ACPI ID
  3. id_table: Compare name
  4. name match: driver.name == device.name (legacy)
```

---

## 32.5 Filesystem Architecture

```
VFS (Virtual Filesystem Switch) Architecture:

Application: open("/etc/passwd", O_RDONLY)
     │
     ▼ syscall
┌─────────────────────────────────────────────────┐
│  System Call Layer: sys_open()                    │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│  VFS Layer                                       │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐ │
│  │ struct file │  │struct dentry│ │struct inode │ │
│  │ (open file) │  │("/passwd") │  │ (metadata)  │ │
│  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘ │
│         └────────────────┴───────────────┘       │
│  Dentry Cache (dcache) — path lookup cache       │
│  Inode Cache — metadata cache                    │
│  Page Cache — file data cache                    │
└──────────────────────┬──────────────────────────┘
                       │ file_operations->read()
┌──────────────────────▼──────────────────────────┐
│  Filesystem Layer                                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐         │
│  │   ext4   │ │   xfs    │ │  btrfs   │ ...     │
│  │  .read   │ │  .read   │ │  .read   │         │
│  │  .write  │ │  .write  │ │  .write  │         │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘         │
└───────┼────────────┼────────────┼────────────────┘
        │            │            │
┌───────▼────────────▼────────────▼────────────────┐
│  Block Layer                                      │
│  I/O Scheduler → Block Device Driver → Hardware   │
└──────────────────────────────────────────────────┘
```

---

## 32.6 Networking Architecture

```
Network Stack:

Application:  send(fd, buf, len, 0)
     │
     ▼
┌────────────────────────────────────┐
│  Socket Layer                      │
│  struct socket → struct sock       │
│  AF_INET, SOCK_STREAM             │
└────────────┬───────────────────────┘
             │
┌────────────▼───────────────────────┐
│  Transport Layer                   │
│  ┌──────┐  ┌──────┐  ┌──────┐     │
│  │ TCP  │  │ UDP  │  │ SCTP │     │
│  └──┬───┘  └──┬───┘  └──┬───┘     │
└─────┼─────────┼─────────┼─────────┘
      │         │         │
┌─────▼─────────▼─────────▼─────────┐
│  Network Layer                     │
│  ┌──────┐  ┌──────┐               │
│  │ IPv4 │  │ IPv6 │  Routing      │
│  └──┬───┘  └──┬───┘  Netfilter    │
└─────┼─────────┼───────────────────┘
      │         │
┌─────▼─────────▼───────────────────┐
│  Link Layer                        │
│  ┌──────┐  ┌──────┐  ┌──────┐     │
│  │ ARP  │  │Bridge│  │ VLAN │     │
│  └──────┘  └──────┘  └──────┘     │
└────────────┬───────────────────────┘
             │
┌────────────▼───────────────────────┐
│  Network Device Driver             │
│  ├── NAPI polling                  │
│  ├── Ring buffers (Tx/Rx)          │
│  └── DMA to/from NIC               │
└────────────┬───────────────────────┘
             │
     ────────▼────────
     │   NIC (HW)   │
     ─────────────────
```

---

## Interview Questions

**Q1: Describe the layered architecture of the Linux kernel.**
A: From top to bottom: (1) System call interface — boundary between user and kernel. (2) Core subsystems — process management, memory management, VFS, networking, IPC. (3) Driver layer — block, char, network, input, GPU drivers. (4) Hardware abstraction — architecture-specific code under `arch/`. Each layer communicates through well-defined interfaces. Drivers register with subsystems via the driver model (bus_type, device, driver).

**Q2: How does VFS abstract different filesystem implementations?**
A: VFS defines common data structures (superblock, inode, dentry, file) and operations tables (file_operations, inode_operations, super_operations). Each filesystem implements these operations. When an application calls `read()`, VFS looks up the file's operations table and calls the filesystem-specific read. The dentry cache and page cache sit between VFS and the filesystem, providing performance optimizations.

---

## Summary

- The kernel has a layered architecture: syscall interface → core subsystems → drivers → hardware
- Memory management spans VMA (per-process) → page tables → buddy allocator → slab allocator
- Interrupt handling flows from hardware through the interrupt controller to kernel handlers
- The driver model uses bus/device/driver with matching and binding via probe()
- VFS abstracts filesystems behind common data structures and operations
- The network stack follows OSI layers: socket → transport → network → link → driver

---

*Next: [Chapter 33 — Glossary and Terminology](Chapter_33_Glossary.md)*
