# Chapter 29: Kernel Debugging Techniques

## Learning Goals
- Understand kernel debugging tools and techniques
- Know KGDB, crash dumps, and live debugging
- Grasp ftrace, perf, and tracing infrastructure
- Understand pstore, ramoops, and panic handling

---

## 29.1 Debugging Landscape

```
Kernel Debugging Techniques:

┌─────────────────────────────────────────────────────────┐
│                     DEBUGGING METHODS                    │
├─────────────────┬───────────────────┬───────────────────┤
│   PRINTK-BASED  │   INTERACTIVE     │   POST-MORTEM     │
│                 │                   │                   │
│ - printk/pr_*   │ - KGDB/KDB       │ - crash/kdump     │
│ - dynamic debug │ - JTAG/OpenOCD    │ - pstore/ramoops  │
│ - trace_printk  │ - QEMU + GDB     │ - serial log      │
│ - dev_dbg       │                   │ - last_kmsg       │
├─────────────────┼───────────────────┼───────────────────┤
│   TRACING       │   MEMORY DEBUG    │   STATIC ANALYSIS │
│                 │                   │                   │
│ - ftrace        │ - KASAN (addr)    │ - sparse          │
│ - perf          │ - KMSAN (uninit)  │ - smatch          │
│ - eBPF/bpftrace │ - KFENCE (prod)   │ - coccinelle      │
│ - trace-cmd     │ - UBSAN           │ - gcc warnings    │
│ - Perfetto      │ - lockdep         │ - clang-tidy      │
└─────────────────┴───────────────────┴───────────────────┘
```

---

## 29.2 KGDB — Kernel GDB

```
KGDB Setup:

Target machine (being debugged):
  Kernel config:
    CONFIG_KGDB=y
    CONFIG_KGDB_SERIAL_CONSOLE=y
    CONFIG_DEBUG_INFO=y

  Kernel command line:
    kgdboc=ttyS0,115200 kgdbwait
    │                    │
    │                    └── Break into debugger at boot
    └── KGDB over serial console on ttyS0

Host machine (running GDB):
  $ gdb vmlinux
  (gdb) target remote /dev/ttyUSB0
  (gdb) set remotebaud 115200
  (gdb) continue

  # Set breakpoints
  (gdb) break do_fork
  (gdb) break drivers/net/my_driver.c:42

  # Examine state
  (gdb) bt            # backtrace
  (gdb) info threads   # all CPUs
  (gdb) print current  # current task
  (gdb) lx-dmesg       # kernel dmesg (with scripts)

  # Trigger break from target
  $ echo g > /proc/sysrq-trigger

KGDB over network (kgdboe):
  kgdboe=@192.168.1.10/eth0,@192.168.1.20/
  (Less reliable, not for production)
```

---

## 29.3 ftrace — Function Tracer

```
ftrace — Built-in Kernel Tracing:

# Enable function tracing
$ cd /sys/kernel/debug/tracing

# Available tracers
$ cat available_tracers
function function_graph nop

# Trace all functions
$ echo function > current_tracer
$ echo 1 > tracing_on
$ cat trace | head -20
# tracer: function
#   TASK-PID   CPU#  TIMESTAMP  FUNCTION
      bash-1234  [001] 12345.678: sys_read <-el0_svc
      bash-1234  [001] 12345.678: ksys_read <-sys_read
      bash-1234  [001] 12345.678: vfs_read <-ksys_read

# Function graph — shows call hierarchy
$ echo function_graph > current_tracer
$ cat trace
 1)               |  sys_read() {
 1)               |    vfs_read() {
 1)               |      new_sync_read() {
 1)   0.852 us    |        ext4_file_read_iter();
 1)   1.234 us    |      }
 1)   1.567 us    |    }
 1)   2.345 us    |  }

# Filter to specific functions
$ echo 'my_driver_*' > set_ftrace_filter
$ echo 'schedule' >> set_ftrace_filter

# Trace events (tracepoints)
$ echo 1 > events/sched/sched_switch/enable
$ echo 1 > events/irq/irq_handler_entry/enable
$ cat trace
```

