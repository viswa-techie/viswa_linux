# Chapter 26: Documentation & References

## Learning Goals
- Know where to find official Linux filesystem documentation
- Master key kernel documentation files for FS developers
- Build a reference library of essential papers, books, and online resources
- Understand how to stay current with FS development

---

## 26.1 Official Kernel Documentation

### In-Tree Documentation (Documentation/)

```
Key FS documentation files in the kernel source tree:

Documentation/filesystems/
├── vfs.rst                  ← VFS interface documentation (MUST READ)
│                              Describes all operation tables:
│                              file_operations, inode_operations,
│                              super_operations, address_space_operations
│
├── path-lookup.rst          ← Path resolution (namei.c) explained
│                              RCU-walk, ref-walk, link following
│
├── locking.rst              ← VFS locking rules
│                              Which locks protect which operations
│
├── porting.rst              ← Guide for porting FS to new kernel versions
│
├── ext4/                    ← ext4-specific documentation
│   ├── ext4.rst             ← ext4 overview, mount options
│   ├── blockalloc.rst       ← Block allocation (mballoc)
│   ├── journal.rst          ← jbd2 journal documentation
│   └── ...
│
├── xfs/                     ← XFS documentation
│   ├── xfs-self-describing-metadata.rst
│   └── ...
│
├── btrfs/                   ← Btrfs documentation
│
├── f2fs/                    ← f2fs design documentation
│   └── f2fs.rst
│
├── nfs/                     ← NFS client/server docs
│
├── fuse.rst                 ← FUSE interface
│
├── overlayfs.rst            ← OverlayFS documentation
│
├── proc.rst                 ← /proc filesystem
│
├── sysfs.rst                ← /sys filesystem
│
├── tmpfs.rst                ← tmpfs documentation
│
├── squashfs.rst             ← SquashFS documentation
│
├── ubifs.rst                ← UBIFS documentation
│
├── fscrypt.rst              ← Filesystem encryption API
│
├── fsverity.rst             ← Filesystem verity documentation
│
└── mount_api.rst            ← New mount API documentation

Documentation/admin-guide/
├── ext4.rst                 ← Admin guide for ext4
└── xfs.rst                  ← Admin guide for XFS

Documentation/block/
├── stat.rst                 ← /sys/block/*/stat format
├── bfq-iosched.rst          ← BFQ I/O scheduler
└── blk-mq.rst               ← Multi-queue block layer
```

### How to Read Kernel Documentation

```bash
# Build HTML documentation
$ make htmldocs
$ xdg-open Documentation/output/filesystems/vfs.html

# Or read directly (reStructuredText format)
$ less Documentation/filesystems/vfs.rst

# Search documentation
$ grep -r "address_space_operations" Documentation/filesystems/

# Online version (always up-to-date):
# https://docs.kernel.org/filesystems/
```

---

## 26.2 Essential Kernel Source Files (Reference List)

```
VFS Core (must read for understanding):
─────────────────────────────────────────
  include/linux/fs.h          All VFS structures and operation tables
  fs/namei.c                  Path resolution (link_path_walk, do_last)
  fs/open.c                   open() syscall implementation
  fs/read_write.c             read()/write() implementation
  fs/dcache.c                 Dentry cache (d_lookup, d_alloc, shrink)
  fs/inode.c                  Inode lifecycle (iget, iput, evict)
  fs/super.c                  Superblock management
  fs/namespace.c              Mount/unmount operations

Page Cache:
─────────────────────────────────────────
  mm/filemap.c                Core page cache (filemap_read, fault)
  mm/readahead.c              Readahead algorithm
  mm/page-writeback.c         Dirty page management, writeback

ext4 (most documented and read):
─────────────────────────────────────────
  fs/ext4/ext4.h              Main header (all structures)
  fs/ext4/super.c             Mount, registration, superblock
  fs/ext4/inode.c             Inode operations, writepages
  fs/ext4/file.c              File operations
  fs/ext4/namei.c             Directory operations
  fs/ext4/extents.c           Extent tree
  fs/ext4/mballoc.c           Multi-block allocator

Block Layer:
─────────────────────────────────────────
  include/linux/bio.h         bio structure
  block/blk-mq.c              Multi-queue dispatch
  block/bio.c                 bio management
```

---

## 26.3 Essential Books

