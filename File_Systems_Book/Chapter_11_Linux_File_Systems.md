# Chapter 11: Linux File Systems — ext2/ext3/ext4, XFS, Btrfs

## Learning Goals
- Know the key features and internals of each major Linux file system
- Understand when to choose one over another
- Master ext4 features: extents, flex_bg, bigalloc, inline data
- Know XFS strengths: scalability, reflink, DAX
- Know Btrfs capabilities: snapshots, RAID, checksums, send/receive

---

## 11.1 ext4 — The Default Linux File System

### Key Features

```
Feature                  │ Details
─────────────────────────┼──────────────────────────────────────
Extent-based allocation  │ Up to 128 MB per extent (vs 4KB indirect blocks)
Delayed allocation       │ Allocate blocks at writeback, not write()
Multi-block allocator    │ Buddy system for contiguous allocation
Journal checksumming     │ CRC32C on journal transactions
Metadata checksums       │ CRC32C on inodes, dir entries, extent tree
Nanosecond timestamps    │ Stored in extra inode fields
Inline data              │ Small files stored in inode itself
Large file support       │ Up to 16 TB (with 4KB blocks)
Large FS support         │ Up to 1 EB (exabyte)
Online resize            │ Grow FS without unmounting
Encryption (fscrypt)     │ Per-file encryption (AES-256)
fastcommit               │ Faster journal commits
Casefold                 │ Case-insensitive directories (Android)
```

### ext4 Feature Flags

```bash
# View features of an ext4 filesystem
$ sudo tune2fs -l /dev/sda1 | grep features
Filesystem features: has_journal ext_attr resize_inode dir_index
                     filetype extent 64bit flex_bg sparse_super2
                     large_file huge_file dir_nlink extra_isize
                     metadata_csum

Key features:
  has_journal    → ext3/ext4 journaling enabled
  extent         → Extent-based block mapping (not indirect)
  flex_bg        → Flexible block groups (grouped metadata)
  64bit          → 64-bit block numbers (FS > 16 TB)
  metadata_csum  → CRC32C metadata checksums
  dir_index      → HTree hashed directory index
  encrypt        → fscrypt support
  casefold       → Case-insensitive directories
  inline_data    → Small files in inode
```

### flex_bg (Flexible Block Groups)

```
Traditional: Each block group has its own bitmaps + inode table
flex_bg: Groups bitmaps and inode tables together

Without flex_bg (16 block groups):
  BG0: [SB|GDT|BB|IB|IT......|DATA..........]
  BG1: [         BB|IB|IT......|DATA..........]
  BG2: [         BB|IB|IT......|DATA..........]
  ...

With flex_bg (flex_bg_size=16):
  BG0: [SB|GDT|BB0|BB1|...|BB15|IB0|IB1|...|IB15|IT0|IT1|...|IT15]
  BG1: [DATA...................................................]
  BG2: [DATA...................................................]
  ...
  BG15:[DATA...................................................]

Benefits:
  - All metadata for 16 groups in one location
  - Reduces seeks for metadata operations
  - Larger contiguous data regions
  - Better SSD performance (sequential metadata writes)
```

### ext4 Inline Data

```
Small files can be stored entirely within the inode:

Standard inode (256 bytes):
  ┌──────────────────────────────┐
  │ inode fixed fields (128 B)   │
  │ i_block[15] (60 B)          │ ← Can hold ~60 bytes of data
  │ extra fields (68 B)         │ ← With extended attrs: more space
  └──────────────────────────────┘

With inline_data feature:
  - Files ≤ 60 bytes: stored in i_block[] area
  - Files ≤ ~256 bytes: stored in inode + xattr area
  - No data block allocated → saves space AND I/O

  $ mkfs.ext4 -O inline_data /dev/sda1
  
  # Check if file is inline:
  $ sudo debugfs -R "stat <inode_num>" /dev/sda1
  # Look for: "inline data"
```

---

## 11.2 XFS — Scalable High-Performance FS

### Key Features

```
Feature                  │ Details
─────────────────────────┼──────────────────────────────────────
B+tree everything        │ Inodes, extents, directories, free space
Allocation Groups        │ Parallel allocation (1 lock per AG)
Delayed allocation       │ Like ext4, delays block allocation
Speculative prealloc     │ Preallocates beyond file end
Write-ahead logging      │ Journal for metadata consistency
Reflink (CoW copies)     │ Instant file copies, shared extents
Online defrag            │ xfs_fsr
Online grow              │ xfs_growfs (no shrink!)
DAX support              │ Direct Access for persistent memory
Reverse mapping          │ Maps physical→logical (for scrub/repair)
Realtime subvolume       │ Dedicated area for realtime I/O
Max file size            │ 8 EB
Max FS size              │ 8 EB
```

