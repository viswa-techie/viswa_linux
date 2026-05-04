# Chapter 27: Security in Device Drivers

## Chapter Overview

Drivers run in kernel space with full privileges — a single vulnerability can compromise the entire system. This chapter covers the kernel security model, common driver vulnerabilities, protection mechanisms, and secure design principles.

---

## 27.1 Kernel Security Model

```
┌───────────────────────────────────────┐
│          User Space (Ring 3)          │
│  Limited access — syscall boundary    │
├───────────────────────────────────────┤
│         Kernel Space (Ring 0)         │
│  ├── Core kernel (mm, sched, vfs)     │
│  ├── Drivers (full kernel access)     │
│  ├── LSM hooks (SELinux, AppArmor)    │
│  └── seccomp (syscall filtering)      │
├───────────────────────────────────────┤
│          Hardware (Ring -1: SMM)      │
└───────────────────────────────────────┘
```

Drivers have the **same privilege level** as the kernel itself. A buggy driver can:
- Read/write any physical memory
- Corrupt kernel data structures
- Escalate privileges for any process
- Bypass all access controls

---

## 27.2 Common Driver Vulnerabilities

### 1. Buffer Overflow

```c
/* VULNERABLE */
static ssize_t my_write(struct file *f, const char __user *buf,
                         size_t count, loff_t *off)
{
    char kbuf[256];
    copy_from_user(kbuf, buf, count);  /* count could be > 256! */
}

/* SECURE */
static ssize_t my_write(struct file *f, const char __user *buf,
                         size_t count, loff_t *off)
{
    char kbuf[256];
    if (count > sizeof(kbuf))
        return -EINVAL;
    if (copy_from_user(kbuf, buf, count))
        return -EFAULT;
}
```

### 2. Missing Access Checks

```c
/* VULNERABLE: any user can read hardware registers */
static ssize_t regs_show(struct device *dev, struct device_attribute *attr,
                          char *buf)
{
    return sysfs_emit(buf, "0x%08x\n", readl(priv->base));
}

/* SECURE: proper file permissions + capability check */
static ssize_t regs_show(...)
{
    if (!capable(CAP_SYS_RAWIO))
        return -EPERM;
    return sysfs_emit(buf, "0x%08x\n", readl(priv->base));
}
```

### 3. Use-After-Free

```c
/* VULNERABLE: race between close and IRQ */
static int my_release(struct inode *inode, struct file *f)
{
    kfree(priv);            /* Free data */
    return 0;
}
/* Meanwhile, IRQ handler still uses priv → use-after-free */

/* SECURE: use reference counting or devm_ */
static int my_release(struct inode *inode, struct file *f)
{
    kref_put(&priv->ref, my_free);  /* Only free when refcount=0 */
    return 0;
}
```

### 4. Integer Overflow in ioctl

```c
/* VULNERABLE */
case MY_ALLOC:
    size = arg * sizeof(struct item);  /* Integer overflow possible! */
    buf = kmalloc(size, GFP_KERNEL);

/* SECURE */
case MY_ALLOC:
    if (arg > MAX_ITEMS)
        return -EINVAL;
    buf = kcalloc(arg, sizeof(struct item), GFP_KERNEL);  /* Safe multiply */
```

### 5. Information Leak

```c
/* VULNERABLE: leaking kernel stack data */
struct my_status st;
st.field1 = read_reg(REG1);
/* st padding bytes uninitialized → leak to userspace */
copy_to_user(ubuf, &st, sizeof(st));

/* SECURE: zero entire struct first */
struct my_status st = {};
st.field1 = read_reg(REG1);
copy_to_user(ubuf, &st, sizeof(st));
```

---

## 27.3 Kernel Protection Mechanisms

### Memory Protection

| Mechanism | Protection |
|-----------|------------|
| **KASLR** | Randomize kernel base address |
| **KASAN** | Detect use-after-free, out-of-bounds |
| **Stack canaries** | Detect stack buffer overflow |
| **STACKLEAK** | Clear kernel stack between syscalls |
| **HARDENED_USERCOPY** | Validate `copy_to/from_user()` bounds |
| **CONFIG_STRICT_DEVMEM** | Restrict /dev/mem access |

### copy_to/from_user() Importance

```c
/* These functions: */
/* 1. Verify the pointer is actually in user space */
/* 2. Handle page faults gracefully (return -EFAULT, not crash) */
/* 3. With HARDENED_USERCOPY: check buffer doesn't span slab objects */

/* NEVER access user memory directly: */
/* char c = *(char __user *)ubuf;  ← WRONG, crash/vulnerability */
```

