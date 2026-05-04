# Chapter 4: Power States in Systems

## Learning Goals
- Understand all ACPI-defined power states (G, S, C, P, D states)
- Learn processor idle states and their hardware implementation
- Understand device power states and transitions
- Know the trade-offs between power savings and wake-up latency
- Learn how Linux maps hardware states to kernel abstractions

---

## 1. System-Wide Power States (G-States / S-States)

### 1.1 Global States

```
  ACPI Global Power States
  ┌───────────────────────────────────────────────────────────┐
  │                                                            │
  │  G0 (S0) — Working                                        │
  │  │  System fully operational                               │
  │  │  CPU executing, all devices available                   │
  │  │                                                         │
  │  │  ┌─ S0ix (Modern Standby / Connected Standby) ──────┐ │
  │  │  │  System appears off but can wake instantly         │ │
  │  │  │  Network connected, notifications received        │ │
  │  │  │  Used in phones, tablets, modern laptops          │ │
  │  │  └──────────────────────────────────────────────────┘ │
  │  │                                                         │
  │  G1 — Sleeping                                             │
  │  │  ├── S1: Power-on Standby                              │
  │  │  │   CPU stops, caches maintained, ~2s resume          │
  │  │  │   (Rarely implemented in modern HW)                 │
  │  │  │                                                     │
  │  │  ├── S2: CPU Off                                       │
  │  │  │   CPU powered off, caches lost                      │
  │  │  │   (Rarely implemented)                              │
  │  │  │                                                     │
  │  │  ├── S3: Suspend-to-RAM (STR)                          │
  │  │  │   Only RAM powered (+ wake logic)                   │
  │  │  │   ~3-5W, resume ~2-5 seconds                       │
  │  │  │   Most common sleep state                           │
  │  │  │                                                     │
  │  │  └── S4: Suspend-to-Disk (Hibernate)                   │
  │  │      RAM image saved to disk, all power off            │
  │  │      ~0W, resume ~10-30 seconds                        │
  │  │      Survives power failure                            │
  │  │                                                         │
  │  G2 (S5) — Soft Off                                        │
  │  │  PSU on (5V standby), wake events possible             │
  │  │  Wake-on-LAN, power button, RTC alarm                  │
  │  │  Full boot required                                    │
  │  │                                                         │
  │  G3 — Mechanical Off                                       │
  │     No power at all, power cord removed                    │
  │     Only power button can restart                          │
  └───────────────────────────────────────────────────────────┘
```

### 1.2 Linux System Sleep States

```c
/* kernel/power/suspend.c — Linux sleep state definitions */
/* Exposed via /sys/power/state */

/* Available states (read from /sys/power/state):
 * "freeze"   — Suspend-to-Idle (s2idle): software-only
 * "standby"  — Power-On Standby (S1)
 * "mem"      — Suspend-to-RAM (S3) or s2idle
 * "disk"     — Hibernate (S4)
 */

/* /sys/power/mem_sleep controls what "mem" maps to:
 * "s2idle"   — suspend-to-idle
 * "shallow"  — standby (S1)
 * "deep"     — suspend-to-RAM (S3)
 */
```

### 1.3 Comparison of Sleep States

```
  ┌────────────┬──────────┬──────────┬───────────┬──────────┐
  │ State      │ Power    │ Resume   │ RAM       │ Context  │
  │            │ Draw     │ Time     │ Powered   │ Saved    │
  ├────────────┼──────────┼──────────┼───────────┼──────────┤
  │ S0ix       │ ~100mW   │ <1s      │ Yes       │ In RAM   │
  │ S1/Standby │ ~1-3W    │ ~2s      │ Yes       │ In RAM   │
  │ S3/STR     │ ~1-5W    │ ~3-5s    │ Yes       │ In RAM   │
  │ S4/Disk    │ ~0W      │ ~10-30s  │ No        │ On disk  │
  │ S5/Soft Off│ ~0.5W    │ Full boot│ No        │ None     │
  │ G3/Mech Off│ 0W       │ Full boot│ No        │ None     │
  └────────────┴──────────┴──────────┴───────────┴──────────┘
```

