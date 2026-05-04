# Chapter 34: Embedded & Automotive Driver Development

## Chapter Overview

Embedded systems have unique constraints: limited resources, real-time requirements, specific SoCs, and regulatory standards. This chapter covers BSP drivers, SoC drivers, peripheral drivers, and automotive-specific driver development.

---

## 34.1 BSP (Board Support Package) Drivers

A BSP provides everything needed to boot Linux on a specific board/SoC:

```
BSP Components:
├── Bootloader (U-Boot / UEFI)
├── Device Tree (board-specific .dts)
├── Kernel config (defconfig)
├── Clock drivers (drivers/clk/vendor/)
├── Pin control drivers (drivers/pinctrl/vendor/)
├── Reset drivers (drivers/reset/)
├── Power domain drivers (drivers/pmdomain/)
├── Interrupt controller drivers (drivers/irqchip/)
├── Timer drivers (drivers/clocksource/)
└── SoC-specific code (drivers/soc/vendor/)
```

### BSP Driver Typical Probe

```c
/* Clock controller driver — provides clocks to other drivers */
static int my_clk_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    void __iomem *base;
    int i;

    base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(base))
        return PTR_ERR(base);

    /* Register all clocks provided by this controller */
    for (i = 0; i < NUM_CLKS; i++) {
        hw[i] = my_register_clk(dev, base, &clk_data[i]);
        if (IS_ERR(hw[i]))
            return PTR_ERR(hw[i]);
    }

    return devm_of_clk_add_hw_provider(dev, of_clk_hw_onecell_get,
                                         clk_hw_data);
}
```

---

## 34.2 SoC-Specific Drivers

### Common SoC Peripherals

| Peripheral | Subsystem | Example Driver |
|-----------|-----------|----------------|
| UART | tty/serial | drivers/tty/serial/8250/ |
| I2C controller | i2c/busses | drivers/i2c/busses/i2c-qcom-geni.c |
| SPI controller | spi | drivers/spi/spi-qcom-qspi.c |
| GPIO controller | gpio | drivers/gpio/gpio-*.c |
| DMA controller | dma | drivers/dma/ |
| Watchdog | watchdog | drivers/watchdog/ |
| PWM | pwm | drivers/pwm/ |
| ADC | iio | drivers/iio/adc/ |
| Thermal | thermal | drivers/thermal/ |
| Display | drm | drivers/gpu/drm/vendor/ |
| Camera | media | drivers/media/platform/vendor/ |

### Qualcomm SoC Example (SA8155P)

```
Device Tree for Qualcomm SA8155P:
    arch/arm64/boot/dts/qcom/sa8155p.dtsi

Key drivers:
    drivers/clk/qcom/gcc-sm8150.c          ← Global clock controller
    drivers/pinctrl/qcom/pinctrl-sm8150.c  ← Pin muxing
    drivers/i2c/busses/i2c-qcom-geni.c     ← GENI I2C
    drivers/spi/spi-geni-qcom.c            ← GENI SPI
    drivers/tty/serial/qcom_geni_serial.c  ← GENI UART
    drivers/iommu/arm/arm-smmu.c           ← SMMU (IOMMU)
    drivers/gpu/drm/msm/                   ← Display/GPU
    drivers/media/platform/qcom/           ← Camera
```

---

## 34.3 Peripheral Drivers (External Devices)

Devices connected via buses (I2C, SPI, USB) to the SoC.

### I2C Sensor Driver Example

```c
static int my_sensor_probe(struct i2c_client *client)
{
    struct device *dev = &client->dev;
    struct my_sensor *sensor;
    int ret;

    sensor = devm_kzalloc(dev, sizeof(*sensor), GFP_KERNEL);
    if (!sensor)
        return -ENOMEM;

    sensor->regmap = devm_regmap_init_i2c(client, &my_regmap_config);
    if (IS_ERR(sensor->regmap))
        return PTR_ERR(sensor->regmap);

    /* Verify chip ID */
    ret = regmap_read(sensor->regmap, REG_CHIP_ID, &val);
    if (ret || val != EXPECTED_CHIP_ID) {
        dev_err(dev, "unexpected chip ID: 0x%x\n", val);
        return -ENODEV;
    }

    /* Register with IIO subsystem */
    indio_dev = devm_iio_device_alloc(dev, 0);
    indio_dev->info = &my_iio_info;
    indio_dev->channels = my_channels;
    indio_dev->num_channels = ARRAY_SIZE(my_channels);

    return devm_iio_device_register(dev, indio_dev);
}

static const struct of_device_id my_sensor_of_match[] = {
    { .compatible = "vendor,temp-sensor-v2" },
    { },
};
MODULE_DEVICE_TABLE(of, my_sensor_of_match);

static struct i2c_driver my_sensor_driver = {
    .driver = {
        .name = "my-sensor",
        .of_match_table = my_sensor_of_match,
        .pm = &my_sensor_pm_ops,
    },
    .probe = my_sensor_probe,
};
module_i2c_driver(my_sensor_driver);
```

---

## 34.4 Automotive Driver Development

### Automotive-Specific Requirements

