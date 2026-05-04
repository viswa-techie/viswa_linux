# Linux Memory Management — Complete Book

## Master Index

> **36 Chapters** covering everything from hardware fundamentals to expert-level kernel internals.
> Target audience: Embedded engineers, kernel developers, system programmers, and interview candidates.

---

## Part I: Foundations (Chapters 1–6)

| # | Chapter | Key Topics |
|---|---------|------------|
| 1 | [Introduction & Motivation](Chapter_01_Introduction.md) | Why memory management matters, goals, book roadmap |
| 2 | [History of Memory Management](Chapter_02_History.md) | Flat → segmented → paged, Unix/Linux evolution |
| 3 | [Hardware Architecture](Chapter_03_Hardware_Architecture.md) | CPU caches, DRAM, memory hierarchy, latency numbers |
| 4 | [Virtual Memory Concepts](Chapter_04_Virtual_Memory_Concepts.md) | Address translation, page tables, TLB, demand paging |
| 5 | [Address Translation Deep Dive](Chapter_05_Address_Translation.md) | x86_64 4/5-level walk, PTE format, ARM64 Stage-1/2 |
| 6 | [Linux Memory Architecture](Chapter_06_Linux_Memory_Architecture.md) | Zones (DMA/Normal/HighMem), NUMA nodes, watermarks |

## Part II: Core Kernel Subsystems (Chapters 7–12)

| # | Chapter | Key Topics |
|---|---------|------------|
| 7 | [Linux Virtual Memory Layout](Chapter_07_Virtual_Memory_Layout.md) | User/kernel VA layout, x86_64 canonical hole, ARM64 |
| 8 | [Page Allocation (Buddy)](Chapter_08_Page_Allocation.md) | Buddy algorithm, alloc_pages, GFP flags, migration types |
| 9 | [Kernel Memory Allocators](Chapter_09_Kernel_Allocators.md) | SLUB, kmalloc, kmem_cache, vmalloc, percpu |
| 10 | [Page Cache & Buffer Cache](Chapter_10_Page_Cache.md) | address_space, XArray lookup, readahead, writeback |
| 11 | [Demand Paging](Chapter_11_Demand_Paging.md) | Lazy allocation, fault-in flow, zero pages |
| 12 | [Page Fault Handling](Chapter_12_Page_Fault_Handling.md) | do_page_fault, minor/major, signals, userfaultfd |

## Part III: Advanced Mechanisms (Chapters 13–18)

| # | Chapter | Key Topics |
|---|---------|------------|
| 13 | [Copy-on-Write (COW)](Chapter_13_COW.md) | fork optimization, wp_page_copy, KSM |
| 14 | [Memory Reclaim & Compaction](Chapter_14_Memory_Reclaim.md) | kswapd, direct reclaim, LRU, MGLRU, compaction |
| 15 | [Swap Management](Chapter_15_Swap.md) | Swap areas, swap cache, zswap, zram, swap slots |
| 16 | [Huge Pages](Chapter_16_Huge_Pages.md) | THP, hugetlbfs, khugepaged, split/collapse |
| 17 | [NUMA Memory Management](Chapter_17_NUMA.md) | Policies (bind/interleave/preferred), NUMA balancing |
| 18 | [CMA (Contiguous Memory Allocator)](Chapter_18_CMA.md) | Reserved regions, movable migration, driver use |

## Part IV: Drivers & Hardware (Chapters 19–21)

| # | Chapter | Key Topics |
|---|---------|------------|
| 19 | [IOMMU & Device Memory](Chapter_19_IOMMU.md) | IOVA, DMA remapping, IOMMU groups, VFIO |
| 20 | [DMA Memory Management](Chapter_20_DMA_Memory.md) | Coherent vs streaming, scatter-gather, SWIOTLB, devm |
| 21 | [Mmap in Drivers](Chapter_21_Mmap_in_Drivers.md) | remap_pfn_range, vm_ops->fault, dma-buf, V4L2 |

## Part V: Security (Chapter 22)

| # | Chapter | Key Topics |
|---|---------|------------|
| 22 | [Memory Security](Chapter_22_Memory_Security.md) | KPTI, ASLR/KASLR, NX, SMAP/PAN, MTE, CFI, W^X |

## Part VI: Containers, Debugging, Tuning (Chapters 23–27)

