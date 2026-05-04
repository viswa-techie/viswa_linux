# Chapter 24: Kernel Driver APIs — Consolidated Reference

## Chapter Overview

This chapter consolidates the most important kernel APIs used in driver development into categorized tables with signatures, descriptions, and usage patterns.

---

## 24.1 Driver Registration APIs

### Platform Driver

| API | Description |
|-----|-------------|
| `platform_driver_register(drv)` | Register platform driver |
| `platform_driver_unregister(drv)` | Unregister platform driver |
| `module_platform_driver(drv)` | Helper: auto init/exit |
| `builtin_platform_driver(drv)` | For built-in (non-module) drivers |

### PCI Driver

| API | Description |
|-----|-------------|
| `pci_register_driver(drv)` | Register PCI driver |
| `module_pci_driver(drv)` | Helper: auto init/exit |
| `pci_enable_device(pdev)` | Enable PCI device (must call before access) |
| `pci_disable_device(pdev)` | Disable PCI device |
| `pci_set_master(pdev)` | Enable bus mastering (for DMA) |
| `pci_request_regions(pdev, name)` | Claim all BARs |

### I2C Driver

| API | Description |
|-----|-------------|
| `i2c_add_driver(drv)` | Register I2C driver |
| `module_i2c_driver(drv)` | Helper: auto init/exit |
| `i2c_smbus_read_byte_data(client, reg)` | SMBus read |
| `i2c_smbus_write_byte_data(client, reg, val)` | SMBus write |
| `i2c_transfer(adapter, msgs, num)` | Raw I2C transfer |

### SPI Driver

| API | Description |
|-----|-------------|
| `spi_register_driver(drv)` | Register SPI driver |
| `module_spi_driver(drv)` | Helper: auto init/exit |
| `spi_write(spi, buf, len)` | SPI write |
| `spi_read(spi, buf, len)` | SPI read |
| `spi_sync_transfer(spi, xfers, num)` | Full-duplex transfer |

### Character Device

| API | Description |
|-----|-------------|
| `alloc_chrdev_region(&dev, 0, count, name)` | Allocate major/minor dynamically |
| `cdev_init(cdev, fops)` | Initialize cdev |
| `cdev_add(cdev, devno, count)` | Add cdev to system |
| `cdev_del(cdev)` | Remove cdev |
| `class_create(name)` | Create device class |
| `device_create(class, parent, devt, data, fmt, ...)` | Create device node |

---

## 24.2 Device/Resource APIs

### Memory-Mapped I/O

| API | Description |
|-----|-------------|
| `devm_platform_ioremap_resource(pdev, idx)` | Map MMIO resource (recommended) |
| `devm_ioremap(dev, phys, size)` | Map physical to virtual |
| `readl(addr)` / `writel(val, addr)` | 32-bit MMIO read/write (with barrier) |
| `readl_relaxed(addr)` / `writel_relaxed(val, addr)` | Without barrier (ARM) |
| `readl_poll_timeout(addr, val, cond, delay, timeout)` | Poll register |
| `ioread32(addr)` / `iowrite32(val, addr)` | Portable (MMIO or PIO) |

### Regmap

| API | Description |
|-----|-------------|
| `devm_regmap_init_mmio(dev, base, cfg)` | MMIO regmap |
| `devm_regmap_init_i2c(client, cfg)` | I2C regmap |
| `devm_regmap_init_spi(spi, cfg)` | SPI regmap |
| `regmap_read(map, reg, &val)` | Read register |
| `regmap_write(map, reg, val)` | Write register |
| `regmap_update_bits(map, reg, mask, val)` | Read-modify-write |
| `regmap_bulk_read(map, reg, buf, count)` | Burst read |

### Clock

| API | Description |
|-----|-------------|
| `devm_clk_get(dev, con_id)` | Get clock reference |
| `clk_prepare_enable(clk)` | Enable clock |
| `clk_disable_unprepare(clk)` | Disable clock |
| `clk_get_rate(clk)` | Get frequency in Hz |
| `clk_set_rate(clk, rate)` | Set frequency |

### Reset

| API | Description |
|-----|-------------|
| `devm_reset_control_get_exclusive(dev, id)` | Get reset control |
| `reset_control_assert(rstc)` | Assert reset (active) |
| `reset_control_deassert(rstc)` | Deassert reset |
| `reset_control_reset(rstc)` | Pulse reset |

### GPIO

| API | Description |
|-----|-------------|
| `devm_gpiod_get(dev, con_id, flags)` | Get GPIO descriptor |
| `gpiod_direction_output(desc, val)` | Set as output |
| `gpiod_direction_input(desc)` | Set as input |
| `gpiod_set_value(desc, val)` | Set output value |
| `gpiod_get_value(desc)` | Read value |
| `gpiod_to_irq(desc)` | Convert GPIO to IRQ number |

---

## 24.3 Interrupt APIs

| API | Description |
|-----|-------------|
| `devm_request_irq(dev, irq, handler, flags, name, data)` | Request IRQ |
| `devm_request_threaded_irq(dev, irq, hard, thread, flags, name, data)` | Threaded IRQ |
| `platform_get_irq(pdev, idx)` | Get IRQ from platform resources |
| `platform_get_irq_byname(pdev, name)` | Get IRQ by name |
| `disable_irq(irq)` / `enable_irq(irq)` | Disable/enable specific IRQ |
| `irq_set_irq_wake(irq, on)` | Mark as wake source |

### IRQ Handler Return Values

```c
IRQ_NONE          /* Not our interrupt */
IRQ_HANDLED       /* Handled successfully */
IRQ_WAKE_THREAD   /* Wake threaded handler */
```

---

## 24.4 Memory Allocation APIs

