# Chapter 8: Directory Management and Path Lookup

## Learning Goals
- Understand how directories are stored on disk (ext4 HTree, XFS B+tree)
- Master the Linux path resolution algorithm (namei)
- Know dentry cache (dcache) mechanics and RCU-walk optimization
- Understand symlinks, hard links, and mount point crossing

---

## 8.1 Directory On-Disk Formats

### ext4 Directory Entries

```
On disk, a directory is a special file containing entries:

  ext4_dir_entry_2 structure:
  ┌──────────┬──────────┬────────────┬───────────┬──────────────────┐
  │ inode (4)│rec_len(2)│name_len (1)│file_type(1)│ name (variable) │
  └──────────┴──────────┴────────────┴───────────┴──────────────────┘

  Example directory block (4096 bytes):
  ┌────────────────────────────────────────────────────────────────────┐
  │ inode=2    rec_len=12  name_len=1 type=DIR  name="."             │
  │ inode=2    rec_len=12  name_len=2 type=DIR  name=".."            │
  │ inode=100  rec_len=20  name_len=10 type=REG name="README.md"     │
  │ inode=101  rec_len=16  name_len=6  type=DIR name="src"           │
  │ inode=102  rec_len=4036 name_len=8 type=REG name="Makefile"      │
  │ ←── Last entry has rec_len extending to end of block              │
  └────────────────────────────────────────────────────────────────────┘

  file_type values:
    EXT4_FT_UNKNOWN   = 0
    EXT4_FT_REG_FILE  = 1
    EXT4_FT_DIR       = 2
    EXT4_FT_CHRDEV    = 3
    EXT4_FT_BLKDEV    = 4
    EXT4_FT_FIFO      = 5
    EXT4_FT_SOCK      = 6
    EXT4_FT_SYMLINK   = 7
```

### ext4 HTree (Hashed B-Tree) Indexing

```
For large directories (thousands of entries), linear search is too slow.
ext4 uses HTree: a hash-based B-tree for O(1) lookup.

Small directory (< 1 block):
  ┌────────────────────────────────────┐
  │ Linear list of dir entries         │
  │ (searched sequentially)            │
  └────────────────────────────────────┘

Large directory (HTree):
  Root block (HTree root):
  ┌──────────────────────────────────────────────────────┐
  │ dx_root: hash_version, indirect_levels, count, limit │
  │ ┌────────────────────────────────────────────────┐   │
  │ │ hash=0x00000000 → block 5                      │   │
  │ │ hash=0x40000000 → block 12                     │   │
  │ │ hash=0x80000000 → block 23                     │   │
  │ │ hash=0xC0000000 → block 31                     │   │
  │ └────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────┘
           │              │              │
           ▼              ▼              ▼
  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
  │ Leaf block 5 │ │ Leaf block 12│ │ Leaf block 23│
  │ dir_entry_2  │ │ dir_entry_2  │ │ dir_entry_2  │
  │ dir_entry_2  │ │ dir_entry_2  │ │ dir_entry_2  │
  │ ...          │ │ ...          │ │ ...          │
  └──────────────┘ └──────────────┘ └──────────────┘

Lookup algorithm:
  1. Hash the filename: hash = TEA_hash("myfile.txt")
  2. Search HTree root for bracket containing hash
  3. Read the leaf block
  4. Linear search within leaf block
  → O(1) block reads instead of O(N) for large directories
```

### XFS Directory Format

```
XFS uses three formats based on directory size:

1. Short-form (inline in inode):
   ┌─────────────────────┐
   │ Inode data fork:    │
   │ entry1, entry2, ... │  (fits in inode, no extra blocks)
   └─────────────────────┘
   Used when: few entries, total size < inode data fork space

2. Block format (single data block):
   One directory block containing all entries + free space info

3. Leaf format (B+tree):
   ┌──────────────┐
   │ B+tree nodes │ → keyed by hash(name)
   └──────┬───────┘
          ▼
   ┌──────────────┐
   │ Data blocks  │ → contain actual dir entries
   └──────────────┘

4. Node format (multi-level B+tree):
   Used for very large directories (millions of entries)
```

