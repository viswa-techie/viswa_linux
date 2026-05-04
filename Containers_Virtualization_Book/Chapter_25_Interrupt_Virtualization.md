# Chapter 25: Interrupt Virtualization and Timer Management

## Learning Goals
- Understand virtual APIC and interrupt injection
- Learn posted interrupts for exit-less interrupt delivery
- Master virtual timer architecture (kvmclock, TSC)
- Know interrupt routing: MSI-X, IOAPIC emulation

---

## 1. Virtual APIC Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Physical system: CPU ← LAPIC ← IOAPIC ← devices       │
  │                                                           │
  │  Virtual system:                                         │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Guest vCPU                               │            │
  │  │   ├── Virtual LAPIC (per vCPU)           │            │
  │  │   │   - In-kernel (KVM) emulation        │            │
  │  │   │   - APICv: hardware-accelerated      │            │
  │  │   │   - APIC page mapped to guest        │            │
  │  │   │                                      │            │
  │  │   └── Interrupt injection:               │            │
  │  │       1. KVM sets interrupt in VMCS      │            │
  │  │       2. On VM entry, CPU delivers it    │            │
  │  │       3. Guest IDT handler runs          │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Virtual IOAPIC (one per VM):                            │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Emulated in KVM kernel module            │            │
  │  │ Routes device interrupts to target vCPU  │            │
  │  │ Redirection table: IRQ → vCPU + vector   │            │
  │  │                                          │            │
  │  │ For virtio: bypassed by MSI-X (no IOAPIC)│            │
  │  │ MSI-X writes directly to LAPIC MMIO addr │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Interrupt injection methods:                            │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. Direct injection (VMCS):              │            │
  │  │    KVM sets VM-entry interrupt info field │            │
  │  │    CPU delivers on next VM entry          │            │
  │  │    Requires VM exit first!                │            │
  │  │                                          │            │
  │  │ 2. Posted Interrupts (APICv):            │            │
  │  │    KVM writes to Posted Interrupt         │            │
  │  │    Descriptor (PID) in memory            │            │
  │  │    CPU checks PID on VM entry AND         │            │
  │  │    delivers without VM exit!              │            │
  │  │    → guest never exits for interrupts    │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Posted Interrupts (APICv)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Posted Interrupts: deliver interrupts to guest          │
  │  WITHOUT a VM exit                                       │
  │                                                           │
  │  Posted Interrupt Descriptor (PID):                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Bits 0-255: Posted Interrupt Requests    │            │
  │  │   (one bit per interrupt vector)         │            │
  │  │                                          │            │
  │  │ Outstanding Notification (ON) bit        │            │
  │  │ Suppress Notification (SN) bit           │            │
  │  │ Notification Vector (NV)                 │            │
  │  │ Notification Destination (NDST)          │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Flow:                                                   │
  │  1. Device sends MSI-X interrupt                        │
  │  2. IOMMU remaps to PID (Posted Interrupt)              │
  │  3. CPU sets bit in PID vector bitmap                   │
  │  4. If guest is RUNNING on that CPU:                    │
  │     → CPU delivers interrupt directly to guest          │
  │     → NO VM exit!                                       │
  │  5. If guest is NOT running (vCPU scheduled out):       │
  │     → Interrupt recorded in PID                         │
  │     → Delivered on next VM entry                        │
  │                                                           │
  │  Without posted interrupts:                              │
  │  Device → IRQ → VM exit → KVM → inject → VM entry      │
  │  (2 context switches per interrupt!)                    │
  │                                                           │
  │  With posted interrupts:                                 │
  │  Device → IOMMU → PID → guest receives directly         │
  │  (0 context switches!)                                   │
  │                                                           │
  │  Requirements: Intel APICv (VT-x + VT-d + posted IRQ)  │
  │  AMD equivalent: AVIC (Advanced Virtual Interrupt Ctrl) │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Timer Virtualization

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Virtual timers:                                         │
  │                                                           │
  │  ┌──────────────┬──────────────────────────────────────┐│
  │  │ Timer        │ Virtualization approach               ││
  │  ├──────────────┼──────────────────────────────────────┤│
  │  │ PIT (8254)   │ Fully emulated in KVM (legacy)       ││
  │  │              │ Generates periodic VM exits           ││
  │  ├──────────────┼──────────────────────────────────────┤│
  │  │ LAPIC timer  │ Emulated in KVM, preemption timer    ││
  │  │              │ used for deadline mode                ││
  │  ├──────────────┼──────────────────────────────────────┤│
  │  │ HPET         │ Emulated in QEMU userspace           ││
  │  │              │ (slow, causes VM exits)               ││
  │  ├──────────────┼──────────────────────────────────────┤│
  │  │ TSC          │ Passed through (hardware counter)    ││
  │  │              │ TSC scaling for migration             ││
  │  │              │ TSC offset per-vCPU in VMCS          ││
  │  ├──────────────┼──────────────────────────────────────┤│
  │  │ kvmclock     │ Paravirtual clock (PV)               ││
  │  │              │ Shared memory page with host          ││
  │  │              │ Guest reads without VM exit           ││
  │  │              │ Handles TSC frequency changes         ││
  │  └──────────────┴──────────────────────────────────────┘│
  │                                                           │
  │  kvmclock (paravirtual clock):                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ KVM writes clock parameters to shared page:          │
  │  │   struct pvclock_vcpu_time_info {                    │
  │  │       u32 version;                                   │
  │  │       u64 tsc_timestamp;                             │
  │  │       u64 system_time;                               │
  │  │       u32 tsc_to_system_mul;                         │
  │  │       s8  tsc_shift;                                 │
  │  │       u8  flags;                                     │
  │  │   };                                                 │
  │  │                                                       │
  │  │ Guest reads: time = system_time +                    │
  │  │   ((rdtsc() - tsc_timestamp) * tsc_to_system_mul)   │
  │  │   >> (32 - tsc_shift)                                │
  │  │                                                       │
  │  │ → No VM exit! (VDSO-compatible)                      │
  │  │ → Handles vCPU migration between host CPUs          │
  │  │   (different TSC frequencies)                        │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. irqfd and ioeventfd

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  irqfd and ioeventfd: decouple I/O from QEMU process    │
  │                                                           │
  │  ioeventfd (guest → host notification):                  │
  │  ┌──────────────────────────────────────────┐            │
  │  │ QEMU: register addr with KVM             │            │
  │  │   ioctl(vmfd, KVM_IOEVENTFD, {           │            │
  │  │       .datamatch = 0,                    │            │
  │  │       .addr = 0xFE003000,  /* MMIO addr */│            │
  │  │       .fd = eventfd(0, 0),               │            │
  │  │       .flags = KVM_IOEVENTFD_FLAG_MMIO   │            │
  │  │   })                                     │            │
  │  │                                          │            │
  │  │ Guest writes to 0xFE003000 →             │            │
  │  │   KVM intercepts (no exit to QEMU!) →    │            │
  │  │   signals eventfd →                      │            │
  │  │   vhost worker thread wakes up           │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  irqfd (host → guest interrupt):                         │
  │  ┌──────────────────────────────────────────┐            │
  │  │ QEMU: register interrupt with KVM         │            │
  │  │   ioctl(vmfd, KVM_IRQFD, {               │            │
  │  │       .fd = eventfd(0, 0),               │            │
  │  │       .gsi = 24,  /* guest IRQ number */ │            │
  │  │   })                                     │            │
  │  │                                          │            │
  │  │ vhost signals eventfd →                  │            │
  │  │   KVM reads eventfd →                    │            │
  │  │   KVM injects IRQ 24 into guest →        │            │
  │  │   (via posted interrupts if available)   │            │
  │  │   NO QEMU involvement!                   │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How do posted interrupts eliminate VM exits for interrupt delivery?**
