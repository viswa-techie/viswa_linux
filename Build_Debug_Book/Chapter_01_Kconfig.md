# Chapter 1: Kconfig Configuration System

## Learning Goals
- Understand Kconfig language syntax and semantics
- Learn how .config is generated and used by Kbuild
- Master config option types: bool, tristate, string, int, hex
- Know dependency mechanisms: depends on, select, imply

---

## 1. Kconfig Language Overview

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Kconfig = kernel configuration language                 │
  │  Files: Kconfig throughout kernel source tree            │
  │                                                           │
  │  kernel/                                                 │
  │  ├── Kconfig         (top-level, sources others)        │
  │  ├── arch/arm64/Kconfig                                 │
  │  ├── drivers/Kconfig                                    │
  │  │   ├── drivers/net/Kconfig                            │
  │  │   ├── drivers/gpu/Kconfig                            │
  │  │   └── drivers/usb/Kconfig                            │
  │  ├── fs/Kconfig                                          │
  │  └── ...                                                │
  │                                                           │
  │  Top-level Kconfig:                                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │ mainmenu "Linux/$(ARCH) Kernel Config"   │            │
  │  │                                          │            │
  │  │ source "arch/$(SRCARCH)/Kconfig"         │            │
  │  │                                          │            │
  │  │ # This sources ALL sub-Kconfig files     │            │
  │  │ # recursively based on ARCH              │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Configuration frontends:                                │
  │  ┌──────────────────────────────────────────┐            │
  │  │ make menuconfig    — ncurses TUI         │            │
  │  │ make nconfig        — newer ncurses TUI  │            │
  │  │ make xconfig        — Qt GUI             │            │
  │  │ make gconfig        — GTK GUI            │            │
  │  │ make config         — line-by-line CLI   │            │
  │  │ make oldconfig      — update old .config │            │
  │  │ make olddefconfig   — oldconfig + defaults│           │
  │  │ make defconfig      — arch default config│            │
  │  │ make allmodconfig   — all as modules     │            │
  │  │ make allyesconfig   — all yes            │            │
  │  │ make allnoconfig    — minimal            │            │
  │  │ make localmodconfig — only loaded modules│            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Kconfig Option Types

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  config EXT4_FS                                          │
  │      tristate "The Extended 4 filesystem"                │
  │      depends on BLOCK                                    │
  │      select CRC16                                        │
  │      select CRYPTO_CRC32C                                │
  │      help                                                │
  │        This is the next generation of ext3 filesystem.  │
  │        If unsure, say Y.                                │
  │                                                           │
  │  Option types:                                           │
  │  ┌────────────┬───────────────────────────────────────┐ │
  │  │ Type       │ Values              │ In .config      │ │
  │  ├────────────┼─────────────────────┼─────────────────┤ │
  │  │ bool       │ y / n               │ CONFIG_FOO=y    │ │
  │  │            │                     │ # CONFIG_FOO    │ │
  │  │            │                     │   is not set   │ │
  │  ├────────────┼─────────────────────┼─────────────────┤ │
  │  │ tristate   │ y / m / n           │ CONFIG_FOO=y    │ │
  │  │            │ y=built-in          │ CONFIG_FOO=m    │ │
  │  │            │ m=module            │ # not set       │ │
  │  │            │ n=disabled          │                 │ │
  │  ├────────────┼─────────────────────┼─────────────────┤ │
  │  │ string     │ "text"              │ CONFIG_FOO="bar"│ │
  │  ├────────────┼─────────────────────┼─────────────────┤ │
  │  │ int        │ integer             │ CONFIG_FOO=256  │ │
  │  ├────────────┼─────────────────────┼─────────────────┤ │
  │  │ hex        │ hex value           │ CONFIG_FOO=0x100│ │
  │  └────────────┴─────────────────────┴─────────────────┘ │
  │                                                           │
  │  tristate is critical:                                   │
  │  y → code compiled into vmlinux (always present)        │
  │  m → code compiled as .ko module (loaded on demand)     │
  │  n → code not compiled at all                           │
  │                                                           │
  │  In C code:                                              │
  │  #ifdef CONFIG_EXT4_FS     → true if y or m            │
  │  #if IS_ENABLED(...)       → true if y or m            │
  │  #if IS_BUILTIN(...)       → true only if y            │
  │  #if IS_MODULE(...)        → true only if m            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Dependencies and Selection

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  depends on — option is only visible/selectable          │
  │  when dependency is met                                  │
  │                                                           │
  │  config USB_STORAGE                                      │
  │      tristate "USB Mass Storage support"                 │
  │      depends on USB && SCSI                              │
  │      # Can only enable if BOTH USB and SCSI are on      │
  │                                                           │
  │  select — forces another option ON when this is enabled  │
  │  (reverse dependency — USE SPARINGLY)                    │
  │                                                           │
  │  config EXT4_FS                                          │
  │      select CRC16                                        │
  │      # Enabling EXT4 automatically enables CRC16        │
  │      # CRC16 can't be turned off while EXT4 is on      │
  │                                                           │
  │  imply — weak select (suggests but doesn't force)        │
  │                                                           │
  │  config FOO                                              │
  │      imply BAR                                           │
  │      # Enabling FOO sets BAR=y by default but           │
  │      # user CAN turn BAR off                            │
  │                                                           │
  │  Dependency problems:                                    │
  │  ┌──────────────────────────────────────────┐            │
  │  │ select can override depends on!           │            │
  │  │                                          │            │
  │  │ config A                                 │            │
  │  │     depends on X                         │            │
  │  │                                          │            │
  │  │ config B                                 │            │
  │  │     select A                             │            │
  │  │     # B selects A even if X is off!      │            │
  │  │     # This causes build errors          │            │
  │  │     # A's code compiles but X isn't there │            │
  │  │                                          │            │
  │  │ Rule: only select options with NO deps   │            │
  │  │ (libraries: CRC, CRYPTO, BITOPS, etc.)  │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Menu structure:                                         │
  │  ┌──────────────────────────────────────────┐            │
  │  │ menu "Network device support"            │            │
  │  │     depends on NET                       │            │
  │  │                                          │            │
  │  │     config NETDEVICES                    │            │
  │  │         bool "Network device support"    │            │
  │  │         default y                        │            │
  │  │                                          │            │
  │  │     config NET_VENDOR_INTEL              │            │
  │  │         bool "Intel devices"             │            │
  │  │         depends on NETDEVICES            │            │
  │  │                                          │            │
  │  │ endmenu                                  │            │
  │  │                                          │            │
  │  │ menuconfig USB                           │            │
  │  │     tristate "USB support"               │            │
  │  │     # Creates a menu that can be toggled │            │
  │  │     # on/off AND entered                 │            │
  │  │                                          │            │
  │  │ if USB  # Everything until endif needs   │            │
  │  │         # USB to be enabled              │            │
  │  │ config USB_STORAGE                       │            │
  │  │     tristate "USB Mass Storage"          │            │
  │  │ endif                                    │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. .config File and defconfig

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  .config lifecycle:                                      │
  │                                                           │
  │  1. Start: make defconfig                               │
  │     arch/arm64/configs/defconfig → .config               │
  │                                                           │
  │  2. Customize: make menuconfig                           │
  │     .config updated with user choices                   │
  │                                                           │
  │  3. Build: make                                          │
  │     .config → include/generated/autoconf.h               │
  │     .config → include/config/auto.conf                   │
  │                                                           │
  │  Generated files:                                        │
  │  ┌──────────────────────────────────────────┐            │
  │  │ autoconf.h:                              │            │
  │  │   #define CONFIG_EXT4_FS 1               │            │
  │  │   #define CONFIG_EXT4_FS_MODULE 1        │            │
  │  │   /* CONFIG_XFS is not set */            │            │
  │  │                                          │            │
  │  │ auto.conf:                               │            │
  │  │   CONFIG_EXT4_FS=y                       │            │
  │  │   CONFIG_DEFAULT_HOSTNAME="(none)"       │            │
  │  │   # Used by Kbuild makefiles             │            │
  │  │                                          │            │
  │  │ auto.conf.cmd:                           │            │
  │  │   # Dependencies for auto.conf rebuild   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Saving custom defconfig:                                │
  │  make savedefconfig                                      │
  │  # Creates defconfig (minimal, only non-default values) │
  │  cp defconfig arch/arm64/configs/my_defconfig            │
  │                                                           │
  │  Config fragments (merge multiple configs):              │
  │  scripts/kconfig/merge_config.sh                         │
  │    arch/arm64/configs/defconfig \                        │
  │    kernel/configs/debug.config \                         │
  │    kernel/configs/android.config                         │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Explain the difference between `depends on`, `select`, and `imply` in Kconfig.**
**A:** `depends on` creates a forward dependency: the option is only visible and selectable when the dependency expression is true. If `config A depends on B`, you cannot enable A unless B is already enabled. Disabling B automatically disables A. `select` creates a reverse dependency: enabling the selecting option forces the selected option on. If `config A select B`, enabling A automatically enables B. B cannot be disabled while A is enabled. This is dangerous: if B has its own `depends on C`, select bypasses that — B gets forced on even if C is off, potentially causing build errors. Rule: only `select` leaf/library options that have no dependencies (CRC, CRYPTO primitives). `imply` is a weak select: if `config A imply B`, enabling A sets B to y by default, but the user can manually turn B off. It's a suggestion, not a force. Used for optional companion features. Example: a NIC driver might `imply` its PTP (Precision Time Protocol) support — enabled by default but user can disable if not needed.

**Q2: How does `make oldconfig` work and when do you use it?**
**A:** `oldconfig` reads an existing `.config` file and processes it against the current kernel source's Kconfig files. For every config symbol that exists in the old `.config` and still exists in the current Kconfig, it keeps the old value. For every NEW symbol (added in a newer kernel version), it prompts the user interactively to choose a value. For removed symbols, it silently drops them. Use cases: (1) Upgrading kernel version — your old `.config` may be missing symbols added in the new version. `oldconfig` preserves your customizations and asks only about new options. (2) `olddefconfig` is the non-interactive variant — it accepts defaults for new symbols without prompting. Used in automated build systems and CI. (3) `make listnewconfig` shows new options without prompting, useful for reviewing what changed. The process internally: Kconfig parser reads all Kconfig files to build the symbol database, loads `.config`, resolves dependencies, identifies symbols without values, and prompts or defaults them. The resulting `.config` is guaranteed to be valid (all dependencies satisfied).

---

## Summary

- Kconfig: hierarchical configuration language for kernel build options
- Types: bool (y/n), tristate (y/m/n), string, int, hex
- depends on: forward dependency — option hidden unless dependency met
- select: reverse dependency — forces option on (use only for leaf libs)
- imply: weak select — default yes but user can override
- .config → autoconf.h (C code) + auto.conf (Makefiles)
- defconfig: minimal config with only non-default values
- make oldconfig: preserve old .config, prompt for new symbols

---

[Next: Kbuild System →](Chapter_02_Kbuild.md)
