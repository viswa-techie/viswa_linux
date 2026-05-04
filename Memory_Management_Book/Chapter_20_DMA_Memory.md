# Chapter 20: Memory Management for Device Drivers

## Chapter Overview

Device drivers must manage memory for DMA transfers, MMIO regions, and kernel buffers. This chapter covers the Linux DMA API in depth, coherent vs streaming mappings, bounce buffers, and best practices for driver memory management.

---

## 20.1 DMA Allocation APIs

```c
#include <linux/dma-mapping.h>

/*
 * COHERENT DMA ALLOCATION
 * -----------------------
 * Allocates physically contiguous memory accessible by both CPU and device
 * with hardware-maintained coherency (no manual cache operations needed).
 * Expensive: uses CMA or low-memory zones.
 */
void *dma_alloc_coherent(struct device *dev, size_t size,
                         dma_addr_t *dma_handle, gfp_t gfp);
void dma_free_coherent(struct device *dev, size_t size,
                       void *cpu_addr, dma_addr_t dma_handle);

/* Example: Allocate a 4KB coherent DMA buffer */
dma_addr_t dma_addr;
void *buf = dma_alloc_coherent(dev, 4096, &dma_addr, GFP_KERNEL);
if (!buf)
    return -ENOMEM;
/* CPU uses 'buf' pointer, device uses 'dma_addr' */
/* Both see same data without manual cache flushing */

/*
 * STREAMING DMA MAPPING  
 * ---------------------
 * Maps existing kernel memory for DMA. CPU must sync cache manually.
 * Cheaper: no allocation, works with any kernel memory.
 */
dma_addr_t dma_map_single(struct device *dev, void *cpu_addr,
                          size_t size, enum dma_data_direction dir);
void dma_unmap_single(struct device *dev, dma_addr_t dma_addr,
                      size_t size, enum dma_data_direction dir);

/* Directions:
 * DMA_TO_DEVICE     — CPU writes, device reads  (e.g., TX)
 * DMA_FROM_DEVICE   — Device writes, CPU reads  (e.g., RX)
 * DMA_BIDIRECTIONAL — Both read and write
 * DMA_NONE          — Debug/error checking only
 */
```

### Coherent vs Streaming Comparison

```
┌────────────────────┬──────────────────────┬──────────────────────┐
│ Feature            │ Coherent             │ Streaming            │
├────────────────────┼──────────────────────┼──────────────────────┤
│ Allocation         │ dma_alloc_coherent() │ kmalloc() + map      │
│ Cache coherency    │ Hardware-managed      │ Manual sync required │
│ Lifetime           │ Long-lived            │ Per-transfer         │
│ CPU overhead       │ None (HW coherent)   │ Cache flush/inval    │
│ Memory source      │ CMA / low zones       │ Any kernel memory    │
│ Contiguity         │ Physically contiguous │ Not required (SG ok) │
│ Typical use        │ Descriptor rings      │ Data buffers         │
│ Cost               │ Expensive (alloc)     │ Cheap (map only)     │
│ ARM cache          │ Non-cacheable mapping │ Cache ops per xfer   │
│ x86 cache          │ Write-combining/UC    │ Normal WB + flush    │
└────────────────────┴──────────────────────┴──────────────────────┘
```

---

## 20.2 Scatter-Gather DMA

```c
/*
 * Scatter-gather: Map multiple non-contiguous buffers for device DMA.
 * IOMMU makes them appear contiguous to device.
 * Without IOMMU: each segment is a separate DMA transfer.
 */

#include <linux/scatterlist.h>

struct scatterlist sg[MAX_SEGS];
sg_init_table(sg, nents);

/* Add pages to scatter list */
for (i = 0; i < nents; i++) {
    sg_set_page(&sg[i], pages[i], PAGE_SIZE, 0);
}

/* Map entire scatter-gather list for device */
int mapped = dma_map_sg(dev, sg, nents, DMA_FROM_DEVICE);
if (mapped == 0)
    return -EIO;

/* Program device with each mapped segment */
for_each_sg(sg, s, mapped, i) {
    dma_addr_t addr = sg_dma_address(s);
    unsigned int len = sg_dma_len(s);
    /* Program device DMA descriptor: addr, len */
}

/* After DMA completes */
dma_unmap_sg(dev, sg, nents, DMA_FROM_DEVICE);
```

