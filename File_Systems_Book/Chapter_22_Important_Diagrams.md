# Chapter 22: Important Diagrams

## Learning Goals
- Consolidate key architectural diagrams for quick reference
- Understand VFS object relationships visually
- Visualize on-disk structures for ext4, XFS, Btrfs
- Master the page cache and writeback lifecycle diagrams

---

## 22.1 VFS Object Relationship Map

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VFS OBJECT RELATIONSHIPS                              │
│                                                                         │
│  ┌──────────────┐                                                       │
│  │ task_struct   │  (per process)                                       │
│  │  └─files──────┼───────────────────────────────┐                      │
│  └──────────────┘                               │                      │
│                                                  ▼                      │
│                                    ┌──────────────────┐                 │
│                                    │  files_struct     │                 │
│                                    │   └─fdt──────────┼──┐              │
│                                    └──────────────────┘  │              │
│                                                          ▼              │
│                   FD Table: [0] [1] [2] [3] [4] ...                    │
│                              │   │   │   │                              │
│                              ▼   ▼   ▼   ▼                              │
│  ┌───────────────────────────────────────────────────────┐              │
│  │              struct file (one per open())             │              │
│  │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐     │              │
│  │  │file[0] │  │file[1] │  │file[2] │  │file[3] │     │              │
│  │  │f_pos=0 │  │f_pos=0 │  │f_pos=0 │  │f_pos=42│     │              │
│  │  │f_mode  │  │f_mode  │  │f_mode  │  │f_mode  │     │              │
│  │  │f_op────┼  │f_op    │  │f_op    │  │f_op────┼──┐  │              │
│  │  │f_path  │  │f_path  │  │f_path  │  │f_path  │  │  │              │
│  │  │ .dentry┼┐ │        │  │        │  │ .dentry┼┐ │  │              │
│  │  └────────┘│ └────────┘  └────────┘  └────────┘│ │  │              │
│  └────────────┼─────────────────────────────────────┘ │  │              │
│               │                                    │  │  │              │
│               ▼                                    ▼  │  │              │
│  ┌──────────────────┐              ┌──────────────────┐│  │              │
│  │  struct dentry    │              │  struct dentry    ││  │              │
│  │  d_name="stdin"  │              │  d_name="data.txt"││  │              │
│  │  d_parent────────┼──→ ...       │  d_parent─────────┼┤  │              │
│  │  d_inode─────────┼──┐           │  d_inode──────────┼┤  │              │
│  │  d_subdirs       │  │           │  d_subdirs        ││  │              │
│  └──────────────────┘  │           └──────────────────┘│  │              │
│                        │                               │  │              │
│                        ▼                               ▼  │              │
│  ┌──────────────────────────────────────────────────────┐ │              │
│  │                struct inode (one per file on disk)    │ │              │
│  │  i_ino = 131073                                      │ │              │
│  │  i_mode = 0644 (regular file, rw-r--r--)             │ │              │
│  │  i_uid, i_gid                                        │ │              │
│  │  i_size = 8192                                       │ │              │
│  │  i_nlink = 1                                         │ │              │
│  │  i_atime, i_mtime, i_ctime                           │ │              │
│  │  i_op ────→ ext4_file_inode_operations               │ │              │
│  │  i_fop ───→ ext4_file_operations ←────────────────────┘              │
│  │  i_mapping → address_space (page cache)              │               │
│  │  i_sb ────→ struct super_block                       │               │
│  └──────────────────┬───────────────────────────────────┘               │
│                     │                                                   │
│                     ▼                                                   │
│  ┌──────────────────────────────────────────────────────┐               │
│  │            struct super_block (one per mount)         │               │
│  │  s_dev = 8:1                                         │               │
│  │  s_type → ext4_fs_type                               │               │
│  │  s_op → ext4_sops                                    │               │
│  │  s_root → dentry of "/"                              │               │
│  │  s_fs_info → ext4_sb_info (FS-private data)          │               │
│  │  s_inodes (list of all inodes)                       │               │
│  └──────────────────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 22.2 Dentry Cache (dcache) Structure

