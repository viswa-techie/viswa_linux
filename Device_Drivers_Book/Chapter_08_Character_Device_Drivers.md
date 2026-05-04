# Chapter 8: Character Device Drivers

## Chapter Overview

Character device drivers are the most common and foundational driver type. This chapter provides a complete, production-quality implementation covering registration, file operations, and the ioctl interface.

---

## 8.1 Character Device Driver Overview

```
User Application
    │
    ├── open("/dev/mydev", O_RDWR)  → fops->open()
    ├── read(fd, buf, len)          → fops->read()
    ├── write(fd, buf, len)         → fops->write()
    ├── ioctl(fd, cmd, arg)         → fops->unlocked_ioctl()
    ├── poll(fd, ...)               → fops->poll()
    ├── mmap(fd, ...)               → fops->mmap()
    └── close(fd)                   → fops->release()
           │
           ▼
    VFS routes via major/minor → cdev → file_operations → YOUR CODE
```

### Key Components

```
┌──────────────────────────────────┐
│    struct cdev                    │
│  ┌─────────────────────────────┐ │
│  │ dev_t dev    (major:minor)  │ │
│  │ struct file_operations *ops │ │
│  │ struct kobject kobj         │ │
│  │ struct module *owner        │ │
│  └─────────────────────────────┘ │
│                                  │
│  + struct class (for udev)       │
│  + struct device (for sysfs)     │
└──────────────────────────────────┘
```

---

## 8.2 Device Numbers (Major/Minor)

```
$ ls -la /dev/ttyS0
crw-rw---- 1 root dialout 4, 64 ... /dev/ttyS0
                            ^  ^
                     major──┘  └──minor

Major number: identifies the driver (4 = serial)
Minor number: identifies the device instance (64 = ttyS0)
```

### Static vs Dynamic Allocation

```c
/* Static: pick your own major (NOT recommended — conflicts) */
register_chrdev_region(MKDEV(240, 0), 4, "my_dev");

/* Dynamic: kernel assigns major (RECOMMENDED) */
dev_t devno;
alloc_chrdev_region(&devno, 0, 4, "my_dev");
/* devno now holds assigned major:0, supports minors 0-3 */

int major = MAJOR(devno);
int minor = MINOR(devno);
```

---

## 8.3 Device File Creation

### Method 1: Automatic via class + device (preferred)

```c
static struct class *my_class;
static struct device *my_device;
static dev_t my_devno;

/* In init/probe: */
my_class = class_create("my_class");
if (IS_ERR(my_class))
    return PTR_ERR(my_class);

my_device = device_create(my_class, NULL, my_devno, NULL, "mydev");
if (IS_ERR(my_device)) {
    class_destroy(my_class);
    return PTR_ERR(my_device);
}
/* Result: /dev/mydev created automatically by udev */
```

### Method 2: Manual with mknod

```bash
# After loading module, check major number:
cat /proc/devices | grep my_dev
# 240 my_dev

sudo mknod /dev/mydev c 240 0
sudo chmod 666 /dev/mydev
```

---

## 8.4 struct file_operations

The central structure connecting VFS to your driver:

```c
/* include/linux/fs.h (simplified) */
struct file_operations {
    struct module *owner;
    loff_t (*llseek)(struct file *, loff_t, int);
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*read_iter)(struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter)(struct kiocb *, struct iov_iter *);
    __poll_t (*poll)(struct file *, struct poll_table_struct *);
    long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
    long (*compat_ioctl)(struct file *, unsigned int, unsigned long);
    int (*mmap)(struct file *, struct vm_area_struct *);
    int (*open)(struct inode *, struct file *);
    int (*flush)(struct file *, fl_owner_t id);
    int (*release)(struct inode *, struct file *);
    int (*fsync)(struct file *, loff_t, loff_t, int datasync);
    int (*fasync)(int, struct file *, int);
};
```

---

## 8.5 open() Operation

