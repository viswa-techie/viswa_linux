# Chapter 31: Boot Flow Diagrams

## Learning Goals
- Visualize the complete boot flow with detailed diagrams
- Understand timing relationships between boot stages
- See data flow between firmware, bootloader, and kernel
- Grasp the state of hardware at each boot phase

---

## 31.1 Master Boot Flow Diagram

```
╔════════════════════════════════════════════════════════════════════╗
║           COMPLETE LINUX BOOT FLOW — POWER TO USERSPACE          ║
╠════════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  ┌──────────────────────────────────────────────────────────┐    ║
║  │  POWER ON                                                │    ║
║  │  CPU executes from reset vector (ROM/flash)              │    ║
║  └──────────────────────┬───────────────────────────────────┘    ║
║                         │                                        ║
║                         ▼                                        ║
║  ┌──────────────────────────────────────────────────────────┐    ║
║  │  FIRMWARE (UEFI / TF-A / SoC ROM)                        │    ║
║  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │    ║
║  │  │ CPU Init │→ │ RAM Init │→ │ Device   │               │    ║
║  │  │ Clocks   │  │ Training │  │ Discovery│               │    ║
║  │  └──────────┘  └──────────┘  └────┬─────┘               │    ║
║  │  State: No OS, no MMU, minimal HW                        │    ║
║  └──────────────────────────────────┬───────────────────────┘    ║
║                                     │                            ║
║                                     ▼                            ║
║  ┌──────────────────────────────────────────────────────────┐    ║
║  │  BOOTLOADER (GRUB / U-Boot / ABL)                        │    ║
║  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │    ║
║  │  │ Load DTB │→ │ Load     │→ │ Load     │               │    ║
║  │  │ /ACPI    │  │ Kernel   │  │ initramfs│               │    ║
║  │  └──────────┘  └──────────┘  └────┬─────┘               │    ║
║  │  State: RAM available, devices probed minimally          │    ║
║  └──────────────────────────────────┬───────────────────────┘    ║
║                                     │                            ║
║                                     ▼                            ║
║  ┌──────────────────────────────────────────────────────────┐    ║
║  │  KERNEL EARLY BOOT (Assembly → C)                        │    ║
║  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │    ║
║  │  │ MMU On   │→ │ Parse    │→ │ Memory   │               │    ║
║  │  │ CPU Init │  │ DT/ACPI  │  │ Setup    │               │    ║
║  │  └──────────┘  └──────────┘  └────┬─────┘               │    ║
║  │  State: Single CPU, no interrupts, no scheduler         │    ║
║  └──────────────────────────────────┬───────────────────────┘    ║
║                                     │                            ║
║                                     ▼                            ║
║  ┌──────────────────────────────────────────────────────────┐    ║
║  │  KERNEL SUBSYSTEM INIT (start_kernel)                    │    ║
║  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐            │    ║
║  │  │Sched   │ │IRQ     │ │Timer   │ │Console │            │    ║
║  │  │Init    │ │Init    │ │Init    │ │Init    │            │    ║
║  │  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘            │    ║
║  │  State: Interrupts on, scheduler ready, VFS mounted      │    ║
║  └──────────────────────────────────┬───────────────────────┘    ║
║                                     │                            ║
║                                     ▼                            ║
║  ┌──────────────────────────────────────────────────────────┐    ║
║  │  KERNEL INIT (PID 1 kernel thread)                       │    ║
║  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐            │    ║
║  │  │Drivers │ │SMP     │ │initram │ │exec    │            │    ║
║  │  │Probe   │ │Boot    │ │unpack  │ │/init   │            │    ║
║  │  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘            │    ║
║  │  State: All CPUs online, devices probed, FS available    │    ║
║  └──────────────────────────────────┬───────────────────────┘    ║
║                                     │                            ║
║                                     ▼                            ║
║  ┌──────────────────────────────────────────────────────────┐    ║
║  │  USER SPACE (systemd / Android init)                     │    ║
║  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐            │    ║
║  │  │Mount   │ │Network │ │Services│ │Login   │            │    ║
║  │  │FileSys │ │Setup   │ │Start   │ │Ready   │            │    ║
║  │  └────────┘ └────────┘ └────────┘ └────────┘            │    ║
║  │  State: System fully operational                         │    ║
║  └──────────────────────────────────────────────────────────┘    ║
╚════════════════════════════════════════════════════════════════════╝
```

---

## 31.2 CPU State During Boot

```
CPU State Transitions During Boot:

Power On ──────────────────────────────────────────► Userspace

│  RESET   │  FIRMWARE  │  BOOTLOADER  │   KERNEL   │  USER  │
│          │            │              │            │        │

x86:
│ Real Mode │ Protected  │ Protected   │ Long Mode  │ Long   │
│ 16-bit    │ Mode 32-bit│ or Long Mode│ 64-bit     │ Mode   │
│ No paging │ Paging opt │              │ Paging ON  │ Ring 3 │
│           │            │              │ Ring 0     │        │

ARM64:
│ EL3       │ EL3→EL2    │  EL2 or EL1 │   EL1      │  EL0   │
│ Secure    │ ATF running│  U-Boot     │   Kernel   │  Apps  │
│ MMU off   │ MMU varies │  MMU off    │   MMU ON   │ MMU ON │
│           │            │              │            │        │

Memory Map Evolution:
│ Physical  │ Identity   │  Identity   │ Virtual    │ Virtual│
│ only      │ mapped     │  mapped     │ Linear map │ mmap   │
│ No alloc  │ Firmware   │  Bootloader │ Buddy+Slab │ malloc │
│           │ allocator  │  allocator  │            │        │
```

