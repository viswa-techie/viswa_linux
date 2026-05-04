# Chapter 32: Memory in Virtualization (KVM, Xen)

## Chapter Overview

Virtualization adds a second layer of memory management between guest VMs and physical hardware. This chapter covers hardware-assisted memory virtualization (EPT/NPT/Stage-2), KVM memory management, memory overcommit techniques (KSM, ballooning), and VFIO/device passthrough.

---

## 32.1 Two-Dimensional Address Translation

```
Without virtualization (native):
  Process VA → MMU (page table walk) → PA → Memory

With virtualization (2D translation):
  Guest VA → Guest MMU → Guest PA (GPA)
                            ↓
  GPA → Hypervisor MMU (EPT/NPT) → Host PA (HPA) → Memory

┌─────────────────────────────────────────────────────────────────┐
│ GUEST VM                                                         │
│                                                                  │
│ Guest Process:       Guest Kernel:                              │
│ ┌──────────┐         ┌─────────────────────┐                   │
│ │ Guest VA │ ───────→│ Guest Page Tables    │                   │
│ │ 0x7FFF.. │         │ (managed by guest OS)│                   │
│ └──────────┘         └──────────┬──────────┘                   │
│                                 │ GPA                           │
│                                 ▼                               │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
┌─────────────────────────────────▼───────────────────────────────┐
│ HYPERVISOR (KVM + hardware)                                      │
│                                                                  │
│ ┌──────────────────────┐                                        │
│ │ EPT / NPT / Stage-2  │  Hardware-assisted 2nd level           │
│ │ Page Tables           │  GPA → HPA translation                │
│ │ (managed by KVM)      │                                        │
│ └──────────┬───────────┘                                        │
│            │ HPA (Host Physical Address)                         │
│            ▼                                                     │
│ ┌──────────────────────┐                                        │
│ │ Physical Memory      │                                        │
│ │ RAM hardware          │                                        │
│ └──────────────────────┘                                        │
└──────────────────────────────────────────────────────────────────┘

Hardware implementations:
  Intel VT-x: EPT (Extended Page Tables)     — CR3 for guest, EPTP for EPT
  AMD-V:      NPT (Nested Page Tables)       — CR3 for guest, nCR3 for NPT
  ARM:        Stage-2 translation            — TTBR0 for guest, VTTBR for stage-2
```

---

## 32.2 EPT/NPT Page Walk Cost

```
4-level guest + 4-level EPT (worst case TLB miss):

Guest page walk needs 4 guest table lookups.
Each guest table lookup requires its OWN EPT walk (4 lookups).
Plus 1 final EPT walk for the actual data page.

Total memory accesses = (4 guest levels × 4 EPT lookups) + 4 final = 20
With P4D (5-level): up to 5 × 5 + 5 = 30!

Step-by-step:
1. Read guest PGD entry:
   Guest PGD GPA → EPT walk (4 reads) → HPA → read PGD entry
   
2. Read guest PUD entry:  
   Guest PUD GPA → EPT walk (4 reads) → HPA → read PUD entry

3. Read guest PMD entry:
   Guest PMD GPA → EPT walk (4 reads) → HPA → read PMD entry

4. Read guest PTE entry:
   Guest PTE GPA → EPT walk (4 reads) → HPA → read PTE entry

5. Access actual data page:
   Data GPA → EPT walk (4 reads) → HPA → access data

Total: 4 + 4 + 4 + 4 + 4 = 20 memory accesses!

Mitigations:
┌────────────────────────────────────────────────────────────────┐
│ Technique          │ Reduces walks by                         │
├────────────────────┼──────────────────────────────────────────┤
│ TLB caching        │ Guest VA → HPA cached directly          │
│ VPID/VMID tags     │ No TLB flush on VM switch               │
│ Guest huge pages   │ 3 guest levels × 4 EPT = 16 accesses   │
│ Host huge pages    │ 4 guest × 3 EPT levels = 15 accesses   │
│ Both huge pages    │ 3 × 3 + 3 = 12 accesses (best)         │
│ Page walk caching  │ Intel: EPT paging structure cache       │
│ 1GB guest+host     │ 2 × 2 + 2 = 6 accesses (optimal)      │
└────────────────────┴──────────────────────────────────────────┘
```

