# Chapter 27: Interview Preparation

## Learning Goals
- Master the top 50 filesystem interview questions with detailed answers
- Know how to whiteboard VFS architecture and I/O paths
- Practice scenario-based troubleshooting questions
- Prepare system design answers for filesystem-related problems

---

## 27.1 Fundamental Questions

### Q1: What is the VFS? Why does Linux need it?

```
Answer:
The Virtual File System (VFS) is an abstraction layer that provides a uniform
interface for all filesystem types. Without VFS, every application would need
to know how to talk to ext4, XFS, NFS, etc.

VFS provides:
  1. Common system call interface: open(), read(), write(), close()
  2. Polymorphism via operation tables (file_operations, inode_operations, etc.)
  3. Shared caching infrastructure (dcache, icache, page cache)
  4. Uniform path resolution (namei.c)

VFS objects: superblock, inode, dentry, file
Each filesystem fills in operation function pointers.

Key source: fs/namei.c, fs/open.c, fs/read_write.c, include/linux/fs.h
```

### Q2: Explain the difference between an inode and a dentry.

```
Answer:
  inode (struct inode):
    - Represents a FILE on disk (metadata: size, permissions, block map)
    - One inode per file, identified by inode number
    - Does NOT contain the filename
    - Lives in icache (slab cache)

  dentry (struct dentry):
    - Represents a NAME in a directory (directory entry)
    - Maps a filename string → inode
    - Lives in dcache (dentry cache)
    - Can be: positive (has inode), negative (caches "doesn't exist"), unused

  Relationship:
    Multiple dentries can point to same inode (hard links)
    dentry→d_inode = pointer to associated inode
    inode→i_dentry = list of dentries referring to this inode

  Why separate?
    - One file can have multiple names (hard links)
    - dcache enables fast path resolution without disk I/O
    - Negative dentries prevent repeated failed lookups
```

### Q3: Trace the complete path of open("/etc/passwd", O_RDONLY).

```
Answer:
1. sys_openat() → do_sys_openat2()
2. get_unused_fd_flags() — allocate fd number
3. do_filp_open() → path_openat()
4. Path resolution (namei.c):
   - Start from root dentry or current directory
   - For each component ("etc", "passwd"):
     a. lookup_fast() → check dcache (RCU-walk first)
     b. If miss: lookup_slow() → inode->i_op->lookup()
        → ext4_lookup() → read directory from disk
        → Create and populate dentry
5. do_last():
   - may_open() → inode_permission()
     → security_inode_permission() (SELinux)
     → generic_permission() (DAC check rwx bits)
   - vfs_open() → do_dentry_open()
     → Sets file->f_op = inode->i_fop (ext4_file_operations)
     → Calls file->f_op->open() → ext4_file_open()
6. fd_install(fd, file) — install in process fd table
7. Return fd to userspace
```

### Q4: What happens when you call read(fd, buf, 4096)?

```
Answer:
1. ksys_read() → vfs_read() → file->f_op->read_iter()
2. ext4_file_read_iter() → generic_file_read_iter()
3. filemap_read() (mm/filemap.c):
   a. Look up page in page cache (xarray by index)
   b. CACHE HIT: copy data directly to user buffer → done!
   c. CACHE MISS: trigger readahead
      → page_cache_sync_readahead()
      → ext4_readahead() → ext4_map_blocks()
      → submit_bio(READ) → block layer → device
      → Interrupt on completion → mark page uptodate
   d. copy_folio_to_iter() → copy to user space
4. Return bytes read

Key optimization: page cache eliminates disk I/O for hot files.
Sequential reads benefit enormously from readahead prefetching.
```

### Q5: Explain write() and why it returns before data is on disk.

