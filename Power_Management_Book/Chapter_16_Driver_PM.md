# Chapter 16: Power Management in Device Drivers

## Learning Goals
- Understand how to implement PM in Linux device drivers
- Learn system sleep callbacks vs runtime PM callbacks
- Know PM patterns for different bus types (platform, PCI, I2C, SPI, USB)
- Understand DMA and interrupt handling during PM transitions
- Learn power-aware driver design principles

---

## 1. Driver PM Callback Structure

```
  Driver PM Integration Points
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  struct dev_pm_ops {                                     │
  │                                                           │
  │    /* ─── System Sleep ─── */                            │
  │    .prepare            After children suspended          │
  │    .suspend            Save state, stop I/O              │
  │    .suspend_late       After IRQ-enabled phase           │
  │    .suspend_noirq      IRQs disabled, final HW access   │
  │    .resume_noirq       First resume, IRQs disabled      │
  │    .resume_early       Before IRQ-enabled phase          │
  │    .resume             Restore state, restart I/O        │
  │    .complete           After all devices resumed         │
  │                                                           │
  │    /* ─── Runtime PM ─── */                              │
  │    .runtime_suspend    Device idle → power down          │
  │    .runtime_resume     Device needed → power up          │
  │    .runtime_idle       Check if device can suspend       │
  │                                                           │
  │    /* ─── Hibernate ─── */                               │
  │    .freeze / .thaw     Snapshot save/restore             │
  │    .poweroff / .restore System power off/on              │
  │  };                                                       │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Complete PM Driver Example

```c
/* Full driver with system sleep + runtime PM */
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/pm.h>
#include <linux/pm_runtime.h>
#include <linux/clk.h>
#include <linux/regulator/consumer.h>
#include <linux/io.h>

struct my_device {
    void __iomem *base;
    struct clk *clk;
    struct regulator *vdd;
    u32 saved_regs[16];
    bool powered;
};

/* ── Runtime PM callbacks ────────────────────────── */

static int my_runtime_suspend(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);

    /* 1. Stop DMA if active */
    writel(0, mydev->base + DMA_CTRL);

    /* 2. Save critical registers */
    for (int i = 0; i < 16; i++)
        mydev->saved_regs[i] = readl(mydev->base + i * 4);

    /* 3. Disable clock */
    clk_disable_unprepare(mydev->clk);

    /* 4. Disable power supply */
    regulator_disable(mydev->vdd);

    mydev->powered = false;
    return 0;
}

static int my_runtime_resume(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);
    int ret;

    /* 1. Enable power supply */
    ret = regulator_enable(mydev->vdd);
    if (ret)
        return ret;

    /* 2. Wait for voltage stabilization */
    usleep_range(100, 200);

    /* 3. Enable clock */
    ret = clk_prepare_enable(mydev->clk);
    if (ret) {
        regulator_disable(mydev->vdd);
        return ret;
    }

    /* 4. Restore registers */
    for (int i = 0; i < 16; i++)
        writel(mydev->saved_regs[i], mydev->base + i * 4);

    mydev->powered = true;
    return 0;
}

/* ── System sleep callbacks ───────────────────────── */

static int my_suspend(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);

    /* If runtime PM already suspended, nothing to do */
    if (pm_runtime_suspended(dev))
        return 0;

    /* Otherwise, do the same as runtime suspend */
    return my_runtime_suspend(dev);
}

static int my_resume(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);

    /* If was runtime-suspended before system sleep,
     * leave it suspended — runtime PM will resume on demand */
    if (pm_runtime_suspended(dev))
        return 0;

    return my_runtime_resume(dev);
}

/* ── PM ops definition ────────────────────────────── */

static const struct dev_pm_ops my_pm_ops = {
    SET_SYSTEM_SLEEP_PM_OPS(my_suspend, my_resume)
    SET_RUNTIME_PM_OPS(my_runtime_suspend,
                        my_runtime_resume, NULL)
};

/* ── Probe / Remove ───────────────────────────────── */

