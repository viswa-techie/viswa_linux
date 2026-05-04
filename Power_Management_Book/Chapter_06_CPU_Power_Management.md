# Chapter 6: CPU Power Management

## Learning Goals
- Understand CPU power management as a whole — DVFS, idle, and thermal
- Learn how the kernel coordinates frequency, idle, and thermal subsystems
- Know the key kernel APIs for CPU PM
- Understand per-core vs per-cluster power management
- Learn heterogeneous CPU topology power management

---

## 1. CPU Power Management Overview

```
  CPU Power Management Subsystems
  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  ┌─── CPUFreq (Active Power) ────────────────────────────┐  │
  │  │  Controls voltage and frequency of running CPUs        │  │
  │  │  When: CPU is in C0 (executing code)                   │  │
  │  │  Goal: Match performance to workload demand            │  │
  │  │  Key: P-states (performance states)                    │  │
  │  └────────────────────────────────────────────────────────┘  │
  │                                                               │
  │  ┌─── CPUIdle (Idle Power) ──────────────────────────────┐  │
  │  │  Controls idle state depth when CPU has nothing to do  │  │
  │  │  When: CPU runs out of work (idle loop)                │  │
  │  │  Goal: Enter deepest state that saves energy           │  │
  │  │  Key: C-states (idle states)                           │  │
  │  └────────────────────────────────────────────────────────┘  │
  │                                                               │
  │  ┌─── Thermal (Safety) ──────────────────────────────────┐  │
  │  │  Limits CPU performance to stay within thermal budget  │  │
  │  │  When: Temperature exceeds thresholds                  │  │
  │  │  Goal: Prevent damage while maintaining operation      │  │
  │  │  Key: Trip points, cooling devices                     │  │
  │  └────────────────────────────────────────────────────────┘  │
  │                                                               │
  │  ┌─── EAS (Optimization) ────────────────────────────────┐  │
  │  │  Scheduler considers energy impact of task placement   │  │
  │  │  When: Task needs a CPU (wakeup, migration)            │  │
  │  │  Goal: Minimize total system energy                    │  │
  │  │  Key: Energy model, capacity                           │  │
  │  └────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────┘
```

---

## 2. DVFS (Dynamic Voltage and Frequency Scaling)

### 2.1 DVFS Concept

```
  DVFS: Adjust both voltage and frequency together
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Why both? Transistor switching speed depends on voltage  │
  │  Lower voltage → slower max frequency → much less power  │
  │                                                           │
  │  Voltage-Frequency Relationship:                         │
  │                                                           │
  │  Freq     │         ╱                                    │
  │  (GHz)    │       ╱                                      │
  │           │     ╱                                        │
  │  2.0──────│── ╱──── F_max at V=1.2V                     │
  │           │ ╱                                            │
  │  1.5──────╱────── F_max at V=1.0V                       │
  │         ╱ │                                              │
  │  1.0──╱───│────── F_max at V=0.8V                       │
  │     ╱     │                                              │
  │  0.5╱─────│────── F_max at V=0.6V                       │
  │           │                                              │
  │           └──────────────────────── Voltage (V)          │
  │           0.6   0.8   1.0   1.2                          │
  │                                                           │
  │  Power at each point:                                    │
  │  P(1.2V, 2.0G) = α×C×1.44×2G = 2.88 (relative)        │
  │  P(0.8V, 1.0G) = α×C×0.64×1G = 0.64 (relative)        │
  │  → 4.5x power reduction!                                │
  └──────────────────────────────────────────────────────────┘
```

### 2.2 DVFS Transition Sequence

