# Chapter 26: Source Code Walkthrough — Key mm/ Files

## Chapter Overview

This chapter provides a guided tour of the Linux kernel memory management source code. Understanding the source structure is essential for kernel developers, bug fixers, and anyone preparing for senior-level interviews. We cover the most important files in `mm/`, their roles, and key functions.

---

## 26.1 The mm/ Directory Structure

```
linux/mm/
├── memory.c          ← Page fault handling, copy_to/from_user
├── mmap.c            ← mmap/munmap, VMA management
├── page_alloc.c      ← Buddy allocator, page allocation
├── vmscan.c          ← Page reclaim (LRU, kswapd, direct reclaim)
├── slab.c            ← SLAB allocator (legacy)
├── slab_common.c     ← Shared slab infrastructure
├── slub.c            ← SLUB allocator (default)
├── vmalloc.c         ← vmalloc, vmap, ioremap
├── memcontrol.c      ← Memory cgroup controller
├── oom_kill.c        ← OOM killer
├── swap.c            ← LRU list management
├── swapfile.c        ← Swap area management
├── swap_state.c      ← Swap cache
├── page-writeback.c  ← Dirty page writeback
├── filemap.c         ← Page cache (file mapping)
├── readahead.c       ← File readahead
├── mprotect.c        ← Memory protection changes
├── mremap.c          ← Memory region remapping
├── madvise.c         ← madvise() syscall
├── mlock.c           ← mlock/munlock
├── huge_memory.c     ← Transparent huge pages
├── hugetlb.c         ← Hugetlb filesystem / explicit huge pages
├── compaction.c      ← Memory compaction
├── migrate.c         ← Page migration (NUMA, compaction)
├── memblock.c        ← Early boot memory allocator
├── percpu.c          ← Per-CPU memory allocator
├── kasan/            ← KASAN implementation
├── kmemleak.c        ← Memory leak detector
├── kfence/           ← KFENCE implementation
├── dmapool.c         ← DMA pool allocator
├── cma.c             ← Contiguous Memory Allocator
├── init-mm.c         ← init_mm (kernel's mm_struct)
├── gup.c             ← get_user_pages (pin user pages)
├── rmap.c            ← Reverse mapping (page → VMA lookup)
├── ksm.c             ← Kernel Same-page Merging
└── userfaultfd.c     ← User-space page fault handling

include/linux/
├── mm.h              ← Core MM declarations
├── mm_types.h        ← mm_struct, vm_area_struct, struct page
├── gfp.h             ← GFP flags
├── slab.h            ← kmalloc, kmem_cache declarations
├── page-flags.h      ← PG_locked, PG_dirty, etc.
├── memcontrol.h      ← Memory cgroup declarations
└── swap.h            ← Swap declarations
```

---

## 26.2 mm/memory.c — Page Fault Handling

```c
/*
 * mm/memory.c — The core of demand paging
 * 
 * Key functions:
 */

/* Main page fault entry point (called from arch-specific code) */
vm_fault_t handle_mm_fault(struct vm_area_struct *vma,
                           unsigned long address,
                           unsigned int flags,
                           struct pt_regs *regs);

/* Walk page table levels, create if needed */
static vm_fault_t __handle_mm_fault(struct vm_area_struct *vma,
                                     unsigned long address,
                                     unsigned int flags);
/* Steps:
 * 1. pgd_offset(mm, address)     → find/create PGD entry
 * 2. p4d_alloc(mm, pgd, address) → find/create P4D entry
 * 3. pud_alloc(mm, p4d, address) → find/create PUD entry
 * 4. pmd_alloc(mm, pud, address) → find/create PMD entry
 * 5. handle_pte_fault()          → handle at PTE level
 */

/* PTE-level fault dispatch */
static vm_fault_t handle_pte_fault(struct vm_fault *vmf);
/* Dispatches to:
 * - do_anonymous_page()   → first-touch anonymous page
 * - do_fault()            → file-backed page (calls vma->vm_ops->fault)
 * - do_swap_page()        → page swapped out, bring back
 * - do_wp_page()          → copy-on-write fault
 * - do_numa_page()        → NUMA hint fault (migration)
 */

/* Copy-on-write handler */
static vm_fault_t do_wp_page(struct vm_fault *vmf);

/* Zero-page optimization for anonymous first-touch */
static vm_fault_t do_anonymous_page(struct vm_fault *vmf);

/* copy_to_user / copy_from_user implementation base */
unsigned long copy_to_user(void __user *to, const void *from, unsigned long n);
unsigned long copy_from_user(void *to, const void __user *from, unsigned long n);
```

