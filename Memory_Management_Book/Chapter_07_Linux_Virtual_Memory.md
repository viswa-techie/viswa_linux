# Chapter 7: Linux Virtual Memory Management

## Chapter Overview

This chapter covers how Linux manages virtual memory for processes — the `mm_struct`, `vm_area_struct` (VMA), process memory layout, and the key system calls `mmap()` and `brk()`. This is the layer between user-space memory requests and the physical page allocator.

---

## 7.1 Process Address Space

Every Linux process has a private virtual address space described by a **memory descriptor** (`mm_struct`).

```
Process "cat /etc/passwd":
┌──────────────────────────────────────────────┐
│ task_struct                                  │
│   ├── mm → mm_struct                         │
│   │        ├── pgd (page table root)         │
│   │        ├── mmap (VMA list)               │
│   │        ├── mm_rb / mm_mt (VMA tree)      │
│   │        ├── map_count = 15 VMAs           │
│   │        ├── total_vm = 2000 pages         │
│   │        ├── start_code, end_code          │
│   │        ├── start_data, end_data          │
│   │        ├── start_brk, brk               │
│   │        ├── start_stack                   │
│   │        └── arg_start, arg_end            │
│   └── ...                                     │
└──────────────────────────────────────────────┘
```

---

## 7.2 Memory Layout of a Linux Process

```
High Address
0x7FFFFFFFFFFF ┌──────────────────────────┐
               │     Kernel Space          │ (not accessible from user)
0x7FFFFFFFF000 ├──────────────────────────┤
               │  [vdso] / [vsyscall]     │ Virtual Dynamic Shared Object
               ├──────────────────────────┤
               │                          │
               │     Stack                │ ← Grows DOWN (toward lower addresses)
               │     (8MB default limit)  │    RSP register points here
               │                          │
               ├──────────────────────────┤ ← Stack limit (RLIMIT_STACK)
               │                          │
               │  Memory-mapped region    │ ← mmap(), shared libraries
               │  (grows DOWN)            │    ld-linux.so, libc.so, libm.so
               │                          │
               ├──────────────────────────┤
               │                          │
               │     Heap                 │ ← Grows UP (brk/sbrk/malloc)
               │                          │
               ├──────────────────────────┤ ← brk (program break)
               │     BSS                  │ ← Uninitialized global data (zeroed)
               ├──────────────────────────┤
               │     Data                 │ ← Initialized global data (.data)
               ├──────────────────────────┤
               │     Text (Code)          │ ← Executable code (.text), R/X
               ├──────────────────────────┤
               │     ELF headers          │
0x555555554000 └──────────────────────────┘
               │     NULL page guard      │ ← First page unmapped (catch NULL deref)
0x000000000000 └──────────────────────────┘
Low Address
```

### Viewing Process Memory Layout

```bash
$ cat /proc/self/maps
555555554000-555555558000 r--p 00000000 08:01 131073     /usr/bin/cat
555555558000-55555555d000 r-xp 00004000 08:01 131073     /usr/bin/cat  # .text
55555555d000-555555560000 r--p 00009000 08:01 131073     /usr/bin/cat  # .rodata
555555560000-555555561000 r--p 0000b000 08:01 131073     /usr/bin/cat
555555561000-555555562000 rw-p 0000c000 08:01 131073     /usr/bin/cat  # .data
555555562000-555555583000 rw-p 00000000 00:00 0          [heap]
7ffff7c00000-7ffff7c28000 r--p 00000000 08:01 262147     /usr/lib/libc.so.6
7ffff7c28000-7ffff7dbd000 r-xp 00028000 08:01 262147     /usr/lib/libc.so.6
...
7ffff7fc3000-7ffff7fc7000 r--p 00000000 00:00 0          [vvar]
7ffff7fc7000-7ffff7fc9000 r-xp 00000000 00:00 0          [vdso]
7ffff7fc9000-7ffff7fca000 r--p 00000000 08:01 262143     /usr/lib/ld-linux-x86-64.so.2
7ffffffde000-7ffffffff000 rw-p 00000000 00:00 0          [stack]

# Fields: address range, permissions (r/w/x/p=private/s=shared), 
#          offset in file, device, inode, pathname
```

---

## 7.3 Kernel Virtual Address Space Layout