| Requirement | Impact on Driver |
|-------------|-----------------|
| **ASIL (ISO 26262)** | Memory protection, error detection, redundancy |
| **Real-time** | Deterministic latency (PREEMPT_RT) |
| **Functional safety** | Watchdog, error reporting, safe states |
| **Cybersecurity** | Secure boot, authenticated firmware, IOMMU |
| **Long lifecycle** | Stable APIs, long-term support (10+ years) |
| **Temperature** | -40°C to +85°C or +105°C operation |

### Key Automotive Subsystems

```
Automotive Linux Stack:
├── ADAS (Advanced Driver Assistance)
│   ├── Camera drivers (V4L2 + ISP)
│   ├── Radar interface drivers
│   ├── LIDAR interface drivers
│   └── GPU compute drivers (DRM)
│
├── Infotainment (IVI)
│   ├── Display drivers (DRM/KMS)
│   ├── Audio drivers (ALSA/ASoC)
│   ├── Touchscreen drivers (Input)
│   ├── Bluetooth drivers
│   └── Wi-Fi drivers
│
├── Vehicle Network
│   ├── CAN bus drivers (SocketCAN)
│   ├── LIN bus drivers
│   ├── Ethernet AVB/TSN drivers
│   └── MOST bus drivers
│
└── System
    ├── eMMC/UFS storage drivers
    ├── Watchdog drivers
    ├── Power management
    └── Secure storage (TEE)
```

### CAN Bus Driver (SocketCAN)

```c
/* CAN is integrated into Linux networking stack */
#include <linux/can/dev.h>

static const struct net_device_ops my_can_netdev_ops = {
    .ndo_open      = my_can_open,
    .ndo_stop      = my_can_close,
    .ndo_start_xmit = my_can_start_xmit,
};

static int my_can_probe(struct platform_device *pdev)
{
    struct net_device *ndev;
    struct my_can_priv *priv;

    ndev = alloc_candev(sizeof(*priv), TX_ECHO_SKB_MAX);
    if (!ndev)
        return -ENOMEM;

    priv = netdev_priv(ndev);
    priv->can.bittiming_const = &my_bittiming_const;
    priv->can.do_set_mode = my_can_set_mode;

    ndev->netdev_ops = &my_can_netdev_ops;
    ndev->flags |= IFF_ECHO;

    platform_set_drvdata(pdev, ndev);
    return register_candev(ndev);
}
```

### Automotive Watchdog Pattern

```c
/* Watchdog: reset system if driver fails to ping within timeout */
static int my_wdt_probe(struct platform_device *pdev)
{
    struct watchdog_device *wdd;

    wdd = devm_kzalloc(dev, sizeof(*wdd), GFP_KERNEL);
    wdd->info = &my_wdt_info;
    wdd->ops = &my_wdt_ops;
    wdd->timeout = 30;          /* 30 second timeout */
    wdd->min_timeout = 1;
    wdd->max_timeout = 60;

    watchdog_init_timeout(wdd, 0, dev);
    watchdog_set_nowayout(wdd, true);  /* Cannot be stopped */

    return devm_watchdog_register_device(dev, wdd);
}
```

---

## 34.5 Cross-Compilation for Embedded

```bash
# ARM64 cross-compilation
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

# Build kernel
make defconfig
make -j$(nproc) Image dtbs modules

# Build out-of-tree driver
make -C /path/to/kernel M=$PWD modules

# Install to rootfs
make INSTALL_MOD_PATH=/path/to/rootfs modules_install
```

---

## 34.6 Embedded Constraints

| Constraint | Driver Impact |
|-----------|---------------|
| **Limited RAM** | Minimize allocations, use `kvmalloc()`, streaming DMA |
| **Limited flash** | Compress modules, strip debug symbols |
| **No swap** | Never assume abundant memory |
| **Battery** | Aggressive runtime PM, autosuspend |
| **Real-time** | Avoid unbounded loops, use `PREEMPT_RT`, threaded IRQs |
| **No debug tools** | Build in debugfs, devcoredump from start |

---

## Interview Questions

**Q1: What is a BSP and what does it contain?**
A: Board Support Package — everything needed to boot Linux on a specific board: bootloader config, Device Tree, kernel defconfig, and drivers for SoC-specific hardware (clocks, pin muxing, resets, power domains, interrupt controllers). Without the BSP, the kernel cannot initialize the SoC.

**Q2: How is automotive driver development different from consumer?**
A: 1) Functional safety (ISO 26262) — need watchdogs, error detection, safe states. 2) Temperature range (-40 to +105°C). 3) Long lifecycle (10+ years). 4) Real-time requirements (PREEMPT_RT). 5) Cybersecurity (secure boot, IOMMU). 6) CAN/LIN vehicle network integration.

**Q3: What is SocketCAN and why is it important for automotive?**
A: SocketCAN integrates CAN bus into the Linux networking stack. CAN devices appear as network interfaces (`can0`), applications use standard socket APIs. This means standard tools (`ip`, `candump`, `cansend`) work, and CAN applications are portable. It's the standard for Linux-based automotive systems.

---

*Next: [Chapter 35 — References and Documentation](Chapter_35_References.md)*
