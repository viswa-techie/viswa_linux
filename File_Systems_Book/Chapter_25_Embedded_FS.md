# Chapter 25: Embedded & Flash File Systems

## Learning Goals
- Understand flash storage challenges: wear leveling, write amplification, bad blocks
- Master JFFS2, UBIFS, YAFFS2 for raw NAND flash
- Know SquashFS and EROFS for read-only compressed filesystems
- Understand f2fs design for managed flash (SSD/eMMC/UFS)
- Apply embedded FS knowledge to automotive and IoT systems

---

## 25.1 Flash Memory Challenges

```
Flash storage fundamentals:

NOR Flash:
  ┌────────────────────────────────────────┐
  │ Byte-addressable, execute-in-place     │
  │ Read: fast (random byte access)        │
  │ Write: slow (one byte at a time)       │
  │ Erase: very slow (whole sector ~64KB)  │
  │ Endurance: ~100,000 erase cycles       │
  │ Use: bootloaders, firmware, small data │
  └────────────────────────────────────────┘

NAND Flash:
  ┌────────────────────────────────────────┐
  │ Page-addressable (read/write by page)  │
  │ Read: fast (~25μs per page)            │
  │ Write: medium (~200μs per page)        │
  │ Erase: slow (~1-2ms per block)         │
  │ Endurance: 1,000-100,000 cycles       │
  │ Use: storage (SSD, eMMC, UFS)          │
  └────────────────────────────────────────┘

Key constraints:
  1. Write amplification: Must erase before write
     Erase unit (block) >> Write unit (page)
     Block = 64-256 pages, Page = 2-16 KB

  2. Wear leveling: Cells degrade with erase cycles
     Must distribute erases evenly across flash

  3. Read disturb: Reading a page can corrupt neighbors
     Periodic scrubbing needed

  4. Bad blocks: Flash cells die over time
     FS must handle bad block management

  ┌──────────────────────────────────────────────────────┐
  │ Flash hierarchy:                                      │
  │                                                      │
  │  ┌─────────────────────────────────────┐             │
  │  │ Chip                                 │             │
  │  │ ┌──────────────────────────────┐    │             │
  │  │ │ Die                           │    │             │
  │  │ │ ┌───────────────────────┐    │    │             │
  │  │ │ │ Plane                  │    │    │             │
  │  │ │ │ ┌────────────────┐   │    │    │             │
  │  │ │ │ │ Block (erase)  │   │    │    │             │
  │  │ │ │ │ ┌──────────┐  │   │    │    │             │
  │  │ │ │ │ │ Page (r/w)│  │   │    │    │             │
  │  │ │ │ │ │ 4-16 KB  │  │   │    │    │             │
  │  │ │ │ │ └──────────┘  │   │    │    │             │
  │  │ │ │ │ (64-256 pages)│   │    │    │             │
  │  │ │ │ └────────────────┘   │    │    │             │
  │  │ │ └───────────────────────┘    │    │             │
  │  │ └──────────────────────────────┘    │             │
  │  └─────────────────────────────────────┘             │
  └──────────────────────────────────────────────────────┘
```

---

## 25.2 Raw Flash vs Managed Flash

```
┌────────────────────────────────────────────────────────────────────┐
│ Raw NAND Flash                    │ Managed Flash (eMMC/UFS/SSD)   │
│ ┌──────────────┐                  │ ┌──────────────┐               │
│ │ MTD Subsystem │                  │ │ Block Device  │               │
│ │ (Memory Tech  │                  │ │ (looks like   │               │
│ │  Devices)     │                  │ │  a hard disk) │               │
│ └──────┬───────┘                  │ └──────┬───────┘               │
│        │                          │        │                        │
│ ┌──────▼───────┐                  │ ┌──────▼───────┐               │
│ │ JFFS2 / UBIFS│ ← Flash-aware   │ │ ext4/XFS/f2fs│ ← Block FS   │
│ │ YAFFS2       │   filesystems   │ │              │               │
│ └──────┬───────┘                  │ └──────┬───────┘               │
│        │                          │        │                        │
│ ┌──────▼───────┐                  │ ┌──────▼───────┐               │
│ │ NAND Driver  │ ← direct flash  │ │ FTL (firmware)│ ← handles    │
│ │ (MTD)        │   access        │ │ Wear leveling │   all flash   │
│ └──────┬───────┘                  │ │ Bad blocks    │   management  │
│        │                          │ │ GC            │               │
│        ▼                          │ └──────┬───────┘               │
│   Raw NAND chip                   │        ▼                        │
│   (bare flash)                    │   NAND chip(s)                  │
│                                   │   (inside eMMC/UFS/SSD)         │
└────────────────────────────────────────────────────────────────────┘

Linux MTD subsystem:
  /dev/mtd0   ← character device (byte access)
  /dev/mtdblock0 ← block device emulation (read-only recommended)

  $ cat /proc/mtd
  dev:    size   erasesize  name
  mtd0: 00100000 00020000 "bootloader"
  mtd1: 00e00000 00020000 "kernel"
  mtd2: 0f000000 00020000 "rootfs"
```

