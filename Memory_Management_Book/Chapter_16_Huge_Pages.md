# Chapter 16: Huge Pages and Transparent Huge Pages

## Chapter Overview

Standard 4KB pages mean managing millions of PTEs for large applications. Huge pages (2MB, 1GB) reduce TLB pressure and page table overhead. This chapter covers explicit huge pages (hugetlbfs), Transparent Huge Pages (THP), and their performance benefits.

---

## 16.1 Huge Page Concept

```
Standard 4KB pages:                    2MB Huge Pages:
1GB memory = 262,144 pages             1GB memory = 512 pages
= 262,144 PTEs                         = 512 PMD entries
= 262,144 TLB entries needed           = 512 TLB entries needed
(only ~1000-2000 TLB entries exist)    (fits in TLB!)

TLB coverage with 4KB pages:  2048 entries × 4KB  = 8MB
TLB coverage with 2MB pages:  2048 entries × 2MB  = 4GB ← 500x more!
TLB coverage with 1GB pages:  2048 entries × 1GB  = 2TB
```

### Page Sizes on x86_64

```
4KB pages:   PGD → PUD → PMD → PTE → 4KB frame
2MB pages:   PGD → PUD → PMD (PS=1) → 2MB frame  (PMD is leaf)
1GB pages:   PGD → PUD (PS=1) → 1GB frame         (PUD is leaf)
```

---

## 16.2 Transparent Huge Pages (THP)

THP automatically uses 2MB pages for anonymous memory without application changes.

```c
/* User space: NO code changes needed! */
void *mem = mmap(NULL, 16*1024*1024, PROT_READ|PROT_WRITE,
                 MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
/* If THP is enabled, kernel may back this with 2MB huge pages */

/* Control THP behavior */
/* /sys/kernel/mm/transparent_hugepage/enabled */
/* always: try huge pages for all anonymous mappings */
/* madvise: only for madvise(MADV_HUGEPAGE) regions */
/* never: disable THP */

/* Per-mapping hint */
madvise(addr, length, MADV_HUGEPAGE);   /* Enable THP for this region */
madvise(addr, length, MADV_NOHUGEPAGE); /* Disable THP for this region */
```

### THP Allocation and Collapse

```
THP allocation happens:
1. On page fault: If a 2MB-aligned region is faulted, allocate huge page
2. khugepaged: Background daemon that collapses small pages into huge pages

khugepaged collapse flow:
┌────┬────┬────┬────┬────┬────┬────┬────┐
│4KB │4KB │4KB │4KB │4KB │4KB │4KB │...│  512 × 4KB pages
└────┴────┴────┴────┴────┴────┴────┴────┘
                    ↓ khugepaged scans
┌──────────────────────────────────────────┐
│              2MB Huge Page                │  Single huge page
└──────────────────────────────────────────┘

Requirements for collapse:
- All 512 pages must be present and in the same VMA
- Region must be 2MB aligned
- No special pages (zero pages OK)
```

---

## 16.3 Huge Page Allocation (hugetlbfs)

```bash
# Reserve explicit huge pages at boot
# kernel command line: hugepages=512 hugepagesz=2M

# Or at runtime
$ echo 512 > /proc/sys/vm/nr_hugepages   # Reserve 512 × 2MB = 1GB

# View huge page info
$ cat /proc/meminfo | grep -i huge
AnonHugePages:    524288 kB  # THP usage
ShmemHugePages:        0 kB
HugePages_Total:     512     # Reserved hugetlbfs pages
HugePages_Free:      256
HugePages_Rsvd:      128
HugePages_Surp:        0
Hugepagesize:       2048 kB

# Using hugetlbfs in code
#include <sys/mman.h>
void *addr = mmap(NULL, 2*1024*1024, PROT_READ|PROT_WRITE,
                  MAP_PRIVATE|MAP_ANONYMOUS|MAP_HUGETLB, -1, 0);
```

---

## 16.4 Performance Benefits

| Metric | 4KB Pages | 2MB Huge Pages | Improvement |
|--------|-----------|----------------|-------------|
| TLB misses | High | Very low | 10-100x fewer |
| Page table memory | High | Low (fewer entries) | ~500x less |
| Page fault rate | High | Lower | Fewer faults |
| Cache efficiency | Normal | Better (fewer PTEs) | Noticeable |

### When NOT to Use Huge Pages

- Small working sets (< 10MB): No benefit, wastes memory
- Sparse access patterns: Internal fragmentation (2MB minimum)
- Memory-constrained systems: Huge page allocation may fail
- Real-time systems: Huge page allocation/compaction adds latency

---

## Interview Questions

1. **Q: What are Transparent Huge Pages?**
   A: THP automatically uses 2MB pages for anonymous memory without code changes. The kernel allocates huge pages at fault time or collapses existing small pages via khugepaged.

2. **Q: What is the difference between THP and hugetlbfs?**
   A: THP is automatic and transparent (no code changes). Hugetlbfs requires explicit reservation and mmap with MAP_HUGETLB. Hugetlbfs pages are pre-reserved and guaranteed; THP may fall back to 4KB pages.

---

## Summary & Key Takeaways

1. Huge pages (2MB, 1GB) reduce TLB pressure and page table overhead.
2. THP automatically provides 2MB pages for anonymous memory.
3. hugetlbfs provides pre-reserved, guaranteed huge pages.
4. khugepaged collapses small pages into huge pages in the background.
5. Huge pages provide 10-100x reduction in TLB misses for large working sets.

---

*Next: [Chapter 17 — NUMA Memory Management](Chapter_17_NUMA.md)*
