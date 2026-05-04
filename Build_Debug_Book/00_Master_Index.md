# Linux Kernel Build System and Debugging Mastery
## Master Index — 30 Chapters

```
  ┌──────────────────────────────────────────────────────────┐
  │         BUILD SYSTEM + DEBUGGING: COMPLETE STACK         │
  │                                                           │
  │  ┌────────────────────────────────────────────────────┐  │
  │  │                  User Tools                        │  │
  │  │  make menuconfig │ perf │ ftrace │ crash │ gdb    │  │
  │  ├────────────────────────────────────────────────────┤  │
  │  │              Build Infrastructure                  │  │
  │  │  Kconfig │ Kbuild │ Makefiles │ scripts/          │  │
  │  ├────────────────────────────────────────────────────┤  │
  │  │             Compilation Pipeline                   │  │
  │  │  GCC/Clang │ LD │ objcopy │ modules │ DTB/ACPI   │  │
  │  ├────────────────────────────────────────────────────┤  │
  │  │              Tracing Infrastructure                │  │
  │  │  ftrace │ tracepoints │ kprobes │ perf_events     │  │
  │  ├────────────────────────────────────────────────────┤  │
  │  │              eBPF Subsystem                        │  │
  │  │  verifier │ JIT │ maps │ helpers │ BPF CO-RE      │  │
  │  ├────────────────────────────────────────────────────┤  │
  │  │              Debug Infrastructure                  │  │
  │  │  printk │ dynamic debug │ KASAN │ UBSAN │ lockdep │  │
  │  ├────────────────────────────────────────────────────┤  │
  │  │              Crash Analysis                        │  │
  │  │  kdump │ crash tool │ vmcore │ dmesg decode       │  │
  │  └────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────┘
```

---

## Part I: Kernel Build System (Chapters 1-6)

| # | Chapter | Key Topics |
|---|---------|------------|
| 1 | [Kconfig System](Chapter_01_Kconfig.md) | Kconfig language, menuconfig, .config, tristate, depends on, select, defconfig |
| 2 | [Kbuild System](Chapter_02_Kbuild.md) | obj-y/obj-m, ccflags, header dependencies, build stages, vmlinux linking |
| 3 | [Kernel Makefiles](Chapter_03_Makefiles.md) | Top-level Makefile, sub-makefiles, make targets, build variables, clean/mrproper |
| 4 | [Cross-Compilation](Chapter_04_Cross_Compilation.md) | ARCH, CROSS_COMPILE, toolchains, multi-arch, ARM/ARM64/RISC-V builds |
| 5 | [Modules: Build and Load](Chapter_05_Modules.md) | Out-of-tree modules, modprobe, depmod, module signing, DKMS, module parameters |
| 6 | [Device Tree and ACPI](Chapter_06_DT_ACPI.md) | DTS/DTB compilation, dtc, overlays, ACPI tables, firmware description |

## Part II: Compilation and Toolchain (Chapters 7-10)

| # | Chapter | Key Topics |
|---|---------|------------|
| 7 | [GCC and Clang for Kernel](Chapter_07_GCC_Clang.md) | Compiler flags, -O2, -Wall, LTO, CFI, gcc plugins, Clang advantages |
| 8 | [Linker Scripts and vmlinux](Chapter_08_Linker_vmlinux.md) | vmlinux.lds, sections, KEEP, initcall levels, System.map, kallsyms |
| 9 | [Kernel Image Formats](Chapter_09_Image_Formats.md) | vmlinux, bzImage, zImage, Image.gz, uImage, boot protocol, decompression |
| 10 | [Initramfs and Root FS](Chapter_10_Initramfs.md) | initrd vs initramfs, cpio archive, gen_init_cpio, switch_root, dracut/mkinitramfs |

## Part III: printk and Logging (Chapters 11-13)

| # | Chapter | Key Topics |
|---|---------|------------|
| 11 | [printk Architecture](Chapter_11_printk.md) | Ring buffer, log levels, rate limiting, printk_safe, console drivers, structured logging |
| 12 | [Dynamic Debug](Chapter_12_Dynamic_Debug.md) | pr_debug, dev_dbg, dyndbg control file, per-file/function/line enable, format flags |
| 13 | [dmesg and Kernel Logging](Chapter_13_dmesg_Logging.md) | dmesg, journalctl -k, /dev/kmsg, syslog, netconsole, serial console, pstore |

