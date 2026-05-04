# Chapter 2: Kbuild System

## Learning Goals
- Understand obj-y, obj-m, and obj-n build mechanics
- Learn how Kbuild processes Makefiles recursively
- Master build stages: preprocessing, compilation, linking
- Know ccflags, asflags, ldflags, and header dependencies

---

## 1. Kbuild Makefile Syntax

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Kbuild Makefiles are NOT standard GNU Makefiles        │
  │  They use special Kbuild variables processed by         │
  │  scripts/Makefile.build                                  │
  │                                                           │
  │  Core variables:                                         │
  │  ┌──────────────────────────────────────────┐            │
  │  │ obj-y += foo.o                           │            │
  │  │ # foo.c compiled and linked into vmlinux │            │
  │  │                                          │            │
  │  │ obj-m += bar.o                           │            │
  │  │ # bar.c compiled as bar.ko module        │            │
  │  │                                          │            │
  │  │ obj-$(CONFIG_BAZ) += baz.o               │            │
  │  │ # If CONFIG_BAZ=y → obj-y (built-in)    │            │
  │  │ # If CONFIG_BAZ=m → obj-m (module)      │            │
  │  │ # If CONFIG_BAZ=n → obj-n (not built)   │            │
  │  │                                          │            │
  │  │ obj-$(CONFIG_FOO) += foo/                │            │
  │  │ # Recurse into foo/ subdirectory         │            │
  │  │ # ONLY if CONFIG_FOO=y or m             │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Multi-file modules:                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ obj-$(CONFIG_E1000) += e1000.o           │            │
  │  │ e1000-y := e1000_main.o e1000_hw.o \     │            │
  │  │            e1000_ethtool.o                │            │
  │  │                                          │            │
  │  │ # Result: e1000.ko contains all three    │            │
  │  │ # e1000-y = objects always included      │            │
  │  │ # e1000-$(CONFIG_E1000_DEBUG) += debug.o │            │
  │  │ # Conditional object in multi-file module│            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Compiler/linker flags:                                  │
  │  ┌──────────────────────────────────────────┐            │
  │  │ ccflags-y += -DDEBUG -I$(src)/include    │            │
  │  │ # Applies to ALL files in this directory │            │
  │  │                                          │            │
  │  │ CFLAGS_foo.o += -DFOO_SPECIAL           │            │
  │  │ # Applies ONLY to foo.c compilation      │            │
  │  │                                          │            │
  │  │ asflags-y += -DASSEMBLY                  │            │
  │  │ # Assembly file flags                    │            │
  │  │                                          │            │
  │  │ ldflags-y += -T custom.lds               │            │
  │  │ # Linker flags                           │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Build Process Overview

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  make (top-level):                                       │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. Configuration                         │            │
  │  │    .config → include/generated/autoconf.h │            │
  │  │    .config → include/config/auto.conf     │            │
  │  │                                          │            │
  │  │ 2. Recursive build                       │            │
  │  │    Top Makefile → scripts/Makefile.build  │            │
  │  │    Processes each directory's Kbuild file │            │
  │  │                                          │            │
  │  │    For each obj-y file:                  │            │
  │  │    foo.c → foo.o (compile)               │            │
  │  │    All obj-y .o → built-in.a (archive)   │            │
  │  │                                          │            │
  │  │    For each obj-m file:                  │            │
  │  │    bar.c → bar.o → bar.ko (link module)  │            │
  │  │                                          │            │
  │  │ 3. Final linking                         │            │
  │  │    All built-in.a archives →              │            │
  │  │    vmlinux (ELF) →                       │            │
  │  │    vmlinux.bin (objcopy, stripped) →      │            │
  │  │    bzImage/Image (compressed, bootable)  │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Build flow diagram:                                     │
  │                                                           │
  │  .c files ──→ .o files ──→ built-in.a ──→ vmlinux      │
  │  (compile)    (per-dir)    (archive)       (link)       │
  │                                                           │
  │  .c files ──→ .o files ──→ .ko modules                  │
  │  (compile)    (link into module)                         │
  │                                                           │
  │  vmlinux ──→ System.map + vmlinux.bin ──→ bzImage       │
  │  (ELF)       (symbols)   (raw binary)    (bootable)     │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Header Dependencies and Fixdep

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Problem: .c file includes headers that check CONFIG_*  │
  │  Changing CONFIG_FOO should rebuild files that use it   │
  │                                                           │
  │  Solution: fixdep tool                                   │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. GCC generates .foo.o.d (makedepend):  │            │
  │  │    foo.o: foo.c include/linux/module.h \  │            │
  │  │           include/generated/autoconf.h    │            │
  │  │                                          │            │
  │  │ 2. fixdep processes .foo.o.d:            │            │
  │  │    - Scans for CONFIG_* references       │            │
  │  │    - Generates .foo.o.cmd with:          │            │
  │  │      deps = include/config/FOO.h         │            │
  │  │      (auto-generated empty file)         │            │
  │  │                                          │            │
  │  │ 3. On next build:                        │            │
  │  │    If CONFIG_FOO changes →               │            │
  │  │    include/config/FOO.h touched →        │            │
  │  │    foo.o gets rebuilt                     │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  include/config/ contains one empty file per config     │
  │  option. Timestamp = when that option last changed.     │
  │  This is how Kbuild knows what to rebuild.              │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Built-in vs Module Compilation

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Built-in (obj-y):                                       │
  │  ┌──────────────────────────────────────────┐            │
  │  │                                          │            │
  │  │ drivers/net/ethernet/intel/e1000/        │            │
  │  │   e1000_main.o ─┐                        │            │
  │  │   e1000_hw.o ───┼─→ built-in.a          │            │
  │  │   e1000_ethtool.o┘                       │            │
  │  │                                          │            │
  │  │ drivers/net/built-in.a ─┐                │            │
  │  │ drivers/block/built-in.a┼→ vmlinux      │            │
  │  │ fs/built-in.a ──────────┘                │            │
  │  │                                          │            │
  │  │ Linked at build time, always in kernel   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Module (obj-m):                                         │
  │  ┌──────────────────────────────────────────┐            │
  │  │                                          │            │
  │  │ Compilation:                             │            │
  │  │ foo.c → foo.o (normal compile)           │            │
  │  │ foo.mod.c generated (module metadata)    │            │
  │  │ foo.o + foo.mod.o → foo.ko (modpost+ld)  │            │
  │  │                                          │            │
  │  │ modpost step:                            │            │
  │  │ scripts/mod/modpost processes all .o     │            │
  │  │ - Generates Module.symvers              │            │
  │  │ - Creates .mod.c (module info section)  │            │
  │  │ - Checks symbol versioning (CRC)        │            │
  │  │ - Verifies exports match                │            │
  │  │                                          │            │
  │  │ .ko contains:                            │            │
  │  │ - .text, .data, .bss (code and data)    │            │
  │  │ - .modinfo section (license, author,     │            │
  │  │   description, alias, depends)          │            │
  │  │ - __versions section (CRC of used symbols)│           │
  │  │ - .gnu.linkonce.this_module              │            │
  │  │   (struct module instance)               │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Explain what happens when you write `obj-$(CONFIG_FOO) += foo.o` in a Kbuild Makefile.**
