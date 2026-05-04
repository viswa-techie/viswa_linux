# Chapter 30: Glossary of Memory Management Terms

## Chapter Overview

Comprehensive glossary of all memory management related terms, acronyms, and concepts referenced throughout this book.

---

## A

**Address Space** — The set of valid addresses a process or the kernel can use. User address space is per-process (isolated); kernel address space is shared.

**ASLR (Address Space Layout Randomization)** — Security mechanism that randomizes the base addresses of stack, heap, libraries, and executable at each execution, defeating address-prediction attacks.

**Anonymous Memory** — Memory not backed by a file (heap, stack, mmap MAP_ANONYMOUS). Must be swapped to disk if reclaimed.

**Accessed Bit (A-bit)** — PTE flag set by hardware when a page is read or written. Used by LRU algorithms to estimate recency of access.

**alloc_pages()** — Core kernel function to allocate 2^order contiguous physical pages from the buddy allocator.

**address_space** — Kernel structure representing a mapping between a file (or anonymous object) and its cached pages (page cache).

---

## B

**Buddy Allocator** — Linux's physical page allocator. Manages free pages in orders (0-10, i.e., 1 to 1024 pages). Splits larger blocks on allocation, merges adjacent blocks on free.

**brk()** — System call to adjust the program break (end of heap). Used by malloc for small allocations.

**Bounce Buffer** — Temporary buffer in DMA-accessible memory, used when the target buffer is in high memory that the device cannot reach. See SWIOTLB.

**BSS** — Block Started by Symbol. Uninitialized global/static data segment, zero-filled at load time.

---

## C

**Cache (CPU)** — Fast SRAM memory between CPU and DRAM. L1 (~1ns), L2 (~5ns), L3 (~15ns). Holds copies of recently accessed data.

**Cache Line** — Unit of data transfer between CPU cache and memory. Typically 64 bytes on x86_64/ARM64.

**CMA (Contiguous Memory Allocator)** — Reserves memory at boot for large contiguous allocations (DMA, huge pages) while allowing movable pages to use it when not needed.

**cgroup (Control Group)** — Kernel mechanism to limit, account, and isolate resource usage of process groups. Memory cgroup limits RAM/swap per group.

**COW (Copy-on-Write)** — Optimization where fork() shares pages between parent and child as read-only. Pages are copied only when one process writes (on COW fault).

**Compaction** — Process of migrating movable pages to create larger contiguous free blocks, reducing fragmentation.

**CR3** — x86 register holding the physical address of the current page table (PGD). Changed on context switch.

**Copy-from/to-user** — `copy_from_user()` / `copy_to_user()` — safe kernel functions to transfer data between user and kernel space, with SMAP/PAN handling.

---

## D

**Demand Paging** — Pages are allocated only when first accessed (page fault), not when the virtual mapping is created. Saves memory.

**Dirty Bit (D-bit)** — PTE flag set by hardware when a page is written. Dirty pages must be written back before reclaim.

**Dirty Page** — A page whose contents differ from the on-disk copy. Must be written back (flushed) before the page can be freed.

**Direct Mapping** — Kernel maps all physical RAM at a fixed VA offset (PAGE_OFFSET). `phys_to_virt(pa) = pa + PAGE_OFFSET`.

**Direct Reclaim** — Synchronous page reclaim performed by the allocating process when kswapd cannot free enough memory.

**DMA (Direct Memory Access)** — Hardware mechanism allowing devices to transfer data to/from memory without CPU involvement.

**DMA Zone** — Memory zone for the first 16MB, accessible by legacy ISA DMA devices.

**dma-buf** — Kernel framework for sharing DMA buffers between device drivers without copying.

---

## E

**EPT (Extended Page Tables)** — Intel hardware feature for virtualizing memory. Two-level address translation: guest VA → guest PA (guest page tables) → host PA (EPT).

---

## F

**False Sharing** — Performance pathology where two CPUs modify different variables on the same cache line, causing excessive cache line bouncing.

**Folio** — Kernel abstraction (Linux 5.16+) representing one or more contiguous pages, replacing raw `struct page` for compound page operations.

**Fragmentation** — Internal: wasted space within an allocation. External: free memory exists but not in contiguous blocks large enough for the request.

**Freelist** — Linked list of free objects within a slab (SLUB) or free page blocks within a buddy order.

---

## G

**GFP Flags (Get Free Pages)** — Flags passed to page allocator specifying context constraints: `GFP_KERNEL` (may sleep), `GFP_ATOMIC` (cannot sleep), `__GFP_DMA` (DMA zone), `__GFP_ZERO` (zero-fill).

**Guard Page** — Unmapped page placed adjacent to a region to detect overflow. Used for kernel stacks (VMAP_STACK) and KFENCE.

