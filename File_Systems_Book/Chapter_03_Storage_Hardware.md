# Chapter 3: Storage Hardware Architecture

## Learning Goals
- Understand HDD physical structure and performance characteristics
- Know SSD/flash internals: NAND types, FTL, write amplification
- Learn block device abstraction in the Linux kernel
- Appreciate how hardware shapes file system design

---

## 3.1 Hard Disk Drive (HDD) Architecture

### Physical Structure

```
                 Spindle Motor
                     │
        ┌────────────┼────────────┐
        │   ┌────────┴────────┐   │
        │   │    Platter 0    │   │
        │   │   ┌──────────┐  │   │  ← Tracks (concentric circles)
        │   │   │ ████████ │  │   │  ← Sectors (512B or 4KB segments)
        │   │   │ ████████ │  │   │
        │   │   └──────────┘  │   │
        │   └─────────────────┘   │
        │   ┌─────────────────┐   │
        │   │    Platter 1    │   │
        │   └─────────────────┘   │
        │         ...             │
        │                         │
        │  Read/Write Head ◄──────│── Actuator Arm
        └─────────────────────────┘

Terminology:
  Platter   = One disk surface, both sides usable
  Track     = One concentric ring on a platter surface
  Sector    = Smallest addressable unit (historically 512B, now 4KiB)
  Cylinder  = All tracks at the same radius across all platters
  Head      = One read/write head per platter surface
```

### HDD Performance Characteristics

```
Operation               │ Typical Time       │ Impact on FS Design
────────────────────────┼────────────────────┼──────────────────────
Seek time               │ 3-15 ms            │ Locality matters!
Rotational latency      │ 2-8 ms (avg half rot)│ Sequential > random
Transfer rate (seq.)    │ 100-250 MB/s       │ Large I/O preferred
Random IOPS             │ 75-200 IOPS        │ Minimize random I/O
Queue depth             │ 32 (NCQ)           │ Elevator scheduling

Key insight: 1 random I/O ≈ 10 ms = could transfer ~1 MB sequentially!

This is why FS design focuses on:
  → Placing related data close together (locality)
  → Large, sequential writes (delayed allocation)
  → Caching to avoid disk access entirely
```

### Sector Sizes: 512B vs 4KiB (Advanced Format)

```
Traditional:  512 bytes/sector
Advanced:     4096 bytes/sector (physical), may emulate 512B (logical)

Linux perspective:
  - Kernel block layer works in "sectors" = 512 bytes (always!)
  - Hardware reports physical & logical sector sizes
  - File system aligns to physical sector boundary for performance

Check sector sizes:
  $ cat /sys/block/sda/queue/physical_block_size   → 4096
  $ cat /sys/block/sda/queue/logical_block_size     → 512
  $ blockdev --getpbsz /dev/sda                     → 4096
```

---

## 3.2 Solid-State Drive (SSD) Architecture

### NAND Flash Fundamentals

```
Flash Memory Hierarchy:
  ┌─────────────────────────────────────────┐
  │              SSD Package                │
  │  ┌─────────┐ ┌─────────┐ ┌─────────┐  │
  │  │  Die 0  │ │  Die 1  │ │  Die N  │  │
  │  │ ┌─────┐ │ │ ┌─────┐ │ │ ┌─────┐ │  │
  │  │ │Plane│ │ │ │Plane│ │ │ │Plane│ │  │
  │  │ │ ┌─┐ │ │ │ │ ┌─┐ │ │ │ │ ┌─┐ │ │  │
  │  │ │ │B│ │ │ │ │ │B│ │ │ │ │ │B│ │ │  │  B = Block
  │  │ │ │P│ │ │ │ │ │P│ │ │ │ │ │P│ │ │  │  P = Page
  │  │ │ └─┘ │ │ │ │ └─┘ │ │ │ │ └─┘ │ │  │
  │  │ └─────┘ │ │ └─────┘ │ │ └─────┘ │  │
  │  └─────────┘ └─────────┘ └─────────┘  │
  └─────────────────────────────────────────┘

Key constraints:
  Read unit  = Page (4KB-16KB)
  Write unit = Page (4KB-16KB) — only to ERASED pages
  Erase unit = Block (128-512 pages, i.e., 512KB-8MB)

  ⚠ You CANNOT overwrite a page in place!
    1. Read entire block
    2. Erase block (all pages → 0xFF)
    3. Write back with modified data
```

