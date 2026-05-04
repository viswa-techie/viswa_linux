# Chapter 7: File System Operations — open, read, write, close

## Learning Goals
- Trace the complete kernel path for open(), read(), write(), close()
- Understand file position tracking and concurrent access
- Know the role of readahead in read performance
- Master direct I/O vs buffered I/O

---

## 7.1 open() — Opening a File

### System Call Entry

```c
/* fs/open.c */
SYSCALL_DEFINE3(open, const char __user *, filename,
                int, flags, umode_t, mode)
{
    /* Delegates to openat with AT_FDCWD (current working directory) */
    return do_sys_openat2(AT_FDCWD, filename,
                          &(struct open_how) {
                              .flags = flags,
                              .mode  = mode,
                          });
}
```

### Complete open() Flow

```
open("/home/user/file.txt", O_RDWR | O_CREAT, 0644)
  │
  ▼
do_sys_openat2(AT_FDCWD, filename, how)
  │
  ├── 1. build_open_flags()
  │       Parse flags: O_RDWR → FMODE_READ | FMODE_WRITE
  │       O_CREAT → LOOKUP_OPEN | LOOKUP_CREATE
  │
  ├── 2. getname(filename)
  │       Copy path from user space → struct filename
  │       If short enough, embed in stack; else kmalloc
  │
  ├── 3. get_unused_fd_flags(flags)
  │       Find lowest available fd in current->files->fdt
  │       Set CLOEXEC if O_CLOEXEC
  │       Returns fd number (e.g., 3)
  │
  ├── 4. do_filp_open(dfd, name, op)
  │     │
  │     └── path_openat(nd, op, flags)
  │           │
  │           ├── path_init(nd, flags)
  │           │     Start from root (/) or cwd based on path
  │           │     nd->path = root or cwd
  │           │
  │           ├── link_path_walk("home/user/file.txt", nd)
  │           │     For each component:
  │           │       ├── may_lookup() → permission check (execute on dir)
  │           │       ├── walk_component()
  │           │       │     ├── lookup_fast() → dcache lookup
  │           │       │     │     ├── d_lookup(parent, name) → HIT
  │           │       │     │     └── MISS → lookup_slow()
  │           │       │     │           ├── d_alloc_parallel()
  │           │       │     │           └── inode->i_op->lookup()
  │           │       │     │                 → ext4_lookup()
  │           │       │     │                 → find inode on disk
  │           │       │     │
  │           │       │     └── step_into() → advance nd to next component
  │           │       │
  │           │       └── Handle mount points, symlinks
  │           │
  │           ├── do_open(nd, file, op)
  │           │     ├── may_open() → permission check (read/write)
  │           │     ├── vfs_open(path, file)
  │           │     │     └── do_dentry_open(file, inode, open)
  │           │     │           ├── file->f_inode = inode
  │           │     │           ├── file->f_mapping = inode->i_mapping
  │           │     │           ├── file->f_op = fops_get(inode->i_fop)
  │           │     │           ├── security_file_open(file)
  │           │     │           └── file->f_op->open(inode, file)
  │           │     │                 → ext4_file_open()
  │           │     │
  │           │     └── If O_CREAT and file doesn't exist:
  │           │           lookup_open() → inode->i_op->create()
  │           │             → ext4_create()
  │           │
  │           └── return file
  │
  ├── 5. fd_install(fd, file)
  │       current->files->fdt->fd[fd] = file
  │       Now accessible via read(fd, ...)
  │
  └── return fd (e.g., 3)
```

### O_CREAT: Creating a New File

```
When O_CREAT is set and file doesn't exist:

  lookup_open()
    │
    ├── dir->i_op->create(idmap, dir, dentry, mode, excl)
    │     │
    │     └── ext4_create()                      [fs/ext4/namei.c]
    │           ├── ext4_new_inode_start_handle()
    │           │     ├── Start journal transaction
    │           │     ├── Find free inode in bitmap
    │           │     ├── Initialize inode (mode, uid, gid, timestamps)
    │           │     └── Return new inode
    │           │
    │           ├── ext4_add_nondir()
    │           │     └── ext4_add_entry()        ← Add to parent directory
    │           │           ├── Find space in directory block
    │           │           └── Write directory entry (name + inode number)
    │           │
    │           └── ext4_journal_stop()
    │
    └── d_instantiate(dentry, inode)              ← Connect dentry ↔ inode
```

---

## 7.2 read() — Reading File Data

### System Call Entry

