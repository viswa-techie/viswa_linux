# Chapter 31: Linux vs Other OS Memory Management

## Chapter Overview

This chapter provides detailed comparisons of memory management across Linux, Windows, macOS/iOS, FreeBSD, Android, and RTOS systems. Understanding differences helps in porting, debugging cross-platform issues, and interview preparation.

---

## 31.1 Architecture Comparison

```
┌────────────────┬───────────────┬───────────────┬───────────────┬───────────────┐
│ Feature        │ Linux         │ Windows       │ macOS/iOS     │ FreeBSD       │
├────────────────┼───────────────┼───────────────┼───────────────┼───────────────┤
│ Page size      │ 4KB (default) │ 4KB           │ 16KB (ARM)    │ 4KB           │
│                │ ARM64: 4/16/64│               │ 4KB (Intel)   │               │
│ VA bits        │ 48 or 57      │ 48            │ 47 (user)     │ 48            │
│ Page table     │ 4/5-level     │ 4-level       │ 4-level       │ 4-level       │
│ levels         │ radix tree    │               │               │               │
│ Page allocator │ Buddy system  │ PFN database  │ Free queue    │ Buddy-like    │
│ Slab allocator │ SLUB          │ Pool allocator│ Zone allocator│ UMA           │
│ Overcommit     │ Yes (tunable) │ Yes (commit   │ No (macOS)    │ Yes           │
│                │               │ charge)       │ Yes (iOS kill)│               │
│ Swap           │ Partition/file│ Pagefile      │ Swap file     │ Partition     │
│ OOM handling   │ OOM killer    │ Hard commit   │ Jetsam (iOS)  │ OOM killer    │
│                │               │ limit         │               │               │
│ Huge pages     │ THP + hugetlb │ Large pages   │ Superpage     │ Superpage     │
│ MM lock        │ mmap_lock     │ VAD tree lock │ VM map lock   │ VM map lock   │
│ VMA structure  │ vm_area_struct│ VAD (Virtual  │ vm_map_entry  │ vm_map_entry  │
│                │ + maple tree  │ Address       │               │               │
│                │               │ Descriptor)   │               │               │
│ Page reclaim   │ LRU/MGLRU    │ Working set   │ FIFO +        │ LRU (pageout) │
│                │ kswapd        │ trimming      │ compression   │ pagedaemon    │
│ ASLR           │ Yes (strong)  │ Yes           │ Yes (strong)  │ Yes           │
│ KPTI           │ Yes           │ KVA Shadow    │ Yes           │ Yes           │
└────────────────┴───────────────┴───────────────┴───────────────┴───────────────┘
```

---

## 31.2 Linux vs Windows Memory Management

```
Virtual Address Space Layout:
─────────────────────────────
Linux (x86_64):                     Windows (x86_64):
┌────────────────────┐ 0xFFFF...   ┌────────────────────┐ 0xFFFF...
│ Kernel space       │ (128TB)     │ Kernel space        │ (128TB)
│ Direct map, vmalloc│             │ HAL, kernel, drivers│
│ modules, text      │             │ paged pool, nonpaged│
├────────────────────┤             ├────────────────────┤
│ Canonical gap      │             │ (gap)              │
├────────────────────┤             ├────────────────────┤
│ User space         │ (128TB)     │ User space          │ (128TB)
│ stack,heap,mmap    │             │ stack,heap,DLLs    │
└────────────────────┘ 0x0000...   └────────────────────┘ 0x0000...

Key Differences:
┌──────────────────┬──────────────────────┬──────────────────────────┐
│ Aspect           │ Linux                │ Windows                  │
├──────────────────┼──────────────────────┼──────────────────────────┤
│ Memory model     │ Flat, overcommit     │ Commit charge model      │
│                  │ malloc rarely fails  │ VirtualAlloc can fail    │
│                  │                      │ when commit limit reached│
│                  │                      │                          │
│ Allocation API   │ mmap, brk            │ VirtualAlloc,            │
│                  │                      │ VirtualAllocEx           │
│                  │                      │                          │
│ Working set      │ LRU + MGLRU         │ Working set trimmer      │
│ management       │ (global LRU scan)   │ (per-process WS limits)  │
│                  │                      │                          │
│ Shared memory    │ shmem/tmpfs, mmap    │ Section objects          │
│                  │ MAP_SHARED           │ (Memory-mapped files)    │
│                  │                      │                          │
│ Kernel pool      │ SLUB (unified)       │ Paged pool + NonPaged    │
│                  │                      │ pool (separate)          │
│                  │                      │                          │
│ Page replacement │ Global LRU           │ Per-process working set  │
│                  │ (scan all pages)     │ + standby/modified lists │
│                  │                      │                          │
│ Swap             │ Partition or file    │ Pagefile.sys             │
│                  │ Multiple swap areas  │ Multiple pagefiles       │
│                  │                      │                          │
│ Driver DMA       │ DMA API + IOMMU     │ WDF DMA, MDL             │
│                  │ dma_alloc_coherent   │ (Memory Descriptor List) │
│                  │                      │                          │
│ Memory debugging │ KASAN, kmemleak     │ Driver Verifier,         │
│                  │ KFENCE               │ Application Verifier     │
└──────────────────┴──────────────────────┴──────────────────────────┘

Windows Memory States:
┌──────────┬──────────────────────────────────────────────────────────┐
│ State    │ Description                                              │
├──────────┼──────────────────────────────────────────────────────────┤
│ Free     │ Not committed, not reserved                             │
│ Reserved │ VA reserved but no physical/pagefile backing            │
│ Committed│ Physical or pagefile backing guaranteed (commit charge) │
│          │ Pages may be: Active (in RAM), Standby (was active),    │
│          │ Modified (dirty standby), or Paged out (in pagefile)    │
└──────────┴──────────────────────────────────────────────────────────┘
Linux doesn't have "reserved" state — mmap creates VMA immediately.
Linux overcommit means commit may succeed even without backing.
```

