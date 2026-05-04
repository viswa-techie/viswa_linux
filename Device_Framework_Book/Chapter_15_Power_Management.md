# Chapter 15: Power Management Framework

## Learning Goals
- Understand Linux power management architecture (suspend/resume, runtime PM)
- Know system sleep states (S0, S1, S2, S3, S4) and driver PM callbacks
- Grasp Runtime PM for dynamic device power management
- Implement suspend/resume and Runtime PM in a device driver

---

## 15.1 Power Management Architecture

```
Linux Power Management:

Two PM Models:
┌─────────────────────────────────┬─────────────────────────────────┐
│  System Sleep (Suspend/Resume)  │  Runtime PM (per-device)        │
├─────────────────────────────────┼─────────────────────────────────┤
│ ● System-wide transition       │ ● Individual device power       │
│ ● Triggered by user/policy     │ ● Automatic when device idle    │
│ ● All devices suspended        │ ● Only idle devices suspended   │
│ ● S3 (suspend-to-RAM)          │ ● System stays running          │
│ ● S4 (suspend-to-disk/hibernate│ ● Critical for mobile/embedded  │
│ ● echo mem > /sys/power/state  │ ● pm_runtime_get/put_sync()     │
└─────────────────────────────────┴─────────────────────────────────┘

System Sleep Flow:
  User: echo mem > /sys/power/state
         │
         ▼
  ┌──────────────────────────────────────────────┐
  │ PM Core (kernel/power/)                       │
  │                                               │
  │ 1. Freeze user-space processes                │
  │ 2. Call device .suspend() (leaf → root)       │
  │    ├── Driver saves HW state                  │
  │    ├── Disables interrupts                    │
  │    └── Stops DMA                              │
  │ 3. Disable non-boot CPUs                      │
  │ 4. Platform: enter low-power state (S3/S4)    │
  │                                               │
  │ --- System sleeping (RAM self-refresh) ---    │
  │                                               │
  │ 5. Wake event (button, RTC, network)          │
  │ 6. Platform: exit low-power state             │
  │ 7. Re-enable CPUs                             │
  │ 8. Call device .resume() (root → leaf)        │
  │    ├── Driver restores HW state               │
  │    ├── Re-enables interrupts                  │
  │    └── Restarts DMA                           │
  │ 9. Thaw user-space processes                  │
  └──────────────────────────────────────────────┘
```

---

## 15.2 Driver PM Callbacks

```c
/* System sleep: suspend/resume callbacks */

static int my_driver_suspend(struct device *dev)
{
    struct my_device *priv = dev_get_drvdata(dev);

    /* Save hardware state */
    priv->saved_ctrl = readl(priv->regs + CTRL_REG);
    priv->saved_config = readl(priv->regs + CONFIG_REG);

    /* Stop ongoing operations */
    my_stop_dma(priv);
    my_disable_irq(priv);

    /* Disable clocks */
    clk_disable_unprepare(priv->clk);

    return 0;
}

static int my_driver_resume(struct device *dev)
{
    struct my_device *priv = dev_get_drvdata(dev);

    /* Re-enable clocks */
    clk_prepare_enable(priv->clk);

    /* Restore hardware state */
    writel(priv->saved_ctrl, priv->regs + CTRL_REG);
    writel(priv->saved_config, priv->regs + CONFIG_REG);

    /* Re-enable operations */
    my_enable_irq(priv);
    my_start_dma(priv);

    return 0;
}

/* PM operations structure */
static const struct dev_pm_ops my_pm_ops = {
    SET_SYSTEM_SLEEP_PM_OPS(my_driver_suspend, my_driver_resume)
    SET_RUNTIME_PM_OPS(my_runtime_suspend, my_runtime_resume, NULL)
};

static struct platform_driver my_driver = {
    .driver = {
        .name = "my-device",
        .pm   = &my_pm_ops,
    },
    .probe  = my_probe,
    .remove = my_remove,
};
```

---

## 15.3 Runtime PM

```
Runtime PM — Dynamic Per-Device Power Management:

  ┌──────────────────────────────────────────────┐
  │  Device Active (RPM_ACTIVE)                   │
  │  ├── Clocks enabled, power on                 │
  │  ├── Processing requests                      │
  │  └── pm_runtime_get_sync() called by user     │
  └──────────────────────┬───────────────────────┘
                         │ pm_runtime_put_autosuspend()
                         │ (start autosuspend timer)
                         ▼
  ┌──────────────────────────────────────────────┐
  │  Autosuspend Delay Timer Running              │
  │  ├── Device still powered (optimistic)       │
  │  ├── If new request arrives: cancel timer     │
  │  └── If timer expires → runtime_suspend()    │
  └──────────────────────┬───────────────────────┘
                         │ timer expired
                         ▼
  ┌──────────────────────────────────────────────┐
  │  Device Suspended (RPM_SUSPENDED)             │
  │  ├── runtime_suspend() callback called        │
  │  ├── Clocks gated, power domain off           │
  │  ├── State saved to RAM                       │
  │  └── Zero power consumption                   │
  └──────────────────────┬───────────────────────┘
                         │ pm_runtime_get_sync()
                         │ (new request arrives)
                         ▼
  ┌──────────────────────────────────────────────┐
  │  Device Active again                          │
  │  ├── runtime_resume() callback called         │
  │  ├── Clocks re-enabled, state restored        │
  │  └── Ready to process request                 │
  └──────────────────────────────────────────────┘
```