```
1. "Understanding the Linux Kernel" — Bovet & Cesati
   └─→ Chapter 12: The Virtual Filesystem
   └─→ Chapter 15: The Page Cache
   └─→ Chapter 16: Accessing Files
   The classic. Best for VFS architecture understanding.

2. "Linux Kernel Development" — Robert Love
   └─→ Chapter 13: The Virtual Filesystem
   Concise and practical. Good for interview prep.

3. "Linux Device Drivers, 3rd Edition" — Corbet, Rubini, Kroah-Hartman
   └─→ Free online: https://lwn.net/Kernel/LDD3/
   Essential for understanding driver ↔ filesystem interaction.

4. "Operating Systems: Three Easy Pieces" (OSTEP) — Arpaci-Dusseau
   └─→ Part 3: Persistence
   └─→ Chapters: File System Interface, Implementation, FFS, FSCK, Journaling
   Free online: https://pages.cs.wisc.edu/~remzi/OSTEP/
   Best for fundamentals. University-level, beautifully written.

5. "The Design and Implementation of the FreeBSD Operating System"
   └─→ Chapters on VFS and UFS
   FreeBSD VFS is ancestor of Linux VFS. Good conceptual background.

6. "File Systems: Design and Implementation" — Giampaolo
   └─→ Covers BeOS FS design. Excellent for FS design principles.

7. "XFS: The Big Storage File System for Linux" — SGI documentation
   └─→ XFS Algorithms & Data Structures (official reference)
   └─→ https://xfs.wiki.kernel.org/
```

---

## 26.4 Essential Papers

```
Classic Papers:
──────────────────────────────────────────────────────────────
1. "A Fast File System for UNIX" — McKusick et al., 1984
   Original FFS paper. Introduced cylinder groups, 
   locality-aware allocation. Foundation for ext2/ext4.

2. "The Design and Implementation of a Log-Structured File System"
   — Rosenblum & Ousterhout, 1992
   Log-structured FS paper. Foundation for f2fs, JFFS2.

3. "Journaling the Linux ext2fs Filesystem" — Tweedie, 2000
   ext3 journaling design. jbd2 is direct descendant.

4. "BTRFS: The Linux B-tree Filesystem" — Rodeh, Bacik, Mason, 2013
   Btrfs design paper. CoW B-trees, snapshots, checksums.

5. "F2FS: A New File System for Flash Storage" — Lee et al., 2015
   Samsung's f2fs design. Multi-head logging, hot/cold separation.

6. "The Scalable Commutativity Rule" — Clements et al., 2013
   ScaleFS concepts. How FS design affects scalability.

Modern Papers:
──────────────────────────────────────────────────────────────
7. "Barrier-Enabled IO Stack for Flash Storage" — Won et al., 2018
   Rethinking barriers for flash storage performance.

8. "Optimizing Every Operation in a Write-Optimized File System"
   — Conway et al., 2017 (BetrFS)
   B-epsilon tree filesystem. Novel approach to write optimization.

9. "EROFS: A Compression-Friendly Readonly File System for Resource-
   Scarce Devices" — Gao et al., 2019
   EROFS design paper for embedded/mobile.
```

---

## 26.5 Online Resources

```
Essential websites:
──────────────────────────────────────────────────────────────
1. Kernel source browser:
   https://elixir.bootlin.com/linux/latest/source
   → Best online source navigation with cross-references

2. LWN.net (Linux Weekly News):
   https://lwn.net/Kernel/Index/
   → Best for tracking FS development, new features, patches
   → Search for "ext4", "XFS", "Btrfs" articles

3. Kernel documentation:
   https://docs.kernel.org/filesystems/
   → Official rendered documentation

4. Linux Foundation wiki:
   https://wiki.linuxfoundation.org/

5. XFS wiki:
   https://xfs.wiki.kernel.org/
   → XFS algorithms, data structures, administration

6. Btrfs wiki:
   https://btrfs.wiki.kernel.org/
   → Btrfs design, status, FAQ

7. ext4 wiki:
   https://ext4.wiki.kernel.org/
   → ext4 features, design, TODO

8. Kernel newbies:
   https://kernelnewbies.org/
   → Changelogs per kernel version (what FS features were added)

9. Phoronix:
   https://www.phoronix.com/
   → FS benchmarks and performance comparisons

10. Brendan Gregg's blog:
    https://brendangregg.com/
    → Performance analysis, eBPF/bpftrace for FS debugging
```

---

## 26.6 Mailing Lists & Conferences

```
Mailing Lists (where FS development happens):
──────────────────────────────────────────────────────────────
  linux-fsdevel@vger.kernel.org     ← Main FS development list
  linux-ext4@vger.kernel.org        ← ext4 specific
  linux-xfs@vger.kernel.org         ← XFS specific
  linux-btrfs@vger.kernel.org       ← Btrfs specific
  linux-f2fs-devel@lists.sourceforge.net  ← f2fs specific
  linux-block@vger.kernel.org       ← Block layer
  linux-mm@kvack.org                ← Memory management (page cache)

  Archive: https://lore.kernel.org/

Conferences:
──────────────────────────────────────────────────────────────
  Linux Storage, Filesystem, Memory Management & BPF Summit (LSF/MM/BPF)
    → Yearly, invitation-only. Defines FS roadmap.
    → Proceedings on LWN.net

  Linux Plumbers Conference
    → FS track with deep technical talks

  Kernel Recipes
    → European conference, excellent FS talks on YouTube

  FAST (USENIX Conference on File and Storage Technologies)
    → Academic papers on storage systems
    → https://www.usenix.org/conference/fast
```

---

## 26.7 Useful Kernel Config Options

