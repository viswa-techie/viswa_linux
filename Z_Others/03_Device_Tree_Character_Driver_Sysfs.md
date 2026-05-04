# Device Tree Bindings & Character Drivers / Sysfs

## Device Tree — Mental Model

```
Device Tree = Hardware description database (replaces board files)
Format: DTS (source) → DTB (binary blob loaded by bootloader)

board.dts
  └── includes SoC dtsi (chip-level)
      └── includes board-level overrides

Node structure:
  label: node-name@unit-address {
      compatible = "vendor,model";   ← used by kernel to match driver
      reg = <base_addr length>;      ← register base + size
      interrupts = <GIC_SPI 36 IRQ_TYPE_LEVEL_HIGH>;
      clocks = <&gcc GCC_MY_CLK>;
      clock-names = "core";
      resets = <&gcc MY_RESET>;
      status = "okay";               ← "disabled" to turn off
  };
```

---

## Reading DT in a Kernel Driver

```c
static int my_probe(struct platform_device *pdev) {
    struct device *dev = &pdev->dev;
    struct device_node *np = dev->of_node;

    /* Read reg (handled automatically via platform_get_resource) */
    struct resource *res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    void __iomem *base = devm_ioremap_resource(dev, res);

    /* Read custom property */
    u32 timeout_ms;
    of_property_read_u32(np, "my,timeout-ms", &timeout_ms);

    /* Read a child node */
    struct device_node *child;
    for_each_child_of_node(np, child) {
        const char *name;
        of_property_read_string(child, "label", &name);
    }

    /* Read GPIO */
    struct gpio_desc *gpiod = devm_gpiod_get(dev, "reset", GPIOD_OUT_LOW);
    gpiod_set_value_cansleep(gpiod, 1);  // assert reset

    return 0;
}

static const struct of_device_id my_of_match[] = {
    { .compatible = "vendor,my-device" },
    {}
};
MODULE_DEVICE_TABLE(of, my_of_match);
```

---

## Character Driver — Full Skeleton

```c
#include <linux/cdev.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

#define DEVICE_NAME "mydev"
#define BUF_SIZE 256

static dev_t dev_num;
static struct cdev my_cdev;
static struct class *my_class;
static char kbuf[BUF_SIZE];

static ssize_t my_read(struct file *f, char __user *ubuf, size_t len, loff_t *off) {
    size_t avail = min(len, (size_t)(BUF_SIZE - *off));
    if (avail == 0) return 0;
    if (copy_to_user(ubuf, kbuf + *off, avail)) return -EFAULT;
    *off += avail;
    return avail;
}
static ssize_t my_write(struct file *f, const char __user *ubuf, size_t len, loff_t *off) {
    size_t to_copy = min(len, (size_t)BUF_SIZE);
    if (copy_from_user(kbuf, ubuf, to_copy)) return -EFAULT;
    return to_copy;
}
static long my_ioctl(struct file *f, unsigned int cmd, unsigned long arg) {
    switch (cmd) {
    case MY_IOCTL_RESET: memset(kbuf, 0, BUF_SIZE); break;
    default: return -ENOTTY;
    }
    return 0;
}
static const struct file_operations my_fops = {
    .owner = THIS_MODULE,
    .read = my_read, .write = my_write, .unlocked_ioctl = my_ioctl,
};

static int __init my_init(void) {
    alloc_chrdev_region(&dev_num, 0, 1, DEVICE_NAME);
    cdev_init(&my_cdev, &my_fops);
    cdev_add(&my_cdev, dev_num, 1);
    my_class = class_create(THIS_MODULE, DEVICE_NAME);
    device_create(my_class, NULL, dev_num, NULL, DEVICE_NAME);  // creates /dev/mydev
    return 0;
}
static void __exit my_exit(void) {
    device_destroy(my_class, dev_num);
    class_destroy(my_class);
    cdev_del(&my_cdev);
    unregister_chrdev_region(dev_num, 1);
}
```

---

## Sysfs / Debugfs Interfaces

```c
/* Sysfs — expose attributes in /sys/class/mydev/attr_name */
static ssize_t myattr_show(struct device *dev, struct device_attribute *attr, char *buf) {
    return sysfs_emit(buf, "%d\n", my_value);
}
static ssize_t myattr_store(struct device *dev, struct device_attribute *attr,
                             const char *buf, size_t count) {
    kstrtoint(buf, 10, &my_value);
    return count;
}
static DEVICE_ATTR_RW(myattr);  // creates device_attr_myattr
// in probe: device_create_file(dev, &dev_attr_myattr);

/* Debugfs — /sys/kernel/debug/mydriver/ */
#include <linux/debugfs.h>
struct dentry *ddir = debugfs_create_dir("mydriver", NULL);
debugfs_create_u32("reg_val", 0644, ddir, &my_reg_val);
debugfs_create_file("dump", 0444, ddir, priv, &dump_fops);
// cleanup: debugfs_remove_recursive(ddir);
```

---

## Interview Questions

- Q: What does `compatible` in a DT node do?
- Q: What is `devm_ioremap_resource` and why is `devm_` preferred?
- Q: What is the difference between sysfs and debugfs?
- Q: Why use `copy_to_user`/`copy_from_user` instead of direct memcpy in a driver?
- Q: What is `alloc_chrdev_region` and what does the minor number represent?
