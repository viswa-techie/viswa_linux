# Chapter 24: Memory Debugging Tools

## Chapter Overview

Linux provides a rich set of tools for memory analysis, debugging leaks, detecting corruption, and performance tuning. This chapter covers userspace tools (`/proc/meminfo`, `vmstat`, `slabtop`, `valgrind`), kernel tools (`ftrace`, `perf`), and embedded/Android tools (`dumpsys`, `procrank`).

---

## 24.1 /proc/meminfo — System Memory Overview

```bash
$ cat /proc/meminfo
MemTotal:       16384000 kB    # Total physical RAM
MemFree:         2048000 kB    # Completely unused pages
MemAvailable:    8192000 kB    # Estimated available for new alloc
                               # (free + reclaimable cache/buffers)
Buffers:          512000 kB    # Block device cache (metadata)
Cached:          5120000 kB    # Page cache (file data)
SwapCached:        64000 kB    # Swap pages also in RAM (swap cache)
Active:          6400000 kB    # Recently accessed pages
Inactive:        4800000 kB    # Not recently accessed (reclaimable)
Active(anon):    3200000 kB    # Active anonymous pages (heap/stack)
Inactive(anon):  1600000 kB    # Inactive anonymous pages
Active(file):    3200000 kB    # Active file cache pages
Inactive(file):  3200000 kB    # Inactive file cache pages
SwapTotal:       8192000 kB    # Total swap space
SwapFree:        8000000 kB    # Unused swap
Dirty:             32000 kB    # Pages waiting for writeback
Writeback:             0 kB    # Pages being written to disk now
AnonPages:       4800000 kB    # Anonymous mapped pages
Mapped:          1024000 kB    # Files mmap'd into page tables
Shmem:            256000 kB    # Shared memory (shmem/tmpfs)
KReclaimable:     512000 kB    # Kernel reclaimable (slab etc.)
Slab:             768000 kB    # Total slab allocator memory
SReclaimable:     512000 kB    # Reclaimable slab (dentry/inode cache)
SUnreclaim:       256000 kB    # Non-reclaimable slab
KernelStack:       32000 kB    # Kernel thread stacks
PageTables:        64000 kB    # Page table memory
VmallocTotal:  34359738367 kB  # Total vmalloc address space
VmallocUsed:      128000 kB    # Used vmalloc memory
HugePages_Total:       0       # Reserved huge pages
HugePages_Free:        0       # Free huge pages
Hugepagesize:       2048 kB    # Huge page size
```

### Memory Accounting Formula

```
Used memory   = MemTotal - MemFree - Buffers - Cached - SReclaimable
MemAvailable ≈ MemFree + Active(file) + Inactive(file) + SReclaimable
                - min_watermark_pages

┌──────────────────────────────────────────────────────────────────┐
│                        MemTotal (16 GB)                         │
├────────┬──────────┬───────────┬──────────┬─────────┬────────────┤
│MemFree │ Active   │ Inactive  │ Slab     │Kernel   │ Other      │
│ 2GB    │ (anon+   │ (anon+    │ 768MB    │Stack+PT │            │
│        │  file)   │  file)    │          │ 96MB    │            │
│        │ 6.4GB    │ 4.8GB     │          │         │            │
├────────┴──────────┴───────────┴──────────┴─────────┴────────────┤
│                                                                  │
│ "Available" ≈ Free + Reclaimable file cache + Reclaimable slab  │
│             ≈ 2GB + ~3GB(inactive file) + 512MB = ~5.5GB        │
│ Actual MemAvailable is kernel's best estimate = 8GB              │
└──────────────────────────────────────────────────────────────────┘
```

---

## 24.2 /proc/<pid>/maps and smaps — Per-Process Memory