```
dcache — In-memory tree mirroring directory hierarchy:

                    ┌──────────┐
                    │ dentry   │
                    │ name="/" │ ← root dentry (sb->s_root)
                    │ parent=self│
                    └────┬─────┘
                         │ d_subdirs
            ┌────────────┼────────────┐
            ▼            ▼            ▼
       ┌──────────┐ ┌──────────┐ ┌──────────┐
       │ "home"   │ │ "etc"    │ │ "var"    │
       │ d_inode→ │ │ d_inode→ │ │ d_inode→ │
       └────┬─────┘ └────┬─────┘ └──────────┘
            │             │
       ┌────┴────┐   ┌────┴────┐
       ▼         ▼   ▼         ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ "user"   │ │ "root"   │ │ "passwd" │
  │ d_inode→ │ │ d_inode→ │ │ d_inode→ │
  └────┬─────┘ └──────────┘ └──────────┘
       │
       ▼
  ┌──────────┐ ┌──────────┐
  │"data.txt"│ │"DELETED" │ ← Negative dentry
  │ d_inode→ │ │ d_inode  │   (caches "file doesn't exist")
  │ inode obj│ │  = NULL  │
  └──────────┘ └──────────┘

  Lookup: /home/user/data.txt
  ├→ d_lookup("home") → hit → advance
  ├→ d_lookup("user") → hit → advance
  └→ d_lookup("data.txt") → hit → return inode immediately!

  dcache LRU:
    Unused dentries (refcount=0) go on LRU list
    Memory pressure → shrink_dcache_sb() prunes LRU dentries
```

---

## 22.3 ext4 On-Disk Layout

```
ext4 disk layout:

Block 0          Block 1          Block 2     ...
┌────────────┬────────────────┬────────────┬──────────────────┐
│ Boot Block │  Superblock    │   Group    │                  │
│ (1024B)    │  (1024B)       │   Desc.   │  ...             │
│            │  (at offset    │   Table   │                  │
│ (MBR/GPT) │   1024 in      │            │                  │
│            │   block group 0│            │                  │
└────────────┴────────────────┴────────────┴──────────────────┘

Block Group Layout (repeated for each group):
┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────────┐
│Super     │Group     │Data Block│Inode     │Inode     │Data Blocks   │
│Block     │Descriptors│Bitmap   │Bitmap    │Table     │              │
│(backup?) │(backup?) │(1 block) │(1 block) │(N blocks)│(data)        │
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────────┘

The superblock and group descriptors are backed up in certain groups
(groups 0, 1, 3, 5, 7, 9, 25, 27, 49, ...) — sparse superblock feature

Flex Block Groups (flex_bg):
┌──────────────────────────────────────────────────────────────────────┐
│ BG 0            │ BG 1            │ BG 2            │ BG 3           │
│ SB+GDT │ bitmaps│ more bitmaps   │ inode tables    │ data blocks    │
│ for all │ for   │ for adjacent   │ for all groups  │ for all groups │
│ groups  │ group0│ groups         │ in flex group   │ in flex group  │
└──────────────────────────────────────────────────────────────────────┘
  ↑                                                                    ↑
  Metadata concentrated at front → less seeking for metadata operations
```

### ext4 Inode On-Disk Structure

