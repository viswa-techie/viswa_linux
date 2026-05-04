# Chapter 3: Hardware Architecture for Memory Management

## Chapter Overview

Memory management is a partnership between hardware and software. The CPU provides the Memory Management Unit (MMU), Translation Lookaside Buffer (TLB), caches, and memory controllers. The OS (Linux kernel) programs these hardware units to implement virtual memory, protection, and efficient allocation. This chapter covers every hardware component involved in memory management.

---

## 3.1 CPU Memory Subsystem Overview

The CPU's memory subsystem is the hardware infrastructure that connects the processor cores to main memory.

```
┌─────────────────────────────────────────────────────────┐
│                        CPU Package                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐│
│  │  Core 0  │  │  Core 1  │  │  Core 2  │  │  Core 3  ││
│  │┌────────┐│  │┌────────┐│  │┌────────┐│  │┌────────┐││
│  ││L1I+L1D ││  ││L1I+L1D ││  ││L1I+L1D ││  ││L1I+L1D │││
│  │├────────┤│  │├────────┤│  │├────────┤│  │├────────┤││
│  ││  L2    ││  ││  L2    ││  ││  L2    ││  ││  L2    │││
│  │├────────┤│  │├────────┤│  │├────────┤│  │├────────┤││
│  ││  TLB   ││  ││  TLB   ││  ││  TLB   ││  ││  TLB   │││
│  ││  MMU   ││  ││  MMU   ││  ││  MMU   ││  ││  MMU   │││
│  │└────────┘│  │└────────┘│  │└────────┘│  │└────────┘││
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘│
│  ┌──────────────────────────────────────────────────────┐│
│  │            Shared L3 Cache (LLC)                     ││
│  └──────────────────────────────────────────────────────┘│
│  ┌──────────────────────────────────────────────────────┐│
│  │          Memory Controller (IMC)                     ││
│  └─────────────────────┬────────────────────────────────┘│
└────────────────────────┼─────────────────────────────────┘
                         │ Memory Bus (DDR4/DDR5)
              ┌──────────┴──────────┐
              │    DRAM DIMMs       │
              │   (Main Memory)     │
              └─────────────────────┘
```

### Key Components

| Component | Function | Per-Core? |
|-----------|----------|-----------|
| ALU/FPU | Computation | Yes |
| Register File | Fastest storage | Yes |
| L1 Cache | Fastest cache (split I/D) | Yes |
| L2 Cache | Larger, unified cache | Yes (or shared) |
| L3 Cache (LLC) | Last-level cache | Shared |
| TLB | Address translation cache | Yes |
| MMU | Address translation hardware | Yes |
| Page Table Walker | Hardware page table traversal | Yes |
| Memory Controller | DRAM interface | Shared |

---

## 3.2 Memory Management Unit (MMU)

The MMU is the core hardware component that translates virtual addresses to physical addresses.

### What the MMU Does

1. **Address Translation**: Virtual → Physical address mapping
2. **Permission Checking**: Read/Write/Execute permissions per page
3. **Access Control**: User/Supervisor mode checking
4. **Cache Control**: Memory type attributes (cacheable, write-through, etc.)

### MMU Translation Flow

```
                    Virtual Address (from CPU)
                           │
                           ▼
                    ┌──────────────┐
                    │  TLB Lookup  │
                    └──────┬───────┘
                           │
                    ┌──────┴──────┐
                    │             │
                 TLB Hit      TLB Miss
                    │             │
                    │      ┌──────┴──────┐
                    │      │  Page Table  │
                    │      │   Walker     │
                    │      └──────┬───────┘
                    │             │
                    │      ┌──────┴──────┐
                    │      │  Page Table  │
                    │      │  in Memory   │
                    │      └──────┬───────┘
                    │             │
                    │      ┌──────┴──────┐
                    │      │  PTE Found? │
                    │      └──────┬───────┘
                    │         │        │
                    │       Yes       No
                    │         │        │
                    │    Fill TLB   Page Fault
                    │         │     (→ Kernel)
                    │         │
                    ▼         ▼
              Physical Address
                    │
                    ▼
              ┌──────────────┐
              │   L1 Cache   │
              └──────────────┘
```

### x86_64 MMU Configuration

