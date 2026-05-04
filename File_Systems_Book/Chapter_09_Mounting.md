# Chapter 9: File System Mounting

## Learning Goals
- Understand the mount system call and kernel internals
- Know mount namespace, bind mounts, and propagation
- Master the mount tree and its data structures
- Understand rootfs, initramfs, and early boot mounting

---

## 9.1 Mount Fundamentals

```
Mounting = attaching a file system instance to a directory in the VFS tree

Before mount:
  /                          (rootfs: ext4 on /dev/sda2)
  ├── home/                  (empty directory)
  ├── boot/
  └── etc/

  $ mount -t ext4 /dev/sda3 /home

After mount:
  /                          (rootfs: ext4 on /dev/sda2)
  ├── home/                  ← mount point (now shows sda3 contents)
  │   ├── alice/
  │   └── bob/
  ├── boot/
  └── etc/

What the kernel does:
  1. Look up /dev/sda3 → struct block_device
  2. Look up file_system_type "ext4" → ext4_fs_type
  3. Call ext4_fill_super() → read superblock, create VFS objects
  4. Create struct mount linking to mount point dentry
  5. Attach to mount tree
```

---

## 9.2 Mount Data Structures

```c
/* fs/mount.h */
struct mount {
    struct hlist_node mnt_hash;        /* Hash table bucket */
    struct mount *mnt_parent;          /* Parent mount */
    struct dentry *mnt_mountpoint;     /* Dentry of mount point */
    struct vfsmount mnt;               /* Public portion */
    union {
        struct rcu_head mnt_rcu;
        struct llist_node mnt_llist;
    };
    struct list_head mnt_mounts;       /* List of children mounts */
    struct list_head mnt_child;        /* Sibling in parent's children */
    struct list_head mnt_instance;     /* Per-superblock instance list */
    const char *mnt_devname;           /* Device name "/dev/sda3" */
    struct list_head mnt_list;         /* Mount list */
    int mnt_id;                        /* Unique mount ID */
    int mnt_group_id;                  /* Peer group ID (propagation) */
    int mnt_expiry_mark;               /* Expiry status */
    struct hlist_head mnt_pins;
    struct hlist_head mnt_stuck_children;
};

/* include/linux/mount.h (public) */
struct vfsmount {
    struct dentry *mnt_root;           /* Root dentry of mounted FS */
    struct super_block *mnt_sb;        /* Superblock */
    int mnt_flags;                     /* MNT_NOSUID, MNT_NODEV, etc. */
};
```

### Mount Tree Visualization

```
Example system mount tree:

  mount(id=1, root)
  │ mnt_root = dentry(/)
  │ mnt_sb = super_block(ext4, /dev/sda2)
  │
  ├── mount(id=2, parent=1)
  │   mnt_mountpoint = dentry(/proc)
  │   mnt_root = dentry(/) [of procfs]
  │   mnt_sb = super_block(procfs)
  │
  ├── mount(id=3, parent=1)
  │   mnt_mountpoint = dentry(/sys)
  │   mnt_root = dentry(/) [of sysfs]
  │   mnt_sb = super_block(sysfs)
  │
  ├── mount(id=4, parent=1)
  │   mnt_mountpoint = dentry(/home)
  │   mnt_root = dentry(/) [of ext4 on sda3]
  │   mnt_sb = super_block(ext4, /dev/sda3)
  │
  └── mount(id=5, parent=1)
      mnt_mountpoint = dentry(/tmp)
      mnt_root = dentry(/) [of tmpfs]
      mnt_sb = super_block(tmpfs)

Process perspective (struct path):
  path.mnt → struct vfsmount (which mount are we in?)
  path.dentry → struct dentry (which file/dir within that mount?)
```

---

## 9.3 The mount() System Call

### Complete Flow

