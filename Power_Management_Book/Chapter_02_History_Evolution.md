# Chapter 2: History and Evolution of Power Management

## Learning Goals
- Understand the historical evolution from no PM to modern PM frameworks
- Learn the APM standard and its limitations
- Understand ACPI architecture and its role in modern systems
- Trace the evolution of Linux power management subsystems
- Know how PM has evolved with process technology scaling

---

## 1. The Pre-Power-Management Era (Pre-1990)

### 1.1 Early Computing — No PM

Early computers had no concept of power management. They ran at full power whenever turned on.

```
  1970s-1980s Computing
  ┌─────────────────────────────────────────────┐
  │  - Desktop/mainframe only (wall power)       │
  │  - No battery devices                        │
  │  - Full power consumption always             │
  │  - CRT monitors draw 100+ watts             │
  │  - No idle detection                         │
  │  - Manual power on/off only                  │
  │                                              │
  │  Power management = toggle power switch      │
  └─────────────────────────────────────────────┘
```

### 1.2 First Steps: Screen Blanking

The earliest "power management" was CRT screen blanking in X Window System (1984) — purely a display feature with no system-level PM.

---

## 2. APM Era (1992-2004)

### 2.1 Advanced Power Management (APM)

Intel and Microsoft introduced APM 1.0 in 1992 for laptops.

```
  APM Architecture
  ┌────────────────────────────┐
  │       Applications         │
  ├────────────────────────────┤
  │     APM-Aware Driver       │  ← APM BIOS calls
  ├────────────────────────────┤
  │       APM BIOS             │  ← Firmware handles PM
  │  (Real mode / 16-bit)      │
  ├────────────────────────────┤
  │       Hardware             │
  └────────────────────────────┘

  Key: BIOS controlled everything — OS had limited role
```

### 2.2 APM Power States

```
  APM Defined States:
  ┌────────────────────────────────────────────────┐
  │  Full On    → System fully operational          │
  │  APM Enable → PM is active, monitoring idle     │
  │  Standby    → Most devices off, fast resume     │
  │  Suspend    → Only RAM powered (like S3)        │
  │  Off        → Complete power down               │
  └────────────────────────────────────────────────┘
```

### 2.3 APM Versions

| Version | Year | Key Features |
|---------|------|-------------|
| APM 1.0 | 1992 | Basic power states, BIOS-driven |
| APM 1.1 | 1994 | CPU idle call, power status reporting |
| APM 1.2 | 1996 | Timer-based resume, improved events |

### 2.4 APM Limitations

```
  Problems with APM:
  ┌──────────────────────────────────────────────────┐
  │  1. BIOS-centric: OS has little control          │
  │  2. No per-device PM: all-or-nothing states      │
  │  3. Buggy BIOS implementations                   │
  │  4. 16-bit real-mode code in 32-bit OS          │
  │  5. No standard device enumeration               │
  │  6. No thermal management                        │
  │  7. Limited to x86 PC architecture               │
  │  8. No CPU frequency scaling                     │
  └──────────────────────────────────────────────────┘
```

### 2.5 APM in Linux

```c
/* Linux APM implementation (simplified — from arch/x86/kernel/apm_32.c) */
/* APM was a character device /dev/apm_bios */

static int apm_suspend(void)
{
    /* Save device states */
    device_suspend(PMSG_SUSPEND);

    /* Call BIOS to enter suspend */
    error = set_system_power_state(APM_STATE_SUSPEND);

    /* Resume path: restore devices */
    device_resume(PMSG_RESUME);
    return error;
}

/* APM BIOS call via real-mode interface */
static int apm_bios_call(struct apm_bios_call *call)
{
    /* Switch to real mode, call BIOS, return to protected mode */
    /* This was fragile and caused many bugs */
}
```

---

## 3. ACPI Era (1996-Present)

### 3.1 Advanced Configuration and Power Interface

ACPI 1.0 (1996) fundamentally changed PM by moving control from BIOS to the operating system.