```
ext4 inode (256 bytes default):

Offset  Size   Field                Description
──────  ─────  ────────────────── ────────────────────
0x00    2      i_mode              File type + permissions
0x02    2      i_uid               Owner UID (low 16 bits)
0x04    4      i_size_lo           File size (low 32 bits)
0x08    4      i_atime             Last access time
0x0C    4      i_ctime             Change time
0x10    4      i_mtime             Modification time
0x14    4      i_dtime             Deletion time
0x18    2      i_gid               Group ID (low 16 bits)
0x1A    2      i_links_count       Hard link count
0x1C    4      i_blocks_lo         Block count (512-byte sectors)
0x20    4      i_flags             EXT4_EXTENTS_FL, etc.
0x28    60     i_block[15]         Block map / extent tree root
                                    ┌─ Direct blocks [0-11]
                                    ├─ Indirect [12]
                                    ├─ Double indirect [13]
                                    └─ Triple indirect [14]
                                    OR (with EXT4_EXTENTS_FL):
                                    ┌─ Extent header (12 bytes)
                                    └─ 4 extent entries (48 bytes)
0x64    4      i_generation        File version (NFS)
0x68    4      i_file_acl_lo       Extended attributes block
0x6C    4      i_size_high         File size (high 32 bits)
...     ...    (extra fields for 256-byte inode)
0x82    4      i_crtime            Creation time
0x86    4      i_crtime_extra      Creation time (extra precision)
0x8A    2      i_extra_isize       Size of extra inode fields
0x8C    2      i_checksum_hi       Inode checksum (high)
0x90    4      i_projid            Project ID
```

### ext4 Extent Tree

```
ext4 extent:

  Extent Header (12 bytes):
  ┌──────────┬──────────┬──────────┬──────────┐
  │ magic    │ entries  │ max      │ depth    │
  │ 0xF30A   │ count    │ entries  │ (0=leaf) │
  └──────────┴──────────┴──────────┴──────────┘

  Leaf Extent Entry (12 bytes):
  ┌──────────┬──────────┬──────────┐
  │ ee_block │ ee_len   │ ee_start │  (physical block48)
  │ (logical)│ (count)  │ (high16) │
  └──────────┴──────────┴──────────┘

  Example: file mapped by 2 extents
  ┌──────────────────────────────────┐
  │ Extent Header: entries=2, depth=0│
  ├──────────────────────────────────┤
  │ Extent 1: log=0, len=100,       │  Logical blocks 0-99 →
  │           phys=500000            │  Physical blocks 500000-500099
  ├──────────────────────────────────┤
  │ Extent 2: log=100, len=50,      │  Logical blocks 100-149 →
  │           phys=600000            │  Physical blocks 600000-600049
  └──────────────────────────────────┘

  For large files, extent tree has depth > 0:
         ┌─────┐ (root in inode, depth=1)
         │ Idx │
         └──┬──┘
         ┌──┴──────────┐
         ▼              ▼
    ┌─────────┐    ┌─────────┐  (leaf blocks on disk)
    │Ext1 Ext2│    │Ext3 Ext4│
    │Ext3 ...│    │...      │
    └─────────┘    └─────────┘
```

---

## 22.4 XFS Architecture

```
XFS Allocation Group Layout:

  ┌──────────────────────────────────────────────────────────┐
  │                    XFS Volume                             │
  │  ┌────────────┬────────────┬────────────┬────────────┐   │
  │  │ AG 0       │ AG 1       │ AG 2       │ AG 3       │   │
  │  └────────────┴────────────┴────────────┴────────────┘   │
  └──────────────────────────────────────────────────────────┘

  Each Allocation Group (AG):
  ┌──────────────────────────────────────────────────────────┐
  │ AG Header Section                                        │
  │ ┌──────────┬──────────┬──────────┬──────────┐            │
  │ │ AGF      │ AGI      │ AGFL     │ Free     │            │
  │ │ (Free    │ (Inode   │ (Free    │ Space    │            │
  │ │  Space   │  Info)   │  List)   │ B+Trees  │            │
  │ │  Header) │          │          │          │            │
  │ └──────────┴──────────┴──────────┴──────────┘            │
  │                                                          │
  │ B+Trees per AG:                                          │
  │   Free space by block number (bnobt)                     │
  │   Free space by size (cntbt)                             │
  │   Inode B+tree (inobt)                                   │
  │   Free inode B+tree (finobt)                             │
  │   Reverse mapping B+tree (rmapbt)                        │
  │   Reference count B+tree (refcountbt)                    │
  │                                                          │
  │ Data blocks + inode chunks                               │
  └──────────────────────────────────────────────────────────┘

  XFS advantages:
    - Each AG is independently lockable → parallel allocation
    - B+trees for everything → O(log n) lookups
    - Scales to exabytes
```

---

## 22.5 Btrfs Architecture

