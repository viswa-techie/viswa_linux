# Chapter 2: History and Evolution of File Systems

## Learning Goals
- Trace file system evolution from early computing to modern Linux
- Understand why journaling was invented
- Know the progression from ext2 → ext3 → ext4
- Appreciate how flash storage changed file system design

---

## 2.1 Early File Systems in Computing

```
Timeline of file system evolution:

1950s   │ Tape-based sequential files (no random access)
1960s   │ IBM OS/360: VSAM, ISAM (indexed sequential)
1969    │ Unix file system (Ken Thompson, Dennis Ritchie)
        │ → Single flat inode table, simple block allocation
1970s   │ Unix FFS (Fast File System, BSD)
        │ → Cylinder groups, better locality
1980s   │ FAT12/FAT16 (MS-DOS), HFS (Macintosh)
1990s   │ ext2 (Linux), NTFS (Windows NT), HFS+ (Mac)
        │ → Journaling: ext3, XFS, JFS, ReiserFS
2000s   │ ext4, Btrfs, ZFS, UBIFS
        │ → CoW, checksums, flash-awareness
2010s   │ F2FS, EROFS, bcachefs, APFS
        │ → SSD/flash optimization, encryption
2020s   │ io_uring integration, folios in page cache
        │ → Continued ext4/Btrfs/XFS maturation
```

---

## 2.2 File Systems in Unix Systems

### Original Unix File System (1969-1970s)

```
Simple design:
  ┌───────────┬─────────────┬──────────────────────┐
  │ Boot block│ Superblock  │ Inode array │ Data   │
  └───────────┴─────────────┴─────────────┴────────┘

Problems:
  - All inodes at start of disk → long seeks to data
  - No locality: file data scattered randomly
  - Fragmentation over time → terrible performance
```

### Berkeley FFS (1984)

```
Key innovation: Cylinder Groups
  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
  │ Cylinder Group 0│  │ Cylinder Group 1│  │ Cylinder Group 2│
  │ ┌────┬────┬──┐ │  │ ┌────┬────┬──┐ │  │ ┌────┬────┬──┐ │
  │ │ SB │Ino │Dat│ │  │ │ SB │Ino │Dat│ │  │ │ SB │Ino │Dat│ │
  │ └────┴────┴──┘ │  │ └────┴────┴──┘ │  │ └────┴────┴──┘ │
  └────────────────┘  └────────────────┘  └────────────────┘

Improvements:
  - Inodes near their data → fewer seeks
  - Directory files near their contents
  - Larger blocks (4KB-8KB)
  - Rotational layout optimization

FFS became the basis for Linux's ext2.
```

---

## 2.3 Evolution of Linux File Systems

```
Linux File System Timeline:

1992 │ Minix FS      │ First FS for Linux, 14-char filenames, 64MB max
1993 │ ext (Extended) │ Overcame Minix limits, still basic
1993 │ ext2           │ Production-ready, FFS-inspired, block groups
     │                │ Standard Linux FS for years
2001 │ ext3           │ ext2 + journaling (backward compatible)
     │                │ Three journal modes: data, ordered, writeback
2006 │ ext4           │ Extents, 1EB support, delayed allocation,
     │                │ multiblock allocator, persistent preallocation
2007 │ Btrfs          │ Started by Oracle, CoW, snapshots, checksums,
     │                │ RAID, compression, subvolumes
2009 │ ext4 default   │ Became default FS in most Linux distributions
2012 │ F2FS           │ Samsung's Flash-Friendly FS, log-structured
2019 │ EROFS          │ Enhanced Read-Only FS, compressed, for Android
2023 │ bcachefs       │ Merged into mainline, CoW, checksums, encryption
```

### ext2 → ext3 → ext4 Progression

```
Feature            │ ext2          │ ext3          │ ext4
───────────────────┼───────────────┼───────────────┼───────────────
Journaling         │ No            │ Yes           │ Yes (+ checksum)
Max file size      │ 2 TB          │ 2 TB          │ 16 TB
Max FS size        │ 4 TB          │ 16 TB         │ 1 EB
Block mapping      │ Indirect      │ Indirect      │ Extents
Allocation         │ Bitmap        │ Bitmap        │ Multiblock + delayed
Timestamps         │ 1-second      │ 1-second      │ Nanosecond
Online resize      │ No            │ Limited       │ Yes
Backward compat.   │ —             │ Can mount as  │ Can mount as ext3
                   │               │ ext2          │
Dir indexing       │ Linear        │ HTree (hash)  │ HTree
Checksums          │ No            │ No            │ Journal + metadata
Encryption         │ No            │ No            │ fscrypt (inline)
```

---

## 2.4 Development of Modern File Systems

### XFS (Silicon Graphics, 1993 → Linux 2001)

