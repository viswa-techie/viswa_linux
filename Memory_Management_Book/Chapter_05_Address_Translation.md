# Chapter 5: Address Translation Mechanism

## Chapter Overview

This chapter provides a deep dive into how virtual addresses are translated to physical addresses in Linux. We cover the exact bit layout of virtual addresses, multi-level page table structures, page table entries, the TLB lookup process, and the complete hardware page walk on x86_64 and ARM64. This is essential knowledge for kernel developers, driver writers, and anyone debugging memory issues.

---

## 5.1 Virtual Address Format

### x86_64 (4-Level Paging, 48-bit VA)

```
63          48 47    39 38    30 29    21 20    12 11        0
┌─────────────┬────────┬────────┬────────┬────────┬──────────┐
│  Sign Ext   │ PML4   │  PDPT  │   PD   │   PT   │  Offset  │
│  (16 bits)  │ Index  │ Index  │ Index  │ Index  │ (12 bits)│
│  must match │(9 bits)│(9 bits)│(9 bits)│(9 bits)│          │
│  bit 47     │        │        │        │        │          │
└─────────────┴────────┴────────┴────────┴────────┴──────────┘

Canonical address requirement:
- Bits 63:48 must be copies of bit 47
- User space: bits 63:48 = 0 → range 0x0000000000000000 - 0x00007FFFFFFFFFFF
- Kernel:     bits 63:48 = 1 → range 0xFFFF800000000000 - 0xFFFFFFFFFFFFFFFF
- "Hole": 0x0000800000000000 - 0xFFFF7FFFFFFFFFFF → access causes #GP fault
```

### x86_64 (5-Level Paging, 57-bit VA — LA57)

```
63          57 56    48 47    39 38    30 29    21 20    12 11        0
┌─────────────┬────────┬────────┬────────┬────────┬────────┬──────────┐
│  Sign Ext   │  PML5  │ PML4   │  PDPT  │   PD   │   PT   │  Offset  │
│  (7 bits)   │ Index  │ Index  │ Index  │ Index  │ Index  │ (12 bits)│
│             │(9 bits)│(9 bits)│(9 bits)│(9 bits)│(9 bits)│          │
└─────────────┴────────┴────────┴────────┴────────┴────────┴──────────┘

User space: 0 to 0x00FFFFFFFFFFFFFF (64 PB)
Kernel:     0xFF00000000000000 to 0xFFFFFFFFFFFFFFFF  
```

### ARM64 (4KB granule, 48-bit VA)

```
63          48 47    39 38    30 29    21 20    12 11        0
┌─────────────┬────────┬────────┬────────┬────────┬──────────┐
│  TTBR select│  L0    │   L1   │   L2   │   L3   │  Offset  │
│  (bit 55 or │ Index  │ Index  │ Index  │ Index  │ (12 bits)│
│   top bits) │(9 bits)│(9 bits)│(9 bits)│(9 bits)│          │
└─────────────┴────────┴────────┴────────┴────────┴──────────┘

TTBR selection:
- VA[63:48] = 0x0000 → TTBR0_EL1 (user space)
- VA[63:48] = 0xFFFF → TTBR1_EL1 (kernel space)
```

---

## 5.2 Page Table Structures

### Linux Generic Page Table Abstraction

Linux uses a **5-level page table** abstraction (since kernel 4.11) that compiles down to fewer levels on architectures that don't support 5 levels.

```
Linux Page Table Levels:
┌─────┐     ┌─────┐     ┌─────┐     ┌─────┐     ┌─────┐
│ PGD │────→│ P4D │────→│ PUD │────→│ PMD │────→│ PTE │──→ Page Frame
└─────┘     └─────┘     └─────┘     └─────┘     └─────┘

PGD = Page Global Directory     (top level)
P4D = Page 4th-level Directory  (5-level paging only, else folded)
PUD = Page Upper Directory      
PMD = Page Middle Directory     (can be huge page: 2MB on x86)
PTE = Page Table Entry          (points to 4KB page frame)
```

