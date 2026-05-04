# Chapter 6: Linux Memory Architecture Overview

## Chapter Overview

This chapter provides a birds-eye view of how Linux organizes and manages memory. We cover the kernel/user space split, memory zones, NUMA nodes, the fundamental `struct page` data structure, and how the kernel tracks every physical page frame in the system.

---

## 6.1 Linux Memory Subsystem Overview

```
Linux Memory Management Subsystem Architecture:

┌─────────────────────────────────────────────────────────────────┐
│                     User Space Programs                         │
│    malloc() / mmap() / brk() / shm_open() / mlock()           │
└──────────────────────────┬──────────────────────────────────────┘
                           │ System Calls
┌──────────────────────────▼──────────────────────────────────────┐
│                     VMA Layer (Virtual Memory Areas)            │
│    mm_struct → vm_area_struct → mmap / brk / mprotect          │
├─────────────────────────────────────────────────────────────────┤
│                     Page Fault Handler                          │
│    do_page_fault() → handle_mm_fault() → demand paging / CoW   │
├─────────────────────────────────────────────────────────────────┤
│                     Page Cache / File Mapping                   │
│    address_space → radix tree/xarray → file-backed pages       │
├────────────┬──────────────────────────────┬─────────────────────┤
│  Slab/SLUB │      Page Allocator          │     vmalloc         │
│  Allocator │    (Buddy System)            │    Allocator        │
│ (sub-page) │  alloc_pages() / __get_free  │  vmalloc()          │
│            │  _pages()                     │  (non-contiguous)   │
├────────────┴──────────────────────────────┴─────────────────────┤
│                     Physical Page Frame Management              │
│    struct page / struct folio → mem_map[] / vmemmap             │
├─────────────────────────────────────────────────────────────────┤
│                     Zone Allocator                              │
│    ZONE_DMA / ZONE_DMA32 / ZONE_NORMAL / ZONE_HIGHMEM         │
├─────────────────────────────────────────────────────────────────┤
│                     NUMA Node Management                        │
│    pg_data_t per node → zone lists → page free lists           │
├─────────────────────────────────────────────────────────────────┤
│                     Memory Reclaim                              │
│    kswapd / direct reclaim / LRU / MGLRU / compaction          │
├─────────────────────────────────────────────────────────────────┤
│                     Swap / zswap / zram                         │
│    swap_info_struct → swap slots → backing store               │
└─────────────────────────────────────────────────────────────────┘
```

### Key Source Files

| File | Purpose |
|------|---------|
| `mm/memory.c` | Core page fault handling, page table manipulation |
| `mm/mmap.c` | VMA management, mmap/munmap |
| `mm/page_alloc.c` | Buddy allocator, zone management |
| `mm/vmscan.c` | Page reclaim, kswapd |
| `mm/slab.c` / `mm/slub.c` | Slab/SLUB allocators |
| `mm/vmalloc.c` | vmalloc implementation |
| `mm/swap.c` | Swap management |
| `mm/compaction.c` | Memory compaction |
| `mm/hugetlb.c` | Huge page management |
| `include/linux/mm.h` | Core MM definitions |
| `include/linux/mmzone.h` | Zone and node definitions |
| `include/linux/mm_types.h` | struct page, mm_struct, vm_area_struct |

---

## 6.2 Kernel vs User Memory

### Address Space Layout (x86_64, 48-bit)

```
0xFFFFFFFFFFFFFFFF ┌─────────────────────────────┐
                   │  Fixmap area                 │
                   │  VSYSCALL page               │
0xFFFFFFFFFE000000 ├─────────────────────────────┤
                   │  Modules (-2GB to 0)         │
0xFFFFFFFFC0000000 ├─────────────────────────────┤
                   │  Kernel text (mapped at       │
                   │  compilation-time address)    │
0xFFFFFFFF80000000 ├─────────────────────────────┤
                   │  EFI runtime mappings         │
                   ├─────────────────────────────┤
                   │  %esp fixup stacks            │
                   ├─────────────────────────────┤
                   │  vmemmap (virtual mem map)    │  ← struct page array
0xFFFFEA0000000000 ├─────────────────────────────┤
                   │  Virtual memory map (vmalloc) │
0xFFFFC90000000000 ├─────────────────────────────┤
                   │  Direct mapping of all phys  │  ← PAGE_OFFSET
                   │  memory (phys memory map)    │     Kernel uses this
0xFFFF888000000000 ├─────────────────────────────┤     for most accesses
                   │  LDT remap area              │
                   ├─────────────────────────────┤
                   │  Hole (guard region)          │
0xFFFF800000000000 ├═════════════════════════════┤  ← Kernel/User boundary
                   │                              │
                   │  Non-canonical address hole   │  ← Access causes #GP
                   │                              │
0x00007FFFFFFFFFFF ├═════════════════════════════┤
                   │  User Space                  │
                   │  ┌────────────────────────┐  │
                   │  │  Stack (grows down)    │  │  ~0x7FFFFFFFE000
                   │  ├────────────────────────┤  │
                   │  │  mmap region           │  │  ~0x7F0000000000
                   │  │  (shared libs, mmap)   │  │
                   │  ├────────────────────────┤  │
                   │  │  Heap (grows up)       │  │  ~0x555555559000
                   │  │  (brk / malloc)        │  │
                   │  ├────────────────────────┤  │
                   │  │  BSS (uninitialized)   │  │
                   │  ├────────────────────────┤  │
                   │  │  Data (initialized)    │  │
                   │  ├────────────────────────┤  │
                   │  │  Text (code, R/X)      │  │  ~0x555555554000
                   │  └────────────────────────┘  │
0x0000000000000000 └─────────────────────────────┘
```

