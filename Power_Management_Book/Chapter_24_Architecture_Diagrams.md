# Chapter 24: Important Architecture Diagrams

## Learning Goals
- Visualize the complete Linux PM subsystem relationships
- See how PM frameworks interconnect at the kernel level
- Understand hardware-software boundary for PM
- Reference consolidated architecture views for interviews

---

## 1. Complete PM Stack Overview

```
  Linux Power Management — Full Stack
  ═══════════════════════════════════════════════════════════════

  ┌─ Userspace ─────────────────────────────────────────────────┐
  │  PowerTOP │ Thermald │ Android PowerManager │ systemd-logind │
  │           │          │        │              │                │
  │  /sys/power/ │ /sys/class/thermal/ │ /sys/devices/.../power/ │
  └──────┬────────────┬──────────────────┬──────────────────────┘
         │            │                  │
  ═══════╪════════════╪══════════════════╪════ Kernel ═══════════
         │            │                  │
  ┌──────▼────────────▼──────────────────▼──────────────────────┐
  │                  PM Core Layer                               │
  │  ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌──────────────┐  │
  │  │System    │ │Device PM │ │Runtime PM │ │PM QoS        │  │
  │  │Suspend   │ │Framework │ │Framework  │ │Framework     │  │
  │  │kernel/   │ │drivers/  │ │drivers/   │ │kernel/power/ │  │
  │  │power/    │ │base/     │ │base/power/│ │drivers/base/ │  │
  │  │suspend.c │ │power/    │ │runtime.c  │ │power/qos.c  │  │
  │  └────┬─────┘ └─────┬────┘ └─────┬─────┘ └──────┬───────┘  │
  └───────┼──────────────┼───────────┼───────────────┼──────────┘
          │              │           │               │
  ┌───────▼──────────────▼───────────▼───────────────▼──────────┐
  │                 Subsystem Frameworks                         │
  │  ┌─────────┐ ┌────────┐ ┌────────┐ ┌──────┐ ┌───────────┐ │
  │  │CPUFreq  │ │CPUIdle │ │Thermal │ │Clock │ │Regulator  │ │
  │  │         │ │        │ │        │ │(CCF) │ │           │ │
  │  │Governors│ │Governors│ │Governors│ │      │ │Constraints│ │
  │  │schedutil│ │menu/teo│ │step_wise│ │      │ │           │ │
  │  │ondemand │ │        │ │IPA     │ │      │ │           │ │
  │  └────┬────┘ └───┬────┘ └───┬────┘ └──┬───┘ └─────┬─────┘ │
  └───────┼──────────┼──────────┼─────────┼───────────┼─────────┘
          │          │          │         │           │
  ┌───────▼──────────▼──────────▼─────────▼───────────▼─────────┐
  │              PM-Capable Device Drivers                       │
  │  ┌────────────────────────────────────────────────────────┐ │
  │  │  dev_pm_ops: .suspend/.resume/.runtime_suspend/resume  │ │
  │  │  Platform │ PCI │ I2C │ SPI │ USB │ Network │ Block   │ │
  │  └──────────────────────────┬─────────────────────────────┘ │
  └─────────────────────────────┼───────────────────────────────┘
                                │
  ══════════════════════════════╪════ Hardware ══════════════════
                                │
  ┌─────────────────────────────▼───────────────────────────────┐
  │  ┌──────┐  ┌──────────┐  ┌────────┐  ┌──────┐ ┌────────┐ │
  │  │ PMIC │  │ Clock    │  │ Power  │  │ CPU  │ │Thermal │ │
  │  │      │  │ Tree     │  │ Domain │  │DVFS  │ │Sensors │ │
  │  │Bucks │  │ PLL      │  │Switches│  │P/C   │ │        │ │
  │  │LDOs  │  │ Dividers │  │Isolation│ │States│ │        │ │
  │  └──────┘  └──────────┘  └────────┘  └──────┘ └────────┘ │
  └─────────────────────────────────────────────────────────────┘
```

---

## 2. CPU Power Management Integration

