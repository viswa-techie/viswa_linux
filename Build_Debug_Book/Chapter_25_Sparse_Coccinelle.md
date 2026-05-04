# Chapter 25: Sparse and Coccinelle

## Learning Goals
- Understand Sparse static analysis for kernel code
- Learn Coccinelle semantic patching language
- Master annotation types (__user, __kernel, __iomem, __rcu)
- Know how to write SmPL scripts for automated refactoring

---

## 1. Sparse — Static Checker

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Sparse: semantic checker for C code (by Linus Torvalds)│
  │  Finds bugs that GCC/Clang can't — type annotation bugs │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Key annotations:                         │            │
  │  │                                          │            │
  │  │ __user   — pointer to user-space memory  │            │
  │  │            Must use copy_from_user() to  │            │
  │  │            access, not direct deref      │            │
  │  │                                          │            │
  │  │ __kernel — pointer to kernel-space memory│            │
  │  │            Direct dereference OK         │            │
  │  │                                          │            │
  │  │ __iomem  — pointer to MMIO/device memory │            │
  │  │            Must use readl/writel, not     │            │
  │  │            direct access                 │            │
  │  │                                          │            │
  │  │ __rcu    — RCU-protected pointer         │            │
  │  │            Must use rcu_dereference()    │            │
  │  │            to read                       │            │
  │  │                                          │            │
  │  │ __percpu — per-CPU pointer               │            │
  │  │            Must use per_cpu_ptr() etc.   │            │
  │  │                                          │            │
  │  │ __bitwise — type-checked integers        │            │
  │  │            Prevents endian mix-ups       │            │
  │  │            (__le32, __be32, __le16, etc.) │            │
  │  │                                          │            │
  │  │ __acquires(lock), __releases(lock)       │            │
  │  │ __must_hold(lock)                        │            │
  │  │            Lock context annotations      │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Usage:                                                  │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Check single file:                    │            │
  │  │ make C=1 drivers/net/e1000/e1000_main.o  │            │
  │  │ # C=1: check modified files only        │            │
  │  │ # C=2: check ALL files                  │            │
  │  │                                          │            │
  │  │ # Example warnings:                     │            │
  │  │ foo.c:45 warning: incorrect type in      │            │
  │  │   argument 1 (different address spaces)  │            │
  │  │   expected void *dst                     │            │
  │  │   got void __user *ubuf                  │            │
  │  │                                          │            │
  │  │ → means: direct memcpy from __user ptr   │            │
  │  │   instead of copy_from_user()            │            │
  │  │   → SECURITY BUG (missing check)        │            │
  │  │                                          │            │
  │  │ bar.c:78 warning: incorrect type in      │            │
  │  │   assignment (different base types)      │            │
  │  │   expected restricted __le32 [usertype]  │            │
  │  │   got unsigned int                       │            │
  │  │                                          │            │
  │  │ → means: assigning host-endian to        │            │
  │  │   little-endian field without cpu_to_le32│            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Coccinelle — Semantic Patch Engine

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Coccinelle (SmPL — Semantic Patch Language):            │
  │  Pattern matching and transformation on C code          │
  │  Used extensively in kernel development                 │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ SmPL basics:                             │            │
  │  │                                          │            │
  │  │ // Find and fix: kzalloc without check   │            │
  │  │ @@                                       │            │
  │  │ expression E;                            │            │
  │  │ @@                                       │            │
  │  │                                          │            │
  │  │ - E = kmalloc(...)                       │            │
  │  │ + E = kzalloc(...)                       │            │
  │  │                                          │            │
  │  │ // Replace deprecated API:               │            │
  │  │ @@                                       │            │
  │  │ expression dev, size;                    │            │
  │  │ @@                                       │            │
  │  │                                          │            │
  │  │ - kzalloc(size, GFP_KERNEL)              │            │
  │  │ + devm_kzalloc(dev, size, GFP_KERNEL)    │            │
  │  │                                          │            │
  │  │ // Find missing NULL checks:             │            │
  │  │ @@                                       │            │
  │  │ expression *E;                           │            │
  │  │ statement S;                             │            │
  │  │ @@                                       │            │
  │  │                                          │            │
  │  │ E = kmalloc(...);                        │            │
  │  │ + if (!E) return -ENOMEM;                │            │
  │  │ S                                        │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Usage:                                                  │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Run built-in kernel scripts:          │            │
  │  │ make coccicheck MODE=report              │            │
  │  │ make coccicheck MODE=patch               │            │
  │  │                                          │            │
  │  │ # On specific directory:                │            │
  │  │ make coccicheck MODE=report \             │            │
  │  │   M=drivers/net/                         │            │
  │  │                                          │            │
  │  │ # Custom script:                        │            │
  │  │ spatch --sp-file my_rule.cocci \          │            │
  │  │   --dir drivers/                         │            │
  │  │                                          │            │
  │  │ # Kernel includes 50+ cocci scripts in  │            │
  │  │ # scripts/coccinelle/ for:              │            │
  │  │ # - API check (deprecated functions)    │            │
  │  │ # - Free (missing free, double free)    │            │
  │  │ # - Null (missing null check)           │            │
  │  │ # - Locks (unbalanced lock/unlock)      │            │
  │  │ # - Iterator (wrong list iterator usage)│            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: What is the __user annotation and why is it critical for kernel security?**
