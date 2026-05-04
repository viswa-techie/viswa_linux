# Chapter 13: dmesg and Kernel Logging

## Learning Goals
- Master dmesg command options and filtering
- Understand /dev/kmsg interface and structured logging
- Learn pstore for persistent crash logs
- Know serial console, netconsole, and remote logging

---

## 1. dmesg Command

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  dmesg reads kernel ring buffer (/dev/kmsg):             │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ dmesg                  — all messages    │            │
  │  │ dmesg -T               — human timestamps│            │
  │  │ dmesg -t               — no timestamps   │            │
  │  │ dmesg -l err           — only errors     │            │
  │  │ dmesg -l err,warn      — errors+warnings │            │
  │  │ dmesg -f kern          — facility=kern   │            │
  │  │ dmesg -w               — follow (like tail -f)│     │
  │  │ dmesg -c               — read and clear  │            │
  │  │ dmesg -H               — human-readable  │            │
  │  │ dmesg --since "1 hour ago"               │            │
  │  │ dmesg | grep -i error                    │            │
  │  │                                          │            │
  │  │ # systemd:                               │            │
  │  │ journalctl -k          — kernel messages │            │
  │  │ journalctl -k -p err   — errors only     │            │
  │  │ journalctl -k -b -1   — previous boot   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  /dev/kmsg structured format:                            │
  │  ┌──────────────────────────────────────────┐            │
  │  │ cat /dev/kmsg (raw):                     │            │
  │  │ 6,1234,56789012,-;e1000 loaded           │            │
  │  │ │ │    │         │                       │            │
  │  │ │ │    │         └── message text        │            │
  │  │ │ │    └──────────── timestamp (μs)      │            │
  │  │ │ └───────────────── sequence number     │            │
  │  │ └─────────────────── level + facility    │            │
  │  │                                          │            │
  │  │ Writing to /dev/kmsg:                    │            │
  │  │ echo "<6>My message" > /dev/kmsg         │            │
  │  │ # Injects userspace message at level 6   │            │
  │  │ # Used by systemd, udev for logging      │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. pstore — Persistent Storage for Crashes

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Problem: kernel panic → reboot → dmesg is lost         │
  │  pstore: saves log to persistent storage before reboot  │
  │                                                           │
  │  Backends:                                               │
  │  ┌──────────────────────────────────────────┐            │
  │  │ ramoops: reserves RAM region that        │            │
  │  │   survives warm reboot                   │            │
  │  │   DT: ramoops { compatible="ramoops";    │            │
  │  │                  memory-region=<&pstore>; │            │
  │  │                  record-size=0x20000;     │            │
  │  │                  console-size=0x20000; }; │            │
  │  │                                          │            │
  │  │ EFI pstore: uses UEFI variables          │            │
  │  │   (x86 with UEFI firmware)               │            │
  │  │                                          │            │
  │  │ ERST: ACPI Error Record Serialization    │            │
  │  │   Table (server platforms)               │            │
  │  │                                          │            │
  │  │ blk pstore: writes to block device       │            │
  │  │   (SSD/eMMC partition)                   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  After reboot:                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ mount -t pstore none /sys/fs/pstore      │            │
  │  │ ls /sys/fs/pstore/                        │            │
  │  │   dmesg-ramoops-0    — kernel log        │            │
  │  │   console-ramoops-0  — console output    │            │
  │  │   pmsg-ramoops-0     — userspace messages│            │
  │  │   ftrace-ramoops-0   — ftrace buffer     │            │
  │  │                                          │            │
  │  │ cat /sys/fs/pstore/dmesg-ramoops-0       │            │
  │  │ → shows last kernel messages before crash│            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: A production server panicked and rebooted. How do you retrieve the crash logs?**
**A:** Multiple approaches, from most to least likely: (1) **journalctl -k -b -1**: if the system uses systemd with persistent journal (`Storage=persistent` in journald.conf), previous boot's kernel messages may be saved. (2) **pstore**: check `/sys/fs/pstore/` — if ramoops or EFI pstore was configured, the last kernel dmesg, console output, and even ftrace buffers are preserved across reboot. Read `dmesg-ramoops-*` files. (3) **Serial console log**: if the server has a serial console connected to a log aggregator (common in data centers), the full panic output including the Oops/call trace was captured. (4) **netconsole**: if netconsole was configured, the remote receiver has the log. (5) **kdump**: if `CONFIG_CRASH_DUMP` and kexec were configured, a crash kernel was booted and captured a vmcore (memory dump) to `/var/crash/`. Use the `crash` tool to analyze it: `crash vmlinux vmcore` → `bt` for backtrace, `dmesg` for kernel log. (6) **/var/log/kern.log** or **/var/log/syslog**: if rsyslog was running and managed to write before the panic. (7) **IPMI SOL** (Intelligent Platform Management Interface Serial-Over-LAN): enterprise servers capture console output via BMC even during kernel panics. Best practice: configure at least two of these (pstore + serial/netconsole + kdump).

---

## Summary

- dmesg: reads kernel ring buffer; -T for human timestamps, -l for level filtering
- journalctl -k: systemd equivalent; -b -1 for previous boot
- /dev/kmsg: structured format (level, seq, timestamp, text); writable from userspace
- pstore: preserves logs across crash/reboot (ramoops, EFI, ERST backends)
- Console hierarchy: earlycon → serial → VGA/fbcon → netconsole
- Best practice: configure pstore + serial/netconsole + kdump for production servers

---

[Previous: Dynamic Debug ←](Chapter_12_Dynamic_Debug.md) | [Next: ftrace Architecture →](Chapter_14_ftrace.md)