```c
/* Simplified DVFS transition in kernel */
static int cpu_dvfs_set_target(struct cpufreq_policy *policy,
                                unsigned int target_freq)
{
    unsigned int old_freq = policy->cur;
    struct dev_pm_opp *opp;
    unsigned long new_volt, old_volt;

    /* Look up voltage for target frequency */
    opp = dev_pm_opp_find_freq_ceil(cpu_dev, &target_freq);
    new_volt = dev_pm_opp_get_voltage(opp);

    if (target_freq > old_freq) {
        /* Scaling UP: voltage first, then frequency */
        regulator_set_voltage(cpu_reg, new_volt, new_volt);
        clk_set_rate(cpu_clk, target_freq);
    } else {
        /* Scaling DOWN: frequency first, then voltage */
        clk_set_rate(cpu_clk, target_freq);
        regulator_set_voltage(cpu_reg, new_volt, new_volt);
    }

    return 0;
}
```

---

## 3. CPU Idle Management

### 3.1 The Idle Loop

```c
/* kernel/sched/idle.c — CPU idle entry point */
static void do_idle(void)
{
    while (!need_resched()) {
        /* Check if there's work to do */
        if (cpu_idle_force_poll || tick_check_broadcast_expired()) {
            cpu_relax();
            continue;
        }

        /* Ask cpuidle to select and enter an idle state */
        cpuidle_idle_call();
    }
}

/* cpuidle selects appropriate C-state based on:
 * 1. Expected idle duration (prediction)
 * 2. PM QoS latency constraints
 * 3. Available idle states for this CPU
 * 4. Previous idle behavior (for prediction)
 */
```

### 3.2 Idle State Selection Flow

```
  CPU Idle Flow
  ┌──────────────────────────────────────────────────────┐
  │                                                       │
  │  Scheduler: No runnable tasks                        │
  │     │                                                │
  │     ▼                                                │
  │  do_idle()                                           │
  │     │                                                │
  │     ▼                                                │
  │  cpuidle_idle_call()                                 │
  │     │                                                │
  │     ├── Governor selects state                       │
  │     │   ├── menu governor: predict idle duration     │
  │     │   │   based on timer expiration + history      │
  │     │   │                                            │
  │     │   └── TEO governor: use timer distribution     │
  │     │       statistics                               │
  │     │                                                │
  │     ├── Check PM QoS constraints                     │
  │     │   └── Filter states with latency > constraint  │
  │     │                                                │
  │     ├── Check disabled states                        │
  │     │   └── Skip states with disable=1               │
  │     │                                                │
  │     ▼                                                │
  │  cpuidle_enter_state(drv, dev, selected_state)       │
  │     │                                                │
  │     ▼                                                │
  │  target_state->enter(dev, drv, index)                │
  │  ├── Execute HLT/MWAIT instruction (x86)            │
  │  └── Execute WFI/WFE instruction (ARM)              │
  │     │                                                │
  │     ▼  (interrupt or timer fires)                    │
  │  CPU wakes up, returns from enter()                  │
  │     │                                                │
  │     ▼                                                │
  │  Update idle statistics                              │
  │  Return to scheduler                                 │
  └──────────────────────────────────────────────────────┘
```

---

## 4. CPU Hotplug for Power Savings

### 4.1 CPU Hotplug Power Management

```
  CPU Hotplug as PM Mechanism
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  When load is very low, entire CPUs can be offlined:     │
  │                                                           │
  │  8-core system under light load:                         │
  │  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐│
  │  │CPU0│ │CPU1│ │CPU2│ │CPU3│ │CPU4│ │CPU5│ │CPU6│ │CPU7││
  │  │ ON │ │ ON │ │ ON │ │ ON │ │ ON │ │ ON │ │ ON │ │ ON ││
  │  └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘│
  │  Power: 8 × idle_power (even if mostly idle)             │
  │                                                           │
  │  After hotplug-off unused CPUs:                          │
  │  ┌────┐ ┌────┐ ┌────┐ ┌────┐                            │
  │  │CPU0│ │CPU1│ │CPU2│ │CPU3│  CPU4-7: OFFLINE            │
  │  │ ON │ │ ON │ │ ON │ │ ON │  (fully power gated)        │
  │  └────┘ └────┘ └────┘ └────┘                            │
  │  Power: 4 × idle_power + 0 (offline CPUs draw ~0)       │
  │                                                           │
  │  $ echo 0 > /sys/devices/system/cpu/cpu7/online          │
  │  $ echo 1 > /sys/devices/system/cpu/cpu7/online          │
  │                                                           │
  │  Trade-off: Hotplug latency (~100ms) vs power savings    │
  │  Used in: Android (big core offlining), servers           │
  └──────────────────────────────────────────────────────────┘
```