### NAND Types

```
Type   │ Bits/Cell │ Endurance (P/E)  │ Speed    │ Cost    │ Use
───────┼───────────┼──────────────────┼──────────┼─────────┼──────────
SLC    │ 1         │ 50K-100K cycles  │ Fastest  │ $$$$$   │ Enterprise
MLC    │ 2         │ 3K-10K           │ Fast     │ $$$$    │ Enterprise
TLC    │ 3         │ 1K-3K            │ Medium   │ $$$     │ Consumer
QLC    │ 4         │ 100-1K           │ Slowest  │ $$      │ Archive
PLC    │ 5         │ < 100            │ Very slow│ $       │ Future
```

### Flash Translation Layer (FTL)

```
Problem: File systems assume in-place overwrites. Flash cannot.

Solution: FTL = firmware that maps logical blocks → physical pages

  Logical Block Address (LBA)     Physical Page Address (PPA)
  ┌──────────┐                    ┌──────────────────────┐
  │ LBA 0    │ ──────────────────►│ Die 0, Block 5, Pg 3 │
  │ LBA 1    │ ──────────────────►│ Die 1, Block 2, Pg 7 │
  │ LBA 2    │ ──────────────────►│ Die 0, Block 8, Pg 1 │
  │ ...      │                    │ ...                    │
  └──────────┘                    └──────────────────────┘

FTL responsibilities:
  1. Logical → Physical mapping table
  2. Wear leveling (spread writes across all blocks)
  3. Garbage collection (reclaim stale pages)
  4. Bad block management
  5. Write buffer management
```

### Write Amplification

```
Write amplification factor (WAF) = Physical writes / Logical writes

Example: Write 4KB to a full SSD
  1. SSD picks a block with stale pages
  2. Reads all VALID pages from that block (e.g., 100 valid pages)
  3. Erases the entire block
  4. Writes back 100 valid pages + your 1 new page
  → WAF = 101:1 (worst case)

Minimizing WAF:
  - TRIM/discard commands (tell SSD which pages are free)
  - Over-provisioning (extra capacity for GC)
  - Sequential writes (reduce GC overhead)
  - Flash-aware FS (F2FS, UBIFS)
```

### TRIM / Discard

```
TRIM tells the SSD: "These logical blocks are no longer in use."
→ SSD can proactively erase them, improving future GC

Linux support:
  $ mount -o discard /dev/sda1 /mnt        ← Continuous TRIM
  $ fstrim /mnt                             ← Periodic TRIM (preferred)
  $ systemctl enable fstrim.timer           ← Weekly TRIM via systemd

Kernel path:
  unlink() → ext4_free_blocks() → sb_issue_discard()
  → blk_queue_discard() → SCSI UNMAP / ATA TRIM
```

---

## 3.3 eMMC and UFS (Embedded Storage)

### eMMC (embedded MultiMediaCard)

```
                  ┌──────────────────────┐
  Host ◄──────────┤     eMMC Package     │
  (SoC)  MMC Bus  │  ┌────────────────┐  │
                  │  │ NAND Flash     │  │
                  │  │ (TLC/MLC)      │  │
                  │  ├────────────────┤  │
                  │  │ Controller     │  │
                  │  │ (simple FTL)   │  │
                  │  └────────────────┘  │
                  └──────────────────────┘

  - Parallel interface (1/4/8 bit bus)
  - Sequential read: 100-400 MB/s
  - Common in budget phones, IoT, automotive
  - Linux driver: drivers/mmc/
```

### UFS (Universal Flash Storage)

```
  - Serial interface (LVDS lanes)
  - Sequential read: 1-4 GB/s (UFS 3.1+)
  - Full command queuing (like NVMe)
  - Used in flagship phones, automotive grade
  - Linux driver: drivers/ufs/
```

