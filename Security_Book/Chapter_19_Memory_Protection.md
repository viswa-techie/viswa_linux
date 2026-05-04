# Chapter 19: Memory Protection Mechanisms (NX/SMEP/SMAP/KPTI)

## Learning Goals
- Understand NX bit and W^X enforcement
- Know SMEP and SMAP CPU protections
- Understand KPTI (Kernel Page Table Isolation) and Meltdown mitigation
- Know how these protections work together

---

## 19.1 NX Bit (No-eXecute) / W^X

```
NX: Mark memory pages as non-executable.
Prevents code execution from data regions (stack, heap).

Without NX:
  Attacker writes shellcode to stack → jumps to it → code runs

With NX:
  Stack is marked non-executable → jump to stack → CPU fault → killed

  Page table entry bit 63 (x86_64): NX bit
    0 = executable
    1 = non-executable

W^X (Write XOR Execute):
  A page can be Writable OR Executable, NEVER both.

  ┌──────────────────────────────────────────────────┐
  │  Process Memory Layout with NX                    │
  │                                                   │
  │  .text (code)         R-X   (read + execute)      │
  │  .rodata              R--   (read only)            │
  │  .data                RW-   (read + write)         │
  │  .bss                 RW-   (read + write)         │
  │  heap                 RW-   (read + write)         │
  │  stack                RW-   (read + write)         │
  │  mmap regions         RW- or R-X (per mapping)     │
  │                                                   │
  │  No region is RWX (writable + executable)          │
  └──────────────────────────────────────────────────┘

Linux kernel enforcement:
  CONFIG_STRICT_KERNEL_RWX=y    Kernel code R-X, data RW-
  CONFIG_STRICT_MODULE_RWX=y    Modules: same separation
  Prevents writing to kernel text or executing kernel data
```

---

## 19.2 SMEP (Supervisor Mode Execution Prevention)

```
SMEP: CPU prevents kernel from executing user-space code.

Attack: ret2usr (return to userspace)
  1. Exploit kernel bug to control instruction pointer
  2. Point IP to attacker's user-space code
  3. Kernel executes user-space code with kernel privileges

  Without SMEP:
    Kernel executes user-space code → full compromise

  With SMEP:
    CPU detects user-page execution in supervisor mode → #PF fault

  ┌────────────────────────────────────────────────┐
  │  CPU CR4 register:                              │
  │    Bit 20: SMEP                                 │
  │    When set: any attempt to execute code from   │
  │    a page with User bit set causes page fault   │
  │    if CPU is in supervisor mode (ring 0).       │
  └────────────────────────────────────────────────┘

  Attack flow:
    ┌──────────┐   ret to    ┌──────────┐
    │ Kernel   │ ──────────► │ User     │
    │ (ring 0) │  0x7fff...  │ memory   │
    └──────────┘             └──────────┘

    Without SMEP: User code runs as kernel → game over
    With SMEP:    CPU fault → kernel oops → attack blocked

  Check SMEP support:
    grep smep /proc/cpuinfo
    dmesg | grep SMEP
```

---

## 19.3 SMAP (Supervisor Mode Access Prevention)

```
SMAP: CPU prevents kernel from reading/writing user-space memory.

  Why: Kernel bugs often read/write user-space pointers.
  Without SMAP: kernel can freely access user memory → data leak/corruption
  With SMAP: kernel access to user pages → fault

  ┌────────────────────────────────────────────────┐
  │  CPU CR4 register:                              │
  │    Bit 21: SMAP                                 │
  │    When set: supervisor mode cannot read/write  │
  │    pages with User bit set.                     │
  │                                                 │
  │    Exception: EFLAGS.AC flag                    │
  │    stac (set AC flag)  → temporarily allow      │
  │    clac (clear AC flag) → re-enable SMAP        │
  └────────────────────────────────────────────────┘

  Kernel code that NEEDS to access user memory:
    copy_from_user() / copy_to_user()
    get_user() / put_user()

  These functions temporarily disable SMAP:
    stac;                    /* Allow user access */
    /* copy data */
    clac;                    /* Re-enable SMAP */

  ┌──────────────────────────────────────────────┐
  │  Kernel code:                                 │
  │                                               │
  │  /* WRONG — would fault with SMAP */          │
  │  char *p = (char *)user_ptr;                  │
  │  value = *p;   ← PAGE FAULT (SMAP)            │
  │                                               │
  │  /* CORRECT — uses proper accessor */         │
  │  get_user(value, user_ptr);                   │
  │    internally: stac → read → clac             │
  └──────────────────────────────────────────────┘

  SMEP + SMAP together:
    Kernel cannot execute user code (SMEP)
    Kernel cannot read/write user data (SMAP, except via copy_*_user)
    Closes both ret2usr and data-only attacks from user memory
```

