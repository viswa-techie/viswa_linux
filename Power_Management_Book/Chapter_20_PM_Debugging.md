# Chapter 20: Power Management Debugging

## Learning Goals
- Master PM debugging techniques using kernel infrastructure
- Learn to diagnose suspend/resume failures and hangs
- Know runtime PM debugging with sysfs and ftrace
- Understand thermal throttling diagnosis
- Learn to analyze PM-related performance issues

---

## 1. Suspend/Resume Debugging

### 1.1 Identifying Slow Devices

```bash
# Enable suspend/resume timing
echo 1 > /sys/power/pm_print_times

# Trigger suspend
echo mem > /sys/power/state

# dmesg output shows per-device timing:
# PM: suspend of devices complete after 234.567 msecs
# PM: late suspend of devices complete after 12.345 msecs  
# PM: noirq suspend of devices complete after 5.678 msecs
#
# Individual device timing:
# my-i2c 1-0048: suspend took 150.234 msecs
# usb 1-1: suspend took 45.678 msecs

# Detailed device timing
echo 1 > /sys/power/pm_debug_messages
# Shows every callback entry/exit with timestamps
```

### 1.2 Suspend Test Modes

```bash
# Test suspend phases individually
# (stops at specified phase, then resumes)

echo freezer > /sys/power/pm_test
echo mem > /sys/power/state
# Tests only process freezer — skips device suspend

echo devices > /sys/power/pm_test
echo mem > /sys/power/state
# Tests device suspend/resume — doesn't actually sleep

echo platform > /sys/power/pm_test
echo mem > /sys/power/state
# Tests platform prepare — stops before entering sleep

echo processors > /sys/power/pm_test
echo mem > /sys/power/state
# Tests CPU disable — stops before final suspend

echo core > /sys/power/pm_test
echo mem > /sys/power/state
# Full path but stops at final step

# Reset to normal
echo none > /sys/power/pm_test
```

### 1.3 Last Resume Reason

```bash
# What woke the system?
cat /sys/power/pm_wakeup_irq
# Shows IRQ number that caused wakeup

cat /proc/interrupts | grep <irq_number>
# Identify which device triggered the wakeup

# Wakeup source statistics
cat /sys/kernel/debug/wakeup_sources
# name          active_count  event_count  wakeup_count  ...
# eventpoll     0             0            0
# rtc0          1             1            1
# power_button  3             3            3
```

---

## 2. Ftrace for PM Debugging

```bash
# Trace suspend/resume events
cd /sys/kernel/debug/tracing

# Enable PM trace events
echo 1 > events/power/cpu_frequency/enable
echo 1 > events/power/cpu_idle/enable
echo 1 > events/power/suspend_resume/enable
echo 1 > events/power/clock_enable/enable
echo 1 > events/power/clock_disable/enable

# Enable device PM events
echo 1 > events/rpm/rpm_suspend/enable
echo 1 > events/rpm/rpm_resume/enable
echo 1 > events/rpm/rpm_return_int/enable

# Thermal events
echo 1 > events/thermal/thermal_temperature/enable
echo 1 > events/thermal/thermal_zone_trip/enable

# Start tracing
echo 1 > tracing_on

# Trigger suspend
echo mem > /sys/power/state

# Read trace
cat trace

# Example output:
# suspend_resume: suspend_enter[s2idle] begin
# device_pm_callback_start: my-device suspend, parent=platform
# device_pm_callback_end:   my-device suspend, err=0 time=1234
# cpu_idle: state=4294967295 cpu_id=0    # exit idle
# cpu_idle: state=3 cpu_id=0             # enter C3
# suspend_resume: suspend_enter[s2idle] end
```

### 2.1 Function Graph Tracing for PM

```bash
# Trace specific PM functions
echo function_graph > current_tracer

# Filter to PM functions
echo 'dpm_*' > set_ftrace_filter
echo 'pm_runtime_*' >> set_ftrace_filter
echo '__device_suspend' >> set_ftrace_filter

echo 1 > tracing_on
echo mem > /sys/power/state
echo 0 > tracing_on

cat trace
# Shows call graph with timing:
#  3)               |  dpm_suspend() {
#  3)               |    __device_suspend() {
#  3)               |      driver_suspend() {
#  3) + 15.234 us   |        my_driver_suspend();
#  3) + 18.567 us   |      }
#  3) + 22.123 us   |    }
#  3) + 234.567 us  |  }
```

---

## 3. Runtime PM Debugging

```bash
# Per-device runtime PM status
cat /sys/devices/.../power/runtime_status
# active | suspended | suspending | resuming

cat /sys/devices/.../power/runtime_usage
# Reference count (>0 means actively used)

cat /sys/devices/.../power/runtime_active_time
# Total time in active state (ms)

cat /sys/devices/.../power/runtime_suspended_time
# Total time in suspended state (ms)

cat /sys/devices/.../power/autosuspend_delay_ms
# Autosuspend timeout

cat /sys/devices/.../power/control
# auto | on
# "auto" = runtime PM enabled
# "on"   = runtime PM disabled (always active)

# Disable runtime PM for debugging
echo on > /sys/devices/.../power/control

# Force runtime suspend for testing
echo auto > /sys/devices/.../power/control
echo 0 > /sys/devices/.../power/autosuspend_delay_ms
```