### How Levels Map to Architectures

| Linux Level | x86_64 (4-level) | x86_64 (5-level) | ARM64 (4K/48-bit) |
|-------------|-------------------|-------------------|---------------------|
| PGD | PML4 | PML5 | Level 0 |
| P4D | (folded into PGD) | PML4 | (folded) |
| PUD | PDPT | PDPT | Level 1 |
| PMD | PD | PD | Level 2 |
| PTE | PT | PT | Level 3 |

```c
/* include/linux/pgtable.h — Generic page table API */

/* Walk the page table */
pgd_t *pgd = pgd_offset(mm, address);     /* Get PGD entry */
p4d_t *p4d = p4d_offset(pgd, address);    /* Get P4D entry */
pud_t *pud = pud_offset(p4d, address);    /* Get PUD entry */
pmd_t *pmd = pmd_offset(pud, address);    /* Get PMD entry */
pte_t *pte = pte_offset_map(pmd, address);/* Get PTE entry */

/* Extract physical address from PTE */
unsigned long pfn = pte_pfn(*pte);
phys_addr_t phys = PFN_PHYS(pfn) | (address & ~PAGE_MASK);
```

---

## 5.3 Multi-Level Page Tables

### Why Multi-Level?

A single-level page table for 48-bit address space with 4KB pages would need:
- 2^36 entries × 8 bytes = **512 GB** per process — impossible!

Multi-level page tables are **sparse**: only populated where the process has actual mappings.

```
Single-Level (impractical):           Multi-Level (practical):
┌─────────────────────────┐           ┌────────┐ PGD (4KB)
│   512 GB of page table  │           │ Entry 0│──→ NULL (not mapped)
│   entries — even for    │           │ Entry 1│──→ NULL
│   a tiny process!       │           │ Entry 2│──→ ┌────────┐ PUD (4KB)
│                         │           │  ...   │    │ Entry 0│──→ NULL
│                         │           │Entry511│    │ Entry 1│──→ ┌────┐PMD
└─────────────────────────┘           └────────┘    │  ...   │    │    │
                                                    └────────┘    └────┘
                                      Only a few KB for a simple process!
```

### Memory Cost Analysis

```
Typical process page table memory usage:

Simple program (few mappings):
  PGD: 1 page (4KB)
  P4D: 1 page (4KB)  [or folded]
  PUD: 1-2 pages (4-8KB)
  PMD: 2-5 pages (8-20KB)
  PTE: 5-20 pages (20-80KB)
  Total: ~40-120KB

Large application (Chrome with many tabs):
  Thousands of VMAs, many shared libraries
  Total: ~1-10MB of page tables

Database server (1TB working set):
  Millions of PTEs
  Total: ~2-4GB of page tables (reduced with huge pages)
```

---

## 5.4 Page Directory Hierarchy

### Complete 4-Level Hierarchy (x86_64)

```
CR3 Register
  │
  └──→ PML4 Table (Page Global Directory)
       512 entries, each 8 bytes = 4KB total
       Each entry covers 512GB of virtual space
       │
       ├── PML4[0] ──→ PDPT Table (Page Upper Directory)
       │                512 entries, each covers 1GB
       │                │
       │                ├── PDPT[0] ──→ PD Table (Page Middle Directory)
       │                │               512 entries, each covers 2MB
       │                │               │
       │                │               ├── PD[0] ──→ PT Table (Page Table)
       │                │               │             512 entries, each covers 4KB
       │                │               │             │
       │                │               │             ├── PT[0] → Frame 0x1A000
       │                │               │             ├── PT[1] → Frame 0x2B000
       │                │               │             └── ...
       │                │               │
       │                │               ├── PD[1] → (can be 2MB huge page!)
       │                │               └── ...
       │                │
       │                ├── PDPT[1] → (can be 1GB huge page!)
       │                └── ...
       │
       ├── PML4[256-511] ──→ Kernel space (shared across all processes)
       └── ...

Number of potential entries at each level:
PML4: 512 entries → covers 256TB (user) + 256TB (kernel)
PDPT: 512 entries → each covers 1GB
PD:   512 entries → each covers 2MB
PT:   512 entries → each covers 4KB
```