---

## 19.4 KPTI (Kernel Page Table Isolation)

```
KPTI: Separate kernel and user page tables. Meltdown mitigation.

Meltdown attack (CVE-2017-5754):
  CPU speculative execution reads kernel memory from user space.
  Even though access faults, speculated data enters cache.
  Side-channel (cache timing) recovers the data.
  → Read entire kernel memory from unprivileged user process.

Without KPTI:
  ┌────────────────────────────────┐
  │  User process page table       │
  │                                │
  │  User memory: 0 - 0x7fff...   │  ← accessible
  │  Kernel memory: 0xffff...     │  ← mapped but not accessible
  │                                │
  │  Speculative execution reads   │
  │  kernel memory → Meltdown!     │
  └────────────────────────────────┘

With KPTI:
  ┌────────────────────────┐  ┌──────────────────────────┐
  │  User page table        │  │  Kernel page table        │
  │                         │  │                           │
  │  User memory: mapped    │  │  User memory: mapped      │
  │  Kernel: NOT mapped     │  │  Kernel: fully mapped     │
  │  (only tiny trampoline) │  │                           │
  │                         │  │                           │
  │  Speculative exec →     │  │  Used only in kernel mode │
  │  nothing to read!       │  │                           │
  └────────────────────────┘  └──────────────────────────┘

  On syscall entry:
    Switch from user page table → kernel page table
  On syscall exit:
    Switch from kernel page table → user page table

Performance impact:
  ~5% overhead on syscall-heavy workloads
  Mitigated by PCID (Process Context Identifiers) — avoid TLB flush

Config:
  CONFIG_PAGE_TABLE_ISOLATION=y       # Enable KPTI
  Boot: pti=on/off/auto             # Control at boot
  nopti                              # Disable KPTI
```

---

## 19.5 Stack Protection

```
Multiple stack protections:

  1. Stack Canaries (Stack Protector):
     Random value placed between local vars and return address.
     Overflow overwrites canary → detected before return → abort.

     CONFIG_STACKPROTECTOR=y
     CONFIG_STACKPROTECTOR_STRONG=y   (protects more functions)

     ┌──────────────────────────┐
     │  High address             │
     │  Return address           │ ← attacker target
     │  CANARY VALUE             │ ← overflow detected here
     │  Local variables          │ ← buffer overflow starts
     │  Low address              │
     └──────────────────────────┘

  2. Shadow Call Stack (ARM64):
     Separate stack for return addresses only.
     Main stack overflow cannot corrupt return addresses.
     CONFIG_SHADOW_CALL_STACK=y

  3. Stack Clash Protection:
     Guard pages between stack and other mappings.
     Prevents stack growing into heap or mmap.
     CONFIG_VMAP_STACK=y — each thread gets individual VM-mapped stack
                           with guard pages

  4. STACKLEAK:
     Erase kernel stack on syscall return.
     Prevents leaking stack data to user space.
     CONFIG_GCC_PLUGIN_STACKLEAK=y
```

---

## 19.6 Combined Protection Matrix

