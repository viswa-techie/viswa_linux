# Chapter 18: Contiguous Memory Allocation (CMA)

## Chapter Overview

Some devices (cameras, displays, video decoders) require large, physically contiguous memory buffers that can't be provided by the buddy allocator after the system has been running. CMA reserves regions of physical memory at boot that can be used for normal allocations when idle but reclaimed for DMA when needed.

---

## 18.1 CMA Concept

```
Traditional problem:
Boot → Run for hours → Memory fragmented → Need 16MB contiguous for camera → FAIL!

CMA solution:
Boot → Reserve 64MB CMA region → Used for normal movable pages 
     → Camera needs 16MB → Migrate movable pages out → Allocate contiguous for camera
     → Camera done → Free → Movable pages can use the region again
```

---

## 18.2 DMA Memory Requirements

Many devices need physically contiguous memory for DMA:
- Camera ISP: Large frame buffers (8-32MB per frame)
- Display controller: Framebuffer (several MB)
- Video codec: Compressed/decompressed buffers
- GPU: Command/data buffers

---

## 18.3 CMA Configuration

```bash
# Boot parameter to set CMA size
# cma=256M    # Reserve 256MB for CMA

# Device tree (embedded systems):
# reserved-memory {
#     cma: linux,cma {
#         compatible = "shared-dma-pool";
#         reusable;
#         size = <0x0 0x10000000>;  /* 256MB */
#     };
# };
```

```c
/* Kernel: Using CMA */
#include <linux/dma-mapping.h>

/* DMA allocation that uses CMA */
void *buf = dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);
/* If dev's DMA uses CMA, this allocates from CMA region */

dma_free_coherent(dev, size, buf, dma_handle);
```

---

## 18.4 CMA Allocator Mechanism

```
CMA Region in physical memory:

Normal operation:
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ M  │ M  │ M  │ M  │ M  │ M  │ M  │ M  │  All movable pages (user data)
└────┴────┴────┴────┴────┴────┴────┴────┘
CMA region

DMA allocation request (4 pages contiguous):
Step 1: Migrate movable pages out
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ M  │ M  │FREE│FREE│FREE│FREE│ M  │ M  │  Pages migrated elsewhere
└────┴────┴────┴────┴────┴────┴────┴────┘

Step 2: Allocate contiguous block
┌────┬────┬════╤════╤════╤════┬────┬────┐
│ M  │ M  │DMA │DMA │DMA │DMA │ M  │ M  │  Contiguous DMA buffer
└────┴────┴════╧════╧════╧════┴────┴────┘

Step 3: After DMA done, free
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ M  │ M  │FREE│FREE│FREE│FREE│ M  │ M  │  Available for movable again
└────┴────┴────┴────┴────┴────┴────┴────┘
```

---

## Interview Questions

1. **Q: What is CMA and when is it needed?**
   A: CMA (Contiguous Memory Allocator) reserves physical memory regions at boot for large contiguous DMA allocations. The reserved memory is used for normal movable pages when not needed for DMA.

2. **Q: Why can't the buddy allocator provide large contiguous memory?**
   A: After running, physical memory becomes fragmented with unmovable kernel allocations scattered throughout. CMA ensures a reserved region contains only movable pages that can be migrated on demand.

---

## Summary

1. CMA solves the contiguous memory allocation problem for DMA devices.
2. CMA regions hold movable pages normally, migrated out when DMA needs them.
3. Configured via boot parameters or device tree (embedded).
4. Used transparently via `dma_alloc_coherent()`.

---

*Next: [Chapter 19 — IOMMU and Device Memory](Chapter_19_IOMMU.md)*