---

## 8.2 Path Resolution (namei)

### The Algorithm

```
Path: /home/user/documents/file.txt

Step-by-step resolution:

  Start: nd.path = root dentry (/) of current mount namespace
  
  Component 1: "home"
    ├── d_lookup(root_dentry, "home")    ← dcache lookup
    ├── If miss: root_inode->i_op->lookup(root_inode, dentry, flags)
    │            → ext4_lookup()
    │            → search directory blocks for "home"
    │            → find inode number → iget_locked() → read inode from disk
    ├── Check: is "home" a mount point? If so, follow mount
    ├── Check: is "home" a symlink? If so, resolve it
    └── Advance: nd.path.dentry = home_dentry

  Component 2: "user"
    ├── Permission check: may_lookup(home_inode) → check +x on dir
    ├── d_lookup(home_dentry, "user")
    └── Advance: nd.path.dentry = user_dentry

  Component 3: "documents"
    ├── d_lookup(user_dentry, "documents")
    └── Advance: nd.path.dentry = documents_dentry

  Component 4: "file.txt"  (LAST component — special handling)
    ├── d_lookup(documents_dentry, "file.txt")
    ├── If O_CREAT: may create file here
    └── Return: nd.path.dentry = file_dentry, nd.inode = file_inode
```

### namei Source Code Flow

```c
/* fs/namei.c — simplified path walk */

/* Main entry point for path resolution */
static int filename_lookup(int dfd, struct filename *name,
                           unsigned flags, struct path *path)
{
    struct nameidata nd;
    int err;

    err = path_lookupat(&nd, flags, path);
    return err;
}

static int path_lookupat(struct nameidata *nd, unsigned flags,
                         struct path *path)
{
    int err;

    /* Start with RCU-walk (fast, lockless) */
    err = link_path_walk(name, nd);
    if (!err)
        err = complete_walk(nd);

    /* If RCU-walk failed, retry with ref-walk (slower, with locks) */
    if (err == -ECHILD) {
        nd->flags &= ~LOOKUP_RCU;
        err = link_path_walk(name, nd);
    }

    return err;
}

/* Walk through each component */
static int link_path_walk(const char *name, struct nameidata *nd)
{
    while (*name == '/')
        name++;  /* Skip leading slashes */

    while (1) {
        /* Extract component name */
        const char *next = strchrnul(name, '/');
        int len = next - name;

        /* Hash the name for dcache lookup */
        unsigned int hash = full_name_hash(nd->path.dentry, name, len);

        /* Permission check on parent directory */
        err = may_lookup(nd);

        /* Walk this component */
        err = walk_component(nd, WALK_MORE);

        /* Handle . and .. */
        if (name[0] == '.') {
            if (len == 1)
                continue;           /* "." = stay at current */
            if (name[1] == '.')
                follow_dotdot(nd);  /* ".." = go to parent */
        }

        name = next;
        while (*name == '/')
            name++;

        if (!*name)
            break;  /* Last component */
    }
    return 0;
}
```

---

## 8.3 RCU-Walk vs Ref-Walk

### RCU-Walk (Fast Path)

