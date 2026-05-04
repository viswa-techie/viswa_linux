# Chapter 7: CPUFreq Subsystem

## Learning Goals
- Understand the CPUFreq framework architecture
- Learn all CPUFreq governors and their algorithms
- Know how CPUFreq drivers interact with hardware
- Understand frequency policies and their sysfs interface
- Learn to write and configure CPUFreq components

---

## 1. CPUFreq Architecture

```
  CPUFreq Framework Architecture
  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  ┌───── Userspace ─────────────────────────────────────────┐ │
  │  │  /sys/devices/system/cpu/cpuN/cpufreq/                  │ │
  │  │  ├── scaling_governor    (select governor)               │ │
  │  │  ├── scaling_cur_freq    (current frequency)             │ │
  │  │  ├── scaling_min_freq    (minimum policy freq)           │ │
  │  │  ├── scaling_max_freq    (maximum policy freq)           │ │
  │  │  ├── scaling_setspeed    (userspace governor target)     │ │
  │  │  └── scaling_available_governors                         │ │
  │  └──────────────────────┬──────────────────────────────────┘ │
  │                         │                                     │
  │  ┌──────────────────────▼──────────────────────────────────┐ │
  │  │              CPUFreq Core (cpufreq.c)                    │ │
  │  │                                                          │ │
  │  │  struct cpufreq_policy {                                │ │
  │  │      unsigned int min, max;  /* Policy limits */         │ │
  │  │      unsigned int cur;       /* Current freq */          │ │
  │  │      struct cpufreq_governor *governor;                  │ │
  │  │      struct cpufreq_cpuinfo cpuinfo;  /* HW limits */   │ │
  │  │      cpumask_var_t cpus;    /* CPUs sharing policy */    │ │
  │  │  };                                                      │ │
  │  └─────────────┬────────────────────┬──────────────────────┘ │
  │                │                    │                         │
  │  ┌─────────────▼──────────┐  ┌─────▼──────────────────────┐ │
  │  │   GOVERNORS (Policy)   │  │   DRIVERS (Mechanism)       │ │
  │  │                        │  │                              │ │
  │  │  schedutil             │  │  intel_pstate               │ │
  │  │  ondemand              │  │  acpi-cpufreq               │ │
  │  │  conservative          │  │  amd-pstate                 │ │
  │  │  powersave             │  │  cpufreq-dt (device tree)   │ │
  │  │  performance           │  │  qcom-cpufreq-hw            │ │
  │  │  userspace             │  │  mediatek-cpufreq           │ │
  │  └────────────────────────┘  └──────────────────────────────┘ │
  └──────────────────────────────────────────────────────────────┘
```

---

## 2. CPUFreq Governors

### 2.1 Governor Overview

| Governor | Algorithm | Use Case | Latency |
|----------|-----------|----------|---------|
| **performance** | Always max frequency | Benchmarks, low-latency | N/A |
| **powersave** | Always min frequency | Max battery, idle systems | N/A |
| **ondemand** | Sample load, jump to max if > threshold | Legacy general use | ~10ms |
| **conservative** | Sample load, step up/down gradually | Smooth transitions | ~20ms |
| **userspace** | User sets frequency via sysfs | Manual control | N/A |
| **schedutil** | Scheduler-driven, utilization-based | Modern default | ~1ms |

### 2.2 Schedutil Governor (Modern Default)

