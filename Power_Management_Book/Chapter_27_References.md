# Chapter 27: Documentation and References

## Learning Goals
- Know where to find authoritative PM documentation
- Access kernel documentation, LWN articles, and conference talks
- Find specifications for ACPI, PSCI, and other standards
- Access vendor-specific PM documentation

---

## 1. Kernel Documentation

```
  Essential Kernel Documentation for PM
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  In-tree documentation (Documentation/):                 │
  │                                                           │
  │  PM Core:                                                │
  │  Documentation/power/runtime_pm.rst      Runtime PM API  │
  │  Documentation/power/devices.rst         Driver PM guide │
  │  Documentation/power/pci.rst             PCI PM          │
  │  Documentation/power/opp.rst             OPP framework   │
  │  Documentation/power/energy-model.rst    Energy Model    │
  │  Documentation/admin-guide/pm/          PM admin guide   │
  │  Documentation/admin-guide/pm/sleep-states.rst           │
  │  Documentation/admin-guide/pm/cpuidle.rst                │
  │                                                           │
  │  Scheduler:                                              │
  │  Documentation/scheduler/sched-energy.rst    EAS guide   │
  │                                                           │
  │  Device Tree:                                            │
  │  Documentation/devicetree/bindings/power/                │
  │  Documentation/devicetree/bindings/opp/opp-v2.yaml       │
  │  Documentation/devicetree/bindings/thermal/              │
  │  Documentation/devicetree/bindings/regulator/            │
  │                                                           │
  │  Online: https://docs.kernel.org/power/                  │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Standards and Specifications

```
  Industry Standards
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ACPI Specification:                                     │
  │  - UEFI Forum: uefi.org/specifications                   │
  │  - Defines G/S/C/P/D/T states                           │
  │  - Chapters: CPU PM (8), Device PM (7), Thermal (11)    │
  │                                                           │
  │  ARM PSCI Specification:                                 │
  │  - ARM DEN0022: Power State Coordination Interface       │
  │  - developer.arm.com/documentation                       │
  │  - CPU_ON, CPU_OFF, CPU_SUSPEND, SYSTEM_SUSPEND          │
  │                                                           │
  │  ARM SCMI Specification:                                 │
  │  - ARM DEN0056: System Control and Management Interface  │
  │  - Protocol for clock, power, sensor, performance mgmt   │
  │                                                           │
  │  PCI PM Specification:                                   │
  │  - PCI Power Management Interface Spec (pcisig.com)      │
  │  - D0-D3 states, PME (Power Management Event)            │
  │                                                           │
  │  USB PM:                                                 │
  │  - USB 2.0/3.0 specifications                            │
  │  - Selective suspend, link PM (U1/U2 states)             │
  │                                                           │
  │  JEDEC DDR Self-Refresh:                                 │
  │  - JESD79/JESD209 (DDR4/LPDDR5 specifications)          │
  │  - Self-refresh, partial array self-refresh              │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Key LWN.net Articles

```
  Essential LWN Articles on PM
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Runtime PM:                                             │
  │  - "Runtime power management" (2009)                     │
  │  - "Runtime PM improvements" (2012)                      │
  │                                                           │
  │  CPUFreq:                                                │
  │  - "Schedutil: a new CPUFreq governor" (2016)            │
  │  - "The ongoing power management battle" (2017)          │
  │                                                           │
  │  EAS:                                                    │
  │  - "Energy-aware scheduling" (2018)                      │
  │  - "Completing energy-aware scheduling" (2019)           │
  │  - "Uclamp: Per-task utilization clamping" (2019)        │
  │                                                           │
  │  Thermal:                                                │
  │  - "The thermal subsystem" (2014)                        │
  │  - "Intelligent Power Allocation" (2015)                 │
  │                                                           │
  │  General PM:                                             │
  │  - "Suspend-to-idle" (2016)                              │
  │  - "The power management landscape" (2018)               │
  │  - "Power management for systems on modules" (2020)      │
  │                                                           │
  │  Search: lwn.net/Kernel/Index/#Power_management          │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Conference Talks and Presentations

```
  Recommended Conference Talks
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Linux Plumbers Conference (LPC):                        │
  │  - Power Management and Energy-Aware Scheduling micconf  │
  │  - Annual updates on PM subsystem development            │
  │                                                           │
  │  Embedded Linux Conference (ELC):                        │
  │  - "Understanding the Linux Power Management Framework"  │
  │  - "Energy-Aware Scheduling in Practice"                 │
  │  - "Thermal Management in Embedded Linux"                │
  │                                                           │
  │  Kernel Recipes:                                         │
  │  - "The CPUFreq Subsystem" by Rafael Wysocki             │
  │  - "Linux Runtime PM" by Kevin Hilman                    │
  │                                                           │
  │  Linux Foundation Events:                                │
  │  - Search: events.linuxfoundation.org                    │
  │  - YouTube: The Linux Foundation channel                 │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Kernel Source Reading Guide

