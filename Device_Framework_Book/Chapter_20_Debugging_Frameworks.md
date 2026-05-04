# Chapter 20: Debugging Frameworks

## Learning Goals
- Know framework-specific debugging tools and techniques
- Use debugfs, tracepoints, and ftrace for framework analysis
- Debug probe failures, resource conflicts, and data path issues
- Master systematic debugging methodology for multi-framework problems

---

## 20.1 Debugging Methodology

```
Systematic Framework Debugging:

1. Identify the layer:
   ┌─────────────────────────────────────┐
   │ User space → strace, logcat         │
   │ Framework API → dmesg, debugfs      │
   │ Driver → printk, ftrace, kgdb       │
   │ Hardware → devmem, JTAG, scope      │
   └─────────────────────────────────────┘

2. Common failure categories:
   ├── Probe failure: -EPROBE_DEFER, missing resource
   ├── Timeout: DMA not completing, I2C NAK
   ├── Data corruption: DMA cache coherency
   ├── Performance: interrupt storm, lock contention
   └── Power: device not waking, clock gated
```

---

## 20.2 Probe Failure Debugging

```bash
# Check which devices failed to probe
$ cat /sys/kernel/debug/devices_deferred

# Check driver binding
$ ls /sys/bus/i2c/drivers/tmp102/
$ cat /sys/bus/i2c/devices/1-0048/uevent

# Check if device tree node was parsed
$ ls /proc/device-tree/soc/i2c@40005400/sensor@48/
$ cat /proc/device-tree/soc/i2c@40005400/sensor@48/compatible

# dmesg for probe errors
$ dmesg | grep -i "probe\|defer\|error\|fail"

# Common probe failures:
#   -EPROBE_DEFER (-517): dependency not ready
#   -ENODEV (-19):  device not found (wrong address/compatible)
#   -ENOMEM (-12):  allocation failed
#   -EBUSY (-16):   resource already claimed
#   -EINVAL (-22):  invalid DT property
```

---

## 20.3 Framework-Specific Debug

```bash
#=== V4L2 Debug ===
$ v4l2-ctl -d /dev/video0 --all          # device info
$ v4l2-ctl -d /dev/video0 --log-status   # kernel-side dump
$ media-ctl -d /dev/media0 -p            # pipeline topology
$ echo 0x1f > /sys/module/videobuf2_common/parameters/debug

#=== DRM Debug ===
$ cat /sys/kernel/debug/dri/0/state       # full display state
$ modetest -M my-display -c              # connectors
$ cat /sys/kernel/debug/dri/0/framebuffer # active FBs
$ echo 0x1f > /sys/module/drm/parameters/debug
# DRM debug levels:
#   0x01: CORE    0x02: DRIVER
#   0x04: KMS     0x08: PRIME
#   0x10: ATOMIC  0x20: VBL

#=== ALSA Debug ===
$ cat /proc/asound/cards
$ cat /proc/asound/card0/pcm0p/info
$ cat /proc/asound/card0/codec#0
$ amixer -c 0 contents                   # all controls

#=== I2C Debug ===
$ i2cdetect -y 1                          # scan bus
$ echo 0x1 > /sys/module/i2c_core/parameters/debug
# If 0x48 not found: check pull-ups, address, power

#=== GPIO Debug ===
$ cat /sys/kernel/debug/gpio              # all GPIO states
$ gpioinfo gpiochip0                      # detailed info

#=== Clock Debug ===
$ cat /sys/kernel/debug/clk/clk_summary  # clock tree

#=== Regulator Debug ===
$ cat /sys/kernel/debug/regulator/regulator_summary

#=== Power Management Debug ===
$ cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
$ cat /sys/power/pm_print_times           # suspend timing
$ echo 1 > /sys/power/pm_debug_messages
```

---

## 20.4 Ftrace and Tracepoints

