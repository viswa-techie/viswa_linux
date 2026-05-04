# Chapter 23: Power Management Flow Diagrams

## Learning Goals
- Visualize complete PM flows from userspace to hardware
- Understand suspend/resume call chains end-to-end
- See runtime PM state machine transitions
- Trace CPU frequency/idle decision paths
- Understand thermal throttling event flow

---

## 1. System Suspend Flow (S3 / s2idle)

```
  Complete Suspend Flow
  ═══════════════════════════════════════════════════════════════

  User: echo mem > /sys/power/state
       │
       ▼
  ┌─ kernel/power/main.c ──────────────────────────────────────┐
  │  state_store()                                              │
  │    → pm_suspend(PM_SUSPEND_MEM)                            │
  │      → enter_state(PM_SUSPEND_MEM)                          │
  └────────┬────────────────────────────────────────────────────┘
           │
           ▼
  ┌─ Phase 1: Freeze Processes ────────────────────────────────┐
  │  suspend_prepare()                                         │
  │    → pm_notifier_call_chain(PM_SUSPEND_PREPARE)            │
  │    → suspend_freeze_processes()                            │
  │      → freeze_processes()          ← userspace frozen      │
  │      → freeze_kernel_threads()     ← kthreads frozen       │
  └────────┬───────────────────────────────────────────────────┘
           │
           ▼
  ┌─ Phase 2: Suspend Devices ─────────────────────────────────┐
  │  suspend_devices_and_enter()                               │
  │    → dpm_suspend_start(PMSG_SUSPEND)                       │
  │      → dpm_prepare()                                       │
  │      │   for each device (children first):                 │
  │      │     device_prepare() → dev->pm->prepare()           │
  │      │                                                     │
  │      → dpm_suspend()                                       │
  │        for each device (children first):                   │
  │          __device_suspend()                                │
  │            → dev->pm->suspend()     ← driver saves state   │
  │                                                            │
  │    → dpm_suspend_late()                                    │
  │      for each device:                                      │
  │        → dev->pm->suspend_late()                           │
  │                                                            │
  │    → dpm_suspend_noirq()           ← IRQs disabled        │
  │      for each device:                                      │
  │        → dev->pm->suspend_noirq()  ← final HW access     │
  └────────┬───────────────────────────────────────────────────┘
           │
           ▼
  ┌─ Phase 3: Enter Sleep ─────────────────────────────────────┐
  │  suspend_enter()                                           │
  │    → syscore_suspend()             ← IRQ/timer suspend     │
  │                                                            │
  │    if s2idle:                                              │
  │      → s2idle_loop()                                       │
  │        → cpuidle_enter_s2idle()    ← deepest C-state      │
  │        → wait for wakeup IRQ                               │
  │                                                            │
  │    if S3:                                                  │
  │      → suspend_ops->enter()        ← platform sleep       │
  │        → ACPI S3 / PSCI CPU_SUSPEND                       │
  │        → [SYSTEM SLEEPS HERE]                              │
  └────────┬───────────────────────────────────────────────────┘
           │ (wakeup IRQ)
           ▼
  ┌─ Phase 4: Resume ──────────────────────────────────────────┐
  │  (Reverse order)                                           │
  │    → syscore_resume()                                      │
  │    → dpm_resume_noirq()                                    │
  │    → dpm_resume_early()                                    │
  │    → dpm_resume()                  ← drivers restore state │
  │    → dpm_complete()                                        │
  │    → thaw_processes()              ← unfreeze tasks        │
  │    → pm_notifier_call_chain(PM_POST_SUSPEND)               │
  └────────────────────────────────────────────────────────────┘
```

---

## 2. Runtime PM State Machine