### Capability Checks

```c
#include <linux/capability.h>

/* Common capability checks in drivers */
if (!capable(CAP_SYS_RAWIO))     /* Raw I/O access (registers, ports) */
if (!capable(CAP_SYS_ADMIN))     /* Administrative operations */
if (!capable(CAP_NET_ADMIN))     /* Network config */
if (!capable(CAP_SYS_MODULE))    /* Module loading */
```

---

## 27.4 IOMMU as Security Boundary

```
Without IOMMU:
  Device DMA → can access ANY physical memory
  (malicious/buggy device can read kernel memory)

With IOMMU:
  Device DMA → IOMMU page table → only mapped memory accessible
  (device contained to its DMA buffers)
```

Key for: PCIe devices, VFIO (VM passthrough), Thunderbolt security.

---

## 27.5 Secure Driver Design Principles

### 1. Validate All Input

```c
static long my_ioctl(struct file *f, unsigned int cmd, unsigned long arg)
{
    struct my_config cfg;

    if (cmd != MY_SET_CONFIG)
        return -ENOTTY;

    if (copy_from_user(&cfg, (void __user *)arg, sizeof(cfg)))
        return -EFAULT;

    /* Validate EVERY field */
    if (cfg.width == 0 || cfg.width > MAX_WIDTH)
        return -EINVAL;
    if (cfg.height == 0 || cfg.height > MAX_HEIGHT)
        return -EINVAL;
    if (cfg.format >= NUM_FORMATS)
        return -EINVAL;
}
```

### 2. Use devm_ for Resource Management

```c
/* Automatic cleanup prevents resource leaks on error paths */
base = devm_platform_ioremap_resource(pdev, 0);
clk  = devm_clk_get(dev, NULL);
irq  = devm_request_irq(dev, ...);
/* All freed automatically on remove — no manual cleanup needed */
```

### 3. Minimize Attack Surface

```c
/* Only expose necessary interfaces */
/* - Use 0644 not 0666 for sysfs */
/* - Use debugfs for debug info (not sysfs) */
/* - Don't expose raw register access to userspace */
/* - Use appropriate capabilities checks */
```

### 4. Handle Errors — Don't Ignore Them

```c
/* WRONG: silent failure */
ret = clk_prepare_enable(priv->clk);
writel(val, priv->base + REG);  /* Clock may not be running! */

/* RIGHT: check and propagate */
ret = clk_prepare_enable(priv->clk);
if (ret) {
    dev_err(dev, "failed to enable clock: %d\n", ret);
    return ret;
}
```

---

## 27.6 Security Checklist for Driver Review

```
□ All copy_to/from_user() return values checked
□ All ioctl arguments validated (range, overflow)
□ Structures zeroed before copy_to_user (no info leak)
□ No direct user pointer dereference
□ Proper locking (no TOCTOU races)
□ Reference counting for shared objects
□ Error paths free all resources (or use devm_)
□ Appropriate file permissions (not world-writable)
□ Capability checks for privileged operations
□ DMA buffers properly bounded
□ No hardcoded credentials or keys
□ Integer arithmetic checked for overflow
```

---

## Interview Questions

**Q1: Why can't you dereference a user-space pointer directly in kernel?**
A: 1) The pointer might be invalid (NULL, unmapped). 2) The page might be swapped out. 3) On some architectures, user/kernel address spaces are separate. 4) `copy_from_user()` handles faults gracefully (returns -EFAULT) instead of crashing. 5) HARDENED_USERCOPY validates the kernel buffer too.

**Q2: How does KASLR protect the kernel?**
A: KASLR randomizes the kernel's virtual base address at each boot. An attacker can't predict where kernel code/data resides, making return-oriented programming (ROP) attacks harder.

**Q3: What is a TOCTOU (time-of-check-time-of-use) vulnerability?**
A: Checking a condition and then acting on it without holding a lock. Example: check if buffer has space, then another thread fills it, then you write → overflow. Fix: hold a lock across check+use, or use atomic operations.

**Q4: How does IOMMU prevent DMA attacks?**
A: Without IOMMU, a PCI device can DMA to any physical address — a malicious device could read kernel memory. IOMMU interposes a page table: the device sees IOVAs that only map to explicitly granted physical pages. Unauthorized addresses cause IOMMU faults.

---

*Next: [Chapter 28 — Performance Optimization](Chapter_28_Performance.md)*
