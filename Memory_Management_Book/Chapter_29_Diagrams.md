# Chapter 29: Important Diagrams and Visual References

## Chapter Overview

This chapter consolidates the most important memory management diagrams into one reference. These visual representations cover address spaces, data structures, hardware architecture, and kernel subsystem relationships.

---

## 29.1 x86_64 Virtual Address Space Layout

```
x86_64 with 4-level paging (48-bit VA):

0xFFFFFFFFFFFFFFFF ┌──────────────────────────────────┐
                   │ Fixmap, VDSO                      │
0xFFFFFFFF80000000 ├──────────────────────────────────┤
                   │ Kernel text (vmlinux)              │ __START_KERNEL_map
0xFFFFFFFF00000000 ├──────────────────────────────────┤
                   │ Modules (-2GB to -1GB)             │ MODULES_VADDR
0xFFFFFFFE00000000 ├──────────────────────────────────┤
                   │ (hole)                             │
0xFFFFEA0000000000 ├──────────────────────────────────┤
                   │ vmemmap (struct page array)        │ VMEMMAP_START
0xFFFFE90000000000 ├──────────────────────────────────┤
                   │ (hole)                             │
0xFFFFD00000000000 ├──────────────────────────────────┤
                   │ KASAN shadow memory                │
0xFFFF888000000000 ├──────────────────────────────────┤
                   │ Direct physical mapping            │ PAGE_OFFSET
                   │ (all physical RAM mapped here)     │ phys_to_virt(0)
0xFFFF880000000000 ├──────────────────────────────────┤
                   │ vmalloc / ioremap area             │ VMALLOC_START
0xFFFF800000000000 ├──────────────────────────────────┤
                   │                                    │
                   │  ═══ CANONICAL GAP ═══             │ Non-canonical
                   │  (hole: bits 47:63 must match)     │ addresses
                   │                                    │
0x00007FFFFFFFFFFF ├──────────────────────────────────┤
                   │ User stack (grows down)            │
                   │ ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓                │
                   ├──────────────────────────────────┤
                   │ mmap region (libraries, shared)    │
                   │ (grows down from stack)            │
                   ├──────────────────────────────────┤
                   │ Heap (brk, grows up)               │
                   │ ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑                │
                   ├──────────────────────────────────┤
                   │ BSS (uninitialized data)           │
                   ├──────────────────────────────────┤
                   │ Data (initialized globals)         │
                   ├──────────────────────────────────┤
                   │ Text (code, read-only)             │
0x0000000000400000 ├──────────────────────────────────┤ (non-PIE)
                   │ NULL page (unmapped, guard)        │
0x0000000000000000 └──────────────────────────────────┘
```

---

## 29.2 ARM64 Virtual Address Space Layout (48-bit)

```
0xFFFFFFFFFFFFFFFF ┌──────────────────────────────────┐
                   │ Fixmap, kernel vectors             │
0xFFFF800080000000 ├──────────────────────────────────┤
                   │ Kernel image (text, data)          │
0xFFFF800010000000 ├──────────────────────────────────┤
                   │ vmemmap                            │
                   ├──────────────────────────────────┤
                   │ vmalloc/ioremap                    │
                   ├──────────────────────────────────┤
                   │ Linear map (direct phys mapping)   │ PAGE_OFFSET
0xFFFF000000000000 ├──────────────────────────────────┤
                   │                                    │
                   │  ═══ TTBR split ═══                │
                   │  TTBR1 = kernel (above)            │
                   │  TTBR0 = user   (below)            │
                   │                                    │
0x0000FFFFFFFFFFFF ├──────────────────────────────────┤
                   │ User address space                 │
                   │  Stack, mmap, heap, text           │
                   │  (same layout as x86_64 user)      │
0x0000000000000000 └──────────────────────────────────┘

ARM64 uses two page table bases:
  TTBR0_EL1 → user page tables (address bit 55=0)
  TTBR1_EL1 → kernel page tables (address bit 55=1)
  Natural KPTI: switching TTBR0 on context switch
```

---

## 29.3 struct page Memory Layout (64 bytes)