```
# Essential config options for filesystem work:

## Core VFS
CONFIG_BLOCK=y              # Block layer support
CONFIG_LBDAF=y              # Large block device support

## Filesystems
CONFIG_EXT4_FS=y            # ext4
CONFIG_EXT4_FS_POSIX_ACL=y  # ACL support
CONFIG_EXT4_FS_SECURITY=y   # SELinux xattr support
CONFIG_XFS_FS=m             # XFS (module)
CONFIG_BTRFS_FS=m           # Btrfs (module)
CONFIG_F2FS_FS=m            # f2fs (module)
CONFIG_SQUASHFS=y           # SquashFS
CONFIG_SQUASHFS_LZ4=y       # LZ4 compression
CONFIG_EROFS_FS=y           # EROFS
CONFIG_OVERLAY_FS=y         # OverlayFS
CONFIG_TMPFS=y              # tmpfs
CONFIG_FUSE_FS=m            # FUSE

## Encryption & Security
CONFIG_FS_ENCRYPTION=y      # fscrypt
CONFIG_FS_VERITY=y          # fs-verity
CONFIG_DM_CRYPT=m           # dm-crypt
CONFIG_DM_VERITY=y          # dm-verity

## Flash
CONFIG_MTD=y                # Memory Technology Devices
CONFIG_UBIFS_FS=m           # UBIFS
CONFIG_JFFS2_FS=m           # JFFS2

## Debugging
CONFIG_EXT4_DEBUG=y         # ext4 debug messages
CONFIG_DEBUG_FS=y           # debugfs
CONFIG_FTRACE=y             # Function tracer
CONFIG_BPF_SYSCALL=y        # eBPF support
```

---

## 26.8 Quick Command Reference

```bash
# ── INFORMATION ──────────────────────────────────────────
df -h                       # Disk space usage
df -i                       # Inode usage
mount                       # Show mounts
cat /proc/mounts            # Detailed mount info
cat /proc/filesystems       # Registered FS types
lsblk                       # Block device list
blkid                       # Block device UUIDs and types
findmnt                     # Tree view of mounts

# ── CREATION & FORMATTING ───────────────────────────────
mkfs.ext4 /dev/sda1         # Format as ext4
mkfs.xfs /dev/sda1          # Format as XFS
mkfs.btrfs /dev/sda1        # Format as Btrfs
mkfs.f2fs /dev/sda1         # Format as f2fs
mksquashfs dir/ out.sqsh    # Create SquashFS image
mkfs.erofs out.erofs dir/   # Create EROFS image

# ── MOUNTING ─────────────────────────────────────────────
mount -t ext4 /dev/sda1 /mnt
mount -o noatime,data=ordered /dev/sda1 /mnt
mount -t squashfs image.sqsh /mnt -o loop
mount --bind /src /dst      # Bind mount
mount -o remount,rw /       # Remount read-write

# ── TUNING ───────────────────────────────────────────────
tune2fs -l /dev/sda1        # Show ext4 parameters
tune2fs -O ^has_journal     # Remove journal
tune2fs -c 30 /dev/sda1     # fsck every 30 mounts
dumpe2fs /dev/sda1          # Dump ext4 superblock
xfs_info /mnt               # Show XFS parameters
btrfs filesystem show       # Show Btrfs info

# ── REPAIR ───────────────────────────────────────────────
e2fsck -f /dev/sda1         # Check ext4
xfs_repair /dev/sda1        # Repair XFS
btrfs check /dev/sda1       # Check Btrfs (careful!)

# ── DEBUGGING ────────────────────────────────────────────
debugfs /dev/sda1            # ext4 debugger
xfs_db /dev/sda1             # XFS debugger
strace -e trace=file cmd     # Trace file syscalls
iostat -xz 1                 # I/O statistics
cat /proc/meminfo | grep Cache  # Page cache size
echo 3 > /proc/sys/vm/drop_caches  # Drop caches (testing)
```

---

## Interview Questions

1. **Where is the VFS interface documented in the kernel source tree?**
2. **Name three essential books for learning Linux filesystem internals.**
3. **What mailing list would you subscribe to for ext4 development?**
4. **How do you navigate kernel source code effectively?**
5. **Name three important academic papers on filesystem design.**

---

## Summary

- Kernel docs: `Documentation/filesystems/vfs.rst` is the starting point for VFS
- Source: `include/linux/fs.h`, `fs/namei.c`, `mm/filemap.c` are the core files
- Books: OSTEP for fundamentals, "Understanding the Linux Kernel" for depth, LKD for interviews
- Online: elixir.bootlin.com for source, LWN.net for development news, docs.kernel.org for official docs
- Papers: FFS (1984), LFS (1992), ext3 journaling (2000), Btrfs (2013), f2fs (2015)
- Community: linux-fsdevel mailing list, LSF/MM summit, Linux Plumbers Conference
- Practice: build kernel with debug options, use ftrace/perf/bpftrace on FS tracepoints

---

*Next: [Chapter 27 — Interview Preparation](Chapter_27_Interview_Preparation.md)*