```bash
# Virtual memory map of a process:
$ cat /proc/$(pidof nginx)/maps
5612a6b54000-5612a6b56000 r--p 00000000 08:01 131234 /usr/sbin/nginx
5612a6b56000-5612a6baa000 r-xp 00002000 08:01 131234 /usr/sbin/nginx
5612a6baa000-5612a6bc4000 r--p 00056000 08:01 131234 /usr/sbin/nginx
5612a6bc5000-5612a6bc8000 rw-p 00070000 08:01 131234 /usr/sbin/nginx
5612a8c00000-5612a9000000 rw-p 00000000 00:00 0      [heap]
7f8ba3200000-7f8ba3400000 r--p 00000000 08:01 262180 /lib/libc.so.6
...
7ffd3a2de000-7ffd3a2ff000 rw-p 00000000 00:00 0      [stack]
7ffd3a3fe000-7ffd3a400000 r-xp 00000000 00:00 0      [vdso]

# Detailed memory info per VMA:
$ cat /proc/$(pidof nginx)/smaps
5612a8c00000-5612a9000000 rw-p 00000000 00:00 0 [heap]
Size:               4096 kB     # VMA virtual size
KernelPageSize:        4 kB     # Page size used
Rss:                2048 kB     # Resident in RAM
Pss:                2048 kB     # Proportional share (RSS / #sharers)
Shared_Clean:          0 kB     # Shared, not modified
Shared_Dirty:          0 kB     # Shared, modified
Private_Clean:      1024 kB     # Private, not modified
Private_Dirty:      1024 kB     # Private, modified
Referenced:         2048 kB     # Accessed recently
Anonymous:          2048 kB     # Not file-backed
Swap:                  0 kB     # Swapped out
SwapPss:               0 kB     # Proportional swap
Locked:                0 kB     # mlock'd

# Summary of all VMAs:
$ cat /proc/$(pidof nginx)/smaps_rollup
Rss:               45632 kB    # Total resident
Pss:               12800 kB    # Total proportional (shared pages split)
Shared_Clean:      32768 kB    # Shared libs
Private_Dirty:      8192 kB    # Process-specific dirty pages
```

---

## 24.3 vmstat — Virtual Memory Statistics

```bash
# Real-time memory activity:
$ vmstat 1
procs ------memory------  --swap-- -----io---- -system- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy
 2  0      0 2048000 512000 5120000  0    0    12    45  1200 3400 15  5
 1  0      0 2040000 512000 5128000  0    0     0    32  1100 3200 12  3
 3  1   1024 2000000 512000 5100000  4    8   256   128  1500 4000 25  8
                                     ↑    ↑
                            swap in  swap out → memory pressure!

# Key columns:
# swpd:  Swap used (KB) — should be near 0 normally
# free:  Free memory (KB)
# buff:  Buffer cache (KB)
# cache: Page cache (KB)
# si:    Swap in (KB/s) — pages read from swap
# so:    Swap out (KB/s) — pages written to swap
# bi:    Block in (KB/s) — disk reads
# bo:    Block out (KB/s) — disk writes

# Extended memory stats:
$ vmstat -s
     16384000 K total memory
     10240000 K used memory
      6400000 K active memory
      4800000 K inactive memory
      2048000 K free memory
       512000 K buffer memory
      5120000 K swap cache
      8192000 K total swap
            0 K used swap
        12345 pages paged in
        67890 pages paged out
          100 pages swapped in
           50 pages swapped out
      1234567 interrupts
      5678901 CPU context switches
```

---

## 24.4 slabtop — Kernel Slab Allocator Monitor

