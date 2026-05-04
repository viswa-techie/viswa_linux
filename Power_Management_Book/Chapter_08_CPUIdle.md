# Chapter 8: CPUIdle Subsystem

## Learning Goals
- Understand the CPUIdle framework architecture
- Learn CPUIdle governors and their idle prediction algorithms
- Know how idle states are registered and selected
- Understand the relationship between CPUIdle and tickless kernel
- Learn to configure and debug CPU idle behavior

---

## 1. CPUIdle Framework Architecture

```
  CPUIdle Framework
  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  Scheduler: no runnable tasks → do_idle()                    │
  │      │                                                        │
  │      ▼                                                        │
  │  ┌─── CPUIdle Core (drivers/cpuidle/cpuidle.c) ───────────┐ │
  │  │                                                          │ │
  │  │  cpuidle_idle_call()                                     │ │
  │  │      │                                                   │ │
  │  │      ├── 1. Governor selects state index                 │ │
  │  │      │      cpuidle_select(drv, dev, &stop_tick)         │ │
  │  │      │                                                   │ │
  │  │      ├── 2. Possibly stop tick (NO_HZ)                   │ │
  │  │      │      tick_nohz_idle_stop_tick()                   │ │
  │  │      │                                                   │ │
  │  │      ├── 3. Enter selected state                         │ │
  │  │      │      cpuidle_enter(drv, dev, index)               │ │
  │  │      │      → driver->states[index].enter()              │ │
  │  │      │                                                   │ │
  │  │      └── 4. Record actual residency                      │ │
  │  │             cpuidle_reflect(drv, dev, index)             │ │
  │  └──────────────────────────────────────────────────────────┘ │
  │                                                               │
  │  ┌─── Governor ────────┐  ┌─── Driver ──────────────────┐  │
  │  │  menu / TEO / ladder │  │  acpi_idle / intel_idle     │  │
  │  │                      │  │  arm_idle / psci_idle       │  │
  │  │  Selects which state │  │  Programs HW to enter state │  │
  │  │  to enter            │  │  (HLT/MWAIT/WFI)           │  │
  │  └──────────────────────┘  └─────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────┘
```

---

## 2. CPUIdle Governors

### 2.1 Menu Governor

```c
/* drivers/cpuidle/governors/menu.c */
/* Default governor — predicts idle duration using multiple factors */

static int menu_select(struct cpuidle_driver *drv,
                        struct cpuidle_device *dev,
                        bool *stop_tick)
{
    struct menu_device *data = this_cpu_ptr(&menu_devices);
    int latency_req = cpuidle_governor_latency_req(dev->cpu);
    unsigned int predicted_us;

    /* Step 1: Get expected next timer event */
    unsigned int expected_us = ktime_to_us(
        tick_nohz_get_sleep_length(&delta_tick));

    /* Step 2: Predict actual idle duration
     * Uses: timer expiration, recent history, correction factors
     * The prediction adjusts for the fact that interrupts
     * (not just timers) can wake the CPU */
    predicted_us = get_typical_interval(data, expected_us);

    /* Apply correction factor from recent accuracy */
    predicted_us = (predicted_us * data->correction_factor[bucket])
                    >> RESOLUTION_SHIFT;

    /* Step 3: Select deepest state that fits */
    for (i = CPUIDLE_DRIVER_STATE_START; i < drv->state_count; i++) {
        struct cpuidle_state *s = &drv->states[i];

        if (s->disabled || s->exit_latency_ns > latency_req)
            continue;

        /* Target residency must be <= predicted idle time */
        if (s->target_residency_ns <= predicted_us * 1000)
            idx = i;
    }

    /* Step 4: Decide whether to stop the tick */
    *stop_tick = (predicted_us > TICK_USEC);

    return idx;
}
```

### 2.2 TEO Governor (Timer Events Oriented)

