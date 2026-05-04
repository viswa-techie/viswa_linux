# Chapter 17: Device Tree Integration with Frameworks

## Learning Goals
- Understand how device tree bindings connect hardware descriptions to framework drivers
- Know common DT patterns: phandles, specifier cells, property naming
- Grasp device tree overlay and runtime modification
- Debug device tree issues with dtc, procfs, and sysfs

---

## 17.1 Device Tree — Framework Binding Overview

```
Device Tree connects hardware description to framework drivers:

DTS File (hardware description):
┌──────────────────────────────────────────────┐
│  / {                                          │
│    model = "My Board";                        │
│                                               │
│    clocks { ... }     ──► Clock framework     │
│    regulators { ... } ──► Regulator framework │
│    thermal-zones{...} ──► Thermal framework   │
│                                               │
│    i2c@40005400 {                              │
│      compatible = "vendor,i2c"; ──► I2C adapter│
│      clocks = <&rcc I2C1_CLK>;  ──► Clock ref │
│                                               │
│      sensor@48 {                              │
│        compatible = "ti,tmp102"; ──► I2C driver│
│        reg = <0x48>;             ──► I2C addr  │
│        vdd-supply = <&ldo1>;     ──► Regulator │
│        interrupts = <5 IRQ_TYPE_EDGE_FALLING>;│
│      };                                       │
│    };                                         │
│  };                                           │
└──────────────────────────────────────────────┘

Each property type maps to a specific framework API:
  compatible    → Driver matching
  reg           → Bus address / MMIO region
  clocks        → devm_clk_get()
  *-supply      → devm_regulator_get()
  *-gpios       → devm_gpiod_get()
  interrupts    → platform_get_irq()
  dmas          → dma_request_chan()
  pinctrl-*     → Pinmux configuration
  power-domains → PM domain association
```

---

## 17.2 Phandle References and Specifier Cells

```dts
/* Phandle: a reference from one node to another */

/* Provider declares #*-cells to specify arguments */
gpio1: gpio@40020000 {
    gpio-controller;
    #gpio-cells = <2>;           /* consumer provides 2 args */
    interrupt-controller;
    #interrupt-cells = <2>;
};

clk: clock-controller@10000 {
    #clock-cells = <1>;          /* consumer provides 1 arg: clock ID */
};

/* Consumer references provider with phandle + args */
my_device {
    /* gpio phandle: <&provider pin_number flags> */
    reset-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;
    /*             ^phandle ^pin ^flag (2 cells) */

    /* clock phandle: <&provider clock_id> */
    clocks = <&clk CLK_UART0>;
    /*        ^phandle ^id (1 cell) */

    /* interrupt: <parent irq_num type> */
    interrupt-parent = <&gpio1>;
    interrupts = <5 IRQ_TYPE_EDGE_FALLING>;
    /*            ^irq ^type (2 cells) */

    /* Multiple references */
    clocks = <&clk CLK_UART0>, <&clk CLK_APB>;
    clock-names = "uart", "apb";
    /* Driver: devm_clk_get(dev, "uart") → first clock */
    /* Driver: devm_clk_get(dev, "apb")  → second clock */
};
```

---

## 17.3 Common Framework Property Patterns

```dts
/* Comprehensive device tree node with all framework bindings */
my_camera: camera@40050000 {
    compatible = "vendor,camera-isp";
    reg = <0x40050000 0x1000>;        /* MMIO region */
    reg-names = "isp-regs";

    /* Interrupts */
    interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;

    /* Clocks */
    clocks = <&clk CLK_ISP>, <&clk CLK_AHB>;
    clock-names = "isp", "ahb";

    /* Regulators */
    vdd-supply = <&ldo3>;
    vio-supply = <&ldo4>;

    /* GPIOs */
    reset-gpios = <&gpio1 12 GPIO_ACTIVE_LOW>;
    enable-gpios = <&gpio2 5 GPIO_ACTIVE_HIGH>;

    /* DMA */
    dmas = <&dma1 4 1>;
    dma-names = "rx";

    /* Pin control */
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&camera_pins_active>;
    pinctrl-1 = <&camera_pins_sleep>;

    /* Power domain */
    power-domains = <&pd_camera>;

    /* Misc */
    status = "okay";

    /* Child device (I2C sensor connected via ISP I2C) */
    port {
        isp_in: endpoint {
            remote-endpoint = <&sensor_out>;
        };
    };
};
```

---

## 17.4 Reading Properties in Driver Code

