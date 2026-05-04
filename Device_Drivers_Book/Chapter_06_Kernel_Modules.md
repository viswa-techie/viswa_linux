# Chapter 6: Linux Kernel Modules

## Chapter Overview

Kernel modules are the mechanism by which drivers are loaded and unloaded at runtime without rebooting. This chapter covers the complete module lifecycle, from writing to compiling to loading.

---

## 6.1 Kernel Module Concept

A **kernel module** is a piece of code that can be loaded into the Linux kernel at runtime. It extends kernel functionality without recompiling or rebooting.

```
┌──────────────────────────────────────────┐
│            vmlinux (base kernel)          │
│   ┌──────────┐  ┌──────────┐            │
│   │ Built-in │  │ Built-in │            │
│   │ Driver A │  │ Subsys B │            │
│   └──────────┘  └──────────┘            │
│                                          │
│   ┌──────────┐  ┌──────────┐  ← loaded  │
│   │ Module C │  │ Module D │    at      │
│   │ (.ko)    │  │ (.ko)    │    runtime │
│   └──────────┘  └──────────┘            │
└──────────────────────────────────────────┘
```

### Why Modules?
- Load drivers only when hardware is present → saves memory
- Develop and test drivers without rebooting
- Distribute proprietary drivers separately (legally complex)
- Users can customize kernel functionality

---

## 6.2 Loadable Kernel Modules (LKM)

### Anatomy of a .ko File

A `.ko` (kernel object) is an ELF relocatable file containing:

```
my_driver.ko (ELF format)
├── .text          ← executable code
├── .data          ← initialized data
├── .bss           ← uninitialized data
├── .rodata        ← read-only data (strings, tables)
├── .modinfo       ← MODULE_LICENSE, MODULE_AUTHOR, etc.
├── .init.text     ← __init functions (freed after init)
├── .exit.text     ← __exit functions
├── __versions     ← CRC of kernel symbols (version magic)
├── .symtab        ← symbols this module exports/uses
└── .gnu.linkonce.this_module  ← struct module
```

### The struct module

```c
/* kernel/module/internal.h */
struct module {
    enum module_state state;    /* MODULE_STATE_LIVE, GOING, COMING */
    char name[MODULE_NAME_LEN];
    struct list_head list;
    
    int (*init)(void);          /* module_init function pointer */
    void (*exit)(void);         /* module_exit function pointer */
    
    unsigned int num_syms;      /* exported symbols */
    const struct kernel_symbol *syms;
    
    struct module_sect_attrs *sect_attrs;
    struct module_notes_attrs *notes_attrs;
    
    unsigned long core_size;    /* size of core section */
    void *core_layout;          /* core section in kernel memory */
    /* ... many more fields */
};
```

---

## 6.3 Module Lifecycle

```
         ┌─────────────┐
         │  Not Loaded  │
         └──────┬───────┘
                │  insmod / modprobe
                ▼
         ┌─────────────┐
         │   COMING     │  load_module() running
         │ (initializing)│  sections allocated, relocations applied
         └──────┬───────┘
                │  module->init() called
                ▼
         ┌─────────────┐
         │    LIVE      │  Module fully operational
         │  (running)   │  .init.text freed
         └──────┬───────┘
                │  rmmod
                ▼
         ┌─────────────┐
         │   GOING      │  module->exit() called
         │ (cleaning up)│  resources freed
         └──────┬───────┘
                │  memory freed
                ▼
         ┌─────────────┐
         │  Unloaded    │
         └─────────────┘
```

---

## 6.4 Module Initialization and Cleanup

### Minimal Module

```c
/* my_driver.c */
#include <linux/module.h>
#include <linux/init.h>

static int __init my_init(void)
{
    pr_info("my_driver: loaded\n");
    /* Register with subsystem, allocate resources */
    return 0;    /* 0 = success, negative = error */
}

static void __exit my_exit(void)
{
    pr_info("my_driver: unloaded\n");
    /* Deregister, free resources */
}

module_init(my_init);    /* Called on insmod */
module_exit(my_exit);    /* Called on rmmod */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Developer Name");
MODULE_DESCRIPTION("Example device driver");
MODULE_VERSION("1.0");
```

### __init and __exit Sections

```c
static int __init my_init(void) { ... }
/*            ^
 * __init places this function in the .init.text section.
 * After init completes, this section is freed to save memory.
 * NEVER call an __init function after initialization!
 */

static void __exit my_exit(void) { ... }
/*            ^
 * __exit marks this for the .exit.text section.
 * If built-in (not module), the exit function is discarded
 * since built-in drivers can't be removed.
 */

static int __initdata my_param = 42;
/*            ^
 * __initdata places data in .init.data (freed after init)
 */
```

---

## 6.5 Module Dependencies

