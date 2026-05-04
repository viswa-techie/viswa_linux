# Chapter 21: Complete Flow Diagrams

## Learning Goals
- Visualize the complete paths for open, read, write, close, fsync, mmap
- Understand cross-layer interactions: application → VFS → FS → block → device
- Trace data flow through page cache, journal, and block layer
- Master the "big picture" of how all FS subsystems connect

---

## 21.1 Complete open() Flow

```
Application: fd = open("/home/user/data.txt", O_RDWR)
│
├── SYSCALL ENTRY ──────────────────────────────────────────
│   sys_openat(AT_FDCWD, "/home/user/data.txt", O_RDWR, 0)
│   └─→ do_sys_openat2()
│       ├─→ get_unused_fd_flags()         ← Allocate fd number
│       └─→ do_filp_open()               ← Main work
│
├── PATH RESOLUTION (fs/namei.c) ──────────────────────────
│   path_openat()
│   │
│   │  Component walk: "/" → "home" → "user" → "data.txt"
│   │
│   │  For each component:
│   │  ┌──────────────────────────────────────────────────┐
│   │  │ walk_component()                                  │
│   │  │  ├─→ lookup_fast()              ← dcache lookup   │
│   │  │  │   └─→ __d_lookup_rcu()       ← RCU-walk (fast) │
│   │  │  │       Hit? → advance to next component         │
│   │  │  │                                                │
│   │  │  └─→ lookup_slow()              ← dcache miss     │
│   │  │      └─→ __lookup_slow()                          │
│   │  │          └─→ inode->i_op->lookup()                │
│   │  │              └─→ ext4_lookup()  ← Read dir on disk│
│   │  │                  └─→ ext4_find_entry()            │
│   │  │                      └─→ HTree search             │
│   │  └──────────────────────────────────────────────────┘
│   │
│   │  Last component: "data.txt"
│   │  └─→ do_last()
│   │      ├─→ may_open()              ← Permission check
│   │      │   ├─→ inode_permission()
│   │      │   │   ├─→ security_inode_permission() ← SELinux
│   │      │   │   └─→ generic_permission()        ← DAC check
│   │      │   └─→ may_o_create if O_CREAT
│   │      │
│   │      └─→ vfs_open()
│   │          └─→ do_dentry_open()
│   │              ├─→ file->f_op = inode->i_fop  ← Set operations
│   │              ├─→ security_file_open()        ← LSM check
│   │              └─→ file->f_op->open()
│   │                  └─→ ext4_file_open()        ← FS-specific init
│   │
│   └─→ Return struct file *
│
├── FILE DESCRIPTOR SETUP ──────────────────────────────────
│   fd_install(fd, file)
│   │  current->files->fdt->fd[fd] = file
│   │
│   └─→ Return fd to userspace
│
└── RESULT: fd = 3 (or -1 on error with errno set)

Data Structures After open():
  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │ task_struct   │     │ files_struct  │     │ fdtable      │
  │  ├─files──────┼────→│  ├─fdt───────┼────→│  fd[0]=stdin │
  │              │     │              │     │  fd[1]=stdout│
  └──────────────┘     └──────────────┘     │  fd[2]=stderr│
                                             │  fd[3]───────┼──┐
                                             └──────────────┘  │
                        ┌──────────────────────────────────────┘
                        ▼
                  ┌──────────────┐     ┌──────────────┐
                  │ struct file   │     │ struct dentry │
                  │  f_pos = 0   │     │  d_name="data│
                  │  f_flags=RDWR│     │  .txt"       │
                  │  f_op=ext4_  │     │  d_inode─────┼──┐
                  │   file_ops   │     └──────────────┘  │
                  │  f_path.dentry┼───→                   │
                  │  f_mapping───┼──┐  ┌──────────────┐  │
                  └──────────────┘  │  │ struct inode  │←─┘
                                    │  │  i_ino=131073│
                                    │  │  i_mode=0644 │
                                    │  │  i_size=8192 │
                                    │  │  i_mapping───┼──┐
                                    │  └──────────────┘  │
                                    │                     │
                                    │  ┌──────────────┐  │
                                    └─→│address_space  │←─┘
                                       │  page_tree    │
                                       │  a_ops=ext4_  │
                                       │  aops         │
                                       └──────────────┘
```

---

## 21.2 Complete read() Flow