---

## 25.3 JFFS2 (Journaling Flash File System 2)

```
JFFS2 — Log-structured FS for raw NOR/NAND flash:

Design:
  ┌──────────────────────────────────────────────────────┐
  │ Flash is written sequentially, like a circular log    │
  │                                                      │
  │ ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐  │
  │ │Node │Node │Node │Node │Node │CLEAN│CLEAN│DIRTY│  │
  │ │Data │Data │Dirent│Data │Data │     │     │     │  │
  │ │v2  │v3  │     │v1  │v4  │     │     │(GC) │  │
  │ └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘  │
  │   ↑ write pointer moves forward →                    │
  │                                                      │
  │ Node types:                                          │
  │   JFFS2_NODETYPE_DIRENT     — directory entry        │
  │   JFFS2_NODETYPE_INODE      — file data/metadata     │
  │   JFFS2_NODETYPE_CLEANMARKER— erased block marker    │
  │                                                      │
  │ Garbage Collection:                                   │
  │   Old/obsolete nodes → erase block → rewrite live data│
  └──────────────────────────────────────────────────────┘

Characteristics:
  ✓ Wear leveling (GC moves data across flash)
  ✓ Power-loss safe (log-structured, no in-place updates)
  ✓ Compression (zlib, lzo)
  ✗ Slow mount (must scan entire flash to build FS tree)
  ✗ High RAM usage (keeps full inode map in RAM)
  ✗ GC can cause write latency spikes
  ✗ Not suitable for large partitions (>256MB)

Usage:
  mount -t jffs2 /dev/mtdblock2 /mnt
  mkfs.jffs2 -r rootfs_dir/ -o rootfs.jffs2 -e 128KiB
```

---

## 25.4 UBIFS (Unsorted Block Image File System)

```
UBIFS — Modern flash FS for raw NAND (via UBI):

Architecture:
  ┌──────────────────────────────────────────────────────┐
  │ UBIFS                                                │
  │  └─→ UBI (Unsorted Block Images) ← Volume manager   │
  │       └─→ MTD (Memory Technology Devices)            │
  │            └─→ Raw NAND flash                        │
  └──────────────────────────────────────────────────────┘

  UBI handles:
    - Wear leveling (across entire flash)
    - Bad block management
    - Logical-to-physical eraseblock mapping
    - Atomic eraseblock writes

  UBIFS handles:
    - File/directory management
    - Indexing (wandering B-tree)
    - Journaling (write-ahead log)
    - Compression (zlib, lzo, zstd)
    - Write-back caching

UBIFS on-flash structure:
  ┌──────────────────────────────────────────────────────┐
  │ Superblock area  │ Master area │ Journal │ Main area │
  │ (fixed location) │ (2 copies)  │ (log +  │ (data +   │
  │                  │             │  buds)  │  index)   │
  └──────────────────────────────────────────────────────┘

  Wandering B-tree:
    Index nodes are Copy-on-Write → never updated in place
    On commit: new index written to clean space
    Old index locations become garbage → GC reclaims

    ┌─────┐    After update:    ┌─────┐
    │Root │                     │Root'│ (new location)
    └──┬──┘                     └──┬──┘
       │                           │
    ┌──┴──┐                     ┌──┴──┐
    │Node │                     │Node'│ (new location)
    └──┬──┘                     └──┬──┘
       │                           │
    ┌──┴──┐                     ┌──┴──┐
    │Leaf │                     │Leaf'│ (new data)
    └─────┘                     └─────┘
```

### UBIFS vs JFFS2