---

## 5.5 Page Table Entries (PTE)

### x86_64 PTE Format (4KB Page)

```
Bit  63   62:52  51:M   M-1:12     11:9   8    7    6    5    4    3    2    1    0
┌────┬────────┬──────┬───────────┬──────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ NX │ Avail  │ Rsvd │ Phys Addr │ Avl  │ G  │PAT │ D  │ A  │PCD │PWT │U/S │R/W │ P  │
│    │(soft)  │      │ [M-1:12]  │(soft)│    │    │    │    │    │    │    │    │    │
└────┴────────┴──────┴───────────┴──────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘

P   (0):  Present — 1 if page is in physical memory
R/W (1):  Read/Write — 0 = read-only, 1 = read-write  
U/S (2):  User/Supervisor — 0 = kernel only, 1 = user accessible
PWT (3):  Page Write-Through cache policy
PCD (4):  Page Cache Disable
A   (5):  Accessed — set by MMU on read/write (used by LRU)
D   (6):  Dirty — set by MMU on write (used by writeback)
PAT (7):  Page Attribute Table index bit
G   (8):  Global — not flushed on CR3 reload (kernel pages)
Avl (11:9): Available for OS use (Linux uses these!)
PFN (M-1:12): Physical Frame Number
NX  (63): No-Execute — 1 = page not executable
```

### Linux's Use of PTE Bits

```c
/* arch/x86/include/asm/pgtable_types.h */

#define _PAGE_PRESENT   (1UL << 0)
#define _PAGE_RW        (1UL << 1)
#define _PAGE_USER      (1UL << 2)
#define _PAGE_PWT       (1UL << 3)
#define _PAGE_PCD       (1UL << 4)
#define _PAGE_ACCESSED  (1UL << 5)
#define _PAGE_DIRTY     (1UL << 6)
#define _PAGE_PSE       (1UL << 7)  /* Page Size Extension (huge page) */
#define _PAGE_GLOBAL    (1UL << 8)
#define _PAGE_NX        (1UL << 63)

/* Software-defined bits (in available bits) */
#define _PAGE_SOFT_DIRTY    (1UL << 9)  /* Soft-dirty tracking */
#define _PAGE_SPECIAL       (1UL << 10) /* Special mapping */
#define _PAGE_PROTNONE      (1UL << 8)  /* NUMA balancing marker */

/* When P=0 (not present), the PTE format changes completely: */
/* Linux stores swap entry information in non-present PTEs */
/*
 * Non-present PTE (swap entry):
 * Bit 0: 0 (not present)
 * Bits 1-4: swap type
 * Bits 5-63: swap offset
 */
```

### ARM64 PTE Format

```
ARM64 Page Table Descriptor (Stage 1, 4KB granule, Level 3):

Bits 63:52  51:48   47:12        11:2          1:0
┌──────────┬──────┬─────────────┬─────────────┬─────┐
│ Upper    │ Rsvd │ Output Addr │ Lower Block │Type │
│ Attrs    │      │ (OA[47:12]) │ Attributes  │     │
└──────────┴──────┴─────────────┴─────────────┴─────┘

Upper Attributes:
  [63] PBHA (Page-Based Hardware Attributes)  
  [54] UXN (Unprivileged Execute Never)
  [53] PXN (Privileged Execute Never)
  [52] Contiguous hint

Lower Attributes:
  [11]    nG (not Global)
  [10]    AF (Access Flag) 
  [9:8]   SH (Shareability: non/inner/outer)
  [7:6]   AP (Access Permission)
  [4:2]   AttrIndx (memory type index into MAIR)

AP field:
  AP[2:1] = 00: EL1 R/W, EL0 none
  AP[2:1] = 01: EL1 R/W, EL0 R/W
  AP[2:1] = 10: EL1 R/O, EL0 none
  AP[2:1] = 11: EL1 R/O, EL0 R/O
```

