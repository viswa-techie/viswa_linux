# Chapter 24: Memory Initialization

## Learning Goals
- Understand memory initialization during kernel boot
- Know memblock, zone, and page allocator setup
- Grasp NUMA awareness during memory init
- Understand slab allocator initialization

---

## 24.1 Memory Init Overview

```
Memory Initialization Timeline:

start_kernel()
  │
  ├── setup_arch()
  │   ├── early_fixmap_init()         ← Fixed virtual mappings
  │   ├── early_ioremap_init()        ← Early I/O remapping
  │   ├── setup_machine_fdt()         ← Parse DT /memory nodes
  │   │   └── early_init_dt_scan_memory() ← Register with memblock
  │   ├── arm64_memblock_init()       ← Configure memblock
  │   │   ├── memblock_remove() reserved regions
  │   │   ├── memblock_reserve() kernel image
  │   │   └── memblock_reserve() initrd
  │   └── paging_init()              ← Create kernel page tables
  │       └── map_mem()              ← Map all physical memory
  │
  ├── mm_core_init()
  │   ├── mem_init()                  ← Free memblock → buddy allocator
  │   ├── kmem_cache_init()           ← Slab allocator init
  │   └── vmalloc_init()             ← vmalloc subsystem init
  │
  └── ... rest of boot ...
```

---

## 24.2 Memblock — Early Boot Allocator

```
Memblock Architecture:

Before the page allocator exists, the kernel needs to
allocate memory. Memblock is the early boot allocator.

struct memblock {
    struct memblock_type memory;     ← All usable RAM
    struct memblock_type reserved;   ← Regions in use (kernel, DT, etc.)
};

Example system with 4GB RAM:

memblock.memory:
┌──────────────────────────────────────────┐
│  Region 0: 0x40000000 - 0x7FFFFFFF (1GB)│ ← DRAM bank 1
│  Region 1: 0x80000000 - 0x13FFFFFFF(3GB)│ ← DRAM bank 2
└──────────────────────────────────────────┘

memblock.reserved:
┌──────────────────────────────────────────┐
│  Region 0: 0x40080000 - 0x41200000      │ ← Kernel image
│  Region 1: 0x44000000 - 0x44100000      │ ← Device tree blob
│  Region 2: 0x45000000 - 0x46000000      │ ← initramfs
│  Region 3: 0x48000000 - 0x48010000      │ ← Page tables
└──────────────────────────────────────────┘

Free memory = memory regions - reserved regions
```

```c
/* Memblock API — used during early boot */

/* Add a memory region (from DT /memory node) */
memblock_add(base, size);

/* Reserve a region (make it unavailable) */
memblock_reserve(base, size);

/* Remove a region entirely (not usable RAM) */
memblock_remove(base, size);

/* Allocate from memblock */
void *ptr = memblock_alloc(size, align);

/* Example: DT memory scanning */
int __init early_init_dt_scan_memory(void)
{
    /* Iterate /memory nodes in device tree */
    for_each_of_allnodes(node) {
        if (!of_node_is_type(node, "memory"))
            continue;
        /* Read reg property for base/size pairs */
        while ((endp - reg) >= (dt_root_addr_cells + dt_root_size_cells)) {
            base = dt_mem_next_cell(dt_root_addr_cells, &reg);
            size = dt_mem_next_cell(dt_root_size_cells, &reg);
            memblock_add(base, size);
        }
    }
}
```

---

## 24.3 Page Table Creation

```
paging_init() — Create Kernel Page Tables:

Physical Memory Layout:
0x40000000 ┌──────────────────┐
           │  Kernel Image    │
           │  (.text, .data,  │
           │   .rodata, .bss) │
0x41200000 ├──────────────────┤
           │  Free RAM        │
           │  (managed by     │
           │   buddy system)  │
           │                  │
0x13FFFFFFF└──────────────────┘

Virtual Address Space (ARM64, 48-bit VA):
0xFFFF000000000000 ┌───────────────────┐ ← Kernel space start
                   │  Linear map       │
                   │  PAGE_OFFSET      │
                   │  (all phys RAM    │
                   │   mapped here)    │
                   ├───────────────────┤
                   │  vmalloc region   │
                   ├───────────────────┤
                   │  Kernel image     │
                   │  (KIMAGE_VADDR)   │
                   ├───────────────────┤
                   │  Fixmap           │
                   ├───────────────────┤
                   │  PCI I/O          │
                   ├───────────────────┤
                   │  Modules          │
0xFFFFFFFFFFFFFFFF └───────────────────┘

map_mem():
  For each memblock.memory region:
    __map_memblock(start, end)
      → __create_pgd_mapping()
        → Fill PGD → PUD → PMD → PTE entries
        → Use 2MB block mappings where possible
```

---

## 24.4 Zone Initialization

