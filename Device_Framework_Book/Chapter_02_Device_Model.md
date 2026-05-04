# Chapter 2: Linux Device Model Overview

## Learning Goals
- Understand the Linux device model architecture (bus-device-driver)
- Know struct device, struct device_driver, struct bus_type
- Grasp sysfs and kobject foundations
- Understand device registration and matching flow

---

## 2.1 Device Model Architecture

```
Linux Device Model — The Foundation of All Frameworks:

             ┌──────────────────────────────┐
             │         sysfs (/sys)          │
             │  Exposes entire device model  │
             │  to user space as a tree      │
             └──────────────┬───────────────┘
                            │
┌───────────────────────────▼───────────────────────────┐
│                    DEVICE MODEL                        │
│                                                       │
│   ┌───────────┐    ┌───────────┐    ┌───────────┐    │
│   │  struct    │    │  struct   │    │  struct    │    │
│   │ bus_type   │    │  device   │    │device_drv  │    │
│   │           │◄───│           │───►│           │    │
│   │ .match()  │    │ .bus      │    │ .bus      │    │
│   │ .probe()  │    │ .driver   │    │ .probe()  │    │
│   │ .remove() │    │ .parent   │    │ .remove() │    │
│   └───────────┘    │ .of_node  │    │ .of_match │    │
│                    └───────────┘    └───────────┘    │
│                                                       │
│   Every device has: a bus, a driver, a parent         │
│   The bus matches devices to drivers                  │
│   The driver's probe() initializes the device         │
└───────────────────────────────────────────────────────┘
```

---

## 2.2 struct device — The Device Object

```c
/* include/linux/device.h — struct device (simplified) */
struct device {
    struct kobject kobj;              /* sysfs representation */
    struct device *parent;            /* parent device */
    struct bus_type *bus;             /* bus this device is on */
    struct device_driver *driver;     /* driver that bound this device */
    struct device_node *of_node;      /* device tree node */

    void *platform_data;             /* legacy platform data */
    void *driver_data;               /* driver-private data */

    struct dev_pm_info power;         /* power management info */
    struct dev_pm_domain *pm_domain; /* PM domain */

    const struct dma_map_ops *dma_ops; /* DMA operations */
    u64 *dma_mask;                    /* DMA addressing mask */

    struct list_head devres_head;     /* managed resources */
    struct class *class;              /* device class */
    dev_t devt;                       /* major:minor */
};

/* Every hardware component is represented as a struct device */
/* Examples: CPU, GPIO controller, I2C adapter, camera sensor */
```

```
Device Hierarchy (/sys/devices/):

/sys/devices/
├── platform/                    ← Platform bus devices
│   ├── 12400000.i2c/           ← I2C controller
│   │   └── i2c-0/              ← I2C adapter
│   │       └── 0-0020/         ← I2C client (sensor)
│   ├── 13200000.spi/           ← SPI controller
│   └── 11000000.gpio/          ← GPIO controller
├── pci0000:00/                  ← PCI bus
│   └── 0000:01:00.0/           ← PCI device (GPU)
│       └── drm/
│           └── card0            ← DRM device
└── virtual/                     ← Virtual devices
    ├── input/
    │   └── input0               ← Input device
    └── sound/
        └── card0                ← Sound card
```

---

## 2.3 struct device_driver — The Driver Object

```c
/* include/linux/device/driver.h */
struct device_driver {
    const char *name;                    /* driver name */
    const struct bus_type *bus;           /* bus type */

    struct module *owner;                /* THIS_MODULE */

    const struct of_device_id *of_match_table;  /* DT matching */
    const struct acpi_device_id *acpi_match_table;

    int (*probe)(struct device *dev);    /* called when matched */
    void (*remove)(struct device *dev);  /* called on unbind */

    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);

    const struct dev_pm_ops *pm;         /* PM callbacks */
};

/* Platform driver — most common on embedded systems */
struct platform_driver {
    int (*probe)(struct platform_device *pdev);
    int (*remove)(struct platform_device *pdev);
    struct device_driver driver;         /* embedded base driver */
};
```

---

## 2.4 struct bus_type — The Bus Object