```c
static int my_open(struct inode *inode, struct file *filp)
{
    struct my_dev *dev;

    /* Get private data from inode->i_cdev */
    dev = container_of(inode->i_cdev, struct my_dev, cdev);
    filp->private_data = dev;   /* Store for later use in read/write */

    /* Check access mode */
    if ((filp->f_flags & O_ACCMODE) == O_WRONLY) {
        /* write-only mode handling */
    }

    /* Limit to single opener (optional) */
    if (atomic_read(&dev->open_count) > 0)
        return -EBUSY;
    atomic_inc(&dev->open_count);

    dev_dbg(dev->dev, "opened by PID %d\n", current->pid);
    return 0;   /* nonseekable_open(inode, filp) for non-seekable */
}
```

### container_of Pattern

```c
/*
 * container_of(ptr, type, member) — get the struct from a member pointer
 *
 * struct my_dev {
 *     int x;
 *     struct cdev cdev;   ← inode->i_cdev points here
 *     int y;
 * };
 *
 * container_of(inode->i_cdev, struct my_dev, cdev)
 *   → returns pointer to the enclosing my_dev struct
 */
```

---

## 8.6 read() Operation

```c
static ssize_t my_read(struct file *filp, char __user *buf,
                        size_t count, loff_t *ppos)
{
    struct my_dev *dev = filp->private_data;
    ssize_t retval = 0;

    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;

    /* Check bounds */
    if (*ppos >= dev->size)
        goto out;   /* EOF */
    if (*ppos + count > dev->size)
        count = dev->size - *ppos;

    /* Copy data to user space */
    if (copy_to_user(buf, dev->data + *ppos, count)) {
        retval = -EFAULT;
        goto out;
    }

    *ppos += count;
    retval = count;

out:
    mutex_unlock(&dev->lock);
    return retval;
}
```

### Blocking Read (Wait for Data)

```c
static ssize_t my_read(struct file *filp, char __user *buf,
                        size_t count, loff_t *ppos)
{
    struct my_dev *dev = filp->private_data;

    /* Wait until data available */
    if (wait_event_interruptible(dev->read_wq, dev->data_ready))
        return -ERESTARTSYS;

    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;

    /* copy_to_user... */
    dev->data_ready = false;

    mutex_unlock(&dev->lock);
    return count;
}

/* Non-blocking mode: */
if (filp->f_flags & O_NONBLOCK) {
    if (!dev->data_ready)
        return -EAGAIN;   /* Would block */
}
```

---

## 8.7 write() Operation

```c
static ssize_t my_write(struct file *filp, const char __user *buf,
                         size_t count, loff_t *ppos)
{
    struct my_dev *dev = filp->private_data;
    ssize_t retval = 0;

    if (mutex_lock_interruptible(&dev->lock))
        return -ERESTARTSYS;

    /* Bounds check */
    if (*ppos >= dev->buf_size) {
        retval = -ENOSPC;
        goto out;
    }
    if (*ppos + count > dev->buf_size)
        count = dev->buf_size - *ppos;

    /* Copy data from user space */
    if (copy_from_user(dev->data + *ppos, buf, count)) {
        retval = -EFAULT;
        goto out;
    }

    *ppos += count;
    dev->size = max(dev->size, (size_t)*ppos);
    retval = count;

    /* Wake up readers waiting for data */
    dev->data_ready = true;
    wake_up_interruptible(&dev->read_wq);

out:
    mutex_unlock(&dev->lock);
    return retval;
}
```

---

## 8.8 release() Operation

```c
static int my_release(struct inode *inode, struct file *filp)
{
    struct my_dev *dev = filp->private_data;

    atomic_dec(&dev->open_count);
    dev_dbg(dev->dev, "closed by PID %d\n", current->pid);
    return 0;
}
```

> **Note**: `release` is called when the **last** file descriptor to the file is closed (i.e., `struct file` refcount reaches 0). `flush` is called on every `close()`.

---

## 8.9 ioctl Interface

### Defining Commands