```
  Runtime PM State Transitions
  ═══════════════════════════════════════════════════════════════

                     pm_runtime_get_sync()
                     pm_runtime_resume()
                ┌─────────────────────────────┐
                │                             │
                ▼                             │
  ┌─────────────────┐                ┌────────┴──────────┐
  │   RPM_ACTIVE    │                │  RPM_SUSPENDED    │
  │                 │                │                    │
  │ usage_count > 0 │                │ usage_count = 0   │
  │ Device powered  │                │ Device unpowered  │
  │ Registers OK    │                │ State saved       │
  └────────┬────────┘                └────────▲──────────┘
           │                                  │
           │ pm_runtime_put()                 │
           │ (usage_count → 0)                │
           │                                  │
           ▼                                  │
  ┌─────────────────┐        success  ┌───────┴──────────┐
  │ RPM_SUSPENDING  ├───────────────►│                    │
  │                 │                │  rpm_suspend()     │
  │ →runtime_suspend│                │  completes         │
  │  callback runs  │                │                    │
  └────────┬────────┘                └────────────────────┘
           │ failure
           ▼
  ┌─────────────────┐
  │   RPM_ACTIVE    │ (reverted on error)
  └─────────────────┘

  Autosuspend flow:
  pm_runtime_put_autosuspend()
       │
       ▼
  [Start autosuspend timer: N ms]
       │ timer expires
       ▼
  [pm_runtime_autosuspend_expiration()]
       │ check last_busy timestamp
       ├─ Still within delay → re-arm timer
       └─ Expired → rpm_suspend()
```

---

## 3. CPU Frequency Change Flow

```
  Schedutil Frequency Update Flow
  ═══════════════════════════════════════════════════════════════

  Scheduler: task enqueue/dequeue/tick
       │
       ▼
  ┌─ kernel/sched/cpufreq_schedutil.c ─────────────────────┐
  │  sugov_update_shared() / sugov_update_single()          │
  │    │                                                     │
  │    ▼                                                     │
  │  get_next_freq()                                        │
  │    util = cpu_util_cfs(cpu) + cpu_util_rt(cpu)          │
  │    freq = 1.25 × util × max_freq / capacity            │
  │    │                                                     │
  │    ▼                                                     │
  │  [freq changed significantly?]                          │
  │    NO → return (skip)                                    │
  │    YES ↓                                                 │
  │  sugov_fast_switch() or sugov_deferred_update()         │
  └────────┬────────────────────────────────────────────────┘
           │
           ▼
  ┌─ drivers/cpufreq/cpufreq.c ────────────────────────────┐
  │  cpufreq_driver_fast_switch() or __cpufreq_driver_target│
  │    → driver->fast_switch() or driver->target_index()    │
  └────────┬────────────────────────────────────────────────┘
           │
           ▼
  ┌─ CPUFreq Driver (e.g., cpufreq-dt) ────────────────────┐
  │  cpufreq_dt_target_index()                              │
  │    │                                                     │
  │    ▼                                                     │
  │  [Scaling UP?]                                          │
  │    YES: regulator_set_voltage() → clk_set_rate()        │
  │    NO:  clk_set_rate() → regulator_set_voltage()        │
  │    │                                                     │
  │    ▼                                                     │
  │  [Hardware: PLL relocks, voltage ramp completes]        │
  └─────────────────────────────────────────────────────────┘
```

---

## 4. CPU Idle Entry Flow

```
  CPU Idle Decision and Entry Flow
  ═══════════════════════════════════════════════════════════════

  CPU run queue empty (no tasks to run)
       │
       ▼
  ┌─ kernel/sched/idle.c ──────────────────────────────────┐
  │  do_idle()                                              │
  │    → tick_nohz_idle_enter()    ← stop periodic tick     │
  │    → cpuidle_idle_call()                                │
  └────────┬────────────────────────────────────────────────┘
           │
           ▼
  ┌─ drivers/cpuidle/cpuidle.c ────────────────────────────┐
  │  cpuidle_idle_call()                                    │
  │    → cpuidle_select()          ← ask governor           │
  └────────┬────────────────────────────────────────────────┘
           │
           ▼
  ┌─ Governor (e.g., menu) ────────────────────────────────┐
  │  menu_select()                                          │
  │    1. Get next timer event (predicted idle duration)    │
  │    2. Apply PM QoS constraints (max exit latency)      │
  │    3. Use correction factor from history                │
  │    4. Select deepest state where:                      │
  │       exit_latency ≤ QoS limit                         │
  │       target_residency ≤ predicted idle                │
  │    → return selected state index                        │
  └────────┬────────────────────────────────────────────────┘
           │
           ▼
  ┌─ Idle Driver (e.g., intel_idle) ────────────────────────┐
  │  cpuidle_enter()                                        │
  │    → driver->enter(state)                               │
  │    → intel_idle() / arm_enter_idle_state()              │
  │                                                         │
  │    Intel:  MWAIT instruction with C-state hint          │
  │    ARM:    WFI or PSCI CPU_SUSPEND                      │
  │                                                         │
  │    [CPU IDLE — waiting for interrupt]                    │
  │                                                         │
  │    ← IRQ/timer fires → CPU wakes                        │
  └────────┬────────────────────────────────────────────────┘
           │
           ▼
  ┌─ Post-idle ────────────────────────────────────────────┐
  │  cpuidle_reflect()             ← governor learns       │
  │  tick_nohz_idle_exit()         ← restart tick           │
  │  → schedule next task                                   │
  └─────────────────────────────────────────────────────────┘
```