```
Answer:
1. vfs_write() → file->f_op->write_iter()
2. ext4_file_write_iter() → generic_file_write_iter()
3. generic_perform_write():
   a. a_ops->write_begin() → ext4_write_begin()
      - Start journal transaction
      - Find/create page in page cache
   b. copy_page_from_iter() — copy user data into cache page
   c. a_ops->write_end() → ext4_write_end()
      - Mark page DIRTY
      - Update inode size if file grew
      - Stop journal transaction
4. Return bytes written to user — page is DIRTY in cache, NOT on disk

WHY: Performance. Writing to RAM is ~1000x faster than disk.
The kernel's writeback threads flush dirty pages later:
  - When dirty_background_ratio is exceeded
  - When dirty_expire_centisecs elapses
  - When sync()/fsync() is called
  - When memory pressure requires page reclaim

To ensure durability: call fsync(fd) after critical writes.
```

---

## 27.2 Intermediate Questions

### Q6: What is the page cache? Why is it unified?

```
Answer:
The page cache is an in-memory cache of file data organized by
(inode, page_offset) pairs in struct address_space using an xarray.

Unified means: ONE cache for ALL local filesystems (ext4, XFS, Btrfs, etc.)
and also for block device access and memory-mapped files.

Benefits of unification:
  - One pool of memory for all file data → efficient RAM usage
  - mmap() and read() see the same cached pages → coherent
  - Writeback subsystem handles all dirty pages uniformly
  - LRU reclaim works across all file data equally

Key structures:
  struct address_space → one per inode, contains xarray of folios
  struct folio → represents one or more cached pages
  Page states: PG_uptodate, PG_dirty, PG_writeback, PG_locked
```

### Q7: How does ext4 journaling work?

```
Answer:
ext4 uses jbd2 (Journaling Block Device 2) for crash consistency.

Write-Ahead Logging (WAL):
  1. Metadata changes logged to journal FIRST
  2. Then written to their final locations

Transaction lifecycle:
  1. RUNNING: handles accumulate changes
  2. LOCKED: no new handles, prepare to commit
  3. FLUSH: write descriptor + metadata blocks to journal
  4. COMMIT: write commit block (with checksum)
  5. FINISHED: original locations updated, journal space reclaimed

Journal modes:
  data=ordered (default): data written before metadata commit
  data=journal:           both data and metadata journaled (safest)
  data=writeback:         only metadata journaled (fastest, risky)

Recovery: on mount, scan journal, replay committed transactions,
discard incomplete transactions → filesystem consistent.
```

### Q8: Compare ext4, XFS, and Btrfs.

```
Answer:
┌──────────────┬────────────────┬────────────────┬────────────────┐
│              │ ext4           │ XFS            │ Btrfs          │
├──────────────┼────────────────┼────────────────┼────────────────┤
│ Design       │ Traditional    │ B+tree based   │ CoW B-tree     │
│ Journal      │ jbd2 (WAL)     │ WAL            │ CoW (no WAL)   │
│ Max FS size  │ 1 EB           │ 8 EB           │ 16 EB          │
│ Max file size│ 16 TB          │ 8 EB           │ 16 EB          │
│ Block alloc  │ mballoc        │ AG B+trees     │ extent tree    │
│ Snapshots    │ No (use LVM)   │ No (use LVM)   │ Native (cheap) │
│ Checksums    │ Metadata only  │ Metadata only  │ Data+Metadata  │
│ Compression  │ No             │ No             │ Yes (zstd/lzo) │
│ Reflink      │ No             │ Yes            │ Yes            │
│ RAID         │ No (use mdraid)│ No (use mdraid)│ Built-in       │
│ Maturity     │ Very stable    │ Very stable    │ Improving      │
│ Best for     │ General, boot  │ Large files    │ Snapshots, NAS │
│ Default on   │ Debian, Ubuntu │ RHEL, SUSE     │ Fedora (soon?) │
└──────────────┴────────────────┴────────────────┴────────────────┘
```

### Q9: What is the dcache? How does RCU-walk path resolution work?

```
Answer:
dcache is an in-memory tree of dentry structures caching directory entries.
Enables path resolution without hitting disk for cached paths.

RCU-walk (fast path):
  - Uses RCU (Read-Copy-Update) to traverse dcache WITHOUT taking locks
  - No atomic increment of dentry reference counts
  - Validates each step with sequence counters
  - If anything changes during walk → falls back to ref-walk

ref-walk (slow path):
  - Takes reference counts (d_lockref) on each dentry
  - Guaranteed correctness
  - Needed when: creating files, crossing mount points, following symlinks

Performance impact:
  RCU-walk: ~100ns for cached path resolution
  ref-walk: ~500ns (still fast, but 5x slower due to atomics)
  Disk lookup: ~5ms (50,000x slower than RCU-walk)
```