| API | Flags | Description |
|-----|-------|-------------|
| `devm_kzalloc(dev, size, gfp)` | GFP_KERNEL | Managed, zeroed |
| `devm_kcalloc(dev, n, size, gfp)` | GFP_KERNEL | Managed, array |
| `kmalloc(size, gfp)` | Any | Physically contiguous |
| `kzalloc(size, gfp)` | Any | Zeroed kmalloc |
| `vmalloc(size)` | — | Virtually contiguous |
| `kvmalloc(size, gfp)` | GFP_KERNEL | Try kmalloc, fall back to vmalloc |
| `krealloc(ptr, size, gfp)` | Any | Resize allocation |

### GFP Flags Quick Reference

| Flag | Context | Description |
|------|---------|-------------|
| `GFP_KERNEL` | Process | Can sleep, most common |
| `GFP_ATOMIC` | IRQ/spinlock | Cannot sleep |
| `GFP_DMA` | DMA | From DMA zone (legacy ISA) |
| `GFP_NOWAIT` | Either | No waiting, no warning |

---

## 24.5 DMA APIs

| API | Description |
|-----|-------------|
| `dma_set_mask_and_coherent(dev, mask)` | Set DMA address mask |
| `dma_alloc_coherent(dev, size, &handle, gfp)` | Coherent (non-cacheable) buffer |
| `dma_free_coherent(dev, size, vaddr, handle)` | Free coherent buffer |
| `dma_map_single(dev, ptr, size, dir)` | Map single buffer (streaming) |
| `dma_unmap_single(dev, handle, size, dir)` | Unmap single buffer |
| `dma_map_sg(dev, sg, nents, dir)` | Map scatter-gather list |
| `dma_unmap_sg(dev, sg, nents, dir)` | Unmap scatter-gather list |
| `dma_sync_single_for_cpu(dev, handle, size, dir)` | Sync: device → CPU |
| `dma_sync_single_for_device(dev, handle, size, dir)` | Sync: CPU → device |

### DMA Direction Constants

```c
DMA_TO_DEVICE       /* CPU → Device (TX) */
DMA_FROM_DEVICE     /* Device → CPU (RX) */
DMA_BIDIRECTIONAL   /* Both directions */
DMA_NONE            /* Not used for DMA */
```

---

## 24.6 Power Management APIs

| API | Description |
|-----|-------------|
| `pm_runtime_enable(dev)` | Enable runtime PM |
| `pm_runtime_disable(dev)` | Disable runtime PM |
| `pm_runtime_get_sync(dev)` | Increment usage, resume if suspended |
| `pm_runtime_put(dev)` | Decrement usage, maybe suspend |
| `pm_runtime_put_autosuspend(dev)` | Autosuspend after delay |
| `pm_runtime_set_autosuspend_delay(dev, ms)` | Set autosuspend delay |
| `pm_runtime_use_autosuspend(dev)` | Enable autosuspend |

---

## 24.7 Synchronization APIs

| API | Description |
|-----|-------------|
| `spin_lock_init(lock)` | Initialize spinlock |
| `spin_lock(lock)` / `spin_unlock(lock)` | Lock/unlock (process context) |
| `spin_lock_irqsave(lock, flags)` | Lock + save IRQ state |
| `spin_unlock_irqrestore(lock, flags)` | Unlock + restore IRQ state |
| `mutex_init(mutex)` | Initialize mutex |
| `mutex_lock(mutex)` / `mutex_unlock(mutex)` | Lock/unlock (sleepable) |
| `init_completion(comp)` | Initialize completion |
| `wait_for_completion_timeout(comp, jiffies)` | Wait with timeout |
| `complete(comp)` | Signal completion |

---

## 24.8 Device Tree Property APIs

| API | Description |
|-----|-------------|
| `of_property_read_u32(np, name, &val)` | Read u32 property |
| `of_property_read_string(np, name, &str)` | Read string property |
| `of_property_read_bool(np, name)` | Check boolean property |
| `of_get_child_count(np)` | Count child nodes |
| `for_each_child_of_node(np, child)` | Iterate children |
| `device_property_read_u32(dev, name, &val)` | Firmware-agnostic (DT/ACPI) |

---

## 24.9 Logging APIs

| API | Description |
|-----|-------------|
| `dev_err(dev, fmt, ...)` | Error message with device prefix |
| `dev_warn(dev, fmt, ...)` | Warning |
| `dev_info(dev, fmt, ...)` | Informational |
| `dev_dbg(dev, fmt, ...)` | Debug (requires CONFIG_DYNAMIC_DEBUG) |
| `dev_err_probe(dev, err, fmt, ...)` | Error + handle -EPROBE_DEFER |
| `pr_err(fmt, ...)` | Error without device prefix |

---

## Interview Questions

**Q1: What is the difference between `devm_request_irq()` and `request_irq()`?**
A: `devm_request_irq()` is device-managed — the IRQ is automatically freed when the driver is unbound. `request_irq()` requires explicit `free_irq()` in remove. Always prefer `devm_*` variants.

**Q2: When should you use `readl_relaxed()` vs `readl()`?**
A: `readl()` includes a memory barrier, ensuring all previous writes are visible before the read. `readl_relaxed()` has no barrier — use it only when ordering doesn't matter (e.g., reading a status register that doesn't depend on prior writes). On x86, they're identical due to strong memory ordering.

**Q3: Why call `dma_set_mask_and_coherent()` before DMA operations?**
A: It tells the kernel the device's DMA address width. A device with 32-bit DMA on a 64-bit system needs `DMA_BIT_MASK(32)` to ensure buffers are allocated in the lower 4GB. Without this, DMA may fail silently or hit SWIOTLB.

---

*Next: [Chapter 25 — Debugging Techniques](Chapter_25_Debugging.md)*
