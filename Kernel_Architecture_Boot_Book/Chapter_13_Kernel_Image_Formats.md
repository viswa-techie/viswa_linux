# Chapter 13: Kernel Image Formats

## Learning Goals
- Understand the different Linux kernel image formats
- Know the structure of vmlinux, vmlinuz, zImage, bzImage
- Understand kernel compression and decompression
- Grasp FIT images and Image format (ARM64)

---

## 13.1 Kernel Image Structure

The kernel build process produces several image formats from the source:

```
Kernel Build Outputs:

Source Code (C, assembly)
       │
       │  make
       ▼
┌──────────────┐
│   vmlinux    │  ← Raw ELF binary (uncompressed, with symbols)
│  (~100+ MB)  │     Not bootable directly (ELF format)
└──────┬───────┘
       │ objcopy (strip, convert to raw binary)
       ▼
┌──────────────┐
│  Image       │  ← Raw binary (ARM64) — directly bootable
│  (~20-30 MB) │     No ELF header, no compression
└──────┬───────┘
       │ Compress (gzip, lz4, lzma, zstd, xz)
       ▼
┌──────────────────┐
│ zImage / bzImage │  ← Compressed kernel + decompressor stub
│  (~5-10 MB)      │     Self-extracting — decompresses to Image
└──────────────────┘
       │ (optional) wrap in U-Boot header
       ▼
┌──────────────────┐
│  uImage          │  ← U-Boot legacy image format
│                  │     Header + compressed kernel
└──────────────────┘
       │ (optional) bundle into FIT
       ▼
┌──────────────────┐
│  FIT image       │  ← kernel + DTB + initramfs + signatures
│  (image.itb)     │     Modern U-Boot format
└──────────────────┘
```

---

## 13.2 vmlinuz Format

```
vmlinux vs vmlinuz:

vmlinux:
  ├── ELF format (Executable and Linkable Format)
  ├── Contains all kernel code + data + symbols
  ├── Uncompressed (~100+ MB with debug symbols)
  ├── Used for debugging (GDB, crash analysis, addr2line)
  ├── NOT directly bootable (ELF header confuses bootloaders)
  └── Has debug info: DWARF sections, symbol table

vmlinuz (note the 'z' for compressed):
  ├── Compressed, bootable kernel image
  ├── Contains: decompressor stub + compressed vmlinux
  ├── Size: ~5-10 MB (varies by compression)
  ├── What GRUB and most bootloaders actually load
  └── Self-extracting: decompresses itself into RAM

File locations:
  /boot/vmlinuz-6.8.0-40-generic    ← Compressed, bootable
  (vmlinux with debug: only available if you build kernel yourself
   or install linux-image-*-dbg package)
```

```bash
# Examine vmlinux (ELF)
file vmlinux
# vmlinux: ELF 64-bit LSB executable, ARM aarch64, statically linked

readelf -h vmlinux | head -20
# Entry point address: 0xFFFF800010000000

nm vmlinux | grep start_kernel
# ffff800010e60ba0 T start_kernel

# Extract vmlinux from vmlinuz
scripts/extract-vmlinux /boot/vmlinuz-$(uname -r) > vmlinux

# Size comparison
ls -lh vmlinux vmlinuz
#  -rwxr-xr-x  vmlinux   106M    (uncompressed ELF)
#  -rw-r--r--  vmlinuz   8.2M    (compressed)
```

---

## 13.3 zImage and bzImage

```
zImage Structure (ARM32):

┌──────────────────────────────────────────────┐
│  Self-extracting header                      │
│  ┌──────────────────────────────────────┐    │
│  │  Decompressor stub (assembly + C)    │    │
│  │  - Sets up minimal environment       │    │
│  │  - Decompresses payload              │    │
│  │  - Jumps to decompressed kernel      │    │
│  └──────────────────────────────────────┘    │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │  Compressed kernel payload           │    │
│  │  (gzip, lz4, lzma, xz, zstd)        │    │
│  └──────────────────────────────────────┘    │
└──────────────────────────────────────────────┘

Boot flow:
  1. Bootloader loads zImage to memory
  2. Jumps to zImage entry point
  3. Decompressor stub runs:
     a. Determines safe decompression location
     b. Decompresses kernel to final location
     c. Relocates if necessary
  4. Jumps to decompressed kernel start

bzImage (x86):
  bz = "big zImage" — NOT bzip2 compressed!
  ├── Can be loaded above 1 MB (zImage was below 512 KB)
  ├── Standard format for x86 Linux
  ├── Contains: setup header + compressed kernel
  └── Uses Linux boot protocol (header at offset 0x1F1)


ARM64 — No zImage, Uses "Image":
  ├── Uncompressed raw binary (no self-extractor)
  ├── Bootloader loads and jumps to it directly
  ├── Newer kernels support self-decompressing Image.gz
  └── booti command in U-Boot handles it
```

---

## 13.4 Kernel Compression Mechanisms

