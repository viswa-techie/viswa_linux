# Chapter 27: Complete Boot Flow

## Learning Goals
- Understand the end-to-end boot flow from power-on to login prompt
- Know the x86 and ARM64 boot flow differences
- Grasp timing and dependencies across boot stages
- Understand Android-specific boot flow

---

## 27.1 x86 Boot Flow — Complete

```
x86 Boot: Power On → Login

┌─────────────────────────────────────────────────────────┐
│ PHASE 1: FIRMWARE (~0-3 seconds)                        │
│                                                         │
│ CPU Reset Vector (0xFFFFFFF0)                           │
│    │                                                    │
│    ▼                                                    │
│ UEFI Firmware                                           │
│    ├── SEC: Security phase (cache-as-RAM)               │
│    ├── PEI: Pre-EFI Init (memory init, CPU init)        │
│    ├── DXE: Driver Execution (PCIe, USB, storage)       │
│    ├── BDS: Boot Device Selection                       │
│    │   └── Read EFI System Partition (ESP)              │
│    └── Execute bootloader: \EFI\BOOT\BOOTX64.EFI       │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│ PHASE 2: BOOTLOADER (~1-3 seconds)                      │
│                                                         │
│ GRUB2                                                   │
│    ├── core.img loaded by UEFI                          │
│    ├── Load grub.cfg from /boot/grub/                   │
│    ├── Display menu (optional, timeout)                 │
│    ├── Load vmlinuz-5.15 to memory                     │
│    ├── Load initramfs-5.15.img to memory               │
│    ├── Set kernel command line                          │
│    │   root=UUID=xxx ro quiet splash                    │
│    └── Jump to kernel entry point                      │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│ PHASE 3: KERNEL EARLY BOOT (~1-5 seconds)               │
│                                                         │
│ Kernel Entry (arch/x86/boot/header.S)                   │
│    ├── Real mode setup (if BIOS boot)                   │
│    ├── Switch to protected mode (32-bit)                │
│    ├── Switch to long mode (64-bit)                     │
│    ├── Decompress kernel (if bzImage)                   │
│    ├── startup_64:                                      │
│    │   ├── Identity-map kernel                          │
│    │   ├── Enable paging                                │
│    │   └── Jump to start_kernel()                       │
│    │                                                    │
│    └── start_kernel():                                  │
│        ├── setup_arch()         (ACPI, memory, PCI)     │
│        ├── mm_core_init()       (page allocator, slab)  │
│        ├── sched_init()         (scheduler)             │
│        ├── init_IRQ()           (APIC setup)            │
│        ├── time_init()          (TSC, HPET)             │
│        ├── console_init()       (early console)         │
│        └── rest_init()                                  │
│            ├── kernel_thread(kernel_init)  → PID 1      │
│            ├── kernel_thread(kthreadd)     → PID 2      │
│            └── cpu_startup_entry() → idle               │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│ PHASE 4: KERNEL INIT + INITRAMFS (~2-10 seconds)        │
│                                                         │
│ kernel_init() (PID 1):                                  │
│    ├── do_basic_setup()                                 │
│    │   ├── driver_init()       (driver model)           │
│    │   └── do_initcalls()      (all module_init)        │
│    │       ├── PCIe enumeration                         │
│    │       ├── USB init                                 │
│    │       ├── Storage driver (NVMe/AHCI)               │
│    │       └── Filesystem registration                  │
│    ├── smp_init()              (secondary CPUs)         │
│    ├── populate_rootfs()       (unpack initramfs)       │
│    └── exec /init              (user space begins)      │
│                                                         │
│ initramfs /init (systemd-based):                        │
│    ├── Load storage drivers (if modular)                │
│    ├── Assemble root device (LVM/RAID/dm-crypt)         │
│    ├── Mount real root → /sysroot                       │
│    └── switch_root /sysroot /usr/lib/systemd/systemd    │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│ PHASE 5: USER SPACE (~3-30 seconds)                     │
│                                                         │
│ systemd (PID 1):                                        │
│    ├── basic.target                                     │
│    │   ├── Mount filesystems (/etc/fstab)               │
│    │   ├── udev device enumeration                      │
│    │   └── Set hostname, timezone                       │
│    ├── multi-user.target                                │
│    │   ├── NetworkManager.service                       │
│    │   ├── sshd.service                                 │
│    │   └── other services...                            │
│    └── graphical.target                                 │
│        ├── Display manager (gdm/lightdm)                │
│        └── Login prompt                                 │
│                                                         │
│ Total: ~10-60 seconds (hardware dependent)              │
└─────────────────────────────────────────────────────────┘
```

