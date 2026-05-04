# Chapter 15: Memory-Mapped Files and mmap

## Learning Goals
- Understand how mmap maps file data into process address space
- Know the page fault handling mechanism for mmap'd files
- Master shared vs private mappings and their use cases
- Understand msync, MAP_POPULATE, and performance considerations

---

## 15.1 mmap Fundamentals

```
mmap() maps a file (or anonymous memory) into the process's virtual address space:

  void *addr = mmap(NULL, length, PROT_READ|PROT_WRITE,
                    MAP_SHARED, fd, offset);

  After mmap:
  ┌─────────────────────────────────────────────────────┐
  │ Process Virtual Address Space                        │
  │                                                     │
  │ ┌──────────┐                                        │
  │ │ Stack    │ 0x7FFF...                              │
  │ ├──────────┤                                        │
  │ │ (free)   │                                        │
  │ ├──────────┤                                        │
  │ │ mmap'd   │ ◄── addr returned by mmap()            │
  │ │ file     │     Points to file data via page cache │
  │ │ region   │                                        │
  │ ├──────────┤                                        │
  │ │ Heap     │                                        │
  │ ├──────────┤                                        │
  │ │ BSS/Data │                                        │
  │ ├──────────┤                                        │
  │ │ Text     │ 0x0040...                              │
  │ └──────────┘                                        │
  └─────────────────────────────────────────────────────┘

  Key insight: The mapped region uses THE SAME page cache pages
  as read()/write(). No extra copy needed!

  ┌──────────────┐       ┌──────────────────────┐
  │ Process PTE  │──────►│ Page Cache Page       │
  │ (page table) │       │ (file data, shared)   │
  └──────────────┘       └──────────────────────┘
                               ▲
                         ┌─────┘
  ┌──────────────┐       │
  │ read(fd,...) │───────┘  ← read() also uses same page
  └──────────────┘
```

---

## 15.2 mmap System Call

```c
/* System call: arch/x86/kernel/sys_x86_64.c → mm/mmap.c */
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);

/* Parameters: */
addr    = NULL (kernel chooses) or hint address
length  = Size of mapping (rounded up to page boundary)
prot    = PROT_READ | PROT_WRITE | PROT_EXEC | PROT_NONE
flags   = MAP_SHARED | MAP_PRIVATE | MAP_ANONYMOUS | MAP_FIXED | ...
fd      = File descriptor (ignored for MAP_ANONYMOUS)
offset  = Offset into file (must be page-aligned)

/* Returns: pointer to mapped region, or MAP_FAILED */
```

### Mapping Types

```
┌────────────────────────────────────────────────────────────────────┐
│ MAP_SHARED + file fd                                               │
│                                                                    │
│ - Multiple processes share SAME physical pages                     │
│ - Writes visible to all mappers AND to file on disk               │
│ - Changes written back to file (like write())                     │
│ - Used for: shared memory, memory-mapped I/O, interprocess comm   │
│                                                                    │
│ Process A page table ──► [Page Cache Page] ◄── Process B page table│
│                          File data on disk                         │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ MAP_PRIVATE + file fd (Copy-on-Write)                              │
│                                                                    │
│ - Initially shares page cache pages (read-only)                   │
│ - On write: page is COPIED → private copy for this process        │
│ - Changes NOT written back to file                                │
│ - Used for: loading shared libraries (.so), executable text       │
│                                                                    │
│ Read:  Process PTE (read-only) → Page Cache Page (shared)         │
│ Write: Process PTE → COPY of page (private, not in page cache)    │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│ MAP_ANONYMOUS (+ MAP_SHARED or MAP_PRIVATE)                        │
│                                                                    │
│ - No file backing — zero-initialized pages                        │
│ - MAP_PRIVATE|MAP_ANONYMOUS: process heap, stack growth           │
│ - MAP_SHARED|MAP_ANONYMOUS: shared between parent+child (fork)    │
│ - Used for: malloc() large allocations, thread stacks             │
└────────────────────────────────────────────────────────────────────┘
```

---

## 15.3 mmap Kernel Implementation