### XFS Internal Structure

```
XFS Allocation Group Layout:
  ┌────────────────────────────────────────────────────────────┐
  │                    Allocation Group                         │
  │                                                            │
  │  ┌──────────────────┐                                     │
  │  │ AG Superblock    │  Free Space B+tree (by block #)     │
  │  │ AG Free Space    │──────────────────────────┐          │
  │  │ AG Inode Info    │  Free Space B+tree (by size) ──┐    │
  │  └──────────────────┘                           │    │    │
  │                                                 │    │    │
  │  ┌──────────────────┐                           │    │    │
  │  │ Inode B+tree     │  Maps inode # → disk loc  │    │    │
  │  └──────────────────┘                           │    │    │
  │                                                 │    │    │
  │  Data / Free blocks ◄──────────────────────────┘────┘    │
  └────────────────────────────────────────────────────────────┘

Per-inode extent map:
  Small files: inline in data fork of inode
  Large files: B+tree of extent records
    [logical_offset, physical_block, length, flag]
```

### XFS Reflink (Copy-on-Write Copies)

```
Reflink: create a copy that shares physical blocks

  $ cp --reflink=always source.img dest.img
  # Instant! No data copied. Both files share same extents.

  Before write:
    source.img: Extent A → [blocks 1000-2000]
    dest.img:   Extent A → [blocks 1000-2000]  (shared, refcount=2)

  After writing to dest.img:
    source.img: Extent A → [blocks 1000-2000]  (refcount=1)
    dest.img:   Extent B → [blocks 3000-3500]  (new, CoW'd)
                Extent A → [blocks 1500-2000]  (still shared for unchanged range)

  Use cases:
    - VM disk image cloning
    - Backup with deduplication
    - Build system (fast copies of source trees)
```

---

## 11.3 Btrfs — Modern Copy-on-Write FS

### Key Features

```
Feature                  │ Details
─────────────────────────┼──────────────────────────────────────
Copy-on-Write            │ Never overwrites data in place
Snapshots                │ Instant read-only or read-write snapshots
Subvolumes               │ Independent filesystem trees in one FS
Built-in RAID            │ RAID 0, 1, 10, 5, 6 (5/6 unstable)
Data checksums           │ CRC32C on all data (detect bit rot)
Metadata checksums       │ CRC32C on all metadata
Compression              │ zlib, lzo, zstd (transparent)
Send/Receive             │ Incremental backup between snapshots
Online defrag            │ btrfs filesystem defragment
Online resize            │ Grow and shrink
Deduplication            │ Offline dedup tools (duperemove)
Quotas                   │ Per-subvolume quota groups (qgroups)
```

### Btrfs Architecture

```
Btrfs uses a forest of B-trees:

  ┌──────────────────────────────────────────────────────┐
  │  Superblock (fixed locations: 64KB, 64MB, 256GB)     │
  │  Points to: Tree of tree roots                       │
  └──────────────┬───────────────────────────────────────┘
                 │
  ┌──────────────▼───────────────────────────────────────┐
  │  Root Tree (holds roots of all other trees)          │
  │  ├── FS Tree root (default subvolume)                │
  │  ├── FS Tree root (subvolume "home")                │
  │  ├── Extent Tree root                               │
  │  ├── Chunk Tree root                                │
  │  ├── Device Tree root                               │
  │  ├── Checksum Tree root                             │
  │  └── ...                                            │
  └──────────────────────────────────────────────────────┘

  FS Tree: inodes, dir entries, file extents, xattrs
  Extent Tree: block allocation tracking, back-references
  Chunk Tree: logical → physical address mapping
  Checksum Tree: CRC32C for every data block
```

### Btrfs Snapshots

