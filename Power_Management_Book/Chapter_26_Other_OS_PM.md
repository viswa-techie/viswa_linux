# Chapter 26: Power Management in Other Operating Systems

## Learning Goals
- Compare Linux PM with Windows, macOS, RTOS, and QNX approaches
- Understand common PM concepts across operating systems
- Know OS-specific PM features and their Linux equivalents
- Gain broader perspective for interviews

---

## 1. Comparison Overview

```
  PM Feature Comparison Across Operating Systems
  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  Feature        │ Linux    │ Windows  │ macOS    │ RTOS      │
  │  ────────────────┼──────────┼──────────┼──────────┼──────────│
  │  Sleep states   │ s2idle   │ Modern   │ Power    │ Tickless  │
  │                 │ S3, S4   │ Standby  │ Nap      │ idle      │
  │                 │          │ S3, S4   │ S3       │           │
  │                 │          │          │          │           │
  │  CPU freq       │ CPUFreq  │ PPM      │ DVFS     │ App-     │
  │  scaling        │ governors│ (Proc    │ (kernel  │ controlled│
  │                 │          │ Power    │ managed) │           │
  │                 │          │ Manager) │          │           │
  │                 │          │          │          │           │
  │  Device PM      │ Runtime  │ Device   │ IOKit    │ Custom    │
  │                 │ PM       │ Power    │ Power    │ per-driver│
  │                 │ framework│ IRP      │ Mgmt     │           │
  │                 │          │          │          │           │
  │  Thermal       │ Thermal  │ DTM      │ IOKit    │ Custom    │
  │                 │ framework│ (MS)     │ thermal  │           │
  │                 │          │ DPTF(Int)│          │           │
  │                 │          │          │          │           │
  │  App PM hints  │ uclamp   │ Power    │ App Nap  │ N/A       │
  │                 │ cgroups  │ Throttle │ QoS      │           │
  └──────────────────────────────────────────────────────────────┘
```

---

## 2. Windows Power Management

```
  Windows PM Architecture
  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  ┌── Userspace ──────────────────────────────────────────┐  │
  │  │  Power Plans: Balanced │ Power Saver │ High Perf      │  │
  │  │  powercfg.exe — command-line PM control               │  │
  │  └──────────────────────────┬────────────────────────────┘  │
  │                             │                                │
  │  ┌──────────────────────────▼────────────────────────────┐  │
  │  │  Kernel: Power Manager (PoFx)                         │  │
  │  │                                                        │  │
  │  │  Device Power IRP:                                     │  │
  │  │  IRP_MN_SET_POWER  ← Like Linux dev_pm_ops callbacks  │  │
  │  │  IRP_MN_QUERY_POWER                                   │  │
  │  │                                                        │  │
  │  │  Processor Power Manager (PPM):                       │  │
  │  │  ← Like Linux CPUFreq + CPUIdle combined              │  │
  │  │  Manages P-states and C-states via ACPI               │  │
  │  │                                                        │  │
  │  │  Runtime PM (PoFx):                                   │  │
  │  │  PoFxRegisterDevice()  ← Like pm_runtime_enable()     │  │
  │  │  PoFxActivateComponent ← Like pm_runtime_get()        │  │
  │  │  PoFxIdleComponent     ← Like pm_runtime_put()        │  │
  │  │                                                        │  │
  │  │  Modern Standby (S0 Low Power Idle):                  │  │
  │  │  ← Like Linux s2idle                                   │  │
  │  │  System stays in S0 but devices suspended              │  │
  │  │  Network/app events can wake briefly                   │  │
  │  └──────────────────────────────────────────────────────┘  │
  │                                                               │
  │  Key differences from Linux:                                 │
  │  - Power Plans provide user-facing profiles (no Linux equiv) │
  │  - DPTF: Intel Dynamic Platform Thermal Framework            │
  │  - DRIPS: Deepest Runtime Idle Platform State monitoring     │
  │  - Connected Standby replaces S3 on modern hardware          │
  └──────────────────────────────────────────────────────────────┘
```

---

## 3. macOS / iOS Power Management

