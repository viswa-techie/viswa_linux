# Chapter 11: Demand Paging

## Chapter Overview

Demand paging is the mechanism by which Linux delays physical memory allocation until a page is actually accessed. This lazy approach means `mmap()` and `malloc()` return instantly without allocating physical RAM. Pages are loaded only when a page fault occurs.

---

## 11.1 Concept of Lazy Allocation

```
Traditional (eager) allocation:        Linux (lazy/demand) allocation:
mmap(1GB)                              mmap(1GB) 
  → Allocate 1GB physical RAM            → Create VMA only (instant!)
  → Map all page tables                   → No physical pages
  → Takes seconds, may OOM                → No page tables
  → Returns                               → Returns in microseconds

First access to page 0:                First access to page 0:
  → Just works (already mapped)           → PAGE FAULT!
                                          → Kernel allocates 1 page (4KB)
                                          → Maps it in page table
                                          → Resumes instruction
```

### Why Lazy Allocation Works

Most programs allocate far more memory than they use:
- A program with `malloc(1GB)` may only touch 50MB
- A large array allocated but only first few elements used
- Many .so pages never referenced

```
Typical overcommit scenario:
Process A: mmap 4GB (uses 500MB)
Process B: mmap 2GB (uses 200MB)
Process C: mmap 3GB (uses 300MB)
Total mapped: 9GB
Total RAM: 4GB
Actually used: 1GB ← Fits easily!
```

---

## 11.2 Page Fault Triggers

A page fault occurs when the CPU accesses a virtual address with no valid page table entry.

```
Situations that cause page faults:

1. First access to mmap'd region         → Demand paging
2. First access to malloc'd memory        → Zero page allocation
3. Stack growth beyond current mapping    → Stack expansion
4. Access to file-backed page not in RAM  → Load from disk
5. Write to COW (copy-on-write) page      → Copy and remap
6. Access to swapped-out page             → Swap in from disk
7. NUMA page migration fault              → Migrate page
```

---

## 11.3 Page Loading Mechanism

### Anonymous Page (heap, stack, BSS)

```
1. Process accesses uninitialized heap page
2. Page fault → handle_mm_fault() → handle_pte_fault()
3. PTE is empty (pte_none) → do_anonymous_page()
4. Allocate a new physical page (zeroed)
5. Create PTE mapping VA → new physical page
6. Return to process (instruction re-executed)

Code path:
handle_mm_fault()
  → __handle_mm_fault()
    → handle_pte_fault()
      → do_anonymous_page()    (anonymous, first access)
      → do_fault()             (file-backed)
      → do_swap_page()         (swapped out)
      → do_wp_page()           (copy-on-write)
      → do_numa_page()         (NUMA migration)
```

### File-Backed Page

```
1. Process reads mmap'd file page
2. Page fault → handle_pte_fault() → do_fault()
3. Check page cache for the page
4. If not in page cache:
   a. Allocate new page
   b. Add to page cache
   c. Read data from filesystem (disk I/O)
   d. Create PTE mapping
5. If in page cache:
   a. Map the cached page
   b. No disk I/O needed!
6. Return to process
```

---

## 11.4 Zero Page Allocation

For anonymous pages (not backed by a file), the kernel provides a **zero page**.

```c
/* Optimization: Zero page sharing for reads */
/*
 * When a process first reads from anonymous memory,
 * the kernel maps the special ZERO_PAGE instead of
 * allocating a new page. All read-only zero pages
 * share this single physical page.
 *
 * Only when the process WRITES does a real page get allocated.
 */

/* arch/x86/include/asm/pgtable.h */
#define ZERO_PAGE(vaddr)  (virt_to_page(empty_zero_page))

/* mm/memory.c — do_anonymous_page() simplified */
static vm_fault_t do_anonymous_page(struct vm_fault *vmf)
{
    struct page *page;
    
    /* Read fault → map the zero page (shared, read-only) */
    if (!(vmf->flags & FAULT_FLAG_WRITE)) {
        /* Use the system-wide zero page */
        entry = pte_mkspecial(
            pfn_pte(my_zero_pfn(vmf->address), vma->vm_page_prot));
        /* Maps the zero page — no new allocation needed! */
        vmf->pte = entry;
        return 0;
    }
    
    /* Write fault → allocate real page, zero it */
    page = alloc_zeroed_user_highpage_movable(vma, vmf->address);
    if (!page)
        return VM_FAULT_OOM;
    
    /* Map the new page read-write */
    entry = mk_pte(page, vma->vm_page_prot);
    entry = pte_mkwrite(pte_mkdirty(entry));
    set_pte_at(mm, vmf->address, vmf->pte, entry);
    
    return 0;
}
```

---

## 11.5 Demand Paging Advantages

| Advantage | Explanation |
|-----------|-------------|
| Fast startup | Programs start instantly without loading all pages |
| Memory efficiency | Only used pages consume physical RAM |
| Overcommit | Can allocate more virtual memory than physical RAM |
| Sharing | Zero pages and COW pages share physical frames |
| Lazy loading | Shared libraries loaded only as code is reached |

### Performance Characteristics

```
First access to a page:
  Anonymous (zero): ~1-5µs (allocate page, zero it, update PTE)
  File-backed (cached): ~1-3µs (find in page cache, update PTE)
  File-backed (not cached): ~100µs (SSD) to ~10ms (HDD) — MAJOR page fault

Subsequent accesses: 0 overhead (TLB cached)

Cost measurement:
  $ perf stat -e page-faults ./my_program
  
  10,547 page-faults   # Total page faults during execution
```

---

## Interview Questions

1. **Q: What is demand paging?**
   A: Demand paging delays physical memory allocation until a page is actually accessed. On access, a page fault occurs, the kernel allocates a physical page, maps it, and the instruction is re-executed.

2. **Q: What is the zero page optimization?**
   A: When a process reads anonymous memory for the first time, the kernel maps a shared read-only zero page instead of allocating a new page. A real page is only allocated when the process writes.

3. **Q: What is the difference between a minor and major page fault?**
   A: Minor fault: page is in memory (page cache) but not mapped — no disk I/O. Major fault: page must be loaded from disk — expensive! Check with `time -v` or `/proc/pid/stat`.

---

## Summary & Key Takeaways

1. Demand paging = lazy allocation: virtual memory mapped but physical pages allocated on first access.
2. Page faults are the mechanism: CPU triggers fault, kernel allocates page, resumes execution.
3. Anonymous pages get zero-filled pages; file-backed pages are loaded from the page cache or disk.
4. The zero page optimization shares a single physical page for all read-only anonymous accesses.
5. This enables overcommit, fast program startup, and efficient memory usage.

---

*Previous: [Chapter 10 — Page Cache and File System Caching](Chapter_10_Page_Cache.md)*
*Next: [Chapter 12 — Page Fault Handling](Chapter_12_Page_Fault_Handling.md)*