```
struct page (simplified, showing union variants):

Byte offset:
0    ┌──────────────────────────────────────────────┐
     │ unsigned long flags (PG_locked, PG_dirty...) │  8 bytes
8    ├──────────────────────────────────────────────┤
     │ UNION (depends on page type):                │
     │                                              │
     │ ┌─ Page Cache/Anonymous ─────────────────┐   │
     │ │ struct list_head lru  (16 bytes)       │   │
     │ │ struct address_space *mapping (8 bytes)│   │
     │ │ pgoff_t index (8 bytes)                │   │
     │ │ unsigned long private (8 bytes)        │   │
     │ └───────────────────────────────────────┘   │
     │                                              │
     │ ┌─ SLUB Slab ───────────────────────────┐   │
     │ │ struct kmem_cache *slab_cache          │   │
     │ │ void *freelist                         │   │
     │ │ union { counters, struct { inuse, ... }}│  │
     │ └───────────────────────────────────────┘   │
     │                                              │
     │ ┌─ Compound (Huge Page) ────────────────┐   │
     │ │ unsigned long compound_head            │   │
     │ │ unsigned char compound_order           │   │
     │ │ unsigned int compound_nr               │   │
     │ └───────────────────────────────────────┘   │
48   ├──────────────────────────────────────────────┤
     │ atomic_t _refcount (4 bytes)                 │
52   ├──────────────────────────────────────────────┤
     │ atomic_t _mapcount (4 bytes)                 │
56   ├──────────────────────────────────────────────┤
     │ padding / memcg_data (8 bytes)               │
64   └──────────────────────────────────────────────┘

Key fields:
  flags:     Page state bits (locked, dirty, uptodate, referenced...)
  _refcount: Number of references (free when drops to 0)
  _mapcount: Number of page table entries pointing here (-1 = unmapped)
  mapping:   Owner (address_space for file, anon_vma for anonymous)
  index:     Offset within the mapping (page number in file)
  lru:       Linkage in LRU list (active/inactive)

Size: 64 bytes per 4KB page = 1.56% memory overhead
16GB RAM = 4M pages × 64 bytes = 256MB struct page overhead
```

---

## 29.4 Page Table Walk (x86_64, 4-level)

```
Virtual Address (48-bit, bits 63:48 = sign extension):

63    48 47    39 38    30 29    21 20    12 11       0
┌────────┬────────┬────────┬────────┬────────┬──────────┐
│Sign ext│ PGD idx│ PUD idx│ PMD idx│ PTE idx│  Offset  │
│(16 bits)│(9 bits)│(9 bits)│(9 bits)│(9 bits)│(12 bits) │
└────────┴───┬────┴───┬────┴───┬────┴───┬────┴────┬─────┘
             │        │        │        │         │
             ▼        ▼        ▼        ▼         │
CR3──→┌─────────┐  ┌──────┐  ┌──────┐  ┌──────┐  │
      │ PGD     │  │ PUD  │  │ PMD  │  │ PTE  │  │
      │ 512     │  │ 512  │  │ 512  │  │ 512  │  │
      │ entries │  │entries│  │entries│  │entries│  │
      │         │  │      │  │      │  │      │  │
      │ [idx]───┼─→│[idx]─┼─→│[idx]─┼─→│[idx] │  │
      └─────────┘  └──────┘  └──────┘  └──┬───┘  │
                                           │      │
                                           ▼      │
                                     ┌──────────┐ │
                                     │ Physical  │ │
                                     │ Page Frame│◄┘ + offset
                                     │           │
                                     └──────────┘

PTE format (64-bit):
63   62  52 51        12 11  9 8 7 6 5 4 3 2 1 0
┌────┬──────┬───────────┬─────┬──┬─┬─┬─┬─┬─┬─┬─┐
│ NX │ Avail│   PFN     │ AVL │G │PAT│D│A│PCD│PWT│U│W│P│
└────┴──────┴───────────┴─────┴──┴─┴─┴─┴─┴─┴─┴─┘
P=Present, W=Writable, U=User, A=Accessed, D=Dirty
NX=No-Execute, PFN=Physical Frame Number (40 bits)
```

