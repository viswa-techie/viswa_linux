# Chapter 4: Linux File System Architecture Overview

## Learning Goals
- Understand the full Linux storage stack from syscall to disk
- Know the role of VFS as an abstraction layer
- See how file systems plug into VFS
- Understand the relationship between VFS, page cache, and block layer

---

## 4.1 The Complete Linux Storage Stack

```
┌──────────────────────────────────────────────────────────────────┐
│                        USER SPACE                                │
│  Application:  open(), read(), write(), close(), mmap()          │
│                fstat(), lseek(), readdir(), mkdir(), unlink()     │
└──────────────────────────┬───────────────────────────────────────┘
                           │ syscall (software interrupt / VDSO)
┌──────────────────────────▼───────────────────────────────────────┐
│                     SYSTEM CALL LAYER                             │
│  sys_open(), sys_read(), sys_write()  (fs/open.c, fs/read_write.c)│
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│                   VIRTUAL FILE SYSTEM (VFS)                      │
│                                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  ┌───────────┐ │
│  │ struct file  │  │struct dentry│  │struct    │  │struct     │ │
│  │ (open file)  │  │(path cache) │  │inode     │  │super_block│ │
│  └──────┬──────┘  └──────┬──────┘  │(metadata)│  │(FS state) │ │
│         │                │         └────┬─────┘  └─────┬─────┘ │
│         └────────┬───────┘              │              │        │
│                  │                      │              │        │
│  ┌───────────────▼──────────────────────▼──────────────▼──────┐ │
│  │            Operation Tables (function pointers)             │ │
│  │  file_operations  inode_operations  super_operations        │ │
│  │  dentry_operations  address_space_operations                │ │
│  └───────────────┬────────────────────────────────────────────┘ │
└──────────────────┼──────────────────────────────────────────────┘
                   │ Dispatch to specific FS implementation
┌──────────────────▼──────────────────────────────────────────────┐
│              CONCRETE FILE SYSTEMS                               │
│                                                                   │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐       │
│  │ ext4 │ │ XFS  │ │Btrfs │ │tmpfs │ │procfs│ │ NFS  │ ...   │
│  └──┬───┘ └──┬───┘ └──┬───┘ └──────┘ └──────┘ └──┬───┘       │
│     │        │        │                           │             │
└─────┼────────┼────────┼───────────────────────────┼─────────────┘
      │        │        │                           │
┌─────▼────────▼────────▼───────────────┐    ┌─────▼──────────┐
│          PAGE CACHE                    │    │ Network Stack  │
│  (mm/filemap.c, page/folio cache)     │    │ (TCP/IP, RPC)  │
│  Caches file data in memory           │    └────────────────┘
│  Manages readahead, writeback         │
└───────────────┬───────────────────────┘
                │  submit_bio()
┌───────────────▼───────────────────────┐
│           BLOCK LAYER                  │
│  struct bio → merge → schedule         │
│  I/O schedulers: mq-deadline, bfq     │
│  Device mapper (LVM, dm-crypt, RAID)  │
└───────────────┬───────────────────────┘
                │
┌───────────────▼───────────────────────┐
│        BLOCK DEVICE DRIVERS            │
│  SCSI, NVMe, MMC, UFS                 │
└───────────────┬───────────────────────┘
                │
┌───────────────▼───────────────────────┐
│        PHYSICAL HARDWARE               │
│  HDD, SSD, eMMC, UFS, NVMe            │
└───────────────────────────────────────┘
```

---

## 4.2 VFS: The Abstraction Layer

### Why VFS?

```
Problem: Linux supports 50+ file system types.
  Every application would need to know how each FS works.

Solution: VFS provides a UNIFORM INTERFACE.

  Application sees:   open(), read(), write(), stat()
  VFS translates to:  ext4_file_open(), ext4_read_iter(), etc.

  ┌──────────────────┐
  │  Application     │
  │  read(fd, ...)   │
  └───────┬──────────┘
          │
  ┌───────▼──────────────────────────┐
  │          VFS Layer               │
  │  vfs_read() → looks up           │
  │  file->f_op->read_iter()        │
  └───────┬──────┬──────┬───────────┘
          │      │      │
   ┌──────▼─┐ ┌─▼────┐ ┌▼─────┐
   │ ext4   │ │ XFS  │ │ NFS  │
   │ .read_ │ │.read_│ │.read_│
   │ iter() │ │iter()│ │iter()│
   └────────┘ └──────┘ └──────┘
```

### VFS Key Principle: Everything Is a File

