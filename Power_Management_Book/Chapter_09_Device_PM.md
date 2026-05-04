# Chapter 9: Device Power Management

## Learning Goals
- Understand the device PM framework and callback model
- Learn how bus-type PM operations work
- Know the suspend/resume device ordering mechanism
- Understand device PM states and transitions
- Learn to implement PM in device drivers

---

## 1. Device PM Framework Overview

```
  Device PM Framework
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Three levels of PM operations:                          │
  │                                                           │
  │  ┌─ Device Driver ─────────────────────────────────┐    │
  │  │  struct dev_pm_ops (driver-specific)              │    │
  │  │  - Knows device hardware details                  │    │
  │  │  - Saves/restores device-specific registers       │    │
  │  └──────────────────────────────────────────────────┘    │
  │                    ▼ (falls through if not set)           │
  │  ┌─ Bus/Class/Type ────────────────────────────────┐    │
  │  │  struct dev_pm_ops (bus-level)                    │    │
  │  │  - Generic PM for all devices on this bus         │    │
  │  │  - PCI: save config space, set D-state            │    │
  │  │  - USB: selective suspend, remote wakeup          │    │
  │  │  - Platform: generic DT-based PM                  │    │
  │  └──────────────────────────────────────────────────┘    │
  │                    ▼ (falls through if not set)           │
  │  ┌─ PM Core ───────────────────────────────────────┐    │
  │  │  Default behavior (drivers/base/power/main.c)     │    │
  │  │  - Device PM list ordering                        │    │
  │  │  - Async coordination                             │    │
  │  │  - Error handling                                 │    │
  │  └──────────────────────────────────────────────────┘    │
  │                                                           │
  │  Lookup priority: driver->pm > type->pm > class->pm >   │
  │                   bus->pm > no-op                         │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. PM Callbacks Deep Dive

### 2.1 System Suspend Callbacks

```c
/* Complete callback set for system suspend/resume */
static const struct dev_pm_ops my_dev_pm_ops = {
    /* --- System Suspend --- */
    .prepare        = my_prepare,       /* 1st: prevent new children */
    .suspend        = my_suspend,       /* 2nd: quiesce device */
    .suspend_late   = my_suspend_late,  /* 3rd: final HW access */
    .suspend_noirq  = my_suspend_noirq, /* 4th: IRQs disabled */

    /* --- System Resume --- */
    .resume_noirq   = my_resume_noirq,  /* 1st: IRQs disabled */
    .resume_early   = my_resume_early,  /* 2nd: early resume */
    .resume         = my_resume,        /* 3rd: full resume */
    .complete       = my_complete,      /* 4th: allow children */

    /* --- Hibernate --- */
    .freeze         = my_freeze,
    .thaw           = my_thaw,
    .poweroff       = my_poweroff,
    .restore        = my_restore,

    /* --- Runtime PM --- */
    .runtime_suspend = my_runtime_suspend,
    .runtime_resume  = my_runtime_resume,
    .runtime_idle    = my_runtime_idle,
};
```

### 2.2 Practical Suspend/Resume Implementation

```c
/* Typical device driver PM implementation */

static int my_device_suspend(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);

    /* 1. Stop accepting new work */
    mydev->suspended = true;

    /* 2. Wait for in-flight operations to complete */
    flush_workqueue(mydev->wq);

    /* 3. Disable interrupts from device */
    writel(0, mydev->regs + IRQ_ENABLE);

    /* 4. Save hardware state */
    mydev->saved_config = readl(mydev->regs + CONFIG_REG);
    mydev->saved_mode   = readl(mydev->regs + MODE_REG);

    /* 5. Put hardware in low-power state */
    writel(POWER_DOWN, mydev->regs + POWER_REG);

    /* 6. Disable clocks */
    clk_disable_unprepare(mydev->clk);

    return 0;
}