```c
/* CR3 register points to the Page Map Level 4 (PML4) table */
/* Linux loads CR3 during context switch */

/* arch/x86/include/asm/mmu_context.h */
static inline void load_cr3(pgd_t *pgdir)
{
    write_cr3(__sme_pa(pgdir));
}

/* Key x86 Control Registers for Memory Management */
/* CR0: Contains PG (paging enable) bit */
/* CR2: Contains the faulting virtual address on page fault */
/* CR3: Contains the physical address of the top-level page table */
/* CR4: Contains PSE (page size extension), PAE, LA57 bits */
```

### ARM64 MMU Configuration

```c
/* ARM64 uses TTBR0_EL1 (user space) and TTBR1_EL1 (kernel space) */
/* Two separate page table trees — elegant split */

/* TTBR0_EL1: Translation Table Base Register 0 — user space */
/* TTBR1_EL1: Translation Table Base Register 1 — kernel space */
/* TCR_EL1: Translation Control Register — configures page sizes, levels */

/*
 * ARM64 virtual address split:
 * 0x0000_0000_0000_0000 - 0x0000_FFFF_FFFF_FFFF → TTBR0 (user)
 * 0xFFFF_0000_0000_0000 - 0xFFFF_FFFF_FFFF_FFFF → TTBR1 (kernel)
 */
```

---

## 3.3 Translation Lookaside Buffer (TLB)

The TLB is a specialized cache that stores recent virtual-to-physical address translations, avoiding expensive page table walks.

### TLB Structure

```
TLB Entry:
┌──────────────┬────────────────┬──────────┬────────────┐
│  Virtual Page│  Physical Frame│  Flags   │   ASID     │
│  Number (VPN)│  Number (PFN)  │  (RWXUG) │(Addr Space)│
├──────────────┼────────────────┼──────────┼────────────┤
│  0x7FFFF     │  0x12345       │  RW-U    │  42        │
│  0x00400     │  0x00789       │  R-XU    │  42        │
│  0xFFFFF800  │  0xABCDE       │  RW-S    │  0 (global)│
└──────────────┴────────────────┴──────────┴────────────┘
```

### TLB Hierarchy

Modern CPUs have multiple TLB levels:

| TLB Level | Entries | Latency | Associativity |
|-----------|---------|---------|---------------|
| L1 ITLB (instructions) | 64-128 | 1 cycle | 4-8 way |
| L1 DTLB (data) | 64-128 | 1 cycle | 4-8 way |
| L2 STLB (shared/unified) | 1024-2048 | 7-8 cycles | 8-12 way |

### TLB Miss Cost

```
TLB Hit:    ~1 cycle (included in cache access)
L2 TLB Hit: ~7-8 cycles
TLB Miss (4-level page walk):
  └── Level 4 (PML4): memory access (~100 cycles if not cached)
  └── Level 3 (PDPT): memory access (~100 cycles if not cached)
  └── Level 2 (PD):   memory access (~100 cycles if not cached)
  └── Level 1 (PT):   memory access (~100 cycles if not cached)
  Total worst case: ~400-800 cycles (mitigated by page walk caches)
```

### ASID (Address Space Identifier)

Without ASID, the entire TLB must be flushed on every context switch. ASIDs tag TLB entries with a process identifier, allowing entries from multiple processes to coexist.

```c
/* ARM64: ASID is stored in TTBR0_EL1 (upper bits) */
/* x86_64: PCID (Process Context Identifier) — equivalent of ASID */

/* Linux x86_64 PCID support (since kernel 4.14) */
/* arch/x86/mm/tlb.c */
#define TLB_NR_DYN_ASIDS    6  /* Number of dynamic ASIDs */

/* On context switch, Linux assigns a PCID to avoid full TLB flush */
```

### TLB Shootdown

When a page table entry is modified (e.g., page unmapped), all CPUs that might have the old translation cached must be notified to invalidate their TLB entries. This is called a **TLB shootdown**.

```
CPU 0 unmaps a page:
┌───────┐  IPI (Inter-Processor Interrupt)  ┌───────┐
│ CPU 0 │ ──────────────────────────────────→│ CPU 1 │
│ flush │                                    │ flush │
│ local │                                    │ TLB   │
│ TLB   │ ──────────────────────────────────→│ entry │
└───────┘                                    └───────┘
            ──────────────────────────────────→┌───────┐
                                               │ CPU 2 │
                                               │ flush │
                                               └───────┘
```

