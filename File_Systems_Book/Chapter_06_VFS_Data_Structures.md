# Chapter 6: Core VFS Data Structures

## Learning Goals
- Master the four pillar structures: super_block, inode, dentry, file
- Understand every critical field and its purpose
- Know how FS-specific structures embed/extend VFS structures
- See real ext4/XFS examples of structure usage

---

## 6.1 struct super_block — File System Instance

### Complete Structure Walkthrough

```c
/* include/linux/fs.h — struct super_block (simplified, key fields) */

struct super_block {
    /* Identity */
    struct list_head    s_list;         /* Global list of all superblocks */
    dev_t               s_dev;          /* Device identifier (major:minor) */
    unsigned char       s_blocksize_bits; /* log2(block_size) */
    unsigned long       s_blocksize;    /* Block size in bytes (1024, 2048, 4096) */
    loff_t              s_maxbytes;     /* Max file size for this FS */
    struct file_system_type *s_type;    /* Pointer to FS type (ext4_fs_type) */

    /* Operation tables */
    const struct super_operations *s_op;       /* FS operations */
    const struct dquot_operations *dq_op;      /* Quota operations */
    const struct export_operations *s_export_op; /* NFS export ops */

    /* Flags */
    unsigned long       s_flags;        /* Mount flags (SB_RDONLY, SB_NOSUID...) */
    unsigned long       s_iflags;       /* Internal flags */
    unsigned long       s_magic;        /* Magic number (0xEF53 for ext4) */

    /* Root */
    struct dentry       *s_root;        /* Root dentry of this FS */

    /* Writeback */
    struct rw_semaphore s_umount;       /* Protection during unmount */
    int                 s_count;        /* Reference count */
    atomic_t            s_active;       /* Active reference count */

    /* Inode management */
    struct list_lru     s_dentry_lru;   /* LRU list of unused dentries */
    struct list_lru     s_inode_lru;    /* LRU list of unused inodes */

    /* FS-private data */
    void                *s_fs_info;     /* FS-specific info (ext4_sb_info *) */

    /* Timing */
    struct timespec64   s_time_min;     /* Earliest timestamp this FS supports */
    struct timespec64   s_time_max;     /* Latest timestamp */
    u32                 s_time_gran;    /* Timestamp granularity (ns) */

    /* Security */
    void                *s_security;    /* LSM security data */

    /* Block device */
    struct block_device *s_bdev;        /* Associated block device */
    struct backing_dev_info *s_bdi;     /* Backing device info */

    /* Encryption / Compression */
    const struct fscrypt_operations *s_cop; /* fscrypt operations */

    /* ... many more fields ... */
};
```

### super_operations

```c
struct super_operations {
    /* Inode management */
    struct inode *(*alloc_inode)(struct super_block *sb);
    void (*destroy_inode)(struct inode *);
    void (*free_inode)(struct inode *);

    /* Inode I/O */
    void (*dirty_inode)(struct inode *, int flags);
    int (*write_inode)(struct inode *, struct writeback_control *wbc);
    void (*evict_inode)(struct inode *);

    /* Superblock management */
    void (*put_super)(struct super_block *);
    int (*sync_fs)(struct super_block *sb, int wait);
    int (*freeze_super)(struct super_block *, enum freeze_holder);
    int (*thaw_super)(struct super_block *, enum freeze_holder);
    int (*statfs)(struct dentry *, struct kstatfs *);
    int (*remount_fs)(struct super_block *, int *, char *);

    /* Quota */
    ssize_t (*quota_read)(struct super_block *, int, char *, size_t, loff_t);
    ssize_t (*quota_write)(struct super_block *, int, const char *, size_t, loff_t);

    /* Display mount options */
    int (*show_options)(struct seq_file *, struct dentry *);
};
```

### ext4 Superblock Example

