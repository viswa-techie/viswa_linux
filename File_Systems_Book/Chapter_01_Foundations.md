# Chapter 1: Foundations of File Systems

## Learning Goals
- Understand what a file system is and why it exists
- Know the role of file systems in operating systems
- Grasp the file abstraction concept
- Navigate the Linux file system hierarchy
- Master core file system terminology
- Understand persistent storage fundamentals

---

## 1.1 What Is a File System?

A **file system** is the method and data structure an operating system uses to organize, store, retrieve, and manage data on a storage device.

```
Without a file system:             With a file system:
┌──────────────────────┐           ┌──────────────────────┐
│ Raw disk: just bytes │           │ /home/user/report.txt│
│ 0x00 0xFF 0xAB 0x12 │           │ /etc/config.yaml     │
│ Where does file A    │           │ /var/log/syslog      │
│ start? How big is it?│           │                      │
│ No structure at all! │           │ Names, sizes, perms  │
└──────────────────────┘           │ all tracked!         │
                                   └──────────────────────┘
```

A file system provides:
1. **Naming** — Human-readable names for data
2. **Organization** — Hierarchical directory structure
3. **Metadata** — Size, permissions, timestamps, ownership
4. **Allocation** — Which disk blocks belong to which file
5. **Reliability** — Data integrity via journaling, checksums
6. **Access control** — Who can read/write/execute

---

## 1.2 Role of File Systems in Operating Systems

```
┌─────────────────────────────────────────────────────┐
│                  User Applications                   │
│          open(), read(), write(), close()            │
└──────────────────────┬──────────────────────────────┘
                       │ System call interface
┌──────────────────────▼──────────────────────────────┐
│              Virtual File System (VFS)               │
│  Uniform interface regardless of underlying FS       │
└──────────────────────┬──────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   ┌────▼────┐   ┌─────▼────┐  ┌─────▼─────┐
   │  ext4   │   │   XFS    │  │  Btrfs    │  ...
   └────┬────┘   └─────┬────┘  └─────┬─────┘
        │              │              │
┌───────▼──────────────▼──────────────▼───────────────┐
│                 Block I/O Layer                       │
│          I/O schedulers, request queues              │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│              Device Drivers (SCSI, NVMe, MMC)        │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│           Physical Storage (HDD, SSD, eMMC)          │
└─────────────────────────────────────────────────────┘
```

The file system sits between user applications and raw storage hardware, providing:
- **Abstraction** — Applications see files, not disk blocks
- **Portability** — Same API (open/read/write) for any storage device
- **Sharing** — Multiple processes access the same file safely
- **Persistence** — Data survives power cycles

---

## 1.3 File Abstraction Concept

In Unix/Linux philosophy: **"Everything is a file."**

```
Type                │ Example              │ Description
────────────────────┼──────────────────────┼──────────────────────
Regular file        │ /etc/passwd          │ Data bytes on disk
Directory           │ /home/user/          │ Container of entries
Symbolic link       │ /usr/bin/python →    │ Pointer to another path
                    │   /usr/bin/python3   │
Block device        │ /dev/sda             │ Block-addressable device
Character device    │ /dev/ttyS0           │ Stream-oriented device
Named pipe (FIFO)   │ /tmp/myfifo          │ Inter-process communication
Socket              │ /var/run/docker.sock │ Network/IPC endpoint
```

Each "file" in the kernel is represented by:
- **inode** — Metadata (size, permissions, block pointers)
- **dentry** — Directory entry (name → inode mapping)
- **file** — Open file instance (position, mode, operations)

```
User sees:              Kernel sees:
  /home/user/doc.txt      dentry("doc.txt") → inode #12345
                           inode #12345 → {size: 4096, blocks: [100,101],
                                           uid: 1000, perms: 0644}
                           blocks [100, 101] → actual disk sectors
```

---

## 1.4 File System Hierarchy

Linux follows the **Filesystem Hierarchy Standard (FHS)**:

```
/                          Root of everything
├── bin/                   Essential user binaries (ls, cp, cat)
├── boot/                  Kernel image, bootloader files
├── dev/                   Device files (sda, tty, null)
├── etc/                   System configuration files
├── home/                  User home directories
├── lib/                   Shared libraries
├── mnt/                   Temporary mount points
├── opt/                   Optional/third-party software
├── proc/                  Process information (virtual)
├── root/                  Root user's home
├── run/                   Runtime variable data
├── sbin/                  System administration binaries
├── sys/                   Kernel objects (virtual, sysfs)
├── tmp/                   Temporary files
├── usr/                   User utilities and applications
│   ├── bin/               User commands
│   ├── lib/               Libraries
│   └── share/             Architecture-independent data
└── var/                   Variable data (logs, mail, spool)
    ├── log/               Log files
    └── tmp/               Preserved between reboots
```

In Linux, different directories can be different file systems mounted at different points:

```
/          → ext4 on /dev/sda1
/boot      → ext2 on /dev/sda2
/home      → xfs on /dev/sdb1
/proc      → procfs (virtual, no disk)
/sys       → sysfs (virtual, no disk)
/tmp       → tmpfs (RAM-based)
```

---

## 1.5 File System Terminology

