# Chapter 12: Platform Device Drivers

## Chapter Overview

Platform drivers are the most common driver type in embedded Linux, handling all non-enumerable SoC peripherals. This chapter covers the complete platform driver lifecycle from Device Tree description through probe to removal.

---

## 12.1 Platform Devices Overview

**Platform devices** are peripherals directly integrated into the SoC or connected via non-discoverable buses. Since they can't announce themselves, they require Device Tree or ACPI descriptions.

```
Examples of Platform Devices:
├── UART controllers
├── I2C/SPI bus controllers
├── GPIO controllers
├── Timer/watchdog peripherals
├── DMA engines
├── Display controllers
├── Camera interfaces
├── USB PHYs
└── Crypto engines
```

---

## 12.2 Platform Bus Architecture

```
┌───────────────────────────────────────────────────────────┐
│       Device Tree (.dts)                                   │
│  ┌──────────────────────────────────────────┐             │
│  │ my-uart@2000000 {                        │             │
│  │   compatible = "vendor,my-uart";         │             │
│  │   reg = <0x2000000 0x1000>;              │             │
│  │   interrupts = <GIC_SPI 42 ...>;         │             │
│  │   clocks = <&clk_uart>;                  │             │
│  │ };                                       │             │
│  └──────────────────┬───────────────────────┘             │
│                     │ of_platform_populate()                │
│                     ▼                                      │
│  ┌──────────────────────────────────────────┐             │
│  │       platform_device                     │             │
│  │  .name = "my-uart"                        │             │
│  │  .resource[0] = { IORESOURCE_MEM, ...}   │             │
│  │  .resource[1] = { IORESOURCE_IRQ, ...}   │             │
│  │  .dev.of_node = <DT node pointer>        │             │
│  └──────────────────┬───────────────────────┘             │
│                     │ platform_match()                      │
│                     ▼                                      │
│  ┌──────────────────────────────────────────┐             │
│  │        platform_driver                     │             │
│  │  .driver.of_match_table = my_dt_ids[]     │             │
│  │  .probe = my_probe                        │             │
│  │  .remove = my_remove                      │             │
│  └──────────────────────────────────────────┘             │
└───────────────────────────────────────────────────────────┘
```

---

## 12.3 Platform Device Registration

### From Device Tree (Modern Method)

```c
/* Kernel automatically creates platform_devices from DT nodes
 * with the "compatible" property during of_platform_populate()
 * (called at boot from init/main.c → arch code)
 */

/* DT node: */
/* uart@2000000 {
 *     compatible = "vendor,my-uart";
 *     reg = <0x2000000 0x1000>;
 *     interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;
 *     clocks = <&uart_clk>;
 *     clock-names = "apb";
 * };
 */
```

### Legacy Board File (Pre-DT, still in some code)

```c
static struct resource my_uart_resources[] = {
    {
        .start = 0x2000000,
        .end   = 0x2000FFF,
        .flags = IORESOURCE_MEM,
    },
    {
        .start = 42,
        .end   = 42,
        .flags = IORESOURCE_IRQ,
    },
};

static struct platform_device my_uart_device = {
    .name = "my-uart",
    .id   = 0,
    .num_resources = ARRAY_SIZE(my_uart_resources),
    .resource = my_uart_resources,
};

/* During board init: */
platform_device_register(&my_uart_device);
```

---

## 12.4 Platform Driver Probe Mechanism

### Complete Production-Quality Probe

```c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/io.h>
#include <linux/clk.h>
#include <linux/interrupt.h>
#include <linux/pm_runtime.h>

#define REG_CTRL    0x00
#define REG_STATUS  0x04
#define REG_DATA    0x08
#define REG_CFG     0x0C

struct my_uart_priv {
    void __iomem *base;
    struct clk *clk;
    int irq;
    struct device *dev;
    spinlock_t lock;
};

static irqreturn_t my_uart_irq(int irq, void *dev_id)
{
    struct my_uart_priv *priv = dev_id;
    u32 status = readl(priv->base + REG_STATUS);

    if (!(status & BIT(0)))
        return IRQ_NONE;

    /* Handle RX data available */
    writel(BIT(0), priv->base + REG_STATUS);  /* Clear IRQ */
    return IRQ_HANDLED;
}

static int my_uart_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct my_uart_priv *priv;
    u32 fifo_depth;
    int ret;

    /* 1. Allocate private data */
    priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;
    priv->dev = dev;
    spin_lock_init(&priv->lock);

    /* 2. Map registers */
    priv->base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(priv->base))
        return PTR_ERR(priv->base);

    /* 3. Get clock */
    priv->clk = devm_clk_get(dev, "apb");
    if (IS_ERR(priv->clk)) {
        return dev_err_probe(dev, PTR_ERR(priv->clk),
                             "failed to get clock\n");
    }

    ret = clk_prepare_enable(priv->clk);
    if (ret)
        return ret;

    /* 4. Read DT properties */
    ret = of_property_read_u32(dev->of_node, "fifo-depth", &fifo_depth);
    if (ret)
        fifo_depth = 16;  /* default */

    /* 5. Get and register IRQ */
    priv->irq = platform_get_irq(pdev, 0);
    if (priv->irq < 0) {
        ret = priv->irq;
        goto err_clk;
    }

    ret = devm_request_irq(dev, priv->irq, my_uart_irq, 0,
                           dev_name(dev), priv);
    if (ret) {
        dev_err(dev, "failed to request IRQ %d: %d\n", priv->irq, ret);
        goto err_clk;
    }

    /* 6. Initialize hardware */
    writel(fifo_depth, priv->base + REG_CFG);
    writel(0x03, priv->base + REG_CTRL);  /* Enable TX + RX */

    /* 7. Enable runtime PM */
    pm_runtime_set_active(dev);
    pm_runtime_enable(dev);

    /* 8. Store driver data */
    platform_set_drvdata(pdev, priv);

    dev_info(dev, "UART probed: IRQ=%d, FIFO=%u\n", priv->irq, fifo_depth);
    return 0;

err_clk:
    clk_disable_unprepare(priv->clk);
    return ret;
}
```

