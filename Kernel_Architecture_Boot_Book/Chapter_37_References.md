# Chapter 37: References and Resources

## Learning Goals
- Know essential kernel documentation and books
- Find key online resources for kernel development
- Understand where to find kernel source and tools
- Build a learning path for continuous kernel study

---

## 37.1 Essential Books

```
Recommended Reading List:

KERNEL INTERNALS:
┌────────────────────────────────────────────────────────────────┐
│  1. "Linux Kernel Development" — Robert Love (3rd ed.)        │
│     Best introduction. Covers all major subsystems.            │
│     Start here if new to kernel development.                  │
│                                                                │
│  2. "Understanding the Linux Kernel" — Bovet & Cesati (3rd)   │
│     Deep dive into internals. Heavy on memory management      │
│     and process scheduling. Reference book.                    │
│                                                                │
│  3. "Linux Device Drivers" — Corbet, Rubini, Kroah-Hartman    │
│     (3rd ed.) Available free at lwn.net/Kernel/LDD3/           │
│     Essential for driver development.                          │
│                                                                │
│  4. "Professional Linux Kernel Architecture" — Mauerer        │
│     Comprehensive, 1400+ pages. Covers 2.6 kernel.            │
│     Great for understanding design decisions.                  │
└────────────────────────────────────────────────────────────────┘

BOOT AND FIRMWARE:
┌────────────────────────────────────────────────────────────────┐
│  5. "Essential Linux Device Drivers" — Venkateswaran          │
│     Covers boot process, platform drivers, embedded.          │
│                                                                │
│  6. "Mastering Embedded Linux Programming" — Simmonds          │
│     Bootloaders, kernel config, root filesystems, Yocto.      │
│                                                                │
│  7. "Linux Kernel in a Nutshell" — Kroah-Hartman              │
│     Free online. Focus on building and configuring kernel.    │
└────────────────────────────────────────────────────────────────┘

ARCHITECTURE SPECIFIC:
┌────────────────────────────────────────────────────────────────┐
│  8. "ARM System Developer's Guide" — Sloss, Symes, Wright    │
│     ARM architecture fundamentals.                             │
│                                                                │
│  9. "ARM Architecture Reference Manual" (ARM ARM)              │
│     Official ARM documentation. Free from developer.arm.com.  │
│                                                                │
│ 10. "x86-64 System Programming" — AMD/Intel manuals            │
│     Intel SDM (Software Developer's Manual).                   │
└────────────────────────────────────────────────────────────────┘
```

---

## 37.2 Online Resources

```
Key Websites:

SOURCE CODE AND NAVIGATION:
  ├── Elixir (Bootlin): elixir.bootlin.com
  │   Cross-referenced kernel source browser
  │
  ├── kernel.org: www.kernel.org
  │   Official kernel releases and git repositories
  │
  └── GitHub mirror: github.com/torvalds/linux
      Convenient for browsing and searching

DOCUMENTATION:
  ├── Kernel docs: www.kernel.org/doc/html/latest/
  │   Official kernel documentation (RST format)
  │
  ├── LWN.net: lwn.net
  │   Best kernel news and in-depth articles
  │   Worth the subscription for serious developers
  │
  └── Bootlin training: bootlin.com/training/
      Free embedded Linux and kernel training materials

DEVICE TREE:
  ├── devicetree.org: www.devicetree.org
  │   Device Tree specification
  │
  └── DT bindings: kernel source Documentation/devicetree/bindings/

MAILING LISTS:
  ├── linux-kernel (LKML): lkml.org
  │   Main kernel development mailing list
  │
  ├── linux-arm-kernel: lists.infradead.org
  │   ARM-specific kernel development
  │
  └── U-Boot: lists.denx.de/pipermail/u-boot/
      U-Boot development mailing list
```

---

## 37.3 Kernel Source Documentation

```
In-Tree Documentation (Documentation/):

Key directories:
  Documentation/
  ├── admin-guide/           ← Sysadmin documentation
  │   ├── kernel-parameters.txt  ← ALL boot parameters
  │   └── sysrq.txt          ← SysRq key usage
  │
  ├── process/               ← Development process
  │   ├── coding-style.rst   ← Coding style guide
  │   ├── submitting-patches.rst  ← How to submit patches
  │   └── howto.rst           ← Getting started
  │
  ├── core-api/              ← Core kernel APIs
  │   ├── memory-allocation.rst  ← kmalloc, vmalloc, etc.
  │   ├── workqueue.rst       ← Work queue API
  │   └── printk-basics.rst   ← Logging guide
  │
  ├── driver-api/            ← Driver development
  │   ├── driver-model/       ← Bus/device/driver model
  │   ├── gpio/               ← GPIO API
  │   └── dma-buf/            ← DMA buffer sharing
  │
  ├── devicetree/            ← Device tree
  │   ├── bindings/           ← DT binding documentation
  │   └── usage-model.rst     ← How DT works
  │
  ├── arm64/                 ← ARM64 specific
  │   ├── booting.rst         ← ARM64 boot protocol
  │   ├── memory.rst          ← Memory layout
  │   └── elf_hwcaps.rst      ← CPU features
  │
  └── trace/                 ← Tracing
      ├── ftrace.rst          ← ftrace documentation
      └── events.rst          ← Trace events
```