```
RCU-walk: Lock-free path traversal for cached paths

Performance comparison:
  RCU-walk:  ~100-200 ns for cached path lookup
  Ref-walk:  ~500-1000 ns (takes locks at each step)

How it works:
  1. rcu_read_lock()
  2. For each component:
     a. d_lookup_rcu(parent, name)     ← Use RCU, no locks
     b. Read dentry->d_seq seqcount
     c. Read dentry->d_inode pointer
     d. Verify seqcount unchanged      ← Detect concurrent modification
     e. If valid → proceed
     f. If invalid → abort RCU-walk, retry with ref-walk
  3. rcu_read_unlock()

  ┌─────────────────────────────────────────────────┐
  │              Path: /home/user/file.txt           │
  │                                                  │
  │  RCU-walk attempts:                              │
  │    "/" → d_lookup_rcu → seq_ok ✓                 │
  │    "home" → d_lookup_rcu → seq_ok ✓             │
  │    "user" → d_lookup_rcu → seq_ok ✓             │
  │    "file.txt" → d_lookup_rcu → seq_ok ✓         │
  │    SUCCESS! No locks taken at all.               │
  │                                                  │
  │  If any seq check fails:                         │
  │    Drop everything, switch to ref-walk           │
  │      → Take d_lock on each dentry               │
  │      → Increment refcount                        │
  │      → Slower but always correct                 │
  └─────────────────────────────────────────────────┘
```

### When RCU-Walk Fails

```
RCU-walk must drop down to ref-walk when:

  1. Concurrent rename/delete modifies a dentry (seqcount changed)
  2. Mount point crossing (need to check mount table)
  3. Automount triggered
  4. d_revalidate needed (NFS, FUSE — remote FS)
  5. Symlink resolution (some symlinks need locks)
  6. Permission check requires blocking operation

In practice: RCU-walk succeeds >99% of the time for local FS.
```

---

## 8.4 Symlinks and Hard Links

### Hard Links

```
Hard link: Multiple directory entries point to SAME inode

  $ ln /home/user/file.txt /home/user/link.txt

  Directory "user":
  ┌───────────┬──────────────┐
  │ inode=1234 │ "file.txt"  │
  │ inode=1234 │ "link.txt"  │  ← Same inode number!
  └───────────┴──────────────┘

  Inode 1234:
  ┌──────────────────────┐
  │ i_nlink = 2          │  ← Two directory entries
  │ i_size = 1024        │
  │ data blocks → [...]  │
  └──────────────────────┘

  Kernel path: sys_link() → vfs_link() → dir->i_op->link()
    → ext4_link()
      ├── Increment inode->i_nlink
      └── Add new directory entry

  Restrictions:
    - Cannot hard-link directories (prevents cycles)
    - Must be on same file system (same super_block)
    - Permission check: write on target directory
```

### Symbolic Links

```
Symlink: A special file whose content IS the target path

  $ ln -s /home/user/file.txt /home/user/symlink.txt

  Inode for symlink.txt:
  ┌──────────────────────────────┐
  │ i_mode = S_IFLNK | 0777     │
  │ Content: "/home/user/file.txt" │  ← The target path string
  └──────────────────────────────┘

  Resolution during path walk:
    walk_component("symlink.txt")
      ├── Found symlink inode
      ├── Check: nd->depth < MAXSYMLINKS (40)  ← Prevent infinite loops
      ├── inode->i_op->get_link()
      │     → ext4_get_link()
      │     → Read symlink target string
      ├── Push current path walk state
      └── Restart path walk with symlink target

  Symlink storage in ext4:
    - Short symlinks (< 60 bytes): stored inline in inode
    - Long symlinks: stored in a data block
```

### .. and Mount Points

```
Following ".." across mount points:

  Mounts:
    /dev/sda2 on /
    /dev/sda3 on /home

  Process does: chdir("/home/user") then chdir("..")

  Path walk for "..":
    current dentry = "user" (in /home filesystem)
    parent dentry = root of /home filesystem
    
    But wait! The root of /home filesystem IS a mount point.
    follow_dotdot() must cross the mount boundary:
      ├── Current at root of mounted FS
      ├── mnt->mnt_mountpoint → dentry "home" in root FS
      └── Continue ".." resolution in root FS
            → return root dentry of / FS
```

---

## 8.5 Directory Operations

### mkdir()

