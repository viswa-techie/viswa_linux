# Chapter 28: End-to-End Flow Diagrams

## Chapter Overview

This chapter provides comprehensive flow diagrams tracing memory operations from userspace through the kernel. Each diagram follows the complete path of a memory operation, showing every subsystem involved.

---

## 28.1 Flow: malloc() → Physical Memory

```
User calls malloc(4096):

┌─────────────────────────────────────────────────────────────────────────┐
│ USERSPACE                                                               │
│                                                                         │
│ malloc(4096)                                                            │
│   └─→ glibc arena: check thread-local free list                       │
│         ├─→ Found free chunk → return pointer (NO SYSCALL!)            │
│         └─→ No free chunk → need more memory from kernel               │
│               ├─→ Small: brk() syscall (extend heap)                   │
│               └─→ Large (>128KB default): mmap(MAP_ANONYMOUS)          │
│                                                                         │
└─────────────────────────┬───────────────────────────────────────────────┘
                          │ syscall
┌─────────────────────────▼───────────────────────────────────────────────┐
│ KERNEL: mmap (MAP_ANONYMOUS)                                            │
│                                                                         │
│ do_mmap()                                                               │
│   ├─→ get_unmapped_area() — find free VA range in process              │
│   ├─→ mmap_region() — create vm_area_struct (VMA)                      │
│   │     ├─→ Set vm_flags: VM_READ|VM_WRITE|VM_ANONYMOUS               │
│   │     ├─→ Insert VMA into maple tree (mm->mm_mt)                    │
│   │     └─→ Return virtual address to userspace                        │
│   └─→ NOTE: NO physical pages allocated yet!                           │
│                                                                         │
└─────────────────────────┬───────────────────────────────────────────────┘
                          │ returns VA
┌─────────────────────────▼───────────────────────────────────────────────┐
│ USERSPACE: First write to malloc'd buffer                               │
│                                                                         │
│ buf[0] = 'A';  ← triggers PAGE FAULT (no PTE yet!)                    │
│                                                                         │
└─────────────────────────┬───────────────────────────────────────────────┘
                          │ #PF exception
┌─────────────────────────▼───────────────────────────────────────────────┐
│ KERNEL: Page Fault Handler                                              │
│                                                                         │
│ exc_page_fault(regs, error_code)     [arch/x86/mm/fault.c]            │
│   └─→ do_user_addr_fault()                                            │
│         ├─→ find_vma(mm, address) — find VMA for faulting address      │
│         ├─→ check permissions (VMA flags vs access type)               │
│         └─→ handle_mm_fault(vma, address, flags)   [mm/memory.c]      │
│               ├─→ Walk/create page table levels:                       │
│               │     pgd_offset(mm, addr) → pgd entry                  │
│               │     p4d_alloc() → p4d entry                           │
│               │     pud_alloc() → pud entry                           │
│               │     pmd_alloc() → pmd entry                           │
│               │                                                        │
│               └─→ handle_pte_fault()                                   │
│                     └─→ PTE is empty + anonymous VMA                   │
│                         └─→ do_anonymous_page()                        │
│                               ├─→ alloc_zeroed_user_highpage()         │
│                               │     └─→ __alloc_pages(GFP_HIGHUSER)   │
│                               │           └─→ Buddy allocator          │
│                               │                 └─→ Physical page!    │
│                               ├─→ Clear page to zero                   │
│                               ├─→ Create PTE: page frame + RW + User  │
│                               ├─→ Set PTE in page table               │
│                               └─→ Return: fault resolved              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

Result: buf[0] = 'A' succeeds. Physical page now mapped.
```

---

## 28.2 Flow: fork() → Copy-on-Write