```
  Apple PM Architecture
  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  IOKit Power Management (macOS):                             │
  │                                                               │
  │  IOService power states:                                     │
  │  ┌───────────────────────────────────────────────┐          │
  │  │  kIOPMPowerOff     (0) ← Like Linux D3cold    │          │
  │  │  kIOPMDoze         (1) ← Like Linux D1/D2     │          │
  │  │  kIOPMPowerOn      (2) ← Like Linux D0        │          │
  │  └───────────────────────────────────────────────┘          │
  │                                                               │
  │  Driver PM callbacks:                                        │
  │  setPowerState(powerStateOrdinal)                            │
  │  ← Like Linux .runtime_suspend/.runtime_resume              │
  │                                                               │
  │  Power assertions (macOS):                                   │
  │  IOPMAssertionCreateWithName()                               │
  │  ← Like Linux wakeup sources / Android wake locks            │
  │  Types: PreventUserIdleSystemSleep,                          │
  │         PreventUserIdleDisplaySleep                           │
  │                                                               │
  │  Apple Silicon (M-series) PM:                                │
  │  ┌───────────────────────────────────────────────┐          │
  │  │  - Unified memory architecture (less DDR PM)   │          │
  │  │  - DVFS per-cluster (like big.LITTLE)          │          │
  │  │  - Neural Engine PM (separate power domain)    │          │
  │  │  - Aggressive per-core power gating            │          │
  │  │  - QoS-based scheduling (similar to uclamp):   │          │
  │  │    QOS_CLASS_USER_INTERACTIVE                   │          │
  │  │    QOS_CLASS_BACKGROUND                         │          │
  │  │    ← Maps tasks to efficiency/performance cores │          │
  │  └───────────────────────────────────────────────┘          │
  │                                                               │
  │  App Nap (macOS):                                            │
  │  - Background apps get timer coalescing (batch wakeups)      │
  │  - CPU throttled for non-visible apps                        │
  │  - Similar concept to Android Doze mode                      │
  └──────────────────────────────────────────────────────────────┘
```

---

## 4. RTOS Power Management

```
  RTOS PM (FreeRTOS / Zephyr / QNX)
  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  FreeRTOS (bare-metal/MCU):                                  │
  │  ┌───────────────────────────────────────────────┐          │
  │  │  Tickless Idle:                                │          │
  │  │  - When no tasks ready, enter low-power mode   │          │
  │  │  - Suppress tick interrupt for idle duration    │          │
  │  │  - configUSE_TICKLESS_IDLE = 1                 │          │
  │  │  - vPortSuppressTicksAndSleep(expected_idle)   │          │
  │  │  ← Like Linux tickless (NO_HZ) + cpuidle       │          │
  │  │                                                │          │
  │  │  No formal PM framework — app directly controls│          │
  │  │  peripherals and clocks via HAL                 │          │
  │  └───────────────────────────────────────────────┘          │
  │                                                               │
  │  Zephyr RTOS:                                                │
  │  ┌───────────────────────────────────────────────┐          │
  │  │  PM subsystem (inspired by Linux):             │          │
  │  │  - pm_state: PM_STATE_ACTIVE, SUSPEND_TO_IDLE, │          │
  │  │    STANDBY, SUSPEND_TO_RAM, SUSPEND_TO_DISK    │          │
  │  │  - Device PM: pm_device_action_run()           │          │
  │  │  - pm_policy: residency-based state selection  │          │
  │  │  ← Most Linux-like RTOS PM framework           │          │
  │  └───────────────────────────────────────────────┘          │
  │                                                               │
  │  QNX (Automotive):                                           │
  │  ┌───────────────────────────────────────────────┐          │
  │  │  Power Manager resource manager (/dev/pm):     │          │
  │  │  - Modes: active, standby, sleep, off          │          │
  │  │  - PM policies configurable per mode            │          │
  │  │  - Clients register for mode change notification│          │
  │  │  - Used in automotive (ADAS, IVI systems)       │          │
  │  │  ← More centralized than Linux distributed PM   │          │
  │  └───────────────────────────────────────────────┘          │
  └──────────────────────────────────────────────────────────────┘
```

---

## 5. Key Concept Mapping