```
mount("ext4", "/dev/sda3", "/home", MS_RELATIME, "")
  │
  ▼
SYSCALL_DEFINE5(mount, ...)               [fs/namespace.c]
  │
  ▼
do_mount(dev_name, dir_name, type, flags, data)
  │
  ├── 1. path_mount(dev_name, path, type, flags, data)
  │
  ├── 2. Determine mount operation type:
  │     ├── MS_REMOUNT → do_remount()
  │     ├── MS_BIND → do_loopback() (bind mount)
  │     ├── MS_MOVE → do_move_mount()
  │     └── default → do_new_mount()
  │
  └── 3. do_new_mount(dev_name, path, type, sb_flags, data)
        │
        ├── get_fs_type(type)            ← Find "ext4" in registered FS list
        │     → returns ext4_fs_type
        │
        ├── fs_context_for_mount(fs_type, sb_flags)
        │     ├── Allocate struct fs_context
        │     └── fs_type->init_fs_context() → ext4_init_fs_context()
        │
        ├── Parse mount options (data string)
        │     vfs_parse_fs_string() for each option
        │
        ├── vfs_get_tree(fc)
        │     └── fc->ops->get_tree(fc)
        │           └── ext4_get_tree(fc)
        │                 └── get_tree_bdev(fc, ext4_fill_super)
        │                       │
        │                       ├── Open block device /dev/sda3
        │                       │     blkdev_get_by_path()
        │                       │
        │                       ├── Check if already mounted
        │                       │     (same device → share superblock)
        │                       │
        │                       ├── sget_fc() → allocate struct super_block
        │                       │
        │                       └── ext4_fill_super(sb, fc)
        │                             ├── Read on-disk superblock
        │                             ├── Verify magic (0xEF53)
        │                             ├── Parse FS features
        │                             ├── sb->s_op = &ext4_sops
        │                             ├── Allocate root inode
        │                             ├── sb->s_root = d_make_root(root_inode)
        │                             └── Initialize journal (jbd2)
        │
        ├── do_new_mount_fc(fc, path, mnt_flags)
        │     ├── vfs_create_mount(fc) → allocate struct mount
        │     ├── Set mnt_flags (nosuid, nodev, etc.)
        │     └── do_add_mount(mount, mp, mnt_flags)
        │           ├── lock_mount(path)
        │           ├── graft_tree(mount, mp)
        │           │     → Attach mount to mount tree
        │           │     → Set mnt_mountpoint, mnt_parent
        │           └── unlock_mount(mp)
        │
        └── Return 0 (success)
```

---

## 9.4 Mount Types

### Bind Mounts

```
Bind mount: makes a directory visible at another location

  $ mount --bind /home/user/project /opt/project

  Before:                             After:
  /home/user/project/                 /home/user/project/  (original)
  ├── src/                            ├── src/
  └── Makefile                        └── Makefile
                                      
  /opt/project/  (empty)              /opt/project/  (SAME content)
                                      ├── src/
                                      └── Makefile

  Both paths see the SAME files (same inodes, same data).
  Changes in one path appear in the other.

  Kernel: do_loopback() → clone mount
    → new struct mount, SAME struct super_block
    → mnt_root = same dentry as source

  Read-only bind mount:
    $ mount --bind /usr/share/doc /opt/docs
    $ mount -o remount,bind,ro /opt/docs
```

### OverlayFS Mounts

```
OverlayFS: layer multiple directories

  mount -t overlay overlay -o \
    lowerdir=/base,upperdir=/changes,workdir=/work /merged

  ┌─────────────┐
  │   /merged   │  ← Unified view (user sees this)
  └──────┬──────┘
         │ overlay
  ┌──────┴──────────────────────┐
  │  upperdir: /changes         │  ← Writable layer (modifications)
  │  lowerdir: /base            │  ← Read-only layer (base image)
  └─────────────────────────────┘

  Used by:
    - Docker/Podman containers (image layers)
    - Android (system + vendor overlay)
    - Live Linux ISOs
```

### Move Mount

```
  mount --move /old/mountpoint /new/mountpoint

  Detaches mount from old location, reattaches at new.
  No data copied — just changing the mount point.
```

---

## 9.5 Mount Propagation

```
Mount propagation controls how mount events spread between namespaces.

Propagation types:
  ┌─────────────────────────────────────────────────────────┐
  │ Type       │ Behavior                                    │
  │────────────┼─────────────────────────────────────────────│
  │ shared     │ Mount/unmount events propagate to all peers │
  │ private    │ No propagation (isolated)                   │
  │ slave      │ Receives events from master, doesn't send   │
  │ unbindable │ Cannot be bind-mounted, no propagation      │
  └─────────────────────────────────────────────────────────┘

Set propagation:
  $ mount --make-shared /mnt
  $ mount --make-private /mnt
  $ mount --make-slave /mnt
  $ mount --make-unbindable /mnt

Example: Container with shared /home
  Host mounts USB at /home/user/usb
  → Container also sees /home/user/usb (shared propagation)

  Host:                    Container:
  /home (shared) ◄────────► /home (shared peer)
    └── user/usb ◄─────────► user/usb  (propagated!)
```

