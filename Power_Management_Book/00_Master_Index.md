# Linux Power Management — Complete Guide
# Master Index

## Overview

This book covers Linux power management from hardware fundamentals through kernel frameworks, driver integration, embedded optimization, and interview preparation. Power management is critical in embedded systems, mobile devices, automotive platforms, and servers — affecting battery life, thermal behavior, and energy efficiency.

---

## Book Structure

### Part I: Foundations (Chapters 1-3)
| # | Chapter | Key Topics |
|---|---------|-----------|
| 1 | [Foundations of Power Management](Chapter_01_Foundations.md) | Energy efficiency, power consumption, performance trade-offs, metrics |
| 2 | [History and Evolution](Chapter_02_History_Evolution.md) | APM, ACPI, Linux PM evolution |
| 3 | [Hardware Architecture](Chapter_03_Hardware_Architecture.md) | CPU power states, voltage/frequency scaling HW, power domains |

### Part II: Power States (Chapters 4-5)
| # | Chapter | Key Topics |
|---|---------|-----------|
| 4 | [Power States in Systems](Chapter_04_Power_States.md) | C-states, P-states, D-states, sleep/suspend states |
| 5 | [Linux PM Architecture](Chapter_05_PM_Architecture.md) | PM framework, subsystems, kernel layers, policies |

### Part III: CPU Power Management (Chapters 6-8)
| # | Chapter | Key Topics |
|---|---------|-----------|
| 6 | [CPU Power Management](Chapter_06_CPU_Power_Management.md) | Frequency scaling, DVFS, idle management, governors |
| 7 | [CPUFreq Subsystem](Chapter_07_CPUFreq.md) | CPUFreq drivers, governors, frequency policies |
| 8 | [CPUIdle Subsystem](Chapter_08_CPUIdle.md) | Idle state management, selection algorithms, idle governors |

### Part IV: Device & Runtime PM (Chapters 9-11)
| # | Chapter | Key Topics |
|---|---------|-----------|
| 9 | [Device Power Management](Chapter_09_Device_PM.md) | Device power states, PM framework, suspend/resume |
| 10 | [Runtime Power Management](Chapter_10_Runtime_PM.md) | Runtime PM concept, transitions, APIs |
| 11 | [System Suspend and Resume](Chapter_11_Suspend_Resume.md) | Suspend-to-RAM, suspend-to-disk, resume process |

### Part V: Power Infrastructure (Chapters 12-15)
| # | Chapter | Key Topics |
|---|---------|-----------|
| 12 | [Power Domains](Chapter_12_Power_Domains.md) | Generic power domains, SoC power domains, genpd |
| 13 | [Clock Framework](Chapter_13_Clock_Framework.md) | Clock tree, clock gating, CCF (Common Clock Framework) |
| 14 | [Regulator Framework](Chapter_14_Regulator_Framework.md) | Voltage regulators, regulator drivers, power supply |
| 15 | [Thermal Management](Chapter_15_Thermal_Management.md) | Thermal zones, cooling devices, thermal governors |

### Part VI: Integration (Chapters 16-19)
| # | Chapter | Key Topics |
|---|---------|-----------|
| 16 | [PM in Device Drivers](Chapter_16_Driver_PM.md) | Power-aware drivers, suspend/resume callbacks, runtime PM |
| 17 | [PM and Device Tree](Chapter_17_Device_Tree_PM.md) | Power domains, clocks, regulators in DT |
| 18 | [Embedded System PM](Chapter_18_Embedded_PM.md) | SoC optimization, battery devices, automotive PM |
| 19 | [Energy-Aware Scheduling](Chapter_19_EAS.md) | EAS, scheduler-PM interaction, capacity/energy model |

### Part VII: Debugging & Tools (Chapters 20-21)
| # | Chapter | Key Topics |
|---|---------|-----------|
| 20 | [PM Debugging](Chapter_20_PM_Debugging.md) | Suspend failures, PM logs, tracing power events |
| 21 | [Power Monitoring Tools](Chapter_21_PM_Tools.md) | powertop, turbostat, perf energy profiling |