---

## 26.3 mm/mmap.c — VMA Management

```c
/*
 * mm/mmap.c — Virtual Memory Area management
 * 
 * Manages the process address space: mmap, munmap, brk, VMA operations.
 */

/* mmap system call implementation */
unsigned long do_mmap(struct file *file, unsigned long addr,
                      unsigned long len, unsigned long prot,
                      unsigned long flags, unsigned long pgoff,
                      unsigned long *populate, struct list_head *uf);

/* Find free virtual address range */
unsigned long get_unmapped_area(struct file *file, unsigned long addr,
                                unsigned long len, unsigned long pgoff,
                                unsigned long flags);

/* munmap implementation */
int do_munmap(struct mm_struct *mm, unsigned long start, size_t len,
              struct list_head *uf);

/* brk() system call (heap management) */
SYSCALL_DEFINE1(brk, unsigned long, brk);

/* VMA operations */
struct vm_area_struct *find_vma(struct mm_struct *mm, unsigned long addr);
/* Find VMA containing addr (or next VMA above addr) */

/* VMA merging: try to merge adjacent VMAs with same properties */
struct vm_area_struct *vma_merge(...);
/* Reduces VMA count → less memory for VMA structs, faster lookup */

/* mmap_lock (was mmap_sem): protects the VMA tree */
/* Readers: mmap_read_lock(mm)  / mmap_read_unlock(mm)  */
/* Writers: mmap_write_lock(mm) / mmap_write_unlock(mm) */
/* Per-VMA locks (6.1+): vma_start_read() / vma_end_read() */
```

---

## 26.4 mm/page_alloc.c — Buddy Allocator

```c
/*
 * mm/page_alloc.c — Physical page allocation (buddy system)
 * 
 * The foundation: all memory ultimately comes from here.
 */

/* Core allocation function */
struct page *__alloc_pages(gfp_t gfp, unsigned int order,
                           int preferred_nid, nodemask_t *nodemask);
/* Steps:
 * 1. get_page_from_freelist() — fast path (watermark check)
 * 2. If fails: __alloc_pages_slowpath()
 *    a. Wake kswapd
 *    b. Try reclaim (direct reclaim)
 *    c. Try compaction
 *    d. If GFP allows: invoke OOM killer
 *    e. Retry allocation
 */

/* Free pages back to buddy */
void __free_pages(struct page *page, unsigned int order);

/* Buddy merge on free: */
/* If buddy is free and same order → merge into order+1 block */
/* Repeat until no more merging possible */

/* Per-CPU page cache (PCP): */
/* Single pages (order-0) cached per-CPU for fast alloc/free */
/* Avoids zone lock contention for most allocations */
struct per_cpu_pages {
    int count;          /* number of pages in list */
    int high;           /* high watermark (drain if exceeded) */
    int batch;          /* chunk size for buddy refill/drain */
    struct list_head lists[]; /* per-migrate-type free lists */
};

/* Zone watermarks: */
/* WMARK_MIN  — critical: direct reclaim + throttle */
/* WMARK_LOW  — wake kswapd */
/* WMARK_HIGH — kswapd target: stop reclaim */
```

---

