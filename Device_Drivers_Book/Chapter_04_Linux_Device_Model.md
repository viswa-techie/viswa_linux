# Chapter 4: Linux Device Model

## Chapter Overview

The Linux Device Model (LDM) is the unified framework that ties together buses, devices, drivers, and classes into a coherent hierarchy. Introduced in Linux 2.6, it is the backbone of how every driver finds its device, manages power, and exposes information to user space.

---

## 4.1 Overview of Linux Device Model

The device model provides:
1. **Unified device hierarchy** — every device has a parent, forming a tree
2. **Device-driver matching** — buses match devices to compatible drivers
3. **Power management** — walk the tree to suspend/resume in correct order
4. **User-space visibility** — sysfs exposes the entire model as a filesystem
5. **Hot-plug support** — uevents notify user space of device appearance/removal

---

## 4.2 Device Abstraction in Linux

Every hardware component is represented by a `struct device`:

```c
/* include/linux/device.h */
struct device {
    struct kobject kobj;            /* sysfs representation */
    struct device *parent;          /* parent in device tree */
    struct bus_type *bus;           /* bus this device is on */
    struct device_driver *driver;   /* driver bound to this device */
    struct device_node *of_node;    /* Device Tree node */
    struct class *class;            /* device class (tty, input, net...) */
    dev_t devt;                     /* major/minor device number */
    void *platform_data;            /* legacy platform data */
    void *driver_data;              /* driver private data */
    struct dev_pm_info power;       /* power management info */
    struct dev_pm_domain *pm_domain;
    const struct dma_map_ops *dma_ops;
    u64 *dma_mask;
    /* ... many more fields */
};
```

### Using driver_data

```c
/* Store private data */
struct my_priv {
    void __iomem *regs;
    int irq;
    struct clk *clk;
};

static int my_probe(struct platform_device *pdev)
{
    struct my_priv *priv;
    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    platform_set_drvdata(pdev, priv);   /* same as dev_set_drvdata() */
    return 0;
}

static void my_remove(struct platform_device *pdev)
{
    struct my_priv *priv = platform_get_drvdata(pdev);
    /* priv freed automatically (devm_kzalloc) */
}
```

---

## 4.3 Device Hierarchy in the Kernel

Devices form a tree rooted at the system bus:

```
/sys/devices/
├── LNXSYSTM:00/                  ← ACPI root (x86)
│   └── LNXSYBUS:00/
│       ├── PNP0A08:00/           ← PCI root complex
│       │   └── 0000:00:1f.0/    ← PCI device (LPC bridge)
│       │       └── serial8250/   ← UART on LPC
│       └── PNP0C02:00/           ← System board
├── platform/                      ← Platform bus
│   ├── serial8250/
│   ├── i8042/                     ← PS/2 controller
│   └── my-device@10000000/
├── pci0000:00/                    ← PCI domain
│   └── 0000:00:02.0/            ← GPU
│       └── drm/
│           └── card0/
└── virtual/
    └── misc/
        └── vhost-net/
```

### ARM/Embedded Hierarchy (Device Tree based)
```
/sys/devices/
├── platform/
│   ├── soc/
│   │   ├── 2000000.uart/        ← UART at address 0x2000000
│   │   ├── 2010000.i2c/         ← I2C controller
│   │   │   └── 0-0050/          ← I2C device at addr 0x50
│   │   │       └── eeprom/
│   │   ├── 2020000.spi/         ← SPI controller
│   │   │   └── spi0.0/          ← SPI device CS=0
│   │   └── 2030000.ethernet/
│   └── gpio-leds/
```

---

## 4.4 Device Objects

### struct device Lifecycle

```
device_initialize()          ← allocate, set defaults
       │
       ▼
device_add()                ← add to bus, sysfs, trigger uevent
       │
       ▼  [bus->match() finds a driver]
driver_probe_device()       ← call driver's probe()
       │
       ▼  [device in use]
       ...
       │
       ▼  [removal]
device_del()                ← remove from bus, sysfs
       │
       ▼
put_device()               ← drop reference, free when refcount=0
```

### device_register() = device_initialize() + device_add()

```c
/* drivers/base/core.c */
int device_register(struct device *dev)
{
    device_initialize(dev);
    return device_add(dev);
}
```

---

## 4.5 Driver Objects

```c
/* include/linux/device/driver.h */
struct device_driver {
    const char *name;
    const struct bus_type *bus;
    struct module *owner;
    const char *mod_name;

    int (*probe)(struct device *dev);
    void (*remove)(struct device *dev);
    void (*shutdown)(struct device *dev);
    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);

    const struct of_device_id *of_match_table;  /* DT matching */
    const struct acpi_device_id *acpi_match_table;

    struct driver_private *p;  /* private data */
};
```

