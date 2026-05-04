# Chapter 21: Driver Communication with User Space

## Chapter Overview

Drivers must expose interfaces for user-space applications to interact with hardware. This chapter covers all mechanisms: device files, ioctl, sysfs, procfs, and netlink.

---

## 21.1 Device Files (/dev)

The primary interface for character and block devices.

```bash
# Common device files
/dev/ttyS0        # Serial port (char)
/dev/sda          # Disk (block)
/dev/video0       # Camera (char)
/dev/i2c-0        # I2C bus (char)
/dev/spidev0.0    # SPI device (char)
/dev/null         # Null device (char)
```

Device files are created by udev based on `device_create()`:

```c
/* In driver */
my_class = class_create("myclass");
my_device = device_create(my_class, parent, devno, NULL, "mydev%d", minor);
/* Result: udev creates /dev/mydev0 with proper permissions */
```

---

## 21.2 ioctl Interface

For device-specific control commands that don't fit read/write. See Chapter 8 for full implementation details.

```c
/* Define commands */
#define MY_RESET     _IO('M', 0)
#define MY_GET_STAT  _IOR('M', 1, struct my_status)
#define MY_SET_CFG   _IOW('M', 2, struct my_config)

/* Driver handler */
static long my_ioctl(struct file *f, unsigned int cmd, unsigned long arg)
{
    switch (cmd) {
    case MY_RESET:
        hw_reset(priv);
        return 0;
    case MY_GET_STAT: {
        struct my_status st;
        get_status(priv, &st);
        return copy_to_user((void __user *)arg, &st, sizeof(st)) ? -EFAULT : 0;
    }
    default:
        return -ENOTTY;
    }
}
```

---

## 21.3 sysfs Interface

Best for simple configuration and status read-out.

```
/sys/devices/platform/my-device/
├── status          ← read-only sysfs attribute
├── enable          ← read-write sysfs attribute
├── firmware_ver    ← read-only
└── power/
    └── runtime_status
```

```c
/* Read-only attribute */
static ssize_t status_show(struct device *dev,
                           struct device_attribute *attr, char *buf)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    return sysfs_emit(buf, "%u\n", readl(priv->base + REG_STATUS));
}
static DEVICE_ATTR_RO(status);

/* Read-write attribute */
static ssize_t enable_show(struct device *dev,
                           struct device_attribute *attr, char *buf)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    return sysfs_emit(buf, "%d\n", priv->enabled);
}

static ssize_t enable_store(struct device *dev,
                            struct device_attribute *attr,
                            const char *buf, size_t count)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    bool val;
    if (kstrtobool(buf, &val))
        return -EINVAL;
    priv->enabled = val;
    writel(val, priv->base + REG_CTRL);
    return count;
}
static DEVICE_ATTR_RW(enable);

/* Attribute group */
static struct attribute *my_attrs[] = {
    &dev_attr_status.attr,
    &dev_attr_enable.attr,
    NULL,
};
ATTRIBUTE_GROUPS(my);

/* Auto-created in driver definition */
static struct platform_driver my_driver = {
    .driver = {
        .name = "my-device",
        .dev_groups = my_groups,
    },
};
```

```bash
# Userspace access
cat /sys/devices/platform/my-device/status
echo 1 > /sys/devices/platform/my-device/enable
```

### sysfs Rules
- **One value per file** (not multiple values)
- **Human-readable text** (not binary)
- Use `sysfs_emit()` (bounds-safe, not `sprintf()`)
- sysfs is for simple config — complex data → ioctl or debugfs

---

## 21.4 procfs Interface

Legacy interface, primarily for system-wide information. New drivers should use sysfs.

```c
#include <linux/proc_fs.h>
#include <linux/seq_file.h>

static int my_proc_show(struct seq_file *m, void *v)
{
    seq_printf(m, "Status: %d\n", my_status);
    seq_printf(m, "Count: %lu\n", my_count);
    return 0;
}

static int my_proc_open(struct inode *inode, struct file *file)
{
    return single_open(file, my_proc_show, NULL);
}

static const struct proc_ops my_proc_ops = {
    .proc_open    = my_proc_open,
    .proc_read    = seq_read,
    .proc_lseek   = seq_lseek,
    .proc_release = single_release,
};

/* In init: */
proc_create("my_driver_info", 0444, NULL, &my_proc_ops);

/* In exit: */
remove_proc_entry("my_driver_info", NULL);
```

---

## 21.5 Netlink Communication

For high-bandwidth, structured kernel-to-userspace communication.

```c
/* Kernel side */
#include <net/genetlink.h>

static struct genl_family my_genl_family = {
    .name    = "MY_DRIVER",
    .version = 1,
    .maxattr = MY_ATTR_MAX,
    .ops     = my_genl_ops,
    .n_ops   = ARRAY_SIZE(my_genl_ops),
};

/* Register during init: */
genl_register_family(&my_genl_family);
```

### Communication Method Comparison

| Method | Direction | Data Type | Latency | Use Case |
|--------|-----------|-----------|---------|----------|
| **read/write** | Both | Byte stream | Low | Data transfer |
| **ioctl** | Both | Structured | Low | Device control |
| **sysfs** | Both | Text | Medium | Config/status |
| **procfs** | Mostly read | Text | Medium | System info (legacy) |
| **debugfs** | Both | Any | N/A | Debug only |
| **netlink** | Both | Structured | Low | Events, multi-subscriber |
| **mmap** | Shared | Raw memory | Zero-copy | Video, framebuffers |

---

## Interview Questions

**Q1: When should you use sysfs vs ioctl?**
A: sysfs: simple, single-value configuration (enable/disable, read status). ioctl: complex, structured commands (configure DMA, read multi-field status, firmware upload). Rule: if it's one integer, sysfs. If it's a struct, ioctl.

**Q2: Why is `sysfs_emit()` preferred over `sprintf()`?**
A: `sysfs_emit()` is page-size-bounded and prevents buffer overflow. sysfs attributes use a single PAGE_SIZE buffer — `sprintf()` could overflow it.

**Q3: What is debugfs and when should you use it?**
A: `/sys/kernel/debug/` — a filesystem for developer/debug information. Not part of the stable ABI (can change without notice). Use for: register dumps, internal state, performance counters. Never expose user-facing configuration here.

---

*Next: [Chapter 22 — Kernel I/O Subsystems](Chapter_22_IO_Subsystems.md)*
