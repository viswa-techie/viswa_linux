# Chapter 10: Runtime Power Management

## Learning Goals
- Understand runtime PM concept and its difference from system PM
- Learn the runtime PM state machine and transitions
- Know the runtime PM API and how to use it in drivers
- Understand autosuspend and usage counting
- Learn runtime PM debugging techniques

---

## 1. Runtime PM Concept

```
  System PM vs Runtime PM
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  System PM (suspend/hibernate):                          │
  │  - ALL devices transition together                        │
  │  - Triggered by user action (close lid, systemctl)       │
  │  - System-wide state change                               │
  │                                                           │
  │  Runtime PM:                                              │
  │  - INDIVIDUAL devices suspend/resume independently        │
  │  - Triggered automatically when device is idle            │
  │  - System stays in S0 (working state)                     │
  │  - Much more frequent transitions                         │
  │                                                           │
  │  Example: Display controller                              │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │  Time ──────────────────────────────────────►   │     │
  │  │  User activity:  ████░░░░░░░░████░░░░████       │     │
  │  │  Display HW:     ON  SUSPENDED ON  SUSP ON      │     │
  │  │                                                  │     │
  │  │  Display suspends/resumes many times without     │     │
  │  │  the system ever entering S3/S4                  │     │
  │  └─────────────────────────────────────────────────┘     │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Runtime PM State Machine

```
  Runtime PM States and Transitions
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │          pm_runtime_resume()                              │
  │       ┌──────────────────────────────────┐               │
  │       │                                  │               │
  │       ▼                                  │               │
  │  ┌──────────┐    pm_runtime_suspend()   ┌┴─────────┐    │
  │  │  ACTIVE  │ ─────────────────────────►│SUSPENDED │    │
  │  │          │                            │          │    │
  │  │ RPM_ACTIVE│◄─────────────────────────│RPM_SUSP  │    │
  │  └────┬─────┘    pm_runtime_resume()    └──────────┘    │
  │       │                                                   │
  │       │ pm_runtime_suspend()                              │
  │       │ (transition in progress)                          │
  │       ▼                                                   │
  │  ┌──────────┐                                            │
  │  │SUSPENDING│  (callback running)                        │
  │  │          │                                            │
  │  └──────────┘                                            │
  │                                                           │
  │  States:                                                 │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │ RPM_ACTIVE    — Device powered and operational    │    │
  │  │ RPM_SUSPENDED — Device in low-power state         │    │
  │  │ RPM_SUSPENDING — Suspend callback executing       │    │
  │  │ RPM_RESUMING  — Resume callback executing         │    │
  │  └──────────────────────────────────────────────────┘    │
  │                                                           │
  │  /sys/devices/.../power/runtime_status                   │
  │  Shows: "active", "suspended", "suspending", "resuming"  │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Runtime PM API

### 3.1 Core API

```c
/* include/linux/pm_runtime.h */

/* === Initialization === */
pm_runtime_enable(dev);     /* Enable runtime PM for device */
pm_runtime_disable(dev);    /* Disable (block transitions) */
pm_runtime_set_active(dev); /* Set initial state to active */
pm_runtime_set_suspended(dev); /* Set initial state to suspended */

/* === Usage Counting === */
pm_runtime_get(dev);        /* Increment usage, async resume */
pm_runtime_get_sync(dev);   /* Increment usage, sync resume */
pm_runtime_put(dev);        /* Decrement usage, async suspend */
pm_runtime_put_sync(dev);   /* Decrement usage, sync suspend */
pm_runtime_put_autosuspend(dev); /* Decrement, delayed suspend */

/* === Direct Control === */
pm_runtime_suspend(dev);    /* Request suspend (if usage == 0) */
pm_runtime_resume(dev);     /* Request resume */
pm_runtime_idle(dev);       /* Check if should suspend */

/* === Autosuspend === */
pm_runtime_use_autosuspend(dev);       /* Enable autosuspend */
pm_runtime_set_autosuspend_delay(dev, ms); /* Set delay */
pm_runtime_mark_last_busy(dev);        /* Reset autosuspend timer */

/* === Status Checks === */
pm_runtime_active(dev);     /* Is device active? */
pm_runtime_suspended(dev);  /* Is device suspended? */
pm_runtime_enabled(dev);    /* Is runtime PM enabled? */
```

### 3.2 Usage Count Model

