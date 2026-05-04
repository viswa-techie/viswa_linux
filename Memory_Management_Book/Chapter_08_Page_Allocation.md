# Chapter 8: Page Allocation Mechanisms

## Chapter Overview

This chapter covers the Linux **buddy allocator** — the foundational physical page allocator. Every page in the system ultimately comes from the buddy allocator. We cover how it works, allocation orders, GFP flags, fragmentation handling, and the `alloc_pages()` API.

---

## 8.1 Page Frame Allocation

When the kernel needs physical memory, it calls the page allocator. The page allocator:

1. Selects the appropriate NUMA node
2. Selects the appropriate zone (DMA, DMA32, Normal)
3. Allocates from the buddy system free lists

```
Allocation Request Flow:
alloc_pages(GFP_KERNEL, order)
       │
       ▼
┌─────────────────┐
│ Select NUMA node│ (prefer local)
└────────┬────────┘
         ▼
┌─────────────────┐
│ Select zone     │ (based on GFP flags)
│ from zonelist   │
└────────┬────────┘
         ▼
┌─────────────────┐
│ Try per-CPU     │ (fast path, order 0 only)
│ page cache      │
└────────┬────────┘
    │    │
   Hit  Miss
    │    │
    │    ▼
    │ ┌─────────────────┐
    │ │ Buddy allocator │ (free_area[order])
    │ │ search           │
    │ └────────┬────────┘
    │     │    │
    │    Found Not Found
    │     │    │
    │     │    ▼
    │     │ ┌─────────────────┐
    │     │ │ Try higher order│ (split larger block)
    │     │ └────────┬────────┘
    │     │     │    │
    │     │   Found  Not Found
    │     │     │    │
    │     │     │    ▼
    │     │     │ ┌─────────────────┐
    │     │     │ │ Reclaim/compact │ (kswapd, direct reclaim)
    │     │     │ └────────┬────────┘
    │     │     │          │
    │     │     │    ┌─────┴──────┐
    │     │     │  Success     Failure
    │     │     │    │           │
    ▼     ▼     ▼    ▼           ▼
  Return pages              OOM killer or
                            allocation failure
```

---

## 8.2 Buddy Allocator Algorithm

The buddy allocator manages physical memory in power-of-2 sized blocks.

### Core Concept

```
Memory is divided into blocks of 2^order pages:
Order 0: 1 page    (4KB)
Order 1: 2 pages   (8KB)
Order 2: 4 pages   (16KB)
...
Order 9: 512 pages (2MB)
Order 10: 1024 pages (4MB)  ← MAX_ORDER - 1 (default MAX_ORDER = 11)

Each order has a free list of available blocks of that size.
```

### How Allocation Works

```
Request: Allocate order-1 (8KB, 2 pages)

free_area[0]:  [blk] [blk] [blk]        ← order 0 (4KB blocks)
free_area[1]:  (empty!)                   ← order 1 (8KB blocks) NONE!
free_area[2]:  [====block====]            ← order 2 (16KB block) available!
free_area[3]:  ...

Step 1: Look in free_area[1] → Empty!
Step 2: Look in free_area[2] → Found a 16KB block!
Step 3: Split the 16KB block into two 8KB "buddies":
        [====block====] → [buddy_A][buddy_B]
Step 4: Return buddy_A to caller, put buddy_B in free_area[1]

After allocation:
free_area[0]:  [blk] [blk] [blk]
free_area[1]:  [buddy_B]                  ← leftover buddy
free_area[2]:  (empty)
```

### How Deallocation Works (Merging)

```
Free: Return an order-1 block at physical address P

Step 1: Find its buddy (XOR with block size)
        buddy_addr = P ^ (PAGE_SIZE << order)
        buddy_addr = P ^ 0x2000  (for order 1)

Step 2: Is the buddy free and same order?
        ├── Yes → Merge! Remove buddy from free_area[1]
        │         Create order-2 block, try to merge with ITS buddy
        │         (recursive merging up to MAX_ORDER)
        └── No  → Just add our block to free_area[1]

Example merging:
Before: free_area[1]: [buddy_B]
Free buddy_A (its neighbor):
  buddy_A + buddy_B → merge into order-2 block
  free_area[1]: (empty)
  free_area[2]: [====merged====]

This recursive merging is what makes the buddy allocator 
resistant to external fragmentation!
```

