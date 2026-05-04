# Chapter 17: Kernel Initialization Stages

## Learning Goals
- Understand the ordered initialization stages from early boot to driver init
- Know what happens at each initcall level
- Trace the complete initialization pipeline

---

## 17.1 Early Boot Initialization

Before `start_kernel()` runs, the assembly boot code performs critical initialization:

```
Early Boot (Assembly Phase):

ARM64:                              x86_64:
┌──────────────────────────┐       ┌──────────────────────────┐
│ 1. Validate CPU state    │       │ 1. Enter long mode (64-bit)│
│ 2. Save DTB pointer (x0) │       │ 2. Set up GDT             │
│ 3. Create page tables    │       │ 3. Create page tables     │
│    - TTBR0: identity map │       │    - PGD for identity     │
│    - TTBR1: kernel map   │       │    - PGD for kernel       │
│ 4. Enable MMU            │       │ 4. Enable paging          │
│ 5. Setup stack            │       │ 5. Setup stack            │
│ 6. Clear BSS             │       │ 6. Clear BSS              │
│ 7. → start_kernel()      │       │ 7. → start_kernel()       │
└──────────────────────────┘       └──────────────────────────┘
```

---

## 17.2 Architecture Initialization — setup_arch()

`setup_arch()` is the architecture-specific initialization called from `start_kernel()`:

```c
/* arch/arm64/kernel/setup.c — simplified */
void __init setup_arch(char **cmdline_p)
{
    /* Parse device tree — extract memory, CPU info */
    setup_machine_fdt(__fdt_pointer);

    /* Initialize memblock from DT /memory nodes */
    arm64_memblock_init();

    /* Set up page tables (full kernel page tables) */
    paging_init();

    /* Parse DT for CPU topology */
    setup_nr_cpu_ids();

    /* ACPI or DT — hardware description */
    acpi_boot_table_init();

    /* SMP setup — find secondary CPUs */
    smp_init_cpus();

    /* Return command line for parameter parsing */
    *cmdline_p = boot_command_line;
}
```

```
setup_arch() Key Actions by Platform:

ARM64:
  ├── Parse FDT (Flattened Device Tree)
  ├── arm64_memblock_init() — register RAM regions
  ├── paging_init() — set up full page tables
  ├── unflatten_device_tree() — build in-memory DT
  ├── smp_init_cpus() — discover secondary CPUs
  └── Setup cpu_ops (spin-table or PSCI)

x86_64:
  ├── Parse E820 memory map from firmware
  ├── e820__memblock_setup() — register RAM regions
  ├── init_mem_mapping() — set up page tables
  ├── Find ACPI tables (RSDP, MADT, SRAT)
  ├── parse_smp_cfg() — discover CPUs via MADT
  └── Calculate max_pfn, high_memory
```

---

## 17.3 Memory Subsystem Initialization

```
Memory Initialization Timeline:

                start_kernel()
                     │
Phase 1: memblock    │  setup_arch() → arm64_memblock_init()
(boot allocator)     │  ├── Add memory regions from DT: memblock_add()
                     │  ├── Reserve kernel image: memblock_reserve()
                     │  ├── Reserve DTB: memblock_reserve()
                     │  └── Reserve initramfs: memblock_reserve()
                     │
Phase 2: paging      │  paging_init()
(page tables)        │  ├── Create full kernel page tables
                     │  ├── Map all physical memory (linear map)
                     │  └── Set up vmalloc region
                     │
Phase 3: zones       │  zone_sizes_init() → free_area_init()
(buddy allocator)    │  ├── Initialize zones (DMA, DMA32, Normal)
                     │  ├── Set up buddy free lists (order 0-10)
                     │  └── Transfer pages from memblock to buddy
                     │
Phase 4: slab        │  mm_core_init() → kmem_cache_init()
(object allocator)   │  ├── Bootstrap SLUB allocator
                     │  ├── Create initial caches (kmalloc-8..kmalloc-8k)
                     │  └── NOW kmalloc() works!
                     │
Phase 5: vmalloc     │  vmalloc_init()
                     │  └── vmalloc() now available
                     │
Phase 6: per-CPU     │  setup_per_cpu_areas()
                     │  └── Allocate per-CPU memory for each CPU
```

---

## 17.4 Interrupt Subsystem Initialization

