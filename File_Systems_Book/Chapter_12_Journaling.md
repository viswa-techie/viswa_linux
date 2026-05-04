# Chapter 12: Journaling and Crash Recovery

## Learning Goals
- Understand the crash consistency problem and why journaling exists
- Master jbd2 (ext3/ext4 journaling) internals
- Know the three journal modes and their trade-offs
- Understand XFS write-ahead logging and Btrfs CoW recovery
- Learn fsck and recovery procedures

---

## 12.1 The Crash Consistency Problem

```
Creating a new file requires multiple disk updates:

  1. Allocate inode → Update inode bitmap
  2. Initialize inode → Write inode data
  3. Allocate data block → Update block bitmap
  4. Write file data → Write data block
  5. Update directory → Add dir entry (name → inode)
  6. Update parent inode → Modify mtime, size

If power fails BETWEEN any two steps:

  After step 1 only:
    Inode bitmap says "allocated" but inode is garbage
    → orphan inode, bitmap leak

  After steps 1-2, before step 5:
    Inode exists but no directory entry points to it
    → orphan inode (unreachable file)

  After steps 1-5, before step 3:
    File exists but data block not marked allocated
    → Another file could get same block → DATA CORRUPTION

  EVERY combination of partial completion is a potential disaster.
```

### Solutions to Crash Consistency

```
Method           │ Approach                    │ Used By
─────────────────┼─────────────────────────────┼─────────────
fsck (no journal)│ Scan entire FS after crash  │ ext2
                 │ Detect and fix inconsistencies│
                 │ VERY SLOW for large FS       │
─────────────────┼─────────────────────────────┼─────────────
Journaling       │ Log changes before applying  │ ext3, ext4
                 │ Replay log after crash       │ XFS, JFS
                 │ Fast recovery (seconds)      │ NTFS, HFS+
─────────────────┼─────────────────────────────┼─────────────
Copy-on-Write    │ Never overwrite in place     │ Btrfs, ZFS
                 │ Atomic pointer update        │ APFS
                 │ Old data always intact        │
─────────────────┼─────────────────────────────┼─────────────
Soft Updates     │ Careful ordering of writes   │ FreeBSD UFS
                 │ + background fsck            │
─────────────────┼─────────────────────────────┼─────────────
Log-structured   │ Always write sequentially    │ F2FS, LFS
                 │ + garbage collection         │
```

---

## 12.2 Journaling Concepts

### Write-Ahead Logging (WAL)

```
Core principle: Write the INTENT before the ACTION.

  Normal operation:
    1. Write changes to journal FIRST (sequential writes, fast)
    2. Journal commit record written (marks transaction complete)
    3. Apply changes to actual disk locations ("checkpoint")
    4. Mark journal space as reusable

              Journal              Disk (actual locations)
  Time ─►
    T1: [TX start]
    T2: [metadata block A']       
    T3: [metadata block B']       
    T4: [TX commit] ─────────── Transaction is durable!
    T5:                          Write A' to location A
    T6:                          Write B' to location B
    T7: [TX freed]               Journal space reclaimed

  CRASH at T3: Transaction incomplete → discard (no harm)
  CRASH at T5: Transaction complete → replay journal (write A', B')
  CRASH at T6: A written, B not → replay writes B'
```

### Journal Layout (ext4/jbd2)

```
Journal is a circular log (special file or reserved area):

  ┌──────────────────────────────────────────────────────────┐
  │                    Journal Area                           │
  │                                                          │
  │  ┌─────────┐ ┌──────────────────────┐ ┌─────────┐      │
  │  │ Journal  │ │ Transaction 1        │ │ Trans 2 │      │
  │  │ Superblk │ │ [Desc][Block][Block] │ │ [D][B]  │ ...  │
  │  │          │ │ [Commit]             │ │ [Commit] │      │
  │  └─────────┘ └──────────────────────┘ └──────────┘      │
  │  ▲                                                ▲      │
  │  │                                                │      │
  │  journal_start                           journal_head    │
  │                                          (wraps around!) │
  └──────────────────────────────────────────────────────────┘

Transaction anatomy:
  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
  │Descriptor│ │ Metadata │ │ Metadata │ │ Metadata │ │ Commit   │
  │ Block    │ │ Block 1  │ │ Block 2  │ │ Block 3  │ │ Block    │
  │(lists    │ │(copy of  │ │(copy of  │ │(copy of  │ │(checksum │
  │ blocks   │ │ modified │ │ modified │ │ modified │ │ of entire│
  │ in tx)   │ │ block)   │ │ block)   │ │ block)   │ │ tx)      │
  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
```

