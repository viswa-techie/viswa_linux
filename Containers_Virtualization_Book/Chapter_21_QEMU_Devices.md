# Chapter 21: QEMU and Device Emulation

## Learning Goals
- Understand QEMU architecture and its role alongside KVM
- Learn device emulation vs paravirtualization trade-offs
- Master virtio device architecture and virtqueues
- Know ioeventfd/irqfd for kernel-level virtio acceleration

---

## 1. QEMU Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  QEMU = Quick EMUlator                                   │
  │  Two modes:                                              │
  │  1. Full emulation (no KVM) — emulates entire CPU       │
  │     (TCG: Tiny Code Generator, binary translation)      │
  │  2. KVM accelerated — CPU runs native, QEMU does I/O   │
  │                                                           │
  │  QEMU process structure (one process per VM):            │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ QEMU Process                                  │       │
  │  │  ┌────────────┐  ┌────────────┐              │       │
  │  │  │ Main thread│  │ I/O thread │              │       │
  │  │  │ (event     │  │ (disk ops) │              │       │
  │  │  │  loop)     │  │            │              │       │
  │  │  └────────────┘  └────────────┘              │       │
  │  │  ┌────────────┐  ┌────────────┐              │       │
  │  │  │ vCPU       │  │ vCPU       │              │       │
  │  │  │ thread 0   │  │ thread 1   │              │       │
  │  │  │ KVM_RUN    │  │ KVM_RUN    │              │       │
  │  │  │ loop       │  │ loop       │              │       │
  │  │  └────────────┘  └────────────┘              │       │
  │  │                                               │       │
  │  │  Guest RAM = mmap'd region in QEMU address   │       │
  │  │  space. KVM creates EPT to map this to GPA.  │       │
  │  │                                               │       │
  │  │  QEMU big lock (BQL/QEMU global mutex):      │       │
  │  │  Protects device state from concurrent access│       │
  │  │  vCPU threads release BQL during KVM_RUN     │       │
  │  │  Acquire BQL when handling VM exits           │       │
  │  └──────────────────────────────────────────────┘       │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Device Emulation Models

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Three approaches to VM device I/O:                     │
  │                                                           │
  │  1. Full Emulation (e.g., e1000 NIC, IDE disk):          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Guest driver → I/O instruction → VM exit │            │
  │  │ → KVM returns to QEMU                    │            │
  │  │ → QEMU emulates device register access   │            │
  │  │ → QEMU updates device state              │            │
  │  │ → KVM injects interrupt to guest         │            │
  │  │                                          │            │
  │  │ Advantage: unmodified guest drivers      │            │
  │  │ Disadvantage: SLOW (many VM exits)       │            │
  │  │ Each register access = 1 VM exit!        │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  2. Paravirtualization (virtio):                         │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Guest virtio driver → shared memory ring │            │
  │  │ → single notification (1 VM exit)        │            │
  │  │ → host processes entire batch            │            │
  │  │                                          │            │
  │  │ Advantage: HIGH performance (batched I/O)│            │
  │  │ Disadvantage: needs virtio guest driver  │            │
  │  │ (available in Linux, Windows, FreeBSD)   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  3. Device Passthrough (VFIO):                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Guest driver → real hardware (via IOMMU) │            │
  │  │ → NO VM exits for data path              │            │
  │  │ → native hardware performance             │            │
  │  │                                          │            │
  │  │ Advantage: NATIVE performance            │            │
  │  │ Disadvantage: device dedicated to one VM │            │
  │  │ Need IOMMU (VT-d/AMD-Vi)                │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Performance comparison (network throughput):            │
  │  e1000 emulation:  ~1 Gbps    (many exits)              │
  │  virtio-net:       ~10 Gbps   (batched, fewer exits)    │
  │  VFIO passthrough: ~25+ Gbps  (zero exits, native)     │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Virtio Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  virtio: standard paravirtualized I/O framework          │
  │  OASIS standard (virtio 1.0, 1.1, 1.2)                  │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Guest                                    │            │
  │  │ ┌────────────────────────────────────┐   │            │
  │  │ │ virtio driver (e.g., virtio-net)   │   │            │
  │  │ │ → places buffers in virtqueue      │   │            │
  │  │ │ → kicks device (MMIO write)        │   │            │
  │  │ └──────────────┬─────────────────────┘   │            │
  │  │                │                          │            │
  │  │    ┌───────────▼────────────┐             │            │
  │  │    │ Virtqueue (shared mem) │             │            │
  │  │    │ ┌──────┐ ┌──────────┐ │             │            │
  │  │    │ │Desc  │ │ Avail    │ │             │            │
  │  │    │ │Table │ │ Ring     │ │             │            │
  │  │    │ │      │ │          │ │             │            │
  │  │    │ │ addr │ │ idx: 5   │ │             │            │
  │  │    │ │ len  │ │ ring[]:  │ │             │            │
  │  │    │ │ flags│ │ [3,4,5]  │ │             │            │
  │  │    │ └──────┘ └──────────┘ │             │            │
  │  │    │ ┌──────────┐          │             │            │
  │  │    │ │ Used     │          │             │            │
  │  │    │ │ Ring     │          │             │            │
  │  │    │ │ idx: 3   │          │             │            │
  │  │    │ │ ring[]:  │          │             │            │
  │  │    │ │ [0,1,2]  │          │             │            │
  │  │    │ └──────────┘          │             │            │
  │  │    └───────────────────────┘             │            │
  │  └──────────────────┬───────────────────────┘            │
  │                     │ notification                        │
  │  ┌──────────────────▼───────────────────────┐            │
  │  │ Host (QEMU or vhost)                     │            │
  │  │ ┌────────────────────────────────────┐   │            │
  │  │ │ virtio backend                     │   │            │
  │  │ │ → reads available buffers          │   │            │
  │  │ │ → processes I/O (disk read, packet)│   │            │
  │  │ │ → places results in used ring      │   │            │
  │  │ │ → injects interrupt to guest       │   │            │
  │  │ └────────────────────────────────────┘   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  virtio devices:                                         │
  │  virtio-net (type 1), virtio-blk (type 2),              │
  │  virtio-console (type 3), virtio-rng (type 4),          │
  │  virtio-balloon (type 5), virtio-scsi (type 8),         │
  │  virtio-gpu (type 16), virtio-fs (type 26)              │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. vhost — Kernel-Level Virtio Backend

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Problem: QEMU-based virtio backend                     │
  │  Guest → VM exit → KVM → QEMU (userspace) → process    │
  │  Two context switches per I/O operation!                │
  │                                                           │
  │  Solution: vhost — move backend INTO the kernel         │
  │  Guest → VM exit → KVM → vhost (kernel) → process      │
  │  Only ONE context switch!                                │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Without vhost:                           │            │
  │  │ Guest → ioeventfd → [user] QEMU         │            │
  │  │                     → read/write data    │            │
  │  │                     → irqfd → Guest IRQ  │            │
  │  │                                          │            │
  │  │ With vhost-net:                          │            │
  │  │ Guest → ioeventfd → [kernel] vhost       │            │
  │  │                     → tap device I/O     │            │
  │  │                     → irqfd → Guest IRQ  │            │
  │  │ (QEMU not involved in data path!)        │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  ioeventfd: guest writes to MMIO address →              │
  │    kernel delivers eventfd to vhost thread              │
  │    (no VM exit to QEMU!)                                │
  │                                                           │
  │  irqfd: kernel injects interrupt into guest             │
  │    via posted interrupt (no VM exit!)                    │
  │                                                           │
  │  vhost variants:                                         │
  │  ┌──────────────┬──────────────────────────────────┐    │
  │  │ vhost-net    │ Kernel module, virtio-net backend │    │
  │  │ vhost-scsi   │ Kernel module, virtio-scsi backend│    │
  │  │ vhost-user   │ Userspace backend via Unix socket │    │
  │  │              │ (e.g., DPDK, SPDK)                │    │
  │  │ vhost-vdpa   │ Vendor NIC hardware acceleration  │    │
  │  └──────────────┴──────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Common QEMU Command Line