static int my_device_resume(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);

    /* 1. Re-enable clocks */
    clk_prepare_enable(mydev->clk);

    /* 2. Power up hardware */
    writel(POWER_UP, mydev->regs + POWER_REG);
    usleep_range(100, 200); /* Wait for power stabilization */

    /* 3. Restore hardware state */
    writel(mydev->saved_config, mydev->regs + CONFIG_REG);
    writel(mydev->saved_mode, mydev->regs + MODE_REG);

    /* 4. Re-enable interrupts */
    writel(IRQ_MASK_ALL, mydev->regs + IRQ_ENABLE);

    /* 5. Mark as resumed */
    mydev->suspended = false;

    return 0;
}
```

---

## 3. Device Suspend/Resume Ordering

```
  Device Tree PM Ordering
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Device hierarchy:                                       │
  │                                                           │
  │  platform_bus                                            │
  │  ├── soc                                                 │
  │  │   ├── i2c_controller                                 │
  │  │   │   ├── pmic (child of i2c)                        │
  │  │   │   └── touchscreen (child of i2c)                 │
  │  │   ├── spi_controller                                 │
  │  │   │   └── flash_chip (child of spi)                  │
  │  │   └── usb_controller                                 │
  │  │       └── usb_device                                 │
  │  └── pci_host                                            │
  │      ├── gpu                                             │
  │      └── nic                                             │
  │                                                           │
  │  SUSPEND order (children FIRST):                         │
  │  pmic → touchscreen → i2c_ctrl → flash → spi_ctrl →    │
  │  usb_device → usb_ctrl → gpu → nic → pci_host → soc    │
  │                                                           │
  │  RESUME order (parents FIRST):                           │
  │  soc → pci_host → nic → gpu → usb_ctrl → usb_device → │
  │  spi_ctrl → flash → i2c_ctrl → touchscreen → pmic       │
  │                                                           │
  │  Why? Parent must be active to communicate with children │
  │  e.g., I2C controller must be running to access PMIC     │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Bus-Specific PM

### 4.1 Platform Device PM

```c
/* Platform bus PM — most common on embedded/SoC systems */
static int platform_pm_suspend(struct device *dev)
{
    struct platform_driver *drv = to_platform_driver(dev->driver);
    int ret = 0;

    /* Priority: driver->pm > driver->suspend (legacy) */
    if (drv && drv->driver.pm && drv->driver.pm->suspend)
        ret = drv->driver.pm->suspend(dev);
    else if (drv && drv->suspend)
        ret = drv->suspend(to_platform_device(dev), PMSG_SUSPEND);

    return ret;
}
```

### 4.2 PCI PM Integration

```c
/* PCI devices get automatic config space save/restore */
static int pci_pm_suspend(struct device *dev)
{
    struct pci_dev *pci_dev = to_pci_dev(dev);
    const struct dev_pm_ops *pm = dev->driver ? dev->driver->pm : NULL;

    /* Driver's suspend callback */
    if (pm && pm->suspend)
        error = pm->suspend(dev);

    /* PCI core: save config space */
    if (!error)
        pci_save_state(pci_dev);

    return error;
}

static int pci_pm_resume(struct device *dev)
{
    struct pci_dev *pci_dev = to_pci_dev(dev);
    const struct dev_pm_ops *pm = dev->driver ? dev->driver->pm : NULL;

    /* PCI core: restore config space */
    pci_restore_state(pci_dev);

    /* Driver's resume callback */
    if (pm && pm->resume)
        error = pm->resume(dev);

    return error;
}
```

---

## 5. Convenience Macros

```c
/* include/linux/pm.h — Simplified PM ops declaration */

/* Use when runtime PM and system PM callbacks are the same */
#define SET_SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn) \
    .suspend = suspend_fn, \
    .resume = resume_fn, \
    .freeze = suspend_fn, \
    .thaw = resume_fn, \
    .poweroff = suspend_fn, \
    .restore = resume_fn

#define SET_RUNTIME_PM_OPS(suspend_fn, resume_fn, idle_fn) \
    .runtime_suspend = suspend_fn, \
    .runtime_resume = resume_fn, \
    .runtime_idle = idle_fn

/* Modern approach: DEFINE_SIMPLE_DEV_PM_OPS */
static DEFINE_SIMPLE_DEV_PM_OPS(my_pm_ops,
                                 my_suspend, my_resume);

/* Complete example */
static const struct dev_pm_ops my_pm_ops = {
    SET_SYSTEM_SLEEP_PM_OPS(my_suspend, my_resume)
    SET_RUNTIME_PM_OPS(my_rt_suspend, my_rt_resume, my_rt_idle)
};

/* Use SET_LATE variants for _late callbacks */
#define SET_LATE_SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn) \
    .suspend_late = suspend_fn, \
    .resume_early = resume_fn, \
    /* ... */

/* Use NOIRQ variants for interrupt-disabled callbacks */
#define SET_NOIRQ_SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn) \
    .suspend_noirq = suspend_fn, \
    .resume_noirq = resume_fn, \
    /* ... */
```