```c
/* fs/read_write.c */
SYSCALL_DEFINE3(read, unsigned int, fd,
                char __user *, buf, size_t, count)
{
    return ksys_read(fd, buf, count);
}

ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count)
{
    struct fd f = fdget_pos(fd);            /* Get struct file from fd */
    if (!f.file)
        return -EBADF;
    
    loff_t pos = file_pos_read(f.file);     /* Current file position */
    ssize_t ret = vfs_read(f.file, buf, count, &pos);
    if (ret >= 0)
        file_pos_write(f.file, pos);        /* Update position */
    
    fdput_pos(f);
    return ret;
}
```

### vfs_read() Dispatch

```c
ssize_t vfs_read(struct file *file, char __user *buf,
                 size_t count, loff_t *pos)
{
    /* Security check */
    if (!(file->f_mode & FMODE_READ))
        return -EBADF;
    if (!file->f_op->read && !file->f_op->read_iter)
        return -EINVAL;
    
    ret = rw_verify_area(READ, file, pos, count);
    if (ret)
        return ret;
    
    /* Dispatch to file system */
    if (file->f_op->read)
        ret = file->f_op->read(file, buf, count, pos);
    else if (file->f_op->read_iter)
        ret = new_sync_read(file, buf, count, pos);
        /* → builds kiocb + iov_iter, calls f_op->read_iter() */
    
    return ret;
}
```

### Buffered Read Path (ext4)

```
read(fd, buf, 4096) where fd is on ext4
  │
  ▼
f_op->read_iter(kiocb, iter)
  │
  ▼
ext4_file_read_iter()                        [fs/ext4/file.c]
  │
  ├── If Direct I/O (O_DIRECT): ext4_dio_read_iter() → skip page cache
  │
  └── Buffered I/O: generic_file_read_iter()  [mm/filemap.c]
        │
        └── filemap_read(iocb, iter, retval)
              │
              ├── for each page needed:
              │     │
              │     ├── filemap_get_pages()
              │     │     │
              │     │     ├── filemap_get_read_batch()
              │     │     │     Look in page cache (xa_find in i_pages xarray)
              │     │     │
              │     │     ├── If page FOUND (cache hit):
              │     │     │     ├── Check if page is uptodate
              │     │     │     ├── If uptodate → use it
              │     │     │     └── If not → wait for I/O completion
              │     │     │
              │     │     └── If page NOT FOUND (cache miss):
              │     │           ├── page_cache_sync_readahead()
              │     │           │     Start readahead (read ahead window)
              │     │           │     a_ops->readahead()
              │     │           │       → ext4_readahead()
              │     │           │         → mpage_readahead()
              │     │           │           → submit_bio(READ, ...)
              │     │           │
              │     │           └── Wait for the needed page
              │     │
              │     └── copy_folio_to_iter()
              │           copy_to_user(buf, page_data, bytes)
              │
              └── Return bytes read
```

### Readahead

```
Readahead predicts sequential access and prefetches pages:

Sequential read pattern detected:
  read(fd, buf, 4096)   ← Want page 0
  read(fd, buf, 4096)   ← Want page 1 → sequential!

  Readahead window:
    ┌────┬────┬────┬────┬────┬────┬────┬────┐
    │ P0 │ P1 │ P2 │ P3 │ P4 │ P5 │ P6 │ P7 │
    └────┴────┴────┴────┴────┴────┴────┴────┘
     ▲requested       ▲readahead window (async prefetch)

  struct file_ra_state {
      pgoff_t start;          /* Readahead start */
      unsigned int size;      /* Readahead pages */
      unsigned int async_size;/* Async prefetch size */
      unsigned int ra_pages;  /* Max readahead (from bdi) */
  };

  Default max readahead: 128KB (32 pages on 4K systems)
  Tunable: /sys/block/sda/queue/read_ahead_kb

  Initial window: small (4 pages)
  Grows on sequential access: doubles up to max
  Random access: readahead disabled for that fd
```

---

## 7.3 write() — Writing File Data

### Buffered Write Path

