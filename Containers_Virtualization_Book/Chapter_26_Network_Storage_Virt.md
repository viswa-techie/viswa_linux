# Chapter 26: Network and Storage Virtualization

## Learning Goals
- Understand SR-IOV network virtualization with OVS and DPDK
- Learn virtual switching architectures
- Master storage virtualization: virtio-blk, virtio-scsi, NVMe
- Know distributed storage integration with VMs

---

## 1. Virtual Switching

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Virtual switch: L2 forwarding between VMs and host     │
  │                                                           │
  │  ┌─────────────────────────────────────────┐             │
  │  │           Physical NIC (pNIC)           │             │
  │  │                 │                        │             │
  │  │           ┌─────┴─────┐                 │             │
  │  │           │ Linux     │                 │             │
  │  │           │ Bridge    │                 │             │
  │  │           │ (or OVS)  │                 │             │
  │  │           └──┬──┬──┬──┘                 │             │
  │  │              │  │  │                    │             │
  │  │         ┌────┘  │  └────┐               │             │
  │  │         │       │       │               │             │
  │  │       tap0    tap1    tap2              │             │
  │  │         │       │       │               │             │
  │  │       VM 1    VM 2    VM 3              │             │
  │  └─────────────────────────────────────────┘             │
  │                                                           │
  │  Linux bridge (simple):                                  │
  │  - Kernel-space L2 switch                               │
  │  - brctl or ip link add br0 type bridge                 │
  │  - TAP interfaces (vnet0, vnet1) per VM                 │
  │  - STP, VLAN support, basic forwarding                  │
  │  - Performance: ~5 Gbps software path                   │
  │                                                           │
  │  Open vSwitch (OVS):                                     │
  │  - Production-grade virtual switch                      │
  │  - OpenFlow programmable forwarding rules               │
  │  - Kernel datapath (fast path) + userspace daemon       │
  │  - VXLAN, GRE, Geneve tunneling support                │
  │  - DPDK mode: userspace datapath (10-40 Gbps)          │
  │  - Used by: OpenStack, Kubernetes (OVN/OVS-CNI)        │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. DPDK and High-Performance Networking

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  DPDK = Data Plane Development Kit                       │
  │  Bypass kernel entirely for packet processing           │
  │                                                           │
  │  Traditional path:                                       │
  │  NIC → IRQ → kernel driver → sk_buff → bridge → tap     │
  │  → QEMU → virtio backend → guest                       │
  │  (many copies, context switches, cache misses)           │
  │                                                           │
  │  DPDK path:                                              │
  │  NIC → DPDK PMD (poll-mode driver in userspace)         │
  │  → OVS-DPDK → vhost-user → guest virtio               │
  │  (zero-copy possible, dedicated CPU cores polling)      │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Guest                                     │            │
  │  │   virtio-net driver                       │            │
  │  │     │                                     │            │
  │  │     ▼                                     │            │
  │  │ vhost-user (shared memory)                │            │
  │  │     │                                     │            │
  │  │     ▼                                     │            │
  │  │ OVS-DPDK (userspace switch)               │            │
  │  │     │                                     │            │
  │  │     ▼                                     │            │
  │  │ DPDK PMD (userspace NIC driver)           │            │
  │  │     │                                     │            │
  │  │     ▼                                     │            │
  │  │ Physical NIC (via UIO/VFIO)               │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Performance comparison:                                 │
  │  ┌──────────────────┬──────────────────────┐            │
  │  │ Method           │ Throughput (64B pkts) │            │
  │  ├──────────────────┼──────────────────────┤            │
  │  │ Kernel bridge    │ ~2 Mpps              │            │
  │  │ OVS (kernel)     │ ~3 Mpps              │            │
  │  │ OVS-DPDK         │ ~14 Mpps             │            │
  │  │ SR-IOV           │ ~20 Mpps             │            │
  │  └──────────────────┴──────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Storage Virtualization

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Storage backend options:                                │
  │                                                           │
  │  ┌────────────────┬────────────────────────────────────┐│
  │  │ Method         │ Description                         ││
  │  ├────────────────┼────────────────────────────────────┤│
  │  │ virtio-blk     │ Simple block device                 ││
  │  │                │ Single queue (multiqueue in 5.x+)   ││
  │  │                │ Guest sees /dev/vda                  ││
  │  │                │ QEMU backend: qcow2/raw file       ││
  │  │                │ Fast, simple, limited SCSI features ││
  │  ├────────────────┼────────────────────────────────────┤│
  │  │ virtio-scsi    │ Full SCSI protocol over virtio      ││
  │  │                │ Multiple LUNs, SCSI commands       ││
  │  │                │ Hot-plug, SCSI passthrough          ││
  │  │                │ More feature-rich than virtio-blk  ││
  │  ├────────────────┼────────────────────────────────────┤│
  │  │ NVMe emulation │ QEMU emulates NVMe controller      ││
  │  │                │ Guest uses NVMe driver              ││
  │  │                │ Multiple submission/completion      ││
  │  │                │ queues (multiqueue native)          ││
  │  ├────────────────┼────────────────────────────────────┤│
  │  │ NVMe passthru  │ Physical NVMe passed via VFIO      ││
  │  │                │ Native performance                  ││
  │  │                │ No sharing between VMs              ││
  │  └────────────────┴────────────────────────────────────┘│
  │                                                           │
  │  Disk image formats:                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ raw:    Fixed-size, direct mapping        │            │
  │  │         Fastest I/O, wastes space         │            │
  │  │                                          │            │
  │  │ qcow2:  QEMU Copy-On-Write v2            │            │
  │  │         Thin provisioning (grows on use) │            │
  │  │         Snapshots, backing files          │            │
  │  │         Encryption (LUKS)                │            │
  │  │         Compression                       │            │
  │  │         Slightly slower than raw          │            │
  │  │                                          │            │
  │  │ vmdk/vhdi/vdi: compatibility formats     │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. vhost-user-blk and SPDK

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  SPDK = Storage Performance Development Kit              │
  │  (Like DPDK but for storage)                             │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Guest                                     │            │
  │  │   virtio-blk driver                       │            │
  │  │     │                                     │            │
  │  │     ▼                                     │            │
  │  │ vhost-user-blk (shared memory)            │            │
  │  │     │                                     │            │
  │  │     ▼                                     │            │
  │  │ SPDK vhost target (userspace)             │            │
  │  │     │                                     │            │
  │  │     ▼                                     │            │
  │  │ SPDK NVMe driver (userspace, poll-mode)   │            │
  │  │     │                                     │            │
  │  │     ▼                                     │            │
  │  │ Physical NVMe SSD (via UIO/VFIO)          │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Benefits:                                               │
  │  - Zero kernel involvement on data path                 │
  │  - Millions of IOPS per VM                              │
  │  - Sub-10μs latency                                     │
  │  - Multiple VMs share one NVMe via SPDK                 │
  │                                                           │
  │  SPDK vs kernel storage:                                 │
  │  ┌──────────────────┬───────────┬──────────┐            │
  │  │                  │ Kernel    │ SPDK     │            │
  │  ├──────────────────┼───────────┼──────────┤            │
  │  │ IOPS (4K random) │ ~300K     │ ~1.5M   │            │
  │  │ Latency (4K)     │ ~25μs     │ ~7μs    │            │
  │  │ CPU per IOP      │ High      │ Low     │            │
  │  │ CPU dedication   │ Shared    │ Polled  │            │
  │  └──────────────────┴───────────┴──────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Compare Linux bridge, OVS kernel, OVS-DPDK, and SR-IOV for VM networking.**
