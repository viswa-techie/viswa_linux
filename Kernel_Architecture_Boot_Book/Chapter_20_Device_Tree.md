# Chapter 20: Device Tree

## Learning Goals
- Understand the device tree concept and why it exists
- Master DTS/DTB syntax and compilation
- Know how the kernel parses and uses device tree
- Understand device tree overlays, bindings, and driver integration

---

## 20.1 Device Tree Concept

The **Device Tree** (DT) is a data structure describing hardware that the kernel cannot discover automatically. It decouples hardware description from kernel code.

```
Why Device Tree Exists:

Before DT (ARM Linux ≤ 3.x):
┌─────────────────────────────────────────┐
│  Each board had HARDCODED C files:      │
│  arch/arm/mach-*/board-*.c              │
│  - Every GPIO, IRQ, MMIO address        │
│  - 100s of board files                  │
│  - New board = new kernel build         │
│  - MAINTENANCE NIGHTMARE               │
└─────────────────────────────────────────┘

With DT (ARM Linux 3.x+):
┌─────────────────────────────────────────┐
│  Hardware described in .dts text file   │
│  Compiled to .dtb binary blob           │
│  Loaded by bootloader alongside kernel  │
│  Same kernel binary + different DTB     │
│  = runs on different boards!            │
│  CLEAN SEPARATION                       │
└─────────────────────────────────────────┘
```

---

## 20.2 Hardware Description in Embedded Systems

```
Device Tree File Hierarchy:

.dtsi files (SoC-level — shared by all boards using this SoC):
   arch/arm64/boot/dts/vendor/soc.dtsi
   ├── CPU definitions
   ├── Interrupt controller
   ├── Timer
   ├── SoC peripherals (UART, I2C, SPI, GPIO)
   └── Most entries: status = "disabled"

.dts files (Board-level — specific to one board):
   arch/arm64/boot/dts/vendor/board.dts
   ├── #include "soc.dtsi"
   ├── Enable needed peripherals: status = "okay"
   ├── Board-specific pins, GPIOs
   ├── Connected external devices (sensors, displays)
   └── Memory configuration

Compilation:
   board.dts + soc.dtsi → board.dtb (binary blob, 50-200 KB)
```

---

## 20.3 Device Tree Syntax

```dts
/* Complete Device Tree Example */
/dts-v1/;

#include <dt-bindings/interrupt-controller/arm-gic.h>
#include <dt-bindings/clock/my-soc-clocks.h>
#include <dt-bindings/gpio/gpio.h>

/ {
    /* Root properties */
    model = "MyCompany MyBoard Rev2";
    compatible = "myco,myboard", "myco,soc8000";
    #address-cells = <2>;    /* 64-bit addresses */
    #size-cells = <2>;       /* 64-bit sizes */

    /* CPU descriptions */
    cpus {
        #address-cells = <1>;
        #size-cells = <0>;

        cpu0: cpu@0 {
            device_type = "cpu";
            compatible = "arm,cortex-a78";
            reg = <0x0>;
            enable-method = "psci";    /* How to boot secondary CPUs */
            clock-frequency = <2000000000>;  /* 2 GHz */
        };

        cpu1: cpu@1 {
            device_type = "cpu";
            compatible = "arm,cortex-a55";
            reg = <0x1>;
            enable-method = "psci";
            clock-frequency = <1400000000>;  /* 1.4 GHz */
        };
    };

    /* Memory */
    memory@80000000 {
        device_type = "memory";
        reg = <0x0 0x80000000 0x0 0x80000000>;  /* 2 GB at 0x80000000 */
    };

    /* Chosen — boot parameters */
    chosen {
        bootargs = "console=ttyS0,115200 earlycon";
        stdout-path = "serial0:115200n8";
    };

    /* SoC peripherals (behind a bus) */
    soc {
        compatible = "simple-bus";
        #address-cells = <2>;
        #size-cells = <2>;
        ranges;    /* 1:1 address pass-through */

        /* Interrupt Controller */
        gic: interrupt-controller@10000000 {
            compatible = "arm,gic-v3";
            #interrupt-cells = <3>;
            interrupt-controller;
            reg = <0x0 0x10000000 0x0 0x10000>,  /* Distributor */
                  <0x0 0x10100000 0x0 0x100000>;  /* Redistributor */
        };

        /* Clock Controller */
        clk: clock-controller@20000000 {
            compatible = "myco,soc8000-clk";
            reg = <0x0 0x20000000 0x0 0x1000>;
            #clock-cells = <1>;
        };

        /* UART */
        uart0: serial@40010000 {
            compatible = "myco,uart-v2", "ns16550a";
            reg = <0x0 0x40010000 0x0 0x100>;
            interrupts = <GIC_SPI 45 IRQ_TYPE_LEVEL_HIGH>;
            clocks = <&clk CLK_UART0>;
            clock-names = "uart_clk";
            status = "okay";
        };

        /* I2C Controller */
        i2c0: i2c@40020000 {
            compatible = "myco,i2c-v3";
            reg = <0x0 0x40020000 0x0 0x100>;
            interrupts = <GIC_SPI 50 IRQ_TYPE_LEVEL_HIGH>;
            clocks = <&clk CLK_I2C0>;
            #address-cells = <1>;
            #size-cells = <0>;
            clock-frequency = <400000>;    /* 400 kHz (fast mode) */
            status = "okay";

            /* I2C devices on this bus */
            temperature_sensor: tmp105@48 {
                compatible = "ti,tmp105";
                reg = <0x48>;    /* I2C address */
            };

            touchscreen: goodix@5d {
                compatible = "goodix,gt9271";
                reg = <0x5d>;
                interrupt-parent = <&gpio0>;
                interrupts = <12 IRQ_TYPE_EDGE_FALLING>;
                reset-gpios = <&gpio0 14 GPIO_ACTIVE_LOW>;
                irq-gpios = <&gpio0 12 GPIO_ACTIVE_HIGH>;
            };
        };

        /* GPIO Controller */
        gpio0: gpio@40030000 {
            compatible = "myco,gpio-v2";
            reg = <0x0 0x40030000 0x0 0x100>;
            interrupts = <GIC_SPI 60 IRQ_TYPE_LEVEL_HIGH>;
            gpio-controller;
            #gpio-cells = <2>;
            interrupt-controller;
            #interrupt-cells = <2>;
        };
    };

    /* Board-level: LEDs */
    leds {
        compatible = "gpio-leds";
        status-led {
            label = "board:green:status";
            gpios = <&gpio0 5 GPIO_ACTIVE_HIGH>;
            linux,default-trigger = "heartbeat";
        };
    };
};
```