---

## H

**Huge Page** — Page larger than the default 4KB. 2MB (PMD-level) or 1GB (PUD-level) on x86_64. Reduces TLB pressure.

**hugetlbfs** — Filesystem for explicitly mapping huge pages. Pages must be pre-reserved.

---

## I

**IOMMU (I/O Memory Management Unit)** — Hardware that translates device DMA addresses (IOVA) to physical addresses, providing device isolation and scatter-gather.

**IOVA (I/O Virtual Address)** — Virtual address used by devices for DMA, translated by IOMMU to physical address.

**ioremap()** — Kernel function to map device physical memory (MMIO registers) into kernel virtual address space.

---

## K

**KASLR (Kernel ASLR)** — Randomizes kernel text, modules, and data locations at boot time.

**KASAN (Kernel Address Sanitizer)** — Debug tool using shadow memory to detect out-of-bounds, use-after-free, and other memory errors in the kernel.

**KFENCE (Kernel Electric Fence)** — Low-overhead, sampling-based memory error detector using guard pages. Suitable for production.

**kmemleak** — Kernel memory leak detector that scans for unreferenced allocations.

**kmalloc()** — Kernel function to allocate physically contiguous memory from the SLUB allocator. For sizes 8 bytes to several KB.

**KPTI (Kernel Page Table Isolation)** — Separate page tables for user and kernel mode, mitigating Meltdown.

**kswapd** — Per-NUMA-node kernel daemon that performs background page reclaim when free memory is low.

**KSM (Kernel Same-page Merging)** — Scans for pages with identical content across processes and merges them COW-style. Used by KVM for VM memory deduplication.

---

## L

**LRU (Least Recently Used)** — Page replacement policy. Linux maintains 5 LRU lists per zone: active/inactive for anonymous and file pages, plus unevictable.

**Linear Mapping** — See Direct Mapping.

---

## M

**Major Fault** — Page fault requiring disk I/O (page not in RAM; must read from file or swap).

**Maple Tree** — Data structure (Linux 6.1+) replacing the red-black tree for VMA management in mm_struct. Better cache efficiency.

**memcg** — Short for memory cgroup (memory control group).

**MESI (Modified, Exclusive, Shared, Invalid)** — Cache coherency protocol ensuring data consistency across CPU caches.

**MGLRU (Multi-Gen LRU)** — Advanced page reclaim algorithm (Linux 6.1+) using multiple generations instead of binary active/inactive lists for better working set estimation.

**Minor Fault** — Page fault resolved without disk I/O (page already in RAM but not mapped in page table).

**mlock()** — System call that locks pages in RAM, preventing them from being swapped out.

**mmap()** — System call to create a virtual memory mapping (file-backed or anonymous).

**MMIO (Memory-Mapped I/O)** — Device registers accessed by mapping them into CPU address space and using load/store instructions.

**MMU (Memory Management Unit)** — CPU hardware that translates virtual addresses to physical addresses using page tables.

**mm_struct** — Kernel structure representing a process's entire virtual address space.

**MTE (Memory Tagging Extension)** — ARM64 hardware feature that assigns tags to memory pointers and granules, detecting use-after-free and overflow at hardware speed.

---

## N

**NUMA (Non-Uniform Memory Access)** — Multi-socket architecture where memory access latency depends on which CPU accesses which memory bank. Local access is faster than remote.

**NX Bit (No-Execute)** — Page table entry bit that prevents code execution from a page. x86 bit 63. ARM: XN (Execute Never).

---

## O

**OOM (Out of Memory) Killer** — Kernel mechanism that kills processes when memory cannot be freed through reclaim. Selects victims by oom_badness score.

**Overcommit** — Linux allows allocating more virtual memory than physical RAM + swap. Pages aren't backed until touched (demand paging).

---

## P

**Page** — Basic unit of memory management. Default 4KB on x86_64 and ARM64 (configurable on ARM64: 4KB, 16KB, 64KB).

**Page Cache** — Kernel cache of file data in RAM. File reads check page cache first; cache misses trigger disk I/O.

**Page Fault** — CPU exception when accessing a virtual address without a valid page table mapping. Triggers demand paging.

**PAGE_OFFSET** — Kernel virtual address where the direct mapping of physical memory starts.

**Page Frame** — A physical page of memory (as opposed to a virtual page).

**Page Frame Number (PFN)** — Physical address >> PAGE_SHIFT. Unique identifier for each physical page frame.

**Page Table** — Multi-level data structure mapping virtual addresses to physical addresses. x86_64 uses 4 or 5 levels.

**PAN (Privileged Access Never)** — ARM64 feature preventing kernel from accessing user memory without explicit override.

