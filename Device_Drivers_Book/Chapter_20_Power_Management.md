# Chapter 20: Power Management in Drivers

## Chapter Overview

Power management is critical for embedded and mobile devices. Drivers must implement system suspend/resume and runtime PM callbacks to enable power savings and fast wake-up.

---

## 20.1 Power Management Concepts

```
Power Management in Linux:

  ┌────────────────────┐
  │   System PM         │  Full system suspend (S3/S2R, S4/S2D)
  │   (Sleep/Hibernate) │  All devices suspended, CPU(s) off
  └────────┬───────────┘
           │
  ┌────────▼───────────┐
  │   Runtime PM        │  Per-device power management
  │   (autosuspend)     │  Device idles → power off individually
  └────────┬───────────┘  CPU stays running
           │
  ┌────────▼───────────┐
  │   Clock Gating      │  Disable device clock when idle
  │   (fine-grained)    │  Saves dynamic power
  └────────────────────┘
```

---

## 20.2 Device Power States

| State | Power | Latency | Context |
|-------|-------|---------|---------|
| **D0** (Active) | Full | None | All registers valid |
| **D1** (Light sleep) | Reduced | Low | Partial context saved |
| **D2** (Deep sleep) | Lower | Medium | Most context lost |
| **D3** (Off) | Minimal/None | High | All context lost |

---

## 20.3 Runtime Power Management

Individual devices can be powered off when idle, independent of system state.

```c
#include <linux/pm_runtime.h>

/* In probe(): enable runtime PM */
pm_runtime_set_active(dev);             /* Mark as active */
pm_runtime_enable(dev);                 /* Enable RPM */
pm_runtime_set_autosuspend_delay(dev, 200); /* 200ms autosuspend delay */
pm_runtime_use_autosuspend(dev);

/* In I/O path: mark device as busy */
pm_runtime_get_sync(dev);              /* Resume if suspended, inc refcount */
/* ... do hardware access ... */
pm_runtime_mark_last_busy(dev);
pm_runtime_put_autosuspend(dev);       /* Dec refcount, autosuspend after delay */

/* In remove(): */
pm_runtime_disable(dev);
pm_runtime_set_suspended(dev);
```

### Runtime PM Callbacks

```c
static int my_runtime_suspend(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    /* Save state, disable clocks, power off */
    clk_disable_unprepare(priv->clk);
    return 0;
}

static int my_runtime_resume(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    /* Restore clocks, restore state */
    clk_prepare_enable(priv->clk);
    writel(priv->saved_cfg, priv->base + REG_CFG);
    return 0;
}
```

### Runtime PM Flow

```
Device in use (D0)
       │
       └── pm_runtime_put_autosuspend()
              │
              ▼ (after 200ms idle)
       runtime_suspend callback
       → clk_disable, power off
       Device in D3
              │
              ▼ (next I/O operation)
       pm_runtime_get_sync()
       → runtime_resume callback
       → clk_enable, restore regs
       Device in D0
```

---

## 20.4 System Suspend and Resume

Full system sleep — all devices must suspend.

```
System Suspend Flow:
userspace: echo mem > /sys/power/state
       │
       ▼
Freeze userspace processes
       │
       ▼
Walk device tree (leaf → root order):
  For each device:
    device->driver->pm->suspend(dev)
       │
       ▼
Disable non-boot CPUs
       │
       ▼
arch_suspend_enter()  → CPU to low power
       │
       ▼  (wake event: RTC, GPIO, USB)
CPU resumes
       │
       ▼
Walk device tree (root → leaf order):
  For each device:
    device->driver->pm->resume(dev)
       │
       ▼
Thaw userspace processes
```

### System PM Callbacks

```c
static int my_suspend(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);

    /* Save device state */
    priv->saved_ctrl = readl(priv->base + REG_CTRL);
    priv->saved_cfg = readl(priv->base + REG_CFG);

    /* Disable device */
    writel(0, priv->base + REG_CTRL);

    /* Disable clock */
    clk_disable_unprepare(priv->clk);

    return 0;
}

static int my_resume(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);

    /* Re-enable clock */
    clk_prepare_enable(priv->clk);

    /* Restore device state */
    writel(priv->saved_cfg, priv->base + REG_CFG);
    writel(priv->saved_ctrl, priv->base + REG_CTRL);

    return 0;
}
```

