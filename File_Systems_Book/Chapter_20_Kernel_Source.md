# Chapter 20: Kernel Source Code Navigation

## Learning Goals
- Navigate the Linux kernel source tree for filesystem code
- Understand key files and their roles in the VFS and major filesystems
- Know how to read kernel FS code: patterns, macros, data structures
- Build a mental map of the fs/ directory hierarchy

---

## 20.1 Kernel Source Tree — FS Overview

```
linux/
├── fs/                         ← ALL filesystem code lives here
│   ├── Makefile
│   ├── Kconfig
│   │
│   ├── ──── VFS Core ────
│   ├── namei.c                 ← Path resolution (do_filp_open, path_lookupat)
│   ├── open.c                  ← open() syscall (do_sys_open, do_filp_open)
│   ├── read_write.c            ← read/write/lseek syscalls
│   ├── file.c                  ← File descriptor management
│   ├── inode.c                 ← Inode allocation, lifecycle
│   ├── dcache.c                ← Dentry cache (d_lookup, d_alloc)
│   ├── super.c                 ← Superblock management
│   ├── namespace.c             ← Mount namespace implementation
│   ├── mount.h                 ← Internal mount structures
│   ├── file_table.c            ← System-wide file table
│   ├── stat.c                  ← stat() family syscalls
│   ├── attr.c                  ← File attribute changes (chmod, chown)
│   ├── xattr.c                 ← Extended attributes (setxattr, getxattr)
│   ├── ioctl.c                 ← ioctl() dispatch
│   ├── locks.c                 ← File locking (flock, fcntl)
│   ├── aio.c                   ← Asynchronous I/O
│   ├── io_uring.c              ← io_uring interface (or io_uring/)
│   ├── buffer.c                ← Buffer head management (legacy)
│   ├── mpage.c                 ← Multi-page I/O (mpage_readpages)
│   ├── direct-io.c             ← O_DIRECT implementation
│   ├── splice.c                ← splice/sendfile
│   ├── readdir.c               ← Directory iteration (getdents)
│   ├── select.c                ← Select/poll for FDs
│   ├── posix_acl.c             ← POSIX ACL implementation
│   ├── pipe.c                  ← Pipe filesystem
│   ├── eventfd.c               ← Event file descriptors
│   ├── signalfd.c              ← Signal file descriptors
│   ├── timerfd.c               ← Timer file descriptors
│   │
│   ├── ──── Major Filesystems ────
│   ├── ext4/                   ← ext4 (most widely used)
│   │   ├── super.c             ← Superblock ops, mount
│   │   ├── inode.c             ← Inode operations
│   │   ├── file.c              ← File operations
│   │   ├── dir.c               ← Directory operations
│   │   ├── namei.c             ← ext4 path operations
│   │   ├── extents.c           ← Extent tree management
│   │   ├── mballoc.c           ← Multi-block allocator
│   │   ├── balloc.c            ← Block allocation
│   │   ├── ialloc.c            ← Inode allocation
│   │   ├── page-io.c           ← Page I/O for ext4
│   │   ├── readpage.c          ← Read path
│   │   ├── ext4_jbd2.c         ← Journal interface
│   │   ├── fsync.c             ← ext4_sync_file()
│   │   ├── xattr.c             ← Extended attributes
│   │   ├── acl.c               ← ACL support
│   │   ├── crypto.c            ← fscrypt integration
│   │   ├── verity.c            ← fs-verity integration
│   │   ├── ioctl.c             ← ext4 ioctls
│   │   ├── resize.c            ← Online resize
│   │   ├── migrate.c           ← Indirect→extent migration
│   │   ├── inline.c            ← Inline data support
│   │   └── ext4.h              ← Main ext4 header
│   │
│   ├── xfs/                    ← XFS
│   │   ├── xfs_super.c         ← Mount, superblock
│   │   ├── xfs_inode.c         ← Inode management
│   │   ├── xfs_file.c          ← File operations
│   │   ├── xfs_dir2.c          ← Directory format
│   │   ├── xfs_alloc.c         ← Block allocation
│   │   ├── xfs_bmap.c          ← Block mapping (extents)
│   │   ├── xfs_log.c           ← XFS log (WAL)
│   │   ├── xfs_trans.c         ← Transaction management
│   │   ├── xfs_refcount.c      ← Reflink/dedup
│   │   └── xfs_mount.h         ← Mount point structure
│   │
│   ├── btrfs/                  ← Btrfs
│   │   ├── super.c             ← Mount
│   │   ├── inode.c             ← Inode operations
│   │   ├── file.c              ← File operations
│   │   ├── ctree.c             ← B-tree implementation (core)
│   │   ├── disk-io.c           ← Disk I/O
│   │   ├── extent-tree.c       ← Extent management
│   │   ├── transaction.c       ← Transaction management
│   │   ├── volumes.c           ← Multi-device/RAID
│   │   ├── compression.c       ← Transparent compression
│   │   ├── send.c              ← btrfs send/receive
│   │   └── space-info.c        ← Space accounting
│   │
│   ├── ──── Special Filesystems ────
│   ├── proc/                   ← /proc
│   ├── sysfs/                  ← /sys
│   ├── debugfs/                ← /sys/kernel/debug
│   ├── tmpfs/ (→ mm/shmem.c)  ← tmpfs/shmem
│   ├── devpts/                 ← /dev/pts
│   ├── configfs/               ← /sys/kernel/config
│   ├── tracefs/                ← /sys/kernel/tracing
│   ├── ramfs/                  ← RAM filesystem
│   ├── hugetlbfs/              ← Huge page filesystem
│   │
│   ├── ──── Network Filesystems ────
│   ├── nfs/                    ← NFS client
│   ├── nfsd/                   ← NFS server
│   ├── cifs/                   ← SMB/CIFS client
│   ├── 9p/                     ← Plan 9
│   ├── fuse/                   ← FUSE
│   │
│   ├── ──── Flash/Embedded ────
│   ├── ubifs/                  ← UBIFS
│   ├── jffs2/                  ← JFFS2
│   ├── squashfs/               ← SquashFS (read-only)
│   ├── erofs/                  ← EROFS (read-only, compressed)
│   ├── f2fs/                   ← Flash-Friendly FS
│   │
│   ├── ──── Other ────
│   ├── overlayfs/              ← OverlayFS (container layers)
│   ├── fat/                    ← FAT/VFAT
│   ├── ntfs3/                  ← NTFS (modern driver)
│   ├── isofs/                  ← ISO 9660 (CD-ROM)
│   ├── udf/                    ← UDF (DVD/BD)
│   └── crypto/                 ← fscrypt core (fs/crypto/)
│
├── mm/                         ← Memory management (page cache here)
│   ├── filemap.c               ← Page cache core (filemap_read, etc.)
│   ├── readahead.c             ← Readahead algorithm
│   ├── page-writeback.c        ← Dirty page management
│   ├── vmscan.c                ← Page reclaim (LRU scanning)
│   ├── mmap.c                  ← Memory mapping
│   ├── shmem.c                 ← tmpfs/shmem implementation
│   └── truncate.c              ← Truncate pages from cache
│
├── block/                      ← Block layer
│   ├── blk-core.c              ← Core block I/O
│   ├── blk-mq.c                ← Multi-queue block layer
│   ├── bio.c                   ← BIO management
│   ├── mq-deadline.c           ← Deadline I/O scheduler
│   ├── bfq-iosched.c           ← BFQ I/O scheduler
│   ├── partitions/             ← Partition table parsing
│   └── blk-crypto.c            ← Inline encryption
│
├── include/linux/              ← Headers
│   ├── fs.h                    ← VFS core structures
│   ├── dcache.h                ← Dentry definitions
│   ├── mount.h                 ← Mount structures
│   ├── pagemap.h               ← Page cache definitions
│   ├── buffer_head.h           ← Buffer head
│   ├── bio.h                   ← BIO structure
│   ├── blkdev.h                ← Block device
│   ├── fscrypt.h               ← fscrypt API
│   ├── posix_acl.h             ← ACL structures
│   └── xattr.h                 ← Extended attribute API
│
└── security/                   ← Security subsystem
    ├── selinux/                ← SELinux
    │   └── hooks.c             ← LSM hooks for FS access
    └── integrity/
        ├── ima/                ← IMA
        └── evm/                ← EVM
```

