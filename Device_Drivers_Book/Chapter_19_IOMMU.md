# Chapter 19: IOMMU and Device Address Translation

## Chapter Overview

The IOMMU (I/O Memory Management Unit) provides address translation and isolation for device DMA, similar to how the CPU MMU provides virtual memory for processes. This chapter covers IOMMU architecture, DMA remapping, and device isolation.

---

## 19.1 IOMMU Architecture

```
WITHOUT IOMMU:                       WITH IOMMU:
Device DMA uses physical address      Device DMA uses I/O virtual address

Device → phys addr → DRAM            Device → IOVA → IOMMU → phys addr → DRAM

Problems:                             Benefits:
• Device sees all physical memory     • Device limited to mapped pages
• DMA to wrong addr = corruption      • Enables scatter-gather without
• No isolation between devices          physically contiguous memory
                                      • Device isolation (VFIO)
```

```
┌──────────┐     IOVA      ┌──────────┐      PA       ┌──────────┐
│  Device   │──────────────→│  IOMMU   │──────────────→│  DRAM    │
│  (DMA)    │   0x1000      │(translates│   0x8000_0000│          │
└──────────┘               │  IOVA→PA) │              └──────────┘
                            │           │
                            │ I/O page  │
                            │ tables    │
                            └──────────┘
```

### IOMMU Hardware Implementations

| Platform | IOMMU | Source |
|----------|-------|--------|
| Intel | VT-d (Virtualization Technology for Directed I/O) | `drivers/iommu/intel/` |
| AMD | AMD-Vi | `drivers/iommu/amd/` |
| ARM | SMMU (System Memory Management Unit) | `drivers/iommu/arm/arm-smmu*` |
| IBM | TCE (Translation Control Entry) | `arch/powerpc/` |

---

## 19.2 Device Virtual Addressing (IOVA)

```
IOVA Space per device/group:
┌──────────────────────────┐
│  0x0000_0000_1000        │ → maps to PA 0x8000_0000
│  0x0000_0000_2000        │ → maps to PA 0x8001_5000
│  0x0000_0000_3000        │ → maps to PA 0x8003_A000
│  ...                     │
│  (non-contiguous PA,     │
│   contiguous IOVA)       │
└──────────────────────────┘
```

When `dma_map_single()` is called with an IOMMU present:
1. Allocate IOVA from the device's IOVA space
2. Map IOVA → physical address in IOMMU page table
3. Return IOVA to driver (as the "DMA address")
4. Device DMAs to IOVA → IOMMU translates to PA

---

## 19.3 DMA Address Translation

```
dma_map_single(dev, cpu_virt, len, dir)
       │
       ├── No IOMMU: return virt_to_phys(cpu_virt)
       │     (DMA address = physical address)
       │
       └── With IOMMU:
              │
              ├── Allocate IOVA from device's domain
              ├── Map IOVA → PA in IOMMU page table
              ├── Flush IOMMU TLB (IOTLB)
              └── Return IOVA (device uses this)
```

### IOMMU Domain Types

| Type | Use |
|------|-----|
| **DMA domain** | Default: kernel manages IOMMU mappings for DMA API |
| **Identity domain** | 1:1 mapping (IOVA = PA), like no IOMMU |
| **Unmanaged domain** | Userspace-managed mappings (VFIO) |
| **Blocking domain** | Block all DMA (isolation) |

---

## 19.4 Device Isolation

### IOMMU Groups

An IOMMU group is the smallest set of devices that the IOMMU can isolate:

```
┌────────────────────────────────┐
│ IOMMU Group 1                  │
│ ┌──────────┐  ┌──────────┐   │
│ │ PCIe fn 0│  │ PCIe fn 1│   │  Same device, can't isolate
│ │ (NIC)    │  │ (NIC)    │   │  functions from each other
│ └──────────┘  └──────────┘   │
└────────────────────────────────┘

┌────────────────────────────────┐
│ IOMMU Group 2                  │
│ ┌──────────┐                  │
│ │ GPU      │                  │  Fully isolated device
│ └──────────┘                  │
└────────────────────────────────┘
```

### VFIO: Userspace Device Access

```
VFIO uses IOMMU to safely pass devices to userspace/VMs:

VM / Userspace Application
       │
       ├── VFIO ioctl: MAP_DMA(iova=X, pa=Y)
       │
       ▼
IOMMU maps IOVA X → PA Y in device's page table
       │
Device DMA to X → IOMMU → PA Y → Guest memory
       │
Only mapped pages accessible → full isolation
```

---

## Debugging

```bash
# Check IOMMU status
dmesg | grep -i iommu

# See IOMMU groups
ls /sys/kernel/iommu_groups/
ls /sys/kernel/iommu_groups/0/devices/

# See what group a device belongs to
readlink /sys/bus/pci/devices/0000:02:00.0/iommu_group
# ../../../kernel/iommu_groups/5

# IOMMU in passthrough mode?
cat /proc/cmdline | grep iommu
# intel_iommu=on / iommu=pt (passthrough)
```

---

## Interview Questions

**Q1: What is the IOMMU and why is it important for drivers?**
A: IOMMU translates device DMA addresses (IOVAs) to physical addresses, like a CPU MMU but for devices. Benefits: memory isolation (device can only access mapped pages), scatter-gather without physically contiguous memory, and enabling safe device passthrough to VMs (VFIO).

**Q2: Does the driver need to change when an IOMMU is present?**
A: No — the DMA API (`dma_map_single`, `dma_alloc_coherent`) is IOMMU-transparent. Without IOMMU, it returns physical addresses. With IOMMU, it returns IOVAs. Drivers should always use the DMA API, never raw physical addresses.

**Q3: What is an IOMMU group?**
A: The smallest isolation unit — a set of devices that cannot be independently isolated. All devices in a group share the same IOMMU domain. For VFIO passthrough, all devices in a group must be assigned to the same VM.

---

*Next: [Chapter 20 — Power Management in Drivers](Chapter_20_Power_Management.md)*