static int my_probe(struct platform_device *pdev)
{
    struct my_device *mydev;

    mydev = devm_kzalloc(&pdev->dev, sizeof(*mydev), GFP_KERNEL);

    mydev->base = devm_platform_ioremap_resource(pdev, 0);
    mydev->clk = devm_clk_get(&pdev->dev, NULL);
    mydev->vdd = devm_regulator_get(&pdev->dev, "vdd");

    clk_prepare_enable(mydev->clk);
    regulator_enable(mydev->vdd);
    mydev->powered = true;

    platform_set_drvdata(pdev, mydev);

    /* Enable runtime PM with autosuspend */
    pm_runtime_set_active(&pdev->dev);
    pm_runtime_set_autosuspend_delay(&pdev->dev, 200); /* 200ms */
    pm_runtime_use_autosuspend(&pdev->dev);
    pm_runtime_enable(&pdev->dev);

    return 0;
}

static void my_remove(struct platform_device *pdev)
{
    pm_runtime_disable(&pdev->dev);
    /* devm handles cleanup */
}

static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "my-device",
        .pm   = &my_pm_ops,
    },
};
```

---

## 3. Bus-Specific PM Patterns

### 3.1 PCI Driver PM

```c
/* PCI driver with PM */
static int my_pci_suspend(struct device *dev)
{
    struct pci_dev *pdev = to_pci_dev(dev);
    struct my_pci_dev *priv = pci_get_drvdata(pdev);

    /* Stop DMA engines */
    my_stop_dma(priv);

    /* Disable interrupts */
    my_disable_irqs(priv);

    /* Save device-specific state */
    my_save_state(priv);

    /* PCI core saves PCI config space and sets power state */
    return 0;
}

static int my_pci_resume(struct device *dev)
{
    struct pci_dev *pdev = to_pci_dev(dev);
    struct my_pci_dev *priv = pci_get_drvdata(pdev);

    /* PCI core restores config space */
    /* Reinitialize device */
    my_restore_state(priv);
    my_enable_irqs(priv);
    my_restart_dma(priv);

    return 0;
}

static DEFINE_SIMPLE_DEV_PM_OPS(my_pci_pm,
                                 my_pci_suspend,
                                 my_pci_resume);

static struct pci_driver my_pci_driver = {
    .name    = "my-pci-device",
    .id_table = my_pci_ids,
    .probe   = my_pci_probe,
    .remove  = my_pci_remove,
    .driver.pm = pm_sleep_ptr(&my_pci_pm),
};
```

### 3.2 I2C Driver PM

```c
/* I2C sensor with runtime PM */
static int sensor_runtime_suspend(struct device *dev)
{
    struct i2c_client *client = to_i2c_client(dev);

    /* Put sensor in low-power/standby mode via I2C register */
    i2c_smbus_write_byte_data(client, CTRL_REG,
                               POWER_DOWN_MODE);
    return 0;
}

static int sensor_runtime_resume(struct device *dev)
{
    struct i2c_client *client = to_i2c_client(dev);

    /* Wake sensor to active mode */
    i2c_smbus_write_byte_data(client, CTRL_REG,
                               ACTIVE_MODE);

    /* Wait for measurement to stabilize */
    msleep(10);
    return 0;
}

