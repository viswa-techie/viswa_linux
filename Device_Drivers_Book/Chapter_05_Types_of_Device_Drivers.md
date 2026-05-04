# Chapter 5: Types of Device Drivers

## Chapter Overview

Linux categorizes drivers by the type of hardware they manage and how they interface with the kernel. This chapter covers every major driver category, their architecture, and when to use each.

---

## 5.1 Character Device Drivers

The most fundamental and common driver type. Provides byte-stream access via `/dev` nodes.

```
User: open("/dev/mydev", O_RDWR)
       │
       ▼
  VFS → cdev lookup → file_operations → driver functions
```

**Characteristics**:
- Sequential access (though seeking possible)
- No kernel buffering layer (driver manages buffers)
- Direct read()/write()/ioctl() mapping
- Examples: UART, GPIO, sensors, /dev/null, /dev/random, cameras

```c
static const struct file_operations my_fops = {
    .owner   = THIS_MODULE,
    .open    = my_open,
    .read    = my_read,
    .write   = my_write,
    .unlocked_ioctl = my_ioctl,
    .release = my_release,
};
```

---

## 5.2 Block Device Drivers

For random-access storage devices.

```
User: read(fd_sda, buf, 4096)
       │
       ▼
 VFS → filesystem → page cache → block layer → request queue → driver
```

**Characteristics**:
- Fixed-size block transfers (typically 512B or 4KB)
- Request queue with I/O scheduling
- Page cache integration
- Supports filesystems (ext4, f2fs, btrfs)
- Examples: HDD, SSD, eMMC, NVMe, SD card, ramdisk

```c
static const struct block_device_operations my_bdev_ops = {
    .owner   = THIS_MODULE,
    .open    = my_disk_open,
    .release = my_disk_release,
    .ioctl   = my_disk_ioctl,
};
```

### Char vs Block Comparison

| Aspect | Character | Block |
|--------|-----------|-------|
| Access pattern | Sequential/byte-stream | Random/block-aligned |
| Buffering | No kernel buffering | Page cache + I/O scheduler |
| Request queue | None | `struct request_queue` |
| Device node | /dev/ttyS0, /dev/video0 | /dev/sda, /dev/mmcblk0 |
| Major/minor | `alloc_chrdev_region()` | `register_blkdev()` |
| Use case | Sensors, terminals, control | Storage devices |

---

## 5.3 Network Device Drivers

Network drivers don't use `/dev` — they work through the socket interface.

```
User: send(sockfd, data, len, 0)
       │
       ▼
  Socket layer → protocol (TCP/IP) → net_device_ops → driver → NIC hardware
```

**Characteristics**:
- No device file — accessed via sockets
- Packet-oriented (sk_buff structures)
- NAPI for high-performance polling
- Transmit queue + receive path
- Examples: Ethernet, Wi-Fi, CAN, Bluetooth

```c
static const struct net_device_ops my_netdev_ops = {
    .ndo_open       = my_net_open,
    .ndo_stop       = my_net_stop,
    .ndo_start_xmit = my_net_xmit,
    .ndo_set_mac_address = my_set_mac,
};
```

---

## 5.4 Platform Device Drivers

For non-enumerable devices on SoCs — the most common driver type in embedded Linux.

```
Device Tree:
  my-device@10000000 {
      compatible = "vendor,my-device";
      reg = <0x10000000 0x1000>;
      interrupts = <GIC_SPI 42 IRQ_TYPE_LEVEL_HIGH>;
  };
       │
       ▼
  Platform bus matches "compatible" string
       │
       ▼
  driver->probe() called
```

**Characteristics**:
- Device described in Device Tree or ACPI (not self-enumerating)
- Uses `platform_device` and `platform_driver`
- Resources (MMIO, IRQ, clocks) from DT or platform data
- Most SoC peripherals: UART, I2C/SPI controllers, timers, DMA engines

```c
static const struct of_device_id my_dt_ids[] = {
    { .compatible = "vendor,my-device" },
    { }
};
MODULE_DEVICE_TABLE(of, my_dt_ids);

static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "my-device",
        .of_match_table = my_dt_ids,
        .pm = &my_pm_ops,
    },
};
module_platform_driver(my_driver);
```

---

## 5.5 Virtual Device Drivers

Drivers for non-physical devices — purely software abstractions.