```
write(fd, "hello world", 11)
  │
  ▼
vfs_write(file, buf, count, pos)              [fs/read_write.c]
  │
  ▼
f_op->write_iter(kiocb, iter)
  │
  ▼
ext4_file_write_iter()                        [fs/ext4/file.c]
  │
  ├── ext4_inode_attach_jinode()              ← Journal inode data
  │
  └── generic_file_write_iter()               [mm/filemap.c]
        │
        ├── file_remove_privs()               ← Clear setuid/setgid bits
        │
        └── generic_perform_write()
              │
              for each page of data:
              │
              ├── a_ops->write_begin()
              │     → ext4_write_begin()       [fs/ext4/inode.c]
              │       ├── ext4_journal_start() ← Start transaction
              │       ├── grab_cache_page_write_begin()
              │       │     ├── Find page in cache OR allocate new
              │       │     └── Lock the page
              │       ├── ext4_block_map() or ext4_da_map_blocks()
              │       │     ├── Delayed allocation: reserve metadata
              │       │     └── Or: allocate blocks now
              │       └── Return locked page ready for writing
              │
              ├── copy_page_from_iter(page, offset, bytes, iter)
              │     → Copy user data into page cache page
              │
              └── a_ops->write_end()
                    → ext4_write_end()
                      ├── ext4_journal_stop()
                      ├── block_write_end()    ← Mark buffers dirty
                      ├── mark_inode_dirty()   ← Update size, mtime
                      ├── set_page_dirty()     ← Mark page for writeback
                      └── unlock_page()
              
  At this point, data is in page cache (DIRTY), NOT on disk.
  write() returns to user space.
```

### Delayed Allocation (ext4)

```
With delayed allocation (default in ext4):

  write() time:
    ┌─────────────────────────┐
    │ Data in page cache      │  ← Only reserve space, don't allocate blocks
    │ (dirty pages)           │
    │ Blocks: NOT allocated   │  
    └─────────────────────────┘

  writeback time (background, or sync):
    ┌─────────────────────────┐
    │ ext4_writepages()       │
    │ ├── ext4_da_get_blocks()│  ← NOW allocate contiguous extent
    │ ├── write data to disk  │  ← I/O for data + metadata
    │ └── Clean dirty pages   │
    └─────────────────────────┘

  Benefits of delayed allocation:
    - Better block allocation (more data → better extent choices)
    - Fewer metadata updates (batch allocations)
    - Short-lived temp files may never write to disk
    - Reduced fragmentation

  Risk: Data loss window between write() and writeback
    write() returns → data in RAM only → power loss → DATA LOST
    Mitigated by: fsync(), O_SYNC, dirty_expire_centisecs (30s default)
```

---

## 7.4 Writeback — Flushing Dirty Pages to Disk

```
Dirty pages are written to disk by background threads:

  Trigger conditions:
    1. Timer: dirty_writeback_centisecs (default: 500 = 5 seconds)
    2. Memory pressure: too many dirty pages
    3. Explicit: sync(), fsync(), fdatasync()
    4. Threshold: vm.dirty_ratio (% of RAM) exceeded

  ┌─────────────────────────────────────────┐
  │  Background writeback flow              │
  │                                         │
  │  kworker/bdi thread wakes up            │
  │    │                                    │
  │    ▼                                    │
  │  wb_workfn() → wb_do_writeback()        │
  │    │                                    │
  │    ▼                                    │
  │  writeback_sb_inodes()                  │
  │    │                                    │
  │    ▼  for each dirty inode              │
  │  writeback_single_inode()               │
  │    │                                    │
  │    ▼                                    │
  │  a_ops->writepages() → ext4_writepages()│
  │    │                                    │
  │    ▼                                    │
  │  submit_bio() → Block layer → Disk      │
  └─────────────────────────────────────────┘

Tunables:
  /proc/sys/vm/dirty_background_ratio    = 10  (start writeback at 10% dirty)
  /proc/sys/vm/dirty_ratio               = 20  (block writers at 20% dirty)
  /proc/sys/vm/dirty_writeback_centisecs = 500 (writeback thread wakes every 5s)
  /proc/sys/vm/dirty_expire_centisecs    = 3000 (expire dirty data at 30s)
```

---

## 7.5 fsync() and fdatasync()

```
fsync(fd):
  Flush all dirty data AND metadata for this file to disk.
  Returns only when data is physically on stable storage.

fdatasync(fd):
  Flush dirty data + necessary metadata (size changes).
  Skips metadata that doesn't affect data retrieval (atime, mtime).
  Slightly faster than fsync().

  fsync(fd)
    │
    ▼
  vfs_fsync(file, 0)                         [fs/sync.c]
    │
    ▼
  file->f_op->fsync(file, start, end, datasync)
    │
    ▼
  ext4_sync_file()                            [fs/ext4/fsync.c]
    ├── filemap_write_and_wait_range()        ← Write dirty pages
    │     ├── filemap_fdatawrite_range()      ← Submit I/O
    │     └── filemap_fdatawait_range()       ← Wait for completion
    │
    ├── jbd2_complete_transaction()           ← Commit journal
    │     ├── Write journal entries to disk
    │     └── Wait for journal I/O completion
    │
    └── blkdev_issue_flush()                  ← Send cache flush to device
          → Ensures disk cache is written to media
```

