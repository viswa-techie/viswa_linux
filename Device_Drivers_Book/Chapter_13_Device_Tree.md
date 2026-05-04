# Chapter 13: Device Tree

## Chapter Overview

Device Tree (DT) is the standard mechanism for describing non-discoverable hardware in embedded Linux systems. Every ARM/ARM64, RISC-V, and many MIPS systems use it. This chapter covers DT syntax, binding conventions, overlays, and how drivers consume DT data.

---

## 13.1 Device Tree Concept

A Device Tree is a **data structure** describing hardware components. The kernel parses it at boot instead of hard-coding hardware descriptions.

```
Without Device Tree:                  With Device Tree:
────────────────────                 ────────────────
arch/arm/mach-xxx/                   arch/arm/boot/dts/
  board-myboard.c                      myboard.dts
  (C code with register                (declarative hardware
   addresses, IRQ numbers,              description, compiled
   platform_device structs)             to .dtb binary)

One kernel binary per board           ONE kernel binary for ALL boards
```

---

## 13.2 Hardware Description in Embedded Systems

```
.dts (source) ──dtc──→ .dtb (binary blob) ──→ Bootloader passes to kernel
                                                     │
                                                     ▼
                                              Kernel parses at boot
                                              Creates platform_devices
```

### File Types

| Extension | Purpose |
|-----------|---------|
| `.dts` | Device Tree Source — board-specific, top-level |
| `.dtsi` | Device Tree Source Include — SoC-level, shared |
| `.dtb` | Device Tree Blob — compiled binary |
| `.dtbo` | Device Tree Blob Overlay — runtime patches |

```
SoC include:  soc-vendor.dtsi     (defines SoC peripherals)
       │
       ▼ #include
Board DTS:    myboard.dts          (enables/configures peripherals)
       │
       ▼ dtc
Binary:       myboard.dtb          (passed by bootloader)
```

---

## 13.3 Device Tree Syntax

### Basic Structure

```dts
/dts-v1/;

/ {                                    /* Root node */
    model = "My Board v1.0";
    compatible = "vendor,myboard";

    #address-cells = <1>;             /* 1 cell (32-bit) for addresses */
    #size-cells = <1>;                /* 1 cell for sizes */

    chosen {
        bootargs = "console=ttyS0,115200";
        stdout-path = "serial0:115200n8";
    };

    memory@80000000 {
        device_type = "memory";
        reg = <0x80000000 0x40000000>;  /* 1GB at 0x80000000 */
    };

    soc {
        compatible = "simple-bus";
        #address-cells = <1>;
        #size-cells = <1>;
        ranges;                        /* 1:1 address translation */

        uart0: serial@2000000 {
            compatible = "vendor,my-uart";
            reg = <0x2000000 0x1000>;
            interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;
            clocks = <&clk_uart>;
            clock-names = "apb";
            status = "okay";
        };

        i2c0: i2c@2010000 {
            compatible = "vendor,my-i2c";
            reg = <0x2010000 0x1000>;
            #address-cells = <1>;
            #size-cells = <0>;
            clock-frequency = <400000>;

            temp-sensor@48 {
                compatible = "ti,tmp102";
                reg = <0x48>;
            };
        };
    };
};
```

---

## 13.4 Device Tree Nodes and Properties

### Common Properties

| Property | Purpose | Example |
|----------|---------|---------|
| `compatible` | Match string for driver binding | `"vendor,my-uart"` |
| `reg` | Register address + size | `<0x2000000 0x1000>` |
| `interrupts` | IRQ specification | `<GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>` |
| `clocks` | Clock phandle(s) | `<&clk_uart>` |
| `clock-names` | Clock name strings | `"apb"` |
| `status` | Enable/disable | `"okay"` or `"disabled"` |
| `dmas` | DMA channel phandle(s) | `<&dma 0 1>` |
| `resets` | Reset controller phandle | `<&reset 5>` |
| `pinctrl-0` | Pin configuration | `<&pinctrl_uart>` |

### Reading Properties in Driver

```c
static int my_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct device_node *np = dev->of_node;
    u32 fifo_depth, baudrate;
    const char *label;
    bool dma_enabled;

    /* Read u32 */
    of_property_read_u32(np, "fifo-depth", &fifo_depth);

    /* Read string */
    of_property_read_string(np, "label", &label);

    /* Read boolean (present = true) */
    dma_enabled = of_property_read_bool(np, "dma-enabled");

    /* Read array */
    u32 sizes[4];
    of_property_read_u32_array(np, "buffer-sizes", sizes, 4);

    /* Use unified device_property_* API (works with DT + ACPI) */
    device_property_read_u32(dev, "fifo-depth", &fifo_depth);

    /* Get match data (driver-specific per compatible) */
    const struct of_device_id *match = of_match_device(my_dt_ids, dev);
    if (match && match->data) {
        const struct my_hw_data *hw = match->data;
        /* Use hw->max_speed, hw->has_dma, etc. */
    }

    return 0;
}
```