```
Application: n = read(fd, buffer, 4096)
│
├── SYSCALL (fs/read_write.c) ──────────────────────────────
│   ksys_read(fd, buf, 4096)
│   └─→ fdget_pos(fd)              ← Get struct file from fd
│       └─→ vfs_read(file, buf, 4096, &pos)
│           ├─→ rw_verify_area()   ← Check file bounds
│           └─→ new_sync_read()
│               └─→ call_read_iter(file, &kiocb, &iter)
│                   └─→ file->f_op->read_iter()
│
├── FILESYSTEM READ (fs/ext4/file.c) ──────────────────────
│   ext4_file_read_iter()
│   │  ├─→ Is this direct I/O? (O_DIRECT flag)
│   │  │   YES → ext4_dio_read_iter() → skip page cache
│   │  │   NO  → generic_file_read_iter()
│   │  │
│   │  └─→ generic_file_read_iter()
│   │      └─→ filemap_read()
│   │
├── PAGE CACHE (mm/filemap.c) ─────────────────────────────
│   filemap_read()
│   │
│   │  for each page in requested range:
│   │  ┌──────────────────────────────────────────────────┐
│   │  │ (a) filemap_get_folio(mapping, index)            │
│   │  │     └─→ xa_load(&mapping->i_pages, index)        │
│   │  │                                                   │
│   │  │  CACHE HIT (folio found and uptodate):           │
│   │  │  └─→ copy_folio_to_iter() → copy to user buf     │
│   │  │      └─→ Return immediately! (fastest path)       │
│   │  │                                                   │
│   │  │  CACHE MISS (folio not found):                    │
│   │  │  └─→ page_cache_sync_readahead()                  │
│   │  │      └─→ Trigger readahead (see below)            │
│   │  │  └─→ filemap_get_folio() again                    │
│   │  │  └─→ folio_wait_locked() if not uptodate          │
│   │  │  └─→ copy_folio_to_iter()                         │
│   │  └──────────────────────────────────────────────────┘
│   │
├── READAHEAD (mm/readahead.c) ────────────────────────────
│   page_cache_sync_ra()
│   └─→ ondemand_readahead()
│       │  Calculate how many pages to prefetch
│       │  (based on history, sequential detection)
│       └─→ ra_submit()
│           └─→ read_pages()
│               └─→ mapping->a_ops->readahead()
│                   └─→ ext4_readahead()
│
├── FILESYSTEM DISK READ (fs/ext4/readpage.c) ──────────────
│   ext4_readahead()
│   └─→ ext4_mpage_readpages()
│       │  For each page to read:
│       │  ├─→ ext4_map_blocks()      ← Logical→Physical mapping
│       │  │   └─→ Extent tree lookup
│       │  │       └─→ Physical block number
│       │  │
│       │  └─→ submit_bio(READ, bio)  ← Send I/O to block layer
│       │      └─→ bio contains: physical blocks + page pointers
│
├── BLOCK LAYER (block/blk-mq.c) ──────────────────────────
│   submit_bio()
│   └─→ blk_mq_submit_bio()
│       ├─→ I/O scheduler (mq-deadline / bfq / none)
│       │   └─→ May merge with adjacent request
│       └─→ blk_mq_dispatch_rq_list()
│           └─→ q->mq_ops->queue_rq()
│               └─→ nvme_queue_rq() (or scsi, mmc, etc.)
│
├── DEVICE DRIVER ──────────────────────────────────────────
│   Send command to hardware (NVMe submission queue, SCSI CDB, etc.)
│   └─→ DMA transfer: disk → RAM (page frames)
│
├── COMPLETION (interrupt) ─────────────────────────────────
│   Device interrupt → nvme_irq()
│   └─→ nvme_complete_rq()
│       └─→ blk_mq_complete_request()
│           └─→ bio->bi_end_io = mpage_end_io
│               └─→ folio_mark_uptodate(folio)
│               └─→ folio_unlock(folio)  ← Wake waiters
│
├── DATA COPY TO USER ──────────────────────────────────────
│   Back in filemap_read():
│   copy_folio_to_iter(folio, offset, bytes, iter)
│   └─→ copyout() → copy_to_user() → data in user buffer
│
└── RETURN: n = 4096 (bytes read)
```

---

## 21.3 Complete write() Flow

