# Chapter 28: Interview Preparation — Power Management

## Learning Goals
- Master the top PM interview questions with detailed answers
- Practice explaining PM concepts at whiteboard level
- Build confidence with code-level PM questions
- Prepare for system design PM discussions

---

## 1. Fundamental Concepts

### Q1: Explain the relationship P = C × V^2 × f and its impact on PM.

**A:** Dynamic power consumption of CMOS circuits follows P = C × V^2 × f, where C is switched capacitance, V is supply voltage, and f is clock frequency. Key insights:

- Power grows **quadratically** with voltage — reducing voltage from 1.2V to 0.8V saves 56% power
- Power grows **linearly** with frequency — halving frequency halves power
- DVFS (Dynamic Voltage and Frequency Scaling) exploits this: lower frequency allows lower voltage, giving cubic-like power reduction
- Example: Running at 50% frequency with proportionally reduced voltage uses ~12.5% power (0.5 × 0.5^2)
- Static/leakage power (I_leak × V) also matters at small process nodes, motivating power gating

### Q2: What is the difference between clock gating, power gating, and retention?

**A:**
| Feature | Clock Gating | Power Gating | Retention |
|---------|-------------|--------------|-----------|
| Mechanism | Stop clock signal | Cut power supply | Cut logic power, keep SRAM power |
| Dynamic power | Zero | Zero | Zero |
| Leakage power | Still present | Zero | Near zero |
| State | Preserved | **Lost** | Preserved (SRAM only) |
| Entry latency | ~1ns | ~100μs-1ms | ~50-500μs |
| Exit latency | ~1ns | ~100μs-1ms (must restore) | ~10-100μs |
| Use case | Short idle | Long idle | Medium idle |
| Linux mechanism | clk_disable() | regulator_disable() + genpd | genpd idle states |

### Q3: Explain race-to-idle vs DVFS. When is each better?

**A:** Race-to-idle runs at maximum speed to finish work quickly, then enters deep idle. DVFS runs at just-enough speed, consuming less active power but staying active longer.

- **Race-to-idle wins** when: idle power is very low (power gating possible), response latency matters, workload is bursty
- **DVFS wins** when: idle power is significant (no power gating), thermal limits restrict max freq, workload is continuous/streaming
- Modern systems use **both**: EAS places tasks efficiently (DVFS), schedutil scales frequency proportionally, then CPU enters idle when done (race-to-idle within the scaled frequency)

---

## 2. Linux PM Architecture

### Q4: Describe the Linux PM stack from top to bottom.

**A:**
```
Userspace: PowerTOP, thermald, Android PowerManager
    ↓ sysfs (/sys/power/, /sys/class/thermal/, /sys/devices/.../power/)
PM Core: System suspend (kernel/power/), Device PM (drivers/base/power/)
    ↓ Runtime PM, PM QoS, wakeup sources
Subsystems: CPUFreq, CPUIdle, Thermal, Clock, Regulator, OPP, EAS
    ↓ Governors (schedutil, menu, step_wise, IPA)
Drivers: dev_pm_ops (.suspend, .resume, .runtime_suspend, .runtime_resume)
    ↓ Register access, DMA control
Hardware: PMIC, clock tree, power switches, CPU P/C-states, thermal sensors
```

Key design principles:
- **Policy vs mechanism** separation (governors decide, drivers execute)
- **Distributed PM** — each subsystem manages its own power independently
- **Hierarchical** — power domains, device trees, bus ordering

### Q5: How does system suspend work? Walk through the S3 flow.

**A:**
1. `echo mem > /sys/power/state` → `state_store()` → `enter_state()`
2. **Freeze processes**: `freeze_processes()` sends fake signals, tasks enter TASK_FROZEN
3. **Freeze kernel threads**: Freezable kthreads stop
4. **Suspend devices** (child-before-parent order):
   - `.prepare()` — can devices suspend? Return 0 or error
   - `.suspend()` — save state, stop I/O (IRQs enabled)
   - `.suspend_late()` — late cleanup
   - `.suspend_noirq()` — IRQs disabled, final HW access