```
Cheap snapshots via CoW:

  $ btrfs subvolume snapshot /mnt/data /mnt/data/snap1

  Before snapshot:
    Root → [Node A] → [Leaf: file data blocks]

  After snapshot:
    Root (current) → [Node A] → [Leaf: same blocks] (refcount=2)
    Root (snap1)   → [Node A] → [Leaf: same blocks]

  Write to current:
    Root (current) → [Node A'] → [Leaf': NEW blocks]
    Root (snap1)   → [Node A]  → [Leaf: OLD blocks] (preserved)

  Snapshot cost: one tree root copy (a few blocks), instant!

  # Rollback to snapshot
  $ btrfs subvolume set-default <snap1_id> /mnt

  # Send snapshot to another disk (incremental backup)
  $ btrfs send /mnt/snap1 | btrfs receive /backup/
  $ btrfs send -p /mnt/snap1 /mnt/snap2 | btrfs receive /backup/
```

### Btrfs Data Integrity

```
Btrfs verifies data on EVERY read:

  Write path:
    data → CRC32C(data) → store checksum in Checksum Tree
    data → write to disk

  Read path:
    Read data from disk
    Read stored checksum from Checksum Tree
    Compute CRC32C(read_data)
    Compare: computed == stored?
      ├── Match → return data
      └── Mismatch → DATA CORRUPTION detected!
            ├── If RAID: read from mirror, repair
            └── If no RAID: return -EIO (and log)

  $ sudo btrfs scrub start /mnt
  # Reads and verifies ALL data on disk
  
  $ sudo btrfs scrub status /mnt
  Scrub started:    Mon Jan 15 10:00:00 2024
  Status:           finished
  Total to scrub:   100.00GiB
  Bytes scrubbed:   100.00GiB
  Errors:           0
```

---

## 11.4 Feature Comparison

```
Feature              │ ext4        │ XFS         │ Btrfs
─────────────────────┼─────────────┼─────────────┼──────────────
Max file size        │ 16 TB       │ 8 EB        │ 16 EB
Max FS size          │ 1 EB        │ 8 EB        │ 16 EB
Journaling           │ Yes (jbd2)  │ Yes (WAL)   │ No (CoW)
Copy-on-Write        │ No          │ Reflink     │ Full CoW
Snapshots            │ No          │ No          │ Yes
Data checksums       │ No          │ No          │ CRC32C
Metadata checksums   │ CRC32C      │ CRC32C      │ CRC32C
Compression          │ No          │ No          │ zlib/lzo/zstd
Built-in RAID        │ No          │ No          │ Yes
Subvolumes           │ No          │ No          │ Yes
Encryption           │ fscrypt     │ No*         │ No*
Online grow          │ Yes         │ Yes         │ Yes
Online shrink        │ Yes         │ No          │ Yes
Defragmentation      │ e4defrag    │ xfs_fsr     │ btrfs defrag
Allocation           │ mballoc/    │ AG/B+tree   │ Extent tree/
                     │ bitmap      │             │ chunk alloc
Default distro       │ Ubuntu,     │ RHEL,       │ openSUSE,
                     │ Debian      │ CentOS,     │ Fedora
                     │             │ Amazon Linux│ (Workstation)

* fscrypt support planned/in-progress for XFS and Btrfs
```

---

## 11.5 When to Choose Which FS

```
Use ext4 when:
  ✓ General-purpose server/desktop
  ✓ Well-understood, battle-tested
  ✓ Moderate-sized files and FS
  ✓ Need encryption (fscrypt)
  ✓ Automotive/embedded (stable, well-supported)
  ✓ You want the safe default choice

Use XFS when:
  ✓ Large files (media, databases, VMs)
  ✓ High-throughput sequential I/O
  ✓ Multi-threaded write workloads
  ✓ Reflink for VM image management
  ✓ Enterprise/server (RHEL default)
  ✓ Large file systems (> 16 TB)

Use Btrfs when:
  ✓ Need snapshots and rollback
  ✓ Data integrity verification (checksums)
  ✓ Compression saves disk space
  ✓ Incremental backups (send/receive)
  ✓ Multiple subvolumes (separate / and /home)
  ✓ Experimental/advanced desktop use
  
  ⚠ Avoid Btrfs RAID 5/6 (write hole not fully fixed)
```

---

## 11.6 File System Creation (mkfs)

```bash
# Create ext4 filesystem
$ mkfs.ext4 -L "mydata" -m 1 -O metadata_csum,64bit /dev/sda1
#           -L label  -m reserved% -O features

# Create XFS filesystem
$ mkfs.xfs -L "mydata" -f /dev/sda1
#           -L label  -f force

# Create Btrfs filesystem
$ mkfs.btrfs -L "mydata" -m raid1 -d raid1 /dev/sda1 /dev/sdb1
#             -L label    -m metadata  -d data  (multi-device RAID)

# ext4 tuning after creation
$ tune2fs -c 0 -i 0 /dev/sda1        # Disable fsck count/interval
$ tune2fs -o journal_data_writeback /dev/sda1  # Change journal mode
$ tune2fs -O ^has_journal /dev/sda1   # Remove journal
$ tune2fs -O encrypt /dev/sda1        # Enable encryption feature
```