```
Application: n = write(fd, data, 4096)
│
├── SYSCALL (fs/read_write.c) ──────────────────────────────
│   ksys_write(fd, buf, 4096)
│   └─→ vfs_write(file, buf, 4096, &pos)
│       └─→ new_sync_write()
│           └─→ file->f_op->write_iter()
│
├── FILESYSTEM WRITE (fs/ext4/file.c) ─────────────────────
│   ext4_file_write_iter()
│   │  ├─→ O_DIRECT? → ext4_dio_write_iter() → bypass cache
│   │  └─→ generic_file_write_iter()
│   │      └─→ __generic_file_write_iter()
│   │          └─→ generic_perform_write()
│   │
├── WRITE TO PAGE CACHE ───────────────────────────────────
│   generic_perform_write()
│   │
│   │  for each page in write range:
│   │  ┌──────────────────────────────────────────────────┐
│   │  │ (1) a_ops->write_begin()                         │
│   │  │     └─→ ext4_write_begin()                       │
│   │  │         ├─→ Start journal transaction             │
│   │  │         ├─→ grab_cache_page_write_begin()         │
│   │  │         │   └─→ Find or create page in cache      │
│   │  │         └─→ ext4_map_blocks() (if not delayed)    │
│   │  │                                                   │
│   │  │ (2) copy_page_from_iter()                         │
│   │  │     └─→ Copy user data into page cache page       │
│   │  │                                                   │
│   │  │ (3) a_ops->write_end()                            │
│   │  │     └─→ ext4_write_end()                          │
│   │  │         ├─→ Mark page dirty                       │
│   │  │         ├─→ Update inode size if extended          │
│   │  │         └─→ Stop journal transaction               │
│   │  │                                                   │
│   │  │ Page is now DIRTY in page cache                   │
│   │  │ write() RETURNS TO USER immediately!              │
│   │  └──────────────────────────────────────────────────┘
│   │
│   └─→ Return bytes written
│
├── WRITEBACK (later, asynchronous) ────────────────────────
│   Triggered by:
│   │  - dirty_background_ratio exceeded
│   │  - dirty_expire_centisecs elapsed
│   │  - sync() / fsync() from application
│   │  - Memory pressure (reclaim needs pages)
│   │
│   wb_workfn() → wb_do_writeback()
│   └─→ wb_writeback()
│       └─→ writeback_sb_inodes()
│           └─→ __writeback_single_inode()
│               └─→ do_writepages()
│                   └─→ mapping->a_ops->writepages()
│                       └─→ ext4_writepages()
│
├── ext4 WRITEBACK (fs/ext4/inode.c) ──────────────────────
│   ext4_writepages()
│   │  ├─→ Collect dirty pages
│   │  ├─→ ext4_map_blocks()        ← Allocate physical blocks NOW
│   │  │   └─→ Delayed allocation resolved
│   │  ├─→ submit_bio(WRITE, bio)   ← Submit to block layer
│   │  └─→ Wait for completion
│   │
├── JOURNAL (fs/jbd2/) ────────────────────────────────────
│   Metadata changes go through journal:
│   │  jbd2_journal_start()
│   │  │  └─→ Start transaction
│   │  jbd2_journal_get_write_access()
│   │  │  └─→ Log metadata block to journal
│   │  jbd2_journal_stop()
│   │  │  └─→ Mark transaction for commit
│   │  jbd2_journal_commit_transaction()
│   │      └─→ Write journal blocks to disk
│   │      └─→ Write barrier (flush)
│   │      └─→ Commit block written → transaction durable
│   │
├── BLOCK LAYER + DEVICE ──────────────────────────────────
│   Same as read: submit_bio → scheduler → driver → DMA → disk
│
└── COMPLETION
    Data and metadata are now on persistent storage.
```

---

## 21.4 Complete fsync() Flow