---

## 5.6 Address Translation Flow

### Complete Translation Flow Diagram

```
                        Virtual Address
                    ┌────────────────────┐
                    │ PGD│PUD│PMD│PTE│Off│
                    └─┬──┴─┬─┴─┬─┴─┬─┴──┘
                      │    │   │   │  │
          ┌───────────┘    │   │   │  │
          │                │   │   │  │
          ▼                │   │   │  │
    ┌──────────┐           │   │   │  │
CR3→│ PGD Table│           │   │   │  │
    │ (PML4)   │           │   │   │  │
    │ [idx]────┼───┐       │   │   │  │
    └──────────┘   │       │   │   │  │
                   ▼       │   │   │  │
             ┌──────────┐  │   │   │  │
             │ PUD Table│  │   │   │  │
             │ (PDPT)   │←─┘   │   │  │
             │ [idx]────┼───┐  │   │  │
             └──────────┘   │  │   │  │
                            ▼  │   │  │
                      ┌──────────┐ │  │
                      │ PMD Table│ │  │
                      │  (PD)   │←─┘  │
                      │ [idx]───┼──┐  │
                      └─────────┘  │  │
                                   ▼  │
                             ┌──────────┐
                             │ PTE Table│
                             │  (PT)   │←─┘
                             │ [idx]───┼──→ Physical Frame Number
                             └─────────┘         │
                                                  │
                                    ┌─────────────┘
                                    ▼
                              Physical Address = PFN << 12 | Offset
```

### Step-by-Step Example

```
Virtual Address: 0x00007F4A 12345678

Step 0: Break down address
  Binary: 0000 0000 0000 0000 0111 1111 0100 1010 
          0001 0010 0011 0100 0101 0110 0111 1000

  PGD index: bits[47:39] = 0x0FE = 254
  PUD index: bits[38:30] = 0x128 = 296  
  PMD index: bits[29:21] = 0x091 = 145
  PTE index: bits[20:12] = 0x145 = 325
  Offset:    bits[11:0]  = 0x678

Step 1: PGD lookup
  PGD base = CR3 value (e.g., 0x00000001A0000)
  PGD entry address = 0x00000001A0000 + 254*8 = 0x00000001A07F0
  Read PGD[254] → contains PUD base address (e.g., 0x00000002B0000)

Step 2: PUD lookup  
  PUD entry address = 0x00000002B0000 + 296*8 = 0x00000002B0940
  Read PUD[296] → contains PMD base address (e.g., 0x00000003C0000)

Step 3: PMD lookup
  PMD entry address = 0x00000003C0000 + 145*8 = 0x00000003C0488
  Read PMD[145] → contains PTE base address (e.g., 0x00000004D0000)

Step 4: PTE lookup
  PTE entry address = 0x00000004D0000 + 325*8 = 0x00000004D0A28
  Read PTE[325] → contains PFN (e.g., 0x5E000)
  
Step 5: Form physical address
  Physical Address = 0x5E000 << 12 | 0x678 = 0x5E000678

  Wait — that's wrong! PFN is already shifted in the PTE.
  PTE value & PFN_MASK gives physical frame base directly.
  Physical Address = (PTE & PFN_MASK) | Offset = 0x5E000000 | 0x678 = 0x5E000678
```

---

## 5.7 TLB Lookup Process

