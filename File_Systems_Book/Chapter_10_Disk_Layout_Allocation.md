# Chapter 10: Disk Layout and Block Allocation

## Learning Goals
- Understand ext4 disk layout in detail (superblock, block groups, bitmaps, inode table)
- Master block allocation strategies (bitmap, extent, delayed allocation)
- Know XFS and Btrfs allocation approaches
- Understand fragmentation and defragmentation

---

## 10.1 ext4 Disk Layout

### Overall Structure

```
ext4 divides the disk into Block Groups:

  ┌───────────┬──────────────┬──────────────┬──────────────┬────────┐
  │ Boot      │ Block Group 0│ Block Group 1│ Block Group 2│ ...    │
  │ Sector    │              │              │              │        │
  │(1024 bytes)│              │              │              │        │
  └───────────┴──────────────┴──────────────┴──────────────┴────────┘

Each Block Group (with 4KB blocks, 32K blocks/group = 128 MB/group):
  ┌─────────────────────────────────────────────────────────────────┐
  │ Superblock │ GDT │ Data Block │ Inode  │ Inode │ Data Blocks  │
  │ (copy)     │     │ Bitmap     │ Bitmap │ Table │              │
  │            │     │ (1 block)  │(1 block)│       │              │
  │ 1 block    │ N   │ 1 block    │1 block │ M     │ remaining    │
  │            │blocks│            │        │blocks │ blocks       │
  └─────────────────────────────────────────────────────────────────┘

Notes:
  - Superblock + GDT: NOT in every group (sparse_super feature)
    → Only in groups 0, 1, and powers of 3, 5, 7
    → Saves space: groups 0, 1, 3, 5, 7, 9, 25, 27, 49, ...
  - Data Block Bitmap: 1 bit per block → 32768 blocks per group
  - Inode Bitmap: 1 bit per inode
  - Inode Table: stores all inodes for this group
```

### ext4 Superblock (On-Disk)

```c
/* fs/ext4/ext4.h — on-disk superblock */
struct ext4_super_block {
    __le32  s_inodes_count;        /* Total inodes */
    __le32  s_blocks_count_lo;     /* Total blocks (low 32 bits) */
    __le32  s_r_blocks_count_lo;   /* Reserved blocks */
    __le32  s_free_blocks_count_lo;/* Free blocks */
    __le32  s_free_inodes_count;   /* Free inodes */
    __le32  s_first_data_block;    /* First data block (0 or 1) */
    __le32  s_log_block_size;      /* Block size = 1024 << this */
    __le32  s_blocks_per_group;    /* Blocks per group */
    __le32  s_inodes_per_group;    /* Inodes per group */
    __le32  s_mtime;               /* Last mount time */
    __le32  s_wtime;               /* Last write time */
    __le16  s_mnt_count;           /* Mount count */
    __le16  s_max_mnt_count;       /* Max mount count before fsck */
    __le16  s_magic;               /* Magic: 0xEF53 */
    __le16  s_state;               /* FS state (clean/error) */
    __le16  s_errors;              /* Error behavior */
    __le32  s_lastcheck;           /* Last fsck time */
    __le32  s_checkinterval;       /* Max interval between fscks */
    __le32  s_creator_os;          /* OS that created FS */
    __le32  s_rev_level;           /* Revision level */
    __le32  s_feature_compat;      /* Compatible features */
    __le32  s_feature_incompat;    /* Incompatible features */
    __le32  s_feature_ro_compat;   /* Read-only compat features */
    __u8    s_uuid[16];            /* Volume UUID */
    char    s_volume_name[16];     /* Volume label */
    /* ... many more fields ... */
};

/* Reading superblock with tune2fs: */
/* $ sudo tune2fs -l /dev/sda1 */
```

### ext4 Inode (On-Disk)