### 3.1 Runtime PM Ftrace

```bash
# Trace runtime PM transitions
echo 1 > /sys/kernel/debug/tracing/events/rpm/enable

# Output shows:
# rpm_suspend: my-device flags-4 usage=0 disable=0
# rpm_return_int: my-device usagecount=0 status=2 (suspended)
# rpm_resume: my-device flags-0 usage=1 disable=0
# rpm_return_int: my-device usagecount=1 status=0 (active)
```

---

## 4. Debugging Suspend Hangs

```
  Suspend Hang Diagnosis Flowchart
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  System hangs during suspend?                            │
  │       │                                                   │
  │       ▼                                                   │
  │  [Enable pm_debug_messages, pm_print_times]              │
  │       │                                                   │
  │       ▼                                                   │
  │  [Use pm_test modes to isolate phase]                    │
  │       │                                                   │
  │       ├─ Freezer hangs? → Check for unfreeeable tasks    │
  │       │   (TASK_UNINTERRUPTIBLE in non-PM code)          │
  │       │                                                   │
  │       ├─ Device suspend hangs? → Check last device in    │
  │       │   dmesg, bisect drivers (unbind suspect driver)  │
  │       │                                                   │
  │       ├─ Late/noirq hangs? → Likely IRQ or DMA issue    │
  │       │   Driver doing DMA after IRQs disabled           │
  │       │                                                   │
  │       └─ Platform/core hangs? → ACPI/PSCI firmware issue │
  │           Check firmware/ATF logs                         │
  │                                                           │
  │  Serial console is ESSENTIAL for debugging hangs:        │
  │  console=ttyS0,115200 no_console_suspend                 │
  │                                                           │
  │  Kernel config:                                          │
  │  CONFIG_PM_DEBUG=y                                       │
  │  CONFIG_PM_ADVANCED_DEBUG=y                              │
  │  CONFIG_PM_SLEEP_DEBUG=y                                 │
  └──────────────────────────────────────────────────────────┘
```

```bash
# Kernel command line for PM debugging
# no_console_suspend → Keep serial console alive during suspend
# initcall_debug → Show every init function timing
# pm_debug_messages → Verbose PM messages

# Example driver hang investigation
echo devices > /sys/power/pm_test
echo mem > /sys/power/state
# Last line in dmesg before hang identifies the blocking driver

# Unbind suspect driver
echo "my-device" > /sys/bus/platform/drivers/my_driver/unbind
echo mem > /sys/power/state
# If suspend succeeds, that driver is the problem
```

---

## 5. Thermal Debugging

```bash
# Monitor thermal in real-time
watch -n 1 'cat /sys/class/thermal/thermal_zone*/temp'

# Check if throttling is active
for zone in /sys/class/thermal/thermal_zone*; do
    echo "$(cat $zone/type): $(cat $zone/temp)m°C"
    for trip in $zone/trip_point_*_temp; do
        num=$(echo $trip | grep -o '[0-9]*_temp' | cut -d_ -f1)
        echo "  trip $num: $(cat $trip)m°C ($(cat ${trip/_temp/_type}))"
    done
done

# Check cooling device states
for cd in /sys/class/thermal/cooling_device*; do
    echo "$(cat $cd/type): state=$(cat $cd/cur_state)/$(cat $cd/max_state)"
done

# Trace thermal events
echo 1 > /sys/kernel/debug/tracing/events/thermal/enable
cat /sys/kernel/debug/tracing/trace_pipe
# thermal_temperature: thermal_zone=cpu-thermal id=0 temp_input=85500
# cdev_update: type=cpufreq target=3
```

---

## 6. Power Domain Debugging

```bash
# Generic power domain status
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary

# Output:
# domain                    status   children
#     /device                runtime status
# ──────────────────────────────────────────
# pd-gpu                    off
#     /devices/gpu@20000     suspended
# pd-isp                    off
#     /devices/isp@30000     suspended
# pd-display                on
#     /devices/display@40000 active
# pd-top                    on
#     pd-gpu
#     pd-isp
#     pd-display

# Shows:
# - Which domains are on/off
# - Which devices are in each domain
# - Device runtime PM status within domain
```

---

## 7. CPUFreq/CPUIdle Debugging