| # | Chapter | Key Topics |
|---|---------|------------|
| 23 | [Memory Cgroups](Chapter_23_Memory_Cgroups.md) | cgroup v1/v2, memory.max/high/low, PSI, containers |
| 24 | [Memory Debugging Tools](Chapter_24_Memory_Debugging_Tools.md) | /proc, vmstat, perf, valgrind, ASan, Android tools |
| 25 | [Kernel Memory Debugging](Chapter_25_Kernel_Memory_Debugging.md) | KASAN, kmemleak, KFENCE, SLUB debug, ftrace |
| 26 | [Source Code Walkthrough](Chapter_26_Source_Code_Walkthrough.md) | mm/ directory, key files, reading strategies |
| 27 | [Performance Tuning](Chapter_27_Performance_Tuning.md) | /proc/sys/vm tunables, huge page tuning, NUMA opt |

## Part VII: Reference & Visualization (Chapters 28–30)

| # | Chapter | Key Topics |
|---|---------|------------|
| 28 | [End-to-End Flow Diagrams](Chapter_28_Flow_Diagrams.md) | malloc→phys, fork→COW, reclaim→OOM, 7 flows |
| 29 | [Architecture Diagrams](Chapter_29_Diagrams.md) | VA layouts, struct page, page walk, subsystem map |
| 30 | [Glossary](Chapter_30_Glossary.md) | A–Z definitions of 100+ MM terms |

## Part VIII: Comparative & Advanced (Chapters 31–33)

| # | Chapter | Key Topics |
|---|---------|------------|
| 31 | [OS Comparison](Chapter_31_OS_Comparison.md) | Linux vs Windows, macOS, Android, RTOS |
| 32 | [Virtualization Memory](Chapter_32_Virtualization_Memory.md) | EPT/NPT, KVM, ballooning, KSM, VFIO, live migration |
| 33 | [Modern Memory Technologies](Chapter_33_Modern_Memory.md) | PMEM/DAX, GPU/HMM, CXL, tiering, confidential VM |

## Part IX: Further Learning & Interview Prep (Chapters 34–36)

| # | Chapter | Key Topics |
|---|---------|------------|
| 34 | [Research Papers & Reading](Chapter_34_Research_Papers.md) | Key papers, online resources, source reading order |
| 35 | [Resources & Exercises](Chapter_35_Resources.md) | Dev setup, hands-on labs, community, conferences |
| 36 | [Interview Questions (55+)](Chapter_36_Interview_Questions.md) | Beginner → Expert, all topics, design scenarios |

---

## Quick Navigation by Topic

### If you need to understand...

| Topic | Start Here | Then Read |
|-------|-----------|-----------|
| How malloc works end-to-end | Ch 4, Ch 11 | Ch 8, Ch 12, Ch 28 |
| Page fault handling | Ch 12 | Ch 11, Ch 13, Ch 21 |
| Why my system is OOM killing | Ch 14, Ch 10 | Ch 15, Ch 23, Ch 24 |
| Kernel slab allocators | Ch 9 | Ch 8, Ch 25, Ch 26 |
| Driver DMA / mmap | Ch 20, Ch 21 | Ch 19, Ch 18 |
| Container memory limits | Ch 23 | Ch 14, Ch 24 |
| Memory security features | Ch 22 | Ch 5, Ch 7 |
| Performance optimization | Ch 27 | Ch 16, Ch 17, Ch 24 |
| Virtualization memory | Ch 32 | Ch 5, Ch 19 |
| Interview preparation | Ch 36 | Ch 28, Ch 29, Ch 30 |

---

## Recommended Reading Paths

### Path 1: Application Developer (2-3 days)
Ch 1 → Ch 4 → Ch 7 → Ch 11 → Ch 12 → Ch 16 → Ch 24 → Ch 27

### Path 2: Kernel Developer (1-2 weeks)
Ch 1–6 → Ch 7–12 → Ch 13–18 → Ch 25–26 → Ch 28–29

### Path 3: Driver Developer (1 week)
Ch 1 → Ch 4–5 → Ch 8–9 → Ch 18–21 → Ch 25

### Path 4: Interview Prep (3-5 days)
Ch 36 (questions) → Ch 28 (flows) → Ch 29 (diagrams) → Ch 30 (glossary) → weak-area chapters

### Path 5: Complete Coverage (2-4 weeks)
Chapters 1 through 36 in order.

---

*Generated: Linux Memory Management Complete Study Guide — 36 Chapters*