```c
/* fs/ext4/super.c */
static const struct super_operations ext4_sops = {
    .alloc_inode    = ext4_alloc_inode,
    .free_inode     = ext4_free_in_core_inode,
    .destroy_inode  = ext4_destroy_inode,
    .write_inode    = ext4_write_inode,
    .dirty_inode    = ext4_dirty_inode,
    .evict_inode    = ext4_evict_inode,
    .put_super      = ext4_put_super,
    .sync_fs        = ext4_sync_fs,
    .freeze_fs      = ext4_freeze,
    .unfreeze_fs    = ext4_unfreeze,
    .statfs         = ext4_statfs,
    .show_options   = ext4_show_options,
};

/* ext4-specific superblock data embedded via s_fs_info */
struct ext4_sb_info {
    unsigned long s_desc_size;       /* Size of group descriptors */
    unsigned long s_inodes_per_block;
    unsigned long s_blocks_per_group;
    unsigned long s_inodes_per_group;
    unsigned long s_itb_per_group;   /* Inode table blocks per group */
    unsigned long s_desc_per_block;
    ext4_fsblk_t s_sb_block;        /* Superblock block number */
    struct ext4_super_block *s_es;   /* Pointer to on-disk superblock */
    struct buffer_head *s_sbh;       /* Buffer head for superblock */
    struct ext4_group_desc **s_group_desc; /* Block group descriptors */
    struct journal_s *s_journal;     /* Journal handle */
    unsigned long s_mount_opt;       /* Mount options (bitmask) */
    /* ... hundreds more fields ... */
};
```

---

## 6.2 struct inode — File Metadata

### Complete Structure Walkthrough

```c
/* include/linux/fs.h — struct inode (key fields) */

struct inode {
    /* Identity */
    umode_t             i_mode;         /* File type + permissions (rwxrwxrwx) */
    unsigned short      i_opflags;      /* Inode operation flags */
    kuid_t              i_uid;          /* Owner user ID */
    kgid_t              i_gid;          /* Owner group ID */
    unsigned int        i_flags;        /* FS flags (S_IMMUTABLE, S_APPEND...) */
    unsigned long       i_ino;          /* Inode number (unique within FS) */
    
    /* Link count and size */
    unsigned int        i_nlink;        /* Hard link count */
    loff_t              i_size;         /* File size in bytes */
    
    /* Timestamps */
    struct timespec64   __i_atime;      /* Last access time */
    struct timespec64   __i_mtime;      /* Last modification time */
    struct timespec64   __i_ctime;      /* Last status change time */

    /* Block info */
    unsigned int        i_blkbits;      /* log2(block_size) */
    blkcnt_t            i_blocks;       /* File size in 512-byte blocks */
    
    /* Operation tables */
    const struct inode_operations  *i_op;    /* Inode operations */
    const struct file_operations   *i_fop;   /* Default file operations */
    struct super_block              *i_sb;    /* Superblock this inode belongs to */

    /* Page cache */
    struct address_space           *i_mapping; /* Page cache mapping */
    struct address_space            i_data;    /* Embedded address_space */
    
    /* Locking */
    struct rw_semaphore             i_rwsem;   /* Read-write semaphore */
    
    /* Security */
    void                           *i_security; /* LSM security data */
    
    /* Device (if special file) */
    dev_t                           i_rdev;    /* Device number (for dev files) */
    
    /* Private FS data */
    void                           *i_private; /* FS-specific private pointer */
    
    /* Reference counting */
    atomic_t                        i_count;   /* Reference count */
    atomic_t                        i_writecount; /* Writers count */
    
    /* Listing */
    struct hlist_node               i_hash;    /* Hash list (icache) */
    struct list_head                i_io_list; /* Writeback list */
    struct list_head                i_lru;     /* LRU eviction list */
    struct list_head                i_sb_list; /* Per-superblock list */
    
    /* Union for FS-specific info */
    union {
        struct pipe_inode_info  *i_pipe;   /* Pipe info */
        struct cdev             *i_cdev;   /* Char device */
        char                    *i_link;   /* Symlink target */
    };
};
```

### i_mode Encoding

```
i_mode is a 16-bit field:

  Bits 15-12: File type
    S_IFSOCK  0140000  Socket
    S_IFLNK   0120000  Symbolic link
    S_IFREG   0100000  Regular file
    S_IFBLK   0060000  Block device
    S_IFDIR   0040000  Directory
    S_IFCHR   0020000  Character device
    S_IFIFO   0010000  Named pipe (FIFO)

  Bits 11-9: Special bits
    S_ISUID   0004000  Set-user-ID
    S_ISGID   0002000  Set-group-ID
    S_ISVTX   0001000  Sticky bit

  Bits 8-0: Permission bits (rwx for owner, group, others)
    S_IRWXU   0000700  Owner: rwx
    S_IRWXG   0000070  Group: rwx
    S_IRWXO   0000007  Other: rwx

  Example: 0100644 = Regular file, rw-r--r--
```

