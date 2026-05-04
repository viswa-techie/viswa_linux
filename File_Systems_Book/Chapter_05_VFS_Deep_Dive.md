# Chapter 5: Virtual File System (VFS) Deep Dive

## Learning Goals
- Understand VFS design philosophy and architecture
- Master the VFS dispatch mechanism (function pointer tables)
- Trace how VFS routes system calls to file system implementations
- Know VFS caching strategies (dentry cache, inode cache)

---

## 5.1 VFS Design Philosophy

```
The VFS provides a common framework so that:

  1. Applications use ONE set of system calls (open, read, write, stat...)
  2. Each file system implements its own backend logic
  3. The kernel dispatches calls through function pointer tables

This is the "Strategy Pattern" / "Polymorphism" in C:

  ┌─────────────────────────────────────────────────┐
  │              VFS "Interface"                     │
  │  ┌────────────┐ ┌────────────┐ ┌────────────┐  │
  │  │file_       │ │inode_      │ │super_      │  │
  │  │operations  │ │operations  │ │operations  │  │
  │  │ .read_iter │ │ .lookup    │ │ .statfs    │  │
  │  │ .write_iter│ │ .create    │ │ .sync_fs   │  │
  │  │ .open      │ │ .unlink    │ │ .alloc_inode│ │
  │  └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ │
  │        │              │              │          │
  └────────┼──────────────┼──────────────┼──────────┘
           │              │              │
  ┌────────▼──┐   ┌──────▼──┐   ┌──────▼──────┐
  │ ext4      │   │ XFS     │   │ Btrfs       │
  │ ext4_file_│   │ xfs_file│   │ btrfs_file_ │
  │ read_iter │   │ _read_  │   │ read_iter   │
  │           │   │ iter    │   │             │
  └───────────┘   └─────────┘   └─────────────┘
```

### VFS vs Object-Oriented Design

```
C doesn't have classes, but VFS uses the same patterns:

OOP Concept            │ VFS C Implementation
───────────────────────┼──────────────────────────────
Interface              │ struct xxx_operations (function pointers)
Abstract class         │ VFS objects (inode, dentry, file, super_block)
Concrete class         │ FS-specific implementation (ext4, XFS)
Polymorphic dispatch   │ obj->operations->method()
Inheritance            │ Container: FS struct embeds VFS struct
Constructor            │ alloc_inode() / d_alloc() / get_empty_filp()
Destructor             │ destroy_inode() / d_release() / fput()

Example "polymorphic call":
  file->f_op->read_iter(kiocb, iter);
  // If file is on ext4 → calls ext4_file_read_iter()
  // If file is on XFS  → calls xfs_file_read_iter()
  // If file is on NFS  → calls nfs_file_read()
```

---

## 5.2 VFS Object Lifecycle

### Superblock Lifecycle

```
mount("ext4", "/dev/sda1", "/mnt", ...)
  │
  ▼
vfs_get_tree()                          [fs/super.c]
  │
  ▼
ext4_init_fs_context()                  [fs/ext4/super.c]
  │
  ▼
ext4_fill_super()                       ← Read superblock from disk
  ├── Allocate struct super_block
  ├── Read ext4 on-disk superblock (block group 0)
  ├── Initialize s_op = &ext4_sops
  ├── Allocate root inode
  ├── Create root dentry (d_make_root)
  └── Return mounted FS

umount("/mnt")
  │
  ▼
generic_shutdown_super()
  ├── sync_filesystem()                 ← Flush dirty data
  ├── s_op->put_super()                ← FS-specific cleanup
  └── Free super_block
```

### Inode Lifecycle

```
Opening a file triggers inode lookup:

1. Path lookup: /home/user/file.txt
   │
   ▼
2. For each component (/home, /user, /file.txt):
   dentry cache lookup
   │
   ├── Dentry HIT → inode pointer in dentry → done
   │
   └── Dentry MISS → parent_inode->i_op->lookup()
         │
         ▼
       ext4_lookup()
         ├── Search directory entries on disk
         ├── Find inode number for "file.txt"
         ├── iget_locked(sb, ino)         ← lookup/create inode
         │     ├── Check inode hash table
         │     ├── If not found: alloc_inode() → read from disk
         │     └── Return struct inode
         └── d_splice_alias()             ← connect dentry ↔ inode

Inode eviction:
  When memory pressure or FS unmount:
    evict_inode() → ext4_evict_inode()
      ├── Truncate page cache for this inode
      ├── If inode->i_nlink == 0, free disk blocks
      └── Free in-memory inode
```

### Dentry Lifecycle

