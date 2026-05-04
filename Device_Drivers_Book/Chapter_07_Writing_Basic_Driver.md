# Chapter 7: Writing a Basic Device Driver

## Chapter Overview

This chapter walks through writing a complete, minimal Linux device driver from scratch — covering the skeleton structure, essential headers, registration, and debugging techniques.

---

## 7.1 Driver Skeleton Structure

Every driver follows this basic pattern:

```c
#include <linux/module.h>          /* Required for all modules */
#include <linux/init.h>            /* __init, __exit */
#include <linux/platform_device.h> /* For platform drivers */

/* === Private Data === */
struct my_priv {
    void __iomem *base;
    int irq;
};

/* === Probe: initialize hardware === */
static int my_probe(struct platform_device *pdev) { ... }

/* === Remove: cleanup === */
static void my_remove(struct platform_device *pdev) { ... }

/* === Match table === */
static const struct of_device_id my_of_match[] = { ... };

/* === Driver structure === */
static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = { .name = "my-dev", .of_match_table = my_of_match },
};

/* === Registration macro === */
module_platform_driver(my_driver);

/* === Module metadata === */
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Author");
MODULE_DESCRIPTION("Description");
```

### File Organization for Complex Drivers

```
drivers/misc/my_driver/
├── Kconfig                    ← Build configuration
├── Makefile                   ← Build rules
├── my_driver.h                ← Shared definitions
├── my_driver_core.c           ← probe/remove, main logic
├── my_driver_hw.c             ← Hardware register access
├── my_driver_irq.c            ← Interrupt handling
└── my_driver_sysfs.c          ← sysfs attributes
```

---

## 7.2 Kernel Header Files

### Essential Headers

```c
/* Module infrastructure */
#include <linux/module.h>           /* MODULE_*, module_init/exit */
#include <linux/init.h>             /* __init, __exit, __initdata */
#include <linux/kernel.h>           /* printk, pr_*, container_of */

/* Device model */
#include <linux/device.h>           /* struct device, dev_info/err */
#include <linux/platform_device.h>  /* platform_driver, platform_device */
#include <linux/of.h>               /* Device Tree matching */
#include <linux/of_device.h>        /* of_match_device */

/* Character device */
#include <linux/fs.h>               /* file_operations */
#include <linux/cdev.h>             /* cdev_init, cdev_add */
#include <linux/uaccess.h>          /* copy_to/from_user */

/* Memory / I/O */
#include <linux/io.h>               /* ioremap, readl, writel */
#include <linux/slab.h>             /* kmalloc, kzalloc, kfree */
#include <linux/dma-mapping.h>      /* DMA APIs */

/* Interrupts */
#include <linux/interrupt.h>        /* request_irq, free_irq */

/* Synchronization */
#include <linux/mutex.h>            /* mutex_lock/unlock */
#include <linux/spinlock.h>         /* spin_lock/unlock */

/* Power management */
#include <linux/pm.h>               /* PM callbacks */
#include <linux/pm_runtime.h>       /* Runtime PM */

/* Clocks, regulators, GPIO */
#include <linux/clk.h>              /* clk_get, clk_enable */
#include <linux/regulator/consumer.h> /* regulator_enable */
#include <linux/gpio/consumer.h>    /* gpiod_get, gpiod_set_value */
```

---

## 7.3 Driver Initialization Functions

### Platform Driver Probe (Most Common Pattern)

```c
static int my_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct my_priv *priv;
    struct resource *res;
    int ret;

    /* 1. Allocate private data (auto-freed on remove) */
    priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    /* 2. Map hardware registers */
    priv->base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(priv->base))
        return PTR_ERR(priv->base);

    /* 3. Get and enable clock */
    priv->clk = devm_clk_get(dev, NULL);
    if (IS_ERR(priv->clk))
        return PTR_ERR(priv->clk);
    ret = clk_prepare_enable(priv->clk);
    if (ret)
        return ret;

    /* 4. Get IRQ and register handler */
    priv->irq = platform_get_irq(pdev, 0);
    if (priv->irq < 0) {
        ret = priv->irq;
        goto err_clk;
    }
    ret = devm_request_irq(dev, priv->irq, my_irq_handler,
                           0, dev_name(dev), priv);
    if (ret)
        goto err_clk;

    /* 5. Initialize hardware */
    writel(0x01, priv->base + REG_CTRL);   /* Enable device */

    /* 6. Store private data */
    platform_set_drvdata(pdev, priv);

    dev_info(dev, "probed @ %pR, IRQ %d\n",
             platform_get_resource(pdev, IORESOURCE_MEM, 0),
             priv->irq);
    return 0;

err_clk:
    clk_disable_unprepare(priv->clk);
    return ret;
}
```

### Probe Error Handling Pattern