5. **Disable non-boot CPUs**: CPU hotplug offline
6. **Syscore suspend**: IRQ controllers, timers
7. **Platform enter**: ACPI S3 or ARM PSCI → CPU powers off
8. **[SYSTEM SLEEPS]** — only wakeup IRQ can exit
9. **Resume**: exact reverse order — syscore → devices (parent-first) → thaw processes

### Q6: What is runtime PM? How does it differ from system suspend?

**A:**
| Aspect | Runtime PM | System Suspend |
|--------|-----------|----------------|
| Scope | Individual device | Entire system |
| Trigger | Usage count (get/put) | User/policy (echo mem > state) |
| When | Any time during normal operation | Explicit sleep request |
| Other devices | Stay active | All suspended |
| Processes | Keep running | Frozen |
| Frequency | Thousands of times per second | Rarely (lid close, timeout) |

Runtime PM API:
- `pm_runtime_get_sync()` → increment usage, resume device
- `pm_runtime_put_autosuspend()` → decrement usage, schedule suspend after delay
- Autosuspend prevents rapid on/off cycling (configurable delay in ms)
- System suspend checks `pm_runtime_suspended()` to skip already-suspended devices (direct_complete)

---

## 3. CPU Power Management

### Q7: Explain CPUFreq governors. When would you use each?

**A:**
- **schedutil** (default, recommended): Frequency proportional to scheduler utilization. Required for EAS. `freq = 1.25 × util × max_freq / capacity`. Best for general use.
- **ondemand**: Samples CPU load periodically, jumps to max if load > threshold (80%), gradually decreases. Good for interactive workloads but less efficient than schedutil.
- **conservative**: Like ondemand but ramps up gradually (step-by-step). Lower performance than ondemand, slightly better power.
- **performance**: Always maximum frequency. For benchmarking or latency-critical servers.
- **powersave**: Always minimum frequency. For thermal emergency or extreme battery saving.
- **userspace**: Allows userspace daemon to set exact frequency. For testing or specialized control.

### Q8: How does the cpuidle menu governor select C-states?

**A:**
1. **Predict idle duration**: Check next timer event, apply correction factor from historical accuracy
2. **Check PM QoS**: Maximum allowed exit latency constraint
3. **Iterate states** (deepest to shallowest):
   - `exit_latency ≤ PM QoS latency limit`?
   - `target_residency ≤ predicted idle duration`?
   - Both true → select this state
4. **Enter state**: Driver executes `MWAIT` (Intel) or `WFI`/PSCI (ARM)
5. **Reflect**: After wakeup, update correction factor based on actual vs predicted duration

Break-even: If idle time < `target_residency`, energy wasted on entry/exit exceeds savings.

### Q9: What is EAS and how does it work?

**A:** Energy-Aware Scheduling places tasks on the most energy-efficient CPU using the Energy Model.

**Algorithm** (find_energy_efficient_cpu):
1. Task wakes up → evaluate candidate CPUs
2. For each candidate, compute total system energy:
   - Sum utilization on each performance domain (big/LITTLE)
   - Map utilization to required OPP frequency
   - `energy = OPP_power × busy_ratio`
3. Pick CPU with lowest total system energy
4. If energy difference < 6%, prefer previous CPU (cache warmth)

**Prerequisites**: Heterogeneous CPUs + Energy Model registered + schedutil governor active + `sched_energy_aware=1`

**Auto-disable**: When system is >80% utilized (overutilized), EAS falls back to load balancing for throughput.

---

## 4. Device and Framework Questions

### Q10: How do power domains (genpd) work?

**A:** The Generic Power Domain framework manages SoC power islands:

- **Registration**: Provider driver registers `struct generic_pm_domain` with `power_on/power_off` callbacks
- **Device attachment**: Devices reference domains via DT (`power-domains = <&pd DOMAIN_ID>`)
- **Auto power management**: When all devices in a domain are runtime-suspended, genpd calls `power_off`. When any device resumes, genpd calls `power_on` first
- **Hierarchy**: Domains can have parent-child relationships. Child powers off only if parent allows. Parent stays on while any child is active
- **Idle states**: Domains can have multiple idle states with different latency/residency, similar to CPU C-states

### Q11: Explain DVFS ordering: why voltage before frequency on scale-up?

**A:**
- **Scale UP** (increase performance): Raise voltage FIRST, then increase frequency
  - Reason: Higher frequency needs higher voltage for timing margins. If frequency increases while voltage is too low, logic gates can't switch fast enough → timing violations → data corruption