---

## 12.3 jbd2 — ext3/ext4 Journal Implementation

### Transaction Lifecycle

```c
/* Transaction lifecycle in jbd2 */

/* 1. Start a transaction handle */
handle_t *handle = jbd2_journal_start(journal, nblocks);
/* nblocks = max metadata blocks this operation will modify */

/* 2. Associate a buffer with the transaction */
jbd2_journal_get_write_access(handle, bh);
/* bh = buffer_head of metadata block to modify */

/* 3. Modify the buffer (actual changes) */
modify_buffer(bh);

/* 4. Mark buffer as dirty in the transaction */
jbd2_journal_dirty_metadata(handle, bh);

/* 5. Stop the handle (finish this operation) */
jbd2_journal_stop(handle);
/* Transaction stays open for more operations */

/* 6. Background: commit transaction */
/* jbd2_journal_commit_transaction() runs when:
 *   - commit interval expires (default 5 sec)
 *   - fsync() called
 *   - journal space needed
 */
```

### Transaction Commit Process

```
jbd2_journal_commit_transaction():
  │
  ├── Phase 1: Lock transaction
  │     Mark running transaction as "locked"
  │     No new handles can join this transaction
  │     Wait for all active handles to complete
  │
  ├── Phase 2: Flush dirty data (ordered mode)
  │     filemap_fdatawrite() for all inodes in transaction
  │     Wait for data I/O completion
  │     (ensures data on disk before metadata)
  │
  ├── Phase 3: Write descriptor + metadata blocks to journal
  │     For each dirty metadata buffer:
  │       ├── Write descriptor block (list of blocks)
  │       └── Write copy of metadata block to journal
  │     Submit all these writes
  │
  ├── Phase 4: Wait for journal writes to complete
  │     Wait for all I/O in phase 3
  │
  ├── Phase 5: Write commit block (with checksum)
  │     ├── Compute CRC32C of entire transaction
  │     ├── Write commit block with checksum
  │     └── Issue storage FLUSH barrier
  │     Transaction is NOW durable!
  │
  ├── Phase 6: Checkpoint (later, asynchronous)
  │     Write metadata blocks to their actual disk locations
  │     When done: mark journal space as free
  │
  └── Phase 7: Cleanup
        Free transaction buffers
        Update journal superblock
```

### Journal Modes

```
ext4 journal modes (set at mount time):

┌────────────────────────────────────────────────────────────────┐
│ data=journal                                                    │
│                                                                │
│ Journals: Metadata AND file data                               │
│ Safety:   Highest — data + metadata always consistent          │
│ Speed:    Slowest — all data written twice (journal + disk)    │
│ Use:      Very safety-critical applications                    │
│                                                                │
│ Write flow:                                                    │
│   data → journal → commit → data → disk                       │
│          metadata → journal → commit → metadata → disk          │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ data=ordered (DEFAULT)                                          │
│                                                                │
│ Journals: Metadata only                                        │
│ Ordering: Data written to disk BEFORE metadata committed       │
│ Safety:   Good — no stale/garbage data visible after crash     │
│ Speed:    Medium                                               │
│                                                                │
│ Write flow:                                                    │
│   data → disk (MUST complete first)                            │
│   metadata → journal → commit → metadata → disk               │
│                                                                │
│ Guarantees: After crash, file contains either:                 │
│   - Old data (before write) — safe                             │
│   - New data (after write) — safe                              │
│   - Never garbage/stale data from other files                  │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ data=writeback                                                  │
│                                                                │
│ Journals: Metadata only                                        │
│ Ordering: NO ordering between data and metadata writes         │
│ Safety:   Lowest — possible stale/garbage data after crash     │
│ Speed:    Fastest                                              │
│                                                                │
│ Risk scenario:                                                 │
│   1. File is extended (metadata written: new size)             │
│   2. CRASH before data written                                 │
│   3. After recovery: file has new size but OLD/GARBAGE data    │
│      in newly allocated blocks → SECURITY RISK (data leak)     │
└────────────────────────────────────────────────────────────────┘

Performance comparison (approximate):
  data=journal:   100% baseline
  data=ordered:   150-200% throughput
  data=writeback: 200-250% throughput
```