```
Dentry states:
  ┌──────────────────────────┐
  │ d_alloc() → UNUSED       │  ← Just allocated, no inode
  │     │                    │
  │     ▼                    │
  │ d_splice_alias() →       │
  │   IN-USE (positive)      │  ← Connected to valid inode
  │     │                    │
  │     ├─── unlink()        │
  │     │      ▼             │
  │     │  NEGATIVE          │  ← Dentry exists but inode gone
  │     │  (caches "not found")   (speeds up failed lookups)
  │     │                    │
  │     ├─── Memory pressure │
  │     │      ▼             │
  │     │  LRU → d_free()   │  ← Released back to slab
  │     │                    │
  │     └─── close last ref  │
  │            ▼             │
  │         UNUSED → LRU    │  ← Can be reclaimed
  └──────────────────────────┘

Negative dentry optimization:
  $ stat /nonexistent   ← First call: disk lookup, ENOENT
  $ stat /nonexistent   ← Second call: negative dentry HIT (fast!)
```

### File Lifecycle

```
fd = open("/path/to/file", O_RDWR)
  │
  ▼
do_sys_openat2()                        [fs/open.c]
  │
  ▼
do_filp_open()
  ├── path_openat()                     ← Path resolution
  │     └── Resolve dentries → final inode
  │
  ├── alloc_empty_file()                ← Allocate struct file
  │     f->f_inode = inode
  │     f->f_op = inode->i_fop         ← Copy file_operations
  │     f->f_pos = 0                   ← File position starts at 0
  │     f->f_mode = FMODE_READ|FMODE_WRITE
  │
  ├── f->f_op->open()                  ← FS-specific open
  │     → ext4_file_open()
  │
  └── fd_install(fd, f)                ← Install in process fd table
        current->files->fdt[fd] = f

close(fd)
  │
  ▼
filp_close()
  ├── f->f_op->flush()                 ← Optional flush
  └── fput(f)
        ├── Decrement f_count
        └── If f_count == 0:
              f->f_op->release()       ← FS-specific release
              dput(f->f_path.dentry)   ← Release dentry ref
              mntput(f->f_path.mnt)    ← Release mount ref
              kmem_cache_free(filp_cachep, f)
```

---

## 5.3 The Dentry Cache (dcache)

### Architecture

```
The dentry cache is one of the most performance-critical caches in Linux.
Every path lookup uses dcache. Without it, every component in
/home/user/project/src/main.c would require a disk read.

dcache structure:
  ┌─────────────────────────────────────────┐
  │             dcache hash table            │
  │  ┌─────────────────────────────────┐    │
  │  │ Bucket 0: dentry → dentry → ... │    │
  │  │ Bucket 1: dentry → dentry → ... │    │
  │  │ ...                             │    │
  │  │ Bucket N: dentry → dentry → ... │    │
  │  └─────────────────────────────────┘    │
  │                                         │
  │  Hash key: (parent dentry pointer,      │
  │             name hash, name length)     │
  │                                         │
  │  Lookup: O(1) average                   │
  └─────────────────────────────────────────┘

  Also maintains:
    - LRU list for unused dentries (reclaimable under memory pressure)
    - Per-superblock list (s_dentries)
    - Parent-child tree (d_subdirs linked list)
```

### dcache Lookup Algorithm

```c
/* Simplified dcache lookup path (fs/dcache.c) */

struct dentry *d_lookup(const struct dentry *parent, const struct qstr *name)
{
    unsigned int hash = name->hash;
    struct hlist_bl_head *b = d_hash(hash);  /* hash table bucket */
    struct dentry *dentry;

    hlist_bl_for_each_entry_rcu(dentry, node, b, d_hash) {
        if (dentry->d_name.hash != hash)
            continue;
        if (dentry->d_parent != parent)
            continue;
        if (!d_same_name(dentry, parent, name))
            continue;
        /* Found it! */
        return dentry;
    }
    return NULL;  /* Not in cache */
}
```

### dcache Statistics

```bash
# View dcache usage:
$ cat /proc/sys/fs/dentry-state
76543  34567  45  0  0  0
 │      │     │
 │      │     └── age_limit (not used)
 │      └── unused dentries (on LRU, reclaimable)
 └── total allocated dentries

# Slab allocator stats for dentry:
$ slabtop | grep dentry
  OBJS  ACTIVE  USE  OBJ SIZE  SLABS OBJ/SLAB  CACHE SIZE  NAME
 12345  10234  82%    0.19K    617      20       2468K  dentry
```

---

## 5.4 The Inode Cache (icache)

