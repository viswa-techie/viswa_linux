# Chapter 8: Linux Kernel Modules

## Learning Goals
- Understand the kernel module concept and why it exists
- Write, build, load, and unload kernel modules
- Understand module dependencies and symbol resolution
- Know module parameters, licensing, and best practices

---

## 8.1 Kernel Module Concept

A **kernel module** is a piece of kernel code that can be loaded and unloaded at runtime without rebooting.

```
Static (Built-in) vs Dynamic (Module):

CONFIG_EXT4_FS=y  ←── Built into vmlinux (always present)
CONFIG_EXT4_FS=m  ←── Compiled as ext4.ko (load on demand)

┌───────────────────────────────────────────────────────┐
│                    vmlinux                             │
│  (statically linked kernel image)                     │
│                                                       │
│  Core: scheduler, MM, VFS, syscall, networking        │
│  Built-in drivers: CONFIG_xxx=y                       │
└───────────────────────┬───────────────────────────────┘
                        │ insmod / modprobe
          ┌─────────────┼──────────────────┐
          │             │                  │
    ┌─────▼────┐  ┌─────▼────┐  ┌──────────▼─────┐
    │  ext4.ko │  │  usb.ko  │  │  my_driver.ko  │
    │  (module)│  │  (module)│  │  (module)       │
    └──────────┘  └──────────┘  └────────────────┘
    Loadable Kernel Modules (.ko files)
```

Why modules exist:

| Reason | Details |
|--------|---------|
| **Memory efficiency** | Only load drivers for present hardware |
| **No reboot needed** | Add/remove functionality at runtime |
| **Development speed** | Rebuild one module instead of entire kernel |
| **Distribution flexibility** | Ship one kernel, many optional modules |
| **Hardware support** | Support new hardware without kernel recompile |

---

## 8.2 Loadable Kernel Modules (LKM)

Complete module example:

```c
/* hello_module.c — Complete kernel module */
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Engineer");
MODULE_DESCRIPTION("Hello World kernel module");
MODULE_VERSION("1.0");

static char *whom = "World";
module_param(whom, charp, 0644);  /* /sys/module/hello_module/parameters/whom */
MODULE_PARM_DESC(whom, "Name to greet");

static int count = 1;
module_param(count, int, 0644);
MODULE_PARM_DESC(count, "Number of greetings");

/* Called when module is loaded (insmod/modprobe) */
static int __init hello_init(void)
{
    int i;
    for (i = 0; i < count; i++)
        pr_info("Hello, %s! (%d/%d)\n", whom, i + 1, count);
    return 0;  /* 0 = success, negative errno = failure */
}

/* Called when module is unloaded (rmmod) */
static void __exit hello_exit(void)
{
    pr_info("Goodbye, %s!\n", whom);
}

module_init(hello_init);  /* Register init function */
module_exit(hello_exit);  /* Register exit function */
```

Building the module:

```makefile
# Makefile for out-of-tree module
obj-m += hello_module.o

# For multi-file modules:
# obj-m += my_driver.o
# my_driver-objs := core.o utils.o hw.o

KDIR := /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# Build
make

# Load with parameters
sudo insmod hello_module.ko whom="Linux" count=3

# Check messages
dmesg | tail -5
# [  123.456] Hello, Linux! (1/3)
# [  123.456] Hello, Linux! (2/3)
# [  123.456] Hello, Linux! (3/3)

# View module info
modinfo hello_module.ko
lsmod | grep hello

# View/change parameter at runtime
cat /sys/module/hello_module/parameters/whom   # "Linux"
echo "Kernel" > /sys/module/hello_module/parameters/whom

# Unload
sudo rmmod hello_module
dmesg | tail -1
# [  130.789] Goodbye, Linux!
```

---

## 8.3 Module Lifecycle

```
Module Lifecycle:

                    .ko file on disk
                         │
                    ┌────▼─────┐
              ┌─────│  insmod  │────┐
              │     │ modprobe │    │
              │     └──────────┘    │
              │                     │
              │ 1. Load .ko into    │ If dependencies needed,
              │    kernel memory    │ modprobe loads them first
              │                     │
              │ 2. Resolve symbols  │ Link to kernel exports
              │    (EXPORT_SYMBOL)  │
              │                     │
              │ 3. Call module_init │ __init function
              │    function         │
              │                     │
              ▼                     │
       ┌──────────────┐            │
       │  MODULE       │            │
       │  LOADED       │            │
       │               │            │
       │  - Hooked into│            │
       │    driver model│           │
       │  - IRQs registered         │
       │  - /dev entries │          │
       │  - sysfs attrs  │          │
       └──────┬───────┘            │
              │                     │
              │ rmmod               │
              │                     │
              │ 1. Check refcount   │ refcount > 0 → refuse
              │    (lsmod: Used by) │
              │                     │
              │ 2. Call module_exit │ __exit function
              │    function         │ (cleanup, unregister)
              │                     │
              │ 3. Free module      │
              │    memory           │
              ▼                     │
       ┌──────────────┐            │
       │  UNLOADED    │            │
       └──────────────┘            │
```

```c
/* __init and __exit annotations */

static int __init my_init(void) { ... }
/* __init = this function is only needed during init
   Its memory is freed after init completes */

static void __exit my_exit(void) { ... }
/* __exit = not needed if module is built-in (CONFIG_xxx=y)
   Compiler discards it for built-in modules */

/* __initdata = data only needed during init */
static int __initdata initial_value = 42;
/* Memory freed after init, saving RAM */
```

---

## 8.4 Module Dependencies