```
Linux VFS treats EVERYTHING as a file:
  /home/user/document.txt     ← Regular file
  /dev/sda                    ← Block device
  /dev/ttyS0                  ← Character device
  /proc/cpuinfo               ← Virtual file (procfs)
  /sys/class/net/eth0/mtu     ← Virtual file (sysfs)
  /tmp/socket.sock            ← Unix domain socket
  /dev/null                   ← Special device
  pipe(fd)                    ← Anonymous pipe

All accessed via the same syscall interface:
  open() → read()/write() → close()
```

---

## 4.3 The Four Pillars of VFS

```
VFS Object          │ Represents              │ Kernel Structure
────────────────────┼──────────────────────────┼───────────────────
Superblock          │ A mounted file system    │ struct super_block
Inode               │ A specific file (on disk)│ struct inode
Dentry              │ A path component         │ struct dentry
File                │ An open file (per-process)│ struct file

Relationships:
  super_block (1)  ←──────── contains ─────────>> inode (many)
  inode (1)        ←──────── referenced by ────>> dentry (many)
  dentry (1)       ←──────── opened as ────────>> file (many)

Example: Two processes open /etc/passwd
  super_block:  1 (for the / filesystem)
  inode:        1 (inode for /etc/passwd)
  dentry:       2 (one for "etc", one for "passwd")
  file:         2 (one per process, different file positions)
```

### Operation Tables

```c
/* Each VFS object has associated operation tables (function pointers) */

struct super_operations {    /* How to manage FS state */
    struct inode *(*alloc_inode)(struct super_block *sb);
    void (*destroy_inode)(struct inode *);
    void (*dirty_inode)(struct inode *, int flags);
    int (*write_inode)(struct inode *, struct writeback_control *);
    int (*statfs)(struct dentry *, struct kstatfs *);
    int (*sync_fs)(struct super_block *sb, int wait);
    /* ... */
};

struct inode_operations {    /* How to manipulate files/dirs */
    struct dentry *(*lookup)(struct inode *, struct dentry *, unsigned int);
    int (*create)(struct mnt_idmap *, struct inode *, struct dentry *, umode_t, bool);
    int (*link)(struct dentry *, struct inode *, struct dentry *);
    int (*unlink)(struct inode *, struct dentry *);
    int (*mkdir)(struct mnt_idmap *, struct inode *, struct dentry *, umode_t);
    int (*rename)(struct mnt_idmap *, struct inode *, struct dentry *,
                  struct inode *, struct dentry *, unsigned int);
    /* ... */
};

struct file_operations {     /* How to read/write open files */
    loff_t (*llseek)(struct file *, loff_t, int);
    ssize_t (*read_iter)(struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter)(struct kiocb *, struct iov_iter *);
    int (*open)(struct inode *, struct file *);
    int (*release)(struct inode *, struct file *);
    int (*mmap)(struct file *, struct vm_area_struct *);
    long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
    int (*fsync)(struct file *, loff_t, loff_t, int datasync);
    /* ... */
};

struct address_space_operations {  /* How to manage page cache */
    int (*read_folio)(struct file *, struct folio *);
    int (*writepages)(struct address_space *, struct writeback_control *);
    bool (*dirty_folio)(struct address_space *, struct folio *);
    /* ... */
};
```

---

## 4.4 File System Registration

### How a File System Plugs Into VFS

```c
/* Every file system registers itself with VFS */

/* fs/ext4/super.c */
static struct file_system_type ext4_fs_type = {
    .owner          = THIS_MODULE,
    .name           = "ext4",
    .init_fs_context = ext4_init_fs_context,
    .parameters     = ext4_param_specs,
    .kill_sb        = kill_block_super,
    .fs_flags       = FS_REQUIRES_DEV | FS_ALLOW_IDMAP,
};

static int __init ext4_init_fs(void)
{
    /* ... initialize caches, workqueues ... */
    err = register_filesystem(&ext4_fs_type);
    return err;
}

module_init(ext4_init_fs);
module_exit(ext4_exit_fs);
```

```
Registration flow:
  1. module_init() → ext4_init_fs()
  2. register_filesystem(&ext4_fs_type)
      → adds to global linked list: file_systems
  3. Now available for mount:
      $ mount -t ext4 /dev/sda1 /mnt

List registered file systems:
  $ cat /proc/filesystems
  nodev   sysfs
  nodev   tmpfs
  nodev   proc
  nodev   devtmpfs
          ext4
          xfs
          btrfs
  ...

"nodev" = does not require a block device
```

---

## 4.5 Mount Architecture

```
Mounting connects a file system to the directory tree:

  mount -t ext4 /dev/sda2 /home

Before mount:
  /
  ├── bin/
  ├── etc/
  └── home/           ← empty mount point

After mount:
  /
  ├── bin/
  ├── etc/
  └── home/           ← now shows contents of /dev/sda2
      ├── user1/
      └── user2/

Kernel creates:
  struct super_block    → populated by ext4_fill_super()
  struct mount          → links FS to mount point
  Root dentry + inode   → root directory of mounted FS
```

