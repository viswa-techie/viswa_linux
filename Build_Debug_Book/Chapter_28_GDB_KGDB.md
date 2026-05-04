# Chapter 28: GDB and KGDB

## Learning Goals
- Understand kernel debugging with GDB over KGDB
- Learn QEMU+GDB setup for kernel development
- Master kernel-specific GDB scripts and helpers
- Know /proc/kcore for live kernel inspection

---

## 1. KGDB — Kernel GDB Stub

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  KGDB: GDB stub built into the kernel                   │
  │  Allows remote GDB debugging of a live kernel           │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Architecture:                            │            │
  │  │                                          │            │
  │  │ ┌──────────┐   serial/    ┌──────────┐  │            │
  │  │ │ Dev Host │   network    │ Target   │  │            │
  │  │ │          │ ←──────────→ │          │  │            │
  │  │ │ GDB      │              │ KGDB     │  │            │
  │  │ │ (with    │              │ (in-kernel│  │            │
  │  │ │  vmlinux)│              │  stub)   │  │            │
  │  │ └──────────┘              └──────────┘  │            │
  │  │                                          │            │
  │  │ Setup (target):                          │            │
  │  │ CONFIG_KGDB=y                            │            │
  │  │ CONFIG_KGDB_SERIAL_CONSOLE=y             │            │
  │  │ CONFIG_KGDB_KDB=y  (optional: in-kernel  │            │
  │  │                      debugger shell)     │            │
  │  │ CONFIG_DEBUG_INFO=y                      │            │
  │  │ CONFIG_FRAME_POINTER=y                   │            │
  │  │                                          │            │
  │  │ Kernel cmdline:                          │            │
  │  │ kgdboc=ttyS0,115200   (serial port)     │            │
  │  │ kgdbwait               (break at boot)  │            │
  │  │                                          │            │
  │  │ Trigger break:                           │            │
  │  │ echo g > /proc/sysrq-trigger             │            │
  │  │ (or SysRq+G on keyboard)                │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  GDB session (host):                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ gdb vmlinux                              │            │
  │  │ (gdb) target remote /dev/ttyUSB0         │            │
  │  │ # or: target remote hostname:port        │            │
  │  │                                          │            │
  │  │ (gdb) b do_sys_openat2                   │            │
  │  │ Breakpoint 1 at 0xffffffff812345:        │            │
  │  │   file fs/open.c, line 1234              │            │
  │  │                                          │            │
  │  │ (gdb) c                                  │            │
  │  │ Continuing.                              │            │
  │  │ → Target halts when breakpoint hit       │            │
  │  │   (ENTIRE system frozen)                │            │
  │  │                                          │            │
  │  │ (gdb) bt                                 │            │
  │  │ (gdb) p current->comm                    │            │
  │  │ (gdb) p *(struct inode *)0xffff8880...   │            │
  │  │ (gdb) n    # next line                  │            │
  │  │ (gdb) s    # step into                  │            │
  │  │ (gdb) c    # continue                   │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. QEMU + GDB (Best for Development)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  QEMU provides built-in GDB server — easiest setup     │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Start QEMU with GDB server:          │            │
  │  │ qemu-system-x86_64 \                    │            │
  │  │   -kernel arch/x86/boot/bzImage \        │            │
  │  │   -initrd initramfs.cpio.gz \            │            │
  │  │   -append "nokaslr console=ttyS0" \      │            │
  │  │   -nographic \                           │            │
  │  │   -s  ← GDB server on :1234            │            │
  │  │   -S  ← freeze at startup              │            │
  │  │                                          │            │
  │  │ # In another terminal:                  │            │
  │  │ gdb vmlinux                              │            │
  │  │ (gdb) target remote :1234                │            │
  │  │ (gdb) hbreak start_kernel               │            │
  │  │ # hbreak = hardware breakpoint          │            │
  │  │ # (needed for kernel code)              │            │
  │  │ (gdb) c                                  │            │
  │  │ → breaks at start_kernel                │            │
  │  │                                          │            │
  │  │ Key: nokaslr disables address           │            │
  │  │   randomization so symbols match        │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Kernel GDB helper scripts:                              │
  │  ┌──────────────────────────────────────────┐            │
  │  │ CONFIG_GDB_SCRIPTS=y                     │            │
  │  │ → creates scripts/gdb/ with helpers     │            │
  │  │                                          │            │
  │  │ # Add to ~/.gdbinit:                    │            │
  │  │ add-auto-load-safe-path /path/to/kernel  │            │
  │  │                                          │            │
  │  │ (gdb) lx-dmesg        — show dmesg     │            │
  │  │ (gdb) lx-lsmod        — list modules   │            │
  │  │ (gdb) lx-ps           — process list   │            │
  │  │ (gdb) lx-symbols      — load module syms│            │
  │  │ (gdb) lx-cmdline      — kernel cmdline  │            │
  │  │ (gdb) lx-cpus         — CPU state       │            │
  │  │ (gdb) lx-timerlist    — active timers   │            │
  │  │ (gdb) p $lx_current() — current task    │            │
  │  │ (gdb) lx-list-check <addr> — list check │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  /proc/kcore — live kernel memory:                       │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Read-only GDB on running kernel:      │            │
  │  │ gdb vmlinux /proc/kcore                  │            │
  │  │ (gdb) p jiffies    — current jiffies    │            │
  │  │ (gdb) p init_task  — PID 0 task struct  │            │
  │  │                                          │            │
  │  │ # No breakpoints, no stepping           │            │
  │  │ # Just inspection of kernel state       │            │
  │  │ # Great for looking at data structures  │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: What are the limitations of using GDB/KGDB for kernel debugging compared to tracing?**
**A:** (1) **System-wide halt**: when KGDB hits a breakpoint, the ENTIRE system freezes — all CPUs are stopped. This makes it impossible to debug timing-sensitive issues, real-time workloads, or network services (connections time out). Ftrace/eBPF trace without stopping the system. (2) **Not production-safe**: you'd never attach KGDB to a production server — the debug session halts all workloads. Tracing tools (ftrace, eBPF, perf) are designed for production use. (3) **Single-point observation**: GDB gives you a snapshot at one point in time. Tracing gives you a continuous stream of events over time — much better for understanding patterns, races, and intermittent issues. (4) **Scalability**: you can't reasonably single-step through millions of function calls; tracing records them all. (5) **Setup complexity**: KGDB needs serial/network connection between two machines, or QEMU. Tracing just needs the kernel (ftrace is built-in, eBPF needs bpf() syscall). However, GDB/KGDB excels at: (1) Examining complex data structures (follow 10 levels of nested pointers). (2) Early boot debugging (before tracing infrastructure is up). (3) Debugging panics/hangs where you need to inspect frozen state. (4) Understanding control flow of unfamiliar code with single-stepping. Best practice: use tracing for observation, GDB for deep control-flow analysis and data structure inspection.

---

## Summary

- KGDB: in-kernel GDB stub — remote debug via serial/network; halts entire system
- QEMU + GDB (-s -S): easiest kernel debug setup for development
- nokaslr: disable address randomization for symbol matching
- CONFIG_GDB_SCRIPTS: lx-dmesg, lx-ps, lx-lsmod, lx-symbols helper commands
- /proc/kcore: read-only inspection of live kernel memory via GDB
- GDB for deep inspection; tracing (ftrace/eBPF) for production observation

---

[Previous: kdump and crash ←](Chapter_27_kdump_crash.md) | [Next: CI, Testing, and Fuzzing →](Chapter_29_CI_Testing.md)
