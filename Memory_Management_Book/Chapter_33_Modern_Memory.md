# Chapter 33: Modern Memory Management (Persistent Memory, GPU, CXL)

## Chapter Overview

Modern systems introduce new memory types beyond traditional DRAM: persistent memory (PMEM/Intel Optane), GPU memory (VRAM), CXL-attached memory, and heterogeneous memory. This chapter covers how Linux manages these emerging memory technologies.

---

## 33.1 Persistent Memory (PMEM)

```
Persistent Memory: Byte-addressable, non-volatile memory on the memory bus.
Survives reboot. Accessed like RAM, persists like storage.

Traditional:                     With PMEM:
┌────────┐   ┌─────────┐        ┌────────┐   ┌─────────────┐
│  CPU   │───│  DRAM    │        │  CPU   │───│  DRAM       │
└────────┘   │ (volatile)│        └────┬───┘   │ (volatile)  │
             └─────────┘              │       └─────────────┘
             ┌─────────┐              │       ┌─────────────┐
             │  SSD    │              └───────│  PMEM       │
             │(block I/O)│                     │(byte-addr,  │
             └─────────┘                      │ persistent)  │
                                              └─────────────┘
                                              ┌─────────────┐
                                              │  SSD        │
                                              └─────────────┘

PMEM Characteristics:
┌──────────────┬──────────────┬──────────────┬──────────────┐
│              │ DRAM         │ PMEM         │ NVMe SSD     │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ Latency      │ ~80ns        │ ~300ns       │ ~10µs        │
│ Bandwidth    │ ~50GB/s      │ ~10GB/s      │ ~7GB/s       │
│ Persistence  │ No           │ Yes          │ Yes          │
│ Interface    │ Memory bus   │ Memory bus   │ PCIe         │
│ Access       │ Load/store   │ Load/store   │ Block I/O    │
│ Capacity     │ ~128GB/DIMM  │ ~512GB/DIMM  │ ~8TB         │
│ Cost/GB      │ ~$5          │ ~$2          │ ~$0.10       │
│ Endurance    │ Unlimited    │ High         │ Limited      │
└──────────────┴──────────────┴──────────────┴──────────────┘

Linux PMEM modes:
1. fsdax:  Filesystem with DAX (Direct Access) — file mmap bypasses page cache
2. devdax: Device DAX — raw /dev/daxX.Y, mmap directly
3. sector: Block device mode — use like regular storage
4. raw:    No namespace — raw NVDIMM access
```

### DAX (Direct Access) — Bypassing Page Cache

```c
/*
 * DAX: mmap a file on PMEM filesystem → pages mapped directly
 * from PMEM without copying through page cache.
 * No double-copy, no writeback — writes go directly to persistent media.
 */

/* Traditional file mmap: */
/* App VA → Page table → Page Cache (DRAM copy) → Filesystem → Disk */
/* Two copies: disk→cache, cache→app (or just mapping cache page) */

/* DAX file mmap: */
/* App VA → Page table → PMEM page DIRECTLY */
/* Zero copies! No page cache involvement! */

/* Filesystem support: ext4, xfs with dax mount option */
/* mount -o dax /dev/pmem0 /mnt/pmem */

/* DAX mmap flow: */
/* 1. open("/mnt/pmem/file") */
/* 2. mmap(fd) → page fault → dax_iomap_fault() */
/* 3. Get physical PMEM address for file offset */
/* 4. Map PMEM PA directly into process page table */
/* 5. Process load/store goes directly to PMEM media! */

/* Ensuring persistence: */
void *data = mmap(NULL, size, PROT_WRITE, MAP_SHARED, fd, 0);
memcpy(data, source, len); 
/* Data in CPU cache — not yet persistent! */

/* Flush CPU cache to ensure data reaches PMEM: */
/* Intel: CLWB (Cache Line Write Back) or CLFLUSHOPT */
/* ARM: DC CVAP (Data Cache Clean by VA to Point of Persistence) */
pmem_persist(data, len);  /* Library call wrapping CLWB + SFENCE */
/* Now data is persistent — survives power failure */
```

---

## 33.2 GPU Memory Management

