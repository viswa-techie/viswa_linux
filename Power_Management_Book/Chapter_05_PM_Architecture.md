# Chapter 5: Linux Power Management Architecture

## Learning Goals
- Understand the overall architecture of Linux PM subsystems
- Learn the PM core framework and its interactions
- Understand the policy vs mechanism separation in Linux PM
- Know the key data structures and APIs in the PM framework
- Learn how PM callbacks are organized and invoked

---

## 1. Linux PM Architecture Overview

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                         USER SPACE                                │
  │  ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────────┐   │
  │  │powertop │ │turbostat │ │systemctl │ │ thermald          │   │
  │  │         │ │          │ │ suspend  │ │ tlp, laptop-mode  │   │
  │  └────┬────┘ └────┬─────┘ └────┬─────┘ └────────┬──────────┘   │
  ├───────┴────────────┴────────────┴────────────────┴──────────────┤
  │                       SYSFS INTERFACE                             │
  │  /sys/power/      /sys/devices/.../power/    /sys/class/thermal/ │
  │  /sys/devices/system/cpu/cpufreq/            /sys/class/regulator│
  ├──────────────────────────────────────────────────────────────────┤
  │                        KERNEL SPACE                               │
  │                                                                   │
  │  ┌──────────────── PM CORE ─────────────────────────────────┐   │
  │  │  kernel/power/main.c — PM framework initialization       │   │
  │  │  kernel/power/suspend.c — System suspend/resume          │   │
  │  │  kernel/power/hibernate.c — Hibernate support            │   │
  │  │  kernel/power/qos.c — PM Quality of Service              │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │                                                                   │
  │  ┌──── CPU PM ─────────┐  ┌──── Device PM ────────────────┐   │
  │  │  CPUFreq framework  │  │  drivers/base/power/           │   │
  │  │  ├── Governors      │  │  ├── Runtime PM (runtime.c)    │   │
  │  │  │  (schedutil,     │  │  ├── Wakeup (wakeup.c)        │   │
  │  │  │   ondemand)      │  │  ├── Sysfs (sysfs.c)          │   │
  │  │  ├── Scaling drivers│  │  ├── Main (main.c)            │   │
  │  │  │  (intel_pstate,  │  │  └── QoS (qos.c)             │   │
  │  │  │   acpi-cpufreq)  │  │                                │   │
  │  │  │                  │  │  Power Domains (domain.c)      │   │
  │  │  CPUIdle framework  │  │  Generic PD (genpd)            │   │
  │  │  ├── Governors      │  │                                │   │
  │  │  │  (menu, TEO)     │  │  OPP framework                │   │
  │  │  └── State tables   │  │  (drivers/opp/)               │   │
  │  └────────────────────┘  └────────────────────────────────┘   │
  │                                                                   │
  │  ┌──── Infrastructure ───────────────────────────────────────┐   │
  │  │  Clock Framework (drivers/clk/) — CCF                     │   │
  │  │  Regulator Framework (drivers/regulator/)                 │   │
  │  │  Thermal Framework (drivers/thermal/)                     │   │
  │  │  Energy Model (kernel/power/energy_model.c)               │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │                                                                   │
  │  ┌──── Bus PM ──────────────────────────────────────────────┐   │
  │  │  PCI PM │ USB PM │ Platform PM │ I2C/SPI PM │ AMBA PM    │   │
  │  └──────────────────────────────────────────────────────────┘   │
  ├──────────────────────────────────────────────────────────────────┤
  │                      HARDWARE / FIRMWARE                          │
  │  ACPI/DSDT │ PSCI/ATF │ PMIC │ PLLs │ Voltage Regulators       │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 2. PM Core Framework

### 2.1 Core Components

```c
/* kernel/power/main.c — PM core initialization */

/* PM core creates /sys/power/ directory */
static int __init pm_init(void)
{
    int error = pm_start_workqueue();
    power_kobj = kobject_create_and_add("power", NULL);
    /* Creates sysfs attributes:
     * /sys/power/state
     * /sys/power/mem_sleep
     * /sys/power/disk
     * /sys/power/pm_async
     * /sys/power/wakeup_count
     */
    error = sysfs_create_group(power_kobj, &attr_group);
    pm_print_times_init();
    return pm_autosleep_init();
}
core_initcall(pm_init);
```

