# Chapter 14: Regulator Framework

## Learning Goals
- Understand Linux regulator framework for voltage/power supply management
- Know regulator consumer and provider (PMIC) driver models
- Grasp voltage constraints, coupling, and suspend configuration
- Write regulator consumer and provider drivers

---

## 14.1 Regulator Framework Architecture

```
Regulator Framework:

  ┌──────────────────────────────────────────────────┐
  │  Regulator Consumers                              │
  │  ├── Display driver:  regulator_enable/disable    │
  │  ├── WiFi driver:     regulator_set_voltage       │
  │  ├── Sensor driver:   regulator_get/put           │
  │  └── Any peripheral needing power supply          │
  ├──────────────────────────────────────────────────┤
  │  Regulator Core (drivers/regulator/core.c)        │
  │  ├── Voltage/current constraint enforcement       │
  │  ├── Enable/disable reference counting            │
  │  ├── Multi-consumer arbitration                   │
  │  ├── Suspend voltage configuration                │
  │  └── debugfs: /sys/kernel/debug/regulator/        │
  ├──────────────────────────────────────────────────┤
  │  Regulator Drivers (PMIC-specific)                │
  │  ├── Qualcomm RPMH regulators                     │
  │  ├── TI TPS65910/TPS65217                         │
  │  ├── Maxim MAX77686                               │
  │  ├── Dialog DA9063                                │
  │  ├── Fixed regulators (GPIO-controlled)           │
  │  └── regulator_ops: set_voltage, enable, etc.     │
  └──────────────────────────────────────────────────┘
                       │
Hardware:              ▼
  ┌──────────────────────────────────────────────────┐
  │  PMIC (Power Management IC)                       │
  │  ├── BUCK converters (high-efficiency, CPU/DDR)   │
  │  ├── LDO regulators (low-noise, analog circuits) │
  │  ├── Switch regulators (on/off only)              │
  │  └── Controlled via I2C/SPI registers             │
  └──────────────────────────────────────────────────┘

Typical power tree:
  BATTERY → PMIC
               ├── BUCK1 (1.0V) → CPU core
               ├── BUCK2 (1.8V) → DDR memory
               ├── LDO1  (3.3V) → SD card
               ├── LDO2  (1.8V) → Display I/O
               ├── LDO3  (2.8V) → Camera sensor
               └── LDO4  (3.3V) → WiFi module
```

---

## 14.2 Regulator Consumer API

```c
/* Regulator consumer — typical driver usage */
#include <linux/regulator/consumer.h>

static int my_sensor_probe(struct platform_device *pdev)
{
    struct regulator *vdd;
    struct regulator *vio;

    /* Get regulators from device tree */
    vdd = devm_regulator_get(&pdev->dev, "vdd");  /* main power */
    if (IS_ERR(vdd))
        return PTR_ERR(vdd);

    vio = devm_regulator_get(&pdev->dev, "vio");   /* I/O power */
    if (IS_ERR(vio))
        return PTR_ERR(vio);

    /* Set voltage range (min, max in microvolts) */
    ret = regulator_set_voltage(vdd, 2800000, 3000000);  /* 2.8-3.0V */

    /* Enable power supply */
    ret = regulator_enable(vdd);  /* PMIC turns on LDO */
    ret = regulator_enable(vio);

    /* Read actual voltage */
    int voltage = regulator_get_voltage(vdd);
    dev_info(&pdev->dev, "VDD = %d uV\n", voltage);

    /* Optional: get regulator for always-on supply */
    struct regulator *vdd_opt;
    vdd_opt = devm_regulator_get_optional(&pdev->dev, "vdd-opt");
    /* Returns -ENODEV if not specified in DT (no dummy regulator) */

    return 0;
}

static void my_sensor_remove(struct platform_device *pdev)
{
    /* devm handles disable and put */
    regulator_disable(priv->vdd);
    regulator_disable(priv->vio);
}
```

---

## 14.3 Regulator Provider (PMIC Driver)

```c
/* Regulator provider — PMIC LDO/BUCK driver */
#include <linux/regulator/driver.h>

/* Voltage table for LDO (discrete steps) */
static const unsigned int ldo1_voltages[] = {
    1800000, 2500000, 2800000, 3000000, 3300000,
};

static const struct regulator_ops my_ldo_ops = {
    .list_voltage    = regulator_list_voltage_table,
    .map_voltage     = regulator_map_voltage_ascend,
    .set_voltage_sel = regulator_set_voltage_sel_regmap,
    .get_voltage_sel = regulator_get_voltage_sel_regmap,
    .enable          = regulator_enable_regmap,
    .disable         = regulator_disable_regmap,
    .is_enabled      = regulator_is_enabled_regmap,
};

static const struct regulator_desc my_regulators[] = {
    {
        .name = "LDO1",
        .id = 0,
        .ops = &my_ldo_ops,
        .type = REGULATOR_VOLTAGE,
        .n_voltages = ARRAY_SIZE(ldo1_voltages),
        .volt_table = ldo1_voltages,
        .vsel_reg = PMIC_LDO1_VSEL,     /* voltage select register */
        .vsel_mask = 0x07,
        .enable_reg = PMIC_LDO1_CTRL,   /* enable register */
        .enable_mask = BIT(7),
        .owner = THIS_MODULE,
    },
    /* More regulators... */
};

static int my_pmic_probe(struct i2c_client *client)
{
    struct regmap *regmap;
    struct regulator_config config = {};

    regmap = devm_regmap_init_i2c(client, &pmic_regmap_config);

    config.dev = &client->dev;
    config.regmap = regmap;

    for (int i = 0; i < ARRAY_SIZE(my_regulators); i++) {
        config.init_data = pdata->init_data[i];
        config.of_node = pdata->of_node[i];

        struct regulator_dev *rdev;
        rdev = devm_regulator_register(&client->dev,
                                       &my_regulators[i],
                                       &config);
        if (IS_ERR(rdev))
            return PTR_ERR(rdev);
    }

    return 0;
}
```