```c
/* trace_printk — for temporary debugging (never submit!) */
void my_function(int val)
{
    trace_printk("my_function called with val=%d\n", val);
    /* Output goes to ftrace buffer, not dmesg */
    /* Much lower overhead than printk */
}
```

---

## 29.4 Crash Dump Analysis (kdump)

```
kdump — Capture Kernel Crash Dumps:

Setup:
  1. Reserve memory for crash kernel:
     crashkernel=256M    (kernel command line)

  2. Load crash kernel:
     $ kexec -p /boot/vmlinuz-crash \
         --initrd=/boot/initramfs-crash.img \
         --append="root=UUID=... irqpoll maxcpus=1"

  3. When kernel panics:
     → kexec boots crash kernel from reserved memory
     → Crash kernel mounts old root, copies /proc/vmcore
     → vmcore saved to /var/crash/

Analysis with crash tool:
  $ crash vmlinux /var/crash/vmcore

  crash> bt                    # Backtrace of crashing task
  crash> bt -a                 # All CPUs
  crash> log                   # Kernel log buffer
  crash> ps                    # Process list at crash time
  crash> files <pid>           # Open files of a process
  crash> vm <pid>              # Virtual memory map
  crash> kmem -s               # Slab cache info
  crash> struct task_struct <addr>  # Examine structure
  crash> dis <function>        # Disassemble function
```

---

## 29.5 pstore — Persistent Storage

```
pstore/ramoops — Survive Reboots:

Reserve RAM that firmware preserves across reboot:

Device tree:
  reserved-memory {
      ramoops@a0000000 {
          compatible = "ramoops";
          reg = <0 0xa0000000 0 0x100000>;  /* 1MB */
          record-size  = <0x20000>;  /* 128KB per record */
          console-size = <0x40000>;  /* 256KB for console */
          ftrace-size  = <0x20000>;  /* 128KB for ftrace */
          pmsg-size    = <0x20000>;  /* 128KB for pmsg */
      };
  };

After panic + reboot:
  $ ls /sys/fs/pstore/
  dmesg-ramoops-0     ← Last kernel log before crash
  console-ramoops-0   ← Console output
  ftrace-ramoops-0    ← Ftrace buffer
  pmsg-ramoops-0      ← Userspace messages (Android)

  $ cat /sys/fs/pstore/dmesg-ramoops-0
  [  123.456] BUG: kernel NULL pointer dereference
  [  123.456] IP: my_driver_probe+0x42/0x100
  ...

Android:
  $ cat /sys/fs/pstore/pmsg-ramoops-0  # logcat from previous boot
  /proc/last_kmsg  (legacy, some Android kernels)
```

---

## 29.6 Memory Debugging

```
KASAN — Kernel Address Sanitizer:

CONFIG_KASAN=y

Detects:
  ├── Use-after-free (access freed memory)
  ├── Out-of-bounds (buffer overflow)
  ├── Stack buffer overflow
  └── Global buffer overflow

Example output:
  BUG: KASAN: slab-use-after-free in my_func+0x42/0x100
  Read of size 4 at addr ffff8880012345678 by task test/1234
  
  Call Trace:
   my_func+0x42/0x100
   do_something+0x10/0x50
  
  Allocated by task 1234:
   kmalloc+0x50/0x80
   my_init+0x20/0x40
  
  Freed by task 1234:
   kfree+0x30/0x60
   my_cleanup+0x10/0x30

UBSAN — Undefined Behavior Sanitizer:
  CONFIG_UBSAN=y
  Detects: signed overflow, shift errors, alignment issues

Lockdep — Lock Dependency Validator:
  CONFIG_PROVE_LOCKING=y
  Detects: potential deadlocks, lock ordering violations
  
  Example:
  WARNING: possible circular locking dependency detected
  task/1234 is trying to acquire lock B
  but task already holds lock A
  Lock A → Lock B   (this task)
  Lock B → Lock A   (seen elsewhere)
```

