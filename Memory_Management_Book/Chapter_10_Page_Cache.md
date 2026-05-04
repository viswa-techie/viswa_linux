# Chapter 10: Page Cache and File System Caching

## Chapter Overview

The page cache is Linux's mechanism for caching file data in memory, dramatically reducing disk I/O. Nearly every file read/write goes through the page cache. This chapter covers its architecture, writeback, dirty page management, readahead, and the evolution from buffer cache to the modern page cache.

---

## 10.1 Page Cache Architecture

```
Application: read(fd, buf, 4096)
                │
                ▼
        ┌───────────────┐
        │  VFS Layer    │
        └───────┬───────┘
                │
        ┌───────▼───────┐
        │  Page Cache   │  ← Check if page is cached
        │  (in RAM)     │
        └───────┬───────┘
            │       │
          Hit     Miss
            │       │
            ▼       ▼
        Return   ┌─────────────┐
        cached   │ Read from   │
        page     │ filesystem  │
                 │ (disk I/O)  │
                 └──────┬──────┘
                        │
                        ▼
                 Add page to cache
                 Return data

The page cache is indexed by (inode, offset):
┌──────────────────────────────────────────────┐
│  address_space (per-inode)                   │
│  ┌────────────────────────────────────────┐  │
│  │  XArray (radix tree replacement)       │  │
│  │                                        │  │
│  │  Index 0 → struct page (offset 0)     │  │
│  │  Index 1 → struct page (offset 4KB)   │  │
│  │  Index 2 → NULL (not cached)          │  │
│  │  Index 3 → struct page (offset 12KB)  │  │
│  │  ...                                   │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

```c
/* include/linux/fs.h */
struct address_space {
    struct inode *host;           /* Owner inode */
    struct xarray i_pages;        /* XArray of cached pages */
    atomic_t i_mmap_writable;     /* Count of writable shared mappings */
    struct rb_root_cached i_mmap; /* Tree of private/shared mappings */
    unsigned long nrpages;        /* Number of cached pages */
    pgoff_t writeback_index;      /* Writeback starts here */
    const struct address_space_operations *a_ops;
    unsigned long flags;
    /* ... */
};
```

---

## 10.2 File Read Caching

```c
/* How read() uses the page cache */
/* mm/filemap.c */

ssize_t generic_file_read_iter(struct kiocb *iocb, struct iov_iter *iter)
{
    /* For each page in the requested range: */
    struct folio *folio = filemap_get_folio(mapping, index);
    
    if (!folio) {
        /* Page not in cache → read from disk */
        folio = filemap_alloc_folio(mapping_gfp_mask(mapping), 0);
        /* Add to page cache */
        filemap_add_folio(mapping, folio, index, gfp);
        /* Issue I/O to fill the page */
        mapping->a_ops->read_folio(file, folio);
    }
    
    /* Wait for page to be up-to-date */
    folio_wait_locked(folio);
    
    /* Copy data to user buffer */
    copy_folio_to_iter(folio, offset, bytes, iter);
}
```

### Page Cache Lookup Performance

```
Page cache lookup: O(log n) in XArray
  → Typically a few cache-line accesses
  → ~50-200ns for a hit

Compared to disk:
  SSD: ~100,000ns (100µs)
  HDD: ~10,000,000ns (10ms)

Speedup: 500x-100,000x for cached reads!
```

---

## 10.3 Writeback Mechanism

When a page is modified (written to), it becomes **dirty** and must eventually be written back to disk.

```
Write path:
write(fd, data, size)
       │
       ▼
┌──────────────────┐
│ Find/alloc page  │  In page cache
│ in page cache    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Copy data to     │  memcpy to cached page
│ page in cache    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Mark page DIRTY  │  SetPageDirty(page)
│ Return to user   │  ← write() returns here (FAST!)
└────────┬─────────┘
         │
         │  (Later, asynchronously)
         ▼
