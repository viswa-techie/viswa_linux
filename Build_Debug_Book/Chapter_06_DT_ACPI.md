# Chapter 6: Device Tree and ACPI

## Learning Goals
- Understand Device Tree Source (DTS) syntax and compilation
- Learn DT overlays and runtime modification
- Master ACPI tables and firmware description
- Know when to use DT vs ACPI

---

## 1. Device Tree Fundamentals

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Device Tree: data structure describing hardware         │
  │  Used by: ARM, ARM64, RISC-V, PowerPC, MIPS             │
  │  NOT used by: x86 (uses ACPI instead)                   │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ .dts (Device Tree Source)                │            │
  │  │   │ human-readable text                  │            │
  │  │   ▼                                      │            │
  │  │ dtc (Device Tree Compiler)               │            │
  │  │   │                                      │            │
  │  │   ▼                                      │            │
  │  │ .dtb (Device Tree Blob)                  │            │
  │  │   │ flattened binary                     │            │
  │  │   ▼                                      │            │
  │  │ Bootloader passes DTB to kernel          │            │
  │  │   │                                      │            │
  │  │   ▼                                      │            │
  │  │ Kernel unflatten → device tree in memory │            │
  │  │   │                                      │            │
  │  │   ▼                                      │            │
  │  │ Drivers match by compatible property     │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  DTS syntax example:                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ /dts-v1/;                                │            │
  │  │                                          │            │
  │  │ / {                                      │            │
  │  │   model = "My Board";                    │            │
  │  │   compatible = "myvendor,myboard";       │            │
  │  │   #address-cells = <2>;                  │            │
  │  │   #size-cells = <2>;                     │            │
  │  │                                          │            │
  │  │   memory@80000000 {                      │            │
  │  │     device_type = "memory";              │            │
  │  │     reg = <0x0 0x80000000 0x0 0x40000000>;│           │
  │  │   };         /* 1GB at 0x80000000 */     │            │
  │  │                                          │            │
  │  │   uart0: serial@9000000 {                │            │
  │  │     compatible = "ns16550a";             │            │
  │  │     reg = <0x0 0x9000000 0x0 0x1000>;    │            │
  │  │     interrupts = <GIC_SPI 1 IRQ_TYPE_LEVEL_HIGH>;│   │
  │  │     clock-frequency = <24000000>;        │            │
  │  │     status = "okay";                     │            │
  │  │   };                                     │            │
  │  │                                          │            │
  │  │   i2c@a000000 {                          │            │
  │  │     compatible = "vendor,i2c-ctrl";      │            │
  │  │     #address-cells = <1>;                │            │
  │  │     #size-cells = <0>;                   │            │
  │  │     reg = <0x0 0xa000000 0x0 0x1000>;    │            │
  │  │                                          │            │
  │  │     sensor@48 {                          │            │
  │  │       compatible = "ti,tmp102";          │            │
  │  │       reg = <0x48>;                      │            │
  │  │     };                                   │            │
  │  │   };                                     │            │
  │  │ };                                       │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Key properties:                                         │
  │  compatible: driver matching string (most important)    │
  │  reg: register/memory address and size                  │
  │  interrupts: interrupt specifier                        │
  │  status: "okay" (enable) or "disabled"                  │
  │  clocks, clock-names: clock references                  │
  │  #address-cells, #size-cells: child addressing          │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. DT Includes and Overlays

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  DTS include hierarchy:                                  │
  │  ┌──────────────────────────────────────────┐            │
  │  │ SoC DTSI (common for all boards):        │            │
  │  │   arch/arm64/boot/dts/qcom/sa8155p.dtsi  │            │
  │  │   └── defines SoC internals: CPUs, GIC,  │            │
  │  │       peripherals, clocks                │            │
  │  │                                          │            │
  │  │ Board DTS (board-specific):              │            │
  │  │   arch/arm64/boot/dts/qcom/sa8155p-adp.dts│          │
  │  │   └── #include "sa8155p.dtsi"            │            │
  │  │   └── overrides: enables specific UARTs, │            │
  │  │       I2C devices, display, camera       │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Override mechanism:                                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │ /* In SoC DTSI: */                       │            │
  │  │ uart0: serial@9000000 {                  │            │
  │  │     status = "disabled";  /* off by default*/│       │
  │  │ };                                       │            │
  │  │                                          │            │
  │  │ /* In board DTS: */                      │            │
  │  │ &uart0 {                                 │            │
  │  │     status = "okay";  /* enable on this board*/│     │
  │  │     pinctrl-0 = <&uart0_pins>;           │            │
  │  │ };                                       │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  DT Overlays (.dtbo):                                    │
  │  ┌──────────────────────────────────────────┐            │
  │  │ /dts-v1/;                                │            │
  │  │ /plugin/;  /* marks this as overlay */   │            │
  │  │                                          │            │
  │  │ &i2c1 {                                  │            │
  │  │     sensor@48 {                          │            │
  │  │         compatible = "ti,tmp102";        │            │
  │  │         reg = <0x48>;                    │            │
  │  │     };                                   │            │
  │  │ };                                       │            │
  │  │                                          │            │
  │  │ Applied at boot or runtime:              │            │
  │  │ - U-Boot: fdt apply overlay.dtbo         │            │
  │  │ - Linux: configfs /sys/kernel/config/    │            │
  │  │   device-tree/overlays/                  │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. ACPI (x86 Firmware Description)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ACPI = Advanced Configuration and Power Interface       │
  │  Used by x86 (and some ARM64 servers)                   │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ ACPI tables (in firmware/BIOS):          │            │
  │  │                                          │            │
  │  │ RSDP → XSDT → points to all tables:     │            │
  │  │   DSDT: main device description (AML)   │            │
  │  │   SSDT: supplemental descriptions       │            │
  │  │   MADT: interrupt controller (APIC)     │            │
  │  │   FADT: power management                │            │
  │  │   MCFG: PCIe config space               │            │
  │  │   HPET: high precision timer            │            │
  │  │   SRAT/SLIT: NUMA topology              │            │
  │  │                                          │            │
  │  │ DSDT contains AML (ACPI Machine Language):│           │
  │  │   Method(_STA) { Return(0x0F) }         │            │
  │  │   → device status (present, enabled)    │            │
  │  │                                          │            │
  │  │   Method(_CRS) {                        │            │
  │  │       IRQ(Level, ...) {14}              │            │
  │  │       IO(Decode16, 0x3F8, 0x3F8, 1, 8) │            │
  │  │   }                                     │            │
  │  │   → current resource settings           │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  DT vs ACPI:                                             │
  │  ┌──────────────┬───────────────┬──────────────────────┐│
  │  │              │ Device Tree   │ ACPI                  ││
  │  ├──────────────┼───────────────┼──────────────────────┤│
  │  │ Platform     │ ARM, RISC-V   │ x86, ARM servers     ││
  │  │ Source       │ Kernel tree   │ Firmware/BIOS        ││
  │  │ Language     │ DTS text      │ ASL → AML bytecode   ││
  │  │ Execution   │ Static data   │ Executable methods   ││
  │  │ Modifier     │ Overlays (.dtbo)│ SSDT supplements   ││
  │  │ Power mgmt  │ Separate      │ Built-in (_PS0, _PS3)││
  │  │ Shipped by   │ Kernel/BSP    │ Hardware vendor      ││
  │  └──────────────┴───────────────┴──────────────────────┘│
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does a kernel driver match with a device described in the Device Tree?**
**A:** The matching mechanism uses the `compatible` property. In the DTS: a device node has `compatible = "vendor,device-v2", "vendor,device";` — this is an ordered list from most specific to most generic. In the driver: a `struct of_device_id` match table lists compatible strings the driver supports: `{ .compatible = "vendor,device" }, { .compatible = "vendor,device-v2" }`. During kernel boot, the OF (Open Firmware) core unflatten the DTB into an in-memory tree. The platform bus iterates device nodes with `status = "okay"`, creates `struct platform_device` for each, and attempts to match against registered drivers using `of_match_device()`. Matching is done by comparing the node's compatible strings (in order) against the driver's of_device_id table. The first match wins. On match, the driver's `probe()` function is called. The driver can then read device-specific properties: `of_property_read_u32(node, "clock-frequency", &freq)`, get register addresses via `platform_get_resource()`, get IRQs via `platform_get_irq()`, and parse child nodes. The compatible string is also used for module autoloading: `MODULE_DEVICE_TABLE(of, match_table)` generates aliases so `modprobe` can automatically load the driver when a matching device is found in the DT.