### Driver Registration

```c
/* Bus-specific wrappers are preferred: */
platform_driver_register(&my_platform_driver);
i2c_add_driver(&my_i2c_driver);
pci_register_driver(&my_pci_driver);
usb_register(&my_usb_driver);
spi_register_driver(&my_spi_driver);

/* Or the generic: */
driver_register(&my_driver);
```

---

## 4.6 Bus Objects

```c
/* include/linux/device/bus.h */
struct bus_type {
    const char *name;            /* "platform", "pci", "i2c"... */
    const char *dev_name;

    int (*match)(struct device *dev, struct device_driver *drv);
    int (*probe)(struct device *dev);
    void (*remove)(struct device *dev);
    void (*shutdown)(struct device *dev);

    int (*uevent)(const struct device *dev, struct kobj_uevent_env *env);
    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);

    const struct dev_pm_ops *pm;
};
```

### How Buses Match Devices to Drivers

```
                 Bus Match Function
                 ──────────────────
          Device appears (or driver registers)
                        │
                        ▼
              bus->match(dev, drv)
                        │
              ┌─────────┴─────────┐
              │                   │
         return 1              return 0
        (MATCH!)             (no match)
              │
              ▼
        bus->probe(dev)
         or drv->probe(dev)
```

### Match Functions by Bus Type

| Bus | Match Logic | Match Table |
|-----|-------------|-------------|
| **Platform** | Compare `compatible` string (DT), name, or ACPI ID | `of_match_table`, `acpi_match_table` |
| **PCI** | Compare vendor_id + device_id | `pci_device_id` table |
| **USB** | Compare vendor_id + product_id + class | `usb_device_id` table |
| **I2C** | Compare `compatible` (DT) or name | `of_match_table`, `i2c_device_id` |
| **SPI** | Compare `compatible` (DT) or name | `of_match_table`, `spi_device_id` |

```c
/* Example: Platform bus match (drivers/base/platform.c) */
static int platform_match(struct device *dev, struct device_driver *drv)
{
    struct platform_device *pdev = to_platform_device(dev);

    /* 1. Try Device Tree matching first */
    if (of_driver_match_device(dev, drv))
        return 1;

    /* 2. Try ACPI matching */
    if (acpi_driver_match_device(dev, drv))
        return 1;

    /* 3. Try ID table matching */
    if (pdrv->id_table)
        return platform_match_id(pdrv->id_table, pdev) != NULL;

    /* 4. Fall back to name matching */
    return (strcmp(pdev->name, drv->name) == 0);
}
```

---

## 4.7 Class Objects

Classes group devices by **function** rather than by how they connect:

```c
/* include/linux/device/class.h */
struct class {
    const char *name;            /* "tty", "input", "net", "block"... */
    void (*dev_release)(struct device *dev);
    char *(*devnode)(const struct device *dev, umode_t *mode);
    /* ... */
};
```

### Example: The Same Physical Device in Bus vs Class View

```
BUS VIEW (how it connects):     CLASS VIEW (what it does):
/sys/bus/                        /sys/class/
├── pci/                         ├── net/
│   └── devices/                 │   └── eth0 → ../../devices/pci.../net/eth0
│       └── 0000:02:00.0/       ├── input/
├── usb/                         │   └── input3 → ../../devices/usb.../input3
│   └── devices/                 ├── tty/
│       └── 1-2/                 │   └── ttyACM0 → ../../devices/usb.../ttyACM0
└── i2c/                         └── hwmon/
    └── devices/                     └── hwmon0 → ../../devices/platform.../hwmon0
        └── 0-0048/
```

A single device (e.g., USB serial adapter) appears in:
- `/sys/bus/usb/devices/1-2` (bus view: how it connects)
- `/sys/class/tty/ttyACM0` (class view: what it provides)

---

## 4.8 sysfs Representation of Devices

sysfs is the **user-visible manifestation** of the device model:

```
/sys/
├── bus/             ← One directory per registered bus_type
│   ├── platform/
│   │   ├── devices/    ← Symlinks to /sys/devices/...
│   │   └── drivers/    ← Registered platform drivers
│   │       ├── my-driver/
│   │       │   ├── bind        ← Write device name to force bind
│   │       │   ├── unbind      ← Write device name to force unbind
│   │       │   └── uevent
│   │       └── ...
│   ├── pci/
│   └── i2c/
├── class/           ← One directory per device class
│   ├── tty/
│   ├── input/
│   └── net/
├── devices/         ← Actual device hierarchy (canonical location)
│   ├── platform/
│   │   └── my-device/
│   │       ├── driver → ../../../bus/platform/drivers/my-driver
│   │       ├── subsystem → ../../../bus/platform
│   │       ├── power/
│   │       │   ├── runtime_status
│   │       │   ├── control
│   │       │   └── autosuspend_delay_ms
│   │       ├── uevent
│   │       ├── my_custom_attr    ← Driver-created sysfs attribute
│   │       └── of_node → ../../../firmware/devicetree/...
│   └── ...
├── firmware/        ← Device Tree, ACPI tables
│   └── devicetree/
│       └── base/
└── module/          ← One directory per loaded module
    └── my_driver/
        ├── parameters/
        │   └── debug_level
        └── refcnt
```

