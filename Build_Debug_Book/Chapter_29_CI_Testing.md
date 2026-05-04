# Chapter 29: CI, Testing, and Fuzzing

## Learning Goals
- Understand kernel testing frameworks (kselftest, KUnit)
- Learn KernelCI infrastructure and automated testing
- Master syzkaller kernel fuzzer architecture
- Know static analysis integration in CI pipelines

---

## 1. Kernel Testing Frameworks

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌───────────────┬──────────────────────────────────────┐│
  │  │ Framework     │ Description                         ││
  │  ├───────────────┼──────────────────────────────────────┤│
  │  │ kselftest     │ Userspace test programs that exercise││
  │  │               │ kernel interfaces via syscalls.      ││
  │  │               │ Lives in tools/testing/selftests/    ││
  │  │               │ Tests: seccomp, net, bpf, mm, sched ││
  │  │               │                                      ││
  │  │               │ make -C tools/testing/selftests run_tests││
  │  │               │ make -C tools/testing/selftests/     ││
  │  │               │   TARGETS=bpf run_tests              ││
  │  ├───────────────┼──────────────────────────────────────┤│
  │  │ KUnit         │ In-kernel unit test framework.       ││
  │  │               │ Runs in kernel context (not userspace)││
  │  │               │ Tests: individual functions, helpers, ││
  │  │               │   data structures, parsers           ││
  │  │               │                                      ││
  │  │               │ ./tools/testing/kunit/kunit.py run   ││
  │  │               │ → builds UML kernel, runs tests,    ││
  │  │               │   reports pass/fail                  ││
  │  ├───────────────┼──────────────────────────────────────┤│
  │  │ LTP           │ Linux Test Project — comprehensive   ││
  │  │               │ syscall + POSIX compliance tests.    ││
  │  │               │ ~3000 test cases.                    ││
  │  ├───────────────┼──────────────────────────────────────┤│
  │  │ xfstests      │ Filesystem-specific test suite.      ││
  │  │               │ (ext4, xfs, btrfs, f2fs)            ││
  │  └───────────────┴──────────────────────────────────────┘│
  │                                                           │
  │  KUnit example:                                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ #include <kunit/test.h>                  │            │
  │  │                                          │            │
  │  │ static void test_list_add(struct kunit   │            │
  │  │     *test) {                             │            │
  │  │   struct list_head list, node;           │            │
  │  │   INIT_LIST_HEAD(&list);                 │            │
  │  │   INIT_LIST_HEAD(&node);                 │            │
  │  │                                          │            │
  │  │   list_add(&node, &list);               │            │
  │  │   KUNIT_EXPECT_FALSE(test,               │            │
  │  │     list_empty(&list));                  │            │
  │  │   KUNIT_EXPECT_PTR_EQ(test,              │            │
  │  │     list.next, &node);                   │            │
  │  │ }                                        │            │
  │  │                                          │            │
  │  │ static struct kunit_case my_cases[] = {  │            │
  │  │   KUNIT_CASE(test_list_add),             │            │
  │  │   {}                                     │            │
  │  │ };                                       │            │
  │  │ static struct kunit_suite my_suite = {   │            │
  │  │   .name = "my_list_tests",               │            │
  │  │   .test_cases = my_cases,                │            │
  │  │ };                                       │            │
  │  │ kunit_test_suite(my_suite);              │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. syzkaller — Kernel Fuzzer

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  syzkaller: coverage-guided kernel syscall fuzzer       │
  │  (by Google — finds hundreds of kernel bugs per year)   │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Architecture:                            │            │
  │  │                                          │            │
  │  │ ┌──────────┐   ┌────────────────────┐   │            │
  │  │ │ syz-manager│─→│ VM instances       │   │            │
  │  │ │ (host)   │  │ │ ┌──────────────┐  │   │            │
  │  │ │          │  │ │ │ syz-executor │  │   │            │
  │  │ │ Coverage │←─│ │ │ (runs syscalls│  │   │            │
  │  │ │ analysis │  │ │ │  in kernel)  │  │   │            │
  │  │ │          │  │ │ └──────────────┘  │   │            │
  │  │ │ Corpus   │  │ │ ┌──────────────┐  │   │            │
  │  │ │ mutation │  │ │ │ syz-fuzzer   │  │   │            │
  │  │ │          │  │ │ │ (generates   │  │   │            │
  │  │ │ Crash    │  │ │ │  programs)   │  │   │            │
  │  │ │ reporting│  │ │ └──────────────┘  │   │            │
  │  │ └──────────┘  │ └────────────────────┘   │            │
  │  │               │ (QEMU/GCE VMs)          │            │
  │  └───────────────┴──────────────────────────┘            │
  │                                                           │
  │  How it works:                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. Generates random syscall sequences   │            │
  │  │    (understands syscall grammar —        │            │
  │  │     syzlang descriptions)               │            │
  │  │                                          │            │
  │  │ 2. Executes in QEMU/GCE VMs with        │            │
  │  │    KASAN, KCSAN, UBSAN, lockdep enabled │            │
  │  │                                          │            │
  │  │ 3. Monitors KCOV (code coverage):        │            │
  │  │    if new code paths reached → keep the  │            │
  │  │    input, mutate further                 │            │
  │  │                                          │            │
  │  │ 4. Detects crashes:                     │            │
  │  │    - Kernel oops/panic                   │            │
  │  │    - KASAN reports                       │            │
  │  │    - KCSAN data races                    │            │
  │  │    - UBSAN reports                       │            │
  │  │    - Lockdep warnings                    │            │
  │  │    - Hung task / RCU stalls              │            │
  │  │                                          │            │
  │  │ 5. Minimizes reproducer:                │            │
  │  │    removes unnecessary syscalls until    │            │
  │  │    minimal crash program found           │            │
  │  │                                          │            │
  │  │ 6. Reports to syzbot dashboard:          │            │
  │  │    syzkaller.appspot.com                 │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  KCOV — kernel code coverage:                            │
  │  ┌──────────────────────────────────────────┐            │
  │  │ CONFIG_KCOV=y                            │            │
  │  │ CONFIG_KCOV_INSTRUMENT_ALL=y             │            │
  │  │                                          │            │
  │  │ Per-thread coverage via /sys/kernel/      │            │
  │  │   debug/kcov (mmap interface)            │            │
  │  │                                          │            │
  │  │ Records: which basic blocks were         │            │
  │  │ executed during a syscall                │            │
  │  │ → guides fuzzer to explore new paths     │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. KernelCI

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  KernelCI: community kernel CI infrastructure           │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Pipeline:                                │            │
  │  │ 1. Monitor git repos (mainline, stable,  │            │
  │  │    next, subsystem trees)                │            │
  │  │ 2. Build: many configs × architectures   │            │
  │  │    (x86, ARM, ARM64, MIPS, RISC-V, ...)  │            │
  │  │ 3. Boot test: boot on real hardware      │            │
  │  │    (LAVA lab) or QEMU                    │            │
  │  │ 4. Test: kselftest, LTP, igt (GPU)       │            │
  │  │ 5. Report: dashboard, email, IRC         │            │
  │  │                                          │            │
  │  │ Other CI efforts:                        │            │
  │  │ - 0-day/Intel: build + boot + kselftest  │            │
  │  │ - Red Hat CKI: enterprise-focused        │            │
  │  │ - Google syzbot: continuous fuzzing       │            │
  │  │ - Linaro LKFT: functional testing        │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How would you set up continuous testing for a kernel driver you maintain?**