### mmap Flow

```
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0)
  │
  ▼
SYSCALL_DEFINE6(mmap, ...)                    [arch/x86/kernel/sys_x86_64.c]
  │
  ▼
ksys_mmap_pgoff()
  │
  ▼
vm_mmap_pgoff(file, addr, len, prot, flags, pgoff)
  │
  ▼
do_mmap(file, addr, len, prot, flags, pgoff, ...)   [mm/mmap.c]
  │
  ├── 1. Find free virtual address range
  │       get_unmapped_area()
  │       → walk VMA tree, find gap ≥ len
  │
  ├── 2. Create VMA (Virtual Memory Area)
  │       vm_area_alloc()
  │       vma->vm_file = file
  │       vma->vm_pgoff = pgoff
  │       vma->vm_ops = file->f_op->mmap() sets this
  │
  ├── 3. Call file's mmap handler
  │       file->f_op->mmap(file, vma)
  │       │
  │       └── ext4_file_mmap()
  │             vma->vm_ops = &ext4_file_vm_ops
  │             → Contains fault handler
  │
  ├── 4. No pages mapped yet!
  │       VMA created but page tables empty
  │       Pages will be mapped on first access (demand paging)
  │
  └── 5. Return virtual address

After return: addr is valid but accessing it will trigger a page fault.
```

### Page Fault Handling

```
First access to mmap'd region:

  *addr = 'x';  ← Write to mmap'd address
  │
  ▼
  CPU: page table entry is empty → PAGE FAULT exception
  │
  ▼
  do_page_fault()                             [arch/x86/mm/fault.c]
  │
  ▼
  handle_mm_fault(vma, address, flags)        [mm/memory.c]
  │
  ▼
  __handle_mm_fault()
  │
  ▼
  handle_pte_fault()
  │
  ├── PTE empty and VMA has vm_ops->fault:
  │     do_fault()
  │     │
  │     ├── MAP_SHARED write:
  │     │     do_shared_fault()
  │     │     ├── vma->vm_ops->fault(vmf)
  │     │     │     → filemap_fault()         [mm/filemap.c]
  │     │     │       ├── Find page in page cache
  │     │     │       ├── If miss: readpage() → I/O
  │     │     │       └── Return page
  │     │     │
  │     │     ├── vma->vm_ops->page_mkwrite(vmf)
  │     │     │     → ext4_page_mkwrite()
  │     │     │       ├── Journal start
  │     │     │       ├── Allocate blocks
  │     │     │       └── Allow page to be dirtied
  │     │     │
  │     │     └── Install PTE mapping (read-write)
  │     │
  │     ├── MAP_PRIVATE write:
  │     │     do_cow_fault()
  │     │     ├── Read page from page cache
  │     │     ├── Allocate NEW page (private copy)
  │     │     ├── Copy data to new page
  │     │     └── Install PTE to private page
  │     │
  │     └── Read fault:
  │           do_read_fault()
  │           ├── filemap_fault() → page cache lookup/read
  │           └── Install PTE (read-only for MAP_PRIVATE)

Subsequent accesses: PTE is populated → no page fault → direct memory access
```

---

## 15.4 File-Backed VM Operations

```c
/* mm/filemap.c — default file VM operations */
const struct vm_operations_struct generic_file_vm_ops = {
    .fault    = filemap_fault,     /* Handle page fault */
    .map_pages = filemap_map_pages, /* Map surrounding pages */
    .page_mkwrite = filemap_page_mkwrite, /* Before write to shared page */
};

/* ext4-specific VM ops (fs/ext4/file.c) */
static const struct vm_operations_struct ext4_file_vm_ops = {
    .fault        = filemap_fault,
    .map_pages    = filemap_map_pages,
    .page_mkwrite = ext4_page_mkwrite,  /* ext4-specific: journal + alloc */
};
```

### filemap_fault() — The Fault Handler