```c
/* kernel/sched/cpufreq_schedutil.c */
/* Schedutil: frequency = utilization × max_freq / max_capacity */

static void sugov_update_single(struct update_util_data *hook,
                                 u64 time, unsigned int flags)
{
    struct sugov_cpu *sg_cpu = container_of(hook, struct sugov_cpu,
                                            update_util);
    struct sugov_policy *sg_policy = sg_cpu->sg_policy;
    unsigned long util, max;
    unsigned int next_f;

    /* Get current utilization and max capacity */
    util = cpu_util_cfs(sg_cpu->cpu);
    max  = arch_scale_cpu_capacity(sg_cpu->cpu);

    /* Apply 1.25 margin (run slightly faster than needed) */
    util = min(util + (util >> 2), max);

    /* Map utilization to frequency */
    next_f = get_next_freq(sg_policy, util, max);
    /*
     * get_next_freq:
     * freq = (util / max) × policy->cpuinfo.max_freq
     *
     * Example: util=512, max=1024, max_freq=2000MHz
     * freq = (512/1024) × 2000 = 1000 MHz
     */

    /* Send frequency request to driver */
    if (next_f != sg_policy->next_freq)
        cpufreq_driver_fast_switch(sg_policy->policy, next_f);
}
```

```
  Schedutil vs Ondemand:
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Ondemand (sampling-based):                              │
  │  Time ──────────────────────────────────────────►        │
  │  Load: ░░░░████████████░░░░░░░█████░░░░░░░░░░           │
  │  Freq: ────┐  ┌───────┐  ┌──┐  ┌──┐                    │
  │       min──┘  └──max──┘  └──┘  └──┘                    │
  │  Problem: 10ms sampling delay → misses short bursts      │
  │                                                           │
  │  Schedutil (scheduler-driven):                           │
  │  Time ──────────────────────────────────────────►        │
  │  Load: ░░░░████████████░░░░░░░█████░░░░░░░░░░           │
  │  Freq: ──┐┌────────────┐  ┌┐┌─────┐                    │
  │       min┘└──────max───┘  └┘└──────┘                    │
  │  Advantage: Immediate response, no sampling delay        │
  └──────────────────────────────────────────────────────────┘
```

### 2.3 Ondemand Governor

```c
/* drivers/cpufreq/cpufreq_ondemand.c */
/* Samples CPU utilization every sampling_rate ms */

static void od_check_cpu(struct cpufreq_policy *policy)
{
    unsigned int load;

    /* Calculate load since last sample */
    load = get_cpu_load();  /* Percentage 0-100 */

    if (load > up_threshold) {     /* Default: 95% */
        /* Jump directly to max frequency */
        cpufreq_driver_target(policy, policy->max,
                              CPUFREQ_RELATION_H);
    } else {
        /* Proportional: freq = min + (max - min) × load / 100 */
        unsigned int freq = policy->min +
            (policy->max - policy->min) * load / 100;
        cpufreq_driver_target(policy, freq,
                              CPUFREQ_RELATION_L);
    }
}

/* Tunable parameters via sysfs:
 * /sys/devices/system/cpu/cpufreq/ondemand/
 *   sampling_rate          (μs, default ~10ms)
 *   up_threshold           (%, default 95)
 *   sampling_down_factor   (delay downward transition)
 *   io_is_busy             (count I/O wait as busy)
 */
```

### 2.4 Conservative Governor

```c
/* drivers/cpufreq/cpufreq_conservative.c */
/* Steps frequency up/down gradually rather than jumping */

static void cs_check_cpu(struct cpufreq_policy *policy)
{
    unsigned int load = get_cpu_load();

    if (load > up_threshold) {
        /* Step UP by freq_step percentage */
        unsigned int step = (policy->max * freq_step) / 100;
        cpufreq_driver_target(policy, policy->cur + step,
                              CPUFREQ_RELATION_H);
    } else if (load < down_threshold) {
        /* Step DOWN by freq_step percentage */
        unsigned int step = (policy->max * freq_step) / 100;
        cpufreq_driver_target(policy, policy->cur - step,
                              CPUFREQ_RELATION_L);
    }
    /* If between thresholds: maintain current frequency */
}

/* Tunables:
 * up_threshold    (default 80)
 * down_threshold  (default 20)
 * freq_step       (default 5%)
 * sampling_rate   (μs)
 */
```

---

## 3. CPUFreq Drivers

### 3.1 Generic DT-Based Driver