### Q10: How does mmap work? What happens on page fault?

```
Answer:
mmap() call:
  1. Creates a VMA (vm_area_struct) in process address space
  2. Sets vma->vm_ops = filesystem's vm_ops
  3. NO pages allocated yet → page table entries are empty
  4. Returns virtual address

First access (e.g., reading the mapped region):
  1. CPU finds empty PTE → PAGE FAULT exception
  2. handle_mm_fault() → do_fault()
  3. vma->vm_ops->fault() → filemap_fault()
  4. filemap_fault() looks up page cache:
     - Found? Use existing page
     - Not found? Read from disk, add to page cache
  5. Install PTE: virtual address → physical page frame
  6. Return to application → access succeeds without intervention

For write to MAP_SHARED:
  - vm_ops->page_mkwrite() called before first write
  - ext4_page_mkwrite() allocates blocks, starts journal
  - Page marked dirty → flushed during writeback
```

---

## 27.3 Advanced Questions

### Q11: How does delayed allocation work in ext4?

```
Answer:
Without delayed allocation:
  write() → immediately allocate physical blocks → fragmented

With delayed allocation (delalloc, default in ext4):
  1. write() → data written to page cache → pages marked dirty
  2. Physical blocks are NOT allocated yet (only in-memory reservation)
  3. Reservation: filesystem reserves block count (ensures ENOSPC accurate)
  4. At writeback time (when flusher runs or fsync()):
     - mballoc sees ALL dirty pages for this file
     - Allocates contiguous blocks for entire dirty range
     - Result: much better locality, less fragmentation

Benefit: By deferring allocation until writeback, the allocator
has more information about the total write size and can make
better block placement decisions.

Risk: If system crashes before writeback, data is lost (but
this is true for any write-behind caching).
```

### Q12: Design a filesystem for an automotive head unit.

```
Answer:
Requirements:
  - Fast boot (display within 2 seconds)
  - Power-loss resilient
  - Verified/secure boot
  - OTA updateable
  - Flash wear management

Partition layout:
  /boot        Raw partition (signed bootloader)
  /kernel_a    Raw partition (signed kernel image)
  /kernel_b    Raw partition (A/B slot for OTA)
  /system_a    SquashFS + dm-verity (compressed, read-only, verified)
  /system_b    SquashFS + dm-verity (A/B slot)
  /vendor      EROFS + dm-verity (vendor HALs, read-only)
  /data        f2fs + fscrypt (writable, encrypted, flash-friendly)
  /persist     ext4 with data=journal (small, critical data, survives reset)
  /misc        Raw (bootloader control block)

Key decisions:
  1. SquashFS with LZ4 for /system: fast decompression (~3 GB/s),
     smaller footprint, no corruption risk
  2. dm-verity: Merkle tree verification ensures integrity
  3. f2fs for /data: optimized for flash write patterns (eMMC)
  4. fscrypt: per-user encryption (File-Based Encryption)
  5. A/B partitions: seamless OTA, rollback on failure
  6. noatime on all mounts: reduce unnecessary writes
  7. Monitor eMMC life_time: alert at 80% consumed
```

### Q13: Explain the I/O stack from application to disk.