### Part VIII: Reference (Chapters 22-28)
| # | Chapter | Key Topics |
|---|---------|-----------|
| 22 | [Kernel Source Code Map](Chapter_22_Source_Code.md) | Key directories, files, data structures |
| 23 | [PM Flow Diagrams](Chapter_23_Flow_Diagrams.md) | CPUFreq, runtime PM, suspend, resume flows |
| 24 | [Architecture Diagrams](Chapter_24_Architecture_Diagrams.md) | Full PM architecture, state transitions, power domains |
| 25 | [Glossary](Chapter_25_Glossary.md) | All PM terminology definitions |
| 26 | [PM in Other Operating Systems](Chapter_26_Other_OS_PM.md) | Windows, macOS, Android, RTOS power management |
| 27 | [Documentation and References](Chapter_27_References.md) | Kernel docs, ACPI specs, books, papers |
| 28 | [Interview Preparation](Chapter_28_Interview_Preparation.md) | Comprehensive Q&A, scenarios, quick reference |

---

## Reading Paths

**Embedded Engineer Track:**
Ch 1 → 3 → 4 → 6 → 7 → 8 → 12 → 13 → 14 → 17 → 18 → 19 → 28

**Driver Developer Track:**
Ch 1 → 5 → 9 → 10 → 16 → 17 → 13 → 14 → 20 → 28

**Thermal/Performance Track:**
Ch 1 → 4 → 6 → 7 → 15 → 19 → 21 → 20 → 28

**Interview Fast Track:**
Ch 1 → 4 → 5 → 7 → 8 → 10 → 11 → 15 → 25 → 28

---

## Linux Power Management Stack Overview

```
  ┌──────────────────────────────────────────────────────────────┐
  │                      USER SPACE                              │
  │                                                              │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────┐ │
  │  │ powertop │  │turbostat │  │ systemd  │  │ thermal-    │ │
  │  │          │  │          │  │ (sleep/  │  │ daemon      │ │
  │  │          │  │          │  │  suspend)│  │             │ │
  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬──────┘ │
  ├───────┴──────────────┴─────────────┴───────────────┴────────┤
  │                   /sys/power  /sys/class/thermal             │
  │                   /sys/devices/.../power                     │
  ├─────────────────────────────────────────────────────────────┤
  │                        KERNEL                                │
  │                                                              │
  │  ┌─── PM Core ────────────────────────────────────────┐     │
  │  │  kernel/power/: suspend, hibernate, PM framework    │     │
  │  └────────────────────────────────────────────────────┘     │
  │                                                              │
  │  ┌─── CPU PM ──────────┐  ┌─── Device PM ─────────────┐    │
  │  │  CPUFreq (governors)│  │  Runtime PM (autosuspend)  │    │
  │  │  CPUIdle (C-states) │  │  System suspend/resume     │    │
  │  │  DVFS               │  │  Power domains (genpd)     │    │
  │  └────────────────────┘  └────────────────────────────┘    │
  │                                                              │
  │  ┌─── Infrastructure ─────────────────────────────────┐     │
  │  │  Clock Framework (CCF) │ Regulator Framework       │     │
  │  │  Thermal Framework     │ Energy Model              │     │
  │  │  OPP (Operating Perf Points)                       │     │
  │  └────────────────────────────────────────────────────┘     │
  │                                                              │
  │  ┌─── Scheduler ──────────────────────────────────────┐     │
  │  │  Energy-Aware Scheduling (EAS)                      │     │
  │  │  Capacity-aware task placement                      │     │
  │  └────────────────────────────────────────────────────┘     │
  │                                                              │
  ├─────────────────────────────────────────────────────────────┤
  │                      HARDWARE                                │
  │  PMIC │ Voltage Regulators │ Clock Generators │ Thermal      │
  │  CPU (P-states/C-states) │ Power Domains │ DVFS controllers  │
  └──────────────────────────────────────────────────────────────┘
```

---

**Total: 28 Chapters + Master Index**