┌──────────────────┐
│ Writeback thread │  pdflush/flush-X/bdi-default
│ writes dirty     │  (background kernel threads)
│ pages to disk    │
└──────────────────┘
```

### Writeback Triggers

```
Writeback occurs when:
1. Dirty pages exceed dirty_ratio (default: 20% of RAM)
   → All processes block on writes (synchronous writeback)

2. Dirty pages exceed dirty_background_ratio (default: 10% of RAM)
   → Background writeback threads start writing

3. Page has been dirty for > dirty_expire_centisecs (default: 30 seconds)
   → Timer-based writeback

4. sync() / fsync() / fdatasync() system calls
   → Explicit writeback

5. Memory pressure (reclaim needs clean pages)
   → Reclaim-triggered writeback
```

```bash
# View/tune writeback parameters
$ cat /proc/sys/vm/dirty_ratio          # 20 (percent of RAM)
$ cat /proc/sys/vm/dirty_background_ratio # 10
$ cat /proc/sys/vm/dirty_expire_centisecs # 3000 (30 seconds)
$ cat /proc/sys/vm/dirty_writeback_centisecs # 500 (5 seconds check interval)
```

---

## 10.4 Dirty Pages

```c
/* A page becomes dirty when written to */

/* Setting dirty flag */
void set_page_dirty(struct page *page);
void folio_mark_dirty(struct folio *folio);

/* Checking dirty flag */
int PageDirty(struct page *page);

/* The dirty page accounting */
/*
 * global_node_page_state(NR_FILE_DIRTY) → total dirty pages
 * 
 * Each zone tracks dirty page count for writeback throttling
 */

/* Writeback a single page */
int write_one_page(struct page *page);

/* Writeback range */
int filemap_fdatawrite_range(struct address_space *mapping,
                             loff_t start, loff_t end);
```

### Dirty Page Lifecycle

```
Page State Machine:

    ┌─────────┐
    │  Clean  │ ← Initial state (just read from disk)
    └────┬────┘
         │ write()
         ▼
    ┌─────────┐
    │  Dirty  │ ← Modified, not yet written to disk
    └────┬────┘
         │ writeback begins
         ▼
    ┌──────────┐
    │Writeback │ ← Being written to disk (PG_writeback set)
    └────┬─────┘
         │ I/O completes
         ▼
    ┌─────────┐
    │  Clean  │ ← Back to clean (can be reclaimed freely)
    └─────────┘
```

---

## 10.5 Page Cache vs Buffer Cache

### Historical Context

Early Linux (and Unix) had two separate caches:
- **Buffer cache**: Cached disk blocks (512-byte sectors)
- **Page cache**: Cached file data (4KB pages)

This led to **double caching** — the same data in both caches!

```
Before unification (Linux < 2.4):
┌──────────────┐     ┌──────────────┐
│  Page Cache   │     │ Buffer Cache │
│ (file pages)  │     │ (disk blocks)│
│               │     │              │
│  File data    │     │  Same data!  │  ← WASTE!
└──────────────┘     └──────────────┘

After unification (Linux 2.4+):
┌──────────────────────────────┐
│         Page Cache            │
│  (includes buffer heads for  │
│   filesystem metadata)       │
│                               │
│  All file I/O goes through   │
│  the page cache              │
└──────────────────────────────┘
```

### Buffer Heads (Still Used for Metadata)

```c
/* Buffer heads are still used for filesystem block management */
struct buffer_head {
    unsigned long b_state;          /* Buffer state flags */
    struct buffer_head *b_this_page; /* Circular list within page */
    struct page *b_page;            /* Page this buffer belongs to */
    sector_t b_blocknr;             /* Block number */
    size_t b_size;                  /* Block size */
    char *b_data;                   /* Pointer to data within page */
    struct block_device *b_bdev;    /* Block device */
    /* ... */
};
```

---

## 10.6 Readahead Mechanism

Readahead predicts which pages will be needed next and loads them from disk in advance.

```
Sequential read pattern:
read page 0 → cache miss → load from disk
read page 1 → cache miss → load from disk (detect sequential pattern!)
   ↓
