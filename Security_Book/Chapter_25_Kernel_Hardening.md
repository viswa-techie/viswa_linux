# Chapter 25: Kernel Security Hardening

## Learning Goals
- Understand kernel hardening configuration options
- Know GCC plugins and compiler-based protections
- Understand sysctl security settings
- Know kernel self-protection mechanisms

---

## 25.1 Kernel Hardening Overview

```
Kernel hardening: Reduce attack surface and make exploitation harder.

  Three categories:
    1. Compile-time: CONFIG options, GCC plugins
    2. Boot-time: kernel command line parameters
    3. Runtime: sysctl settings

  ┌──────────────────────────────────────────────────────┐
  │  Kernel Hardening Layers                              │
  │                                                       │
  │  Compile-time protections:                            │
  │    STRICT_KERNEL_RWX, STACKPROTECTOR, FORTIFY_SOURCE  │
  │    INIT_STACK_ALL, RANDSTRUCT, STACKLEAK              │
  │                                                       │
  │  Memory protections:                                  │
  │    KASLR, SMEP, SMAP, KPTI, NX                       │
  │    SLAB_FREELIST_RANDOM, SHUFFLE_PAGE_ALLOCATOR       │
  │                                                       │
  │  Runtime restrictions:                                │
  │    kptr_restrict, dmesg_restrict, yama/ptrace_scope   │
  │    unprivileged_bpf_disabled, perf_event_paranoid     │
  │                                                       │
  │  Integrity:                                           │
  │    MODULE_SIG_FORCE, LOCKDOWN, IMA, EVM               │
  └──────────────────────────────────────────────────────┘
```

---

## 25.2 Compile-Time Hardening Options

```
Critical CONFIG options:

  Memory safety:
    CONFIG_STRICT_KERNEL_RWX=y     # Kernel code R-X, data RW-
    CONFIG_STRICT_MODULE_RWX=y     # Same for modules
    CONFIG_FORTIFY_SOURCE=y        # Detect buffer overflows in string ops
    CONFIG_INIT_STACK_ALL_ZERO=y   # Zero-initialize all stack variables
    CONFIG_INIT_ON_ALLOC_DEFAULT_ON=y  # Zero heap allocations
    CONFIG_INIT_ON_FREE_DEFAULT_ON=y   # Zero heap on free

  Stack protection:
    CONFIG_STACKPROTECTOR=y        # Stack canaries
    CONFIG_STACKPROTECTOR_STRONG=y # Canaries on more functions
    CONFIG_VMAP_STACK=y            # Guard pages around stacks
    CONFIG_GCC_PLUGIN_STACKLEAK=y  # Erase stack on syscall return

  Heap/slab protection:
    CONFIG_SLAB_FREELIST_RANDOM=y  # Randomize slab freelist
    CONFIG_SLAB_FREELIST_HARDENED=y # Harden freelist pointer
    CONFIG_SHUFFLE_PAGE_ALLOCATOR=y # Randomize page allocation

  Structure randomization:
    CONFIG_GCC_PLUGIN_RANDSTRUCT=y # Randomize struct member order
    # Makes struct layout attacks impossible

  Overflow detection:
    CONFIG_UBSAN=y                 # Undefined behavior sanitizer
    CONFIG_KASAN=y                 # Kernel address sanitizer (debug)

  ┌──────────────────────────────────────────────────────┐
  │  FORTIFY_SOURCE example:                              │
  │                                                       │
  │  memcpy(dst, src, len);                               │
  │  If compiler knows sizeof(dst) < len at compile time  │
  │  → BUILD ERROR                                        │
  │  If detected at runtime → BUG() / kernel panic        │
  │                                                       │
  │  Catches: buffer overflows in memcpy, strcpy, sprintf │
  │  Zero performance cost for non-buggy code             │
  └──────────────────────────────────────────────────────┘
```

---

## 25.3 GCC Security Plugins

