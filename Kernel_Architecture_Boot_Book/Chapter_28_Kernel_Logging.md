# Chapter 28: Kernel Logging

## Learning Goals
- Understand the kernel log ring buffer and printk
- Know dmesg, journald, and syslog architectures
- Grasp log levels, rate limiting, and dynamic debug
- Understand early boot console and serial logging

---

## 28.1 printk Architecture

```
printk() — Kernel Logging System:

Driver/Subsystem calls:
  printk(KERN_ERR "Device %s failed: %d\n", name, err);
  pr_err("Device %s failed: %d\n", name, err);  /* preferred */
  dev_err(dev, "probe failed: %d\n", err);       /* device context */

printk() internals:
  ┌──────────────────────────────────────────────────┐
  │  printk(fmt, ...)                                │
  │    │                                             │
  │    ├── Format string into printk buffer          │
  │    │                                             │
  │    ├── Store in kernel log buffer (ring buffer)  │
  │    │   ┌────────────────────────────────┐       │
  │    │   │  Ring Buffer (default 256KB)   │       │
  │    │   │  log_buf_len= cmdline param    │       │
  │    │   │  ┌──┬──┬──┬──┬──┬──┬──┬──┐   │       │
  │    │   │  │M1│M2│M3│M4│M5│M6│M7│M8│   │       │
  │    │   │  └──┴──┴──┴──┴──┴──┴──┴──┘   │       │
  │    │   │  Oldest ────────────► Newest   │       │
  │    │   │  (overwrites oldest when full) │       │
  │    │   └────────────────────────────────┘       │
  │    │                                             │
  │    ├── Wake up /dev/kmsg readers                 │
  │    │   ├── syslogd / rsyslogd                    │
  │    │   └── systemd-journald                      │
  │    │                                             │
  │    └── Output to console (if level <= console_loglevel) │
  │        ├── Serial console (ttyS0)                │
  │        ├── VGA console                           │
  │        └── Netconsole (over network)             │
  └──────────────────────────────────────────────────┘
```

---

## 28.2 Log Levels

```
Kernel Log Levels (0 = most severe):

Level  Macro           Meaning
─────  ──────────────  ──────────────────────────────
  0    KERN_EMERG      System is unusable
  1    KERN_ALERT      Action must be taken immediately
  2    KERN_CRIT       Critical conditions
  3    KERN_ERR        Error conditions
  4    KERN_WARNING    Warning conditions
  5    KERN_NOTICE     Normal but significant
  6    KERN_INFO       Informational
  7    KERN_DEBUG      Debug-level messages

Console loglevel:
  Only messages with level < console_loglevel are printed
  Default console_loglevel = 7 (show up to KERN_INFO)

# Check current levels
$ cat /proc/sys/kernel/printk
7       4       1       7
│       │       │       └── default_console_loglevel
│       │       └── minimum_console_loglevel
│       └── default_message_loglevel
└── console_loglevel

# Change console loglevel
$ echo 8 > /proc/sys/kernel/printk    # Show all messages
$ dmesg -n 1                          # Only KERN_EMERG to console
```

```c
/* Preferred logging macros */

/* Basic printk (avoid in new code) */
printk(KERN_INFO "Hello %s\n", name);

/* pr_* macros (preferred for standalone code) */
pr_emerg("System halting\n");
pr_err("Operation failed: %d\n", err);
pr_warn("Resource low\n");
pr_info("Driver loaded\n");
pr_debug("Debug value: %d\n", val);  /* compiled out unless DEBUG */

/* dev_* macros (preferred for drivers — includes device name) */
dev_err(&pdev->dev, "probe failed: %d\n", ret);
dev_info(&pdev->dev, "initialized successfully\n");
dev_dbg(&pdev->dev, "register 0x%x = 0x%x\n", reg, val);

/* Output: "my_device 0000:01:00.0: probe failed: -19" */
```

---

## 28.3 Early Boot Console