```c
/* Linux TLB shootdown implementation */
/* arch/x86/mm/tlb.c */

void flush_tlb_mm_range(struct mm_struct *mm,
                        unsigned long start,
                        unsigned long end,
                        unsigned int stride_shift,
                        bool freed_tables)
{
    /* Send IPI to all CPUs running this mm */
    /* Each CPU invalidates the specified TLB range */
}

/* Individual page invalidation (x86) */
static inline void __invlpg(unsigned long addr)
{
    asm volatile("invlpg (%0)" ::"r" (addr) : "memory");
}
```

---

## 3.4 Page Table Walker

The Page Table Walker (PTW) is hardware that automatically traverses multi-level page tables on a TLB miss.

### Hardware vs Software Page Table Walk

| Architecture | Page Table Walk | Notes |
|-------------|-----------------|-------|
| x86/x86_64 | **Hardware** | CR3 points to PML4/PML5 |
| ARM64 | **Hardware** | TTBR0/TTBR1 point to tables |
| MIPS | **Software** | TLB miss causes exception, OS walks |
| RISC-V | **Hardware** (SV39/48/57) | satp register points to table |
| SPARC | **Software** (traditionally) | TSB (Translation Storage Buffer) |

### x86_64 Hardware Page Walk (4-Level)

```
Virtual Address (48-bit): 
┌────────┬────────┬────────┬────────┬──────────────┐
│ PML4   │  PDPT  │   PD   │   PT   │   Offset     │
│ [47:39]│ [38:30]│ [29:21]│ [20:12]│   [11:0]     │
│ 9 bits │ 9 bits │ 9 bits │ 9 bits │  12 bits     │
└────┬───┴────┬───┴────┬───┴────┬───┴──────┬───────┘
     │        │        │        │          │
     │   CR3 ─┘        │        │          │
     │    │             │        │          │
     ▼    ▼             ▼        ▼          │
┌─────────┐     ┌─────────┐  ┌─────────┐  │
│  PML4   │────→│  PDPT   │─→│   PD    │  │
│  Entry  │     │  Entry  │  │  Entry  │  │
└─────────┘     └─────────┘  └────┬────┘  │
                                   │       │
                              ┌────▼────┐  │
                              │   PT    │  │
                              │  Entry  │  │
                              └────┬────┘  │
                                   │       │
                              Physical Frame + Offset
                              = Physical Address
```

### Page Table Walk in Assembly (Conceptual)

```asm
; x86_64 hardware page table walk (what the PTW does internally)
; Given virtual address in RAX

; Step 1: Extract PML4 index (bits 47:39)
mov rbx, rax
shr rbx, 39
and rbx, 0x1FF          ; 9-bit index

; Step 2: Read PML4 entry
mov rcx, cr3             ; PML4 base physical address
mov rdx, [rcx + rbx*8]  ; Read PML4 entry (8 bytes each)
; Check Present bit, extract next-level address

; Step 3: Extract PDPT index (bits 38:30)
mov rbx, rax
shr rbx, 30
and rbx, 0x1FF

; Step 4: Read PDPT entry
and rdx, ~0xFFF         ; Mask to get PDPT base address
mov rdx, [rdx + rbx*8]  ; Read PDPT entry

; ... continue for PD and PT levels ...
; Final: combine physical frame from PT entry with page offset (bits 11:0)
```

---

## 3.5 Cache Hierarchy (L1, L2, L3)

### Cache Organization

```
Cache Line (typical: 64 bytes):
┌─────────────────────────────────────────────────────────┐
│  Tag  │  Index  │  Offset  │          Data (64B)        │
└─────────────────────────────────────────────────────────┘

Cache Set (4-way associative):
Set N: ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
       │ Way 0  │ │ Way 1  │ │ Way 2  │ │ Way 3  │
       │Tag|Data│ │Tag|Data│ │Tag|Data│ │Tag|Data│
       └────────┘ └────────┘ └────────┘ └────────┘
```

### Cache Addressing

For a physically-indexed, physically-tagged (PIPT) cache:

```
Physical Address: 0x0000_1234_5678_9ABC

┌──────────────────┬──────────┬────────┐
│     Tag          │  Index   │ Offset │
│  (identifies     │ (selects │ (byte  │
│   cache line)    │  set)    │ within │
│                  │          │ line)  │
└──────────────────┴──────────┴────────┘

For 32KB, 8-way, 64B line cache:
- Offset: 6 bits (64 bytes per line)
- Index: 6 bits (64 sets = 32KB / 8 ways / 64B)
- Tag: remaining bits
```

### Cache Types by Addressing

| Type | Index | Tag | Used In | Notes |
|------|-------|-----|---------|-------|
| PIPT | Physical | Physical | L2, L3 | No aliases, slow (needs translation first) |
| VIPT | Virtual | Physical | L1 | Fast, possible aliasing |
| VIVT | Virtual | Virtual | Rare (old ARM) | Fast, aliasing + coherence issues |

### Linux Kernel Cache Management

```c
/* Flushing caches — architecture-specific */
/* arch/x86/include/asm/cacheflush.h */

/* Flush a range of virtual addresses from cache */
void clflush_cache_range(void *vaddr, unsigned int size);

/* Write-back and invalidate all caches */
void wbinvd(void);  /* x86: Write-Back and Invalidate */

/* ARM64 cache maintenance */
/* arch/arm64/include/asm/cacheflush.h */
void __flush_dcache_area(void *addr, size_t len);
void invalidate_icache_range(unsigned long start, unsigned long end);

/* DMA-related cache operations */
/* Ensure device sees updated data */
dma_sync_single_for_device(dev, dma_addr, size, direction);
/* Ensure CPU sees device-written data */
dma_sync_single_for_cpu(dev, dma_addr, size, direction);
```

---

## 3.6 Cache Coherence Protocols

In multi-core systems, each core has private caches. Cache coherence ensures all cores see a consistent view of memory.

### MESI Protocol (Used in x86)

```
           ┌──────────┐
     ┌────→│ Modified │←────┐
     │     │   (M)    │     │
     │     └────┬─────┘     │
     │          │           │
     │     ┌────▼─────┐    │
     ├────→│ Exclusive│────┤
     │     │   (E)    │    │
     │     └────┬─────┘    │
     │          │           │
     │     ┌────▼─────┐    │
     ├────→│  Shared  │────┤
     │     │   (S)    │    │
     │     └────┬─────┘    │
     │          │           │
     │     ┌────▼─────┐    │
     └─────│ Invalid  │────┘
           │   (I)    │
           └──────────┘

States:
M (Modified):  Only this cache has the line, it's dirty
E (Exclusive): Only this cache has the line, it's clean
S (Shared):    Multiple caches may have this line, all clean
I (Invalid):   This cache line is not valid
```

### MOESI Protocol (Used in AMD)
Adds **O (Owned)** state: this cache has the dirty copy and is responsible for providing it to other cores, avoiding write-back to memory.

### Cache Coherence Impact on Linux Kernel

```c
/* False sharing: Two variables on the same cache line accessed by different CPUs */

/* BAD: False sharing */
struct bad_shared {
    atomic_t cpu0_counter;  /* Both on same cache line! */
    atomic_t cpu1_counter;  /* Cache line bounces between CPUs */
};

/* GOOD: Pad to separate cache lines */
struct good_separated {
    atomic_t cpu0_counter;
    char __padding[L1_CACHE_BYTES - sizeof(atomic_t)];
    atomic_t cpu1_counter;
} ____cacheline_aligned;

/* Linux per-CPU variables avoid coherence traffic entirely */
DEFINE_PER_CPU(unsigned long, my_counter);

/* Access without locking — each CPU has its own copy */
this_cpu_inc(my_counter);
```

---

## 3.7 Memory Controller

The Integrated Memory Controller (IMC) manages all communication between the CPU and DRAM.