```c
/* Documentation/x86/x86_64/mm.rst */

/*
 * x86_64 kernel virtual memory layout (4-level paging):
 *
 * Start addr    |   Offset   |     End addr     |  Size   | Description
 * ==============|============|==================|=========|============
 * 0000000000000 |     0      | 00007fffffffffff |  128 TB | User space
 * ______________|____________|__________________|_________|____________
 * ffff800000000 |  -128 TB   | ffff87ffffffffff |    8 TB | Guard hole
 * ffff880000000 |  -120 TB   | ffff887fffffffff |  0.5 TB | LDT remap
 * ffff888000000 |  -119.5TB  | ffffc87fffffffff |   64 TB | Direct mapping
 * ffffc88000000 |            | ffffc8ffffffffff |  0.5 TB | Unused hole
 * ffffc90000000 |   -55.5TB  | ffffe8ffffffffff |   32 TB | vmalloc/ioremap
 * ffffe90000000 |   -23.5TB  | ffffe9ffffffffff |    1 TB | Virtual memory map
 * ffffea0000000 |   -22.5TB  | ffffeaffffffffff |    1 TB | (continued)
 * ffffeb0000000 |            | ffffebffffffffff |    1 TB | Unused
 * ffffec0000000 |            | fffffbffffffffff |   64 TB | KASAN shadow mem
 * fffffc0000000 |    -4 TB   | fffffdffffffffff |    2 TB | Unused
 * fffffe0000000 |    -2 TB   | fffffe7fffffffff |  0.5 TB | cpu_entry_area
 * ffffffff80000 |    -2 GB   | ffffffff9fffffff |  512 MB | Kernel text
 * ffffffffa0000 |            | fffffffffeffffff | 1520 MB | Module mapping
 * ffffffffff000 |   -16 MB   | ffffffffffffffff |   16 MB | Fixmap
 */
```

---

## 7.4 Memory Descriptors

### mm_struct — The Per-Process Memory Descriptor

```c
/* include/linux/mm_types.h (simplified) */

struct mm_struct {
    /* VMA management */
    struct maple_tree mm_mt;          /* Maple tree of VMAs (6.1+) */
    /* Previously: struct rb_root mm_rb (Red-black tree) */
    
    unsigned long mmap_base;          /* Base of mmap region */
    unsigned long task_size;          /* Size of user address space */
    
    pgd_t *pgd;                       /* Page table root */
    
    atomic_t mm_users;                /* Users of this mm (including threads) */
    atomic_t mm_count;                /* References to mm_struct itself */
    
    int map_count;                    /* Number of VMAs */
    
    spinlock_t page_table_lock;       /* Protects page table changes */
    struct rw_semaphore mmap_lock;    /* Protects VMA tree — THE BIG LOCK */
    
    unsigned long total_vm;           /* Total pages mapped */
    unsigned long locked_vm;          /* Locked (mlock'd) pages */
    unsigned long pinned_vm;          /* Pinned (can't be migrated) */
    unsigned long data_vm;            /* Data pages */
    unsigned long exec_vm;            /* Exec pages */
    unsigned long stack_vm;           /* Stack pages */
    
    unsigned long start_code, end_code;   /* Code region */
    unsigned long start_data, end_data;   /* Data region */
    unsigned long start_brk, brk;         /* Heap region */
    unsigned long start_stack;            /* Stack start */
    unsigned long arg_start, arg_end;     /* argv region */
    unsigned long env_start, env_end;     /* envp region */
    
    /* NUMA policy */
    struct mempolicy *mempolicy;
    
    /* ... */
};
```

---

## 7.5 struct mm_struct Deep Dive

### The mmap_lock (formerly mmap_sem)

The `mmap_lock` is a reader-writer semaphore that protects the VMA tree. It's one of the most contended locks in the kernel.

```c
/* Taking the lock */
mmap_read_lock(mm);    /* For reading VMA tree (page fault, /proc/maps) */
mmap_read_unlock(mm);

mmap_write_lock(mm);   /* For modifying VMA tree (mmap, munmap, mremap) */
mmap_write_unlock(mm);

/* The lock contention problem:
 * - Page faults need read lock (very frequent!)
 * - mmap/munmap need write lock
 * - On heavy mmap workloads, this becomes a bottleneck
 * 
 * Solution (Linux 6.4+): Per-VMA locks
 * Each VMA has its own lock for page fault handling
 * mmap_lock still needed for VMA creation/deletion
 */
```

### Per-VMA Locks (Linux 6.4+)