```bash
$ sudo slabtop -o
 Active / Total Objects (% used)    : 1234567 / 1500000 (82.3%)
 Active / Total Slabs (% used)      : 45678 / 50000 (91.4%)
 Active / Total Caches (% used)     : 200 / 250 (80.0%)
 Active / Total Size (% used)       : 450.00M / 550.00M (81.8%)

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
250000 240000  96%    0.19K   6250       40     25000K dentry
180000 170000  94%    0.69K  11250       16     45000K inode_cache
120000 110000  91%    1.06K   7500       16     30000K ext4_inode_cache
 80000  75000  93%    0.12K   2500       32      5000K kernfs_node_cache
 60000  58000  96%    0.57K   4286       14      8572K radix_tree_node
 50000  48000  96%    0.25K   1563       32      6252K kmalloc-256
 40000  39000  97%    0.50K   2500       16      5000K kmalloc-512
 30000  28000  93%    4.00K   3750        8     30000K kmalloc-4k
 25000  24000  96%    0.06K    391       64       782K kmalloc-64

# Key caches to watch:
# dentry         — directory entry cache (VFS path lookup)
# inode_cache    — VFS inode cache  
# ext4_inode     — filesystem-specific inodes
# task_struct    — per-process descriptor
# mm_struct      — per-process memory descriptor
# vm_area_struct — per-VMA descriptor

# If a slab cache grows unbounded → possible kernel memory leak
# Compare OBJ SIZE × OBJS to understand total memory per cache
```

---

## 24.5 perf — Memory Performance Events

```bash
# Count page faults:
$ perf stat -e page-faults,minor-faults,major-faults -- ./myapp
 Performance counter stats for './myapp':
         12,345      page-faults
         12,300      minor-faults    # Resolved in RAM
             45      major-faults    # Required disk I/O
      2.345678 seconds time elapsed

# Profile memory allocation hotspots:
$ perf record -e kmem:kmalloc -- ./myapp
$ perf report
# Shows which code paths allocate the most kernel memory

# TLB miss profiling:
$ perf stat -e dTLB-load-misses,dTLB-store-misses,iTLB-load-misses ./myapp
         5,678  dTLB-load-misses    # Data TLB misses
         1,234  dTLB-store-misses   # Data TLB store misses
           456  iTLB-load-misses    # Instruction TLB misses
# High TLB misses → consider huge pages

# Cache miss profiling:
$ perf stat -e cache-references,cache-misses,LLC-load-misses ./myapp
     1,234,567  cache-references
        12,345  cache-misses       # 1.0% miss rate
         5,678  LLC-load-misses    # Last-level cache misses (→ DRAM)

# Memory access pattern profiling (requires hardware support):
$ perf mem record ./myapp
$ perf mem report
# Shows per-instruction memory access latency distribution
```

---

## 24.6 valgrind/memcheck — Userspace Memory Debugging

```bash
# Detect memory leaks, use-after-free, buffer overflows:
$ valgrind --tool=memcheck --leak-check=full ./myapp

==12345== Memcheck, a memory error detector
==12345== Invalid read of size 4
==12345==    at 0x4012AB: process_data (myapp.c:42)
==12345==  Address 0x5234044 is 4 bytes after a block of size 64 alloc'd
==12345==    at 0x483BE63: malloc (vg_replace_malloc.c:381)
==12345==    by 0x4011FA: init_buffer (myapp.c:20)

==12345== HEAP SUMMARY:
==12345==     in use at exit: 1,024 bytes in 4 blocks
==12345==   total heap usage: 100 allocs, 96 frees, 50,000 bytes allocated

==12345== 1,024 bytes in 4 blocks are definitely lost
==12345==    at 0x483BE63: malloc (vg_replace_malloc.c:381)
==12345==    by 0x401234: create_item (myapp.c:30)
==12345==    by 0x401456: main (myapp.c:80)

# Summary:
# "still reachable" — not freed but pointer exists (usually OK)
# "definitely lost" — no pointer to allocation → REAL LEAK
# "indirectly lost" — reachable only through "lost" blocks
# "possibly lost"   — pointer points inside block → suspicious
```

---

## 24.7 AddressSanitizer (ASan) — Compile-Time Memory Checker

