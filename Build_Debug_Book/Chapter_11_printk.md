# Chapter 11: printk Architecture

## Learning Goals
- Understand printk ring buffer and log levels
- Learn console drivers and output paths
- Master rate limiting and printk_safe contexts
- Know structured logging and /dev/kmsg

---

## 1. printk Ring Buffer

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  printk() = kernel's printf() — writes to ring buffer   │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Ring buffer (struct printk_ringbuffer):  │            │
  │  │                                          │            │
  │  │ ┌────┬────┬────┬────┬────┬────┬────┐    │            │
  │  │ │msg1│msg2│msg3│msg4│msg5│msg6│msg7│    │            │
  │  │ └────┴────┴────┴────┴────┴────┴────┘    │            │
  │  │ ↑ oldest                    newest ↑    │            │
  │  │                                          │            │
  │  │ When full: oldest messages overwritten   │            │
  │  │ Default size: 2^CONFIG_LOG_BUF_SHIFT     │            │
  │  │   (typically 2^17 = 128KB)               │            │
  │  │ Override: log_buf_len=1M on cmdline      │            │
  │  │                                          │            │
  │  │ Each record contains:                    │            │
  │  │ - Timestamp (ns)                         │            │
  │  │ - Log level (0-7)                        │            │
  │  │ - Facility (kern, user, etc.)            │            │
  │  │ - Caller ID (task or IRQ context)        │            │
  │  │ - Text message                           │            │
  │  │ - Key-value dictionary (structured)      │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Log levels:                                             │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 0 KERN_EMERG    — system unusable        │            │
  │  │ 1 KERN_ALERT    — action must be taken   │            │
  │  │ 2 KERN_CRIT     — critical conditions    │            │
  │  │ 3 KERN_ERR      — error conditions       │            │
  │  │ 4 KERN_WARNING  — warning conditions     │            │
  │  │ 5 KERN_NOTICE   — normal, significant    │            │
  │  │ 6 KERN_INFO     — informational          │            │
  │  │ 7 KERN_DEBUG    — debug-level messages   │            │
  │  │                                          │            │
  │  │ pr_err("error: %d\n", err);  → level 3  │            │
  │  │ pr_info("loaded\n");          → level 6  │            │
  │  │ pr_debug("val=%d\n", x);     → level 7  │            │
  │  │ dev_err(dev, "fail\n");      → level 3  │            │
  │  │   + device name prefix                   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Console loglevel (what appears on console):             │
  │  ┌──────────────────────────────────────────┐            │
  │  │ /proc/sys/kernel/printk                  │            │
  │  │ → 4  4  1  7                             │            │
  │  │   │  │  │  └─ default level for printk()│            │
  │  │   │  │  └──── minimum console loglevel  │            │
  │  │   │  └─────── default console loglevel  │            │
  │  │   └────────── current console loglevel  │            │
  │  │                                          │            │
  │  │ Only messages with level < console_loglevel│          │
  │  │ are printed to the console                │            │
  │  │ Level 4 = warnings and above shown        │            │
  │  │ echo 8 > /proc/sys/kernel/printk          │            │
  │  │ → show everything including debug         │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Console Drivers and Output

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  printk output flow:                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ printk("message")                        │            │
  │  │   │                                      │            │
  │  │   ├──→ Ring buffer (always)              │            │
  │  │   │                                      │            │
  │  │   ├──→ Console drivers (if level high    │            │
  │  │   │    enough):                          │            │
  │  │   │    ├── Serial console (earlycon)     │            │
  │  │   │    ├── VGA text console              │            │
  │  │   │    ├── fbcon (framebuffer console)   │            │
  │  │   │    └── netconsole (UDP to remote)    │            │
  │  │   │                                      │            │
  │  │   └──→ /dev/kmsg (userspace reads)       │            │
  │  │        ├── dmesg command                 │            │
  │  │        ├── journald (systemd-journald)   │            │
  │  │        └── rsyslog/syslog-ng             │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Early console (earlycon):                               │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Before regular console drivers init:     │            │
  │  │ earlycon=uart8250,mmio,0x3f8,115200n8    │            │
  │  │                                          │            │
  │  │ Uses hardcoded MMIO/PIO writes           │            │
  │  │ Essential for debugging early boot       │            │
  │  │ Replaced by regular console after init   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Rate limiting:                                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ printk_ratelimited("event %d\n", n);     │            │
  │  │   Default: 10 messages per 5 seconds     │            │
  │  │                                          │            │
  │  │ printk_once("init done\n");              │            │
  │  │   Only first call prints                 │            │
  │  │                                          │            │
  │  │ pr_info_ratelimited("...");              │            │
  │  │ dev_err_ratelimited(dev, "...");         │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does printk work in interrupt and NMI context?**
**A:** printk must work everywhere — process context, IRQ context, and even NMI (Non-Maskable Interrupt). In normal process context: printk acquires the logbuf_lock (now lockless ring buffer in newer kernels), appends the message to the ring buffer, and attempts to flush to consoles synchronously (calling console drivers directly while holding the console_sem). In IRQ context: same path, but console output may be deferred if the console_sem is already held — the message goes to the ring buffer immediately, and a later printk call or irq_work flushes it. In NMI context (panic, watchdog): the kernel can't acquire locks safely (would deadlock). The printk_safe mechanism uses per-CPU temporary buffers: NMI printk writes to a per-CPU NMI buffer, and when NMI returns, the buffer is flushed to the main ring buffer. Since kernel 5.15+, printk uses an atomic/lockless ring buffer (`struct printk_ringbuffer`) with multi-producer safe writes, eliminating the logbuf_lock entirely. Console output in atomic contexts may use "nbcon" (non-blocking console) infrastructure (in development) to avoid blocking or deadlocking.

**Q2: What is netconsole and when is it useful?**
**A:** netconsole is a kernel console driver that sends kernel log messages over UDP to a remote machine. Configuration: `netconsole=6665@192.168.1.1/eth0,6666@192.168.1.2/aa:bb:cc:dd:ee:ff` — sends from local port 6665, interface eth0, to remote IP on port 6666. On the remote machine: `nc -u -l -p 6666` receives messages. Useful when: (1) No serial port available (many modern servers lack serial). (2) Need to capture kernel panic/oops output from a machine that crashes before disk logging. (3) Remote debugging of headless servers or embedded devices. (4) Capturing output that happens after disk I/O fails (storage driver panic). Limitations: only works after the network driver is initialized (not useful for early boot — use earlycon for that). The kernel sends UDP packets directly from the NIC driver layer, bypassing the network stack, so it works even when networking is otherwise broken. Can be loaded as a module (`modprobe netconsole ...`) or built-in with command line parameters.

---

## Summary

- printk: writes to ring buffer (128KB default, lockless), optionally to console
- 8 log levels: KERN_EMERG (0) to KERN_DEBUG (7); console_loglevel filters display
- Console drivers: serial, VGA, fbcon, netconsole — flushed synchronously if possible
- Rate limiting: printk_ratelimited(), printk_once() prevent log flooding
- /dev/kmsg: userspace access to ring buffer; dmesg reads it
- NMI-safe: per-CPU NMI buffers; deferred flushing for atomic contexts
- earlycon: hardcoded early boot console before regular drivers initialize

---

[Previous: Initramfs ←](Chapter_10_Initramfs.md) | [Next: Dynamic Debug →](Chapter_12_Dynamic_Debug.md)
