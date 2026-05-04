# Chapter 35: References and Documentation

## Chapter Overview

This chapter consolidates the most important references, books, websites, kernel documentation, and learning resources for Linux device driver development.

---

## 35.1 Essential Books

| Book | Author(s) | Coverage |
|------|----------|----------|
| **Linux Device Drivers, 3rd Edition** | Corbet, Rubini, Kroah-Hartman | Classic foundational text (Linux 2.6, concepts still valid) |
| **Linux Kernel Development, 3rd Edition** | Robert Love | Kernel internals (scheduler, mm, VFS) |
| **Essential Linux Device Drivers** | Sreekrishnan Venkateswaran | Comprehensive driver types coverage |
| **Understanding the Linux Kernel, 3rd Edition** | Bovet, Cesati | Deep kernel internals |
| **Linux Kernel Programming** | Kaiwan N. Billimoria | Modern (5.x), practical approach |
| **Linux Driver Development with Raspberry Pi** | Luca Salzano | Hands-on with real hardware |

---

## 35.2 Kernel Documentation (In-Tree)

Located in `Documentation/` within the kernel source tree.

### Driver Development

| Path | Content |
|------|---------|
| `Documentation/driver-api/` | Driver API guides |
| `Documentation/driver-api/driver-model/` | Bus, device, driver model |
| `Documentation/driver-api/dma-api.rst` | DMA API guide |
| `Documentation/driver-api/regmap.rst` | Regmap framework |
| `Documentation/driver-api/gpio/` | GPIO subsystem |
| `Documentation/driver-api/pin-control.rst` | Pinctrl subsystem |

### Device Tree

| Path | Content |
|------|---------|
| `Documentation/devicetree/bindings/` | DT binding documents (YAML) |
| `Documentation/devicetree/usage-model.rst` | DT usage in Linux |
| `Documentation/devicetree/of_unittest.rst` | DT unit tests |

### Kernel Infrastructure

| Path | Content |
|------|---------|
| `Documentation/locking/` | Locking guide, lockdep |
| `Documentation/memory-barriers.txt` | Memory ordering |
| `Documentation/trace/ftrace.rst` | Ftrace guide |
| `Documentation/process/coding-style.rst` | Coding style rules |
| `Documentation/process/submitting-patches.rst` | How to submit patches |

### Building In-Tree Docs

```bash
# Build HTML documentation
make htmldocs
# Output in Documentation/output/

# Build specific section
make SPHINXDIRS="driver-api" htmldocs
```

---

## 35.3 Online Resources

### Official

| Resource | URL | Content |
|----------|-----|---------|
| **Kernel.org** | kernel.org | Source, releases |
| **Elixir (Bootlin)** | elixir.bootlin.com | Online source browser |
| **LWN.net** | lwn.net | Kernel development news |
| **Kernelnewbies** | kernelnewbies.org | Beginner guides, changelog |
| **KernelDoc** | docs.kernel.org | Built documentation |

### Learning

| Resource | URL | Content |
|----------|-----|---------|
| **Bootlin Training** | bootlin.com/training | Free slides: kernel, drivers, DT |
| **The Linux Foundation Training** | training.linuxfoundation.org | LFD401, LFD420 courses |
| **eLinux.org** | elinux.org | Embedded Linux wiki |
| **KUnit Docs** | docs.kernel.org/dev-tools/kunit | Unit testing guide |

### Mailing Lists

| List | Purpose |
|------|---------|
| linux-kernel@vger.kernel.org | Main kernel development |
| linux-driver-devel@vger.kernel.org | Driver development |
| devicetree@vger.kernel.org | Device Tree discussions |
| linux-arm-kernel@lists.infradead.org | ARM kernel |
| linux-pci@vger.kernel.org | PCI subsystem |

---

## 35.4 Key Kernel Source Files to Study

### For Beginners (Start Here)

```
1. drivers/misc/dummy-irq.c           ← Simplest IRQ driver
2. drivers/misc/dummy_dma.c           ← Simple DMA example
3. drivers/leds/leds-gpio.c           ← Clean GPIO-based driver
4. drivers/hwmon/tmp102.c             ← Clean I2C sensor driver
5. drivers/input/keyboard/gpio_keys.c ← Well-documented input driver
```

### For Intermediate

```
1. drivers/tty/serial/8250/           ← Classic UART driver
2. drivers/i2c/busses/i2c-designware-*.c ← I2C controller
3. drivers/spi/spi-pl022.c            ← SPI controller (ARM)
4. drivers/gpio/gpio-pl061.c          ← GPIO controller
5. drivers/watchdog/sp805_wdt.c       ← Watchdog driver
```

### For Advanced

```
1. drivers/net/ethernet/intel/e1000e/  ← Production Ethernet driver
2. drivers/gpu/drm/msm/               ← Qualcomm display driver
3. drivers/media/platform/            ← V4L2 camera drivers
4. drivers/iommu/arm/arm-smmu-v3.c    ← ARM SMMU
5. drivers/nvme/host/core.c           ← NVMe storage driver
```

---

## 35.5 Development Tools

| Tool | Purpose | Usage |
|------|---------|-------|
| **gcc / clang** | Kernel compilation | `make CC=clang` |
| **sparse** | Static analysis | `make C=1` |
| **coccinelle** | Semantic patching | `make coccicheck` |
| **checkpatch.pl** | Style checking | `./scripts/checkpatch.pl -f file.c` |
| **perf** | Performance profiling | `perf top -g -K` |
| **ftrace** | Function tracing | `/sys/kernel/debug/tracing/` |
| **QEMU** | Hardware emulation | `qemu-system-aarch64` |
| **GDB + KGDB** | Kernel debugging | Over serial or QEMU `-s` |
| **addr2line** | Oops decoding | `addr2line -e vmlinux` |
| **devmem2** | Direct memory access | `devmem2 0x1e000000 w` |
| **dtc** | DT compiler | `dtc -I dts -O dtb` |

---

## 35.6 Kernel Configuration for Driver Development

```bash
# Essential debug configs for development
CONFIG_DYNAMIC_DEBUG=y          # Dynamic debug messages
CONFIG_DEBUG_INFO=y             # Debug symbols
CONFIG_DEBUG_FS=y               # debugfs support
CONFIG_KASAN=y                  # Address sanitizer
CONFIG_LOCKDEP=y                # Lock dependency checking
CONFIG_PROVE_LOCKING=y          # Deadlock detection
CONFIG_DEBUG_ATOMIC_SLEEP=y     # Detect sleeping in atomic context
CONFIG_FAULT_INJECTION=y        # Fault injection support
CONFIG_REGMAP_DEBUGFS=y         # Regmap debug in debugfs
CONFIG_DMA_API_DEBUG=y          # DMA API usage debugging
CONFIG_PM_DEBUG=y               # Power management debug
CONFIG_PM_ADVANCED_DEBUG=y      # Detailed PM debug
```

---

## 35.7 Standards and Specifications

| Standard | Relevance |
|----------|-----------|
| **PCI Express Spec** | PCI driver development |
| **USB Spec** | USB driver development |
| **I2C Spec (NXP)** | I2C driver timing/protocol |
| **SPI (Motorola)** | SPI driver modes |
| **AMBA/AXI (ARM)** | SoC bus architecture |
| **GIC Spec (ARM)** | Interrupt controller |
| **SMMU Spec (ARM)** | IOMMU for ARM |
| **Devicetree Spec** | DT format and bindings |
| **ACPI Spec** | x86 device discovery |
| **ISO 26262** | Automotive functional safety |

---

*Next: [Chapter 36 — Interview Preparation](Chapter_36_Interview_Preparation.md)*