```
Btrfs uses Copy-on-Write B-trees for everything:

  ┌──────────────────────────────────────────────────────────────────┐
  │                    Btrfs Volume                                   │
  │                                                                  │
  │  Superblock (at fixed offsets: 64KB, 64MB, 256GB)               │
  │  Points to → Root Tree Root                                      │
  │                                                                  │
  │  ┌──────────────────────────────────────────────────────┐       │
  │  │              Root Tree                                │       │
  │  │  ┌────────────────────────────────────────────────┐  │       │
  │  │  │        Pointers to sub-trees                    │  │       │
  │  │  └──┬──────────┬──────────┬──────────┬────────────┘  │       │
  │  │     │          │          │          │                │       │
  │  │     ▼          ▼          ▼          ▼                │       │
  │  │  FS Tree   Extent Tree  Chunk Tree  Checksum Tree    │       │
  │  │  (files,   (block       (logical→  (CRC32 per       │       │
  │  │   dirs,    allocation)   physical   data block)      │       │
  │  │   inodes)               mapping)                     │       │
  │  └──────────────────────────────────────────────────────┘       │
  │                                                                  │
  │  Copy-on-Write:                                                  │
  │  ┌─────────────────────────────────────┐                        │
  │  │ Before write:                        │                        │
  │  │    Root → A → B → C (leaf with data) │                        │
  │  │                                      │                        │
  │  │ After write (modify leaf C):         │                        │
  │  │    Root' → A' → B' → C' (new copy)   │                        │
  │  │    Root → A → B → C (old, snapshot)   │                        │
  │  │                                      │                        │
  │  │ Old tree is the snapshot!            │                        │
  │  └─────────────────────────────────────┘                        │
  │                                                                  │
  │  Chunk/Device Layer:                                             │
  │  ┌──────────────────────────────────────────────┐               │
  │  │  Chunk Tree maps logical → physical           │               │
  │  │  Logical addr 0-1GB → Disk1 offset 0-1GB     │               │
  │  │  Logical addr 1-2GB → Disk2 offset 0-1GB     │               │
  │  │  (handles RAID, multi-device)                 │               │
  │  └──────────────────────────────────────────────┘               │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 22.6 Page Cache Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        PAGE CACHE                                     │
│                                                                      │
│  struct address_space (one per inode):                               │
│  ┌──────────────────────────────────────────────────────┐            │
│  │  host = inode                                         │            │
│  │  i_pages = xarray (radix tree replacement)            │            │
│  │  a_ops = ext4_aops                                    │            │
│  │  nrpages = 5                                          │            │
│  └──────┬───────────────────────────────────────────────┘            │
│         │                                                            │
│         ▼  XArray (indexed by page offset)                           │
│  ┌──────────────────────────────────────────────────────┐            │
│  │ Index:  [0]      [1]      [2]      [3]      [4]     │            │
│  │         │        │        │        │        │        │            │
│  │         ▼        ▼        ▼        ▼        ▼        │            │
│  │  ┌─────────┐┌─────────┐┌─────────┐┌─────────┐┌──────┐           │
│  │  │ folio 0 ││ folio 1 ││ folio 2 ││ folio 3 ││folio4│           │
│  │  │ uptodate││ uptodate││  DIRTY  ││  DIRTY  ││uptod.│           │
│  │  │ clean   ││ clean   ││writeback││ pending ││clean │           │
│  │  │ LRU:act ││ LRU:act ││ LRU:act ││ LRU:act ││LRU: │           │
│  │  │         ││         ││         ││         ││inact │           │
│  │  └─────────┘└─────────┘└─────────┘└─────────┘└──────┘           │
│  └──────────────────────────────────────────────────────┘            │
│                                                                      │
│  Page States:                                                        │
│  ┌────────────────────────────────────────────────────┐              │
│  │ PG_uptodate  — Contents match disk (valid data)    │              │
│  │ PG_dirty     — Modified, not yet written to disk   │              │
│  │ PG_writeback — Currently being written to disk     │              │
│  │ PG_locked    — Under I/O, readers must wait        │              │
│  │ PG_lru       — On LRU list for reclaim             │              │
│  │ PG_active    — Recently accessed (active LRU)      │              │
│  │ PG_referenced — Reference bit for LRU decisions    │              │
│  └────────────────────────────────────────────────────┘              │
│                                                                      │
│  Folio lifecycle:                                                    │
│  ┌────────┐    ┌──────────┐    ┌─────────┐    ┌───────────┐         │
│  │Allocate│───→│Read from │───→│Uptodate │───→│Write (app)│         │
│  │(empty) │    │disk      │    │(clean)  │    │→ Dirty    │         │
│  └────────┘    └──────────┘    └─────────┘    └─────┬─────┘         │
│                                                      │               │
│                                     ┌────────────────┘               │
│                                     ▼                                │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                       │
│  │Writeback │───→│Uptodate  │───→│Reclaim   │                       │
│  │(to disk) │    │(clean)   │    │(free mem)│                       │
│  └──────────┘    └──────────┘    └──────────┘                       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 22.7 Writeback Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                       WRITEBACK SUBSYSTEM                             │
│                                                                      │
│  Triggers:                                                           │
│  ┌─────────────────────┐                                             │
│  │ dirty_background_   │  ← Background writeback starts here        │
│  │ ratio (default 10%) │                                             │
│  ├─────────────────────┤                                             │
│  │ dirty_ratio         │  ← Foreground throttling starts here       │
│  │ (default 20%)       │     (writer blocks until flushed)          │
│  ├─────────────────────┤                                             │
│  │ dirty_expire_       │  ← Pages older than this get written       │
│  │ centisecs (3000)    │     (30 seconds default)                   │
│  ├─────────────────────┤                                             │
│  │ dirty_writeback_    │  ← Kernel wakes flusher this often        │
│  │ centisecs (500)     │     (5 seconds default)                    │
│  └─────────────────────┘                                             │
│                                                                      │
│  Writeback flow:                                                     │
│  ┌─────────────────┐                                                 │
│  │ Per-BDI flusher  │  (one per block device / backing_dev_info)    │
│  │ kworker thread   │                                                │
│  └────────┬────────┘                                                 │
│           │                                                          │
│           ▼                                                          │
│  ┌─────────────────────────────────────┐                             │
│  │ wb_writeback()                       │                             │
│  │ └─→ writeback_sb_inodes()            │                             │
│  │     └─→ for each dirty inode:        │                             │
│  │         __writeback_single_inode()    │                             │
│  │         └─→ do_writepages()           │                             │
│  │             └─→ a_ops->writepages()   │                             │
│  │                 └─→ ext4_writepages() │                             │
│  └─────────────────────┬───────────────┘                             │
│                        │                                              │
│                        ▼                                              │
│  ┌─────────────────────────────────┐                                 │
│  │ ext4_writepages()                │                                 │
│  │ ├─→ Find dirty folios           │                                 │
│  │ ├─→ ext4_map_blocks() (resolve  │                                 │
│  │ │   delayed allocation)          │                                 │
│  │ ├─→ Build bios                   │                                 │
│  │ └─→ submit_bio(WRITE)           │                                 │
│  └─────────────────────┬───────────┘                                 │
│                        │                                              │
│                        ▼                                              │
│  ┌─────────────────────────────────┐                                 │
│  │ Block layer → Device → Disk     │                                 │
│  │ On completion:                   │                                 │
│  │   clear_page_dirty_for_io()     │                                 │
│  │   end_page_writeback()           │                                 │
│  │   → Page transitions:            │                                 │
│  │     DIRTY → WRITEBACK → CLEAN   │                                 │
│  └─────────────────────────────────┘                                 │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 22.8 Journal (jbd2) Transaction Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                     JBD2 TRANSACTION LIFECYCLE                        │
│                                                                      │
│  Running Transaction (T_RUNNING):                                    │
│  ┌──────────────────────────────────────────────┐                    │
│  │ Application writes arrive                     │                    │
│  │ jbd2_journal_start() → get handle             │                    │
│  │ jbd2_journal_get_write_access() → buffer tracked│                  │
│  │ Modify metadata buffer                         │                    │
│  │ jbd2_journal_dirty_metadata() → mark for journal│                  │
│  │ jbd2_journal_stop() → release handle           │                    │
│  │ Multiple handles accumulate in one transaction  │                    │
│  └──────────────────────┬───────────────────────┘                    │
│                         │ Commit timer or sync request                │
│                         ▼                                            │
│  Committing Transaction (T_LOCKED → T_FLUSH → T_COMMIT):           │
│  ┌──────────────────────────────────────────────┐                    │
│  │ Step 1: Lock transaction (no new handles)     │                    │
│  │                                               │                    │
│  │ Step 2: Write descriptor blocks               │                    │
│  │   ┌─────────┐┌─────────┐┌─────────┐          │                    │
│  │   │ Desc    ││ Meta    ││ Meta    │          │                    │
│  │   │ Block   ││ Block 1 ││ Block 2 │  ...    │                    │
│  │   │ (tags)  ││ (copy)  ││ (copy)  │          │                    │
│  │   └─────────┘└─────────┘└─────────┘          │                    │
│  │                                               │                    │
│  │ Step 3: Wait for all journal writes to disk   │                    │
│  │                                               │                    │
│  │ Step 4: Write COMMIT block (with checksum)    │                    │
│  │   ┌─────────────┐                             │                    │
│  │   │ Commit Block│  (marks transaction durable)│                    │
│  │   │ checksum    │                             │                    │
│  │   └─────────────┘                             │                    │
│  │                                               │                    │
│  │ Step 5: Issue disk flush (write barrier)      │                    │
│  └──────────────────────┬───────────────────────┘                    │
│                         │                                            │
│                         ▼                                            │
│  Checkpointing (T_FINISHED):                                        │
│  ┌──────────────────────────────────────────────┐                    │
│  │ After original metadata blocks written to    │                    │
│  │ final locations, journal space reclaimed      │                    │
│  │                                               │                    │
│  │ Journal is circular:                          │                    │
│  │ ┌──────────────────────────────────────┐      │                    │
│  │ │  [T3 data] [T4 data] ... [T2 old]   │      │                    │
│  │ │        ↑ head          ↑ tail        │      │                    │
│  │ │        (newest)        (oldest)      │      │                    │
│  │ └──────────────────────────────────────┘      │                    │
│  └──────────────────────────────────────────────┘                    │
│                                                                      │
│  Recovery (on mount after crash):                                    │
│  ┌──────────────────────────────────────────────┐                    │
│  │ Scan journal from tail to head                │                    │
│  │ If transaction has commit block → replay it   │                    │
│  │ If no commit block → discard (incomplete)     │                    │
│  │ → Filesystem consistent after replay          │                    │
│  └──────────────────────────────────────────────┘                    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 22.9 LRU and Page Reclaim

```
┌──────────────────────────────────────────────────────────────────────┐
│                    PAGE RECLAIM (LRU Lists)                           │
│                                                                      │
│  Two-list LRU scheme:                                                │
│                                                                      │
│  Active List (recently accessed pages):                              │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┐                              │
│  │ P1  │ P2  │ P3  │ P4  │ P5  │ P6  │  ← Hot end (MRU)           │
│  └─────┴─────┴─────┴─────┴─────┴─────┘  → Cold end                 │
│                               │                                      │
│           Demotion (no recent access)                                │
│                               │                                      │
│                               ▼                                      │
│  Inactive List (candidates for eviction):                            │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┐                              │
│  │ P7  │ P8  │ P9  │ P10 │ P11 │ P12 │  → Eviction                 │
│  └─────┴─────┴─────┴─────┴─────┴─────┘                              │
│                               │                                      │
│                               │ If accessed → promote back to Active │
│                               │ If not → evict                       │
│                               ▼                                      │
│                        ┌────────────┐                                │
│                        │   Evict    │                                │
│                        │ ├─ Clean?  │→ Free immediately              │
│                        │ └─ Dirty?  │→ Write back first, then free   │
│                        └────────────┘                                │
│                                                                      │
│  Separate LRU lists for:                                             │
│  ┌────────────────────────────────────────┐                          │
│  │ LRU_INACTIVE_ANON  — Anonymous pages   │                          │
│  │ LRU_ACTIVE_ANON    — Anonymous (hot)   │                          │
│  │ LRU_INACTIVE_FILE  — File-backed pages │ ← Most relevant for FS  │
│  │ LRU_ACTIVE_FILE    — File-backed (hot) │ ← Most relevant for FS  │
│  │ LRU_UNEVICTABLE    — Locked pages      │                          │
│  └────────────────────────────────────────┘                          │
│                                                                      │
│  swappiness controls anon vs file balance:                           │
│    swappiness=60 (default): balanced                                 │
│    swappiness=0:  prefer evicting file pages (keep anon in RAM)      │
│    swappiness=100: treat equally                                     │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 22.10 Mount Tree

