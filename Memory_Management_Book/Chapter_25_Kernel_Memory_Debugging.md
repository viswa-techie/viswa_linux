# Chapter 25: Kernel Memory Debugging (KASAN, kmemleak, KFENCE)

## Chapter Overview

The Linux kernel includes powerful compile-time and runtime memory debugging tools. This chapter covers KASAN (Kernel Address Sanitizer), kmemleak (kernel memory leak detector), KFENCE (Kernel Electric Fence), and other kernel debugging infrastructure for memory corruption, leaks, and use-after-free bugs.

---

## 25.1 KASAN — Kernel Address Sanitizer

```
KASAN detects:
- Out-of-bounds access (heap, stack, global)
- Use-after-free
- Double-free
- Invalid free

KASAN modes:
┌────────────────┬──────────────────┬──────────────────┬──────────────────┐
│ Mode           │ Generic KASAN    │ SW-tag KASAN     │ HW-tag KASAN     │
├────────────────┼──────────────────┼──────────────────┼──────────────────┤
│ Architecture   │ Any              │ ARM64            │ ARM64 + MTE      │
│ Overhead       │ ~2x memory       │ ~1.5x memory     │ ~1.1x memory     │
│                │ ~3x CPU          │ ~2x CPU          │ ~1.05x CPU       │
│ Detection      │ Byte-level       │ Tag-based        │ Hardware-assisted│
│ Config         │ CONFIG_KASAN_    │ CONFIG_KASAN_    │ CONFIG_KASAN_    │
│                │ GENERIC          │ SW_TAGS          │ HW_TAGS          │
│ Shadow memory  │ 1/8th of RAM     │ 1/16th or 1/32  │ Hardware tags    │
│ Production use │ No (too slow)    │ No (moderate)    │ Yes (low overhead)│
└────────────────┴──────────────────┴──────────────────┴──────────────────┘
```

### Generic KASAN: Shadow Memory

```
Every 8 bytes of kernel memory has 1 byte of shadow:

Kernel Memory:  [8 bytes][8 bytes][8 bytes][8 bytes]
Shadow Memory:  [  00  ] [  00  ] [  03  ] [  FF  ]
                  valid    valid   3 valid  all invalid
                                   bytes    (freed/OOB)
                                   
Shadow values:
0x00       = all 8 bytes accessible
0x01-0x07  = only first N bytes accessible (partial)
0xFx       = various invalid states:
  0xFA     = freed memory (use-after-free zone)
  0xFB     = SLUB redzone (out-of-bounds guard)
  0xFC     = kmalloc redzone
  0xFD     = freed by kfree
  0xFE     = shadow gap
  0xFF     = shadow not initialized

Shadow address calculation (x86_64):
  shadow_addr = (addr >> 3) + KASAN_SHADOW_OFFSET
  
Memory layout:
┌───────────────────┐ 0xFFFFFFFFFFFFFFFF
│ Kernel virtual    │ 
│ address space     │ → shadow at KASAN_SHADOW_OFFSET + (addr >> 3)
├───────────────────┤
│ KASAN shadow      │ 1/8th of kernel VA space  
│ memory region     │ (requires significant RAM!)
├───────────────────┤
│ ...               │
└───────────────────┘
```

### KASAN in Action

```c
/* Enable: CONFIG_KASAN=y + CONFIG_KASAN_GENERIC=y */
/* Boot with: kasan.stacktrace=on */

/* Example bug: heap out-of-bounds */
char *buf = kmalloc(64, GFP_KERNEL);
buf[65] = 'X';  /* Out-of-bounds write! */

/* KASAN output: */
/*
==================================================================
BUG: KASAN: slab-out-of-bounds in my_function+0x42/0x100
Write of size 1 at addr ffff888012345040 by task myapp/1234

CPU: 0 PID: 1234 Comm: myapp Not tainted 6.1.0 #1
Call Trace:
 dump_stack_lvl+0x49/0x63
 print_report+0x171/0x486
 kasan_report+0xb4/0x130
 my_function+0x42/0x100
 my_caller+0x2a/0x50
 
Allocated by task 1234:
 kasan_save_stack+0x22/0x50
 kmalloc+0xaa/0x120
 my_function+0x20/0x100

The buggy address belongs to the object at ffff888012345000
 which belongs to the cache kmalloc-64 of size 64
The buggy address is located 1 bytes to the right of
 64-byte region [ffff888012345000, ffff888012345040)
==================================================================
*/
```

