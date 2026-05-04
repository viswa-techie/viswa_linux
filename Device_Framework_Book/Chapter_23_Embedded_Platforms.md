# Chapter 23: Embedded Platform Patterns

## Learning Goals
- Understand common embedded driver architecture patterns
- Know BSP structure and platform bring-up methodology
- Grasp device tree-driven versus platform-data approaches
- Apply framework best practices for production-quality drivers

---

## 23.1 Board Support Package (BSP) Structure

```
BSP — Board Support Package:

arch/arm64/boot/dts/vendor/
├── soc.dtsi           ← SoC-level: all peripherals, status="disabled"
├── board-common.dtsi  ← Common board features
└── board-v1.dts       ← Specific board: enable used peripherals

drivers/
├── clk/vendor/        ← SoC clock controller
├── pinctrl/vendor/    ← Pin multiplexing
├── soc/vendor/        ← SoC-specific (GCC, RPMH)
├── i2c/busses/        ← I2C adapter
├── spi/               ← SPI controller
├── gpio/              ← GPIO controller
├── regulator/         ← PMIC regulators
├── media/platform/vendor/ ← Camera/ISP
├── gpu/drm/vendor/    ← Display controller
└── sound/soc/vendor/  ← Audio platform

BSP Bring-up Order:
1. Boot (U-Boot / ABL → kernel → DTB)
2. Clock controller: all SoC clocks registered
3. Pinctrl: pin muxing configured
4. PMIC (I2C/SPI): regulators registered
5. GPIO controller: GPIOs available
6. Bus controllers: I2C/SPI/USB
7. Peripheral drivers: camera, display, audio
8. Root filesystem mounted
9. User-space init (systemd / Android init)
```

---

## 23.2 Driver Architecture Patterns

```c
/* Pattern 1: Single-file driver with all framework interactions */

struct my_device {
    struct device *dev;
    void __iomem *regs;
    struct clk *clk;
    struct regulator *vdd;
    struct gpio_desc *reset;
    struct dma_chan *dma_chan;
    int irq;
    struct regmap *regmap;
};

static int my_probe(struct platform_device *pdev)
{
    struct my_device *priv;
    int ret;

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    priv->dev = &pdev->dev;
    platform_set_drvdata(pdev, priv);

    /* Resources — all use devm_ for auto-cleanup */
    priv->regs = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(priv->regs))
        return PTR_ERR(priv->regs);

    priv->irq = platform_get_irq(pdev, 0);
    if (priv->irq < 0)
        return priv->irq;

    priv->clk = devm_clk_get(&pdev->dev, NULL);
    if (IS_ERR(priv->clk))
        return PTR_ERR(priv->clk);

    priv->vdd = devm_regulator_get(&pdev->dev, "vdd");
    if (IS_ERR(priv->vdd))
        return PTR_ERR(priv->vdd);

    priv->reset = devm_gpiod_get_optional(&pdev->dev, "reset",
                                           GPIOD_OUT_HIGH);

    /* Power up */
    ret = regulator_enable(priv->vdd);
    if (ret)
        return ret;

    ret = clk_prepare_enable(priv->clk);
    if (ret)
        goto err_regulator;

    if (priv->reset) {
        gpiod_set_value(priv->reset, 0);
        usleep_range(1000, 2000);
    }

    /* Request IRQ */
    ret = devm_request_irq(&pdev->dev, priv->irq, my_isr,
                           0, dev_name(&pdev->dev), priv);
    if (ret)
        goto err_clk;

    /* Runtime PM */
    pm_runtime_set_active(&pdev->dev);
    pm_runtime_enable(&pdev->dev);
    pm_runtime_set_autosuspend_delay(&pdev->dev, 200);
    pm_runtime_use_autosuspend(&pdev->dev);

    return 0;

err_clk:
    clk_disable_unprepare(priv->clk);
err_regulator:
    regulator_disable(priv->vdd);
    return ret;
}
```

---

## 23.3 Variant Support with Match Data

