# Chapter 25: Glossary of Power Management Terms

## Learning Goals
- Reference key PM terminology quickly
- Understand acronyms used across PM subsystems
- Clarify distinctions between similar PM concepts

---

## A

**ACPI** — Advanced Configuration and Power Interface. Industry standard (Intel/Microsoft/HP/Toshiba) defining hardware PM interfaces. Defines G/S/C/D states. Primary PM interface on x86 platforms.

**Active Cooling** — Removing heat using power-consuming mechanisms (fans, Peltier coolers). Contrasts with passive cooling (throttling). Triggered by ACTIVE trip points.

**ALPM** — Aggressive Link Power Management. SATA feature that puts PHY links in low-power states during idle periods.

**APM** — Advanced Power Management. Legacy BIOS-based PM standard (1992-2004), replaced by ACPI. Simple interface: BIOS controlled everything.

**ASPM** — Active State Power Management. PCI Express power feature that puts PCIe links into L0s/L1 low-power states during idle periods.

**Autosuspend** — Runtime PM feature where device automatically suspends after a configurable idle timeout (`autosuspend_delay_ms`). Prevents rapid on/off cycling.

---

## B

**Bang-Bang Governor** — Thermal governor using simple on/off control. Above trip → cooling state = max (fan full). Below trip-hysteresis → cooling state = 0 (fan off). No intermediate states.

**big.LITTLE** — ARM's heterogeneous CPU architecture combining high-performance "big" cores (Cortex-A7x) with energy-efficient "LITTLE" cores (Cortex-A5x). Basis for EAS.

**Break-Even Time** — Minimum idle duration where entering a deeper C-state saves energy. If idle < break-even time, the entry/exit overhead wastes more energy than staying in a shallower state.

**Buck Converter** — Switching voltage regulator that steps down voltage using inductors and PWM. 85-95% efficient. Used for high-current digital rails (CPU, DDR).

---

## C

**C-State** — CPU idle power state (ACPI). C0 = active, C1 = halt, C3 = sleep (caches flushed), C6 = deep power down. Deeper states save more power but have higher exit latency.

**CCF** — Common Clock Framework. Linux subsystem managing clock tree — rates, parents, gating, dividers. Consumer and provider APIs.

**Clock Gating** — Disabling clock signal to a logic block. Eliminates dynamic power but leakage remains. State preserved. Fast entry/exit (nanoseconds).

**Cooling Device** — Thermal framework actuator that reduces temperature. Types: cpufreq-cooling (reduce CPU freq), devfreq-cooling (reduce GPU freq), fan-cooling (increase airflow).

**CPUFreq** — Linux subsystem for CPU frequency scaling. Uses governors (schedutil, ondemand) to select operating frequency. Works with regulator framework for voltage.

**CPUIdle** — Linux subsystem managing CPU idle states. Governors (menu, TEO) predict idle duration and select appropriate C-state.

---

## D

**D-State** — Device power state (ACPI). D0 = fully on, D1/D2 = intermediate, D3hot = software off (bus power), D3cold = hardware off (no power).

**dev_pm_ops** — Linux structure defining PM callbacks for device drivers. Contains system sleep (.suspend/.resume), runtime PM (.runtime_suspend/.runtime_resume), and hibernate (.freeze/.thaw) callbacks.

**direct_complete** — Optimization where system suspend skips device callbacks if device is already runtime-suspended. Reduces suspend/resume time.

**DPM List** — Device PM list. Ordered list of all devices for suspend/resume. Children before parents during suspend, parents before children during resume.

**DVFS** — Dynamic Voltage and Frequency Scaling. Adjusting CPU/GPU voltage and frequency together for power savings. Power = C × V^2 × f.

**DynamIQ** — ARM's successor to big.LITTLE. Allows mixing different CPU types within a single cluster (e.g., 1 big + 3 LITTLE in same cluster).

---

## E

**EAS** — Energy-Aware Scheduling. Linux scheduler feature for heterogeneous CPUs. Places tasks on most energy-efficient CPU using the Energy Model.

**Energy Model (EM)** — Framework mapping CPU performance states (OPP) to power consumption. Used by EAS for task placement decisions.

**EPP** — Energy Performance Preference. Intel HWP hint (0-255) controlling hardware's power-performance tradeoff. 0 = max performance, 255 = max energy saving.

---

## F

**Freeze** — Process state during system suspend. Frozen processes cannot run until thawed during resume. Implemented via fake signals.

**freq_qos** — Frequency QoS. Constraints on CPUFreq min/max frequency. Used by thermal cooling to cap maximum frequency.

---

## G

**G-State** — Global system state (ACPI). G0 = working, G1 = sleeping (S1-S4), G2 = soft off (S5), G3 = mechanical off.

**genpd** — Generic Power Domain framework. Manages SoC power domains — automatically powers domains on/off based on runtime PM status of attached devices.

---

## H

**HWP** — Hardware P-states (Intel). Hardware autonomously selects CPU frequency based on EPP hint. Faster than OS-managed P-states.

**Hysteresis** — Temperature margin below a trip point before deactivating cooling. Prevents oscillation at trip boundaries. Example: trip=85°C, hysteresis=5°C → deactivate at 80°C.

---

## I

**IPA** — Intelligent Power Allocation. Thermal governor using PID controller to distribute power budget across cooling device actors proportionally. Maximizes performance within thermal limits.

**IRQ Wake** — Interrupt capable of waking the system from sleep. Configured via `enable_irq_wake()`. Only GIC/APIC wakeup-capable IRQs can exit suspend.

---

## L

**Ladder Governor** — CPUIdle governor that moves one state at a time (up on short idle, down on long idle). Simple but less optimal than menu governor. Used on periodic-tick systems.

