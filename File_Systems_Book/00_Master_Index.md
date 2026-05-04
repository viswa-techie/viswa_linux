# Linux File Systems & VFS — Master Index

## Book Overview

A comprehensive 27-chapter study guide covering the Linux Virtual File System,
major filesystem implementations (ext4, XFS, Btrfs), page cache, journaling,
security, debugging, embedded/flash filesystems, and interview preparation.

**Target audience:** Linux kernel developers, embedded engineers, automotive software engineers
**Kernel versions:** 5.x — 6.x
**Prerequisites:** C programming, basic OS concepts, familiarity with Linux userspace

---

## Chapter Index

| # | Chapter | Key Topics |
|---|---------|------------|
| 01 | [Foundations](Chapter_01_Foundations.md) | Storage stack, "everything is a file", FHS, terminology |
| 02 | [History & Evolution](Chapter_02_History_Evolution.md) | Unix FS → FFS → ext2/3/4, journaling history, modern timeline |
| 03 | [Storage Hardware](Chapter_03_Storage_Hardware.md) | HDD, SSD, eMMC, UFS internals, FTL, TRIM, struct bio, I/O schedulers |
| 04 | [Linux FS Architecture](Chapter_04_Linux_FS_Architecture.md) | Storage stack diagram, VFS overview, 4 pillars, FS registration |
| 05 | [VFS Deep Dive](Chapter_05_VFS_Deep_Dive.md) | VFS polymorphism, object lifecycles, dcache, icache, fd table, RCU-walk |
| 06 | [VFS Data Structures](Chapter_06_VFS_Data_Structures.md) | struct super_block, inode, dentry, file, address_space — all fields |
| 07 | [FS Operations](Chapter_07_FS_Operations.md) | open/read/write/close/fsync paths, readahead, O_DIRECT, io_uring |
| 08 | [Directory Management](Chapter_08_Directory_Management.md) | ext4 HTree, path resolution, symlinks, hard links, inotify/fanotify |
| 09 | [Mounting](Chapter_09_Mounting.md) | mount() syscall, bind/overlay mounts, propagation, namespaces |
| 10 | [Disk Layout & Allocation](Chapter_10_Disk_Layout_Allocation.md) | ext4 block groups, extents, mballoc, delayed allocation, XFS AGs |
| 11 | [Linux File Systems](Chapter_11_Linux_File_Systems.md) | ext4, XFS, Btrfs features, comparison, mkfs, mount options |
| 12 | [Journaling](Chapter_12_Journaling.md) | jbd2 internals, transaction lifecycle, journal modes, crash recovery |
| 13 | [Special File Systems](Chapter_13_Special_File_Systems.md) | procfs, sysfs, tmpfs, debugfs, devtmpfs, configfs, tracefs |
| 14 | [Network File Systems](Chapter_14_Network_File_Systems.md) | NFS architecture, SMB/CIFS, cache coherency, 9P |
| 15 | [mmap](Chapter_15_Mmap.md) | mmap() internals, page fault handling, MAP_SHARED/PRIVATE, msync |
| 16 | [Page Cache](Chapter_16_Page_Cache.md) | Folios, xarray, readahead, writeback, dirty thresholds, LRU reclaim |
| 17 | [Performance](Chapter_17_Performance.md) | Benchmarking (fio), tuning (noatime, dirty_ratio), SSD/HDD/embedded |
| 18 | [Security](Chapter_18_Security.md) | Permissions, ACLs, capabilities, fscrypt, dm-crypt, SELinux, IMA, dm-verity |
| 19 | [Debugging Tools](Chapter_19_Debugging_Tools.md) | strace, debugfs, fsck, blktrace, iostat, ftrace, perf, eBPF |
| 20 | [Kernel Source Code](Chapter_20_Kernel_Source.md) | Source tree navigation, key files, code reading patterns |
| 21 | [Complete Flow Diagrams](Chapter_21_Flow_Diagrams.md) | Full open/read/write/fsync/mmap/create/delete/mount paths |
| 22 | [Important Diagrams](Chapter_22_Important_Diagrams.md) | VFS objects, ext4/XFS/Btrfs layout, page cache, writeback, LRU |
| 23 | [Glossary](Chapter_23_Glossary.md) | 100+ terms with definitions and chapter references |
| 24 | [OS Comparison](Chapter_24_OS_Comparison.md) | Linux vs Windows/macOS/QNX/RTOS filesystem architectures |
| 25 | [Embedded & Flash FS](Chapter_25_Embedded_FS.md) | JFFS2, UBIFS, SquashFS, EROFS, f2fs, automotive partition design |
| 26 | [Documentation & References](Chapter_26_Documentation.md) | Books, papers, online resources, mailing lists, kernel docs |
| 27 | [Interview Preparation](Chapter_27_Interview_Preparation.md) | Top 50 Q&A, whiteboard topics, scenario debugging, design questions |