### 4.2 Hotplug vs Deep C-States

```
  C6 vs Hotplug comparison:
  ┌──────────────────┬──────────────────┬─────────────────┐
  │ Aspect           │ C6 (deep idle)   │ CPU Hotplug Off │
  ├──────────────────┼──────────────────┼─────────────────┤
  │ Power savings    │ High (~95%)      │ Maximum (~99%)  │
  │ Wake latency     │ ~200μs           │ ~100ms          │
  │ State preserved  │ Retention SRAM   │ No (cold boot)  │
  │ Scheduler aware  │ Yes (idle)       │ Yes (removed)   │
  │ Interrupts       │ Can still target │ Migrated away   │
  │ Timer ticks      │ Stopped (NO_HZ)  │ None            │
  │ Use case         │ Short idle        │ Prolonged idle  │
  └──────────────────┴──────────────────┴─────────────────┘
```

---

## 5. Heterogeneous CPU Power Management

### 5.1 big.LITTLE / DynamIQ PM Strategy

```
  Heterogeneous CPU PM (ARM big.LITTLE)
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Workload       → CPU Selection    → DVFS                │
  │                                                           │
  │  Very light     → 1 LITTLE core   → Low OPP              │
  │  (UI idle)         (others off)                           │
  │                                                           │
  │  Light          → 4 LITTLE cores  → Mid OPP              │
  │  (web browse)                                             │
  │                                                           │
  │  Medium         → 4 LITTLE cores  → High OPP             │
  │  (app switch)     + 1-2 big cores   + Low OPP             │
  │                                                           │
  │  Heavy          → 4 LITTLE + 4 big → Max OPP             │
  │  (gaming)         (all cores active)                      │
  │                                                           │
  │  Power profile:                                          │
  │  ┌────────────────────────────────────────────┐          │
  │  │  Phone @ idle:    ~50mW (1 LITTLE, low OPP)│          │
  │  │  Phone @ active: ~500mW (2-4 cores, mid)  │          │
  │  │  Phone @ gaming:  ~5W (all cores, max)    │          │
  │  └────────────────────────────────────────────┘          │
  │                                                           │
  │  Capacity values (relative):                             │
  │  LITTLE (A55): capacity = 380 (38% of max)              │
  │  big (A78):    capacity = 1024 (100%)                    │
  │  Scheduler uses these for task placement                 │
  └──────────────────────────────────────────────────────────┘
```

### 5.2 CPU Capacity and Frequency

```c
/* Capacity computation for heterogeneous CPUs */
/*
 * arch_scale_cpu_capacity(cpu) returns the CPU's
 * maximum capacity relative to the most capable CPU.
 *
 * Example (Cortex-A55 + Cortex-A78):
 * A78 at 2.4 GHz: capacity = 1024 (reference)
 * A55 at 1.8 GHz: capacity = 380
 *
 * If A78 is thermally throttled to 1.2 GHz:
 * Effective capacity = 1024 * (1200/2400) = 512
 * This is "thermal pressure" — scheduler sees reduced capacity
 */

/* arch_scale_freq_capacity(cpu) = current_freq / max_freq */
/* If CPU runs at 1.5 GHz out of 2.0 GHz max:
 * freq_scale = 1500/2000 * 1024 = 768 */
```

---

## 6. CPU PM Monitoring