```c
/* Pattern 2: Single driver supporting multiple hardware variants */

struct my_variant {
    unsigned int max_speed;
    unsigned int num_channels;
    bool has_dma;
    const struct reg_map *regs;  /* different register layouts */
};

static const struct my_variant variant_v1 = {
    .max_speed = 1000000,
    .num_channels = 2,
    .has_dma = false,
};

static const struct my_variant variant_v2 = {
    .max_speed = 5000000,
    .num_channels = 4,
    .has_dma = true,
};

static const struct of_device_id my_of_match[] = {
    { .compatible = "vendor,my-dev-v1", .data = &variant_v1 },
    { .compatible = "vendor,my-dev-v2", .data = &variant_v2 },
    { }
};

static int my_probe(struct platform_device *pdev)
{
    const struct my_variant *var;

    var = of_device_get_match_data(&pdev->dev);
    if (!var)
        return -ENODEV;

    if (var->has_dma)
        setup_dma(priv);

    for (int i = 0; i < var->num_channels; i++)
        setup_channel(priv, i);

    return 0;
}
```

---

## 23.4 Error Handling Patterns

```c
/* Pattern 3: Proper error handling with devm_ cleanup */

/* BAD — manual cleanup is error-prone */
static int bad_probe(struct platform_device *pdev)
{
    clk = clk_get(&pdev->dev, NULL);        /* needs clk_put */
    buf = kmalloc(size, GFP_KERNEL);         /* needs kfree */
    irq = request_irq(irq, handler, ...);   /* needs free_irq */
    /* If request_irq fails, must undo kmalloc and clk_get */
    /* Easy to forget cleanup on every error path */
}

/* GOOD — devm_ handles cleanup automatically */
static int good_probe(struct platform_device *pdev)
{
    clk = devm_clk_get(&pdev->dev, NULL);         /* auto clk_put */
    buf = devm_kmalloc(&pdev->dev, size, GFP_KERNEL); /* auto kfree */
    ret = devm_request_irq(&pdev->dev, irq, ...);  /* auto free_irq */

    if (ret)
        return ret;  /* ALL devm_ resources freed automatically */
}

/* devm_ cleanup order: LIFO (reverse of allocation) */
/* probe: devm_clk_get → devm_kmalloc → devm_request_irq */
/* remove: free_irq → kfree → clk_put (automatic) */

/* Common devm_ functions:
 * devm_kzalloc()          devm_clk_get()
 * devm_ioremap_resource() devm_regulator_get()
 * devm_request_irq()      devm_gpiod_get()
 * devm_regmap_init_i2c()  devm_spi_alloc_master()
 * devm_input_allocate_device()
 */
```

---

## 23.5 Regmap-Based Driver Pattern

```c
/* Pattern 4: regmap-based driver (I2C/SPI agnostic) */

/* This driver works with BOTH I2C and SPI without changes */

struct my_sensor {
    struct device *dev;
    struct regmap *regmap;
    const struct my_variant *variant;
};

#define REG_WHO_AM_I    0x0F
#define REG_CTRL        0x20
#define REG_STATUS      0x27
#define REG_DATA_X_L    0x28

static const struct regmap_config my_regmap_config = {
    .reg_bits   = 8,
    .val_bits   = 8,
    .max_register = 0x3F,
    .cache_type = REGCACHE_RBTREE,
    .volatile_reg = my_volatile_reg,  /* uncached registers */
};

static bool my_volatile_reg(struct device *dev, unsigned int reg)
{
    /* Data and status registers should not be cached */
    return (reg >= REG_STATUS && reg <= REG_DATA_X_L + 5);
}

static int my_sensor_common_probe(struct device *dev,
                                   struct regmap *regmap)
{
    unsigned int id;

    regmap_read(regmap, REG_WHO_AM_I, &id);
    if (id != EXPECTED_ID)
        return -ENODEV;

    /* Initialize */
    regmap_write(regmap, REG_CTRL, 0x47);  /* enable, 50Hz */
    return 0;
}

/* I2C variant */
static int my_sensor_i2c_probe(struct i2c_client *client)
{
    struct regmap *regmap = devm_regmap_init_i2c(client,
                                                  &my_regmap_config);
    return my_sensor_common_probe(&client->dev, regmap);
}

/* SPI variant */
static int my_sensor_spi_probe(struct spi_device *spi)
{
    struct regmap *regmap = devm_regmap_init_spi(spi,
                                                  &my_regmap_config);
    return my_sensor_common_probe(&spi->dev, regmap);
}
```