---

## Reading Paths

### Path 1: Quick Interview Prep (4 chapters)
1. Chapter 05 — VFS Deep Dive (core concepts)
2. Chapter 07 — FS Operations (open/read/write paths)
3. Chapter 21 — Complete Flow Diagrams (visual reference)
4. Chapter 27 — Interview Preparation (Q&A, scenarios)

### Path 2: Kernel Developer Track (10 chapters)
1. Chapter 04 — Linux FS Architecture
2. Chapter 05 — VFS Deep Dive
3. Chapter 06 — VFS Data Structures
4. Chapter 07 — FS Operations
5. Chapter 10 — Disk Layout & Allocation
6. Chapter 12 — Journaling
7. Chapter 16 — Page Cache
8. Chapter 19 — Debugging Tools
9. Chapter 20 — Kernel Source Code
10. Chapter 21 — Complete Flow Diagrams

### Path 3: Automotive/Embedded Track (8 chapters)
1. Chapter 03 — Storage Hardware (eMMC, UFS)
2. Chapter 09 — Mounting (OverlayFS, namespaces)
3. Chapter 13 — Special File Systems (procfs, sysfs)
4. Chapter 17 — Performance (embedded tuning)
5. Chapter 18 — Security (dm-verity, fscrypt, SELinux)
6. Chapter 24 — OS Comparison (Linux vs QNX vs RTOS)
7. Chapter 25 — Embedded & Flash FS (UBIFS, SquashFS, f2fs)
8. Chapter 27 — Interview Preparation

### Path 4: Complete Sequential (all 27 chapters)
Read chapters 1 through 27 in order for the full learning experience.

---

## Key Kernel Source Files Quick Reference

| File | Purpose |
|------|---------|
| `include/linux/fs.h` | All VFS core structures |
| `fs/namei.c` | Path resolution |
| `fs/open.c` | open() syscall |
| `fs/read_write.c` | read()/write() syscalls |
| `fs/dcache.c` | Dentry cache |
| `fs/inode.c` | Inode lifecycle |
| `mm/filemap.c` | Page cache core |
| `mm/readahead.c` | Readahead algorithm |
| `mm/page-writeback.c` | Writeback/dirty management |
| `fs/ext4/super.c` | ext4 mount/registration |
| `fs/ext4/inode.c` | ext4 inode/writepages |
| `fs/ext4/extents.c` | ext4 extent tree |
| `block/blk-mq.c` | Multi-queue block layer |

---

## Essential Commands Quick Reference

```bash
# Info
df -h / df -i / mount / lsblk / blkid / findmnt / cat /proc/filesystems

# Create
mkfs.ext4 / mkfs.xfs / mkfs.btrfs / mkfs.f2fs / mksquashfs / mkfs.erofs

# Mount
mount -t ext4 -o noatime / mount --bind / mount -o remount,rw

# Tune
tune2fs -l / xfs_info / btrfs filesystem show

# Debug
strace -e trace=file / debugfs / e2fsck / xfs_repair / iostat -xz 1
blktrace / perf record / bpftrace / cat /proc/PID/stack

# Monitor
cat /proc/meminfo | grep Cache / echo 3 > /proc/sys/vm/drop_caches
cat /sys/fs/ext4/*/session_write_kbytes
cat /sys/block/mmcblk0/device/life_time
```

---

## Book Statistics

- **Chapters:** 27
- **Total files:** 28 (27 chapters + this index)
- **Coverage:** VFS internals, ext4/XFS/Btrfs, page cache, journaling, security, embedded FS, debugging, interview prep
- **Part of series:** Book 5 of the Linux Kernel Internals series
  - Book 1: [Device Drivers](../Device_Drivers_Book/)
  - Book 2: [Memory Management](../Memory_Management_Book/)
  - Book 3: [Process Scheduling](../Process_Scheduling_Book/)
  - Book 4: [Interrupts & Concurrency](../Interrupt_Concurrency_Book/)
  - Book 5: File Systems & VFS (this book)

---

*Generated as comprehensive study material for Linux kernel filesystem internals.*