---

## 31.3 Linux vs macOS/iOS Memory Management

```
┌──────────────────┬──────────────────────┬──────────────────────────┐
│ Aspect           │ Linux                │ macOS / iOS              │
├──────────────────┼──────────────────────┼──────────────────────────┤
│ Kernel           │ Monolithic (Linux)   │ Hybrid (XNU = Mach + BSD)│
│                  │                      │ VM from Mach microkernel │
│                  │                      │                          │
│ VM heritage      │ Original Linux MM    │ Mach VM (CMU)            │
│                  │                      │                          │
│ Page size ARM    │ 4KB/16KB/64KB        │ 16KB (Apple Silicon)     │
│                  │                      │                          │
│ Memory pressure  │ PSI + kswapd         │ Memory pressure events   │
│  notification    │ /proc/pressure/memory│ dispatch_source+         │
│                  │                      │ os_proc_available_memory │
│                  │                      │                          │
│ iOS Jetsam       │ N/A (OOM killer)     │ memorystatus: ranked     │
│                  │                      │ kill by priority band    │
│                  │                      │ (foreground protected)   │
│                  │                      │                          │
│ Compressed mem   │ zswap/zram (optional)│ Built-in compressor      │
│                  │                      │ (always active on macOS) │
│                  │                      │ Compressed before swap   │
│                  │                      │                          │
│ Kernel VA        │ PAGE_OFFSET map      │ Separate kernel map      │
│                  │                      │ (Mach zone allocator)    │
│                  │                      │                          │
│ Allocator        │ SLUB + buddy         │ Zone allocator (Mach)    │
│                  │                      │ kalloc (kernel malloc)   │
│                  │                      │                          │
│ Security         │ KPTI, KASLR, NX      │ KTRR, PAC, PPL, KASLR   │
│ (ARM)            │ PAN, MTE             │ iBoot chain of trust     │
│                  │                      │                          │
│ Shared libs      │ mmap + page cache    │ dyld shared cache        │
│                  │                      │ (single shared mapping)  │
└──────────────────┴──────────────────────┴──────────────────────────┘

macOS Memory Compression:
┌────────────────────────────────────────────────────────────┐
│ Under memory pressure:                                      │
│                                                            │
│ Linux: Page → LRU inactive → swap to disk (slow)          │
│                                                            │
│ macOS: Page → Compressor (in-RAM, fast)                   │
│        If still pressured → compressed pages → swap file  │
│        Decompression: ~10× faster than disk read          │
│                                                            │
│ macOS always tries compression before swap.                │
│ Linux zswap is similar but optional, not default.          │
└────────────────────────────────────────────────────────────┘
```

---

## 31.4 Linux vs Android Memory Management

