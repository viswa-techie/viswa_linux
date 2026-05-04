# Chapter 22: Memory Security Mechanisms

## Chapter Overview

Memory security is a multi-layered defense against exploitation. This chapter covers hardware and software mechanisms in Linux: KPTI (Meltdown mitigation), KASLR/ASLR, NX/XN bits, stack protectors, SMEP/SMAP/PAN, memory tagging (MTE), and Control Flow Integrity.

---

## 22.1 Kernel Page Table Isolation (KPTI)

```
KPTI mitigates Meltdown (CVE-2017-5754): speculative execution
allows user process to read kernel memory via cache side-channels.

Solution: Maintain TWO sets of page tables per process:

BEFORE KPTI (single page table):
┌─────────────────────────────┐
│ Kernel mappings (full)      │ ← Userspace can speculatively
│ 0xFFFF800000000000+         │    access and leak via cache!
├─────────────────────────────┤
│ User mappings               │
│ 0x0000000000000000+         │
└─────────────────────────────┘

AFTER KPTI (two page tables):

Kernel mode PGD:               User mode PGD:
┌───────────────────┐          ┌───────────────────┐
│ Kernel (full)     │          │ Kernel (minimal)   │ ← Only entry/exit
│ All kernel code   │          │ stubs, IDT, GDT    │    trampolines
│ All kernel data   │          │ trampoline code    │
├───────────────────┤          ├───────────────────┤
│ User (full)       │          │ User (full)        │
│ All user pages    │          │ All user pages     │
└───────────────────┘          └───────────────────┘
CR3 = kernel PGD               CR3 = user PGD

On syscall entry:  switch CR3 to kernel PGD
On syscall exit:   switch CR3 to user PGD
On interrupt:      switch CR3 to kernel PGD
On iret:           switch CR3 to user PGD
```

```c
/* Kernel source: arch/x86/entry/entry_64.S */

/* Syscall entry: switch to kernel page table */
SYM_INNER_LABEL(entry_SYSCALL_64_safe_stack, SYM_L_GLOBAL)
    SWITCH_TO_KERNEL_CR3 scratch_reg=%rsp
    /* Now running with full kernel page table */

/* Syscall exit: switch back to user page table */
    SWITCH_TO_USER_CR3_NOSTACK scratch_reg=%rdi
    /* Now kernel memory is invisible to userspace */

/* PCID optimization: Avoid TLB flush on CR3 switch */
/* Use PCID tags to keep TLB entries for both user and kernel */
```

### KPTI Performance Impact

```
┌──────────────────────┬──────────────┬─────────────────────────┐
│ Workload             │ Impact       │ Notes                   │
├──────────────────────┼──────────────┼─────────────────────────┤
│ Syscall-heavy        │ 5-30%        │ CR3 switch overhead     │
│ I/O intensive        │ 10-20%       │ Frequent kernel entry   │
│ Compute-heavy        │ < 1%         │ Rarely enters kernel    │
│ With PCID            │ 1-5%         │ TLB preserved on switch │
│ Without PCID         │ 10-30%       │ Full TLB flush          │
│ Database workloads   │ 5-15%        │ Many small syscalls     │
└──────────────────────┴──────────────┴─────────────────────────┘

# Check if KPTI is active:
$ cat /sys/devices/system/cpu/vulnerabilities/meltdown
Mitigation: PTI

$ dmesg | grep "page tables isolation"
[    0.000000] Kernel/User page tables isolation: enabled
```

---

## 22.2 Address Space Layout Randomization (KASLR/ASLR)