### 6.1 Key Monitoring Interfaces

```
  CPU Power Monitoring
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  CPUFreq:                                                │
  │  $ cat /sys/devices/system/cpu/cpu0/cpufreq/             │
  │      scaling_cur_freq      # Current frequency (KHz)     │
  │      scaling_min_freq      # Min allowed                  │
  │      scaling_max_freq      # Max allowed                  │
  │      scaling_governor      # Active governor              │
  │      cpuinfo_min_freq      # HW minimum                  │
  │      cpuinfo_max_freq      # HW maximum                  │
  │      scaling_available_frequencies                        │
  │      scaling_available_governors                          │
  │                                                           │
  │  CPUIdle:                                                │
  │  $ cat /sys/devices/system/cpu/cpu0/cpuidle/             │
  │      state0/name     # "POLL"                            │
  │      state0/usage    # Entry count                       │
  │      state0/time     # Total time in state (μs)          │
  │      state1/name     # "C1"                              │
  │      state1/latency  # Exit latency (μs)                 │
  │      ...                                                  │
  │                                                           │
  │  perf stat (hardware counters):                          │
  │  $ perf stat -e power/energy-pkg/ sleep 5                │
  │  → Package energy consumed in 5 seconds                  │
  │                                                           │
  │  turbostat (Intel-specific):                             │
  │  $ turbostat --interval 1                                │
  │  → Per-core C-state residency, frequency, temperature   │
  └──────────────────────────────────────────────────────────┘
```

### 6.2 Runtime Power Estimation

```c
/* Simple CPU power estimation */
/* For heterogeneous systems, each CPU type has different power curve */

struct cpu_power_model {
    unsigned int max_freq_khz;
    unsigned int max_power_mw;     /* Power at max OPP */
    unsigned int idle_power_mw;    /* Power at deepest idle */
};

/* Approximate current power based on frequency and utilization */
static unsigned int estimate_cpu_power(struct cpu_power_model *model,
                                        unsigned int freq_khz,
                                        unsigned int util_pct)
{
    /* Power scales roughly with V² * f ≈ f³ (since V ∝ f) */
    unsigned long freq_ratio = (freq_khz * 1000) / model->max_freq_khz;
    unsigned long power = model->max_power_mw;

    power = (power * freq_ratio * freq_ratio * freq_ratio) >> 30;
    power = (power * util_pct) / 100;

    return power + model->idle_power_mw;
}
```

---

## 7. CPU PM Coordination

### 7.1 Cross-Subsystem Interactions

