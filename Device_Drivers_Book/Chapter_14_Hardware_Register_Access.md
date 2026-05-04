# Chapter 14: Hardware Register Access

## Chapter Overview

Device drivers communicate with hardware through register access. This chapter covers memory-mapped I/O (MMIO), port I/O, the readl/writel API, and safe register access patterns.

---

## 14.1 Memory-Mapped I/O (MMIO)

Most modern hardware uses MMIO — device registers appear at specific physical addresses in the memory map.

```
Physical Address Map (ARM64 SoC Example):
┌──────────────────┐ 0xFFFFFFFF
│  Peripheral MMIO │
│  0x40000000-...  │ ← GPIO, UART, I2C, SPI registers
├──────────────────┤
│  PCIe MMIO       │
│  0x30000000-...  │ ← PCI BAR space
├──────────────────┤
│                  │
│  DRAM            │
│  0x80000000-...  │ ← Physical RAM
├──────────────────┤
│  Boot ROM        │
│  0x00000000      │
└──────────────────┘
```

### MMIO Access Flow

```
Driver Code:        writel(0x01, priv->base + REG_CTRL)
       │
       ▼
CPU generates memory bus transaction to physical address
       │
       ▼
Bus interconnect routes to device (not DRAM controller)
       │
       ▼
Device register written with value 0x01
```

---

## 14.2 Port-Mapped I/O (PIO)

Legacy x86 mechanism using dedicated IN/OUT instructions. Rarely used in embedded systems.

```c
/* Port I/O (x86 only — ISA-era devices) */
#include <linux/ioport.h>
#include <asm/io.h>

outb(value, port);       /* Write byte to I/O port */
outw(value, port);       /* Write 16-bit */
outl(value, port);       /* Write 32-bit */
val = inb(port);         /* Read byte */
val = inw(port);         /* Read 16-bit */
val = inl(port);         /* Read 32-bit */

/* Example: legacy serial port */
#define COM1_PORT 0x3F8
outb(data_byte, COM1_PORT);  /* TX data register */
```

| Aspect | MMIO | Port I/O |
|--------|------|----------|
| Access | Load/store instructions | IN/OUT instructions |
| Address space | Shared with memory | Separate I/O space |
| Platforms | All (ARM, x86, MIPS) | x86 only |
| Modern use | Everything | Legacy ISA devices |
| Cacheability | Must be uncached | N/A (separate space) |

---

## 14.3 Accessing Hardware Registers

### Step 1: Request and Map the Region

```c
static int my_probe(struct platform_device *pdev)
{
    /* Method 1: Combined (preferred) */
    void __iomem *base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(base))
        return PTR_ERR(base);

    /* Method 2: Manual steps */
    struct resource *res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    if (!res)
        return -ENODEV;

    /* Request exclusive access to the region */
    if (!devm_request_mem_region(&pdev->dev, res->start,
                                 resource_size(res), "my-dev"))
        return -EBUSY;

    /* Map physical address to kernel virtual address */
    void __iomem *base = devm_ioremap(&pdev->dev, res->start,
                                       resource_size(res));
    if (!base)
        return -ENOMEM;
}
```

### Step 2: Read/Write Registers

```c
/* Read a 32-bit register */
u32 status = readl(priv->base + REG_STATUS);

/* Write a 32-bit register */
writel(0x01, priv->base + REG_CTRL);

/* Sized variants */
u8  val8  = readb(addr);     writeb(val, addr);    /* 8-bit  */
u16 val16 = readw(addr);     writew(val, addr);    /* 16-bit */
u32 val32 = readl(addr);     writel(val, addr);    /* 32-bit */
u64 val64 = readq(addr);     writeq(val, addr);    /* 64-bit */
```

---

## 14.4 Register Mapping Techniques

### ioremap Variants

| Function | Cache Mode | Use Case |
|----------|-----------|----------|
| `ioremap()` | Uncached (default for MMIO) | Standard device registers |
| `ioremap_wc()` | Write-combining | Framebuffers, large transfers |
| `ioremap_wt()` | Write-through | Some display devices |
| `ioremap_cache()` | Cacheable | RAM-like regions |

### regmap: Abstraction Layer for Register Access

```c
#include <linux/regmap.h>

/* Register map configuration */
static const struct regmap_config my_regmap_config = {
    .reg_bits = 32,              /* Register address width */
    .val_bits = 32,              /* Register value width */
    .reg_stride = 4,             /* Register spacing (bytes) */
    .max_register = 0xFF,        /* Last valid register */
    .cache_type = REGCACHE_FLAT, /* Enable register caching */
};

static int my_probe(struct platform_device *pdev)
{
    void __iomem *base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(base))
        return PTR_ERR(base);

    struct regmap *regmap = devm_regmap_init_mmio(&pdev->dev, base,
                                                   &my_regmap_config);
    if (IS_ERR(regmap))
        return PTR_ERR(regmap);

    /* Read register */
    unsigned int val;
    regmap_read(regmap, REG_STATUS, &val);

    /* Write register */
    regmap_write(regmap, REG_CTRL, 0x01);

    /* Read-modify-write (atomic bit manipulation) */
    regmap_update_bits(regmap, REG_CTRL,
                       BIT(3) | BIT(2),     /* mask */
                       BIT(3));              /* set bit 3, clear bit 2 */

    return 0;
}
```

### regmap Backends

| Backend | Function | Bus |
|---------|----------|-----|
| `devm_regmap_init_mmio()` | Memory-mapped I/O | Platform |
| `devm_regmap_init_i2c()` | I2C client | I2C |
| `devm_regmap_init_spi()` | SPI device | SPI |
| `devm_regmap_init_sdw()` | SoundWire | SoundWire |