**A:** Multi-layered approach: (1) **Unit tests (KUnit)**: write tests for pure functions in your driver (parsers, state machines, protocol handlers). Run with `kunit.py run` — builds a UML kernel, executes tests in seconds, no hardware needed. Add to CI as first gate. (2) **Integration tests (kselftest)**: write tests in `tools/testing/selftests/` that exercise your driver through its userspace interface (ioctl, sysfs, devfs). Test normal operation, error paths, edge cases. Run in QEMU with your driver built-in. (3) **Build matrix**: compile your driver across architectures (x86_64, ARM64, ARM32, RISC-V) and configs (defconfig, allmodconfig, allyesconfig). `allmodconfig` enables most options and catches missing includes and config dependencies. (4) **Sanitizer builds**: separate CI jobs with KASAN, KCSAN, UBSAN, lockdep enabled — each catches different bug classes. Run your test suite under each. (5) **Static analysis**: `make C=1 M=your/driver/` (Sparse), `make coccicheck M=your/driver/` (Coccinelle), `smatch` for additional pattern checks. (6) **Fuzzing**: if your driver has a complex userspace interface, write syzkaller descriptions (`.txt` files in `sys/linux/`) and run syzkaller overnight — it will find edge cases you'd never think of. (7) **Hardware-in-the-loop**: for drivers that need real hardware (not mock-able), use LAVA or custom test rigs with actual devices.

---

## Summary

- kselftest: userspace tests for kernel interfaces; tools/testing/selftests/ — ~50 subsystems
- KUnit: in-kernel unit tests; runs in UML; fast, no hardware needed
- syzkaller: coverage-guided syscall fuzzer; finds hundreds of bugs/year with KASAN+KCSAN
- KCOV: per-thread kernel code coverage — guides fuzzer to new paths
- KernelCI: community CI — build × arch, boot on LAVA labs, run test suites
- CI pipeline: KUnit → kselftest → sanitizer builds → static analysis → fuzzing

---

[Previous: GDB and KGDB ←](Chapter_28_GDB_KGDB.md) | [Next: Interview Preparation →](Chapter_30_Interview_Prep.md)