```
Answer:
  Application: write(fd, data, len)
      │
  1. SYSCALL → vfs_write() → file->f_op->write_iter()
      │
  2. FILESYSTEM → ext4_file_write_iter()
      │         → generic_perform_write()
      │         → Copy data to page cache, mark dirty
      │
  3. PAGE CACHE → Page is dirty in address_space
      │         → Writeback triggered by dirty thresholds or fsync
      │
  4. WRITEBACK  → ext4_writepages()
      │         → ext4_map_blocks() (allocate physical blocks)
      │         → Build bio structures
      │         → submit_bio(WRITE)
      │
  5. BLOCK LAYER → blk_mq_submit_bio()
      │          → I/O scheduler (merge, reorder)
      │          → blk_mq_dispatch_rq_list()
      │
  6. DEVICE DRIVER → nvme_queue_rq() or scsi_queue_rq()
      │            → DMA setup
      │            → Send command to hardware
      │
  7. HARDWARE → Flash/disk write
      │
  8. COMPLETION → Interrupt → bio->bi_end_io()
               → Clear writeback flag → Page is clean
```

### Q14: What causes "No space left on device" when df shows free space?

```
Answer:
Three possible causes:

1. Inode exhaustion (most common):
   $ df -i /mount
   Filesystem  Inodes  IUsed  IFree IUse%
   /dev/sda1   100000  100000     0  100%   ← All inodes used!
   Fix: remove files, or reformat with mkfs -i (smaller inode ratio)

2. Reserved blocks (ext4 reserves 5% for root by default):
   Non-root user sees "no space" while root still has reserved blocks.
   Fix: tune2fs -m 1 /dev/sda1 (reduce to 1%)

3. Deleted files still open:
   File unlinked but process still has fd open → blocks not freed.
   $ lsof | grep deleted
   Fix: restart the process holding the deleted file open.
```

### Q15: How would you debug a filesystem performance problem?

```
Answer:
Systematic approach:

Step 1: Identify the bottleneck layer
  $ iostat -xz 1           ← Check %util, await, throughput
  High %util + high await = disk bottleneck
  Low %util + slow app = application or FS layer issue

Step 2: Check page cache effectiveness
  $ cat /proc/meminfo | grep Cached
  $ sar -B 1               ← Page fault rate
  Low cache hit = too little RAM or random access pattern

Step 3: Trace filesystem operations
  $ strace -T -e trace=file myapp  ← Find slow syscalls
  $ cat /proc/PID/stack            ← Where is process stuck?

Step 4: Kernel-level analysis
  $ perf record -g -a -- sleep 10  ← CPU profiling
  $ echo 1 > /sys/kernel/debug/tracing/events/ext4/enable
  $ cat trace_pipe                 ← ext4 tracepoints

Step 5: Block-level analysis
  $ blktrace -d /dev/sda           ← Detailed I/O tracing
  $ btt -i trace                   ← Analyze latency breakdown

Common fixes:
  - Add noatime mount option
  - Increase readahead for sequential workloads
  - Use appropriate I/O scheduler (none for NVMe, bfq for HDD)
  - Tune dirty_ratio for workload pattern
  - Use O_DIRECT for database workloads with own cache
```

---

## 27.4 Scenario Questions

### S1: A process is in D state (uninterruptible sleep). Diagnose.

```
Answer:
1. Find the process:
   $ ps aux | grep ' D'

2. Check what it's waiting for:
   $ cat /proc/PID/stack
   Typical output:
     ext4_write_begin → waiting for journal
     nfs_file_read → waiting for NFS server
     blk_mq_get_tag → waiting for block device queue

3. Common causes:
   a. NFS server unreachable → network issue
   b. Disk I/O timeout → check dmesg for ata/scsi errors
   c. Too many dirty pages → writeback congestion
   d. dm-crypt CPU exhaustion → encryption overhead

4. Check dmesg:
   $ dmesg | grep -i 'error\|timeout\|reset'

5. Resolution:
   - NFS: check network, mount with soft,timeo=5
   - Disk: replace failing disk, check SMART data
   - Congestion: tune dirty_ratio or add more RAM
```

### S2: Filesystem went read-only unexpectedly.

```
Answer:
Cause: Kernel detected a filesystem error and remounted read-only
to prevent further corruption.

Diagnosis:
  $ dmesg | grep -i 'ext4\|error\|remount'
  EXT4-fs error (device sda1): ... Remounting filesystem read-only

Recovery:
  1. Unmount: $ umount /dev/sda1 (may need to stop services first)
  2. Check: $ e2fsck -n /dev/sda1 (dry-run first)
  3. Fix: $ e2fsck -y /dev/sda1
  4. Remount: $ mount /dev/sda1 /mnt

Prevention:
  - Check disk health: smartctl -a /dev/sda
  - Set error behavior: tune2fs -e panic /dev/sda1 (for automotive)
  - Use UPS or battery backup to prevent power-loss corruption
```

