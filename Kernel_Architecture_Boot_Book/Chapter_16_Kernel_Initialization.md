# Chapter 16: Kernel Initialization Process

## Learning Goals
- Trace the kernel entry point from bootloader handoff
- Understand architecture-specific initialization
- Know the kernel setup routines and early memory init
- Grasp the progression from assembly to C code

---

## 16.1 Kernel Entry Point

When the bootloader jumps to the kernel, execution begins in architecture-specific assembly code.

```
Kernel Entry — From Bootloader to C:

Bootloader (U-Boot / GRUB)
    │
    │ Jump to kernel entry point
    │ (x0 = DTB address on ARM64)
    ▼
┌────────────────────────────────────────────────────┐
│  arch/arm64/kernel/head.S                          │
│  _head:  (absolute entry point)                    │
│                                                    │
│  1. Validate CPU state (EL2/EL1, endianness)       │
│  2. Create initial page tables (identity map)       │
│  3. Enable MMU                                     │
│  4. Set up initial stack                           │
│  5. Clear BSS section                              │
│  6. Branch to start_kernel()                       │
└──────────────────────┬─────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────┐
│  init/main.c                                       │
│  start_kernel()   ← First C function!              │
│                                                    │
│  This is where ALL subsystem init happens          │
│  (scheduler, memory, interrupts, drivers, etc.)    │
└────────────────────────────────────────────────────┘
```

ARM64 head.S details:

```asm
/* arch/arm64/kernel/head.S — simplified */

    .section ".head.text", "ax"

_head:
    /* Kernel entry point — bootloader jumps here */
    b   primary_entry           /* Branch to setup code */
    .long   0                   /* Reserved */
    .quad   _kernel_offset_le   /* Image load offset from start of RAM */
    .quad   _kernel_size_le     /* Effective image size */
    .quad   _kernel_flags_le    /* Kernel flags */
    /* ... header fields per boot protocol ... */

primary_entry:
    bl  preserve_boot_args      /* Save x0-x3 (DTB addr, etc.) */
    bl  init_kernel_el          /* Determine exception level */
    bl  set_cpu_boot_mode_flag  /* Record EL2 or EL1 */

    /* Create initial page tables */
    bl  __create_page_tables    /* Identity map + kernel map */

    /* Enable MMU */
    bl  __primary_switch
        /* Inside __primary_switch: */
        /* 1. Load TTBR0/TTBR1 with page table addresses */
        /* 2. Set SCTLR_EL1.M = 1 (enable MMU) */
        /* 3. Instruction barrier */
        /* 4. Now running with virtual addresses! */

    /* Now in virtual address space */
    ldr x8, =__primary_switched
    br  x8

__primary_switched:
    /* Set up stack pointer for start_kernel */
    adr_l   x5, init_task       /* PID 0 task_struct */
    msr     sp_el0, x5          /* thread_info pointer */
    adr_l   x8, vectors
    msr     vbar_el1, x8        /* Set exception vector */

    /* Clear BSS */
    adr_l   x0, __bss_start
    mov     x1, xzr
    adr_l   x2, __bss_stop
    bl      __pi_memset          /* Zero BSS section */

    /* Jump to C code! */
    b       start_kernel         /* Never returns */
```

x86_64 entry:

```
x86_64 Kernel Entry Path:

bzImage loaded by GRUB
    │
    ▼
arch/x86/boot/header.S
    │  (16-bit real mode setup code)
    ▼
arch/x86/boot/main.c
    │  (detect memory, setup video, etc.)
    ▼
arch/x86/boot/compressed/head_64.S
    │  (decompress kernel, set up 64-bit mode)
    ▼  
arch/x86/kernel/head_64.S
    │  (set up page tables, GDT, IDT)
    ▼
x86_64_start_kernel()
    │  (arch/x86/kernel/head64.c)
    ▼
start_kernel()   ← Same function as ARM64!
```

---

## 16.2 Architecture-Specific Initialization

```c
/* What happens before start_kernel() — per architecture */

/* ARM64 (arch/arm64/kernel/head.S, setup.c): */
/* 1. Validate EL (exception level) — drop to EL1 if in EL2 */
/* 2. Create page tables:
       - Identity map: VA = PA (needed while enabling MMU)
       - Kernel map: 0xFFFF... → kernel physical location
   3. Enable MMU and caches
   4. Set up stack pointer
   5. Clear BSS
   6. Save DTB pointer (from x0)
   7. Call start_kernel()
*/

/* x86_64 (arch/x86/kernel/head_64.S, head64.c): */
/* 1. Set up initial page tables (identity map) */
/* 2. Enter 64-bit long mode */
/* 3. Set up GDT (Global Descriptor Table) */
/* 4. Set up IDT (Interrupt Descriptor Table) — early */
/* 5. Clear BSS */
/* 6. Call x86_64_start_kernel() → start_kernel() */
```

---

## 16.3 Kernel Setup Routines