```
  ┌──────────────────────────────────────────────────────────┐
  │                ACPI Architecture                          │
  │                                                           │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │              OS PM Layer (OSPM)                    │    │
  │  │  - Policy decisions live HERE                     │    │
  │  │  - OS controls transitions                        │    │
  │  └────────────────────┬─────────────────────────────┘    │
  │                       │                                   │
  │  ┌────────────────────▼─────────────────────────────┐    │
  │  │            ACPI Tables + AML Interpreter          │    │
  │  │  - DSDT, SSDT: device descriptions               │    │
  │  │  - AML bytecode: hardware abstraction             │    │
  │  │  - OS interprets AML to control hardware          │    │
  │  └────────────────────┬─────────────────────────────┘    │
  │                       │                                   │
  │  ┌────────────────────▼─────────────────────────────┐    │
  │  │           ACPI Registers + Hardware               │    │
  │  │  - PM1a/PM1b control registers                    │    │
  │  │  - GPE (General Purpose Event) registers          │    │
  │  │  - Fixed hardware interface                       │    │
  │  └──────────────────────────────────────────────────┘    │
  │                                                           │
  │  Key Shift: BIOS → OS owns power management policy       │
  └──────────────────────────────────────────────────────────┘
```

### 3.2 ACPI Global States (G-states)

```
  G-states (System-wide):
  ┌─────────────────────────────────────────────────────────┐
  │                                                          │
  │  G0 (Working)     ──► System is on and running           │
  │    │                                                     │
  │    ├── S0 (Working state)                                │
  │    │                                                     │
  │  G1 (Sleeping)    ──► System appears off but can resume  │
  │    │                                                     │
  │    ├── S1 (Power on Suspend: CPU stops, RAM refreshed)   │
  │    ├── S2 (CPU off, RAM refreshed)                       │
  │    ├── S3 (Suspend-to-RAM: most HW off, RAM powered)    │
  │    ├── S4 (Suspend-to-Disk/Hibernate: everything off)   │
  │    │                                                     │
  │  G2 (Soft Off)    ──► Minimal power, needs full boot     │
  │    │                                                     │
  │    ├── S5 (Soft Off: PSU on, wake events possible)       │
  │    │                                                     │
  │  G3 (Mechanical Off) ──► No power at all                 │
  │                                                          │
  └─────────────────────────────────────────────────────────┘
```

### 3.3 ACPI Processor States

```
  C-states (Processor Idle):
  ┌───────┬──────────────────────────────────────────────┐
  │ State │ Description                                   │
  ├───────┼──────────────────────────────────────────────┤
  │ C0    │ Active — executing instructions               │
  │ C1    │ Halt — clock stopped, instant resume          │
  │ C1E   │ Enhanced Halt — voltage reduced               │
  │ C2    │ Stop-Clock — deeper, ~100μs resume            │
  │ C3    │ Sleep — caches may be flushed, ~1ms resume    │
  │ C6    │ Deep Power Down — core voltage off            │
  │ C7    │ Package deep idle — shared caches flushed     │
  └───────┴──────────────────────────────────────────────┘

  P-states (Processor Performance at C0):
  ┌───────┬──────────────────────────────────────────────┐
  │ State │ Description                                   │
  ├───────┼──────────────────────────────────────────────┤
  │ P0    │ Maximum performance (highest freq/voltage)    │
  │ P1    │ Less than max (next step down)                │
  │ P2    │ ...                                           │
  │ Pn    │ Lowest performance (lowest freq/voltage)      │
  └───────┴──────────────────────────────────────────────┘

  T-states (Processor Throttling — deprecated):
  ┌───────┬──────────────────────────────────────────────┐
  │ State │ Description                                   │
  ├───────┼──────────────────────────────────────────────┤
  │ T0    │ No throttling (100% duty cycle)               │
  │ T1-T7 │ Increasing throttling (clock modulation)      │
  └───────┴──────────────────────────────────────────────┘
```

### 3.4 ACPI Device States (D-states)

```
  D-states (Per-device power):
  ┌───────┬──────────────────────────────────────────────┐
  │ State │ Description                                   │
  ├───────┼──────────────────────────────────────────────┤
  │ D0    │ Fully on — device operational                 │
  │ D1    │ Intermediate — device context may be lost     │
  │ D2    │ Intermediate — less power, more context lost  │
  │ D3hot │ Off but enumerable — software-visible         │
  │ D3cold│ Fully off — power removed                     │
  └───────┴──────────────────────────────────────────────┘
```

### 3.5 ACPI Versions

| Version | Year | Key Additions |
|---------|------|-------------|
| 1.0 | 1996 | Initial spec: power states, OSPM, AML |
| 2.0 | 2000 | 64-bit support, P-states, processor aggregation |
| 3.0 | 2004 | CPPC, active cooling, improved thermals |
| 4.0 | 2009 | USB/PCI PM improvements, extended states |
| 5.0 | 2011 | Hardware-reduced profile (for ARM/embedded) |
| 6.0 | 2015 | CPPC v2, GED (Generic Event Device) |
| 6.3 | 2019 | CXL support, improved platform PM |
| 6.5 | 2022 | RISC-V support, enhanced thermal |