```
ASLR randomizes memory layout to defeat return-to-libc and ROP attacks.

Without ASLR (predictable):        With ASLR (randomized):
┌────────────────┐ 0x7FFF...       ┌────────────────┐ 0x7FFF...
│ Stack          │ 0x7FFFFFFFE000  │ Stack          │ 0x7FFD3A2FE000
├────────────────┤                 ├────────────────┤
│                │                 │                │ random gap
├────────────────┤                 ├────────────────┤
│ mmap/libs      │ 0x7F0000000000  │ mmap/libs      │ 0x7F8BA3200000
├────────────────┤                 ├────────────────┤
│                │                 │                │ random gap
├────────────────┤                 ├────────────────┤
│ Heap (brk)     │ 0x555555600000  │ Heap (brk)     │ 0x5612A8C00000
├────────────────┤                 ├────────────────┤
│ Program text   │ 0x555555554000  │ Program text   │ 0x5612A6B54000
└────────────────┘ 0x0             └────────────────┘ 0x0

ASLR Entropy:
┌──────────────┬─────────────┬──────────────────────────┐
│ Region       │ Bits entropy│ Possible positions       │
├──────────────┼─────────────┼──────────────────────────┤
│ Stack        │ 22 bits     │ ~4 million               │
│ mmap base    │ 28 bits     │ ~268 million             │
│ PIE executable│ 28 bits    │ ~268 million             │
│ Heap         │ 13 bits     │ ~8,192                   │
│ VDSO         │ 11 bits     │ ~2,048                   │
└──────────────┴─────────────┴──────────────────────────┘
```

### KASLR (Kernel ASLR)

```
KASLR randomizes kernel text, modules, and data locations at boot.

# Kernel text base (normally 0xFFFFFFFF81000000):
# With KASLR: 0xFFFFFFFF81000000 + random_offset

# Check KASLR status:
$ cat /proc/cmdline | grep -o 'nokaslr\|kaslr'
# (empty means KASLR is enabled by default)

# Kernel randomizes these at boot:
1. Kernel text base address
2. Kernel module base address  
3. Physical memory mapping base (physmap)
4. vmalloc base address
5. vmemmap base address

# Each independently randomized:
# Physical mapping: PAGE_OFFSET      + random
# vmalloc:          VMALLOC_START    + random
# vmemmap:          VMEMMAP_START    + random
# Kernel text:      __START_KERNEL   + random
```

```bash
# View ASLR setting:
$ cat /proc/sys/kernel/randomize_va_space
2
# 0 = disabled, 1 = stack+mmap+VDSO, 2 = 1 + heap (brk)

# See randomized layout:
$ cat /proc/self/maps | head -20
5612a6b54000-5612a6b56000 r--p 00000000 ... /usr/bin/cat
7f8ba3200000-7f8ba3400000 r--p 00000000 ... /lib/libc.so.6
7ffd3a2de000-7ffd3a2ff000 rw-p 00000000 ... [stack]
```

---

## 22.3 NX Bit (No-Execute) / XN (Execute Never)

```
NX/XN: Page-level permission that prevents code execution from data pages.
Defeats classic buffer overflow → shellcode execution attacks.

Page Table Entry (x86_64):
Bit 63: NX (No-Execute)
  0 = page is executable
  1 = page is NOT executable — #PF if instruction fetch attempted

Memory Region Protection:
┌──────────────────┬─────┬───────┬────────┬───────────────────────┐
│ Region           │ R   │ W     │ X      │ Notes                 │
├──────────────────┼─────┼───────┼────────┼───────────────────────┤
│ .text (code)     │ Yes │ No    │ Yes    │ Read+Execute only     │
│ .rodata          │ Yes │ No    │ No     │ Read-only data        │
│ .data            │ Yes │ Yes   │ No     │ Read+Write, no exec   │
│ Stack            │ Yes │ Yes   │ No     │ NX stack              │
│ Heap             │ Yes │ Yes   │ No     │ NX heap               │
│ Kernel text      │ Yes │ No    │ Yes    │ Read+Execute          │
│ Kernel data      │ Yes │ Yes   │ No     │ NX for kernel data    │
│ Module text      │ Yes │ No    │ Yes    │ Module code            │
└──────────────────┴─────┴───────┴────────┴───────────────────────┘

W^X Policy: A page should NEVER be both Writable AND Executable.
If writable: mark NX (no execute)
If executable: mark read-only (no write)
```

