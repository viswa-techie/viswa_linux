# Chapter 18: Kernel Init Functions

## Learning Goals
- Deep-dive into start_kernel(), setup_arch(), rest_init(), kernel_init()
- Trace exact code flow with kernel source references
- Understand the transition from kernel initialization to user space

---

## 18.1 start_kernel() — The Heart of Kernel Initialization

```c
/* init/main.c — The most important function in the kernel */

asmlinkage __visible void __init __no_sanitize_address start_kernel(void)
{
    char *command_line;
    char *after_dashes;

    set_task_stack_end_magic(&init_task);  /* Canary on PID 0 stack */

    /* === PHASE 1: Architecture & Early Setup === */
    smp_setup_processor_id();       /* Which CPU are we on? */
    debug_objects_early_init();     /* Debug infrastructure */
    cgroup_init_early();            /* Control groups (early) */
    local_irq_disable();            /* Interrupts OFF during init */

    boot_cpu_init();                /* Mark boot CPU as online */
    page_address_init();            /* High memory mapping (32-bit) */

    pr_notice("%s", linux_banner);  /* "Linux version 6.x ..." */

    setup_arch(&command_line);      /* ARCHITECTURE-SPECIFIC init */
    setup_command_line(command_line); /* Save copies of cmdline */

    setup_nr_cpu_ids();             /* How many CPUs? */
    setup_per_cpu_areas();          /* Allocate per-CPU data */
    smp_prepare_boot_cpu();         /* Boot CPU-specific prep */

    /* === PHASE 2: Core Subsystems === */
    build_all_zonelists(NULL);      /* Memory zone lists */
    page_alloc_init();              /* Page allocator notifiers */
    parse_early_param();            /* Parse early boot params */

    setup_log_buf(0);               /* printk ring buffer */
    vfs_caches_init_early();        /* Dentry/inode hash tables */

    trap_init();                    /* Exception handlers */

    mm_core_init();                 /* Memory management core */
    /* After this: kmalloc() works! */

    sched_init();                   /* Initialize scheduler */
    preempt_disable();              /* Cannot preempt during init */

    idr_init_cache();               /* ID radix tree cache */
    rcu_init();                     /* Read-Copy-Update */

    /* === PHASE 3: Interrupts & Timing === */
    trace_init();                   /* Ftrace */
    context_tracking_init();        /* User/kernel tracking */
    early_irq_init();               /* Allocate IRQ descriptors */
    init_IRQ();                     /* Architecture IRQ setup */
    tick_init();                    /* Clock tick framework */
    rcu_init_nohz();                /* Tickless RCU */
    init_timers();                  /* Timer wheel */
    srcu_init();                    /* Sleepable RCU */
    hrtimers_init();                /* High-resolution timers */
    softirq_init();                 /* Softirq subsystem */
    timekeeping_init();             /* System clock */
    time_init();                    /* Architecture time init */

    /* === PHASE 4: Enable Scheduling & Remaining Init === */
    local_irq_enable();             /* INTERRUPTS ON! */
    kmem_cache_init_late();         /* SLUB late init */
    console_init();                 /* Console drivers */

    lockdep_init();                 /* Lock dependency checker */
    locking_selftest();             /* Lock self-tests */

    /* === PHASE 5: Process Infrastructure === */
    calibrate_delay();              /* BogoMIPS (CPU speed) */
    pid_idr_init();                 /* PID allocator */
    anon_vma_init();                /* Anonymous VMA cache */
    fork_init();                    /* fork infrastructure */
    proc_caches_init();             /* /proc caches */
    uts_ns_init();                  /* UTS namespace */
    key_init();                     /* Kernel key management */
    security_init();                /* LSM (SELinux/AppArmor) */
    vfs_caches_init();              /* VFS caches (full) */
    pagecache_init();               /* Page cache */
    signals_init();                 /* Signal queues */
    seq_file_init();                /* Sequential file */
    proc_root_init();               /* /proc filesystem */

    cpuset_init();                  /* CPU sets */
    cgroup_init();                  /* Control groups (full) */

    /* === PHASE 6: Create Init Process === */
    rest_init();                    /* → Creates PID 1, PID 2, goes idle */
    /* NEVER REACHES HERE */
}
```

---

## 18.2 setup_arch() — Architecture-Specific Initialization

