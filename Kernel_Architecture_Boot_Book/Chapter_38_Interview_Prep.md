# Chapter 38: Interview Preparation

## Learning Goals
- Master comprehensive Q&A for kernel architecture and boot interviews
- Practice explaining complex concepts concisely
- Prepare for system design questions involving boot and kernel
- Build confidence for embedded, automotive, and Linux kernel roles

---

## 38.1 Fundamentals Questions

**Q1: What is the difference between a monolithic kernel and a microkernel?**
A: A monolithic kernel (Linux) runs all services — drivers, filesystem, networking, scheduling — in a single kernel address space with direct function calls between subsystems. Fast but a driver bug can crash the entire system. A microkernel (QNX, seL4) runs only minimal services (IPC, scheduling) in kernel space; drivers and filesystems run as user-space servers communicating via IPC. More isolation and reliability, but IPC overhead reduces performance.

**Q2: What is the role of the kernel in an operating system?**
A: The kernel provides: (1) Process management — scheduling, creation, termination. (2) Memory management — virtual memory, allocation, protection between processes. (3) Device management — driver framework, interrupt handling, I/O. (4) Filesystem — storage abstraction through VFS. (5) Networking — TCP/IP stack, socket API. (6) Security — access control, capabilities, LSM (SELinux). It runs in privileged mode with full hardware access.

**Q3: What are system calls and how do they work?**
A: System calls are the interface between user space and kernel. When a program calls `read()`, glibc places the syscall number in a register (x8 on ARM64, rax on x86) and arguments in other registers, then executes a trap instruction (`svc` on ARM64, `syscall` on x86-64). This switches to kernel mode, the kernel validates arguments, executes the requested operation, and returns the result. The context switch from user to kernel mode takes ~100-200ns.

---

## 38.2 Boot Process Questions

**Q4: Walk through the complete Linux boot process.**
A: (1) **Firmware** (UEFI/ATF): CPU starts at reset vector, initializes DRAM, discovers hardware. (2) **Bootloader** (GRUB/U-Boot): Loads kernel image, DTB/ACPI tables, and initramfs to RAM. Sets boot parameters. (3) **Kernel early boot** (assembly): Enables MMU, creates page tables, jumps to `start_kernel()`. (4) **start_kernel()**: Initializes subsystems — memory, scheduler, interrupts, timer, VFS, console. (5) **rest_init()**: Creates PID 1 (kernel_init) and PID 2 (kthreadd), boot CPU enters idle. (6) **kernel_init**: Runs initcalls (driver probing), boots secondary CPUs, unpacks initramfs, execs `/sbin/init`. (7) **User space**: init/systemd starts services until reaching default target.

**Q5: What is the purpose of initramfs?**
A: initramfs provides an early user-space environment before the real root filesystem is available. It contains: modules for storage/filesystem (NVMe, ext4), tools for complex root assembly (LVM, dm-crypt, RAID), and a `/init` script that loads drivers, assembles the root device, mounts it, and calls `switch_root`. Without initramfs, all root-path drivers must be compiled into the kernel.

**Q6: What is start_kernel() and what does it do?**
A: `start_kernel()` in `init/main.c` is the first C function called during boot. It initializes every kernel subsystem in order: `setup_arch()` (arch-specific hw discovery), `mm_core_init()` (buddy/slab allocator), `sched_init()` (scheduler + per-CPU run queues), `init_IRQ()` (interrupt controllers), `time_init()` (timekeeping), `console_init()`, `vfs_caches_init()` (VFS + rootfs), then `rest_init()` which creates the first kernel threads and enters idle.

**Q7: How does the kernel discover hardware on ARM vs x86?**
A: ARM uses **Device Tree** — a data structure (DTB) passed by the bootloader describing all hardware: CPUs, memory, buses, devices, interrupts. The kernel parses it during `setup_arch()` and creates platform devices. x86 uses **ACPI** — firmware provides standardized tables describing hardware. For PCI devices on both architectures, runtime enumeration discovers devices on the bus.