```
mkdir("/home/user/newdir", 0755)
  │
  ▼
SYSCALL_DEFINE2(mkdir, pathname, mode)
  │
  ▼
do_mkdirat(AT_FDCWD, name, mode)
  │
  ├── filename_lookup() → resolve parent: /home/user/
  │
  ├── mnt_want_write()                    ← Get write access to mount
  │
  ├── vfs_mkdir(idmap, dir_inode, dentry, mode)
  │     │
  │     ├── may_create(dir, dentry)       ← Permission check
  │     │
  │     └── dir->i_op->mkdir(idmap, dir, dentry, mode)
  │           │
  │           └── ext4_mkdir()             [fs/ext4/namei.c]
  │                 ├── ext4_new_inode_start_handle()
  │                 │     ├── Allocate inode from bitmap
  │                 │     ├── Set i_mode = S_IFDIR | mode
  │                 │     └── Start journal transaction
  │                 │
  │                 ├── ext4_init_new_dir()
  │                 │     ├── Allocate directory block
  │                 │     └── Create "." and ".." entries
  │                 │
  │                 ├── ext4_add_entry()   ← Add "newdir" to parent
  │                 │
  │                 ├── ext4_inc_count()   ← Increment parent nlink
  │                 │
  │                 └── ext4_journal_stop()
  │
  └── mnt_drop_write()
```

### unlink() and rmdir()

```
unlink("/home/user/file.txt")
  │
  ▼
vfs_unlink(dir_inode, dentry, delegated_inode)
  │
  ├── dir->i_op->unlink(dir, dentry)
  │     └── ext4_unlink()
  │           ├── ext4_delete_entry()     ← Remove dir entry
  │           ├── ext4_orphan_add()       ← Add to orphan list
  │           └── drop_nlink(inode)       ← inode->i_nlink--
  │
  └── If nlink reaches 0 AND no open files:
        → ext4_evict_inode()
          ├── Truncate all data blocks
          ├── Free inode from bitmap
          └── Remove from orphan list

  Note: If file is still open:
    - nlink = 0 but inode not freed yet
    - Data accessible via existing fd
    - Freed when last fd is closed (in fput → evict_inode)
    - Shows as "(deleted)" in /proc/pid/fd/
```

### readdir() / getdents()

```
getdents64(fd, buf, count)  (called by readdir/ls)
  │
  ▼
iterate_dir(file, ctx)                    [fs/readdir.c]
  │
  ▼
file->f_op->iterate_shared(file, ctx)
  │
  ▼
ext4_readdir()                            [fs/ext4/dir.c]
  │
  ├── For small directories:
  │     Read directory blocks sequentially
  │     For each entry: ctx->actor(ctx, name, namlen, offset, ino, type)
  │       → filldir64() → copy to user buffer
  │
  └── For HTree directories:
        Use hash tree to iterate in hash order
        (Note: directory iteration order is NOT alphabetical)

  User space:
    dp = opendir("/home/user");
    while ((entry = readdir(dp)) != NULL)
        printf("%s\n", entry->d_name);
    closedir(dp);
```

---

## 8.6 Rename Operation

```
rename("/tmp/old.txt", "/tmp/new.txt")
  │
  ▼
vfs_rename(rd)    /* rd = struct renamedata */
  │
  ├── Lock ordering:
  │     Lock old_dir->i_rwsem
  │     Lock new_dir->i_rwsem (if different)
  │     Lock source inode i_rwsem
  │     Lock target inode i_rwsem (if exists)
  │
  ├── old_dir->i_op->rename(old_dir, old_dentry, new_dir, new_dentry, flags)
  │     └── ext4_rename2()
  │           ├── Journal transaction start
  │           ├── If target exists: remove old target entry
  │           ├── Add new entry in new_dir
  │           ├── Remove old entry from old_dir
  │           ├── Update ".." in source if directory
  │           ├── Update nlink counts
  │           └── Journal transaction commit
  │
  └── d_move(old_dentry, new_dentry)    ← Update dcache
        ├── Rehash dentry (new parent, new name)
        └── Move in dcache tree

RENAME_NOREPLACE: fail if target exists
RENAME_EXCHANGE:  atomically swap two entries
RENAME_WHITEOUT:  used by OverlayFS
```