---

## 20.2 Key Files — Detailed Guide

### VFS Core: Path Resolution

```c
/* fs/namei.c — The heart of path resolution */

/*
 * Entry point for opening a file:
 *   sys_open → do_sys_open → do_filp_open → path_openat
 */
static struct file *path_openat(struct nameidata *nd,
                                const struct open_flags *op,
                                unsigned flags)
{
    struct file *file;
    int error;

    file = alloc_empty_file(op->open_flag, current_cred());

    /* Walk the path component by component */
    error = link_path_walk(s, nd);

    /* Handle the last component (create, open, etc.) */
    error = do_last(nd, file, op);

    return file;
}

/*
 * Key functions in namei.c:
 *   link_path_walk()    — Walk path components
 *   walk_component()    — Process one component
 *   lookup_fast()       — Try dcache lookup first (fast path)
 *   lookup_slow()       — Fall back to inode->i_op->lookup
 *   may_open()          — Permission check before open
 *   try_to_unlazy()     — Switch from RCU-walk to ref-walk
 */
```

### VFS Core: Read/Write

```c
/* fs/read_write.c — Read and write syscalls */

/*
 * sys_read → ksys_read → vfs_read → (file->f_op->read_iter)
 */
ssize_t vfs_read(struct file *file, char __user *buf,
                 size_t count, loff_t *pos)
{
    /* Permission check */
    ret = rw_verify_area(READ, file, pos, count);

    /* Dispatch to filesystem read implementation */
    if (file->f_op->read_iter) {
        ret = new_sync_read(file, buf, count, pos);
        /* → calls file->f_op->read_iter() */
    } else if (file->f_op->read) {
        ret = file->f_op->read(file, buf, count, pos);
    }

    return ret;
}

/*
 * For ext4: file->f_op->read_iter = ext4_file_read_iter()
 *   → generic_file_read_iter()
 *     → filemap_read()           (mm/filemap.c)
 *       → page cache lookup
 *       → readahead if miss
 *       → copy_page_to_iter() to userspace
 */
```

