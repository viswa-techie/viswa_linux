# Chapter 36: Interview Preparation — 50+ Questions and Answers

## Chapter Overview

This chapter provides 55+ interview questions spanning all topics in this book. Questions are organized by difficulty (Beginner → Intermediate → Advanced → Expert) and cover hardware, kernel internals, drivers, security, debugging, and system design.

---

## SECTION A: Beginner Level (Conceptual Understanding)

### Q1: What is virtual memory and why is it used?
**A:** Virtual memory gives each process the illusion of a large, private, contiguous address space, independent of physical RAM. Benefits: process isolation (one process can't corrupt another's memory), memory overcommit (allocate more virtual than physical), demand paging (allocate physical pages only when accessed), and shared libraries (multiple processes share one physical copy of libc).

### Q2: What is a page? What is a page frame?
**A:** A page is a fixed-size unit of virtual memory (typically 4KB). A page frame is the corresponding unit of physical memory. The MMU maps virtual pages to physical page frames via page tables. The kernel tracks page frames using `struct page` (one per frame).

### Q3: What is a page fault?
**A:** A CPU exception triggered when a process accesses a virtual address that has no valid mapping in the page table. The kernel handles it by: allocating a physical page (minor fault, ~1µs), reading from disk (major fault, ~10ms for HDD), or sending SIGSEGV if the access is illegal.

### Q4: What is the difference between a minor and major page fault?
**A:** Minor fault: page is resolved without disk I/O (e.g., first touch of anonymous page, page already in page cache). Major fault: requires disk I/O (read from file or swap). Major faults are ~1000-10000× slower.

### Q5: What is a TLB?
**A:** Translation Lookaside Buffer — a CPU cache of recent virtual-to-physical address translations. TLB hit avoids expensive page table walk (TLB: ~1ns, page walk: ~10-100ns). x86_64 has ~512-1500 entries (L1 dTLB + L2 sTLB).

### Q6: What are huge pages and why use them?
**A:** Pages larger than the default 4KB (2MB or 1GB on x86_64). Each TLB entry covers more memory: 512 TLB entries × 4KB = 2MB coverage, but 512 × 2MB = 1GB coverage. Reduces TLB misses dramatically for large working sets.

### Q7: What is the difference between stack and heap memory?
**A:** Stack: automatic, LIFO, fast (pointer adjustment), limited size (~8MB default), stores local variables/return addresses. Heap: dynamic (malloc/free), slower (allocator overhead), virtually unlimited, stores long-lived objects. Stack is per-thread; heap is per-process.

### Q8: What does `malloc()` actually do?
**A:** `malloc()` is a userspace library function (glibc). It manages an arena (pool) of memory. If the arena has space, it returns a chunk immediately (no syscall). If not, it calls `mmap(MAP_ANONYMOUS)` or `brk()` to get more memory from the kernel. The kernel creates a VMA but allocates NO physical pages until first access (demand paging).

### Q9: What is swap?
**A:** Disk-backed extension of physical memory. When RAM is full, the kernel writes inactive anonymous pages to swap (swap out) to free physical RAM. When those pages are accessed again, they're read back (swap in). Adds latency but prevents OOM.

### Q10: What is the OOM killer?
**A:** When the kernel cannot free enough memory through reclaim, the OOM (Out of Memory) killer selects and kills a process to free memory. Victim selection based on `oom_badness()` score (primarily RSS + swap usage, adjusted by `oom_score_adj`).

---

## SECTION B: Intermediate Level (Kernel Internals)

### Q11: Describe the Linux virtual address space layout on x86_64.
**A:** 48-bit VA space split by canonical gap: User space (0x0000000000000000—0x00007FFFFFFFFFFF, 128TB) contains text, data, heap (grows up), mmap/libraries, stack (grows down). Kernel space (0xFFFF800000000000—0xFFFFFFFFFFFFFFFF, 128TB) contains vmalloc area, direct physical mapping (PAGE_OFFSET at 0xFFFF888000000000), vmemmap, KASAN shadow, modules, and kernel text.

### Q12: What is a VMA (`vm_area_struct`)?
**A:** A Virtual Memory Area describes a contiguous region of a process's virtual address space with uniform permissions and backing. Fields: `vm_start`, `vm_end` (boundaries), `vm_flags` (permissions: read/write/exec), `vm_file` (backing file or NULL for anonymous), `vm_ops` (fault handler callbacks). All VMAs for a process are stored in the `mm_struct->mm_mt` (maple tree).

