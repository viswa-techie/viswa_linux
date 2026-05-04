# Chapter 23: Important Kernel Data Structures

## Chapter Overview

Every driver interacts with a handful of core data structures. Understanding their fields, relationships, and lifecycle is essential for driver development. This chapter dissects the five most critical structures in depth.

---

## 23.1 struct device (include/linux/device.h)

The universal representation of any device in Linux.

```c
struct device {
    struct kobject kobj;              /* sysfs representation */
    struct device *parent;            /* Parent device (bus controller, etc.) */
    const char *init_name;            /* Initial name before kobj takes over */

    struct bus_type *bus;             /* Bus this device sits on */
    struct device_driver *driver;     /* Bound driver (NULL if unbound) */
    void *driver_data;               /* Private data for the driver */

    struct device_node *of_node;      /* Device Tree node */
    struct fwnode_handle *fwnode;     /* Firmware node (DT or ACPI) */

    dev_t devt;                       /* Major:minor for device node */
    struct class *class;              /* Device class */

    /* Power management */
    struct dev_pm_info power;
    struct dev_pm_domain *pm_domain;

    /* DMA */
    const struct dma_map_ops *dma_ops;
    u64 *dma_mask;
    u64 coherent_dma_mask;

    /* IOMMU */
    struct iommu_group *iommu_group;

    /* Resource management */
    struct list_head devres_head;     /* Managed resources (devm_*) */
};
```

### Key Fields Explained

| Field | Purpose | Driver Interaction |
|-------|---------|--------------------|
| `parent` | Device hierarchy | Set by bus during enumeration |
| `driver` | Bound driver pointer | Set during probe, cleared on remove |
| `driver_data` | Private data | `dev_set_drvdata()` / `dev_get_drvdata()` |
| `of_node` | DT node | `of_property_read_*()` |
| `devres_head` | devm_ allocations | Auto-freed on driver unbind |
| `power` | PM state | `pm_runtime_*()` APIs |
| `dma_mask` | DMA address range | Set by bus (PCI) or driver |

### Lifecycle

```
alloc_device()           → device structure allocated
device_initialize()      → kobj init, devres init
device_add()             → sysfs entry, bus notification, probe
device_del()             → remove from bus, sysfs cleanup
put_device()             → reference count → kfree
```

---

## 23.2 struct device_driver (include/linux/device/driver.h)

Represents a driver that can bind to devices.

```c
struct device_driver {
    const char *name;                 /* Driver name (matches in sysfs) */
    const struct bus_type *bus;       /* Bus this driver works on */

    struct module *owner;             /* THIS_MODULE for refcounting */

    const struct of_device_id *of_match_table;   /* Device Tree matching */
    const struct acpi_device_id *acpi_match_table; /* ACPI matching */

    int  (*probe)(struct device *dev);  /* Bind to device */
    void (*remove)(struct device *dev); /* Unbind from device */
    void (*shutdown)(struct device *dev); /* System shutdown */

    int  (*suspend)(struct device *dev, pm_message_t state);
    int  (*resume)(struct device *dev);

    const struct dev_pm_ops *pm;      /* Power management ops */

    struct driver_private *p;         /* Internal: device list, etc. */
};
```

### Matching Flow

```
Bus calls match()
    │
    ├── Platform bus: of_match_table → compatible string
    ├── PCI bus: pci_device_id table → vendor:device
    ├── USB bus: usb_device_id table → vendor:product
    ├── I2C bus: of_match_table or i2c_device_id
    └── SPI bus: of_match_table or spi_device_id
    │
    ▼ (match found)
bus_type->probe() → driver->probe(dev)
```

---

## 23.3 struct bus_type (include/linux/device/bus.h)

Defines a bus — the glue between devices and drivers.

```c
struct bus_type {
    const char *name;                 /* "platform", "pci", "usb", "i2c" */

    int (*match)(struct device *dev, struct device_driver *drv);
    int (*probe)(struct device *dev);
    void (*remove)(struct device *dev);

    int (*uevent)(const struct device *dev, struct kobj_uevent_env *env);

    const struct dev_pm_ops *pm;      /* Bus-level PM */

    /* Internal */
    struct subsys_private *p;         /* Device/driver lists */
};
```

### Bus Match Implementations

```c
/* Platform bus (simplified) */
static int platform_match(struct device *dev, struct device_driver *drv)
{
    struct platform_device *pdev = to_platform_device(dev);

    /* 1. Try OF (Device Tree) */
    if (of_driver_match_device(dev, drv))
        return 1;

    /* 2. Try ACPI */
    if (acpi_driver_match_device(dev, drv))
        return 1;

    /* 3. Try ID table */
    if (pdrv->id_table)
        return platform_match_id(pdrv->id_table, pdev) != NULL;

    /* 4. Fall back to name match */
    return strcmp(pdev->name, drv->name) == 0;
}
```

---

## 23.4 struct platform_device (include/linux/platform_device.h)

Represents a non-enumerable device (SoC peripherals, memory-mapped).