### Creating sysfs Attributes

```c
/* Simple read-only attribute */
static ssize_t status_show(struct device *dev,
                           struct device_attribute *attr, char *buf)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    return sysfs_emit(buf, "%d\n", priv->status);
}

/* Simple read-write attribute */
static ssize_t enable_store(struct device *dev,
                            struct device_attribute *attr,
                            const char *buf, size_t count)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    int val;
    if (kstrtoint(buf, 10, &val))
        return -EINVAL;
    priv->enabled = val;
    return count;
}

static DEVICE_ATTR_RO(status);
static DEVICE_ATTR_RW(enable);

static struct attribute *my_attrs[] = {
    &dev_attr_status.attr,
    &dev_attr_enable.attr,
    NULL,
};
ATTRIBUTE_GROUPS(my);

/* In driver definition: */
static struct platform_driver my_driver = {
    .driver = {
        .name = "my-driver",
        .dev_groups = my_groups,   /* Auto-created on probe */
    },
    .probe = my_probe,
    .remove = my_remove,
};
```

---

## 4.9 Device-Driver Binding Process

### Complete Binding Flow

```
  ┌──────────────────┐              ┌──────────────────┐
  │  New Device Added │              │ New Driver Loaded │
  │  (device_add())   │              │(driver_register())│
  └────────┬─────────┘              └────────┬─────────┘
           │                                  │
           ▼                                  ▼
  Iterate all drivers on bus         Iterate all devices on bus
           │                                  │
           └──────────┬───────────────────────┘
                      │
                      ▼
            bus->match(dev, drv)
                      │
               ┌──────┴──────┐
               │  Match?     │
               │  YES    NO  │
               └──┬──────┬───┘
                  │      └──── try next
                  ▼
          driver_probe_device()
                  │
                  ▼
          really_probe()              ← drivers/base/dd.c
                  │
                  ├── dev->bus->probe(dev)     or
                  ├── drv->probe(dev)
                  │
                  ▼
          [ driver's probe() runs ]
                  │
           ┌──────┴──────┐
           │ probe returns│
           │  0      <0   │
           └──┬──────┬───┘
              │      └──── binding fails, try next driver
              ▼
      dev->driver = drv          ← binding complete
      sysfs link created
      uevent sent
```

### really_probe() Source (simplified)

```c
/* drivers/base/dd.c */
static int really_probe(struct device *dev, struct device_driver *drv)
{
    /* ... */
    dev->driver = drv;

    if (dev->bus->probe)
        ret = dev->bus->probe(dev);
    else if (drv->probe)
        ret = drv->probe(dev);

    if (ret) {
        dev->driver = NULL;
        return ret;
    }

    driver_bound(dev);     /* Create sysfs links, send uevent */
    return 0;
}
```

### Deferred Probing

When probe needs a resource not yet available (e.g., regulator, clock, GPIO provider):

```c
static int my_probe(struct platform_device *pdev)
{
    struct clk *clk = devm_clk_get(&pdev->dev, "main");
    if (IS_ERR(clk)) {
        if (PTR_ERR(clk) == -EPROBE_DEFER)
            return -EPROBE_DEFER;   /* Try again later */
        return PTR_ERR(clk);
    }
    /* ... continue probe ... */
}
```

```
Probe returns -EPROBE_DEFER
       │
       ▼
  Device added to deferred_probe_list
       │
       ▼  [later: another driver probes successfully]
  deferred_probe_work_func() triggers
       │
       ▼
  Re-attempt probe for deferred devices
```

---

## The kobject Foundation

Everything in the device model is built on `kobject`:

```
                    kobject
                   /       \
             device          driver
            /     \
     platform_    pci_
     device       dev

kobject → sysfs directory
kset    → group of kobjects (sysfs subdirectory)
ktype   → defines sysfs attributes + release function
```