### Buddy Allocator Tree Visualization

```
Order 3 (32KB):  [================block================]
                        │                    │
                    Split into buddies:
Order 2 (16KB):  [====buddy_L====] + [====buddy_R====]
                    │          │       │          │
Order 1 (8KB):   [bud_LL][bud_LR]   [bud_RL][bud_RR]
                  │    │  │    │     │    │  │    │
Order 0 (4KB):  [a][b][c][d]       [e][f][g][h]

Buddy relationship:
- a's buddy is b (XOR with 0x1000)
- [a,b]'s buddy is [c,d] (XOR with 0x2000)
- [a,b,c,d]'s buddy is [e,f,g,h] (XOR with 0x4000)

Buddy Address Calculation:
buddy_pfn = pfn ^ (1 << order)
Example: PFN 0x100, order 1 → buddy = 0x100 ^ 0x2 = 0x102
```

---

## 8.3 Memory Orders and Block Sizes

```c
/* include/linux/mmzone.h */
#define MAX_ORDER 11  /* Default: orders 0-10 */

/* Sizes: */
/* Order 0:  1 page  = 4KB    */
/* Order 1:  2 pages = 8KB    */
/* Order 2:  4 pages = 16KB   */
/* Order 3:  8 pages = 32KB   */
/* Order 4:  16 pages = 64KB  */
/* Order 5:  32 pages = 128KB */
/* Order 6:  64 pages = 256KB */
/* Order 7:  128 pages = 512KB */
/* Order 8:  256 pages = 1MB  */
/* Order 9:  512 pages = 2MB  ← Huge page size */
/* Order 10: 1024 pages = 4MB ← Maximum allocation */
```

### Free Area Structure

```c
/* include/linux/mmzone.h */

struct free_area {
    struct list_head free_list[MIGRATE_TYPES];  /* One list per migrate type */
    unsigned long nr_free;                       /* Number of free blocks */
};

/* Zone contains free_area for each order */
struct zone {
    struct free_area free_area[MAX_ORDER];
    /* ... */
};

/* Migrate types for anti-fragmentation */
enum migratetype {
    MIGRATE_UNMOVABLE,     /* Kernel allocations (can't be moved) */
    MIGRATE_MOVABLE,       /* User pages (can be migrated/compacted) */
    MIGRATE_RECLAIMABLE,   /* Page cache (can be reclaimed) */
    MIGRATE_PCPTYPES,      /* Number of types in per-CPU lists */
    MIGRATE_HIGHATOMIC,    /* Reserved for high-order atomic allocs */
    MIGRATE_CMA,           /* Contiguous Memory Allocator region */
    MIGRATE_ISOLATE,       /* Being isolated for hotplug */
};
```

### Viewing Buddy Allocator State

```bash
$ cat /proc/buddyinfo
Node 0, zone      DMA      1    1    0    0    2    1    1    0    1    1    3
Node 0, zone    DMA32   3094 1948 1266  859  401  214   89   40   18    5  243
Node 0, zone   Normal 220498 145390 86495 45601 20116 6798 1826 492  72   0    0

# Format: counts of free blocks at each order (0-10)
# Node 0, zone Normal:
#   Order 0 (4KB):  220498 free blocks
#   Order 1 (8KB):  145390 free blocks
#   ...
#   Order 9 (2MB):  0 free blocks     ← No 2MB blocks available!
#   Order 10 (4MB): 0 free blocks
```

---

## 8.4 Page Splitting and Merging

### Splitting (on allocation)

```c
/* mm/page_alloc.c */

static inline void expand(struct zone *zone, struct page *page,
                          int low, int high,
                          struct free_area *area, int migratetype)
{
    unsigned long size = 1 << high;
    
    while (high > low) {
        high--;
        size >>= 1;
        
        /* The upper half becomes a free buddy */
        struct page *buddy = &page[size];
        
        /* Add buddy to free list at lower order */
        add_to_free_list(buddy, zone, high, migratetype);
        set_buddy_order(buddy, high);
    }
}

/* Example: Allocating order 0 from an order 3 block
 * 
 * Order 3: [AAAAAAAA]  (8 pages)
 * Split:   [AAAA][bbbb]  → bbbb goes to free_area[2]
 * Split:   [AA][cc][bbbb] → cc goes to free_area[1]  
 * Split:   [A][d][cc][bbbb] → d goes to free_area[0]
 * Return:  A (1 page, order 0)
 * 
 * Result: 3 buddies created at orders 0, 1, 2
 */
```

