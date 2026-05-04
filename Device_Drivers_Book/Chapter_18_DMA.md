# Chapter 18: Direct Memory Access (DMA)

## Chapter Overview

DMA allows hardware devices to transfer data directly to/from system memory without CPU involvement. This chapter covers DMA architecture, Linux DMA APIs, coherent vs streaming mappings, and scatter-gather DMA.

---

## 18.1 DMA Concept

```
WITHOUT DMA (PIO):                  WITH DMA:
CPU reads/writes every byte          CPU sets up transfer, device does rest

CPU ──read──→ Device reg             CPU: program DMA controller
CPU ←─data──  Device reg              │     (source, dest, length)
CPU ──store─→ Memory                   ▼
(repeat for each byte)              Device ──DMA──→ Memory
                                    Device ──IRQ──→ CPU: "transfer done"
CPU utilization: 100%               CPU utilization: ~0% during transfer
```

---

## 18.2 DMA Controller Architecture

```
┌──────────┐        ┌──────────────┐        ┌──────────┐
│   CPU    │        │  DMA Engine  │        │  Device  │
│          │───────→│  (programs)  │        │ (UART,   │
│          │        │              │        │  SPI,    │
└──────────┘        │   Channel 0  │←──────→│  etc.)   │
                    │   Channel 1  │        └──────────┘
                    │   ...        │
                    └──────┬───────┘
                           │ Bus Master
                           ▼
                    ┌──────────────┐
                    │ System Memory│
                    │  (DRAM)      │
                    └──────────────┘
```

### DMA Types

| Type | Description | Example |
|------|-------------|---------|
| **Third-party DMA** | Separate DMA controller moves data | SoC DMA engine for UART/SPI |
| **Bus mastering DMA** | Device itself is bus master | PCIe NIC, NVMe |
| **Scatter-gather DMA** | DMA from/to multiple non-contiguous buffers | All modern DMA |

---

## 18.3 DMA Transfer Mechanisms

### Coherent (Consistent) DMA

```c
/* Allocate DMA-coherent buffer:
 * - CPU and device see same data (no cache sync needed)
 * - Physically contiguous
 * - Lifetime: usually entire driver lifetime
 */
dma_addr_t dma_handle;
void *cpu_addr = dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);
if (!cpu_addr)
    return -ENOMEM;

/* cpu_addr: CPU virtual address for driver to access */
/* dma_handle: bus/DMA address for device to access */

/* Free */
dma_free_coherent(dev, size, cpu_addr, dma_handle);
```

### Streaming DMA

```c
/* Map existing kernel buffer for DMA:
 * - Must sync cache manually
 * - For per-transfer buffers
 */
dma_addr_t dma_addr = dma_map_single(dev, buf, len, DMA_TO_DEVICE);
if (dma_mapping_error(dev, dma_addr))
    return -EIO;

/* Program device with dma_addr, start transfer */
/* ... wait for completion ... */

/* Unmap after transfer complete */
dma_unmap_single(dev, dma_addr, len, DMA_TO_DEVICE);
```

---

## 18.4 DMA APIs in Linux

### DMA Direction

| Direction | Meaning |
|-----------|---------|
| `DMA_TO_DEVICE` | CPU → Device (TX) |
| `DMA_FROM_DEVICE` | Device → CPU (RX) |
| `DMA_BIDIRECTIONAL` | Both directions |
| `DMA_NONE` | For debugging only |

---

## 18.5 dma_map_single()

```c
/* Complete example: TX transfer */
static int my_dma_tx(struct my_priv *priv, void *data, size_t len)
{
    dma_addr_t dma_addr;

    dma_addr = dma_map_single(priv->dev, data, len, DMA_TO_DEVICE);
    if (dma_mapping_error(priv->dev, dma_addr))
        return -EIO;

    /* Write DMA address to device register */
    writel(lower_32_bits(dma_addr), priv->base + REG_DMA_ADDR_LO);
    writel(upper_32_bits(dma_addr), priv->base + REG_DMA_ADDR_HI);
    writel(len, priv->base + REG_DMA_LEN);
    writel(DMA_START, priv->base + REG_DMA_CTRL);

    /* Wait for completion (IRQ or polling) */
    wait_for_completion_timeout(&priv->dma_done, HZ);

    /* Unmap */
    dma_unmap_single(priv->dev, dma_addr, len, DMA_TO_DEVICE);
    return 0;
}

/* Cache sync for streaming mappings */
/* If reusing a mapped buffer: */
dma_sync_single_for_cpu(dev, dma_addr, len, DMA_FROM_DEVICE);
/* ... read data with CPU ... */
dma_sync_single_for_device(dev, dma_addr, len, DMA_FROM_DEVICE);
/* ... device can DMA again ... */
```

---

## 18.6 dma_alloc_coherent()