### Comparison for Automotive / Embedded

```
Feature          │ eMMC 5.1      │ UFS 3.1       │ NVMe SSD
─────────────────┼───────────────┼───────────────┼───────────────
Seq. Read        │ 400 MB/s      │ 2.1 GB/s      │ 3-7 GB/s
Seq. Write       │ 200 MB/s      │ 1.2 GB/s      │ 2-5 GB/s
Random IOPS      │ 10K           │ 70K           │ 500K+
Interface        │ Parallel      │ Serial        │ PCIe
Command Queue    │ 1             │ 32            │ 64K
Power            │ Low           │ Low           │ Medium-High
Temperature      │ -40 to 85°C   │ -40 to 105°C  │ 0 to 70°C
Use case         │ IoT, budget   │ Premium mobile│ Server, desktop
                 │ automotive    │ automotive    │
```

---

## 3.4 Block Device Abstraction in Linux

### The Block Layer

```
User space:   read(fd, buf, size)
                │
Kernel:       VFS → File System (ext4)
                │
              Block Layer (bio requests)
                │
              I/O Scheduler (mq-deadline, bfq, kyber, none)
                │
              Block Device Driver (SCSI, NVMe, MMC)
                │
Hardware:     HDD / SSD / eMMC / UFS

Key abstractions:
  struct block_device  ← represents one block device (/dev/sda)
  struct bio           ← one block I/O request
  struct request       ← merged bio(s) queued for dispatch
  struct gendisk       ← registered disk identity
  struct request_queue ← per-device I/O queue
```

### struct bio — The Block I/O Unit

```c
/* include/linux/bio.h */
struct bio {
    struct bio          *bi_next;       /* request queue link */
    struct block_device *bi_bdev;       /* target block device */
    unsigned int        bi_opf;         /* REQ_OP_READ, REQ_OP_WRITE, etc. */
    unsigned short      bi_vcnt;        /* number of bio_vecs */
    sector_t            bi_iter.bi_sector; /* start sector (512B units) */
    struct bio_vec      *bi_io_vec;     /* scatter-gather list */
    bio_end_io_t        *bi_end_io;     /* completion callback */
    void                *bi_private;
};

struct bio_vec {
    struct page *bv_page;   /* page containing data */
    unsigned int bv_len;    /* bytes in this segment */
    unsigned int bv_offset; /* offset within page */
};
```

### I/O Schedulers

```
Scheduler   │ Algorithm              │ Best For
────────────┼────────────────────────┼──────────────────────
none        │ FIFO, no reordering    │ NVMe SSDs (fast enough)
mq-deadline │ Deadline + batching    │ General purpose, default
bfq         │ Budget Fair Queuing    │ Desktop, interactive I/O
kyber       │ Token-based, low lat   │ Fast devices, latency

Check / change scheduler:
  $ cat /sys/block/sda/queue/scheduler
  [mq-deadline] kyber bfq none

  $ echo "bfq" > /sys/block/sda/queue/scheduler
```

---

## 3.5 Sectors, Blocks, and Pages

```
Concept         │ Size          │ Defined By        │ Used By
────────────────┼───────────────┼───────────────────┼──────────────
Sector          │ 512 B (kernel)│ Block layer       │ bio, request
Physical sector │ 512 B or 4 KB│ Hardware           │ Disk firmware
FS block        │ 1-4 KB       │ mkfs (format time) │ File system
Page            │ 4 KB (ARM/x86)│ MMU (hardware)    │ Page cache

Relationships (typical ext4, 4KB blocks, 4KB pages):
  1 page = 1 FS block = 8 kernel sectors = 1 physical sector (Advanced Format)

Why alignment matters:
  - Unaligned writes cross sector boundaries → read-modify-write
  - Unaligned partition start → every I/O misaligned
  - Modern mkfs tools align automatically
```

### Checking Alignment

```bash
# Partition alignment
$ sudo parted /dev/sda align-check optimal 1
1 aligned

# File system block size
$ tune2fs -l /dev/sda1 | grep "Block size"
Block size:               4096

# Page size
$ getconf PAGESIZE
4096
```