### KASAN Use-After-Free Detection

```c
char *buf = kmalloc(64, GFP_KERNEL);
kfree(buf);
buf[0] = 'X';  /* Use-after-free! */

/* KASAN output: */
/*
BUG: KASAN: slab-use-after-free in my_function+0x50/0x100
Write of size 1 at addr ffff888012345000 by task myapp/1234

Freed by task 1234:
 kasan_save_stack+0x22/0x50
 kfree+0x9a/0x110
 my_function+0x40/0x100

Allocated by task 1234:
 kmalloc+0xaa/0x120
 my_function+0x20/0x100
*/

/* KASAN remembers both allocation AND free call stacks! */
/* Quarantine: freed memory is quarantined (not reused immediately) */
/* to increase the chance of catching use-after-free */
```

---

## 25.2 kmemleak — Kernel Memory Leak Detector

```
kmemleak scans kernel memory for allocated blocks with no pointer
references. If nothing points to an allocation, it's a leak.

How it works:
1. Track all kmalloc/vmalloc/kmem_cache_alloc calls
2. Periodically scan kernel memory (data, stacks, per-CPU areas)
3. Look for pointers to tracked allocations
4. Report unreferenced allocations as potential leaks

┌────────────────────────────────────────────────────────────────┐
│ Kernel Memory                                                  │
│                                                                │
│  ptr1 ──────→ [Allocation A]  ← Referenced → NOT leaked       │
│                                                                │
│  ptr2 ──────→ [Allocation B]  ← Referenced → NOT leaked       │
│                                                                │
│               [Allocation C]  ← NO pointer! → LEAKED!         │
│                                                                │
│  ptr3         [freed memory]  ← ptr3 is dangling              │
│    └──X──→   (already freed)                                   │
└────────────────────────────────────────────────────────────────┘
```

```bash
# Enable: CONFIG_DEBUG_KMEMLEAK=y

# Usage:
$ echo scan > /sys/kernel/debug/kmemleak     # Trigger scan
$ cat /sys/kernel/debug/kmemleak              # View results

unreferenced object 0xffff888012345000 (size 256):
  comm "mydriver", pid 1234, jiffies 4294967890 (age 120.5s)
  hex dump (first 32 bytes):
    00 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00
  backtrace:
    [<ffffffff81234567>] kmalloc+0xaa/0x120
    [<ffffffffa0001234>] my_driver_init+0x50/0x100
    [<ffffffff81012345>] do_one_initcall+0x42/0x200

# Commands:
$ echo clear > /sys/kernel/debug/kmemleak    # Clear results
$ echo scan > /sys/kernel/debug/kmemleak     # Scan now
$ echo off > /sys/kernel/debug/kmemleak      # Disable

# False positives: kmemleak may report allocations stored in
# ways it can't detect (encoded pointers, hardware registers).
# Use kmemleak_not_leak() / kmemleak_ignore() to suppress.
```

---

## 25.3 KFENCE — Kernel Electric Fence

```
KFENCE: Low-overhead sampling-based memory error detector.
Designed for PRODUCTION use (unlike KASAN which is too expensive).

How it works:
1. Small pool of guard pages (~256KB default, configurable)
2. Randomly divert kmalloc/slab allocations to KFENCE pool
3. Each KFENCE allocation is surrounded by guard pages
4. Out-of-bounds → page fault on guard page
5. After free → page permissions revoked → use-after-free = fault

┌────────────────────────────────────────────────────────────────┐
│ KFENCE Pool Layout:                                            │
│                                                                │
│ [GUARD][object A][GUARD][object B][GUARD][object C][GUARD]    │
│ PAGE   ← alloc → PAGE  ← alloc → PAGE  ← alloc → PAGE       │
│ (no    (rw)      (no    (rw)      (no    (rw)      (no        │
│  access)         access)          access)           access)    │
│                                                                │
│ Write past object A → hits GUARD page → page fault!           │
│ Read freed object B → page marked no-access → page fault!     │
└────────────────────────────────────────────────────────────────┘

CONFIG_KFENCE=y
kfence.sample_interval=100  # Check every 100ms (default)

Overhead: < 1% (suitable for production!)
Detection rate: Statistical (samples ~1 in N allocations)
Coverage: Increases over uptime as more allocations sampled
```

