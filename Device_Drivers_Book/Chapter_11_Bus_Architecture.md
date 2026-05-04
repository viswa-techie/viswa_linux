# Chapter 11: Bus Architecture in Linux

## Chapter Overview

Every device connects to the system through a bus. Linux abstracts all buses through `struct bus_type`, creating a uniform framework for device enumeration, driver matching, and power management.

---

## 11.1 Bus Concept in Hardware Systems

```
┌──────────────────────────────────────────────────────┐
│                      CPU                              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐             │
│  │ Core 0  │  │ Core 1  │  │ Core 2  │             │
│  └────┬────┘  └────┬────┘  └────┬────┘             │
│       └──────────┬──┴──────────┘                     │
│              System Bus / Interconnect                 │
│       ┌──────────┼──────────────┐                     │
│  ┌────▼────┐ ┌───▼───┐  ┌──────▼──────┐             │
│  │  DRAM   │ │ PCIe  │  │ Platform    │             │
│  │ Ctrl    │ │ Root  │  │ Bus (AHB/   │             │
│  │         │ │Complex│  │  APB/AXI)   │             │
│  └─────────┘ └───┬───┘  └──┬──┬──┬───┘             │
│                  │          │  │  │                   │
│            ┌─────┤     UART I2C SPI                  │
│            │     │     GPIO Timer                     │
│          GPU   NIC                                    │
│         (PCIe) (PCIe)                                │
└──────────────────────────────────────────────────────┘
```

### Bus Types in a Typical SoC

| Bus | Speed | Use Case | Discovery |
|-----|-------|----------|-----------|
| **Platform/AHB/APB** | SoC-speed | On-chip peripherals | Device Tree |
| **PCIe** | 16 GT/s per lane | GPUs, NICs, NVMe | Self-enumerating |
| **USB** | 480Mbps–20Gbps | Peripherals | Self-enumerating |
| **I2C** | 100–400 KHz | Sensors, PMIC | DT + address |
| **SPI** | 1–100 MHz | Flash, displays | DT + CS line |
| **SDIO** | Up to 208 MHz | Wi-Fi, BT | Card detection |
| **CAN** | 1 Mbps / 5 Mbps FD | Automotive | Static config |

---

## 11.2 Bus Subsystem in Linux Kernel

### struct bus_type

```c
struct bus_type {
    const char *name;
    const char *dev_name;

    /* Core operations */
    int (*match)(struct device *dev, struct device_driver *drv);
    int (*uevent)(const struct device *dev, struct kobj_uevent_env *env);
    int (*probe)(struct device *dev);
    void (*remove)(struct device *dev);
    void (*shutdown)(struct device *dev);

    /* Power management */
    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);
    const struct dev_pm_ops *pm;

    /* Driver core private */
    struct subsys_private *p;
};
```

### Registration

```c
/* Example: custom bus registration */
static int my_bus_match(struct device *dev, struct device_driver *drv)
{
    /* Match logic: compare dev name to driver name */
    return strcmp(dev_name(dev), drv->name) == 0;
}

static struct bus_type my_bus_type = {
    .name  = "mybus",
    .match = my_bus_match,
};

static int __init my_bus_init(void)
{
    return bus_register(&my_bus_type);
    /* Creates /sys/bus/mybus/ */
}
```

---

## 11.3 Device Enumeration

### Enumerable Buses (PCIe, USB)

```
Power On
   │
   ▼
BIOS/FW scans PCIe topology
   │
   ▼
Assigns BARs, IRQs
   │
   ▼
Linux PCI subsystem reads config space
   │
   ▼
Creates pci_dev for each function
   │
   ▼
Matches pci_device_id → calls probe()
```

### Non-Enumerable Buses (Platform, I2C, SPI)

```
Kernel boots
   │
   ▼
Device Tree parsed (of_platform_populate)
   │
   ▼
Creates platform_device / i2c_client / spi_device
   │
   ▼
Bus match (compatible string) → calls probe()
```

---

## 11.4 Bus-Driver-Device Relationship

```
┌──────────────────────────────────────────────────┐
│                    BUS                            │
│                (struct bus_type)                   │
│                                                   │
│    Device List          Driver List               │
│    ──────────          ────────────               │
│    ┌────────┐          ┌──────────┐              │
│    │ Dev A  │          │ Driver X │              │
│    │(no drv)│──match──→│ (probe A)│  ✓           │
│    └────────┘          └──────────┘              │
│    ┌────────┐          ┌──────────┐              │
│    │ Dev B  │          │ Driver Y │              │
│    │(drv=Y) │←──bound──│ (owns B) │  ✓           │
│    └────────┘          └──────────┘              │
│    ┌────────┐                                    │
│    │ Dev C  │ ← no matching driver yet           │
│    │(no drv)│                                    │
│    └────────┘                                    │
└──────────────────────────────────────────────────┘
```

### Binding Triggers

1. **New device added**: iterate all drivers on bus → try match
2. **New driver registered**: iterate all devices on bus → try match
3. **Manual bind**: `echo "device_name" > /sys/bus/XXX/drivers/YYY/bind`

---

## 11.5 Bus Driver Architecture by Type

| Bus | Match Key | Key Structures | Source |
|-----|----------|---------------|--------|
| Platform | DT compatible, name | `platform_device`, `platform_driver` | `drivers/base/platform.c` |
| PCI | Vendor/Device ID | `pci_dev`, `pci_driver` | `drivers/pci/` |
| USB | Vendor/Product/Class | `usb_device`, `usb_driver` | `drivers/usb/core/` |
| I2C | DT compatible, addr | `i2c_client`, `i2c_driver` | `drivers/i2c/` |
| SPI | DT compatible, CS | `spi_device`, `spi_driver` | `drivers/spi/` |

---

## Debugging

```bash
# List all registered buses
ls /sys/bus/
# i2c  pci  platform  sdio  spi  usb  ...

# List devices on a bus
ls /sys/bus/i2c/devices/
ls /sys/bus/platform/devices/

# List drivers on a bus
ls /sys/bus/platform/drivers/

# See device-driver binding
cat /sys/bus/platform/devices/2000000.uart/driver_override
readlink /sys/bus/platform/devices/2000000.uart/driver

# Force unbind/rebind
echo "2000000.uart" > /sys/bus/platform/drivers/my-uart/unbind
echo "2000000.uart" > /sys/bus/platform/drivers/my-uart/bind
```

---

## Interview Questions

**Q1: What is the difference between an enumerable and non-enumerable bus?**
A: Enumerable buses (PCI, USB) autodiscover devices — hardware announces its identity through config space or descriptors. Non-enumerable buses (platform, I2C, SPI) require external description (Device Tree or ACPI tables) since hardware cannot announce itself.

**Q2: How does the kernel decide which driver to bind to a device?**
A: The bus's match() function is called for every (device, driver) pair. It checks compatibility: DT compatible string (platform/I2C/SPI), vendor/device ID (PCI), vendor/product ID (USB). First match triggers probe().

**Q3: What happens if no driver matches a device?**
A: The device remains unbound (dev->driver = NULL). It's visible in sysfs but not functional. When a matching driver is later loaded (modprobe), the bus re-triggers matching automatically.

---

*Next: [Chapter 12 — Platform Device Drivers](Chapter_12_Platform_Device_Drivers.md)*