---

## 13.5 Device Tree Overlays

Overlays modify the base DT at runtime (e.g., for add-on boards, capes):

```dts
/* Base DT has: */
i2c0: i2c@2010000 {
    status = "okay";
    #address-cells = <1>;
    #size-cells = <0>;
};

/* Overlay adds a sensor: */
/dts-v1/;
/plugin/;

&i2c0 {
    accelerometer@1d {
        compatible = "nxp,mma8453";
        reg = <0x1d>;
        interrupt-parent = <&gpio1>;
        interrupts = <5 IRQ_TYPE_EDGE_FALLING>;
    };
};
```

```bash
# Apply overlay at runtime (if supported)
mkdir -p /sys/kernel/config/device-tree/overlays/accel
cat my_overlay.dtbo > /sys/kernel/config/device-tree/overlays/accel/dtbo

# Remove overlay
rmdir /sys/kernel/config/device-tree/overlays/accel
```

---

## 13.6 Device Tree and Driver Binding

### Complete Flow

```
1. Bootloader loads .dtb into memory, passes address to kernel
          │
2. Kernel unflatten_device_tree() → builds in-memory tree
          │
3. of_platform_populate() → walks tree, creates platform_device
   for each node with "compatible" property
          │
4. platform_bus_type.match() → compares DT compatible to
   driver's of_match_table
          │
5. Match found → really_probe() → driver->probe() called
          │
6. Driver reads DT properties via of_property_read_*()
   and maps resources (reg → ioremap, interrupts → request_irq)
```

### Binding Document Convention

```
Documentation/devicetree/bindings/serial/vendor,my-uart.yaml

properties:
  compatible:
    const: vendor,my-uart
  reg:
    maxItems: 1
  interrupts:
    maxItems: 1
  clocks:
    maxItems: 1
  fifo-depth:
    $ref: /schemas/types.yaml#/definitions/uint32
    description: Hardware FIFO depth
    default: 16

required:
  - compatible
  - reg
  - interrupts
  - clocks
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| `drivers/of/fdt.c` | Flatten device tree parsing |
| `drivers/of/base.c` | DT property access functions |
| `drivers/of/platform.c` | DT → platform_device creation |
| `include/linux/of.h` | DT API (`of_property_read_*`) |
| `scripts/dtc/` | Device Tree Compiler source |
| `Documentation/devicetree/bindings/` | Binding specifications |

---

## OS Comparison

| Aspect | Linux DT | Windows ACPI | macOS DeviceTree | Zephyr DT |
|--------|----------|-------------|------------------|-----------|
| Format | DTS (text) → DTB | ASL → AML | Open Firmware | DTS → DTB |
| Passed by | Bootloader (r2/x0) | Firmware (UEFI) | iBoot | Build system |
| Parser | `unflatten_device_tree` | ACPICA | IODeviceTreeSupport | Generated C macros |
| Driver match | `compatible` string | `_HID`, `_CID` | `compatible` | `compatible` |

---

## Interview Questions

**Q1: What is Device Tree and why is it needed?**
A: DT is a data structure describing non-discoverable hardware. Needed because SoC peripherals can't announce themselves (unlike PCIe/USB). It separates hardware description from kernel code, enabling one kernel binary to boot on multiple boards.

**Q2: What is the `compatible` property?**
A: A string (or list of strings) identifying the hardware. The kernel matches it against driver `of_match_table` entries. Convention: `"manufacturer,device"`. Multiple entries enable fallback: `compatible = "vendor,uart-v2", "vendor,uart"`.

**Q3: How does a driver access a DT-described register region?**
A: DT `reg = <0x2000000 0x1000>` → kernel creates IORESOURCE_MEM → driver calls `devm_platform_ioremap_resource(pdev, 0)` → returns `void __iomem *` pointer for readl/writel.

**Q4: What is the `status` property?**
A: Controls whether a node is enabled. `"okay"` means the device exists and should be configured. `"disabled"` means it exists but shouldn't be used. SoC .dtsi sets most nodes to "disabled"; board .dts enables the ones actually connected.

---

*Next: [Chapter 14 — Hardware Register Access](Chapter_14_Hardware_Register_Access.md)*