```c
/* mm/filemap.c (simplified) */
vm_fault_t filemap_fault(struct vm_fault *vmf)
{
    struct file *file = vmf->vma->vm_file;
    struct inode *inode = file_inode(file);
    pgoff_t index = vmf->pgoff;
    struct folio *folio;

    /* Try to find the page in page cache */
    folio = filemap_get_folio(inode->i_mapping, index);
    
    if (!folio) {
        /* Page cache miss: need to read from disk */
        /* Trigger readahead too */
        filemap_fault_recheck_pte_none(vmf);
        folio = filemap_fault_get_folio(vmf, index);
        /* This calls a_ops->read_folio() internally */
    }
    
    /* Wait for page to be uptodate */
    if (!folio_test_uptodate(folio))
        wait_on_folio_locked(folio);
    
    /* Set up the page table entry */
    vmf->page = folio_page(folio, index - folio->index);
    return VM_FAULT_LOCKED;
}
```

---

## 15.5 msync — Synchronizing Mapped Memory

```
msync() flushes modified pages back to disk:

  msync(addr, length, flags);

  Flags:
    MS_SYNC      → Write dirty pages and wait for completion
    MS_ASYNC     → Schedule writeback but don't wait
    MS_INVALIDATE → Invalidate cached copies (re-read from file)

  Flow:
    msync(addr, 4096, MS_SYNC)
      │
      ▼
    Find all dirty pages in the range
      │
      ▼
    For each dirty page:
      ├── filemap_write_and_wait_range()
      │     ├── Write page to disk (via a_ops->writepage)
      │     └── Wait for I/O completion
      └── Clear dirty flag

  Important: Without msync(MS_SYNC) or fsync(), data may be
  in page cache only → lost on power failure!
```

---

## 15.6 mmap Use Cases

### 1. Reading Large Files

```c
/* Efficient file reading with mmap */
int fd = open("large_file.dat", O_RDONLY);
struct stat st;
fstat(fd, &st);

void *data = mmap(NULL, st.st_size, PROT_READ,
                  MAP_PRIVATE, fd, 0);
close(fd);  /* fd can be closed after mmap */

/* Access sequentially — kernel does readahead automatically */
for (size_t i = 0; i < st.st_size; i += 4096) {
    process_page(data + i);
}

/* Advise kernel about access pattern */
madvise(data, st.st_size, MADV_SEQUENTIAL);

munmap(data, st.st_size);
```

### 2. Shared Memory Between Processes

```c
/* Process A: create shared mapping */
int fd = open("/tmp/shared.dat", O_RDWR | O_CREAT, 0644);
ftruncate(fd, 4096);
int *shared = mmap(NULL, 4096, PROT_READ|PROT_WRITE,
                   MAP_SHARED, fd, 0);
shared[0] = 42;  /* Visible to Process B */
msync(shared, 4096, MS_SYNC);

/* Process B: map same file */
int fd = open("/tmp/shared.dat", O_RDONLY);
int *shared = mmap(NULL, 4096, PROT_READ,
                   MAP_SHARED, fd, 0);
printf("%d\n", shared[0]);  /* Reads 42 */
```

### 3. Database File Access (mmap I/O)

```
Many databases use mmap for data files:

  SQLite (mmap mode):
    mmap entire database file
    Read: direct memory access (no read() syscall)
    Write: modify pages → automatic writeback
    
  Advantages:
    - No system call overhead for reads
    - Kernel manages I/O scheduling and caching
    - Multiple processes share page cache pages
    
  Disadvantages:
    - No control over I/O order (harder to guarantee consistency)
    - SIGBUS on I/O error or file truncation
    - Cannot use O_DIRECT (mmap IS the page cache)
    - Hard to control page eviction
    
  Many high-performance databases (PostgreSQL, MySQL InnoDB)
  prefer read()/write() with their own buffer pool for more control.
```

---

## 15.7 madvise — I/O Hints