```bash
# Start a VM with KVM acceleration
qemu-system-x86_64 \
    -enable-kvm \                      # Use KVM, not TCG
    -cpu host \                        # Expose host CPU features
    -smp 4 \                           # 4 vCPUs
    -m 4G \                            # 4GB RAM
    -drive file=disk.qcow2,if=virtio \ # virtio-blk disk
    -netdev user,id=net0 \             # User-mode networking
    -device virtio-net-pci,netdev=net0 \ # virtio NIC
    -display none \                    # No graphics
    -serial stdio \                    # Console on terminal
    -monitor unix:/tmp/qemu.sock,server,nowait  # QMP socket
```

---

## Interview Questions

**Q1: Compare emulated, paravirtualized (virtio), and passthrough device models.**
**A:** **Emulated** (e.g., e1000 NIC): QEMU models every register of a real hardware device. Guest uses unmodified drivers. Every register access by the guest causes a VM exit → KVM returns to QEMU → QEMU emulates the register → result returned to guest. For a single packet, dozens of register accesses may be needed. Performance: ~1 Gbps networking. Use case: legacy OS without virtio drivers. **Paravirtualized (virtio)**: Guest uses a special driver that knows it's in a VM. Guest places I/O requests in a shared memory ring (virtqueue descriptor table + available ring), then writes to a single MMIO address to notify the host. The host processes all pending requests in batch, places results in the used ring, and injects one interrupt. One notification per batch instead of one per register. Performance: ~10 Gbps. Use case: modern guests (Linux, Windows with virtio drivers). **Passthrough (VFIO)**: A physical device is assigned directly to the guest via IOMMU (VT-d). Guest accesses the real hardware — no VM exits on the data path (DMA goes directly between device and guest memory via IOMMU). Performance: native (~25+ Gbps NIC). Disadvantages: device dedicated to one VM (no sharing unless SR-IOV), limited migration support (device state must be saved).