```
  Runtime PM Usage Count
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  usage_count: reference counter for device activity      │
  │                                                           │
  │  pm_runtime_get_sync(dev);   → count++ → resume if was 0│
  │  pm_runtime_put_autosuspend(dev); → count-- → suspend    │
  │                                    if count reaches 0     │
  │                                                           │
  │  Example: Multiple users of a device                     │
  │                                                           │
  │  Thread A: get_sync() → count=1 (device resumes)        │
  │  Thread B: get_sync() → count=2 (already active, quick) │
  │  Thread A: put()      → count=1 (still active)          │
  │  Thread B: put()      → count=0 → suspend callback      │
  │                                                           │
  │  RULE: Every get() MUST have a matching put()            │
  │  Forgetting put() = device never suspends (power leak)   │
  │  Double put() = BUG, negative count                      │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Complete Driver Example

```c
/* Complete runtime PM implementation for a platform device */

#include <linux/pm_runtime.h>
#include <linux/platform_device.h>
#include <linux/clk.h>

struct my_device {
    void __iomem *regs;
    struct clk *clk;
    int irq;
};

/* Runtime suspend — called when device is idle */
static int my_runtime_suspend(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);

    /* Disable device clocks to save power */
    clk_disable_unprepare(mydev->clk);

    return 0;
}

/* Runtime resume — called when device is needed */
static int my_runtime_resume(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);

    /* Re-enable device clocks */
    return clk_prepare_enable(mydev->clk);
}

/* Runtime idle — check if device should suspend */
static int my_runtime_idle(struct device *dev)
{
    /* Return 0 to allow suspend, or -EBUSY to prevent */
    /* Optional: can call pm_runtime_autosuspend(dev) here */
    return 0;
}

static const struct dev_pm_ops my_pm_ops = {
    SET_RUNTIME_PM_OPS(my_runtime_suspend,
                       my_runtime_resume,
                       my_runtime_idle)
    SET_SYSTEM_SLEEP_PM_OPS(pm_runtime_force_suspend,
                            pm_runtime_force_resume)
};

/* Probe — initialize runtime PM */
static int my_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct my_device *mydev;

    mydev = devm_kzalloc(dev, sizeof(*mydev), GFP_KERNEL);
    platform_set_drvdata(pdev, mydev);

    /* ... hardware init ... */

    /* Configure runtime PM */
    pm_runtime_set_autosuspend_delay(dev, 200); /* 200ms delay */
    pm_runtime_use_autosuspend(dev);
    pm_runtime_set_active(dev);   /* Device starts active */
    pm_runtime_enable(dev);       /* Enable runtime PM */

    return 0;
}

/* Remove — cleanup runtime PM */
static int my_remove(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;

    pm_runtime_disable(dev);
    pm_runtime_set_suspended(dev);

    return 0;
}

/* Using the device — get/put around accesses */
static int my_do_work(struct my_device *mydev)
{
    struct device *dev = mydev->dev;
    int ret;

    /* Resume device if suspended */
    ret = pm_runtime_get_sync(dev);
    if (ret < 0) {
        pm_runtime_put_noidle(dev);
        return ret;
    }

    /* Device is now active — access hardware */
    writel(DATA, mydev->regs + DATA_REG);
    result = readl(mydev->regs + RESULT_REG);

    /* Mark busy (reset autosuspend timer) */
    pm_runtime_mark_last_busy(dev);
    /* Allow device to suspend after delay */
    pm_runtime_put_autosuspend(dev);

    return 0;
}
```

---

## 5. Autosuspend Mechanism

```
  Autosuspend Timeline
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Time ──────────────────────────────────────────────►    │
  │                                                           │
  │  get_sync()  put_autosuspend()                           │
  │     │              │                                      │
  │     ▼              ▼                                      │
  │  ██████████████████░░░░░░░░░░│                           │
  │  ^-- device active --^      ^-- autosuspend_delay --^    │
  │                              ^-- suspend callback        │
  │                                                           │
  │  With mark_last_busy():                                  │
  │  get  work  mark_busy  put_auto                          │
  │   │    │       │          │                               │
  │   ▼    ▼       ▼          ▼                               │
  │  ████████████████████████░░░░░░░░░░│                     │
  │                         ^-- timer restarted --^          │
  │                                                           │
  │  Multiple accesses extend active period:                 │
  │  get  put  get  mark_busy  put_auto                      │
  │   │    │    │      │          │                           │
  │   ▼    ▼    ▼      ▼          ▼                           │
  │  ██████████████████████████████░░░░░░░░│                 │
  │  count: 1   0   1             0   ──► suspend            │
  │  state: active   resumed     active    suspended         │
  │                                                           │
  │  Typical delays:                                         │
  │  I2C controller:    200ms                                │
  │  USB device:        2000ms                               │
  │  Display:           5000ms                               │
  │  Network card:      1000ms                               │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. Parent-Child Runtime PM

