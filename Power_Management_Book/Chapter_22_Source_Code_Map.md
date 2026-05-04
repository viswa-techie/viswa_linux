# Chapter 22: Kernel Source Code Map for Power Management

## Learning Goals
- Navigate the kernel source tree for PM code efficiently
- Understand the relationship between PM source files
- Know which files to read for each PM subsystem
- Identify key data structures and their locations

---

## 1. Top-Level PM Directory Structure

```
  Linux Kernel PM Source Map
  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  kernel/power/                  ← PM Core                    │
  │  ├── main.c                     sysfs (/sys/power/*)         │
  │  ├── suspend.c                  System suspend (S3, s2idle)  │
  │  ├── hibernate.c                Hibernate (S4)               │
  │  ├── snapshot.c                 Memory snapshot for hibernate│
  │  ├── swap.c                     Swap-based hibernate image   │
  │  ├── process.c                  Process freezer              │
  │  ├── wakelock.c                 Android wake locks           │
  │  ├── autosleep.c                Android autosleep            │
  │  ├── energy_model.c             Energy Model framework       │
  │  ├── qos.c                      System-wide PM QoS           │
  │  └── Kconfig                    PM build config              │
  │                                                               │
  │  drivers/base/power/            ← Device PM Core             │
  │  ├── main.c                     Device suspend/resume core   │
  │  ├── runtime.c                  Runtime PM framework         │
  │  ├── domain.c                   Generic Power Domains (genpd)│
  │  ├── wakeup.c                   Wakeup source framework      │
  │  ├── sysfs.c                    Device power sysfs           │
  │  ├── clock_ops.c                PM clock operations          │
  │  └── qos.c                      Device PM QoS                │
  │                                                               │
  │  drivers/cpufreq/               ← CPU Frequency              │
  │  ├── cpufreq.c                  CPUFreq core framework       │
  │  ├── cpufreq_stats.c            Frequency statistics         │
  │  ├── cpufreq-dt.c               Device tree CPUFreq driver   │
  │  ├── intel_pstate.c             Intel P-state driver         │
  │  └── freq_table.c               Frequency table helpers      │
  │                                                               │
  │  drivers/cpuidle/               ← CPU Idle                   │
  │  ├── cpuidle.c                  CPUIdle core                 │
  │  ├── governor.c                 Governor infrastructure      │
  │  ├── governors/                                              │
  │  │   ├── menu.c                 Menu governor                │
  │  │   ├── teo.c                  TEO governor                 │
  │  │   └── ladder.c               Ladder governor              │
  │  ├── dt_idle_states.c           DT idle state parsing        │
  │  └── sysfs.c                    CPUIdle sysfs                │
  │                                                               │
  │  kernel/sched/                  ← Scheduler (EAS)            │
  │  ├── fair.c                     CFS + EAS task placement     │
  │  ├── pelt.c                     Per-Entity Load Tracking     │
  │  ├── core.c                     Scheduler core, uclamp       │
  │  ├── cpufreq_schedutil.c        Schedutil governor           │
  │  └── topology.c                 CPU topology for EAS         │
  └──────────────────────────────────────────────────────────────┘
```

---

## 2. Framework Source Files