```
Designed for large files and parallel I/O:
  - B+ tree for everything (inodes, extents, free space)
  - Allocation groups for parallelism
  - Delayed allocation
  - Excellent for large files and high-bandwidth workloads
  - Default FS for RHEL/CentOS

Strengths: Scalability, parallel writes, large files
Weaknesses: No shrink, recovery time for large volumes
```

### Btrfs (Oracle, 2007 → Mainline 2009)

```
Modern CoW (Copy-on-Write) file system:
  - Never overwrites data in place → atomic updates
  - Built-in RAID (0, 1, 5, 6, 10)
  - Snapshots and subvolumes
  - Online defragmentation
  - Data + metadata checksums (CRC32C)
  - Compression (zlib, lzo, zstd)
  - Send/receive for incremental backup

Strengths: Features, data integrity, flexibility
Weaknesses: RAID 5/6 stability concerns (write hole)
```

### ZFS (Sun Microsystems, 2005 → OpenZFS)

```
Not in mainline Linux (CDDL license conflict):
  - 128-bit addressing
  - Pooled storage model
  - End-to-end checksums
  - Copy-on-write + snapshots
  - Built-in volume management
  - ARC (Adaptive Replacement Cache)

Available via OpenZFS kernel module (DKMS).
```

---

## 2.5 Journaling File Systems

### The Problem

```
Writing a new file requires multiple disk updates:
  1. Allocate inode           ← Update inode bitmap
  2. Initialize inode         ← Write inode fields
  3. Allocate data blocks     ← Update block bitmap
  4. Write data               ← Write data blocks
  5. Update directory         ← Add dentry to parent dir
  6. Update parent inode      ← Modify mtime

Power failure between ANY two steps → INCONSISTENT file system!

Before journaling: Run fsck at boot → scan ENTIRE disk → SLOW (hours for large FS)
With journaling: Replay journal → consistent in seconds
```

### How Journaling Works

```
Normal operation:
  1. Write changes to journal FIRST (sequential, fast)
  2. Write changes to actual disk locations
  3. Mark journal entry as committed

         Journal Area              Main Disk
  ┌──────────────────────┐  ┌──────────────────────┐
  │ TX 1: write inode 42 │  │                      │
  │ TX 1: write block 100│──│──► Apply changes     │
  │ TX 1: COMMIT         │  │    to disk           │
  │ TX 2: ...            │  │                      │
  └──────────────────────┘  └──────────────────────┘

Crash recovery:
  - Incomplete transactions → discard (never reached disk)
  - Complete transactions → replay to ensure disk is updated
  - Recovery in seconds, not hours
```

### Journal Modes (ext3/ext4)

```
Mode          │ Journals            │ Safety   │ Performance
──────────────┼─────────────────────┼──────────┼────────────
data=journal  │ Metadata + Data     │ Highest  │ Slowest
data=ordered  │ Metadata only       │ Good     │ Medium
              │ (data written first)│          │ (DEFAULT)
data=writeback│ Metadata only       │ Lowest   │ Fastest
              │ (data order not     │          │
              │  guaranteed)        │          │
```

---

## Kernel Source References

```
File system implementations:
  fs/ext2/                          ← ext2 implementation
  fs/ext4/                          ← ext4 implementation
  fs/xfs/                           ← XFS implementation
  fs/btrfs/                         ← Btrfs implementation
  fs/f2fs/                          ← F2FS implementation

Journaling:
  fs/jbd2/                          ← Journaling Block Device (ext3/ext4)
  fs/xfs/xfs_log.c                 ← XFS write-ahead log
```

---

## Interview Questions

1. **What problems did the original Unix file system have?**
2. **How did FFS (Berkeley Fast File System) improve locality?**
3. **Trace the evolution from ext2 → ext3 → ext4. What did each add?**
4. **What is journaling? What problem does it solve?**
5. **Compare the three journal modes in ext3/ext4.**
6. **What is Copy-on-Write? Which file systems use it?**
7. **Why is XFS good for large files?**
8. **Why isn't ZFS in the Linux mainline kernel?**
9. **What is F2FS designed for? How does it differ from ext4?**
10. **What changed in the 2010s-2020s era of file system design?**

---

## Summary

- Unix FFS introduced locality with cylinder groups → basis for Linux ext2
- ext2 → ext3 (journaling) → ext4 (extents, delayed alloc, nanosecond timestamps)
- Journaling prevents fsck full-scan after crashes by logging transactions first
- Modern FS: XFS (scalability), Btrfs (CoW + integrity), F2FS (flash-optimized)
- Flash storage drove new FS designs (F2FS, EROFS, UBIFS) in 2010s
- Trend: checksums, encryption, compression built into the file system

---

*Next: [Chapter 3 — Storage Hardware Architecture](Chapter_03_Storage_Hardware.md)*