```c
/* include/linux/kobject.h */
struct kobject {
    const char *name;
    struct list_head entry;
    struct kobject *parent;
    struct kset *kset;
    const struct kobj_type *ktype;
    struct kernfs_node *sd;     /* sysfs directory entry */
    struct kref kref;           /* reference count */
    /* ... */
};
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| `drivers/base/core.c` | `device_add()`, `device_register()`, `device_del()` |
| `drivers/base/bus.c` | `bus_register()`, device-driver matching loop |
| `drivers/base/dd.c` | `really_probe()`, deferred probe logic |
| `drivers/base/driver.c` | `driver_register()` |
| `drivers/base/class.c` | Class registration |
| `drivers/base/platform.c` | Platform bus match/probe |
| `include/linux/device.h` | `struct device` definition |
| `include/linux/kobject.h` | `struct kobject` |
| `lib/kobject.c` | kobject implementation |
| `fs/sysfs/` | sysfs filesystem |

---

## OS Comparison

| Aspect | Linux Device Model | Windows PnP | macOS IOKit | Zephyr |
|--------|-------------------|-------------|-------------|--------|
| Core concept | bus/device/driver hierarchy | Device stack + PnP manager | IOService class tree | devicetree + device array |
| Matching | bus->match() (DT/ACPI/ID table) | INF file + HW ID | IOKit matching dictionary | DT compatible string |
| Hierarchy visibility | sysfs (/sys/) | Device Manager | IORegistryExplorer | Not runtime-visible |
| Deferred probe | -EPROBE_DEFER | PnP manages ordering | IOKit dependency tracking | CONFIG_ ordering |
| Power integration | PM callbacks in device_driver | PnP power IRP | IOPMPowerState | PM subsystem callbacks |
| Hot-plug | Uevents → udev | PnP events → SetupAPI | IOKit notifications | Not typical |

---

## Interview Questions

**Q1: Explain the relationship between bus, device, driver, and class.**
A: Bus represents a communication path (PCI, USB, I2C). Device represents hardware connected to a bus. Driver contains the code to operate a device. Class groups devices by function (net, tty, input). A bus matches a device to its driver via match(); class provides functional grouping orthogonal to bus topology.

**Q2: How does device-driver binding work?**
A: When a device appears (device_add) or driver registers (driver_register), the bus iterates the opposite list. For each candidate pair, bus->match() checks compatibility (DT compatible, PCI vendor/device ID, etc.). On match, really_probe() calls drv->probe(). If probe returns 0, binding is complete (dev->driver set, sysfs links created).

**Q3: What is deferred probing and when does it happen?**
A: When probe() needs a resource provided by another driver that hasn't probed yet (clock, regulator, GPIO), it returns -EPROBE_DEFER. The kernel queues the device and retries after other drivers probe. This handles arbitrary driver initialization ordering.

**Q4: What is a kobject?**
A: The base object for the device model. Provides reference counting (kref), sysfs directory representation, parent-child hierarchy, and uevent notification. Every struct device, struct driver, struct bus_type contains a kobject.

**Q5: How is sysfs structured and what does it expose?**
A: sysfs (/sys) is a virtual filesystem mirroring the device model. /sys/bus/ shows buses and their devices/drivers. /sys/devices/ shows the actual device hierarchy. /sys/class/ groups by function. /sys/module/ shows loaded modules. Drivers can add custom attributes (files) for runtime configuration.

**Q6: What's the difference between /sys/bus/X/devices and /sys/devices?**
A: /sys/devices/ is the canonical (real) location of device directories. /sys/bus/X/devices/ contains symlinks to the real entries in /sys/devices/. /sys/class/Y/ also contains symlinks. The real hierarchy reflects physical topology.

**Q7: Describe the probe() function's responsibilities.**
A: Probe maps hardware resources (ioremap, request_irq, clk_get), initializes the device, allocates private data, registers with subsystems (cdev_add, register_netdev, input_register_device), creates sysfs attributes, and enables the device. Must clean up everything on error.

---

## Summary

| Component | Role | Key Structure | sysfs Location |
|-----------|------|--------------|----------------|
| **Bus** | Communication path | `struct bus_type` | /sys/bus/NAME/ |
| **Device** | Hardware instance | `struct device` | /sys/devices/.../NAME/ |
| **Driver** | Code to operate device | `struct device_driver` | /sys/bus/NAME/drivers/DRV/ |
| **Class** | Functional grouping | `struct class` | /sys/class/NAME/ |
| **kobject** | Underlying sysfs object | `struct kobject` | Each sysfs directory |
| **Match** | Device↔Driver pairing | `bus->match()` | bind/unbind files |
| **Probe** | Driver initialization | `drv->probe()` | Triggered on match |

---

*Next: [Chapter 5 — Types of Device Drivers](Chapter_05_Types_of_Device_Drivers.md)*