---

## 32.3 KVM Memory Management

```c
/* KVM (Kernel-based Virtual Machine) uses Linux MM as its foundation */

/* Guest memory is backed by host userspace (QEMU) memory:
 * QEMU calls mmap() to allocate guest RAM
 * KVM maps this into EPT page tables
 * Guest PA → slot in KVM memslot → host VA → host PA
 */

/* Memory slot registration: */
struct kvm_userspace_memory_region {
    __u32 slot;           /* Slot identifier */
    __u32 flags;          /* KVM_MEM_LOG_DIRTY_PAGES, etc. */
    __u64 guest_phys_addr;/* Guest physical address start */
    __u64 memory_size;    /* Size of the region */
    __u64 userspace_addr; /* Host userspace VA (QEMU mmap'd) */
};
/* ioctl(kvm_fd, KVM_SET_USER_MEMORY_REGION, &region); */

/* KVM EPT fault handling: */
/* Guest accesses GPA with no EPT mapping:
 * 1. EPT violation → VM exit → KVM handler
 * 2. kvm_mmu_page_fault()
 * 3. Find memslot for GPA
 * 4. Get host page (hva_to_pfn → get_user_pages)
 * 5. Install EPT mapping: GPA → HPA
 * 6. Resume guest execution
 */
```

```
KVM Memory Architecture:

  QEMU (userspace):
  ┌─────────────────────────────────────────────────┐
  │ VM RAM = mmap'd anonymous memory                 │
  │ HVA 0x7F0000000000 ──→ 4GB guest RAM            │
  │                                                   │
  │ KVM_SET_USER_MEMORY_REGION:                      │
  │   slot=0, GPA=0x0, size=4GB, HVA=0x7F00...     │
  └────────────────────────────┬────────────────────┘
                               │
  KVM (kernel):                ▼
  ┌─────────────────────────────────────────────────┐
  │ Memory Slots:                                     │
  │   Slot 0: GPA 0x00000000-0xFFFFFFFF → HVA       │
  │   Slot 1: GPA device MMIO ranges                 │
  │                                                   │
  │ EPT Page Tables:                                  │
  │   GPA → HPA (built on demand via EPT faults)     │
  │                                                   │
  │ For each guest page access:                       │
  │   GPA → find memslot → HVA → get host page →    │
  │   HPA → install in EPT                           │
  └─────────────────────────────────────────────────┘
```

---

## 32.4 Memory Ballooning

```
Ballooning: Dynamic memory reclaim from guest VMs.

How it works:
1. Hypervisor tells balloon driver to "inflate" (claim N pages)
2. Guest balloon driver allocates pages inside the guest
3. These pages are now "used" from guest perspective (can't use them)
4. Balloon driver tells hypervisor which GPAs are claimed
5. Hypervisor unmaps those pages from EPT → physical RAM freed
6. Freed host pages available for other VMs or host use

Inflate (reclaim from guest):
┌──── Guest VM ─────────────────┐   ┌──── Hypervisor ──────────┐
│                               │   │                           │
│ Guest RAM: 4GB                │   │ Host RAM: 16GB            │
│ ┌─────────────────────────┐   │   │ VM1: 4GB, VM2: 4GB       │
│ │ Used by guest apps  2GB │   │   │ Free: 8GB                 │
│ ├─────────────────────────┤   │   │                           │
│ │ Balloon (inflated) 1GB  │   │   │ Need more host RAM!       │
│ ├─────────────────────────┤   │   │ → Tell balloon: inflate   │
│ │ Free in guest       1GB │   │   │ → Guest "uses" 1GB for    │
│ └─────────────────────────┘   │   │   balloon                 │
│                               │   │ → KVM unmaps 1GB EPT      │
│ Guest thinks it has 4GB       │   │ → Host gains 1GB free!    │
│ But 1GB is "wasted" in        │   │                           │
│ balloon                       │   │                           │
└───────────────────────────────┘   └───────────────────────────┘

Deflate (return memory to guest):
  Hypervisor tells balloon to release pages
  Balloon frees pages → guest has more available RAM
  EPT mappings restored on next guest access

Linux driver: drivers/virtio/virtio_balloon.c
```

