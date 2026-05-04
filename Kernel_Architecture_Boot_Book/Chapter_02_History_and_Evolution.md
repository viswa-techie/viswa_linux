# Chapter 2: History and Evolution of the Linux Kernel

## Learning Goals
- Trace the lineage from Unix to Linux
- Understand key milestones in Linux kernel development
- Grasp how kernel subsystems evolved over 30+ years
- Understand the Linux development model and release process

---

## 2.1 Early Unix Kernel Architecture

The Linux kernel descends from the Unix tradition. Understanding Unix is essential to understanding Linux.

```
Timeline of Unix Evolution:

1969  ──── Unics (Ken Thompson, Dennis Ritchie @ Bell Labs)
            │     Written in assembly for PDP-7
1971  ──── Unix V1 (PDP-11)
            │
1973  ──── Unix rewritten in C (Dennis Ritchie)
            │     ← This decision shaped ALL modern OS development
1979  ──── Unix V7 (last common ancestor)
            │
      ┌─────┴──────────────┐
      │                    │
1983  BSD                  AT&T System III/V
      │                    │
      ├── 4.2BSD (TCP/IP)  ├── System V Release 4 (SVR4)
      │                    │
      ├── FreeBSD          ├── Solaris
      ├── OpenBSD          ├── HP-UX
      ├── NetBSD           ├── AIX
      │                    │
      └── macOS (Darwin)   └── (commercial Unix declining)
```

Key Unix design principles that Linux inherited:

| Principle | Description |
|-----------|-------------|
| **Everything is a file** | Devices, pipes, sockets accessed via file descriptors |
| **Small, composable tools** | Each program does one thing well |
| **Plain text interfaces** | Configuration and IPC via text streams |
| **Hierarchical file system** | Single root `/`, mount points |
| **Process model** | `fork()` + `exec()` paradigm |
| **Shell as user interface** | Command-line interpreter, scripting |

---

## 2.2 Creation of the Linux Kernel

```
1991: The Birth of Linux

┌─────────────────────────────────────────────────────────┐
│  August 25, 1991 — Linus Torvalds posts to             │
│  comp.os.minix newsgroup:                               │
│                                                         │
│  "I'm doing a (free) operating system                   │
│   (just a hobby, won't be big and professional          │
│    like gnu) for 386(486) AT clones."                   │
│                                                         │
│  September 17, 1991 — Linux 0.01 released               │
│  - 10,239 lines of C code                              │
│  - Only ran on i386                                     │
│  - Needed MINIX to compile                              │
│  - Basic process management, memory, file system        │
└─────────────────────────────────────────────────────────┘
```

Why Linux was created:
- Linus wanted a free Unix-like OS for his 386 PC
- MINIX was educational but not open for modification (at the time)
- GNU project had all the tools (gcc, bash, coreutils) but no kernel (Hurd was delayed)
- Linux + GNU tools = complete free Unix-like system

```
The "GNU/Linux" combination:

GNU Project (1983, Richard Stallman)     Linux Kernel (1991, Linus Torvalds)
┌──────────────────────────┐            ┌──────────────────────────┐
│  gcc (compiler)          │            │  Process management      │
│  glibc (C library)       │            │  Memory management       │
│  bash (shell)            │            │  Device drivers          │
│  coreutils (ls, cp, mv)  │     +      │  File system (VFS)       │
│  make, gdb, emacs        │            │  Network stack           │
│  ... (everything except  │            │  System call interface   │
│       the kernel)        │            │                          │
└──────────────────────────┘            └──────────────────────────┘
                    ↓                              ↓
            ┌──────────────────────────────────────────────┐
            │         Complete Operating System             │
            │         "GNU/Linux" distribution              │
            └──────────────────────────────────────────────┘
```

---

## 2.3 Evolution of Linux Kernel Architecture

```
Linux Kernel Size Growth:

Version    Year    Lines of Code    Key Additions
─────────────────────────────────────────────────────────
0.01       1991        10,239       Basic i386, process, FS
1.0        1994       176,250       Networking, multiple FS
2.0        1996       787,000       SMP support, modules
2.2        1999     1,800,847       Improved SMP, IPv6
2.4        2001     3,377,902       USB, ext3, iptables
2.6        2003     5,929,913       O(1) scheduler, sysfs, NPTL
3.0        2011    14,500,000       Just version renumbering
4.0        2015    19,500,000       Live patching
5.0        2019    23,000,000       Energy-Aware Scheduling
6.0        2022    30,000,000+      EEVDF scheduler, Rust support
```