```c
/* drivers/cpufreq/cpufreq-dt.c */
/* Generic CPUFreq driver for device-tree based platforms */

static int dt_cpufreq_init(struct cpufreq_policy *policy)
{
    struct device *cpu_dev;
    struct dev_pm_opp *opp;

    cpu_dev = get_cpu_device(policy->cpu);

    /* Initialize OPP table from device tree */
    dev_pm_opp_of_cpumask_add_table(policy->cpus);

    /* Get clock and regulator handles */
    cpu_clk = devm_clk_get(cpu_dev, NULL);
    cpu_reg = devm_regulator_get(cpu_dev, "cpu");

    /* Populate frequency table from OPPs */
    dev_pm_opp_init_cpufreq_table(cpu_dev, &freq_table);

    policy->freq_table = freq_table;
    policy->cpuinfo.transition_latency = /* from DT or OPP */;

    return 0;
}

static int dt_cpufreq_set_target(struct cpufreq_policy *policy,
                                  unsigned int index)
{
    struct dev_pm_opp *opp;
    unsigned long freq = freq_table[index].frequency * 1000;

    opp = dev_pm_opp_find_freq_ceil(cpu_dev, &freq);
    /* dev_pm_opp_set_rate() handles voltage/frequency ordering */
    return dev_pm_opp_set_rate(cpu_dev, freq);
}
```

### 3.2 Intel P-State Driver

```c
/* drivers/cpufreq/intel_pstate.c */
/* Dual mode: software governor or hardware-managed (HWP) */

/*
 * Active mode: Intel driver acts as both driver AND governor
 * Does not use generic governors (ondemand/schedutil)
 *
 * Passive mode: Intel driver + generic governors
 * Works with schedutil, ondemand, etc.
 *
 * HWP mode: Hardware manages frequency autonomously
 * OS provides hints via MSR_HWP_REQUEST
 */

/* HWP request programming */
static void intel_pstate_hwp_set(struct cpudata *cpu)
{
    u64 value;

    value = HWP_MIN_PERF(cpu->pstate.min_pstate) |
            HWP_MAX_PERF(cpu->pstate.max_pstate) |
            HWP_DESIRED_PERF(cpu->pstate.desired_pstate) |
            HWP_EPP(cpu->epp);

    wrmsrl_on_cpu(cpu->cpu, MSR_HWP_REQUEST, value);
}

/* EPP (Energy Performance Preference):
 * 0   = Maximum performance
 * 128 = Balanced
 * 255 = Maximum energy savings
 *
 * /sys/devices/system/cpu/cpufreq/policy0/
 *   energy_performance_preference = "performance" | "balance_performance"
 *                                    | "balance_power" | "power"
 */
```

---

## 4. CPUFreq Policy and Configuration

### 4.1 Policy Structure

```
  CPUFreq Policy
  ┌──────────────────────────────────────────────────────┐
  │                                                       │
  │  A policy represents a group of CPUs that share the   │
  │  same frequency (on the same clock domain):           │
  │                                                       │
  │  Per-cluster (ARM):                                   │
  │  ┌── Policy 0 ──────┐  ┌── Policy 1 ──────┐         │
  │  │  CPU0, CPU1,      │  │  CPU4, CPU5,      │         │
  │  │  CPU2, CPU3       │  │  CPU6, CPU7       │         │
  │  │  (LITTLE cluster) │  │  (big cluster)    │         │
  │  │  gov: schedutil   │  │  gov: schedutil   │         │
  │  │  min: 600 MHz     │  │  min: 800 MHz     │         │
  │  │  max: 1800 MHz    │  │  max: 2400 MHz    │         │
  │  └──────────────────┘  └──────────────────┘         │
  │                                                       │
  │  Per-core (Intel HWP):                               │
  │  ┌── Policy 0 ─┐ ┌── Policy 1 ─┐ ┌── Policy N ─┐   │
  │  │  CPU0       │ │  CPU1       │ │  CPUN       │   │
  │  │  Individual │ │  Individual │ │  Individual │   │
  │  └────────────┘ └────────────┘ └────────────┘   │
  └──────────────────────────────────────────────────────┘
```