---

## 20.5 Driver Power Callbacks

### struct dev_pm_ops (Comprehensive)

```c
static const struct dev_pm_ops my_pm_ops = {
    /* System suspend/resume */
    SET_SYSTEM_SLEEP_PM_OPS(my_suspend, my_resume)

    /* Runtime PM */
    SET_RUNTIME_PM_OPS(my_runtime_suspend,
                       my_runtime_resume,
                       NULL)

    /* Late suspend (after IRQs disabled) — rare */
    SET_LATE_SYSTEM_SLEEP_PM_OPS(my_late_suspend, my_late_resume)
};

static struct platform_driver my_driver = {
    .driver = {
        .name = "my-device",
        .pm   = &my_pm_ops,
    },
    /* ... */
};
```

### Using Runtime PM for System PM

```c
/* If runtime PM callbacks do everything needed: */
static const struct dev_pm_ops my_pm_ops = {
    SET_SYSTEM_SLEEP_PM_OPS(
        pm_runtime_force_suspend,   /* System suspend = force runtime suspend */
        pm_runtime_force_resume     /* System resume = force runtime resume */
    )
    SET_RUNTIME_PM_OPS(my_runtime_suspend, my_runtime_resume, NULL)
};
```

---

## Wakeup Sources

```c
/* Mark device as capable of waking the system */
device_init_wakeup(dev, true);

/* In suspend: enable wakeup if configured */
if (device_may_wakeup(dev))
    enable_irq_wake(priv->irq);

/* In resume: disable wakeup */
if (device_may_wakeup(dev))
    disable_irq_wake(priv->irq);
```

```bash
# Userspace: check/configure wakeup
cat /sys/devices/.../power/wakeup          # enabled/disabled
echo enabled > /sys/devices/.../power/wakeup

# Runtime PM status
cat /sys/devices/.../power/runtime_status  # active/suspended
cat /sys/devices/.../power/runtime_usage   # reference count
cat /sys/devices/.../power/control         # auto/on
echo auto > /sys/devices/.../power/control # enable autosuspend
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| `drivers/base/power/runtime.c` | Runtime PM implementation |
| `drivers/base/power/main.c` | System PM device iteration |
| `include/linux/pm.h` | PM callbacks definition |
| `include/linux/pm_runtime.h` | Runtime PM API |
| `kernel/power/suspend.c` | System suspend core |

---

## OS Comparison

| Aspect | Linux | Windows | Android | QNX |
|--------|-------|---------|---------|-----|
| System PM | S3 (mem), S4 (disk) | S1-S4, Modern Standby | Doze, App Standby | Power manager |
| Per-device | Runtime PM | D-states (D0-D3) | Wakelocks + RPM | Power-aware drivers |
| Clock mgmt | clk_disable_unprepare() | Power IRP framework | clk framework | ClockPeriod() |
| Wakeup | IRQ wake, GPIO | WDF wakeup | Partial/Full wakelock | Pulse/Event wake |

---

## Interview Questions

**Q1: What is the difference between system PM and runtime PM?**
A: System PM suspends ALL devices when the whole system sleeps (S3/S4). Runtime PM suspends INDIVIDUAL devices when idle, while the system stays running. A UART with no activity auto-suspends (runtime PM), but all devices suspend during lid close (system PM).

**Q2: Why use `pm_runtime_get_sync()` before hardware access?**
A: The device may be runtime-suspended (clocks off, power gated). `pm_runtime_get_sync()` calls the runtime_resume callback (re-enables clocks, restores registers) before your code accesses hardware. Without it: bus error or stale data.

**Q3: What is autosuspend delay and why is it useful?**
A: A grace period (e.g., 200ms) after the last put before actually suspending. Prevents rapid suspend/resume cycles when a device is used in bursts (e.g., I2C sensor polled every 100ms).

**Q4: How do wakeup sources work?**
A: Certain IRQs can wake the system from suspend. The driver calls `enable_irq_wake(irq)` during suspend to keep the IRQ active. When triggered, the interrupt controller wakes the CPU, which runs the resume path.

---

*Next: [Chapter 21 — Driver Communication with User Space](Chapter_21_Userspace_Communication.md)*