---

## 29.5 Memory Subsystem Relationships

```
┌─────────────────────────────────────────────────────────────────────┐
│                        USERSPACE                                     │
│  malloc/free    mmap/munmap    read/write     brk()                │
└──────┬──────────────┬──────────────┬──────────────┬────────────────┘
       │              │              │              │
  ═════╪══════════════╪══════════════╪══════════════╪═══ SYSCALL ═════
       │              │              │              │
┌──────▼──────┐ ┌─────▼─────┐ ┌─────▼─────┐ ┌─────▼─────┐
│ VMA Manager │ │ VMA Mgr   │ │ VFS Layer │ │ VMA Mgr   │
│ (mmap.c)    │ │ (mmap.c)  │ │ (filemap) │ │ (mmap.c)  │
└──────┬──────┘ └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
       │              │              │              │
       ▼              ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     PAGE FAULT HANDLER                           │
│                     (mm/memory.c)                                │
│  do_anonymous_page | do_fault | do_wp_page | do_swap_page       │
└────────┬────────────────┬────────────────┬──────────────────────┘
         │                │                │
         ▼                ▼                ▼
┌────────────────┐ ┌──────────────┐ ┌──────────────┐
│  PAGE CACHE    │ │ SWAP SUBSYS  │ │ RMAP         │
│  (filemap.c)   │ │ (swap*.c)    │ │ (rmap.c)     │
│  XArray lookup │ │ swap in/out  │ │ page→VMA map │
└───────┬────────┘ └──────┬───────┘ └──────────────┘
        │                 │
        ▼                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                     PAGE RECLAIM ENGINE                          │
│                     (mm/vmscan.c)                                │
│  kswapd | direct reclaim | MGLRU | shrink_node                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
┌────────────────┐ ┌────────────────┐  ┌────────────────┐
│ SLUB ALLOCATOR │ │ PAGE ALLOCATOR │  │  COMPACTION    │
│ (mm/slub.c)    │ │ (page_alloc.c) │  │ (compaction.c) │
│ kmalloc, slab  │ │ BUDDY SYSTEM   │  │ defrag pages   │
│ caches         │ │ alloc_pages()  │  │ migration      │
└───────┬────────┘ └───────┬────────┘  └────────────────┘
        │                  │
        └──────────────────┘
                │
    ┌───────────▼───────────┐
    │    PHYSICAL MEMORY    │
    │    (RAM / HARDWARE)   │
    └───────────────────────┘
```

---

## 29.6 NUMA Topology

```
Dual-Socket NUMA System:

┌─────── Node 0 ─────────────┐    ┌─────── Node 1 ─────────────┐
│                             │    │                             │
│  ┌─── CPU 0 ──┐ ┌─ CPU 1 ─┐│    │┌─── CPU 2 ──┐ ┌─ CPU 3 ─┐│
│  │ L1d 32KB   │ │L1d 32KB ││    ││ L1d 32KB   │ │L1d 32KB ││
│  │ L1i 32KB   │ │L1i 32KB ││    ││ L1i 32KB   │ │L1i 32KB ││
│  │ L2 256KB   │ │L2 256KB ││    ││ L2 256KB   │ │L2 256KB ││
│  └─────┬──────┘ └────┬────┘│    │└─────┬──────┘ └────┬────┘│
│        └──────┬───────┘     │    │      └──────┬───────┘     │
│         L3 Cache (shared)   │    │       L3 Cache (shared)   │
│         30MB                │    │       30MB                │
│               │             │    │             │             │
│        Memory Controller    │    │      Memory Controller    │
│               │             │    │             │             │
│     ┌─────────┴─────────┐   │    │   ┌─────────┴─────────┐   │
│     │ DDR4 64GB (local) │   │    │   │ DDR4 64GB (local) │   │
│     └───────────────────┘   │    │   └───────────────────┘   │
│                             │    │                             │
└──────────────┬──────────────┘    └──────────────┬──────────────┘
               │                                   │
               └──── QPI / UPI Interconnect ───────┘
                     (remote access ~2x latency)

Access Latency:
  CPU 0 → Node 0 RAM: ~80ns  (local)
  CPU 0 → Node 1 RAM: ~140ns (remote, via interconnect)
  Ratio: 1.75× penalty for remote access
```

