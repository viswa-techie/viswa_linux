# Chapter 36: Boot Performance Optimization

## Learning Goals
- Understand boot time measurement and analysis
- Know optimization techniques at each boot stage
- Grasp kernel configuration for fast boot
- Understand deferred initialization and lazy loading

---

## 36.1 Measuring Boot Time

```
Boot Time Measurement Tools:

1. systemd-analyze (systemd systems):
   $ systemd-analyze
   Startup finished in 1.2s (firmware) + 0.8s (loader) +
                      2.1s (kernel) + 5.3s (userspace) = 9.4s

   $ systemd-analyze blame     # Slowest services
   $ systemd-analyze critical-chain  # Critical path
   $ systemd-analyze plot > boot.svg # Visual timeline

2. dmesg timestamps:
   $ dmesg -d | head -20    # Delta time between messages
   [  0.000000] Linux version 6.1.0 ...
   [  0.043851] ACPI: Early table checksum ...
   [  0.095234] Memory: 16268912K/16777216K available

3. initcall_debug:
   Kernel cmdline: initcall_debug
   $ dmesg | grep initcall | sort -t= -k2 -n -r | head
   initcall my_slow_driver+0x0/0x100 returned 0 after 523 msecs

4. ftrace boot tracing:
   Kernel cmdline: trace_event=initcall:* ftrace_dump_on_oops

5. Bootchart:
   $ systemd-bootchart    # Generate SVG boot chart

6. grabserial (serial timestamp):
   $ grabserial -d /dev/ttyUSB0 -t -m "Starting kernel"
   # Timestamps each line from serial console
```

---

## 36.2 Firmware Optimization

```
Firmware Stage Optimization:

Typical firmware time: 1-5 seconds

Optimization techniques:
┌─────────────────────────────────────────────────────────┐
│  1. Fast boot mode                                      │
│     ├── Skip POST (Power-On Self Test)                  │
│     ├── Skip memory training (use saved values)         │
│     ├── Skip PCI enumeration (defer to OS)              │
│     └── Skip USB enumeration                            │
│                                                         │
│  2. UEFI Capsule Fast Boot                              │
│     ├── Cache firmware variables                        │
│     ├── Skip non-boot device scan                       │
│     └── Use saved configuration from previous boot      │
│                                                         │
│  3. DDR training optimization                           │
│     ├── Train once, save parameters to NV storage       │
│     ├── Reuse training data on subsequent boots         │
│     └── Saves 500ms-2s per boot                         │
│                                                         │
│  Typical savings: 1-3 seconds                           │
└─────────────────────────────────────────────────────────┘
```

---

## 36.3 Bootloader Optimization

```
Bootloader Stage Optimization:

Typical bootloader time: 1-3 seconds

Falcon Mode (U-Boot):
┌─────────────────────────────────────────────────────────┐
│  Normal: SPL → U-Boot (full) → Load kernel → Boot      │
│                                                         │
│  Falcon: SPL → Load kernel → Boot (skip full U-Boot!)  │
│                                                         │
│  SPL directly loads kernel+DTB from known locations     │
│  No U-Boot shell, no env parsing, no delay              │
│  Saves: 500ms-2s                                        │
└─────────────────────────────────────────────────────────┘

Other bootloader optimizations:
  ├── Set bootdelay=0 (no user wait)
  ├── Remove unused drivers from bootloader
  ├── Pre-compute kernel load address
  ├── Use DMA for storage reads
  ├── Skip HDMI/display init in bootloader
  └── Use XIP (execute-in-place) from NOR flash
```

---

## 36.4 Kernel Configuration Optimization

