# Chapter 9: Kernel Memory Allocators

## Chapter Overview

The buddy allocator handles page-sized allocations, but the kernel frequently needs smaller objects (task_struct: 6KB, inode: 600B, dentry: 192B). This chapter covers the sub-page allocators — **kmalloc**, **vmalloc**, **slab/SLUB/SLOB** — that efficiently manage these smaller allocations.

---

## 9.1 kmalloc Allocator

`kmalloc()` is the most common kernel memory allocation function. It allocates **physically contiguous** memory from slab caches.

```c
/* include/linux/slab.h */
void *kmalloc(size_t size, gfp_t flags);
void kfree(const void *ptr);

/* Size-specific caches behind kmalloc:
 * kmalloc-8, kmalloc-16, kmalloc-32, kmalloc-64,
 * kmalloc-128, kmalloc-256, kmalloc-512, kmalloc-1024,
 * kmalloc-2048, kmalloc-4096, kmalloc-8192
 * 
 * Request for 100 bytes → allocated from kmalloc-128 cache
 * Internal fragmentation: 28 bytes wasted
 */

/* Common usage patterns */
struct my_struct *obj = kmalloc(sizeof(*obj), GFP_KERNEL);
if (!obj)
    return -ENOMEM;
/* ... use obj ... */
kfree(obj);

/* Zeroed allocation */
struct my_struct *obj = kzalloc(sizeof(*obj), GFP_KERNEL);

/* Array allocation (with overflow checking) */
int *arr = kmalloc_array(100, sizeof(int), GFP_KERNEL);

/* Reallocation */
obj = krealloc(obj, new_size, GFP_KERNEL);
```

### kmalloc Size Classes

| Cache Name | Size | Max Useful Bytes |
|-----------|------|------------------|
| kmalloc-8 | 8 | 8 |
| kmalloc-16 | 16 | 16 |
| kmalloc-32 | 32 | 32 |
| kmalloc-64 | 64 | 64 |
| kmalloc-96 | 96 | 96 |
| kmalloc-128 | 128 | 128 |
| kmalloc-192 | 192 | 192 |
| kmalloc-256 | 256 | 256 |
| kmalloc-512 | 512 | 512 |
| kmalloc-1k | 1024 | 1024 |
| kmalloc-2k | 2048 | 2048 |
| kmalloc-4k | 4096 | 4096 |
| kmalloc-8k | 8192 | 8192 |

### kmalloc vs alloc_pages

| Feature | kmalloc | alloc_pages |
|---------|---------|-------------|
| Minimum size | 8 bytes | 4096 bytes (1 page) |
| Maximum size | ~8MB (order 11) | Same |
| Physically contiguous | Yes | Yes |
| Overhead | Slab metadata | No metadata |
| Alignment | Natural (power of 2) | Page-aligned |
| Use case | Small kernel objects | Page-sized or larger |

---

## 9.2 vmalloc Allocator

`vmalloc()` allocates **virtually contiguous** but possibly **physically non-contiguous** memory.

```c
/* include/linux/vmalloc.h */
void *vmalloc(unsigned long size);
void vfree(const void *addr);

/* How it works:
 * 1. Allocate individual pages (order-0) from buddy allocator
 * 2. Map them into contiguous virtual addresses in vmalloc region
 * 3. Set up page tables for the mapping
 *
 * Address range: 0xFFFFC90000000000 — 0xFFFFE8FFFFFFFFFF (x86_64)
 */
```

```
vmalloc memory layout:

Kernel Virtual Address Space:
┌─────────────────────────────────────────┐
│          vmalloc region                  │
│ ┌───────┐  ┌───────┐  ┌───────┐        │
│ │vmal_1 │  │vmal_2 │  │vmal_3 │        │
│ │ 3 pgs │  │ 1 pg  │  │ 5 pgs │        │
│ └───┬───┘  └───┬───┘  └───┬───┘        │
│     │          │          │              │
└─────┼──────────┼──────────┼──────────────┘
      │          │          │
Physical RAM (scattered pages):
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│   │ A │   │ B │ C │   │ D │   │ E │   │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘

vmal_1 maps to: pages A, D, E (non-contiguous!)
vmal_2 maps to: page B
vmal_3 maps to: pages C, and others...

Guard pages:
┌───────┐ ┌──────┐ ┌───────┐ ┌──────┐ ┌───────┐
│guard  │ │vmal_1│ │guard  │ │vmal_2│ │guard  │
│(unmap)│ │      │ │(unmap)│ │      │ │(unmap)│
└───────┘ └──────┘ └───────┘ └──────┘ └───────┘
Guard pages catch buffer overflows!
```