### Q13: How does the buddy allocator work?
**A:** Physical pages are managed in orders 0-10 (1 page to 1024 pages = 4MB). Free blocks are kept in per-order lists. On allocation: find smallest order ≥ requested → if larger, split in half repeatedly until correct order. On free: check if buddy block is also free → merge into larger block → repeat. This naturally defragments memory.

### Q14: What is the SLUB allocator?
**A:** The default kernel memory allocator for objects smaller than a page. Maintains pre-allocated caches of fixed-size objects (kmalloc-8, kmalloc-16, ..., kmalloc-8192, plus custom caches). Fast path: per-CPU freelist, uses cmpxchg (lock-free). Medium path: take from per-node partial slab. Slow path: allocate new slab from buddy allocator.

### Q15: What is the page cache?
**A:** In-memory cache of file data. When a file is read, pages are kept in the page cache (indexed by `address_space` + `pgoff_t`). Subsequent reads hit the cache (~100ns) instead of disk (~10ms). Uses XArray for lookup. Page cache is reclaimable under memory pressure (clean pages freed instantly, dirty pages written back first).

### Q16: How does `mmap()` work?
**A:** `mmap()` creates a virtual memory mapping. Kernel: finds free VA range (`get_unmapped_area()`), creates VMA (`vm_area_struct`), inserts into process maple tree. NO physical pages allocated. On first access → page fault → `handle_mm_fault()` → allocates page + populates PTE. For file mappings: page cache page mapped directly. For anonymous: zero-filled page allocated.

### Q17: Explain Copy-on-Write (COW).
**A:** Optimization for `fork()`. Instead of copying all parent pages, both parent and child share the same physical pages, marked read-only. On first write by either process → write-protection fault → `do_wp_page()` → allocate new page, copy content, give writer the new page (writable), other process keeps the original. Enormously efficient for fork+exec pattern.

### Q18: What are the LRU lists in Linux?
**A:** 5 LRU lists per zone per NUMA node: active_anon, inactive_anon, active_file, inactive_file, unevictable. New pages start on inactive. If accessed again (referenced), promoted to active. Pages age from active → inactive → reclaim candidate. Reclaim scans inactive tails: clean file pages freed, dirty pages written back, anon pages swapped. MGLRU (6.1+) uses multiple generations instead.

### Q19: What is kswapd?
**A:** Per-NUMA-node kernel daemon for background page reclaim. Woken when free pages drop below WMARK_LOW watermark. Scans LRU lists, reclaiming pages until free pages reach WMARK_HIGH. Runs asynchronously — doesn't block allocating processes. If kswapd can't keep up, allocating processes do "direct reclaim" synchronously.

### Q20: What is `struct page` and why is it important?
**A:** 64-byte structure describing every physical page frame. Uses unions for different page types (page cache, slab, compound/huge page). Key fields: `flags` (PG_locked, PG_dirty...), `_refcount` (references), `_mapcount` (PTE mappings), `lru` (LRU list linkage), `mapping` (owner). With 16GB RAM = 4M pages × 64 bytes = 256MB overhead.

---

## SECTION C: Advanced Level (Deep Kernel / Driver Knowledge)

### Q21: Trace the complete path from `malloc()` to physical memory allocation.
**A:** `malloc()` → glibc arena check → no free chunk → `mmap(MAP_ANONYMOUS)` syscall → `do_mmap()` → `get_unmapped_area()` finds free VA → `mmap_region()` creates VMA → returns VA (no physical page). First write → #PF → `exc_page_fault()` → `do_user_addr_fault()` → `find_vma()` → `handle_mm_fault()` → walk/create PGD/P4D/PUD/PMD → `handle_pte_fault()` → `do_anonymous_page()` → `alloc_zeroed_user_highpage()` → `__alloc_pages(GFP_HIGHUSER)` → buddy allocator → physical page → create PTE → set in page table → return.

### Q22: How does the kernel handle a page fault for a memory-mapped file?
**A:** `handle_pte_fault()` → PTE empty + file-backed VMA → `do_fault()` → `do_read_fault()` / `do_cow_fault()` / `do_shared_fault()` → `__do_fault()` → `vma->vm_ops->fault()` → `filemap_fault()` → search page cache (XArray lookup) → HIT: use cached page → MISS: allocate page, add to cache, submit read I/O, wait for completion → `finish_fault()` → set PTE pointing to page cache page → return.