/* Reading sensor data with runtime PM */
static int sensor_read_temp(struct my_sensor *sensor)
{
    int ret, temp;

    /* Resume device before I2C access */
    ret = pm_runtime_resume_and_get(sensor->dev);
    if (ret)
        return ret;

    /* Read temperature */
    temp = i2c_smbus_read_word_data(sensor->client, TEMP_REG);

    /* Mark device for autosuspend */
    pm_runtime_mark_last_busy(sensor->dev);
    pm_runtime_put_autosuspend(sensor->dev);

    return temp;
}
```

---

## 4. DMA and Interrupt Handling During PM

```
  PM Transition: DMA & IRQ Timeline
  ┌──────────────────────────────────────────────────────┐
  │                                                       │
  │  SUSPEND path:                                       │
  │  .suspend()          .suspend_noirq()                │
  │  ───────────────────┼───────────────────             │
  │  IRQs enabled       │ IRQs disabled                  │
  │  DMA still possible │ No DMA, no IRQs               │
  │                      │                                │
  │  Driver must:       │ Driver must:                   │
  │  - Stop new DMA     │ - Final HW quiesce             │
  │  - Complete pending │ - Power gate if needed          │
  │  - Disable device   │                                 │
  │    IRQs             │                                 │
  │                                                       │
  │  RESUME path:                                        │
  │  .resume_noirq()    .resume()                        │
  │  ───────────────────┼───────────────────             │
  │  IRQs disabled      │ IRQs enabled                   │
  │  No DMA             │ DMA possible                   │
  │                      │                                │
  │  Driver must:       │ Driver must:                   │
  │  - Basic HW init    │ - Restore full state           │
  │  - Power ungate     │ - Enable device IRQs           │
  │                      │ - Restart DMA                  │
  └──────────────────────────────────────────────────────┘
```

```c
/* DMA-safe suspend */
static int my_dma_suspend(struct device *dev)
{
    struct my_dev *d = dev_get_drvdata(dev);

    /* Stop submitting new DMA requests */
    set_bit(SUSPENDED, &d->flags);

    /* Wait for in-flight DMA to complete */
    dmaengine_synchronize(d->dma_chan);

    /* Terminate any remaining transfers */
    dmaengine_terminate_sync(d->dma_chan);

    /* Now safe to power down */
    return 0;
}
```

---

## 5. Wakeup-Capable Drivers

```c
/* Driver that can wake system from sleep */
static int my_probe(struct platform_device *pdev)
{
    int irq = platform_get_irq(pdev, 0);

    /* Mark device as wakeup capable */
    device_init_wakeup(&pdev->dev, true);

    /* Request IRQ that can wake system */
    ret = devm_request_irq(&pdev->dev, irq, my_irq_handler,
                           IRQF_TRIGGER_FALLING, "my-wakeup",
                           mydev);
    return 0;
}

static int my_suspend(struct device *dev)
{
    struct my_dev *mydev = dev_get_drvdata(dev);

    if (device_may_wakeup(dev)) {
        /* Enable wake IRQ */
        enable_irq_wake(mydev->irq);
    } else {
        /* Disable IRQ if not waking */
        disable_irq(mydev->irq);
    }

    return 0;
}

static int my_resume(struct device *dev)
{
    struct my_dev *mydev = dev_get_drvdata(dev);

    if (device_may_wakeup(dev))
        disable_irq_wake(mydev->irq);
    else
        enable_irq(mydev->irq);

    return 0;
}
```

```bash
# Userspace wakeup control
echo enabled  > /sys/devices/.../power/wakeup  # Allow wake
echo disabled > /sys/devices/.../power/wakeup  # Don't wake
cat /sys/devices/.../power/wakeup_count         # Wake events
```

---

## 6. PM Convenience Macros

```c
/* Macro overview */

/* System sleep only (suspend/resume) */
DEFINE_SIMPLE_DEV_PM_OPS(name, suspend_fn, resume_fn);

/* System sleep with SET_ form */
SET_SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn)

/* Runtime PM only */
SET_RUNTIME_PM_OPS(suspend_fn, resume_fn, idle_fn)

/* Late suspend/resume */
SET_LATE_SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn)

/* noirq suspend/resume */
SET_NOIRQ_SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn)

/* Use pm_sleep_ptr() to compile-out PM when CONFIG_PM_SLEEP=n */
static struct platform_driver my_drv = {
    .driver = {
        .pm = pm_sleep_ptr(&my_pm_ops), /* NULL if PM disabled */
    },
};

/* Use pm_ptr() for both system and runtime PM */
.driver = {
    .pm = pm_ptr(&my_pm_ops),
};