---

## 3.6 Storage Stack Latency

```
Component                  │ Latency (approx.)
───────────────────────────┼──────────────────
L1 Cache hit               │ 1 ns
L2 Cache hit               │ 4 ns
L3 Cache hit               │ 12 ns
DRAM access                │ 60-100 ns
Page cache hit             │ ~100 ns
Optane / Intel PMEM        │ 300-500 ns
NVMe SSD (random read)     │ 10-100 μs
SATA SSD (random read)     │ 50-200 μs
eMMC (random read)         │ 200-500 μs
HDD (random read)          │ 3-15 ms
Network (NFS, local)       │ 0.1-1 ms
Network (NFS, WAN)         │ 10-200 ms

Key takeaway: Page cache hit is ~100,000x faster than HDD random I/O!
FS design priority: KEEP DATA IN CACHE.
```

---

## 3.7 Storage in Automotive / Embedded Context

```
Typical Automotive Platform (e.g., SA8155P):
  ┌─────────────┐
  │   SoC       │
  │  ┌───────┐  │
  │  │ CPU   │  │
  │  │ cores │──┼─── UFS 2.1/3.0 (main storage, 64-256 GB)
  │  │       │  │      ├── Android rootfs (ext4)
  │  │       │  │      ├── Userdata (ext4 or f2fs)
  │  │       │  │      └── Vendor partitions
  │  └───────┘  │
  │             │
  │  ┌───────┐  │
  │  │Hypervi│──┼─── eMMC (backup, QNX, early boot)
  │  │sor    │  │
  │  └───────┘  │
  └─────────────┘
  
Considerations:
  - Write endurance: eMMC/UFS have limited P/E cycles
  - Temperature: automotive grade (-40°C to 105°C)
  - Sudden power loss: must not corrupt FS
  - Read-only partitions: SquashFS, EROFS
  - OTA updates: A/B partition scheme
```

---

## Kernel Source References

```
Block device:
  include/linux/blk_types.h         ← struct bio, bio_vec
  include/linux/blkdev.h            ← struct block_device, request_queue
  block/blk-core.c                  ← Core block layer
  block/blk-mq.c                    ← Multi-queue block layer
  block/mq-deadline.c               ← mq-deadline scheduler
  block/bfq-iosched.c               ← BFQ scheduler

Storage drivers:
  drivers/scsi/                     ← SCSI/SATA/SAS
  drivers/nvme/                     ← NVMe
  drivers/mmc/                      ← eMMC/SD
  drivers/ufs/                      ← UFS
```

---

## Interview Questions

1. **What are the key performance characteristics of HDDs that file system designers must consider?**
2. **Explain the flash write asymmetry: why can't you overwrite a NAND page?**
3. **What is the FTL and what problems does it solve?**
4. **What is write amplification? How can the OS reduce it?**
5. **What does TRIM/discard do? Why is periodic fstrim preferred over mount -o discard?**
6. **What is a struct bio in the Linux kernel?**
7. **Compare eMMC vs UFS for automotive platforms.**
8. **Why does the kernel always use 512-byte sectors internally?**
9. **Name the Linux I/O schedulers and when each is appropriate.**
10. **Why does alignment between FS blocks, pages, and physical sectors matter?**

---

## Summary

- HDDs: seek time dominates → FS must optimize for locality, sequential access
- SSDs: no seek penalty, but erase-before-write constraint → FTL, GC, TRIM essential
- eMMC/UFS: embedded flash with integrated controller, automotive/mobile grade
- Block layer abstracts all storage as sectors → struct bio carries I/O through kernel
- I/O schedulers (mq-deadline, bfq, kyber, none) optimize request ordering
- Alignment between sectors, FS blocks, and pages prevents costly read-modify-write
- Automotive: UFS/eMMC, power-loss resilience, temperature range, write endurance

---

*Next: [Chapter 4 — Linux File System Architecture](Chapter_04_Linux_FS_Architecture.md)*