```
$ modinfo my_complex_driver.ko
...
depends:        i2c-core,regmap-i2c,industrialio
...

Dependency chain:
  my_complex_driver
       │
       ├── depends on i2c-core        (I2C subsystem)
       ├── depends on regmap-i2c      (Register map framework)
       └── depends on industrialio    (IIO subsystem)
```

### How Dependencies Are Resolved

```bash
# insmod: NO dependency resolution — fails if dependencies missing
sudo insmod my_complex_driver.ko
# Error: Unknown symbol i2c_transfer (err 0)

# modprobe: reads modules.dep, loads dependencies first
sudo modprobe my_complex_driver
# Loads: i2c-core → regmap-i2c → industrialio → my_complex_driver
```

### Symbol Dependencies

```c
/* Module A exports a symbol */
void helper_function(int x) { ... }
EXPORT_SYMBOL(helper_function);        /* Available to all modules */
EXPORT_SYMBOL_GPL(helper_function);    /* Available only to GPL modules */

/* Module B uses it */
extern void helper_function(int x);
/* Kernel linker resolves this at module load time */
```

---

## 6.6 Kernel Module Compilation

### Out-of-Tree Kbuild Makefile

```makefile
# Makefile
obj-m += my_driver.o                  # Single-file module

# Multi-file module:
obj-m += my_complex.o
my_complex-objs := main.o hw.o irq.o  # Links main.o + hw.o + irq.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean

# Cross-compile for ARM64:
# KDIR=/path/to/arm64/kernel
# make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
```

### In-Tree Kbuild Integration

```makefile
# drivers/misc/Kconfig
config MY_DRIVER
	tristate "My awesome driver"
	depends on I2C
	select REGMAP_I2C
	help
	  Driver for the awesome device.
	  Say Y to build-in, M for module, N to disable.

# drivers/misc/Makefile
obj-$(CONFIG_MY_DRIVER) += my_driver.o
```

### Build Process

```
make modules
       │
       ▼
  Kbuild reads obj-m/obj-y
       │
       ▼
  gcc -c my_driver.c -o my_driver.o     ← Compile
       │
       ▼
  ld -r my_driver.o -o my_driver.ko     ← Link (relocatable)
       │
       ▼
  modpost: generate __versions (CRC)     ← Version magic
       │
       ▼
  sign module (if CONFIG_MODULE_SIG)     ← Optional signature
       │
       ▼
  my_driver.ko ready
```

---

## 6.7 Kernel Module Loading and Unloading

### Loading a Module

```bash
# Method 1: insmod (no dependency resolution)
sudo insmod /path/to/my_driver.ko
sudo insmod /path/to/my_driver.ko debug_level=3   # With parameters

# Method 2: modprobe (with dependency resolution)
sudo modprobe my_driver
sudo modprobe my_driver debug_level=3

# Check if loaded
lsmod | grep my_driver
cat /proc/modules | grep my_driver

# Read module info
modinfo my_driver
modinfo /path/to/my_driver.ko
```

### Unloading a Module

```bash
# Unload
sudo rmmod my_driver
sudo modprobe -r my_driver     # Also removes unused dependencies

# Check if module is in use (refcount)
lsmod | grep my_driver
# my_driver  16384  2        ← "2" means 2 users (cannot unload)

# Force unload (dangerous! — can crash)
sudo rmmod -f my_driver        # Requires CONFIG_MODULE_FORCE_UNLOAD
```

### Module Parameters

```c
static int debug_level = 0;
module_param(debug_level, int, 0644);
MODULE_PARM_DESC(debug_level, "Debug verbosity level (0-3)");

static char *device_name = "default";
module_param(device_name, charp, 0444);
MODULE_PARM_DESC(device_name, "Name of the device");

static bool enable_dma = true;
module_param(enable_dma, bool, 0644);
MODULE_PARM_DESC(enable_dma, "Enable DMA transfers");
```

Parameters visible in sysfs:
```bash
$ ls /sys/module/my_driver/parameters/
debug_level  device_name  enable_dma

$ cat /sys/module/my_driver/parameters/debug_level
0

$ echo 2 > /sys/module/my_driver/parameters/debug_level   # Change at runtime
```

### Auto-Loading via udev

```
1. Device appears (USB plug, DT match, PCI scan)
2. Kernel sends uevent with MODALIAS
3. udev receives uevent
4. udev runs: modprobe $MODALIAS
5. modprobe searches modules.alias for matching module
6. Module loaded → driver probes device
```

```bash
# See module aliases
modinfo my_i2c_driver | grep alias
# alias: of:N*T*Cvendor,temp-sensor
# alias: i2c:temp-sensor

# See device's modalias
cat /sys/devices/platform/soc/2010000.i2c/0-0048/modalias
# of:Ntemp-sensorT<NULL>Cvendor,temp-sensor
```

---

## Complete Module Template

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * my_driver.c - Complete module template
 */