/* EXPORT_GPL_DEV_PM_OPS for shared PM ops across modules */
EXPORT_GPL_DEV_PM_OPS(shared_pm_ops) = {
    SET_SYSTEM_SLEEP_PM_OPS(common_suspend, common_resume)
    SET_RUNTIME_PM_OPS(common_rt_suspend, common_rt_resume, NULL)
};
```

---

## 7. Common PM Pitfalls

```
  Common Driver PM Mistakes
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  1. Accessing hardware after power is removed            │
  │     BUG: readl(base + REG) after regulator_disable()     │
  │     FIX: Save state BEFORE disabling power               │
  │                                                           │
  │  2. Not synchronizing DMA before suspend                 │
  │     BUG: DMA writes to memory after system sleeps        │
  │     FIX: dmaengine_terminate_sync() in .suspend          │
  │                                                           │
  │  3. Wrong order of clock/regulator operations            │
  │     BUG: clk_enable before regulator_enable              │
  │     FIX: Power on → stabilize → clock on                 │
  │          Clock off → power off                            │
  │                                                           │
  │  4. Missing runtime PM get/put around register access    │
  │     BUG: Register read while device is runtime-suspended │
  │     FIX: pm_runtime_resume_and_get() before HW access    │
  │                                                           │
  │  5. Forgetting pm_runtime_set_active() before enable     │
  │     BUG: Device starts in "suspended" state with active  │
  │          hardware → state mismatch                        │
  │     FIX: Call pm_runtime_set_active() in probe            │
  │                                                           │
  │  6. Not handling runtime-suspended state in system sleep  │
  │     BUG: Double power-down (runtime + system)            │
  │     FIX: Check pm_runtime_suspended(dev) in .suspend     │
  └──────────────────────────────────────────────────────────┘
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `drivers/base/power/main.c` | System sleep device PM core |
| `drivers/base/power/runtime.c` | Runtime PM framework |
| `include/linux/pm.h` | PM ops structures and macros |
| `include/linux/pm_runtime.h` | Runtime PM API |
| `Documentation/power/devices.rst` | Driver PM documentation |
| `Documentation/power/runtime_pm.rst` | Runtime PM documentation |

---

## Interview Questions

**Q1: How should a driver handle system suspend when it is already runtime-suspended?**
**A:** The driver's `.suspend()` callback should check `pm_runtime_suspended(dev)`. If true, the device hardware is already powered down by runtime PM — the driver skips redundant power-down steps. On resume, if the device was runtime-suspended before sleep, `.resume()` should leave it suspended and return 0 — runtime PM will resume it on-demand when next used. This avoids unnecessary power-up/power-down cycles (the "direct_complete" optimization).

**Q2: What is the correct order for enabling clocks and regulators?**
**A:** Power-up: enable regulator → wait for voltage stabilization → enable/prepare clock → de-assert reset → access registers. Power-down: stop I/O → assert reset → disable/unprepare clock → disable regulator. The rationale: clock circuits need stable voltage to operate correctly, and registers need both power and clock to be accessible.

**Q3: How does pm_runtime_resume_and_get() differ from pm_runtime_get_sync()?**
**A:** Both increment the usage counter and resume the device synchronously. `pm_runtime_resume_and_get()` (newer API) decrements the counter on failure, so you don't need manual cleanup. `pm_runtime_get_sync()` (older API) leaves the counter incremented even on failure — the caller must call `pm_runtime_put_noidle()` on error. Always prefer `pm_runtime_resume_and_get()` in new code.

---

## Summary

- Drivers implement PM via `struct dev_pm_ops` with system sleep and runtime PM callbacks
- System sleep: suspend/resume saves/restores state around S3/S2idle transitions
- Runtime PM: individual devices power on/off based on usage (pm_runtime_get/put)
- Order matters: regulator → clock → register access (up), reverse (down)
- Check pm_runtime_suspended() in system sleep to avoid double operations
- DMA must be synchronized/terminated before suspend
- Wakeup-capable devices use enable_irq_wake() during suspend
- Use convenience macros (DEFINE_SIMPLE_DEV_PM_OPS, pm_ptr) for clean PM code

---

[Previous Chapter: Thermal Management ←](Chapter_15_Thermal_Management.md) | [Next Chapter: Power Management and Device Tree →](Chapter_17_Device_Tree_PM.md)
