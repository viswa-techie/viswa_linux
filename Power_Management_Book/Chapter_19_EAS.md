# Chapter 19: Energy-Aware Scheduling (EAS)

## Learning Goals
- Understand how the Linux scheduler optimizes for energy efficiency
- Learn the Energy Model framework and capacity-aware scheduling
- Know how EAS selects CPUs for task placement
- Understand schedutil governor integration with EAS
- Learn EAS tuning and debugging

---

## 1. Why Energy-Aware Scheduling

```
  Traditional Scheduler vs Energy-Aware Scheduler
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Traditional (performance-first):                        │
  │  - Spread tasks across all CPUs for throughput            │
  │  - Wake idle CPUs to run tasks → more power              │
  │  - No concept of CPU energy cost                         │
  │                                                           │
  │  Energy-Aware (EAS):                                     │
  │  - Pack tasks on fewer CPUs when possible                │
  │  - Prefer LITTLE cores for light tasks                   │
  │  - Consider energy cost when placing tasks               │
  │  - Only spread when needed for performance               │
  │                                                           │
  │  Example: Light browser task (10% CPU)                   │
  │    Traditional: May run on big core at high freq          │
  │    EAS: Runs on LITTLE core at low freq → 5× less energy │
  │                                                           │
  │  big core at 2GHz: P = 0.5 × 0.9² × 2.0 = 810mW        │
  │  LITTLE at 1GHz:   P = 0.3 × 0.6² × 1.0 = 108mW        │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Heterogeneous CPU Architecture

```
  ARM DynamIQ big.LITTLE
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌────── LITTLE Cluster ──────┐  ┌── big Cluster ─────┐ │
  │  │                             │  │                     │ │
  │  │  CPU0    CPU1    CPU2   CPU3│  │  CPU4   CPU5   CPU6│ │
  │  │  A55     A55     A55    A55 │  │  A78    A78    A78  │ │
  │  │                             │  │                     │ │
  │  │  Capacity: 446              │  │  Capacity: 1024    │ │
  │  │  Max freq: 1.8 GHz         │  │  Max freq: 2.8 GHz │ │
  │  │  Energy/MHz: LOW            │  │  Energy/MHz: HIGH  │ │
  │  │                             │  │                     │ │
  │  │  Best for:                  │  │  Best for:         │ │
  │  │  - Background tasks         │  │  - UI rendering    │ │
  │  │  - System services          │  │  - Gaming          │ │
  │  │  - Light workloads          │  │  - Compilation     │ │
  │  └─────────────────────────────┘  └────────────────────┘ │
  │                                                           │
  │  Capacity = (max_freq × IPC) normalized to 1024          │
  │  Scheduler uses capacity to match tasks to CPUs          │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Energy Model Framework

```c
/* Energy Model: maps CPU performance states to power */
/* Registered by cpufreq driver or DT during boot */

/*
 * struct em_perf_state {
 *     unsigned long frequency;  /* KHz */
 *     unsigned long power;      /* milliwatts at max utilization */
 *     unsigned long cost;       /* energy cost (power / capacity) */
 * };
 *
 * Energy Model table for LITTLE cluster (example):
 *
 * ┌──────────┬──────────┬──────────┬──────────┐
 * │ Freq(MHz)│ Cap      │ Power(mW)│ Cost     │
 * ├──────────┼──────────┼──────────┼──────────┤
 * │ 500      │ 124      │ 50       │ 403      │
 * │ 800      │ 198      │ 100      │ 505      │
 * │ 1200     │ 297      │ 200      │ 673      │
 * │ 1500     │ 372      │ 350      │ 941      │
 * │ 1800     │ 446      │ 550      │ 1233     │
 * └──────────┴──────────┴──────────┴──────────┘
 *
 * Energy Model table for big cluster:
 *
 * ┌──────────┬──────────┬──────────┬──────────┐
 * │ Freq(MHz)│ Cap      │ Power(mW)│ Cost     │
 * ├──────────┼──────────┼──────────┼──────────┤
 * │ 500      │ 183      │ 100      │ 546      │
 * │ 1000     │ 366      │ 350      │ 956      │
 * │ 1500     │ 549      │ 700      │ 1275     │
 * │ 2000     │ 732      │ 1200     │ 1639     │
 * │ 2500     │ 915      │ 2000     │ 2186     │
 * │ 2800     │ 1024     │ 3000     │ 2930     │
 * └──────────┴──────────┴──────────┴──────────┘
 *
 * Cost = power × max_capacity / capacity
 * Lower cost = more energy efficient
 * LITTLE at 500MHz (cost 403) is most efficient
 */
```