```
Mount tree (visible via /proc/self/mountinfo):

  ┌───────────────────────────────────────────────────────┐
  │ Mount Tree                                             │
  │                                                       │
  │  [mount: /] ──── rootfs / ext4 on /dev/sda1           │
  │   ├── [mount: /proc] ── proc on proc                  │
  │   ├── [mount: /sys] ── sysfs on sysfs                 │
  │   │    └── [mount: /sys/kernel/debug] ── debugfs       │
  │   ├── [mount: /dev] ── devtmpfs                        │
  │   │    └── [mount: /dev/pts] ── devpts                 │
  │   ├── [mount: /tmp] ── tmpfs                           │
  │   ├── [mount: /home] ── ext4 on /dev/sda2             │
  │   └── [mount: /mnt/usb] ── vfat on /dev/sdb1          │
  │                                                       │
  │  struct mount {                                        │
  │    mnt_parent → parent mount                           │
  │    mnt_mountpoint → dentry where mounted               │
  │    mnt_root → root dentry of this FS                   │
  │    mnt_sb → super_block                                │
  │    mnt_child → sibling list                            │
  │    mnt_mounts → children list                          │
  │  }                                                     │
  │                                                       │
  │  Path resolution: when crossing mount point,           │
  │  lookup_mnt() switches from parent FS to child FS     │
  └───────────────────────────────────────────────────────┘
```