```
GPU memory hierarchy:
┌──────────────────────────────────────────────────────────┐
│ GPU Die                                                   │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐            │
│  │  SM    │ │  SM    │ │  SM    │ │  SM    │            │
│  │Registers│ │Registers│ │Registers│ │Registers│           │
│  │ Shared │ │ Shared │ │ Shared │ │ Shared │            │
│  │ Memory │ │ Memory │ │ Memory │ │ Memory │            │
│  └────┬───┘ └────┬───┘ └────┬───┘ └────┬───┘            │
│       └──────┬───┘          └────┬─────┘                 │
│              │                    │                        │
│         L2 Cache (shared, ~6MB)                           │
│              │                                            │
└──────────────┼────────────────────────────────────────────┘
               │
        ┌──────┴──────┐
        │ VRAM (GDDR6)│ Dedicated GPU memory
        │ 8-24GB      │ ~500-1000 GB/s bandwidth
        └──────┬──────┘
               │ PCIe
        ┌──────┴──────┐
        │ System RAM  │ CPU accessible
        │ (DRAM)      │ ~50 GB/s bandwidth
        └─────────────┘

Linux GPU memory management (DRM/TTM):

drm_gem_object: Base class for GPU buffer objects (GEM)
  ├── GPU memory types:
  │   VRAM:   High bandwidth, GPU-local
  │   GTT:    System RAM mapped for GPU access
  │   System: CPU memory, GPU accesses via PCIe
  │
  ├── Buffer placement: driver decides VRAM vs system
  │   LRU eviction: move cold buffers VRAM → system
  │   Promotion: move hot buffers system → VRAM
  │
  └── Sharing: dma-buf/PRIME for multi-GPU, display, video

TTM (Translation Table Manager):
  Per-GPU page tables mapping GPU VA → VRAM/system
  Similar to CPU page tables but for GPU address space
  Handles migration between VRAM and system memory
```

### Unified Memory (CUDA, HMM)

```
Heterogeneous Memory Management (HMM):
Allows GPU to share CPU page tables — unified address space.

Traditional (explicit):          HMM / Unified:
CPU: ptr_cpu = malloc()          Shared: ptr = malloc()
GPU: ptr_gpu = cudaMalloc()      GPU and CPU use SAME ptr!
cudaMemcpy(gpu←cpu)              Migration happens automatically
kernel<<<>>>(ptr_gpu);           kernel<<<>>>(ptr);
cudaMemcpy(cpu←gpu)              CPU accesses trigger migration

HMM mechanism:
1. CPU allocates anonymous memory
2. GPU driver registers as secondary MMU (mmu_notifier)
3. GPU accesses page: HMM migrates page CPU→GPU (VRAM)
4. GPU writes page: invalidated in CPU page tables
5. CPU accesses page: HMM migrates page GPU→CPU
6. Transparent to application — just uses normal pointers!

/* Kernel source: mm/hmm.c, include/linux/hmm.h */
/* Used by: NVIDIA GPU, AMD GPU, Intel GPU drivers */
```

---

## 33.3 CXL Memory (Compute Express Link)

```
CXL (Compute Express Link): PCIe-based memory expansion protocol.
Attaches external memory to CPU over CXL.mem protocol.

┌────────┐    DDR5     ┌───────────────┐
│  CPU   │────────────→│ Local DRAM    │  ~80ns, ~50GB/s
│        │             │ 256GB         │
│        │    CXL      ├───────────────┤
│        │────────────→│ CXL Memory    │  ~150-200ns, ~32GB/s
│        │  (PCIe 5.0) │ (Type 3 dev) │
└────────┘             │ 1-4TB         │
                       └───────────────┘

CXL Memory Types:
┌──────────────┬────────────────────────────────────────────────────┐
│ CXL Type 1   │ Accelerator with own memory (GPU-like)             │
│ CXL Type 2   │ Accelerator with host-managed memory (SmartNIC)   │
│ CXL Type 3   │ Memory expander (just memory, no compute)         │
│              │ ← Most relevant for memory management!            │
└──────────────┴────────────────────────────────────────────────────┘

Linux CXL support:
  CXL memory appears as a new NUMA node (higher latency)
  Kernel can tier memory: hot pages on DRAM, cold pages on CXL
  
  dmesg output:
  [  1.234] cxl_mem 0000:35:00.0: CXL Memory Expander
  [  1.235] SRAT: PXM 2 -> NUMA node 2 -> CXL memory
  
  Tiered memory:
  ┌─────────────────────────────────────────────┐
  │ Tier 0 (fastest): Local DRAM, NUMA node 0   │
  │ Tier 1 (medium):  CXL memory, NUMA node 2   │
  │ Tier 2 (slow):    NVMe / SSD / swap          │
  └─────────────────────────────────────────────┘
  
  Kernel automatically promotes/demotes pages between tiers
  based on access patterns (NUMA balancing + page promotion).
  
  CONFIG_NUMA_BALANCING + CONFIG_TIERED_MEMORY
  Hotness tracking: NUMA hint faults detect access patterns
  Hot CXL pages → promote to DRAM
  Cold DRAM pages → demote to CXL memory
```

---

## 33.4 Memory Tiering