```
Parent process calls fork():

┌─────────────────────────────────────────────────────────────────────────┐
│ KERNEL: do_fork() / kernel_clone()                                      │
│                                                                         │
│ copy_process()                                                          │
│   └─→ copy_mm()                                                        │
│         ├─→ dup_mm() — duplicate mm_struct                             │
│         │     ├─→ Allocate new mm_struct                               │
│         │     ├─→ Copy mm fields (brk, start_stack, etc.)             │
│         │     └─→ Allocate new PGD (page global directory)             │
│         │                                                               │
│         └─→ dup_mmap() — duplicate all VMAs                            │
│               For each VMA in parent:                                   │
│               ├─→ Create new VMA (copy of parent VMA)                  │
│               ├─→ copy_page_range() — "copy" page tables               │
│               │     For each PTE in parent:                             │
│               │     ├─→ DON'T copy pages!                              │
│               │     ├─→ Mark parent PTE as READ-ONLY                   │
│               │     ├─→ Copy PTE to child (also READ-ONLY)             │
│               │     └─→ Increment page refcount                        │
│               └─→ Insert VMA into child's maple tree                   │
│                                                                         │
│ Result after fork:                                                      │
│ Parent page table: [PFN=0x1234, RO, User]  ← was RW, now RO!         │
│ Child page table:  [PFN=0x1234, RO, User]  ← same physical page!     │
│ Page refcount: 2 (both processes reference it)                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

Later: Child writes to shared page:

┌─────────────────────────────────────────────────────────────────────────┐
│ KERNEL: COW Page Fault                                                  │
│                                                                         │
│ Child: buf[0] = 'X';  ← PTE is RO → #PF (write to read-only page)    │
│                                                                         │
│ handle_pte_fault()                                                      │
│   └─→ PTE present + write fault + VMA allows write                     │
│         └─→ do_wp_page()   (Copy-on-Write handler)                     │
│               ├─→ Check page refcount                                  │
│               │     If refcount == 1 → just make writable (reuse)      │
│               │     If refcount > 1 → must COPY:                       │
│               │                                                         │
│               ├─→ Allocate NEW physical page                           │
│               ├─→ Copy contents: old page → new page                   │
│               ├─→ Update child PTE: new PFN, RW                       │
│               ├─→ Decrement old page refcount (now 1)                  │
│               └─→ Parent PTE: restore to RW (only owner now)          │
│                                                                         │
│ BEFORE COW:                        AFTER COW:                          │
│ Parent PTE → [Page A, RO]         Parent PTE → [Page A, RW]           │
│ Child  PTE → [Page A, RO]         Child  PTE → [Page B, RW] (copy!)   │
│ Page A refcount = 2               Page A refcount = 1                   │
│                                   Page B = new copy for child           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 28.3 Flow: File Read → Page Cache → Disk

```
User calls read(fd, buf, 4096):

┌─────────────────────────────────────────────────────────────────────────┐
│ VFS Layer                                                               │
│                                                                         │
│ ksys_read(fd, buf, count)                                              │
│   └─→ vfs_read(file, buf, count, &pos)                                │
│         └─→ file->f_op->read_iter()  (ext4_file_read_iter)            │
│               └─→ generic_file_read_iter()                             │
│                                                                         │
└─────────────────────────┬───────────────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────────────┐
│ Page Cache Layer (mm/filemap.c)                                         │
│                                                                         │
│ filemap_read()                                                          │
│   ├─→ filemap_get_pages() — look up page cache                        │
│   │     ├─→ filemap_get_read_batch()                                  │
│   │     │     └─→ Search XArray (mapping->i_pages) for page at index  │
│   │     │                                                               │
│   │     ├── CACHE HIT ──→ Page found in cache!                        │
│   │     │     ├─→ folio_mark_accessed() — update LRU position         │
│   │     │     └─→ Skip to copy_page_to_iter()                        │
│   │     │                                                               │
│   │     └── CACHE MISS ──→ Page NOT in cache                          │
│   │           └─→ page_cache_sync_readahead()                          │
│   │                 └─→ Readahead: read this page + predicted next     │
│   │                       └─→ Trigger block I/O (see below)           │
│   │                                                                     │
│   └─→ copy_page_to_iter(page, offset, bytes, iter)                    │
│         └─→ Copy from kernel page cache to user buffer                 │
│                                                                         │
└─────────────────────────┬───────────────────────────────────────────────┘
                          │ (on cache miss)
┌─────────────────────────▼───────────────────────────────────────────────┐
│ Block I/O Layer                                                         │
│                                                                         │
│ ext4_readahead() / ext4_read_folio()                                   │
│   └─→ mpage_readahead()                                               │
│         └─→ submit_bio(bio)                                            │
│               └─→ Block device driver                                  │
│                     └─→ NVMe/SCSI command                              │
│                           └─→ Disk/SSD reads sector                    │
│                                 └─→ DMA to page cache page             │
│                                       └─→ Page marked uptodate         │
│                                             └─→ Wake waiting process   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

Timing:
  Cache hit:  ~100ns  (memory copy only)
  Cache miss: ~100µs+ (SSD) or ~10ms+ (HDD) — includes disk I/O
```

---

## 28.4 Flow: Memory Pressure → Reclaim → OOM

```
System running low on memory:

┌─────────────────────────────────────────────────────────────────────────┐
│ TRIGGER: Allocation attempt                                             │
│                                                                         │
│ __alloc_pages(GFP_KERNEL, order=0)                                     │
│   └─→ get_page_from_freelist()                                        │
│         └─→ Check watermarks: free < WMARK_LOW                        │
│               └─→ FAIL: Not enough free pages                          │
│                                                                         │
│ ┌── Slow Path: __alloc_pages_slowpath() ──────────────────────────┐    │
│ │                                                                     │ │
│ │ Step 1: Wake kswapd                                                │ │
│ │   wakeup_kswapd(zone, order, highest_zoneidx)                      │ │
│ │   kswapd runs in background, scanning LRU lists                    │ │
│ │                                                                     │ │
│ │ Step 2: Try allocation again                                       │ │
│ │   get_page_from_freelist() → still fails                           │ │
│ │                                                                     │ │
│ │ Step 3: Direct reclaim                                             │ │
│ │   __perform_reclaim()                                              │ │
│ │     └─→ try_to_free_pages()                                       │ │
│ │           └─→ shrink_node()                                        │ │
│ │                 ├─→ shrink_lruvec()                                │ │
│ │                 │     ├─→ Scan inactive_file list → free clean     │ │
│ │                 │     ├─→ Scan inactive_anon list → swap out       │ │
│ │                 │     └─→ Promote/demote active ↔ inactive         │ │
│ │                 └─→ shrink_slab()                                  │ │
│ │                       └─→ Reclaim dentry/inode caches              │ │
│ │                                                                     │ │
│ │ Step 4: Try allocation again → success? DONE!                      │ │
│ │                                                                     │ │
│ │ Step 5: Try compaction (high-order only)                           │ │
│ │   try_to_compact_pages()                                           │ │
│ │     └─→ Migrate movable pages to create contiguous blocks          │ │
│ │                                                                     │ │
│ │ Step 6: Try allocation again → success? DONE!                      │ │
│ │                                                                     │ │
│ │ Step 7: OOM KILLER (last resort)                                   │ │
│ │   out_of_memory()                                                  │ │
│ │     ├─→ select_bad_process() — pick victim by oom_badness         │ │
│ │     ├─→ oom_kill_process()  — send SIGKILL                        │ │
│ │     └─→ Retry allocation (killed process frees pages)              │ │
│ │                                                                     │ │
│ └─────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

kswapd loop (mm/vmscan.c):
┌──────────────────────────────────────────────┐
│ kswapd:                                       │
│   while (true):                               │
│     sleep until woken (free < WMARK_LOW)      │
│     while (free < WMARK_HIGH):                │
│       scan LRU lists                          │
│       reclaim inactive pages                  │
│       promote active pages as needed          │
│     go back to sleep                          │
└──────────────────────────────────────────────┘
```

---

## 28.5 Flow: mmap'd File → Page Fault → Disk Read

```
fd = open("data.bin"); 
ptr = mmap(NULL, 4096, PROT_READ, MAP_PRIVATE, fd, 0);
data = ptr[0];  ← Page fault!

┌─────────────────────────────────────────────────────────────────────────┐
│ Page Fault for mmap'd file:                                             │
│                                                                         │
│ handle_mm_fault()                                                       │
│   └─→ handle_pte_fault()                                               │
│         └─→ PTE empty + file-backed VMA                                │
│               └─→ do_fault()                                           │
│                     └─→ do_read_fault() (PROT_READ mapping)            │
│                           ├─→ __do_fault()                             │
│                           │     └─→ vma->vm_ops->fault(vmf)           │
│                           │           └─→ filemap_fault()              │
│                           │                 ├─→ Search page cache      │
│                           │                 ├── HIT → use cached page  │
│                           │                 └── MISS:                  │
│                           │                       ├─→ Allocate new page│
│                           │                       ├─→ Add to cache     │
│                           │                       ├─→ Read from disk   │
│                           │                       └─→ Wait for I/O    │
│                           │                                             │
│                           ├─→ finish_fault()                           │
│                           │     └─→ Set PTE: page frame + Read-only   │
│                           └─→ Return: fault resolved                   │
│                                                                         │
│ ptr[0] now reads the file content directly from page cache!            │
│ No memcpy to userspace — page cache page mapped into process.          │
│                                                                         │
│ Same file opened by another process? Same physical page cache page!    │
│ Shared read-only — zero extra memory cost!                              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 28.6 Flow: Kernel Module kmalloc → SLUB → Buddy