### dev_err_probe() — Modern Error Logging

```c
/* Traditional (verbose) */
clk = devm_clk_get(dev, "apb");
if (IS_ERR(clk)) {
    if (PTR_ERR(clk) != -EPROBE_DEFER)
        dev_err(dev, "failed to get clock: %ld\n", PTR_ERR(clk));
    return PTR_ERR(clk);
}

/* Modern (single line, handles EPROBE_DEFER silently) */
clk = devm_clk_get(dev, "apb");
if (IS_ERR(clk))
    return dev_err_probe(dev, PTR_ERR(clk), "failed to get clock\n");
```

---

## 12.5 Platform Driver Remove Function

```c
static void my_uart_remove(struct platform_device *pdev)
{
    struct my_uart_priv *priv = platform_get_drvdata(pdev);

    /* 1. Disable hardware */
    writel(0x00, priv->base + REG_CTRL);

    /* 2. Disable runtime PM */
    pm_runtime_disable(&pdev->dev);

    /* 3. Disable clock */
    clk_disable_unprepare(priv->clk);

    /* devm resources freed automatically after remove() returns */
    dev_info(&pdev->dev, "removed\n");
}
```

### Driver Registration

```c
static const struct of_device_id my_uart_of_match[] = {
    { .compatible = "vendor,my-uart" },
    { .compatible = "vendor,my-uart-v2", .data = &v2_data },
    { }    /* Sentinel — MUST be last */
};
MODULE_DEVICE_TABLE(of, my_uart_of_match);

static const struct dev_pm_ops my_uart_pm_ops = {
    SET_SYSTEM_SLEEP_PM_OPS(my_uart_suspend, my_uart_resume)
    SET_RUNTIME_PM_OPS(my_uart_runtime_suspend,
                       my_uart_runtime_resume, NULL)
};

static struct platform_driver my_uart_driver = {
    .probe  = my_uart_probe,
    .remove = my_uart_remove,
    .driver = {
        .name           = "my-uart",
        .of_match_table = my_uart_of_match,
        .pm             = &my_uart_pm_ops,
    },
};
module_platform_driver(my_uart_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Developer");
MODULE_DESCRIPTION("My UART platform driver");
```

---

## Platform Resource Access APIs

| API | Purpose |
|-----|---------|
| `devm_platform_ioremap_resource(pdev, index)` | Map mem resource from DT `reg` |
| `platform_get_irq(pdev, index)` | Get IRQ number from DT `interrupts` |
| `platform_get_resource(pdev, type, index)` | Raw resource access |
| `devm_clk_get(dev, name)` | Get clock from DT `clocks` |
| `devm_regulator_get(dev, supply)` | Get regulator |
| `devm_gpiod_get(dev, con_id, flags)` | Get GPIO descriptor |
| `of_property_read_u32(node, name, &val)` | Read DT property |
| `device_property_read_u32(dev, name, &val)` | DT/ACPI unified |

---

## Kernel Source References

| File | Purpose |
|------|---------|
| `drivers/base/platform.c` | Platform bus, match, probe |
| `include/linux/platform_device.h` | `struct platform_device/driver` |
| `drivers/of/platform.c` | DT → platform_device creation |
| `include/linux/of.h` | Device Tree property access |

---

## Interview Questions

**Q1: What is the difference between `platform_device` and `platform_driver`?**
A: `platform_device` represents the hardware instance (registers, IRQ, clocks — typically from DT). `platform_driver` contains the code that operates the hardware (probe, remove, PM callbacks). The platform bus matches them via compatible string.

**Q2: What is `devm_platform_ioremap_resource()` and why is it preferred?**
A: Combines `platform_get_resource()` + `devm_ioremap_resource()` in one call. Maps the MMIO region from DT `reg` property into kernel virtual address space. `devm_*` means it's automatically unmapped on driver removal.

**Q3: What is `-EPROBE_DEFER` and give an example?**
A: Returned when a required resource (clock, regulator, GPIO) is provided by another driver that hasn't probed yet. Example: UART driver needs a clock from a clock controller driver that hasn't loaded yet. The kernel retries the probe after other drivers complete.

**Q4: What does `module_platform_driver()` expand to?**
A: Generates `module_init(fn)` calling `platform_driver_register()` and `module_exit(fn)` calling `platform_driver_unregister()`. Eliminates boilerplate.

---

## Summary

| Concept | Key Point |
|---------|-----------|
| Platform device | Non-enumerable SoC peripheral described in DT |
| Resource mapping | `devm_platform_ioremap_resource()` for MMIO |
| IRQ access | `platform_get_irq()` from DT `interrupts` |
| DT matching | `of_match_table` with `compatible` strings |
| Deferred probe | Return `-EPROBE_DEFER` when dependency unavailable |
| `dev_err_probe()` | Modern error + deferred probe logging helper |

---

*Next: [Chapter 13 — Device Tree](Chapter_13_Device_Tree.md)*