```
  Recommended Reading Order for PM Source
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Start here (foundations):                               │
  │  1. include/linux/pm.h            — Data structures      │
  │  2. include/linux/pm_runtime.h    — Runtime PM API       │
  │  3. drivers/base/power/runtime.c  — Runtime PM impl      │
  │                                                           │
  │  System suspend:                                         │
  │  4. kernel/power/main.c           — sysfs entry point    │
  │  5. kernel/power/suspend.c        — Suspend flow         │
  │  6. drivers/base/power/main.c     — Device ordering      │
  │                                                           │
  │  CPU PM:                                                 │
  │  7. drivers/cpufreq/cpufreq.c     — CPUFreq core         │
  │  8. kernel/sched/cpufreq_schedutil.c — Schedutil         │
  │  9. drivers/cpuidle/cpuidle.c     — CPUIdle core         │
  │  10. drivers/cpuidle/governors/menu.c — Menu governor    │
  │                                                           │
  │  Frameworks:                                             │
  │  11. drivers/base/power/domain.c   — genpd               │
  │  12. drivers/thermal/thermal_core.c — Thermal            │
  │  13. drivers/regulator/core.c      — Regulator           │
  │  14. drivers/clk/clk.c             — Common Clock        │
  │                                                           │
  │  Advanced:                                               │
  │  15. kernel/sched/fair.c           — EAS                  │
  │  16. kernel/power/energy_model.c   — Energy Model        │
  │  17. kernel/sched/pelt.c           — PELT                 │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. Vendor-Specific Resources

```
  Vendor PM Documentation
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Qualcomm:                                               │
  │  - RPMh (Resource Power Manager hardened) documentation  │
  │  - RPMH sleep/wake voting mechanism                      │
  │  - drivers/soc/qcom/ kernel source                       │
  │                                                           │
  │  Intel:                                                  │
  │  - Intel SDM: Volume 3, Chapter 14 (Power Management)   │
  │  - intel_pstate documentation                            │
  │  - Tools: turbostat, powertop (Intel-maintained)         │
  │                                                           │
  │  ARM:                                                    │
  │  - ARM Architecture Reference Manual (PM chapters)       │
  │  - PSCI / SCMI specifications                            │
  │  - big.LITTLE technology guides                          │
  │  - developer.arm.com/                                    │
  │                                                           │
  │  Texas Instruments:                                      │
  │  - AM335x/AM57xx Power Management guides                 │
  │  - TI wiki: processors.wiki.ti.com                       │
  │                                                           │
  │  NXP:                                                    │
  │  - i.MX Reference Manuals (Power chapter)                │
  │  - Application Notes on i.MX power management            │
  └──────────────────────────────────────────────────────────┘
```

---

## 7. Tools and Utilities Documentation

```
  PM Tool Documentation
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  PowerTOP:                                               │
  │  - https://github.com/fenrus75/powertop                  │
  │  - man powertop                                          │
  │                                                           │
  │  Turbostat:                                              │
  │  - tools/power/x86/turbostat/ (kernel source)            │
  │  - man turbostat                                         │
  │                                                           │
  │  cpupower:                                               │
  │  - tools/power/cpupower/ (kernel source)                 │
  │  - man cpupower, cpupower-frequency-info                 │
  │                                                           │
  │  perf (power events):                                    │
  │  - perf stat -e power/energy-pkg/                         │
  │  - perf record -e power:cpu_frequency                     │
  │  - man perf-stat                                         │
  │                                                           │
  │  trace-cmd:                                              │
  │  - https://trace-cmd.org/                                 │
  │  - trace-cmd record -e power -e rpm                      │
  │                                                           │
  │  KernelShark:                                            │
  │  - Visual trace analysis (works with trace-cmd output)   │
  │  - kernelshark.org                                       │
  └──────────────────────────────────────────────────────────┘
```

---

## 8. Books

```
  Recommended Books
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Linux PM specific:                                      │
  │  - "Linux Device Drivers" (LDD3) — Chapter on PM        │
  │  - "Linux Kernel Development" by Robert Love — PM ch    │
  │  - "Essential Linux Device Drivers" — PM sections        │
  │                                                           │
  │  Embedded/SoC PM:                                        │
  │  - "Embedded Linux Primer" — Power management chapter    │
  │  - "Building Embedded Linux Systems" — PM optimization   │
  │                                                           │
  │  Hardware/Architecture:                                   │
  │  - ARM System Developer's Guide — Power management       │
  │  - Intel Architecture Software Developer's Manual Vol 3  │
  │                                                           │
  │  General:                                                │
  │  - "Power Management for Embedded Systems" (textbook)    │
  └──────────────────────────────────────────────────────────┘
```

---

## Summary

- Kernel in-tree documentation (Documentation/power/) is the primary authoritative reference
- ACPI and ARM PSCI/SCMI specifications define hardware PM interfaces
- LWN.net provides excellent kernel development context for PM features
- Conference talks (LPC, ELC) offer practical implementation insights
- Source code reading: start with headers (pm.h), then runtime.c, then subsystem-specific
- Vendor documentation essential for SoC-specific PM (Qualcomm RPMh, Intel pstate, ARM PSCI)
- Tools documentation: PowerTOP, turbostat, perf, trace-cmd all have dedicated guides

---

[Previous Chapter: PM in Other Operating Systems ←](Chapter_26_Other_OS_PM.md) | [Next Chapter: Interview Preparation →](Chapter_28_Interview_Prep.md)