```c
/* Energy model registration (in cpufreq driver) */
#include <linux/energy_model.h>

static int my_cpufreq_probe(struct cpufreq_policy *policy)
{
    /* ... setup frequency table ... */

    /* Register energy model from OPP table */
    em_dev_register_perf_domain(cpu_dev,
                                 nr_opps,
                                 &em_cb,  /* callbacks */
                                 policy->cpus, /* CPU mask */
                                 true);    /* milliwatts */
    return 0;
}
```

---

## 4. EAS Task Placement Algorithm

```
  EAS find_energy_efficient_cpu() Flow
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Task wakes up → scheduler calls find_energy_efficient_cpu│
  │       │                                                   │
  │       ▼                                                   │
  │  [Task fits on LITTLE?]  (task_util ≤ LITTLE capacity)   │
  │       │                                                   │
  │       ├─ YES → Compute energy for each candidate CPU:    │
  │       │        E = Σ (for each perf domain)              │
  │       │            em_cpu_energy(pd, util, max_cap)      │
  │       │                                                   │
  │       │   Candidates:                                    │
  │       │   ┌────────────────────────────────────────┐     │
  │       │   │ Option A: Place on LITTLE CPU0         │     │
  │       │   │   LITTLE energy: 120mW  big: 0mW       │     │
  │       │   │   Total: 120mW                         │     │
  │       │   │                                        │     │
  │       │   │ Option B: Place on LITTLE CPU1         │     │
  │       │   │   LITTLE energy: 130mW  big: 0mW       │     │
  │       │   │   Total: 130mW (CPU1 already has load) │     │
  │       │   │                                        │     │
  │       │   │ Option C: Place on big CPU4            │     │
  │       │   │   LITTLE energy: 80mW  big: 200mW      │     │
  │       │   │   Total: 280mW                         │     │
  │       │   └────────────────────────────────────────┘     │
  │       │                                                   │
  │       │   Winner: Option A (120mW) → place on CPU0       │
  │       │                                                   │
  │       ├─ NO (task too big) → Use big core                │
  │       │   (still pick lowest-energy big CPU)             │
  │       │                                                   │
  │       └─ Energy diff < 6% → Use performance heuristic   │
  │          (prefer previous CPU for cache warmth)          │
  └──────────────────────────────────────────────────────────┘
```

```c
/* Simplified EAS energy computation */
/* From kernel/sched/fair.c: find_energy_efficient_cpu() */

/*
 * For each performance domain (LITTLE, big):
 *   1. Sum utilization of all CPUs in domain
 *   2. Add new task's utilization to candidate CPU
 *   3. Find minimum OPP that can handle total utilization
 *   4. energy = OPP_power × busy_ratio
 *
 * Pick the CPU that results in lowest total system energy
 */
```

---

## 5. Schedutil Governor Integration