```
Application: fsync(fd)
│
├── SYSCALL ────────────────────────────────────────────────
│   sys_fsync(fd)
│   └─→ do_fsync(fd, 0)  /* 0 = flush data + metadata */
│       └─→ vfs_fsync(file, 0)
│           └─→ file->f_op->fsync()
│
├── FILESYSTEM (fs/ext4/fsync.c) ──────────────────────────
│   ext4_sync_file()
│   │
│   │  ┌──────────────────────────────────────────────────┐
│   │  │ Step 1: Flush dirty pages to disk                 │
│   │  │  file_write_and_wait_range()                      │
│   │  │  └─→ filemap_write_and_wait_range()               │
│   │  │      ├─→ __filemap_fdatawrite_range()             │
│   │  │      │   └─→ a_ops->writepages()                  │
│   │  │      │       └─→ ext4_writepages()                │
│   │  │      │           └─→ submit_bio(WRITE)            │
│   │  │      └─→ __filemap_fdatawait_range()              │
│   │  │          └─→ Wait for all page I/Os to complete   │
│   │  │                                                   │
│   │  │ Step 2: Flush journal (commit transaction)        │
│   │  │  jbd2_complete_transaction()                      │
│   │  │  └─→ Force current transaction to commit          │
│   │  │  └─→ Write all journal blocks                     │
│   │  │  └─→ Write commit record                          │
│   │  │  └─→ Issue write barrier (flush)                  │
│   │  │                                                   │
│   │  │ Step 3: Issue device flush (write barrier)        │
│   │  │  blkdev_issue_flush()                             │
│   │  │  └─→ Ensures device write cache flushed           │
│   │  │                                                   │
│   │  │ After fsync() returns:                            │
│   │  │  ✓ All data blocks on persistent storage          │
│   │  │  ✓ All metadata (including journal) on storage    │
│   │  │  ✓ Device write cache flushed                     │
│   │  └──────────────────────────────────────────────────┘
│   │
└── RETURN: 0 on success, -1 on error

    Durability guarantee:
    After fsync() returns 0, the data survives power loss
    (assuming hardware write cache honoring flush commands)
```

---

## 21.5 Complete mmap() + Page Fault Flow

```
Application:
  void *addr = mmap(NULL, 4096, PROT_READ|PROT_WRITE,
                    MAP_SHARED, fd, 0);
│
├── MMAP SETUP (mm/mmap.c) ────────────────────────────────
│   sys_mmap → ksys_mmap_pgoff → do_mmap
│   │
│   │  ┌──────────────────────────────────────────────────┐
│   │  │ 1. Find free virtual address range (VMA)         │
│   │  │ 2. Create struct vm_area_struct                   │
│   │  │    vma->vm_file = file                            │
│   │  │    vma->vm_ops = ext4_file_vm_ops                 │
│   │  │    vma->vm_pgoff = offset / PAGE_SIZE             │
│   │  │ 3. Insert VMA into mm->mm_mt (maple tree)        │
│   │  │ 4. Return virtual address                         │
│   │  │                                                   │
│   │  │ NOTE: No pages are allocated yet!                 │
│   │  │ Page table entries are empty                      │
│   │  └──────────────────────────────────────────────────┘
│   │
│   └─→ Return addr (e.g., 0x7f1234000000)
│
├── FIRST ACCESS (triggers page fault) ────────────────────
│   *addr = 42;  /* Write to mapped address */
│   │
│   │  CPU: page table entry is empty → PAGE FAULT
│   │
│   handle_mm_fault()
│   └─→ __handle_mm_fault()
│       └─→ handle_pte_fault()
│           └─→ do_fault()
│               └─→ do_shared_fault() (MAP_SHARED + write)
│                   │
│                   ├─→ __do_fault()
│                   │   └─→ vma->vm_ops->fault()
│                   │       └─→ filemap_fault() (mm/filemap.c)
│                   │           │
│                   │           ├─→ find_get_page()  ← Page in cache?
│                   │           │   YES → return page
│                   │           │   NO  → page_cache_sync_readahead()
│                   │           │         → read from disk
│                   │           │         → return page
│                   │           │
│                   │           └─→ Return page (struct page *)
│                   │
│                   ├─→ __do_fault() returns page
│                   │
│                   ├─→ vma->vm_ops->page_mkwrite()  ← Mark for write
│                   │   └─→ ext4_page_mkwrite()
│                   │       └─→ Allocate blocks if needed
│                   │       └─→ Start journal transaction
│                   │       └─→ Wait for writeback if needed
│                   │       └─→ Return VM_FAULT_LOCKED
│                   │
│                   └─→ Install PTE: virtual addr → physical page
│                       └─→ set_pte_at()
│                           Page table now maps addr → page frame
│
├── SUBSEQUENT ACCESS ─────────────────────────────────────
│   *(addr + 4) = 99;
│   └─→ PTE valid → direct memory access (no fault)
│       Page is dirty (marked by hardware)
│
├── PAGE WRITEBACK ────────────────────────────────────────
│   Dirty pages written back periodically or on msync/munmap:
│   msync(addr, 4096, MS_SYNC)
│   └─→ Flush dirty pages → ext4_writepages() → disk
│
└── UNMAP
    munmap(addr, 4096)
    └─→ Remove VMA, flush dirty pages, release page references
```

