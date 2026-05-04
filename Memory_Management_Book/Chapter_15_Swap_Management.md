# Chapter 15: Swap Management

## Chapter Overview

Swap extends physical memory by using disk as overflow. When RAM is full, anonymous pages (heap, stack) are written to swap to free physical pages. This chapter covers swap architecture, swap cache, and tuning.

---

## 15.1 Swap Concept

```
Without swap:                          With swap:
RAM: 4GB                               RAM: 4GB + Swap: 8GB = 12GB virtual
Process needs 6GB → OOM!               Process needs 6GB → 4GB in RAM, 2GB in swap

┌─────────────────────┐               ┌─────────────────────┐
│   RAM (4GB)          │               │   RAM (4GB)          │
│   Active pages       │               │   Active/hot pages   │
│   + page cache       │               ├─────────────────────┤
│   → OUT OF MEMORY    │               │   Swap (8GB on disk) │
└─────────────────────┘               │   Inactive/cold pages│
                                       └─────────────────────┘
```

---

## 15.2 Swap Partition vs Swap File

```bash
# Swap partition
$ mkswap /dev/sda2
$ swapon /dev/sda2

# Swap file (preferred in modern setups)
$ fallocate -l 4G /swapfile
$ chmod 600 /swapfile
$ mkswap /swapfile
$ swapon /swapfile

# View swap usage
$ swapon --show
NAME       TYPE  SIZE   USED PRIO
/dev/sda2  partition 4G  1.2G  -2
/swapfile  file      4G  200M  -3

$ free -h
              total    used    free   shared  buff/cache   available
Swap:          8.0G    1.4G    6.6G
```

---

## 15.3 Page Swapping Mechanism

```
Swap Out (page eviction):
1. Reclaim selects anonymous page from inactive LRU
2. Find/allocate swap slot
3. Write page to swap slot (disk I/O)
4. Update PTE: present=0, swap entry = (type, offset)
5. Free physical page frame

PTE when swapped out (x86_64):
┌────┬──────────────────────┬────────────┬───┐
│ 0  │    Swap Offset       │ Swap Type  │ 0 │
│(NX)│    (bits 5-57)       │ (bits 1-4) │(P)│
└────┴──────────────────────┴────────────┴───┘
P=0 (not present), hardware ignores all other bits

Swap In (page fault on swapped page):
1. Page fault: PTE present=0, has swap entry
2. do_swap_page() called
3. Check swap cache (may already be in RAM)
4. Read page from swap slot (disk I/O)
5. Allocate physical page, copy data
6. Update PTE: present=1, physical address
7. Add to swap cache (for COW sharing)
8. Free swap slot
```

---

## 15.4 Swap Cache

The swap cache prevents multiple reads of the same swapped page.

```
Scenario: Parent and child share COW page, page is swapped out

Parent faults in the page:
  → Read from swap → Add to swap cache → Map in parent

Child faults in the same page:
  → Check swap cache → FOUND! → Map in child (no disk read!)

Swap cache = page cache indexed by swap entry instead of file offset
```

---

## 15.5 Swappiness Tuning

```bash
# Swappiness controls preference for swapping anonymous pages vs dropping file cache
$ cat /proc/sys/vm/swappiness
60    # Default: balanced

# Range: 0-200 (kernel 5.8+, previously 0-100)
# 0:   Never swap anonymous pages (prefer dropping file cache)
# 60:  Balanced (default)
# 100: Equal preference for file and anonymous reclaim
# 200: Strongly prefer reclaiming anonymous pages

$ echo 10 > /proc/sys/vm/swappiness  # Desktop: prefer keeping apps in RAM
$ echo 80 > /proc/sys/vm/swappiness  # Server: keep file cache for DB workloads

# Modern alternatives to traditional swap:
# zswap: Compressed swap cache in RAM (reduces actual swap I/O)
# zram: Compressed RAM block device used as swap
```

### zswap and zram

```
zswap flow:
Page eviction → Compress in RAM → Write to zswap pool
  → If pool full → Writeback to actual swap device

zram flow:  
Page eviction → Compress in RAM → Store in zram device
  → No disk I/O at all!

Compression ratios: typically 2:1 to 4:1
  → 4GB zram ≈ 8-16GB effective swap with no disk I/O!
```

---

## Interview Questions

1. **Q: What is swap and when is it used?**
   A: Swap is disk-backed storage for anonymous pages when RAM is full. The reclaim system writes cold anonymous pages to swap to free physical pages.

2. **Q: What is the swap cache?**
   A: A cache of recently swapped-in pages in RAM, indexed by swap entry. Prevents redundant disk reads when multiple processes fault in the same swapped page.

3. **Q: What does swappiness control?**
   A: The balance between reclaiming anonymous pages (swap out) vs file pages (drop cache). Higher = more swapping, lower = more file cache eviction.

---

## Summary & Key Takeaways

1. Swap extends memory by writing anonymous pages to disk.
2. Swap can be partition-based or file-based (modern preference).
3. The swap cache avoids redundant disk reads for shared/COW pages.
4. Swappiness (0-200) controls anonymous vs file reclaim preference.
5. zswap/zram provide compressed in-memory swap, dramatically reducing disk I/O.

---

*Next: [Chapter 16 — Huge Pages and Transparent Huge Pages](Chapter_16_Huge_Pages.md)*