---

## 2. CPU Idle States (C-States)

### 2.1 C-State Hierarchy

```
  CPU Idle State Hierarchy
  ┌────────────────────────────────────────────────────────────┐
  │                                                             │
  │  C0 ── Active (executing instructions)                     │
  │  │                                                          │
  │  C1 ── Halt (core clock stopped)                           │
  │  │     Latency: <1μs    Power: ~30% of C0                 │
  │  │     HW: clock gating  Context: fully preserved          │
  │  │                                                          │
  │  C1E ── Enhanced Halt (clock stop + voltage reduce)        │
  │  │      Latency: ~10μs   Power: ~20% of C0                │
  │  │                                                          │
  │  C3 ── Sleep (deeper, caches may flush)                    │
  │  │     Latency: ~100μs   Power: ~10% of C0                │
  │  │     HW: L1 may be lost  Bus snoops may stop            │
  │  │                                                          │
  │  C6 ── Deep Power Down (core power gated)                  │
  │  │     Latency: ~200μs   Power: ~2% of C0                 │
  │  │     HW: VDD_core=0, state in retention SRAM            │
  │  │                                                          │
  │  C7 ── Package Deep (all cores C6 + LLC flush)             │
  │  │     Latency: ~1ms     Power: <1% of C0                 │
  │  │     HW: Package PLL off, L3 flushed                     │
  │  │                                                          │
  │  C8-C10 ── Platform idle (additional Intel states)         │
  │       Display off, PCIe in L1, USB suspended               │
  │       Essentially S0ix broken into sub-states              │
  └────────────────────────────────────────────────────────────┘
```

### 2.2 C-State Cost-Benefit Analysis

```
  ┌─────────────────────────────────────────────────────────┐
  │  Deeper C-states save more power BUT:                   │
  │                                                          │
  │  Power     │                                            │
  │  Savings   │    ████                                    │
  │  (%)       │    ████ ████                               │
  │            │    ████ ████ ████                           │
  │   100% ────│────████─████─████─████──                   │
  │            │    ████ ████ ████ ████                      │
  │            │    ████ ████ ████ ████                      │
  │            └────────────────────────── C-state           │
  │                 C1    C3   C6   C7                       │
  │                                                          │
  │  Wakeup    │                       ████                 │
  │  Latency   │                  ████ ████                 │
  │  (μs)      │             ████ ████ ████                 │
  │            │        ████ ████ ████ ████                 │
  │            └────────────────────────── C-state           │
  │                 C1    C3   C6   C7                       │
  │                 1μs  100μs 200μs 1ms                    │
  │                                                          │
  │  Break-even time: minimum idle duration where            │
  │  entering deeper state saves total energy                │
  │  (entry_energy + exit_energy < idle_savings × duration)  │
  └─────────────────────────────────────────────────────────┘
```

### 2.3 Break-Even Time Calculation

```
  Break-even time for C-state entry:

  Energy_saved = (P_shallow - P_deep) × t_idle
  Energy_cost  = E_entry + E_exit

  Break-even when: Energy_saved = Energy_cost

  t_break_even = (E_entry + E_exit) / (P_shallow - P_deep)

  Example for C6:
  - Entry energy: 50 μJ
  - Exit energy:  30 μJ
  - C1 power:     500 mW
  - C6 power:      20 mW

  t_break_even = (50 + 30) μJ / (500 - 20) mW
               = 80 μJ / 480 mW
               = 167 μs

  → Only enter C6 if expected idle time > 167μs
  → This is what the cpuidle governor calculates
```

---

## 3. CPU Performance States (P-States)

### 3.1 P-State Architecture