```c
/* my_driver_ioctl.h — shared between kernel and userspace */
#include <linux/ioctl.h>

#define MY_IOC_MAGIC 'M'   /* Unique magic number */

/* Command definitions */
#define MY_IOC_RESET    _IO(MY_IOC_MAGIC, 0)           /* No data */
#define MY_IOC_GET_INFO _IOR(MY_IOC_MAGIC, 1, struct my_info)  /* Read */
#define MY_IOC_SET_CFG  _IOW(MY_IOC_MAGIC, 2, struct my_cfg)   /* Write */
#define MY_IOC_XFER     _IOWR(MY_IOC_MAGIC, 3, struct my_xfer) /* R+W */

struct my_info {
    __u32 version;
    __u32 status;
    __u32 fifo_depth;
};

struct my_cfg {
    __u32 baudrate;
    __u8  parity;
    __u8  stopbits;
};
```

### ioctl Implementation

```c
static long my_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    struct my_dev *dev = filp->private_data;
    void __user *argp = (void __user *)arg;
    int ret = 0;

    /* Validate command */
    if (_IOC_TYPE(cmd) != MY_IOC_MAGIC)
        return -ENOTTY;

    /* Check access direction */
    if (_IOC_DIR(cmd) & _IOC_READ)
        if (!access_ok(argp, _IOC_SIZE(cmd)))
            return -EFAULT;
    if (_IOC_DIR(cmd) & _IOC_WRITE)
        if (!access_ok(argp, _IOC_SIZE(cmd)))
            return -EFAULT;

    switch (cmd) {
    case MY_IOC_RESET:
        writel(0x01, dev->base + REG_RESET);
        break;

    case MY_IOC_GET_INFO: {
        struct my_info info = {
            .version = readl(dev->base + REG_VERSION),
            .status  = readl(dev->base + REG_STATUS),
            .fifo_depth = 256,
        };
        if (copy_to_user(argp, &info, sizeof(info)))
            ret = -EFAULT;
        break;
    }

    case MY_IOC_SET_CFG: {
        struct my_cfg cfg;
        if (copy_from_user(&cfg, argp, sizeof(cfg))) {
            ret = -EFAULT;
            break;
        }
        writel(cfg.baudrate, dev->base + REG_BAUD);
        writel(cfg.parity | (cfg.stopbits << 4),
               dev->base + REG_LINE_CTRL);
        break;
    }

    default:
        ret = -ENOTTY;   /* Unknown command */
    }

    return ret;
}
```

### Userspace ioctl Usage

```c
/* User-space application */
#include <fcntl.h>
#include <sys/ioctl.h>
#include "my_driver_ioctl.h"

int main(void)
{
    int fd = open("/dev/mydev", O_RDWR);
    if (fd < 0) { perror("open"); return 1; }

    /* Reset device */
    ioctl(fd, MY_IOC_RESET);

    /* Get info */
    struct my_info info;
    ioctl(fd, MY_IOC_GET_INFO, &info);
    printf("Version: %u, Status: %u\n", info.version, info.status);

    /* Set configuration */
    struct my_cfg cfg = { .baudrate = 115200, .parity = 0, .stopbits = 1 };
    ioctl(fd, MY_IOC_SET_CFG, &cfg);

    close(fd);
    return 0;
}
```

---

## Complete Character Driver Example

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * complete_chardev.c - Feature-complete character device driver
 */
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/uaccess.h>
#include <linux/mutex.h>
#include <linux/slab.h>

#define DEVICE_NAME  "mychardev"
#define BUF_SIZE     4096
#define NUM_DEVICES  4

struct mychar_dev {
    struct cdev cdev;
    struct device *device;
    struct mutex lock;
    char *data;
    size_t size;
    dev_t devno;
};

static struct class *mychar_class;
static dev_t mychar_first;
static struct mychar_dev mychar_devices[NUM_DEVICES];

static int mychar_open(struct inode *inode, struct file *filp)
{
    struct mychar_dev *dev = container_of(inode->i_cdev,
                                          struct mychar_dev, cdev);
    filp->private_data = dev;
    return 0;
}

