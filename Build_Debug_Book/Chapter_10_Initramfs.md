# Chapter 10: Initramfs and Root Filesystem

## Learning Goals
- Understand initrd vs initramfs differences
- Learn how initramfs is built and embedded
- Master the transition from initramfs to real root FS
- Know dracut, mkinitramfs, and custom initramfs creation

---

## 1. initrd vs initramfs

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Problem: kernel needs root filesystem to run /sbin/init│
  │  But root FS may need drivers that aren't built-in      │
  │  (SCSI, NVMe, LVM, LUKS, network boot)                 │
  │  Solution: early userspace in RAM                       │
  │                                                           │
  │  initrd (legacy, deprecated):                            │
  │  ┌──────────────────────────────────────────┐            │
  │  │ - Disk image (ext2/ext4 filesystem)      │            │
  │  │ - Loaded as block device (/dev/ram0)     │            │
  │  │ - Mounted as real filesystem             │            │
  │  │ - Has filesystem overhead                │            │
  │  │ - Fixed size (wastes memory)             │            │
  │  │ - Double caching (in page cache +        │            │
  │  │   block device cache)                    │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  initramfs (modern, since 2.6):                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ - cpio archive (optionally compressed)   │            │
  │  │ - Unpacked directly into tmpfs (ramfs)   │            │
  │  │ - No filesystem driver needed            │            │
  │  │ - Grows/shrinks dynamically              │            │
  │  │ - Single cache (page cache only)         │            │
  │  │ - Can be embedded in vmlinux or separate │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Two ways to provide initramfs:                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. Embedded in vmlinux:                  │            │
  │  │    CONFIG_INITRAMFS_SOURCE="rootfs.cpio" │            │
  │  │    Built into kernel image itself        │            │
  │  │    Used by embedded systems              │            │
  │  │                                          │            │
  │  │ 2. Separate file:                        │            │
  │  │    Bootloader loads kernel + initramfs   │            │
  │  │    separately                            │            │
  │  │    GRUB: initrd /boot/initramfs.img      │            │
  │  │    U-Boot: initrd=0x48000000             │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. initramfs Contents and Boot Flow

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Typical initramfs contents:                             │
  │  ┌──────────────────────────────────────────┐            │
  │  │ /                                        │            │
  │  │ ├── /bin        → busybox, mount, switch_root│       │
  │  │ ├── /sbin       → modprobe                   │       │
  │  │ ├── /lib        → modules for storage/net   │        │
  │  │ │   └── modules/$(uname -r)/               │         │
  │  │ │       ├── nvme.ko                        │         │
  │  │ │       ├── ext4.ko                        │         │
  │  │ │       ├── dm-crypt.ko                    │         │
  │  │ │       └── modules.dep                    │         │
  │  │ ├── /etc        → fstab, udev rules        │        │
  │  │ ├── /dev        → minimal device nodes      │        │
  │  │ ├── /proc       → mount point              │         │
  │  │ ├── /sys        → mount point              │         │
  │  │ └── /init       → init script (PID 1)      │        │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Boot flow:                                              │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. Kernel decompresses and starts         │            │
  │  │ 2. Kernel unpacks initramfs to rootfs     │            │
  │  │    (tmpfs at /)                           │            │
  │  │ 3. Kernel executes /init (PID 1)          │            │
  │  │                                          │            │
  │  │ 4. /init script:                          │            │
  │  │    mount -t proc proc /proc               │            │
  │  │    mount -t sysfs sysfs /sys              │            │
  │  │    mount -t devtmpfs devtmpfs /dev        │            │
  │  │                                          │            │
  │  │ 5. Load necessary modules:               │            │
  │  │    modprobe nvme                          │            │
  │  │    modprobe dm_crypt                      │            │
  │  │                                          │            │
  │  │ 6. Discover and mount real root:          │            │
  │  │    cryptsetup luksOpen /dev/nvme0n1p2 root│            │
  │  │    mount /dev/mapper/root /mnt/root       │            │
  │  │                                          │            │
  │  │ 7. Transfer to real root:                 │            │
  │  │    exec switch_root /mnt/root /sbin/init  │            │
  │  │    # Deletes initramfs, pivots to real FS │            │
  │  │    # Execs real init (systemd/SysVinit)   │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Building Custom initramfs

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Manual creation:                                        │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Create directory structure:            │            │
  │  │ mkdir -p initramfs/{bin,dev,etc,lib,\    │            │
  │  │   lib/modules,proc,sys,mnt/root}         │            │
  │  │                                          │            │
  │  │ # Copy busybox (statically linked):      │            │
  │  │ cp busybox initramfs/bin/                 │            │
  │  │ cd initramfs/bin && ln -s busybox sh     │            │
  │  │                                          │            │
  │  │ # Create init script:                    │            │
  │  │ cat > initramfs/init << 'EOF'            │            │
  │  │ #!/bin/sh                                │            │
  │  │ mount -t proc proc /proc                 │            │
  │  │ mount -t sysfs sysfs /sys                │            │
  │  │ mount -t devtmpfs devtmpfs /dev          │            │
  │  │ echo "initramfs running"                 │            │
  │  │ mount /dev/sda1 /mnt/root                │            │
  │  │ exec switch_root /mnt/root /sbin/init    │            │
  │  │ EOF                                      │            │
  │  │ chmod +x initramfs/init                  │            │
  │  │                                          │            │
  │  │ # Create cpio archive:                   │            │
  │  │ cd initramfs                              │            │
  │  │ find . | cpio -o -H newc | gzip > \      │            │
  │  │   ../initramfs.img                       │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Distribution tools:                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ dracut (Fedora, RHEL, SUSE):             │            │
  │  │   dracut /boot/initramfs.img $(uname -r) │            │
  │  │   dracut --add-drivers "nvme e1000"      │            │
  │  │   dracut --force --kver 6.8.0            │            │
  │  │                                          │            │
  │  │ mkinitramfs (Debian, Ubuntu):             │            │
  │  │   mkinitramfs -o /boot/initrd.img \      │            │
  │  │     $(uname -r)                          │            │
  │  │   update-initramfs -u  # update current  │            │
  │  │                                          │            │
  │  │ mkinitcpio (Arch Linux):                  │            │
  │  │   mkinitcpio -p linux                    │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Explain the difference between initrd and initramfs and why initramfs is now preferred.**