### S3: Application is slow writing small files.

```
Answer:
Analysis: Small file writes are metadata-heavy. Each file creation:
  - Allocate inode (bitmap + inode table)
  - Add directory entry (directory block)
  - Allocate data blocks
  - Journal all metadata changes
  → Multiple disk I/Os per file!

Optimizations:
  1. Batch writes: write many files, then one fsync()
     (Not fsync after every file)

  2. Consider database: SQLite or embedding data in fewer larger files
     (One large file >> many tiny files)

  3. Mount options: noatime, data=writeback (if can tolerate risk)

  4. Use tmpfs + periodic sync for truly temporary files

  5. SSD helps: random write IOPS much higher than HDD

  6. Pre-create files with fallocate() to avoid fragmented allocation

  7. For ext4: increase journal commit interval (-o commit=60)
     Trade-off: more data at risk during crash
```

---

## 27.5 Quick Answer Reference (Top 30)

```
┌───┬────────────────────────────────────────┬──────────────────────────────┐
│ # │ Question                                │ Key Answer Points            │
├───┼────────────────────────────────────────┼──────────────────────────────┤
│ 1 │ What is VFS?                           │ Abstraction layer, op tables │
│ 2 │ inode vs dentry?                       │ inode=file, dentry=name      │
│ 3 │ Page cache purpose?                    │ Cache file data in RAM       │
│ 4 │ What is a folio?                       │ Multi-page cache entry (5.16)│
│ 5 │ ext4 journal modes?                    │ ordered/journal/writeback    │
│ 6 │ What is delayed allocation?            │ Defer block alloc to writeback│
│ 7 │ What is readahead?                     │ Prefetch sequential pages    │
│ 8 │ Open() path resolution?               │ dcache → inode lookup → perms│
│ 9 │ Hard link vs symlink?                  │ Same inode vs path pointer   │
│10 │ What is the dcache?                    │ Dentry tree cache in memory  │
│11 │ RCU-walk vs ref-walk?                  │ Lockless vs locked path walk │
│12 │ When is fsync needed?                  │ When durability required     │
│13 │ What is noatime?                       │ Skip access time updates     │
│14 │ Container_of macro?                    │ Get outer struct from member │
│15 │ How do writes reach disk?              │ Page cache → writeback → bio │
│16 │ What is a bio?                         │ Block I/O descriptor         │
│17 │ ext4 vs XFS vs Btrfs?                  │ Traditional vs B+tree vs CoW│
│18 │ What is dm-verity?                     │ Merkle hash tree for RO parts│
│19 │ What is fscrypt?                       │ Per-file encryption in FS    │
│20 │ UBIFS vs JFFS2?                        │ Modern (UBI) vs legacy (MTD) │
│21 │ SquashFS purpose?                      │ Compressed read-only FS      │
│22 │ What is OverlayFS?                     │ Union mount (upper + lower)  │
│23 │ D state process debug?                 │ /proc/PID/stack + dmesg     │
│24 │ No space but df shows free?            │ Inodes exhausted (df -i)     │
│25 │ I/O scheduler for NVMe?               │ none (no scheduling needed)  │
│26 │ What is write amplification?           │ Flash: erase > write size    │
│27 │ Automotive FS layout?                  │ SquashFS/system, f2fs/data   │
│28 │ SELinux vs AppArmor?                   │ Labels vs paths              │
│29 │ What is IMA?                           │ File integrity measurement   │
│30 │ How to tune writeback?                 │ dirty_ratio, dirty_background│
└───┴────────────────────────────────────────┴──────────────────────────────┘
```

---

## 27.6 Whiteboard Practice Topics

