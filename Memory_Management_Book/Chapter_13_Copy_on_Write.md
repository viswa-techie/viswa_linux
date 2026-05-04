# Chapter 13: Copy-on-Write Mechanism

## Chapter Overview

Copy-on-Write (COW) is one of the most elegant optimizations in virtual memory. It allows `fork()` to be nearly instantaneous by sharing all memory pages between parent and child, only copying when either process writes. This chapter covers the mechanism, implementation, and its critical role in process creation.

---

## 13.1 fork() System Call Memory Behavior

```
Before fork():
┌─────────────┐        Physical RAM:
│ Parent       │        ┌────┬────┬────┬────┐
│ VA 0x1000 ───┼──────→│ P1 │ P2 │ P3 │ P4 │
│ VA 0x2000 ───┼──────→│    │    │    │    │
│ VA 0x3000 ───┼──────→│    │    │    │    │
│ VA 0x4000 ───┼──────→│    │    │    │    │
└─────────────┘        └────┴────┴────┴────┘
                        4 pages, each R/W

After fork() with COW:
┌─────────────┐        Physical RAM:
│ Parent       │        ┌────┬────┬────┬────┐
│ VA 0x1000 ───┼──R/O─→│ P1 │ P2 │ P3 │ P4 │  ← Same physical pages!
│ VA 0x2000 ───┼──R/O─→│    │    │    │    │     Marked READ-ONLY
│ VA 0x3000 ───┼──R/O─→│    │    │    │    │
│ VA 0x4000 ───┼──R/O─→│    │    │    │    │
└─────────────┘        └────┴────┴────┴────┘
┌─────────────┐           ↑
│ Child        │           │
│ VA 0x1000 ───┼──R/O──────┘  Both map to same frames
│ VA 0x2000 ───┼──R/O──────┘  All marked read-only
│ VA 0x3000 ───┼──R/O──────┘  Reference count = 2
│ VA 0x4000 ───┼──R/O──────┘
└─────────────┘

fork() only copies:
- mm_struct (memory descriptor)
- VMAs (virtual memory areas)  
- Page tables (with R/O protection set)
- Does NOT copy actual page contents!
```

---

## 13.2 Shared Page Mechanism

After `fork()`, both processes share physical pages. Each page's `_mapcount` is incremented.

```c
/* mm/memory.c — copy_page_range() during fork() */

static int copy_pte_range(struct mm_struct *dst_mm, struct mm_struct *src_mm,
                          pmd_t *dst_pmd, pmd_t *src_pmd,
                          struct vm_area_struct *vma, unsigned long addr,
                          unsigned long end)
{
    pte_t *src_pte, *dst_pte;
    
    for (; addr < end; addr += PAGE_SIZE) {
        src_pte = pte_offset_map(src_pmd, addr);
        
        if (is_cow_mapping(vma->vm_flags)) {
            /* Make source PTE read-only (even if originally R/W) */
            ptep_set_wrprotect(src_mm, addr, src_pte);
            
            /* Copy PTE to child (also read-only) */
            pte = pte_wrprotect(*src_pte);
            set_pte_at(dst_mm, addr, dst_pte, pte);
        }
        
        /* Increment page reference count */
        get_page(page);  /* page->_refcount++ */
    }
}
```

---

## 13.3 Write Protection

The key to COW is marking pages as **read-only** in both parent and child page tables, even if the VMA allows writes.

```
VMA flags say:  VM_READ | VM_WRITE    (process can read and write)
PTE says:       Read-Only             (hardware enforced)

This mismatch means:
- Read access: works fine (PTE allows)
- Write access: PAGE FAULT (PTE read-only)
  → Kernel checks VMA: writing IS allowed
  → This must be a COW fault → copy the page!
```

---

## 13.4 Copy-on-Write Page Fault

```
COW Fault Flow:

Process (parent or child) writes to shared COW page:
                │
                ▼
    ┌──────────────────────┐
    │  WRITE PAGE FAULT    │  (PTE is read-only)
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │ Find VMA for address │
    │ VMA has VM_WRITE set │  → This IS a valid write
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │ PTE present but R/O  │
    │ → do_wp_page()       │  (Write Protect page handler)
    └──────────┬───────────┘
               │
               ▼
    ┌──────────────────────┐
    │ Check page refcount  │
    │ page->_mapcount > 0? │
    └──────────┬───────────┘
          │            │
     >0 (shared)   =0 (exclusive)
          │            │
          ▼            ▼
    ┌──────────┐  ┌──────────────┐
    │ COPY!    │  │ Just make    │  (Reuse optimization)
    │ Alloc new│  │ PTE writable │
    │ page     │  │ (page is     │
    │ Copy     │  │  already     │
    │ content  │  │  exclusive)  │
    │ Map new  │  └──────────────┘
    │ page R/W │
    │ Unmap old│
    │ page     │
    └──────────┘
```