```c
/* include/linux/device/bus.h */
struct bus_type {
    const char *name;                    /* bus name ("platform", "i2c") */

    int (*match)(struct device *dev,
                 struct device_driver *drv);  /* match device to driver */

    int (*probe)(struct device *dev);     /* called after match */
    void (*remove)(struct device *dev);

    int (*uevent)(const struct device *dev,
                  struct kobj_uevent_env *env); /* hotplug event */

    const struct dev_pm_ops *pm;
};
```

```
Bus Types in Linux:

┌──────────────┬────────────────────┬──────────────────────────┐
│ Bus Type     │ Match Method       │ Used For                 │
├──────────────┼────────────────────┼──────────────────────────┤
│ platform     │ DT compatible /    │ SoC peripherals (most    │
│              │ name match         │ embedded devices)        │
├──────────────┼────────────────────┼──────────────────────────┤
│ i2c          │ DT compatible /    │ I2C sensors, PMICs,      │
│              │ i2c_device_id      │ touchscreens             │
├──────────────┼────────────────────┼──────────────────────────┤
│ spi          │ DT compatible /    │ SPI flash, ADC, display  │
│              │ spi_device_id      │                          │
├──────────────┼────────────────────┼──────────────────────────┤
│ pci          │ Vendor:Device ID   │ GPUs, NICs, NVMe         │
├──────────────┼────────────────────┼──────────────────────────┤
│ usb          │ Vendor:Product ID  │ USB devices              │
├──────────────┼────────────────────┼──────────────────────────┤
│ amba         │ Peripheral ID      │ ARM AMBA/APB devices     │
└──────────────┴────────────────────┴──────────────────────────┘
```

---

## 2.5 Device Registration Flow

```
Device-Driver Binding Flow:

1. DEVICE REGISTRATION:
   ┌──────────────────────────────────────────────────┐
   │  Device Tree parsed during boot                  │
   │  or platform_device_register() called            │
   │       │                                          │
   │       ▼                                          │
   │  device_add(dev)                                 │
   │       │                                          │
   │       ├── kobject_add() → sysfs entry created    │
   │       ├── bus_add_device() → add to bus list     │
   │       └── bus_probe_device()                     │
   │           └── Try matching with registered drvs  │
   └──────────────────────────────────────────────────┘

2. DRIVER REGISTRATION:
   ┌──────────────────────────────────────────────────┐
   │  module_init() → platform_driver_register()      │
   │       │                                          │
   │       ▼                                          │
   │  driver_register(drv)                            │
   │       │                                          │
   │       ├── bus_add_driver() → add to bus list     │
   │       └── driver_attach()                        │
   │           └── Try matching with unbound devices  │
   └──────────────────────────────────────────────────┘

3. MATCHING (bus->match):
   ┌──────────────────────────────────────────────────┐
   │  For platform bus:                               │
   │  1. of_match_table → compare DT "compatible"     │
   │  2. acpi_match_table → compare ACPI IDs          │
   │  3. id_table → compare name strings              │
   │  4. driver.name == device.name (legacy)           │
   │                                                  │
   │  If match found:                                 │
   │       │                                          │
   │       ▼                                          │
   │  driver_probe_device()                           │
   │       └── drv->probe(dev)                        │
   │           ├── Map registers (devm_ioremap)       │
   │           ├── Request IRQs (devm_request_irq)    │
   │           ├── Register with framework            │
   │           └── HW initialization                  │
   └──────────────────────────────────────────────────┘
```

---

## 2.6 kobject and sysfs

```
kobject — Foundation of Device Model:

Every device model object contains a kobject:
  struct device { struct kobject kobj; ... };

kobject provides:
  ├── Reference counting (kref)
  ├── sysfs directory entry
  ├── uevent generation (for udev)
  └── Parent-child relationships

sysfs tree (/sys/):
┌──────────────────────────────────────────┐
│ /sys/                                    │
│ ├── bus/                                 │
│ │   ├── platform/                        │
│ │   │   ├── devices/ → links to devices  │
│ │   │   └── drivers/ → registered drivers│
│ │   ├── i2c/                             │
│ │   ├── spi/                             │
│ │   └── pci/                             │
│ ├── class/                               │
│ │   ├── input/                           │
│ │   ├── sound/                           │
│ │   ├── video4linux/                     │
│ │   └── drm/                            │
│ ├── devices/                             │
│ │   └── (actual device hierarchy)        │
│ └── module/                              │
│     └── (loaded kernel modules)          │
└──────────────────────────────────────────┘
```

