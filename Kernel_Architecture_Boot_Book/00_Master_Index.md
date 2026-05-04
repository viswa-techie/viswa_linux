# Linux Kernel Architecture & Boot Process — Master Index

> **A Complete Study Guide**: 38 chapters covering the entire domain of
> Linux kernel architecture and the boot process from hardware reset to user space.
> Suitable as a university-level course, kernel developer handbook,
> embedded Linux engineer reference, and interview preparation guide.

---

## Book Information

```
Subject  : Linux Kernel Architecture & Boot Process
Scope    : Linux Kernel 5.x — 6.x (x86_64, ARM64, Embedded SoCs)
Chapters : 38
Audience : Kernel developers, embedded Linux engineers, BSP engineers,
           CS students, interview candidates
Format   : Markdown with ASCII diagrams, C code, kernel source references
```

---

## Complete Boot Flow (Quick Reference)

```
Power ON
   ↓
Boot ROM (SoC-specific, reads eFuses, loads SPL)
   ↓
Firmware (BIOS / UEFI / SoC Firmware — hardware init)
   ↓
Bootloader (GRUB / U-Boot — loads kernel + DTB + initramfs)
   ↓
Kernel Image Loaded → Decompressed → Entry Point
   ↓
start_kernel()  →  setup_arch()  →  rest_init()
   ↓
Kernel Subsystems Init (MM, scheduler, interrupts, drivers)
   ↓
Mount Root Filesystem (initramfs → real rootfs)
   ↓
kernel_init() → init / systemd (PID 1)
   ↓
User Space (services, daemons, applications)
```

---

## Complete Chapter List

### Part I: Foundations (Chapters 1–2)

| #  | Chapter | Key Topics |
|----|---------|------------|
| 01 | [Foundations of OS Architecture](Chapter_01_Foundations_of_OS_Architecture.md) | What is an OS, kernel role, kernel vs user space, monolithic vs micro vs hybrid, Linux architecture |
| 02 | [History and Evolution of Linux Kernel](Chapter_02_History_and_Evolution.md) | Unix origins, Linux creation, major milestones, development model, subsystem evolution |

### Part II: Hardware & Kernel Architecture (Chapters 3–8)

| #  | Chapter | Key Topics |
|----|---------|------------|
| 03 | [Hardware Architecture Overview](Chapter_03_Hardware_Architecture.md) | CPU architecture, privilege levels, memory hardware, interrupts, multiprocessing, timers, device interfaces |
| 04 | [Linux Kernel Architecture Overview](Chapter_04_Kernel_Architecture_Overview.md) | Kernel components, core subsystems, modular design, services, execution environment |
| 05 | [Linux Kernel Subsystems](Chapter_05_Kernel_Subsystems.md) | Process mgmt, MM, device drivers, interrupts, I/O, FS, networking, security |
| 06 | [Kernel Space vs User Space](Chapter_06_Kernel_vs_User_Space.md) | Address space separation, syscall interface, kernel APIs, user-kernel communication |
| 07 | [Kernel Data Structures](Chapter_07_Kernel_Data_Structures.md) | task_struct, mm_struct, device structures, linked lists, trees, hash tables |
| 08 | [Linux Kernel Modules](Chapter_08_Kernel_Modules.md) | LKM concept, lifecycle, dependencies, loading/unloading, module parameters |

### Part III: Boot Process Fundamentals (Chapters 9–15)

| #  | Chapter | Key Topics |
|----|---------|------------|
| 09 | [Booting Fundamentals](Chapter_09_Booting_Fundamentals.md) | What is booting, boot stages, firmware init, hardware init during boot |
| 10 | [Firmware Stage](Chapter_10_Firmware_Stage.md) | BIOS architecture, UEFI architecture, firmware responsibilities, boot services |
| 11 | [Bootloaders](Chapter_11_Bootloaders.md) | Bootloader concept, role in Linux, bootloader stages, responsibilities |
| 12 | [Common Bootloaders](Chapter_12_Common_Bootloaders.md) | GRUB deep dive, U-Boot deep dive, Barebox, configuration, comparison |
| 13 | [Kernel Image Formats](Chapter_13_Kernel_Image_Formats.md) | Image structure, vmlinuz, zImage, bzImage, compression, FIT images |
| 14 | [Kernel Boot Parameters](Chapter_14_Kernel_Boot_Parameters.md) | Command line parameters, passing parameters, boot configuration |
| 15 | [Initial RAM Filesystem](Chapter_15_Initial_RAM_Filesystem.md) | initramfs, initrd, early user-space init, root filesystem mounting |

### Part IV: Kernel Initialization Deep Dive (Chapters 16–24)