---

## 11.7 Mounting Options

```bash
# ext4 important mount options
mount -t ext4 -o defaults,noatime,commit=60,barrier=1 /dev/sda1 /mnt

Key ext4 options:
  data=ordered     ← Default: metadata journaled, data written first
  data=journal     ← Both data + metadata journaled (safest, slowest)
  data=writeback   ← Only metadata journaled (fastest, least safe)
  noatime          ← Don't update access time (performance boost)
  discard          ← Enable continuous TRIM for SSDs
  commit=N         ← Journal commit interval (seconds)
  barrier=1        ← Enable write barriers (data safety)
  delalloc         ← Delayed allocation (default)
  nodelalloc       ← Disable delayed allocation

# XFS mount options
mount -t xfs -o noatime,logbufs=8,logbsize=256k /dev/sda1 /mnt

# Btrfs mount options
mount -t btrfs -o compress=zstd:3,noatime,autodefrag,ssd /dev/sda1 /mnt
  compress=zstd:3  ← Zstandard compression level 3
  autodefrag       ← Auto defragmentation on write
  ssd              ← SSD optimizations (auto-detected)
  space_cache=v2   ← Free space tree (faster)
```

---

## Kernel Source References

```
ext4:
  fs/ext4/super.c          ← Mount, fill_super, feature checks
  fs/ext4/inode.c          ← Inode read/write, extent mapping
  fs/ext4/extents.c        ← Extent tree operations
  fs/ext4/mballoc.c        ← Multi-block allocator
  fs/ext4/namei.c          ← Directory operations, HTree
  fs/ext4/file.c           ← file_operations
  fs/ext4/fsync.c          ← fsync implementation

XFS:
  fs/xfs/xfs_super.c       ← Mount, fill_super
  fs/xfs/xfs_iops.c        ← Inode operations
  fs/xfs/xfs_file.c        ← File operations
  fs/xfs/xfs_reflink.c     ← Reflink/CoW
  fs/xfs/libxfs/xfs_bmap.c ← Block mapping

Btrfs:
  fs/btrfs/super.c          ← Mount
  fs/btrfs/inode.c          ← Inode operations
  fs/btrfs/file.c           ← File operations
  fs/btrfs/extent-tree.c    ← Block allocation
  fs/btrfs/ctree.c          ← B-tree operations
  fs/btrfs/disk-io.c        ← Checksum verification
  fs/btrfs/send.c           ← Send/receive
  fs/btrfs/volumes.c        ← RAID management
```

---

## Interview Questions

1. **Compare ext4 extents vs indirect block mapping. Which is better for large files?**
2. **What are ext4 flex block groups? How do they improve performance?**
3. **How does XFS achieve parallel I/O with Allocation Groups?**
4. **What is XFS reflink? How does it differ from hard links?**
5. **Explain Btrfs Copy-on-Write. How do snapshots work?**
6. **How does Btrfs detect data corruption? What happens on a checksum mismatch?**
7. **When would you choose ext4 over XFS? XFS over Btrfs?**
8. **What is delayed allocation and which file systems support it?**
9. **What is the ext4 journal mode "ordered"? Why is it the default?**
10. **How does Btrfs compression work? What compression algorithms are supported?**

---

## Summary

- **ext4**: Default, battle-tested, extents, delayed alloc, journaling, fscrypt. Best general-purpose.
- **XFS**: B+tree-based, AG parallelism, reflink, great for large files. Enterprise choice.
- **Btrfs**: Full CoW, snapshots, checksums, compression, RAID, subvolumes. Feature-rich but less mature.
- Allocation: ext4 uses bitmap+buddy, XFS uses B+tree, Btrfs uses extent tree
- Data safety: ext4 journal, XFS WAL, Btrfs CoW (no journal needed)
- Choose based on: workload (large files→XFS), features needed (snapshots→Btrfs), stability (ext4)

---

*Next: [Chapter 12 — Journaling and Crash Recovery](Chapter_12_Journaling.md)*
