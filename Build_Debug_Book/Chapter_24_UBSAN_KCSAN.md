# Chapter 24: UBSAN, KCSAN, and KFENCE

## Learning Goals
- Understand UBSAN (Undefined Behavior Sanitizer) for C UB detection
- Learn KCSAN (Kernel Concurrency Sanitizer) for data races
- Master KFENCE (Kernel Electric Fence) for production bug detection
- Know configuration and when to use each sanitizer

---

## 1. UBSAN — Undefined Behavior Sanitizer

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  UBSAN detects undefined behavior in C code:             │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Detected bugs:                           │            │
  │  │ - Signed integer overflow                │            │
  │  │ - Shift by >= type width                 │            │
  │  │ - Shift of negative value                │            │
  │  │ - Out-of-bounds array index              │            │
  │  │ - NULL pointer dereference               │            │
  │  │ - Misaligned pointer access              │            │
  │  │ - Unreachable code reached               │            │
  │  │ - Implicit type conversion overflow      │            │
  │  │ - Division by zero                       │            │
  │  │ - VLA bound not positive                 │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  How it works:                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Compiler (GCC/Clang) inserts runtime     │            │
  │  │ checks at operations that can be UB:     │            │
  │  │                                          │            │
  │  │ Original:  result = a + b;  // signed    │            │
  │  │ With UBSAN:                              │            │
  │  │   if (__builtin_add_overflow(a, b, &r))  │            │
  │  │     ubsan_handle_add_overflow(a, b);     │            │
  │  │   result = r;                            │            │
  │  │                                          │            │
  │  │ Overhead: very low (~3-5%)               │            │
  │  │ Can run in production (lightweight)      │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Configuration:                                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ CONFIG_UBSAN=y                           │            │
  │  │ CONFIG_UBSAN_SIGNED_WRAP=y  # overflow  │            │
  │  │ CONFIG_UBSAN_BOUNDS=y       # arrays    │            │
  │  │ CONFIG_UBSAN_SHIFT=y        # shifts    │            │
  │  │ CONFIG_UBSAN_ALIGNMENT=y    # alignment │            │
  │  │ CONFIG_UBSAN_UNREACHABLE=y  # unreachable│           │
  │  │                                          │            │
  │  │ Disable for specific files (Makefile):   │            │
  │  │ UBSAN_SANITIZE_noisy_file.o := n         │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Report example:                                         │
  │  ┌──────────────────────────────────────────┐            │
  │  │ UBSAN: shift-out-of-bounds in            │            │
  │  │   drivers/gpu/drm/foo.c:123:15           │            │
  │  │ shift exponent 32 is too large for       │            │
  │  │   32-bit type 'unsigned int'             │            │
  │  │ CPU: 0 PID: 45 Comm: kworker/0:1        │            │
  │  │ Call Trace:                              │            │
  │  │   ...                                    │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. KCSAN — Kernel Concurrency Sanitizer

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  KCSAN detects data races: concurrent accesses to       │
  │  shared memory where at least one is a write and        │
  │  no synchronization (locks, atomics, barriers).         │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Mechanism:                               │            │
  │  │                                          │            │
  │  │ 1. Watchpoint-based sampling:            │            │
  │  │    - Compiler instruments memory accesses│            │
  │  │    - On access: set a "watchpoint" on    │            │
  │  │      that address (via soft watchpoint)  │            │
  │  │    - Delay briefly (configurable, ~80μs) │            │
  │  │    - If another CPU accesses same addr   │            │
  │  │      during delay → DATA RACE detected  │            │
  │  │                                          │            │
  │  │ 2. Not 100% detection (sampling-based):  │            │
  │  │    - Run tests multiple times for        │            │
  │  │      increasing coverage                 │            │
  │  │    - ~5x slowdown (much less than TSAN)  │            │
  │  │                                          │            │
  │  │ 3. Understands kernel synchronization:   │            │
  │  │    - atomic_t, READ_ONCE, WRITE_ONCE     │            │
  │  │    - spin_lock, mutex, rcu_read_lock    │            │
  │  │    - data_race() annotation (intentional)│            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Configuration:                                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ CONFIG_KCSAN=y                           │            │
  │  │ CONFIG_KCSAN_STRICT=y  # stricter checks │            │
  │  │ CONFIG_KCSAN_REPORT_ONCE_IN_MS=0  # all │            │
  │  │                                          │            │
  │  │ Annotations:                             │            │
  │  │ WRITE_ONCE(x, val); // not a race       │            │
  │  │ val = READ_ONCE(x); // not a race       │            │
  │  │ data_race(x++);     // intentional race │            │
  │  │                      // (acknowledged)   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Report example:                                         │
  │  ┌──────────────────────────────────────────┐            │
  │  │ BUG: KCSAN: data-race in func_a / func_b│            │
  │  │                                          │            │
  │  │ write to 0xffff888012345678 of 4 bytes   │            │
  │  │ by task 1234 on cpu 0:                   │            │
  │  │   func_a+0x20/0x30                       │            │
  │  │                                          │            │
  │  │ read to 0xffff888012345678 of 4 bytes    │            │
  │  │ by task 5678 on cpu 3:                   │            │
  │  │   func_b+0x15/0x25                       │            │
  │  │                                          │            │
  │  │ value changed: 0x00000001 -> 0x00000002  │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. KFENCE — Kernel Electric Fence

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  KFENCE: low-overhead memory bug detector for           │
  │  PRODUCTION kernels (KASAN is too slow for production)  │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Mechanism:                               │            │
  │  │                                          │            │
  │  │ Fixed pool of guarded pages:             │            │
  │  │ ┌─────┬────────┬─────┬────────┬─────┐   │            │
  │  │ │GUARD│ object │GUARD│ object │GUARD│   │            │
  │  │ │page │  page  │page │  page  │page │   │            │
  │  │ │(no  │        │(no  │        │(no  │   │            │
  │  │ │map) │        │map) │        │map) │   │            │
  │  │ └─────┴────────┴─────┴────────┴─────┘   │            │
  │  │                                          │            │
  │  │ Guard pages are unmapped → any access    │            │
  │  │ causes page fault → detected immediately │            │
  │  │                                          │            │
  │  │ Object placed at LEFT or RIGHT edge of   │            │
  │  │ page → detects OOB in that direction     │            │
  │  │                                          │            │
  │  │ Freed objects: unmap the page → instant   │            │
  │  │ detection of any use-after-free          │            │
  │  │                                          │            │
  │  │ Sampling: only 1 in N allocations goes   │            │
  │  │ to KFENCE pool (default: one every       │            │
  │  │ 100ms timer interval)                    │            │
  │  │ Overhead: < 1%                           │            │
  │  │ Pool: 256 objects (fixed size)           │            │
  │  │                                          │            │
  │  │ Probabilistic: won't catch every bug,    │            │
  │  │ but over hours/days of running, catches  │            │
  │  │ many real bugs                           │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Configuration:                                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ CONFIG_KFENCE=y                          │            │
  │  │ CONFIG_KFENCE_SAMPLE_INTERVAL=100  # ms  │            │
  │  │ CONFIG_KFENCE_NUM_OBJECTS=255             │            │
  │  │                                          │            │
  │  │ Boot: kfence.sample_interval=50          │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Comparison:                                             │
  │  ┌──────────────┬───────────┬────────────┬─────────────┐ │
  │  │              │ KASAN     │ KFENCE     │ HW KASAN    │ │
  │  ├──────────────┼───────────┼────────────┼─────────────┤ │
  │  │ Overhead     │ 2-3x      │ < 1%       │ < 5%        │ │
  │  │ Detection    │ 100%      │ sampling   │ 100%        │ │
  │  │ Production   │ No        │ Yes        │ Yes (MTE)   │ │
  │  │ Memory cost  │ 12.5%     │ ~1MB fixed │ 3% (tags)   │ │
  │  │ Architecture │ All       │ All        │ ARM64       │ │
  │  └──────────────┴───────────┴────────────┴─────────────┘ │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How would you set up a CI pipeline to detect kernel memory and concurrency bugs?**