| #  | Chapter | Key Topics |
|----|---------|------------|
| 16 | [Kernel Initialization Process](Chapter_16_Kernel_Initialization.md) | Entry point, arch-specific init, setup routines, early memory, main init |
| 17 | [Kernel Initialization Stages](Chapter_17_Initialization_Stages.md) | Early boot, arch init, MM init, interrupt init, device init |
| 18 | [Kernel Init Functions](Chapter_18_Kernel_Init_Functions.md) | start_kernel(), setup_arch(), rest_init(), kernel_init() deep dive |
| 19 | [Device Initialization](Chapter_19_Device_Initialization.md) | Driver init sequence, enumeration, driver-device binding, platform devices |
| 20 | [Device Tree](Chapter_20_Device_Tree.md) | DT concept, hardware description, DTS syntax, parsing, driver binding |
| 21 | [Multiprocessor Boot](Chapter_21_Multiprocessor_Boot.md) | SMP overview, CPU init, secondary CPU boot, CPU hotplug |
| 22 | [Kernel Scheduling Initialization](Chapter_22_Scheduling_Initialization.md) | Scheduler init, run queue init, per-CPU setup |
| 23 | [Interrupt Initialization](Chapter_23_Interrupt_Initialization.md) | Interrupt subsystem init, controller init, vector setup, GIC/APIC |
| 24 | [Memory Initialization](Chapter_24_Memory_Initialization.md) | Early allocator, page allocator, slab allocator, memblock, zone init |

### Part V: User Space, Boot Flow & Logging (Chapters 25–31)

| #  | Chapter | Key Topics |
|----|---------|------------|
| 25 | [Root Filesystem Initialization](Chapter_25_Root_Filesystem.md) | Root FS concept, mounting, switching root, pivot_root, switch_root |
| 26 | [User Space Initialization](Chapter_26_User_Space_Initialization.md) | init process, systemd, SysVinit, startup services, target units |
| 27 | [Kernel Boot Flow](Chapter_27_Kernel_Boot_Flow.md) | Complete flow: reset → firmware → bootloader → kernel → user space |
| 28 | [Kernel Logging During Boot](Chapter_28_Kernel_Logging.md) | printk mechanism, log buffer, dmesg, structured logging, log levels |
| 29 | [Kernel Debugging During Boot](Chapter_29_Kernel_Debugging.md) | Early printk, earlycon, boot tracing, debugging kernel panic |
| 30 | [Kernel Source Code for Boot](Chapter_30_Kernel_Source_Code.md) | Key directories, important files, code walkthrough, reading tips |
| 31 | [Important Boot Flow Diagrams](Chapter_31_Boot_Flow_Diagrams.md) | Complete flow diagrams, transitions, initialization pipeline |

### Part VI: Diagrams, Reference & Advanced (Chapters 32–38)

| #  | Chapter | Key Topics |
|----|---------|------------|
| 32 | [Architecture & Subsystem Diagrams](Chapter_32_Architecture_Diagrams.md) | Kernel arch diagram, subsystem interaction, DT hierarchy |
| 33 | [Glossary of Important Terms](Chapter_33_Glossary.md) | A-Z definitions: kernel, bootloader, firmware, rootfs, init, modules |
| 34 | [Boot in Embedded Systems](Chapter_34_Embedded_Boot.md) | Embedded Linux boot, SoC boot arch, Boot ROM, secure boot chain |
| 35 | [Boot in Other Operating Systems](Chapter_35_OS_Comparison.md) | Windows, macOS, QNX, RTOS boot comparison |
| 36 | [Boot Performance Optimization](Chapter_36_Boot_Performance.md) | Reducing boot time, fast boot techniques, profiling, parallelization |
| 37 | [Documentation and References](Chapter_37_References.md) | Kernel docs, papers, books, source references, useful links |
| 38 | [Interview Preparation](Chapter_38_Interview_Preparation.md) | Boot process questions, kernel architecture questions, debugging scenarios |

---

## How to Use This Book

```
Beginner Path:          Ch 1 → 2 → 3 → 9 → 10 → 11 → 27 → 33
Embedded Engineer:      Ch 3 → 12 → 13 → 14 → 20 → 34 → 36
Kernel Developer:       Ch 4 → 5 → 16 → 17 → 18 → 19 → 24 → 30
Interview Prep:         Ch 1 → 6 → 9 → 18 → 20 → 27 → 33 → 38
Debug Specialist:       Ch 28 → 29 → 30 → 36
Complete Study:         Ch 1 → 38 (sequential)
```

---

## Key Kernel Source Files Referenced

| File | Content |
|------|---------|
| `init/main.c` | `start_kernel()` — the kernel's main entry point |
| `arch/arm64/kernel/head.S` | ARM64 kernel entry (assembly) |
| `arch/x86/boot/header.S` | x86 boot sector and setup header |
| `arch/arm64/kernel/setup.c` | ARM64 `setup_arch()` |
| `kernel/sched/core.c` | Scheduler initialization |
| `mm/memblock.c` | Early memory allocator |
| `mm/page_alloc.c` | Page allocator initialization |
| `drivers/of/fdt.c` | Device tree (flattened) parsing |
| `drivers/base/init.c` | Driver subsystem initialization |
| `fs/init.c` | Root filesystem mounting |

---

*Begin reading: [Chapter 1 — Foundations of OS Architecture](Chapter_01_Foundations_of_OS_Architecture.md)*
