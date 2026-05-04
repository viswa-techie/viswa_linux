# Chapter 34: Embedded Systems Boot

## Learning Goals
- Understand embedded boot flow differences from desktop/server
- Know boot media options (NAND, NOR, eMMC, SD, SPI)
- Grasp secure boot and verified boot for embedded
- Understand custom bootloader configurations

---

## 34.1 Embedded Boot Flow

```
Embedded Boot Flow (Typical ARM SoC):

┌──────────────────────────────────────────────────────┐
│ 1. ROM Bootloader (in silicon, immutable)            │
│    ├── Initialize minimal clocks and SRAM            │
│    ├── Detect boot media from pins/fusees:           │
│    │   ├── eMMC (JEDEC standard flash)               │
│    │   ├── SD card                                   │
│    │   ├── SPI NOR flash                             │
│    │   ├── NAND flash                                │
│    │   ├── USB (recovery mode)                       │
│    │   └── UART (factory programming)                │
│    ├── Load SPL/MLO to internal SRAM (tiny: 64-256KB)│
│    └── Jump to SPL                                    │
├──────────────────────────────────────────────────────┤
│ 2. SPL (Secondary Program Loader)                     │
│    ├── Initialize DDR/LPDDR memory controller         │
│    ├── Configure PLLs and advanced clocks             │
│    ├── Load full bootloader (U-Boot) to DRAM          │
│    └── Jump to U-Boot                                 │
├──────────────────────────────────────────────────────┤
│ 3. U-Boot (Full Bootloader)                           │
│    ├── Initialize periherals (Ethernet, USB, LCD)    │
│    ├── Load kernel + DTB + rootfs from:               │
│    │   ├── eMMC partition                             │
│    │   ├── NAND volume (UBI/UBIFS)                    │
│    │   ├── NFS (network boot)                         │
│    │   ├── TFTP (development)                         │
│    │   └── USB mass storage                           │
│    └── Boot kernel with bootargs                      │
├──────────────────────────────────────────────────────┤
│ 4. Linux Kernel                                       │
│    ├── Standard boot (see previous chapters)          │
│    └── Mount root filesystem                          │
├──────────────────────────────────────────────────────┤
│ 5. Root Filesystem                                    │
│    ├── Buildroot / Yocto / Debian minimal             │
│    ├── BusyBox init or systemd                        │
│    └── Application starts                             │
└──────────────────────────────────────────────────────┘
```

---

## 34.2 Boot Media Comparison

```
Boot Storage Options:

┌────────────┬──────────┬──────────┬─────────┬───────────────┐
│ Media      │ Speed    │ Size     │ Cost    │ Use Case      │
├────────────┼──────────┼──────────┼─────────┼───────────────┤
│ SPI NOR    │ Slow     │ 1-32MB   │ Medium  │ Bootloader    │
│            │ (50MHz)  │          │         │ only          │
├────────────┼──────────┼──────────┼─────────┼───────────────┤
│ NAND       │ Medium   │ 128MB-4GB│ Low     │ Full system   │
│            │ (40MB/s) │          │         │ with UBIFS    │
├────────────┼──────────┼──────────┼─────────┼───────────────┤
│ eMMC       │ Fast     │ 4-128GB  │ Medium  │ Android/auto  │
│            │ (200MB/s)│          │         │ HS200/HS400   │
├────────────┼──────────┼──────────┼─────────┼───────────────┤
│ SD Card    │ Medium   │ 2-512GB  │ Low     │ Development   │
│            │ (100MB/s)│          │         │ Prototyping   │
├────────────┼──────────┼──────────┼─────────┼───────────────┤
│ NOR Flash  │ Very slow│ 1-64MB   │ High    │ Safety-crit   │
│ (parallel) │ (10MB/s) │          │         │ XIP capable   │
└────────────┴──────────┴──────────┴─────────┴───────────────┘

Partition Layout (eMMC example):

Boot partition 0:  SPL + U-Boot        (4MB)
Boot partition 1:  Backup SPL+U-Boot   (4MB)
User data area:
  ├── Partition 1: boot   (kernel+DTB)  (64MB)
  ├── Partition 2: rootfs (ext4/squashfs)(512MB)
  ├── Partition 3: data   (rw user data) (remaining)
  └── Partition 4: recovery             (128MB)
```

---

## 34.3 NAND Boot and UBI