---

## 4. Linux Power Management Evolution

### 4.1 Timeline

```
  1991 ─── Linux 0.01: No PM
  │
  1996 ─── APM support added (apm.c)
  │         First power management in Linux
  │
  2001 ─── Linux 2.4: ACPI subsystem added
  │         Initial ACPI support via Intel's interpreter
  │
  2002 ─── Linux 2.5: Device model + PM framework
  │         struct device with suspend/resume callbacks
  │
  2004 ─── Linux 2.6.9: CPUFreq subsystem stabilized
  │         Frequency governors: ondemand, conservative
  │
  2006 ─── Linux 2.6.18: Generic PM quality domain
  │         Tickless kernel (NO_HZ) — huge idle power saving
  │
  2007 ─── Linux 2.6.22: Timer migration for NO_HZ
  │
  2009 ─── Linux 2.6.32: Runtime PM framework
  │         Per-device runtime suspend/resume
  │
  2011 ─── Linux 3.0: Generic power domains (genpd)
  │         SoC power domain management
  │
  2012 ─── Linux 3.4: Common Clock Framework (CCF)
  │         Unified clock tree management
  │
  2013 ─── Linux 3.13: Intel RAPL support
  │         Running Average Power Limit
  │
  2014 ─── Linux 3.19: OPP framework rewrite
  │         Operating Performance Points
  │
  2016 ─── Linux 4.4: Full dyntick (NO_HZ_FULL)
  │         Completely tickless on isolated CPUs
  │
  2017 ─── Linux 4.12: Schedutil governor
  │         Scheduler-driven CPU frequency scaling
  │
  2018 ─── Linux 4.15: Energy-Aware Scheduling (EAS)
  │         Merged incrementally through 5.0
  │
  2020 ─── Linux 5.7: Intel EPP (Energy Performance Preference)
  │
  2021 ─── Linux 5.14: Thermal pressure support
  │         Scheduler aware of thermal throttling
  │
  2023 ─── Linux 6.3: AMD P-State EPP driver
  │         AMD Preferred Core ranking
  │
  2024 ─── Linux 6.8: Runtime PM improvements
           Queue-based resume ordering
```

### 4.2 Major Architectural Shifts

```
  Era 1: BIOS-Driven (APM)         Era 2: OS-Driven (ACPI)
  ┌─────────────────────┐         ┌─────────────────────────┐
  │  BIOS makes all     │         │  OS makes policy        │
  │  PM decisions       │         │  decisions               │
  │                     │   ──►   │                          │
  │  OS just asks BIOS  │         │  ACPI provides HW       │
  │  "please suspend"   │         │  abstraction via AML     │
  └─────────────────────┘         └─────────────────────────┘

  Era 3: Hardware-Managed (HWP/CPPC)
  ┌───────────────────────────────────┐
  │  Hardware manages fine-grained    │
  │  PM autonomously                  │
  │                                   │
  │  OS provides hints:              │
  │  - Performance preference         │
  │  - Min/max frequency bounds       │
  │  - Energy/performance bias        │
  │                                   │
  │  HW decides actual frequency     │
  │  within OS-specified bounds       │
  └───────────────────────────────────┘
```

---

## 5. Key PM Standards and Specifications

### 5.1 Standards Timeline

| Year | Standard | Key Contribution |
|------|----------|-----------------|
| 1992 | APM 1.0 | First PC PM standard |
| 1996 | ACPI 1.0 | OS-based PM, power states |
| 1998 | USB PM | Selective suspend for USB |
| 2002 | PCI PM 1.2 | D-states for PCI devices |
| 2004 | PCIe ASPM | Active State PM for PCIe links |
| 2007 | DeviceTree | ARM/embedded hardware description |
| 2011 | ACPI 5.0 | Hardware-reduced (no legacy HW needed) |
| 2013 | PSCI | ARM Power State Coordination Interface |
| 2015 | SCMI | System Control and Management Interface |

### 5.2 ARM-Specific PM Standards