### inode_operations

```c
struct inode_operations {
    /* Lookup: find child inode in directory */
    struct dentry *(*lookup)(struct inode *dir, struct dentry *child,
                             unsigned int flags);
    
    /* Create file */
    int (*create)(struct mnt_idmap *, struct inode *dir,
                  struct dentry *, umode_t, bool excl);
    
    /* Hard link */
    int (*link)(struct dentry *old, struct inode *dir,
                struct dentry *new);
    
    /* Delete file */
    int (*unlink)(struct inode *dir, struct dentry *);
    
    /* Symlink */
    int (*symlink)(struct mnt_idmap *, struct inode *dir,
                   struct dentry *, const char *);
    
    /* Create directory */
    int (*mkdir)(struct mnt_idmap *, struct inode *dir,
                 struct dentry *, umode_t);
    
    /* Remove directory */
    int (*rmdir)(struct inode *dir, struct dentry *);
    
    /* Create device file */
    int (*mknod)(struct mnt_idmap *, struct inode *dir,
                 struct dentry *, umode_t, dev_t);
    
    /* Rename */
    int (*rename)(struct mnt_idmap *, struct inode *old_dir,
                  struct dentry *old, struct inode *new_dir,
                  struct dentry *new, unsigned int flags);
    
    /* Get attributes (stat) */
    int (*getattr)(struct mnt_idmap *, const struct path *,
                   struct kstat *, u32, unsigned int);
    
    /* Set attributes (chmod, chown, truncate) */
    int (*setattr)(struct mnt_idmap *, struct dentry *, struct iattr *);
    
    /* Extended attributes */
    ssize_t (*listxattr)(struct dentry *, char *, size_t);
    
    /* Permission check */
    int (*permission)(struct mnt_idmap *, struct inode *, int);
    
    /* Atomic open (combined lookup+create+open) */
    int (*atomic_open)(struct inode *, struct dentry *,
                       struct file *, unsigned open_flag,
                       umode_t create_mode);
};
```

### ext4 Inode Extension

```c
/* fs/ext4/ext4.h — ext4 embeds its data in the inode via container_of */

struct ext4_inode_info {
    __le32  i_data[15];           /* Block pointers / extent tree root */
    __u32   i_dtime;              /* Deletion time */
    __u32   i_flags;              /* ext4 inode flags (EXT4_EXTENTS_FL, etc.) */
    __u32   i_block_group;        /* Block group this inode belongs to */
    ext4_lblk_t i_dir_start_lookup; /* Directory hash tree start */
    
    /* Extent tree */
    struct ext4_ext_path *i_ext_cache; /* Cached extent path */
    
    /* Reserved for ext4 specific fields */
    struct inode vfs_inode;       /* The VFS inode (must be last!) */
};

/* Access ext4-specific data from VFS inode */
static inline struct ext4_inode_info *EXT4_I(struct inode *inode)
{
    return container_of(inode, struct ext4_inode_info, vfs_inode);
}

/* How container_of works:
   struct ext4_inode_info
     ┌──────────────────────┐
     │ i_data[15]           │
     │ i_dtime              │
     │ i_flags              │  ext4-specific fields
     │ i_block_group        │
     │ ...                  │
     ├──────────────────────┤
     │ struct inode vfs_inode│  ← VFS passes pointer to THIS
     │   .i_ino             │
     │   .i_mode            │
     │   .i_size            │
     └──────────────────────┘

   container_of(inode_ptr, struct ext4_inode_info, vfs_inode)
     → subtracts offset of vfs_inode to get pointer to ext4_inode_info
*/
```

---

## 6.3 struct dentry — Path Component Cache

### Complete Structure