```c
/* fs/ext4/ext4.h — on-disk inode (256 bytes for ext4) */
struct ext4_inode {
    __le16  i_mode;                /* File type + permissions */
    __le16  i_uid;                 /* Owner UID (low 16 bits) */
    __le32  i_size_lo;             /* File size (low 32 bits) */
    __le32  i_atime;               /* Access time */
    __le32  i_ctime;               /* Change time */
    __le32  i_mtime;               /* Modification time */
    __le32  i_dtime;               /* Deletion time */
    __le16  i_gid;                 /* Group GID (low 16 bits) */
    __le16  i_links_count;         /* Hard link count */
    __le32  i_blocks_lo;           /* Blocks count (512B units) */
    __le32  i_flags;               /* File flags (EXT4_EXTENTS_FL, etc.) */
    
    union {
        struct {                   /* Classic indirect block map */
            __le32 i_block[15];
            /* [0-11]: direct blocks
             * [12]:   single indirect
             * [13]:   double indirect
             * [14]:   triple indirect
             */
        };
        struct ext4_extent_header i_extent_header;  /* Extent tree root */
    };
    
    __le32  i_generation;          /* File version (NFS) */
    __le32  i_file_acl_lo;        /* Extended attributes block */
    __le32  i_size_high;          /* File size (high 32 bits) */
    __le32  i_obso_faddr;         /* Obsolete */
    
    /* Extra fields for 256-byte inodes: */
    __le16  i_extra_isize;        /* Extra inode size */
    __le16  i_checksum_hi;        /* CRC32C checksum (high) */
    __le32  i_ctime_extra;        /* Extra change time (nsec) */
    __le32  i_mtime_extra;        /* Extra modify time (nsec) */
    __le32  i_atime_extra;        /* Extra access time (nsec) */
    __le32  i_crtime;             /* Creation time */
    __le32  i_crtime_extra;       /* Extra creation time (nsec) */
    __le32  i_version_hi;         /* High version number */
    __le32  i_projid;             /* Project ID */
};
```

---

## 10.2 Block Mapping: Indirect vs Extent

### Indirect Block Mapping (ext2/ext3 legacy)

```
Inode i_block[15]:
  [0]-[11]: Direct block pointers (12 × 4KB = 48 KB)
  [12]:     Single indirect → block of 1024 pointers (4 MB)
  [13]:     Double indirect → block of pointers to indirect blocks (4 GB)
  [14]:     Triple indirect → 3 levels of indirection (4 TB)

  ┌──────────────────────────────────────────────────┐
  │  Inode i_block[]                                 │
  │  [0] → Block 100                    (direct)     │
  │  [1] → Block 101                    (direct)     │
  │  ...                                             │
  │  [11] → Block 111                   (direct)     │
  │  [12] → Block 200 (indirect block)               │
  │           ├── ptr → Block 300                    │
  │           ├── ptr → Block 301                    │
  │           └── ... (1024 pointers for 4KB blocks) │
  │  [13] → Block 500 (double indirect)              │
  │           ├── ptr → Block 600 (indirect)         │
  │           │         ├── ptr → Block 700          │
  │           │         └── ...                      │
  │           └── ...                                │
  │  [14] → Triple indirect (same pattern, 3 levels) │
  └──────────────────────────────────────────────────┘

Problems with indirect blocks:
  - 1 GB file: needs ~256K indirect pointers (1 MB metadata!)
  - Random access requires traversing pointer tree
  - Not aware of contiguous extents
```

### Extent-Based Mapping (ext4)

```
An extent describes a contiguous range of blocks:

  struct ext4_extent {
      __le32  ee_block;       /* First logical block */
      __le16  ee_len;         /* Number of blocks (max 32768) */
      __le16  ee_start_hi;    /* Physical block (high 16 bits) */
      __le32  ee_start_lo;    /* Physical block (low 32 bits) */
  };

  12 bytes per extent → describes up to 128 MB contiguous data!

  ┌──────────────────────────────────────────────────┐
  │  Inode (extent tree root, fits 4 extents inline) │
  │                                                  │
  │  Extent 0: logical 0-1023 → physical 5000-6023  │
  │  Extent 1: logical 1024-2047 → physical 8000-9023│
  │  Extent 2: logical 2048-10239 → physical 12000-... │
  │  Extent 3: (unused)                              │
  └──────────────────────────────────────────────────┘

  For files with >4 extents:
  ┌───────────┐
  │ Inode     │     B-tree of extents
  │ (root)    │
  │ ┌─┬─┬─┐  │
  │ │I│I│I│  │ ← Internal nodes (index entries)
  │ └─┴─┴─┘  │
  └─────┬─────┘
        │
  ┌─────▼─────┐   ┌──────────┐   ┌──────────┐
  │ Leaf block│   │Leaf block│   │Leaf block│
  │ E E E E  │   │ E E E E  │   │ E E E E  │
  └──────────┘   └──────────┘   └──────────┘
  E = ext4_extent (each describes contiguous range)

Advantage: 1 GB contiguous file = 1 extent = 12 bytes metadata!
  vs indirect: ~1 MB metadata for same file
```