---

## 12.4 Journal Checksum and Fast Commit

### Journal Checksumming (ext4)

```
Without checksums:
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │Descriptor│ │ Block 1  │ │ Commit   │
  │          │ │ (damaged!)│ │          │
  └──────────┘ └──────────┘ └──────────┘
  
  Commit block present → replay corrupted block → FS CORRUPTION!

With checksums (metadata_csum feature):
  ┌──────────┐ ┌──────────┐ ┌──────────────────────┐
  │Descriptor│ │ Block 1  │ │ Commit               │
  │ +checksum│ │ (damaged!)│ │ checksum=CRC32C(all) │
  └──────────┘ └──────────┘ └──────────────────────┘
  
  Recovery: compute CRC32C of descriptor + blocks
  Compare with commit block checksum → MISMATCH → discard transaction!
  
  Also enables async journal commits:
    No need for barriers between data blocks and commit block
    → Significant performance improvement
```

### Fast Commit (ext4, since 5.10)

```
Traditional journal commit: write ALL modified metadata blocks
Fast commit: write only the OPERATION description

  Traditional:
    Transaction: modify inode 100 (256 bytes) + block bitmap (4KB)
    Journal writes: 4KB (descriptor) + 4KB (inode block) + 4KB (bitmap) + 4KB (commit)
    = 16 KB written to journal

  Fast commit:
    Transaction: "append 4KB to file inode 100"
    Journal writes: small operation record (~64 bytes)
    = Much less journal I/O!

  Recovery:
    Fast commit log replayed by re-executing operations
    Falls back to normal journal replay if needed

  Enable: mount -o fastcommit
```

---

## 12.5 XFS Write-Ahead Log

```
XFS uses its own logging system (not jbd2):

  ┌─────────────────────────────────────────┐
  │  XFS Log (circular write-ahead log)      │
  │                                         │
  │  ┌────────┐ ┌─────────────┐ ┌────────┐ │
  │  │ Log    │ │ Log record  │ │ Log    │ │
  │  │ record │ │ (inode mod) │ │ record │ │
  │  │ (mkdir)│ │             │ │ (unlink│ │
  │  └────────┘ └─────────────┘ └────────┘ │
  └─────────────────────────────────────────┘

Key differences from jbd2:
  1. Logical logging (operations, not full block copies)
     - "Set inode 100 field X to Y" instead of copying entire 4KB block
     - Much smaller log records
  2. Delayed logging (batching)
     - Multiple modifications batched into one log write
     - Reduces I/O
  3. Metadata-only journaling
     - XFS always journals metadata only
     - No equivalent of data=journal mode
  4. Log recovery replays operations
     - xfs_log_recover() at mount time
     - Usually < 1 second recovery

XFS intent logging:
  For multi-step operations (e.g., rename):
    1. Log INTENT: "I will rename A to B"
    2. Perform step 1
    3. Perform step 2
    4. Log DONE: "Rename A to B completed"
    
  Crash recovery:
    If INTENT found without DONE → replay/redo the operation
    → Guarantees atomic multi-step operations
```

---

## 12.6 Btrfs Copy-on-Write Recovery

