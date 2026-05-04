# Chapter 14: Regulator Framework

## Learning Goals
- Understand the Linux regulator framework architecture
- Learn regulator consumer and provider APIs
- Know voltage regulator types: buck, LDO, boost
- Understand regulator constraints and coupling
- Learn device tree regulator configuration

---

## 1. Regulator Framework Architecture

```
  Regulator Framework
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌── Consumer API ──────────────────────────────────┐   │
  │  │  regulator_get() / devm_regulator_get()           │   │
  │  │  regulator_enable() / regulator_disable()         │   │
  │  │  regulator_set_voltage()                          │   │
  │  │  regulator_get_voltage()                          │   │
  │  └──────────────────────┬────────────────────────────┘   │
  │                         │                                 │
  │  ┌──────────────────────▼────────────────────────────┐   │
  │  │        Regulator Core (drivers/regulator/core.c)   │   │
  │  │                                                     │   │
  │  │  - Constraint enforcement (min/max voltage)         │   │
  │  │  - Reference counting (auto enable/disable)         │   │
  │  │  - Voltage coupling (supply dependencies)           │   │
  │  │  - Status monitoring                                │   │
  │  └──────────────────────┬────────────────────────────┘   │
  │                         │                                 │
  │  ┌──────────────────────▼────────────────────────────┐   │
  │  │        Regulator Drivers (providers)               │   │
  │  │                                                     │   │
  │  │  struct regulator_ops {                             │   │
  │  │      .enable, .disable, .is_enabled                 │   │
  │  │      .set_voltage_sel, .get_voltage_sel             │   │
  │  │      .set_current_limit, .get_current_limit         │   │
  │  │      .set_mode (normal/idle/standby)                │   │
  │  │  };                                                 │   │
  │  │                                                     │   │
  │  │  PMIC drivers: tps65910, max77686, pmic8xxx, etc.  │   │
  │  └──────────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Consumer API

```c
/* Regulator consumer usage in device driver */
#include <linux/regulator/consumer.h>

static int my_driver_probe(struct platform_device *pdev)
{
    struct regulator *vdd, *vdd_io;

    /* Get regulators by supply name */
    vdd = devm_regulator_get(&pdev->dev, "vdd");
    if (IS_ERR(vdd))
        return PTR_ERR(vdd);

    vdd_io = devm_regulator_get(&pdev->dev, "vdd-io");
    if (IS_ERR(vdd_io))
        return PTR_ERR(vdd_io);

    /* Set voltage (microvolts) */
    ret = regulator_set_voltage(vdd, 1800000, 1800000);
    /* min_uV = 1.8V, max_uV = 1.8V (exact) */

    /* Enable the regulator */
    ret = regulator_enable(vdd);
    if (ret)
        return ret;

    ret = regulator_enable(vdd_io);
    if (ret) {
        regulator_disable(vdd);
        return ret;
    }

    /* Get actual voltage */
    int voltage = regulator_get_voltage(vdd);
    dev_info(&pdev->dev, "VDD: %d μV\n", voltage);

    return 0;
}

static int my_driver_remove(struct platform_device *pdev)
{
    regulator_disable(vdd_io);
    regulator_disable(vdd);
    return 0;
}
```

### 2.1 Bulk Regulators

```c
/* Managing multiple regulators together */
static struct regulator_bulk_data my_regulators[] = {
    { .supply = "vdd"    },
    { .supply = "vdd-io" },
    { .supply = "vref"   },
};

static int my_probe(struct platform_device *pdev)
{
    /* Get all regulators at once */
    ret = devm_regulator_bulk_get(&pdev->dev,
                                   ARRAY_SIZE(my_regulators),
                                   my_regulators);

    /* Enable all at once */
    ret = regulator_bulk_enable(ARRAY_SIZE(my_regulators),
                                my_regulators);

    /* Disable all at once */
    regulator_bulk_disable(ARRAY_SIZE(my_regulators),
                           my_regulators);
}
```

---

## 3. Regulator Provider

```c
/* PMIC regulator driver (provider) */
#include <linux/regulator/driver.h>

static const struct regulator_ops my_buck_ops = {
    .enable         = regulator_enable_regmap,
    .disable        = regulator_disable_regmap,
    .is_enabled     = regulator_is_enabled_regmap,
    .set_voltage_sel = regulator_set_voltage_sel_regmap,
    .get_voltage_sel = regulator_get_voltage_sel_regmap,
    .list_voltage    = regulator_list_voltage_linear,
};