```
  ARM Power Management Architecture
  ┌──────────────────────────────────────────────────┐
  │  ┌───────────────────────────────────────┐       │
  │  │     Linux Kernel (OS)                 │       │
  │  └──────────────┬────────────────────────┘       │
  │                 │ PSCI calls                      │
  │  ┌──────────────▼────────────────────────┐       │
  │  │     ARM Trusted Firmware (ATF/TF-A)   │       │
  │  │     EL3 Secure Monitor                │       │
  │  └──────────────┬────────────────────────┘       │
  │                 │ SCMI protocol                   │
  │  ┌──────────────▼────────────────────────┐       │
  │  │     System Control Processor (SCP)    │       │
  │  │     Manages: clocks, power, sensors   │       │
  │  └──────────────┬────────────────────────┘       │
  │                 │                                 │
  │  ┌──────────────▼────────────────────────┐       │
  │  │     Hardware: PMIC, PLLs, Regulators  │       │
  │  └───────────────────────────────────────┘       │
  └──────────────────────────────────────────────────┘
```

---

## 6. Evolution of CPU Frequency Scaling in Linux

### 6.1 Governor History

```
  2004: performance, powersave   ← Static governors
        │
  2006: ondemand                 ← Reactive (usage-based)
        │
  2006: conservative             ← Gradual frequency changes
        │
  2009: Interactive (Android)    ← Low-latency for mobile
        │
  2016: schedutil               ← Scheduler-integrated
        │
  2020: HWP/EPP                 ← Hardware-managed with OS hints
        │
  2023: amd-pstate-epp          ← AMD equivalent of Intel HWP
```

### 6.2 From Software to Hardware Governors

```c
/* Old approach: Software governor (ondemand) */
static void ondemand_timer(struct cpufreq_policy *policy)
{
    unsigned int load = get_cpu_load();
    if (load > up_threshold)
        cpufreq_set_frequency(policy, policy->max);
    else
        freq = policy->min + load * (policy->max - policy->min) / 100;
}

/* Modern approach: schedutil — scheduler-driven */
static void sugov_update(struct update_util_data *hook,
                          u64 time, unsigned int flags)
{
    /* Frequency based on scheduler utilization signal */
    unsigned long util = cpu_util_cfs(cpu);
    unsigned long max  = arch_scale_cpu_capacity(cpu);
    unsigned int freq  = map_util_freq(util, max, policy->cpuinfo.max_freq);
    /* Direct from scheduler — no sampling delay */
}

/* Latest: Hardware-managed with OS hints */
/* OS sets min/max freq and energy_perf_preference */
wrmsrl(MSR_HWP_REQUEST,
       HWP_MIN_PERF(min) | HWP_MAX_PERF(max) |
       HWP_ENERGY_PERF_PREF(epp));
/* Hardware decides actual frequency autonomously */
```

---

## 7. The Tickless Revolution

### 7.1 Before NO_HZ (Periodic Timer)

```
  Traditional kernel: Timer interrupt every 1ms (HZ=1000)

  Time ─────────────────────────────────────────────►
  CPU:  ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑ ↑
        Timer interrupts wake CPU even when idle!
        
  Result: CPU can NEVER stay in deep idle state
          because timer wakes it up every 1ms
```

### 7.2 After NO_HZ (Dynamic Ticks)

```
  Tickless kernel: Timer fires only when needed

  Time ─────────────────────────────────────────────►
  CPU:  ↑ ↑ ↑           ↑ ↑ ↑ ↑           ↑
        active  ─ idle ─  active  ─ idle ─  active

  Result: CPU can enter deep C-states for longer
          Saves significant power on idle systems
```

### 7.3 NO_HZ Modes

| Mode | Config | Behavior |
|------|--------|----------|
| `CONFIG_HZ_PERIODIC` | Legacy | Periodic ticks always |
| `CONFIG_NO_HZ_IDLE` | Default | Stop ticks when CPU is idle |
| `CONFIG_NO_HZ_FULL` | Aggressive | Stop ticks even when 1 task runs |

---

## 8. Modern PM: Hardware Takes Control

### 8.1 Intel Speed Shift (HWP)

Starting with Skylake (2015), Intel hardware can autonomously manage CPU frequency:

```
  Traditional DVFS                Intel HWP
  ┌─────────────────┐           ┌─────────────────┐
  │  OS samples     │           │  OS provides:    │
  │  CPU load       │           │  - Min/Max freq  │
  │  every 10ms     │           │  - EPP hint      │
  │       │         │           │  - Desired perf  │
  │  OS decides     │           │       │          │
  │  new frequency  │           │  HW autonomously │
  │       │         │           │  adjusts freq    │
  │  OS programs    │           │  every ~1ms or   │
  │  MSR register   │           │  faster          │
  │       │         │           │                  │
  │  Latency: ~10ms │           │  Latency: ~1ms   │
  └─────────────────┘           └─────────────────┘
```

