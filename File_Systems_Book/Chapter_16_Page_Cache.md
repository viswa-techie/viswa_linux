# Chapter 16: File Caching and the Page Cache

## Learning Goals
- Understand the unified page cache architecture
- Know how pages/folios are indexed and managed (xarray)
- Master readahead, writeback, and page reclaim
- Know dirty page thresholds and tuning parameters

---

## 16.1 Page Cache Architecture

```
The page cache is the SINGLE MOST IMPORTANT performance feature in Linux.

Every file read/write goes through the page cache:
  - read() → check page cache first → disk only on miss
  - write() → write to page cache → disk later (writeback)
  - mmap() → page table points to page cache pages directly

Page cache hit rate on typical server: 90-99%!
  → Most I/O never reaches the disk.

Architecture:
  ┌─────────────────────────────────────────────────────────┐
  │                    Page Cache                            │
  │                                                         │
  │  Per-inode: struct address_space                        │
  │    ├── struct xarray i_pages  ← indexed by file offset │
  │    │     ┌──────────────────────────────────────────┐  │
  │    │     │ Index 0 → folio (page containing bytes   │  │
  │    │     │           0 - 4095)                       │  │
  │    │     │ Index 1 → folio (bytes 4096 - 8191)      │  │
  │    │     │ Index 2 → NULL (not cached)               │  │
  │    │     │ Index 3 → folio (bytes 12288 - 16383)    │  │
  │    │     │ ...                                       │  │
  │    │     └──────────────────────────────────────────┘  │
  │    ├── unsigned long nrpages                           │
  │    └── const struct address_space_operations *a_ops    │
  │                                                         │
  │  Total: can use ALL available RAM (reclaimed as needed) │
  └─────────────────────────────────────────────────────────┘

Key: page cache is indexed by (inode, offset) → O(1) lookup
```

---

## 16.2 Pages and Folios

```
Traditional: struct page (represents one 4KB page)
Modern (5.16+): struct folio (represents one or more contiguous pages)

Why folios?
  - Compound pages had ambiguous APIs (head page vs tail page)
  - Folio = a "folio" of one or more pages, always the head
  - Simplifies page cache code
  - Enables large folios (16KB, 64KB, 2MB) for better throughput

  struct folio {
      /* Most fields from struct page */
      unsigned long flags;           /* PG_locked, PG_dirty, PG_uptodate... */
      struct address_space *mapping; /* owning address_space */
      pgoff_t index;                 /* offset in page cache (in PAGE units) */
      void *private;                 /* FS-private data */
      atomic_t _mapcount;            /* how many PTEs map this folio */
      atomic_t _refcount;            /* reference count */
      unsigned int _nr_pages;        /* number of base pages */
      /* ... */
  };

Page flags (important for FS):
  PG_locked    → Page is locked (I/O in progress)
  PG_uptodate  → Page data is current (read from disk successfully)
  PG_dirty     → Page modified, needs writeback
  PG_writeback → Page I/O in progress (being written to disk)
  PG_referenced → Recently accessed (for LRU aging)
  PG_active    → On active LRU list (frequently accessed)
  PG_lru       → On an LRU list
```

---

## 16.3 The xarray (Radix Tree)

```
Since Linux 4.20, the page cache uses xarray (replacing radix tree):

  struct address_space {
      struct xarray i_pages;   /* File offset → folio mapping */
  };

  xarray is a resizable sparse array:
  ┌─────────────────────────────────────────┐
  │  xarray root                            │
  │  ├── [0] → folio (offset 0)            │
  │  ├── [1] → folio (offset 4096)         │
  │  ├── [2] → NULL (not cached)           │
  │  ├── [3] → folio (offset 12288)        │
  │  ├── ...                               │
  │  └── [N] → folio (offset N*4096)       │
  └─────────────────────────────────────────┘

  Operations:
    xa_load(xa, index)         → Find folio at index     O(log N)
    xa_store(xa, index, folio) → Insert/replace folio    O(log N)
    xa_erase(xa, index)        → Remove folio             O(log N)
    xa_find(xa, index, max, filter) → Find next matching  O(log N)

  Internally: multi-level trie with 64-way branching
    Level 0: 64 slots
    Level 1: 64 × 64 = 4K slots
    Level 2: 64 × 64 × 64 = 256K slots
    → Compact for sparse files, efficient for dense files
```

---

## 16.4 Read Path Through Page Cache

