# Chapter 25: Root Filesystem Initialization

## Learning Goals
- Understand how the root filesystem is mounted during boot
- Know the difference between initramfs, initrd, and direct root mount
- Grasp the pivot_root and switch_root mechanisms
- Understand filesystem type detection and mount flow

---

## 25.1 Root Filesystem Mounting Overview

```
Root FS Transition During Boot:

Phase 1: No filesystem at all
  ├── Kernel executing from physical memory
  └── Only memblock allocator available

Phase 2: rootfs (kernel internal)
  ├── start_kernel() → vfs_caches_init()
  │   └── mnt_init()
  │       └── init_rootfs()     ← Create internal rootfs
  │       └── init_mount_tree() ← Mount rootfs at /
  └── This is a tmpfs/ramfs — purely in-memory

Phase 3: initramfs/initrd (if present)
  ├── populate_rootfs()
  │   ├── Unpack initramfs cpio archive into rootfs
  │   └── OR: load initrd as separate ramdisk
  └── /init from initramfs runs (user-space begins)

Phase 4: Real root filesystem
  ├── initramfs /init:
  │   ├── Loads required drivers (storage, FS, RAID)
  │   ├── Assembles root device (md, lvm, dm-crypt)
  │   ├── Mounts real root at /sysroot
  │   └── switch_root /sysroot /sbin/init
  └── OR: kernel directly mounts root=
      └── prepare_namespace() → mount_root()
```

---

## 25.2 VFS Initialization

```
VFS (Virtual Filesystem Switch) Init:

vfs_caches_init():
  │
  ├── dcache_init()           ← Dentry hash table + slab cache
  │   └── Hash table with millions of buckets
  │
  ├── inode_init()            ← Inode hash table + slab cache
  │
  ├── files_init()            ← File descriptor table cache
  │
  ├── mnt_init()
  │   ├── sysfs_init()        ← Mount sysfs
  │   ├── init_rootfs()       ← Register rootfs type
  │   ├── init_mount_tree()   ← Mount rootfs at /
  │   │   ├── vfs_kern_mount(rootfs_fs_type)
  │   │   ├── create_mnt_ns()  ← Create mount namespace
  │   │   └── set_fs_pwd/root  ← Set current->fs
  │   └── Now we have / (rootfs)
  │
  └── bdev_cache_init()       ← Block device cache

After this, the kernel has a root directory but it's empty.
```

---

## 25.3 initramfs Unpacking

```
populate_rootfs() — Unpack initramfs:

Built-in initramfs (linked into kernel):
  ┌─────────────────────────────────────────────┐
  │  __initramfs_start → __initramfs_end        │
  │  (cpio archive embedded in kernel image)    │
  │                                             │
  │  unpack_to_rootfs(__initramfs_start, size)  │
  │  ├── Detect format (cpio newc)              │
  │  ├── For each entry:                        │
  │  │   ├── Create directory or file in rootfs │
  │  │   └── Set permissions and ownership      │
  │  └── Result: /init, /dev/console, /bin/...  │
  └─────────────────────────────────────────────┘

External initramfs (loaded by bootloader):
  ┌─────────────────────────────────────────────┐
  │  Bootloader loads initramfs.img to RAM      │
  │  Passes address via initrd_start/end        │
  │                                             │
  │  If cpio format:                            │
  │    unpack_to_rootfs(initrd_start, size)     │
  │  If old initrd format:                      │
  │    Create /initrd.image file                │
  │    Load to ramdisk /dev/ram0                │
  └─────────────────────────────────────────────┘
```

```bash
# Creating an initramfs
# Method 1: From a directory
find . | cpio -o -H newc | gzip > initramfs.img

# Method 2: Using dracut (Fedora/RHEL)
dracut /boot/initramfs-$(uname -r).img $(uname -r)

# Method 3: Using mkinitramfs (Debian/Ubuntu)
mkinitramfs -o /boot/initrd.img $(uname -r)

# Inspect contents
lsinitramfs /boot/initrd.img
# or
zcat initramfs.img | cpio -t
```

---

## 25.4 Kernel Direct Root Mount

```
prepare_namespace() — When No initramfs /init:

If there's no /init in initramfs, kernel mounts root directly:

prepare_namespace()
  │
  ├── wait_for_device_probe()    ← Wait for root device driver
  │   └── Timeout if device not found
  │
  ├── Parse root= parameter
  │   ├── root=/dev/sda1    → Major/Minor from device name
  │   ├── root=UUID=xxxx    → Lookup by UUID  
  │   ├── root=PARTUUID=xx  → Lookup by partition UUID
  │   └── root=/dev/nfs     → NFS root
  │
  ├── mount_root()
  │   ├── Try each registered filesystem type:
  │   │   ├── mount_block_root("ext4", flags)
  │   │   ├── mount_block_root("xfs", flags)
  │   │   ├── mount_block_root("btrfs", flags)
  │   │   └── ... until one succeeds
  │   └── mount_nfs_root() if NFS
  │
  ├── devtmpfs_mount()           ← Mount devtmpfs at /dev
  │
  └── init_mount(".", "/", NULL, MS_MOVE, NULL)
      └── Move new root to /
```

