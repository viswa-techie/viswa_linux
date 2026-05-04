# Chapter 11: GPIO Subsystem

## Learning Goals
- Understand GPIO controller and consumer models in Linux
- Know gpiolib, gpiod API, and GPIO interrupt handling
- Write GPIO consumer drivers and GPIO controller drivers
- Debug GPIO with sysfs and gpiomon

---

## 11.1 GPIO Subsystem Architecture

```
GPIO Subsystem Architecture:

User Space:
  ┌───────────┐  ┌───────────┐  ┌───────────┐
  │ gpioset   │  │ gpioget   │  │ gpiomon   │
  │ (libgpiod)│  │ (libgpiod)│  │ (libgpiod)│
  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
        │               │               │
        └───────────────┼───────────────┘
                        │  /dev/gpiochipN
Kernel:                 │
  ┌─────────────────────▼────────────────────────────┐
  │              gpiolib (drivers/gpio/gpiolib.c)     │
  │                                                   │
  │  ┌──────────────────────────────────────────┐    │
  │  │  GPIO Consumer API (gpiod_*)              │    │
  │  │  ├── devm_gpiod_get()                     │    │
  │  │  ├── gpiod_direction_output()             │    │
  │  │  ├── gpiod_set_value() / gpiod_get_value()│    │
  │  │  ├── gpiod_to_irq()                      │    │
  │  │  └── Active-low handled transparently     │    │
  │  └──────────────────────────────────────────┘    │
  │                                                   │
  │  ┌──────────────────────────────────────────┐    │
  │  │  GPIO Controller (gpio_chip)              │    │
  │  │  ├── direction_input / direction_output   │    │
  │  │  ├── get / set  (read/write pin)          │    │
  │  │  ├── to_irq (map GPIO to IRQ)             │    │
  │  │  ├── Supports multiple banks/controllers  │    │
  │  │  └── Registered by SoC/PMIC GPIO drivers  │    │
  │  └──────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────┘
                        │
Hardware:               ▼
  ┌──────────────────────────────────────────────┐
  │  GPIO Controller Hardware                     │
  │  ├── Direction register (input/output)        │
  │  ├── Data register (read/write value)         │
  │  ├── Interrupt config (edge/level, enable)    │
  │  └── Pull-up/pull-down configuration          │
  └──────────────────────────────────────────────┘
```

---

## 11.2 GPIO Consumer API (gpiod)

```c
/* Modern GPIO consumer API — gpiod (descriptor-based) */
#include <linux/gpio/consumer.h>

static int my_driver_probe(struct platform_device *pdev)
{
    struct gpio_desc *reset_gpio;
    struct gpio_desc *enable_gpio;
    struct gpio_desc *status_gpio;

    /* Get GPIO from device tree (by name) */
    reset_gpio = devm_gpiod_get(&pdev->dev, "reset", GPIOD_OUT_HIGH);
    if (IS_ERR(reset_gpio))
        return PTR_ERR(reset_gpio);

    /* "enable" GPIO, initially de-asserted (handles active-low) */
    enable_gpio = devm_gpiod_get(&pdev->dev, "enable", GPIOD_OUT_LOW);

    /* Input GPIO */
    status_gpio = devm_gpiod_get(&pdev->dev, "status", GPIOD_IN);

    /* Optional GPIO (returns NULL if not specified in DT) */
    struct gpio_desc *optional_gpio;
    optional_gpio = devm_gpiod_get_optional(&pdev->dev, "power",
                                            GPIOD_OUT_LOW);

    /* Set output value (active-low handled transparently) */
    gpiod_set_value(reset_gpio, 0);    /* de-assert reset */
    msleep(10);
    gpiod_set_value(reset_gpio, 1);    /* assert reset */

    /* Read input */
    int val = gpiod_get_value(status_gpio);
    dev_info(&pdev->dev, "Status GPIO = %d\n", val);

    /* Convert GPIO to IRQ number */
    int irq = gpiod_to_irq(status_gpio);
    devm_request_irq(&pdev->dev, irq, my_isr,
                     IRQF_TRIGGER_FALLING, "my-status", priv);

    return 0;
}
```

---

## 11.3 Device Tree GPIO Bindings