### Mount Hierarchy

```
# Mount tree on a typical Linux system:
$ findmnt --tree
TARGET                        SOURCE     FSTYPE  OPTIONS
/                             /dev/sda2  ext4    rw,relatime
├─/sys                        sysfs      sysfs   rw,nosuid
│ ├─/sys/kernel/security      securityfs securityfs rw
│ ├─/sys/fs/cgroup            cgroup2    cgroup2 rw
│ └─/sys/kernel/debug         debugfs    debugfs rw
├─/proc                       proc       proc    rw,nosuid
├─/dev                        devtmpfs   devtmpfs rw
│ ├─/dev/pts                  devpts     devpts  rw
│ └─/dev/shm                  tmpfs      tmpfs   rw
├─/run                        tmpfs      tmpfs   rw
├─/tmp                        tmpfs      tmpfs   rw
├─/home                       /dev/sda3  ext4    rw
└─/boot                       /dev/sda1  vfat    rw

Many different file system TYPES, all unified under one tree!
```

---

## 4.6 Page Cache Integration

```
The page cache sits between VFS and the block layer:

  VFS (read request)
    │
    ▼
  Page Cache lookup (address_space)
    │
    ├── HIT → return data immediately (no disk I/O!)
    │
    └── MISS → allocate page, issue I/O
              │
              ▼
            Block Layer → Disk
              │
              ▼
            Data returned → fill page → return to caller
              │
              ▼
            Page now CACHED for future reads

Key structures:
  struct address_space {
      struct inode         *host;         /* owning inode */
      struct xarray        i_pages;       /* radix tree of cached pages */
      const struct address_space_operations *a_ops;
      unsigned long        nrpages;       /* number of cached pages */
      /* ... */
  };

Every inode has an address_space → maps file offsets to cached pages
```

### Read Path Through Page Cache

```
read(fd, buf, 4096)
  │
  ▼
vfs_read()                                    [fs/read_write.c]
  │
  ▼
file->f_op->read_iter()                       [dispatch to FS]
  │
  ▼
generic_file_read_iter()                      [mm/filemap.c]
  │
  ▼
filemap_read()
  │
  ├── filemap_get_pages()
  │     │
  │     ├── Find page in page cache (xa_load on i_pages)
  │     │     ├── Found (cache HIT) → return page
  │     │     └── Not found (cache MISS) →
  │     │           │
  │     │           ▼
  │     │         a_ops->read_folio()          [FS-specific I/O]
  │     │           │
  │     │           ▼  (for ext4)
  │     │         ext4_read_folio()
  │     │           │
  │     │           ▼
  │     │         mpage_readpage() → submit_bio()
  │     │           │
  │     │           ▼
  │     │         Wait for I/O completion
  │     │
  │     └── Return page(s)
  │
  ▼
copy_page_to_iter()  → copy data to user buffer
```

---

## 4.7 Write Path Architecture

```
write(fd, buf, 4096)
  │
  ▼
vfs_write()                                   [fs/read_write.c]
  │
  ▼
file->f_op->write_iter()
  │
  ▼
ext4_file_write_iter()                        [fs/ext4/file.c]
  │
  ▼
generic_perform_write()                       [mm/filemap.c]
  │
  ├── Find or create page in page cache
  │
  ├── a_ops->write_begin()                    [ext4_write_begin()]
  │     → allocate blocks, journal transaction
  │
  ├── copy_from_user() → write data to page
  │
  ├── a_ops->write_end()                      [ext4_write_end()]
  │     → mark page dirty, update inode size
  │
  └── Return (data is in page cache, NOT yet on disk)

Later (writeback):
  pdflush / kworker thread
    │
    ▼
  writeback_single_inode()
    │
    ▼
  a_ops->writepages()  →  ext4_writepages()
    │
    ▼
  submit_bio()  →  Block layer → Disk
```

### Write Ordering

```
Important: write() returns BEFORE data hits disk!

  write(fd, data, len)  → data in page cache (dirty page)
  ...                   → user continues working
  [background]          → kernel writeback thread flushes to disk

To ensure data on disk:
  fdatasync(fd)   → flush data + necessary metadata
  fsync(fd)       → flush data + ALL metadata
  sync()          → flush ALL dirty pages system-wide

  O_SYNC flag     → every write waits for disk completion
  O_DSYNC flag    → every write waits for data + size metadata
```

---

## 4.8 Error Handling Architecture

