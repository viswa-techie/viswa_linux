# Chapter 33: Glossary and Terminology

## Learning Goals
- Master essential kernel and boot terminology
- Understand acronyms used in kernel development
- Build vocabulary for technical discussions and interviews

---

## 33.1 Core Kernel Terms

| Term | Definition |
|------|-----------|
| **Kernel** | Core of the OS that manages hardware resources and provides services to user-space programs. Runs in privileged mode. |
| **Monolithic kernel** | Kernel architecture where all services (drivers, FS, networking) run in a single address space. Linux uses this model. |
| **Microkernel** | Kernel architecture where only minimal services (IPC, scheduling) are in kernel space; drivers run as user-space servers. |
| **Loadable Kernel Module (LKM)** | Code that can be loaded into/unloaded from the kernel at runtime without reboot. Compiled as `.ko` files. |
| **System call (syscall)** | Interface between user space and kernel. User programs invoke kernel services via numbered syscalls (e.g., `read=0`, `write=1`). |
| **Context switch** | Process of saving the state of one task and restoring another. Involves saving/restoring registers, stack pointer, page tables. |
| **Preemption** | Ability of the scheduler to interrupt a running task to schedule a higher-priority one. Controlled by `CONFIG_PREEMPT`. |
| **Spinlock** | Lock where the waiting thread busy-waits (spins) until the lock is available. Used in interrupt context where sleeping is forbidden. |
| **Mutex** | Sleeping lock — if the lock is held, the waiting thread sleeps and is woken when the lock is released. Only usable in process context. |
| **Tasklet** | Deferred work mechanism that runs in softirq context. Cannot sleep. Being deprecated in favor of workqueues. |
| **Workqueue** | Deferred work mechanism that runs in process context (kernel threads). Can sleep. Preferred over tasklets. |
| **RCU (Read-Copy-Update)** | Lock-free synchronization mechanism for read-mostly data structures. Readers never block; writers create copies. |

---

## 33.2 Boot and Initialization Terms

| Term | Definition |
|------|-----------|
| **BIOS** | Basic Input/Output System — legacy firmware on x86 that initializes hardware and loads bootloader from MBR. |
| **UEFI** | Unified Extensible Firmware Interface — modern firmware standard replacing BIOS. Supports GPT, Secure Boot, EFI applications. |
| **ATF / TF-A** | ARM Trusted Firmware — reference implementation of secure world software for ARM processors (BL1, BL2, BL31, BL32). |
| **PSCI** | Power State Coordination Interface — ARM standard for CPU power management (CPU_ON, CPU_OFF, SUSPEND). |
| **Bootloader** | Program that loads the kernel into memory and transfers control. Examples: GRUB, U-Boot, ABL. |
| **GRUB** | GRand Unified Bootloader — standard bootloader for x86 Linux distributions. Supports multiple OS and filesystems. |
| **U-Boot** | Universal Boot Loader — standard bootloader for embedded ARM/MIPS systems. Supports device tree, network boot. |
| **Device Tree (DT)** | Data structure describing hardware to the kernel. Written in DTS, compiled to DTB. Used on ARM, RISC-V, PowerPC. |
| **DTB** | Device Tree Blob — compiled binary form of device tree source, passed to kernel by bootloader. |
| **DTS** | Device Tree Source — human-readable text format describing hardware components and their properties. |
| **DTBO** | Device Tree Blob Overlay — partial DT that modifies/extends the base DTB at runtime. |
| **ACPI** | Advanced Configuration and Power Interface — standard for hardware discovery on x86/ARM server platforms. |
| **initramfs** | Initial RAM filesystem — cpio archive unpacked into rootfs during boot. Contains early user-space tools and drivers. |
| **initrd** | Initial RAM Disk — older mechanism using a compressed filesystem image loaded into a ramdisk block device. |
| **initcall** | Kernel initialization function registered at a specific level (0-7). Called sequentially during `do_initcalls()`. |
| **start_kernel()** | Main kernel initialization function in `init/main.c`. Initializes all core subsystems sequentially. |
| **PID 1** | First user-space process (init/systemd). Ancestor of all processes. Cannot be killed. |

---

## 33.3 Memory Terms