---

## 21.6 Complete File Creation Flow

```
Application: fd = open("/mnt/new_file.txt", O_CREAT|O_WRONLY, 0644)
│
├── PATH RESOLUTION ────────────────────────────────────────
│   Walk: "/" → "mnt" → "new_file.txt"
│   Last component "new_file.txt" does not exist
│   └─→ open_last_lookups() determines O_CREAT
│       └─→ lookup_open()
│           └─→ dir_inode->i_op->create()
│
├── INODE CREATION (fs/ext4/namei.c) ───────────────────────
│   ext4_create()
│   │
│   │  ┌──────────────────────────────────────────────────┐
│   │  │ Step 1: Start journal transaction                 │
│   │  │   handle = ext4_journal_start(dir, ...)           │
│   │  │                                                   │
│   │  │ Step 2: Allocate new inode                        │
│   │  │   inode = ext4_new_inode(handle, dir, mode, ...)  │
│   │  │   ├─→ Find free inode number (bitmap)             │
│   │  │   ├─→ Initialize inode fields                     │
│   │  │   │   (i_ino, i_mode, i_uid, i_gid, timestamps)  │
│   │  │   └─→ Mark inode bitmap, update group descriptor  │
│   │  │                                                   │
│   │  │ Step 3: Add directory entry                       │
│   │  │   ext4_add_entry(handle, dentry, inode)           │
│   │  │   ├─→ Find space in directory blocks              │
│   │  │   │   (or split HTree node if needed)             │
│   │  │   └─→ Write new dir entry: {ino, name, type}     │
│   │  │                                                   │
│   │  │ Step 4: Set file operations                       │
│   │  │   inode->i_fop = &ext4_file_operations            │
│   │  │   inode->i_op = &ext4_file_inode_operations       │
│   │  │                                                   │
│   │  │ Step 5: Instantiate dentry                        │
│   │  │   d_instantiate_new(dentry, inode)                │
│   │  │   └─→ dentry->d_inode = inode (now in dcache)    │
│   │  │                                                   │
│   │  │ Step 6: Stop journal transaction                  │
│   │  │   ext4_journal_stop(handle)                       │
│   │  │                                                   │
│   │  │ On-disk changes (journaled):                      │
│   │  │   - Inode bitmap: bit set for new inode           │
│   │  │   - Inode table: new inode written                │
│   │  │   - Directory blocks: new entry added             │
│   │  │   - Group descriptor: free inode count updated    │
│   │  └──────────────────────────────────────────────────┘
│   │
└── RETURN: fd (file now exists and is open)
```

---

## 21.7 Complete File Deletion Flow

