# Chapter 27: kdump and crash

## Learning Goals
- Understand kdump architecture and kexec mechanism
- Learn vmcore capture and crash tool analysis
- Master crash commands for post-mortem debugging
- Know makedumpfile filtering and remote kdump

---

## 1. kdump Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  kdump: captures memory dump when kernel panics         │
  │  Uses kexec to boot a second "crash kernel"            │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Setup (at boot):                         │            │
  │  │                                          │            │
  │  │ ┌───────────────────────────────────────┐│            │
  │  │ │ System RAM                            ││            │
  │  │ │                                       ││            │
  │  │ │ ┌─────────────────────────────────┐   ││            │
  │  │ │ │ Production Kernel               │   ││            │
  │  │ │ │ (running normally)              │   ││            │
  │  │ │ └─────────────────────────────────┘   ││            │
  │  │ │                                       ││            │
  │  │ │ ┌─────────────────┐                   ││            │
  │  │ │ │ Crash Kernel    │ ← reserved RAM   ││            │
  │  │ │ │ (pre-loaded,    │   crashkernel=256M││            │
  │  │ │ │  waiting)       │                   ││            │
  │  │ │ └─────────────────┘                   ││            │
  │  │ └───────────────────────────────────────┘│            │
  │  │                                          │            │
  │  │ On panic:                                │            │
  │  │ ┌───────────────────────────────────────┐│            │
  │  │ │ 1. Kernel panics                      ││            │
  │  │ │ 2. kexec jumps to crash kernel        ││            │
  │  │ │    (no BIOS/bootloader — direct boot) ││            │
  │  │ │ 3. Crash kernel boots in reserved RAM ││            │
  │  │ │ 4. Old kernel's memory is accessible  ││            │
  │  │ │    via /proc/vmcore                   ││            │
  │  │ │ 5. makedumpfile reads /proc/vmcore    ││            │
  │  │ │    and saves compressed vmcore to disk ││            │
  │  │ │ 6. System reboots normally            ││            │
  │  │ └───────────────────────────────────────┘│            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Configuration:                                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Kernel cmdline:                          │            │
  │  │   crashkernel=256M                       │            │
  │  │   # or: crashkernel=256M,high            │            │
  │  │   # or: crashkernel=256M@0x30000000      │            │
  │  │                                          │            │
  │  │ Kernel config:                           │            │
  │  │   CONFIG_CRASH_DUMP=y                    │            │
  │  │   CONFIG_KEXEC=y                         │            │
  │  │   CONFIG_PROC_VMCORE=y                   │            │
  │  │                                          │            │
  │  │ Userspace setup:                         │            │
  │  │   # Load crash kernel:                  │            │
  │  │   kexec -p /boot/vmlinuz \               │            │
  │  │     --initrd=/boot/initrd-kdump \        │            │
  │  │     --append="root=/dev/sda1 1 irqpoll   │            │
  │  │       maxcpus=1 reset_devices"           │            │
  │  │                                          │            │
  │  │   # Or use kdump service:               │            │
  │  │   systemctl enable kdump                 │            │
  │  │   systemctl start kdump                  │            │
  │  │                                          │            │
  │  │ makedumpfile — filter/compress vmcore:    │            │
  │  │   makedumpfile -l -d 31 /proc/vmcore \   │            │
  │  │     /var/crash/vmcore                    │            │
  │  │   # -l: lzo compression                 │            │
  │  │   # -d 31: exclude zero/cache/user pages│            │
  │  │   # → 256MB dump from 64GB system       │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. crash Tool

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  crash: interactive kernel dump analysis tool            │
  │  (like GDB for kernel core dumps)                       │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Open dump:                            │            │
  │  │ crash vmlinux /var/crash/vmcore          │            │
  │  │                                          │            │
  │  │ # Or analyze running kernel:            │            │
  │  │ crash /proc/kcore vmlinux                │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Essential commands:                                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │ bt          — backtrace of panic task    │            │
  │  │ bt -a       — backtrace ALL CPUs         │            │
  │  │ bt <pid>    — backtrace of specific task │            │
  │  │ bt -l       — with source line numbers   │            │
  │  │                                          │            │
  │  │ log         — kernel dmesg log           │            │
  │  │ dmesg       — same as log                │            │
  │  │                                          │            │
  │  │ ps          — process list at crash time │            │
  │  │ ps -m       — show memory usage          │            │
  │  │                                          │            │
  │  │ struct task_struct <addr>                │            │
  │  │              — dump struct contents      │            │
  │  │ struct -o task_struct                     │            │
  │  │              — show struct offsets        │            │
  │  │                                          │            │
  │  │ rd <addr> <count>  — read memory         │            │
  │  │ rd -S <addr>       — read as string      │            │
  │  │ rd -ss <addr>      — read as symbols     │            │
  │  │                                          │            │
  │  │ dis <func>          — disassemble        │            │
  │  │ dis <func>+0x28     — at offset          │            │
  │  │ dis -l <func>       — with source lines  │            │
  │  │                                          │            │
  │  │ sym <addr>           — address → symbol  │            │
  │  │ sym <name>           — symbol → address  │            │
  │  │                                          │            │
  │  │ files <pid>   — open files of process    │            │
  │  │ net           — network interfaces       │            │
  │  │ mount         — mounted filesystems      │            │
  │  │ vm <pid>      — virtual memory map       │            │
  │  │ kmem -s       — slab allocator summary   │            │
  │  │ dev           — device table             │            │
  │  │ irq           — interrupt summary        │            │
  │  │ runq          — run queue (who was running)│          │
  │  │ mod           — loaded modules           │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Analysis workflow:                                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │ crash> log                               │            │
  │  │ → read the panic message                 │            │
  │  │                                          │            │
  │  │ crash> bt                                │            │
  │  │ → see panic backtrace with line numbers  │            │
  │  │                                          │            │
  │  │ crash> bt -a                             │            │
  │  │ → check what all CPUs were doing         │            │
  │  │   (deadlock? all stuck on same lock?)    │            │
  │  │                                          │            │
  │  │ crash> struct my_device <addr>           │            │
  │  │ → inspect the struct that was accessed   │            │
  │  │   at crash time                          │            │
  │  │                                          │            │
  │  │ crash> dis -l my_func+0x28               │            │
  │  │ → see exact assembly at crash point      │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does kdump work and why is kexec needed?**