### 8.2 ARM SCMI

```
  ARM System Control and Management Interface
  ┌─────────────────────────────────────────────────┐
  │  OS (Linux) ←→ Mailbox ←→ SCP Firmware          │
  │                                                  │
  │  Protocols:                                      │
  │  - Clock: frequency management                   │
  │  - Power Domain: on/off/retention                │
  │  - Performance: DVFS                             │
  │  - Sensor: temperature, voltage, current         │
  │  - Reset: domain reset control                   │
  │                                                  │
  │  Transport: shared memory + doorbell interrupt    │
  └─────────────────────────────────────────────────┘
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `arch/x86/kernel/apm_32.c` | Legacy APM driver (historical) |
| `drivers/acpi/` | ACPI subsystem |
| `drivers/acpi/sleep.c` | ACPI sleep state management |
| `drivers/acpi/processor_*.c` | ACPI processor PM (C/P-states) |
| `drivers/cpufreq/intel_pstate.c` | Intel P-state/HWP driver |
| `drivers/cpufreq/amd-pstate.c` | AMD P-state driver |
| `kernel/time/tick-sched.c` | NO_HZ implementation |
| `drivers/firmware/arm_scmi/` | ARM SCMI driver |
| `drivers/firmware/psci/` | ARM PSCI implementation |

---

## Interview Questions

**Q1: What were the main problems with APM?**
**A:** APM was BIOS-centric — the OS had minimal control over PM decisions. Problems included: (1) 16-bit real-mode BIOS calls from a 32-bit OS, (2) no per-device PM — only all-or-nothing system states, (3) buggy BIOS implementations across vendors, (4) no CPU frequency scaling, (5) no thermal management, (6) x86-only — not portable to other architectures.

**Q2: How does ACPI differ from APM architecturally?**
**A:** ACPI moves PM policy from BIOS to the OS (OSPM — OS-directed PM). The BIOS provides hardware description via ACPI tables (DSDT/SSDT) containing AML bytecode. The OS interprets AML to discover and control hardware PM capabilities. This gives the OS full control over when and how to change power states, enabling per-device PM and fine-grained CPU scaling.

**Q3: Explain C-states, P-states, and D-states.**
**A:** C-states are CPU idle states (C0=active, C1-Cn=progressively deeper idle with more power savings but higher wake latency). P-states are CPU performance states at C0 — different voltage/frequency pairs (P0=max, Pn=min). D-states are device power states (D0=fully on to D3cold=power removed). They work independently: a CPU in C0 can be at any P-state, and each device can be in its own D-state.

**Q4: Why was the tickless kernel important for power management?**
**A:** Before NO_HZ, the kernel generated timer interrupts every 1ms (HZ=1000), waking the CPU from idle. With tickless (NO_HZ_IDLE), timer ticks stop when the CPU is idle, allowing it to enter deep C-states for extended periods. This dramatically reduced idle power consumption. NO_HZ_FULL goes further, stopping ticks even when one task is running.

**Q5: What is Intel HWP and why does it obsolete software governors?**
**A:** Intel Hardware P-states (HWP/Speed Shift) lets the CPU hardware autonomously manage frequency with ~1ms latency, compared to ~10ms for software governors. The OS provides hints (min/max freq, energy/performance preference), but hardware makes the actual frequency decisions. This is faster and more responsive because hardware sees microarchitectural signals (cache misses, pipeline stalls) invisible to software.

---

## Summary

- APM (1992) was BIOS-driven with limited OS control and many bugs
- ACPI (1996) shifted PM policy to OS via OSPM and AML bytecode
- Linux PM evolved from APM support (1996) to modern frameworks (2020+)
- Key Linux milestones: device model PM (2.5), CPUFreq (2.6.9), runtime PM (2.6.32), EAS (4.15+)
- Tickless kernel (NO_HZ) eliminated idle timer interrupts for deep sleep states
- Modern trend: Hardware-managed PM (Intel HWP, ARM SCMI) with OS providing policy hints
- ACPI states: G-states (global), S-states (sleep), C-states (CPU idle), P-states (CPU perf), D-states (device)

---

[Previous Chapter: Foundations ←](Chapter_01_Foundations.md) | [Next Chapter: Hardware Architecture →](Chapter_03_Hardware_Architecture.md)