```
Similar to dcache but for inodes:

  ┌─────────────────────────────────────────┐
  │            Inode hash table              │
  │  Key: (superblock pointer, inode number) │
  │                                         │
  │  Bucket 0: inode → inode → ...          │
  │  Bucket 1: inode → inode → ...          │
  │  ...                                    │
  │                                         │
  │  Functions:                             │
  │    iget_locked(sb, ino) → find or alloc │
  │    iput(inode) → release reference      │
  │    ilookup(sb, ino) → find existing     │
  └─────────────────────────────────────────┘

States: I_NEW → I_FREEING → (freed)
        I_DIRTY_SYNC | I_DIRTY_DATASYNC | I_DIRTY_PAGES

# View inode cache stats:
$ cat /proc/sys/fs/inode-state
87654  12345  0  0  0  0  0
 │      │
 │      └── free inodes (not in use, can be freed)
 └── allocated inodes

$ slabtop | grep inode
  OBJS  ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE  NAME
  9876   8765  88%   1.06K    330      30      10560K  ext4_inode_cache
```

---

## 5.5 File Descriptor Table

```
Process File Descriptor Architecture:

struct task_struct (process)
  │
  └── struct files_struct *files
        │
        ├── atomic_t count         ← ref count (shared via clone)
        ├── struct fdtable *fdt
        │     │
        │     ├── unsigned int max_fds      ← current table size
        │     ├── struct file **fd          ← array of file pointers
        │     │     ┌───────────────────────────────────┐
        │     │     │ [0] → struct file (stdin)         │
        │     │     │ [1] → struct file (stdout)        │
        │     │     │ [2] → struct file (stderr)        │
        │     │     │ [3] → struct file (/etc/passwd)   │
        │     │     │ [4] → NULL (unused)               │
        │     │     │ [5] → struct file (socket)        │
        │     │     │ ...                               │
        │     │     └───────────────────────────────────┘
        │     │
        │     ├── unsigned long *open_fds   ← bitmap of open fds
        │     └── unsigned long *close_on_exec ← bitmap for CLOEXEC
        │
        └── int next_fd            ← hint for next fd to allocate

  fd = open(path, flags)
    → finds lowest unused slot in fdt->fd[]
    → stores struct file pointer there
    → returns the index (the fd number)

  read(fd, buf, count)
    → current->files->fdt->fd[fd] → struct file → f_op->read_iter()
```

---

## 5.6 VFS System Call Dispatch

### open() Complete Path

```
User: open("/home/user/file.txt", O_RDWR, 0644)
  │
  ▼
SYSCALL_DEFINE3(open, ...)                  [fs/open.c]
  │
  ▼
do_sys_openat2(AT_FDCWD, filename, &how)
  │
  ├── build_open_flags()                    ← Parse O_RDWR, O_CREAT etc.
  │
  ├── getname(filename)                     ← Copy path from user space
  │     → struct filename with {name="/home/user/file.txt"}
  │
  ├── get_unused_fd_flags()                 ← Find free fd number
  │
  ├── do_filp_open(dfd, name, op)
  │     │
  │     ├── path_openat(nd, op, flags)
  │     │     │
  │     │     ├── path_init()               ← Start from / or cwd
  │     │     │
  │     │     ├── link_path_walk("home/user/file.txt")
  │     │     │     │
  │     │     │     ├── "home" → d_lookup(root, "home")
  │     │     │     │     ├── dcache HIT → advance
  │     │     │     │     └── dcache MISS → inode->i_op->lookup()
  │     │     │     │
  │     │     │     ├── "user" → d_lookup(home_dentry, "user")
  │     │     │     │
  │     │     │     └── "file.txt" → d_lookup(user_dentry, "file.txt")
  │     │     │
  │     │     ├── do_open()                 ← Actually open the file
  │     │     │     ├── may_open()          ← Permission check
  │     │     │     ├── vfs_open()
  │     │     │     │     ├── alloc_empty_file()
  │     │     │     │     ├── do_dentry_open()
  │     │     │     │     │     ├── f->f_op = inode->i_fop
  │     │     │     │     │     └── f->f_op->open(inode, file)
  │     │     │     │     │           → ext4_file_open()
  │     │     │     │     └── return struct file
  │     │     │
  │     │     └── Return opened struct file
  │     │
  │     └── Return struct file
  │
  ├── fd_install(fd, file)                  ← Install in fd table
  │
  └── return fd                             ← Back to user space
```

### read() Complete Path

```
User: read(3, buf, 4096)
  │
  ▼
SYSCALL_DEFINE3(read, fd, buf, count)       [fs/read_write.c]
  │
  ▼
fdget_pos(fd)                               ← Get struct file from fd table
  │                                            current->files->fdt->fd[3]
  ▼
vfs_read(file, buf, count, &pos)
  │
  ├── rw_verify_area()                      ← Check file mode, limits
  │
  ├── file->f_op->read_iter(kiocb, iter)    ← Dispatch to FS
  │     │
  │     ├── [ext4] ext4_file_read_iter()
  │     │     └── generic_file_read_iter()
  │     │           └── filemap_read()       ← Page cache lookup
  │     │
  │     ├── [tmpfs] shmem_file_read_iter()   ← Read from RAM
  │     │
  │     └── [NFS] nfs_file_read()            ← Network RPC
  │
  ├── Update file position (f_pos)
  │
  └── Return bytes_read
```