static const struct regulator_desc my_buck1_desc = {
    .name           = "BUCK1",
    .id             = 0,
    .ops            = &my_buck_ops,
    .type           = REGULATOR_VOLTAGE,
    .n_voltages     = 128,         /* Number of voltage steps */
    .min_uV         = 600000,      /* 0.6V minimum */
    .uV_step        = 12500,       /* 12.5mV per step */
    /* Voltage = min_uV + (selector × uV_step)
     * Selector 0:   600mV
     * Selector 48:  1200mV (600 + 48×12.5)
     * Selector 127: 2187.5mV
     */
    .enable_reg     = PMIC_BUCK1_CTRL,
    .enable_mask    = BIT(7),
    .vsel_reg       = PMIC_BUCK1_VSEL,
    .vsel_mask      = 0x7F,
    .owner          = THIS_MODULE,
};

static int my_pmic_probe(struct i2c_client *client)
{
    struct regulator_config config = {
        .dev = &client->dev,
        .regmap = pmic->regmap,     /* I2C regmap */
    };

    rdev = devm_regulator_register(&client->dev,
                                    &my_buck1_desc, &config);
    return PTR_ERR_OR_ZERO(rdev);
}
```

---

## 4. Device Tree Configuration

```dts
/* PMIC regulator definitions in device tree */
pmic: pmic@48 {
    compatible = "my-pmic";
    reg = <0x48>;

    regulators {
        buck1: BUCK1 {
            regulator-name = "vdd_cpu";
            regulator-min-microvolt = <800000>;
            regulator-max-microvolt = <1400000>;
            regulator-always-on;        /* Never disable */
            regulator-boot-on;          /* Enabled at boot */
            regulator-ramp-delay = <12500>; /* μV/μs */
        };

        buck2: BUCK2 {
            regulator-name = "vdd_gpu";
            regulator-min-microvolt = <600000>;
            regulator-max-microvolt = <1200000>;
            /* No always-on: can be disabled for PM */
        };

        ldo1: LDO1 {
            regulator-name = "vdd_pll";
            regulator-min-microvolt = <1800000>;
            regulator-max-microvolt = <1800000>; /* Fixed 1.8V */
            regulator-always-on;
        };

        ldo2: LDO2 {
            regulator-name = "vdd_sensor";
            regulator-min-microvolt = <2800000>;
            regulator-max-microvolt = <3300000>;
        };
    };
};

/* Consumer references */
cpu@0 {
    compatible = "arm,cortex-a78";
    cpu-supply = <&buck1>;             /* CPUFreq uses this */
};

gpu@10000 {
    compatible = "my-soc,gpu";
    vdd-supply = <&buck2>;             /* GPU supply */
    vdd-io-supply = <&ldo1>;           /* I/O supply */
};

sensor@39 {
    compatible = "my-sensor";
    vdd-supply = <&ldo2>;
};
```

---

## 5. Regulator Constraints

```
  Regulator Constraint Enforcement
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  DT constraints (regulator-min/max-microvolt):           │
  │  Define the SAFE operating range                         │
  │                                                           │
  │  Consumer request: regulator_set_voltage(reg, 900000,    │
  │                                          1100000)         │
  │                                                           │
  │  Framework checks:                                       │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │  DT constraint:  800000 ≤ V ≤ 1400000           │    │
  │  │  Consumer req:   900000 ≤ V ≤ 1100000           │    │
  │  │  Result:         max(800, 900) ≤ V ≤ min(1400, 1100)│ │
  │  │                  900000 ≤ V ≤ 1100000 ✓          │    │
  │  │                                                  │    │
  │  │  If request outside DT limits → REJECTED         │    │
  │  └──────────────────────────────────────────────────┘    │
  │                                                           │
  │  Multiple consumers sharing one regulator:               │
  │  Consumer A wants: 1800000-1800000 μV                    │
  │  Consumer B wants: 1700000-1900000 μV                    │
  │  Result: max(1800, 1700) to min(1800, 1900) = 1800000   │
  │  (Tightest range that satisfies all consumers)           │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. DVFS Regulator Usage

```c
/* CPUFreq + Regulator for DVFS */
/* This is how cpufreq-dt driver handles voltage scaling */

static int cpufreq_set_target(struct cpufreq_policy *policy,
                                unsigned int target_freq)
{
    struct dev_pm_opp *opp;
    unsigned long volt, old_volt;
    unsigned long freq = target_freq * 1000; /* to Hz */

    /* Find OPP for target frequency */
    opp = dev_pm_opp_find_freq_ceil(cpu_dev, &freq);
    volt = dev_pm_opp_get_voltage(opp);

    old_freq = clk_get_rate(cpu_clk);
    old_volt = regulator_get_voltage(cpu_reg);

    if (freq > old_freq) {
        /* Scaling UP: raise voltage FIRST */
        regulator_set_voltage(cpu_reg, volt, volt);
        clk_set_rate(cpu_clk, freq);
    } else {
        /* Scaling DOWN: lower frequency FIRST */
        clk_set_rate(cpu_clk, freq);
        regulator_set_voltage(cpu_reg, volt, volt);
    }

    return 0;
}
```

