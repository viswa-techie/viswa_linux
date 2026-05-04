# Chapter 18: Kernel Address Space Protection (ASLR/KASLR)

## Learning Goals
- Understand Address Space Layout Randomization (ASLR)
- Know Kernel ASLR (KASLR) and its implementation
- Understand entropy sources and randomization granularity
- Know bypasses and mitigations

---

## 18.1 ASLR Fundamentals

```
ASLR: Randomize memory layout to prevent exploit prediction.
Without ASLR, attacker knows where code and data reside.

Without ASLR (deterministic):      With ASLR (randomized):
  0x400000  text (code)              0x5a73f000  text
  0x600000  data                     0x7f2a8000  data
  0x7fff0000 stack                   0x7ff4c000  stack
  0x7f000000 libraries               0x7f8e1000  libraries
  0xb8000000 heap                    0x55d23000  heap

  Same every run → easy ROP/ret2libc   Different every run → hard to predict

ASLR randomizes:
  ┌──────────────────┬──────────────────────────────┐
  │ Region           │ Randomized?                   │
  ├──────────────────┼──────────────────────────────┤
  │ Stack            │ Yes — 22 bits entropy (4MB)   │
  │ mmap/libraries   │ Yes — 28 bits entropy (1TB)   │
  │ Heap (brk)       │ Yes — 13 bits entropy (8KB)   │
  │ Main executable  │ Yes (if PIE) — 28 bits        │
  │ VDSO             │ Yes                           │
  └──────────────────┴──────────────────────────────┘

Enable/check ASLR:
  cat /proc/sys/kernel/randomize_va_space
    0 = disabled
    1 = stack, mmap, VDSO randomized
    2 = 1 + heap randomized (full ASLR, default)
```

---

## 18.2 Position Independent Executables (PIE)

```
PIE: Compile executable as position-independent → enables text ASLR.

  Without PIE:
    Executable loaded at fixed address (0x400000 on x86_64).
    ASLR only randomizes stack/heap/libraries — NOT the code itself.
    Attacker can still use addresses from the executable for ROP.

  With PIE:
    Executable loaded at random address like shared libraries.
    ALL code addresses are randomized.

  gcc -pie -fPIE -o myapp myapp.c        # Compile as PIE
  gcc -no-pie -o myapp myapp.c           # Compile without PIE

  Check if PIE:
    file myapp
    # PIE:     "ELF 64-bit ... shared object"
    # Non-PIE: "ELF 64-bit ... executable"

    readelf -h myapp | grep Type
    # PIE:     Type: DYN (Shared object file)
    # Non-PIE: Type: EXEC (Executable file)

Most modern distros compile all packages as PIE by default.
```

---

## 18.3 KASLR (Kernel ASLR)

```
KASLR: Randomize kernel's own address in memory at boot.

Without KASLR:
  Kernel text always at 0xffffffff81000000
  Attacker knows where every kernel function is
  One info leak → complete kernel compromise

With KASLR:
  Kernel text at random offset (e.g., 0xffffffff81000000 + random)
  Randomization at boot time
  Different each boot

  Kernel regions randomized:
    - Kernel text/code (.text)
    - Kernel data (.data, .rodata)
    - Module load addresses
    - Physical memory mapping
    - vmalloc region
    - vmemmap

  Entropy: ~9 bits for text (512 possible positions)
           ~10 bits for physical memory and modules

Config:
  CONFIG_RANDOMIZE_BASE=y      # Enable KASLR
  Boot parameter: nokaslr      # Disable (for debugging)

  ┌────────────────────────────────────────────────┐
  │  Kernel Virtual Memory (x86_64)                 │
  │                                                 │
  │  Without KASLR:                                 │
  │    0xffffffff81000000  kernel text  ← KNOWN     │
  │    0xffff880000000000  direct map   ← KNOWN     │
  │    0xffffffffa0000000  modules      ← KNOWN     │
  │                                                 │
  │  With KASLR:                                    │
  │    0xffffffff81000000 + X  kernel text           │
  │    0xffff880000000000 + Y  direct map            │
  │    0xffffffffa0000000 + Z  modules               │
  │    X, Y, Z = random offsets at boot              │
  └────────────────────────────────────────────────┘
```

---

## 18.4 KASLR Implementation

```c
/* arch/x86/boot/compressed/kaslr.c */

/*
 * KASLR selects random offset at boot before decompression.
 * Uses multiple entropy sources for the random value.
 */

/* Entropy sources: */
static unsigned long get_random_long(void)
{
    /* Sources prioritized: */
    /* 1. RDRAND/RDSEED CPU instruction (hardware RNG) */
    /* 2. TSC (Time Stamp Counter) */
    /* 3. i8254 PIT timer */
    /* 4. ACPI timer */
    /* 5. Jitter-based entropy */
}

/* Slot-based randomization: */
/*
 * Kernel text randomized in 2MB aligned slots.
 * Available range: ~1GB of virtual space.
 * 9 bits of entropy: 512 possible positions.
 *
 * For modules: separate randomization in module region.
 * Physical mapping: yet another random offset.
 */

/* Boot flow: */
/*
 * 1. BIOS/UEFI → bootloader → compressed kernel
 * 2. kaslr.c: choose_random_location()
 * 3. choose_random_location() selects physical + virtual offset
 * 4. Kernel decompressed to randomized address
 * 5. Fixups applied for new addresses
 * 6. Boot continues at randomized address
 */
```

---

## 18.5 Information Leak Defenses