---

## 29.7 Boot Debugging Options

```
Kernel Command Line Debug Options:

# General debug
debug                       # Set console_loglevel to 10
ignore_loglevel             # Print ALL messages to console
initcall_debug              # Trace initcall timing

# Boot hang debugging
initcall_debug              # Shows each initcall with timing
  [    2.345] calling  my_driver_init+0x0/0x50 @ 1
  [    2.445] initcall my_driver_init+0x0/0x50 returned 0 after 100 ms

# Panic behavior
panic=10                    # Reboot 10 seconds after panic
oops=panic                  # Treat oops as panic
hung_task_panic=1           # Panic on hung task

# Memory debugging
memtest=4                   # Run memtest patterns 4 times
page_poison=1               # Poison freed pages
slub_debug=FZPU             # Full SLUB debugging

# Interrupt debugging
threadirqs                  # Force threaded IRQs (debugging)
irqpoll                     # Try all handlers on unexpected IRQ

# SysRq — Emergency keyboard commands
sysrq_always_enabled=1
# Alt+SysRq+B = reboot
# Alt+SysRq+C = crash (trigger kdump)
# Alt+SysRq+T = show all tasks
# Alt+SysRq+W = show blocked tasks
# Alt+SysRq+L = show all CPU backtraces
```

---

## Kernel Source References

| Function/File | Path | Purpose |
|-------|------|---------|
| kgdb_breakpoint() | kernel/debug/debug_core.c | KGDB core |
| ftrace | kernel/trace/trace.c | Function tracing core |
| kasan_report() | mm/kasan/report.c | KASAN error reporting |
| crash kernel | kernel/kexec.c | Crash dump kexec |
| pstore/ramoops | fs/pstore/ram.c | Persistent storage |
| lockdep | kernel/locking/lockdep.c | Lock dependency checking |
| panic() | kernel/panic.c | Kernel panic handler |

---

## Interview Questions

**Q1: How would you debug a kernel panic that occurs during boot?**
A: (1) Enable `earlycon` for early serial output. (2) Add `initcall_debug` to see which init function hangs/panics. (3) Configure pstore/ramoops to preserve the panic log across reboot. (4) If reproducible, use KGDB with `kgdbwait` to break at boot. (5) Use `ignore_loglevel` to see all messages. (6) Check the panic stack trace for the faulting function and address. (7) Use `addr2line` or GDB to map the address to source code.

**Q2: What is ftrace and when would you use it over printk?**
A: ftrace is the kernel's built-in tracing framework — it can trace function calls, scheduling, IRQs, and custom events with minimal overhead. Use ftrace over printk when: (1) you need to trace call flow across many functions, (2) timing precision matters, (3) the overhead of printk would perturb the problem, (4) you need traces from production without recompiling. ftrace can trace every function call with function_graph tracer.

**Q3: How does kdump work?**
A: At boot, memory is reserved for a crash kernel (`crashkernel=256M`). A secondary kernel is preloaded with `kexec -p`. When the primary kernel panics, it calls `kexec` to boot the crash kernel from the reserved region. The crash kernel treats the primary kernel's memory as `/proc/vmcore`. The crash kernel's initramfs copies vmcore to disk, then reboots. The `crash` tool later analyzes vmcore with the original vmlinux for stack traces, memory state, and process info.

---

## Summary

- Kernel debugging ranges from printk (simple) to KGDB (interactive) to kdump (post-mortem)
- ftrace provides low-overhead function and event tracing via /sys/kernel/debug/tracing
- KASAN, UBSAN, and lockdep detect memory errors, undefined behavior, and deadlocks at runtime
- pstore/ramoops preserves kernel logs and crash data across reboots
- Boot debugging uses earlycon, initcall_debug, and SysRq keys
- kdump captures full memory state on panic for offline analysis

---

*Next: [Chapter 30 — Kernel Source Code Walkthrough](Chapter_30_Source_Walkthrough.md)*
