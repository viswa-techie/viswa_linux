# Chapter 19: IOMMU and Device Memory

## Chapter Overview

The IOMMU (I/O Memory Management Unit) translates device virtual addresses (DVA/IOVA) to physical addresses, providing DMA remapping, device isolation, and scatter-gather support. This chapter covers IOMMU architecture, Linux IOMMU subsystem, and DMA remapping.

---

## 19.1 IOMMU Architecture

```
Without IOMMU:                         With IOMMU:
CPU → VA → MMU → PA                    CPU → VA → MMU → PA
Device → PA → Memory                   Device → IOVA → IOMMU → PA → Memory

Device uses physical                   Device uses IO Virtual Address
addresses directly!                    IOMMU translates to physical
No isolation!                          Full isolation!

┌────────┐                            ┌────────┐
│  CPU   │                            │  CPU   │
│  MMU   │                            │  MMU   │
└───┬────┘                            └───┬────┘
    │ PA                                  │ PA
    ▼                                     ▼
┌───────────┐                         ┌───────────┐
│   RAM     │                         │   RAM     │
└───────────┘                         └───────────┘
    ▲ PA                                  ▲ PA
    │                                     │
┌───┴────┐                            ┌───┴────┐
│ Device │  Direct physical           │ IOMMU  │  Address translation
│ (DMA)  │  access — DANGEROUS!       └───┬────┘  + protection
└────────┘                            ┌───┴────┐
                                      │ Device │  Uses IOVA
                                      │ (DMA)  │  — SAFE!
                                      └────────┘
```

---

## 19.2 Device Virtual Addressing

```
IOMMU translates:
Device IO Virtual Address (IOVA) → Physical Address (PA)

IOMMU page table (similar to CPU page tables):
┌─────────────┐
│  IOVA 0x000 │ → PA 0x1A000  (page 0)
│  IOVA 0x001 │ → PA 0x3B000  (page 1, NON-CONTIGUOUS PA!)
│  IOVA 0x002 │ → PA 0x0C000  (page 2)
│  IOVA 0x003 │ → INVALID (not mapped — device fault!)
└─────────────┘

Benefits:
1. Scatter-gather: device sees contiguous IOVA, physical pages scattered
2. Protection: device can only access mapped pages
3. Isolation: device A can't access device B's memory  
4. 32-bit devices: IOVA can be in 32-bit range even if PA is 64-bit
```

---

## 19.3 DMA Remapping

```c
/* Linux DMA API with IOMMU */
#include <linux/dma-mapping.h>

/* Map a buffer for device access */
dma_addr_t iova = dma_map_single(dev, cpu_addr, size, DMA_TO_DEVICE);
/* Returns IOVA that device uses for DMA */

/* After DMA completes */
dma_unmap_single(dev, iova, size, DMA_TO_DEVICE);

/* Scatter-gather: Map multiple buffers as contiguous for device */
int nents = dma_map_sg(dev, sg_list, nents_orig, DMA_BIDIRECTIONAL);
/* Device sees contiguous IOVA range even though pages are scattered! */
```

---

## 19.4 Device Memory Isolation

```
IOMMU groups: devices sharing an IOMMU domain

IOMMU Domain 1:         IOMMU Domain 2:
┌──────────┐            ┌──────────┐
│ NIC      │            │ GPU      │
│ IOVA map │            │ IOVA map │
│ 0-1GB    │            │ 0-4GB    │
│ → RAM    │            │ → VRAM   │
└──────────┘            └──────────┘

NIC cannot access GPU memory (different domain)
GPU cannot access NIC buffers
VM passthrough: each VM gets its own IOMMU domain
```

```bash
# View IOMMU groups
$ find /sys/kernel/iommu_groups/ -type l
/sys/kernel/iommu_groups/0/devices/0000:00:02.0  # GPU
/sys/kernel/iommu_groups/1/devices/0000:00:1f.0  # Chipset
/sys/kernel/iommu_groups/2/devices/0000:03:00.0  # NIC

# IOMMU implementations:
# Intel: VT-d (Virtualization Technology for Directed I/O)
# AMD: AMD-Vi (AMD I/O Virtualization)
# ARM: SMMU (System Memory Management Unit)
```

---

## IOMMU/DMA Flow Diagram

```
Driver wants device to DMA from buffer at VA 0xFFFF888012340000:

1. Driver calls: dma_map_single(dev, vaddr, 4096, DMA_FROM_DEVICE)

2. DMA subsystem:
   a. Determine IOMMU domain for device
   b. Allocate IOVA from IOVA allocator
   c. Map: IOVA → physical address of buffer
   d. Insert into IOMMU page table
   e. Flush IOMMU TLB (IOTLB)
   f. Return IOVA to driver

3. Driver programs device with IOVA
4. Device DMAs using IOVA
5. IOMMU translates IOVA → PA → RAM access

6. Driver calls: dma_unmap_single(dev, iova, 4096, DMA_FROM_DEVICE)
   a. Remove IOMMU mapping
   b. Free IOVA
   c. Flush IOTLB
```

---

## Interview Questions

1. **Q: What is an IOMMU and why is it needed?**
   A: An IOMMU translates device DMA addresses to physical addresses, providing device isolation (devices can't access arbitrary memory), scatter-gather support (non-contiguous physical pages appear contiguous to devices), and VM passthrough capability.

2. **Q: What is the difference between DMA with and without IOMMU?**
   A: Without IOMMU, devices use physical addresses directly (no isolation, scatter-gather requires physical contiguity). With IOMMU, devices use IOVAs translated by hardware (isolation, scatter-gather, 32-bit bounce buffer elimination).

---

## Summary

1. IOMMU provides address translation for device DMA, analogous to CPU MMU.
2. Enables device isolation, scatter-gather, and secure DMA.
3. Linux DMA API (`dma_map_*`) abstracts IOMMU details.
4. IOMMU groups isolate devices into separate address domains.
5. Critical for virtualization (VM passthrough) and security.

---

*Next: [Chapter 20 — Memory Management for Drivers](Chapter_20_DMA_Memory.md)*