### 2.2 PM Operations Structure

```c
/* include/linux/pm.h — The central PM ops structure */
struct dev_pm_ops {
    /* System suspend/resume */
    int (*prepare)(struct device *dev);
    void (*complete)(struct device *dev);
    int (*suspend)(struct device *dev);
    int (*resume)(struct device *dev);
    int (*freeze)(struct device *dev);
    int (*thaw)(struct device *dev);
    int (*poweroff)(struct device *dev);
    int (*restore)(struct device *dev);

    /* Late/early variants (called with IRQs disabled) */
    int (*suspend_late)(struct device *dev);
    int (*resume_early)(struct device *dev);
    int (*freeze_late)(struct device *dev);
    int (*thaw_early)(struct device *dev);
    int (*poweroff_late)(struct device *dev);
    int (*restore_early)(struct device *dev);

    /* Noirq variants (with interrupts disabled) */
    int (*suspend_noirq)(struct device *dev);
    int (*resume_noirq)(struct device *dev);
    int (*freeze_noirq)(struct device *dev);
    int (*thaw_noirq)(struct device *dev);
    int (*poweroff_noirq)(struct device *dev);
    int (*restore_noirq)(struct device *dev);

    /* Runtime PM */
    int (*runtime_suspend)(struct device *dev);
    int (*runtime_resume)(struct device *dev);
    int (*runtime_idle)(struct device *dev);
};
```

### 2.3 PM Callback Invocation Order

```
  System Suspend Callback Order (per device):
  ┌───────────────────────────────────────────────────────┐
  │                                                        │
  │  1. prepare()           ← Prevent new children         │
  │     │                                                  │
  │  2. suspend()           ← Save state, quiesce device   │
  │     │   (with IRQs enabled, may schedule)              │
  │     │                                                  │
  │  3. suspend_late()      ← Final cleanup                │
  │     │   (IRQs enabled, may NOT schedule)               │
  │     │                                                  │
  │  4. suspend_noirq()     ← IRQs disabled, HW access    │
  │     │   (Interrupts disabled, no scheduling)           │
  │     │                                                  │
  │  ════════ CPU enters sleep / suspend ════════          │
  │     │                                                  │
  │  5. resume_noirq()      ← First to run, IRQs off      │
  │     │                                                  │
  │  6. resume_early()      ← IRQs enabled                 │
  │     │                                                  │
  │  7. resume()            ← Restore state, re-enable     │
  │     │                                                  │
  │  8. complete()          ← Allow new children           │
  │                                                        │
  │  Order: Children suspended BEFORE parents              │
  │         Parents resumed BEFORE children                │
  └───────────────────────────────────────────────────────┘
```

---

## 3. Device PM Framework

### 3.1 struct device PM Fields

```c
/* include/linux/device.h — PM-related fields in struct device */
struct device {
    /* ... other fields ... */

    struct dev_pm_info power;  /* PM state and info */

    /* ... */
};

/* include/linux/pm.h — struct dev_pm_info */
struct dev_pm_info {
    pm_message_t     power_state;        /* Current PM state */
    unsigned int     can_wakeup:1;       /* Can generate wakeups */
    unsigned int     async_suspend:1;    /* Async suspend allowed */
    unsigned int     in_dpm_list:1;      /* In device PM list */
    unsigned int     is_prepared:1;      /* prepare() called */
    unsigned int     is_suspended:1;     /* Suspended */

    /* Runtime PM */
    struct timer_list suspend_timer;     /* Auto-suspend timer */
    int              runtime_auto;       /* Autosuspend enabled */
    atomic_t         usage_count;        /* Runtime PM usage */
    atomic_t         child_count;        /* Active children */
    unsigned int     disable_depth;      /* Nesting depth */
    enum rpm_status  runtime_status;     /* active/suspended */
    int              autosuspend_delay;  /* Delay in ms */

    /* Wakeup */
    struct wakeup_source *wakeup;        /* Wakeup source info */

    /* QoS */
    struct dev_pm_qos *qos;             /* QoS constraints */
};
```