```bash
# CPUFreq statistics
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state
# 500000 12345678    ← time in μs at each frequency
# 800000 23456789
# 1200000 34567890

cat /sys/devices/system/cpu/cpu0/cpufreq/stats/total_trans
# 56789              ← number of frequency transitions

# CPUIdle statistics
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/name
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/time    # total μs
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/usage   # entry count
cat /sys/devices/system/cpu/cpu0/cpuidle/state*/rejected # rejected count

# Disable specific C-state for testing
echo 1 > /sys/devices/system/cpu/cpu0/cpuidle/state3/disable

# Trace frequency changes
echo 1 > /sys/kernel/debug/tracing/events/power/cpu_frequency/enable
# cpu_frequency: state=1200000 cpu_id=0
```

---

## 8. Common PM Debug Kernel Configs

```
  Essential PM Debug Configurations
  ┌──────────────────────────────────────────────────┐
  │                                                   │
  │  CONFIG_PM_DEBUG=y                               │
  │    Enables /sys/power/pm_test                    │
  │    Enables PM debug messages                     │
  │                                                   │
  │  CONFIG_PM_ADVANCED_DEBUG=y                      │
  │    Extra sysfs attributes in /sys/.../power/     │
  │                                                   │
  │  CONFIG_PM_SLEEP_DEBUG=y                         │
  │    PM sleep path debug messages                  │
  │                                                   │
  │  CONFIG_FTRACE=y                                 │
  │  CONFIG_FUNCTION_TRACER=y                        │
  │    Function-level tracing for PM paths           │
  │                                                   │
  │  CONFIG_PM_TRACE=y (x86 only)                    │
  │    Uses RTC to trace last device before hang     │
  │    echo 1 > /sys/power/pm_trace                  │
  │    After hang: dmesg shows "Magic number: ..."   │
  │    with hash identifying the failing device      │
  │                                                   │
  │  CONFIG_SUSPEND_FREEZER=y                        │
  │    Process freezer debug                         │
  │                                                   │
  │  CONFIG_DPM_WATCHDOG=y                           │
  │    Watchdog triggers if device suspend takes     │
  │    too long (CONFIG_DPM_WATCHDOG_TIMEOUT=60)     │
  └──────────────────────────────────────────────────┘
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `kernel/power/suspend.c` | Suspend debug infrastructure |
| `kernel/power/main.c` | PM sysfs (pm_test, pm_debug_messages) |
| `drivers/base/power/main.c` | Device PM timing, DPM watchdog |
| `drivers/base/power/wakeup.c` | Wakeup source debugfs |
| `kernel/trace/trace_events_power.c` | Power trace events |
| `drivers/base/power/domain.c` | genpd debugfs |
| `include/trace/events/power.h` | Power tracepoint definitions |

---

## Interview Questions

**Q1: How do you debug a system that hangs during suspend?**
**A:** Step 1: Enable serial console with `no_console_suspend` kernel parameter. Step 2: Enable `pm_debug_messages` and `pm_print_times` in sysfs for detailed logging. Step 3: Use `pm_test` modes (freezer, devices, platform, processors, core) to isolate which phase hangs. Step 4: If device phase hangs, the last dmesg line identifies the blocking driver. Step 5: Unbind that driver and verify suspend works. Step 6: Use ftrace (dpm_* functions) to get per-callback timing. Step 7: Enable CONFIG_DPM_WATCHDOG to auto-detect stuck drivers.

**Q2: How do you verify runtime PM is working correctly for a device?**
**A:** Check sysfs: `power/runtime_status` should show transitions between active/suspended. `power/runtime_usage` should be 0 when idle. `power/runtime_suspended_time` should increase when device is not in use. Use ftrace rpm events to trace transitions. Verify `power/control` is "auto" (not "on"). Check `power/autosuspend_delay_ms` for reasonable timeout. Monitor power consumption with and without runtime PM to confirm power savings.

**Q3: What is the pm_trace mechanism and when is it useful?**
**A:** `pm_trace` (x86 only, CONFIG_PM_TRACE) writes device identification to the RTC during suspend. If the system hangs and resets, on next boot dmesg shows a hash matching the last device that was being suspended. This is valuable when the hang prevents any console output — the RTC survives power cycles. Caveat: it corrupts the RTC time, so the system clock will be wrong after using pm_trace.

---

## Summary

- `pm_print_times` and `pm_debug_messages` are first-line suspend debug tools
- `pm_test` modes isolate which suspend phase causes problems
- Ftrace power/rpm/thermal events provide fine-grained PM transition tracing
- Runtime PM debugging: sysfs power/ attributes show status, timing, usage count
- Thermal debugging: monitor zone temperatures, trip points, and cooling device states
- Power domain debugging: `pm_genpd_summary` shows domain hierarchy and status
- Serial console with `no_console_suspend` is essential for suspend hang debugging
- DPM watchdog (CONFIG_DPM_WATCHDOG) auto-detects drivers that take too long to suspend

---

[Previous Chapter: Energy-Aware Scheduling ←](Chapter_19_EAS.md) | [Next Chapter: Power Monitoring Tools →](Chapter_21_PM_Tools.md)