### Page Cache: filemap.c

```c
/* mm/filemap.c — Page cache implementation */

/*
 * filemap_read() — The main "read from page cache" function
 *
 * Called from generic_file_read_iter() for buffered reads
 */
static ssize_t filemap_read(struct kiocb *iocb,
                            struct iov_iter *iter,
                            ssize_t already_read)
{
    struct folio *folio;

    for (;;) {
        /* Try to find page in cache */
        folio = filemap_get_folio(mapping, index);

        if (!folio) {
            /* Page not in cache → trigger readahead + read */
            page_cache_sync_readahead(mapping, ra, filp, index, ...);
            folio = filemap_get_folio(mapping, index);
        }

        /* Wait for page to be up-to-date */
        if (!folio_test_uptodate(folio)) {
            error = filemap_update_page(iocb, mapping, iter, folio);
        }

        /* Copy data to user buffer */
        copied = copy_folio_to_iter(folio, offset, bytes, iter);
    }
}

/* Key functions in mm/filemap.c:
 *   filemap_read()           — Buffered read from page cache
 *   filemap_write_and_wait() — Write dirty pages and wait
 *   filemap_fault()          — Handle page fault for mmap'd file
 *   filemap_get_folio()      — Find or create folio in cache
 *   generic_file_read_iter() — Standard .read_iter implementation
 */
```

---

## 20.3 Reading Kernel Code — Patterns

### Pattern 1: Operation Tables (Polymorphism)

```c
/* Every filesystem fills in these operation tables */

/* ext4 file operations (fs/ext4/file.c) */
const struct file_operations ext4_file_operations = {
    .llseek         = ext4_llseek,
    .read_iter      = ext4_file_read_iter,
    .write_iter     = ext4_file_write_iter,
    .open           = ext4_file_open,
    .release        = ext4_release_file,
    .fsync          = ext4_sync_file,
    .mmap           = ext4_file_mmap,
    .splice_read    = generic_file_splice_read,
    .splice_write   = iter_file_splice_write,
    .fallocate      = ext4_fallocate,
};

/* How VFS uses it:
 *   file->f_op->read_iter(kiocb, iter)
 *   file->f_op->write_iter(kiocb, iter)
 *   file->f_op->fsync(file, start, end, datasync)
 *
 * Each FS provides its own implementation
 * Many FSes reuse generic implementations for common ops
 */
```

### Pattern 2: container_of Macro