```
probe() {
    allocate A → fail? return -ENOMEM
    allocate B → fail? goto err_A
    allocate C → fail? goto err_B
    ...
    return 0;    /* success */

err_B:
    free B
err_A:
    free A
    return ret;
}

/* With devm_*: much simpler (resources auto-freed) */
probe() {
    devm_alloc A → fail? return error    /* freed automatically */
    devm_alloc B → fail? return error    /* freed automatically */
    devm_alloc C → fail? return error    /* freed automatically */
    return 0;
}
```

---

## 7.4 Driver Exit Functions

```c
static void my_remove(struct platform_device *pdev)
{
    struct my_priv *priv = platform_get_drvdata(pdev);

    /* Disable hardware */
    writel(0x00, priv->base + REG_CTRL);

    /* Disable clock (if not using devm_clk) */
    clk_disable_unprepare(priv->clk);

    /* devm-managed resources freed automatically after remove() */
    dev_info(&pdev->dev, "removed\n");
}
```

### Cleanup Order Rule

```
Initialization (probe):          Cleanup (remove):
   A → B → C → D                D → C → B → A
                                 (reverse order!)
```

---

## 7.5 Registering a Device Driver

### Method 1: module_platform_driver (preferred)

```c
static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "my-device",
        .of_match_table = my_of_match,
        .pm = &my_pm_ops,
    },
};
module_platform_driver(my_driver);

/* Expands to: */
static int __init my_driver_init(void) {
    return platform_driver_register(&my_driver);
}
module_init(my_driver_init);

static void __exit my_driver_exit(void) {
    platform_driver_unregister(&my_driver);
}
module_exit(my_driver_exit);
```

### Method 2: Manual (when you need extra init logic)

```c
static int __init my_init(void)
{
    int ret;

    /* Do per-module global init */
    my_wq = alloc_workqueue("my_wq", WQ_UNBOUND, 0);
    if (!my_wq)
        return -ENOMEM;

    ret = platform_driver_register(&my_driver);
    if (ret) {
        destroy_workqueue(my_wq);
        return ret;
    }
    return 0;
}

static void __exit my_exit(void)
{
    platform_driver_unregister(&my_driver);
    destroy_workqueue(my_wq);
}

module_init(my_init);
module_exit(my_exit);
```

---

## 7.6 Kernel Logging and Debugging

### printk Log Levels

```c
pr_emerg("System is unusable\n");       /* KERN_EMERG   0 */
pr_alert("Action must be taken\n");      /* KERN_ALERT   1 */
pr_crit("Critical condition\n");         /* KERN_CRIT    2 */
pr_err("Error condition\n");             /* KERN_ERR     3 */
pr_warn("Warning condition\n");          /* KERN_WARNING 4 */
pr_notice("Normal significant\n");       /* KERN_NOTICE  5 */
pr_info("Informational\n");             /* KERN_INFO    6 */
pr_debug("Debug-level message\n");       /* KERN_DEBUG   7 */
```

### Device-Aware Logging (Preferred in Drivers)

```c
/* Always use dev_* in drivers — shows device path */
dev_err(dev,  "Failed to init: %d\n", ret);
dev_warn(dev, "Timeout waiting for device\n");
dev_info(dev, "Device ready, firmware v%d\n", ver);
dev_dbg(dev,  "Register 0x%x = 0x%08x\n", reg, val);

/* Output example: */
/* my-device 2000000.uart: Failed to init: -19 */
/*                ^
/*    Shows device tree path / bus address */
```

### Dynamic Debug

```bash
# Enable all debug messages from a file
echo "file my_driver.c +p" > /sys/kernel/debug/dynamic_debug/control

# Enable debug for specific function
echo "func my_probe +p" > /sys/kernel/debug/dynamic_debug/control

# Enable debug for a module
echo "module my_driver +p" > /sys/kernel/debug/dynamic_debug/control

# Include function name and line number
echo "file my_driver.c +pfl" > /sys/kernel/debug/dynamic_debug/control

# Disable
echo "file my_driver.c -p" > /sys/kernel/debug/dynamic_debug/control
```

### devcoredump

```c
/* For complex devices (GPU, Wi-Fi), dump state on error */
#include <linux/devcoredump.h>

dev_coredumpm(dev, THIS_MODULE, data, datalen, GFP_KERNEL,
              my_read_dump, my_free_dump);
/* Creates /sys/class/devcoredump/devcd*/data */
```

---

## Complete Working Example: Minimal Platform Driver

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * minimal_driver.c - Absolute minimum platform driver
 */
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>
#include <linux/io.h>

#define REG_ID      0x00
#define REG_CTRL    0x04
#define REG_STATUS  0x08

struct minimal_priv {
    void __iomem *base;
};

