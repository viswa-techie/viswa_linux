# Chapter 23: Glossary

## Learning Goals
- Quick-reference glossary of all filesystem terminology
- Link each term to the chapter where it's covered in depth
- Provide concise, precise definitions suitable for interviews

---

## A

**ACL (Access Control List):** Extension of Unix permissions allowing per-user and per-group access rules beyond owner/group/other. Stored as extended attributes (`system.posix_acl_access`). See Chapter 18.

**Address Space:** Kernel structure (`struct address_space`) managing the page cache pages for one inode. Contains xarray of cached pages, operations table (`a_ops`), and writeback state. See Chapter 16.

**Allocation Group (AG):** XFS divides a volume into independent AGs, each with its own B+trees for free space, inodes, and metadata. Enables parallel allocation. See Chapter 11.

**atime:** Last access time of a file. Updated on every read by default, causing metadata writes. `noatime` mount option disables. See Chapter 17.

---

## B

**BDI (Backing Dev Info):** `struct backing_dev_info` — represents a backing device for writeback purposes. Each block device has one. Flusher threads are per-BDI. See Chapter 16.

**BFQ (Budget Fair Queuing):** I/O scheduler that provides fair bandwidth distribution among processes. Good for HDDs and interactive workloads. See Chapter 17.

**bio:** `struct bio` — the primary unit of block I/O in Linux. Contains a list of (page, offset, length) segments describing data to transfer, plus the target block device and sector. See Chapter 3.

**Block Group:** ext4 divides the disk into block groups (~128MB each), each containing a copy of metadata (bitmaps, inode table) plus data blocks. See Chapter 10.

**Block Layer:** Kernel subsystem (`block/`) that manages block I/O requests between filesystems and device drivers. Handles merging, scheduling, and dispatch. See Chapter 3.

**Btrfs:** B-tree filesystem. Copy-on-Write design with built-in snapshots, checksums, compression, RAID, and multi-device support. See Chapter 11.

**Buffer Head:** `struct buffer_head` — legacy structure for block-sized I/O (one per disk block). Being replaced by bio/folio-based I/O. See Chapter 3.

---

## C

**Capability:** Linux breaks root privilege into granular capabilities (e.g., `CAP_DAC_OVERRIDE`, `CAP_SYS_ADMIN`). Files and processes can have capability sets. See Chapter 18.

**Checkpointing (jbd2):** Process of writing committed journal data to its final on-disk location, allowing journal space to be reclaimed. See Chapter 12.

**container_of:** Kernel macro to obtain a pointer to the enclosing structure from a pointer to an embedded member. Used extensively in VFS (e.g., `EXT4_I(inode)`). See Chapter 20.

**CoW (Copy-on-Write):** Writing creates a new copy of a block rather than modifying in place. Used by Btrfs, enables efficient snapshots. See Chapter 11.

**ctime:** Inode change time. Updated when inode metadata changes (permissions, ownership, link count). See Chapter 1.

---

## D

**dcache (Dentry Cache):** In-memory cache of directory entries (`struct dentry`). Forms a tree mirroring the filesystem hierarchy. Dramatically speeds up path resolution. See Chapter 5.

**Delayed Allocation (delalloc):** Deferring physical block allocation until writeback time. Enables better space allocation decisions and reduces fragmentation. Default in ext4. See Chapter 10.

**Dentry:** `struct dentry` — represents a directory entry (name-to-inode mapping) in the VFS. Can be positive (has inode), negative (caches "doesn't exist"), or unused. See Chapter 5, 6.

**Direct I/O (O_DIRECT):** Bypasses the page cache, transferring data directly between user buffers and disk. Used by databases that manage their own caching. See Chapter 7.

**Dirty Page:** A page cache page that has been modified but not yet written to disk. See Chapter 16.

**dm-crypt:** Device mapper target providing transparent full-disk/partition encryption at the block layer. Used via LUKS. See Chapter 18.

**dm-verity:** Device mapper target providing a Merkle hash tree for verifying integrity of read-only partitions. Used in Android Verified Boot. See Chapter 18.

---

## E

**eMMC:** Embedded MultiMediaCard. Flash storage commonly used in automotive and embedded systems. Managed NAND with built-in FTL. See Chapter 3.

**EROFS:** Enhanced Read-Only File System. Compressed read-only filesystem optimized for performance. Alternative to SquashFS for Android. See Chapter 25.

