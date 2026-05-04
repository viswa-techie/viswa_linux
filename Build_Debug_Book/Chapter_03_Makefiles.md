# Chapter 3: Kernel Makefiles

## Learning Goals
- Understand the top-level Makefile structure
- Learn make targets and build variables
- Master clean, mrproper, distclean differences
- Know parallel builds and build output directories

---

## 1. Top-Level Makefile Structure

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Kernel top-level Makefile (~2000 lines):                │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Section 1: Version                       │            │
  │  │   VERSION = 6                            │            │
  │  │   PATCHLEVEL = 8                         │            │
  │  │   SUBLEVEL = 0                           │            │
  │  │   EXTRAVERSION = -rc1                    │            │
  │  │   NAME = Hurr durr I'ma ninja sloth      │            │
  │  │                                          │            │
  │  │ Section 2: Build environment             │            │
  │  │   ARCH, CROSS_COMPILE, CC, LD            │            │
  │  │   HOSTCC, HOSTCXX (for host tools)       │            │
  │  │                                          │            │
  │  │ Section 3: Default target                │            │
  │  │   PHONY := __all                         │            │
  │  │   __all: all                             │            │
  │  │   # "all" = vmlinux + modules            │            │
  │  │                                          │            │
  │  │ Section 4: Include auto.conf             │            │
  │  │   include include/config/auto.conf       │            │
  │  │   # CONFIG_* available as Make vars      │            │
  │  │                                          │            │
  │  │ Section 5: Directories to build          │            │
  │  │   init-y  := init/                       │            │
  │  │   core-y  := usr/ kernel/ certs/ mm/ \   │            │
  │  │             fs/ ipc/ security/ crypto/    │            │
  │  │   drivers-y := drivers/ sound/ firmware/ │            │
  │  │   net-y   := net/                        │            │
  │  │   libs-y  := lib/                        │            │
  │  │                                          │            │
  │  │ Section 6: vmlinux linking               │            │
  │  │   vmlinux: scripts/link-vmlinux.sh       │            │
  │  │   Uses all built-in.a from above dirs    │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Important Make Targets

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Build targets:                                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ make                  — default (all)    │            │
  │  │ make vmlinux          — kernel ELF       │            │
  │  │ make modules          — all .ko files    │            │
  │  │ make bzImage          — compressed (x86) │            │
  │  │ make Image            — kernel image (ARM64)│         │
  │  │ make dtbs             — device tree blobs│            │
  │  │ make modules_install  — install to /lib  │            │
  │  │ make install          — install kernel   │            │
  │  │ make headers_install  — UAPI headers     │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Clean targets:                                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ make clean                               │            │
  │  │   Removes: *.o, *.ko, built-in.a,        │            │
  │  │   vmlinux, modules, most generated files │            │
  │  │   Keeps: .config, Module.symvers         │            │
  │  │                                          │            │
  │  │ make mrproper                            │            │
  │  │   = clean + removes .config, auto.conf,  │            │
  │  │   autoconf.h, all generated config files │            │
  │  │   Returns tree to pristine source state  │            │
  │  │                                          │            │
  │  │ make distclean                           │            │
  │  │   = mrproper + removes editor backups,   │            │
  │  │   patch files, *.orig, tags              │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Info targets:                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ make kernelversion    — print version    │            │
  │  │ make kernelrelease    — version + local  │            │
  │  │ make help             — list all targets │            │
  │  │ make listnewconfig    — show new options │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Build control:                                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ make -j$(nproc)       — parallel build   │            │
  │  │ make V=1              — verbose commands │            │
  │  │ make V=2              — show why rebuild │            │
  │  │ make W=1              — extra warnings   │            │
  │  │ make C=1              — sparse check     │            │
  │  │ make C=2              — sparse, all files│            │
  │  │ make M=drivers/net/   — build one dir    │            │
  │  │ make SUBDIRS=drivers/ — (legacy) same    │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Out-of-Tree Builds (O=)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Build in separate directory (keep source clean):       │
  │                                                           │
  │  make O=/path/to/build defconfig                        │
  │  make O=/path/to/build -j$(nproc)                       │
  │                                                           │
  │  Source tree: /home/user/linux/ (read-only, clean)      │
  │  Build tree:  /path/to/build/  (all generated files)    │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ /path/to/build/                          │            │
  │  │   .config                                │            │
  │  │   vmlinux                                │            │
  │  │   System.map                             │            │
  │  │   include/generated/autoconf.h           │            │
  │  │   drivers/net/ethernet/intel/e1000/      │            │
  │  │     e1000_main.o                         │            │
  │  │     e1000.ko                             │            │
  │  │   ...                                    │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Benefits:                                               │
  │  - Multiple configs from same source                    │
  │  - Source tree stays pristine (git clean)               │
  │  - Build different ARCHs simultaneously                 │
  │                                                           │
  │  Out-of-tree MODULE build:                               │
  │  ┌──────────────────────────────────────────┐            │
  │  │ make -C /lib/modules/$(uname -r)/build \ │            │
  │  │   M=$PWD modules                         │            │
  │  │                                          │            │
  │  │ -C: change to kernel build directory     │            │
  │  │ M=: path to external module source       │            │
  │  │                                          │            │
  │  │ External module Kbuild file:             │            │
  │  │   obj-m += mydriver.o                    │            │
  │  │   mydriver-y := main.o util.o            │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Build Variables and LOCALVERSION

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Key build variables:                                    │
  │  ┌──────────────────────────────────────────┐            │
  │  │ ARCH=arm64              — target arch    │            │
  │  │ CROSS_COMPILE=aarch64-linux-gnu-         │            │
  │  │                         — toolchain prefix│           │
  │  │ CC=clang                — compiler       │            │
  │  │ LD=ld.lld               — linker         │            │
  │  │ LLVM=1                  — use full LLVM  │            │
  │  │ INSTALL_MOD_PATH=/mnt   — module install │            │
  │  │ INSTALL_PATH=/boot      — kernel install │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  LOCALVERSION:                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Appended to kernel version string:       │            │
  │  │                                          │            │
  │  │ CONFIG_LOCALVERSION="-custom"            │            │
  │  │ → uname -r: 6.8.0-custom                │            │
  │  │                                          │            │
  │  │ CONFIG_LOCALVERSION_AUTO=y               │            │
  │  │ → appends git commit: 6.8.0-custom-g123abc│          │
  │  │                                          │            │
  │  │ LOCALVERSION on command line:            │            │
  │  │ make LOCALVERSION=-test                  │            │
  │  │ → 6.8.0-custom-test                     │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Complete build example:                                 │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # ARM64 cross-compilation               │            │
  │  │ export ARCH=arm64                        │            │
  │  │ export CROSS_COMPILE=aarch64-linux-gnu-  │            │
  │  │                                          │            │
  │  │ make defconfig         # default .config │            │
  │  │ make menuconfig        # customize       │            │
  │  │ make -j$(nproc)        # build all       │            │
  │  │ make modules_install \ # install modules │            │
  │  │   INSTALL_MOD_PATH=/mnt/rootfs           │            │
  │  │ make dtbs              # device trees    │            │
  │  │ make install            # install kernel │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Explain the difference between `make clean`, `make mrproper`, and `make distclean`.**