---

## 38.3 Memory Questions

**Q8: What is virtual memory and how does Linux implement it?**
A: Virtual memory gives each process its own address space, isolated from others. Linux implements it using: (1) Multi-level page tables (PGD→PUD→PMD→PTE on ARM64) mapping virtual to physical addresses. (2) The MMU translates addresses using these page tables. (3) TLB caches recent translations. (4) Page faults handle lazy allocation — pages are mapped only when first accessed. (5) Zones (DMA, DMA32, Normal) categorize physical memory by address constraints.

**Q9: Explain the buddy allocator.**
A: The buddy system manages free physical pages in power-of-two groups (order 0=4KB to order 10=4MB). **Allocation**: Find the smallest free block >= requested order. If only a larger block exists, split it recursively — an order N+1 block becomes two order N "buddies." **Freeing**: Check if the adjacent buddy is also free and the same order — if so, merge them into a larger block. Repeat up to MAX_ORDER. This balances splitting (avoiding waste) and merging (reducing fragmentation).

**Q10: What is memblock and when is it used?**
A: memblock is the early boot memory allocator, used before the buddy system is initialized. It tracks two lists: `memory` (all usable RAM from DT/ACPI) and `reserved` (kernel image, DTB, initramfs, page tables). Allocation comes from `memory - reserved`. Once `mem_init()` completes, all free memblock regions are transferred to the buddy allocator, and memblock is no longer used. It's needed because page tables, per-CPU areas, and early structures must be allocated before the page allocator exists.

---

## 38.4 Process and Scheduling Questions

**Q11: What is task_struct?**
A: `task_struct` is the kernel data structure representing a process or thread (~5KB on 64-bit). It contains: process state, PID, scheduling info (priority, policy, vruntime), memory descriptor (`mm_struct`), file descriptor table, signal handlers, credentials (UID/GID), cgroup membership, namespace pointers, and thread-local stack pointer. The kernel maintains one `task_struct` per thread, linked in a doubly-linked list.

**Q12: How does the CFS scheduler work?**
A: CFS (Completely Fair Scheduler) aims to give each task a fair share of CPU time. Each task has a `vruntime` (virtual runtime) that tracks how much CPU it has received, weighted by priority (nice value). CFS always picks the task with the lowest vruntime. Tasks are organized in a red-black tree (O(log N) operations). When a task runs, its vruntime increases; high-priority tasks' vruntime increases slower, so they run more. The scheduler achieves fairness by ensuring all vruntimes converge.

**Q13: What are the differences between spinlocks and mutexes?**
A: **Spinlock**: Busy-waits (spins) until lock is available. Used in interrupt context or when hold time is very short. Cannot sleep while holding. Disables preemption on the local CPU. **Mutex**: Sleeping lock — if locked, the waiter sleeps and is woken when released. Can only be used in process context (not interrupt handlers). Lower CPU waste for longer critical sections. Rule of thumb: use spinlock in interrupt context or for very short (<microsecond) critical sections; use mutex everywhere else.

---

## 38.5 Driver and Device Tree Questions

**Q14: How does a platform driver probe work?**
A: (1) Device tree describes hardware: `compatible = "vendor,device"`. (2) During boot, `of_platform_default_populate()` creates `platform_device` for each DT node. (3) The driver registers with `platform_driver_register()`, providing an `of_match_table` with compatible strings. (4) The bus `match()` function compares device compatible with driver table. (5) On match, the bus calls `driver.probe(pdev)`. (6) probe() maps registers (`devm_ioremap_resource`), requests IRQs, initializes hardware, and registers with subsystems.

**Q15: What is deferred probing and when does it occur?**
A: A driver's `probe()` returns `-EPROBE_DEFER` when a required resource (clock, regulator, GPIO, another driver's service) isn't available yet. The kernel puts the device on a deferred list and retries probe later — after each successful probe of another device, all deferred devices are re-tried. This handles dependency ordering without requiring explicit ordering in device tree. Common in SoCs where clock/regulator providers probe after their consumers.

