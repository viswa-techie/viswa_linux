# Chapter 21: Power Monitoring Tools

## Learning Goals
- Master PowerTOP for power analysis and optimization
- Learn turbostat for CPU power state monitoring
- Know perf for energy profiling
- Understand battery/power supply monitoring tools
- Learn to use trace-cmd and perfetto for PM analysis

---

## 1. PowerTOP

```
  PowerTOP Overview
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  PowerTOP tabs:                                          │
  │                                                           │
  │  [Overview] [Idle Stats] [Freq Stats] [Device Stats]     │
  │  [Tunables]                                              │
  │                                                           │
  │  Overview: Top power consumers + wakeup sources          │
  │  Idle Stats: Time in each C-state per CPU                │
  │  Freq Stats: Time at each frequency per CPU              │
  │  Device Stats: Per-device power state usage              │
  │  Tunables: Auto-tunable PM settings (Good/Bad)           │
  └──────────────────────────────────────────────────────────┘
```

```bash
# Basic usage
sudo powertop

# Generate HTML report
sudo powertop --html=power_report.html

# Auto-tune all settings
sudo powertop --auto-tune
# Sets all "Bad" tunables to "Good":
# - Enables runtime PM for all PCI/USB devices
# - Sets disk I/O scheduler to low-power
# - Enables SATA ALPM
# - Sets USB autosuspend

# Calibrate (run multiple times for accuracy)
sudo powertop --calibrate
# Turns screen on/off, loads/unloads to measure baseline

# CSV output for analysis
sudo powertop --csv=power_data.csv --time=60
```

### 1.1 PowerTOP Output Analysis

```
  PowerTOP Overview Tab Example
  ┌──────────────────────────────────────────────────────────┐
  │  Power est.   Usage       Events/s   Category    Desc    │
  │  ──────────── ─────────── ────────── ─────────── ────── │
  │  3.52 W       100%        ---        Device      Display │
  │  1.23 W       53.2 ms/s   89.2       Process     Xorg    │
  │  890 mW       8.9 ms/s    45.3       Process     chrome  │
  │  234 mW       ---         12.4       Interrupt   i915    │
  │  156 mW       ---         8.9        Timer       tick    │
  │  89 mW        ---         3.2        kWork       flush   │
  │                                                           │
  │  Idle Stats:                                             │
  │  CPU   C0(active)  POLL   C1    C1E    C3     C6        │
  │  CPU0  5.2%        0.0%   1.2%  3.4%   12.3%  78.9%    │
  │  CPU1  3.1%        0.0%   0.8%  2.1%   10.2%  83.8%    │
  │                                                           │
  │  Target: >90% in deepest C-state for idle system        │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Turbostat (Intel)

```bash
# Basic CPU power state monitoring
sudo turbostat --interval 1

# Output:
# Core CPU   Avg_MHz Busy%  Bzy_MHz TSC_MHz  IRQ  C1%  C3%  C6%  Pkg%_pc6
# -    -      234    12.3   1900    2400     456  3.2  8.9  75.6  72.1
# 0    0      456    23.4   1950    2400     234  5.6  12.3 58.7  -
# 0    1      123    6.7    1850    2400     122  2.1  7.8  83.4  -
# 1    2      278    15.6   1780    2400     67   3.4  5.6  75.4  -
# 1    3      89     4.5    1950    2400     33   1.2  10.1 84.2  -

# Key columns:
# Avg_MHz:  Average frequency (weighted by busy time)
# Busy%:    Percent time not in idle state
# Bzy_MHz:  Average frequency when busy
# C1/C3/C6: Percent time in each C-state
# Pkg%_pc6: Package-level C6 (all cores idle)

# Show power consumption (requires RAPL)
sudo turbostat --show \
  Core,CPU,Avg_MHz,Busy%,Bzy_MHz,PkgWatt,CorWatt,RAMWatt

# PkgWatt:  Total package power (W)
# CorWatt:  CPU cores power (W)
# RAMWatt:  DRAM power (W)

# Log to file
sudo turbostat --interval 5 --out turbo_log.csv

# Show only summary (package-level)
sudo turbostat --summary --interval 1
```

---

## 3. Perf for Energy Profiling

```bash
# RAPL energy measurement via perf
sudo perf stat -e power/energy-pkg/,power/energy-cores/,power/energy-ram/ \
    sleep 10