```c
/* Parsing device tree properties in a driver */
#include <linux/of.h>
#include <linux/of_device.h>

static int my_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct device_node *np = dev->of_node;

    /* Read string property */
    const char *name;
    of_property_read_string(np, "label", &name);

    /* Read integer property */
    u32 value;
    of_property_read_u32(np, "vendor,my-param", &value);

    /* Read array */
    u32 array[4];
    of_property_read_u32_array(np, "vendor,my-array", array, 4);

    /* Check boolean property */
    bool flag = of_property_read_bool(np, "vendor,feature-enabled");

    /* Get MMIO resource */
    void __iomem *regs = devm_platform_ioremap_resource(pdev, 0);

    /* Get IRQ */
    int irq = platform_get_irq(pdev, 0);

    /* Framework properties are read automatically:
     * clocks    → devm_clk_get(dev, "name")
     * *-supply  → devm_regulator_get(dev, "name")
     * *-gpios   → devm_gpiod_get(dev, "name", flags)
     * dmas      → dma_request_chan(dev, "name")
     */

    return 0;
}

/* Match table */
static const struct of_device_id my_of_match[] = {
    { .compatible = "vendor,my-device", .data = &variant_a },
    { .compatible = "vendor,my-device-v2", .data = &variant_b },
    { }
};

/* Get match data */
const struct my_variant *var = of_device_get_match_data(dev);
```

---

## 17.5 Device Tree Overlays

```dts
/* Device Tree Overlay — runtime modification */

/* Base DTS: */
/ {
    i2c1: i2c@40005400 {
        #address-cells = <1>;
        #size-cells = <0>;
        status = "okay";
    };
};

/* Overlay: add a sensor to I2C bus */
/dts-v1/;
/plugin/;

&i2c1 {
    #address-cells = <1>;
    #size-cells = <0>;

    temperature_sensor: sensor@48 {
        compatible = "ti,tmp102";
        reg = <0x48>;
    };
};

/* Apply overlay at runtime (ConfigFS) */
/* $ mkdir /sys/kernel/config/device-tree/overlays/sensor */
/* $ cat overlay.dtbo > /sys/kernel/config/device-tree/overlays/sensor/dtbo */

/* Or via U-Boot: */
/* fdt apply overlay.dtbo */
```

---

## 17.6 Debug

```bash
# Inspect device tree at runtime
$ ls /proc/device-tree/
$ cat /proc/device-tree/model
$ cat /proc/device-tree/compatible

# Dump specific node
$ ls /proc/device-tree/soc/i2c@40005400/
$ xxd /proc/device-tree/soc/i2c@40005400/compatible

# Decompile running device tree
$ dtc -I fs -O dts /proc/device-tree/ > running.dts

# Compile / validate DTS
$ dtc -I dts -O dtb -o board.dtb board.dts
$ dtc -I dtb -O dts board.dtb     # decompile

# Check for binding issues
$ make dt_binding_check
$ make dtbs_check

# sysfs device-DT association
$ ls -la /sys/devices/platform/40005400.i2c/of_node
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| OF core | drivers/of/base.c | Device tree parsing |
| OF platform | drivers/of/platform.c | Platform device creation |
| DT bindings | Documentation/devicetree/bindings/ | Binding documentation |
| DTC | scripts/dtc/ | Device tree compiler |
| Overlay | drivers/of/overlay.c | Runtime overlay support |

---

## Interview Questions

**Q1: How does the kernel use device tree to match drivers?**
A: When the kernel boots, the OF (Open Firmware) core parses the device tree blob (DTB) and creates platform devices for nodes with `compatible` properties. For bus nodes (I2C, SPI), the bus framework creates client devices for child nodes. Each driver registers an `of_match_table` with compatible strings. The kernel's bus infrastructure matches device compatible strings against driver match tables. When a match is found, the driver's `probe()` is called with the device, which can then parse further DT properties. Matching priority: exact compatible first, then generic fallback compatibles.

**Q2: What are phandle references and specifier cells?**
A: A phandle is a numeric reference from one DT node to another — like a pointer. Specifier cells are additional arguments that follow the phandle. The provider declares `#<type>-cells = <N>` to specify how many argument cells consumers must provide. Example: `#gpio-cells = <2>` means `<&gpio1 5 FLAGS>` — phandle + 2 cells (pin number and flags). Common: `#clock-cells = <1>` for clock ID, `#interrupt-cells = <2>` for IRQ number + type. The provider's xlate function interprets the cells.

**Q3: What is the `status` property and why is it important?**
A: The `status` property controls whether a device node is active. Values: `"okay"` — device is present and should be used (drivers bind). `"disabled"` — device exists but should not be used (no driver binding). The SoC-level DTSI typically declares all peripherals as `disabled`, and board-level DTS files set `status = "okay"` for peripherals actually wired on that board. This allows one SoC DTSI to serve many boards. Other values: `"reserved"`, `"fail"`, `"fail-sss"`.

---

## Summary

- Device tree describes hardware; frameworks parse DT properties to bind resources
- Phandles reference other nodes; specifier cells provide arguments (pin, ID, flags)
- Standard properties: `compatible`, `reg`, `clocks`, `*-supply`, `*-gpios`, `dmas`
- Drivers use `devm_clk_get()`, `devm_regulator_get()`, etc. — framework reads DT
- Overlays enable runtime device tree modification (ConfigFS or U-Boot)
- Status property (`okay`/`disabled`) controls per-board peripheral activation

---

*Next: [Chapter 18 — Framework Interaction Architecture](Chapter_18_Framework_Interactions.md)*