```c
/* include/linux/dcache.h */

struct dentry {
    /* Flags */
    unsigned int d_flags;              /* Dentry flags (DCACHE_xxx) */

    /* RCU / Sequence lock for lock-free lookups */
    seqcount_spinlock_t d_seq;

    /* Hash table linkage */
    struct hlist_bl_node d_hash;       /* Hash table bucket chain */

    /* Tree structure */
    struct dentry *d_parent;           /* Parent dentry */
    struct qstr d_name;                /* Name of this component */
    /*   struct qstr {
     *       union {
     *           struct { u32 hash; u32 len; };
     *           u64 hash_len;
     *       };
     *       const unsigned char *name;
     *   };
     */

    /* Associated inode */
    struct inode *d_inode;             /* NULL for negative dentry */

    /* Short name storage (avoids separate allocation for short names) */
    unsigned char d_iname[DNAME_INLINE_LEN]; /* Inline name (32 bytes) */

    /* Reference count */
    struct lockref d_lockref;          /* Per-dentry lock + refcount */

    /* Operations */
    const struct dentry_operations *d_op;

    /* Superblock */
    struct super_block *d_sb;

    /* FS-specific data */
    void *d_fsdata;

    /* Child/sibling lists for directory tree */
    struct list_head d_lru;            /* LRU list */
    struct list_head d_child;          /* Sibling list (parent→d_subdirs) */
    struct list_head d_subdirs;        /* Children of this dentry */

    /* Alias list (multiple dentries pointing to same inode) */
    union {
        struct hlist_node d_alias;     /* inode alias list */
        struct hlist_bl_node d_in_lookup_hash; /* During lookup */
        struct rcu_head d_rcu;         /* For RCU-delayed freeing */
    } d_u;
};
```

### Dentry Tree Example

```
File: /home/user/documents/report.txt

Dentry tree in memory:
  [/] (d_inode=inode#2)
   └── d_subdirs:
        [home] (d_parent=[/], d_inode=inode#100)
         └── d_subdirs:
              [user] (d_parent=[home], d_inode=inode#1001)
               └── d_subdirs:
                    [documents] (d_parent=[user], d_inode=inode#1050)
                     └── d_subdirs:
                          [report.txt] (d_parent=[documents], d_inode=inode#1099)

Each dentry:
  - Points to parent (d_parent)
  - Points to inode (d_inode) → metadata
  - Listed in parent's d_subdirs
  - Hashed in dcache for O(1) lookup
  - Name stored inline if ≤ 32 chars, otherwise kmalloc'd
```

### dentry_operations

```c
struct dentry_operations {
    /* Revalidate: is this cached dentry still valid? */
    int (*d_revalidate)(struct dentry *, unsigned int flags);
    /* Used by NFS, AFS (remote FS where server may have changed) */

    /* Weak revalidate: quick check during RCU-walk */
    int (*d_weak_revalidate)(struct dentry *, unsigned int flags);

    /* Custom hash function */
    int (*d_hash)(const struct dentry *, struct qstr *);
    /* Used by case-insensitive FS */

    /* Custom name comparison */
    int (*d_compare)(const struct dentry *, unsigned int,
                     const char *, const struct qstr *);
    /* Used by case-insensitive FS (e.g., VFAT) */

    /* Called when dentry is about to be freed */
    void (*d_release)(struct dentry *);

    /* Called when last ref dropped */
    void (*d_iput)(struct dentry *, struct inode *);

    /* Generate name for d_path() display */
    char *(*d_dname)(struct dentry *, char *, int);

    /* Automount trigger */
    struct vfsmount *(*d_automount)(struct path *);

    /* Manage transit (cross mount point) */
    int (*d_manage)(const struct path *, bool);

    /* Pruning from dcache */
    void (*d_prune)(struct dentry *);
};
```

---

## 6.4 struct file — Open File Description

### Complete Structure

```c
/* include/linux/fs.h — struct file (key fields) */

struct file {
    /* Linked list for superblock's open files */
    union {
        struct llist_node   f_llist;
        struct rcu_head     f_rcuhead;
        unsigned int        f_iocb_flags;
    };

    /* Path (dentry + mount) */
    struct path             f_path;
    /* struct path {
     *     struct vfsmount *mnt;    ← Mount point
     *     struct dentry *dentry;   ← File's dentry
     * };
     */

    /* Inode shortcut */
    struct inode            *f_inode;    /* Cached inode pointer */

    /* Operations */
    const struct file_operations *f_op;  /* File operation table */

    /* Locking */
    spinlock_t              f_lock;

    /* State */
    atomic_long_t           f_count;     /* Reference count */
    unsigned int            f_flags;     /* O_RDONLY, O_NONBLOCK, O_APPEND, etc. */
    fmode_t                 f_mode;      /* FMODE_READ, FMODE_WRITE */

    /* Position */
    struct mutex            f_pos_lock;  /* Protects f_pos */
    loff_t                  f_pos;       /* Current file position (seek offset) */

    /* Credentials of opener */
    const struct cred       *f_cred;     /* Credentials at open() time */

    /* Owner (for FASYNC) */
    struct fown_struct      f_owner;

    /* Readahead state */
    struct file_ra_state    f_ra;        /* Readahead tracking */

    /* Page cache mapping */
    struct address_space    *f_mapping;  /* Page cache for this file */

    /* FS-private data */
    void                    *private_data; /* FS or driver private pointer */

    /* Error state */
    errseq_t                f_wb_err;    /* Writeback error tracking */
};
```

