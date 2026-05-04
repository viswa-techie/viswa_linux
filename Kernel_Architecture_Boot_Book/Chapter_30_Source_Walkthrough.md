# Chapter 30: Kernel Source Code Walkthrough

## Learning Goals
- Navigate the Linux kernel source tree confidently
- Understand the key files in the boot path
- Know how to read and trace kernel code
- Understand kernel coding conventions

---

## 30.1 Kernel Source Tree Structure

```
linux/
├── arch/                   ← Architecture-specific code
│   ├── arm64/
│   │   ├── boot/           ← Boot setup, DTS files
│   │   │   ├── dts/        ← Device tree source files
│   │   │   └── Image       ← Kernel image output
│   │   ├── kernel/
│   │   │   ├── head.S      ← Entry point (assembly)
│   │   │   ├── setup.c     ← setup_arch()
│   │   │   ├── smp.c       ← SMP support
│   │   │   └── psci.c      ← PSCI CPU operations
│   │   ├── mm/
│   │   │   ├── init.c      ← Memory init
│   │   │   └── mmu.c       ← Page table setup
│   │   └── configs/        ← Default kernel configs
│   └── x86/
│       ├── boot/
│       │   ├── header.S    ← Boot header
│       │   └── compressed/ ← Decompression code
│       └── kernel/
│           ├── head_64.S   ← 64-bit entry
│           └── setup.c     ← x86 setup_arch()
│
├── block/                  ← Block I/O layer
├── crypto/                 ← Cryptographic API
├── Documentation/          ← Kernel documentation
│
├── drivers/                ← Device drivers (~60% of kernel)
│   ├── base/               ← Driver model core
│   │   ├── core.c          ← device/driver registration
│   │   ├── bus.c           ← Bus type management
│   │   └── platform.c      ← Platform device/driver
│   ├── char/               ← Character devices
│   ├── gpu/                ← GPU/DRM drivers
│   ├── irqchip/            ← Interrupt controllers (GIC)
│   ├── net/                ← Network drivers
│   ├── of/                 ← Device tree parsing
│   ├── pci/                ← PCI subsystem
│   └── tty/                ← Terminal/serial drivers
│
├── fs/                     ← Filesystem implementations
│   ├── ext4/               ← ext4 filesystem
│   ├── proc/               ← /proc filesystem
│   └── sysfs/              ← /sys filesystem
│
├── include/                ← Header files
│   ├── linux/              ← Core kernel headers
│   │   ├── init.h          ← __init, module_init macros
│   │   ├── module.h        ← Module support
│   │   ├── sched.h         ← task_struct definition
│   │   └── device.h        ← struct device
│   ├── asm-generic/        ← Generic arch headers
│   └── uapi/               ← User-space API headers
│
├── init/                   ← Kernel initialization
│   ├── main.c              ← start_kernel(), kernel_init()
│   ├── do_mounts.c         ← Root FS mounting
│   └── initramfs.c         ← initramfs unpacking
│
├── kernel/                 ← Core kernel code
│   ├── sched/              ← Scheduler
│   │   ├── core.c          ← sched_init(), schedule()
│   │   ├── fair.c          ← CFS scheduler
│   │   └── rt.c            ← Real-time scheduler
│   ├── irq/                ← IRQ subsystem
│   ├── locking/            ← Mutex, spinlock, lockdep
│   ├── printk/             ← Logging subsystem
│   └── time/               ← Timekeeping
│
├── lib/                    ← Library routines
├── mm/                     ← Memory management
│   ├── page_alloc.c        ← Buddy allocator
│   ├── slab.c / slub.c     ← Slab allocator
│   ├── vmalloc.c           ← vmalloc
│   └── memblock.c          ← Early boot allocator
│
├── net/                    ← Networking stack
├── scripts/                ← Build and helper scripts
├── security/               ← Security frameworks (SELinux)
├── sound/                  ← Audio subsystem (ALSA)
├── tools/                  ← User-space tools (perf)
│
├── Kconfig                 ← Top-level config
├── Makefile                ← Top-level makefile
└── MAINTAINERS             ← Who maintains what
```

---

## 30.2 Boot Path — Key Files to Read

