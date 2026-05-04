# Chapter 19: Device Initialization

## Learning Goals
- Understand how drivers register and devices are discovered during boot
- Know the driver-device binding mechanism
- Understand platform device initialization and deferred probing

---

## 19.1 Driver Initialization Sequence

```
Driver Init During Boot:

do_initcalls()
     │
     ├── core_initcall:   driver_init()
     │   ├── devices_init()    ← /sys/devices
     │   ├── buses_init()      ← /sys/bus
     │   ├── classes_init()    ← /sys/class
     │   ├── platform_bus_init() ← Platform bus registered
     │   └── of_core_init()    ← Device Tree core
     │
     ├── postcore_initcall: bus type registrations
     │   ├── pci_driver_init()  ← PCI bus
     │   ├── i2c_init()         ← I2C bus
     │   ├── spi_init()         ← SPI bus
     │   └── usb_init()         ← USB bus
     │
     ├── subsys_initcall: framework init
     │   ├── input_init()       ← Input subsystem
     │   ├── net_dev_init()     ← Network subsystem
     │   └── v4l2_init()        ← Video4Linux
     │
     └── device_initcall: individual drivers (module_init)
         ├── my_i2c_driver_init()
         ├── my_spi_driver_init()
         ├── my_platform_driver_init()
         └── ... hundreds of drivers ...
```

---

## 19.2 Device Enumeration

Devices are discovered through different mechanisms depending on the bus type:

```
Device Discovery Methods:

1. Device Tree (DT) — ARM/embedded
   ┌─────────────────────────────────────────┐
   │  DTB parsed during setup_arch()         │
   │  unflatten_device_tree() creates        │
   │  device_node tree in memory             │
   │                                         │
   │  of_platform_populate():                │
   │  Walks DT → creates platform_device    │
   │  for each node with "compatible"         │
   └─────────────────────────────────────────┘

2. ACPI — x86/server
   ┌─────────────────────────────────────────┐
   │  ACPI tables parsed from firmware       │
   │  DSDT/SSDT describe devices             │
   │  acpi_device created for each entry     │
   │  Bound to drivers via _HID/_CID         │
   └─────────────────────────────────────────┘

3. Bus Enumeration — PCI, USB
   ┌─────────────────────────────────────────┐
   │  PCI: Scan config space at boot         │
   │  USB: Hot-plug enumeration              │
   │  Hardware provides Vendor/Device ID     │
   │  Matched against driver ID tables       │
   └─────────────────────────────────────────┘
```

```c
/* Device Tree → platform_device creation */
/* Called during init: of_platform_default_populate() */

/* For this DT node: */
/*
    soc {
        uart0: serial@40010000 {
            compatible = "vendor,uart-v2";
            reg = <0x40010000 0x1000>;
            interrupts = <GIC_SPI 45 IRQ_TYPE_LEVEL_HIGH>;
            clocks = <&clk CLK_UART0>;
            status = "okay";
        };
    };
*/

/* Kernel creates a platform_device:
   - pdev->name = "40010000.serial"
   - pdev->dev.of_node = <device_node for uart0>
   - pdev->resource[0] = IORESOURCE_MEM: 0x40010000, size 0x1000
   - pdev->resource[1] = IORESOURCE_IRQ: IRQ 45
*/
```

---

## 19.3 Driver-Device Binding

```
The Driver-Device Match & Probe Process:

1. Bus has a list of DEVICES and DRIVERS

   Platform Bus:
   ┌─────────────────────────────────────────────────────┐
   │  Devices (from DT):          Drivers (from initcall):│
   │  ├── 40010000.serial         ├── vendor-uart driver  │
   │  ├── 40020000.i2c            ├── vendor-i2c driver   │
   │  └── 40030000.gpio           └── vendor-gpio driver  │
   └─────────────────────────────────────────────────────┘

2. When a new DEVICE appears OR a new DRIVER registers:
   bus->match(device, driver) is called

3. For platform bus, match checks:
   a. Device Tree "compatible" vs driver's of_match_table  ← Most common
   b. ACPI _HID vs driver's acpi_match_table
   c. Platform device name vs driver name (legacy)
   d. ID table match

4. If match succeeds → call driver->probe(device)

                  Device appears
                      │
                      ▼
              ┌───────────────┐
              │  Bus match()  │─── No match → device waits
              └───────┬───────┘
                      │ Match!
                      ▼
              ┌───────────────┐
              │ driver->probe()│─── Failure → returns error
              └───────┬───────┘    (may defer: -EPROBE_DEFER)
                      │ Success
                      ▼
              ┌───────────────┐
              │ Device bound  │
              │ to driver     │
              │ (functional)   │
              └───────────────┘
```