**Q16: Explain device tree overlays.**
A: DT overlays (DTBO) modify or extend the base device tree at runtime without changing the original DTB. They're used for: (1) Hardware variants — apply different overlays for different boards. (2) Add-on hardware — describe pluggable peripherals. (3) Android — vendor-specific DTBO modifies the generic DTB. The overlay targets specific nodes using `fragment@N { target = <&node>; }` and adds/modifies properties. Applied by bootloader or at runtime via ConfigFS.

---

## 38.6 Interrupt and Timing Questions

**Q17: Explain the difference between top half and bottom half in interrupt handling.**
A: **Top half** (hard IRQ handler): Runs immediately when interrupt fires, with interrupts potentially disabled. Must be fast — acknowledge hardware, read status, schedule work. Cannot sleep. **Bottom half** (deferred processing): Runs later with interrupts enabled. Three mechanisms: (1) Softirq — fixed set (network, timer, scheduler), runs in interrupt context. (2) Tasklet — built on softirq, serialized per tasklet, being deprecated. (3) Workqueue — runs in process context via kernel threads, can sleep. Use workqueues for most cases.

**Q18: What is IRQ domain and why is it needed?**
A: IRQ domains provide a per-controller namespace for hardware IRQ numbers. Problem: multiple interrupt controllers (GIC, GPIO, PCIe MSI) may each have "IRQ 5" but they're different interrupts. Solution: each controller creates an IRQ domain that maps its hardware IRQ numbers to unique Linux virtual IRQ numbers. This allows hierarchical interrupt routing — a GPIO controller's domain is a child of the GIC domain. `irq_domain_create_linear()` creates the mapping; `irq_of_parse_and_map()` resolves DT interrupt references.

---

## 38.7 System Design Questions

**Q19: Design a secure boot system for an automotive ECU.**
A: Chain of trust from hardware root:
(1) **ROM bootloader** — immutable, contains OEM public key in OTP fuses. Verifies SPL signature with RSA-2048/SHA-256.
(2) **SPL** — signed by OEM. Initializes DRAM, verifies U-Boot signature.
(3) **U-Boot** — signed. Uses FIT image with signed kernel, DTB, and initramfs. Verifies all components before booting.
(4) **Kernel** — verified. Enables dm-verity for read-only root filesystem integrity. Hash tree root is signed.
(5) **OTA updates** — A/B partition scheme. New image verified before marking as active. Rollback on boot failure (bootloader counts failed attempts).
(6) **Runtime** — SELinux/AppArmor for MAC. Secure storage for keys (TEE/TrustZone). Measured boot with TPM or ARM CCA.

**Q20: Design a system to boot Linux in under 2 seconds on an embedded ARM SoC.**
A: (1) **Firmware** (200ms): Pre-trained DRAM (skip training), minimal hardware init, skip unnecessary peripherals.
(2) **Bootloader** (100ms): U-Boot Falcon mode — SPL directly loads kernel, skip full U-Boot. DMA-based eMMC reads.
(3) **Kernel** (800ms): LZ4 compression, minimal config (only necessary drivers built-in, no modules), `quiet loglevel=0 lpj=<value>`, `CONFIG_CC_OPTIMIZE_FOR_SIZE`, async probe for non-critical drivers.
(4) **No initramfs**: All root-device drivers built into kernel. Direct mount: `root=/dev/mmcblk0p2 rootfstype=squashfs`.
(5) **User space** (700ms): BusyBox init (not systemd — too heavy). Start only critical application. Defer logging, networking to background.
(6) **Alternative**: Suspend-to-RAM for subsequent boots (<500ms resume).

---

## 38.8 Scenario-Based Questions