Readahead activates:
   ← Prefetch pages 2, 3, 4, 5, 6, 7 in background →
read page 2 → HIT (already loaded by readahead!)
read page 3 → HIT
...
read page 7 → HIT
   ← Prefetch pages 8-15 (window grows exponentially) →
```

```c
/* mm/readahead.c */

/* Readahead window starts small and grows: */
/* Initial: 4 pages (16KB) */
/* Grows to: readahead_max (default 256 pages = 1MB) */

/* Control readahead */
/* /sys/block/sda/queue/read_ahead_kb = 1024 (default, in KB) */

/* Programmatic readahead hint */
#include <fcntl.h>
posix_fadvise(fd, offset, len, POSIX_FADV_SEQUENTIAL);  /* Hint: sequential */
posix_fadvise(fd, offset, len, POSIX_FADV_RANDOM);      /* Hint: random */
posix_fadvise(fd, offset, len, POSIX_FADV_WILLNEED);    /* Hint: prefetch */

/* Kernel internal readahead */
void page_cache_sync_readahead(struct address_space *mapping,
                               struct file_ra_state *ra,
                               struct file *filp,
                               pgoff_t index,
                               unsigned long req_count);
```

### Monitoring Page Cache

```bash
# Page cache usage
$ free -h
              total    used    free   shared  buff/cache   available
Mem:           31Gi    8.2Gi   15Gi    268Mi       8.1Gi      22Gi
#                                              ^^^^^^^^^ 
#                                              Page cache + buffers

# Detailed page cache stats
$ cat /proc/meminfo | grep -E "Cached|Buffers|Dirty|Writeback"
Buffers:          123456 kB
Cached:          8456789 kB
Dirty:             12340 kB
Writeback:             0 kB

# Drop page cache (for benchmarking)
$ echo 1 > /proc/sys/vm/drop_caches   # Free pagecache
$ echo 2 > /proc/sys/vm/drop_caches   # Free dentries+inodes  
$ echo 3 > /proc/sys/vm/drop_caches   # Free all
```

---

## Interview Questions

1. **Q: What is the page cache and why does it exist?**
   A: The page cache caches file data in RAM, keyed by (inode, offset). It exists because disk I/O is 1000-100,000x slower than RAM access. Most read() calls are satisfied from cache without touching disk.

2. **Q: What happens when you write() to a file?**
   A: Data is copied to the page cache page (allocating one if needed), the page is marked dirty, and write() returns. The actual disk write happens later via writeback threads. This makes write() fast but data isn't immediately on disk.

3. **Q: How does readahead work?**
   A: When the kernel detects sequential read patterns, it prefetches upcoming pages in the background. The readahead window starts small (16KB) and grows exponentially up to read_ahead_kb (default 1MB).

4. **Q: When are dirty pages written to disk?**
   A: When dirty pages exceed dirty_background_ratio (10%), or dirty_ratio (20%), or are older than dirty_expire_centisecs (30s), or on explicit sync/fsync, or under memory pressure.

---

## Summary & Key Takeaways

1. The page cache caches file data in RAM, indexed by (inode, page offset) using XArray.
2. Read hits are 500-100,000x faster than disk access.
3. Writes go to cache first (dirty pages), then are written back asynchronously.
4. Readahead prefetches pages ahead of sequential reads, growing the window exponentially.
5. Writeback is triggered by dirty ratios, timers, sync calls, or memory pressure.
6. The buffer cache was unified into the page cache in Linux 2.4.

---

*Previous: [Chapter 9 — Kernel Memory Allocators](Chapter_09_Kernel_Memory_Allocators.md)*
*Next: [Chapter 11 — Demand Paging](Chapter_11_Demand_Paging.md)*