**Q2: When would you use a Device Tree overlay vs modifying the base DTS?**
**A:** Base DTS modification is appropriate for permanent board features — the SoC DTSI describes fixed hardware, and the board DTS enables/configures peripherals that are always present. Overlays (`.dtbo`) are appropriate for: (1) **Add-on boards/shields**: a Raspberry Pi HAT or BeagleBone cape — the base board DTS shouldn't include hardware that's only present with an optional expansion board. The overlay adds the cape's I2C sensors, SPI flash, etc. (2) **Manufacturing variants**: same base board with different sensor or display options. Each variant has an overlay; the bootloader selects the right one based on a board ID. (3) **Runtime reconfiguration**: on some systems (e.g., FPGA-based), hardware can change after boot. Overlays can be applied via configfs at runtime to describe newly-available hardware. (4) **Android DTBO partition**: Android uses a dedicated partition for DT overlays, separating SoC vendor DTB from ODM customizations. The bootloader merges them at boot. Limitations: not all platforms support runtime overlay application (some require bootloader support). Overlay removal at runtime is possible but can be unsafe if drivers are actively using the devices.

---

## Summary

- Device Tree: describes non-discoverable hardware for ARM/RISC-V/PowerPC
- DTS → dtc → DTB → bootloader passes to kernel → unflattened to memory tree
- compatible property: primary driver matching mechanism
- DTSI: SoC common; DTS: board-specific overrides using &label references
- Overlays (.dtbo): add-on hardware, manufacturing variants, runtime changes
- ACPI: x86 firmware with executable AML bytecode; includes power management
- DT = static data (kernel modified); ACPI = executable + static (vendor firmware)

---

[Previous: Modules ←](Chapter_05_Modules.md) | [Next: GCC and Clang for Kernel →](Chapter_07_GCC_Clang.md)