```
Early Console — Before Normal Console Init:

Problem: printk works early, but messages have nowhere to go
until a console driver is registered.

Solution: early_printk / earlycon

Kernel command line:
  earlycon=uart8250,io,0x3f8,115200n8     # x86 legacy UART
  earlycon=pl011,0x09000000               # ARM PL011 UART
  earlycon                                 # Auto-detect from DT

Timeline:
  ┌──────────────────────────────────────────────┐
  │  Boot start                                  │
  │   │                                          │
  │   ├── [No console] Messages buffered only    │
  │   │                                          │
  │   ├── earlycon registration (from DT/cmdline)│
  │   │   └── NOW early messages appear on UART  │
  │   │                                          │
  │   ├── console_init() — register real consoles│
  │   │   └── Buffered messages replayed          │
  │   │                                          │
  │   └── Normal console operation               │
  └──────────────────────────────────────────────┘

Device tree earlycon:
  chosen {
      stdout-path = "serial0:115200n8";
  };
  
  serial0: uart@09000000 {
      compatible = "arm,pl011";
      reg = <0x0 0x09000000 0x0 0x1000>;
  };
```

---

## 28.4 dmesg and /dev/kmsg

```
Accessing Kernel Logs:

dmesg — Read kernel ring buffer:
  $ dmesg                    # All messages
  $ dmesg -T                 # Human-readable timestamps
  $ dmesg -l err             # Only errors
  $ dmesg -l err,warn        # Errors and warnings
  $ dmesg -f kern            # Only kernel facility
  $ dmesg -w                 # Follow (like tail -f)
  $ dmesg -c                 # Read and clear
  $ dmesg | grep -i usb      # Filter for USB

/dev/kmsg — Direct kernel log interface:
  $ cat /dev/kmsg            # Read (blocking)
  $ echo "My message" > /dev/kmsg  # Write from userspace
  
  Format: priority,sequence,timestamp,-;message
  6,1234,5678901234,-;USB device connected

/proc/kmsg — Legacy interface (used by syslogd):
  $ cat /proc/kmsg           # Read and consume (destructive)

Ring buffer size:
  Default: 256KB (2^17 = 131072 bytes typically)
  Kernel cmdline: log_buf_len=1M    # Increase to 1MB
  Typical embedded: 64KB-256KB
  Typical server: 1MB-16MB
```

---

## 28.5 Structured Logging with dev_* and pr_fmt

```c
/* pr_fmt — Add prefix to all pr_* messages in a file */
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

/* All pr_* calls in this file now get the module name prefix */
pr_info("initialized\n");
/* Output: "my_driver: initialized" */

/* dev_* structured output includes device path */
dev_info(dev, "speed set to %d MHz\n", speed);
/* Output: "my_driver 0000:01:00.0: speed set to 100 MHz" */

/* Subsystem-specific logging */
netdev_err(netdev, "link down\n");
/* Output: "eth0: link down" */

/* DRM subsystem */
drm_err(drm, "mode set failed\n");
/* Output: "[drm:my_func] *ERROR* mode set failed" */
```

---

## 28.6 Dynamic Debug

```
Dynamic Debug — Enable/Disable debug messages at runtime:

pr_debug() and dev_dbg() are compiled in (with CONFIG_DYNAMIC_DEBUG)
but disabled by default. Enable selectively:

# Enable debug for a specific file
$ echo 'file drivers/usb/core/hub.c +p' > \
    /sys/kernel/debug/dynamic_debug/control

# Enable for a specific function
$ echo 'func my_probe +p' > \
    /sys/kernel/debug/dynamic_debug/control

# Enable for a module
$ echo 'module my_driver +p' > \
    /sys/kernel/debug/dynamic_debug/control

# Enable for a specific line
$ echo 'file my_driver.c line 42 +p' > \
    /sys/kernel/debug/dynamic_debug/control

# Flags:
#   +p  enable printing
#   -p  disable printing
#   +f  include function name
#   +l  include line number
#   +m  include module name
#   +t  include thread ID

# View all debug points
$ cat /sys/kernel/debug/dynamic_debug/control | head
```