```bash
# Ftrace — trace kernel function calls

# Available tracers
$ cat /sys/kernel/debug/tracing/available_tracers

# Trace specific framework functions
$ echo function > /sys/kernel/debug/tracing/current_tracer
$ echo 'v4l2_*' > /sys/kernel/debug/tracing/set_ftrace_filter
$ echo 1 > /sys/kernel/debug/tracing/tracing_on

# Capture trace
$ cat /sys/kernel/debug/tracing/trace

# Tracepoints (framework-specific events)
$ cat /sys/kernel/debug/tracing/available_events | grep -E "v4l2|drm|regulator|clk"

# Enable specific tracepoints
$ echo 1 > /sys/kernel/debug/tracing/events/regulator/regulator_enable/enable
$ echo 1 > /sys/kernel/debug/tracing/events/clk/clk_enable/enable
$ echo 1 > /sys/kernel/debug/tracing/events/dma_fence/enable

# Function graph tracer (shows call hierarchy)
$ echo function_graph > /sys/kernel/debug/tracing/current_tracer
$ echo i2c_transfer > /sys/kernel/debug/tracing/set_graph_function
$ echo 1 > /sys/kernel/debug/tracing/tracing_on
# Result shows:
#   i2c_transfer() {
#     i2c_adapter_lock() {
#       mutex_lock();
#     }
#     __i2c_transfer() {
#       i2c_qcom_geni_xfer();
#     }
#   }
```

---

## 20.5 Dynamic Debug

```bash
# Dynamic debug — enable/disable pr_debug() at runtime

# See all debug print sites
$ cat /sys/kernel/debug/dynamic_debug/control | head -20

# Enable debug prints for a specific driver
$ echo 'module tmp102 +p' > /sys/kernel/debug/dynamic_debug/control

# Enable for a file
$ echo 'file drivers/i2c/i2c-core-base.c +p' > \
    /sys/kernel/debug/dynamic_debug/control

# Enable with function name and line info
$ echo 'module drm +pflmt' > /sys/kernel/debug/dynamic_debug/control
# Flags: p=print, f=function, l=line, m=module, t=thread

# Disable
$ echo 'module tmp102 -p' > /sys/kernel/debug/dynamic_debug/control
```

---

## 20.6 Common Debug Scenarios

```
Scenario 1: Camera Not Working
┌────────────────────────────────────────────────────────────┐
│ Step 1: Check device probe                                  │
│   $ dmesg | grep imx219                                     │
│   $ cat /sys/kernel/debug/devices_deferred                  │
│                                                             │
│ Step 2: Check I2C communication                             │
│   $ i2cdetect -y 2    (is sensor at expected address?)      │
│   $ i2cget -y 2 0x10 0x00  (read chip ID register)         │
│                                                             │
│ Step 3: Check power/clock/reset                             │
│   $ cat /sys/kernel/debug/regulator/regulator_summary       │
│   $ cat /sys/kernel/debug/clk/clk_summary | grep cam       │
│   $ cat /sys/kernel/debug/gpio | grep reset                │
│                                                             │
│ Step 4: Check V4L2 pipeline                                 │
│   $ media-ctl -d /dev/media0 -p                            │
│   $ v4l2-ctl -d /dev/video0 --all                          │
│                                                             │
│ Step 5: Try capture                                         │
│   $ v4l2-ctl -d /dev/video0 --stream-mmap --stream-count=1 │
│   $ dmesg | tail -20                                        │
└────────────────────────────────────────────────────────────┘

Scenario 2: Display Not Showing
┌────────────────────────────────────────────────────────────┐
│ Step 1: DRM device present?                                 │
│   $ ls /dev/dri/card*                                       │
│   $ cat /sys/kernel/debug/dri/0/state                       │
│                                                             │
│ Step 2: Connector status                                    │
│   $ modetest -c  (connected? modes available?)              │
│                                                             │
│ Step 3: Test pattern                                        │
│   $ modetest -s <conn_id>@<crtc_id>:1920x1080              │
│                                                             │
│ Step 4: Check clocks and power                              │
│   $ cat /sys/kernel/debug/clk/clk_summary | grep pixel     │
│   $ cat /sys/kernel/debug/regulator/regulator_summary       │
│                                                             │
│ Step 5: Enable DRM debug                                    │
│   $ echo 0x1f > /sys/module/drm/parameters/debug            │
│   $ dmesg -w (watch for errors)                             │
└────────────────────────────────────────────────────────────┘

Scenario 3: DMA Timeout
┌────────────────────────────────────────────────────────────┐
│ Check 1: Is DMA controller probed?                          │
│   $ ls /sys/class/dma/                                      │
│                                                             │
│ Check 2: DMA mapping correct?                               │
│   Enable CONFIG_DMA_API_DEBUG in kernel                     │
│   $ cat /sys/kernel/debug/dma-api/errors                    │
│                                                             │
│ Check 3: Cache coherency issue?                             │
│   Missing dma_sync_single_for_cpu() after DMA write?        │
│   Use dma_alloc_coherent() instead for debugging            │
│                                                             │
│ Check 4: IRQ firing?                                        │
│   $ cat /proc/interrupts | grep dma                         │
└────────────────────────────────────────────────────────────┘
```