```
Android = Linux kernel + Android-specific memory extensions

┌──────────────────┬──────────────────────┬──────────────────────────┐
│ Aspect           │ Linux (desktop/server)│ Android (mobile)        │
├──────────────────┼──────────────────────┼──────────────────────────┤
│ Kernel           │ Standard Linux        │ Linux + Android patches │
│                  │                      │                          │
│ Low memory       │ OOM killer           │ Low Memory Killer Daemon │
│ handling         │ (kernel, reactive)    │ (LMKD, proactive)       │
│                  │                      │ + OOM killer as fallback │
│                  │                      │                          │
│ Memory pressure  │ PSI, watermarks      │ vmpressure events +      │
│                  │                      │ PSI (Android 12+)        │
│                  │                      │                          │
│ Swap             │ Disk partition/file  │ zram (compressed RAM)    │
│                  │ Usually SSD/HDD      │ No disk swap (flash wear)│
│                  │                      │                          │
│ Process priority │ OOM score            │ OOM score adj by app     │
│                  │                      │ lifecycle state:         │
│                  │                      │ Foreground → Cached      │
│                  │                      │                          │
│ Buffer sharing   │ dma-buf              │ dma-buf + ION (legacy)   │
│                  │                      │ + Gralloc (graphics)    │
│                  │                      │                          │
│ Memory debugging │ KASAN, kmemleak     │ HWASan, MTE (Pixel 8+)  │
│                  │                      │ malloc_debug, ASan       │
│                  │                      │                          │
│ App memory model │ Process isolation    │ Dalvik/ART VM heap       │
│                  │                      │ + Native heap (NDK)     │
│                  │                      │ + Graphics buffers      │
│                  │                      │                          │
│ Shared memory    │ shmem, mmap         │ Ashmem (legacy)          │
│                  │                      │ memfd (modern)           │
│                  │                      │                          │
│ Memory reporting │ /proc/meminfo        │ dumpsys meminfo          │
│                  │ smaps                │ procrank, showmap        │
└──────────────────┴──────────────────────┴──────────────────────────┘

Android LMKD (Low Memory Killer Daemon):
┌────────────────────────────────────────────────────────────────────┐
│ Monitors memory pressure via PSI and vmpressure.                  │
│ Kills apps based on priority (cached → services → perceptible):  │
│                                                                    │
│ Priority bands (oom_score_adj):                                    │
│  0     = Foreground (NEVER kill — user is looking at it)          │
│  100   = Visible (partially visible activity)                      │
│  200   = Perceptible (playing music, etc.)                        │
│  700   = Previous (last used app)                                  │
│  900   = Cached (background, killable)                            │
│  1000  = Empty (no components, first to die)                      │
│                                                                    │
│ LMKD kills proactively before kernel OOM → smoother UX.          │
└────────────────────────────────────────────────────────────────────┘
```

---

## 31.5 Linux vs RTOS Memory Management

```
┌──────────────────┬──────────────────────┬──────────────────────────┐
│ Aspect           │ Linux (GPOS)         │ RTOS (FreeRTOS, Zephyr)  │
├──────────────────┼──────────────────────┼──────────────────────────┤
│ MMU              │ Required             │ Optional (many RTOS      │
│                  │                      │ run without MMU)         │
│                  │                      │                          │
│ Virtual memory   │ Full VM (per-process)│ Often flat physical      │
│                  │                      │ addressing (no VM)       │
│                  │                      │                          │
│ Address space    │ Isolated per-process │ Shared (all tasks in     │
│                  │                      │ same address space)      │
│                  │                      │                          │
│ Page faults      │ Normal (demand paging)│ Fatal error (no paging) │
│                  │                      │                          │
│ Dynamic alloc    │ malloc, mmap, kmalloc│ Static or pool-based     │
│                  │ (with GC/reclaim)    │ pvPortMalloc (FreeRTOS)  │
│                  │                      │ k_malloc (Zephyr)        │
│                  │                      │                          │
│ Overhead         │ Page tables, struct  │ Minimal (no page tables, │
│                  │ page, VMA trees      │ no struct page)          │
│                  │                      │                          │
│ Determinism      │ Non-deterministic    │ Deterministic (bounded   │
│                  │ (reclaim, compaction)│ allocation time)         │
│                  │                      │                          │
│ Swap             │ Yes (swap partition) │ No (no disk typically)   │
│                  │                      │                          │
│ Memory protection│ Full (per-page perms)│ MPU (limited regions) or │
│                  │ Ring 0/3 separation  │ none                     │
│                  │                      │                          │
│ Typical RAM      │ GBs-TBs             │ KBs-MBs                  │
│                  │                      │                          │
│ Memory model     │ Overcommit + demand  │ Static allocation        │
│                  │ paging               │ (known at compile time)  │
└──────────────────┴──────────────────────┴──────────────────────────┘

RTOS Memory Pattern:
┌────────────────────────────────────────────────────────┐
│ Compile-time allocation (preferred):                    │
│   static uint8_t buffer[1024];                         │
│   → Known size, no runtime failure                     │
│                                                        │
│ Pool-based allocation:                                  │
│   Create fixed-size pools at startup                   │
│   Allocate/free from pool (O(1), deterministic)        │
│   → No fragmentation (fixed-size objects)              │
│                                                        │
│ Dynamic allocation (discouraged):                       │
│   Heap fragmentation → failure in long-running systems │
│   Non-deterministic allocation time                    │
│   → Safety-critical: forbidden by MISRA-C              │
└────────────────────────────────────────────────────────┘
```