```
Btrfs doesn't need a traditional journal because of CoW:

  Write a modified tree node:
    1. Allocate NEW block
    2. Write modified data to new block
    3. Update parent pointer (also CoW'd up to root)
    4. Write new root pointer to superblock

  Before commit:
    Superblock → old root → old tree (consistent)
    New blocks written but not referenced

  After commit:
    Superblock' → new root → new tree (consistent)
    Old blocks now unreferenced (can be freed)

  CRASH between steps 1-3:
    Superblock still points to OLD root → old consistent tree
    New orphan blocks → freed on next mount

  CRASH during step 4:
    Btrfs writes superblock to 3 locations:
      Primary:   offset 64KB
      Secondary: offset 64MB
      Tertiary:  offset 256GB
    At least one superblock is always valid

  Recovery: read superblocks, pick newest valid one
    → Tree rooted at that superblock is consistent
    → No replay needed!

  Tree log (for fsync optimization):
    Btrfs does maintain a "log tree" for fsync():
    - Records pending changes that haven't been in a full commit
    - Much smaller than full transaction commit
    - Replayed on mount if crash after fsync
```

---

## 12.7 fsck — File System Check and Repair

### ext4 fsck (e2fsck)

```bash
# Check ext4 filesystem
$ sudo e2fsck -f /dev/sda1
# -f = force check even if FS appears clean

# Passes of e2fsck:
# Pass 1: Check inodes, blocks, sizes
#   - Verify inode fields are valid
#   - Check block pointers/extents
#   - Detect duplicate block references
#
# Pass 2: Check directory structure
#   - Verify directory entries point to valid inodes
#   - Check "." and ".." entries
#   - Detect invalid file types
#
# Pass 3: Check directory connectivity
#   - Ensure every directory is reachable from root
#   - Reconnect orphans to lost+found
#
# Pass 4: Check reference counts
#   - Verify i_nlink matches actual directory references
#
# Pass 5: Check group summary information
#   - Verify bitmap consistency
#   - Check free block/inode counts

# Automatic repair
$ sudo e2fsck -p /dev/sda1    # Auto-fix safe problems
$ sudo e2fsck -y /dev/sda1    # Auto-answer "yes" to all questions

# Read-only check (no modifications)
$ sudo e2fsck -n /dev/sda1
```

### XFS Repair

```bash
# XFS check
$ sudo xfs_repair /dev/sda1

# Phases:
# Phase 1: Find and verify superblock
# Phase 2: Using internal log (replay journal)
# Phase 3: Check inodes and blocks
# Phase 4: Check directories
# Phase 5: Check free space
# Phase 6: Check AG metadata
# Phase 7: Rebuild reverse maps

# Dry run (no modifications)
$ sudo xfs_repair -n /dev/sda1

# If log is corrupted:
$ sudo xfs_repair -L /dev/sda1   # Zero log (data loss risk!)
```

### Btrfs Check

```bash
# Btrfs check
$ sudo btrfs check /dev/sda1

# Repair (use with caution!)
$ sudo btrfs check --repair /dev/sda1

# Preferred: restore from snapshot
$ btrfs subvolume snapshot /mnt/@broken /mnt/@broken_backup
$ btrfs subvolume set-default <good_snap_id> /mnt

# Scrub (online integrity check)
$ sudo btrfs scrub start /mnt
$ sudo btrfs scrub status /mnt
```

---

## 12.8 Recovery Scenarios

### Crash During File Write (ext4 ordered mode)

```
Scenario: write 8KB to file, crash during writeback

  Before crash:
    Inode: size=0, blocks=0
    Page cache: 2 dirty pages (8KB)
    Journal: no transaction yet (delayed alloc)

  Case 1: Crash before any I/O
    Recovery: file is empty (size=0), no data lost (wasn't on disk yet)
    
  Case 2: Crash during writeback
    Background writeback started:
      - Data blocks written to disk (maybe partial)
      - Journal transaction started
      - CRASH
    Recovery:
      - Journal transaction incomplete → discarded
      - Inode still shows size=0
      - Data blocks written but not referenced → invisible
      → File appears empty (consistent, no corruption)

  Case 3: Crash after journal commit, before checkpoint
    - Journal has: new inode (size=8KB, extent → blocks)
    - Disk has: old inode (size=0) + data blocks
    Recovery:
      - Replay journal → update inode on disk
      → File has 8KB, data is correct (ordered mode ensured data first)
```

