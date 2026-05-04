# Chapter 1: Foundations of Power Management

## Learning Goals
- Understand why power management is essential in modern computing
- Learn the relationship between power, energy and performance
- Know key metrics for evaluating energy efficiency
- Understand the fundamental physics of power consumption in semiconductors
- Learn the core terminology used throughout power management

---

## 1. What Is Power Management?

Power management is the coordinated control of **voltage**, **frequency**, **clock gating**, and **power domains** to minimize energy consumption while maintaining required system performance.

```
  ┌──────────────────────────────────────────────────────┐
  │              POWER MANAGEMENT GOALS                   │
  │                                                       │
  │  ┌──────────────────────────────────────────────┐    │
  │  │  1. Minimize Power Consumption               │    │
  │  │     → Extend battery life                     │    │
  │  │     → Reduce heat generation                  │    │
  │  │     → Lower electricity costs                 │    │
  │  └──────────────────────────────────────────────┘    │
  │  ┌──────────────────────────────────────────────┐    │
  │  │  2. Maintain Required Performance             │    │
  │  │     → Meet real-time deadlines                │    │
  │  │     → Deliver user-expected responsiveness    │    │
  │  │     → Satisfy QoS constraints                 │    │
  │  └──────────────────────────────────────────────┘    │
  │  ┌──────────────────────────────────────────────┐    │
  │  │  3. Manage Thermal Envelope                   │    │
  │  │     → Prevent thermal throttling              │    │
  │  │     → Ensure component safety                 │    │
  │  │     → Meet regulatory requirements            │    │
  │  └──────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────┘
```

---

## 2. Why Power Management Matters

### 2.1 Mobile and Embedded Devices

Battery-powered devices have a finite energy budget. Every milliwatt saved translates directly to longer battery life.

```
  Battery Capacity (mAh) × Voltage (V)
  Energy Budget = ────────────────────────── = Runtime (hours)
                  Average Power Draw (mW)

  Example:
  4000 mAh × 3.7V = 14,800 mWh
  At 2W average → 7.4 hours
  At 1W average → 14.8 hours (2x improvement!)
```

### 2.2 Data Centers

Power constitutes 30-50% of data center operating costs. A single rack can consume 10-40 kW.

### 2.3 Automotive Systems

In electric vehicles, every watt consumed by the infotainment/ADAS system reduces driving range. Power management is safety-critical for thermal control.

### 2.4 Environmental Impact

Global data centers consume ~1-2% of worldwide electricity. Efficient power management directly reduces carbon emissions.

---

## 3. Physics of Power Consumption

### 3.1 Dynamic Power

Dynamic power is consumed when transistors switch states:

```
  P_dynamic = α × C × V² × f

  Where:
    α = Activity factor (0-1, fraction of transistors switching)
    C = Load capacitance
    V = Supply voltage
    f = Clock frequency

  KEY INSIGHT: Power scales with V² — halving voltage
  reduces dynamic power by 4x!
```

### 3.2 Static (Leakage) Power

Even when transistors are not switching, current leaks through the gate oxide and between source/drain:

```
  P_static = V × I_leak

  Leakage current increases with:
  - Higher temperature (exponential)
  - Smaller transistor geometries
  - Lower threshold voltage (Vth)
```

### 3.3 Total Power Breakdown

```
  ┌────────────────────────────────────────────────────┐
  │              TOTAL CHIP POWER                       │
  │                                                     │
  │  ┌───────────────────┐  ┌──────────────────────┐   │
  │  │  Dynamic Power    │  │  Static Power         │   │
  │  │  (Switching)      │  │  (Leakage)            │   │
  │  │                   │  │                        │   │
  │  │  α × C × V² × f  │  │  V × I_leak           │   │
  │  │                   │  │                        │   │
  │  │  Reduced by:      │  │  Reduced by:           │   │
  │  │  - Lower voltage  │  │  - Power gating        │   │
  │  │  - Lower frequency│  │  - Lower temperature   │   │
  │  │  - Clock gating   │  │  - Higher Vth          │   │
  │  │  - Fewer switches │  │  - Turning off domains │   │
  │  └───────────────────┘  └──────────────────────┘   │
  │                                                     │
  │  Modern chips (7nm and below):                      │
  │    Static power can be 30-50% of total power!       │
  └────────────────────────────────────────────────────┘
```

---

## 4. Power Management Techniques Overview

### 4.1 Taxonomy of PM Techniques