### Viewing Extents

```bash
# filefrag shows extent mapping
$ filefrag -v /var/log/syslog
Filesystem type is: ef53
File size of /var/log/syslog is 524288 (128 blocks of 4096 bytes)
 ext:  logical_offset: physical_offset: length:   expected: flags:
   0:        0..      63:   10240.. 10303:     64:
   1:       64..     127:   20480.. 20543:     64:  10304:   last,eof
/var/log/syslog: 2 extents found

# debugfs to examine inode
$ sudo debugfs -R "stat <131074>" /dev/sda1
Inode: 131074   Type: regular    Mode:  0644   Flags: 0x80000
Generation: 12345   Version: 0x00000001
Size: 524288
EXTENTS:
(0-63):10240-10303, (64-127):20480-20543
```

---

## 10.3 Block Allocation Strategies

### ext4 Multi-Block Allocator (mballoc)

```
mballoc goals:
  1. Allocate contiguous blocks (larger extents)
  2. Place related data near each other
  3. Minimize fragmentation
  4. Fast allocation

Algorithm:
  ┌──────────────────────────────────────────────────────┐
  │ Request: allocate N blocks for inode                  │
  │                                                       │
  │ Step 1: Normalization                                 │
  │   Small file: request rounded to small chunk          │
  │   Large file: request rounded to full stripe          │
  │                                                       │
  │ Step 2: Try allocation strategies (in order):          │
  │   a) Exact goal block (if locality hint available)     │
  │   b) Same block group as inode                        │
  │   c) Block group with most free space                 │
  │   d) Any block group                                  │
  │                                                       │
  │ Step 3: Within block group, search bitmap              │
  │   - Use buddy allocator for fast free-space search    │
  │   - Prefer power-of-2 aligned allocations             │
  │                                                       │
  │ Step 4: Store result, update bitmap, return           │
  └──────────────────────────────────────────────────────┘

Buddy system for block allocation:
  Block bitmap (32768 blocks):
  Level 0: ████████████████████████████████ (individual blocks)
  Level 1: ██████████████████  (pairs)
  Level 2: ████████████  (groups of 4)
  ...
  Level N: █  (entire group)

  Finding 8 contiguous blocks: check level 3 → O(1) instead of scanning bitmap
```

### Delayed Allocation (ext4)

```
Delayed allocation = don't allocate blocks until writeback

  write(fd, data, 64KB):
    ┌──────────────────────────────────────────┐
    │ Time of write():                          │
    │   - Copy data to page cache pages         │
    │   - Mark pages dirty                      │
    │   - Reserve metadata space (quota)        │
    │   - Do NOT allocate data blocks           │
    │   - ext4_da_reserve_space()               │
    └──────────────────────────────────────────┘
    
    ... more writes happen ...
    
    ┌──────────────────────────────────────────┐
    │ Time of writeback (background or sync):   │
    │   - ext4_writepages() called              │
    │   - Scan dirty pages → build extents      │
    │   - ext4_da_get_blocks() → NOW allocate   │
    │   - mballoc finds best contiguous range   │
    │   - Write data + metadata                 │
    └──────────────────────────────────────────┘

Benefits:
  1. Write 1000 × 4KB → allocate 1 × 4MB extent (contiguous!)
  2. Temp files deleted before writeback → no disk allocation at all
  3. Better block choices (more info about file's total size)

Risk:
  - Data loss between write() and writeback on power failure
  - Mitigated by: dirty_expire_centisecs, journal, fsync()
```

### Preallocation

```
Preallocate space without writing data:

  fallocate(fd, 0, 0, 1073741824);  /* Reserve 1 GB */

  Kernel:
    ext4_fallocate() → ext4_ext_map_blocks()
    → Allocate extents, mark as uninitialized
    → Blocks reserved but contain no data
    
  Benefits:
    - Guarantees space availability
    - Blocks allocated contiguously
    - Prevents ENOSPC during later writes
    - No data written (fast)
    
  Used by: databases, video recording, log files

  Punch hole (deallocate within file):
    fallocate(fd, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE,
              offset, length);
    → Free blocks within a file without changing size
    → Creates sparse region
```