```
Module Dependency Chain:

  my_camera.ko
       │ depends on
       ▼
  v4l2_common.ko
       │ depends on
       ▼
  videodev.ko
       │ depends on
       ▼
  media.ko (core media framework)

modprobe handles this automatically:
  $ modprobe my_camera
  → loads: media.ko → videodev.ko → v4l2_common.ko → my_camera.ko

insmod does NOT handle dependencies:
  $ insmod my_camera.ko
  → ERROR: Unknown symbol in module (if deps not loaded)
```

Symbol export:

```c
/* Module A exports a function */
int my_helper_function(int x)
{
    return x * 2;
}
EXPORT_SYMBOL(my_helper_function);       /* Available to all modules */
EXPORT_SYMBOL_GPL(my_helper_function);   /* Only to GPL-licensed modules */

/* Module B uses it */
extern int my_helper_function(int x);

static int __init mod_b_init(void)
{
    int result = my_helper_function(21);  /* Uses Module A's function */
    pr_info("Result: %d\n", result);      /* 42 */
    return 0;
}
```

```bash
# View module dependencies
modinfo my_camera.ko | grep depends
# depends: v4l2_common,videodev,media

# View loaded modules and dependencies
lsmod
# Module        Size   Used by
# my_camera     16384  0
# v4l2_common    8192  1 my_camera
# videodev      81920  2 my_camera,v4l2_common
# media         32768  1 videodev

# Generate dependency database
depmod -a

# Dependency file
cat /lib/modules/$(uname -r)/modules.dep
```

---

## 8.5 Module Loading and Unloading

```
insmod vs modprobe:

insmod:
  - Direct loading of a .ko file
  - No dependency resolution
  - Must specify full path
  - $ insmod /path/to/module.ko param=val

modprobe:
  - Loads from /lib/modules/$(uname -r)/
  - Automatic dependency resolution (uses modules.dep)
  - Handles module parameters from /etc/modprobe.d/
  - $ modprobe module_name param=val

Module search path:
  /lib/modules/$(uname -r)/
  ├── kernel/
  │   ├── drivers/         ← Most device drivers
  │   ├── fs/              ← File system modules
  │   ├── net/             ← Network modules
  │   ├── crypto/          ← Crypto modules
  │   └── sound/           ← Sound modules
  ├── modules.dep          ← Dependency database
  ├── modules.alias        ← Device ID → module mapping
  ├── modules.symbols      ← Symbol → module mapping
  └── extra/               ← Out-of-tree modules
```

Module autoloading:

```
Device Plug → Module Auto-Load:

1. Device appears (USB plug, DT match, PCI enumeration)
          │
2. Kernel generates uevent with MODALIAS
   │  MODALIAS=usb:v1234p5678...
   │  MODALIAS=of:Nmydevice<compatible>
          │
3. udev/systemd-udevd receives uevent
          │
4. udev runs: modprobe $MODALIAS
          │
5. modprobe searches modules.alias for matching module
          │
6. Module loaded → probe() called → device works
```

```bash
# View module aliases
modinfo ext4 | grep alias
# alias: fs-ext4

# Manually trigger module loading
modprobe ext4

# Blacklist a module (prevent loading)
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf

# Module options in config
echo "options snd_hda_intel power_save=1" >> /etc/modprobe.d/audio.conf
```

---

## Kernel Source References

| File | Content |
|------|---------|
| `kernel/module/main.c` | Module loading core (load_module()) |
| `include/linux/module.h` | Module structures and macros |
| `include/linux/init.h` | `__init`, `__exit`, `module_init()` |
| `include/linux/moduleparam.h` | Module parameter declarations |
| `scripts/mod/modpost.c` | Module post-processing (symbol checks) |
| `kernel/module/strict_rwx.c` | Module memory protection |

---

## Interview Questions

**Q1: What is the difference between insmod and modprobe?**
A: `insmod` loads a specific `.ko` file directly — no dependency handling, you must load dependencies manually. `modprobe` searches `/lib/modules/$(uname -r)/`, resolves and loads dependencies automatically using `modules.dep`, and reads configuration from `/etc/modprobe.d/`. Always prefer `modprobe` in production.

**Q2: What does `__init` do and why is it important in embedded systems?**
A: `__init` marks a function as only needed during initialization. The kernel places `__init` functions in a special section (`.init.text`) that is freed after boot, reclaiming that memory. On embedded systems with limited RAM, this is significant — hundreds of KB of init code can be freed.

**Q3: What happens if module_init() returns a non-zero value?**
A: The module load fails. The kernel prints an error, does NOT call `module_exit()`, and frees the module memory. The `insmod`/`modprobe` command returns an error. Any resources allocated before the error must be cleaned up before returning the error code — devm_ APIs handle this automatically.

**Q4: What is EXPORT_SYMBOL_GPL vs EXPORT_SYMBOL?**
A: `EXPORT_SYMBOL()` exports a symbol to all kernel modules. `EXPORT_SYMBOL_GPL()` exports only to modules with a GPL-compatible license. This enforces the GPL: proprietary modules can't use GPL-only APIs. Critical kernel APIs (like many driver framework functions) are GPL-only.

---

## Summary

- Kernel modules allow loading/unloading code at runtime without rebooting
- Every module needs `module_init()` and `module_exit()` functions
- `__init`/`__exit` annotations save memory by discarding init-only code
- Dependencies are resolved by `modprobe` using `modules.dep`
- `EXPORT_SYMBOL()`/`EXPORT_SYMBOL_GPL()` controls inter-module symbol visibility
- Module autoloading works through udev detecting device MODALIAS events
- Module parameters are accessible via `/sys/module/<name>/parameters/`

---

*Next: [Chapter 9 — Booting Fundamentals](Chapter_09_Booting_Fundamentals.md)*