```
┌──────────────────────────────────┐
│         Memory Controller        │
│  ┌────────────────────────────┐  │
│  │  Command Queue             │  │
│  │  (Read/Write/Refresh)      │  │
│  ├────────────────────────────┤  │
│  │  Address Mapper            │  │
│  │  (VA → Row/Bank/Column)    │  │
│  ├────────────────────────────┤  │
│  │  Scheduler (Reorders for   │  │
│  │   bank parallelism)        │  │
│  ├────────────────────────────┤  │
│  │  Refresh Controller        │  │
│  │  (periodic DRAM refresh)   │  │
│  └────────────────────────────┘  │
│       │               │          │
│  ┌────┴────┐     ┌────┴────┐    │
│  │Channel 0│     │Channel 1│    │
│  └────┬────┘     └────┬────┘    │
└───────┼───────────────┼──────────┘
        │               │
   ┌────┴────┐     ┌────┴────┐
   │ DIMM 0  │     │ DIMM 0  │
   │ DIMM 1  │     │ DIMM 1  │
   └─────────┘     └─────────┘
```

### Memory Controller Address Mapping

A physical address is decomposed into:
```
Physical Address → { Channel, DIMM, Rank, Bank, Row, Column }

Example DDR4 mapping:
┌────────┬──────┬──────┬──────┬────────┬────────┬────────┐
│ Unused │ Row  │ Bank │ Rank │ Column │ Channel│ Offset │
│        │      │ Group│      │        │  sel   │(within │
│        │      │+Bank │      │        │        │ burst) │
└────────┴──────┴──────┴──────┴────────┴────────┴────────┘
```

---

## 3.8 DRAM Architecture

### DRAM Cell and Array

```
Single DRAM Cell:
      Word Line (Row select)
          │
    ┌─────┴─────┐
    │  Access    │
    │ Transistor │
    └─────┬─────┘
          │
    ┌─────┴─────┐
    │ Capacitor  │ ← Stores 1 bit (charged = 1, discharged = 0)
    └─────┬─────┘
          │
     Bit Line (Column select / sense)

DRAM Bank Structure:
         Bit Line 0   Bit Line 1   Bit Line 2
              │            │            │
Row 0  ──────┼────────────┼────────────┼──── Word Line 0
              │            │            │
Row 1  ──────┼────────────┼────────────┼──── Word Line 1
              │            │            │
Row 2  ──────┼────────────┼────────────┼──── Word Line 2
              │            │            │

      Sense Amplifiers (one per bit line)
```

### DRAM Access Timing

```
DRAM Read Operation:
1. Row Activate (RAS): Open a row → data loaded into row buffer (~13ns tRCD)
2. Column Read (CAS): Select column from row buffer (~13ns tCL)  
3. Precharge: Close the row (~13ns tRP)

DDR5-4800 Typical Timings:
tCL  = 40 cycles (CAS Latency)
tRCD = 40 cycles (RAS to CAS Delay)
tRP  = 40 cycles (Row Precharge) 
tRAS = 76 cycles (Row Active Time)

Effective latency ≈ tRCD + tCL ≈ 80 cycles ≈ ~16.7ns at 4800MT/s
```

### DDR Generations

| Generation | Year | Speed | Voltage | Bandwidth |
|-----------|------|-------|---------|-----------|
| DDR | 2000 | 200-400 MT/s | 2.5V | 1.6-3.2 GB/s |
| DDR2 | 2003 | 400-1066 MT/s | 1.8V | 3.2-8.5 GB/s |
| DDR3 | 2007 | 800-2133 MT/s | 1.5V | 6.4-17 GB/s |
| DDR4 | 2014 | 1600-3200 MT/s | 1.2V | 12.8-25.6 GB/s |
| DDR5 | 2020 | 3200-8400 MT/s | 1.1V | 25.6-67.2 GB/s |

---

## 3.9 Memory Buses and Interconnects

### System Interconnect Architecture

```
Modern Server (2-socket):

┌──────────────────────┐          ┌──────────────────────┐
│      CPU Socket 0     │  QPI/UPI │      CPU Socket 1     │
│  ┌────────────────┐  │◄────────►│  ┌────────────────┐  │
│  │   Cores 0-15   │  │          │  │   Cores 16-31  │  │
│  └───────┬────────┘  │          │  └───────┬────────┘  │
│  ┌───────┴────────┐  │          │  ┌───────┴────────┐  │
│  │   L3 Cache     │  │          │  │   L3 Cache     │  │
│  └───────┬────────┘  │          │  └───────┬────────┘  │
│  ┌───────┴────────┐  │          │  ┌───────┴────────┐  │
│  │ Memory Ctrl    │  │          │  │ Memory Ctrl    │  │
│  └───────┬────────┘  │          │  └───────┬────────┘  │
│          │            │          │          │            │
│    ┌─────┴─────┐      │          │    ┌─────┴─────┐      │
│    │  DDR5     │      │          │    │  DDR5     │      │
│    │  DIMMs    │      │          │    │  DIMMs    │      │
│    └───────────┘      │          │    └───────────┘      │
│   Local Memory        │          │   Local Memory        │
│   (Node 0)            │          │   (Node 1)            │
└──────────────────────┘          └──────────────────────┘
```