---

## 6. Wakeup-Capable Devices

### 6.1 Wakeup Configuration

```c
/* Making a device capable of waking the system */

static int my_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;

    /* Mark device as wakeup-capable */
    device_init_wakeup(dev, true);
    /* This creates /sys/devices/.../power/wakeup = "enabled" */

    /* Configure wakeup IRQ */
    dev_pm_set_wake_irq(dev, irq);
    /* Or: enable_irq_wake(irq); in suspend callback */

    return 0;
}

static int my_suspend(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);

    if (device_may_wakeup(dev)) {
        /* Keep wakeup IRQ active during suspend */
        enable_irq_wake(mydev->irq);
        /* Device stays partially powered for wake detection */
    } else {
        /* Full power down — no wakeup capability */
        disable_irq(mydev->irq);
        power_down_device(mydev);
    }

    return 0;
}

static int my_resume(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);

    if (device_may_wakeup(dev)) {
        disable_irq_wake(mydev->irq);
    }

    /* Check if this device triggered the wakeup */
    if (mydev->irq_pending)
        pm_wakeup_event(dev, 0);

    return 0;
}
```

### 6.2 Userspace Wakeup Control

```bash
# View wakeup-capable devices
grep -r . /sys/devices/*/power/wakeup 2>/dev/null | grep enabled

# Disable wakeup from specific device
echo disabled > /sys/devices/.../power/wakeup

# View wakeup statistics
cat /sys/devices/.../power/wakeup_count        # Total wakeup events
cat /sys/devices/.../power/wakeup_active_count  # Currently active
cat /sys/devices/.../power/wakeup_last_time_ms  # Time of last wakeup
```

---

## 7. DPM (Device Power Management) Internals

### 7.1 DPM List Management

```c
/* drivers/base/power/main.c — DPM list operations */

/* Device is added to dpm_list during device_add() */
void device_pm_add(struct device *dev)
{
    /* Add to tail of dpm_list */
    list_add_tail(&dev->power.entry, &dpm_list);
}

/* System suspend traverses list in reverse (children first) */
int dpm_suspend(pm_message_t state)
{
    while (!list_empty(&dpm_prepared_list)) {
        struct device *dev = to_device(dpm_prepared_list.prev);
        /* ^^^ .prev = reverse order = children before parents */

        error = device_suspend(dev);
        if (error) {
            /* Move failed device aside, abort remaining */
            break;
        }
        /* Move to suspended list */
        list_move(&dev->power.entry, &dpm_suspended_list);
    }
    return error;
}

/* Resume traverses list forward (parents first) */
void dpm_resume(pm_message_t state)
{
    while (!list_empty(&dpm_suspended_list)) {
        struct device *dev = to_device(dpm_suspended_list.next);
        /* ^^^ .next = forward order = parents before children */

        device_resume(dev, state, false);
    }
}
```

---

## 8. PM for Different Device Types

