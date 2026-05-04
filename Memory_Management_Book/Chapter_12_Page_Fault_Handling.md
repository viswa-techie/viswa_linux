# Chapter 12: Page Fault Handling

## Chapter Overview

Page faults are one of the most performance-critical code paths in the Linux kernel. Every demand-paged access, every copy-on-write, every swap-in goes through the page fault handler. This chapter provides a complete walkthrough of the page fault handling flow.

---

## 12.1 Page Fault Types

```
Page Fault Classification:

By cause:
├── Translation fault: No PTE present (first access)
├── Permission fault: PTE exists but access type not allowed
├── Access flag fault: PTE exists but Accessed bit not set (ARM64)
└── Protection key violation (x86 PKU)

By severity:
├── Minor fault: Page in RAM, just needs PTE mapping (~1µs)
├── Major fault: Page on disk, needs I/O (~100µs-10ms)
└── Invalid fault: Bad address → SIGSEGV

By source:
├── User space fault: Process accesses user memory
├── Kernel fault: Kernel accesses user memory (copy_from_user)
└── Kernel oops: Kernel bugs (NULL pointer, use-after-free)
```

---

## 12.2 Minor Page Faults

A minor fault occurs when the page is already in memory but not currently mapped in the process's page table.

```
Scenarios for minor faults:
1. First access to mmap'd file page (page already in page cache)
2. First access to anonymous page (allocate zero page)  
3. Copy-on-write fault (page in RAM, just needs copying)
4. Page was unmapped from this process but still in RAM

Minor Fault Timeline:
┌─────────────────────────────────────────┐
│ CPU → Page Fault → Exception Handler   │
│ → Find VMA → Allocate/find page        │
│ → Update PTE → Return to user          │
│                                         │
│ Total time: ~1-5 microseconds           │
│ No disk I/O!                            │
└─────────────────────────────────────────┘
```

---

## 12.3 Major Page Faults

A major fault requires disk I/O — the page must be loaded from disk or swap.

```
Scenarios for major faults:
1. File-backed page not in page cache (read from filesystem)
2. Swapped-out page (read from swap device/file)
3. Executable page (.text) first access (read from binary)

Major Fault Timeline:
┌─────────────────────────────────────────┐
│ CPU → Page Fault → Find VMA            │
│ → Page not in RAM                       │
│ → Allocate page frame                   │
│ → Issue disk I/O (block layer)          │
│   ├── NVMe SSD: ~100 microseconds      │
│   └── HDD: ~10 milliseconds            │
│ → Wait for I/O completion               │
│ → Map page in page table                │
│ → Return to user                        │
│                                         │
│ Total: 100µs - 10ms (disk-bound)       │
└─────────────────────────────────────────┘
```

---

## 12.4 Page Fault Handling Flow

### Complete Flow Diagram

```
         Hardware Exception (Page Fault)
                    │
                    ▼
    ┌──────────────────────────────┐
    │  Architecture-specific       │   x86: exc_page_fault()
    │  entry point                 │   ARM64: do_mem_abort()
    └──────────────┬───────────────┘
                   │
                   ▼
    ┌──────────────────────────────┐
    │  Was fault in kernel mode?   │
    └──────────────┬───────────────┘
            │              │
         Kernel          User
            │              │
            ▼              ▼
    ┌──────────────┐ ┌─────────────────────┐
    │ Check if in  │ │  Find VMA for       │
    │ exception    │ │  faulting address    │
    │ table (fixup)│ │  find_vma(mm, addr)  │
    └──────┬───────┘ └─────────┬───────────┘
           │                   │
           │            ┌──────┴──────┐
           │         Found         Not Found
           │            │              │
           │            ▼              ▼
           │    ┌──────────────┐  ┌──────────┐
           │    │ Check VMA    │  │ Bad area │
           │    │ permissions  │  │ SIGSEGV  │
           │    └──────┬───────┘  └──────────┘
           │           │
           │    ┌──────┴──────┐
           │  Valid        Invalid
           │    │              │
           │    ▼              ▼
           │ ┌──────────────┐ ┌──────────┐
           │ │handle_mm_    │ │ SIGSEGV  │
           │ │fault()       │ │ or SIGBUS│
           │ └──────┬───────┘ └──────────┘
           │        │
           │        ▼
           │ ┌──────────────────────────┐
           │ │ Walk page table levels:  │
           │ │ PGD→P4D→PUD→PMD→PTE     │
           │ │ Allocate tables if needed│
           │ └──────────┬───────────────┘
           │            │
           │            ▼
           │ ┌───────────────────────────┐
           │ │ handle_pte_fault():       │
           │ │  ├── pte_none() →         │
           │ │  │   ├── file: do_fault() │
           │ │  │   └── anon: do_anon()  │
           │ │  ├── !pte_present() →     │
           │ │  │   └── do_swap_page()   │
           │ │  ├── pte_protnone() →     │
           │ │  │   └── do_numa_page()   │
           │ │  └── write + !pte_write →│
           │ │      └── do_wp_page()     │
           │ └───────────────────────────┘
           │
           ▼
    ┌──────────────────────┐
    │ Kernel oops/panic    │  
    │ (if no fixup entry)  │
    └──────────────────────┘
```