**EVM (Extended Verification Module):** Protects security-critical extended attributes (SELinux labels, IMA hashes, capabilities) via HMAC. See Chapter 18.

**Extent:** Contiguous range of physical blocks mapped to contiguous logical blocks. More efficient than indirect block mapping for large files. Used by ext4, XFS, Btrfs. See Chapter 10.

**ext4:** Fourth Extended Filesystem. The most widely used Linux filesystem. Features: extents, delayed allocation, journaling (jbd2), directory HTree. See Chapter 11.

---

## F

**f2fs:** Flash-Friendly File System. Log-structured FS designed for flash storage (eMMC, SSD). Optimized for flash write patterns. See Chapter 25.

**fallocate():** System call to preallocate disk space for a file without writing data. Reduces fragmentation for known-size files. See Chapter 10.

**FHS (Filesystem Hierarchy Standard):** Standard defining the directory structure of Unix/Linux systems (/, /bin, /etc, /usr, /var, etc.). See Chapter 1.

**File Descriptor (fd):** Integer handle returned by `open()`. Index into the per-process file descriptor table pointing to a `struct file`. See Chapter 5.

**Folio:** Kernel abstraction (since 5.16) replacing raw page references in the page cache. Can represent compound (multi-page) entries. See Chapter 16.

**fscrypt:** Linux kernel framework for filesystem-level encryption. Encrypts file contents and names with per-file keys. Used for Android FBE. See Chapter 18.

**fsync():** System call ensuring all data and metadata for a file are written to persistent storage. See Chapter 7, 21.

**FTL (Flash Translation Layer):** Firmware in SSDs/eMMC that maps logical blocks to physical flash pages, handles wear leveling and garbage collection. See Chapter 3.

**FUSE:** Filesystem in Userspace. Allows filesystem implementation in user space via /dev/fuse. See Chapter 14.

---

## G

**GDT (Group Descriptor Table):** ext4 table containing per-block-group metadata: bitmap locations, inode table location, free counts. See Chapter 10.

---

## H

**Hard Link:** Multiple directory entries pointing to the same inode. Creating a hard link increments `i_nlink`. See Chapter 8.

**HTree:** Hashed B-tree used by ext4 for directory indexing. Enables fast lookups in directories with millions of entries. See Chapter 8.

---

## I

**icache (Inode Cache):** System-wide cache of `struct inode` objects. Managed by slab allocator. Access via `iget_locked()`. See Chapter 5.

**IMA (Integrity Measurement Architecture):** Measures (hashes) files and records measurements in a list extended into TPM. Can appraise (verify) file integrity. See Chapter 18.

**Inode:** `struct inode` — in-memory representation of a file's metadata: type, permissions, size, timestamps, block mapping. One per file. No filename. See Chapter 5, 6.

**Inode Number (i_ino):** Unique identifier for a file within a filesystem. Directory entries map names to inode numbers. See Chapter 1.

**io_uring:** Modern asynchronous I/O interface using submission/completion ring buffers in shared memory. Higher throughput than aio. See Chapter 7.

---

## J

**jbd2 (Journaling Block Device 2):** Kernel journaling layer used by ext4. Manages transactions, journal writes, and crash recovery. See Chapter 12.

**JFFS2 (Journaling Flash File System 2):** Log-structured filesystem for raw NOR/NAND flash. Used in embedded Linux systems without managed flash. See Chapter 25.

**Journal Mode (ext4):**
- **data=journal:** Both data and metadata journaled. Safest, slowest.
- **data=ordered:** Only metadata journaled; data written before metadata commit. Default, good balance.
- **data=writeback:** Only metadata journaled; data may be written after. Fastest, risk of stale data exposure. See Chapter 12.

---

## K

**Kconfig:** Kernel build configuration system. FS options like `CONFIG_EXT4_FS`, `CONFIG_XFS_FS`. See Chapter 20.

---

## L

**LRU (Least Recently Used):** Page reclaim algorithm using active/inactive lists. Pages move between lists based on access patterns. See Chapter 16, 22.

**LUKS (Linux Unified Key Setup):** Standard format for disk encryption. Manages dm-crypt keys with multiple key slots. See Chapter 18.

---

## M

**mballoc (Multi-Block Allocator):** ext4's block allocator that allocates multiple blocks at once for better locality. Includes preallocation, delayed allocation support. See Chapter 10.