---

## 20.4 Device Tree Parsing During Boot

```
DT Parsing Flow:

Bootloader:
  1. Loads DTB to memory (e.g., 0x49000000)
  2. Passes DTB address in x0 (ARM64)

Kernel Assembly (head.S):
  3. Saves DTB pointer: __fdt_pointer = x0

setup_arch():
  4. setup_machine_fdt(__fdt_pointer)
     ├── early_init_dt_scan() — validates FDT magic (0xD00DFEED)
     ├── of_scan_flat_dt(early_init_dt_scan_memory)
     │   └── Extracts /memory nodes → memblock_add()
     ├── of_scan_flat_dt(early_init_dt_scan_chosen)
     │   └── Extracts /chosen/bootargs → boot_command_line
     └── Validate DTB integrity

  5. unflatten_device_tree()
     ├── Converts flat DTB → tree of struct device_node
     └── Now queryable via of_find_*() APIs

do_initcalls():
  6. of_platform_default_populate()
     └── Walk DT tree → create platform_device for each node
         with "compatible" property

Driver probing:
  7. Driver's of_match_table matches DT compatible
     → probe() called with device's DT node accessible
       via dev->of_node
```

```c
/* Kernel Device Tree APIs */

/* Find a node */
struct device_node *np;
np = of_find_node_by_name(NULL, "serial");
np = of_find_compatible_node(NULL, NULL, "myco,uart-v2");

/* Read properties */
const char *str;
of_property_read_string(np, "label", &str);

u32 val;
of_property_read_u32(np, "clock-frequency", &val);

int irq = of_irq_get(np, 0);

/* In a driver's probe function */
static int my_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;

    u32 freq;
    of_property_read_u32(np, "clock-frequency", &freq);

    bool has_dma = of_property_read_bool(np, "dma-capable");

    struct device_node *child;
    for_each_child_of_node(np, child) {
        /* Process child nodes */
    }
}
```

---

## 20.5 Device Tree Overlays

```dts
/* Device Tree Overlay — add/modify hardware at runtime or boot */
/dts-v1/;
/plugin/;

&i2c0 {
    /* Add a new sensor to existing I2C bus */
    pressure_sensor: bmp280@76 {
        compatible = "bosch,bmp280";
        reg = <0x76>;
    };
};

&uart0 {
    /* Change existing node property */
    status = "disabled";   /* Disable UART0 */
};
```

```bash
# Compile DT overlay
dtc -I dts -O dtb -o camera-overlay.dtbo camera-overlay.dts

# Apply overlay in U-Boot
fdt addr $fdt_addr
fdt resize 8192
fdt apply $overlay_addr

# Apply overlay at runtime (configfs)
mkdir /sys/kernel/config/device-tree/overlays/my-overlay
cat camera-overlay.dtbo > /sys/kernel/config/device-tree/overlays/my-overlay/dtbo

# Compile device tree
dtc -I dts -O dtb -o board.dtb board.dts

# Decompile DTB back to readable DTS
dtc -I dtb -O dts -o board-decompiled.dts board.dtb
```

---

## Interview Questions

**Q1: What is a device tree and why is it needed?**
A: A device tree is a hierarchical data structure describing hardware that can't be auto-discovered (SoC peripherals, I2C/SPI devices). It decouples hardware description from kernel code — same kernel binary works on different boards with different DTBs. Before DT, each ARM board required hardcoded C files in the kernel.

**Q2: What is the difference between .dtsi and .dts files?**
A: `.dtsi` (DT Source Include) describes SoC-level hardware shared by all boards using that SoC. `.dts` (DT Source) describes board-specific hardware and includes the SoC `.dtsi`. The SoC file disables peripherals by default (`status = "disabled"`); the board file enables what's connected (`status = "okay"`).

**Q3: How does a driver get its configuration from device tree?**
A: When the kernel populates platform devices from DT, each `platform_device` gets a pointer to its `device_node` (`dev->of_node`). The driver's `probe()` function uses `of_property_read_*()` APIs to read properties (reg, interrupts, clock-frequency, custom values). Resources like MMIO and IRQ are already parsed into `platform_device->resource[]`.

---

## Summary

- Device tree describes non-discoverable hardware in a hierarchical data structure
- DTS (text) → compiled to DTB (binary) → loaded by bootloader → parsed by kernel
- SoC .dtsi defines peripherals (disabled); board .dts enables and configures them
- Kernel parsing: FDT → memblock/cmdline (early) → unflatten → populate platform_devices
- Drivers match via `of_match_table` compatible strings; read config via `of_property_read_*()`
- Overlays allow adding/modifying hardware description at boot or runtime

---

*Next: [Chapter 21 — Multiprocessor Boot](Chapter_21_Multiprocessor_Boot.md)*