### Merging (on free)

```c
/* mm/page_alloc.c */

static inline void __free_one_page(struct page *page,
                                   unsigned long pfn,
                                   struct zone *zone,
                                   unsigned int order,
                                   int migratetype)
{
    while (order < MAX_ORDER - 1) {
        unsigned long buddy_pfn = __find_buddy_pfn(pfn, order);
        struct page *buddy = page + (buddy_pfn - pfn);
        
        /* Check if buddy is free and same order */
        if (!page_is_buddy(page, buddy, order))
            break;
        
        /* Remove buddy from its free list */
        del_page_from_free_list(buddy, zone, order);
        
        /* Merge: combined block starts at lower address */
        unsigned long combined_pfn = pfn & ~(1UL << order);
        page = pfn_to_page(combined_pfn);
        pfn = combined_pfn;
        order++;  /* Try to merge at next order */
    }
    
    /* Add the (possibly merged) block to the appropriate free list */
    add_to_free_list(page, zone, order, migratetype);
}
```

---

## 8.5 Memory Fragmentation Handling

### Anti-Fragmentation: Migrate Types

Linux groups pages by mobility to reduce fragmentation:

```
Physical Memory:
┌───────────────────────────────────────────────────────────┐
│ UNMOVABLE │ MOVABLE │ RECLAIMABLE │ MOVABLE │ UNMOVABLE  │
│ (kernel)  │ (user)  │ (cache)     │ (user)  │ (kernel)   │
└───────────────────────────────────────────────────────────┘

Grouping ensures that:
- Kernel (unmovable) pages cluster together
- User (movable) pages cluster together
- Cache (reclaimable) pages cluster together

This means when movable pages are freed, they form 
large contiguous free blocks (good for huge pages/DMA).
```

### Memory Compaction

When the buddy allocator can't find a large enough block, **compaction** moves movable pages to create contiguous free space.

```
Before compaction:
┌──┬────┬──┬──────┬──┬────┬──────┬──┐
│##│FREE│##│ FREE │##│FREE│ FREE │##│  ## = allocated (movable)
└──┴────┴──┴──────┴──┴────┴──────┴──┘

After compaction:
┌──┬──┬──┬──┬──────────────────────────┐
│##│##│##│##│   LARGE FREE BLOCK       │  All movable pages shifted left
└──┴──┴──┴──┴──────────────────────────┘
                  ↑ Now a huge page can be allocated!
```

```c
/* mm/compaction.c */
/* Compaction scans from both ends:
 * - Scanner from low addresses: finds movable pages
 * - Scanner from high addresses: finds free pages  
 * - Moves movable pages to free locations
 */
```

---

## 8.6 Allocation Flags (GFP Flags)

GFP (Get Free Pages) flags control **where** and **how** pages are allocated.

