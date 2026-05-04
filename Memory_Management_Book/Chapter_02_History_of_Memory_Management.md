# Chapter 2: History of Memory Management

## Chapter Overview

Understanding the history of memory management reveals *why* modern systems work the way they do. Every mechanism in the Linux kernel — from paging to NUMA-aware allocation — was born from solving real problems that earlier systems couldn't handle. This chapter traces the evolution from bare-metal programming with no OS to the sophisticated memory management subsystem in Linux 6.x.

---

## 2.1 Early Computer Memory Management (No OS)

In the earliest computers (1940s-1950s), there was **no operating system** and therefore **no memory management**.

### How It Worked
- Programs were loaded into memory manually (via paper tape, punch cards)
- The program had full access to all physical memory
- The programmer was responsible for managing every byte
- No protection — a bug could overwrite anything

```
Early Computer Memory (e.g., ENIAC, UNIVAC):
┌─────────────────────────────────┐
│         Physical Memory         │
│  ┌───────────────────────────┐  │
│  │   Your Program            │  │
│  │   (code + data + stack)   │  │
│  │                           │  │
│  │   You manage everything   │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
     One program, full control
```

### Problems
- Only one program could run at a time
- No memory protection
- Program size limited to physical memory
- No way to share memory between programs

---

## 2.2 Single Program Memory Model

As computers got faster, **batch processing** emerged. Still one program at a time, but an early "monitor" program managed loading and execution.

```
┌──────────────────────────────┐
│  Resident Monitor (OS)       │  ← Fixed location in memory
├──────────────────────────────┤
│                              │
│  User Program Area           │  ← Single user program
│                              │
├──────────────────────────────┤
│  Unused Memory               │
└──────────────────────────────┘
```

### The Monitor's Role
- Load program from card reader/tape
- Transfer control to program
- When program finishes, load next program
- **No memory protection** — user program could crash the monitor

### Real Example: IBM 7094 (1962)
- IBSYS monitor program occupied first 8KB of memory
- User programs loaded after the monitor
- If a user program wrote to the first 8KB, the system crashed

---

## 2.3 Fixed Partition Memory Systems

With **multiprogramming** (running multiple programs simultaneously), the OS needed to divide memory among programs.

### Fixed Partitions
Memory divided into fixed-size regions at boot time. Each partition holds one process.

```
┌─────────────────────────────┐  0KB
│  Operating System           │
├─────────────────────────────┤  100KB
│  Partition 1 (100KB)        │  ← Process A
├─────────────────────────────┤  200KB
│  Partition 2 (200KB)        │  ← Process B
├─────────────────────────────┤  400KB
│  Partition 3 (300KB)        │  ← Process C
├─────────────────────────────┤  700KB
│  Partition 4 (300KB)        │  ← Empty (waiting)
└─────────────────────────────┘  1000KB
```

### Problems
1. **Internal Fragmentation**: If a 50KB process gets a 200KB partition, 150KB is wasted
2. **Fixed limit**: Can't run more processes than partitions
3. **Size mismatch**: Large programs might not fit in any partition

### Historical Example: IBM OS/360 MFT (Multiprogramming with a Fixed number of Tasks)
- Up to 15 fixed partitions
- Partition sizes set at system generation time
- Couldn't be changed without reboot

---

## 2.4 Dynamic Partition Memory Systems

Dynamic partitions allocate exactly the memory each process needs.

```
Initial State:        After allocating A(100KB), B(200KB), C(150KB):
┌──────────┐          ┌──────────┐
│    OS    │          │    OS    │
├──────────┤          ├──────────┤
│          │          │  A: 100K │
│  Free    │   →      ├──────────┤
│  900KB   │          │  B: 200K │
│          │          ├──────────┤
│          │          │  C: 150K │
│          │          ├──────────┤
│          │          │ Free:450K│
└──────────┘          └──────────┘

After B exits:                  After D(180KB) allocated:
┌──────────┐                    ┌──────────┐
│    OS    │                    │    OS    │
├──────────┤                    ├──────────┤
│  A: 100K │                    │  A: 100K │
├──────────┤                    ├──────────┤
│  HOLE    │  ← External        │  D: 180K │  
│  200K    │    Fragmentation!  ├──────────┤
├──────────┤                    │  HOLE 20K│ ← Small fragment!
│  C: 150K │                    ├──────────┤
├──────────┤                    │  C: 150K │
│ Free:450K│                    ├──────────┤
└──────────┘                    │ Free:450K│
                                └──────────┘
```

### Allocation Strategies