```
Interrupt Init Sequence:

start_kernel()
     │
     ├── early_irq_init()
     │   └── Allocate irq_desc[] array
     │       (one descriptor per IRQ line)
     │
     ├── init_IRQ()
     │   ├── ARM64: irqchip_init() → gic_of_init()
     │   │   ├── Parse DT for interrupt controller
     │   │   ├── Map GIC registers (ioremap)
     │   │   ├── Configure distributor (GIC-D)
     │   │   ├── Configure redistributor (GIC-R)
     │   │   └── Set up IPI handling
     │   │
     │   └── x86: native_init_IRQ()
     │       ├── Initialize APIC
     │       ├── Set up IO-APIC
     │       └── Configure interrupt vectors
     │
     ├── softirq_init()
     │   └── Register default softirqs:
     │       HI_SOFTIRQ, TIMER, NET_TX, NET_RX,
     │       BLOCK, IRQ_POLL, TASKLET, SCHED, HRTIMER, RCU
     │
     └── local_irq_enable()
         └── Interrupts are now ON!
```

---

## 17.5 Device Initialization — The initcall Mechanism

The kernel uses a hierarchical initcall system to ensure proper initialization order:

```
Initcall Levels (executed in this order):

Level  Macro                    Purpose                      Examples
─────────────────────────────────────────────────────────────────────────
0      early_initcall()         Before anything else         trace, IOMMU
1      pure_initcall()          Pure software init           IRQ domains
2      core_initcall()          Core subsystems              driver model, sysfs
3      postcore_initcall()      After core                   bus types (PCI, platform)
4      arch_initcall()          Architecture-specific        SMP, timers
5      subsys_initcall()        Subsystem init               networking, block layer
6      fs_initcall()            Filesystems                  procfs, sysfs, devtmpfs
7      rootfs_initcall()        Root FS                      populate rootfs
       ──── do_initcall_sync() ────
8      device_initcall()        Device drivers               Most drivers (module_init)
       ──── do_initcall_sync() ────
9      late_initcall()          Last                         After all drivers

module_init() = device_initcall() for built-in modules
```

```c
/* How initcalls work — placing function pointers in special sections */

/* Declaring an initcall */
static int __init my_init(void) { ... }
core_initcall(my_init);  /* Places pointer in .initcall2.init section */

/* The linker script creates an array of function pointers: */
/*
 * .initcall.init section layout:
 *   .initcall0.init  ← early_initcall pointers
 *   .initcall1.init  ← pure_initcall pointers
 *   .initcall2.init  ← core_initcall pointers
 *   ...
 *   .initcall7.init  ← device_initcall (module_init) pointers
 *   .initcall8.init  ← late_initcall pointers
 */

/* do_initcalls() walks through all levels sequentially */
static void __init do_initcalls(void)
{
    for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++) {
        for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
            do_one_initcall(*fn);  /* Call each init function */
    }
}
```

```bash
# Debug initcalls: add to kernel command line
initcall_debug

# Output in dmesg:
# [   0.123456] calling  pci_init+0x0/0x28 @ 1
# [   0.123789] initcall pci_init+0x0/0x28 returned 0 after 333 usecs
# [   0.124000] calling  my_driver_init+0x0/0x3c @ 1
# [   0.124500] initcall my_driver_init+0x0/0x3c returned 0 after 500 usecs
```

---

## Interview Questions

**Q1: What is the initcall mechanism and why does it exist?**
A: Initcalls are prioritized function pointers placed in linker sections. They're called in order (level 0-9) during boot, ensuring dependencies are met: core subsystems (level 2) initialize before bus types (level 3), which initialize before device drivers (level 7). Without this ordering, drivers might try to use subsystems that aren't ready.

**Q2: What is the difference between early_initcall and module_init?**
A: `early_initcall()` runs at level 0 — before almost any subsystem. Used for critical early setup (tracing, IOMMU). `module_init()` maps to `device_initcall()` at level 7 — runs after all infrastructure is ready. For modules compiled as .ko, `module_init()` is called at `insmod` time instead.

**Q3: How does setup_arch() differ between architectures?**
A: Each architecture implements `setup_arch()` to handle its unique hardware: ARM64 parses the device tree for memory and CPUs; x86 reads E820 memory map from UEFI/BIOS and ACPI tables for device discovery. Both set up page tables and identify CPUs, but the mechanisms are entirely different.

---

## Summary

- Early boot (assembly) sets up page tables, MMU, stack, and BSS — all before C code runs
- `setup_arch()` handles architecture-specific init: DT/ACPI parsing, memory regions, CPU discovery
- Memory initialization progresses: memblock → page tables → buddy → slab → vmalloc
- Interrupt init: allocate descriptors → initialize controller (GIC/APIC) → enable softirqs → enable IRQs
- Initcalls provide ordered initialization (levels 0-9) ensuring subsystems init before drivers
- `initcall_debug` boot parameter shows every initcall with timing information

---

*Next: [Chapter 18 — Kernel Init Functions](Chapter_18_Kernel_Init_Functions.md)*