### Q23: What are GFP flags and how do they affect allocation?
**A:** GFP (Get Free Pages) flags specify allocation context: `GFP_KERNEL` (may sleep, may reclaim, may do I/O — normal kernel code), `GFP_ATOMIC` (cannot sleep, no reclaim — interrupt/spinlock context), `GFP_HIGHUSER` (user pages, may use HIGHMEM zone), `__GFP_ZERO` (zero-fill), `__GFP_DMA` (allocate from DMA zone < 16MB), `__GFP_NORETRY` (fail rather than reclaim aggressively). Wrong flags → deadlock (sleeping in atomic context) or failed allocations.

### Q24: Explain the difference between `kmalloc()`, `vmalloc()`, and `alloc_pages()`.
**A:** `kmalloc(size, gfp)`: physically contiguous, from SLUB, fast, limited to ~8KB, for most kernel objects. `vmalloc(size)`: virtually contiguous but physically scattered, can allocate large blocks (MBs), slower (requires page table setup), for large buffers. `alloc_pages(gfp, order)`: raw page allocation from buddy, returns `struct page *`, lowest-level allocator.

### Q25: What is `dma_alloc_coherent()` and when is it used?
**A:** Allocates physically contiguous, cache-coherent memory accessible by both CPU and DMA devices. Returns both CPU virtual address and DMA address (bus address). Hardware maintains cache coherency — no manual cache flushing. Used for long-lived DMA buffers like descriptor rings. Expensive (uses CMA or low zones).

### Q26: What is the difference between coherent and streaming DMA mappings?
**A:** Coherent (`dma_alloc_coherent()`): allocates new memory, hardware-coherent (no cache ops), for long-lived buffers. Streaming (`dma_map_single()`): maps existing kernel memory, requires manual cache sync (`dma_sync_single_for_cpu/device()`), for per-transfer data buffers. Streaming is cheaper (no allocation) but needs explicit sync calls.

### Q27: How does a driver implement `mmap()` to share memory with userspace?
**A:** Driver's `file_operations->mmap()` is called with a VMA. Three approaches: (1) `remap_pfn_range()` — map physical pages immediately (DMA buffers, MMIO), (2) `vm_ops->fault()` — map pages on demand via fault handler (flexible, sparse), (3) `dma_mmap_coherent()` — helper for DMA buffer sharing. Must set cache attributes (`pgprot_noncached` for device memory) and VM flags (`VM_IO | VM_DONTEXPAND`).

### Q28: What is reverse mapping (rmap) and why is it needed?
**A:** Rmap maps from a physical page → all VMAs/PTEs that reference it. Needed for reclaim: when freeing a page, the kernel must remove it from ALL process page tables that map it. Without rmap, you'd need to scan every process's page table. With rmap: `page->mapping` → `anon_vma` or `address_space` → walk the chain of VMAs → unmap PTE in each.

### Q29: How does memory compaction work?
**A:** Compaction creates large contiguous free blocks by migrating movable pages. Two scanners: one scans from bottom finding free pages, another scans from top finding movable pages. Pages are migrated (copy content, update PTE, update rmap) from top to bottom, consolidating free space at the top. Triggered by high-order allocation failures or explicitly via `/proc/sys/vm/compact_memory`.

### Q30: What is MGLRU and how does it improve over traditional LRU?
**A:** Multi-Gen LRU (Linux 6.1+) replaces the binary active/inactive split with multiple generations (typically 4). Pages age through generations based on access. The youngest generation contains recently accessed pages; the oldest is eviction candidates. Advantages: better working set estimation, reduced scanning overhead, and significant performance improvement for workloads with diverse access patterns.

---

## SECTION D: Security & Debugging

### Q31: What is KPTI and what vulnerability does it mitigate?
**A:** Kernel Page Table Isolation: maintains separate page tables for user and kernel mode. In userspace, only minimal kernel trampolines are mapped (entry stubs, IDT). Mitigates Meltdown (CVE-2017-5754): speculative execution could read kernel memory and leak it via cache side-channels. On syscall/interrupt: CR3 switches to kernel page table. On return: switches back to user page table.

### Q32: How does KASAN detect use-after-free?
**A:** KASAN maintains shadow memory (1 byte per 8 bytes of kernel memory). On `kfree()`, the freed object's shadow bytes are set to 0xFD (freed). The freed memory is quarantined (not immediately reused). Compiler inserts shadow checks before every memory access. If code accesses freed memory → shadow check finds 0xFD → KASAN reports use-after-free with both allocation and free stack traces.