**A:** Without posted interrupts: when a device generates an interrupt (MSI-X), it causes a VM exit (external interrupt exit reason). KVM's exit handler determines the target vCPU, sets the interrupt information in the VMCS's VM-entry interruption-information field, and does VMRESUME. Total: one VM exit and two context switches per interrupt. With posted interrupts (Intel APICv): KVM allocates a Posted Interrupt Descriptor (PID) per vCPU — a 64-byte structure in memory with a 256-bit vector bitmap and notification fields. The IOMMU's interrupt remapping table is configured to write MSI interrupts directly to the PID instead of generating a physical interrupt. If the target vCPU is currently running (in VMX non-root): the CPU periodically checks the PID (or a notification interrupt triggers processing), sees pending vectors, and delivers them directly to the guest's virtual APIC — no VM exit occurs. If the vCPU is not running: the interrupt is recorded in the PID and delivered on the next VM entry. This is critical for high-throughput I/O (SR-IOV NICs generating millions of interrupts/sec). AMD's equivalent is AVIC (Advanced Virtual Interrupt Controller).

**Q2: What is kvmclock and why is it needed instead of passing through the TSC directly?**
**A:** The TSC (Time Stamp Counter) is a hardware counter incremented every CPU cycle. Problems with passing TSC directly to guests: (1) Different host CPUs may have different TSC frequencies — when a vCPU migrates between host CPUs, the guest sees a TSC jump or rate change, causing time errors. (2) Host CPU frequency scaling changes TSC rate on some older CPUs. (3) TSC may not be synchronized across sockets (older systems). kvmclock is a paravirtual clock: KVM writes clock parameters (TSC offset, multiplier, system time reference) to a shared memory page per-vCPU. The guest reads rdtsc() and applies the formula: `time = system_time + ((rdtsc() - tsc_timestamp) * mul) >> shift`. This requires no VM exit (the page is in guest memory, rdtsc doesn't exit). When a vCPU migrates between host CPUs, KVM updates the shared page with new parameters. The guest's VDSO uses kvmclock, so `clock_gettime()` is fast and accurate even across vCPU migrations. Modern systems with invariant TSC and TSC scaling (VMCS TSC offset/multiplier fields) can often use TSC passthrough directly, but kvmclock remains the default for portability.

---

## Summary

- Virtual LAPIC: per-vCPU, emulated in KVM; APICv enables hardware-assisted APIC
- Posted interrupts: device MSI → IOMMU → PID → guest receives without VM exit
- Virtual IOAPIC: emulated in KVM; modern devices use MSI-X (bypass IOAPIC)
- kvmclock: paravirtual clock via shared memory page — no VM exit for time reads
- TSC virtualization: VMCS TSC offset per-vCPU; TSC scaling for migration
- ioeventfd: guest MMIO write → kernel eventfd → vhost (no QEMU exit)
- irqfd: kernel eventfd → KVM injects guest interrupt (no QEMU involvement)

---

[Previous: Virtio Specification ←](Chapter_24_Virtio_Spec.md) | [Next: Device Passthrough Deep Dive →](Chapter_26_Passthrough_Deep_Dive.md)