# Output:
# 45.23 Joules  power/energy-pkg/     (4.52 W average)
# 23.45 Joules  power/energy-cores/   (2.35 W average)
# 12.34 Joules  power/energy-ram/     (1.23 W average)

# Per-process energy (approximate)
sudo perf stat -e power/energy-pkg/ -p <PID> -- sleep 5

# CPU frequency distribution
sudo perf stat -e cpu-clock,task-clock,cycles,instructions \
    -- my_workload

# Record frequency transitions
sudo perf record -e power:cpu_frequency -a -- sleep 30
sudo perf script
# cpu_frequency: state=2400000 cpu_id=0
# cpu_frequency: state=800000 cpu_id=0

# C-state residency via perf
sudo perf record -e power:cpu_idle -a -- sleep 10
sudo perf script
# cpu_idle: state=3 cpu_id=0     (enter C3)
# cpu_idle: state=4294967295 cpu_id=0  (exit idle)
```

---

## 4. Battery Monitoring

```bash
# Battery information
cat /sys/class/power_supply/BAT0/status        # Charging/Discharging
cat /sys/class/power_supply/BAT0/capacity      # SOC percentage
cat /sys/class/power_supply/BAT0/voltage_now   # Current voltage (μV)
cat /sys/class/power_supply/BAT0/current_now   # Current draw (μA)
cat /sys/class/power_supply/BAT0/power_now     # Power draw (μW)
cat /sys/class/power_supply/BAT0/energy_full   # Full capacity (μWh)
cat /sys/class/power_supply/BAT0/energy_now    # Current energy (μWh)

# Continuous monitoring script
while true; do
    P=$(cat /sys/class/power_supply/BAT0/power_now 2>/dev/null)
    V=$(cat /sys/class/power_supply/BAT0/voltage_now 2>/dev/null)
    C=$(cat /sys/class/power_supply/BAT0/capacity 2>/dev/null)
    echo "$(date +%H:%M:%S) Power: $((P/1000))mW Voltage: $((V/1000))mV SOC: ${C}%"
    sleep 5
done

# upower (desktop systems)
upower -i /org/freedesktop/UPower/devices/battery_BAT0
# Shows detailed battery info including time-to-empty
```

---

## 5. Trace-cmd for PM Analysis

```bash
# Record PM events
sudo trace-cmd record -e power -e rpm -e thermal &
# Run workload...
kill %1

# Generate report
trace-cmd report | head -100

# Record suspend/resume cycle
sudo trace-cmd record -e 'power:suspend_resume' \
    -e 'power:device_pm_callback_*' -e 'rpm:*' &
echo mem > /sys/power/state
trace-cmd report

# Filter specific events
trace-cmd report -F 'rpm_suspend || rpm_resume' | \
    grep my-device

# Generate KernelShark compatible output
trace-cmd record -e power -o pm_trace.dat
kernelshark pm_trace.dat
# Visual timeline of PM events
```

---

## 6. Sysfs PM Monitoring Summary

```
  Sysfs PM Monitoring Points
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  CPU Frequency:                                          │
  │  /sys/devices/system/cpu/cpu*/cpufreq/                  │
  │    scaling_cur_freq        Current frequency             │
  │    scaling_governor        Active governor               │
  │    stats/time_in_state     Time at each freq             │
  │    stats/total_trans       Transition count              │
  │                                                           │
  │  CPU Idle:                                               │
  │  /sys/devices/system/cpu/cpu*/cpuidle/state*/            │
  │    name                    C-state name                  │
  │    time                    Total time (μs)               │
  │    usage                   Entry count                   │
  │    latency                 Exit latency (μs)             │
  │                                                           │
  │  Runtime PM:                                             │
  │  /sys/devices/.../power/                                 │
  │    runtime_status          active/suspended              │
  │    runtime_usage           Reference count               │
  │    runtime_active_time     Active time (ms)              │
  │    runtime_suspended_time  Suspended time (ms)           │
  │    autosuspend_delay_ms    Autosuspend timeout           │
  │                                                           │
  │  Thermal:                                                │
  │  /sys/class/thermal/thermal_zone*/                       │
  │    temp                    Current temperature           │
  │    policy                  Governor                      │
  │    trip_point_*            Trip temperatures             │
  │                                                           │
  │  Power Supply:                                           │
  │  /sys/class/power_supply/*/                              │
  │    status, capacity, voltage_now, current_now            │
  │                                                           │
  │  Regulators:                                             │
  │  /sys/class/regulator/*/                                 │
  │    state, microvolts, num_users                          │
  └──────────────────────────────────────────────────────────┘