```c
/* ARM64: arch/arm64/kernel/setup.c */
void __init setup_arch(char **cmdline_p)
{
    /* Step 1: Parse the Flattened Device Tree */
    setup_machine_fdt(__fdt_pointer);
    /* Extracts: machine name, memory regions, boot args from DT */

    /* Step 2: Initialize memblock allocator */
    arm64_memblock_init();
    /* Adds RAM regions from DT /memory nodes
       Reserves: kernel image, DTB, initrd */

    /* Step 3: Set up full page tables */
    paging_init();
    /* Creates kernel page tables:
       - Linear map: all physical RAM mapped contiguously
       - Kernel .text, .data, .rodata mapped
       - vmalloc region set up */

    /* Step 4: Expand device tree into in-memory structure */
    unflatten_device_tree();
    /* FDT (flat blob) → expanded tree of device_node structs
       Now can be queried: of_find_node_by_name(), etc. */

    /* Step 5: Early CPU topology */
    init_cpu_topology();
    /* Read DT for CPU clusters: big.LITTLE, DynamIQ */

    /* Step 6: Discover secondary CPUs */
    smp_init_cpus();
    /* Find all CPUs from DT, set up cpu_ops (PSCI/spin-table) */

    /* Step 7: Detect CPU features */
    cpufeature_init();
    /* Identify: SVE, MTE, Pointer Auth, BTI, etc. */

    /* Return command line for further parsing */
    *cmdline_p = boot_command_line;
}
```

---

## 18.3 rest_init() — The Birth of User Space

```c
/* init/main.c */
noinline void __ref rest_init(void)
{
    struct task_struct *tsk;
    int pid;

    /* Enable RCU scheduler */
    rcu_scheduler_starting();

    /*
     * Create the init process (PID 1)
     * This will eventually exec /sbin/init (systemd)
     */
    pid = kernel_thread(kernel_init, NULL, CLONE_FS);
    /* kernel_init runs as PID 1 */

    /*
     * Create kthreadd (PID 2)
     * Parent of ALL kernel threads
     */
    pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
    kthreadd_task = find_task_by_vpid(pid);

    /*
     * The boot CPU's PID 0 (init_task / swapper) becomes the idle task.
     * It will loop forever in cpu_idle_loop() when nothing else to run.
     */
    complete(&kthreadd_done);   /* Signal that kthreadd is ready */
    schedule_preempt_disabled();
    cpu_startup_entry(CPUHP_ONLINE);  /* → idle loop forever */
    /* NEVER RETURNS */
}
```

```
Process Tree at rest_init():

PID 0: swapper/idle (init_task)
    │
    ├── PID 1: kernel_init → /sbin/init (systemd)
    │     └── All user-space processes descend from this
    │
    └── PID 2: kthreadd
          ├── ksoftirqd/0
          ├── kworker/0:0
          ├── migration/0
          ├── rcu_preempt
          ├── kcompactd0
          └── ... (all kernel threads)
```

---

## 18.4 kernel_init() — The Path to User Space

```c
/* init/main.c */
static int __ref kernel_init(void *unused)
{
    int ret;

    /* Wait for kthreadd to be ready */
    wait_for_completion(&kthreadd_done);

    /* Run the main initialization work */
    kernel_init_freeable();

    /* Free init memory (functions marked __init) */
    async_synchronize_full();   /* Wait for async init to finish */
    system_state = SYSTEM_RUNNING;
    free_initmem();             /* Free __init sections — saves memory! */

    /* Try to run init process */
    if (ramdisk_execute_command) {
        ret = run_init_process(ramdisk_execute_command);
        if (!ret) return 0;
    }

    if (execute_command) {
        ret = run_init_process(execute_command);
        if (!ret) return 0;
    }

    /* Default init paths — try each in order */
    if (!try_to_run_init_process("/sbin/init") ||
        !try_to_run_init_process("/etc/init") ||
        !try_to_run_init_process("/bin/init") ||
        !try_to_run_init_process("/bin/sh"))
        return 0;

    panic("No working init found.");
}

static noinline void __init kernel_init_freeable(void)
{
    /* Start secondary CPUs */
    smp_init();             /* Boot all secondary CPUs */
    sched_init_smp();       /* SMP scheduler balancing */

    /* Run all subsystem/driver init functions */
    do_basic_setup();
    /* Inside do_basic_setup():
     *   driver_init()       ← Initialize driver model (kobject, sysfs)
     *   do_initcalls()      ← Run ALL initcall levels 0-9
     *                          This is where module_init() runs!
     */

    /* Open /dev/console for stdin/stdout/stderr */
    if (sys_open("/dev/console", O_RDWR, 0) < 0)
        pr_err("Warning: unable to open /dev/console\n");
    sys_dup(0);  /* stdout = stdin */
    sys_dup(0);  /* stderr = stdin */

    /* Mount root filesystem */
    prepare_namespace();
    /* Inside prepare_namespace():
     *   wait_for_device_probe()  ← Wait for root device driver
     *   mount_root()             ← Mount root filesystem
     *   devtmpfs_mount()         ← Mount /dev
     *   sys_mount(".", "/", NULL, MS_MOVE, NULL) ← Move mount
     *   sys_chroot(".")          ← Change root
     */
}
```