```bash
# Faster than valgrind (~2x slowdown vs ~20x for valgrind)
$ gcc -fsanitize=address -g -o myapp myapp.c
$ ./myapp

=================================================================
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x6020000000a0
READ of size 4 at 0x6020000000a0 thread T0
    #0 0x4012ab in process_data myapp.c:42
    #1 0x401456 in main myapp.c:80
0x6020000000a0 is located 0 bytes to the right of 64-byte region [0x602000000060,0x6020000000a0)
allocated by thread T0 here:
    #0 0x7f123456 in malloc asan_malloc_linux.cpp:69
    #1 0x4011fa in init_buffer myapp.c:20

# ASan detects:
# - Heap buffer overflow/underflow
# - Stack buffer overflow
# - Use-after-free
# - Double-free
# - Memory leaks (with -fsanitize=leak)
# - Use-after-return (with ASAN_OPTIONS=detect_stack_use_after_return=1)

# Kernel ASan: CONFIG_KASAN (see Chapter 25)
```

---

## 24.8 /proc/buddyinfo and /proc/pagetypeinfo

```bash
# Buddy allocator free page distribution:
$ cat /proc/buddyinfo
Node 0, zone      DMA      1    1    0    1    2    1    1    0    1    1    3
Node 0, zone    DMA32    512  256  128   64   32   16    8    4    2    1    0
Node 0, zone   Normal  12345 6789 3456 1234  567  234   89   34   12    4    1
#                        2^0  2^1 2^2  2^3  2^4  2^5  2^6  2^7  2^8 2^9 2^10
#                        4KB  8KB 16KB 32KB 64KB 128K 256K 512K 1MB 2MB 4MB

# Page type information (fragmentation analysis):
$ cat /proc/pagetypeinfo
Free pages count per migrate type at order
Node    0, zone   Normal, type    Unmovable   1024    512    256    128
Node    0, zone   Normal, type      Movable   8000   4000   2000   1000
Node    0, zone   Normal, type  Reclaimable   2000   1000    500    250
Node    0, zone   Normal, type      Isolate      0      0      0      0

# Interpretation:
# If high-order (2^9, 2^10) counts are 0 → fragmentation problem
# Can't allocate contiguous large blocks
# Solution: echo 1 > /proc/sys/vm/compact_memory
```

---

## 24.9 Memory Debugging Quick Reference

```
┌────────────────────────┬──────────────────────────────────────────────┐
│ Problem                │ Tool / Command                               │
├────────────────────────┼──────────────────────────────────────────────┤
│ System memory overview │ cat /proc/meminfo; free -h                  │
│ Process RSS/PSS        │ cat /proc/<pid>/smaps_rollup                │
│ Per-VMA breakdown      │ cat /proc/<pid>/smaps                       │
│ Memory over time       │ vmstat 1; sar -r 1                         │
│ Slab allocation        │ slabtop; cat /proc/slabinfo                │
│ Page fault analysis    │ perf stat -e page-faults ./app             │
│ TLB miss analysis      │ perf stat -e dTLB-load-misses ./app        │
│ Cache miss analysis    │ perf stat -e cache-misses ./app            │
│ User memory leaks      │ valgrind --leak-check=full ./app           │
│ User buffer overflow   │ gcc -fsanitize=address ./app               │
│ Kernel memory leak     │ KASAN / kmemleak (Chapter 25)              │
│ Fragmentation          │ cat /proc/buddyinfo                        │
│ Memory pressure        │ cat /proc/pressure/memory                  │
│ Cgroup memory          │ cat /sys/fs/cgroup/X/memory.stat           │
│ NUMA statistics        │ numastat -p <pid>                          │
│ OOM investigation      │ dmesg | grep -i "oom\|out of memory"       │
│ Swap activity          │ vmstat 1 (si/so columns)                   │
│ Page cache drop        │ echo 3 > /proc/sys/vm/drop_caches         │
│ Huge page stats        │ cat /proc/meminfo | grep Huge              │
│ Kernel stack usage     │ cat /proc/<pid>/stack                      │
└────────────────────────┴──────────────────────────────────────────────┘
```

---