```c
/* mm/memory.c — do_wp_page() simplified */

static vm_fault_t do_wp_page(struct vm_fault *vmf)
{
    struct page *old_page = vmf->page;
    struct page *new_page;
    
    /* Check if we're the sole owner of this page */
    if (reuse_swap_page(old_page, NULL)) {
        /* Only one process maps this page → just make it writable */
        wp_page_reuse(vmf);
        return VM_FAULT_WRITE;
    }
    
    /* Page is shared — must copy */
    new_page = alloc_page_vma(GFP_HIGHUSER_MOVABLE, vma, vmf->address);
    if (!new_page)
        return VM_FAULT_OOM;
    
    /* Copy page contents */
    copy_user_highpage(new_page, old_page, vmf->address, vma);
    
    /* Replace old PTE with new one pointing to new page, R/W */
    entry = mk_pte(new_page, vma->vm_page_prot);
    entry = pte_mkwrite(pte_mkdirty(entry));
    
    set_pte_at_notify(mm, vmf->address, vmf->pte, entry);
    
    /* Release reference to old page */
    page_remove_rmap(old_page, vma, false);
    put_page(old_page);
    
    return VM_FAULT_WRITE;
}
```

---

## 13.5 Performance Advantages

### fork() Performance Before and After COW

```
Without COW:
fork() copies ALL memory pages
Process with 1GB of memory → copies 1GB = ~200ms

With COW:
fork() copies only page tables
Process with 1GB (1M PTEs) → copies ~8MB of PTEs = ~2ms
100x faster!

After fork(), child typically calls exec() immediately
→ All COW pages discarded (replaced by new program)
→ The copy was never needed at all!
```

### fork()+exec() Pattern

```c
pid_t pid = fork();    /* COW: instant, share pages */
if (pid == 0) {
    /* Child process */
    exec("/usr/bin/ls");  /* Replace entire address space */
    /* All shared COW pages released without ever being copied! */
}
```

### COW Flow Diagram (fork + write)

```
Time →

Parent:  [fork]──────read P1──────read P2──────write P3──────read P3
Child:   [fork]──────read P1──────write P2──────read P3──────read P2

Physical    [P1 P2 P3]   [P1 P2 P3]   [P1 P2 P3 P2']  [P1 P2 P3 P2' P3']
Pages:      shared R/O    shared R/O   child wrote P2   parent wrote P3
                                       → copied as P2'  → copied as P3'

Pages copied: Only the ones actually written to!
If neither writes: ZERO copies forever.
```

---

## Comparison: COW Across Operating Systems

| Feature | Linux | Windows | macOS | QNX |
|---------|-------|---------|-------|-----|
| fork() with COW | Yes | No real fork (CreateProcess) | Yes | Yes (limited) |
| COW for file mmap | Yes (MAP_PRIVATE) | Yes (copy-on-write section) | Yes | Yes |
| COW efficiency | Excellent (page-level) | N/A for processes | Good | Good |
| vfork() | Yes (shares pages, no COW) | N/A | Yes | Yes |

---

## Interview Questions

1. **Q: Explain Copy-on-Write in the context of fork().**
   A: After fork(), parent and child share all physical pages marked read-only. When either writes, a page fault occurs, the kernel copies that specific page, and the writer gets a private copy. Only written pages are copied.

2. **Q: Why is fork() fast despite duplicating the process?**
   A: COW means fork() only copies the mm_struct, VMAs, and page tables (read-only copies of PTEs). It doesn't copy physical pages. For a 1GB process, this is ~8MB vs 1GB.

3. **Q: What happens if you fork() a process with 100GB mapped but only 1 page is written?**
   A: Only 1 page is physically copied. The other 99.99GB remain shared read-only between parent and child.

---

## Summary & Key Takeaways

1. COW makes fork() nearly instant by sharing physical pages between parent and child.
2. Pages are marked read-only; writes trigger page faults that copy individual pages.
3. fork()+exec() benefits most — exec() discards all shared pages without copying.
4. The COW fault handler (`do_wp_page()`) checks if the page is exclusively owned before copying.
5. COW also applies to MAP_PRIVATE file mappings.

---

*Previous: [Chapter 12 — Page Fault Handling](Chapter_12_Page_Fault_Handling.md)*
*Next: [Chapter 14 — Memory Reclaim Mechanisms](Chapter_14_Memory_Reclaim.md)*