### 3.2 Device PM List

```
  Device PM List (dpm_list)
  ┌────────────────────────────────────────────────────┐
  │                                                     │
  │  During suspend, devices are traversed in REVERSE   │
  │  registration order (children before parents):      │
  │                                                     │
  │  Suspend order:                                     │
  │  device_N → device_N-1 → ... → device_1 → device_0│
  │  (leaf)                                 (root)      │
  │                                                     │
  │  Resume order:                                      │
  │  device_0 → device_1 → ... → device_N-1 → device_N│
  │  (root)                                 (leaf)      │
  │                                                     │
  │  This ensures parent bus/controller is active       │
  │  before child device tries to communicate           │
  └────────────────────────────────────────────────────┘
```

---

## 4. Policy vs Mechanism Separation

### 4.1 Architecture Pattern

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  POLICY (Governors/Algorithms)                           │
  │  "WHEN to change state and WHAT to change to"            │
  │                                                           │
  │  ┌──────────┐  ┌──────────┐  ┌───────────┐             │
  │  │ CPUFreq  │  │ CPUIdle  │  │ Thermal   │             │
  │  │Governors │  │Governors │  │ Governors │             │
  │  │          │  │          │  │           │             │
  │  │schedutil │  │ menu     │  │step_wise  │             │
  │  │ondemand  │  │ TEO      │  │bang_bang  │             │
  │  │conserv.  │  │ ladder   │  │power_alloc│             │
  │  └────┬─────┘  └────┬─────┘  └─────┬─────┘             │
  │       │             │              │                     │
  │  ─────┼─────────────┼──────────────┼─── Policy API ──── │
  │       │             │              │                     │
  │  ┌────▼─────┐  ┌────▼─────┐  ┌─────▼─────┐             │
  │  │ CPUFreq  │  │ CPUIdle  │  │ Thermal   │             │
  │  │ Drivers  │  │ Drivers  │  │ Drivers   │             │
  │  │          │  │          │  │           │             │
  │  │intel_    │  │acpi_idle │  │of_thermal │             │
  │  │pstate    │  │arm_idle  │  │hwmon     │             │
  │  │acpi-cpuf │  │          │  │           │             │
  │  └──────────┘  └──────────┘  └───────────┘             │
  │                                                           │
  │  MECHANISM (Drivers)                                     │
  │  "HOW to actually change hardware state"                 │
  │                                                           │
  └──────────────────────────────────────────────────────────┘
```

### 4.2 CPUFreq Policy/Mechanism Split

```c
/* Policy: Governor decides target frequency */
struct cpufreq_governor {
    char    name[CPUFREQ_NAME_LEN];
    int     (*init)(struct cpufreq_policy *policy);
    void    (*exit)(struct cpufreq_policy *policy);
    int     (*start)(struct cpufreq_policy *policy);
    void    (*stop)(struct cpufreq_policy *policy);
    void    (*limits)(struct cpufreq_policy *policy);
    /* Governor-specific attributes in sysfs */
    struct  attribute_group **attr_groups;
};

/* Mechanism: Driver programs hardware */
struct cpufreq_driver {
    char    name[CPUFREQ_NAME_LEN];
    int     (*init)(struct cpufreq_policy *policy);
    int     (*verify)(struct cpufreq_policy_data *policy);
    int     (*target_index)(struct cpufreq_policy *policy,
                            unsigned int index);
    unsigned int (*get)(unsigned int cpu);
    /* ... */
};