### Orphan Inode Recovery

```
Scenario: File deleted while still open

  Normal operation:
    unlink("file.txt")     → nlink=0, added to orphan list
    process still has fd open → inode not freed
    process exits           → close(fd) → evict_inode() → free blocks

  CRASH while file is in orphan state:
    On next mount:
      ext4_fill_super() → ext4_orphan_cleanup()
        ├── Walk orphan list in journal
        ├── For each orphan inode:
        │     ├── If nlink == 0: truncate + free inode
        │     └── If nlink > 0: truncate to i_size (incomplete write)
        └── Clear orphan list
```

---

## 12.9 Comparison of Recovery Approaches

```
Aspect              │ ext4 (jbd2)      │ XFS (WAL)       │ Btrfs (CoW)
────────────────────┼──────────────────┼─────────────────┼──────────────
Journal type        │ Physical         │ Logical         │ None (CoW)
What's logged       │ Full metadata    │ Operations      │ N/A
                    │ blocks           │                 │
Recovery time       │ Seconds          │ Seconds         │ Instant
Data safety         │ Depends on mode  │ Metadata safe   │ Last commit safe
Data checksums      │ No               │ No              │ Yes (CRC32C)
Space overhead      │ 128 MB default   │ Small (logical) │ None (CoW overhead)
fsync performance   │ Medium           │ Fast            │ Slow (CoW chain)
Corruption detect   │ Journal checksum │ Log checksum    │ Full tree checksum
```

---

## Kernel Source References

```
jbd2 (ext4 journaling):
  fs/jbd2/commit.c        ← Transaction commit logic
  fs/jbd2/recovery.c      ← Journal replay on mount
  fs/jbd2/transaction.c   ← Handle/transaction management
  fs/jbd2/journal.c       ← Journal initialization
  fs/jbd2/checkpoint.c    ← Checkpoint (flush to disk)

ext4 journal interface:
  fs/ext4/super.c          ← Journal initialization (ext4_load_journal)
  fs/ext4/fsync.c          ← fsync → journal commit
  fs/ext4/fast_commit.c    ← Fast commit implementation

XFS log:
  fs/xfs/xfs_log.c        ← Log write
  fs/xfs/xfs_log_recover.c ← Log recovery
  fs/xfs/xfs_trans.c      ← Transaction management

Btrfs:
  fs/btrfs/transaction.c  ← CoW transaction commit
  fs/btrfs/tree-log.c     ← Log tree (for fsync)
  fs/btrfs/disk-io.c      ← Superblock read/validation
```

---

## Interview Questions

1. **What is the crash consistency problem? Give a concrete example.**
2. **How does write-ahead logging (WAL) ensure consistency?**
3. **Explain the three ext4 journal modes. What are the trade-offs?**
4. **What is the difference between physical and logical journaling?**
5. **How does journal checksumming improve reliability?**
6. **What is ext4 fast commit? How does it improve performance?**
7. **How does Btrfs achieve crash consistency without a journal?**
8. **What does e2fsck do in each of its 5 passes?**
9. **What is an orphan inode? How is it recovered?**
10. **Compare ext4 journal-based recovery with Btrfs CoW-based recovery.**

---

## Summary

- Crash consistency: partial writes → inconsistent FS; solved by journaling or CoW
- jbd2: ext4's journal implementation; write metadata to journal, then to disk
- Three modes: data=journal (safest), ordered (default, data-before-metadata), writeback (fastest)
- Journal transaction: descriptor + block copies + commit (with CRC32C checksum)
- XFS: logical logging (operations, not full blocks) with intent/done records
- Btrfs: CoW makes journaling unnecessary; superblock always points to consistent tree
- Recovery: ext4/XFS replay journal in seconds; Btrfs picks latest valid superblock
- fsck: offline consistency check (e2fsck, xfs_repair, btrfs check)
- Orphan list: handles files deleted while open across crashes

---

*Next: [Chapter 13 — Special File Systems: procfs, sysfs, tmpfs, debugfs](Chapter_13_Special_File_Systems.md)*