```
Kernel Config for Fast Boot:

# Kernel size reduction (smaller = faster load + decompress)
CONFIG_CC_OPTIMIZE_FOR_SIZE=y     # -Os instead of -O2
CONFIG_MODULES=y                  # Use modules, not built-in
CONFIG_KERNEL_LZ4=y               # Fast decompression
# LZ4 decompresses ~4x faster than gzip

# Disable unused subsystems
# CONFIG_SOUND is not set           # If no audio
# CONFIG_DRM is not set             # If no display
# CONFIG_USB is not set             # If no USB
# CONFIG_WIRELESS is not set        # If no WiFi
# CONFIG_BT is not set              # If no Bluetooth

# Disable debug features
# CONFIG_DEBUG_INFO is not set
# CONFIG_PRINTK_TIME is not set
# CONFIG_FTRACE is not set
# CONFIG_KASAN is not set

# Fast boot options
CONFIG_PRINTK=y                    # Keep for debugging
# But add "quiet" to cmdline       # Reduce console output

# Deferred initcalls
CONFIG_DEFERRED_STRUCT_PAGE_INIT=y # Defer page struct init
CONFIG_JUMP_LABEL=y                # Static branch optimization
```

```
Kernel Command Line for Fast Boot:

quiet                          # Suppress most boot messages
loglevel=1                     # Only KERN_ALERT and above
lpj=<value>                    # Skip calibrate_delay()
                               # (use value from previous boot)
initcall_blacklist=func1,func2 # Skip specific slow initcalls
raid=noautodetect              # Skip MD RAID detection
rootwait                       # Wait for root device
rootfstype=ext4                # Don't try all FS types
```

---

## 36.5 initramfs Optimization

```
initramfs Size Reduction:

Before: 30MB initramfs → 2 seconds to decompress/unpack

Optimization:
  ├── Include only modules needed for root device
  │   (storage controller, filesystem, crypto if dm-crypt)
  │
  ├── Strip modules:
  │   $ find . -name '*.ko' -exec strip --strip-debug {} \;
  │
  ├── Use LZ4 compression: ~4x faster than gzip
  │   INITRAMFS_COMPRESSION=".lz4"
  │
  ├── Remove unnecessary tools
  │   Keep: busybox, mount, switch_root
  │   Remove: network tools, diagnostics
  │
  └── Consider eliminating initramfs entirely:
      Build essential drivers into kernel
      root=/dev/mmcblk0p2 rootfstype=ext4

After: 3MB initramfs → 200ms to decompress/unpack
```

---

## 36.6 Driver Probe Optimization

```
Driver Probe Time Analysis:

$ dmesg | grep -E 'probe|init.*ms'
[  1.234] my_driver: probe took 523ms     ← TOO SLOW

Common slow probe causes and fixes:

┌──────────────────────┬──────────────────────────────────┐
│ Cause                │ Solution                         │
├──────────────────────┼──────────────────────────────────┤
│ Firmware loading      │ Embed firmware in kernel or     │
│ (request_firmware)   │ include in initramfs             │
├──────────────────────┼──────────────────────────────────┤
│ Device reset delays  │ Use async probe                  │
│ (msleep during probe)│ module_param async_probe         │
├──────────────────────┼──────────────────────────────────┤
│ I2C/SPI comm during  │ Defer non-critical setup         │
│ probe                │ to first use                     │
├──────────────────────┼──────────────────────────────────┤
│ Sequential probe     │ Enable async probing:            │
│                      │ CONFIG_DRIVER_ASYNC_PROBE=y      │
├──────────────────────┼──────────────────────────────────┤
│ Clock/regulator wait │ Ensure providers probe first     │
│ (deferred probe)     │ via device tree ordering         │
└──────────────────────┴──────────────────────────────────┘
```

```c
/* Async probe — driver probes in parallel with others */
static struct platform_driver my_driver = {
    .probe = my_probe,
    .driver = {
        .name = "my-device",
        .probe_type = PROBE_PREFER_ASYNCHRONOUS, /* ← async */
    },
};
```

---

## 36.7 User Space Optimization