```
Detailed read path:

  generic_file_read_iter(kiocb, iter)       [mm/filemap.c]
    │
    ▼
  filemap_read(iocb, iter, already_read)
    │
    │  for each page needed:
    │
    ├── filemap_get_pages(iocb, iter, &fbatch)
    │     │
    │     ├── filemap_get_read_batch(mapping, index, last_index, &fbatch)
    │     │     │
    │     │     ├── xa_find(&mapping->i_pages, index, last_index, XA_PRESENT)
    │     │     │     │
    │     │     │     ├── Found? → Add to batch
    │     │     │     │     Check: folio_test_uptodate(folio)?
    │     │     │     │       ├── Yes → pages ready to use
    │     │     │     │       └── No → need I/O or wait
    │     │     │     │
    │     │     │     └── Not found → need readahead
    │     │     │
    │     │     └── Return batch of found folios
    │     │
    │     ├── If batch empty (cache miss):
    │     │     page_cache_sync_readahead()
    │     │     → Schedule synchronous readahead
    │     │     → May read many pages ahead
    │     │     → Retry lookup
    │     │
    │     └── If found but not uptodate:
    │           folio_wait_locked(folio)
    │           → Wait for in-flight I/O to complete
    │
    ├── copy_folio_to_iter(folio, offset, bytes, iter)
    │     → Copy data from page cache to user buffer
    │     → Or for mmap: just install PTE (no copy)
    │
    └── If reading sequentially:
          page_cache_async_readahead()
          → Trigger async readahead for upcoming pages
```

---

## 16.5 Readahead

```
Readahead: predict sequential access and prefetch pages:

  Adaptive algorithm:
  ┌──────────────────────────────────────────────────────────┐
  │  Read 1: pages 0-0    (first access)                     │
  │    → Readahead: pages 0-3 (start small: 4 pages)         │
  │                                                          │
  │  Read 2: pages 1-1    (sequential detected!)              │
  │    → Async readahead triggered for pages 4-7              │
  │                                                          │
  │  Read 3: pages 2-2                                        │
  │    → Pages 2-3 already in cache (from readahead)          │
  │                                                          │
  │  Read 4: pages 4-4  (hit async readahead marker)          │
  │    → Window doubles: readahead pages 8-15                 │
  │                                                          │
  │  Read 5: pages 8-8  (hit async readahead marker)          │
  │    → Window doubles again: readahead pages 16-31          │
  │                                                          │
  │  Eventually: window reaches maximum (128KB default)       │
  └──────────────────────────────────────────────────────────┘

  Readahead window:
    ┌─────────────────┬──────────────────────────────┐
    │ Already read    │ Readahead window              │
    │ (in cache)      │ ┌─────────┬──────────────┐   │
    │                 │ │ Sync RA │ Async RA     │   │
    │ pages 0-15     │ │ 16-23   │ 24-31        │   │
    │                 │ │ (will   │ (kick off    │   │
    │                 │ │  wait)  │  when accessed)│  │
    └─────────────────┘ └─────────┴──────────────┘   │
                                                      │
    Max readahead: /sys/block/sda/queue/read_ahead_kb │
                   Default: 128 KB (32 pages)         │

  Random access: readahead disabled (no sequential pattern)
```

---

## 16.6 Write Path and Dirty Pages

```
Write path:

  generic_perform_write(file, iter, pos)    [mm/filemap.c]
    │
    for each page of data:
    │
    ├── grab_cache_page_write_begin(mapping, index)
    │     Find or create page in page cache
    │     Lock the page
    │
    ├── a_ops->write_begin(file, mapping, pos, len, &page, &fsdata)
    │     → ext4_write_begin()
    │     → Allocate blocks (or delayed alloc: just reserve)
    │     → Start journal transaction
    │
    ├── copy_page_from_iter(page, offset, bytes, iter)
    │     → Copy user data into page cache page
    │
    └── a_ops->write_end(file, mapping, pos, len, copied, page, fsdata)
          → ext4_write_end()
          → set_page_dirty(page)          ← Mark page DIRTY
          → update inode size and mtime
          → Unlock page

  After write_end: page is DIRTY in page cache.
  Data NOT yet on disk. write() returns to user.
```

### Dirty Page Accounting