## 26.5 mm/vmscan.c — Page Reclaim

```c
/*
 * mm/vmscan.c — Page reclaim engine
 * 
 * Frees pages when memory is low. Heart of memory management.
 */

/* kswapd: per-node background reclaim daemon */
static int kswapd(void *p);
/* Woken when free pages < WMARK_LOW */
/* Scans LRU lists, reclaims until free pages > WMARK_HIGH */

/* Direct reclaim: synchronous, called from allocation path */
static unsigned long try_to_free_pages(struct zonelist *zonelist,
                                        int order, gfp_t gfp_mask,
                                        nodemask_t *nodemask);

/* Core scanning function */
static unsigned long shrink_node(pg_data_t *pgdat,
                                  struct scan_control *sc);
/* For each cgroup (if memcg) and LRU list:
 * 1. Isolate batch of pages from LRU tail
 * 2. Try to reclaim each page:
 *    a. Clean file page → free immediately
 *    b. Dirty file page → trigger writeback → skip for now
 *    c. Anonymous page → swap out (if swap available)
 *    d. Mapped page → unmap from all page tables (rmap)
 *    e. Referenced page → rotate to active list
 */

/* LRU list management */
/* 5 LRU lists per zone per node:
 * LRU_INACTIVE_ANON  — anonymous, not recently accessed
 * LRU_ACTIVE_ANON    — anonymous, recently accessed
 * LRU_INACTIVE_FILE  — file cache, not recently accessed
 * LRU_ACTIVE_FILE    — file cache, recently accessed
 * LRU_UNEVICTABLE    — mlock'd, cannot be reclaimed
 */

/* Multi-Gen LRU (MGLRU) — Linux 6.1+ */
/* mm/vmscan.c: lru_gen_* functions */
/* Multiple generations instead of active/inactive binary split */
/* Better working set estimation, significant perf improvement */
```

---

## 26.6 mm/slub.c — SLUB Allocator

```c
/*
 * mm/slub.c — Default slab allocator since 2.6.23
 * 
 * Provides kmalloc(), kmem_cache_alloc(), etc.
 */

/* Create a new slab cache */
struct kmem_cache *kmem_cache_create(const char *name, unsigned int size,
                                      unsigned int align, slab_flags_t flags,
                                      void (*ctor)(void *));

/* Allocate object from cache */
void *kmem_cache_alloc(struct kmem_cache *s, gfp_t gfpflags);
/* Fast path (99%+ of allocations):
 * 1. Check per-CPU slab (c->freelist)
 * 2. If object available → return it (lock-free, cmpxchg)
 * 3. If per-CPU slab exhausted → get new slab from partial list
 * 4. If no partial slabs → allocate new slab from buddy allocator
 */

/* Free object back to cache */
void kmem_cache_free(struct kmem_cache *s, void *object);
/* Fast path: return to per-CPU freelist (lock-free) */

/* kmalloc implementation: */
/* Pre-created caches: kmalloc-8, kmalloc-16, ..., kmalloc-8192 */
/* kmalloc(size) → find smallest cache >= size → kmem_cache_alloc() */

/* SLUB metadata: */
struct kmem_cache {
    struct kmem_cache_cpu __percpu *cpu_slab;  /* Per-CPU slabs */
    unsigned int size;           /* Object size (with metadata) */
    unsigned int object_size;    /* Original requested size */
    unsigned int offset;         /* Free pointer offset within object */
    struct kmem_cache_order_objects oo; /* Order and objects per slab */
    struct kmem_cache_node *node[];    /* Per-node partial lists */
};
```

---

## 26.7 mm/vmalloc.c — Virtual Contiguous Allocation