```
Errors can occur at every layer:

Layer            │ Error Examples                    │ Handling
─────────────────┼───────────────────────────────────┼──────────────
Syscall          │ ENOENT, EACCES, EINVAL           │ Return -errno
VFS              │ ENOMEM, ENOSPC                    │ Return -errno
File System      │ Corruption, journal error          │ Mark FS error
Page Cache       │ Allocation failure                 │ Retry or fail
Block Layer      │ I/O error, timeout                 │ -EIO to FS
Driver           │ Hardware failure                   │ Report to block
Hardware         │ Bad sector, controller fail        │ Retry / remap

ext4 error policy (set at mount):
  errors=continue  → log error, continue
  errors=remount-ro → remount read-only (DEFAULT)
  errors=panic     → kernel panic (production servers)
```

---

## 4.9 File System Types Classification

```
Category            │ Examples           │ Backing Store     │ Characteristics
────────────────────┼────────────────────┼───────────────────┼─────────────────
Disk-based          │ ext4, XFS, Btrfs   │ Block device      │ Persistent data
Flash-based         │ JFFS2, UBIFS, F2FS │ MTD / block       │ Flash-aware
Network             │ NFS, CIFS, 9P      │ Network server    │ Remote storage
Pseudo / Virtual    │ procfs, sysfs      │ Kernel memory     │ No persistent data
Memory-based        │ tmpfs, ramfs       │ RAM               │ Volatile
Stacking / Overlay  │ OverlayFS, eCryptfs│ Other FS          │ Layer on top
Read-only           │ SquashFS, EROFS    │ Block device      │ Compressed, small
FUSE (user-space)   │ sshfs, ntfs-3g    │ User-space daemon │ Flexible, slower
```

---

## 4.10 Architecture Comparison with Other OSes

```
                         Linux           │ Windows           │ macOS
─────────────────────────────────────────┼───────────────────┼──────────────
Abstraction Layer        VFS             │ I/O Manager +     │ VFS (BSD-derived)
                                         │ Filter Manager    │
Primary FS               ext4/XFS/Btrfs │ NTFS              │ APFS
FS Extensibility         Kernel modules  │ FS Miniport driver│ Kernel extensions
                         or FUSE         │ or Minifilter     │ or FUSE
Caching                  Page cache      │ Cache Manager     │ UBC (Unified
                         (unified)       │ (separate from VM)│ Buffer Cache)
Mount Model              Single tree     │ Drive letters +   │ Single tree
                         (mount points)  │ mount points      │ (mount points)
FS Registration          register_       │ IoRegister        │ vfs_fsadd()
                         filesystem()    │ FileSystem()      │
Max Path                 4096 bytes      │ 32,767 chars (API)│ 1024 bytes
                         (PATH_MAX)      │ 260 chars (Win32) │
Case Sensitivity         Yes (default)   │ No (preserving)   │ No (preserving)
```

---

## Kernel Source References

```
VFS core:
  fs/namei.c              ← Path resolution (lookup)
  fs/open.c               ← open(), close() implementation
  fs/read_write.c         ← read(), write() implementation
  fs/file_table.c         ← File table management
  fs/dcache.c             ← Dentry cache
  fs/inode.c              ← Inode management
  fs/super.c              ← Superblock management
  fs/namespace.c          ← Mount operations
  fs/filesystems.c        ← FS registration

Page cache:
  mm/filemap.c            ← generic_file_read_iter(), filemap_read()
  mm/page-writeback.c     ← Dirty page writeback

Block layer:
  block/blk-core.c        ← Block I/O submission
  block/blk-mq.c          ← Multi-queue dispatch
```

---

## Interview Questions

1. **Draw the complete Linux storage stack from syscall to hardware.**
2. **What is VFS? Why does Linux need it?**
3. **Name the four main VFS objects and their relationships.**
4. **What are file_operations, inode_operations, and super_operations? How do they enable FS pluggability?**
5. **How does a file system register itself with the kernel?**
6. **Trace the read path from read() through page cache to disk I/O.**
7. **Why does write() return before data reaches disk? How do you force data to disk?**
8. **What is address_space and how does it relate to the page cache?**
9. **Compare Linux VFS with Windows I/O Manager.**
10. **What happens when you mount a file system? What kernel objects are created?**

---

## Summary

- Linux storage stack: syscall → VFS → FS → page cache → block layer → hardware
- VFS provides uniform interface: four objects (superblock, inode, dentry, file)
- File systems register via `register_filesystem()` and provide operation tables
- Page cache intercepts all reads/writes: cache hit avoids disk I/O entirely
- Writes go to page cache first, flushed later by writeback threads
- fsync()/fdatasync() force data to disk; O_SYNC makes every write synchronous
- Linux unifies everything under one directory tree; mount attaches FS to tree

---

*Next: [Chapter 5 — Virtual File System (VFS) Deep Dive](Chapter_05_VFS_Deep_Dive.md)*