```c
/* Match and probe example */
static const struct of_device_id my_uart_of_match[] = {
    { .compatible = "vendor,uart-v2" },
    { .compatible = "vendor,uart-v1" },
    { /* sentinel */ }
};

static int my_uart_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    void __iomem *base;
    int irq;

    /* Get resources from DT */
    base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(base))
        return PTR_ERR(base);

    irq = platform_get_irq(pdev, 0);
    if (irq < 0)
        return irq;

    /* Get optional dependencies — may defer */
    struct clk *clk = devm_clk_get(dev, NULL);
    if (IS_ERR(clk))
        return PTR_ERR(clk);  /* -EPROBE_DEFER if clock not ready */

    dev_info(dev, "probed: base=%p irq=%d\n", base, irq);
    return 0;
}

static struct platform_driver my_uart_driver = {
    .probe  = my_uart_probe,
    .remove = my_uart_remove,
    .driver = {
        .name = "vendor-uart",
        .of_match_table = my_uart_of_match,
    },
};
module_platform_driver(my_uart_driver);
```

---

## 19.4 Platform Device Initialization and Deferred Probing

```
Deferred Probing — Handling Dependencies:

Problem: Driver A needs a clock provided by Driver B,
         but Driver B hasn't probed yet.

Timeline:
  do_initcalls() level 7:
    ├── Driver A probe() → devm_clk_get() → clock driver not ready
    │   └── Returns -EPROBE_DEFER
    │       └── Device A added to deferred probe list
    │
    ├── Driver B probe() → SUCCESS
    │   └── Clock framework now has the clock
    │
    └── Deferred probe retry
        └── Driver A probe() → devm_clk_get() → SUCCESS!
            └── Device A functional

Deferred Probe List:
┌──────────────────────────────────────────────────┐
│  deferred_probe_list                              │
│  ├── Device A (waiting for clock)                │
│  ├── Device C (waiting for regulator)            │
│  └── Device E (waiting for GPIO controller)       │
│                                                  │
│  After each successful probe, retry ALL deferred │
│  devices until list is empty or no progress       │
└──────────────────────────────────────────────────┘
```

```bash
# View deferred probes at runtime
cat /sys/kernel/debug/devices_deferred
# Output: List of devices still waiting for dependencies

# View driver binding status
ls /sys/bus/platform/devices/40010000.serial/
# driver → ../../../bus/platform/drivers/vendor-uart  (bound!)
# or no 'driver' symlink = unbound
```

---

## Interview Questions

**Q1: How does the kernel know which driver to use for a device?**
A: The bus match function compares device identifiers against driver tables. For DT-based platforms: the device's `compatible` string is matched against the driver's `of_match_table`. For PCI: vendor/device ID. For USB: vendor/product ID. When a match is found, the bus calls the driver's `probe()` function.

**Q2: What is deferred probing and when does it happen?**
A: Deferred probing occurs when a driver's `probe()` returns `-EPROBE_DEFER`, meaning a dependency isn't ready yet (clock, regulator, GPIO controller). The kernel adds the device to a deferred list and retries after other probes succeed. This naturally handles out-of-order initialization without requiring explicit ordering.

**Q3: What is the difference between platform_device and pci_device?**
A: Platform devices are non-discoverable SoC peripherals described in device tree or ACPI — they exist at known memory addresses. PCI devices are self-describing — the bus can enumerate them by scanning config space, and they provide vendor/device IDs. Platform devices dominate embedded; PCI dominates desktop/server.

---

## Summary

- Driver init follows initcall levels: core (driver model) → postcore (bus types) → device (drivers)
- Devices are discovered via DT (parse + populate), ACPI (table scan), or bus enumeration (PCI/USB)
- Driver-device binding: bus match function compares compatible/ID → probe() called on match
- Deferred probing handles dependency ordering — driver returns -EPROBE_DEFER, retried later
- `devm_*` APIs ensure automatic cleanup on probe failure or device removal

---

*Next: [Chapter 20 — Device Tree](Chapter_20_Device_Tree.md)*