---

## 14.5 readl() and writel() Operations

### Ordering and Barriers

```c
/* readl/writel include memory barriers:
 *   writel = __raw_writel + wmb()
 *   readl  = rmb() + __raw_readl
 * This ensures ordering relative to DMA and other devices.
 */

/* Relaxed variants (no barriers — faster but requires care) */
val = readl_relaxed(addr);
writel_relaxed(val, addr);

/* Example: when ordering matters */
writel(DMA_START, priv->base + REG_DMA_CTRL);  /* writel ensures
    the DMA buffer setup (previous writes) are visible to device
    before the DMA start command */

/* Example: when relaxed is OK */
for (i = 0; i < 256; i++)
    writel_relaxed(data[i], priv->base + REG_FIFO);  /* Bulk FIFO fill */
/* Final barrier after all writes */
writel(TX_START, priv->base + REG_CTRL);  /* Non-relaxed = barrier */
```

### Register Bit Manipulation Pattern

```c
/* Common patterns for register modification */

/* Set bits */
u32 val = readl(base + REG_CTRL);
val |= BIT(3) | BIT(5);
writel(val, base + REG_CTRL);

/* Clear bits */
val = readl(base + REG_CTRL);
val &= ~(BIT(3) | BIT(5));
writel(val, base + REG_CTRL);

/* Modify specific field */
#define REG_CTRL_SPEED_MASK   GENMASK(7, 4)   /* Bits 7:4 */
#define REG_CTRL_SPEED_SHIFT  4
val = readl(base + REG_CTRL);
val &= ~REG_CTRL_SPEED_MASK;
val |= (speed << REG_CTRL_SPEED_SHIFT) & REG_CTRL_SPEED_MASK;
writel(val, base + REG_CTRL);

/* Or use FIELD_PREP / FIELD_GET (preferred) */
#include <linux/bitfield.h>
#define REG_CTRL_SPEED   GENMASK(7, 4)

val = readl(base + REG_CTRL);
val &= ~REG_CTRL_SPEED;
val |= FIELD_PREP(REG_CTRL_SPEED, speed);
writel(val, base + REG_CTRL);

int current_speed = FIELD_GET(REG_CTRL_SPEED, readl(base + REG_CTRL));
```

### Polling a Status Register

```c
#include <linux/iopoll.h>

/* Poll until bit is set (timeout-safe) */
u32 status;
int ret = readl_poll_timeout(priv->base + REG_STATUS, status,
                             status & BIT(0),    /* condition */
                             10,                  /* delay_us between polls */
                             1000);               /* timeout_us */
if (ret) {
    dev_err(dev, "Timeout waiting for ready\n");
    return ret;
}

/* Atomic context variant (no sleeping) */
readl_poll_timeout_atomic(priv->base + REG_STATUS, status,
                          status & BIT(0), 1, 100);
```

---

## Register Access Diagram

```
Driver Code                   Hardware
───────────                   ────────
                              
writel(0x01, base + 0x04)     
       │                      
       ▼                      
  CPU store to virtual addr   
       │                      
       ▼                      
  MMU translates VA → PA      
  (page table: uncached)      
       │                      
       ▼                      
  Bus transaction on AXI/AHB  
       │                      
       ▼                      ┌──────────────┐
  Address decoder routes      │ Device       │
  to device                   │ Register 0x04│ ← receives 0x01
                              │              │
  readl(base + 0x08)          │ Register 0x08│ → returns value
       │                      └──────────────┘
       ▼                      
  CPU load from VA (uncached) 
  → bus read transaction      
  → device returns register   
     value to CPU             
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| `include/asm-generic/io.h` | `readl/writel` definitions |
| `include/linux/io.h` | `ioremap()`, `devm_ioremap()` |
| `include/linux/ioport.h` | `struct resource`, `request_mem_region` |
| `include/linux/regmap.h` | regmap framework |
| `drivers/base/regmap/` | regmap implementation |
| `include/linux/bitfield.h` | `FIELD_PREP`, `FIELD_GET` |
| `include/linux/iopoll.h` | `readl_poll_timeout` |

---

## Interview Questions

**Q1: Why must device registers be mapped as uncacheable?**
A: CPU caches assume memory doesn't change unless the CPU writes it. Device registers change asynchronously (hardware updates status bits). Cached reads would return stale data. `ioremap()` maps as uncacheable (Device-nGnRnE on ARM64, UC on x86).

**Q2: What's the difference between `readl()` and `readl_relaxed()`?**
A: `readl()` includes a memory barrier ensuring proper ordering with other memory operations and DMA. `readl_relaxed()` has no barrier — faster but doesn't guarantee ordering. Use `readl_relaxed()` for bulk register reads where ordering between them doesn't matter.

**Q3: What is regmap and when should you use it?**
A: regmap is a kernel framework abstracting register access across different buses (MMIO, I2C, SPI). Use it when: you have many registers, need register caching (avoid redundant I2C reads), need atomic read-modify-write (`regmap_update_bits`), or your code needs to work over multiple bus types.

**Q4: What is `readl_poll_timeout()` and why use it?**
A: A helper that polls a register until a condition becomes true or a timeout expires. Prevents infinite loops when hardware fails to respond. Returns 0 on success, -ETIMEDOUT on timeout. Always use timeouts when polling hardware.

---

*Next: [Chapter 15 — Interrupt Handling](Chapter_15_Interrupt_Handling.md)*
