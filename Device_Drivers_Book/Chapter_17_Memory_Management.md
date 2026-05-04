# Chapter 17: Memory Management for Drivers

## Chapter Overview

Drivers need memory for buffers, descriptor rings, device state, and DMA. This chapter covers kernel memory allocation APIs, their constraints, and when to use each.

---

## 17.1 Kernel Memory Allocation

### The Allocation Landscape

```
┌─────────────────────────────────────────────────────┐
│                Driver Memory Needs                   │
├──────────┬───────────┬──────────┬───────────────────┤
│ Small    │ Large     │ DMA      │ Per-CPU           │
│ objects  │ buffers   │ buffers  │ data              │
│(< page)  │(> page)   │(coherent)│                   │
├──────────┼───────────┼──────────┼───────────────────┤
│ kmalloc  │ vmalloc   │ dma_     │ alloc_percpu     │
│ kzalloc  │ kvmalloc  │ alloc_   │ per_cpu()        │
│ devm_*   │ devm_*    │ coherent │                   │
└──────────┴───────────┴──────────┴───────────────────┘
     │           │           │
     ▼           ▼           ▼
   SLUB       Page        CMA / low
  allocator   allocator   memory
```

---

## 17.2 kmalloc Usage in Drivers

```c
#include <linux/slab.h>

/* Basic allocation */
void *buf = kmalloc(size, GFP_KERNEL);
if (!buf)
    return -ENOMEM;

/* Zero-initialized */
void *buf = kzalloc(size, GFP_KERNEL);

/* Array allocation (overflow-safe) */
void *arr = kmalloc_array(n, sizeof(*arr), GFP_KERNEL);
void *arr = kcalloc(n, sizeof(*arr), GFP_KERNEL);

/* Free */
kfree(buf);

/* Managed (auto-freed on device removal) */
void *buf = devm_kzalloc(dev, size, GFP_KERNEL);
/* No need to kfree — freed automatically */
```

### GFP Flags

| Flag | Context | Can Sleep | Reclaim | Use Case |
|------|---------|-----------|---------|----------|
| `GFP_KERNEL` | Process | Yes | Yes | Normal driver code |
| `GFP_ATOMIC` | Any | No | No | IRQ, spinlock |
| `GFP_NOIO` | Process | Yes | No I/O | Block driver (avoid recursion) |
| `GFP_DMA` | Any | — | — | ISA DMA (<16MB) |
| `GFP_KERNEL | __GFP_ZERO` | Process | Yes | Yes | Zero-filled (= kzalloc) |

### kmalloc Size Limits

```
kmalloc caches: 8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096, 8192
Maximum: typically 4MB (order-10 buddy allocation)
For larger: use vmalloc or kvmalloc
```

---

## 17.3 vmalloc Usage

```c
#include <linux/vmalloc.h>

/* Virtually contiguous, physically scattered */
void *buf = vmalloc(1024 * 1024);   /* 1MB */
if (!buf) return -ENOMEM;
vfree(buf);

/* Zero-initialized */
void *buf = vzalloc(size);
```

### kmalloc vs vmalloc

| Feature | kmalloc | vmalloc |
|---------|---------|---------|
| Physical | Contiguous | Scattered |
| Virtual | Contiguous | Contiguous |
| Max size | ~4MB | Limited by VA |
| Speed | Fast (no page table setup) | Slower |
| DMA-able | Yes (physical contiguous) | No (without IOMMU) |
| Use case | Small objects, DMA | Large buffers |

### kvmalloc: Best of Both

```c
/* Tries kmalloc first; falls back to vmalloc */
void *buf = kvmalloc(size, GFP_KERNEL);
kvfree(buf);   /* Frees either kmalloc or vmalloc memory */
```

---

## 17.4 Memory Pools

Pre-allocated pools for guaranteed allocation in critical paths:

```c
#include <linux/mempool.h>

mempool_t *pool;

/* Create pool of 16 pre-allocated 256-byte objects */
pool = mempool_create_kmalloc_pool(16, 256);
if (!pool)
    return -ENOMEM;

/* Allocate from pool (NEVER fails if pool has reserves) */
void *obj = mempool_alloc(pool, GFP_KERNEL);

/* Return to pool */
mempool_free(obj, pool);

/* Destroy pool */
mempool_destroy(pool);
```

---

## 17.5 Scatter-Gather Buffers

```c
#include <linux/scatterlist.h>

/* Scatter-gather list for fragmented buffers */
struct scatterlist sg[4];
sg_init_table(sg, 4);

/* Set entries */
sg_set_buf(&sg[0], buf0, len0);
sg_set_buf(&sg[1], buf1, len1);
sg_set_page(&sg[2], page, len2, offset);

/* Iterate */
struct scatterlist *s;
int i;
for_each_sg(sg, s, 4, i) {
    void *addr = sg_virt(s);
    unsigned int len = s->length;
}
```

---

## Memory Allocation Decision Tree

```
Need memory in a driver?
       │
       ├── Small object (< ~128KB)?
       │   ├── Process context → devm_kzalloc(dev, size, GFP_KERNEL)
       │   ├── IRQ context → kmalloc(size, GFP_ATOMIC)
       │   └── Need zero-init → devm_kzalloc() or kzalloc()
       │
       ├── Large buffer (> 128KB)?
       │   ├── Must be phys-contiguous (DMA)? → dma_alloc_coherent()
       │   ├── Not for DMA → devm_kvmalloc() or vzalloc()
       │   └── Uncertain size → kvmalloc() (tries kmalloc, falls back)
       │
       ├── DMA buffer?
       │   ├── Long-lived coherent → dma_alloc_coherent()
       │   ├── Per-transfer → dma_map_single/sg()
       │   └── See Chapter 18
       │
       └── Must never fail (critical path)?
               └── mempool
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| `include/linux/slab.h` | kmalloc, kzalloc, kcalloc |
| `mm/slub.c` | SLUB allocator |
| `mm/vmalloc.c` | vmalloc implementation |
| `include/linux/gfp.h` | GFP flags |
| `include/linux/mempool.h` | Memory pool API |
| `include/linux/scatterlist.h` | SG list API |

---

## Interview Questions

**Q1: What is the difference between `kmalloc()` and `vmalloc()`?**
A: kmalloc returns physically and virtually contiguous memory from the SLUB allocator (fast, limited to ~4MB). vmalloc returns virtually contiguous but physically scattered memory (slower, for large buffers). kmalloc is suitable for DMA; vmalloc is not.

**Q2: Why is `GFP_ATOMIC` needed for interrupt context?**
A: GFP_KERNEL may sleep (trigger reclaim, I/O). Interrupt context cannot sleep. GFP_ATOMIC tells the allocator to never sleep — it uses only pre-allocated emergency reserves.

**Q3: What are `devm_*` allocations?**
A: Device-managed allocations tied to the device's lifecycle. Automatically freed when the device is unbound (remove()) or probe fails. Prevents resource leaks. Always prefer devm_* in drivers.

---

*Next: [Chapter 18 — Direct Memory Access (DMA)](Chapter_18_DMA.md)*