```
  EAS + Schedutil: Complete Picture
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌── CFS Scheduler ──────────────────────────────┐      │
  │  │  1. Track per-task utilization (PELT)          │      │
  │  │  2. EAS: place task on energy-efficient CPU    │      │
  │  │  3. Update CPU utilization                     │      │
  │  └──────────────────┬────────────────────────────┘      │
  │                     │ util_avg changes                   │
  │  ┌──────────────────▼────────────────────────────┐      │
  │  │  Schedutil Governor                            │      │
  │  │  freq = 1.25 × util_avg × max_freq / capacity │      │
  │  │                                                │      │
  │  │  Example: LITTLE CPU0                         │      │
  │  │  util_avg = 200 (of 446 capacity)              │      │
  │  │  freq = 1.25 × 200 × 1800 / 446 = 1009 MHz   │      │
  │  │  → Select 1200 MHz OPP (next available)       │      │
  │  └──────────────────┬────────────────────────────┘      │
  │                     │ frequency request                  │
  │  ┌──────────────────▼────────────────────────────┐      │
  │  │  CPUFreq Driver                                │      │
  │  │  - Set clock to 1200 MHz                      │      │
  │  │  - Set voltage to match OPP (via regulator)   │      │
  │  └──────────────────────────────────────────────┘      │
  │                                                           │
  │  Result: Task runs on efficient core at just-right freq  │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. PELT (Per-Entity Load Tracking)

```
  PELT Utilization Tracking
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  PELT tracks per-task CPU utilization as exponentially   │
  │  weighted moving average (EWMA):                        │
  │                                                           │
  │  util_avg = Σ (contribution_i × decay_factor^i)         │
  │  Half-life: ~32ms (configurable)                        │
  │                                                           │
  │  Example task behavior:                                  │
  │                                                           │
  │  util_avg                                                │
  │   │         ┌───────────┐                                │
  │  800 ──────┤ Heavy work │                                │
  │   │         └───────┬───┘                                │
  │  400 ────────────────┤ → Decays after task sleeps        │
  │   │                  │      ╲                             │
  │  200 ────────────────┼───────╲──── Stabilizes            │
  │   │                  │        ╲                           │
  │   └──────────────────┴────────── Time →                  │
  │                                                           │
  │  EAS uses util_avg to estimate task's CPU demand:        │
  │  - High util_avg → needs big core / high freq            │
  │  - Low util_avg → can use LITTLE core / low freq         │
  │                                                           │
  │  Scheduler entity:                                       │
  │  se->avg.util_avg   → task CPU utilization               │
  │  se->avg.load_avg   → task weighted load (priority)      │
  │  cfs_rq->avg.util_avg → CPU run queue total utilization │
  └──────────────────────────────────────────────────────────┘
```

---

## 7. EAS Prerequisites and Configuration

```bash
# EAS requirements:
# 1. Heterogeneous CPU topology (big.LITTLE or similar)
# 2. Energy Model registered for each performance domain
# 3. schedutil governor active
# 4. CONFIG_ENERGY_MODEL=y
# 5. Scale-invariant CPU utilization

# Check if EAS is active
cat /proc/sys/kernel/sched_energy_aware
# 1 = EAS enabled, 0 = disabled

# Check energy model
cat /sys/kernel/debug/energy_model/pd*/
# Shows performance domains and energy table

# Check CPU capacities
cat /sys/devices/system/cpu/cpu*/cpu_capacity
# cpu0: 446   (LITTLE)
# cpu4: 1024  (big)

# Verify schedutil is active
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
# schedutil

# EAS sysctl tunables
cat /proc/sys/kernel/sched_energy_aware      # 1=on
```

---

## 8. Uclamp (Utilization Clamping)

```
  Uclamp: Per-Task Performance Hints
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  uclamp_min: Minimum utilization (performance floor)     │
  │  uclamp_max: Maximum utilization (power ceiling)         │
  │                                                           │
  │  Example use cases:                                      │
  │                                                           │
  │  UI thread (latency-sensitive):                          │
  │    uclamp_min = 600  → Always gets big core              │
  │    uclamp_max = 1024 → No cap                            │
  │                                                           │
  │  Background download:                                    │
  │    uclamp_min = 0    → Can use any core                  │
  │    uclamp_max = 300  → Capped to LITTLE cores            │
  │                                                           │
  │  Setting uclamp from userspace:                          │
  │  struct sched_attr attr = {                              │
  │      .sched_flags = SCHED_FLAG_UTIL_CLAMP,               │
  │      .sched_util_min = 600,                              │
  │      .sched_util_max = 1024,                             │
  │  };                                                       │
  │  sched_setattr(pid, &attr, 0);                           │
  │                                                           │
  │  Or via cgroup:                                          │
  │  echo 600 > /sys/fs/cgroup/cpu/.../cpu.uclamp.min       │
  │  echo 300 > /sys/fs/cgroup/cpu/.../cpu.uclamp.max       │
  │                                                           │
  │  EAS uses clamped utilization for placement decisions    │
  └──────────────────────────────────────────────────────────┘