```c
/* drivers/cpuidle/governors/teo.c */
/* Uses timer event distribution statistics instead of prediction */

/*
 * TEO maintains a histogram of actual idle durations for each
 * idle state's target residency range:
 *
 * For each state i, TEO tracks:
 *   - hits:   times actual idle was within state i's range
 *   - misses: times actual idle was shorter than state i's range
 *
 *               State0     State1     State2     State3
 *              (POLL)     (C1)       (C3)       (C6)
 * target_res:  0μs        10μs       200μs      1000μs
 *
 * Histogram:
 * ┌─────────┬──────────┬──────────┬──────────┬──────────┐
 * │ Range   │ 0-10μs   │ 10-200μs │200-1000μs│ >1000μs  │
 * │ Hits    │   15     │   42     │   28     │   15     │
 * │ Misses  │   60     │   18     │    2     │    0     │
 * └─────────┴──────────┴──────────┴──────────┴──────────┘
 *
 * TEO selects the state where misses are minimized,
 * avoiding states that are too deep (wasted energy on exit).
 */

static int teo_select(struct cpuidle_driver *drv,
                       struct cpuidle_device *dev,
                       bool *stop_tick)
{
    struct teo_cpu *cpu_data = this_cpu_ptr(&teo_cpus);
    int latency_req = cpuidle_governor_latency_req(dev->cpu);
    unsigned int idx_intercept_sum = 0;
    unsigned int idx_recent = 0;

    /* Analyze hit/miss statistics for each state */
    for (i = 0; i < drv->state_count; i++) {
        /* Check if too many "intercepts" (woke before target) */
        if (cpu_data->state_bins[i].intercepts > threshold)
            idx_intercept_sum++;
    }

    /* Select state based on distribution analysis */
    /* Prefer state with best hit rate */
}
```

### 2.3 Ladder Governor

```
  Ladder Governor (for periodic tick systems):
  ┌──────────────────────────────────────────────────────┐
  │                                                       │
  │  Simple step-up/step-down based on last idle time:   │
  │                                                       │
  │  if (last_idle > promote_threshold)                  │
  │      move to deeper state                             │
  │                                                       │
  │  if (last_idle < demote_threshold)                   │
  │      move to shallower state                          │
  │                                                       │
  │  State:  C1 ← → C3 ← → C6                          │
  │               ↑        ↑                              │
  │            promote  promote                           │
  │            demote   demote                            │
  │               ↓        ↓                              │
  │                                                       │
  │  Only used with CONFIG_HZ_PERIODIC (legacy)          │
  │  menu/TEO are preferred with tickless kernel         │
  └──────────────────────────────────────────────────────┘
```

---

## 3. CPUIdle Drivers

### 3.1 Intel Idle Driver

```c
/* drivers/idle/intel_idle.c */
/* Table-driven idle states for Intel CPUs */

static struct cpuidle_state skl_cstates[] = {
    {
        .name = "C1",
        .desc = "MWAIT 0x00",
        .flags = MWAIT2flg(0x00),
        .exit_latency = 2,           /* μs */
        .target_residency = 2,        /* μs */
        .enter = &intel_idle,
    },
    {
        .name = "C1E",
        .desc = "MWAIT 0x01",
        .flags = MWAIT2flg(0x01) | CPUIDLE_FLAG_ALWAYS_ENABLE,
        .exit_latency = 10,
        .target_residency = 20,
    },
    {
        .name = "C3",
        .desc = "MWAIT 0x10",
        .flags = MWAIT2flg(0x10) | CPUIDLE_FLAG_TLB_FLUSHED,
        .exit_latency = 70,
        .target_residency = 100,
    },
    {
        .name = "C6",
        .desc = "MWAIT 0x20",
        .flags = MWAIT2flg(0x20) | CPUIDLE_FLAG_TLB_FLUSHED,
        .exit_latency = 85,
        .target_residency = 200,
    },
    {
        .name = "C7s",
        .desc = "MWAIT 0x33",
        .flags = MWAIT2flg(0x33) | CPUIDLE_FLAG_TLB_FLUSHED,
        .exit_latency = 124,
        .target_residency = 800,
    },
    {
        .name = "C8",
        .desc = "MWAIT 0x40",
        .flags = MWAIT2flg(0x40) | CPUIDLE_FLAG_TLB_FLUSHED,
        .exit_latency = 200,
        .target_residency = 800,
    },
    { /* End marker */ }
};

/* Enter idle via MWAIT instruction */
static __cpuidle int intel_idle(struct cpuidle_device *dev,
                                 struct cpuidle_driver *drv,
                                 int index)
{
    unsigned long ecx = 1; /* break on interrupt */
    unsigned int cstate = cycledata->states[index].flags;

    mwait_idle_with_hints(eax, ecx);
    /* CPU halts here until interrupt arrives */

    return index;
}
```