**mmap():** System call that maps a file (or anonymous memory) into process virtual address space. File access via memory operations instead of read()/write(). See Chapter 15.

**Mount Namespace:** Kernel namespace providing per-process mount table isolation. Each namespace sees its own set of mounts. See Chapter 9.

**Mount Propagation:** Controls how mount/unmount events propagate between namespaces: shared, private, slave, unbindable. See Chapter 9.

**mtime:** File modification time. Updated when file data is written. See Chapter 1.

---

## N

**Negative Dentry:** A dentry with `d_inode = NULL`, caching the fact that a name does NOT exist in a directory. Speeds up repeated lookups for non-existent files. See Chapter 5.

**NFS (Network File System):** Protocol for accessing files over a network. NFSv4 adds stateful operations, delegations, and compound RPCs. See Chapter 14.

**noatime:** Mount option that prevents updating the access time on file reads. Critical performance optimization. See Chapter 17.

**Namespace (mount):** See Mount Namespace.

---

## O

**OOB (Out-of-Band):** Extra bytes per flash page for ECC and metadata (NAND flash). See Chapter 3.

**Orphan Inode:** An inode with `i_nlink=0` that still has open file descriptors. Added to ext4 orphan list; cleaned up on next mount. See Chapter 21.

**OverlayFS:** Union mount filesystem that layers an upper (writable) FS over a lower (read-only) FS. Used by Docker/containers and Android. See Chapter 9.

---

## P

**Page Cache:** Unified kernel cache of file data in memory. All regular file I/O goes through the page cache (unless O_DIRECT). Managed by `mm/filemap.c`. See Chapter 16.

**Page Fault:** CPU exception when accessing a virtual address not backed by a physical page. For mmap'd files, triggers `filemap_fault()` to read data from disk. See Chapter 15.

**POSIX ACL:** See ACL.

**procfs (/proc):** Virtual filesystem exposing kernel and process information as files/directories. Not backed by storage. See Chapter 13.

---

## R

**RCU-walk:** Fast path for pathname resolution that uses Read-Copy-Update (RCU) instead of dentry reference counting. Avoids atomic operations. Falls back to ref-walk on failure. See Chapter 5, 8.

**Readahead:** Kernel mechanism that prefetches file pages ahead of the current read position, anticipating sequential access. Managed by `mm/readahead.c`. See Chapter 7, 16.

**Reflink:** XFS/Btrfs feature allowing copy-on-write cloning of file data blocks. `cp --reflink`. See Chapter 11.

---

## S

**SELinux:** Security-Enhanced Linux. Mandatory Access Control using security labels on files and processes. Labels stored in `security.selinux` xattr. See Chapter 18.

**Slab Cache:** Kernel memory allocator optimized for frequent allocation/deallocation of same-sized objects. Used for inodes, dentries, etc. (`kmem_cache`). See Chapter 6.

**Snapshot:** Point-in-time copy of a filesystem. In Btrfs, snapshots are cheap due to CoW — only changed blocks are duplicated. See Chapter 11.

**SquashFS:** Compressed read-only filesystem. Used for rootfs on embedded systems. See Chapter 25.

**Superblock:** `struct super_block` — in-memory representation of a mounted filesystem. Contains FS type, block size, root dentry, operations. See Chapter 5, 6.

**Symbolic Link (symlink):** Special file containing a path to another file. Resolved during path resolution (up to 40 levels). See Chapter 8.

**sysfs (/sys):** Virtual filesystem exposing kernel device model (buses, devices, drivers) as directories and files. See Chapter 13.

---

## T

**tmpfs:** RAM-backed filesystem. Data exists only in memory and swap. Fast, non-persistent. Used for /tmp, /run. See Chapter 13.

**TRIM/Discard:** Command telling an SSD that blocks are no longer in use. Enables garbage collection and reduces write amplification. See Chapter 3.

---

## U

**UBIFS (Unsorted Block Image File System):** Flash filesystem for raw NAND via UBI (Unsorted Block Images) volume manager. Compression, write-back caching. See Chapter 25.

**UFS (Universal Flash Storage):** Modern flash storage interface replacing eMMC. Higher performance, used in smartphones and automotive. See Chapter 3.

---

## V