```
  Cross-OS PM Concept Mapping
  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  Linux Concept       → Windows         → macOS               │
  │  ─────────────────────────────────────────────────────────── │
  │  s2idle              → Modern Standby  → Power Nap          │
  │  S3 (suspend-to-RAM) → S3 Sleep       → Safe Sleep          │
  │  S4 (hibernate)      → Hibernate       → Hibernation        │
  │  Runtime PM          → PoFx (D-state)  → IOKit setPowerState│
  │  Wakeup sources      → PoFx wake       → Power assertions   │
  │  CPUFreq governors   → PPM policies    → (kernel managed)   │
  │  EAS / uclamp        → (limited)       → QoS classes        │
  │  PM QoS              → (limited)       → IOPMQos            │
  │  Thermal framework   → DTM / DPTF      → IOKit thermal      │
  │  Power domains       → PEP (Platform   → IOKit power tree   │
  │  (genpd)             │  Energy Plugin) │                     │
  │  Cgroups (cpu)       → Job Objects     → App Nap             │
  │  /sys/power/         → powercfg        → pmset              │
  │  autosleep           → (always-on      → (Power Nap)        │
  │  (Android)           │  standby)       │                     │
  └──────────────────────────────────────────────────────────────┘
```

---

## 6. Unique Features per OS

```
  Unique PM Features
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Linux unique:                                           │
  │  - Open-source PM — fully transparent and debuggable     │
  │  - EAS + Energy Model — most sophisticated energy sched  │
  │  - genpd — hierarchical power domain framework           │
  │  - ftrace PM events — detailed kernel-level PM tracing   │
  │                                                           │
  │  Windows unique:                                         │
  │  - Power Plans (user profiles: balanced/saver/perf)      │
  │  - DRIPS monitoring (SoC-level idle tracking)            │
  │  - Connected Standby (always-connected low power)        │
  │  - SleepStudy report (automatic PM analysis)             │
  │                                                           │
  │  macOS/iOS unique:                                       │
  │  - Tight HW-SW integration (Apple Silicon PM)            │
  │  - App Nap + Timer Coalescing (batch wakeups)            │
  │  - QoS classes for app-level power hints                 │
  │  - Proactive resource management (kill bg apps early)    │
  │                                                           │
  │  RTOS unique:                                            │
  │  - Deterministic PM with bounded latency                 │
  │  - Application directly controls peripherals             │
  │  - Minimal framework overhead                            │
  │  - Power budget designed at application level             │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does Linux PM compare to Windows PM?**
**A:** Both use ACPI S/C/D states on x86. Windows has user-facing Power Plans (no Linux equivalent) and Connected Standby (like Linux s2idle). Windows PoFx is similar to Linux runtime PM — drivers register devices and the framework manages idle/active transitions. Linux advantages: open-source debugging (ftrace), EAS/Energy Model for heterogeneous CPUs, genpd for SoC power domains. Windows advantages: better integration with proprietary hardware (Intel DPTF), SleepStudy automated PM analysis, user-friendly power profiles.

**Q2: How does QNX PM differ from Linux for automotive?**
**A:** QNX uses a centralized Power Manager resource manager (/dev/pm) where clients register for mode change notifications. Linux uses a distributed model — each subsystem (cpufreq, thermal, genpd, runtime PM) manages its own power independently. QNX provides deterministic PM transitions important for safety-critical ADAS. Linux (AAOS) uses Android PowerManager with garage mode and VHAL integration. Linux has richer PM frameworks (EAS, thermal governors) but QNX has simpler, more predictable PM behavior.

---

## Summary

- All modern OSes share fundamental PM concepts: sleep states, device PM, CPU DVFS, thermal management
- Linux has the most sophisticated open-source PM with EAS, genpd, and detailed tracing
- Windows focuses on user experience (Power Plans, Connected Standby, SleepStudy)
- macOS leverages tight HW-SW integration for Apple Silicon-specific PM
- RTOS provides deterministic PM with minimal overhead, suitable for MCU/safety-critical
- Cross-OS PM experience is transferable — C-states, D-states, DVFS, thermal concepts are universal

---

[Previous Chapter: Glossary ←](Chapter_25_Glossary.md) | [Next Chapter: Documentation and References →](Chapter_27_References.md)
