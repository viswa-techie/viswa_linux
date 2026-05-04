# Chapter 18: Embedded System Power Management

## Learning Goals
- Understand power management challenges unique to embedded/mobile systems
- Learn battery-powered system optimization strategies
- Know SoC-level power management (ARM big.LITTLE, power islands)
- Understand peripheral power gating and always-on domains
- Learn automotive and IoT power management patterns

---

## 1. Embedded PM Landscape

```
  Embedded Power Budget
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Mobile Phone (~3500 mAh @ 3.7V = ~13 Wh)              │
  │  ┌──────────────────────────────────────┐                │
  │  │ Component         │ Active │ Sleep   │                │
  │  ├───────────────────┼────────┼─────────┤                │
  │  │ AP (SoC)          │ 2-5W   │ 5-20mW  │                │
  │  │ Display           │ 1-3W   │ 0mW     │                │
  │  │ Modem (cellular)  │ 1-3W   │ 10-50mW │                │
  │  │ WiFi              │ 0.5-1W │ 2-5mW   │                │
  │  │ Audio             │ 0.1-0.5W│ 0mW    │                │
  │  │ Sensors           │ 1-10mW │ <1mW    │                │
  │  │ DRAM              │ 0.5-1W │ 10-50mW │                │
  │  ├───────────────────┼────────┼─────────┤                │
  │  │ Total             │ 5-13W  │ 30-130mW│                │
  │  └──────────────────────────────────────┘                │
  │                                                           │
  │  Target: 24-48 hour standby                              │
  │  Key: Minimize time in active state                      │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. SoC Power Architecture

```
  Typical ARM SoC Power Domains
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌── Always-On Domain ──────────────────────────┐        │
  │  │  Power Management IC (PMIC) controller       │        │
  │  │  Real-Time Clock (RTC)                       │        │
  │  │  Wakeup interrupt controller                 │        │
  │  │  SRAM (retention memory)                     │        │
  │  │  Watchdog timer                              │        │
  │  │  Power: ~1-5 mW (always consuming)           │        │
  │  └──────────────────────────────────────────────┘        │
  │                                                           │
  │  ┌── CPU Domain ─────────┐  ┌── GPU Domain ──────┐      │
  │  │  CPU cluster 0        │  │  GPU cores          │      │
  │  │  CPU cluster 1        │  │  GPU L2 cache       │      │
  │  │  L3 cache             │  │  Video decoder      │      │
  │  │  Per-CPU L1/L2 cache  │  │                     │      │
  │  │  Can power gate       │  │  Can power gate     │      │
  │  │  individually         │  │  when GPU idle      │      │
  │  └──────────────────────┘  └──────────────────────┘      │
  │                                                           │
  │  ┌── Peripheral Domain ──┐  ┌── Modem Domain ────┐      │
  │  │  USB controller       │  │  Baseband processor │      │
  │  │  UART, SPI, I2C       │  │  RF transceiver     │      │
  │  │  Display controller   │  │  Independent power  │      │
  │  │  Camera ISP           │  │  (own firmware)     │      │
  │  │  Audio DSP            │  │                     │      │
  │  └──────────────────────┘  └──────────────────────┘      │
  │                                                           │
  │  ┌── Memory Domain ──────────────────────────────┐       │
  │  │  DDR controller + PHY                          │       │
  │  │  Self-refresh mode during suspend              │       │
  │  │  Power: Active ~1W, Self-refresh ~10-50mW     │       │
  │  └────────────────────────────────────────────────┘       │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Suspend-to-Idle (s2idle) for Embedded