```
Be prepared to draw on a whiteboard:

1. VFS object relationships:
   task_struct → files_struct → file → dentry → inode → super_block

2. Complete read() path:
   vfs_read → filemap_read → page cache → readahead → bio → device

3. ext4 on-disk layout:
   Block groups, superblock, GDT, bitmaps, inode table, data blocks

4. Extent tree:
   Header + entries, logical → physical mapping

5. Writeback flow:
   Dirty page → flusher thread → writepages → submit_bio

6. Journal lifecycle:
   Running → committing (descriptor + metadata + commit) → checkpoint

7. Mount tree:
   / → /proc, /sys, /home, /tmp with struct mount relationships

8. Page cache states:
   Empty → Reading → Uptodate → Dirty → Writeback → Clean → Reclaim

9. Path resolution:
   Component-by-component: dcache lookup → inode lookup → permission check

10. I/O stack:
    Application → VFS → Filesystem → Page Cache → Block → Driver → Device
```

---

## 27.7 Behavioral / Design Questions

### "Tell me about a difficult filesystem bug you debugged."

```
Framework:
  1. Context: What system? What symptoms?
  2. Investigation: What tools did you use? (strace, dmesg, blktrace)
  3. Root cause: What was wrong? (corruption, race, performance)
  4. Fix: How did you resolve it?
  5. Prevention: What did you change to prevent recurrence?

Example answer:
  "In our automotive IVI system, we had intermittent boot failures where
  the system partition wouldn't mount. Using dmesg, I found ext4 errors
  indicating superblock corruption. Investigation with debugfs showed
  the primary superblock was damaged but backup superblocks were fine.
  Root cause: power loss during OTA update writing to the partition.
  Fix: Switch to A/B partition scheme with SquashFS + dm-verity.
  Prevention: Never write to the active system partition."
```

### "Design a file storage system for a distributed application."

```
Key considerations:
  1. Consistency model: strong consistency (fsync path) or eventual?
  2. Replication: single writer or multi-writer?
  3. Caching: local page cache + distributed invalidation?
  4. Metadata: centralized (NFS server) or distributed (Ceph)?
  5. Failure handling: what if node fails mid-write?

Mention:
  - VFS as the abstraction layer
  - Page cache for local caching with coherency protocol
  - NFS delegations or distributed locking for consistency
  - Journal for local crash recovery
  - Replication for durability beyond single node
```

---

## 27.8 Final Study Checklist

```
Before the interview, make sure you can:

□ Draw the VFS object relationship diagram from memory
□ Trace open() and read() through the kernel layers
□ Explain page cache hit/miss behavior
□ Describe ext4 journaling (jbd2) transaction lifecycle
□ Compare ext4, XFS, Btrfs strengths and weaknesses
□ Explain delayed allocation and why it reduces fragmentation
□ Describe RCU-walk path resolution
□ Know the key kernel source files (namei.c, filemap.c, etc.)
□ Explain writeback: dirty_ratio, dirty_background_ratio
□ Debug a hung process (D state) using /proc/PID/stack
□ Design an automotive partition layout
□ Explain fscrypt vs dm-crypt
□ Know SquashFS, UBIFS, f2fs use cases
□ Use strace, debugfs, fsck, iostat, blktrace
□ Explain the I/O stack: VFS → FS → page cache → block → driver
```

---

## Summary

This chapter covers the essential interview preparation:
- 15 detailed Q&A covering VFS fundamentals through advanced topics
- 3 scenario-based troubleshooting questions with systematic approaches
- Quick reference table of top 30 questions with key answer points
- 10 whiteboard diagram topics you should be able to draw from memory
- Design question frameworks for system-level FS architecture
- Final study checklist to verify readiness

Key interview strategy:
1. Start with the high-level architecture (VFS abstraction)
2. Drill down to specific layers when asked (page cache, journal, block)
3. Always mention kernel source locations (shows real understanding)
4. Use diagrams — draw the I/O stack, VFS objects, or ext4 layout
5. Connect theory to practice — mention strace, debugfs, blktrace

---

*This concludes the Linux File Systems & VFS Book.*
*Return to: [Master Index](00_Master_Index.md)*