---

## 37.4 Development Tools

```
Essential Kernel Development Tools:

BUILD:
  ├── GCC or Clang (cross-compilation for ARM)
  │   $ apt install gcc-aarch64-linux-gnu
  │
  ├── Make + Kbuild
  │   $ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
  │   $ make -j$(nproc)
  │
  └── ccache (speed up rebuilds)
      $ apt install ccache

DEBUGGING:
  ├── GDB + KGDB (kernel debugging)
  ├── crash (crash dump analysis)
  │   $ apt install crash
  ├── addr2line (address to source mapping)
  │   $ addr2line -e vmlinux 0xffffffc0081234
  ├── objdump (disassembly)
  │   $ objdump -d vmlinux | less
  └── strace (system call tracing from user space)

TRACING:
  ├── trace-cmd (ftrace frontend)
  │   $ trace-cmd record -e sched_switch
  │   $ trace-cmd report
  ├── perf (performance profiling)
  │   $ perf stat -a sleep 5
  │   $ perf record -g -- my_program
  ├── bpftrace (eBPF tracing)
  │   $ bpftrace -e 'kprobe:do_sys_open { printf("%s\n", comm); }'
  └── Perfetto (Android tracing)

SOURCE NAVIGATION:
  ├── cscope (fast symbol lookup)
  │   $ make cscope && cscope -d
  ├── ctags (tag-based navigation)
  │   $ make tags
  ├── ripgrep (fast grep alternative)
  │   $ rg 'start_kernel' init/
  └── VS Code with clangd (IDE integration)

TESTING:
  ├── QEMU (virtual machine testing)
  │   $ qemu-system-aarch64 -M virt -kernel Image -append "..."
  ├── KUnit (kernel unit testing framework)
  ├── kselftest (kernel self-tests)
  │   $ make -C tools/testing/selftests run_tests
  └── sparse (static analysis)
      $ make C=1 CHECK=sparse
```

---

## 37.5 Learning Path

```
Recommended Learning Path:

BEGINNER (0-6 months):
  1. Read "Linux Kernel Development" (Robert Love)
  2. Build a kernel from source (x86 or ARM)
  3. Write a simple module (hello world, char device)
  4. Understand boot messages (dmesg analysis)
  5. Learn device tree basics

INTERMEDIATE (6-18 months):
  1. Study a specific subsystem in depth
     (pick: drivers, memory, scheduler, or networking)
  2. Read actual kernel source code (use elixir.bootlin.com)
  3. Write a platform driver with DT binding
  4. Debug a real kernel issue (use KGDB, ftrace)
  5. Submit a patch to the kernel

ADVANCED (18+ months):
  1. Deep dive into memory management or scheduler
  2. Work on PREEMPT_RT or eBPF
  3. Contribute to kernel security (KASAN, lockdep)
  4. Understand hypervisor and virtualization (KVM)
  5. Kernel performance optimization and profiling
  6. Boot time optimization on real hardware

EMBEDDED/AUTOMOTIVE TRACK:
  1. Set up Yocto/Buildroot custom image
  2. Port Linux to a new board (BSP development)
  3. Device tree for custom hardware
  4. Boot time optimization
  5. Secure boot implementation
  6. Android kernel and AAOS customization
```

---

## 37.6 Kernel Source Files Quick Reference

```
Most Important Source Files for Boot:

FILE                                    PURPOSE
─────────────────────────────────────   ──────────────────────────
init/main.c                            start_kernel, kernel_init
arch/arm64/kernel/head.S               ARM64 entry point
arch/x86/kernel/head_64.S              x86-64 entry point
arch/arm64/kernel/setup.c              ARM64 setup_arch
arch/x86/kernel/setup.c               x86 setup_arch
mm/memblock.c                          Early boot allocator
mm/page_alloc.c                        Buddy allocator, zones
mm/slub.c                              SLUB slab allocator
kernel/sched/core.c                    Scheduler core
kernel/sched/fair.c                    CFS scheduler
kernel/sched/rt.c                      RT scheduler
kernel/irq/irqdesc.c                   IRQ descriptor management
kernel/printk/printk.c                 Logging infrastructure
drivers/of/fdt.c                       Device tree parsing
drivers/of/platform.c                  DT → platform devices
drivers/base/core.c                    Driver model core
init/do_mounts.c                       Root FS mounting
init/initramfs.c                       initramfs unpacking
include/linux/init.h                   __init, initcall macros
include/linux/sched.h                  task_struct definition
```

---

## Summary

- Essential books: "Linux Kernel Development" (intro), "Understanding the Linux Kernel" (deep), "LDD3" (drivers)
- Key online resources: elixir.bootlin.com (source), lwn.net (articles), kernel.org (official)
- In-tree Documentation/ is comprehensive — start with admin-guide/ and driver-api/
- Development tools: GCC/Clang, GDB, crash, ftrace, perf, QEMU, cscope
- Follow a structured learning path: modules → drivers → subsystem deep dive → contributions
- The 20 source files listed cover 90% of the boot path

---

*Next: [Chapter 38 — Interview Preparation](Chapter_38_Interview_Prep.md)*
