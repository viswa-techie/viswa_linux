# Chapter 29: Important Driver Source Code Locations

## Chapter Overview

Understanding where code lives in the kernel source tree is essential for reading, modifying, and contributing drivers. This chapter maps the key directories and files every driver developer should know.

---

## 29.1 Top-Level Kernel Source Tree

```
linux/
├── arch/           ← Architecture-specific (arm64, x86, riscv)
├── block/          ← Block I/O layer (blk-mq, schedulers)
├── crypto/         ← Cryptographic API
├── Documentation/  ← Kernel docs (including DT bindings)
├── drivers/        ← ALL device drivers ★
├── firmware/       ← Built-in firmware blobs
├── fs/             ← Filesystems (VFS, ext4, proc, sysfs)
├── include/        ← Header files
├── init/           ← Kernel boot (main.c → start_kernel)
├── ipc/            ← Inter-process communication
├── kernel/         ← Core kernel (scheduler, IRQ, module)
├── lib/            ← Helper functions (string, sort, crc)
├── mm/             ← Memory management
├── net/            ← Network stack
├── scripts/        ← Build scripts (checkpatch, dtc)
├── security/       ← LSM, SELinux, AppArmor
├── sound/          ← ALSA sound subsystem
└── tools/          ← Userspace tools (perf, selftests)
```

---

## 29.2 drivers/ Directory — The Heart of Driver Code

```
drivers/
├── base/               ← Device model core ★
│   ├── core.c          ← struct device lifecycle
│   ├── bus.c           ← Bus registration, matching
│   ├── driver.c        ← Driver binding (really_probe)
│   ├── platform.c      ← Platform bus & driver
│   ├── dd.c            ← Device-driver binding logic
│   ├── devres.c        ← devm_* resource management
│   └── property.c      ← Firmware property access
│
├── char/               ← Character device drivers
│   ├── mem.c           ← /dev/mem, /dev/null, /dev/zero
│   └── random.c        ← /dev/random, /dev/urandom
│
├── of/                 ← Device Tree core
│   ├── base.c          ← DT node/property parsing
│   ├── platform.c      ← DT → platform_device creation
│   ├── irq.c           ← DT interrupt parsing
│   └── address.c       ← DT address translation
│
├── clk/                ← Clock framework
│   ├── clk.c           ← Core clock API
│   └── clk-*.c         ← SoC-specific clock drivers
│
├── gpio/               ← GPIO subsystem
│   ├── gpiolib.c       ← GPIO core
│   └── gpio-*.c        ← GPIO controller drivers
│
├── i2c/                ← I2C subsystem
│   ├── i2c-core-base.c ← I2C core
│   └── busses/         ← I2C controller drivers
│
├── spi/                ← SPI subsystem
│   ├── spi.c           ← SPI core
│   └── spi-*.c         ← SPI controller drivers
│
├── pci/                ← PCI subsystem
│   ├── pci.c           ← PCI core
│   ├── pci-driver.c    ← PCI driver binding
│   └── host/           ← PCIe host controller drivers
│
├── usb/                ← USB subsystem
│   ├── core/           ← USB core (hub, device, config)
│   └── host/           ← USB host controller (xhci, ehci)
│
├── net/                ← Network drivers
│   ├── ethernet/       ← Ethernet drivers (vendor subdirs)
│   ├── wireless/       ← Wi-Fi drivers
│   └── phy/            ← Ethernet PHY drivers
│
├── gpu/                ← Graphics drivers
│   └── drm/            ← DRM/KMS framework
│       ├── drm_drv.c   ← DRM core
│       └── amd/ intel/ ← Vendor GPU drivers
│
├── input/              ← Input subsystem
│   ├── input.c         ← Input core
│   ├── evdev.c         ← /dev/input/eventN handler
│   └── touchscreen/    ← Touchscreen drivers
│
├── media/              ← V4L2 / media subsystem
│   ├── v4l2-core/      ← V4L2 framework
│   └── platform/       ← SoC camera/video drivers
│
├── iommu/              ← IOMMU framework
│   ├── iommu.c         ← IOMMU core
│   ├── arm-smmu-v3.c   ← ARM SMMU
│   └── intel/          ← Intel VT-d
│
├── dma/                ← DMA engine framework
│   └── dmaengine.c     ← DMA core API
│
├── irqchip/            ← Interrupt controller drivers
│   ├── irq-gic.c       ← ARM GICv2
│   └── irq-gic-v3.c    ← ARM GICv3
│
├── pinctrl/            ← Pin muxing/configuration
├── regulator/          ← Voltage regulator framework
├── thermal/            ← Thermal management
├── watchdog/           ← Watchdog timer drivers
├── pwm/                ← PWM subsystem
├── mmc/                ← eMMC/SD card subsystem
├── nvme/               ← NVMe subsystem
├── scsi/               ← SCSI subsystem
├── tty/                ← TTY/serial subsystem
│   └── serial/         ← UART drivers
├── firmware/           ← Firmware loading framework
└── iio/                ← Industrial I/O (sensors)
```