**A:** **Linux bridge**: simplest, kernel-space L2 switch using TAP interfaces. Supports VLANs, STP, basic ACLs. Performance: ~2 Mpps (small packets), ~5 Gbps (large packets). Good for small deployments. **OVS kernel**: OpenFlow-programmable virtual switch with kernel datapath. Supports tunneling (VXLAN, Geneve), QoS, connection tracking. Used by OpenStack Neutron. Performance: ~3 Mpps — kernel overhead from sk_buff allocations, interrupts, context switches. **OVS-DPDK**: same OVS control plane but datapath runs in userspace using DPDK poll-mode drivers. NIC bound to UIO/VFIO (bypasses kernel driver). Dedicated CPU cores poll for packets (busy-wait). Performance: ~14 Mpps — 5x faster than kernel OVS. Trade-off: CPU cores dedicated to polling (wasted if traffic is low). Works with vhost-user for guest connectivity (shared memory, zero-copy possible). **SR-IOV**: hardware NIC creates Virtual Functions (VFs), each assigned directly to a VM via VFIO/IOMMU. Performance: ~20 Mpps — near-native, no software switching overhead. Limitations: no software-defined networking (limited L2/L3 filtering in NIC), live migration requires VF hot-unplug, fixed number of VFs per NIC.

**Q2: What is SPDK and how does vhost-user-blk improve storage performance?**
**A:** SPDK (Storage Performance Development Kit) provides userspace, poll-mode NVMe drivers — analogous to DPDK for networking. Traditional storage path: guest I/O → virtio → KVM exit → QEMU → kernel block layer → kernel NVMe driver → device. Each step adds latency (context switches, locks, IRQ handling). SPDK path: SPDK binds the NVMe SSD via VFIO (unbinds kernel driver), runs a poll-mode NVMe driver in userspace with dedicated CPU cores. The SPDK vhost target creates a vhost-user-blk socket. QEMU (or cloud-hypervisor) connects the guest's virtio-blk frontend to this socket via shared memory. Guest I/O writes to the virtqueue in shared memory, SPDK polls the virtqueue directly, submits to NVMe hardware, polls for completion, and places results in the used ring. No kernel involvement on the entire data path. Results: 1.5M+ IOPS (vs ~300K kernel), ~7μs latency (vs ~25μs kernel). Trade-off: dedicated CPU cores for polling, NVMe SSD must be unbound from kernel (can't use it for host storage simultaneously).

---

## Summary

- Linux bridge: simple L2 switch, ~5 Gbps, good for small deployments
- OVS: OpenFlow programmable, tunneling support, used in cloud (OpenStack/K8s)
- OVS-DPDK: userspace datapath, 5x faster than kernel OVS, requires dedicated CPUs
- SR-IOV: near-native hardware performance, limited software-defined features
- virtio-blk: simple virtual disk; virtio-scsi: full SCSI; NVMe: multiqueue native
- SPDK + vhost-user-blk: userspace storage, millions of IOPS, sub-10μs latency

---

[Previous: Interrupt Virtualization ←](Chapter_25_Interrupt_Virtualization.md) | [Next: vhost and vhost-user →](Chapter_27_Vhost.md)