```c
/* Kernel uses container_of to go from embedded struct to outer struct */

struct ext4_inode_info {
    /* ... ext4-specific fields ... */
    __le32 i_data[15];         /* Block map or extent tree root */
    struct inode vfs_inode;    /* Embedded VFS inode (MUST be last) */
};

/* Given a VFS inode, get the ext4-specific structure */
static inline struct ext4_inode_info *EXT4_I(struct inode *inode)
{
    return container_of(inode, struct ext4_inode_info, vfs_inode);
}

/* Usage in ext4 code: */
struct ext4_inode_info *ei = EXT4_I(inode);
/* Now can access ei->i_data[], etc. */

/*
 * container_of implementation:
 * #define container_of(ptr, type, member) ({
 *     const typeof(((type *)0)->member) *__mptr = (ptr);
 *     (type *)((char *)__mptr - offsetof(type, member));
 * })
 *
 * Calculates: address of outer struct = ptr - offset of member
 */
```

### Pattern 3: Slab Cache for FS Objects

```c
/* Each FS creates slab caches for frequent allocations */

/* ext4 inode slab (fs/ext4/super.c) */
static struct kmem_cache *ext4_inode_cachep;

static int __init ext4_init_fs(void)
{
    ext4_inode_cachep = kmem_cache_create("ext4_inode_cache",
                                sizeof(struct ext4_inode_info),
                                0, SLAB_RECLAIM_ACCOUNT | SLAB_MEM_SPREAD,
                                ext4_inode_init_once);
    /* ... */
}

/* Allocation: */
static struct inode *ext4_alloc_inode(struct super_block *sb)
{
    struct ext4_inode_info *ei;
    ei = kmem_cache_alloc(ext4_inode_cachep, GFP_NOFS);
    /* Initialize */
    return &ei->vfs_inode;   /* Return embedded VFS inode */
}

/* Free: */
static void ext4_free_inode_callback(struct rcu_head *head)
{
    struct inode *inode = container_of(head, struct inode, i_rcu);
    kmem_cache_free(ext4_inode_cachep, EXT4_I(inode));
}
```

### Pattern 4: Error Handling

```c
/* Kernel FS code returns negative error codes */

int ext4_get_block(struct inode *inode, sector_t iblock,
                   struct buffer_head *bh, int create)
{
    struct ext4_map_blocks map;
    int ret;

    map.m_lblk = iblock;
    map.m_len = bh->b_size >> inode->i_blkbits;

    ret = ext4_map_blocks(handle, inode, &map, flags);
    if (ret < 0)
        return ret;    /* Propagate error: -EIO, -ENOSPC, etc. */

    if (ret > 0) {
        map_bh(bh, inode->i_sb, map.m_pblk);
        bh->b_state = (bh->b_state & ~EXT4_MAP_FLAGS) | map.m_flags;
    }

    return 0;
}

/* Common error codes in FS:
 * -ENOENT    File not found
 * -EACCES    Permission denied
 * -ENOSPC    No space left
 * -EIO       I/O error
 * -ENOMEM    Out of memory
 * -EEXIST    File exists
 * -ENOTDIR   Not a directory
 * -EISDIR    Is a directory
 * -EMFILE    Too many open files
 * -ELOOP     Too many symlink levels
 * -ENAMETOOLONG  Filename too long
 */
```

### Pattern 5: Filesystem Registration

```c
/* Every filesystem registers with VFS at module init */

/* fs/ext4/super.c */
static struct file_system_type ext4_fs_type = {
    .owner          = THIS_MODULE,
    .name           = "ext4",
    .mount          = ext4_mount,         /* Called on mount() */
    .kill_sb        = kill_block_super,   /* Called on umount() */
    .fs_flags       = FS_REQUIRES_DEV,   /* Needs a block device */
};

static int __init ext4_init_fs(void)
{
    int err;

    /* Create slab caches */
    err = ext4_init_slab_caches();
    if (err)
        return err;

    /* Register filesystem type */
    err = register_filesystem(&ext4_fs_type);
    if (err)
        goto fail;

    return 0;
}

module_init(ext4_init_fs);  /* Called on module load / boot */
module_exit(ext4_exit_fs);  /* Called on module unload */

/* VFS registration:
 * register_filesystem() adds to global linked list
 * When user calls mount -t ext4:
 *   1. VFS finds ext4_fs_type by name
 *   2. Calls ext4_fs_type.mount()
 *   3. ext4_mount → mount_bdev → ext4_fill_super
 *   4. ext4_fill_super reads superblock, sets up FS state
 */
```