/* Flow: Governor → target_freq → Driver → HW */
```

---

## 5. PM Quality of Service (PM QoS)

### 5.1 PM QoS Framework

```
  PM QoS: Constraints on PM Decisions
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Global PM QoS (system-wide):                            │
  │  ┌────────────────────────────────────────────────┐      │
  │  │  CPU_DMA_LATENCY:                              │      │
  │  │  Maximum tolerable CPU wakeup latency          │      │
  │  │  Affects which C-states are allowed            │      │
  │  │                                                │      │
  │  │  Example: Audio driver needs <50μs latency     │      │
  │  │  → CPUIdle won't enter C6 (200μs exit latency)│      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Per-Device PM QoS:                                      │
  │  ┌────────────────────────────────────────────────┐      │
  │  │  DEV_PM_QOS_RESUME_LATENCY:                    │      │
  │  │  Maximum acceptable resume latency for device   │      │
  │  │                                                │      │
  │  │  DEV_PM_QOS_FLAGS:                              │      │
  │  │  NO_POWER_OFF — prevent D3cold                  │      │
  │  │  REMOTE_WAKEUP — require wake capability        │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Multiple requesters, aggregated (minimum latency wins): │
  │  Audio: 50μs ─┐                                         │
  │  USB:  500μs  ├── Effective: min(50, 500, 100) = 50μs   │
  │  Net:  100μs ─┘                                         │
  └──────────────────────────────────────────────────────────┘
```

### 5.2 PM QoS API

```c
/* PM QoS usage example */
#include <linux/pm_qos.h>

struct pm_qos_request my_qos_req;

/* Request max 50μs CPU DMA latency */
static int my_driver_start(void)
{
    cpu_latency_qos_add_request(&my_qos_req, 50);
    /* Now CPUIdle won't enter states with >50μs exit latency */
    return 0;
}

static void my_driver_stop(void)
{
    cpu_latency_qos_remove_request(&my_qos_req);
    /* Constraint removed, deep C-states allowed again */
}

/* Per-device PM QoS */
static void set_device_latency(struct device *dev)
{
    dev_pm_qos_add_request(dev, &req,
                           DEV_PM_QOS_RESUME_LATENCY,
                           100); /* max 100μs resume */
}
```

---

## 6. Wakeup Sources

### 6.1 Wakeup Framework

```
  Wakeup Source Management
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Wakeup sources are events that can wake system from     │
  │  sleep (S3/S4) or prevent sleep entry:                   │
  │                                                           │
  │  ┌──────────────┐  ┌──────────────┐                     │
  │  │ Power button │  │ Keyboard     │                     │
  │  │ (GPIO IRQ)   │  │ (USB device) │                     │
  │  └──────┬───────┘  └──────┬───────┘                     │
  │         │                 │                              │
  │    ┌────▼─────────────────▼────┐                         │
  │    │  Wakeup Source Framework  │                         │
  │    │  (drivers/base/power/     │                         │
  │    │   wakeup.c)               │                         │
  │    │                           │                         │
  │    │  - Tracks active wakeups  │                         │
  │    │  - Prevents suspend if    │                         │
  │    │    wakeup event pending   │                         │
  │    │  - Counts wakeup events   │                         │
  │    └───────────────────────────┘                         │
  │                                                           │
  │  /sys/devices/.../power/wakeup = "enabled"/"disabled"    │
  │  /sys/power/wakeup_count — total wakeup events           │
  └──────────────────────────────────────────────────────────┘
```

### 6.2 Wakeup Source API

```c
/* Wakeup source operations */
#include <linux/pm_wakeup.h>

/* Register device as wakeup source */
device_init_wakeup(dev, true);

/* Report wakeup event (blocks suspend) */
pm_wakeup_event(dev, 200); /* 200ms timeout */

/* Start a wakeup event (manual control) */
__pm_stay_awake(dev->power.wakeup);
/* ... process wakeup event data ... */
__pm_relax(dev->power.wakeup);

/* Check if wakeup event is pending */
if (pm_wakeup_pending()) {
    /* Abort suspend — wakeup event occurred */
    return -EBUSY;
}
```

---

## 7. PM Notifier Chain

### 7.1 Suspend Notifiers

```c
/* Notifier chain for PM events */
#include <linux/suspend.h>