**Q21: A kernel module causes a NULL pointer dereference during boot. How do you debug it?**
A: (1) Read the Oops output from dmesg/serial: note the IP (instruction pointer), call trace, and faulting address. (2) Use `addr2line -e vmlinux <IP>` to find the source file and line. (3) If the function is in a module, use `objdump -d module.ko` to disassemble. (4) Add `initcall_debug` to see which initcall triggered it. (5) If not reproducible, enable KASAN (`CONFIG_KASAN=y`) to catch the exact access. (6) Use KGDB with `kgdbwait` to break before the module loads, set a breakpoint on the function, and step through. (7) Check for uninitialized pointers, missing `of_match_table`, or probe ordering issues.

**Q22: An embedded device takes 45 seconds to boot. How do you diagnose and fix it?**
A: **Diagnose**: (1) Add `printk.time=1` and serial console. (2) `systemd-analyze blame` for slowest services. (3) `dmesg | grep -i "took\|delay\|timeout"` for slow operations. (4) `initcall_debug` for slow kernel init functions. **Typical findings & fixes**: Slow DHCP timeout → use static IP or defer network. Slow storage probe → check DMA configuration. Unnecessary services → mask them. Large initramfs → shrink or eliminate. Missing firmware files → include in initramfs. Slow driver probe → investigate sleep/delay calls. Console output → add `quiet`. Typically can reduce from 45s to 5-10s.

**Q23: Your kernel panics with "Unable to mount root fs". What are the causes?**
A: Common causes: (1) Wrong `root=` parameter — device name doesn't match actual device. (2) Storage driver not loaded — not built-in and not in initramfs. (3) Filesystem module not available — ext4/btrfs needs to be built-in or in initramfs. (4) Device not ready — add `rootwait` to wait for async device probe. (5) Corrupted filesystem — run `fsck` from recovery. (6) Wrong rootfstype — specify explicitly to avoid detection failure. Debug: check the lines before the panic in dmesg for what devices were found and what rootfs types were tried.

---

## 38.9 Quick-Fire Reference

```
Key Numbers to Know:

Page size:           4KB (x86, ARM64 default)
L1 cache:            ~32KB per core
L2 cache:            ~256KB-1MB per core
L3 cache:            ~2-30MB shared
Context switch:      ~1-5μs
System call:         ~100-200ns
Page fault:          ~1-10μs (minor), ~1-10ms (major/disk)
Interrupt latency:   ~1-5μs (bare metal), ~50-100μs (Linux)
task_struct size:     ~5KB
MAX_ORDER:           11 (order 0-10, max 4MB contiguous)
NR_CPUS:             Up to 8192 (configurable)
PID max:             4194304 (2^22)
Initcall levels:     8 (0=early to 7=late)
ARM64 VA bits:       48 (256TB) or 52 (4PB)
```

```
Essential Kernel APIs:

MEMORY:
  kmalloc(size, GFP_KERNEL)     # Allocate kernel memory
  kfree(ptr)                     # Free kernel memory
  alloc_pages(gfp, order)        # Allocate pages
  vmalloc(size)                  # Virtual contiguous alloc

DEVICE:
  platform_driver_register()     # Register platform driver
  devm_ioremap_resource()        # Map device registers
  devm_request_irq()             # Request interrupt
  of_match_table                 # DT matching

SYNC:
  spin_lock/unlock()             # Spinlock (no sleep)
  mutex_lock/unlock()            # Mutex (can sleep)
  wait_event/wake_up()           # Wait queue

LOGGING:
  pr_err/warn/info/debug()       # Kernel logging
  dev_err/warn/info/dbg()        # Device logging
```

---

## Summary

- Boot process questions require knowing firmware → bootloader → kernel → user space flow
- Memory questions focus on virtual memory, page tables, buddy allocator, and slab
- Driver questions center on device tree, matching, probe, and deferred probe
- Interrupt questions test understanding of top/bottom half and IRQ domains
- System design questions combine multiple concepts: secure boot, fast boot, debugging
- Scenario questions test practical debugging skills and systematic thinking
- Know key numbers: page size, context switch time, cache sizes, initcall levels

---

*This concludes the Linux Kernel Architecture & Boot Process book.*
*Return to [Master Index](00_Master_Index.md) for the complete chapter listing.*