| Strategy | Description | Pros | Cons |
|----------|-------------|------|------|
| First Fit | Use first hole that fits | Fast | Fragments front of memory |
| Best Fit | Use smallest hole that fits | Less waste | Slow, creates tiny fragments |
| Worst Fit | Use largest hole | Avoids tiny fragments | Wastes large blocks |
| Next Fit | Start search from last allocation | Better distribution | May miss better fits |

### The Compaction Problem
When external fragmentation gets bad, the OS must **compact** memory — sliding all processes together. This is expensive (copying large amounts of memory) and requires all address references to be relocatable.

### Historical Example: IBM OS/360 MVT (Multiprogramming with a Variable number of Tasks)

---

## 2.5 Segmentation Model

Segmentation was an early attempt to give programs a logical view of memory divided into meaningful segments.

```
Program View:                    Physical Memory:
┌──────────────┐                ┌───────────────────┐
│ Code Segment  │──────────────→│ Code at 0x4000    │
├──────────────┤                ├───────────────────┤
│ Data Segment  │──────────────→│ Data at 0x8000    │
├──────────────┤                ├───────────────────┤
│ Stack Segment │──────────────→│ Stack at 0xC000   │
└──────────────┘                └───────────────────┘

Segment Table:
┌────────┬──────────┬────────┬───────────────┐
│ Segment│  Base    │ Limit  │  Permissions  │
├────────┼──────────┼────────┼───────────────┤
│   0    │  0x4000  │  4KB   │  Execute/Read │
│   1    │  0x8000  │  8KB   │  Read/Write   │
│   2    │  0xC000  │  4KB   │  Read/Write   │
└────────┴──────────┴────────┴───────────────┘

Address = Segment:Offset → Physical = Base + Offset (if Offset < Limit)
```

### Intel x86 Segmentation Legacy

The x86 architecture used segmentation heavily:
- **8086** (1978): 16-bit with segment:offset addressing (20-bit physical = 1MB)
- **80286** (1982): Protected mode with segment descriptors
- **80386+** (1985): Segments + paging (segments became mostly vestigial)
- **x86_64** (2003): Segmentation effectively disabled (flat model)

```c
/* x86 Segment Registers */
CS  /* Code Segment */
DS  /* Data Segment */
SS  /* Stack Segment */
ES  /* Extra Segment */
FS  /* Used by Linux for thread-local storage (TLS) */
GS  /* Used by Linux kernel for per-CPU data */
```

### Linux and Segmentation
Linux uses a **flat memory model** — all segments have base=0 and limit=max. The kernel effectively bypasses segmentation and relies entirely on paging. However, `FS` and `GS` segments are still used:
- `FS`: Points to thread-local storage in user space
- `GS`: Used by kernel for per-CPU data access

---

## 2.6 Paging Model Introduction

Paging solved the fragmentation problem by dividing memory into fixed-size units.

### Key Innovation
Instead of allocating contiguous physical memory, paging maps **virtual pages** to **physical frames** that can be anywhere in RAM.

```
Virtual Memory (Process View):        Physical Memory (RAM):
┌──────────┐ Page 0                    ┌──────────┐ Frame 0
│ Code     │────────────┐              │ (Other)  │
├──────────┤ Page 1     │              ├──────────┤ Frame 1
│ Data     │──────┐     │              │ (Other)  │
├──────────┤ Page 2│    │              ├──────────┤ Frame 2
│ Heap     │──┐   │    └─────────────→│ Code     │
├──────────┤  │   │                    ├──────────┤ Frame 3
│          │  │   └──────────────────→│ Data     │
│  Stack   │  │                        ├──────────┤ Frame 4
├──────────┤  └──────────────────────→│ Heap     │
│ (unused) │                           ├──────────┤ Frame 5
└──────────┘                           │ Stack    │
                                       └──────────┘
     Pages need NOT map to              Frames can be in
     contiguous frames!                 any order!
```

### Why Paging Won

| Feature | Segmentation | Paging |
|---------|-------------|--------|
| External fragmentation | Yes | **No** |
| Internal fragmentation | No | Yes (last page) |
| Variable allocation size | Yes | Fixed page size |
| Memory compaction needed | Yes | **No** |
| Hardware support simplicity | Complex | **Simpler** |
| Swapping granularity | Segment (variable) | Page (fixed) |

### The Original Paging System: Atlas Computer (1962)
The Atlas computer at the University of Manchester implemented the first virtual memory system with **demand paging**. It had 16K words of core memory and 96K words of drum storage, giving programs the illusion of 96K words of memory.

---

## 2.7 Virtual Memory Evolution

### The Concept
Virtual memory allows programs to use more memory than physically available by using disk as an extension of RAM.