```c
/*
 * s2idle is preferred over S3 on modern ARM SoCs because:
 * - No ATF/firmware handoff overhead
 * - Faster resume (~100ms vs ~500ms)
 * - PSCI CPU_SUSPEND achieves similar power savings
 * - Peripherals can selectively stay powered
 */

/* s2idle flow on ARM SoC */
/*
 * 1. Freeze all userspace processes
 * 2. Suspend all devices (drivers save state)
 * 3. Disable all non-wakeup IRQs
 * 4. Each CPU enters deepest idle state:
 *    - psci_cpu_suspend() → ARM Trusted Firmware
 *    - CPU power gated, L1/L2 flushed
 *    - Only GIC wakeup IRQs can exit
 * 5. DDR enters self-refresh
 * 6. System waits in low-power (~10-30mW)
 *
 * Wakeup:
 * 7. GIC wakeup IRQ fires
 * 8. CPU powers on, resumes from reset vector
 * 9. ATF restores CPU state
 * 10. Linux resumes devices, thaw processes
 */
```

---

## 4. Peripheral Power Optimization

```c
/* Aggressive peripheral PM for embedded */

/* Strategy 1: Clock gating unused peripherals */
static int peripheral_runtime_suspend(struct device *dev)
{
    struct my_periph *p = dev_get_drvdata(dev);
    clk_disable_unprepare(p->bus_clk);
    clk_disable_unprepare(p->func_clk);
    return 0;
    /* Bus clock off → register access blocked
     * Func clock off → no operation
     * Power: ~0 (leakage only) */
}

/* Strategy 2: Power gating entire blocks */
static int block_runtime_suspend(struct device *dev)
{
    struct my_block *b = dev_get_drvdata(dev);
    clk_disable_unprepare(b->clk);
    regulator_disable(b->vdd);
    /* Block loses all register state
     * Must save and restore in suspend/resume */
    return 0;
}

/* Strategy 3: Reducing voltage (voltage scaling) */
/* Done via OPP/DVFS — reduce voltage when workload is light */

/* Strategy 4: Memory retention mode */
/*
 * Some SoC blocks have retention power:
 * - Core power off (logic state lost)
 * - Retention power on (SRAM state preserved)
 * - Saves restore time (no register reprogramming)
 * - Uses ~10% of full power
 */
```

---

## 5. Battery and Charging Integration

```
  Battery Power Path
  ┌──────────────────────────────────────────────────┐
  │                                                   │
  │  USB/Wireless ──► Charger IC ──► Battery         │
  │                        │                          │
  │                        ├──► System Power Bus      │
  │                        │    (VSYS)                │
  │                        │                          │
  │                   ┌────▼────┐                     │
  │                   │  PMIC   │                     │
  │                   ├─────────┤                     │
  │                   │ BUCK1 ──├──► CPU (VDD_CORE)   │
  │                   │ BUCK2 ──├──► DDR (VDD_MEM)    │
  │                   │ BUCK3 ──├──► GPU (VDD_GPU)    │
  │                   │ LDO1  ──├──► PLL (VDD_PLL)    │
  │                   │ LDO2  ──├──► I/O (VDD_IO)     │
  │                   └─────────┘                     │
  │                                                   │
  │  Linux interfaces:                               │
  │  /sys/class/power_supply/battery/                │
  │    capacity  → Battery SOC (0-100%)              │
  │    status    → Charging/Discharging/Full         │
  │    voltage_now → Current voltage (μV)            │
  │    current_now → Current draw (μA)               │
  └──────────────────────────────────────────────────┘
```

---

## 6. Automotive Power Management