---

## 20.4 Tracing a Complete Read Path

```
Application: read(fd, buf, 4096)

1. SYSCALL ENTRY (arch/x86/entry/common.c)
   │  sys_read(fd, buf, count)
   │
2. VFS LAYER (fs/read_write.c)
   │  ksys_read()
   │  └─→ vfs_read()
   │      └─→ file->f_op->read_iter()
   │
3. FILESYSTEM (fs/ext4/file.c)
   │  ext4_file_read_iter()
   │  └─→ generic_file_read_iter()
   │
4. PAGE CACHE (mm/filemap.c)
   │  filemap_read()
   │  ├─→ filemap_get_folio()     ← Cache hit? Return immediately!
   │  └─→ page_cache_sync_readahead() ← Cache miss → readahead
   │
5. READAHEAD (mm/readahead.c)
   │  page_cache_sync_ra()
   │  └─→ ondemand_readahead()
   │      └─→ ra_submit()
   │          └─→ read_pages()
   │              └─→ mapping->a_ops->readahead()
   │
6. FILESYSTEM READ (fs/ext4/readpage.c)
   │  ext4_readahead()
   │  └─→ ext4_mpage_readpages()
   │      └─→ ext4_map_blocks()   ← Map logical→physical blocks
   │          └─→ submit_bio()     ← Submit I/O to block layer
   │
7. BLOCK LAYER (block/blk-mq.c)
   │  submit_bio()
   │  └─→ blk_mq_submit_bio()
   │      └─→ blk_mq_get_request()
   │          └─→ blk_mq_try_issue_directly()
   │              └─→ __blk_mq_issue_directly()
   │
8. DEVICE DRIVER (e.g., drivers/nvme/host/pci.c)
   │  nvme_queue_rq()
   │  └─→ Hardware I/O
   │
9. COMPLETION (interrupt context)
   │  nvme_irq() → nvme_complete_rq()
   │  └─→ blk_mq_complete_request()
   │      └─→ bio->bi_end_io()
   │          └─→ mpage_end_io()
   │              └─→ folio_mark_uptodate()
   │                  └─→ folio_unlock()  ← Wake waiting readers
   │
10. RETURN TO USER
    filemap_read() copies data to user buffer
    └─→ copy_folio_to_iter()
        └─→ Return bytes read to application
```

---

## 20.5 Common Code Navigation Commands

```bash
# Search for function definition
$ grep -rn 'static.*ext4_file_read_iter' fs/ext4/
fs/ext4/file.c:123: static ssize_t ext4_file_read_iter(...)

# Find all callers of a function
$ grep -rn 'ext4_map_blocks' fs/ext4/ | head -20

# Find structure definition
$ grep -rn 'struct super_block {' include/
include/linux/fs.h:1234: struct super_block {

# Use cscope for cross-referencing
$ cscope -bqk -R

# Use ctags for jump-to-definition
$ ctags -R .

# Use LXR online: https://elixir.bootlin.com/linux/latest/source
# Excellent for: finding definitions, callers, cross-references

# Find Kconfig option for a filesystem
$ grep -rn 'config EXT4_FS' fs/ext4/Kconfig
config EXT4_FS
    tristate "The Extended 4 (ext4) filesystem"
    select JBD2
    select CRC16
    ...

# Find all tracepoints for a subsystem
$ grep -rn 'TRACE_EVENT' include/trace/events/ext4.h | head
TRACE_EVENT(ext4_da_write_begin, ...)
TRACE_EVENT(ext4_da_write_end, ...)
TRACE_EVENT(ext4_sync_file_enter, ...)
```

---

## 20.6 Building a Kernel Module for FS Experimentation