```bash
# Check KFENCE status:
$ cat /sys/kernel/debug/kfence/stats
enabled: 1
sample interval: 100
num objects: 255
num faults: 3

# KFENCE report example:
$ dmesg
[  123.456789] ==================================================================
[  123.456789] BUG: KFENCE: out-of-bounds read in my_function+0x42/0x100
[  123.456789] Out-of-bounds read at 0xffff888012345040 (1B right of kfence-#42):
[  123.456789]  my_function+0x42/0x100
[  123.456789]  my_caller+0x2a/0x50
[  123.456789] 
[  123.456789] kfence-#42: 0xffff888012345000-0xffff88801234503f, size=64, cache=kmalloc-64
[  123.456789] allocated by task 1234:
[  123.456789]  kmalloc+0xaa/0x120
[  123.456789]  my_function+0x20/0x100
```

---

## 25.4 Comparison: KASAN vs kmemleak vs KFENCE

```
┌──────────────┬────────────────┬────────────────┬──────────────────┐
│              │ KASAN          │ kmemleak       │ KFENCE           │
├──────────────┼────────────────┼────────────────┼──────────────────┤
│ Detects      │ OOB, UAF, DF  │ Memory leaks   │ OOB, UAF         │
│ Overhead     │ 2-3x CPU+mem  │ Periodic scan  │ < 1% CPU         │
│ Production   │ No             │ Debug only     │ Yes!             │
│ Coverage     │ 100%           │ 100% of allocs │ Statistical      │
│ Mechanism    │ Shadow memory  │ Pointer scan   │ Guard pages      │
│ Config       │ CONFIG_KASAN   │ CONFIG_DEBUG_  │ CONFIG_KFENCE    │
│              │                │ KMEMLEAK       │                  │
│ Use case     │ Dev/testing    │ Dev/testing    │ Prod monitoring  │
│ Stack bugs   │ Yes (generic)  │ No             │ No               │
│ Report       │ Immediate      │ On scan        │ Immediate        │
└──────────────┴────────────────┴────────────────┴──────────────────┘
```

---

## 25.5 Other Kernel Memory Debug Options

```
CONFIG_DEBUG_PAGEALLOC:
  Unmaps freed pages from kernel address space.
  Any access to freed page → immediate kernel oops.
  High overhead (TLB flush per free), dev only.

CONFIG_DEBUG_SLAB / CONFIG_SLUB_DEBUG:
  Adds redzones around slab objects (poison bytes 0x5A).
  Checks for corruption on alloc/free.
  $ echo 1 > /sys/kernel/slab/<cache>/sanity_checks

  Slab debug flags (boot: slub_debug=FZPU):
  F — Sanity checks (free validation)
  Z — Red zoning (buffer overflow detection)
  P — Poisoning (use-after-free detection: 0x6B fill)
  U — User tracking (store alloc/free caller info)

CONFIG_PAGE_POISONING:
  Fill freed pages with 0xAA pattern.
  Detect use-after-free of page-level allocations.

CONFIG_DEBUG_VM:
  Extra VM assertions (BUG_ON checks in mm/ code).
  Catches internal MM bugs.

CONFIG_DEBUG_VIRTUAL:
  Validate virt_to_phys() / phys_to_virt() conversions.

CONFIG_MEMCG_KMEM:
  Account kernel memory per-cgroup for leak isolation.
```

### Slab Poisoning Example

```
After kmalloc(64):
┌──────────────────────────────────────────────────────────┐
│ REDZONE │ Object data (64 bytes)              │ REDZONE │
│ 0xBB    │ [usable memory]                     │ 0xBB    │
│ (guard) │                                     │ (guard) │
└──────────────────────────────────────────────────────────┘

After kfree():
┌──────────────────────────────────────────────────────────┐
│ REDZONE │ Poisoned: 0x6B 0x6B 0x6B ...        │ REDZONE │
│ 0xBB    │ (any read returning 0x6B = freed)   │ 0xBB    │
│ (guard) │                                     │ (guard) │
└──────────────────────────────────────────────────────────┘

On next alloc: verify redzone intact (0xBB).
  Corrupted? → kernel BUG: slab corruption detected!
On next alloc: verify poison intact (0x6B).  
  Modified? → kernel BUG: object modified after free!
```

---

## 25.6 ftrace for Memory Debugging