```c
/* Per-VMA lock for page fault optimization */
struct vm_area_struct {
    /* ... */
    struct rw_semaphore lock;  /* Per-VMA lock */
    /* ... */
};

/* Page fault path (optimized):
 * 1. Try to take per-VMA read lock (fast path)
 * 2. If successful, handle fault without mmap_lock
 * 3. If VMA is being modified, fall back to mmap_lock
 * 
 * This dramatically reduces contention on multi-threaded workloads
 */
```

---

## 7.6 struct vm_area_struct (VMA)

A VMA describes a contiguous virtual memory region with uniform properties.

```c
/* include/linux/mm_types.h */

struct vm_area_struct {
    unsigned long vm_start;        /* Start address (inclusive) */
    unsigned long vm_end;          /* End address (exclusive) */
    
    struct mm_struct *vm_mm;       /* Owning mm_struct */
    pgprot_t vm_page_prot;        /* Page protection (PTEs) */
    unsigned long vm_flags;        /* Flags: VM_READ, VM_WRITE, etc. */
    
    /* Tree linkage (maple tree in 6.1+) */
    
    /* Linked list of VMAs (sorted by address) */
    struct vm_area_struct *vm_next, *vm_prev;
    
    /* For file-backed mappings */
    struct file *vm_file;          /* Mapped file (NULL for anonymous) */
    unsigned long vm_pgoff;        /* Offset within file (in pages) */
    
    /* Operations */
    const struct vm_operations_struct *vm_ops;
    
    /* Reverse mapping */
    struct anon_vma *anon_vma;     /* For anonymous pages */
    
    /* ... */
};
```

### VMA Flags

```c
/* include/linux/mm.h */

#define VM_READ         0x00000001  /* Readable */
#define VM_WRITE        0x00000002  /* Writable */
#define VM_EXEC         0x00000004  /* Executable */
#define VM_SHARED       0x00000008  /* Shared mapping */
#define VM_MAYREAD      0x00000010  /* Can be made readable (mprotect) */
#define VM_MAYWRITE     0x00000020  /* Can be made writable */
#define VM_MAYEXEC      0x00000040  /* Can be made executable */
#define VM_GROWSDOWN    0x00000100  /* Stack: grows downward */
#define VM_PFNMAP       0x00000400  /* PFN-mapped (no struct page) */
#define VM_LOCKED       0x00002000  /* mlock'd */
#define VM_IO           0x00004000  /* I/O memory mapping */
#define VM_DONTCOPY     0x00020000  /* Don't copy on fork (VM_WIPEONFORK) */
#define VM_DONTEXPAND   0x00040000  /* Can't expand with mremap */
#define VM_HUGEPAGE     0x00800000  /* THP eligible */
#define VM_MIXEDMAP     0x10000000  /* Can have both PFN and page mappings */
```

### VMA Layout Example

```
Process VMAs (from /proc/pid/maps):

VMA 1: 0x555555554000 - 0x555555558000 [.text]     VM_READ|VM_EXEC
VMA 2: 0x555555558000 - 0x555555560000 [.rodata]   VM_READ
VMA 3: 0x555555560000 - 0x555555562000 [.data+bss]  VM_READ|VM_WRITE
VMA 4: 0x555555562000 - 0x555555583000 [heap]       VM_READ|VM_WRITE
VMA 5: 0x7ffff7c00000 - 0x7ffff7dbd000 [libc.so]   VM_READ|VM_EXEC|VM_SHARED
VMA 6: ...
VMA 7: 0x7ffffffde000 - 0x7ffffffff000 [stack]      VM_READ|VM_WRITE|VM_GROWSDOWN

Binary search in maple tree: O(log n) to find VMA for any address
```

---

## 7.7 Virtual Memory Regions

### Anonymous vs File-Backed Regions

```
Anonymous Memory (no file backing):
├── Stack: grows down, VM_GROWSDOWN
├── Heap: brk/sbrk, mmap(MAP_ANONYMOUS)
├── BSS: zero-initialized globals
└── Anonymous mmap: mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)

File-Backed Memory:
├── Code (.text): mmap of executable file, VM_READ|VM_EXEC
├── Data (.data): mmap of executable file, VM_READ|VM_WRITE
├── Shared libraries: mmap of .so files
├── Memory-mapped files: mmap(fd, ...)
└── Shared memory: shm_open + mmap
```

---

## 7.8 Memory Mappings

### Private vs Shared Mappings