```
Scatter-Gather Memory Layout:

Physical RAM (non-contiguous):       Device view (with IOMMU):
┌──────┐                             ┌──────┐
│Page A│ @ PA 0x1000                  │Seg 0 │ IOVA 0x0000 → PA 0x1000
├──────┤                             ├──────┤
│ .... │                              │Seg 1 │ IOVA 0x1000 → PA 0x5000
├──────┤                             ├──────┤
│Page B│ @ PA 0x5000                  │Seg 2 │ IOVA 0x2000 → PA 0x3000
├──────┤                             └──────┘
│ .... │                              Contiguous IOVA range!
├──────┤
│Page C│ @ PA 0x3000
└──────┘
```

---

## 20.3 DMA Sync Operations for Streaming Mappings

```c
/*
 * When using streaming DMA, CPU must sync cache before/after access.
 * This is because CPU cache and device memory view can be inconsistent.
 */

/* Map buffer for device DMA */
dma_addr_t dma = dma_map_single(dev, buf, len, DMA_FROM_DEVICE);

/* Start DMA transfer... device writes to buffer */

/* Before CPU reads the buffer: invalidate CPU cache */
dma_sync_single_for_cpu(dev, dma, len, DMA_FROM_DEVICE);
/* Now CPU can safely read buf[] - cache is coherent */

/* If CPU modifies buffer and wants device to DMA again: */
dma_sync_single_for_device(dev, dma, len, DMA_FROM_DEVICE);
/* Cache flushed, device will see CPU's writes */

/* Finally unmap */
dma_unmap_single(dev, dma, len, DMA_FROM_DEVICE);
```

```
Cache Coherency Timeline (ARM example):

Time →
        CPU Cache        RAM           Device
────────────────────────────────────────────────
        [stale]         [X]           DMA writes X
sync_for_cpu:           
        invalidate →    [X]           
        [X]            [X]           CPU reads X ✓
        
CPU writes Y:
        [Y]            [X]           
sync_for_device:
        flush →        [Y]           
        [Y]            [Y]           Device reads Y ✓
```

---

## 20.4 DMA Mask and Addressing

```c
/* Set device DMA addressing capability */
int ret = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(64));
if (ret) {
    /* Try 32-bit if 64-bit fails */
    ret = dma_set_mask_and_coherent(dev, DMA_BIT_MASK(32));
    if (ret) {
        dev_err(dev, "No suitable DMA available\n");
        return ret;
    }
}

/*
 * DMA_BIT_MASK(32) = 0xFFFFFFFF       → 4 GB addressable
 * DMA_BIT_MASK(64) = 0xFFFFFFFFFFFFFFFF → Full 64-bit
 * DMA_BIT_MASK(24) = 0xFFFFFF          → 16 MB (ISA legacy)
 *
 * If device is 32-bit but RAM > 4GB:
 * - With IOMMU: IOVA in 32-bit range, PA can be anywhere
 * - Without IOMMU: bounce buffer (SWIOTLB) copies to low memory
 */
```

---

## 20.5 Bounce Buffers (SWIOTLB)

```
Problem: 32-bit device, memory above 4GB, no IOMMU

┌──────────┐     Cannot reach!     ┌──────────────┐
│ Device   │ ──────────X──────────→│ Buffer @ 8GB │
│ (32-bit) │                       └──────────────┘
└──────────┘
                  Bounce buffer solution:
┌──────────┐                       ┌──────────────┐
│ Device   │ ──DMA──→ ┌─────────┐  │ Buffer @ 8GB │
│ (32-bit) │          │ Bounce  │  └──────┬───────┘
└──────────┘          │ < 4GB   │         │
                      └────┬────┘         │
                           └── memcpy ────┘

SWIOTLB (Software I/O TLB):
- Reserved at boot: 64MB default (swiotlb=size kernel param)
- Allocated in low memory (< 4GB)
- Automatic: DMA API uses bounce buffer when needed
- Performance cost: extra memcpy per DMA transfer
```

```bash
# Check SWIOTLB usage
$ dmesg | grep -i swiotlb
[    0.000000] software IO TLB: mapped [mem 0x3bf800000-0x3ff800000] (1024MB)

$ cat /sys/kernel/debug/swiotlb/io_tlb_used
128    # slots currently in use
```

---

## 20.6 Memory-Mapped I/O (MMIO) in Drivers