```dts
/* GPIO consumer in device tree */
my_device {
    compatible = "vendor,my-device";

    /* Named GPIOs: <name>-gpios property */
    reset-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;
    enable-gpios = <&gpio2 10 GPIO_ACTIVE_HIGH>;
    status-gpios = <&gpio1 8 GPIO_ACTIVE_HIGH>;

    /* GPIO flag definitions:
     * GPIO_ACTIVE_HIGH = 0
     * GPIO_ACTIVE_LOW  = 1 (value logic inverted)
     * Bit 0: polarity (0=active-high, 1=active-low)
     * The gpiod API handles polarity transparently:
     *   gpiod_set_value(gpio, 1) = assert (active state)
     *   gpiod_set_value(gpio, 0) = de-assert
     */
};

/* GPIO controller (provider) in device tree */
gpio1: gpio@40020000 {
    compatible = "vendor,soc-gpio";
    reg = <0x40020000 0x400>;
    gpio-controller;
    #gpio-cells = <2>;        /* <pin_number flags> */
    interrupt-controller;
    #interrupt-cells = <2>;
    ngpios = <32>;
};
```

---

## 11.4 GPIO Controller (Provider) Driver

```c
/* GPIO controller driver — manages a bank of GPIOs */
#include <linux/gpio/driver.h>

struct my_gpio {
    struct gpio_chip gc;
    void __iomem *regs;
    spinlock_t lock;
};

static int my_gpio_get(struct gpio_chip *gc, unsigned int offset)
{
    struct my_gpio *priv = gpiochip_get_data(gc);
    return !!(readl(priv->regs + GPIO_DATA) & BIT(offset));
}

static void my_gpio_set(struct gpio_chip *gc, unsigned int offset,
                         int value)
{
    struct my_gpio *priv = gpiochip_get_data(gc);
    unsigned long flags;
    u32 reg;

    spin_lock_irqsave(&priv->lock, flags);
    reg = readl(priv->regs + GPIO_DATA);
    if (value)
        reg |= BIT(offset);
    else
        reg &= ~BIT(offset);
    writel(reg, priv->regs + GPIO_DATA);
    spin_unlock_irqrestore(&priv->lock, flags);
}

static int my_gpio_direction_input(struct gpio_chip *gc,
                                    unsigned int offset)
{
    struct my_gpio *priv = gpiochip_get_data(gc);
    u32 reg = readl(priv->regs + GPIO_DIR);
    reg &= ~BIT(offset);  /* 0 = input */
    writel(reg, priv->regs + GPIO_DIR);
    return 0;
}

static int my_gpio_direction_output(struct gpio_chip *gc,
                                     unsigned int offset, int value)
{
    my_gpio_set(gc, offset, value);
    struct my_gpio *priv = gpiochip_get_data(gc);
    u32 reg = readl(priv->regs + GPIO_DIR);
    reg |= BIT(offset);  /* 1 = output */
    writel(reg, priv->regs + GPIO_DIR);
    return 0;
}

static int my_gpio_probe(struct platform_device *pdev)
{
    struct my_gpio *priv;

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    spin_lock_init(&priv->lock);

    priv->regs = devm_platform_ioremap_resource(pdev, 0);

    priv->gc.label            = "my-gpio";
    priv->gc.parent           = &pdev->dev;
    priv->gc.owner            = THIS_MODULE;
    priv->gc.base             = -1;  /* dynamic numbering */
    priv->gc.ngpio            = 32;
    priv->gc.get              = my_gpio_get;
    priv->gc.set              = my_gpio_set;
    priv->gc.direction_input  = my_gpio_direction_input;
    priv->gc.direction_output = my_gpio_direction_output;
    priv->gc.can_sleep        = false;  /* MMIO-based, fast */

    return devm_gpiochip_add_data(&pdev->dev, &priv->gc, priv);
}
```

---

## 11.5 GPIO Interrupts

```c
/* GPIO IRQ domain — mapping GPIO pins to IRQ numbers */

static int my_gpio_probe(struct platform_device *pdev)
{
    struct gpio_irq_chip *girq = &priv->gc.irq;

    /* Setup IRQ chip */
    static struct irq_chip my_irq_chip = {
        .name         = "my-gpio",
        .irq_ack      = my_gpio_irq_ack,
        .irq_mask     = my_gpio_irq_mask,
        .irq_unmask   = my_gpio_irq_unmask,
        .irq_set_type = my_gpio_irq_set_type,
    };

    girq->chip       = &my_irq_chip;
    girq->handler    = handle_edge_irq;
    girq->parent_handler = my_gpio_irq_handler;
    girq->num_parents    = 1;
    girq->parents        = devm_kcalloc(dev, 1, sizeof(*girq->parents),
                                        GFP_KERNEL);
    girq->parents[0]     = platform_get_irq(pdev, 0);

    return devm_gpiochip_add_data(&pdev->dev, &priv->gc, priv);
}

/* Parent IRQ handler — demux to individual GPIO IRQs */
static void my_gpio_irq_handler(struct irq_desc *desc)
{
    struct gpio_chip *gc = irq_desc_get_handler_data(desc);
    struct my_gpio *priv = gpiochip_get_data(gc);
    struct irq_chip *chip = irq_desc_get_chip(desc);
    u32 pending;

    chained_irq_enter(chip, desc);

    pending = readl(priv->regs + GPIO_INT_STATUS);
    while (pending) {
        int bit = __ffs(pending);
        generic_handle_domain_irq(gc->irq.domain, bit);
        pending &= ~BIT(bit);
    }

    chained_irq_exit(chip, desc);
}
```