### Key Kernel Memory Regions

```c
/* arch/x86/include/asm/page_64_types.h */

#define __PAGE_OFFSET_BASE      _AC(0xffff888000000000, UL)
#define __START_KERNEL_map      _AC(0xffffffff80000000, UL)

/* Physical-to-virtual conversion using direct mapping */
#define __va(x)  ((void *)((unsigned long)(x) + PAGE_OFFSET))
#define __pa(x)  ((unsigned long)(x) - PAGE_OFFSET)

/* Example:
 * Physical address 0x1000 → Virtual address 0xFFFF888000001000
 * This is the "direct mapping" — kernel can access ANY physical
 * page through this region without setting up special mappings.
 */
```

---

## 6.3 Linux Memory Zones

Linux divides physical memory into **zones** based on hardware constraints and addressability.

```
Physical Memory Zones (x86_64):

0x00000000 ┌────────────────────────┐
           │  ZONE_DMA              │  0 - 16MB
           │  (ISA DMA compatible)  │  For legacy devices that can
           │                        │  only address 24-bit (16MB)
0x01000000 ├────────────────────────┤
           │  ZONE_DMA32            │  16MB - 4GB
           │  (32-bit DMA devices)  │  For devices with 32-bit
           │                        │  address limitation
0x100000000├────────────────────────┤
           │                        │
           │  ZONE_NORMAL           │  4GB - end of RAM
           │  (Regular memory)      │  All normal allocations
           │                        │  come from here
           │                        │
           └────────────────────────┘

Note: On 32-bit systems, there's also ZONE_HIGHMEM (>896MB)
      which requires special mapping. Not used on 64-bit.
```

### Zone Definitions

```c
/* include/linux/mmzone.h */

enum zone_type {
    ZONE_DMA,       /* DMA-able pages (0-16MB on x86) */
    ZONE_DMA32,     /* DMA-able pages (0-4GB on x86_64) */
    ZONE_NORMAL,    /* Regular pages */
#ifdef CONFIG_HIGHMEM
    ZONE_HIGHMEM,   /* Pages not permanently mapped (32-bit only) */
#endif
    ZONE_MOVABLE,   /* Movable pages (for memory hotplug/CMA) */
    ZONE_DEVICE,    /* Device memory (persistent memory, GPU) */
    __MAX_NR_ZONES
};

struct zone {
    unsigned long watermark[NR_WMARK];   /* min/low/high watermarks */
    long lowmem_reserve[MAX_NR_ZONES];
    struct pglist_data *zone_pgdat;       /* Parent node */
    struct per_cpu_pages __percpu *per_cpu_pageset;
    struct free_area free_area[MAX_ORDER]; /* Buddy allocator free lists */
    unsigned long zone_start_pfn;
    atomic_long_t managed_pages;
    unsigned long spanned_pages;
    unsigned long present_pages;
    const char *name;
    /* ... */
};
```

### Memory Zone Watermarks

```
Zone memory watermark levels:

Pages Free
  ↑
  │  ┌──────────── Zone Full ──────────────────
  │  │
  │  │  Normal allocation succeeds
  │  │
  ├──┼──────────── High Watermark ─────────────  kswapd stops reclaiming
  │  │
  │  │  kswapd is woken up, starts reclaiming
  │  │
  ├──┼──────────── Low Watermark ──────────────  kswapd is active
  │  │
  │  │  Direct reclaim kicks in (allocating process reclaims)
  │  │
  ├──┼──────────── Min Watermark ──────────────  Emergency only
  │  │
  │  │  Only GFP_ATOMIC / emergency allocations allowed
  │  │
  └──┼──────────── 0 (OOM) ───────────────────  OOM killer invoked
     │
```