| Term | Definition |
|------|-----------|
| **MMU** | Memory Management Unit — hardware that translates virtual addresses to physical addresses using page tables. |
| **TLB** | Translation Lookaside Buffer — CPU cache that stores recent virtual-to-physical address translations for fast lookup. |
| **Page table** | Hierarchical data structure mapping virtual addresses to physical addresses. ARM64: PGD→PUD→PMD→PTE (4 levels). |
| **Page** | Smallest unit of memory management. Typically 4KB. Can be 16KB or 64KB on ARM64. |
| **Huge page** | Large page (2MB or 1GB on x86-64) that reduces TLB misses for large memory regions. |
| **memblock** | Early boot memory allocator. Tracks available and reserved memory regions before buddy system is ready. |
| **Buddy system** | Page allocator that manages free pages in power-of-two groups (order 0-10). Splits/merges to reduce fragmentation. |
| **Slab allocator** | Sub-page allocator (SLAB/SLUB/SLOB) for kernel objects. kmalloc uses slab caches internally. |
| **vmalloc** | Allocates virtually contiguous memory from non-contiguous physical pages. Higher overhead than kmalloc. |
| **OOM killer** | Out-of-Memory killer — selects and kills processes when the system runs out of memory. |
| **swap** | Secondary storage used as extension of RAM. Pages moved to/from swap when memory is under pressure. |
| **DMA** | Direct Memory Access — hardware mechanism allowing devices to transfer data to/from memory without CPU intervention. |
| **IOMMU** | I/O Memory Management Unit — provides address translation and isolation for DMA from devices. |
| **CMA** | Contiguous Memory Allocator — reserves RAM for large contiguous allocations (DMA buffers, GPU). |

---

## 33.4 CPU and Architecture Terms

| Term | Definition |
|------|-----------|
| **SMP** | Symmetric Multiprocessing — all CPUs share memory and are equally capable of running any task. |
| **NUMA** | Non-Uniform Memory Access — memory access time depends on which CPU accesses which memory region. |
| **EL0-EL3** | ARM Exception Levels — EL0 (user), EL1 (kernel), EL2 (hypervisor), EL3 (secure monitor/firmware). |
| **Ring 0-3** | x86 privilege levels — Ring 0 (kernel), Ring 3 (user). Rings 1-2 typically unused in Linux. |
| **ISR** | Interrupt Service Routine — function executed in response to a hardware or software interrupt. |
| **GIC** | Generic Interrupt Controller — ARM standard interrupt controller. Versions: GICv2, GICv3, GICv4. |
| **APIC** | Advanced Programmable Interrupt Controller — x86 interrupt controller. Local APIC (per-CPU) + I/O APIC (device routing). |
| **IRQ** | Interrupt Request — signal from hardware to CPU requesting attention. |
| **NMI** | Non-Maskable Interrupt — interrupt that cannot be disabled. Used for critical errors and watchdogs. |
| **IPI** | Inter-Processor Interrupt — interrupt sent from one CPU to another. Used for TLB shootdown, rescheduling. |
| **Cache coherence** | Mechanism ensuring all CPUs see consistent memory values. MESI/MOESI protocols on x86, AMBA ACE on ARM. |
| **Memory barrier** | Instruction that enforces ordering of memory operations. `mb()`, `rmb()`, `wmb()` in Linux kernel. |

---

## 33.5 Process Management Terms

| Term | Definition |
|------|-----------|
| **Process** | Instance of a running program with its own address space, file descriptors, and credentials. |
| **Thread** | Lightweight execution unit sharing address space with other threads in the same process. |
| **task_struct** | Kernel data structure representing a process/thread. Contains all process state. ~5KB on 64-bit. |
| **CFS** | Completely Fair Scheduler — Linux default scheduler. Allocates CPU time based on virtual runtime. |
| **Run queue** | Per-CPU data structure holding runnable tasks. CFS uses red-black tree, RT uses priority bitmap. |
| **Nice value** | User-space priority (-20 to +19). Lower nice = higher priority. Maps to kernel weight. |
| **Zombie** | Process that has exited but its parent hasn't called wait(). Entry remains in process table. |
| **Orphan** | Process whose parent has exited. Reparented to PID 1 (init), which reaps it. |
| **cgroup** | Control Group — mechanism to organize processes and limit resources (CPU, memory, I/O, network). |
| **namespace** | Isolation mechanism — each namespace provides separate view of system resources (PID, mount, net, user). |