static int my_pm_notifier(struct notifier_block *nb,
                          unsigned long event, void *data)
{
    switch (event) {
    case PM_SUSPEND_PREPARE:
        /* System about to suspend — prepare */
        /* Flush work, stop DMA, etc. */
        break;
    case PM_POST_SUSPEND:
        /* System has resumed */
        /* Restart operations */
        break;
    case PM_HIBERNATION_PREPARE:
        /* About to hibernate */
        break;
    case PM_POST_HIBERNATION:
        /* Resumed from hibernate */
        break;
    }
    return NOTIFY_OK;
}

static struct notifier_block my_pm_nb = {
    .notifier_call = my_pm_notifier,
};

/* Register */
register_pm_notifier(&my_pm_nb);
```

---

## 8. Async Suspend/Resume

### 8.1 Parallel PM Transitions

```
  Synchronous Suspend (sequential):
  Time ──────────────────────────────────────────►
  Dev_A: ████████
  Dev_B:          ████████
  Dev_C:                   ████████
  Total: ────────────────────────── 3T

  Asynchronous Suspend (parallel):
  Time ──────────────────────────────────────────►
  Dev_A: ████████
  Dev_B: ████████
  Dev_C: ████████
  Total: ──────── T (3x faster!)

  Constraint: Children must complete before parents.
  Parent waits for all children then suspends.

  Enable per-device:
  device_enable_async_suspend(dev);
  
  Or via sysfs:
  /sys/devices/.../power/async = "enabled"
```

```c
/* Async suspend infrastructure */
/* drivers/base/power/main.c */

static void dpm_suspend_async(pm_message_t state)
{
    while (!list_empty(&dpm_prepared_list)) {
        struct device *dev = to_device(dpm_prepared_list.prev);

        if (dev->power.async_suspend) {
            /* Submit suspend work to async queue */
            async_schedule_dev(async_suspend, dev);
        } else {
            /* Synchronous — block and wait */
            device_suspend(dev);
        }
    }
    async_synchronize_full();  /* Wait for all async */
}
```

---

## 9. PM Domains (genpd) Architecture

```
  Generic Power Domain Architecture
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Power Domain Hierarchy (SoC-specific):                  │
  │                                                           │
  │  ┌── PD_always_on ────────────────────────────────────┐  │
  │  │  (Never turns off)                                  │  │
  │  │                                                     │  │
  │  │  ┌── PD_cpu ──────────┐  ┌── PD_gpu ──────────┐   │  │
  │  │  │  CPU cores, L2     │  │  GPU cores          │   │  │
  │  │  └───────────────────┘  └─────────────────────┘   │  │
  │  │                                                     │  │
  │  │  ┌── PD_display ──────┐  ┌── PD_camera ───────┐   │  │
  │  │  │  Display ctrl      │  │  ISP, CSI           │   │  │
  │  │  │  DSI PHY           │  │                     │   │  │
  │  │  └───────────────────┘  └─────────────────────┘   │  │
  │  └─────────────────────────────────────────────────────┘  │
  │                                                           │
  │  genpd Framework:                                        │
  │  - Parent domain must be ON if any child is ON           │
  │  - Domain turns off only when ALL devices are suspended  │
  │  - Handles power sequencing and isolation                │
  │  - Integrates with runtime PM                            │
  └──────────────────────────────────────────────────────────┘
```

---

## 10. Energy Model

### 10.1 Energy Model Framework

```c
/* kernel/power/energy_model.c */
/* Provides energy cost estimation for scheduler (EAS) */

struct em_perf_domain {
    struct em_perf_state *table; /* OPP table with power */
    int nr_perf_states;          /* Number of OPPs */
    unsigned long cpus[];        /* CPUs in this domain */
};

struct em_perf_state {
    unsigned long frequency;     /* KHz */
    unsigned long power;         /* mW (at this OPP) */
    unsigned long cost;          /* Normalized energy cost */
    unsigned long flags;
};