```
MAP_PRIVATE (Copy-on-Write):
  ┌─────────┐        ┌──────────┐
  │Process A │──────→│ Page X   │ (read-only)
  └─────────┘   ┌──→│          │
  ┌─────────┐   │   └──────────┘
  │Process B │───┘
  └─────────┘
  
  On write by A:
  ┌─────────┐        ┌──────────┐
  │Process A │──────→│ Copy of X│ (A's private copy)
  └─────────┘        └──────────┘
  ┌─────────┐        ┌──────────┐
  │Process B │──────→│ Page X   │ (original, still shared)
  └─────────┘        └──────────┘

MAP_SHARED:
  ┌─────────┐        ┌──────────┐
  │Process A │──────→│ Page X   │ (read-write, shared)
  └─────────┘   ┌──→│          │
  ┌─────────┐   │   └──────────┘
  │Process B │───┘
  └─────────┘
  
  Write by A → visible to B immediately!
  Write by B → visible to A immediately!
  Changes written back to file (for file mappings).
```

---

## 7.9 mmap System Call

```c
/* User space API */
#include <sys/mman.h>

void *mmap(void *addr,      /* Requested address (hint, usually NULL) */
           size_t length,    /* Mapping size */
           int prot,         /* Protection: PROT_READ|PROT_WRITE|PROT_EXEC */
           int flags,        /* MAP_SHARED|MAP_PRIVATE, MAP_ANONYMOUS, etc. */
           int fd,           /* File descriptor (-1 for anonymous) */
           off_t offset);    /* Offset in file */

/* Common usage patterns */

/* 1. Anonymous private mapping (like malloc for large allocations) */
void *mem = mmap(NULL, 1024*1024, PROT_READ|PROT_WRITE,
                 MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);

/* 2. File mapping (read a file into memory) */
int fd = open("/etc/passwd", O_RDONLY);
void *file = mmap(NULL, file_size, PROT_READ, MAP_PRIVATE, fd, 0);

/* 3. Shared memory IPC */
int fd = shm_open("/myshm", O_CREAT|O_RDWR, 0666);
ftruncate(fd, 4096);
void *shm = mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);

/* 4. Memory-mapped I/O (device driver) */
void *mmio = mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_SHARED,
                  device_fd, 0);
```

### Kernel-Side mmap Implementation

```c
/* mm/mmap.c */

unsigned long do_mmap(struct file *file, unsigned long addr,
                      unsigned long len, unsigned long prot,
                      unsigned long flags, unsigned long pgoff,
                      unsigned long *populate, struct list_head *uf)
{
    struct mm_struct *mm = current->mm;
    struct vm_area_struct *vma;
    
    /* 1. Validate parameters */
    /* 2. Find free address range */
    addr = get_unmapped_area(file, addr, len, pgoff, flags);
    
    /* 3. Check if can merge with adjacent VMAs */
    vma = vma_merge(mm, prev, addr, addr + len, vm_flags, ...);
    
    if (!vma) {
        /* 4. Create new VMA */
        vma = vm_area_alloc(mm);
        vma->vm_start = addr;
        vma->vm_end = addr + len;
        vma->vm_flags = vm_flags;
        vma->vm_file = file;
        vma->vm_pgoff = pgoff;
        
        /* 5. If file-backed, call file's mmap operation */
        if (file)
            error = call_mmap(file, vma);  /* file->f_op->mmap() */
        
        /* 6. Insert VMA into the tree */
        vma_link(mm, vma, prev);
    }
    
    /* NOTE: No physical pages allocated yet! That's demand paging. */
    
    return addr;
}
```

### Key Point: mmap Does NOT Allocate Physical Memory

```
mmap(NULL, 1GB, ...) → Returns immediately!

What actually happens:
1. VMA created: vm_start to vm_end covering 1GB of virtual space
2. NO physical pages allocated
3. NO page table entries created
4. First access → page fault → kernel allocates physical page on demand

This is why you can mmap() more memory than physically available.
This is called "overcommit" and "demand paging."
```

---

## 7.10 brk System Call

The `brk` system call adjusts the **program break** — the end of the heap.

```c
/* User space — rarely called directly, malloc() uses it */
#include <unistd.h>

int brk(void *addr);          /* Set program break to addr */
void *sbrk(intptr_t increment); /* Increment program break, return old value */

/* How malloc uses brk/sbrk (for small allocations, < 128KB default) */
/* For large allocations, malloc uses mmap() directly */

/* Kernel implementation */
/* mm/mmap.c */

SYSCALL_DEFINE1(brk, unsigned long, brk)
{
    unsigned long newbrk, oldbrk, origbrk;
    struct mm_struct *mm = current->mm;
    
    origbrk = mm->brk;
    
    /* Shrinking the heap */
    if (brk <= mm->brk) {
        mm->brk = brk;
        /* May unmap pages if shrinking by full pages */
        __do_munmap(mm, newbrk, oldbrk - newbrk, ...);
        goto success;
    }
    
    /* Growing the heap */
    /* Check resource limits (RLIMIT_DATA) */
    /* Find/expand VMA for the data segment */
    /* Update mm->brk */
    
    /* Again: NO physical pages allocated — demand paging */
    
success:
    return mm->brk;
}
```