---

## 5. Thermal Throttling Flow

```
  Thermal Event Processing
  ═══════════════════════════════════════════════════════════════

  [Polling timer fires (every 250ms-1000ms)]
       │
       ▼
  ┌─ drivers/thermal/thermal_core.c ────────────────────────┐
  │  thermal_zone_device_update()                           │
  │    → tz->ops->get_temp(tz, &temp)  ← read sensor       │
  │    → update_temperature(tz)                             │
  └────────┬────────────────────────────────────────────────┘
           │
           ▼
  ┌─ Trip Point Evaluation ─────────────────────────────────┐
  │  for each trip (lowest to highest):                     │
  │    → handle_thermal_trip(tz, trip)                      │
  │                                                         │
  │    [temp >= trip.temperature?]                          │
  │      │                                                  │
  │      ├── CRITICAL → thermal_zone_device_critical()     │
  │      │              → orderly_poweroff(true)            │
  │      │                                                  │
  │      ├── HOT → thermal_emergency_poweroff()            │
  │      │                                                  │
  │      ├── PASSIVE/ACTIVE → invoke governor              │
  │      │                                                  │
  │      └── Below trip-hysteresis → clear trip             │
  └────────┬────────────────────────────────────────────────┘
           │ (PASSIVE trip crossed)
           ▼
  ┌─ Governor (step_wise) ──────────────────────────────────┐
  │  step_wise_throttle()                                   │
  │    → thermal_zone_get_trip()                            │
  │    → get_target_state()                                 │
  │      trend = get_trend(tz)  (rising/dropping/stable)   │
  │      if rising:  new_state = cur_state + 1             │
  │      if dropping && below hysteresis: new_state -= 1    │
  │    → thermal_cdev_update()                              │
  └────────┬────────────────────────────────────────────────┘
           │
           ▼
  ┌─ Cooling Device ────────────────────────────────────────┐
  │  cdev->ops->set_cur_state(cdev, new_state)             │
  │                                                         │
  │  If CPUFreq cooling:                                   │
  │    → cpufreq_set_cur_state()                           │
  │    → freq_qos_update_request()                         │
  │    → CPUFreq max frequency capped                      │
  │    → CPU runs slower → generates less heat              │
  │                                                         │
  │  If fan cooling:                                       │
  │    → set_pwm(fan, duty_cycle)                          │
  │    → Fan speed increases → removes heat                 │
  └─────────────────────────────────────────────────────────┘
```

---

## 6. Power Domain Lifecycle

```
  genpd Power Domain Flow
  ═══════════════════════════════════════════════════════════════

  Device A runtime suspends    Device B runtime suspends
  (last user calls put)        (last user calls put)
       │                            │
       ▼                            ▼
  [rpm_suspend(dev_A)]         [rpm_suspend(dev_B)]
       │                            │
       ▼                            ▼
  [driver->runtime_suspend()]  [driver->runtime_suspend()]
       │                            │
       └──────────┬─────────────────┘
                  │ Both devices suspended
                  ▼
  ┌─ genpd_runtime_suspend() ──────────────────────────────┐
  │  [All devices in domain suspended?]                     │
  │    YES → genpd_power_off()                              │
  │      → domain->power_off(domain)   ← cut power to block│
  │      → [Hardware: power switch opens]                   │
  │      → domain status = "off"                            │
  │                                                         │
  │  [Parent domain has other active children?]             │
  │    NO → genpd_power_off(parent)    ← cascade up        │
  │    YES → parent stays on                                │
  └─────────────────────────────────────────────────────────┘

  Device A needs to wake up:
  ┌─ genpd_runtime_resume() ───────────────────────────────┐
  │  [Domain off?]                                          │
  │    YES → genpd_power_on()                               │
  │      → [Parent off?] YES → genpd_power_on(parent) first│
  │      → domain->power_on(domain)    ← restore power     │
  │      → [Hardware: power switch closes, ramp-up delay]   │
  │      → domain status = "on"                             │
  │    → driver->runtime_resume(dev_A)                      │
  │      → [Restore registers, enable clock]                │
  │                                                         │
  │  Device B stays suspended (runtime PM independent)      │
  └─────────────────────────────────────────────────────────┘
```