**VFS (Virtual File System):** Kernel abstraction layer (`fs/` core files) providing uniform file operations interface. All filesystems implement VFS operation tables. See Chapter 4, 5.

**VMA (Virtual Memory Area):** `struct vm_area_struct` — represents a contiguous region of a process's virtual address space. Created by mmap(). See Chapter 15.

---

## W

**Writeback:** Process of flushing dirty page cache pages to disk. Triggered by dirty thresholds, timers, or explicit sync. See Chapter 16, 22.

**Write Barrier:** Disk command ensuring all previously submitted writes are committed before subsequent writes. Essential for journal integrity. See Chapter 12.

---

## X

**xarray:** Kernel data structure (replacing radix tree) used to index page cache pages by file offset within `struct address_space`. See Chapter 16.

**xattr (Extended Attribute):** Name-value pairs attached to files. Namespaces: user, system (ACLs), security (SELinux), trusted. See Chapter 18.

**XFS:** High-performance filesystem from SGI. Features: allocation groups, B+trees, delayed allocation, reflink. Scales to exabytes. Default on RHEL. See Chapter 11.

---

## Key Abbreviations Table

```
┌────────────┬──────────────────────────────────────────────┐
│ Abbreviation│ Full Name                                    │
├────────────┼──────────────────────────────────────────────┤
│ ACL        │ Access Control List                          │
│ AG         │ Allocation Group (XFS)                       │
│ AIO        │ Asynchronous I/O                             │
│ AVB        │ Android Verified Boot                        │
│ BDI        │ Backing Dev Info                             │
│ BFQ        │ Budget Fair Queuing                          │
│ CoW        │ Copy on Write                                │
│ CRC        │ Cyclic Redundancy Check                      │
│ DAC        │ Discretionary Access Control                 │
│ DIO        │ Direct I/O                                   │
│ ECC        │ Error-Correcting Code                        │
│ EROFS      │ Enhanced Read-Only File System               │
│ EVM        │ Extended Verification Module                 │
│ FBE        │ File-Based Encryption                        │
│ FDE        │ Full Disk Encryption                         │
│ FHS        │ Filesystem Hierarchy Standard                │
│ FTL        │ Flash Translation Layer                      │
│ FUSE       │ Filesystem in Userspace                      │
│ GDT        │ Group Descriptor Table                       │
│ IMA        │ Integrity Measurement Architecture           │
│ IOPS       │ I/O Operations Per Second                    │
│ JFFS2      │ Journaling Flash File System 2               │
│ LRU        │ Least Recently Used                          │
│ LSM        │ Linux Security Module                        │
│ LUKS       │ Linux Unified Key Setup                      │
│ MAC        │ Mandatory Access Control                     │
│ MLC/TLC/QLC│ Multi/Triple/Quad Level Cell (flash)         │
│ NFS        │ Network File System                          │
│ NVMe       │ Non-Volatile Memory express                  │
│ OTA        │ Over-The-Air update                          │
│ PTE        │ Page Table Entry                             │
│ RCU        │ Read-Copy-Update                             │
│ SB         │ Superblock                                   │
│ SCSI       │ Small Computer System Interface              │
│ TEE        │ Trusted Execution Environment                │
│ TPM        │ Trusted Platform Module                      │
│ TRIM       │ Tell SSD blocks are unused                   │
│ UBIFS      │ Unsorted Block Image File System             │
│ UFS        │ Universal Flash Storage                      │
│ VFS        │ Virtual File System                          │
│ VMA        │ Virtual Memory Area                          │
│ WAL        │ Write-Ahead Log                              │
│ XTS        │ XEX Tweakable Block Cipher with Stealing     │
└────────────┴──────────────────────────────────────────────┘
```

---

## Summary

This glossary covers 100+ terms across:
- VFS core concepts (inode, dentry, superblock, file, address_space)
- Filesystem implementations (ext4, XFS, Btrfs, f2fs, UBIFS, SquashFS)
- Performance (readahead, writeback, page cache, I/O scheduler, noatime)
- Security (ACL, capabilities, fscrypt, dm-crypt, SELinux, IMA, dm-verity)
- Storage hardware (SSD, eMMC, UFS, FTL, TRIM)
- Kernel patterns (container_of, slab cache, operation tables)

Use as a quick reference during study or before interviews.

---

*Next: [Chapter 24 — OS Comparison](Chapter_24_OS_Comparison.md)*