---

## 29.3 include/ Directory — Header Files

```
include/
├── linux/              ← Core kernel headers
│   ├── device.h        ← struct device
│   ├── platform_device.h ← platform_device, platform_driver
│   ├── module.h        ← Module macros
│   ├── interrupt.h     ← IRQ APIs
│   ├── dma-mapping.h   ← DMA APIs
│   ├── of.h            ← Device Tree APIs
│   ├── clk.h           ← Clock APIs
│   ├── gpio/consumer.h ← GPIO consumer API
│   ├── regmap.h        ← Regmap framework
│   ├── pm_runtime.h    ← Runtime PM
│   ├── io.h            ← ioremap, readl/writel
│   ├── spinlock.h      ← Spinlocks
│   ├── mutex.h         ← Mutexes
│   └── slab.h          ← kmalloc/kfree
│
├── asm-generic/        ← Architecture-generic assembly
├── dt-bindings/        ← Device Tree binding constants
├── uapi/               ← User-space API headers
│   └── linux/          ← ioctl definitions visible to userspace
└── trace/              ← Tracepoint headers
```

---

## 29.4 Key Files Every Driver Developer Should Read

| File | Why |
|------|-----|
| `drivers/base/core.c` | Understand device lifecycle |
| `drivers/base/dd.c` | See how probe() is called |
| `drivers/base/platform.c` | Platform bus matching logic |
| `drivers/base/devres.c` | How devm_* cleanup works |
| `drivers/of/platform.c` | DT → platform_device creation |
| `kernel/irq/manage.c` | request_irq implementation |
| `kernel/module/main.c` | Module loading internals |
| `mm/slab_common.c` | kmalloc implementation |
| `fs/sysfs/file.c` | sysfs attribute read/write |
| `kernel/dma/mapping.c` | DMA mapping core |

---

## 29.5 Finding Code in the Source

```bash
# Find where a function is defined
grep -rn 'int platform_driver_register' drivers/base/

# Find all callers of an API
grep -rn 'devm_platform_ioremap_resource' drivers/ | head -20

# Find DT compatible strings for a SoC
grep -rn '"qcom,sa8155p' drivers/

# Find all drivers for a bus type
grep -rn 'module_i2c_driver' drivers/ | wc -l

# Use cscope or ctags for navigation
make cscope    # Generate cscope database
cscope -d      # Browse

# Elixir (online): https://elixir.bootlin.com/linux/latest/source
```

---

## 29.6 Documentation Locations

```
Documentation/
├── driver-api/         ← Driver API documentation
│   ├── driver-model/   ← Bus, device, driver model
│   ├── dma-api.txt     ← DMA API guide
│   └── regmap.rst      ← Regmap framework
├── devicetree/
│   └── bindings/       ← DT binding documents (YAML)
├── admin-guide/
│   └── kernel-parameters.txt
├── process/
│   └── coding-style.rst ← Kernel coding style
└── trace/
    └── ftrace.rst       ← Ftrace documentation
```

---

## Interview Questions

**Q1: Where would you look to understand how `probe()` is called?**
A: `drivers/base/dd.c` — specifically `really_probe()`. This function calls the bus-specific or driver-specific probe after matching. Follow from `driver_probe_device()` → `really_probe()`.

**Q2: How do you find all drivers for a specific SoC vendor?**
A: `grep -rn 'compatible.*"vendor,' drivers/` to find all DT compatible strings for that vendor. Also check `arch/arm64/boot/dts/vendor/` for DT files and `drivers/soc/vendor/` for SoC-specific code.

**Q3: What is the `include/uapi/` directory for?**
A: Headers that define the **user-space API** — ioctl command numbers, structures shared between kernel and userspace, system call numbers. These are part of the stable ABI and must not change in backwards-incompatible ways.

---

*Next: [Chapter 30 — Flow Diagrams](Chapter_30_Flow_Diagrams.md)*