```c
/* include/linux/gfp.h */

/* Zone modifiers — WHERE to allocate */
#define __GFP_DMA        (1 << 0)  /* Allocate from ZONE_DMA */
#define __GFP_DMA32      (1 << 2)  /* Allocate from ZONE_DMA32 */
#define __GFP_HIGHMEM    (1 << 1)  /* Allow ZONE_HIGHMEM (32-bit) */
#define __GFP_MOVABLE    (1 << 3)  /* Allocate from ZONE_MOVABLE */

/* Action modifiers — HOW to allocate */
#define __GFP_RECLAIMABLE (1 << 4)  /* Page is reclaimable (cache) */
#define __GFP_HIGH        (1 << 5)  /* High-priority allocation */
#define __GFP_IO          (1 << 6)  /* Can start I/O (for reclaim) */
#define __GFP_FS          (1 << 7)  /* Can call into filesystem */
#define __GFP_ZERO        (1 << 8)  /* Zero the allocated memory */
#define __GFP_NOWARN      (1 << 9)  /* Don't print warning on failure */
#define __GFP_RETRY_MAYFAIL (1 << 10) /* Retry but may fail */
#define __GFP_NOFAIL      (1 << 11) /* NEVER fail (loop forever) */
#define __GFP_NORETRY     (1 << 12) /* Don't retry on failure */
#define __GFP_COMP        (1 << 14) /* Create compound page */
#define __GFP_NOMEMALLOC  (1 << 16) /* Don't use emergency reserves */
#define __GFP_HARDWALL    (1 << 17) /* Enforce cpuset memory policy */
#define __GFP_ATOMIC      (1 << 18) /* Non-sleeping allocation */

/* Composite flags (most commonly used) */
#define GFP_ATOMIC    (__GFP_HIGH | __GFP_ATOMIC | __GFP_KSWAPD_RECLAIM)
#define GFP_KERNEL    (__GFP_RECLAIM | __GFP_IO | __GFP_FS)
#define GFP_USER      (__GFP_RECLAIM | __GFP_IO | __GFP_FS | __GFP_HARDWALL)
#define GFP_HIGHUSER  (GFP_USER | __GFP_HIGHMEM)
#define GFP_DMA       (__GFP_DMA)
#define GFP_DMA32     (__GFP_DMA32)
```

### GFP Flags Decision Table

| Context | Flag to Use | Can Sleep? | Can Reclaim? |
|---------|------------|-----------|-------------|
| Normal kernel code | `GFP_KERNEL` | Yes | Yes |
| Interrupt handler | `GFP_ATOMIC` | No | No |
| User page allocation | `GFP_HIGHUSER_MOVABLE` | Yes | Yes |
| DMA buffer | `GFP_DMA` or `GFP_DMA32` | Depends | Depends |
| Softirq / bottom half | `GFP_ATOMIC` | No | No |
| Page cache | `GFP_KERNEL | __GFP_RECLAIMABLE` | Yes | Yes |
| Zero-filled page | `GFP_KERNEL | __GFP_ZERO` | Yes | Yes |
| Must not fail | `GFP_KERNEL | __GFP_NOFAIL` | Yes | Yes (loops) |

---

## 8.7 alloc_pages() Mechanism

### API Functions

```c
/* include/linux/gfp.h */

/* Allocate 2^order contiguous pages */
struct page *alloc_pages(gfp_t gfp_mask, unsigned int order);

/* Allocate single page */
#define alloc_page(gfp_mask) alloc_pages(gfp_mask, 0)

/* Allocate and return virtual address */
unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order);
unsigned long __get_free_page(gfp_t gfp_mask);
unsigned long get_zeroed_page(gfp_t gfp_mask);

/* Free pages */
void __free_pages(struct page *page, unsigned int order);
void free_pages(unsigned long addr, unsigned int order);

/* Usage examples */
struct page *p = alloc_pages(GFP_KERNEL, 0);  /* Single page */
void *addr = page_address(p);  /* Get kernel virtual address */
/* ... use the page ... */
__free_pages(p, 0);

/* Or using virtual address directly */
unsigned long addr = __get_free_pages(GFP_KERNEL | __GFP_ZERO, 2);
/* addr points to 4 contiguous zeroed pages (16KB) */
free_pages(addr, 2);
```

### Internal Implementation Path

```c
/* mm/page_alloc.c — simplified call chain */

struct page *alloc_pages(gfp_t gfp, unsigned int order)
{
    return __alloc_pages(gfp, order, preferred_nid, nodemask);
}

struct page *__alloc_pages(gfp_t gfp, unsigned int order,
                           int preferred_nid, nodemask_t *nodemask)
{
    struct page *page;
    
    /* FAST PATH: Try to allocate without reclaim */
    page = get_page_from_freelist(gfp, order, alloc_flags, ac);
    if (likely(page))
        return page;
    
    /* SLOW PATH: Allocation failed, need to reclaim/compact */
    page = __alloc_pages_slowpath(gfp, order, ac);
    return page;
}

/* The slow path does:
 * 1. Wake kswapd (background reclaim)
 * 2. Try allocation again
 * 3. Direct reclaim (calling process reclaims pages)
 * 4. Try allocation again
 * 5. Memory compaction (defragment)
 * 6. Try allocation again
 * 7. OOM killer (kill a process to free memory)
 * 8. Try allocation again
 * 9. Return NULL if still can't allocate
 */
```

