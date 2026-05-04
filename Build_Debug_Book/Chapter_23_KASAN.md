# Chapter 23: KASAN

## Learning Goals
- Understand Kernel Address Sanitizer architecture and shadow memory
- Learn KASAN modes: generic, software tag-based, hardware tag-based
- Master KASAN report interpretation and bug categories
- Know configuration, overhead, and integration with testing

---

## 1. KASAN Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  KASAN = Kernel Address Sanitizer                        │
  │  Detects memory bugs at runtime:                        │
  │  - Use-after-free                                       │
  │  - Out-of-bounds (heap, stack, global)                  │
  │  - Double-free                                          │
  │  - Invalid-free                                         │
  │                                                           │
  │  Shadow memory:                                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │                                          │            │
  │  │ Kernel memory     Shadow memory          │            │
  │  │ ┌──────────┐     ┌──────────┐            │            │
  │  │ │ 8 bytes  │ ──→ │ 1 byte   │            │            │
  │  │ │ of real  │     │ shadow   │            │            │
  │  │ │ memory   │     │ value    │            │            │
  │  │ └──────────┘     └──────────┘            │            │
  │  │                                          │            │
  │  │ Shadow byte meanings:                    │            │
  │  │ 0x00 = all 8 bytes accessible            │            │
  │  │ 0x01-0x07 = first N bytes accessible     │            │
  │  │ 0xFx = various poison values:            │            │
  │  │   0xFA = stack left redzone              │            │
  │  │   0xFB = stack mid redzone               │            │
  │  │   0xFC = stack right redzone             │            │
  │  │   0xFD = stack after return              │            │
  │  │   0xFE = shadow gap                      │            │
  │  │   0xFF = freed memory                   │            │
  │  │                                          │            │
  │  │ Total: 1/8 of kernel memory used for     │            │
  │  │ shadow (~12.5% memory overhead)          │            │
  │  │                                          │            │
  │  │ Shadow address = (addr >> 3) + offset    │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Instrumentation:                                        │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Compiler (GCC/Clang) inserts checks      │            │
  │  │ before every memory access:              │            │
  │  │                                          │            │
  │  │ Original:  val = *ptr;                   │            │
  │  │ With KASAN:                              │            │
  │  │   shadow = (ptr >> 3) + SHADOW_OFFSET;   │            │
  │  │   if (*shadow != 0) {                    │            │
  │  │     if (*shadow < (ptr & 7) + size) {    │            │
  │  │       kasan_report(ptr, size, is_write); │            │
  │  │     }                                    │            │
  │  │   }                                      │            │
  │  │   val = *ptr;  // original access        │            │
  │  │                                          │            │
  │  │ KASAN also hooks:                        │            │
  │  │ - kmalloc/kfree: poison freed memory     │            │
  │  │ - SLAB allocator: add redzones           │            │
  │  │ - Stack frames: add stack redzones       │            │
  │  │ - memcpy/memset/memmove: bounds check    │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. KASAN Modes and Configuration

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌────────────────┬──────────────────────────────────────┐│
  │  │ Mode           │ Details                             ││
  │  ├────────────────┼──────────────────────────────────────┤│
  │  │ Generic KASAN  │ CONFIG_KASAN_GENERIC=y              ││
  │  │                │ Shadow memory (1:8 mapping)         ││
  │  │                │ All bugs detected                   ││
  │  │                │ ~2-3x slowdown, ~12.5% memory      ││
  │  │                │ Best for development/testing        ││
  │  ├────────────────┼──────────────────────────────────────┤│
  │  │ SW Tag-Based   │ CONFIG_KASAN_SW_TAGS=y              ││
  │  │                │ ARM64 only (Top Byte Ignore)        ││
  │  │                │ Tags in pointer top byte            ││
  │  │                │ ~50% less overhead than generic     ││
  │  │                │ Probabilistic (1/256 miss rate)     ││
  │  ├────────────────┼──────────────────────────────────────┤│
  │  │ HW Tag-Based   │ CONFIG_KASAN_HW_TAGS=y              ││
  │  │                │ ARM64 MTE (Memory Tagging Extension)││
  │  │                │ Near-zero overhead (hardware check) ││
  │  │                │ Suitable for production             ││
  │  │                │ Requires ARMv8.5-A MTE hardware     ││
  │  └────────────────┴──────────────────────────────────────┘│
  │                                                           │
  │  Configuration:                                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ CONFIG_KASAN=y                           │            │
  │  │ CONFIG_KASAN_GENERIC=y                   │            │
  │  │ CONFIG_KASAN_INLINE=y  (faster checks)   │            │
  │  │ # or CONFIG_KASAN_OUTLINE=y (smaller code)│           │
  │  │ CONFIG_KASAN_STACK=y  (stack bugs)       │            │
  │  │ CONFIG_SLUB_DEBUG=y  (slab redzones)     │            │
  │  │                                          │            │
  │  │ Boot-time:                               │            │
  │  │ kasan.fault=report  (report only, no panic)│         │
  │  │ kasan.fault=panic   (panic on first bug) │            │
  │  │ kasan.stacktrace=on (include stack trace)│            │
  │  │                                          │            │
  │  │ Quarantine (delays memory reuse):        │            │
  │  │ Freed memory kept in quarantine list     │            │
  │  │ → increases chance of detecting UAF      │            │
  │  │ kasan.quarantine_size=4M                 │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. KASAN Report Example