### Q33: What is KFENCE and why can it run in production?
**A:** KFENCE (Kernel Electric Fence) is a sampling-based memory error detector. It diverts a small fraction of allocations to a pool where each object is surrounded by guard pages (unmapped). OOB access → hits guard page → page fault → detected. After free → page permissions revoked → UAF → fault. Overhead < 1% because only a statistical sample of allocations is protected.

### Q34: How would you diagnose a kernel memory leak?
**A:** (1) Monitor `slabtop` or `/proc/slabinfo` for monotonically growing slab caches. (2) Enable `kmemleak`: `echo scan > /sys/kernel/debug/kmemleak` → reports unreferenced allocations with backtraces showing allocation site. (3) Monitor `/proc/meminfo` SUnreclaim (unreclaimable slab) over time. (4) Enable `page_owner` for per-page tracking. (5) Use `ftrace` kmem events to correlate alloc/free pairs.

### Q35: What is Address Space Layout Randomization (ASLR)?
**A:** ASLR randomizes the base addresses of stack (~22 bits entropy), mmap/libraries (~28 bits), heap (~13 bits), and executable (PIE, ~28 bits) on each execution. Attacker cannot predict where code/data resides, defeating return-to-libc, ROP chain, and address guess attacks. KASLR extends this to kernel text, modules, and physmap at boot.

### Q36: What is the W^X policy?
**A:** Write XOR Execute: a memory page should NEVER be simultaneously writable AND executable. Enforced via NX bit (x86, PTE bit 63) / XN bit (ARM). Data pages (stack, heap) are NX (no execute). Code pages (.text) are read-only (no write). Prevents the classic attack of injecting shellcode into a data buffer and executing it.

### Q37: What is SMAP/PAN and why does it matter?
**A:** SMAP (Intel) / PAN (ARM): prevents the kernel from reading/writing user memory unless explicitly allowed (STAC/CLAC or PAN disable). Without this, a corrupted kernel pointer pointing to userspace could read/write attacker-controlled data. `copy_from_user()`/`copy_to_user()` temporarily disable SMAP/PAN for legitimate user data access.

---

## SECTION E: Cgroups, Containers, Virtualization

### Q38: How do memory cgroups work?
**A:** Each cgroup has a `mem_cgroup` structure tracking memory usage. Every page/folio stores its cgroup owner (`folio->memcg_data`). On allocation: `mem_cgroup_charge()` charges the page to the cgroup. If usage exceeds `memory.max`: cgroup-local OOM killer. `memory.high`: throttle and aggressively reclaim. Limits are hierarchical: child can't exceed parent.

### Q39: What is the difference between `memory.high` and `memory.max` in cgroup v2?
**A:** `memory.max`: hard limit — exceeding triggers OOM killer (processes killed). `memory.high`: soft limit — exceeding causes aggressive reclaim and process throttling (slowdown), but no killing. Best practice: set `memory.high` as the normal operating limit and `memory.max` as the safety net.

### Q40: How does Kubernetes use memory cgroups?
**A:** K8s translates pod resources to cgroups: `limits.memory` → `memory.max`, `requests.memory` → `memory.low`, QoS class → `oom_score_adj` (-998 for Guaranteed, 1000 for BestEffort). Pods exceeding limits are OOM-killed. Under node pressure, BestEffort pods die first, Guaranteed pods last.

### Q41: How does KVM (EPT) handle guest page faults?
**A:** Two scenarios: (1) Guest page fault (guest PTE not present): handled entirely within guest OS, no VM exit. (2) EPT violation (GPA not in EPT): VM exit → KVM handler → `kvm_mmu_page_fault()` → find memslot for GPA → get host page (`gup()`) → install EPT mapping (GPA → HPA) → resume guest. EPT is built lazily, on demand.

### Q42: What is memory ballooning in virtualization?
**A:** Dynamic memory reclaim from VMs. Hypervisor tells guest balloon driver to inflate (allocate pages inside guest). Guest allocates pages → KVM unmaps them from EPT → host reclaims physical pages for other VMs. Deflate: balloon frees pages → guest has more available RAM. Transparent to guest applications.

### Q43: What is KSM and what are its security implications?
**A:** Kernel Same-page Merging deduplicates identical pages across processes/VMs (COW-style sharing). Saves RAM when running many similar VMs. Security risk: timing side-channel — writing to a shared (COW) page takes longer than a non-shared page, allowing an attacker to probe whether another VM has specific memory content. Mitigation: disable across security boundaries.