### Interconnect Technologies

| Technology | Vendor | Bandwidth | Use |
|-----------|--------|-----------|-----|
| QPI (QuickPath) | Intel | 12.8 GT/s | Pre-Skylake servers |
| UPI (Ultra Path) | Intel | 10.4-16 GT/s | Skylake+ servers |
| Infinity Fabric | AMD | 32 GT/s | EPYC processors |
| CXL | Industry std | 32-64 GT/s | Memory expansion |
| NVLink | NVIDIA | 600 GB/s | GPU interconnect |

---

## 3.10 NUMA Architecture

### NUMA (Non-Uniform Memory Access)

In NUMA systems, memory access time depends on the memory location relative to the processor.

```
NUMA Topology:

    Node 0                              Node 1
┌────────────────┐                ┌────────────────┐
│   CPU 0-7      │   Interconnect │   CPU 8-15     │
│   L3 Cache     │◄──────────────►│   L3 Cache     │
│   Memory Ctrl  │    (~100ns     │   Memory Ctrl  │
│       │        │     remote)    │       │        │
│  ┌────┴─────┐  │                │  ┌────┴─────┐  │
│  │ Local    │  │                │  │ Local    │  │
│  │ Memory   │  │                │  │ Memory   │  │
│  │ (~80ns)  │  │                │  │ (~80ns)  │  │
│  └──────────┘  │                │  └──────────┘  │
└────────────────┘                └────────────────┘

Access latencies:
  Local memory:  ~80ns
  Remote memory: ~130ns (1.6x slower!)
```

### Linux NUMA Detection

```bash
# View NUMA topology
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7
node 0 size: 32768 MB
node 0 free: 28444 MB
node 1 cpus: 8 9 10 11 12 13 14 15
node 1 size: 32768 MB
node 1 free: 29012 MB
node distances:
node   0   1
  0:  10  21
  1:  21  10

# View per-node memory info
$ cat /sys/devices/system/node/node0/meminfo
```

```c
/* Linux kernel NUMA structures */
/* include/linux/mmzone.h */

typedef struct pglist_data {
    struct zone node_zones[MAX_NR_ZONES];
    int node_id;
    unsigned long node_start_pfn;
    unsigned long node_spanned_pages;
    unsigned long node_present_pages;
    /* ... */
} pg_data_t;

/* NUMA allocation policy */
/* include/linux/mempolicy.h */
#define MPOL_DEFAULT    0    /* Use process/system default */
#define MPOL_PREFERRED  1    /* Prefer specified node */
#define MPOL_BIND       2    /* Only allocate from specified nodes */
#define MPOL_INTERLEAVE 3    /* Round-robin across nodes */
#define MPOL_LOCAL      4    /* Allocate from local node */
```

---

## 3.11 Hardware Memory Protection Mechanisms

### Page-Level Protection Bits (x86_64 PTE)

```
x86_64 Page Table Entry (64 bits):
┌────┬───┬───┬───┬───┬───┬───┬───┬───┬───┬──────────────────┬───┐
│ NX │   │   │   │   │ G │PAT│ D │ A │PCD│PWT│U/S│R/W│ P │
│ 63 │   │   │   │   │ 8 │ 7 │ 6 │ 5 │ 4 │ 3 │ 2 │ 1 │ 0 │
└────┴───┴───┴───┴───┴───┴───┴───┴───┴───┴──────────────────┴───┘

Bits 51:12 = Physical Frame Number

P   (0):  Present — page is in memory
R/W (1):  Read/Write — 0=read-only, 1=read-write
U/S (2):  User/Supervisor — 0=kernel only, 1=user accessible
PWT (3):  Page Write-Through
PCD (4):  Page Cache Disable
A   (5):  Accessed — set by hardware on access
D   (6):  Dirty — set by hardware on write
PAT (7):  Page Attribute Table
G   (8):  Global — not flushed on CR3 load
NX  (63): No Execute — prevents code execution
```