### 4.2 Configuration Commands

```bash
# View current frequency
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq

# View available governors
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors

# Set governor
echo schedutil > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Set frequency limits
echo 1200000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq
echo 2400000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq

# Set frequency directly (userspace governor only)
echo userspace > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
echo 1800000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed

# View frequency table
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies

# View transition statistics
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/trans_table
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state

# Intel HWP EPP
cat /sys/devices/system/cpu/cpufreq/policy0/energy_performance_preference
echo balance_power > /sys/devices/system/cpu/cpufreq/policy0/energy_performance_preference
```

---

## 5. Frequency Boost / Turbo

### 5.1 Turbo Boost Architecture

```
  Intel Turbo Boost / AMD Precision Boost
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Normal operation:                                       │
  │  P0 → guaranteed max frequency (e.g., 3.0 GHz)         │
  │                                                           │
  │  Turbo/Boost:                                            │
  │  Above P0 → opportunistic higher frequency               │
  │  Depends on: thermal headroom, power budget, active cores│
  │                                                           │
  │  Example (4-core CPU, base 3.0 GHz, turbo 4.5 GHz):     │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │ Active Cores  │ Max Turbo Frequency              │    │
  │  ├───────────────┼─────────────────────────────────┤    │
  │  │ 1 core        │ 4.5 GHz                         │    │
  │  │ 2 cores       │ 4.3 GHz                         │    │
  │  │ 3 cores       │ 4.1 GHz                         │    │
  │  │ 4 cores       │ 3.8 GHz                         │    │
  │  └───────────────┴─────────────────────────────────┘    │
  │                                                           │
  │  Fewer active cores → more thermal/power budget          │
  │  per core → higher achievable frequency                  │
  │                                                           │
  │  Control:                                                │
  │  $ cat /sys/devices/system/cpu/cpufreq/boost             │
  │  1  (enabled)                                            │
  │  $ echo 0 > /sys/devices/system/cpu/cpufreq/boost        │
  │  (disable turbo — cap at base frequency)                 │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. CPUFreq Notifiers and Events

```c
/* CPUFreq notification chain */
/* Register to be notified of frequency changes */

static int my_freq_notifier(struct notifier_block *nb,
                             unsigned long event, void *data)
{
    struct cpufreq_freqs *freq = data;

    switch (event) {
    case CPUFREQ_PRECHANGE:
        /* About to change: freq->old → freq->new */
        pr_info("CPU%u: %u → %u KHz\n",
                freq->policy->cpu, freq->old, freq->new);
        /* Adjust timing-sensitive code */
        break;
    case CPUFREQ_POSTCHANGE:
        /* Change complete */
        /* Update loops_per_jiffy, delay calibration, etc. */
        break;
    }
    return NOTIFY_OK;
}

static struct notifier_block my_nb = {
    .notifier_call = my_freq_notifier,
};

cpufreq_register_notifier(&my_nb, CPUFREQ_TRANSITION_NOTIFIER);
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `drivers/cpufreq/cpufreq.c` | CPUFreq core framework |
| `drivers/cpufreq/freq_table.c` | Frequency table helpers |
| `drivers/cpufreq/cpufreq_stats.c` | Frequency statistics |
| `drivers/cpufreq/cpufreq_ondemand.c` | Ondemand governor |
| `drivers/cpufreq/cpufreq_conservative.c` | Conservative governor |
| `kernel/sched/cpufreq_schedutil.c` | Schedutil governor |
| `drivers/cpufreq/cpufreq-dt.c` | Generic DT-based driver |
| `drivers/cpufreq/intel_pstate.c` | Intel P-state/HWP driver |
| `drivers/cpufreq/amd-pstate.c` | AMD P-state driver |
| `drivers/cpufreq/qcom-cpufreq-hw.c` | Qualcomm CPUFreq HW |
| `include/linux/cpufreq.h` | CPUFreq API |
| `drivers/opp/core.c` | OPP framework |