static ssize_t mychar_read(struct file *filp, char __user *buf,
                           size_t count, loff_t *ppos)
{
    struct mychar_dev *dev = filp->private_data;
    ssize_t ret;

    mutex_lock(&dev->lock);
    if (*ppos >= dev->size) { ret = 0; goto out; }
    if (*ppos + count > dev->size)
        count = dev->size - *ppos;
    if (copy_to_user(buf, dev->data + *ppos, count)) {
        ret = -EFAULT; goto out;
    }
    *ppos += count;
    ret = count;
out:
    mutex_unlock(&dev->lock);
    return ret;
}

static ssize_t mychar_write(struct file *filp, const char __user *buf,
                            size_t count, loff_t *ppos)
{
    struct mychar_dev *dev = filp->private_data;
    ssize_t ret;

    mutex_lock(&dev->lock);
    if (*ppos + count > BUF_SIZE)
        count = BUF_SIZE - *ppos;
    if (count == 0) { ret = -ENOSPC; goto out; }
    if (copy_from_user(dev->data + *ppos, buf, count)) {
        ret = -EFAULT; goto out;
    }
    *ppos += count;
    if (dev->size < *ppos)
        dev->size = *ppos;
    ret = count;
out:
    mutex_unlock(&dev->lock);
    return ret;
}

static int mychar_release(struct inode *inode, struct file *filp)
{
    return 0;
}

static const struct file_operations mychar_fops = {
    .owner   = THIS_MODULE,
    .open    = mychar_open,
    .read    = mychar_read,
    .write   = mychar_write,
    .release = mychar_release,
};

static int __init mychar_init(void)
{
    int ret, i;

    ret = alloc_chrdev_region(&mychar_first, 0, NUM_DEVICES, DEVICE_NAME);
    if (ret < 0)
        return ret;

    mychar_class = class_create(DEVICE_NAME);
    if (IS_ERR(mychar_class)) {
        ret = PTR_ERR(mychar_class);
        goto err_region;
    }

    for (i = 0; i < NUM_DEVICES; i++) {
        struct mychar_dev *dev = &mychar_devices[i];
        dev->devno = MKDEV(MAJOR(mychar_first), i);
        dev->data = kzalloc(BUF_SIZE, GFP_KERNEL);
        if (!dev->data) { ret = -ENOMEM; goto err_devs; }
        mutex_init(&dev->lock);
        cdev_init(&dev->cdev, &mychar_fops);
        dev->cdev.owner = THIS_MODULE;
        ret = cdev_add(&dev->cdev, dev->devno, 1);
        if (ret) goto err_devs;
        dev->device = device_create(mychar_class, NULL, dev->devno,
                                    NULL, DEVICE_NAME "%d", i);
        if (IS_ERR(dev->device)) { ret = PTR_ERR(dev->device); goto err_devs; }
    }
    pr_info("mychardev: registered %d devices, major %d\n",
            NUM_DEVICES, MAJOR(mychar_first));
    return 0;

err_devs:
    for (i--; i >= 0; i--) {
        device_destroy(mychar_class, mychar_devices[i].devno);
        cdev_del(&mychar_devices[i].cdev);
        kfree(mychar_devices[i].data);
    }
    class_destroy(mychar_class);
err_region:
    unregister_chrdev_region(mychar_first, NUM_DEVICES);
    return ret;
}

static void __exit mychar_exit(void)
{
    int i;
    for (i = 0; i < NUM_DEVICES; i++) {
        device_destroy(mychar_class, mychar_devices[i].devno);
        cdev_del(&mychar_devices[i].cdev);
        kfree(mychar_devices[i].data);
    }
    class_destroy(mychar_class);
    unregister_chrdev_region(mychar_first, NUM_DEVICES);
    pr_info("mychardev: unregistered\n");
}