```c
/* Watermark values in /proc/zoneinfo */
/* 
Node 0, zone   Normal
  pages free     4015266
        boost    0
        min      31025
        low      50164
        high     69303
*/
```

---

## 6.4 Memory Nodes (NUMA Nodes)

Each NUMA node has its own set of zones, free page lists, and LRU lists.

```
NUMA System:
┌─────────────────────────┐  ┌─────────────────────────┐
│       Node 0            │  │       Node 1            │
│ ┌─────────────────────┐ │  │ ┌─────────────────────┐ │
│ │ ZONE_DMA   (16MB)   │ │  │ │ ZONE_DMA   (0)     │ │
│ ├─────────────────────┤ │  │ ├─────────────────────┤ │
│ │ ZONE_DMA32 (~4GB)  │ │  │ │ ZONE_DMA32 (0)     │ │
│ ├─────────────────────┤ │  │ ├─────────────────────┤ │
│ │ ZONE_NORMAL(~28GB)  │ │  │ │ ZONE_NORMAL(~32GB)  │ │
│ └─────────────────────┘ │  │ └─────────────────────┘ │
│ pg_data_t node_data[0]  │  │ pg_data_t node_data[1]  │
└─────────────────────────┘  └─────────────────────────┘

Zone fallback order (for Node 0 allocation):
1. Node 0, ZONE_NORMAL  (local, normal)
2. Node 0, ZONE_DMA32   (local, DMA32)
3. Node 1, ZONE_NORMAL  (remote, normal) ← slower!
4. Node 0, ZONE_DMA     (local, DMA)
5. Node 1, ZONE_DMA32   (remote, DMA32)
```

```c
/* include/linux/mmzone.h */

typedef struct pglist_data {
    struct zone node_zones[MAX_NR_ZONES];
    struct zonelist node_zonelists[MAX_ZONELISTS];
    int nr_zones;
    unsigned long node_start_pfn;
    unsigned long node_present_pages;  /* Physical pages in this node */
    unsigned long node_spanned_pages;  /* Including holes */
    int node_id;
    wait_queue_head_t kswapd_wait;
    struct task_struct *kswapd;
    /* LRU lists for page reclaim */
    struct lruvec __lruvec;
    /* ... */
} pg_data_t;

/* Array of all NUMA nodes */
extern struct pglist_data *node_data[];
#define NODE_DATA(nid)  (node_data[nid])
```

---

## 6.5 Memory Sections and Blocks

Linux uses **sections** for memory hotplug granularity and **memblocks** for early boot memory management.

### Memory Sections (SPARSEMEM)

```c
/* include/linux/mmzone.h */

/* SPARSEMEM divides physical memory into sections */
/* Each section has its own mem_map (struct page array) */

#define SECTION_SIZE_BITS    27  /* 128MB sections (x86_64) */
#define PAGES_PER_SECTION    (1UL << (SECTION_SIZE_BITS - PAGE_SHIFT))

struct mem_section {
    unsigned long section_mem_map;  /* Pointer to struct page array */
    struct mem_section_usage *usage;
    /* ... */
};

/* Memory hotplug works at section granularity */
/* online_pages() / offline_pages() operate on sections */
```

### Memblock (Early Boot Allocator)

```c
/* include/linux/memblock.h */

/* Before the buddy allocator is initialized, memblock manages memory */
struct memblock {
    struct memblock_type memory;   /* All available memory regions */
    struct memblock_type reserved; /* Reserved regions (kernel, DTB, etc.) */
};

/* Early boot allocation */
void *memblock_alloc(phys_addr_t size, phys_addr_t align);

/* View memblock state */
/* Boot log shows: */
/* memblock: memory size = 0x200000000 reserved = 0x10000000 */
```

---

## 6.6 Page Frame Abstraction

Every physical page frame in the system is represented by a `struct page` (or `struct folio` in newer kernels).

```
Physical RAM: 16GB = 4,194,304 pages (at 4KB each)
              Each needs a struct page = 64 bytes
              Total struct page array: 256MB

This is the mem_map or vmemmap:
┌──────┬──────┬──────┬──────┬──────┬─────────────────┐
│page 0│page 1│page 2│page 3│page 4│  ... page 4M    │
│ 64B  │ 64B  │ 64B  │ 64B  │ 64B  │                 │
└──────┴──────┴──────┴──────┴──────┴─────────────────┘
  PFN 0  PFN 1  PFN 2  PFN 3  PFN 4  ...

Conversion:
  struct page *p = pfn_to_page(pfn);   /* PFN → struct page */
  unsigned long pfn = page_to_pfn(p);  /* struct page → PFN */
```