**A:** The fundamental challenge: when a kernel panics, you need to save memory to disk, but the kernel that controls disk I/O just crashed — you can't trust it. kdump solves this using kexec (kernel execution): (1) **At boot**: a small "crash kernel" (standard Linux kernel built with crash dump support) is pre-loaded into a reserved memory region (`crashkernel=256M` reservation). kexec loads it but doesn't execute it yet. (2) **On panic**: instead of halting, the panic handler calls `machine_kexec()`. This performs a "warm reboot" — jumps directly to the crash kernel's entry point without going through BIOS/bootloader. The original kernel's memory is preserved (not cleared). (3) **Crash kernel boots**: it runs in the reserved 256MB region, has access to the original kernel's memory via `/proc/vmcore` (an ELF file that maps the old kernel's physical memory). (4) **Dump capture**: `makedumpfile` reads `/proc/vmcore`, filters out unnecessary pages (zero pages, free pages, cache pages using level flags), compresses the rest, and writes to `/var/crash/vmcore`. (5) **Reboot**: system reboots normally. kexec is needed because you can't use the crashed kernel's I/O paths — a separate kernel provides a clean, trustworthy execution environment to perform the dump. The crash kernel runs with minimal hardware (single CPU, `irqpoll` for interrupt handling) to minimize interference.

---

## Summary

- kdump: captures memory dump on panic using crash kernel (pre-loaded via kexec)
- crashkernel=256M: reserves RAM for crash kernel at boot
- makedumpfile: filters/compresses vmcore (can reduce 64GB → 256MB with -d 31)
- crash tool: interactive post-mortem debugger (bt, log, ps, struct, dis, rd)
- Workflow: crash vmlinux vmcore → log → bt → bt -a → inspect structs → dis -l
- Essential for production server debugging — configure before the crash happens

---

[Previous: Oops and Panic ←](Chapter_26_Oops_Panic.md) | [Next: GDB and KGDB →](Chapter_28_GDB_KGDB.md)