**A:** `initrd` (initial ramdisk) is a block device image containing an ext2/ext4 filesystem. The kernel creates a ram disk (`/dev/ram0`), loads the image into it, and mounts it as a regular filesystem. Problems: (1) requires a filesystem driver in the kernel (can't use initrd to load the filesystem driver it needs). (2) Double caching: data is cached in both the block device layer and the page cache. (3) Fixed size: the ram disk has a pre-configured maximum size, wasting memory if unused or failing if too small. `initramfs` is a cpio archive (optionally compressed) that the kernel unpacks directly into a tmpfs (ram-based filesystem). Advantages: (1) tmpfs is always built into the kernel — no filesystem driver dependency. (2) Single-layer caching in the page cache. (3) Dynamic sizing — grows and shrinks as needed. (4) Can be embedded directly into the vmlinux image (`CONFIG_INITRAMFS_SOURCE`). (5) Simpler format: cpio is a flat archive vs. a block device image. The kernel detects the format automatically: if the loaded image starts with a cpio magic number, it's treated as initramfs; otherwise it's treated as legacy initrd. Modern distributions always use initramfs, though the GRUB directive is still called "initrd" for historical reasons.

**Q2: What happens during switch_root and why can't you just chroot to the real root?**
**A:** `switch_root` performs an atomic transition from the initramfs to the real root filesystem. It does things chroot cannot: (1) **Deletes initramfs contents**: recursively removes all files and directories from the current rootfs (the tmpfs). This reclaims the memory used by the initramfs. chroot would leave the initramfs consuming memory forever. (2) **Pivots the mount**: uses `pivot_root()` syscall (or equivalent) to change the root mount to the new filesystem. The old root becomes inaccessible and can be unmounted. (3) **Executes the real init**: `exec`s `/sbin/init` (systemd) on the new root, replacing the initramfs's PID 1 process. Since it uses `exec`, the PID remains 1 — the kernel requires PID 1 to exist. With chroot, the old root would still be accessible (by escaping chroot) and the initramfs memory wouldn't be freed. Additionally, mount points set up in initramfs (like /proc, /sys) need to be either moved to the new root (`mount --move /proc /mnt/root/proc`) or remounted, which switch_root handles. The sequence is: mount real rootfs → move /proc, /sys, /dev → switch_root → exec /sbin/init → systemd takes over as PID 1.

---

## Summary

- initrd (legacy): block device image, fixed size, double caching — deprecated
- initramfs: cpio archive → unpacked to tmpfs, dynamic size, single cache
- Purpose: load modules needed to mount real root FS (storage, crypto, network)
- /init runs as PID 1: mounts /proc /sys /dev, loads modules, finds real root
- switch_root: deletes initramfs, pivots to real root, execs /sbin/init
- Tools: dracut (Fedora), mkinitramfs (Debian), mkinitcpio (Arch)
- Can embed in vmlinux (CONFIG_INITRAMFS_SOURCE) for self-contained boot

---

[Previous: Kernel Image Formats ←](Chapter_09_Image_Formats.md) | [Next: printk Architecture →](Chapter_11_printk.md)
