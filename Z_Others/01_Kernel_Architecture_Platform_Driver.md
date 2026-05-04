# Linux Kernel — Architecture Overview & Mental Model

## The Big Picture

```
┌──────────────────────────────────────────────────────────────┐
│                      USER SPACE                              │
│  App   App   App   Android Framework   HAL (HIDL/AIDL)      │
├───────────────────────────glibc / Bionic libc────────────────┤
│                   System Call Interface                       │
│          (read, write, ioctl, mmap, open, close…)            │
├──────────────────────────────────────────────────────────────┤
│                      KERNEL SPACE                            │
│  ┌────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │ VFS        │  │ Net Stack    │  │ Memory Manager (MM)  │ │
│  │ (ext4,f2fs)│  │ (TCP/IP,CAN) │  │ (slab,vmalloc,mmap)  │ │
│  └─────┬──────┘  └──────────────┘  └──────────────────────┘ │
│  ┌─────▼──────────────────────────────────────────────────┐  │
│  │           Device Driver Layer                           │  │
│  │  char  block  net  i2c  spi  usb  v4l2  drm  platform  │  │
│  └─────────────────────────┬───────────────────────────────┘  │
│                             │ interrupts, DMA, MMIO            │
├─────────────────────────────▼───────────────────────────────┤
│                      HARDWARE                                │
│       CPU (ARM A55)  Peripherals  Memory  SoC Buses         │
└──────────────────────────────────────────────────────────────┘
```

---

## Kernel Subsystems Map

| Subsystem | Location | What it does |
|---|---|---|
| `drivers/char/` | char drivers | Generic char devices |
| `drivers/i2c/` | I2C bus drivers | I2C controller + device drivers |
| `drivers/spi/` | SPI bus drivers | SPI controllers |
| `drivers/gpio/` | GPIO | GPIO chip drivers |
| `drivers/clk/` | Clock framework | CCF — Common Clock Framework |
| `drivers/pinctrl/` | Pin multiplexing | iomux / pinmux |
| `drivers/regulator/` | Power rails | PMIC regulators |
| `drivers/thermal/` | Thermal zones | TSENS, cooling devices |
| `drivers/iio/` | Industrial I/O | ADC, IMU, ambient light |
| `drivers/media/` | V4L2 / media | Camera, video decoders |
| `drivers/gpu/drm/` | DRM/KMS | Display pipeline |
| `drivers/dma/` | DMA engines | DMA controller drivers |
| `sound/soc/` | ASoC | Audio drivers |
| `drivers/net/` | Network | Ethernet, CAN drivers |

---

## Kernel Module Skeleton

```c
// my_driver.c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/platform_device.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Viswa");
MODULE_DESCRIPTION("Example platform driver");

/* Called when module loads */
static int __init my_driver_init(void) {
    pr_info("my_driver: init\n");
    return 0;   // 0 = success, negative = error
}

/* Called when module unloads */
static void __exit my_driver_exit(void) {
    pr_info("my_driver: exit\n");
}

module_init(my_driver_init);
module_exit(my_driver_exit);
```

```makefile
# Makefile
obj-m += my_driver.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
# Load / unload
insmod my_driver.ko
rmmod my_driver
dmesg | tail -20
```

---

## The Platform Device / Driver Model

```c
/* The "match" mechanism:
   Platform device  ---[compatible string]--→  Platform driver
   Defined in DT:        "myco,mydevice"       .of_match_table entry
*/

static const struct of_device_id my_of_match[] = {
    { .compatible = "myco,mydevice-v1" },
    { .compatible = "myco,mydevice-v2" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, my_of_match);

static int my_probe(struct platform_device *pdev) {
    struct device *dev = &pdev->dev;
    struct resource *res;
    void __iomem      *base;

    /* Get MMIO from DT */
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    base = devm_ioremap_resource(dev, res);  // devm = auto-freed on remove
    if (IS_ERR(base))
        return PTR_ERR(base);

    /* Get IRQ from DT */
    int irq = platform_get_irq(pdev, 0);
    if (irq < 0) return irq;

    /* Get a clock from DT (clocks = <&clk CLK_UART>) */
    struct clk *clk = devm_clk_get(dev, "uart");
    if (IS_ERR(clk)) return PTR_ERR(clk);
    clk_prepare_enable(clk);

    dev_info(dev, "probed OK, irq=%d\n", irq);
    return 0;
}

static int my_remove(struct platform_device *pdev) {
    dev_info(&pdev->dev, "removed\n");
    return 0;
}

static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name           = "my-device",
        .of_match_table = my_of_match,
    },
};
module_platform_driver(my_driver);  // macro for init/exit
```

---

## Device Tree Snippet (for the above driver)

```dts
/* In SoC .dtsi */
my_dev: mydevice@40010000 {
    compatible = "myco,mydevice-v1";
    reg = <0x40010000 0x1000>;     /* MMIO base, size */
    interrupts = <GIC_SPI 45 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&gcc GCC_UART_CLK>;
    clock-names = "uart";
    status = "disabled";           /* disabled by default */
};

/* In board .dts — enable it */
&my_dev {
    status = "okay";
    some-gpio = <&tlmm 42 GPIO_ACTIVE_HIGH>;
};
```

---

## `devm_*` Resource Management (You MUST use this)

```c
/* devm_ = "device managed" — automatically freed when device is removed/unbound */

devm_kmalloc(dev, size, GFP_KERNEL)   // vs kmalloc + kfree
devm_kzalloc(dev, size, GFP_KERNEL)   // zero-filled
devm_ioremap_resource(dev, res)        // vs ioremap + iounmap
devm_request_irq(dev, irq, handler, flags, name, data)  // vs request_irq
devm_clk_get(dev, name)               // vs clk_get + clk_put
devm_gpio_request(dev, gpio, label)   // vs gpio_request
devm_regulator_get(dev, name)         // vs regulator_get
```

---

## Kernel Logging

```c
pr_emerg("...");    // KERN_EMERG   — system unusable
pr_alert("...");    // KERN_ALERT   — immediate action needed
pr_crit("...");     // KERN_CRIT    — critical
pr_err("...");      // KERN_ERR     — errors
pr_warn("...");     // KERN_WARNING — warnings
pr_notice("...");   // KERN_NOTICE  — normal but significant
pr_info("...");     // KERN_INFO    — informational
pr_debug("...");    // KERN_DEBUG   — debug (only if DEBUG defined)

/* Device-specific (includes device name automatically) */
dev_err(dev, "failed to probe: %d\n", ret);
dev_info(dev, "clock rate: %lu Hz\n", clk_get_rate(clk));
```

---

## Key Interview Questions

- Q: What is the difference between a platform device and a PCI device?
- Q: What does `devm_` prefix mean and why should you use it?
- Q: How does the kernel match a device tree node to a driver?
- Q: What is the difference between `ioremap` and `ioremap_nocache`?
- Q: In which contexts can you sleep? (hint: interrupt context cannot sleep)
