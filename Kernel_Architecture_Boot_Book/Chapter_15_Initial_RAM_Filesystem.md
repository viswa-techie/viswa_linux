# Chapter 15: Initial RAM Filesystem

## Learning Goals
- Understand why initramfs/initrd exists and what problems it solves
- Know the difference between initramfs and initrd
- Understand early user-space initialization
- Grasp the root filesystem transition process

---

## 15.1 initramfs Concept

**initramfs** (initial RAM filesystem) is a temporary root filesystem loaded into memory during boot. It provides the minimal environment needed to mount the real root filesystem.

```
The Chicken-and-Egg Problem:

  Kernel needs to mount root filesystem → root is on disk
  Disk needs a driver → driver is a kernel module (.ko)
  Module is on the root filesystem → which isn't mounted yet!

  Solution: initramfs — a small RAM-based filesystem that
  contains just enough drivers and tools to mount the real root.

Boot Flow with initramfs:

Bootloader
    │
    ├── Loads kernel into RAM
    ├── Loads initramfs into RAM
    │
    ▼
Kernel starts
    │
    ├── Extracts initramfs into rootfs (tmpfs)
    ├── Runs /init from initramfs
    │     │
    │     ├── Load storage driver modules (USB, NVMe, SCSI)
    │     ├── Load filesystem modules (ext4, btrfs)
    │     ├── Assemble RAID/LVM if needed
    │     ├── Unlock encrypted volumes (LUKS)
    │     ├── Find and mount real root filesystem
    │     └── pivot_root / switch_root → real root
    │
    ▼
Real root filesystem mounted
    │
    ▼
/sbin/init (systemd) starts
```

What initramfs typically contains:

```
initramfs contents:

/
├── init                    ← First script/binary to run (PID 1 initially)
├── bin/
│   ├── busybox            ← Swiss-army-knife utility (ls, mount, sh, etc.)
│   ├── sh → busybox
│   └── mount → busybox
├── sbin/
│   ├── modprobe           ← Module loading
│   └── switch_root        ← Transition to real root
├── lib/
│   └── modules/
│       └── 6.8.0/
│           ├── ext4.ko    ← Filesystem driver
│           ├── sd_mod.ko  ← SCSI disk driver
│           ├── ahci.ko    ← SATA driver
│           ├── xhci_hcd.ko← USB host controller
│           └── nvme.ko    ← NVMe driver
├── etc/
│   └── fstab              ← Minimal mount config
├── dev/
│   ├── console
│   └── null
├── proc/                   ← Mount point for procfs
├── sys/                    ← Mount point for sysfs
└── run/                    ← Runtime data
```

---

## 15.2 initrd Concept

**initrd** (initial RAM disk) is the older mechanism, now largely replaced by initramfs.

```
initrd vs initramfs:

initrd (legacy):
┌────────────────────────────────────────────┐
│  Block device image (ext2 filesystem)       │
│  ├── Loaded as a RAM disk (/dev/ram0)      │
│  ├── Mounted as a block device             │
│  ├── Double-cached (page cache + RAM disk) │
│  ├── Fixed size (must pre-allocate)        │
│  └── pivot_root() to switch to real root   │
└────────────────────────────────────────────┘

initramfs (modern):
┌────────────────────────────────────────────┐
│  cpio archive (possibly compressed)         │
│  ├── Extracted into tmpfs (rootfs)         │
│  ├── No block device needed                │
│  ├── Single-cached (only page cache)       │
│  ├── Grows/shrinks dynamically             │
│  └── switch_root() to real root            │
└────────────────────────────────────────────┘
```

| Feature | initrd | initramfs |
|---------|--------|-----------|
| Format | Filesystem image (ext2/3) | cpio archive |
| Backing | RAM disk block device | tmpfs (page cache) |
| Caching | Double (wasteful) | Single (efficient) |
| Size | Fixed at creation | Dynamic |
| Transition | `pivot_root()` | `switch_root()` |
| Usage | Legacy (pre-2.6) | Modern (2.6+) |
| Tools | `mkinitrd` | `mkinitramfs`, `dracut` |

---

## 15.3 Early User-Space Initialization

The initramfs `/init` script performs boot-critical tasks:

```bash
#!/bin/sh
# Simplified initramfs /init script

# Mount essential virtual filesystems
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev

# Parse kernel command line
CMDLINE=$(cat /proc/cmdline)

# Load essential drivers
modprobe ext4
modprobe sd_mod
modprobe ahci

# Wait for root device to appear
ROOT_DEV=$(echo $CMDLINE | sed 's/.*root=\([^ ]*\).*/\1/')
echo "Waiting for $ROOT_DEV..."
while [ ! -b "$ROOT_DEV" ]; do
    sleep 0.1
done

# Handle encrypted root (if LUKS)
if echo $CMDLINE | grep -q "cryptroot"; then
    cryptsetup luksOpen $ROOT_DEV cryptroot
    ROOT_DEV=/dev/mapper/cryptroot
fi

# Handle LVM root
if echo $CMDLINE | grep -q "lvm"; then
    lvm vgchange -ay
fi

# Mount real root filesystem
mkdir -p /newroot
mount -t ext4 $ROOT_DEV /newroot

# Clean up and switch to real root
umount /proc /sys /dev
exec switch_root /newroot /sbin/init
# switch_root: deletes everything from initramfs,
# changes root to /newroot, runs /sbin/init
```