---

## 20.7 devmem — Direct Register Access

```bash
# devmem2 — read/write hardware registers directly (debug only)

# Read a 32-bit register at physical address
$ devmem 0x40005400 32       # I2C controller base

# Write a register
$ devmem 0x40005400 32 0x01  # write 0x01 to register

# CAUTION: devmem bypasses all kernel safety checks
# Only use for debugging — never in production
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| Debugfs | fs/debugfs/ | Debug filesystem |
| Tracing | kernel/trace/ | Ftrace/tracepoints |
| Dynamic debug | lib/dynamic_debug.c | Runtime debug prints |
| DMA debug | kernel/dma/debug.c | DMA API validation |

---

## Interview Questions

**Q1: How do you debug a device that fails to probe?**
A: Systematic approach: (1) Check `dmesg` for error messages — the return code tells the cause (-517=EPROBE_DEFER, -19=ENODEV). (2) For EPROBE_DEFER, check `/sys/kernel/debug/devices_deferred` to see which dependency is missing. (3) For ENODEV, verify the device tree: correct `compatible` string, `reg` address, and `status = "okay"`. (4) Check if the physical device is responding — for I2C, use `i2cdetect`; for SPI, verify CS and clock. (5) Check power: are regulators enabled? Are clocks running? Is the reset pin in the correct state? (6) Enable framework debug: `dyndbg` for the driver module.

**Q2: How do you use ftrace to debug a framework issue?**
A: (1) Set the tracer: `echo function_graph > current_tracer`. (2) Filter to relevant functions: `echo 'i2c_transfer' > set_graph_function`. (3) Enable tracing: `echo 1 > tracing_on`. (4) Trigger the issue. (5) Read trace: `cat trace`. The function_graph tracer shows call hierarchy with timing — you can see which function is slow or failing. For framework events, use tracepoints: `echo 1 > events/regulator/enable` shows every regulator enable with timestamps. Combine with `trace-cmd` or Perfetto for visualization.

**Q3: What is DMA debug and when should you use it?**
A: `CONFIG_DMA_API_DEBUG` enables runtime checking of DMA API usage. It catches: (1) Using a DMA address after unmapping. (2) Double-mapping/unmapping. (3) Direction mismatches (mapping TO_DEVICE but doing FROM_DEVICE sync). (4) Accessing DMA memory with CPU without proper sync. These bugs cause silent data corruption that's extremely hard to find otherwise. Check errors at `/sys/kernel/debug/dma-api/errors`. Always enable during development; disable in production for performance.

---

## Summary

- Systematic debugging: identify the layer (user/framework/driver/hardware)
- Each framework has specific debugfs nodes: clk_summary, regulator_summary, dri/state
- Ftrace function_graph shows call hierarchies with timing
- Dynamic debug enables pr_debug() prints per module/file at runtime
- Probe failures: check dmesg, devices_deferred, DT nodes, bus detection
- DMA debug catches memory mapping errors that cause silent corruption

---

*Next: [Chapter 21 — End-to-End Flow Diagrams](Chapter_21_Flow_Diagrams.md)*