---

## 23.6 Production Quality Checklist

```
Production-Quality Driver Checklist:

□ Device Tree bindings documented (dt-bindings/ YAML schema)
□ All resources use devm_* managed allocation
□ Error paths return clean (no resource leaks)
□ Runtime PM implemented (clock/power gating when idle)
□ System suspend/resume works correctly
□ Probe deferral handled (EPROBE_DEFER passthrough)
□ Edge cases handled (hot-plug, remove, re-probe)
□ Multi-instance support (no global state)
□ lockdep clean (no AB-BA lock ordering)
□ No busy-waiting (use completion/waitqueue)
□ DMA-safe memory allocation (GFP_DMA / dma_alloc_coherent)
□ SPDX license identifier present
□ No magic numbers (use defines/enums)
□ Kernel coding style (checkpatch.pl clean)
□ W=1 compilation warning-free
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| devres | drivers/base/devres.c | devm_ resource management |
| of_device | drivers/of/device.c | DT match data |
| platform | drivers/base/platform.c | Platform driver model |
| BSP example | arch/arm64/boot/dts/ | Device tree source files |

---

## Interview Questions

**Q1: What is the devm_ resource management pattern and why is it important?**
A: `devm_` (device-managed) functions tie resource allocation to a device's lifecycle. When the device is removed or the driver unbinds, all devm_ resources are automatically freed in reverse order (LIFO). This eliminates manual cleanup in error paths and remove functions — a major source of resource leak bugs. Without devm_, every error path in probe must manually free all previously allocated resources. With devm_, `return -EINVAL;` from anywhere in probe correctly frees everything. It's mandatory for production-quality drivers.

**Q2: How do you write a driver that supports multiple hardware variants?**
A: Use the `of_match_table` with `.data` pointing to variant-specific structures. In `probe()`, call `of_device_get_match_data(dev)` to get the variant structure. The variant struct contains hardware-specific parameters: register offsets (if different), number of channels, maximum speeds, feature flags. The driver uses these parameters instead of #ifdefs. Device tree `compatible` strings select the variant: `"vendor,dev-v1"`, `"vendor,dev-v2"`. This keeps one driver source for multiple hardware revisions.

**Q3: Describe the BSP bring-up order for a new SoC board.**
A: (1) Get basic boot working (U-Boot → kernel → early console). (2) Clock controller driver — all SoC clocks must be registered first since every peripheral needs clocks. (3) Pinctrl — configure pin multiplexing for the signals used on this board. (4) Interrupt controller (GIC) — enable interrupt routing. (5) Timer — system tick. (6) PMIC via I2C/SPI — registers regulators and GPIOs. (7) GPIO controller — enables GPIO-based power control. (8) Bus controllers (I2C, SPI, USB). (9) Storage (eMMC/SD) — mount rootfs. (10) Peripheral drivers (display, camera, audio) — in any order, deferred probing handles dependencies.

---

## Summary

- BSP structure: SoC DTSI + board DTS + vendor drivers under drivers/
- devm_ resource management eliminates cleanup bugs — use it for everything
- Variant support: match data in of_device_id selects hardware-specific parameters
- regmap enables bus-agnostic drivers (I2C/SPI/MMIO with same code)
- Error handling: all error paths should return cleanly with devm_ cleanup
- Production checklist: Runtime PM, suspend/resume, lockdep, checkpatch compliance

---

*Next: [Chapter 24 — References and Resources](Chapter_24_References.md)*