- **Scale DOWN** (reduce performance): Lower frequency FIRST, then reduce voltage
  - Reason: Reducing voltage while high frequency is active causes same timing violation
- Code pattern:
```c
if (new_freq > old_freq) {
    regulator_set_voltage(vdd, new_volt, new_volt); // voltage up first
    clk_set_rate(clk, new_freq);
} else {
    clk_set_rate(clk, new_freq); // frequency down first
    regulator_set_voltage(vdd, new_volt, new_volt);
}
```

### Q12: How does the thermal framework prevent overheating?

**A:**
1. **Thermal zones** poll temperature sensors periodically (250ms when throttling, 1s otherwise)
2. **Trip points** define thresholds:
   - ACTIVE (e.g., 70°C): Turn on fan
   - PASSIVE (e.g., 85°C): Throttle CPU/GPU frequency
   - HOT (e.g., 95°C): Emergency action
   - CRITICAL (e.g., 100°C): Orderly shutdown
3. **Governors** decide cooling level:
   - step_wise: Increase/decrease cooling state by 1 per interval
   - power_allocator (IPA): PID controller distributes power budget to actors proportionally
4. **Cooling devices** execute:
   - cpufreq_cooling: Caps max CPU frequency via freq_qos
   - Fan: Sets PWM duty cycle
5. **Hysteresis** prevents oscillation: 85°C trip with 5°C hysteresis → deactivate at 80°C
6. Closed loop: measure → decide → actuate → temperature changes → repeat

---

## 5. Debugging and Practical Questions

### Q13: How do you debug a system that hangs during suspend?

**A:**
1. **Serial console**: Add `no_console_suspend` to kernel command line — serial stays active during suspend
2. **Enable debugging**: `echo 1 > /sys/power/pm_debug_messages` and `pm_print_times`
3. **Isolate phase**: Use `pm_test` modes — `echo devices > /sys/power/pm_test` then suspend. If it completes, the problem is after device phase
4. **Identify driver**: Last dmesg line before hang shows the blocking device
5. **Verify**: Unbind suspect driver and retry suspend
6. **Deep analysis**: ftrace with `dpm_*` function filter shows per-callback timing
7. **Advanced**: `CONFIG_DPM_WATCHDOG=y` triggers panic if a device takes >60s to suspend, giving a stack trace
8. **x86 specific**: `pm_trace=1` writes device hash to RTC, survives power cycle

### Q14: A driver accesses hardware after the device is runtime-suspended. How do you diagnose and fix?

**A:**
- **Symptom**: Bus error, register reads return 0xDEADBEEF or all ones, data corruption
- **Diagnosis**: Enable ftrace rpm events — check if rpm_suspend happens before the offending register access. Check sysfs `power/runtime_status` — if "suspended" during access, confirmed
- **Fix**: Wrap all hardware access with runtime PM get/put:
```c
ret = pm_runtime_resume_and_get(dev);  // resume device
if (ret)
    return ret;
val = readl(base + REG);               // safe to access
pm_runtime_mark_last_busy(dev);
pm_runtime_put_autosuspend(dev);       // allow suspend after delay
```
- **Prevention**: Audit all register access paths. Use `pm_runtime_get_if_active()` for optional access paths.

### Q15: How would you reduce power consumption of an embedded Linux device?

**A:** Systematic approach:
1. **Measure baseline**: PowerTOP, RAPL (perf stat -e power/energy-pkg/), or external power monitor
2. **CPU**: Ensure schedutil active, verify deep C-states reached (turbostat/cpuidle stats), enable EAS if heterogeneous
3. **Devices**: Enable runtime PM for all devices (`power/control=auto`), set appropriate autosuspend delays, power-gate unused peripherals
4. **Clock gating**: Disable clocks to unused peripherals (`clk_disable_unprepare`)
5. **Thermal**: Ensure not throttling unnecessarily (check cooling device states)
6. **Peripherals**: USB autosuspend, PCI ASPM, SATA ALPM, WiFi power save
7. **Software**: Reduce wakeup sources (timer coalescing), minimize polling, use event-driven I/O
8. **PowerTOP auto-tune**: `powertop --auto-tune` for quick wins
9. **Verify**: Re-measure and compare. Target deepest C-state residency >90% when idle