```
  CPU PM: Frequency × Idle × Thermal × EAS
  ═══════════════════════════════════════════════════════════════

  ┌─────────────────────────────────────────────────────────────┐
  │                    CFS Scheduler                             │
  │                                                             │
  │  ┌── PELT ──────────┐  ┌── EAS ─────────────────────┐     │
  │  │ Track per-task    │  │ find_energy_efficient_cpu() │     │
  │  │ utilization       │  │ Compare energy for each     │     │
  │  │ (util_avg)        │──│ candidate CPU using EM      │     │
  │  └──────┬────────────┘  └──────────┬────────────────┘     │
  │         │                          │                        │
  │         │ util changes             │ task placed            │
  └─────────┼──────────────────────────┼────────────────────────┘
            │                          │
  ┌─────────▼──────────────────────────▼────────────────────────┐
  │  ┌─ Schedutil ─────────────┐  ┌─ Energy Model ──────────┐ │
  │  │ freq = 1.25×util×max/cap│  │ Performance domains      │ │
  │  │ Update per scheduler    │  │ OPP → power mapping      │ │
  │  │ event                   │  │ Capacity per CPU          │ │
  │  └─────────┬───────────────┘  └──────────────────────────┘ │
  │            │ freq request                                   │
  │  ┌─────────▼───────────────┐  ┌─ Thermal Cooling ───────┐ │
  │  │ CPUFreq Driver          │  │ freq_qos limits max freq │ │
  │  │ Sets clock + voltage    │◄─│ When thermal trip crossed│ │
  │  │ (DVFS transition)       │  │ Governor reduces cap     │ │
  │  └─────────────────────────┘  └──────────────────────────┘ │
  │                                                             │
  │  ┌─ CPUIdle ──────────────────────────────────────────────┐│
  │  │ When CPU has no tasks:                                  ││
  │  │ Governor selects C-state based on predicted idle       ││
  │  │ PM QoS constrains deepest allowed state                ││
  │  │ Driver enters WFI/MWAIT/PSCI                           ││
  │  └────────────────────────────────────────────────────────┘│
  └─────────────────────────────────────────────────────────────┘
```

---

## 3. Device Power Management Layers

```
  Device PM: Three-Layer Architecture
  ═══════════════════════════════════════════════════════════════

  ┌── Layer 1: Driver ──────────────────────────────────────────┐
  │                                                              │
  │  static const struct dev_pm_ops my_pm = {                   │
  │      .suspend         = my_suspend,     ← system sleep      │
  │      .resume          = my_resume,                           │
  │      .runtime_suspend = my_rt_suspend,  ← device idle       │
  │      .runtime_resume  = my_rt_resume,                        │
  │  };                                                          │
  │                                                              │
  │  - Driver knows device hardware specifics                    │
  │  - Saves/restores device registers                           │
  │  - Manages device-specific resources                         │
  └──────────────────────────┬───────────────────────────────────┘
                             │
  ┌──────────────────────────▼───────────────────────────────────┐
  │  Layer 2: Bus/Type/Class                                     │
  │                                                              │
  │  PCI: pci_pm_suspend()                                      │
  │    - Saves PCI config space automatically                    │
  │    - Sets PCI power state (D0→D3)                           │
  │    - Manages ASPM link power                                 │
  │                                                              │
  │  Platform: platform_pm_suspend()                             │
  │    - Calls driver ops directly                               │
  │    - No bus-level state management                           │
  │                                                              │
  │  I2C: i2c_device_pm_suspend()                               │
  │    - Calls driver ops                                        │
  │    - Bus adapter may have own PM                             │
  └──────────────────────────┬───────────────────────────────────┘
                             │
  ┌──────────────────────────▼───────────────────────────────────┐
  │  Layer 3: PM Core                                            │
  │                                                              │
  │  dpm_suspend() → __device_suspend() → callback hierarchy:  │
  │    1. driver->pm->suspend()   (if exists)                   │
  │    2. type->pm->suspend()     (if no driver PM)             │
  │    3. class->pm->suspend()    (if no type PM)               │
  │    4. bus->pm->suspend()      (if no class PM)              │
  │                                                              │
  │  Ordering: children before parents (DPM list)               │
  │  Manages: async suspend, direct_complete optimization        │
  └──────────────────────────────────────────────────────────────┘
```

---

## 4. Power Domain Hierarchy