---

## 29.7 LRU Lists and Page Lifecycle

```
Page Lifecycle through LRU Lists:

             ┌──────────────────────────────────┐
             │         PAGE ALLOCATION           │
             │  alloc_pages() / page cache add   │
             └──────────────┬───────────────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │   INACTIVE LIST       │  New pages start here
                │   (head)       (tail) │
                │   [P1][P2][P3][P4]    │  ← Reclaim scans from tail
                └──────────┬────────────┘
                           │
              ┌────────────┼────────────┐
              │ Referenced  │ Not ref'd  │
              │ (accessed)  │ (cold)     │
              ▼            ▼            │
   ┌───────────────────┐  │            │
   │   ACTIVE LIST     │  │  RECLAIM:  │
   │   [A1][A2][A3]    │  │  ├─ Clean file → FREE immediately
   │   (protected)     │  │  ├─ Dirty → writeback → free later
   └────────┬──────────┘  │  └─ Anonymous → swap out → free
            │             │
            │ Aged out    │
            │ (not ref'd  │
            │  for long)  │
            └─────────────┘
              ↓ Demote back to inactive

MGLRU (Multi-Gen LRU, Linux 6.1+):
  Instead of 2 lists (active/inactive), uses multiple generations:

  Gen 3 (youngest, most recently accessed)
  Gen 2
  Gen 1
  Gen 0 (oldest, least recently accessed → reclaim first)

  Pages age through generations based on access patterns.
  Better working set estimation than binary active/inactive.
```

---

## 29.8 Memory Zones and Watermarks

