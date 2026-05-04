# Chapter 8: Linker Scripts and vmlinux

## Learning Goals
- Understand vmlinux.lds linker script structure
- Learn section ordering and KEEP for essential sections
- Master initcall levels and how init functions are ordered
- Know System.map, kallsyms, and symbol resolution

---

## 1. vmlinux.lds Linker Script

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  vmlinux.lds.S → preprocessed → vmlinux.lds             │
  │  Located: arch/$(ARCH)/kernel/vmlinux.lds.S              │
  │  Uses: include/asm-generic/vmlinux.lds.h (macros)       │
  │                                                           │
  │  Key sections in vmlinux:                                │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Virtual Address Space:                   │            │
  │  │                                          │            │
  │  │ ┌────────────────────────┐ high addr     │            │
  │  │ │ .bss                   │ (zero-init)   │            │
  │  │ ├────────────────────────┤               │            │
  │  │ │ .init.data             │ (freed)       │            │
  │  │ │ .init.text             │ (freed)       │            │
  │  │ │ .init.rodata           │ (freed)       │            │
  │  │ ├────────────────────────┤               │            │
  │  │ │ .data..percpu          │ per-CPU data  │            │
  │  │ ├────────────────────────┤               │            │
  │  │ │ .data                  │ read-write    │            │
  │  │ ├────────────────────────┤               │            │
  │  │ │ .rodata                │ read-only     │            │
  │  │ ├────────────────────────┤               │            │
  │  │ │ .text                  │ code          │            │
  │  │ ├────────────────────────┤               │            │
  │  │ │ .head.text             │ entry point   │            │
  │  │ └────────────────────────┘ _text (start) │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Linker script syntax:                                   │
  │  ┌──────────────────────────────────────────┐            │
  │  │ SECTIONS {                               │            │
  │  │   . = KERNEL_START;    /* set address */ │            │
  │  │                                          │            │
  │  │   .head.text : {                         │            │
  │  │     _text = .;         /* symbol */      │            │
  │  │     HEAD_TEXT                             │            │
  │  │   }                                      │            │
  │  │                                          │            │
  │  │   .text : {                              │            │
  │  │     _stext = .;                          │            │
  │  │     TEXT_TEXT           /* actual code */ │            │
  │  │     SCHED_TEXT          /* scheduler */   │            │
  │  │     LOCK_TEXT           /* spinlocks */   │            │
  │  │     _etext = .;                          │            │
  │  │   }                                      │            │
  │  │                                          │            │
  │  │   .rodata : {                            │            │
  │  │     RODATA                               │            │
  │  │   }                                      │            │
  │  │                                          │            │
  │  │   .init.text : {                         │            │
  │  │     _sinittext = .;                      │            │
  │  │     INIT_TEXT           /* __init funcs */│            │
  │  │     _einittext = .;                      │            │
  │  │   }                                      │            │
  │  │                                          │            │
  │  │   .init.data : {                         │            │
  │  │     INIT_DATA                            │            │
  │  │     INIT_CALLS         /* initcalls! */  │            │
  │  │   }                                      │            │
  │  │ }                                        │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Initcall Levels

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Initcalls: ordered init function execution at boot     │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Level  Macro                  Example    │            │
  │  │ ──────────────────────────────────────── │            │
  │  │ 0      early_initcall()       cpu init   │            │
  │  │ 1      pure_initcall()        irq init   │            │
  │  │ 2      core_initcall()        buses      │            │
  │  │ 3      postcore_initcall()    pci enum   │            │
  │  │ 4      arch_initcall()        arch setup │            │
  │  │ 5      subsys_initcall()      net/usb    │            │
  │  │ 6      fs_initcall()          filesystems│            │
  │  │ 7      device_initcall()      drivers    │            │
  │  │ 7s     late_initcall()        cleanup    │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  module_init() → device_initcall() for built-in         │
  │                → module constructor for .ko             │
  │                                                           │
  │  How it works:                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ #define core_initcall(fn) \              │            │
  │  │   static initcall_t \                    │            │
  │  │   __initcall_##fn##2 \                   │            │
  │  │   __used \                               │            │
  │  │   __section(".initcall2.init") = fn;     │            │
  │  │                                          │            │
  │  │ This creates a function pointer in the   │            │
  │  │ .initcall2.init section                  │            │
  │  │                                          │            │
  │  │ Linker script collects them in order:    │            │
  │  │ INIT_CALLS = {                           │            │
  │  │   __initcall0_start = .;                 │            │
  │  │   KEEP(*(.initcall0.init))               │            │
  │  │   __initcall1_start = .;                 │            │
  │  │   KEEP(*(.initcall1.init))               │            │
  │  │   ...                                    │            │
  │  │   __initcall7_start = .;                 │            │
  │  │   KEEP(*(.initcall7.init))               │            │
  │  │ }                                        │            │
  │  │                                          │            │
  │  │ do_initcalls() iterates: for each level, │            │
  │  │ call all function pointers between       │            │
  │  │ __initcallN_start and __initcallN+1_start│            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  KEEP: prevents linker from discarding the              │
  │  section even with LTO/gc-sections (these               │
  │  pointers aren't referenced directly)                   │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. System.map and kallsyms

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  System.map (build-time symbol table):                   │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Generated by: nm vmlinux | sort          │            │
  │  │                                          │            │
  │  │ Format: address type name                │            │
  │  │ ffff800010000000 T _text                 │            │
  │  │ ffff800010081234 T schedule              │            │
  │  │ ffff80001020abcd T ext4_fill_super       │            │
  │  │ ffff800010300000 D jiffies               │            │
  │  │ ffff800010400000 B empty_zero_page       │            │
  │  │                                          │            │
  │  │ Types: T=text, D=data, B=bss, R=rodata  │            │
  │  │        t/d/b = local (static) symbols    │            │
  │  │                                          │            │
  │  │ Used by: crash tool, ksymoops, debugging│            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  kallsyms (runtime symbol table):                        │
  │  ┌──────────────────────────────────────────┐            │
  │  │ CONFIG_KALLSYMS=y                        │            │
  │  │                                          │            │
  │  │ Embeds symbol table INTO vmlinux:        │            │
  │  │ scripts/kallsyms processes nm output     │            │
  │  │ Compressed symbol names in .rodata       │            │
  │  │                                          │            │
  │  │ At runtime:                              │            │
  │  │ cat /proc/kallsyms                       │            │
  │  │ ffff800010081234 T schedule              │            │
  │  │                                          │            │
  │  │ Used by:                                 │            │
  │  │ - Oops/panic stack traces (name decode)  │            │
  │  │ - ftrace (function name display)         │            │
  │  │ - perf (symbol resolution)               │            │
  │  │ - BPF (kprobe attachment by name)        │            │
  │  │                                          │            │
  │  │ CONFIG_KALLSYMS_ALL=y                    │            │
  │  │ Include ALL symbols (not just functions) │            │
  │  │ Needed for data symbol resolution        │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Explain initcall levels and how the kernel ensures drivers initialize in the correct order.**
**A:** Initcall levels provide coarse-grained ordering for kernel initialization. There are 8 levels (0-7, plus 7s for late), each mapped to a linker section (`.initcallN.init`). The `INIT_CALLS` macro in the linker script collects function pointers from these sections in order, using `KEEP()` to prevent LTO from discarding them. `do_initcalls()` iterates through levels sequentially, calling each function pointer. This ensures: core subsystems (level 2: buses, allocators) initialize before subsystem drivers (level 5: USB, networking frameworks) which initialize before specific device drivers (level 7: individual drivers). `module_init()` maps to `device_initcall()` (level 7) for built-in code. WITHIN a level, order depends on link order (which files are listed first in the Makefile's `obj-y`), which is brittle. For fine-grained dependency ordering, drivers use the driver model: probe functions with deferred probing (`-EPROBE_DEFER`). If a driver's probe() finds its dependency (clock, regulator, GPIO) isn't ready yet, it returns `EPROBE_DEFER`. The driver core re-queues it and retries later. This mechanism handles the fact that initcall level ordering alone can't express complex dependency graphs.

**Q2: What is the purpose of `KEEP()` in the linker script and what happens without it?**
**A:** `KEEP()` instructs the linker to retain the specified input sections even when `--gc-sections` (garbage collection) is used. With `-ffunction-sections -fdata-sections` and `--gc-sections`, the linker traces references from the entry point and discards any section not reachable (dead code elimination). `KEEP()` prevents this for sections that are intentionally not directly referenced but are used by runtime iteration patterns. In the kernel, this applies to: (1) **Initcall tables**: function pointers in `.initcallN.init` are placed by macros, not referenced by symbol name. `do_initcalls()` iterates via address range (`__initcall_start` to `__initcall_end`). Without `KEEP()`, the linker would see no references to individual initcall entries and discard them — no drivers would initialize. (2) **Exception tables**: `.ex_table` entries for page fault fixups (e.g., `get_user()`). Referenced by binary search at runtime, not by direct symbol reference. (3) **Static tracepoints**: `.trace_events` section. If discarded, tracing wouldn't work. (4) This is especially critical with LTO, which enables more aggressive dead code elimination. Without `KEEP()` + LTO, large portions of the kernel's initialization infrastructure would be silently dropped.

---

## Summary

- vmlinux.lds.S: linker script defining section layout, addresses, symbols
- Sections: .head.text → .text → .rodata → .data → .init.* → .bss
- .init.text/.init.data: freed after boot (memory reclaimed)
- Initcall levels 0-7: ordered initialization; module_init() = level 7
- KEEP(): prevents linker from discarding unreferenced-but-needed sections
- System.map: build-time symbol table (address → name)
- kallsyms: runtime symbol table embedded in vmlinux, used by Oops/perf/ftrace

---

[Previous: GCC and Clang ←](Chapter_07_GCC_Clang.md) | [Next: Kernel Image Formats →](Chapter_09_Image_Formats.md)