### file_operations (Complete)

```c
struct file_operations {
    struct module *owner;

    /* Seek */
    loff_t (*llseek)(struct file *, loff_t, int);

    /* Read / Write */
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    ssize_t (*read_iter)(struct kiocb *, struct iov_iter *);
    ssize_t (*write_iter)(struct kiocb *, struct iov_iter *);

    /* Async I/O */
    int (*iopoll)(struct kiocb *kiocb, struct io_comp_batch *, unsigned int);

    /* Directory iteration */
    int (*iterate_shared)(struct file *, struct dir_context *);

    /* Poll (select/poll/epoll) */
    __poll_t (*poll)(struct file *, struct poll_table_struct *);

    /* ioctl */
    long (*unlocked_ioctl)(struct file *, unsigned int, unsigned long);
    long (*compat_ioctl)(struct file *, unsigned int, unsigned long);

    /* Memory mapping */
    int (*mmap)(struct file *, struct vm_area_struct *);

    /* Open / Release */
    int (*open)(struct inode *, struct file *);
    int (*release)(struct inode *, struct file *);

    /* Sync */
    int (*fsync)(struct file *, loff_t, loff_t, int datasync);
    int (*fasync)(int, struct file *, int);

    /* File locking */
    int (*lock)(struct file *, int, struct file_lock *);
    int (*flock)(struct file *, int, struct file_lock *);

    /* Splice (zero-copy) */
    ssize_t (*splice_write)(struct pipe_inode_info *, struct file *,
                            loff_t *, size_t, unsigned int);
    ssize_t (*splice_read)(struct file *, loff_t *,
                           struct pipe_inode_info *, size_t, unsigned int);

    /* Fallocate (preallocate space) */
    long (*fallocate)(struct file *, int, loff_t, loff_t);

    /* Copy file range (server-side copy for NFS, reflink for Btrfs) */
    ssize_t (*copy_file_range)(struct file *, loff_t, struct file *,
                               loff_t, size_t, unsigned int);

    /* Clone / Dedupe range */
    int (*remap_file_range)(struct file *, loff_t, struct file *,
                            loff_t, loff_t, unsigned int);

    /* io_uring integration */
    int (*uring_cmd)(struct io_uring_cmd *, unsigned int);
};
```

### ext4 file_operations Example

```c
/* fs/ext4/file.c */
const struct file_operations ext4_file_operations = {
    .llseek         = ext4_llseek,
    .read_iter      = ext4_file_read_iter,
    .write_iter     = ext4_file_write_iter,
    .iopoll         = iocb_bio_iopoll,
    .unlocked_ioctl = ext4_ioctl,
    .compat_ioctl   = ext4_compat_ioctl,
    .mmap           = ext4_file_mmap,
    .open           = ext4_file_open,
    .release        = ext4_release_file,
    .fsync          = ext4_sync_file,
    .splice_read    = ext4_file_splice_read,
    .splice_write   = iter_file_splice_write,
    .fallocate      = ext4_fallocate,
    .copy_file_range = ext4_copy_file_range,
};
```

---

## 6.5 struct address_space — Page Cache Mapping

```c
/* include/linux/fs.h */

struct address_space {
    struct inode        *host;          /* Owning inode */
    struct xarray       i_pages;        /* Page cache (radix tree → xarray) */
    struct rw_semaphore invalidate_lock;/* Lock for page invalidation */
    gfp_t               gfp_mask;      /* Memory allocation flags */
    atomic_t            i_mmap_writable;/* Writable mmap count */
    struct rb_root_cached i_mmap;       /* Tree of private + shared mappings */
    unsigned long       nrpages;        /* Number of cached pages */
    pgoff_t             writeback_index;/* Writeback offset */
    const struct address_space_operations *a_ops; /* Page cache ops */
    unsigned long       flags;          /* AS_xxx flags */
    errseq_t            wb_err;         /* Writeback error */
    spinlock_t          i_lock;         /* Protects i_pages and counts */
    struct list_head    i_private_list; /* ditto */
};
```