```
NAND Flash Boot:

NAND challenges:
  ├── Bad blocks (manufacturing or wear-out)
  ├── Bit errors (require ECC: BCH or LDPC)
  ├── Write/erase in blocks (not bytes)
  ├── Limited write cycles (10K-100K per block)
  └── No XIP (execute-in-place) capability

Solution: UBI/UBIFS
┌─────────────────────────────────────────────┐
│  UBIFS (filesystem)                         │
│  ├── Flash-aware filesystem                 │
│  ├── Compression (zlib/lzo/zstd)            │
│  └── Journal-based, power-cut safe          │
├─────────────────────────────────────────────┤
│  UBI (Unsorted Block Images)                │
│  ├── Wear leveling (even out erase cycles)  │
│  ├── Bad block management                   │
│  ├── Volume management (logical partitions) │
│  └── Atomic updates                         │
├─────────────────────────────────────────────┤
│  MTD (Memory Technology Devices)            │
│  ├── Raw NAND driver                        │
│  ├── ECC engine                             │
│  └── Provides block-like interface          │
├─────────────────────────────────────────────┤
│  NAND Flash Hardware                        │
└─────────────────────────────────────────────┘

# Kernel command line for NAND root
ubi.mtd=3 root=ubi0:rootfs rootfstype=ubifs
```

---

## 34.4 Secure Boot for Embedded

```
Embedded Secure Boot Chain:

┌──────────────────────────────────────────────────┐
│  ROM Bootloader (root of trust)                  │
│  ├── Contains vendor public key (fused in OTP)   │
│  ├── Verifies SPL signature                      │
│  └── Immutable — cannot be updated               │
│                                                  │
│  If verification fails → boot halts              │
└──────────────┬───────────────────────────────────┘
               │ ✓ Verified
               ▼
┌──────────────────────────────────────────────────┐
│  SPL (signed with vendor private key)            │
│  ├── Contains next-stage public key              │
│  ├── Verifies U-Boot signature                   │
│  └── Uses HAB (NXP) / TrustZone (ARM)           │
│                                                  │
│  If verification fails → boot halts              │
└──────────────┬───────────────────────────────────┘
               │ ✓ Verified
               ▼
┌──────────────────────────────────────────────────┐
│  U-Boot FIT Image (signed)                       │
│  ├── Verifies kernel Image signature             │
│  ├── Verifies DTB signature                      │
│  ├── Verifies initramfs signature                │
│  └── Uses RSA-2048/4096 + SHA-256                │
└──────────────┬───────────────────────────────────┘
               │ ✓ Verified
               ▼
┌──────────────────────────────────────────────────┐
│  Kernel + dm-verity                              │
│  ├── Kernel verifies rootfs integrity            │
│  ├── dm-verity: hash tree for read-only rootfs   │
│  └── Any block corruption detected → I/O error   │
└──────────────────────────────────────────────────┘

Android Verified Boot (AVB):
  ├── vbmeta partition contains signed hash of:
  │   ├── boot.img (kernel + ramdisk)
  │   ├── system.img (Android system)
  │   └── vendor.img (vendor HAL)
  └── Hash chain verified from ROM → bootloader → kernel
```

---

## 34.5 U-Boot for Embedded

```
U-Boot Environment and Configuration:

# U-Boot commands for embedded boot
U-Boot> printenv
bootcmd=run mmcboot
mmcboot=mmc dev 0; load mmc 0:1 ${loadaddr} zImage; \
        load mmc 0:1 ${fdtaddr} board.dtb; \
        bootz ${loadaddr} - ${fdtaddr}
bootargs=console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait
loadaddr=0x42000000
fdtaddr=0x43000000

# Network boot (development)
U-Boot> setenv serverip 192.168.1.1
U-Boot> setenv ipaddr 192.168.1.100
U-Boot> tftp ${loadaddr} zImage
U-Boot> tftp ${fdtaddr} board.dtb
U-Boot> bootz ${loadaddr} - ${fdtaddr}

# FIT Image boot (production)
U-Boot> load mmc 0:1 ${loadaddr} image.itb
U-Boot> bootm ${loadaddr}

# Recovery/fallback
U-Boot> if mmc dev 0; then
            run mmcboot;
        else
            run nandboot;
        fi
```

---

## 34.6 Root Filesystem Options