---

## Interview Questions

**Q1: How does the schedutil governor differ from ondemand?**
**A:** Ondemand is sampling-based — it periodically (every ~10ms) reads CPU utilization and adjusts frequency reactively. Schedutil is scheduler-driven — the scheduler invokes it directly when utilization changes (task wakeup, migration, tick), with ~1ms effective response. Schedutil uses PELT (Per-Entity Load Tracking) signals directly, avoiding the sampling delay. This results in faster response to workload changes and tighter integration with the scheduler's view of CPU demand.

**Q2: Explain the Intel HWP (Hardware P-states) architecture.**
**A:** HWP delegates frequency management to the CPU hardware. The OS programs MSR_HWP_REQUEST with min_perf, max_perf, desired_perf, and EPP (Energy Performance Preference, 0-255). The CPU autonomously selects frequency within these bounds, reacting to microarchitectural signals (pipeline utilization, memory stalls) in ~1ms vs ~10ms for software governors. Linux supports HWP through intel_pstate driver in either active mode (driver-controlled) or passive mode (with generic governors).

**Q3: What is a CPUFreq policy and how does it relate to CPU topology?**
**A:** A CPUFreq policy groups CPUs that share the same frequency domain — they run at the same frequency because they share a clock source. On ARM big.LITTLE, each cluster is one policy (e.g., 4 LITTLE cores share policy0, 4 big cores share policy1). On Intel with HWP, each core can be its own policy (per-core DVFS). The policy defines min/max limits, the active governor, and the frequency table for all CPUs in the group.

**Q4: How does turbo boost work and how is it managed in Linux?**
**A:** Turbo/boost allows CPUs to exceed their base frequency when thermal/power budget permits. Fewer active cores mean more headroom per core (e.g., 4.5 GHz single-core vs 3.8 GHz all-core). Linux treats turbo frequencies as valid P-states above P0. The `/sys/devices/system/cpu/cpufreq/boost` sysfs file controls global turbo enable/disable. With HWP, turbo is managed by hardware within OS-specified max_perf bounds.

**Q5: How would you debug a CPUFreq issue where frequency is stuck?**
**A:** Check: (1) `scaling_cur_freq` vs `scaling_min/max_freq` — are limits correct? (2) `scaling_governor` — is it powersave/performance (static)? (3) `/sys/devices/system/cpu/cpufreq/boost` — turbo enabled? (4) Thermal throttling — check `cat /sys/class/thermal/thermal_zone*/temp` and `dmesg | grep thermal`. (5) PM QoS — any latency constraints preventing transitions? (6) `cpufreq/stats/trans_table` — are transitions happening? (7) For HWP: check `energy_performance_preference`. (8) BIOS settings — some lock frequency.

---

## Summary

- CPUFreq separates governors (policy) from drivers (mechanism)
- Schedutil is the modern default — scheduler-driven with ~1ms response
- Ondemand samples periodically (~10ms) and jumps to max above threshold
- Conservative steps frequency up/down gradually
- CPUFreq drivers: intel_pstate (HWP), acpi-cpufreq, cpufreq-dt (ARM), amd-pstate
- Policies group CPUs sharing a clock domain; each has its own governor and limits
- Intel HWP lets hardware manage frequency with OS providing hints (EPP)
- Turbo boost is opportunistic overclocking within thermal/power budget
- Frequency statistics available via sysfs (time_in_state, trans_table)

---

[Previous Chapter: CPU Power Management ←](Chapter_06_CPU_Power_Management.md) | [Next Chapter: CPUIdle Subsystem →](Chapter_08_CPUIdle.md)