---

## 6.7 struct page Overview

`struct page` is the most important data structure in Linux memory management. It's heavily union-optimized to fit in 64 bytes.

```c
/* include/linux/mm_types.h (simplified) */

struct page {
    unsigned long flags;      /* Page flags (PG_locked, PG_dirty, etc.) */
    
    union {
        struct {  /* Page cache and anonymous pages */
            union {
                struct list_head lru;       /* LRU list linkage */
                struct {                    /* Used by SLUB */
                    void *__filler;
                    unsigned int inuse;
                };
            };
            struct address_space *mapping;  /* File mapping or anon_vma */
            pgoff_t index;                  /* Offset within mapping */
            unsigned long private;          /* Used by filesystem */
        };
        
        struct {  /* Slab allocator page */
            struct kmem_cache *slab_cache;
            void *freelist;
            union {
                unsigned long counters;
                struct {
                    unsigned inuse:16;
                    unsigned objects:15;
                    unsigned frozen:1;
                };
            };
        };
        
        struct {  /* Compound (huge) page tail */
            unsigned long compound_head;  /* Bit 0 set → this is a tail page */
        };
        
        struct {  /* Page table page */
            unsigned long _pt_pad_1;
            pgtable_t pmd_huge_pte;
            unsigned long _pt_pad_2;
            spinlock_t ptl;  /* Page table lock */
        };
    };
    
    union {
        atomic_t _mapcount;        /* Count of PTEs mapping this page */
        unsigned int page_type;    /* For special pages */
    };
    
    atomic_t _refcount;            /* Reference count */
    
#ifdef CONFIG_MEMCG
    unsigned long memcg_data;      /* Memory cgroup association */
#endif
};
```

### Important Page Flags

```c
/* include/linux/page-flags.h */

enum pageflags {
    PG_locked,      /* Page is locked (I/O in progress) */
    PG_referenced,  /* Page was recently accessed (LRU) */
    PG_uptodate,    /* Page data is valid */
    PG_dirty,       /* Page has been modified */
    PG_lru,         /* Page is on an LRU list */
    PG_active,      /* Page is on active LRU list */
    PG_workingset,  /* Page is part of working set */
    PG_waiters,     /* Has waiters */
    PG_slab,        /* Used by slab allocator */
    PG_owner_priv_1,/* Used by owner (filesystem) */
    PG_private,     /* Page has private data */
    PG_writeback,   /* Page being written back to disk */
    PG_head,        /* Head page of compound page */
    PG_mappedtodisk,/* Data blocks allocated on disk */
    PG_reclaim,     /* To be reclaimed ASAP */
    PG_swapbacked,  /* Page is backed by swap (anonymous) */
    PG_unevictable, /* Page cannot be evicted */
    PG_mlocked,     /* Page is mlock()'d */
    /* ... */
};

/* Test, set, clear flags */
PageLocked(page)     /* Test PG_locked */
SetPageDirty(page)   /* Set PG_dirty */
ClearPageDirty(page) /* Clear PG_dirty */
```

### struct folio (Modern Linux 5.16+)

```c
/* include/linux/mm_types.h */

/* Folio = "Head page of a contiguous chunk" */
/* Replaces compound page logic, eliminates bugs from */
/* accidentally passing tail pages to functions */

struct folio {
    union {
        struct {
            unsigned long flags;           /* Same as page->flags */
            union {
                struct list_head lru;
            };
            struct address_space *mapping;
            pgoff_t index;
            void *private;
            atomic_t _mapcount;
            atomic_t _refcount;
        };
        struct page page;  /* Overlay with struct page */
    };
    unsigned long _flags_1;
    unsigned long __head;
    unsigned char _folio_dtor;
    unsigned char _folio_order;  /* Order (page count = 2^order) */
    atomic_t _entire_mapcount;
    atomic_t _nr_pages_mapped;
    atomic_t _pincount;
    unsigned int _folio_nr_pages;
};
```

---

## 6.8 mem_map Structure

### FLATMEM (Simple, single-node)

```
One big array of struct page covering all physical memory:

struct page *mem_map;  /* Global page array */

pfn_to_page(pfn) = mem_map + pfn;
page_to_pfn(page) = page - mem_map;

Problem: Wastes memory for holes in physical address space
```