```
                CPU generates Virtual Address (VA)
                            │
                            ▼
                    ┌───────────────┐
                    │  Extract VPN  │  (VA >> PAGE_SHIFT)
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ L1 TLB Lookup │  (1 cycle)
                    │  (ITLB/DTLB)  │
                    └───────┬───────┘
                        │       │
                     Hit       Miss
                        │       │
                        │   ┌───▼───────────┐
                        │   │ L2 TLB Lookup │  (~7 cycles)
                        │   │   (STLB)      │
                        │   └───┬───────────┘
                        │   │       │
                        │  Hit    Miss
                        │   │       │
                        │   │   ┌───▼───────────────┐
                        │   │   │ Hardware Page Walk│  (tens to
                        │   │   │ (walk page tables │   hundreds
                        │   │   │  in memory)       │   of cycles)
                        │   │   └───┬───────────────┘
                        │   │       │           │
                        │   │    Found       Not Found
                        │   │       │           │
                        │   │   Fill TLB     Page Fault
                        │   │       │        Exception
                        ▼   ▼       ▼
                    ┌───────────────────┐
                    │ PFN + Permissions │
                    │ + Memory Type     │
                    └───────┬───────────┘
                            │
                    ┌───────▼───────┐
                    │ Permission    │
                    │ Check         │
                    └───────┬───────┘
                        │       │
                     Pass     Fail
                        │       │
                        ▼    Protection
                Physical     Fault
                Address      (SIGSEGV)
```

---

## 5.8 Page Table Walk Process

### Linux Kernel Page Table Walk Code

```c
/* mm/memory.c — Software page table walk */
/* Used by kernel code that needs to find the PTE for a virtual address */

static int __follow_pte(struct mm_struct *mm, unsigned long address,
                        pte_t **ptepp, spinlock_t **ptlp)
{
    pgd_t *pgd;
    p4d_t *p4d;
    pud_t *pud;
    pmd_t *pmd;
    pte_t *ptep;

    pgd = pgd_offset(mm, address);
    if (pgd_none(*pgd) || pgd_bad(*pgd))
        goto out;

    p4d = p4d_offset(pgd, address);
    if (p4d_none(*p4d) || p4d_bad(*p4d))
        goto out;

    pud = pud_offset(p4d, address);
    if (pud_none(*pud) || pud_bad(*pud))
        goto out;

    /* Check for 1GB huge page at PUD level */
    if (pud_huge(*pud))
        return handle_huge_pud(pud, address);

    pmd = pmd_offset(pud, address);
    if (pmd_none(*pmd) || pmd_bad(*pmd))
        goto out;

    /* Check for 2MB huge page at PMD level */
    if (pmd_huge(*pmd))
        return handle_huge_pmd(pmd, address);

    ptep = pte_offset_map(pmd, address);
    if (!ptep)
        goto out;
    if (!pte_present(*ptep))
        goto unlock;

    *ptepp = ptep;
    return 0;

unlock:
    pte_unmap(ptep);
out:
    return -EINVAL;
}
```

### Kernel Helper Macros for Page Table Walk

```c
/* include/linux/pgtable.h */

/* Get PGD entry for address in given mm */
#define pgd_offset(mm, address)  ((mm)->pgd + pgd_index(address))

/* Extract index at each level */
#define pgd_index(addr)  (((addr) >> PGDIR_SHIFT) & (PTRS_PER_PGD - 1))
#define p4d_index(addr)  (((addr) >> P4D_SHIFT) & (PTRS_PER_P4D - 1))
#define pud_index(addr)  (((addr) >> PUD_SHIFT) & (PTRS_PER_PUD - 1))
#define pmd_index(addr)  (((addr) >> PMD_SHIFT) & (PTRS_PER_PMD - 1))
#define pte_index(addr)  (((addr) >> PAGE_SHIFT) & (PTRS_PER_PTE - 1))

/* x86_64 values: */
#define PAGE_SHIFT    12
#define PMD_SHIFT     21   /* 12 + 9 */
#define PUD_SHIFT     30   /* 12 + 9 + 9 */
#define PGDIR_SHIFT   39   /* 12 + 9 + 9 + 9 */
#define PTRS_PER_PTE  512
#define PTRS_PER_PMD  512
#define PTRS_PER_PUD  512
#define PTRS_PER_PGD  512
```

---