```
Driver calls kmalloc(256, GFP_KERNEL):

┌─────────────────────────────────────────────────────────────────────────┐
│ kmalloc(256, GFP_KERNEL)                                                │
│   └─→ __kmalloc(256, GFP_KERNEL)                                      │
│         └─→ Find slab cache: kmalloc-256 (size class ≥ 256)           │
│                                                                         │
│ SLUB Allocator (mm/slub.c):                                            │
│                                                                         │
│ FAST PATH (no locks!):                                                  │
│ ┌───────────────────────────────────────┐                              │
│ │ Check per-CPU slab (c->freelist)       │                              │
│ │   ├─→ Object available?                │                              │
│ │   │     YES → cmpxchg to take object   │                              │
│ │   │           Return pointer. DONE!    │                              │
│ │   │                                     │                              │
│ │   └─→ NO → per-CPU slab exhausted      │                              │
│ └───────────────────┬───────────────────┘                              │
│                     │                                                   │
│ MEDIUM PATH:        ▼                                                   │
│ ┌───────────────────────────────────────┐                              │
│ │ Get partial slab from per-node list    │                              │
│ │   ├─→ Found partial → make it per-CPU  │                              │
│ │   │     Take object, return. DONE!     │                              │
│ │   │                                     │                              │
│ │   └─→ No partial slabs available        │                              │
│ └───────────────────┬───────────────────┘                              │
│                     │                                                   │
│ SLOW PATH:          ▼                                                   │
│ ┌───────────────────────────────────────┐                              │
│ │ Allocate new slab from buddy allocator │                              │
│ │   └─→ alloc_pages(GFP_KERNEL, order)   │                              │
│ │         └─→ Buddy: find 2^order block   │                              │
│ │               Split larger block if     │                              │
│ │               needed.                   │                              │
│ │                                         │                              │
│ │ Initialize slab:                        │                              │
│ │   Format page as slab objects:          │                              │
│ │   [obj0|obj1|obj2|...|objN]            │                              │
│ │   Set up freelist chain.                │                              │
│ │   Return first object. DONE!            │                              │
│ └─────────────────────────────────────────┘                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 28.7 Flow: Swap Out → Swap In

```
SWAP OUT (memory pressure):

shrink_folio_list() scanning inactive_anon LRU:
  ├─→ folio is anonymous, inactive, unreferenced
  ├─→ try_to_unmap(folio) — remove all page table mappings
  │     └─→ rmap walk: for each VMA mapping this folio:
  │           └─→ Replace PTE with swap entry (PTE type=swap)
  ├─→ Add folio to swap cache
  ├─→ swap_writepage() — write folio to swap device
  │     └─→ bio submit → write to swap partition/file
  └─→ After write complete: free physical page to buddy

Swap PTE format (not present, contains swap location):
┌───────────────────────────────────────────────────────┐
│ Bit 0 = 0 (not present)                               │
│ Bits 1-4: swap type (which swap device)               │
│ Bits 5-58: swap offset (location within swap device)  │
│ Bit 63: soft-dirty tracking                           │
└───────────────────────────────────────────────────────┘

SWAP IN (page accessed again):

Process accesses swapped-out address:
  └─→ Page fault (#PF, PTE not present)
        └─→ handle_pte_fault()
              └─→ PTE is swap entry
                    └─→ do_swap_page()
                          ├─→ Check swap cache (already in memory?)
                          │     YES → use cached page
                          │     NO  → continue:
                          ├─→ Allocate physical page
                          ├─→ Read from swap device
                          │     └─→ bio submit → read from swap
                          ├─→ Wait for I/O completion
                          ├─→ Set PTE: new page frame + RW + User
                          ├─→ Add to swap cache (lazy free from swap)
                          └─→ Process continues
```

---

## Interview Questions

1. **Q: Trace the path from `malloc()` to a physical page being allocated.**
   A: `malloc()` → glibc arena → if no free chunk → `mmap(MAP_ANONYMOUS)` syscall → `do_mmap()` creates VMA (no physical pages yet) → first access triggers page fault → `handle_mm_fault()` → `do_anonymous_page()` → `alloc_zeroed_user_highpage()` → buddy allocator `__alloc_pages()` → physical page returned → PTE created → fault resolved.

2. **Q: What happens during `fork()` from a memory perspective?**
   A: `copy_mm()` → `dup_mmap()` duplicates all VMAs → `copy_page_range()` marks all parent PTEs read-only and copies them to child (COW setup). No physical pages are copied — both processes share the same pages with refcount incremented. On first write by either process → COW fault → `do_wp_page()` allocates new page, copies content, makes it writable.

3. **Q: How does the kernel reclaim memory when under pressure?**
   A: When free < WMARK_LOW: wake kswapd → scan LRU lists → free clean file pages (cheap), writeback dirty pages (async), swap out anonymous pages (expensive). If kswapd insufficient → allocation slow path does direct reclaim (synchronous). If all fails → OOM killer selects and kills highest-oom_badness process.

---

## Summary

This chapter traced complete flows for:
1. **malloc → page fault → physical allocation** (demand paging)
2. **fork → COW → page copy** (copy-on-write)
3. **file read → page cache hit/miss → disk I/O** (file I/O path)
4. **allocation failure → reclaim → OOM** (memory pressure handling)
5. **mmap file → fault → disk read → direct mapping** (zero-copy file access)
6. **kmalloc → SLUB → buddy** (kernel allocation path)
7. **swap out → swap PTE → swap in** (swap lifecycle)

Understanding these end-to-end flows is essential for debugging, performance tuning, and kernel development.

---

*Next: [Chapter 29 — Important Diagrams and Visual References](Chapter_29_Diagrams.md)*