```c
struct platform_device {
    const char *name;                 /* Device name */
    int id;                           /* Instance ID (-1 for single) */
    struct device dev;                /* Embedded struct device */

    u32 num_resources;
    struct resource *resource;        /* MMIO, IRQ resources */

    const struct platform_device_id *id_entry;
};
```

### Resource Access

```c
/* Get memory resource */
struct resource *res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
void __iomem *base = devm_ioremap_resource(&pdev->dev, res);
/* Or combined: */
void __iomem *base = devm_platform_ioremap_resource(pdev, 0);

/* Get IRQ */
int irq = platform_get_irq(pdev, 0);

/* Get named IRQ */
int irq = platform_get_irq_byname(pdev, "tx");

/* Get DT properties via embedded device */
struct device_node *np = pdev->dev.of_node;
of_property_read_u32(np, "clock-frequency", &freq);
```

### How platform_device Is Created

```
Device Tree parsing (of_platform_populate)
    │
    ▼
For each node with "compatible":
    platform_device_alloc()
    platform_device_add()
        → resources from DT (reg → IORESOURCE_MEM, interrupts → IORESOURCE_IRQ)
        → dev.of_node set to DT node
        → added to platform bus → triggers matching
```

---

## 23.5 struct platform_driver (include/linux/platform_device.h)

The driver-side counterpart to `platform_device`.

```c
struct platform_driver {
    int  (*probe)(struct platform_device *pdev);
    int  (*remove)(struct platform_device *pdev);
    void (*shutdown)(struct platform_device *pdev);

    struct device_driver driver;      /* Embedded struct device_driver */

    const struct platform_device_id *id_table;
};
```

### Complete Registration Example

```c
static const struct of_device_id my_of_match[] = {
    { .compatible = "vendor,my-peripheral" },
    { /* sentinel */ },
};
MODULE_DEVICE_TABLE(of, my_of_match);

static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name           = "my-peripheral",
        .of_match_table = my_of_match,
        .pm             = &my_pm_ops,
    },
};
module_platform_driver(my_driver);
```

### module_platform_driver() Expansion

```c
/* Expands to: */
static int __init my_driver_init(void)
{
    return platform_driver_register(&my_driver);
}
module_init(my_driver_init);

static void __exit my_driver_exit(void)
{
    platform_driver_unregister(&my_driver);
}
module_exit(my_driver_exit);
```

---

## 23.6 Relationships Diagram

```
struct bus_type (platform_bus_type)
    │
    ├── device list:
    │   ├── struct platform_device A
    │   │       └── .dev (struct device)
    │   │               ├── .of_node → DT node
    │   │               ├── .driver → bound driver
    │   │               └── .driver_data → private
    │   └── struct platform_device B
    │
    └── driver list:
        ├── struct platform_driver X
        │       └── .driver (struct device_driver)
        │               ├── .of_match_table
        │               └── .pm → &my_pm_ops
        └── struct platform_driver Y
```

---

## 23.7 container_of() — The Glue

```c
/* Given pointer to embedded struct, get containing struct */
#define container_of(ptr, type, member) ({                  \
    const typeof(((type *)0)->member) *__mptr = (ptr);      \
    (type *)((char *)__mptr - offsetof(type, member));      \
})

/* Usage: */
static int my_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;   /* Embedded → parent */
    /* ... */
}

/* In file_ops, recover private data: */
static int my_open(struct inode *inode, struct file *f)
{
    struct my_dev *priv = container_of(inode->i_cdev, struct my_dev, cdev);
    f->private_data = priv;
    return 0;
}
```

---

## Kernel Source References

| File | Content |
|------|---------|
| include/linux/device.h | struct device |
| include/linux/device/driver.h | struct device_driver |
| include/linux/device/bus.h | struct bus_type |
| include/linux/platform_device.h | platform_device, platform_driver |
| drivers/base/core.c | Device lifecycle |
| drivers/base/bus.c | Bus matching/binding |
| drivers/base/platform.c | Platform bus implementation |

---

## Interview Questions

**Q1: What is the relationship between `struct device` and `struct platform_device`?**
A: `struct platform_device` *contains* a `struct device` as an embedded member (`.dev`). The platform layer is a specialization — it adds resources (MMIO, IRQ) and platform-specific ID matching on top of the generic device model.

**Q2: Why does `struct device_driver` have both `of_match_table` and `acpi_match_table`?**
A: To support both Device Tree (ARM, RISC-V, PowerPC) and ACPI (x86 servers, some ARM64) firmware descriptions. The bus `match()` tries OF first, then ACPI, then ID table, then name match — ensuring the driver works across different platforms.

**Q3: What happens to `devres_head` when a driver is unbound?**
A: All resources registered via `devm_*()` APIs are freed in reverse order. This is the managed device resource system — it prevents resource leaks even if the remove function is incomplete or missing.

**Q4: Explain `driver_data` usage pattern.**
A: In `probe()`: allocate private struct, store with `dev_set_drvdata(dev, priv)`. In all other callbacks (remove, suspend, resume, IRQ): retrieve with `dev_get_drvdata(dev)`. This gives every callback access to driver-private state.

---

*Next: [Chapter 24 — Kernel Driver APIs](Chapter_24_Driver_APIs.md)*