### SPARSEMEM_VMEMMAP (Modern)

```
Virtual memory map — struct page array mapped into vmemmap region:

┌────────────────────────────────────────────────┐
│  Virtual Address Space (Kernel)                │
│                                                │
│  vmemmap region: 0xFFFFEA0000000000            │
│  ┌──────────────────────────────────────────┐  │
│  │  page[0] page[1] page[2] ... page[N]    │  │
│  │  (backed by physical pages only where    │  │
│  │   actual RAM exists — sparse!)           │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘

Advantages:
- Constant-time pfn_to_page(): vmemmap + pfn * sizeof(struct page)
- Memory-efficient: only mapped where RAM exists
- Supports memory hotplug
```

```c
/* arch/x86/include/asm/pgtable_64.h */
#define vmemmap ((struct page *)VMEMMAP_START)

/* pfn_to_page using vmemmap */
#define pfn_to_page(pfn)  (vmemmap + (pfn))
#define page_to_pfn(page) ((unsigned long)(page) - (unsigned long)vmemmap) \
                           / sizeof(struct page)
```

---

## Comparison: Memory Architecture Across OS

| Aspect | Linux | Windows | macOS | QNX |
|--------|-------|---------|-------|-----|
| Page descriptor | struct page (64B) | PFN_NUMBER / MMPFN | vm_page | Custom |
| Physical page tracking | mem_map/vmemmap | PFN database | vm_page array | Zone-based |
| Memory zones | DMA, DMA32, Normal | Kernel, User | Varies | Typed pools |
| NUMA support | Full (pg_data_t per node) | Full (KNODE_TABLE) | Limited (Apple Silicon) | Limited |
| Memory hotplug | Yes (SPARSEMEM sections) | Yes | No | Limited |
| Early allocator | memblock | MmInitializePfnDatabase | zone_init | Custom |
| Kernel virtual layout | Configurable | Fixed (MmSystemRangeStart) | Fixed | Microkernel |

---

## Interview Questions

1. **Q: What are memory zones in Linux and why do they exist?**
   A: Zones partition physical memory based on hardware constraints: ZONE_DMA (0-16MB, legacy ISA), ZONE_DMA32 (0-4GB, 32-bit DMA devices), ZONE_NORMAL (all remaining RAM). They ensure that allocation requests with specific address constraints can be satisfied.

2. **Q: What is struct page and why is it exactly 64 bytes?**
   A: struct page represents one physical page frame. It's 64 bytes because: (a) it aligns to a cache line for performance, (b) the system has millions of these (one per 4KB page), so size matters — for 16GB RAM, 4M pages × 64B = 256MB overhead. Heavy use of unions keeps it compact.

3. **Q: Explain the difference between PAGE_OFFSET and vmemmap.**
   A: PAGE_OFFSET is the start of the direct mapping region where all physical memory is linearly mapped into kernel virtual space (phys 0 → PAGE_OFFSET). vmemmap is where the struct page array is mapped — it provides the metadata for each physical page.

4. **Q: How does Linux handle NUMA allocation?**
   A: Each NUMA node has its own pg_data_t with its own zones and free lists. Allocation tries the local node first, then falls back to remote nodes. The zonelist ordering determines fallback priority. NUMA policies (MPOL_BIND, MPOL_INTERLEAVE, etc.) can override defaults.

5. **Q: What is a folio and why was it introduced?**
   A: A folio represents one or more contiguous pages (a compound allocation). It was introduced in 5.16+ to replace error-prone compound page handling. With struct page, it was easy to accidentally pass a tail page to a function expecting a head page. Folios make the API type-safe.

---

## Summary & Key Takeaways

1. Linux memory management is a layered system: VMA → Page Fault Handler → Page Cache → Allocators → Physical Pages → Zones → NUMA Nodes.
2. Kernel and user space are split in the virtual address space — kernel space is shared across all processes.
3. Memory zones (DMA, DMA32, Normal) exist to satisfy hardware addressing constraints for different device types.
4. Each NUMA node has its own zones, free lists, and LRU lists — allocation prefers local memory.
5. Every physical page is tracked by a `struct page` (64 bytes), stored in the vmemmap region.
6. Folios (5.16+) modernize the struct page abstraction for multi-page management.
7. The direct mapping (`PAGE_OFFSET`) gives the kernel fast access to any physical page.

---

*Previous: [Chapter 5 — Address Translation Mechanism](Chapter_05_Address_Translation.md)*
*Next: [Chapter 7 — Linux Virtual Memory Management](Chapter_07_Linux_Virtual_Memory.md)*