```
  ┌──────────────────────────────────────────────────────────┐
  │  ===========================================================
  │  BUG: KASAN: slab-use-after-free in my_func+0x28/0x40
  │  Read of size 4 at addr ffff888012345678 by task test/1234
  │  
  │  CPU: 2 PID: 1234 Comm: test Not tainted 6.1.0
  │  Call Trace:
  │   dump_stack_lvl+0x38/0x48
  │   kasan_report+0xb5/0xe0
  │   my_func+0x28/0x40
  │   caller_func+0x15/0x20
  │  
  │  Allocated by task 1234:
  │   kasan_save_stack+0x1e/0x40
  │   kmalloc+0xab/0xf0
  │   alloc_func+0x20/0x30
  │   caller_func+0x10/0x20
  │  
  │  Freed by task 1234:
  │   kasan_save_stack+0x1e/0x40
  │   kfree+0x9a/0xd0
  │   free_func+0x18/0x20
  │   caller_func+0x12/0x20
  │  
  │  The buggy address belongs to the object at ffff888012345660
  │   which belongs to the cache kmalloc-64 of size 64
  │  ===========================================================
  │
  │  This tells you:
  │  - Bug type: use-after-free
  │  - Access: read of 4 bytes at specific address
  │  - Three stack traces: access, allocation, free
  │  - Cache: kmalloc-64 → object was 64 bytes
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does KASAN detect use-after-free bugs?**
**A:** When memory is freed via `kfree()`, KASAN's hooked free path: (1) Poisons the shadow memory for the entire freed object — writes `0xFF` to shadow bytes corresponding to the freed region. (2) Places the freed object in a quarantine (a delay queue) — the memory is NOT immediately returned to the SLAB allocator's free list. This is crucial: if memory were immediately reused, a use-after-free might access valid (newly allocated) memory and go undetected. (3) When any code later dereferences a pointer to the freed object, the compiler-inserted shadow check runs: it reads the shadow byte, finds `0xFF` (freed poison), and calls `kasan_report()`. The report includes three stack traces: the current (buggy) access, the original allocation site, and the free site — giving the developer full context. The quarantine has a configurable size (default ~3MB per CPU); when full, the oldest quarantined objects are released back to the SLAB allocator and their shadow memory is un-poisoned. Larger quarantine = longer detection window = more memory usage. With `CONFIG_KASAN_STACK=y`, KASAN also detects use-after-return (accessing stack variables after the function returned) by poisoning stack redzones.

---

## Summary

- KASAN: compiler-instrumented memory bug detector (UAF, OOB, double-free)
- Shadow memory: 1 byte per 8 bytes of kernel memory; checked before every access
- Generic mode: ~2-3x slowdown, 12.5% memory overhead — best for CI/testing
- HW tag-based (ARM64 MTE): near-zero overhead — suitable for production
- Reports include 3 stack traces: access, allocation, free site
- Quarantine: delays freed memory reuse to increase UAF detection window

---

[Previous: CO-RE and libbpf ←](Chapter_22_CO_RE.md) | [Next: UBSAN, KCSAN, KFENCE →](Chapter_24_UBSAN_KCSAN.md)
