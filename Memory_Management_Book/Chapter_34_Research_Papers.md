# Chapter 34: Research Papers and Further Reading

## Chapter Overview

This chapter compiles essential research papers, books, documentation, and online resources for deepening your understanding of memory management.

---

## 34.1 Foundational Papers

```
1. "The Multics Virtual Memory: Concepts and Design"
   — Daley & Dennis, 1968
   First practical virtual memory system. Influenced all modern OS designs.

2. "Virtual Memory, Processes, and Sharing in MULTICS"
   — Bensoussan, Clingen, Daley, 1972
   Expanded VM concepts: segmentation + paging combined.

3. "A Fast File System for UNIX"
   — McKusick, Joy, Leffler, Fabry, 1984 (FFS)
   BSD Fast File System. Buffer cache design influenced Linux page cache.

4. "The UNIX Time-sharing System"
   — Ritchie & Thompson, 1974
   Original UNIX. Process address space, swapping, memory model.

5. "The Slab Allocator: An Object-Caching Kernel Memory Allocator"
   — Bonwick, 1994 (USENIX)
   Original slab allocator design for SunOS. Basis for Linux SLAB/SLUB.

6. "SLUB: The Unqueued Slab Allocator"
   — Penberg, 2007 (Linux commit)
   SLUB design: simplified slab allocator, per-CPU fast paths, current default.
```

---

## 34.2 Linux-Specific Papers

```
7. "Understanding the Linux Kernel" (Book)
   — Bovet & Cesati, O'Reilly
   Classic reference for Linux 2.6 kernel internals including MM.

8. "Linux Kernel Development" (Book)  
   — Robert Love, Addison-Wesley
   Accessible kernel internals. Good chapter on memory management.

9. "Professional Linux Kernel Architecture" (Book)
   — Wolfgang Mauerer, Wrox
   Deep dive into kernel architecture. Detailed MM coverage.

10. "Multi-generational LRU Framework" (MGLRU)
    — Yu Zhao, Google, 2022 (merged Linux 6.1)
    Replaced binary active/inactive lists with multi-generation aging.
    Significant improvement for workloads with large working sets.
    LWN coverage: https://lwn.net/Articles/894859/

11. "Maple Tree: A Modern Data Structure for a Complex Problem"
    — Liam Howlett, Oracle, 2022 (merged Linux 6.1)
    Replaced red-black tree for VMA management. Better cache behavior.
    
12. "Folios" 
    — Matthew Wilcox, 2021-2023
    Replacing struct page with struct folio for compound page handling.
    Simplifies page cache, huge page, and filesystem code.
```

---

## 34.3 Hardware Architecture Papers

```
13. "What Every Programmer Should Know About Memory"
    — Ulrich Drepper, Red Hat, 2007
    Essential paper on hardware memory architecture: caches, NUMA,
    TLB, DRAM internals. ~100 pages. FREE online.
    https://people.freebsd.org/~lstewart/articles/cpumemory.pdf

14. "A Primer on Memory Consistency and Cache Coherence"
    — Sorin, Hill, Wood, 2011 (Morgan & Claypool)
    Formal treatment of memory consistency models: TSO, SC, ARM.

15. "Intel® 64 and IA-32 Architectures Software Developer's Manual"
    — Intel Corporation
    Definitive reference for x86_64 paging, TLB, cache, EPT.
    Volume 3A, Chapters 4 (Paging) and 11 (Cache).

16. "ARM Architecture Reference Manual" (Arm ARM)
    — ARM Ltd
    Definitive for ARM64 page table formats, TLB, cache, MTE, PAC.
```

---

## 34.4 Security Papers

```
17. "Meltdown: Reading Kernel Memory from User Space"
    — Lipp et al., 2018 (USENIX Security)
    Speculative execution attack reading kernel memory. Led to KPTI.

18. "Spectre Attacks: Exploiting Speculative Execution"
    — Kocher et al., 2019 (IEEE S&P)
    Branch prediction attacks. Led to retpoline, IBRS mitigations.

19. "KASLR is Dead: Long Live KASLR"
    — Gruss et al., 2017
    Analysis of KASLR effectiveness and side-channel defeats.

20. "MTE: Catching Memory Safety Bugs with Hardware"
    — ARM, Google (Android blog, 2023)
    Practical MTE deployment on Android for heap memory safety.
```

---

## 34.5 Virtualization Papers

```
21. "Memory Resource Management in VMware ESX Server"
    — Waldspurger, 2002 (OSDI)
    Ballooning, content-based page sharing (KSM predecessor),
    idle memory tax. Foundational for VM memory management.

22. "Live Migration of Virtual Machines"
    — Clark et al., 2005 (NSDI)
    Pre-copy live migration algorithm. Dirty page tracking.

23. "KVM: The Linux Virtual Machine Monitor"
    — Kivity et al., 2007 (OLS)
    KVM design: using Linux as hypervisor, reusing Linux MM.
```