## 5.9 TLB Shootdown Mechanism

### Why TLB Shootdowns Are Needed

```
Scenario: Process unmaps a page at VA 0x7F000000

CPU 0 (running the process):
1. Modifies page table entry → set P=0
2. Flushes local TLB entry for 0x7F000000
3. BUT: Other CPUs may have this cached in their TLBs!

Without shootdown:
  CPU 1 still has TLB: VA 0x7F000000 → PFN 0x1234
  CPU 1 accesses 0x7F000000 → gets old, stale physical page!
  This is a SECURITY BUG and CORRECTNESS BUG.
```

### Linux TLB Shootdown Implementation

```c
/* arch/x86/mm/tlb.c */

void flush_tlb_mm_range(struct mm_struct *mm,
                        unsigned long start,
                        unsigned long end,
                        unsigned int stride_shift,
                        bool freed_tables)
{
    struct flush_tlb_info *info;
    
    /* Determine which CPUs need to be told */
    /* mm->cpu_bitmap tracks which CPUs are running this mm */
    
    if (cpumask_any_but(mm_cpumask(mm), smp_processor_id()) < nr_cpu_ids) {
        /* Other CPUs are running this mm — must send IPI */
        info = get_flush_tlb_info(mm, start, end, stride_shift, freed_tables);
        flush_tlb_multi(mm_cpumask(mm), info);
        put_flush_tlb_info();
    }
    
    /* Also flush local TLB */
    /* ... */
}

/* The IPI handler on remote CPUs */
static void flush_tlb_func(void *info)
{
    struct flush_tlb_info *f = info;
    
    /* Invalidate the specified range */
    if (f->end == TLB_FLUSH_ALL) {
        /* Full TLB flush */
        __flush_tlb_all();
    } else {
        /* Range-based flush */
        unsigned long addr;
        for (addr = f->start; addr < f->end; addr += stride) {
            __invlpg(addr);  /* Invalidate single page */
        }
    }
}
```

### TLB Shootdown Performance Impact

```
TLB shootdown cost breakdown:
1. IPI send overhead:          ~1-5µs (interrupt dispatch)
2. IPI delivery latency:       ~1-10µs (depends on target CPU state)
3. TLB invalidation:           ~100ns per entry
4. Total per shootdown event:  ~5-50µs
5. On 128-core NUMA system:    Up to 500µs+ 

Mitigation strategies in Linux:
├── Batching: Accumulate TLB flushes and do one shootdown
├── Lazy TLB: Delay flush if mm is not actively running on a CPU
├── PCID/ASID: Avoid flush on context switch entirely
├── Per-VMA locks (6.4+): Reduce need for TLB shootdowns
└── Range invalidation: INVLPG for small ranges, full flush for large
```

---

## 5.10 Address Space Identifiers

### PCID (Process Context ID) — x86_64

```
Without PCID:
  Context switch A→B: Full TLB flush (EXPENSIVE!)
  All of A's TLB entries lost, even if we switch back to A soon.

With PCID:
  Each process gets a PCID (12-bit → 4096 values)
  TLB entries tagged with PCID
  Context switch: Just change CR3, no flush needed!
  A's entries stay in TLB, ready if we switch back.
```

```c
/* x86_64 CR3 with PCID */
/* CR3[11:0] = PCID (when CR4.PCIDE=1) */
/* CR3[63] = 0 → flush TLB entries with this PCID */
/* CR3[63] = 1 → don't flush (preserve entries) */

/* Linux PCID management */
/* arch/x86/mm/tlb.c */

#define TLB_NR_DYN_ASIDS    6  /* Use up to 6 PCIDs dynamically */

struct tlb_state {
    struct mm_struct *loaded_mm;
    u16 loaded_mm_asid;
    /* ... */
    struct tlb_context ctxs[TLB_NR_DYN_ASIDS];
};

DEFINE_PER_CPU_ALIGNED(struct tlb_state, cpu_tlbstate);
```

### ASID — ARM64