static int minimal_probe(struct platform_device *pdev)
{
    struct minimal_priv *priv;
    u32 id;

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    priv->base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(priv->base))
        return PTR_ERR(priv->base);

    id = readl(priv->base + REG_ID);
    dev_info(&pdev->dev, "Device ID: 0x%08x\n", id);

    writel(0x01, priv->base + REG_CTRL);  /* Enable */

    platform_set_drvdata(pdev, priv);
    return 0;
}

static void minimal_remove(struct platform_device *pdev)
{
    struct minimal_priv *priv = platform_get_drvdata(pdev);
    writel(0x00, priv->base + REG_CTRL);  /* Disable */
    dev_info(&pdev->dev, "removed\n");
}

static const struct of_device_id minimal_of_match[] = {
    { .compatible = "example,minimal-dev" },
    { }
};
MODULE_DEVICE_TABLE(of, minimal_of_match);

static struct platform_driver minimal_driver = {
    .probe  = minimal_probe,
    .remove = minimal_remove,
    .driver = {
        .name = "minimal-dev",
        .of_match_table = minimal_of_match,
    },
};
module_platform_driver(minimal_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Example Author");
MODULE_DESCRIPTION("Minimal platform driver example");
```

### Corresponding Device Tree Node

```dts
/* In the board's .dts file */
minimal-dev@10000000 {
    compatible = "example,minimal-dev";
    reg = <0x10000000 0x1000>;     /* Base address, size */
};
```

### Build and Test

```bash
# Build
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# Load
sudo insmod minimal_driver.ko

# Check
dmesg | tail -5
ls /sys/bus/platform/drivers/minimal-dev/

# Unload
sudo rmmod minimal_driver
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| `include/linux/device.h` | `dev_info/err/dbg` macros |
| `include/linux/platform_device.h` | `platform_driver`, helpers |
| `include/linux/io.h` | `readl/writel`, `ioremap` |
| `lib/dynamic_debug.c` | Dynamic debug infrastructure |
| `drivers/base/platform.c` | Platform device/driver core |

---

## OS Comparison

| Aspect | Linux Driver | Windows WDF | macOS DriverKit |
|--------|-------------|-------------|-----------------|
| Entry point | `module_init()` | `DriverEntry()` | `init()` in IOService |
| Probe | `probe(dev)` | `EvtDeviceAdd()` | `Start()` |
| Remove | `remove(dev)` | `EvtDeviceRelease()` | `Stop()` |
| Logging | `dev_info/err/dbg` | `DbgPrintEx`, WPP | `os_log` |
| Error handling | goto chains / devm_* | WDF auto-cleanup | Swift error handling |

---

## Interview Questions

**Q1: What is the purpose of `devm_kzalloc()` vs `kzalloc()`?**
A: `devm_kzalloc()` ties the allocation to the device lifetime — automatically freed when the device is removed or probe fails. `kzalloc()` must be manually freed with `kfree()`. Always prefer `devm_*` in drivers to prevent resource leaks.

**Q2: What does `platform_set_drvdata()` do?**
A: Stores a pointer (typically to the driver's private data structure) inside the platform_device. Retrieved later with `platform_get_drvdata()`. Internally calls `dev_set_drvdata(&pdev->dev, data)`.

**Q3: Why use `dev_info()` instead of `pr_info()`?**
A: `dev_info()` includes the device name/path in the output (e.g., "my-device 2000000.uart: ..."), making it easy to identify which device the message is from. Essential when multiple instances of the same driver exist.

**Q4: What happens if `probe()` returns an error?**
A: The device-driver binding fails. `really_probe()` clears `dev->driver`, devm resources are freed. The device remains unbound. If the error is -EPROBE_DEFER, the kernel retries later.

**Q5: How do you debug a driver that doesn't probe?**
A: Check: (1) Is the module loaded? (`lsmod`). (2) Is the DT compatible string identical? (`cat /sys/firmware/devicetree/base/.../compatible`). (3) Is the bus match succeeding? (`ls /sys/bus/platform/drivers/my-dev/`). (4) Check dmesg for probe errors. (5) Enable dynamic debug for the driver. (6) Check deferred probe list: `cat /sys/kernel/debug/devices_deferred`.

---

## Summary

| Step | Function | Purpose |
|------|----------|---------|
| 1 | `module_platform_driver()` | Register driver with platform bus |
| 2 | `probe()` | Initialize hardware, get resources |
| 3 | `devm_kzalloc()` | Allocate private data |
| 4 | `devm_platform_ioremap_resource()` | Map MMIO registers |
| 5 | `platform_get_irq()` | Get interrupt number from DT |
| 6 | `devm_request_irq()` | Register interrupt handler |
| 7 | `readl()`/`writel()` | Access hardware registers |
| 8 | `remove()` | Disable hardware on unload |

---

*Next: [Chapter 8 — Character Device Drivers](Chapter_08_Character_Device_Drivers.md)*