Major architectural evolution:

```
                 Kernel Architecture Evolution

v0.01 (1991)          v2.6 (2003)              v6.x (2022+)
┌──────────┐         ┌──────────────┐         ┌──────────────────┐
│ Single   │         │ Preemptible  │         │ Fully preemptible│
│ CPU only │         │ SMP kernel   │         │ PREEMPT_RT merge │
│ x86 only │         │ 20+ archs    │         │ 30+ archs        │
│ No SMP   │   →     │ sysfs model  │   →     │ Device tree      │
│ No modules│        │ LKM mature   │         │ Rust in kernel   │
│ Minimal  │         │ Netfilter    │         │ eBPF subsystem   │
│ drivers  │         │ ALSA, V4L2   │         │ io_uring         │
└──────────┘         └──────────────┘         └──────────────────┘
```

---

## 2.4 Major Kernel Milestones

| Version | Year | Milestone |
|---------|------|-----------|
| 0.01 | 1991 | First release, i386 only, 10K lines |
| 0.12 | 1992 | Adopted GPL license |
| 1.0 | 1994 | First "production" release, networking |
| 1.2 | 1995 | Alpha, SPARC, MIPS architecture support |
| 2.0 | 1996 | SMP support, loadable modules |
| 2.2 | 1999 | Better SMP, larger memory support |
| 2.4 | 2001 | USB, ext3, iptables, /proc improvements |
| 2.6 | 2003 | O(1) scheduler, preempt, NPTL, sysfs, udev |
| 2.6.12 | 2005 | Git adopted for kernel source management |
| 2.6.23 | 2007 | CFS (Completely Fair Scheduler) replaces O(1) |
| 2.6.29 | 2009 | ext4 becomes default, btrfs introduced |
| 3.0 | 2011 | Version renumbering (was 2.6.40) |
| 3.8 | 2013 | ARM multiplatform support |
| 4.0 | 2015 | Live kernel patching |
| 4.18 | 2018 | Spectre/Meltdown mitigations mature |
| 5.0 | 2019 | Energy-Aware Scheduling (EAS) |
| 5.4 | 2019 | LTS, io_uring, kernel lockdown |
| 5.6 | 2020 | WireGuard merged, time64 |
| 5.15 | 2021 | NTFS3 driver, LTS |
| 6.1 | 2022 | Initial Rust support in kernel, LTS |
| 6.6 | 2023 | EEVDF scheduler replaces CFS, LTS |
| 6.12 | 2024 | PREEMPT_RT fully merged (after 20 years!) |

---

## 2.5 Evolution of Kernel Subsystems

### Scheduler Evolution

```
Linux Scheduler History:

v0.01 — v2.4:  Simple round-robin / O(n) scheduler
                └── Scanned ALL tasks to pick next → poor scalability

v2.6.0 (2003):  O(1) Scheduler (Ingo Molnar)
                └── Constant-time pick-next using priority arrays
                └── Problem: poor interactive response

v2.6.23 (2007): CFS — Completely Fair Scheduler (Ingo Molnar)
                └── Red-black tree sorted by virtual runtime
                └── Proportional fair sharing
                └── Excellent interactive performance

v6.6 (2023):    EEVDF — Earliest Eligible Virtual Deadline First
                └── Replaces CFS core algorithm
                └── Better latency guarantees
                └── Eliminates need for latency-nice hacks
```

### Memory Management Evolution

```
Early Linux:        Simple page allocator
v2.0:               Slab allocator (Bonwick)
v2.6:               SLUB allocator (default), SLOB for embedded
v2.6.38:            Transparent Huge Pages (THP)
v3.11:              zswap (compressed swap cache)
v4.20:              Multi-generational LRU preparation
v6.1:               MGLRU (Multi-Gen LRU) merged — better page reclaim
```

### File System Evolution

```
ext2 (1993) → ext3 (2001, journaling) → ext4 (2008, extents, 1EB)
                                              ↑ default for most distros

Parallel development:
  XFS (SGI, 1994 → Linux 2001)  — high-performance, large files
  Btrfs (2009)                   — COW, snapshots, checksums
  F2FS (Samsung, 2012)           — flash-friendly, Android default
  bcachefs (2023)                — COW, compression, encryption
```

---

## 2.6 Linux Kernel Development Model