```

---

## 9. EAS vs Symmetric Systems

```
  When EAS Helps vs When It Doesn't
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  EAS BENEFICIAL (heterogeneous):                         │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ Scenario               │ Savings vs traditional│      │
  │  ├────────────────────────┼───────────────────────┤      │
  │  │ Light mixed workload   │ 20-40% energy saving  │      │
  │  │ Mobile UI interaction  │ 15-30% energy saving  │      │
  │  │ Background services    │ 30-50% energy saving  │      │
  │  └────────────────────────┴───────────────────────┘      │
  │                                                           │
  │  EAS NOT APPLICABLE:                                     │
  │  - Symmetric SMP (all cores identical) → no data         │
  │  - Fully loaded system → all cores busy anyway           │
  │  - Over 16 CPUs → energy computation too expensive       │
  │    (falls back to traditional load balancing)            │
  │                                                           │
  │  EAS auto-disables when:                                 │
  │  - No energy model registered                            │
  │  - schedutil governor not active                         │
  │  - System is overutilized (>80% all CPUs busy)           │
  │    → falls back to spreading tasks for throughput        │
  └──────────────────────────────────────────────────────────┘
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `kernel/sched/fair.c` | CFS scheduler with EAS (find_energy_efficient_cpu) |
| `kernel/sched/pelt.c` | Per-Entity Load Tracking |
| `kernel/sched/core.c` | Scheduler core, uclamp |
| `kernel/sched/cpufreq_schedutil.c` | Schedutil governor |
| `kernel/power/energy_model.c` | Energy Model framework |
| `include/linux/energy_model.h` | Energy Model API |
| `include/linux/sched/topology.h` | Scheduler topology |
| `Documentation/scheduler/sched-energy.rst` | EAS documentation |

---

## Interview Questions

**Q1: How does EAS decide which CPU to place a task on?**
**A:** EAS (find_energy_efficient_cpu) evaluates candidate CPUs by computing total system energy for each placement option. For each candidate, it sums energy across all performance domains: energy = OPP_power × utilization_ratio. It uses the Energy Model to map utilization to the required OPP and its power cost. The CPU resulting in lowest total system energy wins. If the energy difference between candidates is less than 6%, EAS prefers the previous CPU to benefit from cache warmth.

**Q2: What is the relationship between EAS and schedutil?**
**A:** They work as complementary halves: EAS handles task-to-CPU placement (WHERE to run), schedutil handles frequency selection (HOW FAST to run). When EAS places a light task on a LITTLE core, schedutil sets frequency proportional to the CPU's utilization (freq = 1.25 × util × max_freq / capacity). Together they minimize energy: right core + right frequency. EAS requires schedutil — other governors (performance, ondemand) break the energy model because they don't respond proportionally to utilization.

**Q3: What is uclamp and how does it affect EAS?**
**A:** Uclamp (utilization clamping) provides per-task performance hints. uclamp_min sets a minimum utilization floor (boosting — ensures task gets a powerful core), uclamp_max sets a maximum cap (capping — prevents task from using excessive resources). EAS uses the clamped utilization (max(uclamp_min, min(actual_util, uclamp_max))) for energy computation and task placement. This allows Android to boost UI threads (uclamp_min=600) while capping background tasks (uclamp_max=200).

---

## Summary

- EAS optimizes task placement for energy efficiency on heterogeneous CPUs (big.LITTLE)
- Energy Model provides power-at-each-OPP data for each performance domain
- EAS evaluates total system energy for each candidate CPU placement
- PELT tracks per-task utilization with exponential decay (~32ms half-life)
- Schedutil translates CPU utilization to frequency proportionally
- Uclamp provides per-task min/max utilization hints for performance/power tuning
- EAS auto-disables when system is overutilized (>80%) or on symmetric systems
- Prerequisites: heterogeneous topology + energy model + schedutil governor

---

[Previous Chapter: Embedded System Power Management ←](Chapter_18_Embedded_PM.md) | [Next Chapter: Power Management Debugging →](Chapter_20_PM_Debugging.md)