### address_space_operations

```c
struct address_space_operations {
    /* Write dirty page to disk */
    int (*writepage)(struct page *page, struct writeback_control *wbc);
    int (*writepages)(struct address_space *, struct writeback_control *);

    /* Read page from disk */
    int (*read_folio)(struct file *, struct folio *);
    void (*readahead)(struct readahead_control *);

    /* Prepare/commit write to page cache */
    int (*write_begin)(struct file *, struct address_space *,
                       loff_t pos, unsigned len,
                       struct page **pagep, void **fsdata);
    int (*write_end)(struct file *, struct address_space *,
                     loff_t pos, unsigned len, unsigned copied,
                     struct page *page, void *fsdata);

    /* Mark folio as dirty */
    bool (*dirty_folio)(struct address_space *, struct folio *);

    /* Invalidate a folio */
    void (*invalidate_folio)(struct folio *, size_t offset, size_t len);

    /* Release a folio */
    bool (*release_folio)(struct folio *, gfp_t);

    /* Direct I/O */
    ssize_t (*direct_IO)(struct kiocb *, struct iov_iter *iter);

    /* Swap */
    int (*swap_activate)(struct swap_info_struct *, struct file *,
                         sector_t *);
};
```

---

## 6.6 Relationship Map

```
                    struct super_block
                    ┌──────────────────┐
                    │ s_dev            │
                    │ s_blocksize      │
                    │ s_root ──────────┼──────────► root dentry
                    │ s_op (super_ops) │
                    │ s_fs_info ───────┼──────────► ext4_sb_info
                    │ s_bdev ──────────┼──────────► block_device
                    └────────┬─────────┘
                             │ i_sb
                             │
                    struct inode
                    ┌──────────────────┐
                    │ i_ino            │    i_mapping
                    │ i_mode           │───────────► struct address_space
                    │ i_size           │             │ i_pages (xarray)
                    │ i_op (inode_ops) │             │ a_ops
                    │ i_fop (file_ops) │             │ nrpages
                    │ i_sb ────────────┼──► super    └──────────────────
                    └────────┬─────────┘
                             │ d_inode
                             │
                    struct dentry
                    ┌──────────────────┐
                    │ d_name           │
                    │ d_inode ─────────┼──────────► inode
                    │ d_parent ────────┼──────────► parent dentry
                    │ d_subdirs        │──► child dentries
                    │ d_sb ────────────┼──► super_block
                    │ d_op (dentry_ops)│
                    └────────┬─────────┘
                             │ f_path.dentry
                             │
                    struct file
                    ┌──────────────────┐
                    │ f_path           │──► dentry + vfsmount
                    │ f_inode ─────────┼──────────► inode (shortcut)
                    │ f_op (file_ops)  │
                    │ f_pos            │  (seek position)
                    │ f_flags          │  (O_RDONLY, O_APPEND...)
                    │ f_mapping ───────┼──► address_space
                    │ f_count          │  (reference count)
                    └──────────────────┘
                             ▲
                             │ fdt->fd[fd]
                    struct files_struct
                    ┌──────────────────┐
                    │ fdt->fd[0]=stdin │
                    │ fdt->fd[1]=stdout│
                    │ fdt->fd[2]=stderr│
                    │ fdt->fd[3]=file  │──► struct file above
                    └──────────────────┘
                             ▲
                             │ current->files
                    struct task_struct (process)
```

---

## 6.7 Memory Allocation for VFS Objects