```c
/* Kernel enforces NX on kernel data (post-boot): */
/* arch/x86/mm/init_64.c — mark_rodata_ro() */
/* After init: .rodata becomes read-only, .data becomes NX */

/* ARM64 equivalent: XN (Execute Never) bit in PTE */
/* PTE_UXN — User Execute Never */
/* PTE_PXN — Privileged Execute Never */
```

---

## 22.4 SMEP, SMAP, PAN

```
These prevent the kernel from accessing/executing userspace memory:

SMEP (Supervisor Mode Execution Prevention) — Intel x86:
  CPU fault if kernel tries to EXECUTE code from user pages.
  Defeats: ret2user attacks (kernel exploit jumps to user shellcode).

SMAP (Supervisor Mode Access Prevention) — Intel x86:
  CPU fault if kernel tries to READ/WRITE user pages without
  explicit STAC/CLAC instructions.
  Defeats: data-only attacks via user-controlled pointers.

PAN (Privileged Access Never) — ARM64:
  ARM equivalent of SMAP.
  Kernel cannot access user memory unless PAN is temporarily disabled.

┌──────────────────────────────────────────────────────────────────┐
│ Attack                    │ Blocked by │ Mechanism               │
├──────────────────────────────────────────────────────────────────┤
│ Kernel jumps to user code │ SMEP       │ #PF on user page exec   │
│ Kernel reads user data    │ SMAP/PAN   │ #PF on user page access │
│ via corrupted pointer     │            │                         │
│ User reads kernel data    │ KPTI       │ Kernel pages unmapped   │
│ speculative execution     │            │                         │
└──────────────────────────────────────────────────────────────────┘

/* Kernel must use copy_from_user()/copy_to_user() */
/* These temporarily disable SMAP/PAN: */
static inline unsigned long copy_from_user(void *to,
    const void __user *from, unsigned long n)
{
    stac();  /* Disable SMAP — allow user access */
    /* ... copy ... */
    clac();  /* Re-enable SMAP */
}
```

```bash
# Check CPU support:
$ grep -E "smep|smap" /proc/cpuinfo
flags: ... smep smap ...

# ARM64 PAN:
$ dmesg | grep PAN
[    0.000000] CPU features: detected: Privileged Access Never
```

---

## 22.5 Stack Protection

```
Multiple layers of stack protection:

1. Stack Canaries (gcc -fstack-protector):
   ┌────────────────┐ High addr
   │ Return address │ ← Overwrite target
   │ CANARY VALUE   │ ← Random value, checked before return
   │ Local vars     │   If modified: stack smashing detected!
   │ Buffer[]       │ ← Overflow starts here
   └────────────────┘ Low addr

   /* Kernel config: CONFIG_STACKPROTECTOR, CONFIG_STACKPROTECTOR_STRONG */
   /* Uses per-CPU random canary stored in %gs:0x28 (x86_64) */

2. Shadow Stack (Intel CET):
   ┌───────────┐     ┌──────────────┐
   │ Normal    │     │ Shadow Stack │
   │ Stack     │     │ (read-only)  │
   │           │     │              │
   │ ret addr  │ ──= │ ret addr     │ Must match!
   │ locals    │     │              │
   │ buffer    │     │              │
   └───────────┘     └──────────────┘
   Attacker can overwrite normal stack but not shadow stack.
   RET instruction compares both — fault if mismatch.

3. Guard Pages:
   ┌────────────────┐
   │ Stack          │ grows down
   │                │
   ├────────────────┤ ← GUARD PAGE (unmapped, causes #PF)
   │                │   Prevents stack overflow into heap
   │ Heap/other     │
   └────────────────┘
   
   /* Kernel thread stacks: VMAP_STACK uses guard pages */
   /* CONFIG_VMAP_STACK — each kernel thread stack has guard pages */
```

---

## 22.6 Memory Tagging Extension (MTE) — ARM64