```
┌────────────────────┬──────────────────────┬──────────────────────┐
│ Feature             │ JFFS2                 │ UBIFS                 │
├────────────────────┼──────────────────────┼──────────────────────┤
│ Volume manager     │ Direct on MTD        │ Requires UBI layer   │
│ Mount time         │ Slow (full scan)     │ Fast (log replay)    │
│ RAM usage          │ High (full inode map)│ Low (on-demand index)│
│ Max partition      │ ~256MB practical     │ Multiple GB+         │
│ Write-back cache   │ No                   │ Yes (faster writes)  │
│ Compression        │ zlib, lzo            │ zlib, lzo, zstd      │
│ Wear leveling      │ FS-level GC          │ UBI-level (better)   │
│ Bad block handling │ FS-level             │ UBI-level (better)   │
│ Index structure    │ In-RAM inode map     │ On-flash B-tree      │
│ Recommendation     │ Small NOR flash      │ Any NAND > 32MB      │
└────────────────────┴──────────────────────┴──────────────────────┘
```

### UBIFS Commands

```bash
# Create UBI volume
$ ubiformat /dev/mtd2
$ ubiattach -m 2
$ ubimkvol /dev/ubi0 -N rootfs -m   # Use maximum size

# Mount UBIFS
$ mount -t ubifs ubi0:rootfs /mnt

# Create UBIFS image for flashing
$ mkfs.ubifs -r rootfs_dir/ -m 4096 -e 253952 -c 2048 -o rootfs.ubifs
#  -m: minimum I/O size (page size)
#  -e: logical erase block size
#  -c: maximum logical erase block count
```

---

## 25.5 SquashFS (Read-Only Compressed)

```
SquashFS — Compressed read-only filesystem:

Structure:
  ┌────────────────────────────────────────────────────────┐
  │ ┌──────────┬──────────┬──────────┬──────────┬────────┐ │
  │ │Superblock│Compressed│Compressed│ Inode    │Fragment │ │
  │ │          │Data      │Metadata  │ Table    │Table    │ │
  │ │          │Blocks    │          │          │         │ │
  │ └──────────┴──────────┴──────────┴──────────┴────────┘ │
  │                                                        │
  │ Features:                                              │
  │   - Files compressed with gzip/lzo/lz4/xz/zstd        │
  │   - Block-level compression (128KB blocks default)     │
  │   - Deduplication of identical blocks                  │
  │   - Metadata compressed separately                     │
  │   - 32-bit UIDs/GIDs, xattrs                           │
  │   - ~2-3x compression ratio typical                    │
  └────────────────────────────────────────────────────────┘

Usage:
  # Create SquashFS image
  $ mksquashfs rootfs_dir/ rootfs.sqsh -comp lz4 -b 256K

  # Mount
  $ mount -t squashfs rootfs.sqsh /mnt -o loop

  # Or flash to partition and mount
  $ dd if=rootfs.sqsh of=/dev/mmcblk0p3
  $ mount -t squashfs /dev/mmcblk0p3 /system

Automotive use:
  - /system partition: SquashFS + dm-verity
  - Compressed: smaller flash footprint
  - Read-only: no corruption risk, no fsck needed
  - dm-verity: integrity verification at boot
  - Fast boot: LZ4 decompression is very fast (~3 GB/s)
```

---

## 25.6 EROFS (Enhanced Read-Only File System)

```
EROFS — Next-generation read-only FS (replacing SquashFS in some uses):

  ┌────────────────────────────────────────────────────────┐
  │ EROFS advantages over SquashFS:                        │
  │   - Fixed-size output blocks → random access friendly  │
  │   - Smaller metadata overhead                          │
  │   - Faster random read (no need to decompress full     │
  │     block for small access)                            │
  │   - In mainline kernel (fs/erofs/)                     │
  │   - Inline data for small files                        │
  │   - Chunk-based deduplication                          │
  │   - LZ4/LZMA/deflate compression                      │
  └────────────────────────────────────────────────────────┘

  Used by:
    - Android (system partition in some devices)
    - Embedded Linux (read-only root filesystem)

  # Create EROFS image
  $ mkfs.erofs -zlz4hc rootfs.erofs rootfs_dir/

  # Mount
  $ mount -t erofs rootfs.erofs /mnt -o loop
```

---