**LDO** — Low Dropout Regulator. Linear voltage regulator. Efficiency = Vout/Vin. Lower noise than buck converters. Used for noise-sensitive analog circuits.

---

## M

**Menu Governor** — Default CPUIdle governor for tickless systems. Predicts idle duration using timer information and historical correction. Selects deepest C-state that fits.

**MWAIT** — Intel instruction for entering CPU idle states. Hints which C-state to enter. Used by intel_idle driver.

---

## O

**OPP** — Operating Performance Point. A valid frequency-voltage pair for DVFS. Defined in device tree opp-table nodes. Framework: `drivers/opp/`.

**Opportunistic Suspend** — Android power model where system automatically suspends when all wake locks are released. Driven by autosleep.

---

## P

**P-State** — Processor performance state (ACPI). P0 = max performance, P1 = next lower, etc. Each defines a frequency-voltage pair. Managed by CPUFreq.

**Passive Cooling** — Reducing heat generation by throttling performance (lowering frequency). No additional power consumed. Triggered by PASSIVE trip points.

**PELT** — Per-Entity Load Tracking. Scheduler mechanism tracking per-task CPU utilization as exponential moving average (~32ms half-life). Used by EAS and schedutil.

**PM QoS** — Power Management Quality of Service. Framework for devices/apps to set latency/throughput constraints. Prevents PM from violating latency requirements.

**PMIC** — Power Management IC. Chip containing multiple voltage regulators (bucks, LDOs), battery charger, and other PM functions. Controlled via I2C/SPI.

**Power Gating** — Cutting power supply to a logic block entirely. Eliminates both dynamic and leakage power. State lost — must save/restore. Higher latency than clock gating.

**PSCI** — Power State Coordination Interface. ARM standard for CPU PM operations (on/off, suspend, idle). Implemented by ARM Trusted Firmware. Used by Linux for CPU hotplug and idle.

---

## R

**Race-to-Idle** — Strategy of running at maximum performance to complete work quickly, then entering deep idle. Can be more efficient than running slowly for longer. Depends on idle power vs active power ratio.

**RAPL** — Running Average Power Limit. Intel technology for measuring and limiting power consumption of CPU, DRAM, and GPU via MSRs. Used by perf and turbostat.

**Regulator** — Voltage or current supply managed by the Linux regulator framework. Types: buck (switching), LDO (linear), boost (step-up).

**Retention** — Low-power state where logic is powered off but SRAM contents preserved via separate retention power. Faster resume than full power-gate (no register restore needed).

**Runtime PM** — Per-device power management. Devices individually power up/down based on usage (pm_runtime_get/put). Independent of system sleep.

---

## S

**S-State** — System sleep state (ACPI). S0 = working, S1 = standby (CPU stopped), S2 = deeper standby, S3 = suspend-to-RAM, S4 = hibernate, S5 = soft-off.

**s2idle** — Suspend-to-Idle. Software-only suspend: freeze processes, suspend devices, enter deepest CPU idle. No firmware support needed. Faster resume than S3.

**Schedutil** — CPUFreq governor integrated with scheduler. Sets frequency proportional to CPU utilization. Required for EAS. Replaced ondemand for most workloads.

**SCMI** — System Control and Management Interface. ARM standard protocol for communicating with platform firmware for clock, power, sensor, and performance management.

**Step-Wise Governor** — Default thermal governor. Increases/decreases cooling state by one step per polling interval based on temperature trend relative to trip points.

**Sustainable Power** — Maximum continuous power a device can dissipate thermally. Used by IPA governor as the steady-state power budget for PID control.

---

## T

**TEO Governor** — Timer Events Oriented cpuidle governor. Uses timer event statistics (hit/miss per state) to predict idle duration. Better than menu for timer-heavy workloads.

**Thermal Zone** — Logical grouping of temperature sensor + trip points + bound cooling devices. Managed by thermal framework. Polled periodically.

**Tickless (NO_HZ)** — Kernel mode where periodic timer tick is stopped during idle. Allows CPUs to stay in deep C-states longer. Essential for power-efficient idle.

**Trip Point** — Temperature threshold triggering PM action. Types: ACTIVE (fan on), PASSIVE (throttle), HOT (emergency), CRITICAL (shutdown).

---

## U

**Uclamp** — Utilization Clamping. Per-task performance hints: uclamp_min (floor, boost) and uclamp_max (cap, limit). Used by EAS for task placement. Set via sched_setattr() or cgroups.

**Usage Count** — Runtime PM reference counter. When >0, device stays active. When drops to 0, device eligible for suspension. Managed by pm_runtime_get/put.

---

## W

**Wakeup Source** — Kernel object tracking events that prevent/abort system suspend. Created by drivers for devices that can generate wakeup events (buttons, RTC, network).

**Wake Lock** — Android mechanism preventing system from suspending. Mapped to wakeup sources in kernel. Deprecated in favor of direct wakeup_source API.

**WFI** — Wait For Interrupt. ARM instruction halting CPU until interrupt arrives. Simplest idle state (C1 equivalent). Minimal power savings but instant wake.

---

## Summary

This glossary covers 70+ PM terms across all subsystems. Key relationships:
- C-states (CPU idle) vs P-states (CPU performance) vs D-states (device power)
- Clock gating (fast, state preserved) vs power gating (deep, state lost) vs retention (deep, state preserved via SRAM)
- Active cooling (fan) vs passive cooling (throttle)
- Runtime PM (per-device) vs system suspend (whole system)
- DVFS = CPUFreq (frequency) + regulator (voltage) + OPP (mapping)

---

[Previous Chapter: Architecture Diagrams ←](Chapter_24_Architecture_Diagrams.md) | [Next Chapter: PM in Other Operating Systems →](Chapter_26_Other_OS_PM.md)