### 3.2 ARM PSCI Idle Driver

```c
/* drivers/cpuidle/cpuidle-psci.c */
/* Uses PSCI firmware for ARM CPU idle states */

static int psci_enter_idle_state(struct cpuidle_device *dev,
                                  struct cpuidle_driver *drv,
                                  int idx)
{
    u32 *states = __this_cpu_read(psci_power_state);
    u32 state = cycledata->idle_states[idx];

    return CPU_PM_CPU_IDLE_ENTER_PARAM(psci_cpu_suspend_enter,
                                        idx, state);
}

/* PSCI CPU suspend: SMC call to ATF firmware */
static int psci_cpu_suspend_enter(unsigned long state)
{
    /* state encodes power level + state type:
     * StateID[15:0]: platform-specific state ID
     * PowerLevel[25:24]: 0=core, 1=cluster, 2=system
     * StateType[30]: 0=standby(WFI), 1=power-down
     */
    return psci_ops.cpu_suspend(state, __pa_symbol(cpu_resume));
}
```

---

## 4. Idle State Properties

### 4.1 Key Properties

```
  Idle State Properties
  ┌────────────────────┬─────────────────────────────────────┐
  │ Property           │ Meaning                              │
  ├────────────────────┼─────────────────────────────────────┤
  │ exit_latency       │ Worst-case time to resume from this │
  │                    │ state (microseconds)                 │
  ├────────────────────┼─────────────────────────────────────┤
  │ target_residency   │ Minimum time in state to make it    │
  │                    │ energy-worthwhile (includes entry +  │
  │                    │ exit overhead)                       │
  ├────────────────────┼─────────────────────────────────────┤
  │ power_usage        │ Power consumed in this state (μW)   │
  │                    │ (-1 if not available)                │
  ├────────────────────┼─────────────────────────────────────┤
  │ flags              │ State characteristics:               │
  │                    │ CPUIDLE_FLAG_POLLING (busy-wait)     │
  │                    │ CPUIDLE_FLAG_TLB_FLUSHED            │
  │                    │ CPUIDLE_FLAG_TIMER_STOP             │
  └────────────────────┴─────────────────────────────────────┘

  Selection criteria:
  ┌──────────────────────────────────────────────────────────┐
  │  For state to be selected:                               │
  │  1. exit_latency ≤ PM QoS latency constraint            │
  │  2. target_residency ≤ predicted idle duration           │
  │  3. State is not disabled                                │
  │  4. Deeper state = more power savings if (1) and (2) met│
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Tickless Interaction

### 5.1 CPUIdle and NO_HZ Integration

```
  CPUIdle + Tickless Coordination
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Step 1: Governor predicts idle duration                 │
  │          (partly from next timer expiration)             │
  │          predicted = 5ms                                  │
  │                                                           │
  │  Step 2: Governor advises on tick behavior               │
  │          stop_tick = (predicted > tick_period)?          │
  │          If yes → stop periodic tick                      │
  │          If no  → keep tick running                       │
  │                                                           │
  │  Step 3: CPU enters selected C-state                     │
  │                                                           │
  │  Why not always stop tick?                               │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │  Stopping/restarting the tick has overhead:      │    │
  │  │  - Program timer hardware                        │    │
  │  │  - Synchronize with other CPUs                   │    │
  │  │                                                  │    │
  │  │  For short idle (<1 tick period):                │    │
  │  │  Overhead of stopping tick > savings             │    │
  │  │  → Keep tick running, use shallow C-state        │    │
  │  │                                                  │    │
  │  │  For long idle (>1 tick period):                 │    │
  │  │  Stop tick → enter deep C-state → big savings    │    │
  │  └──────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. Coupled Idle States