/* Usage by scheduler:
 * Energy to run task on CPU at given OPP =
 *   em_cpu_energy(pd, max_util, sum_util, allowed_cpu_cap)
 * Scheduler picks CPU/OPP with minimum total energy */
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `kernel/power/main.c` | PM core, sysfs, initialization |
| `kernel/power/suspend.c` | System suspend state machine |
| `kernel/power/hibernate.c` | Hibernate (S4) implementation |
| `kernel/power/qos.c` | Global PM QoS |
| `kernel/power/energy_model.c` | Energy model for EAS |
| `drivers/base/power/main.c` | Device PM list, suspend/resume |
| `drivers/base/power/runtime.c` | Runtime PM framework |
| `drivers/base/power/wakeup.c` | Wakeup source management |
| `drivers/base/power/domain.c` | Generic power domains |
| `drivers/base/power/qos.c` | Per-device PM QoS |
| `include/linux/pm.h` | dev_pm_ops, PM messages |
| `include/linux/pm_runtime.h` | Runtime PM API |
| `include/linux/pm_qos.h` | PM QoS API |
| `include/linux/pm_wakeup.h` | Wakeup source API |

---

## Interview Questions

**Q1: Describe the Linux PM callback invocation order during suspend.**
**A:** For each device, in child-before-parent order: (1) prepare() — prevents new child registration, (2) suspend() — saves device state with IRQs enabled, (3) suspend_late() — final cleanup, IRQs enabled but no scheduling, (4) suspend_noirq() — IRQs disabled, direct HW access. On resume, the reverse: resume_noirq → resume_early → resume → complete, in parent-before-child order.

**Q2: What is PM QoS and how does it affect C-states?**
**A:** PM QoS (Quality of Service) allows drivers and userspace to express latency constraints. The CPU_DMA_LATENCY constraint sets the maximum acceptable CPU wake-up latency. Multiple requesters are aggregated (minimum wins). The CPUIdle governor consults PM QoS before selecting a C-state — if the constraint is 50μs, it won't enter C6 (200μs exit latency). This prevents deep idle when real-time responsiveness is needed.

**Q3: Explain the policy vs mechanism separation in Linux PM.**
**A:** Linux PM separates what decision to make (policy/governor) from how to implement it (mechanism/driver). CPUFreq governors (schedutil, ondemand) decide target frequency; CPUFreq drivers (intel_pstate, acpi-cpufreq) program hardware registers. CPUIdle governors (menu, TEO) predict idle duration and select C-state; CPUIdle drivers interact with hardware. This allows mixing any governor with any driver and swapping policies without modifying hardware code.

**Q4: How does async suspend work and why is it important?**
**A:** Async suspend allows independent devices to suspend/resume in parallel rather than sequentially. When enabled (device_enable_async_suspend), PM callbacks run on async worker threads. The framework ensures children complete before parents start. This significantly reduces suspend/resume time — critical for user experience (laptop lid close → ready in 1-2s). The improvement scales with the number of independent devices.

**Q5: What are wakeup sources and how do they prevent suspend?**
**A:** Wakeup sources are devices/events that can bring the system out of sleep. When a wakeup event occurs during suspend (pm_wakeup_event), the framework sets a pending flag. The suspend code checks pm_wakeup_pending() at multiple points and aborts if set, returning -EBUSY. This prevents the system from sleeping immediately after a key press or network packet. The wakeup_count mechanism allows userspace to coordinate suspend with pending events.

---

## Summary

- Linux PM architecture separates policy (governors) from mechanism (hardware drivers)
- PM core lives in `kernel/power/` — manages system suspend, hibernate, QoS, energy model
- Device PM in `drivers/base/power/` — runtime PM, wakeup sources, power domains, sysfs
- dev_pm_ops provides prepare/suspend/resume/complete callbacks in ordered phases
- Suspend order: children before parents; resume order: parents before children
- PM QoS aggregates latency constraints from multiple requesters
- Wakeup sources can abort suspend and wake the system from sleep states
- Async suspend enables parallel device transitions, reducing suspend/resume time
- Energy model provides power cost data for scheduler-based PM decisions

---

[Previous Chapter: Power States ←](Chapter_04_Power_States.md) | [Next Chapter: CPU Power Management →](Chapter_06_CPU_Power_Management.md)