---

## 10.4 XFS Block Allocation

```
XFS organizes the disk into Allocation Groups (AGs):

  ┌──────────────┬──────────────┬──────────────┬──────────────┐
  │     AG 0     │     AG 1     │     AG 2     │     AG 3     │
  └──────────────┴──────────────┴──────────────┴──────────────┘

Each AG is independent (own B+trees, own lock):
  ┌────────────────────────────────────────────┐
  │ AG Header:                                  │
  │   AG Free Space B+tree (by block number)   │
  │   AG Free Space B+tree (by size)           │
  │   AG Inode B+tree                          │
  │   AG Free Inode B+tree                     │
  │   Data blocks                              │
  └────────────────────────────────────────────┘

Advantages of AG-based design:
  - Multiple AGs → parallel allocation (one lock per AG)
  - B+tree by size → fast "find N contiguous blocks" query
  - Great for multi-threaded workloads

XFS extent format:
  [startoff, startblock, blockcount, flag]
  → Stored in per-inode B+tree (not in inode directly)
  → Inline in inode if few extents (data fork)
```

---

## 10.5 Btrfs Block Allocation

```
Btrfs uses Copy-on-Write (CoW):

  Write to existing block:
    1. Allocate NEW block
    2. Write modified data to new block
    3. Update parent pointer to new block
    4. Old block becomes free (after snapshot ref drops)

  ┌──────────────────────────────────────────────────────┐
  │  Before write:                                        │
  │  Root → Node A → Leaf [Block 100: "Hello"]           │
  │                                                       │
  │  After write to Block 100:                            │
  │  Root' → Node A' → Leaf' [Block 200: "World"]        │
  │  (old tree preserved for snapshots)                   │
  │                                                       │
  │  Root → Node A → Leaf [Block 100: "Hello"] ← snapshot│
  └──────────────────────────────────────────────────────┘

Btrfs space management:
  Chunk/Block Group allocator:
    - Disk divided into chunks (~1 GB each)
    - Each chunk assigned a type: Data, Metadata, System
    - Within chunks: extent tree tracks free space
    
  Extent tree:
    - Global B-tree mapping: [logical offset → physical offset, length]
    - Shared reference counts for CoW snapshots
```

---

## 10.6 Free Space Management

```
Method           │ Used By        │ Approach
─────────────────┼────────────────┼─────────────────────────
Bitmap           │ ext2/ext3/ext4 │ 1 bit per block
B+tree (by size) │ XFS            │ Sorted by free extent size
B+tree (by addr) │ XFS            │ Sorted by block address
Extent tree      │ Btrfs          │ Global extent allocation tree
Free space cache │ Btrfs          │ Cache free extents in file
Free space tree  │ Btrfs          │ B-tree of free space (v2)

ext4 bitmap scanning:
  Block group bitmap: 32768 bits (4 KB block = 128 MB)
  ┌────────────────────────────────────────────────────┐
  │ 1111101110000000011111000000000000111111111111110... │
  │ ^^^^              ^^^^^^^^^^^^^^^^                  │
  │ used              FREE (contiguous!)                │
  └────────────────────────────────────────────────────┘
  
  Buddy allocator speeds this up by pre-computing
  contiguous free ranges at each power-of-2 level.
```

---

## 10.7 Fragmentation

```
Types of fragmentation:

1. Internal fragmentation:
   Wasted space within allocated blocks
   File = 5000 bytes, block = 4096 → uses 2 blocks (8192 bytes)
   → 3192 bytes wasted

2. External fragmentation:
   File data scattered across non-contiguous blocks
   
   Before:
   ┌─A─┬─B─┬─A─┬─C─┬─A─┬─B─┬─A─┐
   
   After defrag:
   ┌─A─┬─A─┬─A─┬─A─┬─B─┬─B─┬─C─┐

Checking fragmentation:
  $ sudo e4defrag -c /home
  Total/best extents                     │ 12345/10000
  Average size of extents                │ 32.5 KB
  Fragmentation score                    │ 15
  [0-30: no problem, 31-55: a little, 56+: need defrag]

  $ filefrag /var/log/syslog
  /var/log/syslog: 5 extents found   ← 5 extents for one file

Defragmentation:
  # ext4 online defragmentation
  $ sudo e4defrag /home/user/large_file.dat
  
  # Btrfs defragmentation
  $ sudo btrfs filesystem defragment /mnt/data/
  
  # XFS has xfs_fsr (file system reorganizer)
  $ sudo xfs_fsr /dev/sda1
```