```c
/* Best for long-lived shared buffers like DMA descriptor rings */
struct my_ring {
    struct my_desc *descs;     /* CPU virtual address */
    dma_addr_t descs_dma;      /* DMA bus address */
    int num_descs;
};

static int my_alloc_ring(struct device *dev, struct my_ring *ring, int n)
{
    ring->num_descs = n;
    ring->descs = dma_alloc_coherent(dev,
        n * sizeof(struct my_desc),
        &ring->descs_dma,
        GFP_KERNEL);
    if (!ring->descs)
        return -ENOMEM;

    /* No cache sync needed — hardware coherent */
    /* Program device with ring->descs_dma */
    return 0;
}
```

### Coherent vs Streaming Comparison

| Feature | Coherent | Streaming |
|---------|----------|-----------|
| API | `dma_alloc_coherent()` | `dma_map_single()` |
| Memory | Allocates new buffer | Maps existing buffer |
| Cache sync | Automatic (uncached) | Manual (`dma_sync_*`) |
| Lifetime | Long (ring buffers) | Short (per-transfer) |
| Performance | Slower access (uncached) | Faster CPU access (cached) |
| Common use | DMA descriptor rings | Data buffers |

---

## 18.7 Scatter-Gather DMA

```c
#include <linux/scatterlist.h>
#include <linux/dma-mapping.h>

/* Map scatter-gather list for DMA */
struct scatterlist sg[MAX_SG];
int nents = /* number of entries */;

sg_init_table(sg, nents);
for (i = 0; i < nents; i++)
    sg_set_buf(&sg[i], buffers[i], lengths[i]);

int mapped = dma_map_sg(dev, sg, nents, DMA_TO_DEVICE);
if (mapped == 0)
    return -EIO;

/* Program device with each segment */
struct scatterlist *s;
for_each_sg(sg, s, mapped, i) {
    dma_addr_t addr = sg_dma_address(s);
    unsigned int len = sg_dma_len(s);
    /* Write to device descriptor ring */
}

/* After transfer: */
dma_unmap_sg(dev, sg, nents, DMA_TO_DEVICE);
```

---

## DMA Flow Diagram

```
Driver probe():
  1. dma_alloc_coherent() → descriptor ring
  2. Allocate data buffers

Driver TX path:
  3. dma_map_single(data, DMA_TO_DEVICE) → get DMA addr
  4. Fill descriptor: DMA addr + length
  5. writel(doorbell) → start device DMA
  6. [Device reads data from memory via DMA]
  7. Device raises IRQ → "TX complete"
  8. dma_unmap_single()

Driver RX path:
  9.  Pre-allocate RX buffers
  10. dma_map_single(rx_buf, DMA_FROM_DEVICE)
  11. Fill RX descriptor with DMA addr
  12. [Device writes received data via DMA]
  13. Device raises IRQ → "RX complete"
  14. dma_sync_single_for_cpu() or dma_unmap_single()
  15. Process data
```

---

## DMA Mask and Addressing

```c
/* Set DMA address width capability */
ret = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(32));
if (ret) {
    dev_err(dev, "32-bit DMA not supported\n");
    return ret;
}

/* For 64-bit DMA capable devices: */
ret = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64));
```

---

## Bounce Buffers (SWIOTLB)

```
When device can only address low memory (e.g., 32-bit DMA)
but buffer is in high memory:

  High Memory Buffer (inaccessible to device)
         │
         │  memcpy() (kernel does this transparently)
         ▼
  Bounce Buffer (in low memory, DMA-able)
         │
         │  DMA transfer
         ▼
  Device

Handled automatically by the DMA API when dma_map_*()
detects the buffer is outside the device's DMA mask.
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| `include/linux/dma-mapping.h` | DMA API |
| `kernel/dma/mapping.c` | DMA core implementation |
| `kernel/dma/swiotlb.c` | Software bounce buffers |
| `kernel/dma/direct.c` | Direct DMA mapping |
| `drivers/dma/` | DMA engine drivers |

---

## Interview Questions

**Q1: What is the difference between `dma_alloc_coherent()` and `dma_map_single()`?**
A: `dma_alloc_coherent()` allocates NEW physically contiguous, cache-coherent memory (no manual sync needed). `dma_map_single()` maps EXISTING kernel memory for DMA (requires `dma_sync_*` for cache coherency). Use coherent for long-lived descriptor rings, streaming for per-transfer data buffers.

**Q2: What is `dma_mapping_error()` and why must you check it?**
A: After `dma_map_single()`, the returned DMA address might be invalid (IOMMU out of space, allocation failure). `dma_mapping_error()` returns true if the mapping failed. Skipping this check leads to DMA to address 0 — data corruption or system crash.

**Q3: What is SWIOTLB?**
A: Software I/O TLB (bounce buffer). When a buffer is at a physical address outside the device's DMA-addressable range, the DMA API transparently allocates a low-memory bounce buffer, copies data, and gives the bounce buffer address to the device.

**Q4: What is scatter-gather DMA?**
A: DMA from/to multiple non-contiguous memory segments in a single transfer. The device reads a list of (address, length) pairs (SG list) from a descriptor ring. Avoids the need for one large contiguous buffer.

---

*Next: [Chapter 19 — IOMMU and Device Address Translation](Chapter_19_IOMMU.md)*