```
  Framework-Level Source Files
  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  drivers/thermal/               ← Thermal Framework          │
  │  ├── thermal_core.c             Thermal zone management      │
  │  ├── thermal_of.c               DT thermal zone parsing      │
  │  ├── thermal_sysfs.c            Thermal sysfs interface      │
  │  ├── gov_step_wise.c            Step-wise governor           │
  │  ├── gov_power_allocator.c      IPA governor (PID)           │
  │  ├── gov_bang_bang.c            On/off governor              │
  │  ├── cpufreq_cooling.c         CPU frequency cooler          │
  │  └── devfreq_cooling.c          Device freq cooler           │
  │                                                               │
  │  drivers/regulator/             ← Regulator Framework        │
  │  ├── core.c                     Regulator core               │
  │  ├── helpers.c                  Regmap-based helpers         │
  │  ├── of_regulator.c             DT regulator parsing         │
  │  ├── devres.c                   Devm regulator API           │
  │  └── <pmic_name>.c              PMIC-specific drivers        │
  │                                                               │
  │  drivers/clk/                   ← Clock Framework            │
  │  ├── clk.c                      CCF core                     │
  │  ├── clk-divider.c              Divider clock type           │
  │  ├── clk-mux.c                  Mux clock type               │
  │  ├── clk-gate.c                 Gate clock type              │
  │  ├── clk-fixed-rate.c           Fixed rate clock             │
  │  └── <vendor>/                  SoC clock drivers            │
  │                                                               │
  │  drivers/opp/                   ← OPP Framework              │
  │  ├── core.c                     OPP core                     │
  │  ├── of.c                       DT OPP parsing               │
  │  └── cpu.c                      CPU OPP helpers              │
  └──────────────────────────────────────────────────────────────┘
```

---

## 3. Key Header Files

```
  Essential PM Header Files
  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  include/linux/pm.h              dev_pm_ops, PM macros       │
  │  include/linux/pm_runtime.h      Runtime PM API              │
  │  include/linux/pm_domain.h       genpd API                   │
  │  include/linux/pm_qos.h          PM QoS API                  │
  │  include/linux/pm_wakeup.h       Wakeup source API           │
  │  include/linux/suspend.h         System suspend API          │
  │  include/linux/cpufreq.h         CPUFreq API                 │
  │  include/linux/cpuidle.h         CPUIdle API                 │
  │  include/linux/thermal.h         Thermal framework API       │
  │  include/linux/energy_model.h    Energy Model API            │
  │  include/linux/regulator/consumer.h  Regulator consumer      │
  │  include/linux/regulator/driver.h    Regulator provider      │
  │  include/linux/clk.h             Clock consumer API          │
  │  include/linux/clk-provider.h    Clock provider API          │
  │  include/linux/pm_opp.h          OPP API                     │
  │                                                               │
  │  include/trace/events/power.h    PM tracepoints              │
  │  include/trace/events/rpm.h      Runtime PM tracepoints      │
  │  include/trace/events/thermal.h  Thermal tracepoints         │
  └──────────────────────────────────────────────────────────────┘
```

---

## 4. Key Data Structures Map

```
  Core PM Data Structures → Source Location
  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  struct dev_pm_ops              include/linux/pm.h           │
  │    → System + runtime PM callbacks for drivers               │
  │                                                               │
  │  struct dev_pm_info             include/linux/pm.h           │
  │    → Per-device PM state (in struct device)                  │
  │    → runtime_status, usage_count, disable_depth              │
  │                                                               │
  │  struct generic_pm_domain       include/linux/pm_domain.h    │
  │    → Power domain: power_on/off, device list, hierarchy     │
  │                                                               │
  │  struct cpufreq_policy          include/linux/cpufreq.h      │
  │    → Per-policy: governor, freq limits, driver data          │
  │                                                               │
  │  struct cpufreq_driver          include/linux/cpufreq.h      │
  │    → Driver ops: target, get, init, exit                     │
  │                                                               │
  │  struct cpuidle_device          include/linux/cpuidle.h      │
  │    → Per-CPU idle: states[], last_residency                  │
  │                                                               │
  │  struct cpuidle_governor        include/linux/cpuidle.h      │
  │    → Idle governor: select(), reflect()                      │
  │                                                               │
  │  struct thermal_zone_device     include/linux/thermal.h      │
  │    → Thermal zone: ops, trips, bound coolers                 │
  │                                                               │
  │  struct thermal_cooling_device  include/linux/thermal.h      │
  │    → Cooling device: ops (get/set state)                     │
  │                                                               │
  │  struct regulator_dev           include/linux/regulator/     │
  │    → Regulator device: desc, constraints, consumers          │
  │                                                               │
  │  struct em_perf_domain          include/linux/energy_model.h │
  │    → Energy model: perf states, nr_perf_states               │
  └──────────────────────────────────────────────────────────────┘
```