---

## 31.3 start_kernel() Detailed Flow

```
start_kernel() Internal Flow:

start_kernel()
│
├─① set_task_stack_end_magic(&init_task)
├─② local_irq_disable()
├─③ boot_cpu_init()
│
├─④ setup_arch(&command_line)           ──── ARCH SPECIFIC
│   ├── Parse DTB/ACPI                       (30% of boot time)
│   ├── Discover memory
│   ├── Reserve memory regions
│   ├── Create page tables
│   └── Identify CPU features
│
├─⑤ setup_per_cpu_areas()              ──── PER-CPU DATA
│
├─⑥ trap_init()                        ──── EXCEPTION HANDLERS
│
├─⑦ mm_core_init()                     ──── MEMORY MANAGEMENT
│   ├── mem_init() (buddy system)
│   ├── kmem_cache_init() (slab)
│   └── vmalloc_init()
│
├─⑧ sched_init()                       ──── SCHEDULER
│   ├── Per-CPU run queues
│   └── Idle thread
│
├─⑨ early_irq_init()                   ──── IRQ DESCRIPTORS
├─⑩ init_IRQ()                         ──── IRQ CONTROLLERS
│
├─⑪ time_init()                        ──── TIMEKEEPING
├─⑫ softirq_init()                     ──── SOFTIRQ
│
├─⑬ local_irq_enable()                 ──── INTERRUPTS ON ←──── !
│
├─⑭ console_init()                     ──── CONSOLE
├─⑮ vfs_caches_init()                  ──── VFS + rootfs
│
├─⑯ signals_init()                     ──── SIGNALS
├─⑰ proc_root_init()                   ──── /proc
│
└─⑱ rest_init()                        ──── CREATE THREADS
    ├── kernel_thread(kernel_init)   PID 1
    ├── kernel_thread(kthreadd)      PID 2
    └── cpu_startup_entry()          PID 0 → idle
```

---

## 31.4 Memory Layout During Boot

```
Physical Memory Layout Evolution:

At Firmware Exit:
0x00000000 ┌─────────────────────┐
           │  Firmware reserved  │
0x00100000 ├─────────────────────┤
           │  (Free)             │
           │                     │
0x40000000 ├─────────────────────┤ ← Kernel loaded here (ARM64)
           │  Kernel Image       │
           │  ├── .text          │
           │  ├── .rodata        │
           │  ├── .data          │
           │  ├── .bss           │
           │  └── .init          │
0x41200000 ├─────────────────────┤
           │  Device Tree Blob   │
0x41300000 ├─────────────────────┤
           │  initramfs          │
0x42000000 ├─────────────────────┤
           │  (Free RAM)         │
           │  ...                │
0xFFFFFFFF └─────────────────────┘  ← End of 4GB (32-bit)

After Kernel Page Tables:
┌──────────────────────────────────────┐
│ Virtual Address Space (48-bit)       │
│                                      │
│ 0x0000_0000_0000_0000               │ ← User space
│     to                               │
│ 0x0000_FFFF_FFFF_FFFF               │   (not used yet)
│                                      │
│ ─── Hole (non-canonical) ───        │
│                                      │
│ 0xFFFF_0000_0000_0000               │ ← Kernel space
│ │  Linear map (PAGE_OFFSET)         │   All physical RAM
│ │  vmalloc region                   │   Vmalloc area
│ │  Kernel image (KIMAGE_VADDR)      │   .text, .data
│ │  Modules                          │   Loadable modules
│ │  Fixmap                           │   Fixed mappings
│ │  PCI I/O                          │   Device MMIO
│ └──────────────────────────────────  │
│ 0xFFFF_FFFF_FFFF_FFFF               │
└──────────────────────────────────────┘
```

---

## 31.5 Device Tree → Driver Binding Flow

```
Device Tree to Driver Execution:

DT Source (.dts):                    Compile            DTB (binary):
┌──────────────┐     dtc           ┌──────────────┐
│ uart@9000000 │  ──────────►      │  FDT Header  │
│ {            │                   │  Memory Res  │
│   compatible │                   │  Struct Block │
│    = "arm,   │                   │  Strings Blk │
│     pl011";  │                   └──────┬───────┘
│   reg = <0x9 │                          │
│    0000000   │                          │
│    0x1000>;  │         Bootloader loads to RAM
│   interrupts │                          │
│    = <0 1 4>;│                          ▼
│ };           │                  ┌───────────────────┐
└──────────────┘                  │ Kernel Boot       │
                                  │ unflatten_device  │
                                  │ _tree()           │
                                  └───────┬───────────┘
                                          │
                                          ▼
                                  ┌───────────────────┐
                                  │ device_node tree   │
                                  │ (linked list in    │
                                  │  kernel memory)    │
                                  └───────┬───────────┘
                                          │
                                          ▼
                                  ┌───────────────────┐
                                  │ of_platform_       │
                                  │ default_populate() │
                                  │ → Create           │
                                  │   platform_device  │
                                  │   for each node    │
                                  └───────┬───────────┘
                                          │
                                          ▼
                                  ┌───────────────────┐
                                  │ Driver matching:   │
                                  │ of_match_table =   │
                                  │  {"arm,pl011"}     │
                                  │ compatible matches!│
                                  │ → Call probe()     │
                                  └───────────────────┘
```