```
VFS uses slab caches for efficient allocation:

Object           │ Slab Cache Name          │ Allocation Function
─────────────────┼──────────────────────────┼──────────────────────
struct inode     │ ext4_inode_cache         │ ext4_alloc_inode()
(FS-specific)    │ xfs_inode               │ xfs_inode_alloc()
                 │ btrfs_inode             │ btrfs_alloc_inode()
struct dentry    │ dentry                   │ d_alloc()
struct file      │ filp                     │ alloc_empty_file()

Why slab caches?
  - Objects are same size → no fragmentation
  - Pre-constructed → fast allocation
  - Per-CPU caching → no lock contention
  - Constructor/destructor → proper initialization

Creating an FS-specific inode cache:
*/

/* fs/ext4/super.c */
static struct kmem_cache *ext4_inode_cachep;

static void __init ext4_init_inode_cache(void)
{
    ext4_inode_cachep = kmem_cache_create("ext4_inode_cache",
        sizeof(struct ext4_inode_info), 0,
        SLAB_RECLAIM_ACCOUNT | SLAB_MEM_SPREAD | SLAB_ACCOUNT,
        init_once);  /* Constructor: called on first alloc */
}

static struct inode *ext4_alloc_inode(struct super_block *sb)
{
    struct ext4_inode_info *ei;
    ei = alloc_inode_sb(sb, ext4_inode_cachep, GFP_NOFS);
    if (!ei)
        return NULL;
    /* Initialize ext4-specific fields */
    ei->i_flags = 0;
    ei->i_block_group = 0;
    return &ei->vfs_inode;  /* Return embedded VFS inode */
}
```

---

## 6.8 Observing VFS Structures

### From User Space

```bash
# See inode information
$ ls -li /etc/passwd
131074 -rw-r--r-- 1 root root 2584 Jan 15 10:30 /etc/passwd
#│      │         │ │    │    │    │              └── filename
#│      │         │ │    │    │    └── mtime
#│      │         │ │    │    └── size (bytes)
#│      │         │ │    └── gid
#│      │         │ └── uid
#│      │         └── nlink
#│      └── mode (type + permissions)
#└── inode number

# Detailed stat
$ stat /etc/passwd
  File: /etc/passwd
  Size: 2584       Blocks: 8          IO Block: 4096   regular file
Device: 802h/2050d  Inode: 131074     Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2024-01-15 10:30:00.000000000 +0000
Modify: 2024-01-15 10:30:00.000000000 +0000
Change: 2024-01-15 10:30:00.000000000 +0000
 Birth: 2024-01-10 08:00:00.000000000 +0000

# File descriptor table of a process
$ ls -la /proc/self/fd/
lrwx------ 1 user user 64 Jan 15 12:00 0 -> /dev/pts/0
lrwx------ 1 user user 64 Jan 15 12:00 1 -> /dev/pts/0
lrwx------ 1 user user 64 Jan 15 12:00 2 -> /dev/pts/0
lr-x------ 1 user user 64 Jan 15 12:00 3 -> /proc/self/fd

# Mount info
$ cat /proc/self/mountinfo
28 1 8:2 / / rw,relatime - ext4 /dev/sda2 rw,errors=remount-ro

# Slab cache stats
$ sudo slabtop -o | grep -E "dentry|inode|filp"
  12000  10234  85%    0.19K    600     20      2400K  dentry
   8000   7543  94%    1.06K    250     32      8000K  ext4_inode_cache
   2000   1876  93%    0.25K    125     16       500K  filp
```

---

## Interview Questions

1. **List the key fields of struct super_block and explain each.**
2. **How does ext4 extend the VFS inode? Explain the container_of pattern.**
3. **What is a dentry? How does it differ from an inode?**
4. **Why does struct file have both f_path.dentry and f_inode? Isn't one redundant?**
5. **What is struct address_space? How does it connect inodes to the page cache?**
6. **Explain the dentry name storage optimization (d_iname).**
7. **Draw the complete relationship between task_struct, files_struct, file, dentry, inode, and super_block.**
8. **Why does Linux use slab caches for VFS objects?**
9. **What happens to VFS objects when a file system is unmounted?**
10. **What is dentry_operations->d_revalidate? When is it needed?**

---

## Summary

- **super_block**: One per mounted FS, holds FS config, root dentry, s_op table
- **inode**: One per file, holds metadata (mode, size, timestamps), i_op + i_fop tables
- **dentry**: One per path component, cached in hash table, links names to inodes
- **file**: One per open(), holds position (f_pos), flags, ref to dentry/inode
- **address_space**: One per inode, manages page cache via xarray (i_pages)
- FS-specific data stored via container_of (ext4_inode_info embeds struct inode)
- VFS objects allocated from slab caches for performance
- All connected: process → fd table → file → dentry → inode → superblock

---

*Next: [Chapter 7 — File System Operations](Chapter_07_FS_Operations.md)*