```
  Parent-Child Runtime PM Coordination
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌── I2C Controller (parent) ──────────────────────┐    │
  │  │  child_count tracks active children              │    │
  │  │                                                  │    │
  │  │  ┌── Sensor (child 1) ──┐  ┌── PMIC (child 2)─┐│    │
  │  │  │  get_sync:           │  │  get_sync:         ││    │
  │  │  │  1. Resume parent    │  │  1. Resume parent  ││    │
  │  │  │  2. Resume self      │  │  2. Resume self    ││    │
  │  │  │                      │  │                    ││    │
  │  │  │  put:                │  │  put:              ││    │
  │  │  │  1. Suspend self     │  │  1. Suspend self   ││    │
  │  │  │  2. Decrement parent │  │  2. Decrement     ││    │
  │  │  │     child_count      │  │     parent count   ││    │
  │  │  └──────────────────────┘  └────────────────────┘│    │
  │  │                                                  │    │
  │  │  Parent suspends only when child_count == 0      │    │
  │  │  Parent auto-resumes when any child needs it     │    │
  │  └──────────────────────────────────────────────────┘    │
  │                                                           │
  │  Automatic: pm_runtime_get_sync(child) automatically     │
  │  resumes the parent first. No explicit parent management  │
  │  needed in child driver.                                  │
  └──────────────────────────────────────────────────────────┘
```

```c
/* Parent auto-resume is handled by the framework */
/* In drivers/base/power/runtime.c: */

static int rpm_resume(struct device *dev, int rpmflags)
{
    /* If device has a parent that is suspended, resume it first */
    if (dev->parent) {
        spin_unlock(&dev->power.lock);
        pm_runtime_get_noresume(dev->parent);

        /* Recursively resume parent */
        retval = rpm_resume(dev->parent, 0);
        if (retval)
            goto out;
    }

    /* Now resume this device */
    callback = RPM_GET_CALLBACK(dev, runtime_resume);
    retval = callback(dev);

    return retval;
}
```

---

## 7. Runtime PM and System Suspend Integration

```c
/* When system suspends (S3), runtime PM state must be coordinated */

/* Option 1: Use pm_runtime_force_suspend/resume (recommended) */
/* These handle the runtime PM ↔ system PM transition automatically */
static const struct dev_pm_ops my_pm_ops = {
    SET_RUNTIME_PM_OPS(my_rt_suspend, my_rt_resume, NULL)
    SET_SYSTEM_SLEEP_PM_OPS(pm_runtime_force_suspend,
                            pm_runtime_force_resume)
};

/* pm_runtime_force_suspend:
 * 1. If device is runtime-active → calls runtime_suspend
 * 2. If device is already runtime-suspended → no-op
 * 3. Disables runtime PM during system sleep
 *
 * pm_runtime_force_resume:
 * 1. Calls runtime_resume to restore device
 * 2. Re-enables runtime PM
 * 3. If device was runtime-suspended before system sleep,
 *    leaves it suspended (no unnecessary resume)
 */

/* Option 2: Custom system PM that's aware of runtime state */
static int my_system_suspend(struct device *dev)
{
    /* Check runtime PM state before doing work */
    if (pm_runtime_suspended(dev)) {
        /* Already suspended by runtime PM — nothing to do */
        return 0;
    }

    /* Device is active — need to suspend it */
    return my_actual_suspend(dev);
}
```

---

## 8. Runtime PM Debugging