---

## 27.2 ARM64 Boot Flow — Complete

```
ARM64 Boot: Power On → Login

┌─────────────────────────────────────────────────────────┐
│ PHASE 1: FIRMWARE                                       │
│                                                         │
│ SoC ROM Code (fixed in silicon)                         │
│    ├── Initialize minimal hardware (SRAM, clocks)       │
│    ├── Load SPL/BL1 from boot media                    │
│    └── Jump to BL1                                      │
│                                                         │
│ ARM Trusted Firmware (ATF/TF-A):                        │
│    ├── BL1: AP trusted ROM (EL3)                        │
│    │   └── Initialize secure world, load BL2            │
│    ├── BL2: Trusted boot firmware (EL1-S)              │
│    │   └── DRAM init, load BL31/BL32/BL33              │
│    ├── BL31: EL3 runtime firmware (PSCI, SCMI)         │
│    ├── BL32: Secure OS (OP-TEE) [optional]             │
│    └── BL33: Non-secure bootloader (U-Boot/UEFI)       │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│ PHASE 2: BOOTLOADER (U-Boot)                            │
│                                                         │
│ U-Boot:                                                 │
│    ├── Board-specific init (PMIC, DDR training)         │
│    ├── Load device tree blob (.dtb) to RAM              │
│    ├── Load kernel Image to RAM                         │
│    ├── Load initramfs/ramdisk to RAM                    │
│    ├── Set bootargs in /chosen DT node                  │
│    └── booti <kernel_addr> <initrd_addr> <dtb_addr>     │
│                                                         │
│ CPU state at kernel entry:                              │
│    - MMU off, D-cache off, I-cache may be on            │
│    - x0 = physical address of DTB                       │
│    - CPU in EL2 (hypervisor) or EL1                     │
│    - All secondary CPUs held by ATF (PSCI)              │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│ PHASE 3: KERNEL (arch/arm64/kernel/head.S)              │
│                                                         │
│    ├── preserve_boot_args     (save x0=DTB address)     │
│    ├── el2_setup              (drop to EL1 if needed)   │
│    ├── __create_page_tables   (identity + kernel map)   │
│    ├── __primary_switch                                  │
│    │   ├── __enable_mmu                                 │
│    │   └── __primary_switched                           │
│    │       └── start_kernel()                           │
│    │                                                    │
│    └── start_kernel() → same flow as x86 from here      │
│        ├── setup_arch()        (parse DTB, setup mem)   │
│        ├── unflatten_device_tree()                      │
│        └── ... (rest identical to x86 flow)             │
└─────────────────────────────────────────────────────────┘
```

---

## 27.3 Android Boot Flow — Complete

```
Android Boot: Power On → Home Screen

┌────────────────────────────────────────────────────┐
│  1. SoC Boot ROM                                   │
│     └── Load primary bootloader (PBL)              │
├────────────────────────────────────────────────────┤
│  2. Primary Bootloader (PBL)                       │
│     ├── DRAM init                                  │
│     └── Load secondary bootloader                  │
├────────────────────────────────────────────────────┤
│  3. Secondary Bootloader (ABL/LK/U-Boot)           │
│     ├── Verified Boot (AVB / dm-verity)            │
│     ├── Load boot.img (kernel + ramdisk + DTB)     │
│     ├── Load vendor_boot.img                       │
│     ├── Load dtbo.img (DT overlays)                │
│     └── Jump to kernel                             │
├────────────────────────────────────────────────────┤
│  4. Linux Kernel                                   │
│     ├── Standard kernel init                       │
│     ├── Mount initramfs from boot.img ramdisk      │
│     └── exec /init (Android init)                  │
├────────────────────────────────────────────────────┤
│  5. Android first_stage_init                       │
│     ├── Mount /dev, /proc, /sys, /dev/pts          │
│     ├── Load SELinux policy                        │
│     ├── Mount partitions (super → system, vendor)  │
│     └── Re-exec /init for second stage             │
├────────────────────────────────────────────────────┤
│  6. Android second_stage_init                      │
│     ├── Property service → /dev/properties         │
│     ├── Parse init.rc files                        │
│     ├── Start essential services:                  │
│     │   ├── ueventd (device nodes)                 │
│     │   ├── logd (logging)                         │
│     │   ├── servicemanager (binder)                │
│     │   ├── hwservicemanager (HIDL)                │
│     │   ├── vold (volumes)                         │
│     │   └── surfaceflinger (display)               │
│     └── Start zygote                               │
├────────────────────────────────────────────────────┤
│  7. Zygote                                         │
│     ├── Preload Java classes and resources         │
│     ├── Fork system_server                         │
│     └── Listen for app launch requests             │
├────────────────────────────────────────────────────┤
│  8. System Server                                  │
│     ├── ActivityManagerService                     │
│     ├── WindowManagerService                       │
│     ├── PackageManagerService                      │
│     ├── PowerManagerService                        │
│     └── ... (100+ system services)                 │
├────────────────────────────────────────────────────┤
│  9. Launcher / Home Screen                         │
│     └── Boot animation ends → Home screen visible  │
└────────────────────────────────────────────────────┘
```