```
Dirty pages tracked at multiple levels:

  Global:
    /proc/sys/vm/dirty_ratio        = 20  (% of total RAM)
    /proc/sys/vm/dirty_background_ratio = 10
    NR_FILE_DIRTY (vmstat counter)
    
  Per-BDI (backing device info):
    /sys/class/bdi/*/read_ahead_kb
    Each device has its own writeback workqueue
    
  Per-inode:
    address_space.nrpages (total cached pages)
    Dirty pages tracked in xarray with tags

When dirty thresholds exceeded:

  ┌──────────────────────────────────────────────────────────┐
  │ % of RAM dirty                                           │
  │                                                          │
  │ 0%        10%                  20%                  100% │
  │ ├──────────┤───────────────────┤──────────────────────┤  │
  │            ▲                   ▲                          │
  │            │                   │                          │
  │   dirty_background_ratio  dirty_ratio                    │
  │   Start background        BLOCK writers!                 │
  │   writeback               (process sleeps in             │
  │   (kworker thread)        balance_dirty_pages())         │
  └──────────────────────────────────────────────────────────┘
```

---

## 16.7 Writeback

```
Dirty pages written to disk by writeback threads:

Writeback triggers:
  1. Background: dirty_background_ratio exceeded (10% default)
  2. Periodic: dirty_writeback_centisecs timer (5 sec default)
  3. Explicit: sync(), fsync(), msync()
  4. Memory pressure: page reclaim needs free pages
  5. Throttle: dirty_ratio exceeded → block writers

Writeback path:
  kworker/bdi thread wakes up
    │
    ▼
  wb_workfn() → wb_do_writeback()
    │
    ▼
  wb_writeback()
    │
    ▼
  writeback_sb_inodes(sb)
    │
    for each dirty inode on sb->s_dirty list:
    │
    ├── writeback_single_inode(inode)
    │     │
    │     ├── do_writepages(mapping, wbc)
    │     │     │
    │     │     └── mapping->a_ops->writepages(mapping, wbc)
    │     │           │
    │     │           └── ext4_writepages(mapping, wbc)
    │     │                 ├── Find dirty pages in xarray
    │     │                 ├── ext4_da_get_blocks() → allocate blocks
    │     │                 ├── Build bios
    │     │                 └── submit_bio() → block layer → disk
    │     │
    │     └── write_inode(inode, wbc)
    │           → ext4_write_inode()
    │           → Write inode metadata to disk
    │
    └── Check: wrote enough? time expired? → stop

Tunables:
  /proc/sys/vm/dirty_background_ratio = 10
  /proc/sys/vm/dirty_ratio = 20
  /proc/sys/vm/dirty_writeback_centisecs = 500 (5 sec)
  /proc/sys/vm/dirty_expire_centisecs = 3000 (30 sec)
  
  dirty_background_bytes / dirty_bytes: absolute values (override ratio)
```

---

## 16.8 Page Reclaim (Eviction)

```
When memory is low, kernel reclaims page cache pages:

  Page reclaim: free pages to satisfy memory allocation requests

  LRU lists (per memory zone):
    ┌──────────────────────────────────────────┐
    │ Active list:     recently accessed pages  │ ← "hot" pages
    │ Inactive list:   older pages             │ ← candidates for eviction
    │                                          │
    │ File-backed:  pages from page cache       │ ← reclaimable
    │ Anonymous:    pages from heap/stack       │ ← need swap
    └──────────────────────────────────────────┘

  Reclaim algorithm:
    1. Scan inactive list, tail to head
    2. For each page:
       ├── If dirty → write back first, then reclaim
       ├── If referenced → promote to active list
       └── If clean and unreferenced → FREE (reclaim!)
    3. If active list too large: demote from active → inactive
    4. Repeat until enough free pages

  Key: CLEAN page cache pages can be freed instantly!
       (The data is on disk, just re-read if needed)
       DIRTY pages must be written back first.

  ┌────────────────────────────────────────────────┐
  │  New page added to page cache                   │
  │       │                                         │
  │       ▼                                         │
  │  Inactive list (tail)                           │
  │       │                                         │
  │       ├── Accessed? → Move to Active list       │
  │       │                                         │
  │       └── Not accessed → Evict (if clean)       │
  │                         or Writeback + Evict    │
  └────────────────────────────────────────────────┘
```

---

## 16.9 Page Cache Monitoring