```bash
# Trace kmalloc/kfree calls:
$ cd /sys/kernel/debug/tracing

# Enable kmalloc tracer:
$ echo 1 > events/kmem/kmalloc/enable
$ echo 1 > events/kmem/kfree/enable
$ echo 1 > tracing_on

# Run workload, then:
$ cat trace | head -20
#           TASK-PID   CPU#  TIMESTAMP  FUNCTION
           myapp-1234  [000]  123.456:  kmalloc: call_site=my_func+0x20 
                                         ptr=ffff888012345000 bytes_req=64 
                                         bytes_alloc=64 gfp_flags=GFP_KERNEL
           myapp-1234  [000]  123.789:  kfree: call_site=my_func+0x80 
                                         ptr=ffff888012345000

# Trace page allocations:
$ echo 1 > events/kmem/mm_page_alloc/enable
$ echo 1 > events/kmem/mm_page_free/enable

# Find allocations without matching frees:
# Export trace → parse with script → find unmatched ptrs → leak candidates
```

---

## 25.7 Debugging Workflow

```
Memory Bug Investigation Flowchart:

Is it reproducible?
├─ Yes → Use KASAN (full detection).
│        Enable CONFIG_KASAN_GENERIC + slub_debug=FZPU.
│        Reproduce → read dmesg for KASAN report.
│
├─ No (rare/intermittent) → Use KFENCE in production.
│        Enable CONFIG_KFENCE.
│        Wait for statistical sampling to catch it.
│
└─ Suspected leak?
   ├─ Kernel leak → Enable kmemleak.
   │   echo scan > /sys/kernel/debug/kmemleak
   │   cat /sys/kernel/debug/kmemleak
   │   Backtrace shows allocation site → find missing kfree().
   │
   ├─ Slab growth → slabtop (watch specific cache grow).
   │   /proc/slabinfo for detailed per-cache stats.
   │   slub_debug=U to track per-object callers.
   │
   └─ Page-level leak → /proc/meminfo SUnreclaim growing.
       CONFIG_DEBUG_PAGEALLOC for freed-page access detection.
       page_owner (CONFIG_PAGE_OWNER) for per-page allocation tracking.
```

---

## Interview Questions

1. **Q: How does KASAN detect out-of-bounds access?**
   A: KASAN maintains a shadow memory map (1 byte per 8 bytes of kernel memory). Each shadow byte encodes how many bytes in the corresponding 8-byte region are valid. Before each memory access, the compiler inserts a check: load shadow byte, compare with access size, if invalid → report bug. Redzones around allocations are marked invalid in shadow.

2. **Q: What is KFENCE and why is it used in production?**
   A: KFENCE (Kernel Electric Fence) is a sampling-based memory error detector with < 1% overhead. It diverts a small fraction of allocations to a pool surrounded by guard pages. Out-of-bounds access hits a guard page (fault), use-after-free hits revoked-permission page (fault). Statistical sampling means it catches bugs over time without performance penalty.

3. **Q: How does kmemleak find memory leaks?**
   A: kmemleak tracks all `kmalloc`/`vmalloc` allocations and periodically scans kernel memory (stacks, data sections, per-CPU areas) for pointers to those allocations. If no pointer references an allocation, it's reported as a potential leak. It's a conservative mark-and-sweep garbage collector that only reports — doesn't free.

4. **Q: What is slab poisoning and redzone checking?**
   A: Poisoning fills freed slab objects with a known pattern (0x6B). If the pattern is disturbed before reallocation, it indicates use-after-free corruption. Redzones are guard bytes (0xBB) placed before/after each slab object. If corrupted, it indicates buffer overflow/underflow. Both enabled via `slub_debug=FZP`.

---

## Summary

1. **KASAN**: Comprehensive memory error detector (OOB, UAF) via shadow memory. Dev/testing only (2-3x overhead).
2. **kmemleak**: Kernel memory leak detector via pointer scanning. Reports unreferenced allocations.
3. **KFENCE**: Production-safe sampling detector via guard pages (< 1% overhead).
4. **SLUB debug**: Redzone + poisoning + user tracking for slab corruption.
5. **DEBUG_PAGEALLOC**: Unmaps freed pages for page-level use-after-free detection.
6. **ftrace**: Trace kmalloc/kfree for allocation tracking and leak analysis.
7. Combine tools: KASAN for testing, KFENCE for production, kmemleak for leaks.

---

*Next: [Chapter 26 — Source Code Walkthrough: Key mm/ Files](Chapter_26_Source_Code_Walkthrough.md)*