**Q2: How does vhost improve virtio performance, and what are ioeventfd/irqfd?**
**A:** Standard virtio: guest notifies host by writing to an MMIO address → VM exit → KVM returns to QEMU userspace → QEMU reads virtqueue → QEMU performs I/O (e.g., writes to tap device) → QEMU injects interrupt via ioctl. This involves two kernel-user context switches per I/O. vhost moves the virtio backend into the kernel: a kernel thread (`vhost-<pid>`) directly processes the virtqueue and performs I/O (e.g., reads/writes to the tap device). QEMU is only involved in setup, not the data path. Two mechanisms enable this: (1) **ioeventfd**: QEMU registers an eventfd with KVM for a specific MMIO address. When the guest writes to that address, KVM signals the eventfd directly — which wakes the vhost kernel thread without returning to QEMU userspace. (2) **irqfd**: QEMU registers an eventfd with KVM for interrupt injection. When vhost signals the eventfd, KVM injects the interrupt into the guest via posted interrupts — no VM exit or userspace involvement. Result: the entire data path (guest → ioeventfd → vhost kernel thread → tap → irqfd → guest interrupt) stays in kernel space. This roughly doubles virtio-net throughput compared to QEMU-based backend.

---

## Summary

- QEMU: VM manager process; one process per VM with vCPU threads + I/O threads
- Three device models: emulated (slow, compatible), virtio (fast, modified drivers), passthrough (native, exclusive)
- Virtio: shared memory virtqueues (descriptor table + avail ring + used ring), batched I/O
- vhost: kernel-level virtio backend, eliminates QEMU from data path
- ioeventfd: guest MMIO write → kernel eventfd (no userspace exit)
- irqfd: kernel eventfd → guest interrupt injection (no userspace involvement)
- QEMU guests with KVM: CPU runs native, only I/O involves emulation

---

[Previous: KVM Architecture ←](Chapter_20_KVM_Architecture.md) | [Next: Memory Virtualization →](Chapter_22_Memory_Virtualization.md)