**A:** `__user` is a Sparse address space annotation (defined as `__attribute__((address_space(1)))`) marking pointers that point to user-space memory. It's critical because: (1) **Security**: user-space pointers MUST be accessed through `copy_from_user()`/`copy_to_user()` which perform address validation (is the pointer actually in user space?) and handle page faults safely. Direct dereference of a user pointer (like `*uptr`) is a security vulnerability — an attacker could pass a kernel address as a "user" pointer and read/write kernel memory. On architectures with SMAP/SMEP (x86) or PAN (ARM64), direct user access faults at hardware level, but on older hardware it could succeed. (2) **Type safety**: Sparse catches mixing of `__user` and `__kernel` pointers at compile time — `warning: incorrect type in argument (different address spaces)`. Without Sparse, these bugs compile silently with GCC/Clang. (3) **Code correctness**: ensures `get_user()`/`put_user()` are used for single values and `copy_from/to_user()` for buffers. Missing `__user` annotations are a common source of kernel CVEs. Every syscall handler that receives user pointers should annotate them: `long sys_read(int fd, char __user *buf, size_t count)`. Sparse is run with `make C=1` and is recommended for all driver/subsystem development.

**Q2: When and how would you use Coccinelle in kernel development?**
**A:** Use cases: (1) **API migrations**: when a kernel API changes (e.g., `pci_alloc_consistent()` → `dma_alloc_coherent()`), a SmPL script can automatically transform all callers across the tree — `spatch` understands C semantics (expressions, types, control flow), not just text patterns. The kernel community does this routinely for tree-wide refactoring (hundreds of files in one patch series). (2) **Bug pattern detection**: `make coccicheck MODE=report` runs ~50 built-in scripts checking for common bugs: missing null checks after allocation, unbalanced lock/unlock, incorrect use of list iterators, double-free patterns, deprecated API usage. (3) **Code review**: before submitting patches, run `make coccicheck M=your/subsystem/` to catch issues automatically. (4) **Custom rules**: if you maintain a subsystem with specific patterns to enforce (e.g., "always call bar() after foo()"), write a `.cocci` script. Compared to grep/sed: Coccinelle understands C syntax — it matches `kmalloc(sizeof(struct foo), GFP_KERNEL)` regardless of whitespace, comments, or macro expansion. It handles multi-line expressions, nested function calls, and control flow dependencies.

---

## Summary

- Sparse: static checker for kernel address space annotations (__user, __iomem, __rcu, __bitwise)
- Run with `make C=1` (modified files) or `C=2` (all files)
- Catches security bugs: direct __user pointer dereference, endian mix-ups, RCU violations
- Coccinelle (SmPL): semantic patch language for C — pattern matching + transformation
- `make coccicheck`: runs 50+ built-in rules for common kernel bug patterns
- Both are essential parts of kernel development quality assurance

---

[Previous: UBSAN, KCSAN, KFENCE ←](Chapter_24_UBSAN_KCSAN.md) | [Next: Oops and Panic Analysis →](Chapter_26_Oops_Panic.md)