### vmalloc vs kmalloc

| Feature | kmalloc | vmalloc |
|---------|---------|---------|
| Physical contiguity | Yes | No |
| Virtual contiguity | Yes | Yes |
| Max size | ~8MB | Very large (TBs) |
| Speed | Fast (no page table setup) | Slower (page table setup) |
| Usable for DMA | Yes | No (not physically contiguous) |
| TLB pressure | Low (direct mapping) | Higher (separate mappings) |
| Use case | Small objects, DMA | Large buffers, modules |

---

## 9.3 kvmalloc

`kvmalloc()` tries `kmalloc()` first, falls back to `vmalloc()` if that fails.

```c
/* include/linux/mm.h */
void *kvmalloc(size_t size, gfp_t flags);
void kvfree(const void *addr);

/* Implementation logic:
 * if (size <= PAGE_SIZE)
 *     return kmalloc(size, flags);
 * 
 * ret = kmalloc(size, flags | __GFP_NOWARN | __GFP_NORETRY);
 * if (ret)
 *     return ret;
 * 
 * return vmalloc(size);  // Fallback
 */

/* Use kvmalloc when:
 * - You need a large buffer but don't need physical contiguity
 * - You want the performance of kmalloc when possible
 * - Typical use: network buffers, filesystem buffers
 */
```

---

## 9.4 Slab Allocator

The original slab allocator (by Jeff Bonwick, based on Solaris slab allocator) caches frequently-used objects.

### Slab Architecture

```
┌─────────────────────────────────────────────────┐
│                kmem_cache ("inode_cache")        │
│  ┌───────────────────────────────────────────┐  │
│  │ Slab 1 (1 or more pages from buddy)       │  │
│  │ ┌──────┬──────┬──────┬──────┬──────┬─────┐│  │
│  │ │obj 0 │obj 1 │obj 2 │FREE  │FREE  │meta ││  │
│  │ │(used)│(used)│(used)│      │      │     ││  │
│  │ └──────┴──────┴──────┴──────┴──────┴─────┘│  │
│  ├───────────────────────────────────────────┤  │
│  │ Slab 2 (full — all objects used)           │  │
│  │ ┌──────┬──────┬──────┬──────┬──────┬─────┐│  │
│  │ │obj 0 │obj 1 │obj 2 │obj 3 │obj 4 │meta ││  │
│  │ │(used)│(used)│(used)│(used)│(used)│     ││  │
│  │ └──────┴──────┴──────┴──────┴──────┴─────┘│  │
│  ├───────────────────────────────────────────┤  │
│  │ Slab 3 (empty — all objects free)          │  │
│  │ ┌──────┬──────┬──────┬──────┬──────┬─────┐│  │
│  │ │FREE  │FREE  │FREE  │FREE  │FREE  │meta ││  │
│  │ └──────┴──────┴──────┴──────┴──────┴─────┘│  │
│  └───────────────────────────────────────────┘  │
│                                                  │
│  Lists: full_slabs, partial_slabs, free_slabs   │
└─────────────────────────────────────────────────┘

Allocation: Take object from partial_slabs first
Deallocation: Return object; if slab becomes empty, 
              move to free_slabs (may be returned to buddy)
```

---

## 9.5 SLUB Allocator

SLUB (the Unqueued Slab Allocator) is the **default allocator since Linux 2.6.23**. It's simpler and more efficient than SLAB.