```c
/* init/main.c — start_kernel() main sequence (simplified) */
asmlinkage __visible void __init start_kernel(void)
{
    /* ── Phase 1: Architecture Setup ── */
    set_task_stack_end_magic(&init_task);    /* Stack canary */
    setup_arch(&command_line);               /* ARCH-SPECIFIC setup */
    setup_command_line(command_line);         /* Save boot params */

    /* ── Phase 2: Core Infrastructure ── */
    setup_nr_cpu_ids();                      /* Count CPUs */
    setup_per_cpu_areas();                   /* Per-CPU variables */
    boot_cpu_init();                         /* Mark boot CPU */
    page_alloc_init();                       /* Register page alloc notifier */

    /* ── Phase 3: Early Parsing ── */
    parse_early_param();                     /* Handle early_param() */
    setup_log_buf(0);                        /* Kernel log buffer */
    vfs_caches_init_early();                 /* Dentry/inode hash tables */

    /* ── Phase 4: Memory System ── */
    mm_core_init();                          /* Memory management core */
    kmem_cache_init();                       /* Slab allocator */

    /* ── Phase 5: Scheduling ── */
    sched_init();                            /* Scheduler */
    preempt_disable();                       /* No preemption during init */

    /* ── Phase 6: IRQ/Timers ── */
    early_irq_init();                        /* IRQ descriptors */
    init_IRQ();                              /* Arch IRQ setup */
    tick_init();                             /* Clock tick */
    init_timers();                           /* Timer subsystem */
    hrtimers_init();                         /* High-res timers */
    time_init();                             /* Time subsystem */

    /* ── Phase 7: Console/Logging ── */
    console_init();                          /* Console drivers */

    /* ── Phase 8: Remaining Init ── */
    fork_init();                             /* fork infrastructure */
    proc_caches_init();                      /* /proc caches */
    signals_init();                          /* Signal infrastructure */

    /* ── Phase 9: Start Rest ── */
    rest_init();                             /* Creates init task, goes idle */
}
```

---

## 16.4 Early Memory Initialization

```
Early Memory Initialization Sequence:

1. memblock (earliest allocator):
   ┌─────────────────────────────────────────┐
   │  memblock — Boot-time memory manager    │
   │                                         │
   │  Tracks: memory regions (available RAM) │
   │          reserved regions (kernel, DTB) │
   │                                         │
   │  API:                                   │
   │  memblock_add(base, size)    ← Add RAM  │
   │  memblock_reserve(base, sz)  ← Reserve  │
   │  memblock_alloc(size, align) ← Allocate │
   │                                         │
   │  Source: Device tree /memory node       │
   │          or E820 table (x86)            │
   └─────────────────────────────────────────┘
             │
             │ After page tables set up
             ▼
2. Page allocator (buddy system):
   ┌─────────────────────────────────────────┐
   │  Buddy allocator initialized            │
   │  Zones set up: DMA, DMA32, Normal       │
   │  Free pages transferred from memblock   │
   │  memblock retired                       │
   └─────────────────────────────────────────┘
             │
             ▼
3. Slab allocator (SLUB):
   ┌─────────────────────────────────────────┐
   │  SLUB caches created                    │
   │  kmalloc() now available!               │
   │  Object caches: task_struct, inode, etc.│
   └─────────────────────────────────────────┘
```

---

## 16.5 Kernel Main Initialization

```
start_kernel() Complete Timeline:

Time 0.000s  ──── start_kernel() entry
             │
             ├── setup_arch()           ← Architecture init [500 µs]
             ├── setup_per_cpu_areas()  ← Per-CPU data       [100 µs]
             ├── parse_early_param()    ← Early params        [50 µs]
             ├── mm_core_init()         ← Memory management   [200 µs]
             ├── kmem_cache_init()      ← Slab allocator      [100 µs]
             ├── sched_init()           ← Scheduler           [100 µs]
             ├── early_irq_init()       ← IRQ descriptors     [50 µs]
             ├── init_IRQ()             ← IRQ controller      [100 µs]
             ├── time_init()            ← Timekeeping         [100 µs]
             ├── console_init()         ← Console output      [200 µs]
             ├── fork_init()            ← Process creation    [50 µs]
             │
Time ~0.002s ├── rest_init()            ← Start init process
             │    │
             │    ├── kernel_thread(kernel_init) ← PID 1
             │    ├── kernel_thread(kthreadd)     ← PID 2
             │    └── cpu_startup_entry()          ← Idle loop
             │
             │   kernel_init():
             │    ├── kernel_init_freeable()
             │    │    ├── do_basic_setup()
             │    │    │    ├── driver_init()      ← Driver model
             │    │    │    └── do_initcalls()      ← ALL module_init
             │    │    └── prepare_namespace()      ← Mount root
             │    └── run_init_process()            ← Exec /sbin/init
             │
Time varies  ──── User space running (init/systemd)
```

---

## Interview Questions

**Q1: What is the first function the kernel runs and where is it?**
A: The very first code is architecture-specific assembly: `_head` in `arch/arm64/kernel/head.S` (ARM64) or `startup_64` in `arch/x86/kernel/head_64.S` (x86_64). The first C function is `start_kernel()` in `init/main.c` — this is the same across all architectures.

**Q2: Why does the kernel need to set up page tables before running C code?**
A: The kernel is compiled to run at high virtual addresses (e.g., 0xFFFF800...). Without page tables and MMU, the CPU can only access physical addresses. The assembly code creates initial page tables that map both identity (VA=PA for current code) and kernel (high VA → kernel PA) mappings, then enables the MMU so the kernel's virtual addresses work.

**Q3: What is the BSS section and why must it be cleared?**
A: BSS (Block Started by Symbol) contains uninitialized global and static variables. In C, these are guaranteed to be zero-initialized. The ELF binary doesn't store zeros for BSS (saves space). The boot code must explicitly zero the BSS section before calling C code, otherwise uninitialized globals would contain garbage values.

---

## Summary

- Kernel entry starts in architecture-specific assembly: page tables, MMU, stack, BSS clearing
- `start_kernel()` in `init/main.c` is the first C function — same on all architectures
- Initialization order: arch setup → memory → scheduler → IRQ → timers → console → drivers
- Early memory uses memblock allocator, then transitions to buddy + slab (kmalloc available)
- `rest_init()` creates PID 1 (kernel_init) and PID 2 (kthreadd), then becomes idle
- `kernel_init()` runs `do_initcalls()` (all module_init) and execs /sbin/init

---

*Next: [Chapter 17 — Kernel Initialization Stages](Chapter_17_Initialization_Stages.md)*