---

## 27.4 Boot Timing Analysis

```bash
# Analyze boot timing on systemd systems
$ systemd-analyze
Startup finished in 2.5s (firmware) + 1.8s (loader) + 
                    3.2s (kernel) + 8.1s (userspace) = 15.6s

$ systemd-analyze blame
7.2s NetworkManager-wait-online.service
2.1s plymouth-quit-wait.service
1.5s firewalld.service
0.8s systemd-udev-settle.service

$ systemd-analyze critical-chain
graphical.target @15.6s
└── multi-user.target @12.3s
    └── NetworkManager.service @4.1s +1.2s
        └── network-pre.target @4.0s
            └── firewalld.service @2.8s +1.2s
                └── basic.target @2.7s
                    └── sockets.target @2.7s
                        └── dbus.socket @2.7s

# Kernel boot timing
$ dmesg | head -5
[    0.000000] Linux version 6.1.0 ...
[    0.000000] Command line: root=UUID=... ro quiet
[    0.043851] ACPI: Early table checksum verification disabled
[    0.095234] Memory: 16268912K/16777216K available
[    0.123456] smpboot: Allowing 8 CPUs

# Android boot timing
$ adb shell cat /proc/bootprof     # MediaTek
$ adb shell dmesg | grep -i boot
```

---

## 27.5 Boot Stage Comparison

```
Stage Comparison: x86 vs ARM64 vs Android

Stage          │ x86              │ ARM64           │ Android
───────────────┼──────────────────┼─────────────────┼─────────────
Firmware       │ UEFI (SEC→DXE)   │ ATF (BL1→BL31)  │ SoC ROM+PBL
Bootloader     │ GRUB2            │ U-Boot          │ ABL/LK
Kernel entry   │ startup_64       │ primary_entry   │ Same as ARM64
Device info    │ ACPI tables      │ Device Tree     │ DT + overlays
CPU startup    │ ACPI+MP table    │ PSCI (ATF)      │ PSCI
Root mount     │ initramfs→disk   │ initramfs→disk  │ initramfs→super
Init process   │ systemd          │ systemd         │ Android init
Verified boot  │ UEFI Secure Boot │ ATF+BL33 verify │ AVB (dm-verity)
```

---

## Interview Questions

**Q1: Walk through the complete boot process of an ARM64 Linux system.**
A: (1) SoC ROM loads BL1 from boot media. (2) ARM Trusted Firmware BL1 → BL2 → BL31 initializes secure world, DRAM, and PSCI. (3) BL33 (U-Boot) loads kernel Image, DTB, and initramfs to RAM. (4) Kernel head.S enables MMU and calls start_kernel(). (5) start_kernel() initializes subsystems: memory, scheduler, interrupts, drivers. (6) rest_init() creates PID 1 (kernel_init) and PID 2 (kthreadd). (7) kernel_init runs initcalls, brings up SMP, unpacks initramfs, and execs /init. (8) User-space init (systemd) starts services until reaching the default target.

**Q2: What is the critical path in boot and how do you optimize it?**
A: The critical path is the longest chain of sequential dependencies from power-on to the target state. To optimize: (1) Measure with `systemd-analyze critical-chain`. (2) Reduce firmware time with fast-boot options. (3) Minimize initramfs (only essential modules). (4) Parallelize services (socket activation). (5) Defer non-critical services. (6) Use kernel command line `quiet` to reduce console output. (7) Compile essential drivers into kernel (skip module loading).

---

## Summary

- x86 boot: UEFI → GRUB → bzImage → startup_64 → start_kernel → systemd
- ARM64 boot: ROM → ATF(BL1-BL31) → U-Boot → head.S → start_kernel → systemd/init
- Android boot: ROM → PBL → ABL → kernel → first_stage_init → zygote → home screen
- All architectures converge at `start_kernel()` — post-init flow is largely identical
- Boot optimization requires measuring and eliminating critical path bottlenecks

---

*Next: [Chapter 28 — Kernel Logging](Chapter_28_Kernel_Logging.md)*