```bash
# View runtime PM status for all devices
find /sys/devices -name runtime_status -exec sh -c \
    'echo "$(cat "$1") $1"' _ {} \;

# View runtime PM statistics
cat /sys/devices/.../power/runtime_status           # Current state
cat /sys/devices/.../power/runtime_active_time      # Total active ms
cat /sys/devices/.../power/runtime_suspended_time   # Total suspended ms
cat /sys/devices/.../power/runtime_usage             # Usage count
cat /sys/devices/.../power/runtime_active_kids       # Active children

# Force device to stay active (disable runtime PM)
echo on > /sys/devices/.../power/control
# Allow runtime PM
echo auto > /sys/devices/.../power/control

# Autosuspend delay
cat /sys/devices/.../power/autosuspend_delay_ms
echo 5000 > /sys/devices/.../power/autosuspend_delay_ms

# Kernel tracing for runtime PM
echo 1 > /sys/kernel/debug/tracing/events/rpm/enable
cat /sys/kernel/debug/tracing/trace
# Shows: rpm_suspend, rpm_resume, rpm_idle events
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `drivers/base/power/runtime.c` | Runtime PM core implementation |
| `include/linux/pm_runtime.h` | Runtime PM API |
| `include/linux/pm.h` | PM ops structure |
| `drivers/base/power/sysfs.c` | Runtime PM sysfs attributes |
| `drivers/base/power/main.c` | System PM ↔ runtime PM coordination |
| `kernel/power/main.c` | PM core |
| `include/trace/events/rpm.h` | Runtime PM trace events |

---

## Interview Questions

**Q1: Explain the runtime PM usage count model.**
**A:** Runtime PM uses a reference counter (usage_count). pm_runtime_get_sync() increments the count and resumes the device if count goes from 0→1. pm_runtime_put() decrements the count and suspends the device if count reaches 0. Multiple users can hold references simultaneously — the device stays active until ALL users release. Every get() must have a matching put(). Forgetting put() causes a power leak; extra put() causes a bug.

**Q2: What is autosuspend and why is it useful?**
**A:** Autosuspend delays the runtime suspend by a configurable time (autosuspend_delay_ms) after the last put(). Instead of immediately suspending when usage reaches 0, the device waits — if it's needed again soon, it avoids the suspend/resume cycle overhead. pm_runtime_mark_last_busy() resets the timer. This is critical for bursty access patterns (e.g., I2C — many register reads in sequence, each with get/put).

**Q3: How does runtime PM handle parent-child relationships?**
**A:** When a child does pm_runtime_get_sync(), the framework automatically resumes the parent first (recursively up the tree). The parent maintains a child_count. When a child suspends, it decrements the parent's child_count. The parent can only suspend when child_count reaches 0. This ensures the parent (e.g., I2C controller) is always active when any child (e.g., sensor) needs it.

**Q4: How does runtime PM interact with system suspend?**
**A:** During system suspend, runtime PM must be coordinated. The recommended approach is pm_runtime_force_suspend/resume: these check if the device is already runtime-suspended (no-op) or runtime-active (calls runtime_suspend callback). During system sleep, runtime PM is disabled to prevent race conditions. On resume, pm_runtime_force_resume restores the correct state and re-enables runtime PM.

**Q5: How would you debug a runtime PM issue where a device never suspends?**
**A:** Check: (1) `runtime_status` — is it stuck in "active"? (2) `runtime_usage` — is the usage count > 0 (missing put)? (3) `control` — is it set to "on" (runtime PM disabled)? (4) `runtime_active_kids` — are children keeping parent active? (5) Enable RPM tracing: `echo 1 > /sys/kernel/debug/tracing/events/rpm/enable` — shows get/put calls with stack traces. (6) Check driver code for missing pm_runtime_put() in error paths.

---

## Summary

- Runtime PM suspends/resumes individual devices independently while the system stays in S0
- Three states: RPM_ACTIVE, RPM_SUSPENDED, and transitional (SUSPENDING/RESUMING)
- Usage count model: get() increments and resumes; put() decrements and may suspend
- Autosuspend adds a configurable delay before actual suspend — avoids thrashing
- Parent-child coordination is automatic: parent resumes before child, suspends after all children
- pm_runtime_force_suspend/resume bridges runtime PM and system PM
- Every pm_runtime_get() must have a matching pm_runtime_put() — leaks cause devices to stay powered
- Debug via sysfs (runtime_status, runtime_usage) and ftrace RPM events

---

[Previous Chapter: Device PM ←](Chapter_09_Device_PM.md) | [Next Chapter: System Suspend and Resume →](Chapter_11_Suspend_Resume.md)