## 24.10 Android-Specific Memory Tools

```bash
# dumpsys meminfo — Per-process memory breakdown:
$ adb shell dumpsys meminfo com.example.app
Applications Memory Usage (in Kilobytes):
Uptime: 123456 Realtime: 234567
** MEMINFO in pid 1234 [com.example.app] **
                   Pss  Private  Private  SwapPss    Heap    Heap    Heap
                 Total    Dirty    Clean    Dirty    Size   Alloc    Free
                ------   ------   ------   ------   ------  ------  ------
  Native Heap    12345    12000      300        0    32768   24576    8192
  Dalvik Heap     8000     7500      400        0    16384   12288    4096
  Dalvik Other    2000     1800      200        0
         Stack      512      512        0        0
        Ashmem     1024     1024        0        0
    Other mmap     2048      256     1792        0
    Code           4096       64     4032        0
       TOTAL     30025    23156     6724        0

# procrank — Process ranking by memory:
$ adb shell procrank
  PID       Vss       Rss       Pss       Uss  cmdline
 1234   1234567    456789    234567    200000  com.example.app
 2345    987654    345678    167890    150000  system_server

# showmap — Detailed mapping breakdown:
$ adb shell showmap <pid>
   start    end     virtual  shared  shared  private private
   addr     addr    size     clean   dirty   clean   dirty   object
   -------- ------  ------   ------  ------  ------  ------  ------
   12c00000 13400000  8192      0     7500     400      0   [anon:dalvik-main]
```

---

## Interview Questions

1. **Q: How do you determine actual memory usage of a process (not just RSS)?**
   A: Use `PSS` (Proportional Set Size) from `/proc/<pid>/smaps_rollup`. PSS divides shared pages proportionally among sharing processes, giving a more accurate picture. For example, if 3 processes share a 12MB library, each gets 4MB PSS. Compare with USS (Unique Set Size) for private-only memory.

2. **Q: What does it mean when `/proc/meminfo` shows low MemFree but high MemAvailable?**
   A: MemFree counts only completely unused pages. MemAvailable includes reclaimable memory (page cache, slab cache). The kernel can quickly free cached pages when needed. Low MemFree with high MemAvailable is normal — the kernel uses free RAM for caching.

3. **Q: How would you diagnose a kernel memory leak?**
   A: 1) Check `slabtop` for monotonically growing slab caches. 2) Enable `kmemleak` (`CONFIG_DEBUG_KMEMLEAK`, `echo scan > /sys/kernel/debug/kmemleak`). 3) Monitor `/proc/meminfo` SUnreclaim over time. 4) Use KASAN for use-after-free detection. 5) `ftrace` function graph for allocation/free tracking.

4. **Q: How do you check if a system is memory-constrained?**
   A: Check: 1) `/proc/pressure/memory` — PSI stall percentages > 0. 2) `vmstat` — non-zero si/so (swap activity). 3) major page faults increasing. 4) MemAvailable trending toward zero. 5) OOM killer messages in `dmesg`. 6) kswapd CPU usage rising.

---

## Summary

1. `/proc/meminfo` gives system-wide memory overview; `MemAvailable` is the key metric.
2. `/proc/<pid>/smaps` provides per-VMA breakdown; PSS is the most accurate usage metric.
3. `vmstat` shows real-time memory activity; watch for swap (si/so) activity.
4. `slabtop` monitors kernel slab allocator for leak detection.
5. `perf` profiles hardware memory events (TLB misses, cache misses, page faults).
6. `valgrind` and ASan detect userspace memory bugs (leaks, overflows, use-after-free).
7. `/proc/buddyinfo` reveals physical memory fragmentation.
8. Android tools: `dumpsys meminfo`, `procrank`, `showmap` for app memory analysis.

---

*Next: [Chapter 25 — Kernel Memory Debugging (KASAN, kmemleak)](Chapter_25_Kernel_Memory_Debugging.md)*