```c
/*
 * mm/vmalloc.c — Allocate virtually contiguous memory
 * 
 * Pages may be physically scattered, mapped contiguously in vmalloc area.
 */

/* Main allocation function */
void *vmalloc(unsigned long size);
/* Steps:
 * 1. Allocate vm_struct to describe the region
 * 2. Find free vmalloc address range
 * 3. Allocate individual pages (page_alloc.c)
 * 4. Map pages into vmalloc VA range (update kernel page tables)
 * 5. Return virtual address
 */

void vfree(const void *addr);
/* Reverse: unmap pages, free pages, free vm_struct */

/* vmap: Map existing pages into vmalloc address space */
void *vmap(struct page **pages, unsigned int count,
           unsigned long flags, pgprot_t prot);

/* ioremap: Map physical I/O memory (device registers) */
void __iomem *ioremap(phys_addr_t phys_addr, size_t size);
/* Similar to vmap but for device MMIO, sets uncacheable PTE flags */
```

---

## 26.8 mm/oom_kill.c — OOM Killer

```c
/*
 * mm/oom_kill.c — Out-of-Memory killer
 * 
 * Last resort when allocation cannot be satisfied.
 */

/* Select victim process */
static void select_bad_process(struct oom_control *oc);
/* For each process:
 *   oom_badness(p) = (RSS + swap + page tables) 
 *                    × (1000 + oom_score_adj) / 1000
 *   Higher score → more likely to be killed
 */

/* Kill the victim */
static void oom_kill_process(struct oom_control *oc, const char *message);
/* Send SIGKILL to victim and its children (same mm) */

/* Per-process OOM adjustment */
/* /proc/<pid>/oom_score     — current OOM score (read-only) */
/* /proc/<pid>/oom_score_adj — adjustment (-1000 to 1000) */
/* -1000 = OOM-immune (critical system process) */
/* 1000  = always kill first (best-effort container) */

/* OOM flow:
 * __alloc_pages_slowpath()
 * → try reclaim → try compaction → all fail
 * → out_of_memory()
 *   → select_bad_process()  
 *   → oom_kill_process()
 *   → retry allocation (killed process frees memory)
 */
```

---

## 26.9 Key Header Files

```c
/* include/linux/mm_types.h */
struct mm_struct {
    struct maple_tree mm_mt;        /* VMA maple tree (6.1+) */
    unsigned long task_size;         /* User VA limit */
    pgd_t *pgd;                     /* Page Global Directory */
    atomic_t mm_users;               /* Address space users */
    atomic_t mm_count;               /* mm_struct reference count */
    unsigned long total_vm;          /* Total pages mapped */
    unsigned long locked_vm;         /* mlock'd pages */
    unsigned long data_vm;           /* Data pages */
    unsigned long stack_vm;          /* Stack pages */
    unsigned long start_brk, brk;   /* Heap boundaries */
    unsigned long start_stack;       /* Stack start */
    unsigned long start_code, end_code;   /* Code segment */
    unsigned long start_data, end_data;   /* Data segment */
    struct rw_semaphore mmap_lock;   /* Protects VMA tree */
};

struct vm_area_struct {
    unsigned long vm_start;          /* VMA start address */
    unsigned long vm_end;            /* VMA end address */
    pgoff_t vm_pgoff;                /* File offset in pages */
    struct file *vm_file;            /* Mapped file (or NULL) */
    vm_flags_t vm_flags;             /* Access permissions/flags */
    const struct vm_operations_struct *vm_ops; /* VMA callbacks */
    struct mm_struct *vm_mm;         /* Owning mm_struct */
};

/* include/linux/mm_types.h — struct page (simplified) */
struct page {
    unsigned long flags;             /* PG_locked, PG_dirty, etc. */
    union {
        struct {                     /* Page cache / anon page */
            struct list_head lru;    /* LRU list linkage */
            struct address_space *mapping; /* Owner */
            pgoff_t index;           /* Offset within mapping */
        };
        struct {                     /* Slab page */
            struct kmem_cache *slab_cache;
            void *freelist;          /* First free object */
        };
        struct {                     /* Compound page (huge) */
            unsigned long compound_head;
        };
    };
    atomic_t _refcount;              /* Reference count */
    atomic_t _mapcount;              /* Number of page table mappings */
    /* ... more union members ... */
};

/* include/linux/gfp.h — GFP flags */
/* GFP_KERNEL    — normal kernel allocation (can sleep) */
/* GFP_ATOMIC    — interrupt/spinlock context (cannot sleep) */
/* GFP_HIGHUSER  — user pages */
/* __GFP_ZERO    — zero-fill the allocation */
/* __GFP_DMA     — allocate from DMA zone */
/* __GFP_NOWARN  — suppress allocation failure warnings */
```