**A:** This is the core Kbuild mechanism that ties Kconfig configuration to compilation. `CONFIG_FOO` is a Make variable expanded from `include/config/auto.conf` (generated from `.config`). Three cases: (1) `CONFIG_FOO=y`: expands to `obj-y += foo.o` — `foo.c` is compiled into `foo.o` and archived into the directory's `built-in.a`, which is eventually linked into `vmlinux`. (2) `CONFIG_FOO=m`: expands to `obj-m += foo.o` — `foo.c` is compiled into `foo.o`, then `modpost` generates `foo.mod.c` (module metadata: license, author, symbol CRCs), compiles it to `foo.mod.o`, and links both into `foo.ko`. The `.ko` file is installed to `/lib/modules/$(uname -r)/` and loaded via `modprobe` or `insmod`. (3) `CONFIG_FOO` is not set (n): expands to `obj- += foo.o` or `obj-n += foo.o` — `foo.c` is not compiled at all. For subdirectories: `obj-$(CONFIG_FOO) += foo/` means Kbuild only recurses into `foo/` when FOO is y or m. This is how entire subsystems are conditionally included or excluded from the build.

**Q2: What is modpost and what does Module.symvers contain?**
**A:** `modpost` (scripts/mod/modpost) is a post-processing step that runs after all kernel objects and modules are compiled but before final `.ko` linking. It performs: (1) **Symbol resolution**: scans all compiled modules' `.o` files and `vmlinux` to check that every undefined symbol a module references is either exported by vmlinux or another module. Undefined symbols cause a warning or error. (2) **Symbol versioning (MODVERSIONS)**: for each exported symbol (`EXPORT_SYMBOL`), modpost computes a CRC of the symbol's type signature. This CRC is stored in `Module.symvers` (format: `CRC symbol module export_type`) and embedded in each `.ko`'s `__versions` section. When `insmod` loads a module, the kernel checks that the CRC in the module matches the CRC in the running kernel — if they differ, the symbol's prototype has changed and loading is refused ("disagrees about version of symbol"). (3) **Module metadata**: generates `.mod.c` containing the `struct module` with name, init/exit functions, and `.modinfo` strings (MODULE_LICENSE, MODULE_AUTHOR, MODULE_ALIAS). `Module.symvers` is essential for out-of-tree module builds — it provides the symbol CRCs that the external module must match. You copy it from the kernel build tree or install the `linux-headers` package.

---

## Summary

- obj-y: compiled into vmlinux (built-in.a archives linked together)
- obj-m: compiled as loadable .ko module (modpost generates metadata)
- obj-$(CONFIG_X): ties Kconfig choice to compilation decision
- ccflags-y: compiler flags for all files; CFLAGS_foo.o: per-file flags
- fixdep: tracks CONFIG_* usage per .c file for precise incremental builds
- modpost: symbol resolution, CRC versioning, .mod.c generation
- Module.symvers: maps exported symbols to CRCs for version checking

---

[Previous: Kconfig System ←](Chapter_01_Kconfig.md) | [Next: Kernel Makefiles →](Chapter_03_Makefiles.md)