```
How protections work together:

  ┌──────────┬─────────────────────────────────────────┐
  │ Attack   │ Mitigations                              │
  ├──────────┼─────────────────────────────────────────┤
  │ Stack    │ NX (no exec stack) + canary + ASLR +     │
  │ overflow │ shadow call stack                         │
  │          │                                          │
  │ ret2usr  │ SMEP (no exec user code from kernel)     │
  │          │                                          │
  │ Data     │ SMAP (kernel can't access user data)     │
  │ from user│                                          │
  │          │                                          │
  │ ret2libc │ ASLR + PIE (randomize addresses)         │
  │ / ROP    │                                          │
  │          │                                          │
  │ Meltdown │ KPTI (separate page tables)               │
  │          │                                          │
  │ Kernel   │ STRICT_KERNEL_RWX (code R-X, data RW-)   │
  │ code mod │                                          │
  │          │                                          │
  │ Info leak│ kptr_restrict + dmesg_restrict + KASLR    │
  │          │                                          │
  │ Stack    │ VMAP_STACK (guard pages) + STACKLEAK      │
  │ clash    │                                          │
  └──────────┴─────────────────────────────────────────┘

Defense in depth: no single mechanism is sufficient.
Kernel security requires ALL protections enabled together.
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| arch/x86/mm/pti.c | KPTI implementation |
| arch/x86/include/asm/pgtable.h | Page table flags (NX bit) |
| arch/x86/kernel/cpu/bugs.c | CPU bug mitigations (Spectre/Meltdown) |
| arch/x86/include/asm/smap.h | SMAP stac/clac operations |
| include/linux/uaccess.h | copy_{to,from}_user with SMAP |
| kernel/stackleak.c | STACKLEAK plugin |

---

## Interview Questions

**Q1: What is KPTI and why was it created?**
A: KPTI (Kernel Page Table Isolation) maintains separate page tables for user and kernel mode. In user mode, the page table maps only user memory plus a tiny kernel trampoline for syscall entry. The full kernel mapping exists only in the kernel page table, switched to on syscall entry. KPTI was created to mitigate the Meltdown vulnerability (CVE-2017-5754) — on affected CPUs, speculative execution could read kernel memory from user space via cache side-channels. With KPTI, kernel memory is simply not mapped in user page tables, so speculative execution has nothing to read. The performance cost is ~5%, reduced by using PCID to avoid TLB flushes on switches.

**Q2: How do SMEP and SMAP protect the kernel?**
A: SMEP (Supervisor Mode Execution Prevention) prevents the kernel from executing code located in user-space memory pages. This blocks "ret2usr" attacks where an exploit redirects kernel execution to attacker-controlled code in user space. SMAP (Supervisor Mode Access Prevention) prevents the kernel from reading or writing user-space memory except through sanctioned functions (copy_to_user/copy_from_user) which temporarily disable SMAP with stac/clac instructions. Together, they enforce: kernel code runs from kernel pages only (SMEP), and kernel accesses user data only through validated interfaces (SMAP). Both are CPU features controlled via CR4 register bits.

**Q3: Explain the W^X security principle.**
A: W^X (Write XOR Execute) means a memory page can be writable or executable, but never both simultaneously. This prevents an attacker from writing code to a writable region and then executing it (classic shellcode injection). The stack, heap, and data segments are marked RW- (writable, not executable). Code segments are R-X (executable, not writable). The NX bit in page table entries enforces this at the hardware level — the CPU faults if trying to execute code from a page with the NX bit set. In the kernel, CONFIG_STRICT_KERNEL_RWX ensures kernel .text is R-X and .data is RW-, preventing kernel code modification.

---

## Summary

- NX bit: hardware non-execute for data pages; W^X principle
- SMEP: CPU prevents kernel executing user-space code (ret2usr)
- SMAP: CPU prevents kernel reading/writing user data (except via copy_*_user)
- KPTI: separate user/kernel page tables (Meltdown mitigation, ~5% overhead)
- Stack canaries: detect buffer overflows before return
- Shadow call stack: separate stack for return addresses (ARM64)
- VMAP_STACK: guard pages detect stack clash
- STACKLEAK: erase kernel stack on syscall return
- All protections must be enabled together — defense in depth

---

Next: [Chapter 20 — Secure Boot](Chapter_20_Secure_Boot.md)