```
User Space Boot Optimization:

1. systemd optimizations:
   ├── Mask unneeded services:
   │   $ systemctl mask NetworkManager-wait-online
   │
   ├── Set DefaultTimeoutStartSec=5s
   │   (default 90s is too long for fast boot)
   │
   ├── Use socket activation:
   │   Service starts only when connection arrives
   │
   └── Analyze dependencies:
       $ systemd-analyze critical-chain

2. Parallel startup:
   ├── systemd already parallelizes by default
   ├── Remove unnecessary After= dependencies
   └── Use Type=notify for faster ready detection

3. Application optimization:
   ├── Prelink shared libraries (avoid runtime relocation)
   ├── Use mmap for large config files
   ├── Lazy-load non-critical modules
   └── Profile with perf: $ perf record -g -p <pid>

4. Read-ahead:
   ├── Use readahead to predict and pre-load files
   └── systemd has built-in readahead support

Typical optimization results:
  Before:  25 seconds (firmware→login)
  After:    5 seconds (same hardware)
  Key wins: quiet+loglevel (1s), initramfs shrink (1.5s),
            disable unused services (3s), async probe (1s),
            firmware fast-boot (2s)
```

---

## 36.8 Suspend/Resume as Fast Boot

```
Hibernate/Suspend for "Instant" Boot:

Suspend to RAM (S3):
  ├── System state saved in DRAM
  ├── DRAM in self-refresh mode
  ├── Resume: ~1-3 seconds
  └── Power: ~0.1W during suspend

Hibernate (S4):
  ├── System state saved to disk (swap)
  ├── Full power off
  ├── Resume: ~5-15 seconds (read from disk)
  └── Power: 0W during hibernate

Snapshot Boot:
  ├── Boot once, save memory snapshot
  ├── Subsequent boots: restore snapshot
  ├── Similar to hibernate but faster
  └── Used in automotive for fast boot

Automotive approach:
  ┌──────────────────────────────────────────┐
  │  Car door opens → resume from S3         │
  │  Rear camera: <1 second                  │
  │  Full HMI: <3 seconds                    │
  │                                          │
  │  If battery too low → cold boot          │
  │  If OTA pending → full restart           │
  └──────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How would you reduce Linux boot time from 30 seconds to under 5 seconds?**
A: Systematic approach: (1) Measure each stage with `systemd-analyze` and `dmesg`. (2) Firmware: enable fast-boot, skip unnecessary POST. (3) Bootloader: use Falcon mode or zero boot delay. (4) Kernel: use LZ4 compression, `quiet` and `loglevel=1`, remove unused drivers, set `lpj=` to skip calibration. (5) initramfs: minimize to essential modules or eliminate entirely. (6) User space: mask unused services, use socket activation, async probe for drivers. Each stage typically saves 1-5 seconds.

**Q2: What is the difference between async probe and deferred probe?**
A: Async probe runs a driver's `probe()` in parallel with other drivers, reducing total probe time. It's opt-in via `PROBE_PREFER_ASYNCHRONOUS`. Deferred probe (`-EPROBE_DEFER`) is when a driver's probe fails because a dependency (clock, regulator, GPIO) isn't ready yet — the probe is retried later. They solve different problems: async reduces wall-clock time, deferred handles ordering dependencies.

**Q3: What kernel command line parameters speed up boot?**
A: `quiet` suppresses messages (saves serial output time). `loglevel=1` only shows critical alerts. `lpj=N` pre-sets loops-per-jiffy skipping calibrate_delay(). `initcall_blacklist=func1,func2` skips specific slow initcalls. `rootfstype=ext4` avoids trying all filesystem types. `raid=noautodetect` skips MD RAID scan. `fastboot` skips certain ACPI checks.

---

## Summary

- Boot time optimization requires measuring each stage: firmware, bootloader, kernel, initramfs, userspace
- Firmware: skip POST, cache DDR training, fast-boot mode
- Bootloader: Falcon mode (skip full U-Boot), zero boot delay
- Kernel: LZ4 compression, quiet boot, disable unused subsystems, async driver probe
- initramfs: minimize size, strip modules, use LZ4, or eliminate entirely
- User space: mask unused services, socket activation, parallel startup
- Suspend/resume provides fastest "boot" for automotive (1-3 seconds from S3)

---

*Next: [Chapter 37 — References and Resources](Chapter_37_References.md)*