---

## 5. Call Flow Entry Points

```
  How to Find PM Code Paths
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  "I want to understand..."                               │
  │                                                           │
  │  System suspend:                                         │
  │    Start: state_store() in kernel/power/main.c           │
  │    → enter_state() → suspend_devices_and_enter()         │
  │    → dpm_suspend() in drivers/base/power/main.c          │
  │                                                           │
  │  Runtime PM:                                             │
  │    Start: pm_runtime_get*/put* in                        │
  │    drivers/base/power/runtime.c                          │
  │    → rpm_suspend() / rpm_resume()                        │
  │    → __rpm_callback() → driver's .runtime_suspend/resume │
  │                                                           │
  │  CPU frequency change:                                   │
  │    Start: sugov_update_shared() in                       │
  │    kernel/sched/cpufreq_schedutil.c                      │
  │    → cpufreq_driver_target() in drivers/cpufreq/cpufreq.c│
  │    → driver's .target_index()                            │
  │                                                           │
  │  CPU idle entry:                                         │
  │    Start: do_idle() in kernel/sched/idle.c               │
  │    → cpuidle_idle_call() in drivers/cpuidle/cpuidle.c    │
  │    → governor->select() → driver->enter()               │
  │                                                           │
  │  Thermal throttling:                                     │
  │    Start: thermal_zone_device_update() in                │
  │    drivers/thermal/thermal_core.c                        │
  │    → handle_thermal_trip() → governor->throttle()        │
  │    → cdev->ops->set_cur_state()                          │
  │                                                           │
  │  Power domain on/off:                                    │
  │    Start: genpd_runtime_suspend/resume() in              │
  │    drivers/base/power/domain.c                           │
  │    → genpd->power_off() / genpd->power_on()             │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. SoC-Specific PM Code

```
  SoC PM Code Locations
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ARM:                                                    │
  │  arch/arm64/kernel/                                      │
  │    suspend.S         ARM64 suspend/resume asm            │
  │    cpuidle.c         ARM64 cpuidle integration           │
  │    topology.c        CPU capacity/topology               │
  │                                                           │
  │  drivers/firmware/                                       │
  │    psci/             PSCI interface (ARM PM)             │
  │    arm_scmi/         SCMI protocol (clock, power, etc)   │
  │                                                           │
  │  Qualcomm:                                               │
  │  drivers/soc/qcom/                                       │
  │    rpmh-rsc.c        RPMh resource state coordinator     │
  │    cmd-db.c          Command DB (resource addresses)     │
  │    spm.c             Subsystem Power Manager             │
  │    kryo-l2-accessors.c  L2 cache power                  │
  │                                                           │
  │  Intel:                                                  │
  │  arch/x86/kernel/                                        │
  │    acpi/             ACPI PM integration                 │
  │  drivers/idle/                                           │
  │    intel_idle.c      Intel-specific C-states             │
  │  drivers/cpufreq/                                        │
  │    intel_pstate.c    Intel P-state driver                │
  │                                                           │
  │  Samsung:                                                │
  │  drivers/soc/samsung/                                    │
  │    exynos-pmu.c      Power Management Unit               │
  │  drivers/clk/samsung/                                    │
  │    clk-exynos*.c     Clock drivers                       │
  └──────────────────────────────────────────────────────────┘