```
Term                │ Definition
────────────────────┼──────────────────────────────────────────
Inode               │ Index node — metadata structure for a file (no name)
Dentry              │ Directory entry — maps name to inode
Superblock          │ Top-level metadata for an entire file system
Block               │ Minimum allocation unit (typically 4KB)
Sector              │ Minimum addressable unit on disk (512B or 4KB)
Extent              │ Contiguous range of blocks allocated to a file
Journal             │ Log of pending changes for crash recovery
Page cache          │ In-memory cache of file data (backed by VM pages)
Buffer cache        │ Cache for raw block device I/O (integrated with page cache)
Mount point         │ Directory where a file system is attached to the tree
VFS                 │ Virtual File System — kernel abstraction layer
Block group         │ Subdivision of disk in ext2/3/4 file systems
B-tree              │ Balanced tree for efficient lookup (XFS, Btrfs)
Copy-on-Write (CoW)│ Write to new location, then update pointer (Btrfs, ZFS)
Writeback           │ Deferred flushing of dirty pages to disk
Metadata            │ Data about data (size, timestamps, permissions)
Hard link           │ Multiple names pointing to the same inode
Soft link (symlink) │ File containing a path to another file
```

---

## 1.6 Persistent Storage Concepts

### Data Must Survive Power Loss

```
Volatile storage (loses data):     Non-volatile storage (keeps data):
  CPU registers                      HDD (magnetic platters)
  CPU cache (L1/L2/L3)              SSD (NAND flash)
  RAM (DRAM / SRAM)                  eMMC / UFS (embedded flash)
                                     NOR flash
                                     NVMe drives
```

### The Consistency Problem

```
When writing a file, multiple disk updates are needed:
  1. Allocate data blocks
  2. Write data to blocks
  3. Update inode (size, blocks, mtime)
  4. Update directory entry (if new file)
  5. Update block/inode bitmaps

If power fails between step 2 and step 3:
  → Data is on disk but inode doesn't point to it
  → File system is INCONSISTENT
  → Solution: Journaling (Chapter 12)
```

---

## 1.7 Data Organization in Storage Systems

### Block-Based Storage

```
Physical disk:
  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
  │Sec 0 │Sec 1 │Sec 2 │Sec 3 │Sec 4 │Sec 5 │Sec 6 │Sec 7 │
  └──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
  Each sector: 512 bytes (traditional) or 4096 bytes (Advanced Format)

File system block (4KB = 8 sectors of 512B):
  ┌────────────────────────────────────────────────────────┐
  │  Block 0 = Sectors 0-7                                 │
  ├────────────────────────────────────────────────────────┤
  │  Block 1 = Sectors 8-15                                │
  ├────────────────────────────────────────────────────────┤
  │  Block 2 = Sectors 16-23                               │
  └────────────────────────────────────────────────────────┘
```

### Logical Organization

```
Typical ext4 disk layout:
  ┌───────┬──────────┬──────────┬──────────┬──────────┬───────┐
  │ Boot  │ Super    │ Block    │ Inode    │ Inode    │ Data  │
  │ Sector│ Block    │ Group    │ Bitmap   │ Table    │ Blocks│
  │       │          │ Desc.    │ + Block  │          │       │
  │       │          │ Table    │ Bitmap   │          │       │
  └───────┴──────────┴──────────┴──────────┴──────────┴───────┘
  │       │                                            │
  │ 1 sec │◄————————— Block Group 0 ————————————————►  │
```

---

## 1.8 File System Types Overview

```
Category              │ Examples              │ Characteristics
──────────────────────┼───────────────────────┼──────────────────────
Disk-based            │ ext4, XFS, Btrfs      │ On block devices
Flash-based           │ JFFS2, UBIFS, F2FS    │ Flash-aware (wear leveling)
Network               │ NFS, SMB/CIFS, Ceph   │ Remote storage
Virtual/Pseudo        │ procfs, sysfs, tmpfs  │ No physical storage
Union/Overlay         │ OverlayFS, UnionFS    │ Merge multiple FS
FUSE                  │ sshfs, ntfs-3g        │ User-space implementation
Read-only             │ SquashFS, EROFS       │ Compressed, ROM
Copy-on-Write         │ Btrfs, ZFS, bcachefs  │ Never overwrite in-place
```

---

## Kernel Source References

```
File system core:
  fs/                               ← All file system code
  fs/filesystems.c                  ← File system registration
  include/linux/fs.h                ← Core FS structures
  include/linux/dcache.h            ← dentry cache
  include/linux/mount.h             ← Mount structures

VFS entry points:
  fs/open.c                         ← open() syscall
  fs/read_write.c                   ← read()/write() syscalls
  fs/namei.c                        ← Path resolution
```

---

## Interview Questions

1. **What is a file system? Why do we need one?**
2. **Explain the "everything is a file" philosophy in Linux.**
3. **What is an inode? How does it differ from a dentry?**
4. **What is the VFS layer? Why does Linux need it?**
5. **What happens at the file system level during a power failure?**
6. **What is the difference between a block and a sector?**
7. **Name the key differences between disk-based and flash-based file systems.**
8. **What is a mount point? Can you mount multiple file systems to the same tree?**
9. **What is the page cache? How does it relate to file systems?**
10. **Explain the Linux storage stack from application to physical disk.**

---

## Summary

- A file system organizes data on storage into named, accessible units (files)
- Linux VFS provides a uniform interface above all specific file system implementations
- The "everything is a file" abstraction extends to devices, pipes, sockets
- Key kernel objects: superblock (FS metadata), inode (file metadata), dentry (name mapping), file (open instance)
- Storage is block-based: sectors → blocks → files
- Power failures cause consistency problems → solved by journaling
- Linux supports disk-based, flash-based, network, and virtual file systems

---

*Next: [Chapter 2 — History and Evolution of File Systems](Chapter_02_History_Evolution.md)*