```
  CPU PM Subsystem Interactions
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Scheduler                                               │
  │  ├── Updates utilization signal ──► CPUFreq (schedutil)  │
  │  ├── Places tasks on CPUs ──► Affects load distribution  │
  │  └── EAS considers energy ──► Energy Model               │
  │                                                           │
  │  CPUFreq                                                 │
  │  ├── Reads utilization ◄── Scheduler                    │
  │  ├── Programs frequency ──► Clock Framework              │
  │  ├── Programs voltage ──► Regulator Framework            │
  │  └── Respects thermal cap ◄── Thermal Framework         │
  │                                                           │
  │  CPUIdle                                                 │
  │  ├── Called from idle loop ◄── Scheduler                │
  │  ├── Enters HW C-state ──► CPU Hardware                  │
  │  └── Respects PM QoS ◄── PM QoS Framework              │
  │                                                           │
  │  Thermal                                                 │
  │  ├── Reads temperature ◄── Thermal sensors              │
  │  ├── Sets max frequency ──► CPUFreq (clamp)             │
  │  └── Reports thermal pressure ──► Scheduler             │
  │                                                           │
  │  Power Domains                                           │
  │  ├── Power gates idle clusters ──► HW power switches    │
  │  └── Triggered by all cores entering C6+ ◄── CPUIdle   │
  └──────────────────────────────────────────────────────────┘
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `kernel/sched/idle.c` | CPU idle entry point (do_idle) |
| `kernel/sched/cpufreq_schedutil.c` | Schedutil governor |
| `drivers/cpufreq/cpufreq.c` | CPUFreq core framework |
| `drivers/cpuidle/cpuidle.c` | CPUIdle core framework |
| `drivers/base/power/runtime.c` | Runtime PM |
| `kernel/power/energy_model.c` | Energy model |
| `drivers/thermal/cpu_cooling.c` | CPU thermal cooling |
| `include/linux/cpufreq.h` | CPUFreq structures and API |
| `include/linux/cpuidle.h` | CPUIdle structures |
| `include/linux/energy_model.h` | Energy model API |

---

## Interview Questions

**Q1: How do CPUFreq and CPUIdle work together?**
**A:** CPUFreq manages active power by scaling frequency/voltage when the CPU is executing (C0). CPUIdle manages idle power by selecting C-states when the CPU has no work. They are complementary: a CPU at C0 uses CPUFreq-selected P-state, and when it enters idle, CPUIdle selects the C-state. The schedutil governor ties them together — it uses scheduler utilization to set frequency, and when utilization drops to zero, the scheduler calls the idle loop where CPUIdle takes over.

**Q2: What is the trade-off between CPU hotplug and deep C-states?**
**A:** Deep C-states (C6) power-gate the core with ~200μs wake latency and save ~95% power. CPU hotplug fully removes the CPU from the scheduler with ~100ms bring-up latency but saves ~99% power (no timer interrupts, no scheduler overhead). Use C-states for short/unpredictable idle periods and hotplug for prolonged idle (minutes+). Android uses hotplug for big cores during screen-off; servers use it for power capping.

**Q3: Explain DVFS voltage/frequency ordering.**
**A:** When increasing frequency: raise voltage first (to meet timing margins), then increase frequency. When decreasing: lower frequency first, then reduce voltage. If frequency increases before voltage is sufficient, transistors can't switch fast enough, causing timing violations and data corruption. The kernel OPP framework and clock/regulator drivers enforce this ordering automatically.

**Q4: How does the thermal framework interact with CPUFreq?**
**A:** When temperature exceeds a passive trip point, the thermal framework acts as a "cooling device" that caps CPUFreq's maximum allowed frequency. It calls cpufreq_cooling to reduce max_freq, limiting how high the governor can scale. Additionally, the scheduler sees "thermal pressure" — reduced effective CPU capacity — so it redistributes tasks away from throttled cores. This prevents thermal runaway while maintaining system stability.

**Q5: What is CPU capacity in the context of power management?**
**A:** CPU capacity is a normalized value (0-1024) representing a CPU's maximum computational capability relative to the most powerful CPU in the system. On heterogeneous platforms (big.LITTLE), LITTLE cores might have capacity 380 and big cores 1024. The scheduler uses capacity for task placement: small tasks go to low-capacity (energy-efficient) cores. Thermal throttling reduces effective capacity, causing the scheduler to adjust task placement dynamically.

---

## Summary

- CPU PM involves four subsystems: CPUFreq (active power), CPUIdle (idle power), Thermal (safety), EAS (optimization)
- DVFS scales voltage and frequency together — power reduces with V²×f (near-cubic)
- Voltage must increase before frequency (up-scaling) and decrease after (down-scaling)
- CPUIdle selects C-states based on predicted idle duration, PM QoS constraints, and state latencies
- CPU hotplug provides maximum power savings at the cost of high resume latency
- Heterogeneous CPUs (big.LITTLE) use capacity values for energy-efficient task placement
- All CPU PM subsystems interact: scheduler → CPUFreq/CPUIdle, thermal → CPUFreq, EAS → energy model

---

[Previous Chapter: PM Architecture ←](Chapter_05_PM_Architecture.md) | [Next Chapter: CPUFreq Subsystem →](Chapter_07_CPUFreq.md)