```
  Generic Power Domain (genpd) Hierarchy
  ═══════════════════════════════════════════════════════════════

  ┌─ SoC Power Domain Map ──────────────────────────────────────┐
  │                                                              │
  │                    ┌─────────────┐                           │
  │                    │  PD_TOP     │ (always-on or last off)   │
  │                    │  VDD_CORE   │                           │
  │                    └──┬──┬──┬────┘                           │
  │                       │  │  │                                │
  │          ┌────────────┘  │  └────────────┐                  │
  │          │               │               │                   │
  │    ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐            │
  │    │ PD_CPU    │  │ PD_GPU    │  │ PD_PERIPH │            │
  │    │           │  │           │  │           │            │
  │    │ cpu0-3    │  │ gpu       │  │ usb, uart │            │
  │    │ L2 cache  │  │ video dec │  │ spi, i2c  │            │
  │    └───────────┘  └───────────┘  │ display   │            │
  │                                   └───────────┘            │
  │                                                              │
  │  Rules:                                                     │
  │  - Child domain can be off only if all its devices suspended│
  │  - Parent must be on if ANY child is on                     │
  │  - Power-on order: parent first, then child                 │
  │  - Power-off order: child first, then parent (if no others) │
  │                                                              │
  │  Runtime PM integration:                                    │
  │  pm_runtime_put(gpu_dev) → last ref → domain suspend       │
  │  → PD_GPU.power_off() → if PD_PERIPH also off              │
  │  → PD_TOP stays on (PD_CPU still active)                    │
  └──────────────────────────────────────────────────────────────┘
```

---

## 5. DVFS Voltage-Frequency Relationship

```
  DVFS Operating Points Visualization
  ═══════════════════════════════════════════════════════════════

  Voltage (V)
   │
  1.35 ────────────────────────────────────── × OPP5 (Turbo)
   │                                        ╱
  1.20 ─────────────────────────────── × OPP4
   │                                 ╱
  1.10 ──────────────────────── × OPP3
   │                          ╱
  1.00 ─────────────── × OPP2
   │                 ╱
  0.80 ──── × OPP1  ╱
   │       ╱
  0.60 × OPP0
   │
   └──┬────┬────┬────┬────┬────┬──── Frequency (GHz)
      0.5  0.8  1.2  1.5  2.0  2.5

  Power at each OPP:  P = C × V² × f
  ┌──────┬────────┬──────────┬──────────────────────┐
  │ OPP  │ Freq   │ Voltage  │ Relative Power       │
  ├──────┼────────┼──────────┼──────────────────────┤
  │ OPP0 │ 0.5GHz │ 0.60V    │ 0.18 (baseline)      │
  │ OPP1 │ 0.8GHz │ 0.80V    │ 0.51 (2.8×)          │
  │ OPP2 │ 1.2GHz │ 1.00V    │ 1.20 (6.7×)          │
  │ OPP3 │ 1.5GHz │ 1.10V    │ 1.82 (10.1×)         │
  │ OPP4 │ 2.0GHz │ 1.20V    │ 2.88 (16.0×)         │
  │ OPP5 │ 2.5GHz │ 1.35V    │ 4.56 (25.3×)         │
  └──────┴────────┴──────────┴──────────────────────┘

  Key insight: 5× frequency increase → 25× power increase
  (due to V² × f relationship)
```

---

## 6. Thermal Management Architecture

```
  Complete Thermal Control Loop
  ═══════════════════════════════════════════════════════════════

  ┌─ Hardware ──────────────┐    ┌─ Software ───────────────────┐
  │                         │    │                               │
  │  ┌──────────────────┐   │    │  ┌─── Thermal Framework ──┐  │
  │  │ Temperature      │   │    │  │                         │  │
  │  │ Sensors          │───┼───►│  │  thermal_zone_device    │  │
  │  │ (on-die, board)  │   │    │  │  - get_temp()           │  │
  │  └──────────────────┘   │    │  │  - trip points          │  │
  │                         │    │  │  - polling interval     │  │
  │                         │    │  └────────┬────────────────┘  │
  │                         │    │           │                    │
  │                         │    │  ┌────────▼────────────────┐  │
  │                         │    │  │  Governor               │  │
  │                         │    │  │  step_wise / IPA        │  │
  │                         │    │  │  Decides throttle level │  │
  │                         │    │  └────────┬────────────────┘  │
  │                         │    │           │                    │
  │                         │    │  ┌────────▼────────────────┐  │
  │  ┌──────────────────┐   │    │  │  Cooling Devices       │  │
  │  │ CPU (freq limit) │◄──┼────│  │  - cpufreq_cooling     │  │
  │  │ GPU (freq limit) │◄──┼────│  │  - devfreq_cooling     │  │
  │  │ Fan (PWM control)│◄──┼────│  │  - fan_cooling         │  │
  │  └──────────────────┘   │    │  └────────────────────────┘  │
  │         │               │    │                               │
  │         │ Less heat     │    │                               │
  │         ▼               │    │                               │
  │  ┌──────────────────┐   │    │                               │
  │  │ Temperature      │   │    │  Closed-loop control:        │
  │  │ decreases        │───┼───►│  Measure → Decide → Actuate  │
  │  └──────────────────┘   │    │  → Repeat                     │
  └─────────────────────────┘    └───────────────────────────────┘
```