```c
/* Runtime PM implementation */

static int my_runtime_suspend(struct device *dev)
{
    struct my_device *priv = dev_get_drvdata(dev);

    clk_disable_unprepare(priv->clk);
    /* Optionally: disable regulator, power domain */

    return 0;
}

static int my_runtime_resume(struct device *dev)
{
    struct my_device *priv = dev_get_drvdata(dev);

    clk_prepare_enable(priv->clk);
    /* Optionally: re-enable regulator */

    return 0;
}

static int my_probe(struct platform_device *pdev)
{
    /* Enable Runtime PM */
    pm_runtime_enable(&pdev->dev);

    /* Set autosuspend delay (ms) */
    pm_runtime_set_autosuspend_delay(&pdev->dev, 200);
    pm_runtime_use_autosuspend(&pdev->dev);

    /* Initial resume (power up) */
    pm_runtime_get_sync(&pdev->dev);

    /* ... initialize hardware ... */

    /* Allow suspend when idle */
    pm_runtime_put_autosuspend(&pdev->dev);

    return 0;
}

/* In a function that accesses hardware */
static int my_do_something(struct my_device *priv)
{
    /* Ensure device is powered before HW access */
    pm_runtime_get_sync(priv->dev);

    /* Access hardware registers */
    writel(val, priv->regs + REG);
    result = readl(priv->regs + STATUS);

    /* Done with hardware — allow suspend after delay */
    pm_runtime_mark_last_busy(priv->dev);
    pm_runtime_put_autosuspend(priv->dev);

    return result;
}

static int my_remove(struct platform_device *pdev)
{
    pm_runtime_disable(&pdev->dev);
    return 0;
}
```

---

## 15.4 Wakeup Sources

```c
/* Wakeup source — device that can wake system from sleep */

static int my_probe(struct platform_device *pdev)
{
    /* Register device as wakeup source */
    device_init_wakeup(&pdev->dev, true);

    /* In suspend: configure IRQ as wakeup */
    enable_irq_wake(priv->irq);
}

static int my_suspend(struct device *dev)
{
    struct my_device *priv = dev_get_drvdata(dev);

    if (device_may_wakeup(dev)) {
        enable_irq_wake(priv->irq);
    } else {
        disable_irq(priv->irq);
    }
    return 0;
}
```

```dts
/* Wakeup in device tree */
power-button {
    compatible = "gpio-keys";
    wakeup-source;  /* can wake from sleep */

    button {
        gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;
        linux,code = <KEY_POWER>;
        wakeup-source;
    };
};
```

---

## 15.5 PM Domains (genpd)

```
Generic Power Domains (genpd):

SoC Power Domains:
┌─────────────────────────────────────────────────┐
│  Always-On Domain                                │
│  ├── Interrupt Controller                        │
│  ├── Timer                                       │
│  └── PMIC I2C                                    │
├─────────────────────────────────────────────────┤
│  Display Power Domain (can be OFF)               │
│  ├── Display controller                          │
│  ├── DSI/HDMI encoder                            │
│  └── GPU                                         │    OFF when
├─────────────────────────────────────────────────┤    display
│  Camera Power Domain (can be OFF)                │    unused
│  ├── ISP                                         │
│  ├── CSI receiver                                │
│  └── Camera sensor power                         │
├─────────────────────────────────────────────────┤
│  Modem Power Domain (can be OFF)                 │
│  ├── Modem processor                             │
│  └── RF frontend                                 │
└─────────────────────────────────────────────────┘

genpd tracks active devices per domain:
  - If ALL devices in domain are runtime-suspended → power off domain
  - If ANY device needs to resume → power on domain first
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| PM core | kernel/power/ | System sleep |
| Runtime PM | drivers/base/power/runtime.c | Per-device runtime PM |
| PM domain | drivers/base/power/domain.c | Generic power domains |
| PM ops | include/linux/pm.h | PM callback structures |
| Runtime PM API | include/linux/pm_runtime.h | Runtime PM functions |

---

## Interview Questions

**Q1: Compare system suspend with Runtime PM.**
A: System suspend: Triggered by user/policy, affects ALL devices. All drivers' `.suspend()` called in order, system enters low-power state (S3). Used for idle laptops/phones. Runtime PM: Per-device, automatic. Each device independently suspends when idle via `pm_runtime_put_autosuspend()` and resumes on demand via `pm_runtime_get_sync()`. System stays running. Runtime PM is more granular — a UART can be suspended while SPI is active. Both use similar driver callbacks (save state, disable clocks), but Runtime PM handles it per-device.

**Q2: What is autosuspend and why is it important?**
A: Autosuspend adds a delay between the last `pm_runtime_put()` and actually calling `runtime_suspend()`. Without it, a device that processes brief requests would suspend/resume for every request — the power of suspend/resume overhead would exceed savings. With autosuspend (e.g., 200ms), the device stays powered if another request comes quickly. `pm_runtime_mark_last_busy()` resets the timer. This amortizes the cost of power transitions over bursts of activity.

**Q3: How do generic power domains (genpd) work?**
A: genpd groups devices sharing a hardware power domain (a SoC region with a single power switch). The domain powers off only when ALL devices in it are runtime-suspended. When any device calls `pm_runtime_get_sync()`, genpd powers on the domain before calling the device's `runtime_resume()`. This is transparent to device drivers — they use normal Runtime PM APIs. genpd handles domain power sequencing, clock requirements, and dependency ordering between parent/child domains.

---

## Summary

- System suspend (S3/S4): system-wide sleep, all devices suspended
- Runtime PM: per-device dynamic power management while system runs
- Driver implements `suspend/resume` (system) and `runtime_suspend/runtime_resume`
- Autosuspend prevents excessive power cycling for bursty workloads
- Wakeup sources allow devices to wake the system from sleep
- Generic power domains (genpd) manage SoC power regions

---

*Next: [Chapter 16 — Thermal Framework](Chapter_16_Thermal_Framework.md)*