```
  Automotive Power States (AAOS)
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌─ FULL ON ─────────────────────────────────────┐       │
  │  │  Vehicle running, all systems active           │       │
  │  │  IVI display on, cameras active                │       │
  │  │  Power: 10-30W for head unit                   │       │
  │  └──────────┬────────────────────────────────────┘       │
  │             │ Ignition off / display off                  │
  │  ┌──────────▼────────────────────────────────────┐       │
  │  │  GARAGE MODE                                   │       │
  │  │  Display off, CPU still active                │       │
  │  │  OTA updates, deferred tasks                  │       │
  │  │  Power: 5-15W                                 │       │
  │  │  Time-limited (e.g., 30 min)                  │       │
  │  └──────────┬────────────────────────────────────┘       │
  │             │ Garage mode complete / timeout              │
  │  ┌──────────▼────────────────────────────────────┐       │
  │  │  DEEP SLEEP (Suspend-to-RAM)                  │       │
  │  │  SoC suspended, DDR self-refresh              │       │
  │  │  CAN/LIN wakeup controller active             │       │
  │  │  Power: 1-5mW                                 │       │
  │  │  Must maintain <3mA for battery protection    │       │
  │  └──────────┬────────────────────────────────────┘       │
  │             │ 12V battery protection threshold            │
  │  ┌──────────▼────────────────────────────────────┐       │
  │  │  SHUTDOWN                                     │       │
  │  │  Complete power off, no standby current       │       │
  │  │  Cold boot required on next start             │       │
  │  └──────────────────────────────────────────────┘        │
  │                                                           │
  │  Wakeup sources: CAN bus, door open, key fob,           │
  │  charger connect, timer (scheduled maintenance)          │
  └──────────────────────────────────────────────────────────┘
```

---

## 7. IoT/MCU Power Optimization

```
  IoT Device Power Profile
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Duty cycling for sensor node:                           │
  │                                                           │
  │  Power                                                   │
  │   │     ┌─┐         ┌─┐         ┌─┐                     │
  │   │  50mW│ │ TX      │ │ TX      │ │ TX                  │
  │   │     │ │         │ │         │ │                      │
  │   │  10mW─┤├─┐       ├─┤├─┐       ├─┤                    │
  │   │       ││S│       │ ││S│       │ │  S = Sense         │
  │   │   1mW─┘└─┘───────┘ └─┘───────┘ │  TX = Transmit     │
  │   │       Sleep       Sleep                               │
  │   └──────────────────────────────── Time →                │
  │                                                           │
  │  Average power = (T_active × P_active + T_sleep          │
  │                   × P_sleep) / T_total                    │
  │                                                           │
  │  Example: 10ms active (50mW) + 990ms sleep (0.01mW)     │
  │  Avg = (10×50 + 990×0.01) / 1000 = 0.51 mW              │
  │  Battery: 1000 mAh × 3.7V = 3700 mWh / 0.51 mW         │
  │         = 7255 hours = ~302 days                          │
  └──────────────────────────────────────────────────────────┘
```

---

## 8. Wakeup Latency Optimization

```
  Wakeup Latency vs Power Savings Tradeoff
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Sleep State    │ Entry  │ Exit   │ Power  │ Use When     │
  │  ───────────────┼────────┼────────┼────────┼─────────────│
  │  WFI            │ <1μs   │ <1μs   │ ~60%   │ Short idle  │
  │  Clock gate     │ ~10μs  │ ~10μs  │ ~30%   │ Milliseconds│
  │  Retention      │ ~100μs │ ~200μs │ ~10%   │ >1ms idle   │
  │  Power gate     │ ~500μs │ ~1ms   │ ~1%    │ >5ms idle   │
  │  System suspend │ ~100ms │ ~200ms │ ~0.1%  │ >10s idle   │
  │                                                           │
  │  PM QoS: Drivers set latency constraints                 │
  │  cpu_latency_qos_request(qos, 100);  /* max 100μs */    │
  │  → Prevents deep C-states that exceed 100μs exit         │
  │                                                           │
  │  dev_pm_qos_add_request(dev, req,                        │
  │      DEV_PM_QOS_RESUME_LATENCY, 500);  /* max 500μs */  │
  │  → Constrains device's power domain depth                │
  └──────────────────────────────────────────────────────────┘
```

---

## 9. Android-Specific PM Features