**PCID (Process Context Identifier)** — x86 feature that tags TLB entries with a process ID, avoiding TLB flush on context switch.

**PGD (Page Global Directory)** — Top level of the page table hierarchy. Pointed to by CR3 (x86) or TTBR (ARM).

**PMD (Page Middle Directory)** — Third level of the page table. 2MB huge pages are mapped at this level.

**Poisoning** — Filling freed memory with known patterns (e.g., 0x6B) to detect use-after-free (corruption of pattern = bug).

**PSI (Pressure Stall Information)** — Kernel interface reporting the percentage of time tasks are stalled waiting for memory, CPU, or I/O resources.

**PTE (Page Table Entry)** — Lowest level page table entry containing physical frame number and permission bits.

**PUD (Page Upper Directory)** — Second level of the page table. 1GB huge pages are mapped at this level.

---

## R

**Readahead** — Kernel reads extra file pages beyond what was requested, predicting sequential access. Reduces future major faults.

**Reclaim** — Process of freeing pages by writing back dirty pages, dropping clean cache pages, or swapping out anonymous pages.

**Redzone** — Guard bytes placed around slab objects to detect buffer overflow. Part of SLUB debug.

**Reverse Mapping (rmap)** — Data structure mapping from a physical page to all VMAs/PTEs that reference it. Needed for reclaim (unmap page from all page tables).

**RSS (Resident Set Size)** — Total physical memory currently mapped into a process (including shared pages, double-counted).

---

## S

**Shadow Memory (KASAN)** — Metadata memory (1/8th of kernel address space) tracking validity of each kernel memory byte.

**Shadow Stack** — Hardware-protected duplicate of return addresses used to detect ROP attacks.

**SLUB** — Default Linux slab allocator (since 2.6.23). Per-CPU freelists, lock-free fast path, minimal metadata.

**SMAP (Supervisor Mode Access Prevention)** — x86 feature preventing kernel from reading/writing user memory without explicit override.

**SMEP (Supervisor Mode Execution Prevention)** — x86 feature preventing kernel from executing code in user pages.

**Slab** — A set of pages from the buddy allocator, divided into fixed-size objects. Managed by SLUB/SLAB allocator.

**Sparsemem** — Memory model where `struct page` array is allocated only for present physical memory sections, saving memory on systems with sparse RAM layout.

**struct page** — 64-byte kernel structure describing every physical page frame. Uses unions to serve multiple page types.

**Swap** — Disk-backed extension of physical memory. Anonymous pages are written to swap when RAM is full.

**Swap Cache** — Pages in RAM that also have a copy on swap. Avoids redundant swap writes on re-fault.

**SWIOTLB (Software I/O TLB)** — Bounce buffer mechanism for 32-bit DMA devices accessing memory above 4GB without an IOMMU.

---

## T

**THP (Transparent Huge Pages)** — Kernel automatically uses 2MB huge pages for anonymous memory without application awareness.

**TLB (Translation Lookaside Buffer)** — CPU cache of recent virtual-to-physical address translations. TLB hit avoids expensive page table walk.

**TLB Shootdown** — IPI (Inter-Processor Interrupt) sent to other CPUs to invalidate stale TLB entries after page table modification.

**TTBR (Translation Table Base Register)** — ARM register holding page table base. TTBR0 for user, TTBR1 for kernel.

---

## V

**VMA (Virtual Memory Area)** — `vm_area_struct`: describes a contiguous region of a process's virtual address space with uniform permissions and backing.

**vmalloc()** — Kernel function to allocate virtually contiguous (but potentially physically scattered) memory.

**vmemmap** — Virtual mapping of the `struct page` array, allowing direct indexing from PFN to struct page.

**VMAP_STACK** — Kernel thread stacks allocated via vmalloc with guard pages for stack overflow detection.

---

## W

**Watermarks** — Per-zone free page thresholds: WMARK_MIN (emergency only), WMARK_LOW (wake kswapd), WMARK_HIGH (kswapd target).

**Writeback** — Process of writing dirty (modified) page cache pages back to their backing storage.

**W^X (Write XOR Execute)** — Security policy: a page should never be both writable and executable simultaneously.

---

## Z

**Zero Page** — Shared read-only page containing all zeros. First read from anonymous page returns this (COW'd on first write).

**Zone** — Physical memory region with specific addressing constraints: DMA (0-16MB), DMA32 (0-4GB), Normal (>4GB), Movable (CMA-like).

**zram** — RAM-backed block device with compression. Pages swapped to zram stay in RAM but compressed (~2:1), saving memory.

**zswap** — Compressed swap cache in RAM. Pages destined for swap are compressed first; only overflow goes to disk.

---

*Next: [Chapter 31 — Linux vs Other OS Memory Management](Chapter_31_OS_Comparison.md)*