---

## 7. Device PM Callback Ordering

```
  Suspend/Resume Callback Order (Per Device)
  ═══════════════════════════════════════════════════════════════

  SUSPEND (top to bottom)          RESUME (bottom to top)
  ─────────────────────────        ────────────────────────
  .prepare()                       .complete()
    │ "Will we suspend?"             │ "All done"
    ▼                                ▲
  .suspend()                       .resume()
    │ Save state, stop I/O           │ Restore state, start I/O
    │ IRQs still enabled             │ IRQs enabled
    ▼                                ▲
  .suspend_late()                  .resume_early()
    │ Late cleanup                   │ Early setup
    │ IRQs still enabled             │ IRQs enabled
    ▼                                ▲
  .suspend_noirq()                 .resume_noirq()
    │ IRQs disabled                  │ IRQs disabled
    │ Final HW access                │ First HW access
    ▼                                ▲
  [SYSTEM SLEEPS]  ══════════►  [SYSTEM WAKES]

  Device ordering:
  Suspend: Children before parents (leaves first)
  Resume:  Parents before children (root first)

  Multiple devices:
  Suspend: DevC → DevB → DevA (if A is parent of B, B of C)
  Resume:  DevA → DevB → DevC
```

---

## Kernel Source Reference

| Flow | Entry Point | Key File |
|------|-------------|----------|
| System suspend | `state_store()` | `kernel/power/main.c` |
| Device suspend | `dpm_suspend()` | `drivers/base/power/main.c` |
| Runtime PM | `rpm_suspend/resume()` | `drivers/base/power/runtime.c` |
| CPU idle | `do_idle()` | `kernel/sched/idle.c` |
| CPU freq | `sugov_update_shared()` | `kernel/sched/cpufreq_schedutil.c` |
| Thermal | `thermal_zone_device_update()` | `drivers/thermal/thermal_core.c` |
| Power domain | `genpd_runtime_suspend()` | `drivers/base/power/domain.c` |

---

## Interview Questions

**Q1: Walk through what happens when you write "mem" to /sys/power/state.**
**A:** `state_store()` parses "mem" → `pm_suspend(PM_SUSPEND_MEM)` → `enter_state()`. Phase 1: PM notifiers called, userspace processes frozen, kernel threads frozen. Phase 2: Devices suspended in child-before-parent order through prepare→suspend→suspend_late→suspend_noirq callbacks. Phase 3: Non-boot CPUs offlined, syscore suspend, platform enters S3 (ACPI) or deepest PSCI idle (s2idle). System sleeps. On wakeup IRQ: reverse order — syscore resume, devices resume (parent-before-child, noirq→early→resume→complete), processes thawed.

**Q2: How does the kernel decide whether to enter C1, C3, or C6?**
**A:** When CPU has no tasks, `do_idle()` calls `cpuidle_select()` which invokes the governor (menu/TEO). The menu governor: (1) reads next timer expiry to predict idle duration, (2) applies a correction factor from historical accuracy, (3) checks PM QoS constraints for maximum allowed exit latency, (4) iterates C-states from deepest to shallowest, selecting the deepest where exit_latency ≤ QoS limit AND target_residency ≤ predicted idle. For example, if predicted idle is 2ms and C6 needs 5ms residency, C3 (1ms residency) is chosen instead.

---

## Summary

- System suspend flows through freeze→device_suspend→enter_sleep→resume→thaw
- Runtime PM uses a state machine: ACTIVE ↔ SUSPENDING → SUSPENDED ↔ RESUMING
- CPU frequency: scheduler utility tracking → schedutil calculates freq → driver sets HW
- CPU idle: governor predicts idle duration → selects deepest valid C-state → driver enters HW idle
- Thermal: poll temperature → check trips → governor adjusts cooling state → cooling device limits freq/fan
- Power domains: last device suspends → domain powers off → cascades to parent if all children off
- All flows follow consistent patterns: policy decision (governor) → mechanism (driver/HW)

---

[Previous Chapter: Kernel Source Code Map ←](Chapter_22_Source_Code_Map.md) | [Next Chapter: Architecture Diagrams →](Chapter_24_Architecture_Diagrams.md)
