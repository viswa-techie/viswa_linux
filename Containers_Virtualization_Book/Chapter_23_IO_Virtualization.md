# Chapter 23: I/O Virtualization and VFIO

## Learning Goals
- Understand IOMMU-based device passthrough
- Learn VFIO framework for safe device assignment
- Master SR-IOV for hardware-level device sharing
- Know IOMMU groups and security boundaries

---

## 1. IOMMU and Device Passthrough

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  IOMMU = I/O Memory Management Unit                      │
  │  (Intel VT-d / AMD-Vi / ARM SMMU)                       │
  │                                                           │
  │  Without IOMMU:                                          │
  │  Devices do DMA using HOST physical addresses            │
  │  Giving a device to a VM → device can DMA anywhere!     │
  │  → Guest can read/write ALL host memory = UNSAFE        │
  │                                                           │
  │  With IOMMU:                                             │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Device DMA address → IOMMU → Physical addr           │
  │  │                                                       │
  │  │ IOMMU page tables (like CPU page tables):            │
  │  │ Device sees: 0x0000 - 0xFFFF (guest phys range)     │
  │  │ IOMMU maps to: scattered host physical pages         │
  │  │                                                       │
  │  │ Device can ONLY access memory mapped in IOMMU        │
  │  │ DMA to unmapped address → IOMMU fault (blocked)     │
  │  │                                                       │
  │  │ ┌─────────┐    ┌────────┐    ┌───────────┐          │
  │  │ │ Device  │───►│ IOMMU  │───►│ Physical  │          │
  │  │ │ (NIC)   │DMA │ (VT-d) │    │ RAM       │          │
  │  │ │ DMA addr│    │ tables │    │ (only     │          │
  │  │ │ = GPA   │    │        │    │  guest's  │          │
  │  │ │         │    │GPA→HPA │    │  pages)   │          │
  │  │ └─────────┘    └────────┘    └───────────┘          │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  IOMMU provides:                                         │
  │  1. DMA remapping — device DMA addr → physical addr    │
  │  2. Interrupt remapping — device MSI → correct vCPU     │
  │  3. Device isolation — one device can't affect another  │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. VFIO Framework

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  VFIO = Virtual Function I/O                             │
  │  Safe userspace interface for device passthrough         │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ QEMU (userspace)                         │            │
  │  │   │                                      │            │
  │  │   ├── open(/dev/vfio/vfio)  ← container fd│           │
  │  │   ├── open(/dev/vfio/42)    ← group fd   │            │
  │  │   ├── ioctl(VFIO_GROUP_GET_DEVICE_FD)     │            │
  │  │   │   → device fd                        │            │
  │  │   │                                      │            │
  │  │   ├── ioctl(VFIO_DEVICE_GET_REGION_INFO) │            │
  │  │   │   → BAR info (size, offset, flags)   │            │
  │  │   │                                      │            │
  │  │   ├── mmap(device_fd, BAR offset)         │            │
  │  │   │   → direct access to device MMIO     │            │
  │  │   │                                      │            │
  │  │   └── ioctl(VFIO_IOMMU_MAP_DMA)          │            │
  │  │       → set up IOMMU mapping             │            │
  │  │       → guest phys addr → host phys addr │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  VFIO + KVM integration:                                 │
  │  1. QEMU opens VFIO device                              │
  │  2. Maps device BARs into guest address space           │
  │  3. Sets up IOMMU to map GPA → HPA for DMA             │
  │  4. Guest driver accesses device directly               │
  │  5. Device DMA uses IOMMU-remapped addresses            │
  │  6. Interrupts: MSI-X → KVM → guest IRQ (irqfd)        │
  │                                                           │
  │  Data path (no QEMU involvement):                       │
  │  Guest → MMIO write to device BAR → hardware            │
  │  Device → DMA read/write → IOMMU → guest memory        │
  │  Device → MSI interrupt → IOMMU remap → KVM → guest    │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. IOMMU Groups

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  IOMMU group = smallest unit of device isolation         │
  │                                                           │
  │  Devices in the same IOMMU group can potentially        │
  │  access each other's memory (peer-to-peer DMA)          │
  │  → Must be assigned to the SAME VM (or all to host)     │
  │                                                           │
  │  PCIe topology and IOMMU groups:                         │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Root Complex                              │            │
  │  │   ├── PCIe slot 1 (own group: 42)        │            │
  │  │   │   └── NIC                             │            │
  │  │   │       → Can assign NIC alone to VM   │            │
  │  │   │                                      │            │
  │  │   ├── PCIe slot 2 (own group: 43)        │            │
  │  │   │   └── GPU                             │            │
  │  │   │       → Can assign GPU alone to VM   │            │
  │  │   │                                      │            │
  │  │   └── Legacy PCI bridge (group: 44)      │            │
  │  │       ├── Sound card                      │            │
  │  │       └── USB controller                  │            │
  │  │       → These SHARE a group!             │            │
  │  │       → Must assign BOTH to same VM      │            │
  │  │       → Or neither (both stay on host)   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Check groups:                                           │
  │  ls /sys/kernel/iommu_groups/42/devices/                 │
  │  → 0000:03:00.0  (should be ONE device for clean       │
  │                    passthrough)                          │
  │                                                           │
  │  ACS (Access Control Services):                          │
  │  PCIe capability that prevents peer-to-peer DMA         │
  │  When ACS is available: each device gets its own group  │
  │  Without ACS: devices behind same switch share a group  │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. SR-IOV (Single Root I/O Virtualization)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  SR-IOV: hardware shares ONE device among MULTIPLE VMs  │
  │                                                           │
  │  Physical NIC with SR-IOV:                               │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Physical Function (PF)                   │            │
  │  │ - Full-featured PCIe function            │            │
  │  │ - Managed by host driver                 │            │
  │  │ - Controls device configuration          │            │
  │  │                                          │            │
  │  │ Virtual Functions (VFs):                 │            │
  │  │ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐   │            │
  │  │ │ VF 0 │ │ VF 1 │ │ VF 2 │ │ VF 3 │   │            │
  │  │ │assign│ │assign│ │assign│ │unused│   │            │
  │  │ │to VM1│ │to VM2│ │to VM3│ │      │   │            │
  │  │ └──────┘ └──────┘ └──────┘ └──────┘   │            │
  │  │                                          │            │
  │  │ Each VF:                                 │            │
  │  │ - Lightweight PCIe function              │            │
  │  │ - Own MMIO BARs, MSI-X interrupts       │            │
  │  │ - Own TX/RX queues in hardware          │            │
  │  │ - Independent DMA (via IOMMU)           │            │
  │  │ - Appears as separate PCIe device       │            │
  │  │ - Assignable to VM via VFIO             │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  # Enable SR-IOV VFs:                                    │
  │  echo 4 > /sys/class/net/enp3s0/device/sriov_numvfs     │
  │  # Creates 4 VFs: enp3s0v0, enp3s0v1, enp3s0v2, enp3s0v3│
  │                                                           │
  │  # Assign VF to VM via VFIO:                             │
  │  echo 0000:03:00.1 > /sys/bus/pci/devices/.../driver/unbind│
  │  echo vfio-pci > /sys/bus/pci/devices/.../driver_override│
  │  echo 0000:03:00.1 > /sys/bus/pci/drivers/vfio-pci/bind │
  │                                                           │
  │  Performance: near-native (same hardware queues)         │
  │  Scalability: dozens of VFs per NIC                     │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Comparison of I/O Approaches

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌──────────────┬───────────┬──────────┬──────────────┐ │
  │  │              │ Emulation │ virtio   │ Passthrough  │ │
  │  │              │ (QEMU)   │          │ (VFIO)       │ │
  │  ├──────────────┼───────────┼──────────┼──────────────┤ │
  │  │ VM exits     │ Many      │ Few      │ Zero (data)  │ │
  │  │ per I/O      │ (per reg) │ (batch)  │              │ │
  │  ├──────────────┼───────────┼──────────┼──────────────┤ │
  │  │ Performance  │ 10-30%    │ 80-95%   │ 100% native  │ │
  │  │ (of native)  │           │          │              │ │
  │  ├──────────────┼───────────┼──────────┼──────────────┤ │
  │  │ Guest driver │ Unmodified│ virtio   │ Native HW    │ │
  │  │ required     │ (emulated)│ driver   │ driver       │ │
  │  ├──────────────┼───────────┼──────────┼──────────────┤ │
  │  │ Sharing      │ Yes       │ Yes      │ No (SR-IOV   │ │
  │  │              │           │          │  to share)   │ │
  │  ├──────────────┼───────────┼──────────┼──────────────┤ │
  │  │ Migration    │ Yes       │ Yes      │ Difficult    │ │
  │  │              │           │          │ (device state)│ │
  │  ├──────────────┼───────────┼──────────┼──────────────┤ │
  │  │ Security     │ Device    │ Device   │ IOMMU        │ │
  │  │              │ model bugs│ model    │ hardware     │ │
  │  │              │ in QEMU   │ simpler  │ isolation    │ │
  │  ├──────────────┼───────────┼──────────┼──────────────┤ │
  │  │ Use case     │ Legacy    │ Default  │ High-perf    │ │
  │  │              │ compat    │ choice   │ (GPU, NIC)   │ │
  │  └──────────────┴───────────┴──────────┴──────────────┘ │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does VFIO enable safe device passthrough to VMs?**