```

---

## 7. Documentation Files

```
  Kernel PM Documentation
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Documentation/power/                                    │
  │  ├── admin-guide/                                        │
  │  │   ├── pm/                                             │
  │  │   │   ├── sleep-states.rst    System sleep states     │
  │  │   │   └── cpuidle.rst         CPUIdle admin guide     │
  │  ├── devices.rst                 Driver PM guide         │
  │  ├── runtime_pm.rst              Runtime PM API          │
  │  ├── pci.rst                     PCI PM guide            │
  │  ├── opp.rst                     OPP framework           │
  │  ├── energy-model.rst            Energy Model            │
  │  └── regulator/                  Regulator docs          │
  │                                                           │
  │  Documentation/devicetree/bindings/                      │
  │  ├── power/                      Power DT bindings       │
  │  ├── opp/opp-v2.yaml             OPP table binding       │
  │  ├── thermal/                    Thermal bindings        │
  │  └── regulator/                  Regulator bindings      │
  │                                                           │
  │  Documentation/scheduler/                                │
  │  └── sched-energy.rst            EAS documentation       │
  └──────────────────────────────────────────────────────────┘
```

---

## Kernel Source Reference

| Area | Primary Source | Header |
|------|---------------|--------|
| PM Core | `kernel/power/` | `include/linux/suspend.h` |
| Device PM | `drivers/base/power/` | `include/linux/pm.h` |
| Runtime PM | `drivers/base/power/runtime.c` | `include/linux/pm_runtime.h` |
| CPUFreq | `drivers/cpufreq/` | `include/linux/cpufreq.h` |
| CPUIdle | `drivers/cpuidle/` | `include/linux/cpuidle.h` |
| Thermal | `drivers/thermal/` | `include/linux/thermal.h` |
| Regulator | `drivers/regulator/` | `include/linux/regulator/` |
| Clock | `drivers/clk/` | `include/linux/clk.h` |
| OPP | `drivers/opp/` | `include/linux/pm_opp.h` |
| Energy Model | `kernel/power/energy_model.c` | `include/linux/energy_model.h` |
| EAS | `kernel/sched/fair.c` | `include/linux/sched/topology.h` |

---

## Interview Questions

**Q1: If you need to understand how system suspend works, which files would you read?**
**A:** Start with `kernel/power/main.c` (state_store — entry point when writing to /sys/power/state). Follow to `kernel/power/suspend.c` (enter_state → suspend_devices_and_enter → suspend_enter). For device ordering: `drivers/base/power/main.c` (dpm_suspend/dpm_resume). For process freezing: `kernel/power/process.c`. For platform-specific: `arch/arm64/kernel/suspend.S` (ARM) or ACPI sleep code (x86). Headers `include/linux/suspend.h` and `include/linux/pm.h` define the APIs.

**Q2: Where is runtime PM implemented and how do you trace a runtime PM call?**
**A:** Implementation: `drivers/base/power/runtime.c`. Entry points: `pm_runtime_get_sync() → __pm_runtime_resume() → rpm_resume()`. The state machine (RPM_ACTIVE/SUSPENDED/SUSPENDING/RESUMING) is managed in `struct dev_pm_info` (defined in `include/linux/pm.h`, embedded in `struct device`). Tracing: tracepoints defined in `include/trace/events/rpm.h` — enable `events/rpm/rpm_suspend` and `events/rpm/rpm_resume` in ftrace. Key function: `__rpm_callback()` invokes the driver's `.runtime_suspend/resume`.

---

## Summary

- PM code spans kernel/power/ (core), drivers/base/power/ (device PM), and subsystem-specific directories
- CPUFreq, CPUIdle, thermal, regulator, clock, and OPP each have dedicated directories under drivers/
- EAS lives in kernel/sched/ alongside the CFS scheduler
- Headers in include/linux/ define all PM APIs and data structures
- SoC-specific PM code lives in drivers/soc/<vendor>/ and arch/<arch>/
- Start from entry points (state_store, pm_runtime_get, do_idle) and follow call chains
- Documentation/power/ contains essential guides for each PM subsystem

---

[Previous Chapter: Power Monitoring Tools ←](Chapter_21_PM_Tools.md) | [Next Chapter: Power Management Flow Diagrams →](Chapter_23_Flow_Diagrams.md)