```
Memory Zones:

┌──────────────────────────────────────────────────────┐
│ Zone     │ x86_64 Range     │ Purpose                │
├──────────┼──────────────────┼────────────────────────┤
│ ZONE_DMA │ 0 - 16MB         │ ISA DMA (legacy)       │
│ ZONE_DMA32│ 0 - 4GB         │ 32-bit DMA devices     │
│ ZONE_NORMAL│ 4GB+           │ Regular allocations    │
│ ZONE_MOVABLE│ (configurable)│ Hotplug/migration      │
│ ZONE_HIGHMEM│ 896MB+ (32bit)│ 32-bit only, not ARM64 │
└──────────────────────────────────────────────────────┘

Zone initialization during boot:

free_area_init()
  │
  ├── for_each_online_node(nid):
  │   └── free_area_init_node(nid)
  │       ├── Calculate zone sizes from memblock
  │       ├── For each zone in node:
  │       │   ├── zone_init_internals(zone)
  │       │   ├── init_currently_empty_zone()
  │       │   ├── Set zone->managed_pages
  │       │   └── Set watermarks (min/low/high)
  │       └── alloc_node_mem_map()
  │           └── Allocate struct page array
  │
  └── Result: each zone tracks its free pages

Per-zone page tracking:
┌─────────────────────────────────────┐
│  Zone: ZONE_NORMAL                  │
│  ├── managed_pages: 524288 (2GB)    │
│  ├── watermark[WMARK_MIN]: 4096     │
│  ├── watermark[WMARK_LOW]: 5120     │
│  ├── watermark[WMARK_HIGH]: 6144    │
│  ├── free_area[0]: order-0 pages    │
│  ├── free_area[1]: order-1 (8KB)    │
│  ├── ...                            │
│  └── free_area[10]: order-10 (4MB)  │
└─────────────────────────────────────┘
```

---

## 24.5 Buddy Allocator Init

```
Buddy System — Page Allocator:

mem_init() → memblock_free_all()
  Transfers all free memblock regions to the buddy allocator.

Buddy system free lists (per zone):

Order 0 (4KB):    page → page → page → ...
Order 1 (8KB):    page → page → ...
Order 2 (16KB):   page → page → ...
Order 3 (32KB):   page → ...
...
Order 10 (4MB):   page → ...

Allocation: alloc_pages(gfp_mask, order)
  1. Find smallest order >= requested
  2. If exact order available → return it
  3. If not → split higher order block
     e.g., need order 2 (16KB), only order 4 (64KB) free:
       Split order 4 → 2x order 3
       Split order 3 → 2x order 2
       Return one order 2, keep others as free buddies

Free: __free_pages(page, order)
  1. Find buddy page
  2. If buddy is also free and same order → merge
  3. Repeat merge up to MAX_ORDER
```

```c
/* Page allocation API */

/* Allocate 2^order contiguous pages */
struct page *pages = alloc_pages(GFP_KERNEL, order);

/* Get a single page */
struct page *page = alloc_page(GFP_KERNEL);

/* Get virtual address directly */
unsigned long addr = __get_free_pages(GFP_KERNEL, order);

/* Free pages */
__free_pages(page, order);

/* GFP flags control allocation behavior */
GFP_KERNEL    /* Normal kernel allocation, can sleep */
GFP_ATOMIC    /* Cannot sleep (interrupt context) */
GFP_DMA       /* Must be in ZONE_DMA */
GFP_HIGHUSER  /* User-space, can be in ZONE_HIGHMEM */
```

---

## 24.6 Slab Allocator Init

```
Slab Allocator — Small Object Cache:

The buddy allocator deals in pages (4KB minimum).
Slab provides sub-page allocations (e.g., 64 bytes for an inode).

kmem_cache_init() timeline:
  │
  ├── Phase 1: Bootstrap
  │   ├── Create kmem_cache for 'struct kmem_cache' itself
  │   └── Create kmem_cache for 'struct kmem_cache_node'
  │
  ├── Phase 2: General caches
  │   ├── kmalloc-8     (8 bytes)
  │   ├── kmalloc-16    (16 bytes)
  │   ├── kmalloc-32    (32 bytes)
  │   ├── kmalloc-64    (64 bytes)
  │   ├── kmalloc-128   (128 bytes)
  │   ├── kmalloc-256   (256 bytes)
  │   ├── kmalloc-512   (512 bytes)
  │   ├── kmalloc-1k
  │   ├── kmalloc-2k
  │   ├── kmalloc-4k
  │   └── kmalloc-8k
  │
  └── Phase 3: Specialized caches (created by subsystems later)
      ├── task_struct cache
      ├── inode_cache
      ├── dentry cache
      └── ... (hundreds of caches)

# View slab caches:
$ cat /proc/slabinfo
$ slabtop
```

```c
/* Slab allocator API */

/* Allocate arbitrary size */
void *ptr = kmalloc(size, GFP_KERNEL);
kfree(ptr);

/* Allocate zeroed memory */
void *ptr = kzalloc(size, GFP_KERNEL);

/* Create a dedicated cache for a specific structure */
struct kmem_cache *cache = kmem_cache_create(
    "my_objects",          /* name */
    sizeof(struct my_obj), /* object size */
    0,                     /* alignment */
    SLAB_HWCACHE_ALIGN,    /* flags */
    NULL                   /* constructor */
);

struct my_obj *obj = kmem_cache_alloc(cache, GFP_KERNEL);
kmem_cache_free(cache, obj);
```

