# Chapter 22: Memory Virtualization

## Learning Goals
- Understand guest memory management with EPT/NPT
- Learn KSM (Kernel Same-page Merging) for memory deduplication
- Master memory ballooning for dynamic memory adjustment
- Know dirty page tracking for live migration

---

## 1. Guest Memory Layout

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  QEMU process memory layout:                             │
  │                                                           │
  │  QEMU address space                                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 0x000000000000 - text/data/bss (QEMU)   │            │
  │  │ ...                                      │            │
  │  │ 0x7f0000000000 - Guest RAM (mmap'd)      │            │
  │  │ ┌──────────────────────────────────────┐ │            │
  │  │ │ Guest physical memory (e.g., 4GB)   │ │            │
  │  │ │ 0x00000000 - 0xFFFFFFFF (GPA)       │ │            │
  │  │ │                                      │ │            │
  │  │ │ Guest kernel, processes, page cache  │ │            │
  │  │ │ all within this mmap'd region        │ │            │
  │  │ └──────────────────────────────────────┘ │            │
  │  │ ...                                      │            │
  │  │ Stack, heap (QEMU own usage)             │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  KVM memory slots:                                       │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Slot 0: GPA 0x00000000 - 0x0009FFFF      │            │
  │  │         → HVA 0x7f0000000000 (640KB)     │            │
  │  │         (low memory)                      │            │
  │  │                                           │            │
  │  │ Slot 1: GPA 0x00100000 - 0xBFFFFFFF      │            │
  │  │         → HVA 0x7f0000100000 (< 3GB)     │            │
  │  │         (main memory below 4GB)           │            │
  │  │                                           │            │
  │  │ Slot 2: GPA 0x100000000 - 0x13FFFFFFF    │            │
  │  │         → HVA 0x7f00C0000000 (above 4GB) │            │
  │  │         (high memory for large VMs)       │            │
  │  │                                           │            │
  │  │ Slot 3: GPA 0xFEC00000 - 0xFECFFFFF      │            │
  │  │         → MMIO (APIC, IOAPIC)            │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  EPT maps: GPA → HPA (physical RAM)                     │
  │  Backed by: QEMU mmap → host page tables → HPA         │
  │  Demand paging: EPT entries populated on first access   │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. KSM (Kernel Same-page Merging)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  KSM: find and merge identical pages across VMs         │
  │                                                           │
  │  Before KSM:                                             │
  │  VM1 RAM: [page A] [page B] [page C]                    │
  │  VM2 RAM: [page A] [page D] [page C]                    │
  │  Total: 6 pages in physical RAM                          │
  │                                                           │
  │  After KSM:                                              │
  │  VM1 RAM: [page A]──┐  [page B]  [page C]──┐           │
  │                      ▼                      ▼            │
  │                 [shared A]           [shared C]          │
  │                      ▲                      ▲            │
  │  VM2 RAM: [page A]──┘  [page D]  [page C]──┘           │
  │  Total: 4 pages (saved 2 pages / 33%)                   │
  │                                                           │
  │  How KSM works:                                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. ksmd kernel thread scans memory       │            │
  │  │ 2. Computes hash of each page            │            │
  │  │ 3. Compares with red-black tree of known │            │
  │  │    page hashes                            │            │
  │  │ 4. If identical: merge (point both PTEs  │            │
  │  │    to same physical page)                 │            │
  │  │ 5. Mark merged page as COW (copy on write)│            │
  │  │ 6. If either VM writes → COW triggers    │            │
  │  │    → new page allocated → no corruption   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  QEMU enables KSM with:                                 │
  │  madvise(addr, len, MADV_MERGEABLE);                     │
  │                                                           │
  │  Config: /sys/kernel/mm/ksm/                             │
  │  run = 1 (enable), pages_to_scan, sleep_millisecs       │
  │                                                           │
  │  Security note: KSM can leak info via timing side       │
  │  channels (COW page fault timing differs for shared     │
  │  vs unshared pages). Disabled by default in some        │
  │  security-sensitive environments.                        │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Memory Ballooning

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Balloon: dynamically adjust VM's usable memory         │
  │  without rebooting                                       │
  │                                                           │
  │  Inflate balloon (take memory from guest):               │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Guest: 4GB allocated                     │            │
  │  │ ┌──────────────────────────────────────┐ │            │
  │  │ │ Guest usable  │ Balloon (1GB)        │ │            │
  │  │ │ 3GB           │ (pages pinned,       │ │            │
  │  │ │               │  returned to host)   │ │            │
  │  │ └──────────────────────────────────────┘ │            │
  │  │                                          │            │
  │  │ 1. Host tells balloon driver: inflate 1G │            │
  │  │ 2. Balloon driver allocates 1GB in guest │            │
  │  │ 3. Tells host which guest pages were     │            │
  │  │    allocated (via virtio-balloon)        │            │
  │  │ 4. Host unmaps those pages from EPT      │            │
  │  │ 5. Host can use that physical memory     │            │
  │  │    for other VMs or host processes       │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Deflate balloon (give memory back to guest):            │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. Host tells balloon: deflate 512M      │            │
  │  │ 2. Balloon driver frees pages in guest   │            │
  │  │ 3. Guest can use those pages again       │            │
  │  │ 4. EPT entries re-populated on access    │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Use cases:                                              │
  │  - Memory overcommit: 40GB total RAM,                   │
  │    10 VMs with "4GB" each (balloon unused)              │
  │  - Dynamic scaling: give more RAM to busy VMs           │
  │  - Live migration: shrink VM before migrating           │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Dirty Page Tracking (for Live Migration)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Live migration: move running VM between hosts           │
  │  VM keeps running during migration!                      │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Phase 1: Pre-copy (iterative)            │            │
  │  │ 1. Copy ALL guest memory to destination  │            │
  │  │ 2. While copying, guest continues running│            │
  │  │ 3. Track DIRTY pages (modified during    │            │
  │  │    copy) using EPT dirty bits            │            │
  │  │ 4. Copy only dirty pages                 │            │
  │  │ 5. Repeat until dirty set is small       │            │
  │  │                                          │            │
  │  │ Phase 2: Stop-and-copy                   │            │
  │  │ 1. Pause guest (brief downtime: ms)      │            │
  │  │ 2. Copy remaining dirty pages            │            │
  │  │ 3. Copy vCPU state (VMCS registers)      │            │
  │  │ 4. Copy device state                     │            │
  │  │ 5. Resume guest on destination           │            │
  │  │                                          │            │
  │  │ Typical downtime: 10-100ms               │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  KVM dirty tracking:                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ KVM_GET_DIRTY_LOG ioctl                  │            │
  │  │ - Returns bitmap of dirty pages since    │            │
  │  │   last call                              │            │
  │  │ - KVM uses EPT access/dirty bits         │            │
  │  │ - Or PML (Page Modification Logging)     │            │
  │  │   which logs dirty GPAs to a buffer      │            │
  │  │   without VM exits                       │            │
  │  │                                          │            │
  │  │ PML (Page Modification Logging):         │            │
  │  │ - Hardware feature (Intel)               │            │
  │  │ - CPU logs dirty GPA to 512-entry log    │            │
  │  │ - VM exit only when log is full          │            │
  │  │ - Much fewer exits than EPT dirty bits   │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does KSM work and what are its security implications?**
**A:** KSM (Kernel Same-page Merging) is a kernel thread (ksmd) that scans memory pages marked with `MADV_MERGEABLE` (QEMU marks guest RAM this way). It computes hashes of pages and stores them in a red-black tree. When two pages have identical content, KSM merges them: both page table entries point to a single physical page, marked Copy-on-Write (COW). If either VM writes to the shared page, a page fault triggers COW — a new physical page is allocated and the write proceeds on the copy, maintaining isolation. Benefits: significant memory savings when running multiple VMs with the same OS (shared kernel text, libraries, zero pages). A host with 64GB RAM can run more VMs than physical memory allows. Security concern: KSM enables timing side-channel attacks. An attacker can detect whether a specific page content exists in another VM by measuring COW page fault latency: accessing a shared page is fast (no COW), while writing to it is slow (COW triggers new allocation). This can leak information about other VMs' memory contents. CVE-2014-0049 and similar attacks demonstrated this. Mitigations: disable KSM for security-sensitive VMs, use `prctl(PR_SET_MEMORY_MERGE, 0)` to opt out, or use confidential computing (SEV/TDX) where memory is encrypted per-VM.

**Q2: Describe the live migration process and how dirty page tracking works.**
**A:** Live migration moves a running VM between physical hosts with minimal downtime. **Pre-copy phase**: (1) QEMU enables KVM dirty page tracking (`KVM_GET_DIRTY_LOG`). (2) Copies all guest memory pages to the destination host over the network. (3) While copying, the guest continues running and modifying pages. KVM tracks which pages become dirty using EPT access/dirty bits (or PML — Page Modification Logging, a hardware feature where the CPU logs dirty GPAs to a 512-entry buffer, requiring VM exits only when the buffer fills). (4) After the initial copy, QEMU retrieves the dirty bitmap and copies only dirty pages to the destination. (5) This repeats iteratively — each round copies fewer pages as the dirty rate converges. **Stop-and-copy phase**: When the dirty set is small enough (or a convergence threshold is met), QEMU pauses the guest, copies the remaining dirty pages, transfers vCPU state (all VMCS fields: registers, VMCS control fields, APIC state) and device state (virtio queue positions, NIC MAC, disk state), then resumes the VM on the destination. Typical downtime: 10-100ms. Write-intensive workloads challenge convergence — pages are re-dirtied faster than they're copied. Solutions: auto-converge (throttle guest CPU to reduce dirty rate), post-copy (start guest on destination immediately, fetch pages on demand via userfaultfd).

---

## Summary

- Guest memory: QEMU mmap'd region → KVM EPT maps GPA to HPA, demand-paged
- KVM memory slots: map GPA ranges to host virtual addresses (QEMU mmap regions)
- KSM: kernel thread merges identical pages across VMs, COW on write (security: timing side-channels)
- Ballooning: virtio-balloon driver inflates/deflates to dynamically adjust guest memory
- Dirty tracking: EPT dirty bits or PML hardware log; used for live migration
- Live migration: pre-copy (iterative dirty page copy) → stop-and-copy (brief pause) → resume on destination
- PML (Page Modification Logging): hardware dirty tracking with fewer VM exits

---

[Previous: QEMU and Device Emulation ←](Chapter_21_QEMU_Devices.md) | [Next: I/O Virtualization →](Chapter_23_IO_Virtualization.md)