---

## 9.6 Mount Namespaces

```
Each mount namespace has its own mount tree:

Process A (default namespace):     Process B (new namespace):
  /                                  /
  ├── home/ (ext4)                   ├── home/ (ext4) [same or different]
  ├── proc/ (procfs)                 ├── proc/ (procfs)
  ├── tmp/ (tmpfs)                   ├── tmp/ (tmpfs) [different instance]
  └── secret/ (ext4) ← exists       └── (no /secret mount)

Creating mount namespace:
  clone(CLONE_NEWNS)  or  unshare(CLONE_NEWNS)

  $ unshare --mount bash
  # Now in new mount namespace
  $ mount -t tmpfs tmpfs /secret
  # This mount is invisible to other namespaces

  Container isolation:
    1. unshare(CLONE_NEWNS) → new mount namespace
    2. Mount container rootfs
    3. pivot_root() → change root
    4. umount old root
    → Process sees container's FS tree only
```

---

## 9.7 /proc/mounts and mountinfo

```bash
# /proc/mounts (symlink to /proc/self/mounts)
$ cat /proc/mounts
/dev/sda2 / ext4 rw,relatime 0 0
proc /proc proc rw,nosuid,nodev,noexec 0 0
tmpfs /tmp tmpfs rw,nosuid,nodev 0 0
/dev/sda3 /home ext4 rw,relatime 0 0

# /proc/self/mountinfo (more detailed)
$ cat /proc/self/mountinfo
22 1 8:2 / / rw,relatime shared:1 - ext4 /dev/sda2 rw
 │  │ │   │ │ │          │           │    │         │
 │  │ │   │ │ │          │           │    │         └── FS options
 │  │ │   │ │ │          │           │    └── device
 │  │ │   │ │ │          │           └── FS type
 │  │ │   │ │ │          └── propagation info (shared:1)
 │  │ │   │ │ └── mount options
 │  │ │   │ └── mount point
 │  │ │   └── root within mounted FS
 │  │ └── major:minor device
 │  └── parent mount ID
 └── mount ID

# findmnt — user-friendly mount display
$ findmnt --target /home
TARGET SOURCE    FSTYPE OPTIONS
/home  /dev/sda3 ext4   rw,relatime
```

---

## 9.8 Early Boot Mounting

```
Boot mount sequence:

1. Bootloader loads kernel + initramfs
   │
   ▼
2. Kernel starts: mount rootfs (empty tmpfs)
   do_mounts_initrd() or initrd_load()
   │
   ▼
3. Unpack initramfs into rootfs
   populate_rootfs() → unpack cpio archive
   │
   ▼
4. initramfs /init runs:
   ├── Load storage drivers (modprobe)
   ├── Detect root device
   ├── mount root FS: mount /dev/sda2 /mnt/root
   │
   ▼
5. Switch root:
   ├── mount --move /proc /mnt/root/proc
   ├── mount --move /sys /mnt/root/sys
   ├── mount --move /dev /mnt/root/dev
   │
   └── exec switch_root /mnt/root /sbin/init
         └── Or: pivot_root + exec /sbin/init
   │
   ▼
6. systemd (PID 1) mounts remaining FS:
   ├── Parse /etc/fstab
   ├── Mount all local FS: mount -a
   ├── Mount /home, /boot, /tmp, etc.
   └── Start services

/etc/fstab format:
  # device      mount   type   options        dump pass
  /dev/sda2     /       ext4   defaults       0    1
  /dev/sda3     /home   ext4   defaults       0    2
  tmpfs         /tmp    tmpfs  defaults,size=2G 0  0
  /dev/sda1     /boot   vfat   defaults       0    2
```

### Android Mount Sequence