---

## 15.4 Root Filesystem Mounting

```
Root Filesystem Transition:

BEFORE switch_root:                 AFTER switch_root:

/  (initramfs tmpfs)                / (real root — /dev/sda2)
├── init                           ├── bin/
├── bin/                           ├── etc/
│   └── busybox                    ├── lib/
├── lib/                           ├── sbin/
│   └── modules/                   │   └── init (systemd)
├── sbin/                          ├── usr/
├── dev/                           ├── var/
├── proc/                          ├── home/
├── sys/                           ├── dev/
└── newroot/ ← real root           ├── proc/
    ├── bin/     mounted here      ├── sys/
    ├── etc/                       └── tmp/
    ├── sbin/
    └── ...                        initramfs memory: FREED
```

```c
/* Kernel implementation of switch_root equivalent */

/* switch_root does:
   1. Walks initramfs, deletes all files (free memory)
   2. Calls chdir(new_root)
   3. Calls mount(".", "/", NULL, MS_MOVE, NULL)  ← Move mount
   4. Calls chroot(".")  ← Change root
   5. Calls execv(init_path)  ← Replace process with real init
*/
```

Building initramfs:

```bash
# Generate initramfs for current kernel (Debian/Ubuntu)
update-initramfs -u -k $(uname -r)
# Output: /boot/initrd.img-$(uname -r)

# Fedora/RHEL: dracut
dracut --force /boot/initramfs-$(uname -r).img $(uname -r)

# List contents of initramfs
lsinitramfs /boot/initrd.img-$(uname -r) | head -30
# or
unmkinitramfs /boot/initrd.img-$(uname -r) /tmp/initramfs-contents/

# Create custom minimal initramfs
mkdir -p initramfs/{bin,dev,etc,lib,proc,sys,sbin,newroot}
cp /bin/busybox initramfs/bin/
# Create /init script
# Package:
cd initramfs
find . | cpio -H newc -o | gzip > ../initramfs.cpio.gz

# Build initramfs into kernel (no separate file needed)
CONFIG_INITRAMFS_SOURCE="/path/to/initramfs-directory"
```

When initramfs is NOT needed:

```
initramfs is optional when:
  - Root device driver is built into kernel (=y, not =m)
  - Root filesystem type is built in (=y)
  - No LVM, RAID, encryption
  - No complex network root setup
  
Common in embedded Linux (Android):
  - All drivers for root are built-in
  - Root = eMMC/UFS partition
  - Boot image includes a small ramdisk for init stage
  - init directly mounts system/vendor partitions
```

---

## Interview Questions

**Q1: Why does initramfs exist? Why can't the kernel just mount the root filesystem directly?**
A: The kernel may need drivers (storage controller, filesystem) that are compiled as modules (.ko files). These modules reside on the root filesystem — creating a chicken-and-egg problem. initramfs provides a minimal filesystem in RAM with just enough drivers and tools to find, initialize, and mount the real root.

**Q2: What is the difference between initrd and initramfs?**
A: initrd is a block device image (ext2) loaded into a RAM disk — double-buffered (page cache + RAM disk memory), fixed size. initramfs is a cpio archive extracted into tmpfs — single-buffered, dynamically sized, more efficient. initramfs replaced initrd in Linux 2.6+ and is the modern standard.

**Q3: When can you skip initramfs?**
A: When all drivers needed for root filesystem access are compiled into the kernel (not as modules): storage driver (CONFIG_MMC=y, CONFIG_AHCI=y), filesystem (CONFIG_EXT4_FS=y), and no LVM/RAID/encryption. Common in embedded Linux where the hardware is known and fixed. Android typically uses a small initramfs but with most drivers built-in.

**Q4: What does switch_root do?**
A: `switch_root` (1) deletes all files from initramfs (freeing memory), (2) moves the mount of the new root to `/`, (3) chroots into the new root, and (4) exec's the real init process. After switch_root, the initramfs memory is completely freed and the real root filesystem becomes `/`.

---

## Summary

- initramfs solves the chicken-and-egg problem: drivers needed to mount root are in the initramfs
- initramfs is a cpio archive extracted into tmpfs; initrd (legacy) was a block device image
- The /init script loads modules, finds root device, handles LVM/encryption, mounts real root
- `switch_root` transitions from initramfs to real root, freeing initramfs memory
- initramfs is optional when all root-access drivers are built into the kernel
- Built with `update-initramfs` (Debian), `dracut` (Fedora), or manually via cpio

---

*Next: [Chapter 16 — Kernel Initialization Process](Chapter_16_Kernel_Initialization.md)*