```
Linux Kernel Release Cycle:

 Merge Window (2 weeks)          RC Phase (6-8 weeks)        Release
 ┌───────────────────┐  ┌───────────────────────────────┐  ┌──────┐
 │ New features      │  │ rc1  rc2  rc3 ... rc7  rc8    │  │ v6.N │
 │ merged from       │→ │ Bug fixes only                │→ │      │
 │ subsystem trees   │  │ No new features               │  │      │
 └───────────────────┘  └───────────────────────────────┘  └──┬───┘
                                                              │
                                                    Merge window for
                                                    v6.N+1 opens
```

```
Patch Flow:

Developer
   │
   ▼
Subsystem Maintainer (e.g., drivers/gpu → Dave Airlie)
   │
   ▼
linux-next (integration testing tree)
   │
   ▼
Linus's tree (mainline) — during merge window
   │
   ▼
Stable releases (Greg KH) — backported fixes
   │
   ▼
LTS releases (maintained for 2-6 years)
   │
   ▼
Vendor kernels (Android, Ubuntu, Red Hat)
```

LTS (Long-Term Support) kernels:

| LTS Kernel | Release | EOL | Used By |
|-----------|---------|-----|---------|
| 5.4 | Nov 2019 | Dec 2025 | Android 11, many embedded |
| 5.10 | Dec 2020 | Dec 2026 | Android 12, Debian 11 |
| 5.15 | Oct 2021 | Dec 2027 | Android 13, Ubuntu 22.04 |
| 6.1 | Dec 2022 | Dec 2028 | Android 14, Debian 12 |
| 6.6 | Oct 2023 | Dec 2029 | Android 15, Ubuntu 24.04 |

---

## Kernel Source References

| File/Directory | Content |
|---------------|---------|
| `MAINTAINERS` | List of all subsystem maintainers |
| `COPYING` | GPL v2 license |
| `Documentation/process/` | Kernel development process docs |
| `Documentation/process/submitting-patches.rst` | How to submit patches |
| `scripts/get_maintainer.pl` | Find maintainer for a file |

---

## OS Comparison

| Aspect | Linux | Windows | macOS | QNX |
|--------|-------|---------|-------|-----|
| First release | 1991 | 1993 (NT 3.1) | 2001 (OS X) | 1982 |
| Source model | Open source (GPL v2) | Proprietary | Partially open (Darwin) | Proprietary |
| Development | Community + companies | Microsoft | Apple | BlackBerry/QNX |
| Architectures | 30+ | x86, ARM | x86, ARM (Apple Silicon) | x86, ARM, MIPS, PPC, SH |
| Kernel lines | ~30M | ~50M (est.) | ~7M (XNU) | ~150K |
| Release cycle | ~9 weeks | ~6 months | ~12 months | Varies |

---

## Interview Questions

**Q1: Why was the Linux kernel written in C and not C++?**
A: C provides direct hardware access, minimal runtime overhead, and was the established systems programming language from Unix. Linus considered C++ too complex for kernel code — its exceptions, RTTI, and implicit constructors add hidden costs. Note: Rust was added as a second language in kernel 6.1 for memory safety.

**Q2: What is the difference between a stable and LTS kernel?**
A: Stable kernels receive bug fixes for the current release cycle (~3 months). LTS (Long-Term Support) kernels receive critical fixes for 2-6 years. Embedded and Android products typically use LTS kernels because they need extended support without frequent upgrades.

**Q3: How does the Linux kernel development model ensure quality?**
A: Through a hierarchical review process: patches go through subsystem maintainers → integration testing in linux-next → merge window into Linus's tree → RC (release candidate) period with bug-fix only policy. Each patch is reviewed by multiple maintainers before merging.

**Q4: What was the significance of the O(1) → CFS scheduler transition?**
A: The O(1) scheduler (2003) solved scalability (constant-time pick-next) but was poor at interactive workloads. CFS (2007) used a red-black tree with virtual runtime to provide proportional fair sharing — dramatically improving desktop responsiveness while maintaining server performance.

---

## Summary

- Linux descends from the Unix tradition, inheriting "everything is a file" and the process model
- Created in 1991 by Linus Torvalds as a hobby project, now 30M+ lines of code
- Key milestones: SMP (2.0), modules, O(1) scheduler (2.6), CFS (2.6.23), EEVDF (6.6), Rust (6.1), PREEMPT_RT merge (6.12)
- Subsystems evolved independently: scheduler, MM, file systems, networking
- Development follows a ~9-week release cycle with merge window + RC stabilization
- LTS kernels are critical for embedded/Android where long-term stability matters

---

*Next: [Chapter 3 — Hardware Architecture Overview](Chapter_03_Hardware_Architecture.md)*