```
Android init mount sequence:

1. First-stage init (in ramdisk):
   ├── mount /proc, /sys, /dev (tmpfs)
   ├── Read device tree → find partitions
   ├── mount -t ext4 /dev/block/.../system /system (read-only)
   ├── mount -t ext4 /dev/block/.../vendor /vendor (read-only)
   └── switch_root to /system

2. Second-stage init:
   ├── Parse /fstab.{device}
   ├── mount -t ext4 /dev/block/.../userdata /data
   │     └── With file-based encryption (FBE)
   ├── mount -t ext4 /dev/block/.../cache /cache
   ├── mount overlayfs for dynamic partitions
   └── dm-verity for system, vendor (integrity check)

Typical automotive partitions:
  /system    → SquashFS or ext4 (read-only, dm-verity)
  /vendor    → ext4 (read-only)
  /data      → ext4 or f2fs (read-write, encrypted)
  /metadata  → ext4 (dm metadata)
  /persist   → ext4 (calibration data)
```

---

## 9.9 Unmounting

```
umount("/home")
  │
  ▼
SYSCALL_DEFINE2(umount, char __user *, name, int, flags)
  │
  ▼
ksys_umount(name, flags)
  │
  ├── Check: Is mount busy? (open files, cwd references)
  │     → If busy: return -EBUSY
  │     → MNT_FORCE: force unmount (data loss risk!)
  │     → MNT_DETACH: lazy unmount (detach now, cleanup when idle)
  │
  ├── do_umount(mnt, flags)
  │     ├── sync_filesystem(sb)          ← Flush dirty data
  │     ├── shrink_dcache_sb(sb)         ← Purge dentries
  │     ├── remove mount from tree
  │     │
  │     └── If last reference to superblock:
  │           deactivate_super(sb)
  │             ├── sb->s_op->put_super(sb)  ← FS cleanup
  │             ├── Close block device
  │             └── Free superblock
  │
  └── Return 0

Lazy unmount (MNT_DETACH):
  - Mount point removed from namespace immediately
  - FS stays alive until last reference dropped
  - Processes with open files can still access
  - Used when normal unmount fails

Force unmount (MNT_FORCE):
  - For network FS only (NFS)
  - Can cause data loss
  - Interrupts blocked operations
```

---

## Kernel Source References

```
Mount implementation:
  fs/namespace.c          ← mount/umount syscalls, mount tree management
  fs/super.c              ← Superblock allocation, fill_super dispatch
  fs/fs_context.c         ← New mount API (fs_context)
  fs/pnode.c              ← Mount propagation (shared, slave, private)

Headers:
  include/linux/mount.h   ← struct vfsmount
  fs/mount.h              ← struct mount (internal)
  include/linux/fs_context.h ← struct fs_context

OverlayFS:
  fs/overlayfs/           ← OverlayFS implementation

Boot:
  init/do_mounts.c        ← Root filesystem mounting
  init/initramfs.c        ← initramfs unpacking
```

---

## Interview Questions

1. **What happens inside the kernel when you run `mount -t ext4 /dev/sda1 /mnt`?**
2. **What are the key data structures for mounts? How do mount and vfsmount relate?**
3. **What is a bind mount? How does it differ from a symlink?**
4. **Explain mount propagation types: shared, private, slave, unbindable.**
5. **How do mount namespaces provide container isolation?**
6. **What is the difference between MNT_DETACH (lazy) and MNT_FORCE unmount?**
7. **Describe the Linux boot mount sequence from initramfs to systemd.**
8. **What is pivot_root() and when is it used?**
9. **How does OverlayFS work? Where is it used?**
10. **What information is in /proc/self/mountinfo that /proc/mounts lacks?**

---

## Summary

- `mount()` creates a new `struct mount` linking FS to a directory in the VFS tree
- `ext4_fill_super()` reads on-disk superblock, sets up VFS objects, initializes journal
- Bind mounts share the same superblock; OverlayFS layers multiple directories
- Mount propagation (shared/private/slave) controls event visibility across namespaces
- Mount namespaces give each process/container its own mount tree
- Boot: initramfs → load drivers → mount root → pivot_root → systemd mounts /etc/fstab
- Unmount: flush data → remove from tree → cleanup superblock (if last reference)

---

*Next: [Chapter 10 — Disk Layout and Block Allocation](Chapter_10_Disk_Layout_Allocation.md)*
