# Chapter 27: vhost and vhost-user Architecture

## Learning Goals
- Understand vhost kernel bypass architecture
- Learn vhost-user protocol for userspace backends
- Master vhost-net, vhost-scsi, and vhost-vsock
- Know vDPA (virtio Data Path Acceleration) framework

---

## 1. vhost Kernel Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Problem: standard virtio backend runs in QEMU           │
  │  Guest I/O → VM exit → KVM → QEMU → kernel → device    │
  │  Two context switches per I/O (kernel↔userspace)        │
  │                                                           │
  │  vhost solution: move backend into kernel                │
  │  Guest I/O → VM exit → KVM → vhost (kernel) → device   │
  │  Zero userspace context switches!                       │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Without vhost:                           │            │
  │  │ ┌──────┐ ┌────────┐ ┌──────────────────┐│            │
  │  │ │Guest │→│ KVM    │→│ QEMU             ││            │
  │  │ │virtio│ │(kernel)│ │ (userspace)      ││            │
  │  │ │driver│ │        │ │  virtio backend  ││            │
  │  │ │      │ │        │ │  ↓               ││            │
  │  │ │      │ │        │ │  kernel syscall  ││            │
  │  │ │      │ │        │ │  ↓               ││            │
  │  │ │      │ │        │ │  NIC driver      ││            │
  │  │ └──────┘ └────────┘ └──────────────────┘│            │
  │  │                                          │            │
  │  │ With vhost-net:                          │            │
  │  │ ┌──────┐ ┌────────────────────────┐      │            │
  │  │ │Guest │→│ KVM + vhost-net        │      │            │
  │  │ │virtio│ │ (all in kernel)        │      │            │
  │  │ │driver│ │   ↓                    │      │            │
  │  │ │      │ │   NIC driver           │      │            │
  │  │ └──────┘ └────────────────────────┘      │            │
  │  │                                          │            │
  │  │ QEMU still handles control path:         │            │
  │  │  - Device discovery, feature negotiation │            │
  │  │  - Queue setup (passes FDs to vhost)     │            │
  │  │  - Migration state                       │            │
  │  │ But QEMU is NOT on the data path!        │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  vhost ioctl interface:                                  │
  │  fd = open("/dev/vhost-net", O_RDWR);                    │
  │  ioctl(fd, VHOST_SET_OWNER, ...);                        │
  │  ioctl(fd, VHOST_SET_MEM_TABLE, &mem_regions);           │
  │  ioctl(fd, VHOST_SET_VRING_NUM, &vring_num);             │
  │  ioctl(fd, VHOST_SET_VRING_ADDR, &vring_addr);          │
  │  ioctl(fd, VHOST_SET_VRING_KICK, &kick_fd);  // ioeventfd│
  │  ioctl(fd, VHOST_SET_VRING_CALL, &call_fd);  // irqfd   │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. vhost-user Protocol

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  vhost-user: move backend to a SEPARATE userspace process│
  │  (instead of kernel)                                     │
  │                                                           │
  │  Use cases: DPDK, SPDK, custom I/O backends             │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Guest VM                                  │            │
  │  │   virtio-net driver                       │            │
  │  │        │                                  │            │
  │  │   ┌────┴────┐                             │            │
  │  │   │ virtqueue│ (shared memory)            │            │
  │  │   └────┬────┘                             │            │
  │  │        │                                  │            │
  │  │   QEMU (master)                           │            │
  │  │        │ unix socket                      │            │
  │  │        │ (control messages only)          │            │
  │  │        ▼                                  │            │
  │  │   Backend process (slave)                 │            │
  │  │     e.g., OVS-DPDK, SPDK                │            │
  │  │        │                                  │            │
  │  │     Backend directly accesses virtqueue   │            │
  │  │     in shared memory (mmap'd)            │            │
  │  │        │                                  │            │
  │  │     Physical NIC (DPDK PMD)               │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Protocol messages (over unix socket):                   │
  │  ┌──────────────────────────────────────────┐            │
  │  │ VHOST_USER_GET_FEATURES                  │            │
  │  │ VHOST_USER_SET_FEATURES                  │            │
  │  │ VHOST_USER_SET_MEM_TABLE   + FDs (shm)  │            │
  │  │ VHOST_USER_SET_VRING_NUM                 │            │
  │  │ VHOST_USER_SET_VRING_ADDR                │            │
  │  │ VHOST_USER_SET_VRING_KICK  + eventfd    │            │
  │  │ VHOST_USER_SET_VRING_CALL  + eventfd    │            │
  │  │ VHOST_USER_SET_VRING_ENABLE              │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Key: guest memory is shared via file-backed mmap       │
  │  QEMU passes memory region FDs to backend               │
  │  Backend mmaps same pages → directly reads/writes       │
  │  virtqueue descriptors without any data copy            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. vhost Variants

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌────────────────┬────────────────────────────────────┐│
  │  │ Module         │ Description                         ││
  │  ├────────────────┼────────────────────────────────────┤│
  │  │ vhost-net      │ Kernel network backend              ││
  │  │                │ /dev/vhost-net                       ││
  │  │                │ Works with TAP + bridge/OVS         ││
  │  │                │ ~2-3x faster than QEMU backend      ││
  │  ├────────────────┼────────────────────────────────────┤│
  │  │ vhost-scsi     │ Kernel SCSI target backend          ││
  │  │                │ Uses LIO (Linux SCSI target)        ││
  │  │                │ Direct kernel path for storage      ││
  │  ├────────────────┼────────────────────────────────────┤│
  │  │ vhost-vsock    │ Host↔guest socket communication    ││
  │  │                │ AF_VSOCK socket family              ││
  │  │                │ No IP networking required           ││
  │  │                │ Used by Firecracker, Kata           ││
  │  ├────────────────┼────────────────────────────────────┤│
  │  │ vhost-user-net │ Userspace network backend           ││
  │  │                │ Used by DPDK/OVS-DPDK              ││
  │  │                │ Highest performance (zero-copy)     ││
  │  ├────────────────┼────────────────────────────────────┤│
  │  │ vhost-user-blk │ Userspace block backend             ││
  │  │                │ Used by SPDK                        ││
  │  │                │ Millions of IOPS                    ││
  │  ├────────────────┼────────────────────────────────────┤│
  │  │ vhost-user-fs  │ Userspace filesystem backend        ││
  │  │ (virtiofsd)    │ FUSE passthrough + DAX             ││
  │  │                │ Used by Kata for rootfs             ││
  │  └────────────────┴────────────────────────────────────┘│
  └──────────────────────────────────────────────────────────┘
```

---

## 4. vDPA Framework

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  vDPA = virtio Data Path Acceleration                    │
  │  Hardware NIC implements virtio ring format              │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │                                          │            │
  │  │  Without vDPA (software backend):        │            │
  │  │  Guest virtio ──→ vhost-net ──→ TAP      │            │
  │  │                    (kernel)     (kernel)  │            │
  │  │                                   │       │            │
  │  │                                 bridge    │            │
  │  │                                   │       │            │
  │  │                                 NIC drv   │            │
  │  │                                          │            │
  │  │  With vDPA (hardware data path):         │            │
  │  │  Guest virtio ──→ NIC hardware directly  │            │
  │  │                    (virtio ring in HW)    │            │
  │  │                                          │            │
  │  │  Control path: kernel vDPA framework     │            │
  │  │  Data path: hardware processes virtqueue │            │
  │  │                                          │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  vDPA architecture:                                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Guest: standard virtio-net driver        │            │
  │  │     ↕ (virtqueue in shared memory)       │            │
  │  │ QEMU: vhost-vdpa backend                 │            │
  │  │     ↕ (ioctl to /dev/vhost-vdpa-0)       │            │
  │  │ Kernel: vDPA bus + vhost-vdpa module     │            │
  │  │     ↕ (vdpa_config_ops)                  │            │
  │  │ Vendor vDPA driver (e.g., mlx5_vdpa)     │            │
  │  │     ↕                                    │            │
  │  │ SmartNIC hardware (virtio ring engine)   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Advantages:                                             │
  │  1. Standard virtio driver in guest (migration-friendly)│
  │  2. Near-native performance (HW data path)             │
  │  3. Live migration: save/restore virtio state           │
  │  4. No SR-IOV driver complexity in guest                │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Explain the difference between vhost, vhost-user, and vDPA.**
**A:** All three are optimizations of the virtio backend (device-side processing), but they differ in where the backend runs. **vhost (kernel)**: the virtio backend runs as a kernel thread (vhost worker) instead of in QEMU userspace. QEMU opens `/dev/vhost-net`, passes memory mappings and eventfd descriptors via ioctls. The kernel thread directly processes the virtqueue, doing I/O via kernel subsystems (e.g., socket send/recv for networking). Benefit: eliminates userspace↔kernel context switches on the data path. Cost: backend code must be in the kernel. **vhost-user**: the backend runs in a separate userspace process (DPDK, SPDK, virtiofsd). QEMU communicates with the backend via a Unix socket (vhost-user protocol). QEMU passes file descriptors for guest memory regions (file-backed shared memory); the backend mmaps the same pages and directly reads/writes virtqueue descriptors. Benefit: backend in userspace (easier to develop, can use DPDK/SPDK). The backend polls the virtqueue on dedicated CPU cores for lowest latency. **vDPA (hardware)**: the physical NIC hardware directly implements the virtio virtqueue format. The kernel's vDPA bus provides control path management, but the data path goes directly from the guest virtqueue to hardware — no software backend at all. Guest uses standard virtio driver (unlike SR-IOV which needs vendor driver), enabling transparent live migration. SmartNIC examples: Mellanox ConnectX-6 Dx (mlx5_vdpa), Intel IPU.

**Q2: How does vhost-user achieve zero-copy I/O between guest and backend?**
**A:** The key mechanism is shared memory. During initialization: (1) QEMU allocates guest RAM as file-backed memory (memfd or hugepage-backed files). (2) Via the vhost-user protocol (VHOST_USER_SET_MEM_TABLE message), QEMU sends the file descriptors for these memory regions to the backend process over the Unix socket (using SCM_RIGHTS). (3) The backend process mmaps these file descriptors at the same offsets, creating a shared view of guest physical memory. Now both QEMU and the backend can access the same physical pages. The virtqueue descriptors contain guest physical addresses pointing to buffers within this shared memory. The backend (e.g., DPDK) can read TX packet data directly from the virtqueue descriptors' buffer addresses or write received packets directly into RX buffers — no memcpy needed. For notification: ioeventfd (guest writes to MMIO → eventfd → backend wakes up) and irqfd (backend signals eventfd → KVM injects interrupt) avoid QEMU involvement entirely. The "zero-copy" claim has nuances: with standard small packets, data is copied between the virtqueue buffer and the NIC DMA buffer. True zero-copy (guest buffer → NIC DMA directly) requires vhost-user zero-copy TX mode where the NIC DMAs from the guest's virtqueue buffer addresses (mapped via IOMMU). This is fragile and rarely used; the more common approach relies on hugepages reducing TLB misses rather than eliminating all copies.

---

## Summary

- vhost-net: kernel backend thread processes virtqueues, 2-3x faster than QEMU backend
- vhost-user: separate userspace backend (DPDK, SPDK), shared memory for zero-copy
- vhost-user protocol: Unix socket for control, shared mmap for data
- vhost-vsock: AF_VSOCK for host↔guest communication (no IP needed)
- vDPA: hardware implements virtio rings — standard driver + native performance + migration
- Control vs data path separation: QEMU handles setup, vhost/vDPA handles I/O

---

[Previous: Network & Storage Virtualization ←](Chapter_26_Network_Storage_Virt.md) | [Next: Device Passthrough & SR-IOV →](Chapter_28_VFIO_SRIOV.md)