```c
/*
 * MMIO: Device registers mapped into CPU address space
 * Must use ioremap() to get kernel virtual address
 */
#include <linux/io.h>

/* Map device registers (physical BAR address) */
void __iomem *regs = ioremap(pci_resource_start(pdev, 0),
                              pci_resource_len(pdev, 0));
if (!regs)
    return -ENOMEM;

/* Read/write device registers - MUST use accessor functions */
u32 val = readl(regs + REG_STATUS);     /* 32-bit read */
writel(0x1, regs + REG_CONTROL);        /* 32-bit write */

/* ioremap variants:
 * ioremap()           — uncacheable, strongly ordered
 * ioremap_wc()        — write-combining (for framebuffers)
 * ioremap_cache()     — cacheable (careful: coherency issues)
 * ioremap_np()        — non-posted (waits for completion)
 */

/* Cleanup */
iounmap(regs);

/* Modern helper: devm_ioremap_resource() manages lifecycle */
regs = devm_ioremap_resource(&pdev->dev, 
                              pci_resource(pdev, 0));
```

```
MMIO Address Space:

Physical Address Space:
┌────────────────────┐ 0xFFFF_FFFF_FFFF_FFFF
│                    │
│  RAM (> 4GB)       │
│                    │
├────────────────────┤ 0x1_0000_0000 (4GB)
│                    │
│  PCI MMIO Region   │ ← Device BARs mapped here
│  GPU VRAM          │   ioremap() creates kernel VA mapping
│  NIC registers     │
│                    │
├────────────────────┤ 
│  RAM (< 4GB)       │
│                    │
├────────────────────┤ 0x0000_0000
│  Legacy I/O        │
└────────────────────┘
```

---

## 20.7 mmap() for Drivers

```c
/*
 * Allow userspace to directly access device memory or DMA buffers
 * via mmap() — zero-copy path for high-performance I/O.
 */

/* In driver's file_operations: */
static int my_mmap(struct file *filp, struct vm_area_struct *vma)
{
    struct my_device *dev = filp->private_data;
    unsigned long pfn;
    
    /* Option 1: Map DMA coherent buffer to userspace */
    pfn = virt_to_phys(dev->dma_buf) >> PAGE_SHIFT;
    vma->vm_page_prot = pgprot_noncached(vma->vm_page_prot);
    
    return remap_pfn_range(vma, vma->vm_start, pfn,
                           vma->vm_end - vma->vm_start,
                           vma->vm_page_prot);
    
    /* Option 2: Map MMIO registers to userspace */
    /* pfn = pci_resource_start(pdev, 0) >> PAGE_SHIFT; */
    /* return io_remap_pfn_range(vma, ...); */
    
    /* Option 3: Use dma_mmap_coherent() for DMA buffers */
    /* return dma_mmap_coherent(dev->dev, vma, 
     *                          dev->dma_buf, dev->dma_addr,
     *                          dev->dma_size); */
}

static const struct file_operations my_fops = {
    .mmap = my_mmap,
    /* ... */
};
```

```
Driver mmap flow:

Userspace:
  fd = open("/dev/mydev", O_RDWR);
  buf = mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
  /* buf now directly accesses device memory — ZERO COPY! */
  
  ┌──────────────┐
  │ User Process  │
  │ VA: buf       │──────────────┐
  └──────────────┘               │
                            Page Table
                                 │
                    ┌────────────▼────────────┐
                    │  Physical Device Memory  │
                    │  (DMA buf or MMIO BAR)   │
                    └─────────────────────────┘
```

---

## 20.8 devm_* Managed Memory API

```c
/*
 * Managed (devm_) allocations are automatically freed when device
 * is removed. Eliminates most cleanup error paths.
 */

/* Managed allocations: */
void *buf = devm_kmalloc(dev, size, GFP_KERNEL);
void *buf = devm_kzalloc(dev, size, GFP_KERNEL);
void *buf = devm_kcalloc(dev, n, size, GFP_KERNEL);

/* Managed DMA: */
void *dma_buf = dmam_alloc_coherent(dev, size, &dma_addr, GFP_KERNEL);

/* Managed MMIO: */
void __iomem *regs = devm_ioremap_resource(dev, res);

/* No need for explicit free in remove() or error paths! */

/* Example: Clean probe function */
static int my_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct my_data *data;
    
    data = devm_kzalloc(dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    
    data->regs = devm_ioremap_resource(dev, 
                    platform_get_resource(pdev, IORESOURCE_MEM, 0));
    if (IS_ERR(data->regs))
        return PTR_ERR(data->regs);
    
    data->dma_buf = dmam_alloc_coherent(dev, BUF_SIZE,
                                         &data->dma_addr, GFP_KERNEL);
    if (!data->dma_buf)
        return -ENOMEM;
    
    /* All memory freed automatically on remove or probe failure */
    return 0;
}
```