```
Kernel GCC plugins: Custom compiler passes for security.

  CONFIG_GCC_PLUGINS=y  (enable plugin infrastructure)

  STACKLEAK:
    CONFIG_GCC_PLUGIN_STACKLEAK=y
    Erases kernel stack at end of every syscall.
    Prevents: info leaks of previous syscall's stack data.
    Performance: ~1% overhead.

  RANDSTRUCT:
    CONFIG_GCC_PLUGIN_RANDSTRUCT=y
    Randomizes order of struct members at compile time.
    Each kernel build has different struct layouts.
    Attacker can't predict field offsets.
    Works on structs with only function pointers or
    all structs with CONFIG_GCC_PLUGIN_RANDSTRUCT_FULL.

  LATENT_ENTROPY:
    CONFIG_GCC_PLUGIN_LATENT_ENTROPY=y
    Generates entropy during boot from code execution timing.
    Improves early boot randomness.

  STRUCTLEAK:
    CONFIG_GCC_PLUGIN_STRUCTLEAK=y
    Zero-initialize structs passed by reference.
    Prevents kernel stack info leaks to user space.
    Replaced by CONFIG_INIT_STACK_ALL_ZERO in newer kernels.
```

---

## 25.4 Sysctl Security Settings

```bash
# Kernel pointer and logging restrictions:
kernel.kptr_restrict = 2           # Hide kernel pointers
kernel.dmesg_restrict = 1          # Restrict dmesg to CAP_SYSLOG
kernel.printk = "3 3 3 3"         # Reduce console verbosity

# Process tracing restrictions (Yama):
kernel.yama.ptrace_scope = 2
  # 0: classic — any process can ptrace others (same UID)
  # 1: restricted — only parent can ptrace child
  # 2: admin-only — only CAP_SYS_PTRACE
  # 3: no ptrace (strictly disabled)

# Restrict BPF:
kernel.unprivileged_bpf_disabled = 1  # No BPF for unprivileged users
net.core.bpf_jit_harden = 2          # Harden BPF JIT (constant blinding)

# Performance monitoring:
kernel.perf_event_paranoid = 3        # No perf for unprivileged users

# Core dumps:
fs.suid_dumpable = 0                  # No core dumps for setuid binaries

# Message queue limits:
kernel.msgmax = 65536                 # Limit message size
kernel.msgmnb = 65536                 # Limit queue size

# SysRq restriction:
kernel.sysrq = 0                     # Disable SysRq (or limit to specific funcs)

# Network hardening (see Chapter 23):
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.tcp_syncookies = 1
net.ipv6.conf.all.accept_redirects = 0
```

---

## 25.5 IMA/EVM (Integrity Measurement Architecture)

```
IMA: Measure and optionally enforce file integrity.
EVM: Protect extended attributes from tampering.

IMA modes:
  Measure: Hash files on access, store in measurement list.
           TPM extends PCR with hash → supports remote attestation.
  Appraise: Verify file signature/hash before allowing execution.
            Blocks execution of tampered binaries.
  Audit: Log integrity violations.

  ┌──────────────────────────────────────────────────────┐
  │  IMA Flow:                                            │
  │                                                       │
  │  Open file → IMA hook → Check policy                  │
  │    ↓                                                  │
  │  Measure: compute hash, extend TPM PCR, log           │
  │  Appraise: verify against stored hash/signature       │
  │    ↓                                                  │
  │  Pass → allow access                                  │
  │  Fail → deny access (appraise mode)                   │
  └──────────────────────────────────────────────────────┘

  IMA policy example (boot parameter):
    ima_policy=tcb              # Trusted Computing Base policy
    ima_appraise=enforce        # Enforce file integrity

  EVM protects these extended attributes:
    security.ima     — IMA measurement/signature
    security.selinux — SELinux label
    security.SMACK64 — Smack label
    security.capability — File capabilities

    EVM HMAC: keyed hash of all protected xattrs.
    If any xattr is modified outside EVM, HMAC mismatch → detected.

Config:
  CONFIG_IMA=y
  CONFIG_IMA_APPRAISE=y
  CONFIG_EVM=y
```

---

## 25.6 Kernel Self-Protection Project (KSPP)