```c
madvise(addr, length, advice);

Advice values:
  MADV_NORMAL      ← Default (moderate readahead)
  MADV_SEQUENTIAL  ← Sequential access (aggressive readahead)
  MADV_RANDOM      ← Random access (disable readahead)
  MADV_WILLNEED    ← Will need soon (prefetch into cache)
  MADV_DONTNEED    ← Don't need anymore (can free pages)
  MADV_FREE        ← Pages may be freed (lazy, deferred)
  MADV_HUGEPAGE    ← Collapse to huge pages if possible
  MADV_DONTFORK    ← Don't copy on fork (for DMA buffers)
  MADV_POPULATE_READ  ← Fault in all pages now (avoid later faults)
  MADV_POPULATE_WRITE ← Fault in all pages writable now

Example: pre-fault all pages
  void *data = mmap(NULL, size, PROT_READ, MAP_PRIVATE, fd, 0);
  madvise(data, size, MADV_WILLNEED);  /* Kick off readahead */
  /* Or use MAP_POPULATE flag: */
  void *data = mmap(NULL, size, PROT_READ, MAP_PRIVATE | MAP_POPULATE, fd, 0);
  /* All pages faulted in during mmap() call */
```

---

## 15.8 mmap vs read/write Performance

```
Scenario: Read a file sequentially

  read() approach:
    syscall → VFS → page cache → copy_to_user → user buffer
    ┌──────────────────────────────────────────────────┐
    │ + Simple, well-understood                         │
    │ + Works with O_DIRECT                             │
    │ + Better error handling (no SIGBUS)               │
    │ - System call overhead per read()                 │
    │ - Extra copy: page cache → user buffer            │
    └──────────────────────────────────────────────────┘

  mmap() approach:
    page fault → page cache → direct access (no copy)
    ┌──────────────────────────────────────────────────┐
    │ + No data copy (zero-copy: PTE maps page cache)   │
    │ + No system call for subsequent accesses           │
    │ + Excellent for random access patterns             │
    │ - Initial page fault overhead                     │
    │ - TLB pressure for large mappings                  │
    │ - SIGBUS on errors (hard to handle)               │
    │ - No control over I/O ordering                    │
    └──────────────────────────────────────────────────┘

  General guidance:
    Sequential read of whole file → read() is fine (readahead helps)
    Random access to large file → mmap() wins
    Need precise I/O control → read()/write()
    Shared memory IPC → mmap(MAP_SHARED)
```

---

## Kernel Source References

```
mmap:
  mm/mmap.c               ← do_mmap(), mmap_region()
  mm/memory.c             ← handle_mm_fault(), do_fault()
  mm/filemap.c            ← filemap_fault(), filemap_page_mkwrite()
  
VMA:
  include/linux/mm_types.h ← struct vm_area_struct
  include/linux/mm.h       ← vm_operations_struct

ext4 mmap:
  fs/ext4/file.c           ← ext4_file_mmap(), ext4_page_mkwrite()

msync:
  mm/msync.c               ← sys_msync()

madvise:
  mm/madvise.c             ← sys_madvise()
```

---

## Interview Questions

1. **How does mmap() work internally? What happens on first access?**
2. **What is the difference between MAP_SHARED and MAP_PRIVATE?**
3. **How does Copy-on-Write work with MAP_PRIVATE file mappings?**
4. **What is page_mkwrite()? Why does ext4 need it?**
5. **When would you use mmap() over read()? And vice versa?**
6. **What does msync() do? When is it necessary?**
7. **What is MADV_SEQUENTIAL? How does it affect readahead?**
8. **Why do some databases prefer read()/write() over mmap()?**
9. **What causes SIGBUS with mmap? How do you handle it?**
10. **How is mmap used for loading shared libraries?**

---

## Summary

- mmap() maps file data into process address space via page table entries
- No data copied: process PTE points directly to page cache pages
- MAP_SHARED: changes visible to all mappers + written to file
- MAP_PRIVATE: CoW semantics, writes create private copy, file unchanged
- Page fault on first access: filemap_fault() reads page from disk if not cached
- msync(MS_SYNC) or fsync() needed to guarantee data on stable storage
- madvise(): hint kernel about access patterns for better readahead
- mmap excels at random access to large files; read() excels at sequential reads

---

*Next: [Chapter 16 — File Caching and the Page Cache](Chapter_16_Page_Cache.md)*