```c
/* mm/slub.c */

struct kmem_cache {
    struct kmem_cache_cpu __percpu *cpu_slab;  /* Per-CPU slab */
    unsigned int size;         /* Object size (including metadata) */
    unsigned int object_size;  /* Requested object size */
    unsigned int offset;       /* Free pointer offset */
    unsigned int min_partial;  /* Minimum partial slabs to keep */
    gfp_t allocflags;
    const char *name;
    struct list_head list;
    struct kmem_cache_node *node[MAX_NUMNODES]; /* Per-node data */
    /* ... */
};

struct kmem_cache_cpu {
    void **freelist;           /* Pointer to next free object */
    struct slab *slab;         /* Current slab being allocated from */
    unsigned long tid;         /* Transaction ID for cmpxchg */
    /* ... */
};
```

### SLUB Allocation Flow

```
kmalloc(100, GFP_KERNEL) → SLUB kmalloc-128 cache

┌──────────────────────────────────────────────┐
│ Step 1: Check per-CPU freelist               │
│         cpu_slab->freelist != NULL?          │
│         ├── YES → Take object, update freelist (lock-free!) │
│         └── NO  → Step 2                     │
├──────────────────────────────────────────────┤
│ Step 2: Current slab exhausted               │
│         Deactivate current slab              │
│         Look for partial slab on this node   │
│         ├── Found → Activate it, go to Step 1│
│         └── None  → Step 3                   │
├──────────────────────────────────────────────┤
│ Step 3: No partial slabs available           │
│         Allocate new pages from buddy        │
│         Initialize as new slab               │
│         Activate it, go to Step 1            │
└──────────────────────────────────────────────┘
```

### SLUB Object Layout

```
SLUB object (no debugging):
┌──────────────────────────────┬──────────┐
│      Object Data             │ FP / pad │
│      (object_size bytes)     │          │
└──────────────────────────────┴──────────┘

FP = Free Pointer (points to next free object when freed)
When object is in use, FP space may be used by the object.

SLUB object (with debugging):
┌──────┬──────────────┬──────┬─────────┬───────┐
│RedZ_L│  Object Data │RedZ_R│ Track   │FP/Pad │
│(guard)│             │(guard)│(alloc/  │       │
│      │              │      │ free    │       │
│      │              │      │ info)   │       │
└──────┴──────────────┴──────┴─────────┴───────┘
RedZ = Red Zone (poison values to detect buffer overflows)
```

---

## 9.6 SLOB Allocator

SLOB (Simple List Of Blocks) is a minimalist allocator for memory-constrained embedded systems.

```c
/* mm/slob.c */
/* 
 * SLOB uses a simple first-fit allocation from a list of pages
 * Very low memory overhead but poor performance
 * Used when CONFIG_SLOB is selected (embedded systems)
 * 
 * Deprecated in Linux 6.4+, planned for removal
 */
```

### Allocator Comparison

| Feature | SLAB | SLUB | SLOB |
|---------|------|------|------|
| Complexity | High | Medium | Low |
| Memory overhead | High (per-slab metadata) | Low (metadata in page struct) | Lowest |
| Per-CPU caching | Yes (arrays) | Yes (per-CPU slab) | No |
| Performance | Good | Best | Poor |
| Scalability | Good | Best | Poor |
| Debugging | Limited | Excellent | Minimal |
| Default since | 2.2 | 2.6.23 | Never (CONFIG_SLOB) |
| Status | Available | **Default** | Deprecated (6.4) |

---

## 9.7 Object Caching

### Creating Custom Slab Caches

```c
/* For frequently allocated objects, create a dedicated cache */

static struct kmem_cache *my_cache;

/* Module init */
my_cache = kmem_cache_create(
    "my_object_cache",           /* Name (visible in /proc/slabinfo) */
    sizeof(struct my_object),    /* Object size */
    0,                           /* Alignment (0 = natural) */
    SLAB_HWCACHE_ALIGN,          /* Flags: align to cache line */
    NULL                         /* Constructor function (or NULL) */
);

/* Allocate from cache */
struct my_object *obj = kmem_cache_alloc(my_cache, GFP_KERNEL);

/* Free to cache */
kmem_cache_free(my_cache, obj);

/* Destroy cache (module cleanup) */
kmem_cache_destroy(my_cache);
```