```
KSPP: Upstream effort to harden the kernel itself.

Key areas and status:
  ┌──────────────────────────────────────┬────────────┐
  │ Protection                          │ Status      │
  ├──────────────────────────────────────┼────────────┤
  │ STRICT_KERNEL_RWX                    │ Mainline    │
  │ STACKPROTECTOR_STRONG                │ Mainline    │
  │ FORTIFY_SOURCE                       │ Mainline    │
  │ KASLR                               │ Mainline    │
  │ VMAP_STACK                           │ Mainline    │
  │ SLAB_FREELIST_HARDENED               │ Mainline    │
  │ INIT_STACK_ALL_ZERO                  │ Mainline    │
  │ INIT_ON_ALLOC/FREE                  │ Mainline    │
  │ GCC_PLUGIN_RANDSTRUCT               │ Mainline    │
  │ GCC_PLUGIN_STACKLEAK                │ Mainline    │
  │ CFI (Control Flow Integrity)        │ Mainline (Clang) │
  │ Shadow Call Stack (ARM64)            │ Mainline    │
  │ FGKASLR (function-level)            │ In progress │
  │ Auto-bounds checking                 │ In progress │
  └──────────────────────────────────────┴────────────┘

CFI (Control Flow Integrity):
  CONFIG_CFI_CLANG=y
  Prevents diverting indirect function calls.
  Before: attacker overwrites function pointer → calls any function.
  With CFI: runtime check verifies target matches expected type.
  Violation → kernel panic.
  Requires Clang compiler.

Recommended hardened kernel config:
  Based on KSPP recommendations + distribution hardening guides.
  Distributions (Fedora, Ubuntu) enable most options by default.
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| security/integrity/ima/ | IMA implementation |
| security/integrity/evm/ | EVM implementation |
| scripts/gcc-plugins/ | GCC plugin framework |
| kernel/stackleak.c | STACKLEAK runtime |
| mm/kasan/ | Kernel Address Sanitizer |
| lib/fortify_kunit.c | FORTIFY_SOURCE tests |
| kernel/cfi.c | Control Flow Integrity |

---

## Interview Questions

**Q1: What are the most important kernel hardening CONFIG options?**
A: The critical options are: (1) `STRICT_KERNEL_RWX` — kernel code R-X, data RW-, prevents code injection. (2) `STACKPROTECTOR_STRONG` — stack canaries detect buffer overflows. (3) `FORTIFY_SOURCE` — compile/runtime detection of string function overflows. (4) `INIT_STACK_ALL_ZERO` — zero all stack variables, prevents info leaks. (5) `SLAB_FREELIST_HARDENED` — protects slab allocator freelists from corruption. (6) `VMAP_STACK` — guard pages around thread stacks. (7) `KASLR` — randomize kernel address space. (8) `MODULE_SIG_FORCE` — only load signed modules. (9) `CFI_CLANG` — control flow integrity for indirect calls. Together these make exploitation significantly harder by removing information leaks, detecting overflows, and preventing code injection.

**Q2: How does IMA/EVM provide file integrity?**
A: IMA (Integrity Measurement Architecture) computes hashes of files when they're accessed and extends TPM PCR registers with these hashes, creating an unforgeable measurement log. In appraise mode, IMA verifies that the file's hash or digital signature matches a stored reference before allowing execution — tampered binaries are blocked. EVM (Extended Verification Module) protects the extended attributes themselves (IMA hashes, SELinux labels, file capabilities) by computing an HMAC over all protected xattrs. If an attacker modifies any protected xattr, the EVM HMAC won't match, and the tampering is detected. Together, IMA/EVM provide: measured boot (what ran), runtime integrity (detect tampering), and attestation (prove to remote party).

**Q3: What is Control Flow Integrity (CFI) and why is it important?**
A: CFI prevents attackers from diverting code execution through corrupted function pointers. Without CFI, an attacker who can overwrite a function pointer (via buffer overflow, use-after-free) can redirect execution to any function. With CFI (CONFIG_CFI_CLANG), the compiler inserts runtime checks before every indirect function call to verify the target function has the expected type signature. If the check fails, the kernel panics. This blocks a major class of kernel exploits that rely on redirecting indirect calls to privilege-escalation functions. CFI requires the Clang compiler and adds minimal performance overhead. Android kernel 5.10+ enables CFI by default.

---

## Summary

- Compile-time: STRICT_KERNEL_RWX, FORTIFY_SOURCE, STACKPROTECTOR, INIT_STACK_ALL_ZERO
- GCC plugins: STACKLEAK (erase stack), RANDSTRUCT (randomize structs), LATENT_ENTROPY
- Sysctl: kptr_restrict, dmesg_restrict, yama/ptrace_scope, unprivileged_bpf_disabled
- Heap hardening: SLAB_FREELIST_HARDENED, SHUFFLE_PAGE_ALLOCATOR
- IMA/EVM: file integrity measurement, appraisal, and xattr protection
- CFI: Control Flow Integrity prevents indirect call exploitation (Clang)
- KSPP: upstream effort to systematically harden the Linux kernel
- Defense in depth: no single option is sufficient — enable all

---

Next: [Chapter 26 — Kernel Security Debugging](Chapter_26_Security_Debugging.md)