```
Application: unlink("/mnt/old_file.txt")
│
├── SYSCALL ────────────────────────────────────────────────
│   sys_unlinkat(AT_FDCWD, "/mnt/old_file.txt", 0)
│   └─→ do_unlinkat()
│       └─→ Path resolution for parent directory
│       └─→ vfs_unlink(dir_inode, dentry)
│
├── VFS UNLINK (fs/namei.c) ───────────────────────────────
│   vfs_unlink()
│   ├─→ security_inode_unlink()    ← SELinux check
│   └─→ dir_inode->i_op->unlink()
│       └─→ ext4_unlink()
│
├── FILESYSTEM UNLINK (fs/ext4/namei.c) ───────────────────
│   ext4_unlink()
│   │
│   │  ┌──────────────────────────────────────────────────┐
│   │  │ Step 1: Start journal transaction                 │
│   │  │                                                   │
│   │  │ Step 2: Remove directory entry                    │
│   │  │   ext4_delete_entry(handle, dir, de, bh)          │
│   │  │   └─→ Zero out dir entry (or mark deleted)        │
│   │  │                                                   │
│   │  │ Step 3: Decrement inode link count                │
│   │  │   drop_nlink(inode)  → inode->i_nlink--           │
│   │  │                                                   │
│   │  │ Step 4: Update timestamps                         │
│   │  │   dir->i_mtime = dir->i_ctime = current_time      │
│   │  │   inode->i_ctime = dir->i_ctime                   │
│   │  │                                                   │
│   │  │ Step 5: Stop journal transaction                  │
│   │  │                                                   │
│   │  │ If i_nlink reaches 0 AND no open file descriptors:│
│   │  │   → Inode marked for deletion (evict_inode)       │
│   │  │   → Blocks freed (ext4_free_blocks)               │
│   │  │   → Inode freed (ext4_free_inode)                 │
│   │  │                                                   │
│   │  │ If i_nlink reaches 0 BUT still open:              │
│   │  │   → File is "unlinked but open"                   │
│   │  │   → Orphan inode added to orphan list             │
│   │  │   → Blocks freed when last fd is closed           │
│   │  └──────────────────────────────────────────────────┘
│
├── DENTRY CLEANUP ─────────────────────────────────────────
│   d_delete(dentry)
│   └─→ Mark dentry as negative (no inode)
│       └─→ Negative dentry may remain in dcache
│           (caches "this name doesn't exist")
│
└── RETURN: 0 on success
```

---

## 21.8 Complete Mount Flow

```
Application: mount -t ext4 /dev/sda1 /mnt
│
├── SYSCALL ────────────────────────────────────────────────
│   sys_mount("/dev/sda1", "/mnt", "ext4", flags, data)
│   └─→ do_mount()
│       └─→ path_mount()
│           └─→ do_new_mount()
│
├── FIND FILESYSTEM TYPE ──────────────────────────────────
│   get_fs_type("ext4")
│   └─→ Search file_systems list
│       └─→ Found: ext4_fs_type
│
├── MOUNT FILESYSTEM ──────────────────────────────────────
│   vfs_get_tree()
│   └─→ fc->ops->get_tree()
│       └─→ ext4_get_tree()
│           └─→ get_tree_bdev()
│               └─→ ext4_fill_super()
│
├── FILL SUPERBLOCK (fs/ext4/super.c) ─────────────────────
│   ext4_fill_super(sb, fc)
│   │
│   │  ┌──────────────────────────────────────────────────┐
│   │  │ Step 1: Read on-disk superblock                   │
│   │  │   bh = sb_bread(sb, 1)  ← Read block 1 (SB)     │
│   │  │   es = (struct ext4_super_block *)bh->b_data      │
│   │  │                                                   │
│   │  │ Step 2: Validate magic number                     │
│   │  │   if (es->s_magic != EXT4_SUPER_MAGIC) error      │
│   │  │                                                   │
│   │  │ Step 3: Set up VFS superblock                     │
│   │  │   sb->s_blocksize = block_size(es)                │
│   │  │   sb->s_op = &ext4_sops                           │
│   │  │   sb->s_xattr = ext4_xattr_handlers               │
│   │  │                                                   │
│   │  │ Step 4: Initialize journal (jbd2)                 │
│   │  │   ext4_load_journal()                             │
│   │  │   └─→ jbd2_journal_load()                         │
│   │  │       └─→ Replay uncommitted transactions         │
│   │  │                                                   │
│   │  │ Step 5: Read root inode                           │
│   │  │   root = ext4_iget(sb, EXT4_ROOT_INO)            │
│   │  │   sb->s_root = d_make_root(root)                  │
│   │  │                                                   │
│   │  │ Step 6: Check + recover orphan inodes             │
│   │  │   ext4_orphan_cleanup()                           │
│   │  └──────────────────────────────────────────────────┘
│
├── ATTACH MOUNT ──────────────────────────────────────────
│   do_new_mount_fc()
│   └─→ vfs_create_mount()       ← Create struct mount
│       └─→ graft_tree()         ← Attach to mount tree
│           └─→ /mnt now points to ext4 root
│
└── RETURN: 0 on success
    /mnt is now accessible with ext4 filesystem
```

---