---

## 2.7 devm_ Managed Resources

```c
/* devm_ — Device-Managed Resources (automatic cleanup) */

static int my_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    void __iomem *regs;
    int irq, ret;

    /* Managed ioremap — freed automatically on remove/error */
    regs = devm_ioremap_resource(dev,
                platform_get_resource(pdev, IORESOURCE_MEM, 0));
    if (IS_ERR(regs))
        return PTR_ERR(regs);

    /* Managed IRQ — freed automatically */
    irq = platform_get_irq(pdev, 0);
    ret = devm_request_irq(dev, irq, my_isr, 0, "my-dev", priv);
    if (ret)
        return ret;

    /* Managed memory — freed automatically */
    priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    /* Managed clock — released automatically */
    clk = devm_clk_get(dev, "core");
    if (IS_ERR(clk))
        return PTR_ERR(clk);

    /* No need for explicit cleanup in remove() or error paths! */
    /* Resources freed in reverse order when device is removed */
    return 0;
}
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| struct device | include/linux/device.h | Core device structure |
| device_add() | drivers/base/core.c | Device registration |
| driver_register() | drivers/base/driver.c | Driver registration |
| struct bus_type | include/linux/device/bus.h | Bus abstraction |
| platform_driver | include/linux/platform_device.h | Platform driver |
| kobject | include/linux/kobject.h | Kernel object |
| devres (devm_) | drivers/base/devres.c | Managed resources |
| of_platform | drivers/of/platform.c | DT device creation |

---

## Interview Questions

**Q1: Explain the Linux device model's bus-device-driver architecture.**
A: The device model has three core objects: (1) **bus_type** — represents a communication bus (platform, I2C, SPI, PCI, USB). It defines how to match devices to drivers. (2) **device** — represents a hardware component. Created from device tree nodes or explicit registration. Has a parent, bus, and optional driver. (3) **device_driver** — implements hardware-specific operations. Registered with a bus and matched to devices via compatible strings, vendor/product IDs, or name. When `bus->match()` succeeds, `driver->probe()` is called.

**Q2: What is the purpose of devm_ (device-managed) resources?**
A: `devm_` functions (devm_kzalloc, devm_ioremap, devm_request_irq, devm_clk_get) allocate resources that are automatically freed when the device is removed or when probe fails. They eliminate the need for explicit cleanup in error paths and remove() functions, preventing resource leaks. Resources are freed in reverse allocation order. This significantly reduces driver bugs — about 30% of driver patches fix resource leak issues that devm_ would have prevented.

**Q3: How does device tree result in device-driver binding?**
A: (1) Bootloader passes DTB to kernel. (2) `of_platform_default_populate()` walks DT nodes and creates `platform_device` for each node with a "compatible" property. (3) When a driver registers with `platform_driver_register()`, the bus matches its `of_match_table` against device compatible strings. (4) On match, `driver->probe()` is called with the device. (5) In probe, the driver uses `dev->of_node` to read device-specific properties (reg, interrupts, clocks) from the DT.

**Q4: What is sysfs and how does it relate to the device model?**
A: sysfs (`/sys`) is a virtual filesystem that exposes the kernel's device model to user space. Every `struct device` gets a directory under `/sys/devices/`. The `/sys/bus/` tree organizes devices by bus type. `/sys/class/` groups devices by function (video4linux, sound, input). User-space tools like udev read sysfs to create device nodes and apply permissions. Attributes in sysfs allow reading/writing device parameters (e.g., `/sys/class/backlight/brightness`).

---

## Summary

- The Linux device model is built on three pillars: bus, device, and driver
- `struct device` represents hardware; `struct device_driver` implements operations; `struct bus_type` matches them
- Device tree nodes are converted to platform_device structs during boot
- The bus match function compares device compatible strings with driver of_match_table
- On match, probe() is called — the driver initializes hardware and registers with frameworks
- sysfs exposes the entire device model as a filesystem under /sys
- devm_ managed resources provide automatic cleanup, eliminating resource leak bugs

---

*Next: [Chapter 3 — Linux Media Framework (V4L2)](Chapter_03_V4L2_Media.md)*