**A:** These are progressively more aggressive cleaning targets. `make clean` removes most generated files: all `.o` object files, `.ko` module files, `built-in.a` archives, `vmlinux`, `Image`/`bzImage`, generated DTBs, and most intermediate files. But it preserves `.config`, `Module.symvers`, and the configuration infrastructure — you can immediately run `make` again to rebuild with the same configuration. `make mrproper` does everything `clean` does PLUS removes all configuration files: `.config`, `include/generated/autoconf.h`, `include/config/auto.conf`, `include/config/*.h` (config dependency files). The source tree returns to the state you'd get from a fresh `git checkout` — you must run `make defconfig` or `make menuconfig` again before building. `make distclean` does everything `mrproper` does PLUS removes editor backup files (`*~`, `*.orig`), patch remnants, `tags`, `TAGS`, and `cscope` database files. Used before creating a source distribution tarball.

**Q2: How do you build an out-of-tree kernel module and what files are needed?**
**A:** An out-of-tree module is built against installed kernel headers using a minimal Kbuild file. Required: (1) A `Kbuild` (or `Makefile`) containing at minimum: `obj-m += mymodule.o` (and for multi-file: `mymodule-y := main.o util.o`). (2) The kernel headers package installed (`linux-headers-$(uname -r)`) which provides the Kbuild infrastructure, auto.conf, Module.symvers, and exported headers. Build command: `make -C /lib/modules/$(uname -r)/build M=$PWD modules`. The `-C` flag makes Make change to the kernel build directory, which reads the kernel's top-level Makefile and Kbuild infrastructure. `M=$PWD` tells Kbuild to build only the module in the specified directory. `modules` is the make target. This invokes Kbuild's `scripts/Makefile.build` which processes the external Kbuild file, compiles `.c` → `.o`, runs modpost (checking symbol versions against the kernel's `Module.symvers`), and links into `.ko`. Install: `make -C /lib/modules/.../build M=$PWD modules_install` copies the `.ko` to `/lib/modules/.../extra/`. Then `depmod -a` rebuilds the module dependency database so `modprobe` can find it.

---

## Summary

- Top-level Makefile: version, build dirs, includes auto.conf, vmlinux linking
- clean: remove objects, keep .config; mrproper: remove .config too; distclean: everything
- make V=1: verbose; make W=1: extra warnings; make C=1: sparse checking
- O=dir: out-of-tree build (keep source clean, multiple configs)
- M=dir: build external module against kernel headers
- LOCALVERSION: append custom string to kernel version
- Parallel build: make -j$(nproc) for speed

---

[Previous: Kbuild System ←](Chapter_02_Kbuild.md) | [Next: Cross-Compilation →](Chapter_04_Cross_Compilation.md)