| Virtual Device | Purpose | Device Node |
|---------------|---------|-------------|
| `/dev/null` | Discard all data | Character |
| `/dev/zero` | Infinite zero bytes | Character |
| `/dev/random` | Random number generator | Character |
| `/dev/loop0` | Loopback block device | Block |
| `/dev/tun` | Virtual network tunnel | Character/Net |
| `/dev/vhost-net` | Virtio host networking | Misc |
| `/dev/fuse` | FUSE filesystem | Misc |
| `/dev/kvm` | KVM hypervisor | Misc |

---

## 5.6 USB Device Drivers

For devices on the Universal Serial Bus.

```
USB Device plugged in
       │
       ▼
  USB HCD (host controller driver) detects
       │
       ▼
  USB core enumerates (GET_DESCRIPTOR)
       ├── Vendor ID: 0x1234
       └── Product ID: 0x5678
       │
       ▼
  usb_match_id() finds matching driver
       │
       ▼
  driver->probe() called with usb_interface
```

```c
static const struct usb_device_id my_usb_ids[] = {
    { USB_DEVICE(0x1234, 0x5678) },   /* vendor, product */
    { USB_DEVICE_INTERFACE_CLASS(0x1234, 0x5678, USB_CLASS_HID) },
    { }
};
MODULE_DEVICE_TABLE(usb, my_usb_ids);

static struct usb_driver my_usb_driver = {
    .name       = "my-usb-driver",
    .id_table   = my_usb_ids,
    .probe      = my_usb_probe,
    .disconnect = my_usb_disconnect,
};
module_usb_driver(my_usb_driver);
```

---

## 5.7 PCI Device Drivers

For devices on the PCI/PCIe bus.

```c
static const struct pci_device_id my_pci_ids[] = {
    { PCI_DEVICE(PCI_VENDOR_ID_INTEL, 0x1234) },
    { PCI_DEVICE_CLASS(PCI_CLASS_NETWORK_ETHERNET << 8, 0xffff00) },
    { }
};
MODULE_DEVICE_TABLE(pci, my_pci_ids);

static struct pci_driver my_pci_driver = {
    .name     = "my-pci-driver",
    .id_table = my_pci_ids,
    .probe    = my_pci_probe,
    .remove   = my_pci_remove,
};
module_pci_driver(my_pci_driver);

static int my_pci_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
    int ret;
    ret = pcim_enable_device(pdev);    /* managed PCI enable */
    if (ret)
        return ret;

    pci_set_master(pdev);              /* enable bus mastering (DMA) */

    void __iomem *regs = pcim_iomap(pdev, 0, 0);  /* map BAR 0 */
    if (!regs)
        return -ENOMEM;

    /* Device is ready, continue setup... */
    return 0;
}
```

### PCI Enumeration

```
BIOS/FW scans PCI buses → assigns BARs, IRQs
       │
       ▼
  Linux PCI subsystem reads configuration space
       │
       ▼
  Creates pci_dev for each function
       │
       ▼
  Matches against pci_driver.id_table
       │
       ▼
  Calls probe() on match
```

---

## 5.8 I2C Device Drivers

Two-wire serial bus, common for sensors, EEPROMs, PMICs.

```c
static const struct of_device_id my_i2c_dt_ids[] = {
    { .compatible = "vendor,temp-sensor" },
    { }
};
MODULE_DEVICE_TABLE(of, my_i2c_dt_ids);

static const struct i2c_device_id my_i2c_ids[] = {
    { "temp-sensor", 0 },
    { }
};
MODULE_DEVICE_TABLE(i2c, my_i2c_ids);

static struct i2c_driver my_i2c_driver = {
    .driver = {
        .name = "temp-sensor",
        .of_match_table = my_i2c_dt_ids,
    },
    .probe  = my_i2c_probe,
    .remove = my_i2c_remove,
    .id_table = my_i2c_ids,
};
module_i2c_driver(my_i2c_driver);

static int my_i2c_probe(struct i2c_client *client)
{
    /* Read WHO_AM_I register */
    int val = i2c_smbus_read_byte_data(client, 0x0F);
    if (val < 0)
        return val;
    dev_info(&client->dev, "Chip ID: 0x%02x\n", val);
    return 0;
}
```

---

## 5.9 SPI Device Drivers

Four-wire serial bus, common for flash, displays, ADCs.