---

## 14.4 Device Tree Regulator Bindings

```dts
/* PMIC (regulator provider) */
pmic: pmic@34 {
    compatible = "vendor,my-pmic";
    reg = <0x34>;

    regulators {
        ldo1: LDO1 {
            regulator-name = "vdd-sensor";
            regulator-min-microvolt = <1800000>;
            regulator-max-microvolt = <3300000>;
            regulator-boot-on;       /* enabled at boot */
        };

        ldo2: LDO2 {
            regulator-name = "vdd-display";
            regulator-min-microvolt = <1800000>;
            regulator-max-microvolt = <1800000>;  /* fixed voltage */
            regulator-always-on;     /* never disable */
        };

        buck1: BUCK1 {
            regulator-name = "vdd-cpu";
            regulator-min-microvolt = <800000>;
            regulator-max-microvolt = <1200000>;
            regulator-ramp-delay = <12500>;  /* uV/us */
        };
    };
};

/* Regulator consumer */
camera_sensor: sensor@36 {
    compatible = "sony,imx219";
    reg = <0x36>;
    vdd-supply = <&ldo1>;     /* <name>-supply links to regulator */
    vio-supply = <&ldo2>;
};

/* Fixed regulator (GPIO-controlled) */
reg_3v3: regulator-3v3 {
    compatible = "regulator-fixed";
    regulator-name = "3v3";
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
    gpio = <&gpio1 10 GPIO_ACTIVE_HIGH>;
    enable-active-high;
    regulator-boot-on;
};
```

---

## 14.5 Debug

```bash
# Regulator debugfs
$ cat /sys/kernel/debug/regulator/regulator_summary
regulator           use  open  bypass  voltage  current  min_uV  max_uV
----------------------------------------------------------------
ldo1                  1     1       0  2800000        0 1800000 3300000
  camera_sensor                                       2800000 3000000
ldo2                  1     1       0  1800000        0 1800000 1800000
buck1                 1     1       0  1000000        0  800000 1200000

# sysfs
$ ls /sys/class/regulator/
$ cat /sys/class/regulator/regulator.0/name
$ cat /sys/class/regulator/regulator.0/microvolts
$ cat /sys/class/regulator/regulator.0/state
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| Regulator core | drivers/regulator/core.c | Framework core |
| Consumer API | include/linux/regulator/consumer.h | Consumer interface |
| Driver API | include/linux/regulator/driver.h | Provider interface |
| Fixed regulator | drivers/regulator/fixed.c | GPIO-controlled fixed |
| regmap helpers | drivers/regulator/helpers.c | regmap-based ops |
| PMIC drivers | drivers/regulator/ | Vendor PMIC drivers |

---

## Interview Questions

**Q1: How does the regulator framework handle multiple consumers?**
A: The regulator core maintains reference counts for enable/disable and arbitrates voltage requests from multiple consumers. When consumer A requests 2.8-3.0V and consumer B requests 2.5-3.3V, the core selects a voltage in the intersection (2.8-3.0V). Enable is reference-counted: the regulator stays on until ALL consumers call `regulator_disable()`. The framework ensures constraints defined in device tree (`regulator-min/max-microvolt`) are never violated.

**Q2: What is the difference between BUCK and LDO regulators?**
A: **BUCK** (switching regulator): Uses inductors and switching MOSFETs. High efficiency (85-95%). Can step down voltage significantly (e.g., 5V → 1V). Generates switching noise. Used for high-current loads (CPU, DDR). **LDO** (Low Drop-Out): Linear regulator, output = input - dropout. Lower efficiency (proportional to Vout/Vin). Very low noise output. Used for noise-sensitive analog circuits (camera sensor, audio codec). In a PMIC, BUCKs handle major power rails; LDOs provide clean power for sensitive peripherals.

**Q3: What do `regulator-always-on` and `regulator-boot-on` mean?**
A: `regulator-always-on`: The regulator cannot be disabled by any consumer. The framework ignores `regulator_disable()` calls. Used for critical power supplies (DDR voltage, always-on domain). `regulator-boot-on`: The regulator is left enabled at boot if the bootloader enabled it. Without this flag, regulators with no consumer may be disabled during boot to save power. Used for regulators that must remain on during early boot before their consumers probe (e.g., display backlight power during splash screen).

---

## Summary

- Regulator framework manages voltage/current supplies from PMICs
- Consumer API: `devm_regulator_get()` → `regulator_set_voltage()` → `regulator_enable()`
- Provider drivers register `regulator_desc` with ops for set_voltage/enable/disable
- Multi-consumer: voltage arbitration (intersection) + reference-counted enable
- Device tree: provider uses `regulators {}` node; consumer uses `<name>-supply` property
- BUCK regulators for efficiency; LDO for low-noise power

---

*Next: [Chapter 15 — Power Management Framework](Chapter_15_Power_Management.md)*