```
Boot Path Source Code Walkthrough:

1. Architecture Entry Point
   ─────────────────────────
   ARM64: arch/arm64/kernel/head.S
     → _head → primary_entry → __primary_switch → start_kernel
   
   x86:   arch/x86/boot/header.S → arch/x86/kernel/head_64.S
     → startup_64 → x86_64_start_kernel → start_kernel

2. start_kernel() and Friends
   ───────────────────────────
   init/main.c:
     start_kernel()        [line ~880]
     rest_init()           [line ~680]
     kernel_init()         [line ~1450]
     kernel_init_freeable() [line ~1500]
     do_basic_setup()      [line ~1400]
     do_initcalls()        [line ~1350]

3. Architecture Setup
   ───────────────────
   arch/arm64/kernel/setup.c:
     setup_arch()          ← Parse DT, init memory, SMP prep
   
   arch/x86/kernel/setup.c:
     setup_arch()          ← Parse ACPI, setup memory map

4. Memory Initialization
   ──────────────────────
   mm/memblock.c:          ← Early allocator
   mm/page_alloc.c:        ← Zone and buddy init
   mm/slub.c:              ← Slab allocator init

5. Root Filesystem
   ────────────────
   init/do_mounts.c:       ← prepare_namespace(), mount_root()
   init/initramfs.c:       ← populate_rootfs()

6. Device Tree
   ────────────
   drivers/of/fdt.c:       ← Flattened DT parsing
   drivers/of/base.c:      ← DT node operations
   drivers/of/platform.c:  ← Create platform devices from DT
```

---

## 30.3 Reading start_kernel()

```c
/* init/main.c — start_kernel() annotated walkthrough */
void __init start_kernel(void)
{
    /* ═══ PHASE 1: Critical Early Setup ═══ */
    
    set_task_stack_end_magic(&init_task);
    /* Set stack canary for PID 0 */
    
    local_irq_disable();
    /* Interrupts OFF until we're ready */
    
    boot_cpu_init();
    /* Mark boot CPU as online/active/present */
    
    setup_arch(&command_line);
    /* ARCHITECTURE SPECIFIC:
       - Parse device tree or ACPI
       - Discover physical memory layout
       - Initialize memblock
       - Create initial page tables
       - Configure CPU features */
    
    /* ═══ PHASE 2: Core Subsystem Init ═══ */
    
    setup_log_buf(0);
    /* Allocate kernel log buffer */
    
    setup_per_cpu_areas();
    /* Allocate per-CPU data for all possible CPUs */
    
    trap_init();
    /* Install exception/fault handlers */
    
    mm_core_init();
    /* Initialize memory management:
       - mem_init() → buddy allocator
       - kmem_cache_init() → slab
       - vmalloc_init() → vmalloc */
    
    sched_init();
    /* Initialize scheduler:
       - Per-CPU run queues
       - CFS, RT, DL scheduling classes
       - Idle thread setup */
    
    /* ═══ PHASE 3: Interrupts and Time ═══ */
    
    early_irq_init();
    init_IRQ();
    /* Set up interrupt controllers (GIC/APIC) */
    
    time_init();
    /* Initialize timekeeping (TSC/arch timer) */
    
    softirq_init();
    /* Register softirq types */
    
    local_irq_enable();
    /* INTERRUPTS ON — timer ticks start */
    
    /* ═══ PHASE 4: Remaining Subsystems ═══ */
    
    console_init();
    /* Register console drivers */
    
    vfs_caches_init();
    /* VFS: dentry/inode caches, mount rootfs */
    
    signals_init();
    /* Signal infrastructure */
    
    proc_root_init();
    /* Mount /proc */
    
    /* ═══ PHASE 5: Transition to Threads ═══ */
    
    rest_init();
    /* Create PID 1 (kernel_init) and PID 2 (kthreadd)
       Boot CPU enters idle loop */
}
```

---

## 30.4 Code Navigation Tools

```bash
# Browsing kernel source

# 1. cscope — Fast symbol lookup
$ make cscope
$ cscope -d
  → Find this symbol: start_kernel
  → Find functions calling: start_kernel
  → Find functions called by: start_kernel

# 2. ctags — Symbol indexing
$ make tags
$ vim -t start_kernel    # Jump to definition

# 3. Online browsers
# https://elixir.bootlin.com/linux/latest/source
# https://git.kernel.org/

# 4. grep/ripgrep for quick searches
$ grep -rn 'start_kernel' init/
$ rg 'setup_arch' --type c

# 5. git log for history
$ git log --oneline arch/arm64/kernel/head.S
$ git log --oneline -p -S 'start_kernel' init/main.c
$ git blame init/main.c | head -50

# 6. Finding config options
$ grep -rn 'CONFIG_SMP' arch/arm64/
$ make menuconfig  # Interactive configuration
```

---

## 30.5 Kernel Coding Conventions