```
  Power Management Techniques
  │
  ├── Voltage/Frequency Scaling (DVFS)
  │   ├── CPU frequency governors
  │   ├── Operating Performance Points (OPP)
  │   └── Adaptive voltage scaling
  │
  ├── Clock Management
  │   ├── Clock gating (stop unused clocks)
  │   ├── Clock scaling (reduce frequency)
  │   └── Clock tree optimization
  │
  ├── Power State Management
  │   ├── Active states (C0 with P-states)
  │   ├── Idle states (C1-Cn)
  │   ├── Sleep states (S1-S4)
  │   └── Off state (S5/G3)
  │
  ├── Power Domain Control
  │   ├── Power gating (cut power to blocks)
  │   ├── Power islands
  │   └── Retention states
  │
  ├── Thermal Management
  │   ├── Thermal throttling
  │   ├── Fan control
  │   └── Skin temperature limits
  │
  └── Software Optimization
      ├── Energy-aware scheduling
      ├── I/O consolidation (batching)
      ├── Wakelock management
      └── Race-to-idle (finish fast, sleep longer)
```

### 4.2 Hardware vs Software PM Responsibility

```
  ┌──────────────────────────┬───────────────────────────┐
  │      HARDWARE             │       SOFTWARE            │
  ├──────────────────────────┼───────────────────────────┤
  │ Voltage regulators       │ PM policy (governors)     │
  │ Clock generators (PLL)   │ Idle detection            │
  │ Power switches (FETs)    │ Workload prediction       │
  │ Thermal sensors          │ Task scheduling           │
  │ Frequency dividers       │ State machine mgmt        │
  │ Power state HW (C/P/D)   │ Driver PM callbacks       │
  │ PMIC                     │ Userspace power tools     │
  │ PMU (Power Mgmt Unit)    │ Wakeup source management  │
  └──────────────────────────┴───────────────────────────┘
```

---

## 5. Power vs Energy vs Performance

### 5.1 Key Distinctions

```
  Power (Watts):   Rate of energy consumption at an instant
                   P = V × I

  Energy (Joules): Total energy consumed over time
                   E = P × t (for constant power)
                   E = ∫ P(t) dt (in general)

  Performance:     Work done per unit time
                   IPC × Frequency (for CPUs)

  Key Insight:
  ┌─────────────────────────────────────────────────────┐
  │  Lower power does NOT always mean lower energy!     │
  │                                                     │
  │  Race-to-idle: Run fast (high power) → finish →     │
  │  sleep (near zero power) → Lower TOTAL energy       │
  │                                                     │
  │  Slow execution: Run slow (low power) → takes       │
  │  longer → may use MORE total energy                 │
  └─────────────────────────────────────────────────────┘
```

### 5.2 Race-to-Idle Example

```
  Time ──────────────────────────────────────────►

  Strategy A: High performance (5W for 2 seconds)
  ████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
  5W × 2s = 10J total, then idle at 0.1W

  Strategy B: Low power (2W for 8 seconds)
  ████████████████████████████████████████░░░░░░░
  2W × 8s = 16J total, then idle at 0.1W

  Strategy A uses LESS total energy despite higher peak power!
```

---

## 6. Energy Efficiency Metrics

### 6.1 Common Metrics

| Metric | Formula | Used For |
|--------|---------|----------|
| Energy per operation | E = P × t_op | CPU/GPU efficiency |
| MIPS/Watt | MIPS / Power | Processor comparison |
| Performance/Watt | Benchmark / Power | System comparison |
| Battery life | Capacity / Avg_Power | Mobile devices |
| PUE (Power Usage Effectiveness) | Total_Facility / IT_Power | Data centers |
| EDP (Energy-Delay Product) | Energy × Delay | Combined metric |

### 6.2 DVFS Efficiency

```
  Frequency  Voltage  Dynamic Power  Time     Energy
  ─────────  ───────  ─────────────  ─────    ──────
  2.0 GHz    1.2V     α×C×1.44×2G   1x       1.00x
  1.5 GHz    1.0V     α×C×1.00×1.5G 1.33x    0.69x
  1.0 GHz    0.8V     α×C×0.64×1.0G 2x       0.44x
  0.5 GHz    0.6V     α×C×0.36×0.5G 4x       0.25x

  Note: Actual savings depend on workload characteristics
  and leakage power contribution.
```

---

## 7. Power Management in the Linux Context

### 7.1 Kernel's Role

The Linux kernel sits between hardware power management mechanisms and userspace applications:

```
  ┌──────────────────────────────────────────────┐
  │            USERSPACE APPLICATIONS             │
  │  (want: performance, responsiveness)          │
  ├──────────────────────────────────────────────┤
  │            KERNEL PM FRAMEWORK                │
  │                                               │
  │  ┌──────────────┐  ┌───────────────┐         │
  │  │   Policy      │  │  Mechanism    │         │
  │  │  (Governors)  │  │  (Drivers)    │         │
  │  │              │  │               │         │
  │  │  "WHEN and   │  │  "HOW to      │         │
  │  │   HOW MUCH"  │  │   actually    │         │
  │  │              │  │   change HW"  │         │
  │  └──────────────┘  └───────────────┘         │
  ├──────────────────────────────────────────────┤
  │            HARDWARE                           │
  │  (provides: power states, DVFS, gating)       │
  └──────────────────────────────────────────────┘
```