---

## 32.5 KSM (Kernel Same-page Merging)

```
KSM scans pages across VMs looking for identical content.
Identical pages are merged COW-style (one physical copy, multiple mappings).

Without KSM:                      With KSM:
VM1: [Page A] [Page B]           VM1: [Page A]→─┐  [Page B]
VM2: [Page A'] [Page C]          VM2: [Page A']→─┤  [Page C]
VM3: [Page A''] [Page D]         VM3: [Page A'']→┘  [Page D]
                                        │
Physical: 3 copies of same page   Physical: 1 copy (shared COW)
                                   Saved: 2 pages of RAM!

KSM is especially effective when running many identical VMs:
100 VMs with same OS → kernel text pages identical → massive dedup

┌────────────────────────────────────────────────────────────────┐
│ KSM Process:                                                    │
│ 1. ksmd kernel thread scans MADV_MERGEABLE regions             │
│ 2. Hash each page content                                       │
│ 3. Compare hashes → find duplicates                            │
│ 4. memcmp() to verify (hash collision protection)              │
│ 5. Merge: keep one page, make others COW-map to it            │
│ 6. On write to merged page → COW fault → unshare              │
└────────────────────────────────────────────────────────────────┘

# Enable KSM:
$ echo 1 > /sys/kernel/mm/ksm/run
$ cat /sys/kernel/mm/ksm/pages_shared    # Deduplicated pages
$ cat /sys/kernel/mm/ksm/pages_sharing   # Pages pointing to shared
$ cat /sys/kernel/mm/ksm/pages_unshared  # Scanned but unique

# WARNING: KSM has security implications
# Timing side-channel: write to COW page takes longer → detect
# if another VM has same page → potential information leak.
# Mitigate: Don't use KSM across security domains.
```

---

## 32.6 VFIO and Device Passthrough

```
VFIO (Virtual Function I/O): Pass physical devices directly to VMs.
Device uses IOMMU to DMA directly to guest memory.

┌──── Guest VM ────────────────────────────────────────┐
│                                                       │
│ Guest driver talks to device directly (no emulation) │
│ Near-native I/O performance                           │
│                                                       │
│ Guest VA → Guest PT → GPA                            │
│                          ↓                            │
└──────────────────────────┼────────────────────────────┘
                           │
┌──── KVM + VFIO ──────────▼────────────────────────────┐
│                                                       │
│ IOMMU configured by VFIO:                             │
│   Device IOVA → IOMMU → HPA (host physical)          │
│                                                       │
│ KVM maps:  GPA → HPA (EPT)                           │
│ VFIO maps: GPA → HPA (IOMMU)  ← same HPA!           │
│                                                       │
│ Device DMAs using GPA → IOMMU translates → HPA       │
│ Same physical page the guest expected!                │
│                                                       │
└───────────────────────────────────────────────────────┘

VFIO Memory Pinning:
  Device DMA needs stable physical addresses (can't page out!)
  VFIO pins all guest RAM (get_user_pages + FOLL_LONGTERM)
  Pinned pages → not reclaimable → memory not overcommittable!
  
  4GB guest with VFIO device → 4GB physical RAM PERMANENTLY pinned
  vs. non-VFIO: 4GB virtual, actual RAM = working set (overcommit ok)
```

---

## 32.7 Dirty Page Tracking (Live Migration)

```
Live migration: Move running VM to another host.
Must track which pages change during copy.

Dirty page tracking phases:
┌─────────────────────────────────────────────────────────────────┐
│ Phase 1: Pre-copy (iterative)                                    │
│   1. Mark all guest pages as dirty (must copy everything)       │
│   2. Copy all pages to destination host (4GB → takes time)      │
│   3. While copying: guest still running → some pages dirtied    │
│   4. Track dirty pages (EPT dirty bit / KVM dirty bitmap)       │
│   5. Copy only dirty pages (smaller set)                        │
│   6. Repeat: copy dirty → track new dirty → copy again          │
│   7. Converge: dirty set shrinks each iteration                 │
│                                                                  │
│ Phase 2: Stop-and-copy (brief downtime)                         │
│   8. Pause guest                                                 │
│   9. Copy remaining dirty pages (small set)                     │
│  10. Transfer CPU state, device state                           │
│  11. Resume guest on destination host                            │
│   Downtime: milliseconds (for small final dirty set)            │
└─────────────────────────────────────────────────────────────────┘

KVM dirty page tracking:
  ioctl(vm_fd, KVM_GET_DIRTY_LOG, &dirty_log);
  Returns bitmap: 1 bit per page (1 = dirtied since last query)

  Implementation options:
  1. Write-protect EPT entries → EPT violation on write → mark dirty
  2. Intel PML (Page Modification Logging) → hardware dirty logging
     CPU logs dirty GPAs to a buffer → no EPT violation overhead
```