---

## 11.6 GPIO Debug

```bash
# Modern: libgpiod tools
$ gpiodetect                     # list GPIO controllers
$ gpioinfo gpiochip0             # list all GPIOs
$ gpioget gpiochip0 5            # read pin 5
$ gpioset gpiochip0 5=1          # set pin 5 high
$ gpiomon gpiochip0 8            # monitor interrupts on pin 8

# Legacy sysfs (deprecated but still commonly used)
$ echo 5 > /sys/class/gpio/export
$ echo out > /sys/class/gpio/gpio5/direction
$ echo 1 > /sys/class/gpio/gpio5/value
$ cat /sys/class/gpio/gpio5/value

# Kernel debugfs
$ cat /sys/kernel/debug/gpio
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| gpiolib | drivers/gpio/gpiolib.c | Core GPIO framework |
| gpiod API | drivers/gpio/gpiolib-devres.c | Managed GPIO consumers |
| gpio_chip | include/linux/gpio/driver.h | Provider interface |
| GPIO DT | drivers/gpio/gpiolib-of.c | Device tree support |
| GPIO char dev | drivers/gpio/gpiolib-cdev.c | /dev/gpiochipN |
| GPIO sysfs | drivers/gpio/gpiolib-sysfs.c | Legacy sysfs interface |

---

## Interview Questions

**Q1: Compare the legacy GPIO API with the gpiod API.**
A: Legacy API uses integer GPIO numbers (`gpio_request(42, "reset")`, `gpio_set_value(42, 1)`) — numbers are platform-specific and fragile. The gpiod API uses opaque `gpio_desc` pointers obtained from device tree (`devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH)`) — no magic numbers. Key gpiod advantages: (1) Active-low handling is transparent — `gpiod_set_value(gpio, 1)` means "assert" regardless of polarity. (2) `devm_` managed — auto-released on driver unbind. (3) Tied to device tree naming, not global numbering. The legacy API is deprecated; all new drivers should use gpiod.

**Q2: How does GPIO interrupt handling work in Linux?**
A: (1) Consumer calls `gpiod_to_irq(gpio)` to get the Linux IRQ number for a GPIO pin. (2) Consumer requests the IRQ with `devm_request_irq()`. (3) The GPIO controller implements an `irq_chip` with mask/unmask/set_type operations. (4) When the hardware GPIO interrupt fires, it triggers the parent IRQ. (5) The GPIO controller's chained IRQ handler reads pending status bits and calls `generic_handle_domain_irq()` for each active pin. (6) The IRQ domain maps GPIO pin numbers to Linux IRQ numbers.

**Q3: What does `can_sleep` mean in a GPIO controller?**
A: `can_sleep = false` means GPIO operations (get/set) can be called from atomic context (interrupts, spinlocks) — typical for memory-mapped controllers (MMIO). `can_sleep = true` means operations may sleep — required for GPIO controllers connected via I2C or SPI (e.g., I2C GPIO expanders like PCA9535), because I2C/SPI transfers can sleep. When `can_sleep = true`, consumers must use `gpiod_get_value_cansleep()` / `gpiod_set_value_cansleep()` instead of the regular functions.

---

## Summary

- gpiolib provides GPIO controller (provider) and consumer frameworks
- gpiod API is the modern consumer API — uses descriptors, handles active-low
- GPIO controllers expose get/set/direction via `gpio_chip` callbacks
- GPIO interrupts use IRQ domains to map pins to Linux IRQ numbers
- Device tree declares GPIOs with `<name>-gpios` properties-
- libgpiod tools (gpioget, gpioset, gpiomon) are the modern user-space interface

---

*Next: [Chapter 12 — DMA Engine Framework](Chapter_12_DMA_Engine.md)*