```c
/* Minimal filesystem module skeleton */
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/init.h>

/* Superblock fill function */
static int myfs_fill_super(struct super_block *sb, void *data, int silent)
{
    struct inode *root_inode;

    sb->s_magic = 0xDEADBEEF;
    sb->s_blocksize = PAGE_SIZE;
    sb->s_op = &myfs_super_ops;

    root_inode = new_inode(sb);
    root_inode->i_ino = 1;
    root_inode->i_mode = S_IFDIR | 0755;
    root_inode->i_op = &simple_dir_inode_operations;
    root_inode->i_fop = &simple_dir_operations;

    sb->s_root = d_make_root(root_inode);
    return 0;
}

static struct dentry *myfs_mount(struct file_system_type *type,
                                  int flags, const char *dev,
                                  void *data)
{
    /* For RAM-based FS, use mount_nodev */
    return mount_nodev(type, flags, data, myfs_fill_super);
}

static struct file_system_type myfs_type = {
    .owner   = THIS_MODULE,
    .name    = "myfs",
    .mount   = myfs_mount,
    .kill_sb = kill_litter_super,
};

static int __init myfs_init(void) {
    return register_filesystem(&myfs_type);
}

static void __exit myfs_exit(void) {
    unregister_filesystem(&myfs_type);
}

module_init(myfs_init);
module_exit(myfs_exit);
MODULE_LICENSE("GPL");
```

```bash
# Build and test:
$ make -C /lib/modules/$(uname -r)/build M=$PWD modules
$ insmod myfs.ko
$ mkdir /mnt/myfs
$ mount -t myfs none /mnt/myfs
$ ls /mnt/myfs    # Empty directory
$ umount /mnt/myfs
$ rmmod myfs
```

---

## 20.7 Key Header Files Reference

```
include/linux/fs.h          ← Core: struct inode, struct file, struct super_block
                                struct file_operations, struct inode_operations
                                struct super_operations, struct address_space_operations

include/linux/dcache.h      ← struct dentry, struct dentry_operations
                                d_lookup(), d_alloc(), d_instantiate()

include/linux/mount.h       ← struct vfsmount (public view)
fs/mount.h                  ← struct mount (internal, full)

include/linux/pagemap.h     ← struct address_space, page cache helpers
                                find_get_page(), add_to_page_cache()

include/linux/buffer_head.h ← struct buffer_head (legacy block I/O)

include/linux/bio.h         ← struct bio, bio_add_page(), submit_bio()

include/linux/blkdev.h      ← struct request_queue, struct gendisk

include/linux/writeback.h   ← Writeback control structures

include/linux/backing-dev.h ← struct backing_dev_info (BDI)

include/linux/fscrypt.h     ← fscrypt (FS encryption) API

include/linux/fsverity.h    ← fs-verity API

include/uapi/linux/fs.h     ← User-visible FS constants (for ioctls, flags)
```

---

## 20.8 Navigating with Bootlin Elixir

```
https://elixir.bootlin.com/linux/latest/source

Best online Linux kernel source browser:

1. Search identifier:
   Type "ext4_file_read_iter" → shows definition and all references

2. Navigate source tree:
   Browse fs/ext4/ → click file → click function → jump to definition

3. Cross-reference:
   Click any function/struct/macro → see all usages across the kernel

4. Version comparison:
   Switch between kernel versions to see how code evolved

5. Useful for understanding:
   - Where a function is called from
   - What implements a VFS operation
   - How data structures are initialized
   - Which config option enables a feature
```

---

## Interview Questions

1. **Where is the VFS path resolution code? Walk through the key files.**
2. **How does ext4 register itself with the VFS? Trace the code path.**
3. **Explain the container_of pattern and how ext4 uses EXT4_I().**
4. **What is the role of mm/filemap.c vs fs/ext4/readpage.c?**
5. **How would you trace a read() call from userspace to disk I/O?**
6. **What is the operation table pattern? Show an example.**
7. **Where is the page cache code? Which file handles readahead?**
8. **How do you build a minimal filesystem module?**
9. **What kernel headers define the VFS core structures?**
10. **How do you find all callers of a kernel function?**

---

## Summary

- All FS code lives under `fs/`; page cache in `mm/filemap.c`; block layer in `block/`
- VFS core: namei.c (path resolution), open.c (open syscall), read_write.c (read/write)
- Each FS has: super.c, inode.c, file.c, dir.c, namei.c in its directory
- Key patterns: operation tables (polymorphism), container_of, slab caches, error propagation
- Read path: read_write.c → filemap.c → readahead.c → FS readpage → block layer → driver
- Write path: read_write.c → filemap.c → FS writepages → block layer → driver
- Use Bootlin Elixir (elixir.bootlin.com) for online navigation
- Build minimal FS modules to experiment with VFS registration and operations

---

*Next: [Chapter 21 — Complete Flow Diagrams](Chapter_21_Flow_Diagrams.md)*