```bash
# Overall memory stats
$ free -h
              total    used    free    shared  buff/cache  available
Mem:           16G     4.2G    1.8G    256M      10G        11G
                                                 ^^^
                                                 Page cache + buffer cache

# Detailed from /proc/meminfo
$ grep -E "Cached|Dirty|Writeback|Buffers" /proc/meminfo
Buffers:          234567 kB    # Buffer cache (block device metadata)
Cached:          8765432 kB    # Page cache (file data)
Dirty:             12345 kB    # Dirty pages (pending writeback)
Writeback:             0 kB    # Currently being written

# Per-process page cache usage
$ cat /proc/self/smaps | grep "Referenced"

# vmstat: watch dirty pages and I/O
$ vmstat 1
procs -----memory----- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy
 1  0      0 1800000 234567 8765432  0    0     0   100  500  800  5  3
                                                     ^^^
                                                     bo = blocks written/s

# Check cache hit rate with cachestat (perf)
$ sudo perf stat -e 'cache-references,cache-misses' -- cat /etc/passwd > /dev/null

# Drop page cache (for testing)
$ echo 1 > /proc/sys/vm/drop_caches    # Drop page cache
$ echo 2 > /proc/sys/vm/drop_caches    # Drop dentries + inodes
$ echo 3 > /proc/sys/vm/drop_caches    # Drop all
```

---

## 16.10 Buffer Cache vs Page Cache

```
Historical: Separate buffer cache (block device) + page cache (files)
Since Linux 2.4: UNIFIED page cache + buffer cache

Current:
  Page cache: caches file data (indexed by inode + offset)
  Buffer heads: track per-block state within a page
    ┌────────────────────────────────────┐
    │  One page (4096 bytes)             │
    │  ┌───────────┬───────────┐        │
    │  │ Block 0   │ Block 1   │        │  (if block_size = 2048)
    │  │ (bh 0)    │ (bh 1)    │        │
    │  └───────────┴───────────┘        │
    │  Each buffer_head tracks:         │
    │    - Block state (dirty, uptodate)│
    │    - Physical block number        │
    │    - Reference to owning page     │
    └────────────────────────────────────┘

  Buffer cache in /proc/meminfo: "Buffers" line
    = metadata blocks read via block device directly
    = sb reads, GDT reads, journal reads
    (Much smaller than Cached: file data)
```

---

## Kernel Source References

```
Page cache core:
  mm/filemap.c             ← filemap_read(), generic_perform_write()
  mm/filemap.c             ← filemap_fault(), filemap_page_mkwrite()
  include/linux/pagemap.h  ← Page cache API declarations

Readahead:
  mm/readahead.c           ← page_cache_sync_readahead(), ondemand_readahead()

Writeback:
  mm/page-writeback.c      ← balance_dirty_pages(), writeback thresholds
  fs/fs-writeback.c        ← writeback_sb_inodes(), wb_workfn()

Page reclaim:
  mm/vmscan.c              ← shrink_page_list(), reclaim logic
  mm/workingset.c          ← Working set detection

Folio:
  include/linux/mm_types.h ← struct folio
  mm/folio-compat.c        ← Folio compatibility helpers

xarray:
  lib/xarray.c             ← xarray implementation
  include/linux/xarray.h   ← xarray API
```

---

## Interview Questions

1. **What is the page cache? Why is it the most important Linux performance feature?**
2. **How are pages indexed in the page cache? Explain the xarray.**
3. **How does readahead work? How does the kernel detect sequential access?**
4. **What happens when dirty pages exceed dirty_background_ratio?**
5. **What is the difference between dirty_background_ratio and dirty_ratio?**
6. **How does page reclaim decide which pages to evict?**
7. **What is the difference between Active and Inactive LRU lists?**
8. **What is a folio? Why was it introduced?**
9. **How do you monitor page cache hit rate and dirty pages?**
10. **What is the unified buffer cache? How do buffer heads relate to pages?**

---

## Summary

- Page cache: caches ALL file data in RAM, indexed by (inode, file_offset)
- xarray: efficient sparse array mapping file offsets to folios
- Read: check page cache → hit or miss → readahead for sequential access
- Write: modify page in cache → mark dirty → writeback later
- Dirty thresholds: background (10%) starts writeback, ratio (20%) blocks writers
- Writeback: kworker threads write dirty pages through FS a_ops->writepages()
- Page reclaim: evict clean pages from inactive LRU; write back dirty pages first
- Folio: modern abstraction replacing raw struct page for page cache
- Monitoring: /proc/meminfo, vmstat, free, drop_caches
- Page cache hit rate >95% typical → most I/O answered from RAM

---

*Next: [Chapter 17 — File System Performance](Chapter_17_Performance.md)*