### Constructor Function

```c
/* Called once when a new slab page is allocated, initializes all objects */
void my_constructor(void *obj)
{
    struct my_object *o = obj;
    INIT_LIST_HEAD(&o->list);
    spin_lock_init(&o->lock);
    o->refcount = 0;
}

my_cache = kmem_cache_create("my_cache", sizeof(struct my_object),
                             0, 0, my_constructor);
```

---

## 9.8 Slab Cache Management

### Viewing Slab Information

```bash
# List all slab caches
$ cat /proc/slabinfo
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab>
kmalloc-256        15120   15360    256       32        2
kmalloc-128        22560   22848    128       32        1
task_struct         1560    1575   6016        5        8
inode_cache         8792    8880    600       27        4
dentry             30128   30940    192       42        2

# More detailed view
$ sudo slabtop
 Active / Total Objects (% used)    : 1286543 / 1356780 (94.8%)
 Active / Total Slabs (% used)      : 43092 / 43092 (100.0%)
 
 OBJS  ACTIVE USE OBJ SIZE SLABS OBJ/SLAB CACHE SIZE NAME
30940  30128  97%    0.19K   740      42      2960K dentry
22848  22560  98%    0.12K   714      32       714K kmalloc-128
15360  15120  98%    0.25K   960      16      3840K kmalloc-256
 8880   8792  99%    0.59K  1480       6      5920K inode_cache

# Per-CPU slab stats
$ cat /sys/kernel/slab/kmalloc-256/cpu_slabs
```

---

## Comparison: Kernel Allocators Across OS

| Feature | Linux (SLUB) | Windows (Pool) | macOS (zalloc) | QNX |
|---------|-------------|----------------|----------------|-----|
| Default allocator | SLUB | Lookaside lists + pool | Zone allocator | Custom |
| Sizes | 8B-8KB (slab), vmalloc above | Pool tags, variable | Fixed zones | Variable |
| Per-CPU caching | Yes | Yes (lookaside) | Yes (magazine) | No |
| Debugging | KASAN, SLUB debug | Pool verifier | Zone corruption check | Limited |
| Fragmentation | Mitigated by size classes | Similar | Zone-based | N/A |

---

## Interview Questions

1. **Q: When should you use kmalloc vs vmalloc?**
   A: Use kmalloc for small allocations (<PAGE_SIZE) and when physical contiguity is needed (DMA). Use vmalloc for large allocations where physical contiguity isn't required. vmalloc is slower due to page table setup.

2. **Q: How does SLUB differ from the original SLAB allocator?**
   A: SLUB stores metadata in struct page (not in a separate slab header), uses lock-free per-CPU allocation via cmpxchg, has simpler code with better debugging, and has no separate full/partial/free slab lists per CPU.

3. **Q: What is kmem_cache_create() used for?**
   A: It creates a dedicated slab cache for a specific object type. This is more efficient than general kmalloc for frequently allocated objects because it eliminates internal fragmentation and can maintain warm objects with constructors.

4. **Q: What happens if kmalloc(100, GFP_KERNEL) fails?**
   A: It returns NULL. The kernel tries reclaim and compaction before failing. You should always check kmalloc's return value. Use `__GFP_NOFAIL` only when failure would be catastrophic (rare).

---

## Summary & Key Takeaways

1. **kmalloc**: Physically contiguous, fast, for small objects (8B-8KB). Backed by SLUB cache.
2. **vmalloc**: Virtually contiguous, physically scattered. For large allocations. Slower.
3. **kvmalloc**: Tries kmalloc first, falls back to vmalloc. Best for medium-large buffers.
4. **SLUB**: Default allocator. Per-CPU lock-free allocation, metadata in struct page.
5. **Object caching**: `kmem_cache_create()` for dedicated caches of specific object types.
6. Always check allocation return values. Always use appropriate GFP flags.

---

*Previous: [Chapter 8 — Page Allocation Mechanisms](Chapter_08_Page_Allocation.md)*
*Next: [Chapter 10 — Page Cache and File System Caching](Chapter_10_Page_Cache.md)*