**A:** Build multiple kernel configurations, each with a specific sanitizer: (1) **KASAN build** (CONFIG_KASAN_GENERIC=y, CONFIG_KASAN_STACK=y): catches use-after-free, heap/stack out-of-bounds, double-free. Run with `kasan.fault=panic` to fail tests on first bug. Best for memory corruption. (2) **KCSAN build** (CONFIG_KCSAN=y, CONFIG_KCSAN_STRICT=y): catches data races. Run tests multiple times (races are timing-dependent). Best for concurrency bugs. (3) **UBSAN build** (CONFIG_UBSAN=y with all sub-options): catches signed overflow, invalid shifts, alignment issues. Low overhead, can combine with other sanitizers. (4) **KFENCE always-on** in all builds (CONFIG_KFENCE=y): negligible overhead, catches a subset of memory bugs opportunistically. (5) **lockdep build** (CONFIG_PROVE_LOCKING=y): detects potential deadlocks from lock ordering violations. Run the full test suite (kselftest, KUnit, syzkaller fuzzing) under each configuration. syzkaller is particularly effective because it generates random syscall sequences that trigger edge cases. CI matrix: at minimum KASAN + KCSAN + UBSAN + lockdep builds. For ARM64 targets with MTE hardware, use HW tag-based KASAN in production-like testing.

---

## Summary

- UBSAN: compiler-inserted UB checks (overflow, shift, alignment) — ~3-5% overhead
- KCSAN: watchpoint-based data race detector — sampling, ~5x slowdown, understands kernel sync
- KFENCE: production memory bug detector — guard pages, sampling, < 1% overhead
- KASAN: most thorough (shadow memory, 100% detection) but 2-3x slowdown
- HW tag-based KASAN (ARM64 MTE): hardware memory tagging, < 5%, production-viable
- CI: use all sanitizers in separate builds + syzkaller fuzzing for maximum coverage

---

[Previous: KASAN ←](Chapter_23_KASAN.md) | [Next: Sparse and Coccinelle →](Chapter_25_Sparse_Coccinelle.md)