---

## 33.6 Filesystem Terms

| Term | Definition |
|------|-----------|
| **VFS** | Virtual Filesystem Switch — abstraction layer providing uniform interface to all filesystem types. |
| **inode** | Index node — data structure storing file metadata (permissions, size, blocks). Uniquely identifies a file within a FS. |
| **dentry** | Directory entry — caches the mapping from filename to inode. Forms the directory tree. |
| **superblock** | Per-filesystem structure containing filesystem-level metadata (type, size, block size, state). |
| **rootfs** | Initial in-memory filesystem mounted during kernel boot. initramfs is unpacked into it. |
| **sysfs** | Virtual filesystem (`/sys`) exposing kernel objects (devices, drivers, buses) as a hierarchy. |
| **procfs** | Virtual filesystem (`/proc`) exposing process and kernel information. |
| **devtmpfs** | Automatic `/dev` filesystem — kernel creates device nodes as devices are discovered. |
| **tmpfs** | RAM-backed filesystem. Data lost on reboot. Used for /tmp, /run, and as rootfs backing. |
| **NFS** | Network File System — allows mounting remote filesystems over network. Can be used as root FS. |

---

## 33.7 Debugging and Tracing Terms

| Term | Definition |
|------|-----------|
| **dmesg** | Command to read kernel ring buffer. Shows boot messages and runtime kernel log. |
| **printk** | Kernel's printf equivalent. Stores to ring buffer and optional console output. |
| **ftrace** | Function Tracer — built-in kernel tracing framework. Traces function calls, events, latencies. |
| **perf** | Performance analysis tool — CPU profiling, hardware counters, tracepoints, sampling. |
| **eBPF** | Extended Berkeley Packet Filter — safe in-kernel virtual machine for custom tracing and networking programs. |
| **KGDB** | Kernel GNU Debugger — allows GDB to debug a running kernel over serial or network. |
| **kdump** | Kernel crash dump mechanism — boots a crash kernel to capture memory state after panic. |
| **pstore** | Persistent Store — preserves kernel logs across reboots using reserved RAM or NVRAM. |
| **KASAN** | Kernel Address Sanitizer — runtime detector for use-after-free and buffer overflow. |
| **lockdep** | Lock Dependency validator — detects potential deadlocks by tracking lock ordering. |
| **Oops** | Kernel error report — indicates a bug (NULL pointer, invalid memory access). System may continue. |
| **Panic** | Fatal kernel error — system cannot continue. Triggers reboot or halt. |
| **SysRq** | Magic System Request — emergency keyboard shortcuts for debugging (reboot, sync, show tasks). |

---

## 33.8 Common Acronyms

| Acronym | Full Form |
|---------|-----------|
| AOSP | Android Open Source Project |
| AAOS | Android Automotive Operating System |
| BSP | Board Support Package |
| DMA | Direct Memory Access |
| DRM | Direct Rendering Manager |
| ELF | Executable and Linkable Format |
| FDT | Flattened Device Tree |
| GPIO | General Purpose Input/Output |
| HAL | Hardware Abstraction Layer |
| IOMMU | I/O Memory Management Unit |
| KVM | Kernel-based Virtual Machine |
| LKM | Loadable Kernel Module |
| LSM | Linux Security Module |
| PCI/PCIe | Peripheral Component Interconnect (Express) |
| PMU | Performance Monitoring Unit |
| SCMI | System Control and Management Interface |
| SELinux | Security-Enhanced Linux |
| SoC | System on Chip |
| SPL | Secondary Program Loader |
| TSC | Time Stamp Counter (x86) |

---

## Summary

- This glossary covers 100+ terms across kernel architecture, boot, memory, CPU, processes, filesystems, and debugging
- Understanding these terms is essential for kernel development, driver writing, and technical interviews
- Terms are organized by domain for quick reference during study or work

---

*Next: [Chapter 34 — Embedded Systems Boot](Chapter_34_Embedded_Boot.md)*