---

## SECTION F: Modern Systems & Expert Level

### Q44: What is DAX and how does it work with persistent memory?
**A:** Direct Access: filesystem on PMEM bypasses the page cache. `mmap()` maps PMEM physical addresses directly into process page tables. Reads/writes go directly to the persistent media. No double-copy (disk→cache→user). `CLWB` instruction flushes CPU cache to ensure data reaches persistence domain.

### Q45: What is CXL memory and how does Linux manage it?
**A:** CXL (Compute Express Link) attaches memory via PCIe 5.0. Appears as a remote NUMA node with higher latency (~150ns vs ~80ns local DRAM). Kernel uses memory tiering: hot pages on DRAM, cold pages demoted to CXL. NUMA balancing detects hot CXL pages and promotes them to DRAM. Transparent to applications.

### Q46: How does HMM enable unified CPU-GPU memory?
**A:** Heterogeneous Memory Management allows GPU and CPU to share virtual address space. GPU driver registers as secondary MMU via `mmu_notifier`. GPU access to unmapped page → HMM migrates page from CPU to GPU VRAM. CPU access to GPU-resident page → HMM migrates back. No explicit copies needed.

### Q47: How does the 2D page walk work in virtualization?
**A:** Guest page table walks need EPT translations. Each of 4 guest levels requires its own 4-level EPT walk, plus 1 final EPT walk for data. Worst case: 4×4 + 4 = 20 memory accesses. Mitigations: TLB caching (VPID/VMID), huge pages (reduce levels), EPT paging structure caches. With 1GB pages on both guest and host: 2×2 + 2 = 6 accesses.

### Q48: Explain the memory reclaim flow when `__alloc_pages()` fails.
**A:** `get_page_from_freelist()` fails (below watermarks) → `__alloc_pages_slowpath()`: (1) Wake kswapd. (2) Retry allocation. (3) Direct reclaim (`try_to_free_pages()` → `shrink_node()`: scan LRU, free clean pages, write back dirty, swap anon). (4) Retry. (5) Try compaction (for high-order). (6) Retry. (7) OOM killer (`out_of_memory()` → `select_bad_process()` → `oom_kill_process()`). (8) Retry.

### Q49: What is the Folio abstraction and why was it introduced?
**A:** A `struct folio` (Linux 5.16+) represents a naturally-aligned group of pages (1 or more). It replaces the ambiguous use of `struct page *` which could mean either a single page or the head of a compound page. Folios simplify page cache and filesystem code, make the head/tail page distinction explicit, and reduce refcount overhead for compound pages.

### Q50: What is the maple tree and why did it replace the red-black tree for VMAs?
**A:** The maple tree (Linux 6.1+) is a B-tree variant optimized for storing ranges (VMA start/end addresses). Replaced the rbtree + linked list in `mm_struct` because: (1) better cache behavior (nodes packaged compactly), (2) RCU-safe (supports lockless VMA lookup), (3) built-in range operations (find gap, range query), (4) reduced memory overhead vs maintaining both rbtree + list.

---

## SECTION G: System Design & Scenario Questions

### Q51: Design a memory allocator for an embedded system with 64KB RAM.
**A:** No MMU, so no virtual memory. Options: (1) Static allocation (MISRA-compliant): arrays at compile-time, zero runtime failure. (2) Fixed-size pool allocator: create pools of common sizes (16, 32, 64, 128 bytes). O(1) alloc/free via bitmap or freelist. No fragmentation. (3) Avoid general-purpose heap (malloc) — fragmentation is fatal in constrained systems. (4) Stack allocation for temporaries. (5) Statically verify worst-case memory usage.

### Q52: A system is experiencing OOM kills despite free memory shown in `/proc/meminfo`. Why?
**A:** Possible causes: (1) Memory fragmentation — free memory exists but not in contiguous blocks for the requested order (check `/proc/buddyinfo`). (2) Zone imbalance — free memory in Normal zone but allocation needs DMA zone. (3) cgroup limit reached — container hit `memory.max` even though host has free memory. (4) Memory leak in unreclaimable slab — check `SUnreclaim` in meminfo. (5) Too many mlock'd pages or hugetlb reserved pages.

### Q53: How would you optimize a database server's memory configuration?
**A:** (1) `vm.swappiness=10` — minimize swapping, keep working set in RAM. (2) Use hugetlbfs for shared buffers — reduces TLB misses significantly. (3) `vm.dirty_ratio=40`, `dirty_background_ratio=10` — buffer writes, reduce I/O spikes. (4) NUMA bind to local node (`numactl --membind`). (5) `overcommit_memory=2` — strict accounting, prevent OOM surprises. (6) Disable THP or set to madvise — THP compaction can cause latency spikes.