#include <linux/module.h>
#include <linux/init.h>
#include <linux/platform_device.h>
#include <linux/of.h>

/* Module parameters */
static int debug_level = 0;
module_param(debug_level, int, 0644);
MODULE_PARM_DESC(debug_level, "Debug level (0=off, 1=info, 2=verbose)");

/* Private data */
struct my_priv {
    void __iomem *regs;
    int irq;
};

static int my_probe(struct platform_device *pdev)
{
    struct my_priv *priv;

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    priv->regs = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(priv->regs))
        return PTR_ERR(priv->regs);

    priv->irq = platform_get_irq(pdev, 0);
    if (priv->irq < 0)
        return priv->irq;

    platform_set_drvdata(pdev, priv);
    dev_info(&pdev->dev, "probed successfully\n");
    return 0;
}

static void my_remove(struct platform_device *pdev)
{
    dev_info(&pdev->dev, "removed\n");
}

static const struct of_device_id my_dt_match[] = {
    { .compatible = "vendor,my-device" },
    { }
};
MODULE_DEVICE_TABLE(of, my_dt_match);

static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "my-device",
        .of_match_table = my_dt_match,
    },
};
module_platform_driver(my_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("My device driver");
MODULE_VERSION("1.0");
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| `kernel/module/main.c` | Module loading, `load_module()` |
| `kernel/module/kmod.c` | `request_module()` for autoloading |
| `include/linux/module.h` | `struct module`, macros |
| `include/linux/init.h` | `__init`, `__exit`, `module_init()` |
| `include/linux/moduleparam.h` | `module_param()` |
| `scripts/mod/modpost.c` | Module post-processing (version CRC) |

---

## Debugging

```bash
# Kernel messages from module
dmesg | grep "my_driver"

# Module section addresses (for debugging with GDB)
cat /sys/module/my_driver/sections/.text
cat /sys/module/my_driver/sections/.data

# See all symbols a module uses
nm my_driver.ko | head -20

# Verify module version magic matches kernel
modinfo my_driver.ko | grep vermagic
uname -r
```

---

## OS Comparison

| Aspect | Linux Module | Windows Driver (.sys) | macOS Kext/Dext |
|--------|-------------|----------------------|-----------------|
| Format | .ko (ELF) | .sys (PE) | .kext (Mach-O bundle) |
| Load mechanism | `insmod`/`modprobe` | PnP Manager + SCM | `kextload`/IOKit matching |
| Auto-load trigger | udev + MODALIAS | PnP + INF | IOKit matching dictionary |
| Parameters | `module_param()` | Registry entries | Info.plist + IORegistryEntry |
| Signing | Optional (`MODULE_SIG`) | Required (WHQL) | Required (notarized) |
| Unload | `rmmod` | Service stop | `kextunload` |

---

## Interview Questions

**Q1: What is the difference between `insmod` and `modprobe`?**
A: `insmod` loads a single .ko by path, with no dependency resolution. `modprobe` reads `/lib/modules/$(uname -r)/modules.dep` and loads all required dependencies first, then loads the requested module by name (not path).

**Q2: What happens when module_init() returns a non-zero value?**
A: The module load fails. The kernel logs an error, frees the allocated memory for the module, and it does not appear in `lsmod`. The error code propagates to the `insmod` command.

**Q3: Why should large data structures not be declared __initdata?**
A: `__initdata` is placed in `.init.data`, which is freed after init. If you accidentally dereference an `__initdata` variable after init, you access freed memory — use-after-free bug. Only data used exclusively during init should be `__initdata`.

**Q4: How does automatic module loading work?**
A: Kernel sends uevent with MODALIAS environment variable → udev receives it → runs `modprobe $MODALIAS` → modprobe searches `modules.alias` (built from `MODULE_DEVICE_TABLE` macros) → finds matching module → loads it.

**Q5: Can you unload a module that is in use?**
A: No. Each open file handle, bound device, or dependency increments the module's `refcnt`. `rmmod` fails with "Module in use." You must close all handles and unbind devices first. `rmmod -f` can force-unload if configured, but risks system crash.

---

## Summary

| Concept | Key Point |
|---------|-----------|
| .ko file | ELF relocatable with module metadata, __init, __exit sections |
| Lifecycle | insmod → init() → LIVE → rmmod → exit() → freed |
| Parameters | `module_param()` with sysfs writability |
| Dependencies | `modprobe` resolves; `insmod` doesn't |
| EXPORT_SYMBOL | Share functions between modules (_GPL for GPL-only) |
| Module autoloading | udev + MODALIAS + MODULE_DEVICE_TABLE |
| devm_* pattern | Automatic cleanup tied to device lifetime |

---

*Next: [Chapter 7 — Writing a Basic Device Driver](Chapter_07_Writing_Basic_Driver.md)*