---

## 12.5 Kernel Page Fault Handler

### x86_64 Entry Point

```c
/* arch/x86/mm/fault.c */

DEFINE_IDTENTRY_RAW_ERRORCODE(exc_page_fault)
{
    unsigned long address = read_cr2();  /* Faulting virtual address */
    irqentry_state_t state;
    
    /* Handle kernel-mode faults that can be fixed up */
    if (unlikely(kfault_fixup(regs, error_code, address)))
        return;
    
    state = irqentry_enter(regs);
    
    handle_page_fault(regs, error_code, address);
    
    irqentry_exit(regs, state);
}

static void handle_page_fault(struct pt_regs *regs,
                              unsigned long error_code,
                              unsigned long address)
{
    /* Kernel fault? */
    if (unlikely(fault_in_kernel_space(address))) {
        do_kern_addr_fault(regs, error_code, address);
        return;
    }
    
    /* User fault */
    do_user_addr_fault(regs, error_code, address);
}
```

### User Space Fault Handler

```c
/* arch/x86/mm/fault.c — simplified */

static void do_user_addr_fault(struct pt_regs *regs,
                               unsigned long error_code,
                               unsigned long address)
{
    struct mm_struct *mm = current->mm;
    struct vm_area_struct *vma;
    vm_fault_t fault;
    
    /* Try per-VMA lock first (fast path, 6.4+) */
    vma = lock_vma_under_rcu(mm, address);
    if (vma) {
        /* Handle fault with per-VMA lock (no mmap_lock needed) */
        fault = handle_mm_fault(vma, address, flags, regs);
        /* ... */
        return;
    }
    
    /* Fall back to mmap_lock */
    mmap_read_lock(mm);
    
    /* Find the VMA containing this address */
    vma = find_vma(mm, address);
    
    if (unlikely(!vma)) {
        /* No VMA → bad area */
        bad_area(regs, error_code, address);
        return;
    }
    
    if (unlikely(vma->vm_start > address)) {
        /* Address is below VMA — check if stack can grow */
        if (!(vma->vm_flags & VM_GROWSDOWN)) {
            bad_area(regs, error_code, address);
            return;
        }
        expand_stack(vma, address);
    }
    
    /* Check permissions */
    if (unlikely(access_error(error_code, vma))) {
        bad_area_access_error(regs, error_code, address, vma);
        return;
    }
    
    /* Handle the fault! */
    fault = handle_mm_fault(vma, address, flags, regs);
    
    if (fault & VM_FAULT_OOM) {
        pagefault_out_of_memory();
    }
    
    mmap_read_unlock(mm);
}
```

---

## 12.6 do_page_fault() Mechanism

### handle_mm_fault() — The Core

```c
/* mm/memory.c */

vm_fault_t handle_mm_fault(struct vm_area_struct *vma,
                           unsigned long address,
                           unsigned int flags,
                           struct pt_regs *regs)
{
    /* Handle huge page faults at PUD/PMD level */
    if (pud_trans_huge(*pud) || pud_devmap(*pud))
        return create_huge_pud(vmf);
    if (pmd_trans_huge(*pmd) || pmd_devmap(*pmd))
        return create_huge_pmd(vmf);
    
    /* Regular 4KB page fault */
    return handle_pte_fault(&vmf);
}

static vm_fault_t handle_pte_fault(struct vm_fault *vmf)
{
    pte_t entry;
    
    if (!vmf->pte) {
        /* PTE doesn't exist yet */
        if (vma_is_anonymous(vmf->vma))
            return do_anonymous_page(vmf);    /* Anonymous page */
        else
            return do_fault(vmf);              /* File-backed page */
    }
    
    entry = vmf->orig_pte;
    
    if (!pte_present(entry)) {
        if (pte_none(entry))
            return do_anonymous_page(vmf);     /* Empty PTE */
        
        if (is_swap_pte(entry))
            return do_swap_page(vmf);          /* Swapped out */
        
        if (is_migration_entry(entry))
            return do_migration_page(vmf);     /* Being migrated */
    }
    
    if (pte_protnone(entry))
        return do_numa_page(vmf);              /* NUMA hint fault */
    
    /* PTE present, write fault, page not writable → COW */
    if (vmf->flags & FAULT_FLAG_WRITE) {
        if (!pte_write(entry))
            return do_wp_page(vmf);            /* Copy-on-Write */
    }
    
    /* Just update accessed bit and return */
    entry = pte_mkyoung(entry);
    set_pte_at(vmf->vma->vm_mm, vmf->address, vmf->pte, entry);
    
    return 0;
}
```