### Q54: A process RSS keeps growing over time. How do you find the leak?
**A:** (1) `cat /proc/PID/smaps_rollup` — track PSS/RSS over time, identify which types grow (anon? file? shmem?). (2) `smaps` — find which VMA region is growing (heap? mmap?). (3) If userspace leak: `valgrind --leak-check=full`, or `gcc -fsanitize=address`. (4) If native heap: use `gdb` + `info proc mappings`, or `/proc/PID/maps` diff over time. (5) Java/managed: JVM heap dump. (6) Check for mmap without munmap.

### Q55: Explain how you'd implement zero-copy networking in a driver.
**A:** (1) Allocate DMA-capable buffer pool at init (`dma_alloc_coherent` or scatter-gather pages). (2) On RX: device DMAs packet directly into buffer, pass buffer to network stack/userspace via `mmap()` or `splice()` — no memcpy. (3) AF_XDP: user registers buffer pool, driver fills directly, user processes in-place. (4) sendfile(): page cache → TCP socket, no user-kernel copy. (5) MSG_ZEROCOPY: kernel maps userspace buffer for device DMA, no copy to socket buffer.

---

## Quick Reference: Key Values to Remember

```
┌────────────────────────────────────────────────────────────────┐
│ Memory Quantities                                              │
├────────────────────────────────────────────────────────────────┤
│ Page size (x86_64):          4096 bytes (4KB)                 │
│ Page size (ARM64):           4KB, 16KB, or 64KB               │
│ Huge page:                   2MB (PMD) or 1GB (PUD) on x86_64 │
│ Cache line:                  64 bytes                          │
│ struct page size:            64 bytes                          │
│ vm_area_struct:              ~200 bytes                        │
│ TLB entries (typical):      512-1500 (L1+L2 combined)         │
│ Page table entry:            8 bytes (64-bit)                  │
│ Max VA (48-bit):            256 TB (128 TB user + 128 TB kern)│
│ Max VA (57-bit):            128 PB (5-level paging)           │
│ Buddy max order:             10 (1024 pages = 4MB)            │
│ kmalloc max:                 Varies (typically 8KB-128KB)      │
│ Default stack size (user):  8MB                                │
│ Default stack size (kernel):16KB (4 pages, x86_64)            │
│ KASAN overhead:             ~2-3× memory + CPU                 │
│ KFENCE overhead:            < 1%                               │
│ Minor fault latency:        ~1µs                               │
│ Major fault latency:        ~100µs (SSD) to ~10ms (HDD)       │
│ TLB hit latency:            ~1ns                               │
│ Page table walk:            ~10-100ns                          │
│ DRAM access:                ~80ns                              │
│ L1 cache:                   ~1ns                               │
│ L2 cache:                   ~5ns                               │
│ L3 cache:                   ~15ns                              │
│ NUMA remote access:         ~140ns (vs ~80ns local)            │
│ CXL memory:                 ~150-200ns                         │
│ PMEM:                       ~300ns                             │
│ SSD I/O:                    ~100µs                             │
│ HDD I/O:                    ~10ms                              │
└────────────────────────────────────────────────────────────────┘
```

---

## Summary

This chapter provides 55+ interview questions covering:
- **Beginner (Q1-Q10)**: Virtual memory, pages, faults, TLB, swap, OOM
- **Intermediate (Q11-Q20)**: VMA, buddy, SLUB, page cache, mmap, COW, LRU
- **Advanced (Q21-Q30)**: Full fault paths, GFP flags, DMA, rmap, compaction, MGLRU
- **Security (Q31-Q37)**: KPTI, KASAN, KFENCE, ASLR, W^X, SMAP/PAN
- **Containers/VM (Q38-Q43)**: cgroups, K8s, KVM/EPT, ballooning, KSM
- **Expert (Q44-Q50)**: DAX/PMEM, CXL, HMM, folio, maple tree, 2D walks
- **Design (Q51-Q55)**: Embedded allocator, OOM debugging, DB tuning, leak detection

Master these and you'll be prepared for kernel engineer and embedded systems interviews from junior to principal level.

---

*This concludes the Linux Memory Management Book. Return to [Master Index](MASTER_INDEX.md) for chapter navigation.*