```
  P-States: Voltage/Frequency Pairs at C0
  ┌──────────────────────────────────────────────────────┐
  │                                                       │
  │  P0 ── Maximum performance                           │
  │  │     Freq: 3.5 GHz  Voltage: 1.2V  Power: 100%   │
  │  │                                                    │
  │  P1 ── High performance                              │
  │  │     Freq: 3.0 GHz  Voltage: 1.1V  Power: 70%    │
  │  │                                                    │
  │  P2 ── Medium performance                            │
  │  │     Freq: 2.5 GHz  Voltage: 1.0V  Power: 50%    │
  │  │                                                    │
  │  P3 ── Low performance                               │
  │  │     Freq: 2.0 GHz  Voltage: 0.9V  Power: 35%    │
  │  │                                                    │
  │  Pn ── Minimum performance                           │
  │        Freq: 0.8 GHz  Voltage: 0.7V  Power: 12%    │
  │                                                       │
  │  Note: P = αCV²f, so power drops steeply:           │
  │  P(1.2V, 3.5G) / P(0.7V, 0.8G) ≈ 10x              │
  └──────────────────────────────────────────────────────┘
```

### 3.2 OPP Table Example

```c
/* Operating Performance Points in device tree */
/* Defines valid voltage/frequency pairs */

cpu0_opp_table: opp-table {
    compatible = "operating-points-v2";

    opp-500000000 {
        opp-hz = /bits/ 64 <500000000>;   /* 500 MHz */
        opp-microvolt = <700000>;          /* 0.7V */
    };
    opp-1000000000 {
        opp-hz = /bits/ 64 <1000000000>;  /* 1.0 GHz */
        opp-microvolt = <850000>;          /* 0.85V */
    };
    opp-1500000000 {
        opp-hz = /bits/ 64 <1500000000>;  /* 1.5 GHz */
        opp-microvolt = <1000000>;         /* 1.0V */
    };
    opp-2000000000 {
        opp-hz = /bits/ 64 <2000000000>;  /* 2.0 GHz */
        opp-microvolt = <1150000>;         /* 1.15V */
        opp-suspend;                       /* Use during suspend */
    };
};
```

---

## 4. Device Power States (D-States)

### 4.1 D-State Definitions

```
  Device Power States
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  D0 — Fully On                                           │
  │  │   Device fully operational                             │
  │  │   All features accessible                              │
  │  │   Maximum power consumption                            │
  │  │                                                        │
  │  D1 — Light Sleep                                        │
  │  │   Reduced functionality                                │
  │  │   Partial context preserved                            │
  │  │   Device-class specific definition                     │
  │  │   Example: Network card — no TX, RX wake only         │
  │  │                                                        │
  │  D2 — Deep Sleep                                         │
  │  │   Less functionality than D1                           │
  │  │   Less context preserved                               │
  │  │   Lower power than D1                                  │
  │  │                                                        │
  │  D3hot — Off (bus-accessible)                            │
  │  │   Device functionally off                              │
  │  │   Power still supplied (Vcc present)                   │
  │  │   Bus enumerable (PCI config readable)                 │
  │  │   Full context lost, device must be reinitialized      │
  │  │                                                        │
  │  D3cold — Off (power removed)                            │
  │      Physically powered off (Vcc removed)                 │
  │      Not enumerable on bus                                │
  │      Minimum possible power (~0)                          │
  │      Requires full re-enumeration on power-up             │
  └──────────────────────────────────────────────────────────┘
```

### 4.2 D-State Transitions

```
  D-State Transition Rules
  ┌──────────────────────────────────────────────┐
  │                                               │
  │  D0 ◄───────────────────────────► D1         │
  │  D0 ◄───────────────────────────► D2         │
  │  D0 ◄───────────────────────────► D3hot      │
  │  D0 ◄───────────────────────────► D3cold     │
  │                                               │
  │  Direct transitions ONLY to/from D0!          │
  │  Cannot go D1→D2 or D2→D3 directly.          │
  │  Must always return to D0 first.              │
  │                                               │
  │  Exception: D3hot → D3cold (remove power)     │
  │             D3cold → D0 (supply power + init) │
  └──────────────────────────────────────────────┘
```

### 4.3 Linux Device PM States