```
Embedded Root Filesystem Choices:

┌──────────────┬──────────────────────┬───────────────────┐
│ Type         │ Characteristics      │ Use Case          │
├──────────────┼──────────────────────┼───────────────────┤
│ initramfs    │ Loaded to RAM        │ Tiny systems      │
│              │ Fast, read-only      │ Recovery          │
│              │ Size limited by RAM  │ Single purpose    │
├──────────────┼──────────────────────┼───────────────────┤
│ SquashFS     │ Read-only, compressed│ Production rootfs │
│              │ Very space efficient │ with overlay RW   │
│              │ LZO/ZSTD compress   │                   │
├──────────────┼──────────────────────┼───────────────────┤
│ ext4         │ Read/write, journaled│ General purpose   │
│              │ Mature, reliable     │ Most common       │
├──────────────┼──────────────────────┼───────────────────┤
│ UBIFS        │ Flash-aware, RW      │ Raw NAND root     │
│              │ Wear leveling        │                   │
├──────────────┼──────────────────────┼───────────────────┤
│ JFFS2        │ Flash-aware, legacy  │ Small NOR flash   │
│              │ Slow mount           │ Being replaced    │
├──────────────┼──────────────────────┼───────────────────┤
│ NFS root     │ Network mounted      │ Development only  │
│              │ Easy iteration       │ Never production  │
└──────────────┴──────────────────────┴───────────────────┘

Build Systems:
  Yocto/OpenEmbedded: Industry standard, highly customizable
  Buildroot: Simple, fast builds, good for small systems
  Debian/Ubuntu: debootstrap for full distro on embedded
  Android: AOSP build system for Android-based embedded
```

---

## 34.7 Automotive Boot (AAOS/QNX)

```
Automotive Boot Requirements:

┌─────────────────────────────────────────────────────┐
│  Requirement          │  Typical Target             │
├───────────────────────┼─────────────────────────────┤
│  Cold boot to rear    │  < 2 seconds                │
│  view camera          │                             │
├───────────────────────┼─────────────────────────────┤
│  Cold boot to HMI     │  < 5-10 seconds             │
├───────────────────────┼─────────────────────────────┤
│  Warm boot (resume)   │  < 1 second                 │
├───────────────────────┼─────────────────────────────┤
│  Secure boot          │  Mandatory (UNECE R156)     │
├───────────────────────┼─────────────────────────────┤
│  OTA update           │  A/B partitions, rollback   │
└───────────────────────┴─────────────────────────────┘

Automotive SoC Boot (e.g., Qualcomm SA8155P):
  ├── Primary Boot Loader (PBL) - ROM
  ├── eXtensible Boot Loader (XBL) - UEFI-based
  ├── ABL (Android Boot Loader) - fastboot
  ├── Kernel + Android (main domain)
  └── QNX/RTOS (safety domain, hypervisor guest)

Early camera display path:
  PBL → XBL → Early camera HAL → Display
  (bypasses full OS boot for safety-critical rear camera)
```

---

## Interview Questions

**Q1: How does embedded boot differ from desktop boot?**
A: Embedded systems: (1) Boot from flash (NAND/eMMC/NOR) instead of disk. (2) Use SoC ROM bootloader as first stage instead of BIOS/UEFI. (3) Need SPL for DRAM init because internal SRAM is too small for full bootloader. (4) Use device tree exclusively (no ACPI). (5) Have strict boot time requirements. (6) Often use read-only or flash-aware filesystems (SquashFS, UBIFS). (7) Secure boot chains are hardware-fused.

**Q2: Why is a two-stage bootloader (SPL + U-Boot) needed on embedded?**
A: The SoC ROM can only load a small amount of code into internal SRAM (typically 64-256KB). DRAM isn't available yet because its controller needs complex initialization (PHY training, calibration). SPL fits in SRAM, initializes DRAM, then loads the full U-Boot into DRAM. U-Boot is too large (500KB-2MB) to fit in SRAM.

**Q3: How does dm-verity work for embedded secure boot?**
A: dm-verity creates a hash tree of the read-only root filesystem. Each 4KB block has a SHA-256 hash stored in the hash tree. When a block is read, the kernel computes its hash and compares it to the tree. If they don't match, the block is corrupted → I/O error is returned. The root hash of the tree is signed and verified by the bootloader, completing the chain of trust from ROM to every filesystem block.

---

## Summary

- Embedded boot uses ROM → SPL → U-Boot → Kernel (more stages than desktop)
- Boot media varies: eMMC, NAND (with UBI), NOR, SD — each has different tradeoffs
- Secure boot chains from ROM (immutable root of trust) through each stage with signature verification
- NAND requires UBI for wear leveling and bad block management
- Automotive boot has strict timing requirements and uses specialized SoC boot paths
- Root filesystem choices: SquashFS (production RO), UBIFS (NAND), ext4 (general), initramfs (tiny)

---

*Next: [Chapter 35 — OS Comparison](Chapter_35_OS_Comparison.md)*