```
KASLR is defeated by any kernel address leak.
Multiple defenses prevent leaking kernel addresses:

  1. kptr_restrict:
     /proc/sys/kernel/kptr_restrict
       0 = show kernel pointers (default for root)
       1 = hide unless CAP_SYSLOG
       2 = always hide
     Affects: /proc/kallsyms, /proc/modules, %pK format

  2. dmesg_restrict:
     /proc/sys/kernel/dmesg_restrict = 1
     Only CAP_SYSLOG can read dmesg (kernel log may contain addrs)

  3. /proc/kallsyms:
     Shows all kernel symbol addresses.
     With kptr_restrict=1: shows 0000000000000000

  4. %pK format specifier:
     printk("%pK", ptr) → hashed or zero for unprivileged users
     vs %px which always shows the real address

  5. Restrict /proc/kcore and /dev/kmem:
     CONFIG_STRICT_DEVMEM=y — block access to kernel memory
     CONFIG_PROC_KCORE=n — remove /proc/kcore

  6. Stack traces:
     CONFIG_KALLSYMS=n (extreme — removes all symbols)
     Or restrict /proc/*/stack visibility

  ┌────────────────────────────────────────────────┐
  │  Attack chain vs defenses:                      │
  │                                                 │
  │  Step 1: Leak kernel address                    │
  │    Defense: kptr_restrict, dmesg_restrict,       │
  │            %pK, restrict /proc/kcore            │
  │                                                 │
  │  Step 2: Calculate kernel base                  │
  │    Defense: KASLR (unknown offset)              │
  │                                                 │
  │  Step 3: Use address for exploit                │
  │    Defense: SMEP, SMAP, KPTI (next chapter)     │
  └────────────────────────────────────────────────┘
```

---

## 18.6 KASLR Limitations and Bypass Techniques

```
Known limitations:

  1. Low entropy:
     ~9 bits for kernel text = 512 positions
     Brute-force possible if crash doesn't kill system
     vs user-space ASLR with 28+ bits

  2. Side channels:
     Spectre/Meltdown can leak kernel addresses despite KASLR
     Timing attacks on page table walks
     Branch predictor state side channels

  3. Single boot randomization:
     Address fixed until reboot
     Long-running systems: more time to leak

  4. Fine-grained KASLR (FGKASLR):
     Randomize at function granularity, not whole kernel
     CONFIG_FG_KASLR — much higher entropy
     Each function placed at random location

  5. KPTI complement:
     KPTI removes kernel mappings from user page tables
     Even if address is known, cannot access via Meltdown

Defense in depth:
  KASLR alone is weak. Combined with:
    KPTI + SMEP + SMAP + kptr_restrict + dmesg_restrict
  Makes exploitation significantly harder.
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| arch/x86/boot/compressed/kaslr.c | KASLR offset selection |
| arch/x86/mm/kaslr.c | Memory layout randomization |
| arch/x86/kernel/process.c | ASLR for user processes |
| fs/binfmt_elf.c | ELF loading with ASLR (randomize_stack_top) |
| mm/mmap.c | arch_mmap_rnd() for mmap randomization |
| include/linux/mm.h | Randomization constants |

---

## Interview Questions

**Q1: How does ASLR prevent exploitation?**
A: ASLR randomizes the memory layout of processes — stack, heap, libraries, and executable code (if PIE) are placed at random addresses each time a program runs. An attacker cannot predict where specific code gadgets (for ROP), libc functions (for ret2libc), or shellcode (on the stack) will be. On x86_64, user-space ASLR provides 28 bits of entropy for mmap/libraries (~256 million positions), 22 bits for the stack, and 13 bits for the heap. Combined with PIE, the executable itself is also randomized. Without an information leak, the attacker must guess the layout correctly, making reliable exploitation extremely difficult.

**Q2: What are the limitations of KASLR?**
A: KASLR has several limitations: (1) Low entropy — only ~9 bits for kernel text (512 positions), making brute-force feasible on systems that don't crash on failed attempts. (2) Per-boot randomization — the offset is fixed for the entire uptime, giving attackers time to discover it. (3) Side-channel attacks — Spectre, Meltdown, and timing side-channels can leak kernel addresses despite KASLR. (4) Information leaks — any kernel pointer leak (/proc/kallsyms, dmesg, printk formatting) defeats KASLR entirely. That's why KASLR must be combined with kptr_restrict, dmesg_restrict, KPTI, SMEP, and SMAP for meaningful security.

**Q3: Why is PIE important for ASLR?**
A: Without PIE (Position Independent Executable), the executable's .text segment is loaded at a fixed address (0x400000 on x86_64). ASLR only randomizes the stack, heap, and libraries, but the attacker can still use code gadgets from the main executable for ROP chains since those addresses are known. With PIE, the executable is compiled as position-independent (gcc -pie -fPIE), allowing the loader to place it at a random address just like shared libraries. This eliminates the last fixed-address code region, forcing the attacker to have an information leak before any exploitation. Modern distributions compile all packages as PIE by default.

---

## Summary

- ASLR randomizes user-space memory layout: stack, heap, mmap, executable (PIE)
- 28+ bits entropy for mmap; 22 bits for stack; 13 bits for heap
- PIE: necessary for executable code randomization
- KASLR: randomizes kernel code/data/module addresses at boot
- KASLR entropy is low (~9 bits) — must combine with other defenses
- Info leak defenses: kptr_restrict, dmesg_restrict, %pK, restrict /proc
- KASLR alone is weak — needs KPTI + SMEP + SMAP + leak prevention
- FGKASLR: function-level randomization for higher entropy

---

Next: [Chapter 19 — Memory Protection Mechanisms](Chapter_19_Memory_Protection.md)