```c
/* include/linux/pm.h — PM messages and states */

typedef struct pm_message {
    int event;
} pm_message_t;

/* PM event types */
#define PM_EVENT_ON           0x0000  /* D0 */
#define PM_EVENT_FREEZE       0x0001  /* For hibernate snapshot */
#define PM_EVENT_SUSPEND      0x0002  /* Going to sleep */
#define PM_EVENT_HIBERNATE    0x0004  /* Going to disk */
#define PM_EVENT_RESUME       0x0010  /* Waking from sleep */
#define PM_EVENT_THAW         0x0020  /* Thawing from hibernate */
#define PM_EVENT_RESTORE      0x0040  /* Restoring from hibernate */

/* Convenience message constants */
#define PMSG_ON        ((struct pm_message){ .event = PM_EVENT_ON, })
#define PMSG_SUSPEND   ((struct pm_message){ .event = PM_EVENT_SUSPEND, })
#define PMSG_RESUME    ((struct pm_message){ .event = PM_EVENT_RESUME, })
```

---

## 5. PCI Power Management

### 5.1 PCI PM States

```
  PCI Device Power States
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  PCI PM Capabilities Register enables D-state control    │
  │                                                           │
  │  D0 ── Full power, config space accessible               │
  │  │                                                        │
  │  D1 ── Optional intermediate state                       │
  │  │     Clock may be stopped                               │
  │  │     Config space accessible                            │
  │  │                                                        │
  │  D2 ── Optional deeper state                             │
  │  │     Internal clocks stopped                            │
  │  │     Config space accessible                            │
  │  │                                                        │
  │  D3hot ── Software off                                   │
  │  │     PME# can be asserted (wake event)                  │
  │  │     Config space partially accessible                  │
  │  │     Vcc still present                                  │
  │  │                                                        │
  │  D3cold ── Hardware off                                  │
  │      Vcc removed, no bus access                           │
  │      Main power rail turned off                           │
  │                                                           │
  │  PCIe ASPM (Active State PM):                            │
  │  ├── L0  — Active (full bandwidth)                       │
  │  ├── L0s — Standby (fast exit ~1μs, 50% BW savings)     │
  │  ├── L1  — Low power (~10μs exit, ~90% savings)          │
  │  ├── L1.1 — Deeper (PCI-PM L1, ~30μs exit)             │
  │  └── L1.2 — Deepest (clkref off, ~100μs exit)          │
  └──────────────────────────────────────────────────────────┘
```

### 5.2 PCI PM in Linux

```c
/* PCI power state management */
#include <linux/pci.h>

/* Setting PCI device power state */
int pci_set_power_state(struct pci_dev *dev, pci_power_t state)
{
    /* state: PCI_D0, PCI_D1, PCI_D2, PCI_D3hot, PCI_D3cold */
    /* Checks PM capability, programs PMCSR register */
    /* Handles delays required by PCI spec */
}

/* PCI driver PM operations */
static const struct dev_pm_ops my_pci_pm_ops = {
    .suspend  = my_pci_suspend,
    .resume   = my_pci_resume,
    .runtime_suspend = my_pci_runtime_suspend,
    .runtime_resume  = my_pci_runtime_resume,
};

static int my_pci_suspend(struct device *dev)
{
    struct pci_dev *pdev = to_pci_dev(dev);

    /* Save device-specific state */
    my_save_state(pdev);

    /* Save PCI config space */
    pci_save_state(pdev);

    /* Enable wake if supported */
    pci_enable_wake(pdev, PCI_D3hot, true);

    /* Move to D3hot */
    pci_set_power_state(pdev, PCI_D3hot);

    return 0;
}
```

---

## 6. USB Power States

### 6.1 USB Suspend