| Compression | Config | Ratio | Decompression Speed | Boot Time Impact | Use Case |
|------------|--------|-------|--------------------|--------------------|----------|
| **gzip** | `CONFIG_KERNEL_GZIP` | Good | Moderate | Balanced | Default, most common |
| **lz4** | `CONFIG_KERNEL_LZ4` | Lower | Very fast | Fastest boot | Embedded, fast boot |
| **lzma** | `CONFIG_KERNEL_LZMA` | Very high | Slow | Slower boot | Size-constrained |
| **xz** | `CONFIG_KERNEL_XZ` | Highest | Very slow | Slowest boot | Minimal storage |
| **zstd** | `CONFIG_KERNEL_ZSTD` | Very good | Fast | Good balance | Modern default |
| **lzo** | `CONFIG_KERNEL_LZO` | Moderate | Fast | Fast boot | Embedded |
| **none** | `CONFIG_KERNEL_UNCOMPRESSED` | 1:1 | Instant | No decompression | Fast boot, large storage |

```
Compression Comparison (typical ARM64 kernel):

Format        Compressed    Decompress Time    Total Boot Impact
────────────────────────────────────────────────────────────────
Uncompressed  30 MB         0 ms               Load: 300 ms
lz4           12 MB         15 ms              Load: 120 ms + 15 ms = 135 ms ✓ fastest
gzip          8 MB          45 ms              Load: 80 ms + 45 ms = 125 ms
zstd          7 MB          30 ms              Load: 70 ms + 30 ms = 100 ms ✓ best balance
xz            5 MB          200 ms             Load: 50 ms + 200 ms = 250 ms
lzma          5 MB          250 ms             Load: 50 ms + 250 ms = 300 ms

Note: Actual times depend on storage speed and CPU.
For fast storage (UFS/NVMe): lower compression wins.
For slow storage (NAND/SD): higher compression may win.
```

### FIT Images (Flattened Image Tree)

```
FIT Image Structure:

/dts-v1/;

/ {
    description = "Kernel FIT Image";
    #address-cells = <1>;

    images {
        kernel {
            description = "Linux kernel";
            data = /incbin/("Image.gz");
            type = "kernel";
            arch = "arm64";
            os = "linux";
            compression = "gzip";
            load = <0x48000000>;
            entry = <0x48000000>;
            hash-1 { algo = "sha256"; };
        };
        fdt-board-a {
            description = "Board A device tree";
            data = /incbin/("board-a.dtb");
            type = "flat_dt";
            arch = "arm64";
            compression = "none";
            hash-1 { algo = "sha256"; };
        };
        ramdisk {
            description = "initramfs";
            data = /incbin/("initramfs.cpio.gz");
            type = "ramdisk";
            arch = "arm64";
            compression = "none";
            hash-1 { algo = "sha256"; };
        };
    };

    configurations {
        default = "board-a";
        board-a {
            description = "Board A configuration";
            kernel = "kernel";
            fdt = "fdt-board-a";
            ramdisk = "ramdisk";
            signature { algo = "sha256,rsa2048"; key-name-hint = "dev"; };
        };
    };
};
```

```bash
# Create FIT image
mkimage -f fit.its image.itb

# Verify FIT image
mkimage -l image.itb

# Boot FIT image in U-Boot
=> fatload mmc 0:1 0x48000000 image.itb
=> bootm 0x48000000
```

---

## Interview Questions

**Q1: What is the difference between vmlinux and vmlinuz?**
A: `vmlinux` is the raw ELF executable with symbols — used for debugging (GDB, addr2line) but not bootable directly. `vmlinuz` is the compressed, bootable kernel — contains a decompressor stub + compressed vmlinux payload. Bootloaders load vmlinuz; developers use vmlinux for crash analysis.

**Q2: Why does ARM64 use "Image" instead of zImage?**
A: ARM64 uses a flat binary `Image` file because the architecture's boot protocol is simpler — the bootloader can load and jump directly. ARM32's zImage needed self-extraction due to memory constraints. ARM64 has ample memory, so the bootloader or kernel can handle decompression via `Image.gz` without a complex self-extractor.

**Q3: When should you choose lz4 vs gzip for kernel compression?**
A: Choose lz4 when boot speed is critical (automotive, Android) — it decompresses 3-4x faster than gzip at the cost of ~50% larger images. Choose gzip for a balance of size and speed. For storage-constrained devices (small SPI flash), use xz/lzma. For modern systems, zstd offers the best balance of compression ratio and decompression speed.

---

## Summary

- vmlinux is the raw ELF binary with debug symbols; vmlinuz is the compressed bootable image
- zImage/bzImage contain a decompressor stub + compressed kernel payload
- ARM64 uses a flat `Image` binary; ARM32 uses zImage; x86 uses bzImage
- Compression choice (lz4/gzip/zstd/xz) trades image size vs decompression speed
- FIT images bundle kernel + DTB + initramfs + signatures for verified boot
- Build outputs: vmlinux → Image (objcopy) → Image.gz (compress) → FIT (bundle)

---

*Next: [Chapter 14 — Kernel Boot Parameters](Chapter_14_Kernel_Boot_Parameters.md)*