---

## 31.6 Initcall Order Diagram

```
Initcall Levels — Execution Order:

Level 0: early_initcall          ← Before most subsystems
         │
Level 1: pure_initcall           ← No dependencies
         │
Level 2: core_initcall           ← Core subsystems
         │  (driver model, kobject, bus types)
         │
Level 3: postcore_initcall       ← After core
         │
Level 4: arch_initcall           ← Architecture-specific
         │  (CPU features, IOMMU, perf)
         │
Level 5: subsys_initcall         ← Subsystem init
         │  (PCI, USB, networking, block)
         │
Level 6: fs_initcall             ← Filesystem init
         │  (ext4, tmpfs, proc, sysfs)
         │
Level 7: device_initcall         ← Device drivers (DEFAULT)
         │  (module_init → level 6)
         │  Most driver init happens here
         │
Level 8: late_initcall           ← After all drivers
         │  (cleanup, non-critical)
         │
         ▼
         All initcalls done → smp_init() → user space
```

---

## 31.7 Process Creation Timeline

```
Process Tree During Boot:

Time    PID   Name            Creator     Purpose
─────   ───   ────            ───────     ───────
t=0     0     swapper/idle    (kernel)    Init task, becomes idle
t=1     1     kernel_init     PID 0       Becomes /sbin/init
t=2     2     kthreadd        PID 0       Creates all kernel threads
t=3     3     ksoftirqd/0     PID 2       Softirq processing CPU 0
t=4     4     kworker/0:0     PID 2       Workqueue CPU 0
t=5     5     migration/0     PID 2       Task migration CPU 0
...     ...   ...             PID 2       More kernel threads
t=10    N     ksoftirqd/1     PID 2       Softirq for CPU 1
                                          (after SMP init)
...
t=20    1     systemd         (exec)      PID 1 exec'd to systemd
t=21    M     systemd-journal PID 1       Journal daemon
t=22    M+1   systemd-udevd   PID 1       Device manager
t=23    M+2   NetworkManager  PID 1       Network management
...
t=30    P     login           PID 1       Login prompt
t=31    P+1   bash            login       User shell

PID 0: idle (per-CPU, never scheduled, runs when nothing else can)
PID 1: init (only user-space process with special kernel protection)
PID 2: kthreadd (only kernel thread that creates other kernel threads)
```

---

## Interview Questions

**Q1: Draw the boot flow from power-on to user-space on ARM64.**
A: Power-on → SoC ROM loads BL1 → BL1 initializes secure world, loads BL2 → BL2 initializes DRAM, loads BL31 (runtime) + BL33 (bootloader) → U-Boot loads kernel Image + DTB + initramfs → Kernel head.S enables MMU, jumps to start_kernel() → start_kernel initializes subsystems (memory, scheduler, IRQs, VFS) → rest_init creates PID 1 + 2 → kernel_init runs initcalls, boots SMP, unpacks initramfs → exec /init (systemd) → services start → login.

**Q2: What is the state of the CPU when it enters start_kernel()?**
A: MMU is on (identity map + kernel map created by head.S). Caches enabled. Running in kernel mode (EL1 on ARM64, Ring 0 on x86). Single CPU only. Interrupts disabled. Stack set up for PID 0 (init_task). No scheduler, no allocator beyond memblock. Device tree address saved. On ARM64, exception vectors installed.

**Q3: What are initcall levels and why do they matter?**
A: Initcall levels (0-7) define the order of kernel initialization functions. Lower levels initialize first: core_initcall (level 2) sets up bus types before subsys_initcall (level 4) initializes PCI, which must happen before device_initcall (level 6) where most drivers probe. Without ordering, a driver might probe before its bus exists. `module_init()` maps to device_initcall (level 6) by default.

---

## Summary

- Boot flows through 6 phases: firmware → bootloader → early kernel → subsystem init → kernel_init → userspace
- CPU transitions from privileged firmware mode through kernel mode to user mode
- start_kernel() has 18+ major initialization steps in a specific order
- Memory evolves from physical-only to virtual with linear mapping
- Device tree flows from DTS source → binary DTB → in-kernel device nodes → platform devices → driver binding
- Initcalls execute in 8 ordered levels ensuring dependencies are satisfied
- Three special processes: PID 0 (idle), PID 1 (init), PID 2 (kthreadd)

---

*Next: [Chapter 32 — Architecture Diagrams](Chapter_32_Architecture_Diagrams.md)*
