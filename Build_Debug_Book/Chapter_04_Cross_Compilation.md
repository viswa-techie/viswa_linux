# Chapter 4: Cross-Compilation

## Learning Goals
- Understand ARCH and CROSS_COMPILE variables
- Learn toolchain setup for ARM64, ARM, RISC-V
- Master Clang/LLVM cross-compilation
- Know common cross-compilation issues

---

## 1. Cross-Compilation Basics

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Cross-compilation: build on HOST (x86_64) for          │
  │  TARGET (arm64, arm, riscv, mips, etc.)                 │
  │                                                           │
  │  Two key variables:                                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │ ARCH=arm64                               │            │
  │  │   Tells Kconfig which arch/ to use       │            │
  │  │   Selects: arch/arm64/Kconfig            │            │
  │  │            arch/arm64/Makefile            │            │
  │  │            arch/arm64/boot/              │            │
  │  │            arch/arm64/configs/           │            │
  │  │                                          │            │
  │  │ CROSS_COMPILE=aarch64-linux-gnu-         │            │
  │  │   Prefix for ALL toolchain binaries:     │            │
  │  │   aarch64-linux-gnu-gcc                  │            │
  │  │   aarch64-linux-gnu-ld                   │            │
  │  │   aarch64-linux-gnu-objcopy              │            │
  │  │   aarch64-linux-gnu-objdump              │            │
  │  │   aarch64-linux-gnu-strip                │            │
  │  │   aarch64-linux-gnu-nm                   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Toolchain name convention:                              │
  │  <arch>-<vendor>-<os>-<abi>-                             │
  │                                                           │
  │  ┌────────────────────┬──────────────────────┐           │
  │  │ Target             │ CROSS_COMPILE=       │           │
  │  ├────────────────────┼──────────────────────┤           │
  │  │ ARM64 (AArch64)    │ aarch64-linux-gnu-   │           │
  │  │ ARM 32-bit         │ arm-linux-gnueabihf- │           │
  │  │ RISC-V 64          │ riscv64-linux-gnu-   │           │
  │  │ MIPS               │ mips-linux-gnu-      │           │
  │  │ PowerPC 64         │ powerpc64le-linux-gnu-│          │
  │  │ x86 (32-bit on 64) │ (none, native)      │           │
  │  └────────────────────┴──────────────────────┘           │
  │                                                           │
  │  Install toolchain (Ubuntu/Debian):                      │
  │  sudo apt install gcc-aarch64-linux-gnu                  │
  │  sudo apt install gcc-arm-linux-gnueabihf                │
  │  sudo apt install gcc-riscv64-linux-gnu                  │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Complete Cross-Build Example

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ARM64 kernel build:                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Set environment                        │            │
  │  │ export ARCH=arm64                        │            │
  │  │ export CROSS_COMPILE=aarch64-linux-gnu-  │            │
  │  │                                          │            │
  │  │ # Configure                              │            │
  │  │ make defconfig                           │            │
  │  │ # or: vendor-specific                    │
  │  │ make vendor_board_defconfig              │            │
  │  │ # or: use fragment:                      │            │
  │  │ scripts/kconfig/merge_config.sh \        │            │
  │  │   arch/arm64/configs/defconfig \         │            │
  │  │   custom_debug.config                    │            │
  │  │                                          │            │
  │  │ # Build                                  │            │
  │  │ make -j$(nproc)                          │            │
  │  │                                          │            │
  │  │ # Output:                                │            │
  │  │ arch/arm64/boot/Image      (kernel)      │            │
  │  │ arch/arm64/boot/Image.gz   (compressed)  │            │
  │  │ arch/arm64/boot/dts/*.dtb  (device trees)│            │
  │  │                                          │            │
  │  │ # Install modules to target rootfs:      │            │
  │  │ make modules_install \                   │            │
  │  │   INSTALL_MOD_PATH=/mnt/target_rootfs    │            │
  │  │                                          │            │
  │  │ # Install DTBs:                          │            │
  │  │ make dtbs_install \                      │            │
  │  │   INSTALL_DTBS_PATH=/mnt/target/boot/dtbs│            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Clang/LLVM cross-compilation:                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Clang is a cross-compiler by default   │            │
  │  │ # (no separate toolchain needed!)        │            │
  │  │                                          │            │
  │  │ make ARCH=arm64 LLVM=1 defconfig         │            │
  │  │ make ARCH=arm64 LLVM=1 -j$(nproc)       │            │
  │  │                                          │            │
  │  │ LLVM=1 sets:                             │            │
  │  │   CC=clang                               │            │
  │  │   LD=ld.lld                              │            │
  │  │   AR=llvm-ar                             │            │
  │  │   NM=llvm-nm                             │            │
  │  │   OBJCOPY=llvm-objcopy                   │            │
  │  │   OBJDUMP=llvm-objdump                   │            │
  │  │   STRIP=llvm-strip                       │            │
  │  │                                          │            │
  │  │ # Clang uses --target flag internally:   │            │
  │  │ clang --target=aarch64-linux-gnu         │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Common Cross-Compilation Issues

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Issue 1: Wrong ARCH                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Symptom: "No rule to make target"        │            │
  │  │ or wrong defconfig                       │            │
  │  │                                          │            │
  │  │ Note: ARCH values don't match uname -m:  │            │
  │  │ x86_64 → ARCH=x86                       │            │
  │  │ aarch64 → ARCH=arm64                     │            │
  │  │ armv7l → ARCH=arm                        │            │
  │  │ riscv → ARCH=riscv                       │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Issue 2: Host tools built with cross compiler           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Kernel builds HOST tools (fixdep, etc.)  │            │
  │  │ These must run on build machine          │            │
  │  │                                          │            │
  │  │ HOSTCC = gcc (native, for host tools)    │            │
  │  │ CC = cross compiler (for kernel code)    │            │
  │  │                                          │            │
  │  │ If HOSTCC not found: install build-      │            │
  │  │ essential / gcc on host                  │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Issue 3: Missing kernel headers for target              │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Some modules need target userspace headers│            │
  │  │ (rare for kernel itself, common for      │            │
  │  │  out-of-tree modules)                    │            │
  │  │                                          │            │
  │  │ make headers_install \                   │            │
  │  │   ARCH=arm64 \                           │            │
  │  │   INSTALL_HDR_PATH=/path/to/sysroot/usr │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Issue 4: MODVERSIONS mismatch                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Module built against different kernel     │            │
  │  │ version/config → CRC mismatch            │            │
  │  │                                          │            │
  │  │ "disagrees about version of symbol"      │            │
  │  │                                          │            │
  │  │ Fix: build module against exact same     │            │
  │  │ kernel source + .config used on target   │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does Clang/LLVM differ from GCC for kernel cross-compilation and what are the advantages?**
**A:** GCC requires a separate cross-compiler toolchain per target architecture — you install `aarch64-linux-gnu-gcc` for ARM64, `arm-linux-gnueabihf-gcc` for ARM32, etc. Each is a distinct binary with the target architecture built into the compiler. Clang is inherently a cross-compiler: a single Clang binary can emit code for any supported architecture using `--target=`. Set `LLVM=1` and `ARCH=arm64` — Clang automatically emits AArch64 code without a separate toolchain (though you still need target binutils, or use `LLVM=1` which uses `llvm-ar`, `ld.lld`, etc.). Advantages of Clang: (1) Single toolchain for all architectures. (2) Better static analysis and warnings (catches bugs GCC misses). (3) CFI (Control-Flow Integrity) and Shadow Call Stack support for kernel hardening. (4) KCFI (kernel CFI) type-based indirect call checking. (5) LTO (Link-Time Optimization) is better supported. (6) Android kernel (GKI) requires Clang. Trade-offs: some older architectures have limited Clang support, some GCC-specific extensions need porting, and inline assembly dialects may differ.

**Q2: You're setting up CI to build the same kernel source for ARM64, ARM32, and x86_64. What's your approach?**
**A:** (1) Install cross-toolchains: `gcc-aarch64-linux-gnu`, `gcc-arm-linux-gnueabihf`, and native `gcc`. Or use `LLVM=1` with a single Clang install for all three. (2) Create per-arch defconfigs or config fragments: `arch/arm64/configs/ci_defconfig`, etc. Use `scripts/kconfig/merge_config.sh` to combine base defconfig with common CI fragment (enable all debug options, KASAN, lockdep). (3) Use out-of-tree builds (`O=`) to build all three in parallel without interfering: `make O=build-arm64 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-`, `make O=build-arm ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-`, `make O=build-x86`. (4) Build targets: `make -j$(nproc)` for vmlinux + modules, `make dtbs` for ARM/ARM64. (5) Run `make C=1` (sparse) for static analysis. (6) For testing: boot the kernels in QEMU for each architecture — `qemu-system-aarch64`, `qemu-system-arm`, `qemu-system-x86_64` — run kselftest or KUnit. (7) Cross-build modules with exact `Module.symvers` from each kernel build to avoid MODVERSIONS mismatch.

---

## Summary

- ARCH= selects arch-specific Kconfig, boot code, defconfigs
- CROSS_COMPILE= prefix for all toolchain binaries (gcc, ld, objcopy)
- Clang/LLVM: LLVM=1 replaces entire GCC toolchain, one binary for all archs
- O= for out-of-tree builds — clean source, multiple configs simultaneously
- INSTALL_MOD_PATH, INSTALL_DTBS_PATH for target rootfs installation
- Common issues: wrong ARCH name, HOSTCC vs CC confusion, MODVERSIONS mismatch

---

[Previous: Kernel Makefiles ←](Chapter_03_Makefiles.md) | [Next: Modules Build and Load →](Chapter_05_Modules.md)