module_init(mychar_init);
module_exit(mychar_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Example");
MODULE_DESCRIPTION("Complete character device driver");
```

---

## VFS → Char Driver Call Path Diagram

```
   User:  fd = open("/dev/mychardev0", O_RDWR)
          │
          ▼
   VFS:   do_sys_openat2()
          │
          ▼
          path_openat() → resolve /dev/mychardev0
          │
          ▼
          inode lookup → inode->i_rdev = MKDEV(major, minor)
          │
          ▼
          chrdev_open()              ← fs/char_dev.c
          │
          ├── kobj_lookup(cdev_map, idev) → finds your cdev
          ├── filp->f_op = cdev->ops  (your file_operations)
          └── filp->f_op->open(inode, filp) → YOUR open()
          │
          ▼
   Subsequent read(fd, buf, len):
          ▼
          ksys_read() → vfs_read() → new_sync_read()
          → filp->f_op->read(filp, buf, len, &pos)
          → YOUR read()
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| `include/linux/fs.h` | `struct file_operations`, `struct inode` |
| `include/linux/cdev.h` | `struct cdev`, `cdev_init/add/del` |
| `fs/char_dev.c` | `chrdev_open()`, `register_chrdev_region()` |
| `include/linux/uaccess.h` | `copy_to/from_user()` |
| `include/linux/ioctl.h` | `_IO`, `_IOR`, `_IOW`, `_IOWR` macros |

---

## OS Comparison

| Aspect | Linux Char Driver | Windows (WDF) | macOS (DriverKit) |
|--------|-------------------|---------------|-------------------|
| File ops | `struct file_operations` | WDF I/O queue callbacks | IOUserClient methods |
| Registration | `cdev_add()` + `device_create()` | `WdfDeviceCreateDeviceInterface()` | `IOServiceOpen()` |
| User access | `/dev/xxx` + open/read/write | `\\.\DeviceName` + CreateFile | IOKit user client |
| ioctl | `_IOC()` macros | `IOCTL_CODE()` macro | `ExternalMethod()` |

---

## Interview Questions

**Q1: Explain the complete flow from user `open("/dev/x")` to driver `open()`.**
A: User calls open() → glibc traps syscall → VFS resolves path → finds inode → inode->i_rdev contains major:minor → chrdev_open() looks up cdev from major:minor → sets filp->f_op = cdev->ops → calls fops->open(inode, filp).

**Q2: What is the difference between `open()` and `release()` in terms of when they're called?**
A: `open()` is called every time a process calls open() on the device. `release()` is called only when the last file descriptor pointing to that struct file is closed (all dup'd and fork'd copies). Use `flush()` if you need per-close notification.

**Q3: Why must ioctl commands use `_IO/_IOR/_IOW/_IOWR` macros?**
A: These encode the direction (read/write), type (magic number), number, and data size into the command integer. This enables: validation of command parameters, automatic argument passing direction checking, and uniqueness across different drivers.

**Q4: What happens if `copy_to_user()` fails?**
A: It returns the number of bytes NOT copied. If non-zero, the driver should return `-EFAULT` to indicate a bad address. The user process may have provided an invalid pointer or unmapped memory.

**Q5: How do you handle concurrent access in a character driver?**
A: Use `mutex_lock()/mutex_unlock()` around shared data access in read/write/ioctl. Use `atomiс` operations for simple counters. Use wait queues for blocking read/write. Use spinlocks only for interrupt-context shared data.

---

## Summary

| Component | Purpose | Key API |
|-----------|---------|---------|
| Device number | Identify driver + instance | `alloc_chrdev_region()` |
| cdev | VFS-to-driver mapping | `cdev_init()`, `cdev_add()` |
| class + device | Auto-create /dev node | `class_create()`, `device_create()` |
| file_operations | User operation handlers | open, read, write, ioctl, release |
| copy_to/from_user | Safe user ↔ kernel transfer | Never dereference __user ptrs |
| ioctl | Device control | `_IO`, `_IOR`, `_IOW`, `_IOWR` |

---

*Next: [Chapter 9 — Block Device Drivers](Chapter_09_Block_Device_Drivers.md)*