---

## 5.7 VFS Locking

```
VFS uses multiple locks to protect shared state:

Lock                │ Protects              │ Type     │ Granularity
────────────────────┼────────────────────────┼──────────┼────────────
inode->i_rwsem      │ Inode data, size      │ rw_sem   │ Per-inode
inode->i_lock       │ Inode metadata fields │ spinlock │ Per-inode
sb->s_lock          │ Superblock fields     │ mutex    │ Per-FS
dentry->d_lock      │ Dentry fields         │ spinlock │ Per-dentry
rename_lock         │ dcache rename safety  │ seqlock  │ Global
file->f_lock        │ File position, flags  │ spinlock │ Per-file
mapping->i_pages    │ Page cache (xarray)   │ RCU +    │ Per-mapping
                    │                       │ xa_lock  │

Locking order (to prevent deadlocks):
  1. rename_lock (if needed)
  2. parent directory i_rwsem
  3. child directory/file i_rwsem
  4. inode->i_lock
  5. dentry->d_lock
```

### RCU in dcache (Lock-Free Fast Path)

```
Path lookup uses RCU for lock-free dcache traversal:

  1. rcu_read_lock()
  2. For each path component:
     a. d_lookup_rcu(parent, name)    ← No locks!
     b. Validate sequence counter
     c. If valid → advance to next component
     d. If invalid → fall back to ref-walk (with locks)
  3. rcu_read_unlock()

This "RCU-walk" makes path lookup incredibly fast
for the common case (cached paths, no concurrent modifications).

Fallback "ref-walk": Takes d_lock, increments refcount, normal locking.
```

---

## 5.8 VFS Mount Namespace

```
Each process has a mount namespace:

  struct task_struct
    └── struct nsproxy *nsproxy
          └── struct mnt_namespace *mnt_ns
                └── struct mount *root
                      └── Tree of struct mount objects

Mount namespaces enable:
  - Containers: each container sees its own mount tree
  - chroot-like isolation without chroot
  - Private mounts (per-process)

Propagation types:
  MS_SHARED     → mount events propagate to all peers
  MS_PRIVATE    → mount events are not shared
  MS_SLAVE      → receives events from master, doesn't send
  MS_UNBINDABLE → cannot be bind-mounted

Example:
  # Create private mount namespace
  $ unshare --mount /bin/bash
  # Mounts here are invisible to parent namespace
  $ mount -t tmpfs tmpfs /mnt
```

---

## Kernel Source References

```
VFS core implementation:
  fs/dcache.c           ← Dentry cache (d_lookup, d_alloc, d_instantiate)
  fs/inode.c            ← Inode cache (iget_locked, iput, ilookup)
  fs/namei.c            ← Path resolution (link_path_walk, filename_lookup)
  fs/open.c             ← open() syscall (do_sys_openat2, do_filp_open)
  fs/read_write.c       ← read/write syscalls (vfs_read, vfs_write)
  fs/file_table.c       ← struct file allocation (alloc_empty_file)
  fs/super.c            ← Superblock management
  fs/namespace.c         ← Mount operations and mount namespaces
  fs/file.c             ← fd table management (fd_install, close_fd)

Headers:
  include/linux/fs.h    ← struct inode, super_block, file_operations
  include/linux/dcache.h ← struct dentry, dentry_operations
  include/linux/mount.h ← struct vfsmount
  include/linux/namei.h ← Path resolution APIs
```

---

## Interview Questions

1. **How does VFS achieve polymorphism in C? Give a concrete example.**
2. **Explain the lifecycle of a dentry from allocation to freeing.**
3. **What is a negative dentry? Why is it useful?**
4. **How does the dentry cache work? What is the hash key?**
5. **Trace the complete path of open("/home/user/file.txt") through VFS.**
6. **What is RCU-walk in path lookup? When does it fall back to ref-walk?**
7. **How does VFS know which file_operations to use for a particular file?**
8. **What is the relationship between struct file and the process fd table?**
9. **Name the major VFS locks and their locking order.**
10. **What are mount namespaces? How do containers use them?**

---

## Summary

- VFS uses function pointer tables for polymorphic dispatch to file systems
- Four core objects: superblock (FS state), inode (file metadata), dentry (path cache), file (open file)
- dcache: hash table mapping (parent, name) → dentry for O(1) path lookup
- icache: hash table mapping (sb, ino) → inode
- RCU-walk enables lock-free path traversal for cached paths
- File descriptors are indices into per-process fd table pointing to struct file
- Mount namespaces provide per-process mount tree isolation for containers

---

*Next: [Chapter 6 — Core VFS Data Structures](Chapter_06_VFS_Data_Structures.md)*