```
  USB Power Management States
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Active ── Normal operation, transfers ongoing            │
  │  │                                                        │
  │  Idle ── No transfers for 3ms (auto-suspend eligible)    │
  │  │                                                        │
  │  Suspended ── Device in low-power mode                   │
  │  │   - Host stops SOF/micro-frame packets                │
  │  │   - Device draws ≤2.5mA (USB 2.0)                    │
  │  │   - wake via remote wakeup signaling                  │
  │  │                                                        │
  │  USB 3.x Link States:                                    │
  │  ├── U0 — Active                                         │
  │  ├── U1 — Standby (fast exit, ~1μs)                     │
  │  ├── U2 — Deeper standby (slower exit, ~100μs)          │
  │  └── U3 — Suspended (link off, ~10ms exit)              │
  │                                                           │
  │  Linux USB Autosuspend:                                  │
  │  /sys/bus/usb/devices/.../power/autosuspend = 2 (secs)  │
  │  /sys/bus/usb/devices/.../power/control = "auto"         │
  └──────────────────────────────────────────────────────────┘
```

---

## 7. Linux Power State Sysfs Interface

### 7.1 System PM

```
  /sys/power/
  ├── state                    # Available sleep states
  │   $ cat /sys/power/state
  │   freeze mem disk
  │
  ├── mem_sleep               # What "mem" means
  │   $ cat /sys/power/mem_sleep
  │   s2idle [deep]           # deep = S3, brackets = selected
  │
  ├── disk                    # Hibernate mode
  │   $ cat /sys/power/disk
  │   [platform] shutdown reboot suspend test_resume
  │
  ├── pm_async                # Async suspend/resume
  ├── pm_test                 # Testing modes
  └── wakeup_count            # Wakeup event counter
```

### 7.2 Device PM

```
  /sys/devices/.../power/
  ├── runtime_status          # "active", "suspended", "resuming"
  ├── runtime_active_time     # Total active time (ms)
  ├── runtime_suspended_time  # Total suspended time (ms)
  ├── control                 # "auto" or "on" (forbid suspend)
  ├── autosuspend_delay_ms    # Delay before auto-suspend
  ├── wakeup                  # "enabled" or "disabled"
  └── wakeup_count            # Number of wakeup events
```

### 7.3 CPU Idle States

```
  /sys/devices/system/cpu/cpu0/cpuidle/
  ├── state0/
  │   ├── name               # "POLL"
  │   ├── latency            # 0 (μs)
  │   ├── power              # -1 (unknown)
  │   ├── time               # Total time in state (μs)
  │   ├── usage              # Number of entries
  │   └── disable            # 0 or 1
  ├── state1/
  │   ├── name               # "C1"
  │   ├── latency            # 2
  │   └── ...
  ├── state2/
  │   ├── name               # "C1E"
  │   ├── latency            # 10
  │   └── ...
  └── state3/
      ├── name               # "C6"
      ├── latency            # 200
      └── ...

  Disable a deep C-state:
  $ echo 1 > /sys/devices/system/cpu/cpu0/cpuidle/state3/disable
```

---

## 8. State Transition Diagrams

### 8.1 Complete State Hierarchy