```
Zone Structure with Watermarks:

Physical Memory Layout (x86_64):
┌─────────────────────────────────────────────────────────────────┐ 
│           Zone Normal (> 4GB)                                   │
│  ┌─────────────────────────────────────────────────────┐        │
│  │████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░│        │
│  │  Used pages                Free pages               │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                  │
│  Watermarks:                                                     │
│  ████████████████████████████████████░░░░│░░░░░│░░░░░░░░░░░░░░│ │
│  ◄────── USED ──────────────────────►    │     │              │ │
│                                     WMARK│WMARK│  WMARK      │ │
│                                     _MIN │_LOW │  _HIGH      │ │
│                                          │     │              │ │
│                              ┌────────────┘     │              │ │
│                              │                  │              │ │
│  When free < WMARK_MIN: ─────┘                  │              │ │
│    Direct reclaim + throttle allocations         │              │ │
│                                                  │              │ │
│  When free < WMARK_LOW: ─────────────────────────┘              │ │
│    Wake kswapd (background reclaim)                             │ │
│                                                                  │
│  When free > WMARK_HIGH: ────────────────────────────────────── │ │
│    kswapd goes to sleep (enough free pages)                     │ │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│ Zone DMA32 (0 - 4GB): 32-bit device DMA                        │
├─────────────────────────────────────────────────────────────────┤
│ Zone DMA (0 - 16MB): Legacy ISA DMA                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 29.9 Complete Memory Management Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    HARDWARE LAYER                                       │
│  ┌──────┐  ┌─────┐  ┌──────────┐  ┌──────┐  ┌───────┐  ┌──────────┐  │
│  │ CPU  │  │ TLB │  │Page Table│  │ MMU  │  │ Cache │  │ DRAM     │  │
│  │      │  │     │  │ Walker   │  │      │  │ L1-L3 │  │ (RAM)    │  │
│  └──┬───┘  └──┬──┘  └────┬─────┘  └──┬───┘  └───┬───┘  └────┬─────┘  │
│     └────┬────┘          │            │          │            │        │
│          └──────────┬────┘            │          │            │        │
│                     └────────────┬────┘          │            │        │
│  VA → TLB hit? → PA                    └─────┬───┘            │        │
│  VA → TLB miss → Page Walk → PTE → PA       │               │        │
│  PA → Cache hit? → Data                      └───────────────┘        │
│  PA → Cache miss → DRAM access → Data                                  │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                    KERNEL MM LAYER                                       │
│                                                                         │
│  Process Management:     Physical Management:     Special Areas:        │
│  ┌──────────────────┐   ┌─────────────────────┐  ┌──────────────────┐ │
│  │ mm_struct         │   │ Buddy Allocator     │  │ vmalloc          │ │
│  │ vm_area_struct    │   │ Per-CPU PCP         │  │ ioremap          │ │
│  │ Page Tables       │   │ Zone Management     │  │ CMA              │ │
│  │ mmap / munmap     │   │ Compaction          │  │ DMA pools        │ │
│  │ Page Fault Handler│   │ Watermarks          │  │ Per-CPU alloc    │ │
│  └──────────────────┘   └─────────────────────┘  └──────────────────┘ │
│                                                                         │
│  Cache Management:       Reclaim:                 Slab Allocator:      │
│  ┌──────────────────┐   ┌─────────────────────┐  ┌──────────────────┐ │
│  │ Page Cache        │   │ kswapd              │  │ SLUB             │ │
│  │ Address Space     │   │ Direct Reclaim      │  │ kmalloc caches   │ │
│  │ Readahead         │   │ MGLRU               │  │ Custom caches    │ │
│  │ Writeback         │   │ OOM Killer          │  │ Per-CPU freelists│ │
│  │ Dirty Tracking    │   │ Swap Management     │  │                  │ │
│  └──────────────────┘   └─────────────────────┘  └──────────────────┘ │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                    CONTROL & SECURITY LAYER                             │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐ │
│  │ Memory Cgroups    │  │ KPTI / KASLR     │  │ KASAN / KFENCE      │ │
│  │ PSI Monitoring    │  │ NX / SMEP / SMAP │  │ kmemleak            │ │
│  │ Per-cgroup OOM    │  │ MTE / CFI        │  │ slub_debug          │ │
│  └──────────────────┘  └──────────────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions

1. **Q: Draw the x86_64 virtual address space layout.**
   A: The 48-bit VA space splits into: user (0x0000000000000000-0x00007FFFFFFFFFFF) containing text, data, heap (grows up), mmap/libs, and stack (grows down); a canonical gap (16-bit sign extension); kernel (0xFFFF800000000000+) containing vmalloc, direct physical mapping (PAGE_OFFSET), vmemmap, KASAN shadow, modules, and kernel text.

2. **Q: What are the PTE bits on x86_64?**
   A: Present(0), Writable(1), User(2), Write-Through(3), Cache-Disable(4), Accessed(5), Dirty(6), PAT(7), Global(8), bits 9-11 available, PFN(12-51), bits 52-62 available, NX(63). The combination controls access permissions, cache behavior, and execution rights.

3. **Q: Explain the relationship between kswapd, watermarks, and direct reclaim.**
   A: Three watermarks per zone: WMARK_HIGH (kswapd sleeps), WMARK_LOW (kswapd wakes), WMARK_MIN (direct reclaim + throttle). When free pages drop below LOW, kswapd scans LRU lists in background. If allocation still fails (below MIN), the allocating process does direct reclaim synchronously. kswapd keeps running until free > HIGH.

---

## Summary

This chapter provides reference diagrams for:
1. **x86_64 and ARM64** virtual address space layouts
2. **struct page** internal layout (64 bytes, union variants)
3. **4-level page table walk** with PTE format
4. **Memory subsystem relationships** (VMA → fault → cache → allocator → hardware)
5. **NUMA topology** (nodes, interconnect, latency)
6. **LRU page lifecycle** (active/inactive lists, MGLRU generations)
7. **Zone watermarks** (MIN/LOW/HIGH thresholds)
8. **Complete MM architecture** (hardware → kernel → control layers)

---

*Next: [Chapter 30 — Glossary of Memory Management Terms](Chapter_30_Glossary.md)*