---

## 25.5 switch_root and pivot_root

```
switch_root — Used by initramfs:

initramfs /init does:
  1. Mount real root: mount /dev/sda1 /sysroot
  2. Move special mounts:
     mount --move /dev  /sysroot/dev
     mount --move /proc /sysroot/proc
     mount --move /sys  /sysroot/sys
  3. switch_root /sysroot /sbin/init
     ├── Change root directory to /sysroot
     ├── Delete all files in old rootfs (free RAM)
     ├── chroot to /sysroot
     └── exec /sbin/init (replaces current process)

pivot_root — Used by containers:
  pivot_root(new_root, put_old)
  ├── new_root becomes /
  ├── old root becomes accessible at put_old
  ├── Old root NOT deleted (unlike switch_root)
  └── Used by: Docker, LXC, systemd-nspawn

Key difference:
  switch_root: deletes old root (initramfs cleanup)
  pivot_root: preserves old root (container isolation)
```

---

## 25.6 Filesystem Type Detection

```c
/* How mount_root() detects filesystem type */

/* Try each registered filesystem */
static int __init mount_block_root(char *name, int flags)
{
    struct file_system_type **p;
    
    /* Iterate all registered filesystem types */
    for (p = file_systems; *p; p++) {
        struct file_system_type *type = *p;
        
        /* Skip non-block filesystems */
        if (!(type->fs_flags & FS_REQUIRES_DEV))
            continue;
        
        /* Try to mount with this filesystem type */
        int err = init_mount(name, "/root", type->name,
                            flags, NULL);
        if (err == 0)
            return 0;  /* Success! */
    }
    
    panic("VFS: Unable to mount root fs on %s", name);
}

/* Each filesystem has a magic number at a known offset */
/* ext4: 0xEF53 at offset 1080 */
/* xfs:  "XFSB" at offset 0   */
/* btrfs: "_BHRfS_M" at offset 64 */
```

---

## Kernel Source References

| Function/File | Path | Purpose |
|-------|------|---------|
| vfs_caches_init() | fs/dcache.c | Initialize VFS caches |
| mnt_init() | fs/namespace.c | Mount namespace init |
| populate_rootfs() | init/initramfs.c | Unpack initramfs |
| prepare_namespace() | init/do_mounts.c | Direct root mount |
| mount_root() | init/do_mounts.c | Try filesystem types |
| init_mount_tree() | fs/namespace.c | Mount rootfs at / |
| devtmpfs_mount() | drivers/base/devtmpfs.c | Mount /dev |

---

## Interview Questions

**Q1: What is the difference between initramfs and initrd?**
A: initramfs is a cpio archive unpacked directly into rootfs (tmpfs in kernel) — it replaces the root. initrd is an older mechanism where a compressed filesystem image is loaded into a ramdisk block device (/dev/ram0) and mounted as root. initramfs is simpler, more flexible, and the current standard. initrd requires block device and filesystem drivers compiled into the kernel.

**Q2: Why is an initramfs needed at all?**
A: The kernel needs to mount the root filesystem, but the root device might require drivers that aren't compiled into the kernel (e.g., NVMe, RAID, dm-crypt, filesystem modules). initramfs provides a minimal userspace that loads these drivers, assembles complex root devices, and then switches to the real root. Without it, all root-path drivers must be built into the kernel.

**Q3: How does switch_root differ from pivot_root?**
A: `switch_root` deletes all files in the old rootfs (freeing initramfs RAM), changes root to the new directory, and execs init. It's used during boot to transition from initramfs to real root. `pivot_root` moves the old root to a subdirectory without deleting it. It's used by containers (Docker, LXC) for filesystem isolation where the old root may still be needed.

---

## Summary

- VFS initialization creates an in-memory rootfs — the first `/` directory
- initramfs cpio archive is unpacked into rootfs, providing `/init` and basic tools
- If initramfs has `/init`, it runs and eventually calls `switch_root` to real root
- If no `/init`, kernel directly mounts `root=` device via `prepare_namespace()`
- `switch_root` deletes initramfs and transitions to real root filesystem
- Filesystem type is auto-detected by trying each registered type

---

*Next: [Chapter 26 — User Space Initialization](Chapter_26_User_Space_Init.md)*