```
MTE: Hardware memory safety — detects use-after-free and buffer overflow.

How it works:
1. Each 16-byte memory granule has a 4-bit TAG (stored in RAM metadata)
2. Each pointer carries a 4-bit TAG in bits [59:56] (TBI — Top Byte Ignore)
3. On memory access: CPU compares pointer tag with memory tag
4. Mismatch → fault (synchronous) or log (asynchronous)

Pointer:   [TAG|      Virtual Address            ]
            4b        56 bits
            ┌──┐
Memory:     │T1│  16 bytes of data
            ├──┤
            │T2│  16 bytes of data
            ├──┤
            │T1│  16 bytes of data
            └──┘

Example — Use-After-Free Detection:
1. malloc(32) → pointer with tag=5, memory tagged=5 → MATCH ✓
2. free(ptr)  → memory re-tagged=9 (random new tag)
3. Use ptr (tag=5) to access memory (tag=9) → MISMATCH → FAULT!

Example — Buffer Overflow Detection:
1. buf = malloc(32) → tag=3 for buf's 32 bytes
2. Adjacent memory → tag=7 (different)
3. buf[33] → pointer tag=3, memory tag=7 → MISMATCH → FAULT!

/* Linux kernel MTE support: */
/* CONFIG_ARM64_MTE — Kernel MTE support */
/* Used by KASAN (hardware mode) for kernel memory checking */

/* Userspace: Android uses MTE for heap protection */
/* /proc/<pid>/status → Mte_tags: ... */
```

---

## 22.7 Control Flow Integrity (CFI)

```
CFI prevents code-reuse attacks (ROP, JOP) by validating indirect calls.

Forward-edge CFI (indirect call targets):
  Before: call [register]  ← can jump anywhere
  After:  check target signature → call [register]
  If target not a valid function entry → trap

Backward-edge CFI (return addresses):
  Shadow stack / return address protection

Linux Kernel CFI:
- CONFIG_CFI_CLANG — Clang CFI for indirect calls
- Function type checking: indirect call must target function
  with matching type signature

┌─────────────────────────────────────────────────────────────┐
│ Without CFI:                                                │
│   Corrupted function pointer → attacker controls jump target│
│                                                             │
│ With CFI:                                                   │
│   Corrupted function pointer → wrong type signature         │
│   → CFI check fails → kernel panic (controlled crash)       │
│   Attacker cannot redirect execution                        │
└─────────────────────────────────────────────────────────────┘

/* Intel IBT (Indirect Branch Tracking): */
/* Hardware CFI — valid indirect branch targets must start with ENDBR64 */
/* Instruction at invalid target → #CP (control protection) fault */

/* ARM BTI (Branch Target Identification): */
/* Valid branch targets must start with BTI instruction */
```

---

## 22.8 Hardened Usercopy

```c
/*
 * copy_to_user() / copy_from_user() validation:
 * Prevents kernel bugs from leaking/corrupting arbitrary memory.
 */

/* CONFIG_HARDENED_USERCOPY checks: */
/* 1. Object is within a single slab allocation */
/* 2. Not crossing slab object boundaries */
/* 3. Not in kernel text segment */
/* 4. Not on the wrong kernel stack */
/* 5. Stack frame validation */

/* Example: Bug that hardened_usercopy catches */
char buf[64];
copy_to_user(ubuf, buf - 100, 200);  /* Overreads kernel stack! */
/* → usercopy: Kernel memory overread attempt detected! */
/* → kernel BUG/panic */
```

---

## 22.9 Security Feature Summary Matrix