```c
static const struct of_device_id my_spi_dt_ids[] = {
    { .compatible = "vendor,my-flash" },
    { }
};
MODULE_DEVICE_TABLE(of, my_spi_dt_ids);

static struct spi_driver my_spi_driver = {
    .driver = {
        .name = "my-flash",
        .of_match_table = my_spi_dt_ids,
    },
    .probe  = my_spi_probe,
    .remove = my_spi_remove,
};
module_spi_driver(my_spi_driver);

static int my_spi_probe(struct spi_device *spi)
{
    u8 tx_buf[2] = { 0x9F, 0x00 };  /* JEDEC READ ID command */
    u8 rx_buf[4];
    struct spi_transfer xfer = {
        .tx_buf = tx_buf,
        .rx_buf = rx_buf,
        .len    = 4,
        .speed_hz = 1000000,
    };
    spi_sync_transfer(spi, &xfer, 1);
    dev_info(&spi->dev, "Flash ID: %02x %02x %02x\n",
             rx_buf[1], rx_buf[2], rx_buf[3]);
    return 0;
}
```

---

## Complete Driver Type Comparison

| Type | Bus | Discovery | Device Node | Key Structure | Registration |
|------|-----|-----------|-------------|---------------|-------------|
| Character | Any | Manual | /dev/xxx | `file_operations` | `cdev_add()` |
| Block | Any | Manual | /dev/sdX | `block_device_operations` | `add_disk()` |
| Network | Any | Manual | None | `net_device_ops` | `register_netdev()` |
| Platform | Platform | DT/ACPI | Varies | `platform_driver` | `platform_driver_register()` |
| USB | USB | Enumerable | Varies | `usb_driver` | `usb_register()` |
| PCI | PCI/PCIe | Enumerable | Varies | `pci_driver` | `pci_register_driver()` |
| I2C | I2C | DT/board | Varies | `i2c_driver` | `i2c_add_driver()` |
| SPI | SPI | DT/board | Varies | `spi_driver` | `spi_register_driver()` |

---

## OS Comparison

| Aspect | Linux | Windows | macOS | QNX |
|--------|-------|---------|-------|-----|
| Char device | `file_operations` | IRP dispatch table | IOUserClient | Resource manager |
| Block device | `blk_mq_ops` | Storage miniport | IOBlockStorageDevice | Block device resource |
| Network | NAPI + `net_device_ops` | NDIS miniport | IONetworkController | io-pkt + dlls |
| Bus system | bus_type abstraction | Bus drivers + filter | IOService matching | HW-specific |
| USB | `usb_driver` | USBD/WDF USB | IOUSBHostDevice | USB DDK |

---

## Interview Questions

**Q1: What is the difference between a platform driver and a PCI driver?**
A: Platform drivers handle non-enumerable devices (SoC peripherals described in DT/ACPI). PCI drivers handle self-enumerating devices that announce their identity via PCI configuration space (vendor/device ID). Platform uses compatible string matching; PCI uses vendor/device ID matching.

**Q2: Why do network drivers not use /dev nodes?**
A: Historical design — networks use the socket API (connect, send, recv), not file I/O (open, read, write). Packet multiplexing is handled by the protocol stack, not by a device file. Network interfaces are named (eth0, wlan0) but accessed via sockets.

**Q3: When would you use a character driver vs. misc device?**
A: Use a full char driver when you need a dedicated major number or complex functionality. Use misc device (`misc_register()`) for simple devices — it uses major 10 and auto-allocates a minor, simplifying registration.

**Q4: How does USB device enumeration work in Linux?**
A: USB HCD detects device attachment → USB core sends GET_DESCRIPTOR → reads vendor/product ID, class, etc. → creates usb_device and usb_interface structs → USB bus match() checks against usb_device_id tables of registered drivers → matching driver's probe() called.

**Q5: What module registration helper macros exist?**
A: `module_platform_driver()`, `module_i2c_driver()`, `module_spi_driver()`, `module_pci_driver()`, `module_usb_driver()`. Each generates `module_init()`/`module_exit()` that calls the respective bus's register/unregister. Reduces boilerplate.

---

## Summary

| Driver Type | Best For | Key Trait |
|-------------|---------|-----------|
| **Character** | Sensors, serial, control | Byte-stream, simple VFS mapping |
| **Block** | Storage (HDD, SSD, eMMC) | Block-aligned, request queue, page cache |
| **Network** | Ethernet, Wi-Fi, CAN | Socket interface, sk_buff, NAPI |
| **Platform** | SoC peripherals | DT-described, non-enumerable |
| **USB** | External peripherals | Hot-plug, self-enumerating |
| **PCI** | High-speed cards | BAR mapping, config space, MSI |
| **I2C** | Low-speed sensors/PMIC | Two-wire, simple register access |
| **SPI** | Flash, displays, ADC | Four-wire, high-speed transfers |

---

*Next: [Chapter 6 — Linux Kernel Modules](Chapter_06_Kernel_Modules.md)*