---

## Interview Questions

1. **Draw the VFS object relationship: task_struct → files_struct → file → dentry → inode.**
2. **Explain the ext4 on-disk layout: block groups, bitmaps, inode table.**
3. **How does the extent tree work? Draw a 2-level extent tree.**
4. **Diagram the page cache and explain folio states.**
5. **What triggers writeback? Draw the writeback flow.**
6. **Explain the jbd2 transaction lifecycle with a diagram.**
7. **How does LRU page reclaim work for file-backed pages?**
8. **Draw the mount tree and explain mount point crossing.**
9. **Compare ext4, XFS, and Btrfs on-disk architectures visually.**
10. **Draw the complete I/O stack from process to device.**

---

## Summary

This chapter provides essential reference diagrams:
- VFS object map: task → files_struct → file → dentry → inode → super_block
- dcache: tree of dentries mirroring directory hierarchy, enables fast path lookups
- ext4 layout: block groups, superblock/GDT/bitmaps/inode table/data blocks
- ext4 extents: header + entries mapping logical→physical blocks efficiently
- XFS: allocation groups with independent B+trees for parallel allocation
- Btrfs: copy-on-write B-trees, root tree → sub-trees for fs/extents/chunks
- Page cache: xarray-indexed folios with state machine (empty→uptodate→dirty→writeback→clean)
- Writeback: dirty thresholds trigger flusher threads → writepages → submit_bio
- Journal: running → committing (descriptor+metadata+commit blocks) → checkpointing
- LRU: active/inactive lists, demotion, promotion, eviction of file-backed pages
- Mount tree: hierarchical struct mount linking superblocks to path tree

---

*Next: [Chapter 23 — Glossary](Chapter_23_Glossary.md)*