```

---

## 7. Scripted PM Audit

```bash
#!/bin/bash
# Quick PM health check script

echo "=== CPU Frequency ==="
for cpu in /sys/devices/system/cpu/cpu[0-9]*; do
    freq=$(cat $cpu/cpufreq/scaling_cur_freq 2>/dev/null)
    gov=$(cat $cpu/cpufreq/scaling_governor 2>/dev/null)
    [ -n "$freq" ] && echo "  $(basename $cpu): ${freq}KHz ($gov)"
done

echo "=== Thermal Zones ==="
for tz in /sys/class/thermal/thermal_zone*; do
    type=$(cat $tz/type)
    temp=$(cat $tz/temp)
    echo "  $type: $((temp/1000))°C"
done

echo "=== Cooling Devices ==="
for cd in /sys/class/thermal/cooling_device*; do
    type=$(cat $cd/type)
    cur=$(cat $cd/cur_state)
    max=$(cat $cd/max_state)
    echo "  $type: $cur/$max"
done

echo "=== Runtime PM (suspended devices) ==="
for dev in /sys/devices/platform/*/power/runtime_status; do
    status=$(cat $dev 2>/dev/null)
    [ "$status" = "suspended" ] && echo "  $(dirname $(dirname $dev) | xargs basename): $status"
done

echo "=== Wakeup Sources (active) ==="
grep -v "^$" /sys/kernel/debug/wakeup_sources 2>/dev/null | \
    awk '$2 > 0 {print "  " $1 ": " $2 " active"}'
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `tools/power/x86/turbostat/` | Turbostat source |
| `tools/perf/` | Perf tool source |
| `tools/power/cpupower/` | cpupower utility |
| `kernel/trace/trace_events_power.c` | Power trace events |
| `drivers/base/power/sysfs.c` | Device power sysfs |
| `drivers/cpufreq/cpufreq_stats.c` | CPUFreq statistics |
| `drivers/cpuidle/sysfs.c` | CPUIdle sysfs |

---

## Interview Questions

**Q1: How would you identify the top power consumers on a Linux system?**
**A:** Use PowerTOP's Overview tab to see ranked power consumers (processes, interrupts, timers, devices). For CPU: turbostat shows per-core busy% and C-state residency — CPUs not reaching deep C-states waste power. For devices: check runtime PM status in sysfs — devices stuck in "active" may need runtime PM. For precision: perf RAPL events (power/energy-pkg/) measure actual package power. Combine with ftrace power events to correlate wakeups with consumers.

**Q2: How do you measure the power impact of a code change?**
**A:** Use perf RAPL energy events: `perf stat -e power/energy-pkg/` before and after. Run identical workloads and compare joules consumed. For mobile without RAPL: measure battery drain rate (power_now) over fixed-duration tests. For CPU PM changes: compare C-state residency (cpuidle sysfs or turbostat) and frequency distributions (cpufreq stats). Use trace-cmd to verify PM transitions happen as expected. Multiple runs for statistical significance.

**Q3: What does it mean when PowerTOP shows a device as "Bad" in Tunables?**
**A:** "Bad" means the device is not using an available power-saving feature. Common examples: PCI/USB runtime PM not enabled (control=on instead of auto), SATA link power management disabled, NMI watchdog enabled (prevents deepest C-state), audio codec not powering down. `powertop --auto-tune` enables all these. In production, selectively apply — some devices have bugs with aggressive PM (e.g., USB devices that disconnect during autosuspend).

---

## Summary

- PowerTOP: comprehensive power analysis, identifies top consumers, auto-tunes PM settings
- Turbostat: real-time CPU frequency, C-state, and RAPL power monitoring (Intel)
- Perf RAPL: measure actual energy consumption in joules for benchmarking
- Battery sysfs: voltage, current, SOC, power draw monitoring
- trace-cmd: record and analyze PM events (frequency, idle, runtime PM, thermal)
- Sysfs provides per-device, per-CPU, thermal, and regulator PM status
- Combine multiple tools: PowerTOP (what), turbostat (CPU detail), trace-cmd (when/why)

---

[Previous Chapter: Power Management Debugging ←](Chapter_20_PM_Debugging.md) | [Next Chapter: Kernel Source Code Map →](Chapter_22_Source_Code_Map.md)