```c
/* ARM64: ASID stored in upper bits of TTBR0_EL1 */
/* 8-bit ASID: 256 values (older cores) */
/* 16-bit ASID: 65536 values (newer cores like Cortex-A76+) */

/* TCR_EL1.AS bit selects 8 vs 16-bit ASID */

/* When ASIDs wrap around, Linux does a full TLB flush */
```

---

## Comparison: Address Translation Across Architectures

| Aspect | x86_64 | ARM64 | RISC-V (Sv48) |
|--------|--------|-------|----------------|
| Page table levels | 4 (5 with LA57) | 3-4 (configurable) | 4 |
| VA bits | 48 (57) | 48 (52 with LVA) | 48 |
| PA bits | 52 | 48 (52 with LPA) | 56 |
| Entries per table | 512 | 512 (4K), 2048 (16K) | 512 |
| Huge page sizes | 2MB, 1GB | 2MB, 1GB (4K granule) | 2MB, 1GB |
| ASID/PCID bits | 12 (PCID) | 8 or 16 (ASID) | 16 (ASID) |
| Page table base reg | CR3 | TTBR0/TTBR1 | satp |
| User/Kernel split | Both in CR3 (shared PGD) | Separate TTBR0/TTBR1 | satp (shared) |
| TLB shootdown | INVLPG + IPI | TLBI + broadcasting | SFENCE.VMA |

---

## Interview Questions

1. **Q: Describe the 4-level page table walk on x86_64.**
   A: CR3 points to PML4. Extract bits[47:39] as PML4 index, read entry to get PDPT base. Extract bits[38:30] as PDPT index, read entry to get PD base. Extract bits[29:21] as PD index, read entry to get PT base. Extract bits[20:12] as PT index, read PTE. Combine PFN from PTE with bits[11:0] offset for physical address.

2. **Q: Why does Linux need 5 levels of page tables?**
   A: To support 57-bit virtual addressing (Intel LA57), which provides 128 PB of virtual address space. The extra P4D level was added in kernel 4.11. On systems without LA57, P4D is folded (compiled away) into PGD.

3. **Q: What is a TLB shootdown? When does it happen?**
   A: When a page table entry is modified (e.g., munmap, mprotect), all CPUs that might have cached the old translation must invalidate their TLB entries. This requires sending IPIs (Inter-Processor Interrupts). It's expensive on large multi-core systems.

4. **Q: What are PCIDs and why are they important?**
   A: PCIDs (Process Context Identifiers) tag TLB entries with a process ID, allowing multiple processes' translations to coexist in the TLB. Without PCIDs, every context switch flushes the entire TLB. With PCIDs, TLB entries from the previous process are preserved.

5. **Q: How does a non-present PTE differ from a present PTE?**
   A: When P=0, the PTE format changes. Linux uses the remaining bits to store swap information (swap type + offset) or migration entry data. The MMU ignores all other bits when P=0, so the OS is free to use them.

---

## Summary & Key Takeaways

1. Virtual addresses are split into indices (9 bits each for 4 levels) plus a 12-bit page offset.
2. Multi-level page tables are sparse — only populated regions consume memory.
3. Each PTE contains the physical frame number plus permission/attribute bits (Present, R/W, U/S, NX, Dirty, Accessed).
4. TLB caches translations; misses trigger expensive hardware page walks (up to 4 memory accesses).
5. TLB shootdowns (IPIs to invalidate remote TLBs) are one of the most expensive operations in the memory subsystem.
6. PCIDs (x86) and ASIDs (ARM64) avoid TLB flushes on context switches by tagging entries.
7. Linux uses a generic 5-level page table API that compiles to the appropriate depth per architecture.

---

*Previous: [Chapter 4 — Virtual Memory Concepts](Chapter_04_Virtual_Memory_Concepts.md)*
*Next: [Chapter 6 — Linux Memory Architecture Overview](Chapter_06_Linux_Memory_Architecture.md)*
