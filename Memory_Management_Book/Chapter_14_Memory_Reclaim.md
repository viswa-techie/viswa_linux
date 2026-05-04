# Chapter 14: Memory Reclaim Mechanisms

## Chapter Overview

When the system runs low on free memory, the kernel must reclaim pages. This chapter covers memory pressure detection, LRU lists, the kswapd daemon, direct reclaim, and the modern MGLRU algorithm.

---

## 14.1 Memory Pressure Detection

```
Zone watermarks determine when reclaim triggers:

Free Pages ↓
  HIGH  ─────── kswapd sleeps (enough free memory)
  LOW   ─────── kswapd wakes up (starts background reclaim)
  MIN   ─────── Direct reclaim (allocating process must reclaim)
  0     ─────── OOM killer invoked
```

---

## 14.2 Page Reclaim Algorithms

### What Can Be Reclaimed?

```
Reclaimable:
├── Clean file pages (page cache) → Just drop them (data on disk)
├── Dirty file pages → Write back, then drop
├── Anonymous pages → Swap out, then free
├── Slab caches → Shrink (dentries, inodes)
└── Compressed swap (zswap) → Partially reclaimable

NOT reclaimable:
├── Kernel code/data
├── Page tables
├── mlock()'d pages
├── Pages under active I/O
└── Pinned pages (DMA, etc.)
```

---

## 14.3 LRU Lists

Linux tracks page age using **Least Recently Used** lists.

```c
/* Traditional LRU (pre-6.1, still available) */
enum lru_list {
    LRU_INACTIVE_ANON,  /* Inactive anonymous pages */
    LRU_ACTIVE_ANON,    /* Active anonymous pages */
    LRU_INACTIVE_FILE,  /* Inactive file pages */
    LRU_ACTIVE_FILE,    /* Active file pages */
    LRU_UNEVICTABLE,    /* Can't be reclaimed (mlock'd) */
    NR_LRU_LISTS
};
```

### Two-List LRU Promotion/Demotion

```
                    ┌──────────────┐
    New page ──────→│   INACTIVE   │──── reclaim candidate ──→ FREE
                    │    list      │
                    └──────┬───────┘
                           │ accessed again (promoted)
                           ▼
                    ┌──────────────┐
                    │    ACTIVE    │
                    │    list      │
                    └──────┬───────┘
                           │ not accessed for a while (demoted)
                           ▼
                    Back to INACTIVE → eventually reclaimed
```

---

## 14.4 Active and Inactive Pages

```c
/* Page promotion: INACTIVE → ACTIVE */
void mark_page_accessed(struct page *page)
{
    if (!PageActive(page) && !PageUnevictable(page) &&
        PageReferenced(page) && PageLRU(page)) {
        activate_page(page);  /* Move to active list */
    } else if (!PageReferenced(page)) {
        SetPageReferenced(page);  /* First access: set referenced bit */
    }
}

/* Page demotion: ACTIVE → INACTIVE */
/* Done by shrink_active_list() during reclaim */
/* Pages without PG_referenced are demoted */
```

---

## 14.5 kswapd Daemon

```c
/* mm/vmscan.c — kswapd is a per-node kernel thread */

/* One kswapd per NUMA node */
/* Woken when zone free pages drop below LOW watermark */
/* Sleeps when free pages reach HIGH watermark */

static int kswapd(void *p)
{
    pg_data_t *pgdat = (pg_data_t *)p;
    
    while (!kthread_should_stop()) {
        /* Wait for wake-up (free pages < LOW) */
        wait_event_interruptible(pgdat->kswapd_wait,
                                 need_to_reclaim(pgdat));
        
        /* Reclaim pages until HIGH watermark reached */
        balance_pgdat(pgdat, order, highest_zoneidx);
    }
}
```

---

## 14.6 Direct Reclaim

When allocation fails and kswapd hasn't freed enough, the **allocating process itself** reclaims memory.

```
Direct reclaim flow:
alloc_pages() fails fast path
       │
       ▼
__alloc_pages_slowpath()
       │
       ├── Wake kswapd (background)
       │
       ├── try_to_free_pages() ← Direct reclaim (THIS process reclaims)
       │   ├── Shrink LRU lists (evict pages)
       │   ├── Shrink slab caches
       │   └── Compact memory (if high order)
       │
       └── Retry allocation
```

### MGLRU (Multi-Gen LRU) — Linux 6.1+

```
Traditional LRU: 2 lists (active/inactive) — coarse aging
MGLRU: Multiple generations — finer aging

Generation 0 (youngest) ← new pages added here
Generation 1
Generation 2
Generation 3 (oldest) ← reclaim from here

Benefits:
- More accurate age tracking
- Better hot/cold page differentiation
- 5-10% improvement in some workloads
- Especially helps with large working sets

Enable: echo 1 > /sys/kernel/mm/lru_gen/enabled
```

---

## Interview Questions

1. **Q: What triggers page reclaim?**
   A: When zone free pages fall below LOW watermark, kswapd wakes up. Below MIN, direct reclaim activates. The OOM killer is the last resort.

2. **Q: What is the difference between kswapd and direct reclaim?**
   A: kswapd is a background kernel thread that reclaims pages asynchronously. Direct reclaim happens in the context of the allocating process — the process is stalled until pages are freed.

3. **Q: What is MGLRU?**
   A: Multi-Gen LRU (Linux 6.1+) replaces the 2-list (active/inactive) LRU with multiple generations for finer-grained page aging. It provides better performance under memory pressure.

---

## Summary & Key Takeaways

1. Memory reclaim frees pages when the system is under memory pressure.
2. Watermarks (HIGH/LOW/MIN) determine when reclaim mechanisms activate.
3. LRU lists track page age; pages move from active to inactive as they age.
4. kswapd reclaims in the background; direct reclaim stalls the allocating process.
5. MGLRU (6.1+) provides more accurate page aging with multiple generations.

---

*Next: [Chapter 15 — Swap Management](Chapter_15_Swap_Management.md)*