---

## 24.7 vmalloc Initialization

```
vmalloc — Virtually Contiguous Allocations:

vmalloc_init():
  ├── Initialize vmap_area free tree
  ├── Set up lazy TLB invalidation
  └── Ready for vmalloc() calls

Physical pages:                Virtual mapping:
┌────┐                        ┌────┐
│ P1 │ (at 0x50000)          │ V1 │ → P1
├────┤                        ├────┤
│    │ (at 0x90000)          │ V2 │ → P3
├────┤                        ├────┤
│ P3 │ (at 0xA0000)          │ V3 │ → P5
├────┤                        ├────┤
│    │                        └────┘
├────┤                     Contiguous in virtual
│ P5 │ (at 0x120000)      but scattered in physical
└────┘

Use cases:
  - Large allocations where physical contiguity not needed
  - Module loading (vmalloc'd pages for .text/.data)
  - /proc and debug buffers
```

---

## 24.8 NUMA Memory Initialization

```
NUMA Topology Discovery:

ACPI SRAT table (x86) or DT numa info (ARM64):

Node 0:                    Node 1:
  CPUs: 0-3                  CPUs: 4-7
  Memory: 0-4GB              Memory: 4-8GB
  Local latency: 10ns        Local latency: 10ns
  Remote latency: 30ns       Remote latency: 30ns

NUMA memory init:
┌──────────────────────────────────────────────┐
│  numa_init()                                 │
│  ├── Parse ACPI SRAT or DT for topology      │
│  ├── numa_register_memblks()                  │
│  │   └── Mark memblock regions with node IDs │
│  └── For each node:                           │
│      ├── Allocate pgdat (pg_data_t)          │
│      ├── Initialize zones for this node       │
│      └── Set up per-node statistics          │
│                                              │
│  Allocation policy (default):                │
│  1. Allocate from local node first           │
│  2. Fall back to nearest neighbor node       │
│  3. Last resort: any node with free memory   │
└──────────────────────────────────────────────┘
```

---

## Kernel Source References

| Function/File | Path | Purpose |
|-------|------|---------|
| memblock_add() | mm/memblock.c | Register memory region |
| memblock_reserve() | mm/memblock.c | Reserve memory region |
| paging_init() | arch/arm64/mm/mmu.c | Create kernel page tables |
| free_area_init() | mm/page_alloc.c | Initialize zones and pages |
| mem_init() | arch/*/mm/init.c | Transfer memblock to buddy |
| kmem_cache_init() | mm/slab.c or mm/slub.c | Slab allocator bootstrap |
| vmalloc_init() | mm/vmalloc.c | vmalloc subsystem init |
| alloc_pages() | mm/page_alloc.c | Buddy system page allocation |
| kmalloc() | include/linux/slab.h | General-purpose allocation |

---

## Interview Questions

**Q1: What is memblock and why is it needed?**
A: Memblock is the early boot memory allocator used before the page allocator (buddy system) is available. During early boot, the kernel needs to allocate memory for page tables, per-CPU areas, and other structures, but the buddy allocator hasn't been set up yet. Memblock tracks two simple lists — available memory regions and reserved regions — and allocates from the difference. Once the buddy system is initialized, all free memblock regions are transferred to it, and memblock is no longer used.

**Q2: How does Linux divide physical memory into zones?**
A: Linux divides memory into zones based on address constraints. ZONE_DMA (below 16MB on x86) for legacy ISA DMA, ZONE_DMA32 (below 4GB) for 32-bit DMA, ZONE_NORMAL (everything else) for general use. Each zone has its own free lists (buddy system), watermarks, and statistics. When allocating, the kernel tries the requested zone first, then falls back to lower zones. The zonelist defines the fallback order.

**Q3: How does the buddy allocator work?**
A: The buddy system manages free pages in power-of-two sized groups (order 0=4KB through order 10=4MB). When allocating order N, it finds the smallest free block >= order N. If only a larger block is available, it splits it recursively: an order N+1 block becomes two order N "buddies." When freeing, it checks if the adjacent buddy is also free — if so, they merge into a larger block. This prevents external fragmentation.

---

## Summary

- Memblock is the early boot allocator — tracks memory and reserved regions from device tree/ACPI
- `paging_init()` creates kernel page tables mapping all physical RAM
- Zones (DMA, DMA32, Normal) partition memory by address constraints
- The buddy allocator manages pages in power-of-two groups (order 0-10)
- Slab allocator provides efficient sub-page allocations (kmalloc, kmem_cache)
- vmalloc provides virtually contiguous mappings from non-contiguous physical pages
- NUMA-aware init assigns memory regions to nodes for locality-optimal allocation

---

*Next: [Chapter 25 — Root Filesystem Initialization](Chapter_25_Root_Filesystem_Init.md)*