## 21.9 I/O Stack Summary Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        USER SPACE                                    │
│  Application → open() → read() → write() → fsync() → close()       │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ system call
═══════════════════════════════ │ ═══════════════════════════════════════
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                        VFS LAYER                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌───────────────┐          │
│  │ namei.c │  │ open.c  │  │ r/w.c   │  │ dcache/icache │          │
│  │ (paths) │  │ (open)  │  │(read/wr)│  │  (caches)     │          │
│  └─────────┘  └─────────┘  └─────────┘  └───────────────┘          │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ file->f_op->read_iter / write_iter
┌──────────────────────────────▼──────────────────────────────────────┐
│                     FILESYSTEM LAYER                                 │
│  ┌───────┐  ┌───────┐  ┌───────┐  ┌───────┐  ┌────────┐           │
│  │ ext4  │  │  XFS  │  │ Btrfs │  │  NFS  │  │ tmpfs  │  ...      │
│  └───┬───┘  └───┬───┘  └───┬───┘  └───┬───┘  └───┬────┘           │
│      │          │          │          │          │ (RAM only)       │
└──────┼──────────┼──────────┼──────────┼──────────┘                  │
       │          │          │          │                              │
┌──────▼──────────▼──────────▼──────────┘                             │
│                    PAGE CACHE                                        │
│  ┌──────────────────────────────────────────────────────┐           │
│  │ mm/filemap.c — Unified page cache for all local FSes │           │
│  │  ├─→ Read: filemap_read()     (cache hit = fast!)    │           │
│  │  ├─→ Write: filemap_write()   (write-back caching)   │           │
│  │  ├─→ Readahead: mm/readahead.c                       │           │
│  │  └─→ Writeback: mm/page-writeback.c                  │           │
│  └──────────────────────────────────────────────────────┘           │
│                         │ submit_bio()                               │
└─────────────────────────┼───────────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────────┐
│                     BLOCK LAYER                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐      │
│  │ bio.c      │  │ blk-mq.c   │  │ scheduler  │  │ dm/md    │      │
│  │ (bio mgmt) │  │ (multiqueue)│  │ (deadline/ │  │(dm-crypt │      │
│  │            │  │            │  │  bfq/none) │  │ dm-verity│      │
│  └────────────┘  └────────────┘  └────────────┘  └──────────┘      │
│                         │ q->mq_ops->queue_rq()                     │
└─────────────────────────┼───────────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────────┐
│                    DEVICE DRIVER                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │  NVMe    │  │  SCSI/   │  │  MMC     │  │  virtio- │            │
│  │          │  │  libata  │  │  (eMMC/  │  │  blk     │            │
│  │          │  │  (SATA)  │  │   UFS)   │  │  (VM)    │            │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘  └─────┬────┘            │
│        │ DMA         │ DMA         │ DMA         │ DMA              │
└────────┼─────────────┼─────────────┼─────────────┼──────────────────┘
         │             │             │             │
    ┌────▼────┐   ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
    │  NVMe   │   │  SATA   │   │  eMMC/  │   │ Virtual │
    │  SSD    │   │  HDD    │   │  UFS    │   │  Disk   │
    └─────────┘   └─────────┘   └─────────┘   └─────────┘
```

---

## Interview Questions

1. **Trace the complete path of read(fd, buf, 4096) from syscall to disk.**
2. **What happens during a page cache miss on read?**
3. **Explain the write-back flow: when do dirty pages actually reach disk?**
4. **What are the three steps of fsync() in ext4?**
5. **Draw the mmap page fault flow for a shared file mapping.**
6. **What journal operations happen during file creation?**
7. **Explain the unlink flow — when are blocks actually freed?**
8. **What happens during mount() of an ext4 filesystem?**
9. **Draw the complete I/O stack from application to hardware.**
10. **What is the role of the page cache vs the block layer?**

---

## Summary

- open(): path resolution (dcache fast path → inode lookup slow path) → permission check → struct file creation
- read(): VFS → page cache lookup → cache hit (fast) or readahead → block I/O → completion → copy to user
- write(): VFS → copy to page cache → mark dirty → return immediately → writeback later
- fsync(): flush dirty pages + commit journal + device flush → durability guaranteed
- mmap(): creates VMA only → page fault on first access → page cache + PTE installation
- create: journal start → inode alloc → dir entry → journal stop
- unlink: journal start → dir entry remove → nlink-- → journal stop → blocks freed when nlink=0 and last fd closed
- mount: read superblock → validate → init journal → read root inode → attach to mount tree

---

*Next: [Chapter 22 — Important Diagrams](Chapter_22_Important_Diagrams.md)*