```
kernel_init() Complete Timeline:

kernel_init() [PID 1]
     │
     ├── wait_for_completion(kthreadd_done)
     │
     ├── kernel_init_freeable()
     │   │
     │   ├── smp_init()
     │   │   ├── Boot CPU 1 → online
     │   │   ├── Boot CPU 2 → online
     │   │   ├── Boot CPU 3 → online
     │   │   └── "SMP: Total 4 processors activated"
     │   │
     │   ├── do_basic_setup()
     │   │   ├── driver_init()        ← sysfs, kobject, device classes
     │   │   └── do_initcalls()       ← ALL init functions
     │   │       ├── Level 0: early (trace, IOMMU)
     │   │       ├── Level 1: pure
     │   │       ├── Level 2: core (driver model)
     │   │       ├── Level 3: postcore (bus types)
     │   │       ├── Level 4: arch (SMP, timers)
     │   │       ├── Level 5: subsys (networking)
     │   │       ├── Level 6: fs (procfs, sysfs)
     │   │       ├── Level 7: device (ALL drivers!) ← Biggest
     │   │       └── Level 8: late
     │   │
     │   ├── open("/dev/console")     ← fd 0, 1, 2
     │   │
     │   └── prepare_namespace()
     │       ├── wait for root device
     │       ├── mount root filesystem
     │       └── switch root
     │
     ├── free_initmem()               ← Free __init memory
     │
     └── run_init_process("/sbin/init")
         └── execve() replaces kernel_init with systemd
             PID 1 is now systemd (or another init)
```

---

## Interview Questions

**Q1: Trace the exact path from kernel entry to user space.**
A: `_head` (assembly) → `start_kernel()` → `setup_arch()` → subsystem initialization → `rest_init()` → creates PID 1 (`kernel_init`) and PID 2 (`kthreadd`) → `kernel_init()` runs `do_initcalls()` (all drivers) → mounts root filesystem → `run_init_process("/sbin/init")` → `execve()` replaces kernel code with systemd, and PID 1 is now user-space.

**Q2: What are PID 0, 1, and 2?**
A: PID 0 is `swapper`/`idle` — the initial `init_task` created statically during compile, becomes the idle loop. PID 1 is `kernel_init` → becomes `/sbin/init` (systemd) — ancestor of ALL user processes. PID 2 is `kthreadd` — the kernel thread daemon, parent of ALL kernel threads.

**Q3: What does free_initmem() do and why is it important?**
A: `free_initmem()` frees memory containing all functions marked `__init` and data marked `__initdata`. These were only needed during boot initialization. On an embedded system, this can free hundreds of KB of RAM. After `free_initmem()`, calling any `__init` function would crash (the memory is returned to the page allocator).

**Q4: What happens if the kernel can't find /sbin/init?**
A: The kernel tries in order: `/sbin/init`, `/etc/init`, `/bin/init`, `/bin/sh`. If none exist, `kernel_init()` calls `panic("No working init found.")` — the kernel halts with a panic message. This means the root filesystem is either not mounted correctly or doesn't contain an init program.

---

## Summary

- `start_kernel()`: ~50 initialization calls in precise order, from arch setup to process infrastructure
- `setup_arch()`: architecture-specific init — DT/ACPI parsing, memblock, page tables, CPU discovery
- `rest_init()`: creates PID 1 (kernel_init) and PID 2 (kthreadd), boot CPU becomes idle
- `kernel_init()`: boots secondary CPUs, runs all initcalls, mounts root, execs /sbin/init
- PID 0 = idle, PID 1 = init (systemd), PID 2 = kthreadd (kernel thread parent)
- `free_initmem()` reclaims `__init` section memory after boot completes

---

*Next: [Chapter 19 — Device Initialization](Chapter_19_Device_Initialization.md)*