---

## 28.7 Rate Limiting and Filtering

```c
/* Rate-limited printing — prevent log flooding */
printk_ratelimited(KERN_ERR "Error occurred\n");

/* pr_* rate-limited variants */
pr_err_ratelimited("Still failing\n");

/* printk_once — print only the first time */
pr_warn_once("This warning appears only once\n");

/* net_ratelimit() — network subsystem rate limiting */
if (net_ratelimit())
    pr_warn("Dropped packet from %pI4\n", &saddr);

/* Kernel configurable rate limit */
/* /proc/sys/kernel/printk_ratelimit = 5     (seconds) */
/* /proc/sys/kernel/printk_ratelimit_burst = 10 (messages) */
/* → Allow 10 messages per 5 seconds */
```

---

## 28.8 Netconsole and Remote Logging

```
Netconsole — Send kernel logs over network:

# Kernel command line
netconsole=6665@192.168.1.10/eth0,6666@192.168.1.20/aa:bb:cc:dd:ee:ff

# Dynamic configuration
$ modprobe netconsole
$ echo "add" > /sys/kernel/config/netconsole/target1/enabled

# Receive on remote machine
$ nc -u -l 6666

Use cases:
  - Embedded systems without serial console
  - Capturing kernel panic output remotely
  - Data center server monitoring
  - Debugging network/storage issues (where disk logging fails)

Serial console (most reliable for boot debugging):
  console=ttyS0,115200n8     # x86
  console=ttyAMA0,115200     # ARM PL011
  console=ttyMSM0,115200n8   # Qualcomm
```

---

## Kernel Source References

| Function/File | Path | Purpose |
|-------|------|---------|
| printk() | kernel/printk/printk.c | Core logging function |
| vprintk_store() | kernel/printk/printk.c | Store to ring buffer |
| register_console() | kernel/printk/printk.c | Register console driver |
| earlycon_init() | drivers/tty/serial/earlycon.c | Early console setup |
| dynamic_debug | lib/dynamic_debug.c | Dynamic debug control |
| dev_printk() | drivers/base/core.c | Device-context logging |
| /dev/kmsg | kernel/printk/printk.c | Userspace log interface |

---

## Interview Questions

**Q1: How does printk work during early boot when no console exists?**
A: printk always stores messages in the kernel ring buffer (log_buf) regardless of console availability. When the first console is registered via `register_console()`, all buffered messages are replayed to that console. For even earlier output, `earlycon` can be configured via kernel command line or device tree to register a minimal UART console before normal console init.

**Q2: What is the difference between /dev/kmsg and /proc/kmsg?**
A: `/dev/kmsg` is the modern interface — supports read (non-destructive), write (inject messages), and seek. Multiple readers can read independently. `/proc/kmsg` is the legacy interface — read is destructive (consumed messages are gone), single reader only (syslogd typically). systemd-journald uses `/dev/kmsg`.

**Q3: How do you debug a kernel crash that happens before the console is available?**
A: (1) Use `earlycon` on kernel command line to get UART output from the earliest point. (2) Use `earlyprintk` for platform-specific early output. (3) Increase `log_buf_len` to preserve more messages. (4) Use pstore/ramoops to persist logs across reboots. (5) Use JTAG/debug probes to read memory directly. (6) Add `initcall_debug` to see which init function hangs.

---

## Summary

- printk stores messages in a kernel ring buffer — accessible via dmesg and /dev/kmsg
- Eight log levels (0-7) from EMERG to DEBUG control severity and console filtering
- earlycon provides boot console before normal console drivers initialize
- dev_* macros are preferred for drivers — they include device identification
- Dynamic debug enables runtime control of pr_debug/dev_dbg messages
- Rate limiting prevents log flooding from high-frequency error paths
- Netconsole and serial console enable remote/reliable log capture

---

*Next: [Chapter 29 — Kernel Debugging Techniques](Chapter_29_Kernel_Debugging.md)*