---

## 6. System Design Questions

### Q16: Design PM for a new SoC with big.LITTLE CPUs.

**A:**
```
Hardware:
- Define power domains: CPU_BIG, CPU_LITTLE, GPU, DISPLAY, PERIPH, ALWAYS_ON
- Design clock tree with gateable clocks per domain
- PMIC with per-domain voltage rails (buck for CPU/GPU, LDO for analog)
- Temperature sensors at CPU, GPU, battery, board

DT Configuration:
- OPP tables for each CPU cluster and GPU
- Power domain hierarchy (PD_TOP → children)
- Thermal zones with trip points and cooling maps
- CPU idle states with latency/residency via PSCI

Linux Kernel:
- CPUFreq driver (cpufreq-dt or vendor-specific)
- CPUIdle driver with PSCI states
- genpd provider for power domains
- Thermal zone + cpufreq/devfreq cooling
- Energy Model registration for EAS
- Runtime PM in all peripheral drivers

Tuning:
- Schedutil governor for frequency scaling
- Menu/TEO governor for idle state selection
- IPA thermal governor for sustained workloads
- Uclamp for UI boost (min=600) and bg cap (max=200)
```

### Q17: How does Android AAOS handle power management differently from standard Linux?

**A:**
- **Autosleep**: System auto-suspends when all wakeup sources released (`/sys/power/autosleep=mem`)
- **Wakeup sources**: Replace deprecated wake_locks. Drivers hold wakeup sources during critical operations
- **PowerManagerService**: Java service managing screen state, interaction timeouts, wake lock tracking
- **Garage Mode**: Automotive-specific — IVI stays active with display off after ignition-off for OTA/deferred work
- **VHAL Power**: Vehicle HAL controls AP power state based on vehicle signals (ignition, CAN)
- **Battery Saver / Doze**: App-level restrictions on background work, network access, alarms
- **Thermal HAL**: Vendor-implemented thermal monitoring reporting to Android framework

---

## 7. Quick-Fire Questions

| Question | Answer |
|----------|--------|
| What does P = αCV²f mean? | Dynamic power = activity × capacitance × voltage² × frequency |
| Name all ACPI C-states | C0 (active), C1 (halt), C1E (enhanced halt), C3 (sleep), C6 (deep power down) |
| What is s2idle? | Software-only suspend: freeze + device suspend + CPU idle (no firmware S3) |
| Runtime PM vs system suspend? | Per-device vs whole-system. Runtime = during operation. System = explicit sleep |
| What does genpd do? | Auto-manages SoC power domains based on device runtime PM status |
| schedutil formula? | freq = 1.25 × util × max_freq / capacity |
| What is PELT half-life? | ~32ms (utilization decays exponentially) |
| EAS disables when? | System overutilized (>80%), no energy model, no schedutil |
| Thermal trip types? | ACTIVE, PASSIVE, HOT, CRITICAL |
| What is uclamp? | Per-task min/max utilization hint for scheduler/EAS |
| Voltage order on scale-up? | Voltage first, then frequency (timing margin) |
| What is PM QoS? | Latency/throughput constraints preventing too-deep PM states |
| direct_complete? | Skip suspend callbacks if device already runtime-suspended |
| DVFS stands for? | Dynamic Voltage and Frequency Scaling |
| IPA governor uses? | PID controller to distribute power budget to cooling actors |

---

## Summary

- Master P = CV²f, clock/power gating differences, and DVFS ordering for fundamentals
- Know the PM stack layers and be able to draw the architecture
- Understand system suspend flow (freeze → suspend devices → enter sleep → reverse)
- Runtime PM: usage counting, autosuspend, state machine (ACTIVE/SUSPENDED)
- CPU PM: CPUFreq governors (schedutil), CPUIdle governors (menu), EAS task placement
- Frameworks: genpd (power domains), thermal (zones+trips+cooling), regulator, CCF
- Debugging: pm_test modes, ftrace PM events, serial console, DPM watchdog
- System design: combine power domains + OPP + thermal + EAS for complete SoC PM

---

[Previous Chapter: Documentation and References ←](Chapter_27_References.md) | [Back to Index →](00_Master_Index.md)