---

## 7.6 close() — Closing a File

```
close(fd)
  │
  ▼
SYSCALL_DEFINE1(close, unsigned int, fd)      [fs/open.c]
  │
  ▼
close_fd(fd)
  │
  ├── Pick up struct file from fd table
  ├── Set fdt->fd[fd] = NULL                  ← fd is now free
  │
  └── filp_close(file, files)
        │
        ├── f->f_op->flush(file, id)          ← Optional (NFS uses this)
        │     Most FS don't implement flush
        │
        └── fput(file)
              │
              ├── atomic_long_dec(&f->f_count)
              │
              └── If count reaches 0:
                    ├── __fput(file)
                    │     ├── fsnotify_close(file)    ← inotify notification
                    │     ├── locks_remove_file(file)  ← Release file locks
                    │     ├── f->f_op->release(inode, file)
                    │     │     → ext4_release_file()
                    │     ├── dput(dentry)             ← Release dentry ref
                    │     ├── mntput(mnt)              ← Release mount ref
                    │     └── file_free(file)          ← Return to slab cache
                    │
                    └── ⚠ Deferred via task_work (runs at syscall exit)

IMPORTANT: close() does NOT write dirty data to disk!
  Dirty pages remain in page cache until writeback.
  Use fsync() before close() for data safety.
```

---

## 7.7 lseek() — Changing File Position

```c
/* fs/read_write.c */
SYSCALL_DEFINE3(lseek, unsigned int, fd, off_t, offset, unsigned int, whence)

/* whence values: */
SEEK_SET  (0)  /* Set position to offset */
SEEK_CUR  (1)  /* Set position to current + offset */
SEEK_END  (2)  /* Set position to file_size + offset */
SEEK_DATA (3)  /* Find next data after offset (sparse files) */
SEEK_HOLE (4)  /* Find next hole after offset (sparse files) */

/* Implementation: */
lseek(fd, offset, whence)
  │
  ▼
vfs_llseek(file, offset, whence)
  │
  ▼
file->f_op->llseek(file, offset, whence)
  │
  ├── generic_file_llseek()          ← Most regular files
  │     mutex_lock(&f->f_pos_lock)
  │     Compute new position
  │     f->f_pos = new_pos
  │     mutex_unlock(&f->f_pos_lock)
  │
  └── ext4_llseek()                  ← ext4 (supports SEEK_DATA/SEEK_HOLE)
        Handles sparse file seeking via extent tree
```

### Sparse Files and SEEK_HOLE

```
Sparse file:
  ┌────────┬─────────────┬────────┬─────────────┬────────┐
  │ Data   │    HOLE     │ Data   │    HOLE     │ Data   │
  │ 0-4095 │ 4096-65535  │ 65536- │ 69632-      │ 131072-│
  │        │ (no blocks) │ 69631  │ 131071      │ 135167 │
  └────────┴─────────────┴────────┴─────────────┴────────┘

  lseek(fd, 0, SEEK_HOLE)  → 4096    (first hole)
  lseek(fd, 0, SEEK_DATA)  → 0       (first data)
  lseek(fd, 4096, SEEK_DATA) → 65536 (next data after hole)

  # Find holes with filefrag:
  $ filefrag -v sparse_file
  ext:  logical_offset: physical_offset: length:   flags:
    0:        0..       0:   12345..  12345:      1:
    1:       16..      16:   23456..  23457:      2:  last
  # Logical blocks 1-15 are holes (not mapped)
```

---

## 7.8 Direct I/O vs Buffered I/O

```
Buffered I/O (default):
  Application ←→ Page Cache ←→ Disk
  - Data copied through page cache
  - Readahead for sequential access
  - Writeback for async writes
  - Best for most workloads

Direct I/O (O_DIRECT):
  Application ←→ Disk (bypasses page cache)
  - No page cache involvement
  - User buffer aligned to block size
  - Synchronous by default
  - Used by databases (they have their own cache)

  open("/data/db.file", O_RDWR | O_DIRECT)
    │
    ▼
  read(fd, aligned_buf, size)
    │
    ▼
  ext4_file_read_iter()
    │
    └── ext4_dio_read_iter()              [fs/ext4/file.c]
          │
          └── iomap_dio_rw()              [fs/iomap/direct-io.c]
                │
                ├── For each block:
                │     ├── ext4_iomap_begin()  ← Map logical → physical
                │     ├── bio_alloc()         ← Allocate bio
                │     ├── bio_add_page()      ← Add USER pages to bio
                │     └── submit_bio()        ← Send directly to disk
                │
                └── Wait for all I/O completion

Requirements for O_DIRECT:
  - Buffer must be aligned to logical block size (typically 512B)
  - Offset must be aligned
  - Size must be multiply of block size
  - Otherwise: -EINVAL
```