```c
/* Linux Kernel Coding Style (Documentation/process/coding-style.rst) */

/* 1. Indentation: Tabs (8 spaces wide) */
if (condition) {
	do_something();	    /* Tab indented */
}

/* 2. Line length: 80 columns (relaxed to 100 in some subsystems) */

/* 3. Brace style: K&R */
if (x) {
	/* ... */
} else {
	/* ... */
}

/* Functions: opening brace on new line */
static int my_function(int arg)
{
	return arg + 1;
}

/* 4. Naming: lowercase_with_underscores */
int my_variable;
void my_function(void);
struct my_structure { };

/* 5. No typedefs for structures (with few exceptions) */
struct device *dev;          /* YES */
/* device_t *dev; */         /* NO */

/* 6. Error handling: goto cleanup pattern */
static int my_init(struct device *dev)
{
	int ret;

	ret = step_one();
	if (ret)
		goto err_one;

	ret = step_two();
	if (ret)
		goto err_two;

	return 0;

err_two:
	undo_step_one();
err_one:
	return ret;
}

/* 7. Comments */
/* Multi-line comment
 * style like this
 */

/* Single line comment like this */

/* 8. __init / __exit annotations */
static int __init my_module_init(void)  /* freed after boot */
static void __exit my_module_exit(void) /* dropped if built-in */
```

---

## 30.6 Tracing Code Execution

```
How to Trace a Boot Path Through Source:

Example: "What happens when the kernel parses the device tree?"

1. Start at the entry point:
   arch/arm64/kernel/head.S
   → "preserve_boot_args" saves DTB address (x0 → x21)

2. Find setup_arch:
   arch/arm64/kernel/setup.c
   → setup_machine_fdt(__fdt_pointer)

3. Follow the call:
   arch/arm64/kernel/setup.c
   → setup_machine_fdt() calls:
     → early_init_dt_scan() in drivers/of/fdt.c
       → early_init_dt_scan_chosen()  ← Parse /chosen (bootargs)
       → early_init_dt_scan_root()    ← Parse / (sizes)
       → early_init_dt_scan_memory()  ← Parse /memory (RAM)

4. Later in boot:
   unflatten_device_tree() in drivers/of/fdt.c
   → __unflatten_device_tree()
     → Convert flat DTB → linked tree of device_node

5. Even later:
   of_platform_default_populate() in drivers/of/platform.c
   → Walk DT nodes
   → Create platform_device for each compatible node
```

---

## Kernel Source References

| File/Directory | Purpose |
|-------|---------|
| init/main.c | Core boot: start_kernel, kernel_init |
| arch/arm64/kernel/head.S | ARM64 entry point |
| arch/x86/kernel/head_64.S | x86-64 entry point |
| arch/*/kernel/setup.c | Architecture-specific setup |
| mm/memblock.c | Early boot memory allocator |
| mm/page_alloc.c | Page allocator (buddy system) |
| kernel/sched/core.c | Scheduler core |
| drivers/of/fdt.c | Device tree parsing |
| init/do_mounts.c | Root filesystem mounting |
| Documentation/process/coding-style.rst | Coding style guide |

---

## Interview Questions

**Q1: How would you trace what happens during a specific initcall?**
A: (1) Add `initcall_debug` to kernel command line — it prints each initcall with timing. (2) Find the initcall in the source by searching for `module_init(func_name)` or `subsys_initcall()`. (3) Use ftrace: `echo func_name > set_ftrace_filter && echo function_graph > current_tracer`. (4) Read the function in the source, following its call chain. (5) Use cscope or elixir.bootlin.com to find callers/callees.

**Q2: What is the `__init` annotation and why is it important?**
A: `__init` marks a function as only needed during initialization. The linker places it in the `.init.text` section. After boot completes, `free_initmem()` releases all `__init` memory back to the page allocator. This saves RAM on running systems. `__initdata` does the same for data. Calling an `__init` function after boot causes undefined behavior — the memory may have been freed and reused.

**Q3: How is the Linux kernel source tree organized?**
A: Top-level directories: `arch/` (architecture-specific code), `drivers/` (device drivers, ~60% of kernel), `fs/` (filesystems), `kernel/` (core — scheduler, locking, IRQ), `mm/` (memory management), `init/` (boot init code), `include/` (headers), `net/` (networking), `lib/` (utility functions). Architecture-specific code is cleanly separated under `arch/<ARCH>/`. The build system uses Kconfig for configuration and Kbuild for compilation.

---

## Summary

- The kernel source tree is organized by subsystem with architecture code under `arch/`
- The boot path starts at `arch/*/kernel/head.S` → `init/main.c:start_kernel()`
- `start_kernel()` has five phases: early setup → core subsystems → interrupts → remaining → threads
- Use cscope, ctags, elixir.bootlin.com, and git to navigate the source effectively
- Kernel coding convention: tabs, K&R braces, lowercase_names, goto error handling
- `__init` annotation marks code that is freed after boot to save memory

---

*Next: [Chapter 31 — Boot Flow Diagrams](Chapter_31_Boot_Flow_Diagrams.md)*