```
Heap growth via brk():

Before:                         After brk(brk + 0x10000):
┌──────────────────┐            ┌──────────────────┐
│ .data / .bss     │            │ .data / .bss     │
├──────────────────┤            ├──────────────────┤
│ Heap (allocated) │            │ Heap (allocated) │
│                  │            │                  │
├──────────────────┤ ← brk     │ New heap space   │ ← 64KB more
│ (unmapped)       │            │ (virtual only!)  │
│                  │            ├──────────────────┤ ← new brk
│                  │            │ (unmapped)       │
└──────────────────┘            └──────────────────┘
```

---

## Comparison: Virtual Memory Management Across OS

| Feature | Linux | Windows | macOS | QNX |
|---------|-------|---------|-------|-----|
| VMA data structure | maple tree (6.1+), rb-tree (older) | VAD tree (AVL) | Mach vm_map (rb-tree) | Custom |
| VMA lock | mmap_lock + per-VMA (6.4+) | Address space lock | vm_map lock | Per-process |
| Heap mechanism | brk() + mmap() | VirtualAlloc + HeapAlloc | vm_allocate / mmap | mmap |
| File mapping | mmap(MAP_SHARED/PRIVATE) | CreateFileMapping | mmap | mmap |
| Overcommit | Yes (configurable: 0/1/2) | Yes (commit charge) | Yes | No (strict) |
| Address randomization | ASLR for stack, mmap, PIE | ASLR (since Vista) | ASLR (since 10.5) | Limited |

---

## Interview Questions

1. **Q: What is a VMA and what does it contain?**
   A: A vm_area_struct describes a contiguous region of virtual memory with uniform permissions. Contains start/end addresses, flags (R/W/X), backing file (if any), and page table protection.

2. **Q: What happens when you call mmap()?**
   A: A VMA is created (or merged with an adjacent one) describing the new mapping. NO physical memory is allocated. On first access, a page fault occurs, and the kernel allocates a physical page on demand.

3. **Q: Explain overcommit in Linux.**
   A: Linux allows processes to allocate more virtual memory than physical RAM exists (controlled by vm.overcommit_memory sysctl: 0=heuristic, 1=always allow, 2=strict). This works because most allocations are never fully used. If memory runs out, the OOM killer is invoked.

4. **Q: How does the mmap_lock affect performance?**
   A: mmap_lock is a reader-writer semaphore protecting VMAs. Page faults take a read lock (concurrent). mmap/munmap take a write lock (exclusive). High mmap rates or many concurrent faults can cause contention. Per-VMA locks (6.4+) significantly reduce this.

5. **Q: How does malloc() use mmap() vs brk()?**
   A: glibc's malloc uses brk() for small allocations (< MMAP_THRESHOLD, default 128KB) and mmap(MAP_ANONYMOUS) for large allocations. brk() grows the heap contiguously; mmap() creates separate mappings that can be individually freed.

---

## Summary & Key Takeaways

1. Each process has an `mm_struct` containing all virtual memory state: VMAs, page tables, and statistics.
2. VMAs (`vm_area_struct`) describe contiguous virtual regions with uniform properties — they form the process's address space.
3. The process memory layout includes text, data, BSS, heap (brk), mmap region, libraries, stack, and kernel mappings.
4. `mmap()` creates VMAs but does NOT allocate physical memory — demand paging handles that on first access.
5. `brk()` adjusts the heap boundary — malloc uses it for small allocations and mmap for large ones.
6. The mmap_lock protects the VMA tree and is a major scalability bottleneck, improved by per-VMA locks in 6.4+.
7. Linux supports overcommit: processes can map more virtual memory than physical RAM, relying on the OOM killer as a safety net.

---

*Previous: [Chapter 6 — Linux Memory Architecture Overview](Chapter_06_Linux_Memory_Architecture.md)*
*Next: [Chapter 8 — Page Allocation Mechanisms](Chapter_08_Page_Allocation.md)*