## Part IV: Tracing Infrastructure (Chapters 14-18)

| # | Chapter | Key Topics |
|---|---------|------------|
| 14 | [ftrace Architecture](Chapter_14_ftrace.md) | function tracer, function_graph, tracefs, trace-cmd, mcount/fentry, trampoline |
| 15 | [Tracepoints and Events](Chapter_15_Tracepoints.md) | TRACE_EVENT macro, static tracepoints, event format, filters, triggers, histogram |
| 16 | [kprobes and kretprobes](Chapter_16_Kprobes.md) | Dynamic instrumentation, int3 breakpoints, kretprobe trampoline, ftrace-based kprobes |
| 17 | [perf Events and PMU](Chapter_17_perf.md) | Hardware counters, perf stat/record/report, PMU architecture, sampling vs counting |
| 18 | [Perfetto and Trace Visualization](Chapter_18_Perfetto.md) | Perfetto + ftrace, trace_processor, SQL queries, Chrome trace format, custom data sources |

## Part V: eBPF (Chapters 19-22)

| # | Chapter | Key Topics |
|---|---------|------------|
| 19 | [eBPF Architecture](Chapter_19_eBPF_Arch.md) | BPF VM, instruction set, registers, verifier, JIT compiler, program types |
| 20 | [BPF Maps and Helpers](Chapter_20_BPF_Maps.md) | Hash/array/ring buffer maps, helper functions, BPF-to-BPF calls, tail calls |
| 21 | [BCC and bpftrace](Chapter_21_BCC_bpftrace.md) | BCC tools, bpftrace one-liners, scripts, probes, maps, printf, hist() |
| 22 | [BPF CO-RE and libbpf](Chapter_22_CORE_libbpf.md) | BTF, CO-RE relocations, libbpf API, skeleton, portability across kernels |

## Part VI: Sanitizers and Static Analysis (Chapters 23-25)

| # | Chapter | Key Topics |
|---|---------|------------|
| 23 | [KASAN and Memory Debugging](Chapter_23_KASAN.md) | Generic/SW-tag/HW-tag KASAN, shadow memory, use-after-free, out-of-bounds detection |
| 24 | [UBSAN, KCSAN, KFENCE](Chapter_24_Sanitizers.md) | Undefined behavior, concurrency sanitizer, memory error sampling, lockdep |
| 25 | [Static Analysis and Sparse](Chapter_25_Static_Analysis.md) | sparse __user/__kernel, Coccinelle semantic patches, smatch, compiler warnings |

## Part VII: Crash Analysis (Chapters 26-28)

| # | Chapter | Key Topics |
|---|---------|------------|
| 26 | [Oops, Panic, and Bug Analysis](Chapter_26_Oops_Panic.md) | Oops decode, call trace reading, addr2line, faddr2line, BUG/WARN macros |
| 27 | [kdump and crash Tool](Chapter_27_kdump_crash.md) | kexec, crash kernel, makedumpfile, crash commands, struct walking, backtrace |
| 28 | [Live Debugging with GDB](Chapter_28_GDB_KGDB.md) | KGDB, GDB scripts, lx-dmesg, lx-ps, vmlinux-gdb.py, QEMU+GDB, /proc/kcore |

## Part VIII: Advanced Topics and Interview (Chapters 29-30)

| # | Chapter | Key Topics |
|---|---------|------------|
| 29 | [Kernel CI and Testing](Chapter_29_CI_Testing.md) | KernelCI, kselftest, KUnit, syzkaller fuzzing, LKFT, 0-day bot, test infrastructure |
| 30 | [Interview Preparation](Chapter_30_Interview_Prep.md) | Comprehensive Q&A, debugging scenarios, build system walkthrough, tool selection |

---

## Prerequisites
- C programming and basic assembly
- Linux command line proficiency
- Kernel source tree familiarity
- Prior books: Kernel Architecture, Device Drivers recommended

## How to Use This Book
Each chapter contains:
1. **Learning Goals** — what you'll master
2. **Numbered Sections** — detailed explanations with ASCII diagrams and code
3. **Interview Questions** — Q&A pairs for interview preparation
4. **Summary** — key takeaways as bullet points
5. **Navigation** — Previous/Next chapter links

---

[Back to Main Index →](../README.md)