### Per-CPU Page Cache (PCP)

For order-0 allocations (single pages), Linux maintains a per-CPU cache to avoid lock contention on the zone free lists.

```c
/* Each CPU has cached pages to reduce buddy allocator contention */
struct per_cpu_pages {
    int count;          /* Number of cached pages */
    int high;           /* High watermark: refill/drain threshold */
    int batch;          /* Pages to add/remove at a time */
    struct list_head lists[NR_PCP_LISTS]; /* Per-migrate-type lists */
};

/* Fast path: Single page from PCP */
/*
 * alloc_pages(GFP_KERNEL, 0)
 *   → rmqueue_pcplist()
 *     → Take page from this_cpu PCP list (no zone lock needed!)
 *     → If PCP empty, refill from buddy in batch
 */
```

---

## Flow Diagram: Complete Page Allocation

```
alloc_pages(GFP_KERNEL, order=2)    Request: 4 contiguous pages
              │
              ▼
    ┌──────────────────┐
    │ Determine zone   │  GFP_KERNEL → ZONE_NORMAL (preferred)
    │ and NUMA node    │  Current node preferred
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │ get_page_from_   │  FAST PATH
    │ freelist()       │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │ Check watermarks │  free_pages > watermark[WMARK_LOW] + order?
    └────────┬─────────┘
          │      │
         Yes     No → try next zone/node
          │
          ▼
    ┌──────────────────┐
    │ rmqueue()        │
    │ (buddy system)   │
    └────────┬─────────┘
             │
    ┌────────┴──────────┐
    │ free_area[2]      │  Look for order-2 block
    │ any free blocks?  │
    └────────┬──────────┘
          │      │
        Yes     No → try free_area[3], split
          │
          ▼
    Return struct page *
    (4 contiguous pages at known physical address)
```

---

## Interview Questions

1. **Q: Explain the buddy allocator algorithm.**
   A: The buddy allocator manages memory in power-of-2 blocks (orders 0-10). On allocation, it finds the smallest available block ≥ requested size, splitting larger blocks as needed. On free, it merges the block with its "buddy" (same-size block at an XOR'd address) recursively up to MAX_ORDER.

2. **Q: How is the buddy of a block found?**
   A: By XORing the PFN with the block size: `buddy_pfn = pfn ^ (1 << order)`. This works because buddy pairs always differ in exactly one bit corresponding to their order.

3. **Q: What are GFP flags? When would you use GFP_ATOMIC vs GFP_KERNEL?**
   A: GFP flags control allocation behavior. GFP_KERNEL allows sleeping and reclaim — use in process context. GFP_ATOMIC doesn't sleep — use in interrupt context, spinlock-held code, or any non-sleepable context.

4. **Q: What is memory compaction?**
   A: Compaction moves movable pages (user pages) to one end of a zone to create contiguous free blocks at the other end. It uses two scanners: one finding movable pages (low→high) and one finding free pages (high→low).

5. **Q: Why does the buddy allocator use per-CPU caches?**
   A: Per-CPU caches (PCP) serve order-0 allocations without taking the zone lock. This eliminates cache-line bouncing and lock contention on the zone free lists, which would be a severe bottleneck on multi-core systems.

---

## Summary & Key Takeaways

1. The buddy allocator is Linux's fundamental physical page allocator, managing memory in power-of-2 blocks.
2. Allocation splits larger blocks; freeing merges buddy pairs — maintaining large contiguous blocks.
3. GFP flags control allocation behavior: where (zone), how (can sleep? reclaim?), and what (zero? compound?).
4. Anti-fragmentation groups pages by mobility (unmovable, movable, reclaimable) to preserve large free blocks.
5. Memory compaction moves pages to defragment memory when large allocations fail.
6. Per-CPU page caches provide fast, lock-free single-page allocation.
7. The slow path escalates: kswapd → direct reclaim → compaction → OOM killer.

---

*Previous: [Chapter 7 — Linux Virtual Memory Management](Chapter_07_Linux_Virtual_Memory.md)*
*Next: [Chapter 9 — Kernel Memory Allocators](Chapter_09_Kernel_Memory_Allocators.md)*