---

## 20.9 Common Driver Memory Pitfalls

```
┌──────────────────────────────────────────────────────────────────┐
│ Pitfall                        │ Solution                       │
├──────────────────────────────────────────────────────────────────┤
│ Using kmalloc for DMA buffer   │ Use dma_alloc_coherent()       │
│ Forgetting cache sync          │ dma_sync_single_for_cpu/device │
│ Not setting DMA mask           │ dma_set_mask_and_coherent()    │
│ DMA to stack/module memory     │ Use kmalloc'd buffers only     │
│ Missing unmap on error path    │ Use devm_* managed APIs        │
│ Accessing MMIO with raw ptr    │ Use readl()/writel() accessors │
│ DMA direction mismatch         │ Match direction in map/unmap   │
│ Buffer lifetime < DMA lifetime │ Ensure buffer outlives DMA     │
│ Unaligned DMA buffers          │ kmalloc returns aligned memory │
│ Forgetting DMA_ATTR flags      │ Use DMA_ATTR_* when needed     │
└──────────────────────────────────────────────────────────────────┘
```

---

## OS Comparison: Device Memory Management

```
┌────────────┬──────────────────────────────────────────────┐
│ OS         │ Device Memory Approach                       │
├────────────┼──────────────────────────────────────────────┤
│ Linux      │ DMA API + IOMMU; devm_* lifecycle management │
│ Windows    │ WDF DMA framework; MDL for system DMA        │
│ macOS      │ IOMemoryDescriptor; IODMACommand              │
│ FreeBSD    │ busdma(9); bus_dma_tag for constraints        │
│ Android    │ Linux DMA + ION/dmabuf for shared buffers     │
│ RTOS       │ Direct physical; no IOMMU typically           │
└────────────┴──────────────────────────────────────────────┘
```

---

## Interview Questions

1. **Q: What is the difference between `dma_alloc_coherent()` and `dma_map_single()`?**
   A: `dma_alloc_coherent()` allocates new physically contiguous memory with hardware-maintained cache coherency (no manual sync). `dma_map_single()` maps existing kernel memory for DMA, requiring manual cache sync operations. Coherent is for long-lived buffers (descriptor rings); streaming is for per-transfer data buffers.

2. **Q: What happens when a 32-bit DMA device needs to access memory above 4GB?**
   A: If an IOMMU is present, it maps a 32-bit IOVA to the high physical address. Without IOMMU, the kernel uses SWIOTLB (bounce buffers): allocates a temporary buffer in low memory, DMAs to/from it, and copies to/from the actual buffer.

3. **Q: Why must you use `readl()`/`writel()` instead of raw pointer dereference for MMIO?**
   A: MMIO accessor functions provide: memory barriers to prevent reordering, proper volatile semantics, endianness conversion, and architecture-specific I/O ordering guarantees. Raw pointer access may be reordered by CPU or compiler.

4. **Q: What is a DMA bounce buffer?**
   A: A temporary buffer in low physical memory (below device's DMA mask limit) used when the target buffer is inaccessible to the device. Data is copied between the bounce buffer and actual buffer. SWIOTLB provides this transparently through the DMA API.

---

## Summary

1. **DMA API** is the standard interface: `dma_alloc_coherent()`, `dma_map_single()`, `dma_map_sg()`.
2. **Coherent** DMA is hardware-coherent (no cache ops) but expensive; **streaming** DMA maps existing memory but needs sync.
3. **Scatter-gather** allows non-contiguous physical pages to appear contiguous to devices.
4. **SWIOTLB** bounce buffers handle 32-bit devices with high memory.
5. **MMIO** access requires `ioremap()` + `readl()`/`writel()`.
6. **devm_*** managed APIs eliminate cleanup paths.
7. Driver `mmap()` enables zero-copy userspace access to device memory.

---

*Next: [Chapter 21 — Memory Mapping in Drivers (mmap)](Chapter_21_Mmap_in_Drivers.md)*