```
  Android Power Management Stack
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌── Android Framework ─────────────────────────────┐   │
  │  │  PowerManagerService                              │   │
  │  │  - Screen on/off state machine                    │   │
  │  │  - Wake lock tracking                             │   │
  │  │  - Battery saver mode                             │   │
  │  └───────────────────────┬──────────────────────────┘   │
  │                          │                               │
  │  ┌───────────────────────▼──────────────────────────┐   │
  │  │  Kernel: /sys/power/wake_lock                    │   │
  │  │  (deprecated, replaced by wakeup_sources)        │   │
  │  │                                                   │   │
  │  │  Opportunistic suspend:                          │   │
  │  │  - System auto-suspends when all wake locks      │   │
  │  │    are released                                   │   │
  │  │  - /sys/power/autosleep = "mem"                  │   │
  │  │  - Any wakeup source prevents suspend            │   │
  │  └───────────────────────┬──────────────────────────┘   │
  │                          │                               │
  │  ┌───────────────────────▼──────────────────────────┐   │
  │  │  Doze Mode (App Standby):                        │   │
  │  │  - Restricts background app network/wakeups      │   │
  │  │  - Maintenance windows for deferred work         │   │
  │  │  - Deep Doze: motion-triggered                   │   │
  │  └──────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────┘
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `kernel/power/suspend.c` | System suspend core |
| `kernel/power/autosleep.c` | Android autosleep |
| `kernel/power/wakelock.c` | Android wake_lock interface |
| `drivers/base/power/wakeup.c` | Wakeup source framework |
| `drivers/base/power/domain.c` | Power domain (genpd) |
| `drivers/cpuidle/` | CPU idle state management |
| `drivers/soc/*/` | SoC-specific PM drivers |
| `drivers/firmware/psci/` | ARM PSCI interface |

---

## Interview Questions

**Q1: How does an embedded system decide which idle state to enter?**
**A:** The cpuidle governor (menu or TEO) predicts idle duration using timer information and historical patterns. It then selects the deepest C-state whose exit latency doesn't violate PM QoS constraints and whose break-even time is less than the predicted idle. For embedded, the genpd framework extends this to power domains — when all devices in a domain are idle, the domain can enter a low-power state with its own latency/residency constraints.

**Q2: What is the difference between clock gating and power gating?**
**A:** Clock gating stops the clock signal to a logic block — dynamic power drops to zero but leakage current continues (transistors still powered). State (register contents) is preserved. Power gating cuts power supply entirely — both dynamic and leakage power go to zero, but all state is lost and must be saved/restored. Clock gating is faster (nanoseconds) with moderate savings. Power gating saves more but has higher latency (microseconds to milliseconds).

**Q3: Why is s2idle preferred over S3 on modern ARM SoCs?**
**A:** Modern ARM SoCs with PSCI can power-gate CPUs individually and put DDR in self-refresh during s2idle, achieving power levels close to S3 without firmware complexity. S2idle advantages: faster resume (no firmware handoff), per-CPU state management, selective peripheral wakeup, no ACPI S3 firmware dependency. ARM systems lack standardized S3 (ACPI-centric), so s2idle via PSCI is the natural fit.

---

## Summary

- Embedded PM focuses on maximizing battery life through aggressive power state management
- SoC power domains: always-on (RTC, wakeup), CPU, GPU, peripherals, memory — independently gateable
- Clock gating (fast, preserves state) vs power gating (maximum savings, loses state)
- s2idle is preferred on ARM — CPU power gate + DDR self-refresh via PSCI
- Automotive: Full ON → Garage Mode → Deep Sleep → Shutdown progression
- IoT: Duty cycling with deep sleep between sensor/transmit bursts
- PM QoS constrains idle depth based on wakeup latency requirements
- Android: opportunistic suspend via autosleep + wakeup sources

---

[Previous Chapter: Power Management and Device Tree ←](Chapter_17_Device_Tree_PM.md) | [Next Chapter: Energy-Aware Scheduling →](Chapter_19_EAS.md)