```
Modern systems with heterogeneous memory need intelligent placement:

Page promotion/demotion in Linux 6.x:

┌────────────────────────────────────────────────────────────┐
│                                                            │
│  DRAM (fast, expensive, limited)                          │
│  ┌────────────────────────────────────────────────────┐   │
│  │ Hot pages: frequently accessed                      │   │
│  │ Promoted from lower tiers when detected as hot      │   │
│  └────────────────────────┬───────────────────────────┘   │
│                           │ ↑ promote (hot)                │
│                           │ ↓ demote (cold)               │
│  CXL Memory / PMEM (medium speed, cheaper, larger)       │
│  ┌────────────────────────┴───────────────────────────┐   │
│  │ Warm pages: occasionally accessed                   │   │
│  │ Demoted from DRAM when DRAM is full                 │   │
│  └────────────────────────┬───────────────────────────┘   │
│                           │ ↓ demote (cold)               │
│  Swap / SSD (slow, cheapest, largest)                    │
│  ┌────────────────────────┴───────────────────────────┐   │
│  │ Cold pages: rarely accessed                         │   │
│  └────────────────────────────────────────────────────┘   │
│                                                            │
└────────────────────────────────────────────────────────────┘

Implementation:
  kswapd on DRAM node → instead of reclaim, demote to CXL node
  NUMA balancing on CXL node → detect hot pages → promote to DRAM
  Net effect: automatic tiering without application changes
  
  /proc/sys/vm/demote_enabled = 1   (enable demotion)
  /sys/bus/node/devices/nodeX/memtier  (tier configuration)
```

---

## 33.5 Confidential Computing Memory

```
Confidential Computing: Encrypt VM memory from the host/hypervisor.

Technologies:
┌──────────────────┬──────────────────────────────────────────────┐
│ Intel TDX         │ Trust Domain Extensions: hardware-encrypted │
│                   │ VM memory. Host/hypervisor cannot read.      │
│                   │ Each TD has unique encryption key.           │
├──────────────────┼──────────────────────────────────────────────┤
│ AMD SEV-SNP       │ Secure Encrypted Virtualization with Secure │
│                   │ Nested Paging. Memory encrypted per-VM.      │
│                   │ Integrity protection prevents host tampering.│
├──────────────────┼──────────────────────────────────────────────┤
│ ARM CCA (Realms)  │ Confidential Compute Architecture           │
│                   │ Realm VMs isolated from hypervisor.          │
│                   │ Granule Protection Tables for memory access.│
└──────────────────┴──────────────────────────────────────────────┘

Memory impact:
  Each VM sees its own encrypted view of physical memory
  Hypervisor sees ciphertext (cannot read guest data)
  Hardware decrypts on CPU access (transparent to guest)
  
  Challenges for Linux MM:
  - Page migration must re-encrypt at destination
  - KSM cannot work (encrypted pages look random)
  - Ballooning requires guest cooperation
  - Dirty tracking needs hardware support
```

---

## Interview Questions

1. **Q: What is DAX and how does it work with persistent memory?**
   A: DAX (Direct Access) bypasses the page cache for files on PMEM filesystems. When a DAX file is mmap'd, the PMEM physical addresses are mapped directly into the process page table. Reads/writes go directly to persistent media without copying through page cache. CLWB instructions ensure data reaches the persistence domain.

2. **Q: How does Linux handle CXL memory?**
   A: CXL memory appears as a separate NUMA node with higher latency. The kernel uses memory tiering: hot pages stay on local DRAM (fast tier), cold pages are demoted to CXL memory (slow tier). NUMA balancing with hint faults detects access patterns for promotion/demotion. Applications are unaware — it's transparent.

3. **Q: What is HMM and why is it important for GPUs?**
   A: Heterogeneous Memory Management allows GPUs to share CPU virtual address space. Instead of separate allocations and explicit copies, CPU and GPU use the same pointers. HMM migrates pages between CPU and GPU memory based on access patterns, using mmu_notifiers for coherency. Simplifies GPU programming significantly.

---

## Summary

1. **PMEM**: Byte-addressable persistent memory. DAX enables direct access bypassing page cache.
2. **GPU Memory**: Managed by DRM/TTM; HMM enables unified CPU-GPU addressing.
3. **CXL Memory**: PCIe-attached memory expansion; appears as remote NUMA node.
4. **Memory Tiering**: Automatic promotion/demotion between fast (DRAM) and slow (CXL/PMEM) tiers.
5. **Confidential Computing**: Hardware-encrypted VM memory (TDX, SEV-SNP, CCA).
6. Linux kernel is rapidly evolving to support these heterogeneous memory technologies.

---

*Next: [Chapter 34 — Research Papers and Further Reading](Chapter_34_Research_Papers.md)*