### 7.2 Key Kernel Subsystems

| Subsystem | Location | Purpose |
|-----------|----------|---------|
| PM Core | `kernel/power/` | System suspend, PM framework |
| CPUFreq | `drivers/cpufreq/` | CPU frequency scaling |
| CPUIdle | `drivers/cpuidle/` | CPU idle state management |
| Runtime PM | `drivers/base/power/runtime.c` | Per-device runtime PM |
| Power Domains | `drivers/base/power/domain.c` | Generic power domains |
| Clock Framework | `drivers/clk/` | Clock tree management |
| Regulator | `drivers/regulator/` | Voltage regulator control |
| Thermal | `drivers/thermal/` | Thermal management |

---

## 8. Key Terminology

| Term | Definition |
|------|-----------|
| **DVFS** | Dynamic Voltage and Frequency Scaling — adjusting both voltage and frequency together |
| **Power Gating** | Cutting power supply to unused blocks to eliminate leakage |
| **Clock Gating** | Stopping the clock signal to unused blocks (saves dynamic power only) |
| **OPP** | Operating Performance Point — a valid voltage/frequency pair |
| **Governor** | Policy algorithm that decides when to change power states |
| **C-state** | CPU idle state (C0=active, C1-Cn=progressively deeper idle) |
| **P-state** | CPU performance state (different voltage/frequency at C0) |
| **S-state** | System sleep state (S0=on, S1-S4=progressively deeper sleep) |
| **D-state** | Device power state (D0=on, D1-D3=reduced power) |
| **Wakeup latency** | Time to transition from low power state back to active |
| **PMIC** | Power Management Integrated Circuit — controls voltage rails |
| **EAS** | Energy-Aware Scheduling — scheduler considers energy impact |
| **Race-to-idle** | Run fast, finish quickly, enter deep idle to save total energy |

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `kernel/power/main.c` | PM framework core |
| `kernel/power/suspend.c` | System suspend infrastructure |
| `include/linux/pm.h` | PM data structures and macros |
| `include/linux/pm_runtime.h` | Runtime PM API declarations |
| `Documentation/power/` | Kernel PM documentation |
| `drivers/base/power/` | Device PM framework |

---

## Interview Questions

**Q1: What is the difference between power and energy?**
**A:** Power (watts) is the instantaneous rate of energy consumption (P = V × I). Energy (joules) is the total power consumed over time (E = ∫P dt). A system can have high instantaneous power but low total energy if it finishes quickly (race-to-idle), or low power but high energy if it runs for a long time.

**Q2: Explain the P_dynamic = α × C × V² × f formula.**
**A:** Dynamic power equals the activity factor (α, fraction of switching transistors) times load capacitance (C) times voltage squared (V²) times frequency (f). The V² term is critical — halving voltage reduces dynamic power by 4x, which is why voltage scaling is the most effective power reduction technique.

**Q3: What is DVFS and why is it effective?**
**A:** DVFS (Dynamic Voltage and Frequency Scaling) reduces both voltage and frequency together. Since dynamic power scales with V²×f, reducing both gives cubic-like power reduction. It works because lower frequencies allow lower voltages while maintaining timing margins.

**Q4: Explain the race-to-idle concept.**
**A:** Race-to-idle means running at high frequency to complete work quickly, then entering a deep idle state. Despite higher peak power, the total energy may be lower because: (1) the work completes faster, and (2) the system spends more time in very-low-power idle states. This is especially effective when idle power is much lower than active power.

**Q5: What is the difference between clock gating and power gating?**
**A:** Clock gating stops the clock signal to a block, eliminating dynamic power (switching) but not leakage power — the block remains powered. Power gating cuts the power supply entirely, eliminating both dynamic and static (leakage) power, but requires saving/restoring state and has longer entry/exit latency.

---

## Summary

- Power management balances energy efficiency against performance requirements
- Dynamic power (P = αCV²f) dominates in active states; static power (leakage) matters at idle
- DVFS exploits V² relationship for significant power savings
- Race-to-idle can save more total energy than running slowly
- Linux kernel separates PM policy (governors) from mechanism (drivers)
- Key kernel PM subsystems: CPUFreq, CPUIdle, Runtime PM, Power Domains, Thermal

---

[Next Chapter: History and Evolution of Power Management →](Chapter_02_History_Evolution.md)