### Comparison

```
Feature                │ Buffered I/O     │ Direct I/O
───────────────────────┼──────────────────┼──────────────────
Page cache             │ Yes              │ No (bypassed)
Readahead              │ Yes              │ No
Alignment requirement  │ None             │ Block-aligned
Copy overhead          │ Extra copy       │ Zero-copy (DMA to user)
Cache benefit          │ Repeat reads fast│ No caching benefit
Typical user           │ General apps     │ Databases
Write persistence      │ Delayed          │ Immediate (with IO)
Memory usage           │ Higher (cache)   │ Lower
Small random I/O       │ Fast (if cached) │ Slow (every I/O = disk)
```

---

## 7.9 io_uring (Modern Async I/O)

```
io_uring provides true asynchronous I/O:

  ┌──────────────────────────────────────────────┐
  │ User space                                   │
  │                                              │
  │  Submission Queue (SQ)    Completion Queue (CQ) │
  │  ┌──────┐                ┌──────┐            │
  │  │ SQE  │ ──────────►    │ CQE  │            │
  │  │ SQE  │   shared       │ CQE  │            │
  │  │ SQE  │   memory       │ CQE  │            │
  │  └──────┘                └──────┘            │
  │      │                       ▲               │
  └──────┼───────────────────────┼───────────────┘
         │ io_uring_enter()       │ (or poll)
  ┌──────▼───────────────────────┼───────────────┐
  │ Kernel                       │               │
  │                                              │
  │  Process SQEs → submit I/O                   │
  │  I/O completion → fill CQEs                  │
  └──────────────────────────────────────────────┘

Advantages over AIO:
  - No copying between user/kernel (shared ring buffer)
  - Works with buffered AND direct I/O
  - Supports linked requests (chains)
  - Supports fixed buffers and fixed files
  - Can do non-I/O operations (connect, accept, etc.)
```

---

## Kernel Source References

```
System calls:
  fs/open.c              ← open(), close(), access()
  fs/read_write.c        ← read(), write(), lseek(), pread(), pwrite()
  fs/sync.c              ← sync(), fsync(), fdatasync()

VFS dispatch:
  fs/read_write.c: vfs_read(), vfs_write()
  fs/open.c: do_filp_open(), do_sys_openat2()

Page cache (read/write):
  mm/filemap.c           ← filemap_read(), generic_perform_write()
  mm/readahead.c         ← page_cache_sync_readahead()
  mm/page-writeback.c    ← writeback mechanics

ext4 specifics:
  fs/ext4/file.c         ← ext4_file_read_iter(), ext4_file_write_iter()
  fs/ext4/inode.c        ← ext4_write_begin(), ext4_writepages()
  fs/ext4/namei.c        ← ext4_create(), ext4_lookup()
  fs/ext4/fsync.c        ← ext4_sync_file()

io_uring:
  io_uring/io_uring.c    ← Core io_uring implementation
```

---

## Interview Questions

1. **Trace the complete path of open("/home/user/file.txt", O_RDWR) from syscall to ext4.**
2. **What happens during a buffered read when the page is NOT in the page cache?**
3. **Explain delayed allocation in ext4. Why is it beneficial? What's the risk?**
4. **What is the difference between fsync() and fdatasync()?**
5. **Does close() write dirty data to disk? Explain.**
6. **What is readahead? How does the kernel detect sequential access?**
7. **Compare direct I/O and buffered I/O. When would you use each?**
8. **What are SEEK_DATA and SEEK_HOLE used for?**
9. **What is write amplification in the context of FS write paths?**
10. **How does io_uring improve over traditional read()/write() and AIO?**

---

## Summary

- **open()**: Path resolution → permission check → alloc file → install in fd table
- **read()**: fd → struct file → f_op->read_iter() → page cache → (maybe disk)
- **write()**: Data goes to page cache first (dirty pages), NOT immediately to disk
- **close()**: Release file object, dentry ref, mount ref; does NOT flush data
- **fsync()**: Force dirty pages + journal to stable storage + disk cache flush
- Readahead: kernel prefetches sequential pages (grows window adaptively)
- Direct I/O: bypasses page cache for databases; requires aligned buffers
- Delayed allocation: reserves space at write(), allocates blocks at writeback

---

*Next: [Chapter 8 — Directory Management and Lookup](Chapter_08_Directory_Management.md)*