```
Virtual Address Space: 4GB          Physical RAM: 1GB
┌─────────────────┐                 ┌─────────────┐
│                 │                 │             │
│  Program sees   │    Only active  │ Active      │
│  4GB of memory  │───→portions ──→│ pages in    │
│                 │    in RAM       │ RAM         │
│                 │                 │             │
└─────────────────┘                 └─────────────┘
        │                                  ↑↓
        │                           ┌─────────────┐
        └──→ Inactive pages ──────→│  Swap on    │
             stored on disk         │  Disk       │
                                    └─────────────┘
```

### Key Virtual Memory Milestones

| Year | System | Innovation |
|------|--------|-----------|
| 1961 | Atlas | First demand paging |
| 1964 | Multics | Segmented virtual memory |
| 1970 | IBM System/370 | Commercial virtual memory |
| 1971 | PDP-11 | Demand paging in minicomputers |
| 1979 | VAX/VMS | Sophisticated VM with working sets |
| 1983 | BSD 4.2 | VM for Unix |
| 1991 | Linux 0.01 | Early Linux VM |
| 2001 | Linux 2.4 | Modern VM subsystem |

---

## 2.8 Memory Management in Unix

Unix pioneered many memory management concepts that Linux inherited.

### Early Unix (PDP-7, 1969)
- **No virtual memory** — processes ran in physical memory
- **Swapping** — entire processes swapped to/from disk
- Very simple memory model

### Unix V6 (PDP-11, 1975)
- Still swapping-based
- No demand paging
- Process memory was contiguous in physical RAM

### BSD Unix (3BSD, 1979)
- First Unix with **demand paging**
- Used the VAX hardware's paging support
- Page replacement using a clock algorithm
- **This was the direct ancestor of Linux's VM**

### System V Release 4 (1988)
- Unified VM/file system (vnode pager)
- Memory-mapped files
- Shared memory

### Key Unix Memory Concepts Inherited by Linux

1. **Process isolation**: Each process has its own address space
2. **Copy-on-write fork()**: BSD introduced COW for fork
3. **mmap()**: Map files into memory (BSD 4.2)
4. **Shared libraries**: Map .so files into multiple processes
5. **Swapping and paging**: Hybrid approach

---

## 2.9 Evolution of Memory Management in Linux Kernel

### Linux 0.01 (1991) — Linus Torvalds' First Release
```c
/* Original mm code was tiny — a few hundred lines */
/* Simple page allocation, limited to 16MB on i386 */
/* No swap, no shared memory, no mmap */
```

### Linux 1.0 (1994)
- Basic swap support
- Simple page allocation
- Limited to i386

### Linux 2.0 (1996)
- SMP (Symmetric Multi-Processing) support
- Better memory management
- Still relatively simple VM

### Linux 2.2 (1999)
- Big kernel lock for memory operations
- Andrea Arcangeli's VM improvements
- Better swap handling

### Linux 2.4 (2001) — Major VM Rewrite
- Rik van Riel's rmap VM
- Reverse mapping for pages
- Much improved page reclaim
- But still had scalability issues under heavy load

### Linux 2.6 (2003) — Modern MM Foundation
```
Major 2.6 Memory Improvements:
├── Object-based reverse mapping (objrmap)
├── NUMA support
├── Slab allocator improvements (SLUB introduced later)
├── Memory cgroups (2.6.25, 2008)
├── Transparent Huge Pages (2.6.38, 2011)
├── CMA - Contiguous Memory Allocator (3.5, 2012)
├── SLUB as default allocator
└── Per-CPU page allocator caches
```

### Linux 3.x-4.x (2011-2019)
- **Compaction**: Proactive memory defragmentation
- **NUMA balancing**: Automatic page migration
- **Memory pressure notifications**: Userspace notification
- **Zswap/Zram**: Compressed swap
- **KSM**: Kernel Samepage Merging (deduplication)
- **5-level page tables** (4.14): Support for 57-bit virtual addresses

### Linux 5.x (2019-2022)
- **Memory tiering**: Support for multiple types of memory (DRAM + PMEM)
- **Maple tree**: New data structure for VMAs (replacing red-black tree)
- **Folios**: New abstraction replacing compound pages (5.16+)
- **DAMOS**: Data Access Monitoring-based Operation Schemes

### Linux 6.x (2022+)
- **MGLRU** (Multi-Gen LRU): Major page reclaim improvement (6.1)
- **Per-VMA locks**: Reduced mmap_lock contention (6.4)
- **Folio completion**: Most MM paths converted to folios
- **Memory tiering improvements**: CXL memory support
- **Large anonymous folios**: Better THP for anonymous memory