## 25.7 f2fs (Flash-Friendly File System)

```
f2fs — Designed for managed flash (eMMC, UFS, SSD):

Architecture (Log-Structured + Multi-Head Logging):
  ┌──────────────────────────────────────────────────────────┐
  │ f2fs on-disk layout:                                      │
  │                                                          │
  │ ┌────────┬─────┬──────┬──────┬──────┬─────────────────┐ │
  │ │Super-  │ CP  │ SIT  │ NAT  │ SSA  │ Main Area       │ │
  │ │block   │area │area  │area  │area  │(data + nodes)   │ │
  │ └────────┴─────┴──────┴──────┴──────┴─────────────────┘ │
  │                                                          │
  │ CP  = Checkpoint (consistency point, 2 copies)           │
  │ SIT = Segment Information Table (valid block bitmap)     │
  │ NAT = Node Address Table (inode → physical location)     │
  │ SSA = Segment Summary Area (reverse mapping)             │
  │                                                          │
  │ Multi-head logging:                                      │
  │   HOT data  → log 1  (frequently updated small files)    │
  │   WARM data → log 2  (normal files)                      │
  │   COLD data → log 3  (multimedia, archives)              │
  │   HOT node  → log 4  (directory inodes)                  │
  │   WARM node → log 5  (file inodes)                       │
  │   COLD node → log 6  (indirect nodes)                    │
  │                                                          │
  │ Separating hot/cold data reduces GC overhead             │
  └──────────────────────────────────────────────────────────┘

Features:
  ✓ Designed for flash write patterns (append-mostly)
  ✓ Multi-head logging separates hot/warm/cold data
  ✓ Adaptive logging: normal mode + in-place update mode
  ✓ Roll-back + roll-forward recovery (CP + fsync logs)
  ✓ Inline data, inline directories
  ✓ Garbage collection optimized for flash
  ✓ TRIM/discard support
  ✓ fscrypt encryption support (Android FBE)
  ✓ Compression (LZ4, zstd) since kernel 5.7

Android usage:
  /data partition often uses f2fs (Samsung, Google Pixel)
  Combined with fscrypt for File-Based Encryption
```

---

## 25.8 Embedded FS Selection Guide

```
┌────────────────────────┬────────────────────────────────────────────┐
│ Storage Type            │ Recommended FS                             │
├────────────────────────┼────────────────────────────────────────────┤
│ Raw NOR flash (small)  │ JFFS2 (< 64MB), cramfs (read-only)       │
│ Raw NAND flash         │ UBIFS over UBI                             │
│ eMMC / UFS (writable)  │ ext4 or f2fs                               │
│ eMMC / UFS (read-only) │ SquashFS or EROFS + dm-verity             │
│ SD card (portable)     │ FAT32/exFAT (compatibility) or ext4       │
│ SSD (server/desktop)   │ ext4, XFS, Btrfs                          │
│ MCU flash (< 1MB)      │ LittleFS (not Linux, bare-metal)          │
│ Network boot           │ NFS root + SquashFS overlays              │
│ Container rootfs       │ OverlayFS over SquashFS                   │
└────────────────────────┴────────────────────────────────────────────┘

Automotive partition layout example:
  ┌───────────────────────────────────────────────────────────────┐
  │ Boot: raw image (bootloader, no FS)                          │
  │ Kernel: raw image (zImage/Image.gz)                          │
  │ System A: SquashFS + dm-verity (read-only, verified)         │
  │ System B: SquashFS + dm-verity (A/B OTA)                     │
  │ Vendor: ext4 or EROFS + dm-verity (read-only)                │
  │ Data: ext4 + fscrypt (writable, encrypted)                   │
  │   or: f2fs + fscrypt (better flash performance)              │
  │ Persist: ext4 (small, writable, survives factory reset)      │
  │ Misc: raw (bootloader control, no FS)                        │
  └───────────────────────────────────────────────────────────────┘
```

---

## 25.9 Write Endurance Monitoring