---

## 31.6 Memory Management in Virtualization

```
Virtualization adds another layer of address translation:

┌─ Guest VM ─────────────────────────────────────────────────┐
│                                                             │
│  Guest Process:  Guest VA → Guest Page Table → Guest PA    │
│                  (managed by guest OS kernel)                │
│                                                             │
└─────────────────────────────┬───────────────────────────────┘
                              │ Guest PA (= HPA before EPT)
                              │ Now: Guest PA is GPA
┌─ Hypervisor ────────────────▼───────────────────────────────┐
│                                                             │
│  GPA → EPT/Stage2 Page Table → Host PA (HPA)              │
│  (managed by hypervisor: KVM, Xen, Hyper-V, VMware)       │
│                                                             │
│  Memory technologies:                                       │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ EPT (Intel):     Extended Page Tables                │  │
│  │ NPT (AMD):       Nested Page Tables                  │  │
│  │ Stage-2 (ARM):   EL1 → EL2 translation              │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Memory optimization:                                       │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Ballooning: Inflate balloon driver → guest frees     │  │
│  │   pages → hypervisor reclaims. Deflate → give back.  │  │
│  │                                                      │  │
│  │ KSM: Kernel Same-page Merging. Scan VM pages for    │  │
│  │   identical content → share COW-style. Saves RAM     │  │
│  │   when running many similar VMs.                     │  │
│  │                                                      │  │
│  │ Huge pages: Both host and guest can use 2MB/1GB     │  │
│  │   pages. Reduces TLB pressure (2D walk: 24 lookups  │  │
│  │   → 4 with 1GB pages).                              │  │
│  │                                                      │  │
│  │ Memory overcommit: Allocate more guest RAM than      │  │
│  │   physical host RAM. KSM + ballooning enables this.  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘

2D Page Walk Cost (worst case, 4-level guest + 4-level host):
Guest VA → 4 guest levels × 4 host levels per guest level = 24 lookups!
With huge pages (guest 2MB + host 2MB): 3 × 3 = 9 lookups (much better)
```

---

## Interview Questions

1. **Q: How does Linux memory management differ from Windows?**
   A: Key differences: Linux uses overcommit (malloc rarely fails) vs Windows' commit charge model. Linux has a global LRU for page replacement; Windows uses per-process working sets. Linux kernel pool is unified SLUB; Windows has separate paged/nonpaged pools. Linux OOM killer kills processes; Windows fails allocations at commit limit.

2. **Q: How does Android handle low memory differently than desktop Linux?**
   A: Android uses LMKD (Low Memory Killer Daemon) that proactively kills cached apps based on priority bands before the kernel OOM killer triggers. Android uses zram (compressed RAM) instead of disk swap to avoid flash wear. Android assigns oom_score_adj based on app lifecycle state (foreground=0, cached=900).

3. **Q: Why don't RTOS systems use virtual memory?**
   A: RTOS systems prioritize determinism and minimal overhead. Virtual memory introduces non-deterministic page faults, TLB misses, and reclaim latency. RTOS devices often have limited RAM (KB-MB range) where page table overhead is proportionally large. Safety-critical systems use static allocation to guarantee memory availability.

4. **Q: What is the performance cost of virtualized memory (2D page table walk)?**
   A: With 4-level guest and 4-level host page tables, a TLB miss requires up to 24 memory accesses (each guest level requires a full host walk). EPT/NPT is hardware-assisted but still expensive. Mitigation: huge pages reduce levels, VPID/VMID tag TLB entries, nested page table caching.

---

## Summary

1. **Linux vs Windows**: Overcommit vs commit charge; global LRU vs working set; SLUB vs paged/nonpaged pools.
2. **Linux vs macOS**: Mach VM heritage; built-in compression; Jetsam (iOS) vs OOM killer.
3. **Linux vs Android**: LMKD for proactive killing; zram instead of disk swap; lifecycle-based priorities.
4. **Linux vs RTOS**: Full VM vs flat physical; dynamic vs static allocation; general-purpose vs deterministic.
5. **Virtualization**: 2D page walks (EPT/NPT/Stage-2); KSM, ballooning, and huge pages for optimization.

---

*Next: [Chapter 32 — Memory in Virtualization (KVM, Xen)](Chapter_32_Virtualization_Memory.md)*