```
┌─────────────────────┬────────┬────────┬──────────────────────────────┐
│ Feature             │ x86_64 │ ARM64  │ Defeats                      │
├─────────────────────┼────────┼────────┼──────────────────────────────┤
│ NX / XN bit         │ PTE.63 │ PTE.XN │ Stack/heap shellcode exec    │
│ ASLR / KASLR        │ Yes    │ Yes    │ Address prediction attacks   │
│ KPTI                │ Yes    │ Yes(*)│ Meltdown side-channel         │
│ SMEP / —            │ CR4.20 │ —      │ ret2user (exec user code)    │
│ SMAP / PAN          │ CR4.21 │ PAN    │ Kernel access to user mem    │
│ Stack canaries      │ %gs:28 │ sp_el0 │ Stack buffer overflow        │
│ Shadow stack        │ CET    │ GCS    │ ROP (return addr overwrite)  │
│ CFI                 │ IBT    │ BTI    │ JOP (indirect call hijack)   │
│ MTE                 │ —      │ MTE    │ Use-after-free, overflow     │
│ Guard pages         │ Yes    │ Yes    │ Stack overflow into adj mem  │
│ Hardened usercopy   │ Yes    │ Yes    │ Kernel info leak/corruption  │
│ W^X                 │ Yes    │ Yes    │ Code injection               │
└─────────────────────┴────────┴────────┴──────────────────────────────┘
(*) ARM64 default kernel mapping already isolated; KPTI for additional hardening
```

---

## OS Comparison: Memory Security

```
┌─────────────┬──────────────────────────────────────────────────────────┐
│ OS          │ Memory Security Features                                │
├─────────────┼──────────────────────────────────────────────────────────┤
│ Linux       │ KPTI, KASLR, NX, SMAP/PAN, CFI, MTE, stack canaries   │
│ Windows     │ KVA Shadow (=KPTI), ASLR, DEP (=NX), CFG, CET         │
│ macOS       │ KTRR, ASLR, W^X, PAC (ARM), stack guard               │
│ Android     │ Linux + MTE + HWASan + scudo hardened allocator         │
│ FreeBSD     │ KPTI, ASLR, W^X, SMAP, stack canaries                 │
│ iOS         │ PAC, PPL, KTRR, ASLR, MTE (A17+)                      │
└─────────────┴──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

1. **Q: What is KPTI and why was it introduced?**
   A: KPTI (Kernel Page Table Isolation) maintains separate page tables for user and kernel mode. Introduced to mitigate the Meltdown vulnerability (CVE-2017-5754) where speculative execution could leak kernel memory to userspace via cache side-channels. In userspace, only a minimal kernel trampoline is mapped, hiding kernel data.

2. **Q: How does ASLR protect against exploitation?**
   A: ASLR randomizes the base addresses of stack, heap, libraries, and executable code on each execution. An attacker cannot predict where specific code/data resides, defeating hard-coded address attacks (return-to-libc, ROP gadget chains). Combined with PIE (Position Independent Executable) for full randomization.

3. **Q: What is the W^X policy?**
   A: Write XOR Execute — a page should never be simultaneously writable AND executable. Enforced via NX bit (x86) / XN bit (ARM). Prevents code injection: writeable data pages can't be executed, executable code pages can't be overwritten.

4. **Q: What is ARM MTE and how does it detect use-after-free?**
   A: Memory Tagging Extension assigns a 4-bit tag to each 16-byte memory granule and corresponding pointer. On free, the memory tag is changed. A dangling pointer still carries the old tag — accessing with mismatched tags causes a fault, detecting use-after-free at the hardware level.

---

## Summary

1. **KPTI** isolates kernel page tables from userspace (Meltdown mitigation).
2. **ASLR/KASLR** randomizes memory layout (both user and kernel).
3. **NX/XN** prevents code execution from data pages (W^X policy).
4. **SMEP/SMAP/PAN** prevents kernel from executing/accessing user memory.
5. **Stack canaries + Shadow stack** detect stack corruption (buffer overflow, ROP).
6. **MTE** provides hardware memory safety on ARM64 (use-after-free, overflow).
7. **CFI/IBT/BTI** validates indirect call/branch targets.
8. Defense-in-depth: multiple overlapping protections needed.

---

*Next: [Chapter 23 — Memory Control Groups (cgroups)](Chapter_23_Memory_Cgroups.md)*