```bash
# eMMC life monitoring
$ cat /sys/block/mmcblk0/device/life_time
0x02 0x02
# Values: 0x01=0-10%, 0x02=10-20%, ..., 0x0A=90-100%, 0x0B=exceeded

# eMMC pre-EOL info
$ cat /sys/block/mmcblk0/device/pre_eol_info
0x01
# 0x01=normal, 0x02=warning (80% consumed), 0x03=urgent

# UFS health
$ cat /sys/block/sda/device/health_descriptor/life_time_estimation_a
0x01

# SMART data for SSDs
$ smartctl -a /dev/nvme0n1

# Strategies to extend flash life:
1. Minimize writes:
   - noatime on all mounts
   - Logs to tmpfs, periodically sync to flash
   - Use SquashFS for read-only partitions

2. Write amplification reduction:
   - f2fs (flash-optimized write patterns)
   - Proper TRIM/discard support
   - Avoid small random writes (batching)

3. Monitoring:
   - Periodic check of life_time sysfs
   - Alert when pre_eol_info reaches 0x02
   - Log write statistics for trend analysis
```

---

## 25.10 Power-Loss Protection

```
Power loss is critical in embedded/automotive:

Problem:
  ┌─────────────────────────────────────────────┐
  │ Writing file:                                │
  │   1. Allocate blocks        ← done          │
  │   2. Write data blocks      ← in progress   │
  │   3. Update inode metadata  ← not yet        │
  │                                              │
  │   POWER LOSS HERE!                           │
  │                                              │
  │ Result: allocated blocks with partial data,  │
  │ inode still points to old data or garbage    │
  └─────────────────────────────────────────────┘

Solutions by FS:
  ext4 (data=journal):  All changes in journal first → replay on boot
  ext4 (data=ordered):  Data before metadata → consistent but may lose last write
  f2fs:                 Checkpoint + roll-forward from fsync log
  UBIFS:                Wandering tree (CoW) + journal → always consistent
  JFFS2:                Log-structured → always consistent (GC handles cleanup)
  Btrfs:                Full CoW → always consistent
  SquashFS:             Read-only → cannot be corrupted by writes

Automotive safeguards:
  1. A/B partitions: if new image fails, boot old one
  2. dm-verity: detect corruption at read time
  3. Battery backup or capacitor: complete pending writes
  4. Atomic write API (if supported): all-or-nothing
  5. /persist partition: small ext4 with data=journal for critical data
```

---

## Kernel Source References

```
Embedded filesystem code:
  fs/jffs2/           ← JFFS2 implementation
  fs/ubifs/           ← UBIFS implementation
  fs/squashfs/        ← SquashFS implementation
  fs/erofs/           ← EROFS implementation
  fs/f2fs/            ← f2fs implementation

MTD subsystem:
  drivers/mtd/        ← MTD core and NAND drivers
  drivers/mtd/ubi/    ← UBI volume manager

Flash storage drivers:
  drivers/mmc/        ← eMMC/SD drivers
  drivers/scsi/ufs/   ← UFS drivers
  drivers/nvme/       ← NVMe SSD drivers
```

---

## Interview Questions

1. **What is the difference between raw NAND flash and managed flash (eMMC)?**
2. **Compare JFFS2 and UBIFS. When would you use each?**
3. **How does UBIFS achieve power-loss protection?**
4. **What is SquashFS? Why is it used for automotive system partitions?**
5. **Explain f2fs multi-head logging. How does it reduce GC overhead?**
6. **How would you design a filesystem layout for an automotive head unit?**
7. **What is write amplification? How do flash FSes mitigate it?**
8. **How do you monitor eMMC/UFS flash wear in Linux?**
9. **Compare SquashFS and EROFS. When to use EROFS?**
10. **What power-loss protection mechanisms exist across different FSes?**

---

## Summary

- Flash challenges: erase-before-write, wear leveling, bad blocks, write amplification
- Raw flash stack: MTD → UBI → UBIFS (modern) or JFFS2 (legacy/NOR)
- Managed flash: eMMC/UFS/SSD have built-in FTL → use ext4 or f2fs
- Read-only: SquashFS (compressed, proven) or EROFS (faster random access)
- f2fs: flash-optimized with multi-head logging, hot/cold separation, designed for Android
- Automotive: SquashFS+dm-verity for system, f2fs/ext4+fscrypt for data, monitor flash life
- Power-loss: journal (ext4), CoW (UBIFS/Btrfs), checkpoint (f2fs), or read-only (SquashFS)
- Write endurance: minimize writes (noatime, tmpfs for logs), monitor life_time sysfs

---

*Next: [Chapter 26 — Documentation & References](Chapter_26_Documentation.md)*