```
  System State Machine
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │                    ┌──────────┐                           │
  │            ┌──────►│  G0/S0   │◄──────┐                  │
  │            │       │ Working  │       │                  │
  │            │       └────┬─────┘       │                  │
  │            │            │             │                  │
  │            │    ┌───────┴───────┐     │                  │
  │            │    │     C0/P0-Pn  │     │                  │
  │            │    │ CPU Active    │     │                  │
  │            │    │               │     │                  │
  │            │    │  ┌──► C1 ──┐  │     │                  │
  │            │    │  ├──► C3 ──┤  │     │                  │
  │            │    │  ├──► C6 ──┤  │     │                  │
  │            │    │  └──► C7 ──┘  │     │                  │
  │            │    │  CPU Idle     │     │                  │
  │            │    └───────────────┘     │                  │
  │            │                          │                  │
  │       ┌────┴─────┐             ┌──────┴──────┐          │
  │       │  G1/S3   │             │   G1/S4     │          │
  │       │  STR     │             │   Hibernate │          │
  │       └──────────┘             └─────────────┘          │
  │                                                           │
  │  Devices independently transition between D0-D3:         │
  │  Each device has its own state machine                    │
  │  Runtime PM manages per-device transitions                │
  └──────────────────────────────────────────────────────────┘
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `kernel/power/suspend.c` | System suspend state machine |
| `kernel/power/hibernate.c` | Hibernate implementation |
| `kernel/power/main.c` | PM core, sysfs interface |
| `drivers/cpuidle/` | CPU idle state framework |
| `drivers/cpuidle/governors/` | C-state selection algorithms |
| `drivers/pci/pci.c` | PCI PM (pci_set_power_state) |
| `drivers/usb/core/driver.c` | USB autosuspend |
| `include/linux/pm.h` | PM state definitions |
| `include/linux/pci.h` | PCI D-state definitions |
| `arch/x86/kernel/acpi/cstate.c` | x86 C-state implementation |

---

## Interview Questions

**Q1: Explain the difference between S3 (Suspend-to-RAM) and S4 (Hibernate).**
**A:** S3 keeps RAM powered while turning off everything else — CPU, devices, displays. System state is preserved in DRAM and resume takes ~3-5 seconds. S3 still consumes ~1-5W. S4 writes the entire RAM image to disk, then powers off everything including RAM (~0W). Resume requires reading the image back from disk (~10-30s). S4 survives power failure; S3 doesn't. Linux implements S3 via `suspend.c` and S4 via `hibernate.c`.

**Q2: What is the break-even time for C-states?**
**A:** Break-even time is the minimum idle duration where entering a deeper C-state saves more energy than it costs. It's calculated as: t = (E_entry + E_exit) / (P_shallow - P_deep). If the CPU expects to be idle for less than this time, it should stay in a shallower state because the transition energy overhead exceeds the savings. The CPUIdle governor predicts idle duration and compares against each state's break-even time.

**Q3: What is S0ix / Modern Standby?**
**A:** S0ix (or Modern Standby / Connected Standby) is an ultra-low-power state within G0 where the system appears off but remains connected to networks. It requires all devices to support aggressive runtime PM, CPU C10+ states, and PCIe L1.2. The system wakes periodically to process notifications, sync email, etc. Power consumption is ~100mW vs ~3W for S3. It replaces S3 on many modern laptops/tablets.

**Q4: Explain PCI device power states and ASPM.**
**A:** PCI defines D0 (on), D1/D2 (intermediate), D3hot (off but powered), D3cold (power removed). ASPM (Active State PM) manages the PCIe link: L0 (active), L0s (standby, ~1μs exit), L1 (low power, ~10μs exit), L1.1/L1.2 (deeper, PLL/refclk off). ASPM operates transparently — the link enters low-power when idle and exits on traffic. In Linux, ASPM policy is set via `/sys/module/pcie_aspm/parameters/policy`.

**Q5: How does Linux expose power state information to userspace?**
**A:** Through sysfs: `/sys/power/state` lists available system sleep states. `/sys/power/mem_sleep` shows the suspend mode. Per-device PM is at `/sys/devices/.../power/` with runtime_status, control (auto/on), wakeup enable. CPU idle states are at `/sys/devices/system/cpu/cpuN/cpuidle/stateN/` with latency, usage, time, and disable controls. CPU frequency info is at `/sys/devices/system/cpu/cpuN/cpufreq/`.

---

## Summary

- ACPI defines G-states (global/system), S-states (sleep), C-states (CPU idle), P-states (CPU performance), D-states (device)
- S3 (suspend-to-RAM) keeps RAM powered; S4 (hibernate) writes to disk; S0ix is modern low-power standby
- C-states range from C1 (clock halt, ~1μs) to C7+ (package deep power down, ~1ms exit)
- P-states are voltage/frequency pairs at C0 — higher P-number means lower performance and power
- D-states: D0 (on) through D3cold (power removed); transitions always through D0
- Break-even time determines whether entering a deeper state is worthwhile
- PCI ASPM transparently manages link power (L0, L0s, L1, L1.1, L1.2)
- Linux exposes all PM states via sysfs for monitoring and control

---

[Previous Chapter: Hardware Architecture ←](Chapter_03_Hardware_Architecture.md) | [Next Chapter: Linux PM Architecture →](Chapter_05_PM_Architecture.md)
