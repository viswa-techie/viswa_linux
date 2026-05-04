# Chapter 9: Kernel Image Formats

## Learning Goals
- Understand vmlinux, bzImage, Image, and zImage
- Learn boot protocol and decompression
- Master uImage, FIT image for embedded
- Know how bootloaders hand off to the kernel

---

## 1. Image Format Overview

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Build produces vmlinux (ELF) → derivatives:            │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ vmlinux                                  │            │
  │  │   Raw ELF binary with debug symbols      │            │
  │  │   NOT bootable directly (too large,      │            │
  │  │   ELF headers, debug sections)           │            │
  │  │   Used by: crash tool, GDB, perf         │            │
  │  │                                          │            │
  │  │ vmlinux → objcopy → vmlinux.bin          │            │
  │  │   Stripped raw binary (no ELF headers)   │            │
  │  │                                          │            │
  │  │ vmlinux.bin → compress → vmlinux.bin.gz  │            │
  │  │   Compressed kernel image                │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Platform-specific bootable images:                      │
  │  ┌──────────────────┬─────────────────────────────────┐ │
  │  │ Format           │ Platform / Usage                 │ │
  │  ├──────────────────┼─────────────────────────────────┤ │
  │  │ bzImage           │ x86 (BIOS/UEFI)                │ │
  │  │                   │ arch/x86/boot/bzImage           │ │
  │  │                   │ Contains setup code +           │ │
  │  │                   │ compressed vmlinux.bin          │ │
  │  ├──────────────────┼─────────────────────────────────┤ │
  │  │ Image             │ ARM64 (AArch64)                │ │
  │  │                   │ arch/arm64/boot/Image           │ │
  │  │                   │ Uncompressed (bootloader may   │ │
  │  │                   │ compress: Image.gz)            │ │
  │  ├──────────────────┼─────────────────────────────────┤ │
  │  │ zImage            │ ARM 32-bit                     │ │
  │  │                   │ Self-extracting compressed     │ │
  │  │                   │ kernel                         │ │
  │  ├──────────────────┼─────────────────────────────────┤ │
  │  │ uImage            │ U-Boot (legacy)                │ │
  │  │                   │ mkimage header + zImage        │ │
  │  │                   │ Load/entry address in header   │ │
  │  ├──────────────────┼─────────────────────────────────┤ │
  │  │ FIT (itb)         │ U-Boot (modern)                │ │
  │  │                   │ Flattened Image Tree           │ │
  │  │                   │ Multiple kernels, DTBs, ramdisk│ │
  │  │                   │ Signatures for verified boot   │ │
  │  └──────────────────┴─────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. x86 bzImage Boot Flow

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  bzImage structure:                                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │ ┌──────────────────────────┐             │            │
  │  │ │ setup.bin (real-mode code)│             │            │
  │  │ │ - Boot sector             │             │            │
  │  │ │ - setup header            │             │            │
  │  │ │ - E820 memory map         │             │            │
  │  │ │ - Video mode setup        │             │            │
  │  │ │ - Switch to protected mode│             │            │
  │  │ ├──────────────────────────┤             │            │
  │  │ │ vmlinux.bin (compressed) │             │            │
  │  │ │ + decompressor stub      │             │            │
  │  │ │ (gzip, lz4, lzma, zstd) │             │            │
  │  │ └──────────────────────────┘             │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Boot sequence:                                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. BIOS/UEFI loads bzImage               │            │
  │  │ 2. setup.bin runs in real mode           │            │
  │  │    - Queries BIOS for memory map (E820)  │            │
  │  │    - Sets up video mode                  │            │
  │  │ 3. Switch to 32-bit protected mode       │            │
  │  │ 4. Decompressor runs                     │            │
  │  │    - Decompresses vmlinux.bin to memory  │            │
  │  │ 5. Jump to startup_64 (arch/x86/kernel/  │            │
  │  │    head_64.S)                            │            │
  │  │ 6. Switch to 64-bit long mode            │            │
  │  │ 7. Set up initial page tables            │            │
  │  │ 8. Call start_kernel() (init/main.c)     │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  UEFI boot (modern):                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ EFI stub built into bzImage              │            │
  │  │ CONFIG_EFI_STUB=y                        │            │
  │  │ UEFI firmware can load bzImage directly  │            │
  │  │ (no separate bootloader needed)          │            │
  │  │ Or: GRUB/systemd-boot as EFI application │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. ARM64 Image Boot Flow

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ARM64 boot:                                             │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. Bootloader (U-Boot/UEFI) loads:       │            │
  │  │    - Image (or Image.gz) to RAM          │            │
  │  │    - DTB to separate RAM location        │            │
  │  │    - Optional: initramfs                 │            │
  │  │                                          │            │
  │  │ 2. Bootloader sets up:                   │            │
  │  │    - MMU off                             │            │
  │  │    - x0 = DTB physical address           │            │
  │  │    - x1-x3 = reserved (0)                │            │
  │  │    - Jump to Image entry point           │            │
  │  │                                          │            │
  │  │ 3. Kernel entry (head.S):                │            │
  │  │    - Validate CPU state                  │            │
  │  │    - Create initial page tables          │            │
  │  │    - Enable MMU                          │            │
  │  │    - Primary CPU → start_kernel()        │            │
  │  │    - Secondary CPUs → spin/PSCI wait     │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Compression variants:                                   │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Image      — uncompressed (~30MB)        │            │
  │  │ Image.gz   — gzip compressed (~12MB)     │            │
  │  │ Image.lz4  — lz4 (fast decompress)       │            │
  │  │ Image.zst  — zstd (good ratio + speed)   │            │
  │  │                                          │            │
  │  │ Bootloader decompresses before jumping   │            │
  │  │ (kernel doesn't self-decompress on ARM64)│            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Trace the complete path from `make bzImage` to a bootable kernel file on x86.**
**A:** (1) All `.c` files are compiled to `.o` object files and archived into `built-in.a` per directory. (2) All archives are linked into `vmlinux` — a statically linked ELF binary containing all built-in code, symbols, and debug info. The linker script (`vmlinux.lds`) places sections at their correct virtual addresses. (3) `objcopy -O binary vmlinux vmlinux.bin` — strips ELF headers, debug sections, and produces a raw binary. (4) Compression: `vmlinux.bin` → `vmlinux.bin.gz` (or lz4, lzma, zstd depending on `CONFIG_KERNEL_GZIP`/etc.). (5) A decompressor stub (`arch/x86/boot/compressed/head_64.S` + `misc.c`) is compiled and linked with the compressed kernel into a combined blob. (6) `arch/x86/boot/setup.bin` is built — this is the real-mode setup code (boot sector, setup header, BIOS calls, protected mode switch). (7) `setup.bin` + compressed kernel blob are concatenated to form `bzImage` at `arch/x86/boot/bzImage`. (8) If `CONFIG_EFI_STUB=y`, an EFI PE header is added so UEFI firmware can load bzImage directly as an EFI application. The "b" in bzImage stands for "big" — it uses a different loading mechanism than the older `zImage` which was limited to 512KB.

**Q2: What is a FIT image and why is it preferred over uImage in modern embedded Linux?**
**A:** uImage (legacy): created by `mkimage`, a simple container with a 64-byte header containing load address, entry point, OS type, architecture, compression type, image name, and CRC32 checksum. It wraps a single kernel image. Limitations: only one component per image (need separate files for kernel, DTB, ramdisk), weak integrity checking (CRC32), no signature support. FIT (Flattened Image Tree): uses a DTS-like description to bundle multiple components in one image. A `.its` (Image Tree Source) file describes: multiple kernel images (different configs), multiple DTBs, ramdisk, and their relationships. `mkimage -f image.its image.itb` produces the FIT image. Advantages: (1) Single file contains kernel + DTB + ramdisk (simpler deployment). (2) Multiple configurations in one image (e.g., same kernel with different DTBs for board variants). (3) SHA-256 hashing and RSA signature verification — essential for verified/secure boot. (4) Selective loading: bootloader picks the right configuration at boot. (5) Extensible: can include custom data types. This is the standard for modern U-Boot deployments, especially where secure boot is required.

---

## Summary

- vmlinux: raw ELF (debugging); vmlinux.bin: stripped binary; bzImage: bootable (x86)
- bzImage: setup.bin (real-mode) + compressed kernel + decompressor
- ARM64 Image: uncompressed (bootloader handles compression); DTB passed separately
- uImage: legacy wrapper (header + kernel); FIT: modern bundle (kernel + DTB + ramdisk + signatures)
- Boot flow: bootloader → setup/entry code → switch modes → page tables → start_kernel()
- Compression options: gzip, lz4 (fast), lzma (small), zstd (balanced)

---

[Previous: Linker Scripts and vmlinux ←](Chapter_08_Linker_vmlinux.md) | [Next: Initramfs and Root FS →](Chapter_10_Initramfs.md)