### Fault Return Values

```c
/* include/linux/mm_types.h */

typedef unsigned int vm_fault_t;

#define VM_FAULT_OOM            0x0001  /* Out of memory */
#define VM_FAULT_SIGBUS         0x0002  /* Bad access (SIGBUS) */
#define VM_FAULT_MAJOR          0x0004  /* Major fault (disk I/O) */
#define VM_FAULT_WRITE          0x0008  /* Write fault */
#define VM_FAULT_HWPOISON       0x0010  /* Hardware memory error */
#define VM_FAULT_HWPOISON_LARGE 0x0020  /* HW error on huge page */
#define VM_FAULT_SIGSEGV        0x0040  /* SIGSEGV */
#define VM_FAULT_NOPAGE         0x0100  /* No page returned */
#define VM_FAULT_LOCKED         0x0200  /* Page locked */
#define VM_FAULT_RETRY           0x0400  /* Retry fault */
#define VM_FAULT_FALLBACK       0x0800  /* Huge page fallback */
#define VM_FAULT_DONE_COW       0x1000  /* COW completed */
```

---

## Monitoring Page Faults

```bash
# Per-process page fault counts
$ cat /proc/<pid>/stat | awk '{print "minor:", $10, "major:", $12}'

# System-wide
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 15742688 123456 8456789 0    0     4    12  150   300  5  2 93  0  0

# Detailed per-process
$ ps -o pid,min_flt,maj_flt,cmd -p <pid>
  PID  MINFL  MAJFL CMD
 1234  55432     12 /usr/bin/my_app

# Real-time monitoring with perf
$ perf stat -e page-faults,minor-faults,major-faults ./my_program
Performance counter stats:
    10,547      page-faults
    10,535      minor-faults     (99.89%)
        12      major-faults     (0.11%)
```

---

## Interview Questions

1. **Q: Walk through the complete page fault handling path.**
   A: Hardware exception → arch-specific handler → determine user/kernel → find VMA → check permissions → handle_mm_fault → walk page table → handle_pte_fault → dispatch: do_anonymous_page (new anon), do_fault (file-backed), do_swap_page (swap in), do_wp_page (COW) → allocate/find page → update PTE → return.

2. **Q: What is the difference between a minor and major page fault?**
   A: Minor: page is in RAM (page cache or just needs PTE setup) — no disk I/O, ~1-5µs. Major: page must be loaded from disk (filesystem or swap) — 100µs to 10ms.

3. **Q: What happens when a process accesses an invalid address?**
   A: The VMA lookup fails (no VMA covers the address), the fault handler sends SIGSEGV to the process. Default handler: crash with core dump.

4. **Q: How do per-VMA locks improve performance?**
   A: Instead of taking the global mmap_lock for every page fault, Linux 6.4+ uses per-VMA locks. Multiple threads can handle page faults in different VMAs concurrently without contention.

---

## Summary & Key Takeaways

1. Page faults are classified by type (translation, permission), severity (minor, major), and source (user, kernel).
2. Minor faults are fast (~1-5µs, no I/O); major faults are slow (~100µs-10ms, disk I/O).
3. The handler flow: find VMA → check permissions → walk page tables → dispatch to specific handler.
4. `handle_pte_fault()` dispatches to: `do_anonymous_page()`, `do_fault()`, `do_swap_page()`, `do_wp_page()`, or `do_numa_page()`.
5. Per-VMA locks (6.4+) significantly reduce mmap_lock contention on multi-threaded workloads.

---

*Previous: [Chapter 11 — Demand Paging](Chapter_11_Demand_Paging.md)*
*Next: [Chapter 13 — Copy-on-Write Mechanism](Chapter_13_Copy_on_Write.md)*