```
Linux Memory Management Evolution Timeline:

1991 ─── 1994 ─── 1999 ─── 2001 ─── 2003 ────────── 2011 ─── 2019 ─── 2022 ──→
 0.01     1.0      2.2      2.4      2.6               3.x      5.x      6.x
 │        │        │        │        │                  │        │        │
 Simple   Swap     Better   rmap     NUMA, SLUB,       THP,     Folios,  MGLRU,
 MM       added    VM       VM       cgroups            CMA,     Maple    per-VMA
                                                        KSM      Tree     locks
```

---

## 2.10 Modern Memory Management Trends

### 1. CXL (Compute Express Link) Memory
```
┌──────────┐     CXL Link     ┌────────────────┐
│   CPU    │◄─────────────────►│  CXL Memory    │
│          │                   │  (DRAM/PMEM)   │
└──────────┘                   └────────────────┘
     ↕                         Appears as another
  Local DRAM                   NUMA node to Linux
```

### 2. Memory Tiering
Linux now supports multiple tiers of memory with automatic data movement:
- Tier 0: Local DRAM (fast)
- Tier 1: Remote DRAM / CXL memory (slower)
- Tier 2: Persistent memory (slowest)

### 3. Heterogeneous Memory Management (HMM)
Support for GPU and accelerator memory:
```c
/* HMM allows GPUs to share the CPU's page tables */
/* Unified virtual addressing across CPU and GPU */
hmm_range_fault()  /* Fault pages for device access */
```

### 4. Memory Safety
- Hardware memory tagging (ARM MTE, Intel MPX)
- KASAN, KCSAN in kernel
- Rust in Linux kernel (memory-safe language support from 6.1+)

---

## Comparison: Memory Management History Across OS

| Era | Linux | Windows | macOS | QNX |
|-----|-------|---------|-------|-----|
| Origins | Unix tradition (1991) | DOS → NT (1993) | Mach + BSD (2001) | Microkernel (1982) |
| First VM | Linux 0.01 (basic) | Windows NT 3.1 | Mach VM (1980s) | QNX 4 (1990s) |
| Page size | 4KB (default) | 4KB (default) | 4KB → 16KB (Apple Silicon) | 4KB |
| Huge pages | 2MB/1GB (configurable) | Large pages (2MB) | Superpage (2MB) | Not standard |
| NUMA | Full support (2.6+) | Full support (Vista+) | Limited | Limited |
| Memory dedup | KSM | Superfetch/compression | Memory compression | No |
| Modern innovation | MGLRU, per-VMA locks, folios | Memory compression, DirectStorage | Unified memory (M-series) | Predictable allocation |

---

## Interview Questions

1. **Q: Describe the evolution from fixed partitions to paging.**
   A: Fixed partitions → internal fragmentation. Dynamic partitions → external fragmentation. Segmentation → still external fragmentation. Paging → fixed-size units eliminate external fragmentation.

2. **Q: What is the difference between swapping and demand paging?**
   A: Swapping moves entire processes to/from disk. Demand paging moves individual pages, loading only when accessed (page fault). Modern Linux uses demand paging; swapping is a last resort.

3. **Q: What was the major Linux VM change in 2.6?**
   A: Object-based reverse mapping (objrmap), which improved page reclaim by tracking which VMAs reference each page, rather than maintaining per-page PTE chains.

4. **Q: What are folios in Linux 6.x?**
   A: Folios are a new abstraction replacing struct page for multi-page allocations. They carry a head page and size, simplifying compound page handling and reducing bugs.

5. **Q: What is MGLRU and why was it significant?**
   A: Multi-Gen LRU is a page reclaim algorithm in Linux 6.1+ that tracks page age across multiple generations, providing better hot/cold page differentiation than the traditional active/inactive LRU.

---

## Summary & Key Takeaways

1. Memory management evolved from no OS (bare metal) → fixed partitions → dynamic partitions → segmentation → paging → virtual memory.
2. Each evolution solved the previous system's biggest problem (usually fragmentation or lack of isolation).
3. Unix established the fundamental memory management paradigms that Linux inherited: process isolation, COW fork, mmap, and demand paging.
4. Linux's MM subsystem has evolved dramatically: from a few hundred lines in 0.01 to one of the most sophisticated memory management systems in any OS.
5. Modern trends include CXL memory, memory tiering, heterogeneous memory, and memory-safe programming (Rust in kernel).
6. Key Linux MM milestones: rmap (2.4), NUMA/SLUB (2.6), THP (3.x), folios (5.16), MGLRU (6.1), per-VMA locks (6.4).

---

*Previous: [Chapter 1 — Foundations of Computer Memory](Chapter_01_Foundations_of_Computer_Memory.md)*
*Next: [Chapter 3 — Hardware Architecture for Memory Management](Chapter_03_Hardware_Architecture.md)*