```
  PM Requirements by Device Type
  ┌──────────────┬───────────────────────────────────────────┐
  │ Device Type  │ PM Considerations                          │
  ├──────────────┼───────────────────────────────────────────┤
  │ Network      │ Save MAC config, disable DMA              │
  │  (NIC)       │ Wake-on-LAN if wakeup enabled             │
  │              │ Quiesce TX/RX queues                       │
  ├──────────────┼───────────────────────────────────────────┤
  │ Storage      │ Flush caches/buffers                       │
  │  (eMMC/NVMe) │ Complete in-flight I/O                     │
  │              │ Enter low-power mode                       │
  ├──────────────┼───────────────────────────────────────────┤
  │ Display      │ Blank screen, disable CRTC/encoder        │
  │  (DRM/KMS)   │ Stop scanout, disable PLLs                │
  │              │ Keep EDID for fast resume                  │
  ├──────────────┼───────────────────────────────────────────┤
  │ Audio        │ Drain buffers, stop DMA                    │
  │  (ALSA/ASoC) │ Power down codec, disable DAI             │
  │              │ Save mixer settings                        │
  ├──────────────┼───────────────────────────────────────────┤
  │ Input        │ Disable scanning (touchscreen)             │
  │              │ Keep wakeup source (power button)          │
  ├──────────────┼───────────────────────────────────────────┤
  │ I2C/SPI      │ Complete pending transfers                 │
  │ Controller   │ Disable controller, gate clock             │
  │              │ Restore bus speed on resume                │
  └──────────────┴───────────────────────────────────────────┘
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `drivers/base/power/main.c` | DPM list, suspend/resume orchestration |
| `drivers/base/power/wakeup.c` | Wakeup source framework |
| `drivers/base/power/sysfs.c` | Device PM sysfs attributes |
| `include/linux/pm.h` | dev_pm_ops, PM macros |
| `include/linux/pm_wakeup.h` | Wakeup API |
| `include/linux/device.h` | struct device PM fields |
| `drivers/pci/pci-driver.c` | PCI bus PM operations |
| `drivers/usb/core/driver.c` | USB bus PM operations |
| `drivers/base/platform.c` | Platform bus PM |

---

## Interview Questions

**Q1: Explain the PM callback invocation priority.**
**A:** When the PM core needs to call a suspend/resume callback, it checks in this priority order: (1) device driver's dev_pm_ops, (2) device type's dev_pm_ops, (3) device class's dev_pm_ops, (4) bus type's dev_pm_ops. The first non-NULL callback found is used. For example, a PCI network driver provides its own .suspend to save NIC-specific state, but PCI bus PM automatically saves/restores the PCI config space regardless.

**Q2: Why are devices suspended in reverse registration order?**
**A:** Children are registered after their parents (parent bus must exist to register child devices). Reverse order means children suspend before parents, which is necessary because: (1) children need the parent's bus to communicate during their suspend (e.g., I2C device needs I2C controller), (2) parent can then safely power down the bus. On resume, forward order ensures parents are active before children attempt communication.

**Q3: What is the difference between suspend, suspend_late, and suspend_noirq?**
**A:** `suspend()` runs with IRQs enabled and scheduling allowed — for saving state and quiescing the device. `suspend_late()` runs with IRQs enabled but scheduling NOT allowed — for final cleanup that must happen after all regular suspends. `suspend_noirq()` runs with IRQs disabled — for hardware operations that must not be interrupted (e.g., programming power-down registers). The split prevents races between devices during the transition.

**Q4: How does a driver implement wakeup capability?**
**A:** (1) Call device_init_wakeup(dev, true) in probe to mark the device as wakeup-capable. (2) In suspend callback, if device_may_wakeup(dev), call enable_irq_wake(irq) to keep the wakeup IRQ active during system sleep. (3) In the IRQ handler during sleep, call pm_wakeup_event(dev, timeout) to signal a wakeup. (4) In resume, call disable_irq_wake(irq). Userspace can enable/disable wakeup via `/sys/devices/.../power/wakeup`.

**Q5: What convenience macros are available for PM ops?**
**A:** `SET_SYSTEM_SLEEP_PM_OPS(suspend, resume)` sets suspend/resume/freeze/thaw/poweroff/restore all to the same pair of functions. `SET_RUNTIME_PM_OPS(suspend, resume, idle)` sets runtime PM callbacks. `DEFINE_SIMPLE_DEV_PM_OPS(name, suspend, resume)` creates a complete pm_ops struct using both. These reduce boilerplate when system PM and hibernate use the same logic, which is the common case.

---

## Summary

- Device PM has three callback levels: driver → type/class → bus, checked in priority order
- 8 suspend/resume callbacks per device: prepare, suspend, suspend_late, suspend_noirq (and reverse)
- Devices suspend in reverse registration order (children first), resume in forward order (parents first)
- Bus PM layers add bus-specific behavior (PCI saves config space, USB handles selective suspend)
- Wakeup-capable devices can trigger system wakeup via enable_irq_wake() and pm_wakeup_event()
- Convenience macros (SET_SYSTEM_SLEEP_PM_OPS, DEFINE_SIMPLE_DEV_PM_OPS) reduce boilerplate
- Async suspend allows parallel device transitions for faster suspend/resume

---

[Previous Chapter: CPUIdle ←](Chapter_08_CPUIdle.md) | [Next Chapter: Runtime Power Management →](Chapter_10_Runtime_PM.md)