---

## 34.6 Modern Memory Technologies

```
24. "An Introduction to Persistent Memory Programming"
    — Intel PMDK documentation
    https://pmem.io/
    Programming guide for PMEM: libpmem, PMDK libraries.

25. "CXL 3.0 Specification"
    — CXL Consortium
    Memory pooling, sharing, and expansion over PCIe.

26. "Heterogeneous Memory Management (HMM)"
    — Documentation/mm/hmm.rst (kernel docs)
    GPU/accelerator memory sharing via unified addressing.

27. "Memory Tiering in Linux"
    — Various LWN.net articles, 2022-2023
    DRAM/CXL/PMEM tiering with automatic promotion/demotion.
```

---

## 34.7 Essential Online Resources

```
Documentation and Code:
┌──────────────────────────────────────────────────────────────────┐
│ Resource                     │ URL                               │
├──────────────────────────────┼───────────────────────────────────┤
│ Linux kernel source browser  │ https://elixir.bootlin.com        │
│ Kernel documentation (MM)    │ Documentation/mm/ in kernel tree  │
│ LWN.net (Linux articles)     │ https://lwn.net                   │
│ Linux MM wiki                │ https://linux-mm.org              │
│ Kernel Newbies               │ https://kernelnewbies.org         │
│ LKML (mailing list)         │ https://lkml.org                  │
│ Free Electrons docs          │ https://bootlin.com/docs/         │
│ KernelShark (ftrace GUI)    │ https://kernelshark.org           │
└──────────────────────────────┴───────────────────────────────────┘

Books (Print):
┌──────────────────────────────────────────────────────────────────┐
│ "Understanding the Linux Kernel"     — Bovet & Cesati (O'Reilly)│
│ "Linux Kernel Development"           — Robert Love              │
│ "Linux Device Drivers"               — Corbet, Rubini, K-Hartman│
│ "Professional Linux Kernel Arch."    — Wolfgang Mauerer         │
│ "Operating Systems: Three Easy Pieces" — Arpaci-Dusseau (FREE) │
│   https://pages.cs.wisc.edu/~remzi/OSTEP/                     │
│ "Computer Architecture: A Quant. Approach" — Hennessy & Patterson│
│ "Modern Operating Systems"           — Andrew Tanenbaum        │
└──────────────────────────────────────────────────────────────────┘

Video Resources:
┌──────────────────────────────────────────────────────────────────┐
│ Linux Foundation Training       │ Kernel internals courses       │
│ MIT 6.828 (Operating Systems)   │ Free lecture videos + xv6 OS  │
│ Linux Plumbers Conference       │ Yearly kernel developer talks │
│ LPC MM track recordings        │ Memory management deep dives  │
└──────────────────────────────────┴───────────────────────────────┘
```

---

## 34.8 Kernel Source Files Reading Order

```
Recommended reading order for studying Linux MM source:

1. Start with data structures:
   include/linux/mm_types.h         ← struct page, mm_struct, vma
   include/linux/gfp.h              ← GFP flags
   
2. Page allocation:
   mm/page_alloc.c                  ← Buddy allocator
   
3. Slab allocation:
   mm/slub.c                        ← kmalloc, slab caches

4. Process address space:
   mm/mmap.c                        ← VMA management, mmap
   
5. Page faults:
   mm/memory.c                      ← Fault handler, COW, demand paging
   
6. Page cache:
   mm/filemap.c                     ← Page cache lookup, readahead
   
7. Page reclaim:
   mm/vmscan.c                      ← kswapd, direct reclaim, MGLRU
   
8. OOM:
   mm/oom_kill.c                    ← OOM killer selection

9. Swap:
   mm/swap_state.c + mm/swapfile.c  ← Swap in/out
   
10. Architecture-specific:
    arch/x86/mm/fault.c             ← x86 page fault entry
    arch/arm64/mm/fault.c           ← ARM64 page fault entry
```

---

## Summary

This chapter provides a curated reading list organized by topic:
1. **Foundational**: Multics VM, Unix, Slab allocator papers
2. **Linux-specific**: Books by Love, Bovet/Cesati; MGLRU, Maple Tree, Folio patches
3. **Hardware**: Drepper's memory paper, Intel/ARM architecture manuals
4. **Security**: Meltdown/Spectre papers, MTE deployment
5. **Virtualization**: VMware memory management, KVM design, live migration
6. **Modern**: PMEM/DAX, CXL, HMM/GPU memory, memory tiering
7. **Online**: elixir.bootlin.com, LWN.net, OSTEP (free textbook)

---

*Next: [Chapter 35 — Resources and Community](Chapter_35_Resources.md)*