---

## 32.8 Memory Overcommit in Virtualization

```
Memory overcommit: Allocate more VM RAM than physical host RAM.

Techniques that enable overcommit:
┌──────────────────┬──────────────────────────────────────────────┐
│ Technique        │ How it saves memory                         │
├──────────────────┼──────────────────────────────────────────────┤
│ Demand paging    │ Guest RAM allocated only when touched        │
│ KSM              │ Deduplicate identical pages across VMs       │
│ Ballooning       │ Reclaim unused guest pages dynamically       │
│ Swap             │ Less-used guest pages swapped to host disk   │
│ Compression      │ zswap/zram for compressed guest pages        │
│ Free page hints  │ Guest tells host about free pages (virtio)   │
└──────────────────┴──────────────────────────────────────────────┘

Example: Host with 32GB RAM running VMs totaling 64GB:
  VM1: 16GB (actual use: 8GB)
  VM2: 16GB (actual use: 6GB)  
  VM3: 16GB (actual use: 10GB)
  VM4: 16GB (actual use: 4GB)
  Total allocated: 64GB
  Total actual use: 28GB → fits in 32GB with room to spare!
  
  Risk: if all VMs actually use their full allocation → OOM
  Mitigation: monitoring + resource limits + ballooning
```

---

## Interview Questions

1. **Q: How does EPT (Extended Page Tables) work?**
   A: EPT adds a second level of address translation managed by the hypervisor. Guest page tables translate Guest VA to Guest PA as usual. EPT then translates Guest PA to Host PA. Both walks happen in hardware. On EPT miss, a VM exit occurs and KVM installs the mapping. Total worst-case cost: ~20 memory accesses per TLB miss.

2. **Q: What is memory ballooning?**
   A: A mechanism to dynamically reclaim physical memory from guest VMs. The balloon driver inside the guest allocates pages (inflates), making them unavailable to the guest. The hypervisor unmaps these from EPT, freeing host physical pages for other uses. Deflating returns memory to the guest. No guest awareness needed beyond the balloon driver.

3. **Q: How does live migration use dirty page tracking?**
   A: Pre-copy: all pages copied to destination while guest runs. EPT write-protection or PML (hardware dirty logging) tracks which pages are modified during copy. Only dirty pages are re-copied in subsequent rounds. The dirty set shrinks each iteration. Finally, guest is paused, remaining dirty pages and state transferred, guest resumed on destination. Downtime is typically milliseconds.

4. **Q: What is KSM and what are its security implications?**
   A: KSM (Kernel Same-page Merging) deduplicates identical pages across processes/VMs by merging them COW-style. Security risk: a COW write to a shared page takes measurably longer than a regular write, creating a timing side-channel. An attacker can detect whether their page content matches another VM's page. Mitigation: disable KSM across security boundaries.

---

## Summary

1. **2D translation**: Guest PT + EPT/NPT/Stage-2 = up to 20 memory accesses per TLB miss.
2. **KVM** uses host userspace memory (QEMU mmap) for guest RAM, with EPT built on demand.
3. **Ballooning** reclaims physical RAM from underutilizing guests dynamically.
4. **KSM** deduplicates identical pages across VMs (but has side-channel risks).
5. **VFIO** passes devices directly to VMs via IOMMU, requiring pinned memory.
6. **Dirty page tracking** enables live migration with minimal downtime.
7. **Overcommit** works through demand paging + KSM + ballooning + swap.

---

*Next: [Chapter 33 — Modern Memory Management (Persistent Memory, GPU, CXL)](Chapter_33_Modern_Memory.md)*