---

## 10.8 Examining Disk Layout

```bash
# ext4 superblock info
$ sudo dumpe2fs /dev/sda1 | head -50
Filesystem volume name:   <none>
Last mounted on:          /
Filesystem UUID:          abcd1234-...
Filesystem features:      has_journal ext_attr resize_inode dir_index
                          filetype extent 64bit flex_bg sparse_super2
Block count:              2621440
Block size:               4096
Blocks per group:         32768
Inodes per group:         8192
Inode size:               256

# Block group details
$ sudo dumpe2fs /dev/sda1 | grep -A 10 "Group 0"
Group 0: (Blocks 0-32767)
  Primary superblock at 0, Group descriptors at 1-1
  Block bitmap at 256 (+256)
  Inode bitmap at 272 (+272)
  Inode table at 288-799 (+288)
  24000 free blocks, 8000 free inodes

# debugfs interactive exploration
$ sudo debugfs /dev/sda1
debugfs: stats              ← FS statistics
debugfs: ls /               ← List root directory
debugfs: stat <2>           ← Inode 2 is always root dir
debugfs: imap <131074>      ← Where is inode 131074 on disk?
debugfs: blocks <131074>    ← Which blocks does it use?
debugfs: dump <131074> /tmp/recovered  ← Extract file
```

---

## Kernel Source References

```
ext4 disk layout:
  fs/ext4/ext4.h           ← On-disk structures (ext4_super_block, ext4_inode)
  fs/ext4/super.c          ← Superblock read/write (ext4_fill_super)
  fs/ext4/balloc.c         ← Block allocation bitmap
  fs/ext4/mballoc.c        ← Multi-block allocator
  fs/ext4/extents.c        ← Extent tree management
  fs/ext4/ialloc.c         ← Inode allocation

XFS:
  fs/xfs/libxfs/xfs_alloc.c ← AG-based block allocation
  fs/xfs/libxfs/xfs_bmap.c  ← Extent/B+tree mapping
  fs/xfs/libxfs/xfs_ialloc.c ← Inode allocation

Btrfs:
  fs/btrfs/extent-tree.c    ← Extent allocation
  fs/btrfs/free-space-cache.c ← Free space management
```

---

## Interview Questions

1. **Draw the ext4 block group layout. What does each section contain?**
2. **Compare indirect block mapping (ext2) with extent-based mapping (ext4).**
3. **What is the ext4 multi-block allocator (mballoc)? How does it reduce fragmentation?**
4. **Explain delayed allocation. What are benefits and risks?**
5. **How does XFS achieve parallel allocation with Allocation Groups?**
6. **What is Copy-on-Write in Btrfs? How does it affect block allocation?**
7. **What is fallocate()? How does preallocation help databases?**
8. **How do you check and fix fragmentation on ext4?**
9. **What is the ext4 buddy allocator?**
10. **Compare ext4 bitmap-based vs XFS B+tree-based free space tracking.**

---

## Summary

- ext4 layout: boot sector → block groups → {superblock, GDT, bitmaps, inode table, data}
- Extents: describe contiguous ranges efficiently (12 bytes → up to 128 MB)
- mballoc: buddy allocator + goal-based allocation for contiguous blocks
- Delayed allocation: defer block allocation to writeback for better extents
- XFS: Allocation Groups with B+trees enable parallel allocation
- Btrfs: Copy-on-Write means blocks are never overwritten in place
- Preallocation (fallocate): reserve contiguous space without writing data
- Fragmentation monitoring: filefrag, e4defrag -c, defragmentation tools

---

*Next: [Chapter 11 — Linux File Systems: ext2/ext3/ext4, XFS, Btrfs](Chapter_11_Linux_File_Systems.md)*