### Supervisor Mode Execution/Access Prevention

```c
/* SMEP (Supervisor Mode Execution Prevention) — CR4.SMEP */
/* Prevents kernel from executing code in user pages */
/* Defeats ret2usr attacks */

/* SMAP (Supervisor Mode Access Prevention) — CR4.SMAP */
/* Prevents kernel from reading/writing user pages unless explicitly allowed */
/* Kernel must use copy_from_user()/copy_to_user() */

/* Linux enables SMEP/SMAP on supported hardware */
/* arch/x86/kernel/cpu/common.c */
```

### ARM64 Memory Protection

```
ARM64 uses:
- PXN (Privileged Execute Never): Like NX for kernel
- UXN (Unprivileged Execute Never): Like NX for user
- AP[2:1] (Access Permission): Read/Write control
- nG (not Global): Per-ASID vs global mapping

ARM64 PTE:
┌─────┬────┬────┬────┬─────┬──────┬─────────┬────┐
│ UXN │PXN │Cont│ nG │ AF  │ SH   │ AP[2:1] │Type│
└─────┴────┴────┴────┴─────┴──────┴─────────┴────┘
```

---

## 3.12 Virtualization Hardware Support (EPT, NPT, Stage-2)

### Problem: Virtualizing Address Translation

```
Without hardware support (Shadow Page Tables):
Guest VA → Guest Page Tables → Guest PA → VMM → Host PA
           (maintained by VMM as shadow of guest's tables)
           Very expensive to maintain!

With hardware support (EPT/NPT):
Guest VA → Guest Page Tables → Guest PA → EPT/NPT → Host PA
           (guest manages)     (hardware walks two tables)
           Guest unchanged!     Transparent to guest!
```

### Intel EPT (Extended Page Tables)

```
Two-dimensional page walk:

Guest Virtual Address
        │
        ▼
Guest Page Tables (4 levels)  ──→  Each guest physical address
        │                            in the walk requires EPT
        ▼                            translation
Guest Physical Address
        │
        ▼
EPT Page Tables (4 levels)
        │
        ▼
Host Physical Address

Worst case: 4 guest levels × 4 EPT levels = up to 24 memory accesses!
(Mitigated by TLB caching of final GVA→HPA translations)
```

### ARM64 Stage-2 Translation

```
ARM64 Virtualization:
- Stage-1: VA → IPA (Intermediate Physical Address) — guest controlled
- Stage-2: IPA → PA — hypervisor controlled (via VTTBR_EL2)

EL0 (User) ──→ Stage 1 (EL1 tables) ──→ IPA ──→ Stage 2 (EL2 tables) ──→ PA
```

```c
/* KVM (Kernel-based Virtual Machine) EPT/Stage-2 management */
/* arch/x86/kvm/mmu/ for x86 EPT */
/* arch/arm64/kvm/hyp/pgtable.c for ARM64 Stage-2 */
```

---

## 3.13 Hardware Page Fault Mechanism

When the MMU cannot translate an address, it generates a **page fault exception**.

### x86_64 Page Fault Mechanism

```
1. CPU attempts memory access with virtual address
2. MMU walks page tables
3. Translation fails (page not present, permission violation, etc.)
4. CPU pushes error code and faulting address:
   - Saves faulting VA in CR2 register
   - Pushes error code on stack
   - Transfers control to IDT entry 14 (Page Fault)

Page Fault Error Code (x86_64):
Bit 0 (P):    0 = non-present page, 1 = protection violation
Bit 1 (W/R):  0 = read access, 1 = write access
Bit 2 (U/S):  0 = kernel mode, 1 = user mode
Bit 3 (RSVD): 1 = reserved bit set in page table
Bit 4 (I/D):  1 = instruction fetch (NX violation)
Bit 5 (PK):   1 = protection key violation
Bit 6 (SS):   1 = shadow stack violation
```