---

## 8.7 Case-Insensitive Directories

```
ext4 supports case-insensitive directories (since 5.2):

  mkfs.ext4 -O casefold /dev/sda1
  chattr +F /mnt/ci_dir/

  In /mnt/ci_dir/:
    "README.md" == "readme.md" == "Readme.MD"

  Implementation:
    - Uses UTF-8 normalization (nfkdi)
    - Custom dentry_operations:
        .d_hash = generic_ci_d_hash       ← Case-folded hash
        .d_compare = generic_ci_d_compare ← Case-folded compare
    - Hash table stores case-folded name
    - Preserves original case on disk

  Used in Android (since Android 11):
    /data partition is case-insensitive by default
```

---

## 8.8 Directory Notifications (inotify / fanotify)

```
Watching directory changes:

  int fd = inotify_init();
  int wd = inotify_add_watch(fd, "/home/user",
                              IN_CREATE | IN_DELETE | IN_MODIFY);
  
  while (1) {
      read(fd, buf, sizeof(buf));
      struct inotify_event *event = (struct inotify_event *)buf;
      printf("Event: %s %s\n",
             event->mask & IN_CREATE ? "CREATE" :
             event->mask & IN_DELETE ? "DELETE" : "OTHER",
             event->name);
  }

Kernel hooks (fs/notify/):
  Placed in VFS functions:
    vfs_create() → fsnotify_create()
    vfs_unlink() → fsnotify_unlink()
    vfs_rename() → fsnotify_move()
    After write() → fsnotify_modify()

  fanotify (newer, more powerful):
    - File access decisions (allow/deny)
    - File system-wide monitoring
    - Used by anti-malware, audit
```

---

## Kernel Source References

```
Path resolution:
  fs/namei.c              ← link_path_walk(), walk_component(), follow_dotdot()
  fs/namei.c              ← filename_lookup(), path_lookupat()

Directory operations:
  fs/ext4/namei.c         ← ext4_lookup(), ext4_create(), ext4_mkdir()
  fs/ext4/namei.c         ← ext4_rename2(), ext4_unlink()
  fs/ext4/dir.c           ← ext4_readdir()
  fs/ext4/hash.c          ← ext4 directory hash

dcache:
  fs/dcache.c             ← d_lookup(), d_alloc(), d_splice_alias()
  include/linux/dcache.h  ← struct dentry

Notifications:
  fs/notify/inotify/      ← inotify implementation
  fs/notify/fanotify/     ← fanotify implementation
```

---

## Interview Questions

1. **How does ext4 store directory entries on disk? What is HTree?**
2. **Trace the path resolution for /home/user/file.txt step by step.**
3. **What is RCU-walk? Why is it faster than ref-walk?**
4. **When does RCU-walk fail and fall back to ref-walk?**
5. **How are hard links different from symbolic links in terms of on-disk representation?**
6. **What happens when you unlink a file that is still open by another process?**
7. **How does ".." work across mount points?**
8. **What is MAXSYMLINKS and why is it needed?**
9. **Explain the rename() atomicity guarantee. What locks are involved?**
10. **How does case-insensitive lookup work in ext4?**

---

## Summary

- Directories stored as files containing (inode, name) entries
- ext4 uses HTree (hash-based B-tree) for O(1) lookup in large directories
- Path resolution (namei) walks component by component, checking dcache first
- RCU-walk: lock-free fast path (>99% success rate on local FS)
- Hard links: multiple dir entries → same inode; symlinks: file containing target path
- unlink() removes dir entry; inode freed only when nlink=0 AND no open fds
- rename() is atomic within single FS; uses careful lock ordering
- Mount point crossing handled transparently in follow_dotdot()

---

*Next: [Chapter 9 — File System Mounting](Chapter_09_Mounting.md)*