### 6.1 Multi-Core Coordination

```
  Coupled Idle States (ARM SoC)
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Problem: Some idle states require ALL cores in a        │
  │  cluster to be idle (e.g., cluster power-down):          │
  │                                                           │
  │  ┌── Cluster ────────────────────────────────────┐       │
  │  │  CPU0: idle (wants C6)                         │       │
  │  │  CPU1: running (in C0)                         │       │
  │  │  CPU2: idle (wants C6)                         │       │
  │  │  CPU3: idle (wants C6)                         │       │
  │  │                                                │       │
  │  │  Cannot power-gate cluster!                    │       │
  │  │  CPU1 is still active.                         │       │
  │  │  CPU0,2,3 enter core-level C6 only.            │       │
  │  └────────────────────────────────────────────────┘       │
  │                                                           │
  │  When ALL cores idle:                                    │
  │  ┌── Cluster ────────────────────────────────────┐       │
  │  │  CPU0: idle (C6)                               │       │
  │  │  CPU1: idle (C6) ← last to enter              │       │
  │  │  CPU2: idle (C6)                               │       │
  │  │  CPU3: idle (C6)                               │       │
  │  │                                                │       │
  │  │  → Last core triggers cluster power-down       │       │
  │  │  → Shared L2 flushed, cluster PLL off          │       │
  │  └────────────────────────────────────────────────┘       │
  │                                                           │
  │  Implemented via: PSCI (ARM), genpd, or custom driver    │
  └──────────────────────────────────────────────────────────┘
```

---

## 7. CPUIdle Sysfs Interface