---

## 7. PM Framework Interactions

```
  How PM Frameworks Interact
  ═══════════════════════════════════════════════════════════════

  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  Runtime PM ←──────→ Power Domains (genpd)                 │
  │  Device idle/active    Domain on/off based on              │
  │  triggers domain       aggregate device status             │
  │  transitions                                                │
  │                                                             │
  │  CPUFreq ←────────→ Regulator + Clock                     │
  │  Frequency change      Clock: set rate                     │
  │  requires matching     Regulator: set voltage              │
  │  voltage               OPP: maps freq↔volt                 │
  │                                                             │
  │  Thermal ←────────→ CPUFreq + CPUIdle                     │
  │  Temperature limit     CPUFreq: cap max frequency          │
  │  affects CPU           CPUIdle: thermal affects idle        │
  │  performance           prediction                           │
  │                                                             │
  │  EAS ←────────────→ Energy Model + Schedutil               │
  │  Task placement        EM: power at each OPP               │
  │  considers energy      Schedutil: sets freq from util      │
  │  cost                                                       │
  │                                                             │
  │  System Suspend ←──→ Runtime PM                            │
  │  System sleep uses     pm_runtime_force_suspend()          │
  │  runtime PM state      Avoids double suspend               │
  │  for optimization      direct_complete shortcut             │
  │                                                             │
  │  PM QoS ←─────────→ CPUIdle + genpd                       │
  │  Latency constraints   Limits deepest C-state              │
  │  from drivers/apps     Limits domain idle depth            │
  └─────────────────────────────────────────────────────────────┘
```

---

## Kernel Source Reference

| Diagram | Key Source Files |
|---------|-----------------|
| Full PM Stack | `kernel/power/`, `drivers/base/power/` |
| CPU PM Integration | `kernel/sched/fair.c`, `drivers/cpufreq/`, `drivers/cpuidle/` |
| Device PM Layers | `drivers/base/power/main.c`, bus-specific PM |
| Power Domain | `drivers/base/power/domain.c` |
| DVFS | `drivers/cpufreq/cpufreq-dt.c`, `drivers/opp/` |
| Thermal Loop | `drivers/thermal/thermal_core.c` |

---

## Interview Questions

**Q1: Draw and explain the Linux PM stack from userspace to hardware.**
**A:** Top: Userspace tools (PowerTOP, thermald, Android PM) interact via sysfs. Middle: PM Core layer (system suspend, device PM, runtime PM, PM QoS). Below: subsystem frameworks (CPUFreq, CPUIdle, thermal, clock, regulator, OPP) providing policy and governance. Below: PM-capable device drivers implementing dev_pm_ops callbacks. Bottom: Hardware (PMIC, clock tree, power switches, CPU P/C-states, thermal sensors). The stack separates policy (governors decide WHAT to do) from mechanism (drivers/HW do it).

**Q2: How do the CPUFreq, thermal, and EAS subsystems interact?**
**A:** EAS places tasks on energy-efficient CPUs using the Energy Model. Schedutil sets CPU frequency proportional to utilization. When temperature crosses a thermal trip, the thermal governor tells cpufreq_cooling to cap the maximum frequency via freq_qos. CPUFreq enforces this cap — even if schedutil requests higher frequency, it's clamped. EAS sees reduced capacity on throttled CPUs and may migrate tasks to cooler CPUs. This creates a feedback loop: thermal limits → frequency cap → EAS task migration → reduced heat.

---

## Summary

- The PM stack has clear layers: userspace → PM core → subsystem frameworks → drivers → hardware
- CPU PM integrates EAS (placement), schedutil (frequency), cpuidle (idle), and thermal (limits)
- Device PM uses three layers: driver callbacks → bus/type/class → PM core
- Power domains form hierarchies — child off before parent, parent on before child
- DVFS power grows as V^2 × f — small frequency increases cause large power jumps
- Thermal forms a closed control loop: measure → decide (governor) → actuate (cooling device)
- PM frameworks are deeply interconnected: runtime PM ↔ genpd, cpufreq ↔ regulator/clock, thermal ↔ cpufreq

---

[Previous Chapter: Power Management Flow Diagrams ←](Chapter_23_Flow_Diagrams.md) | [Next Chapter: Glossary of PM Terms →](Chapter_25_Glossary.md)