```c
/* Linux page fault handler entry point */
/* arch/x86/mm/fault.c */

DEFINE_IDTENTRY_RAW_ERRORCODE(exc_page_fault)
{
    unsigned long address = read_cr2();  /* Get faulting address */
    /* ... handle the fault ... */
    handle_page_fault(regs, error_code, address);
}
```

### ARM64 Page Fault Mechanism

```
ARM64 generates different exceptions:
- Data Abort (EL1/EL0): Data access fault
- Instruction Abort (EL1/EL0): Instruction fetch fault
- ESR_EL1: Exception Syndrome Register contains fault type

Fault Status Code (FSC) in ESR_EL1:
0b000100: Translation fault, level 0
0b000101: Translation fault, level 1
0b000110: Translation fault, level 2
0b000111: Translation fault, level 3
0b001001: Access flag fault, level 1
0b001101: Permission fault, level 1
```

---

## Comparison: Hardware Memory Support Across Architectures

| Feature | x86_64 | ARM64 | RISC-V | MIPS |
|---------|--------|-------|--------|------|
| MMU | Hardware | Hardware | Hardware | Software TLB refill |
| Page table levels | 4 (5 with LA57) | 3-4 (configurable) | 3-5 (Sv39/48/57) | Software-defined |
| Page sizes | 4K, 2M, 1G | 4K, 16K, 64K base + sections | 4K, 2M, 1G | 4K, variable |
| TLB management | Hardware refill | Hardware refill | Hardware refill | Software refill |
| ASID/PCID | PCID (12-bit) | ASID (8/16-bit) | ASID (16-bit) | ASID (8-bit) |
| NX bit | Yes (bit 63) | UXN/PXN | Yes | Yes |
| Virtualization | EPT (VT-x) | Stage-2 | H-extension | VZ extension |
| Cache coherence | MESI/MESIF | MOESI/ACE | Implementation-defined | Implementation-defined |

---

## Interview Questions

1. **Q: What is the MMU and what does it do?**
   A: The MMU (Memory Management Unit) translates virtual addresses to physical addresses using page tables, enforces memory protection (R/W/X permissions, user/kernel separation), and works with the TLB for fast translations.

2. **Q: What happens on a TLB miss?**
   A: On a TLB miss, the hardware Page Table Walker traverses the multi-level page table in memory. If a valid mapping is found, it fills the TLB. If not (page not present), a page fault exception is raised.

3. **Q: Explain cache coherence and the MESI protocol.**
   A: In multi-core systems, each core has private caches. MESI ensures consistency: Modified (dirty, exclusive), Exclusive (clean, exclusive), Shared (clean, multiple caches), Invalid (stale). State transitions occur on reads/writes via the coherence bus.

4. **Q: What is a TLB shootdown and why is it expensive?**
   A: When a page table entry changes, all CPUs that might cache the old translation must invalidate it. This requires Inter-Processor Interrupts (IPIs), which stall the target CPUs. On large NUMA systems with hundreds of CPUs, this can be very expensive.

5. **Q: What is the difference between EPT (Intel) and ARM64 Stage-2 translation?**
   A: Both provide hardware-assisted nested address translation for virtualization. EPT uses a separate 4-level table walked by hardware on each guest physical address. ARM64 Stage-2 uses VTTBR_EL2 and supports different page sizes. Both avoid the overhead of shadow page tables.

---

## Summary & Key Takeaways

1. The MMU is the cornerstone hardware component for virtual memory, translating addresses and enforcing protection.
2. The TLB caches translations for speed; TLB misses trigger hardware page walks that are 10-100x slower.
3. Multi-level caches (L1/L2/L3) with coherence protocols (MESI/MOESI) keep multi-core systems consistent.
4. NUMA architecture means memory access time is non-uniform — Linux must be NUMA-aware for performance.
5. Hardware page faults trap to the OS kernel, which handles them by mapping pages, allocating memory, or killing processes.
6. Virtualization extensions (EPT, Stage-2) add a second layer of address translation in hardware.
7. Modern hardware provides security features (NX, SMEP, SMAP) that Linux leverages for protection.

---

*Previous: [Chapter 2 — History of Memory Management](Chapter_02_History_of_Memory_Management.md)*
*Next: [Chapter 4 — Virtual Memory Concepts](Chapter_04_Virtual_Memory_Concepts.md)*