**A:** VFIO provides a userspace interface for assigning physical devices to VMs while maintaining security through IOMMU isolation. The process: (1) Unbind the device from its host driver and bind it to `vfio-pci` driver. (2) QEMU opens `/dev/vfio/vfio` (container fd) and `/dev/vfio/<group>` (IOMMU group fd). (3) QEMU gets a device fd via `VFIO_GROUP_GET_DEVICE_FD`. (4) QEMU queries device BARs via `VFIO_DEVICE_GET_REGION_INFO` and mmaps them — these MMIO regions are mapped into guest address space so the guest driver can access device registers directly (no VM exits). (5) QEMU sets up IOMMU DMA mappings via `VFIO_IOMMU_MAP_DMA` — this configures the IOMMU page tables so that when the device does DMA using guest physical addresses, the IOMMU translates them to the correct host physical pages. (6) Device interrupts (MSI-X) are remapped through the IOMMU's interrupt remapping table and delivered to KVM via irqfd, which injects them into the guest. Security: the IOMMU ensures the device can ONLY access memory regions explicitly mapped for its VM. DMA to any other address causes an IOMMU fault. IOMMU groups ensure that devices that can peer-to-peer DMA are in the same isolation boundary.

**Q2: What is SR-IOV and how does it solve the device sharing problem?**
**A:** Without SR-IOV, device passthrough is exclusive: one physical device assigned to one VM. SR-IOV is a PCIe specification that enables a single physical device (typically a NIC or storage controller) to present multiple virtual instances to the PCIe bus. The Physical Function (PF) is the full-featured device managed by the host driver — it controls device configuration, firmware, and VF creation. Virtual Functions (VFs) are lightweight PCIe functions, each with their own MMIO BARs, MSI-X interrupt vectors, and dedicated hardware TX/RX queues. Each VF appears as a separate PCIe device and can be assigned to a different VM via VFIO. The hardware NIC internally multiplexes/demultiplexes traffic between VFs based on MAC address or VLAN. Performance is near-native because each VM's VF driver talks directly to dedicated hardware queues — no software emulation or virtio backend overhead. Typical NIC: 64-128 VFs per physical port. Trade-offs: (1) VF driver needed in guest (but standard driver — same as PF's vendor driver). (2) Live migration is complex (must save/restore VF state or hot-unplug VF). (3) VF features may be limited vs PF (restricted configuration access). (4) Hardware-specific — not all devices support SR-IOV.

---

## Summary

- IOMMU: translates device DMA addresses → physical addresses, isolates device memory access
- VFIO: userspace framework for safe device passthrough; uses IOMMU for DMA/interrupt remapping
- Data path with VFIO: guest → device MMIO (no VM exit) → device DMA via IOMMU → guest memory
- IOMMU groups: smallest isolation unit; devices sharing a bus/bridge must go to same VM
- SR-IOV: hardware creates Virtual Functions (VFs) from one Physical Function (PF)
- Each VF: own PCIe BARs, MSI-X, hardware queues → assignable to different VMs
- SR-IOV performance: near-native; trade-off: migration complexity, hardware-specific

---

[Previous: Memory Virtualization ←](Chapter_22_Memory_Virtualization.md) | [Next: Virtio Specification →](Chapter_24_Virtio_Spec.md)