---

## 7. Regulator Power Management

```c
/* Runtime PM with regulator control */
static int my_runtime_suspend(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);

    /* Disable clock first, then regulator */
    clk_disable_unprepare(mydev->clk);
    regulator_disable(mydev->vdd);

    return 0;
}

static int my_runtime_resume(struct device *dev)
{
    struct my_device *mydev = dev_get_drvdata(dev);
    int ret;

    /* Enable regulator first, then clock */
    ret = regulator_enable(mydev->vdd);
    if (ret)
        return ret;

    /* Wait for voltage to stabilize */
    usleep_range(100, 200);

    ret = clk_prepare_enable(mydev->clk);
    if (ret) {
        regulator_disable(mydev->vdd);
        return ret;
    }

    return 0;
}

/*
 * Power-up order:  Regulator ON → wait → Clock ON → Reset de-assert
 * Power-down order: Reset assert → Clock OFF → Regulator OFF
 */
```

---

## 8. Regulator Debugging

```bash
# View all regulators
cat /sys/kernel/debug/regulator/regulator_summary

# Example output:
# regulator               use  open   bypass voltage current   min   max
# ──────────────────────────────────────────────────────────────────────
# vdd_cpu                   1     2        0 1100mV    0mA   800mV  1400mV
#    consumer: cpu0                       1100mV
# vdd_gpu                   0     0        0   OFF     0mA   600mV  1200mV
# vdd_pll                   1     3        0 1800mV    0mA  1800mV  1800mV
# vdd_sensor                0     0        0   OFF     0mA  2800mV  3300mV

# Per-regulator sysfs
ls /sys/class/regulator/regulator.0/
# name, state, microvolts, min_microvolts, max_microvolts,
# num_users, type

cat /sys/class/regulator/regulator.0/state      # enabled/disabled
cat /sys/class/regulator/regulator.0/microvolts  # Current voltage
```

---

## Kernel Source Reference

| File/Directory | Purpose |
|---------------|---------|
| `drivers/regulator/core.c` | Regulator framework core |
| `drivers/regulator/helpers.c` | Common regulator helpers |
| `drivers/regulator/of_regulator.c` | DT regulator parsing |
| `include/linux/regulator/consumer.h` | Consumer API |
| `include/linux/regulator/driver.h` | Provider API |
| `include/linux/regulator/machine.h` | Board constraints |
| `drivers/regulator/<pmic>.c` | PMIC-specific drivers |

---

## Interview Questions

**Q1: How does the regulator framework enforce voltage constraints?**
**A:** Constraints come from DT (regulator-min/max-microvolt) and consumer requests (regulator_set_voltage min/max). The framework computes the intersection of all constraints — the tightest range satisfying all consumers and DT limits. If a consumer requests a voltage outside the DT-defined safe range, the request is rejected. Multiple consumers sharing a regulator get the narrowest overlapping voltage range.

**Q2: Why is voltage/clock ordering important during PM transitions?**
**A:** On power-up: regulator must be enabled first to provide stable voltage before clocks start (otherwise, clock circuits operate at undefined voltage causing unreliable behavior). On power-down: clocks must be disabled first to prevent glitches during voltage removal. For DVFS: voltage must increase before frequency goes up (to meet timing margins), and frequency must decrease before voltage goes down.

**Q3: What is the difference between a buck converter and an LDO?**
**A:** A buck (switching) converter uses inductors and high-frequency switching to step down voltage with 85-95% efficiency. An LDO (Low Dropout) linear regulator drops voltage by dissipating excess as heat — efficiency is V_out/V_in. LDOs output cleaner (less ripple) power, suitable for noise-sensitive analog circuits. Bucks are used for high-current digital rails (CPU, DDR) where efficiency is critical.

---

## Summary

- Linux regulator framework manages voltage and current supply to devices
- Consumer API: regulator_get, regulator_enable/disable, regulator_set_voltage
- Provider drivers implement regulator_ops for PMIC hardware control
- Constraints from DT define safe voltage ranges; framework enforces intersection of all constraints
- DVFS requires ordered voltage/frequency transitions (voltage first when scaling up)
- Multiple consumers can share a regulator; framework computes compatible voltage
- Debug via `/sys/kernel/debug/regulator/regulator_summary` and `/sys/class/regulator/`

---

[Previous Chapter: Clock Framework ←](Chapter_13_Clock_Framework.md) | [Next Chapter: Thermal Management →](Chapter_15_Thermal_Management.md)