---

## 26.10 Reading Kernel Source Tips

```
How to navigate the memory management source code:

1. Start with the entry point:
   Syscall → search for SYSCALL_DEFINE in mm/*.c
   Example: SYSCALL_DEFINE6(mmap, ...) in mm/mmap.c

2. Follow the call chain:
   do_mmap() → mmap_region() → call_mmap(file, vma)
   Each function is relatively short and well-documented.

3. Use cscope/ctags/LXR:
   $ make cscope    # Generate cross-reference database
   Online: https://elixir.bootlin.com (searchable Linux source)

4. Read the comments:
   Kernel mm/ code has excellent comments explaining WHY,
   not just WHAT. The algorithms are well-documented.

5. Key commit messages:
   $ git log --oneline mm/vmscan.c | head -20
   Commit messages often explain design decisions.

6. Documentation:
   Documentation/mm/ — Kernel MM documentation
   Documentation/admin-guide/mm/ — Sysadmin MM tuning

7. Data structure first:
   Understand struct page, mm_struct, vm_area_struct FIRST.
   Then the code operating on them makes sense.
```

---

## Interview Questions

1. **Q: Which file handles page faults in the Linux kernel?**
   A: `mm/memory.c` — specifically `handle_mm_fault()` which is called from architecture-specific fault handlers. It walks page tables and dispatches to `do_anonymous_page()`, `do_fault()`, `do_swap_page()`, or `do_wp_page()` based on the fault type.

2. **Q: Where is the buddy allocator implemented?**
   A: `mm/page_alloc.c` — `__alloc_pages()` is the core allocation function, `__free_pages()` handles freeing with buddy merging. The per-CPU page cache is also here.

3. **Q: What is the role of `mm/vmscan.c`?**
   A: It implements page reclaim — the engine that frees memory when under pressure. Contains `kswapd()` (background daemon), `try_to_free_pages()` (direct reclaim), and `shrink_node()` (core scanning of LRU lists). Also contains the MGLRU implementation.

4. **Q: Why is `struct page` so complex with unions?**
   A: A `struct page` represents a physical page frame, but pages serve many purposes: page cache, anonymous memory, slab objects, compound pages (huge pages), etc. The union allows the same 64-byte structure to store different metadata depending on the page's current use, saving memory (billions of struct pages in large systems).

---

## Summary

1. **mm/memory.c**: Page fault handling — the demand paging engine.
2. **mm/mmap.c**: VMA management — mmap, munmap, brk, address space layout.
3. **mm/page_alloc.c**: Buddy allocator — physical page allocation foundation.
4. **mm/vmscan.c**: Page reclaim — kswapd, direct reclaim, LRU scanning.
5. **mm/slub.c**: SLUB allocator — kmalloc, slab caches.
6. **mm/vmalloc.c**: Virtually contiguous allocation and ioremap.
7. **mm/oom_kill.c**: Last-resort OOM killer.
8. Key data structures: `struct page`, `mm_struct`, `vm_area_struct`, `pgd_t/pte_t`.
9. Use https://elixir.bootlin.com for online kernel code browsing.

---

*Next: [Chapter 27 — Performance Tuning and Optimization](Chapter_27_Performance_Tuning.md)*