```bash
# View all idle states for CPU0
ls /sys/devices/system/cpu/cpu0/cpuidle/

# State details
cat /sys/devices/system/cpu/cpu0/cpuidle/state0/name       # POLL
cat /sys/devices/system/cpu/cpu0/cpuidle/state1/name       # C1
cat /sys/devices/system/cpu/cpu0/cpuidle/state2/name       # C1E
cat /sys/devices/system/cpu/cpu0/cpuidle/state3/name       # C6

# View latency and residency
cat /sys/devices/system/cpu/cpu0/cpuidle/state3/latency    # exit_latency (μs)
cat /sys/devices/system/cpu/cpu0/cpuidle/state3/residency  # target_residency

# Usage statistics
cat /sys/devices/system/cpu/cpu0/cpuidle/state3/usage      # entry count
cat /sys/devices/system/cpu/cpu0/cpuidle/state3/time       # total time (μs)
cat /sys/devices/system/cpu/cpu0/cpuidle/state3/rejected    # rejected entries
cat /sys/devices/system/cpu/cpu0/cpuidle/state3/above       # should've been shallower
cat /sys/devices/system/cpu/cpu0/cpuidle/state3/below       # should've been deeper

# Disable a specific C-state
echo 1 > /sys/devices/system/cpu/cpu0/cpuidle/state3/disable

# Governor selection
cat /sys/devices/system/cpu/cpuidle/current_governor
echo teo > /sys/devices/system/cpu/cpuidle/current_governor

# View available governors
cat /sys/devices/system/cpu/cpuidle/available_governors
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `drivers/cpuidle/cpuidle.c` | CPUIdle core framework |
| `drivers/cpuidle/driver.c` | Driver registration |
| `drivers/cpuidle/governor.c` | Governor management |
| `drivers/cpuidle/sysfs.c` | Sysfs interface |
| `drivers/cpuidle/governors/menu.c` | Menu governor |
| `drivers/cpuidle/governors/teo.c` | TEO governor |
| `drivers/cpuidle/governors/ladder.c` | Ladder governor |
| `drivers/idle/intel_idle.c` | Intel CPU idle driver |
| `drivers/cpuidle/cpuidle-psci.c` | ARM PSCI idle driver |
| `drivers/cpuidle/cpuidle-arm.c` | Generic ARM idle |
| `include/linux/cpuidle.h` | CPUIdle API and structures |
| `kernel/sched/idle.c` | Scheduler idle loop |

---

## Interview Questions

**Q1: How does the menu governor predict idle duration?**
**A:** The menu governor uses multiple factors: (1) next timer expiration — the primary hint for expected idle time, (2) historical idle durations — a rolling window of recent idle periods to detect patterns, (3) correction factors — adjusts for the fact that interrupts (not just timers) can wake the CPU. It computes a "typical interval" from recent history and applies correction factors based on which bucket (short/medium/long) the prediction falls in. It then selects the deepest state whose target_residency fits within this prediction.

**Q2: What is the TEO governor and how does it differ from menu?**
**A:** TEO (Timer Events Oriented) uses statistical analysis of timer event distribution rather than prediction. It maintains per-state hit/miss counters: a "hit" means the CPU idled long enough for that state, a "miss" (intercept) means it woke too early. TEO selects states where the intercept rate is acceptably low. Unlike menu, TEO doesn't try to predict specific idle duration — it makes probabilistic decisions based on historical state selection success rates.

**Q3: Explain the relationship between CPUIdle and the tickless kernel.**
**A:** CPUIdle and NO_HZ work together: the governor predicts idle duration partly from the next timer expiration (which NO_HZ provides). If the predicted idle is long enough (> tick period), the governor advises stopping the tick, allowing deeper C-states. Without NO_HZ, the periodic tick would wake the CPU every 1-4ms, preventing deep idle. The governor's `stop_tick` output coordinates this: short idles keep the tick, long idles stop it.

**Q4: What are coupled idle states?**
**A:** Coupled idle states are power states that require coordination across multiple CPUs — typically cluster-level power-down on ARM SoCs. A cluster can only be powered off when ALL cores in it are idle. The last core to enter idle triggers the cluster power-down (L2 flush, PLL off). On ARM, this is managed via PSCI firmware — the OS tells each core to enter idle, and PSCI/ATF handles the cluster-level coordination. This saves more power than per-core idle alone.

**Q5: How do you disable a deep C-state and why might you need to?**
**A:** Write 1 to `/sys/devices/system/cpu/cpuN/cpuidle/stateN/disable`. Reasons: (1) latency-sensitive workloads (real-time, audio, trading) where wakeup latency from deep C-states causes deadline violations, (2) debugging — deep C-states can mask or cause issues with interrupt handling, (3) known hardware bugs — some platforms have buggy C-state entry/exit. Alternatively, use PM QoS to set a CPU_DMA_LATENCY constraint, which automatically filters states with excessive exit latency.

---

## Summary

- CPUIdle framework separates governors (state selection) from drivers (hardware control)
- Menu governor predicts idle duration using timers, history, and correction factors
- TEO governor uses statistical hit/miss analysis of timer events per state
- Ladder governor uses simple step-up/step-down (legacy, periodic tick systems)
- Intel idle driver uses MWAIT; ARM uses WFI/PSCI firmware calls
- State selection considers: exit_latency vs PM QoS, target_residency vs predicted idle
- Tickless kernel coordination: short idle → keep tick; long idle → stop tick for deep C-state
- Coupled idle states require all cores idle for cluster power-down
- Sysfs provides per-state usage statistics and disable controls

---

[Previous Chapter: CPUFreq ←](Chapter_07_CPUFreq.md) | [Next Chapter: Device Power Management →](Chapter_09_Device_PM.md)
