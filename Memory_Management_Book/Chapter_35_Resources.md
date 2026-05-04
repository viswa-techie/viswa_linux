# Chapter 35: Resources and Community

## Chapter Overview

This chapter provides practical resources for continued learning: kernel development setup, community engagement, conferences, mailing lists, and hands-on exercises for memory management.

---

## 35.1 Setting Up a Kernel Development Environment

```bash
# ── Minimal Kernel Build for MM Experimentation ──

# 1. Get kernel source:
$ git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
$ cd linux

# 2. Configure with debug options:
$ make defconfig
$ scripts/config --enable CONFIG_DEBUG_INFO
$ scripts/config --enable CONFIG_KASAN
$ scripts/config --enable CONFIG_KASAN_GENERIC
$ scripts/config --enable CONFIG_DEBUG_KMEMLEAK
$ scripts/config --enable CONFIG_KFENCE
$ scripts/config --enable CONFIG_SLUB_DEBUG
$ scripts/config --enable CONFIG_DEBUG_VM
$ scripts/config --enable CONFIG_DEBUG_PAGEALLOC
$ scripts/config --enable CONFIG_PAGE_OWNER
$ scripts/config --enable CONFIG_MEMCG
$ scripts/config --enable CONFIG_TRANSPARENT_HUGEPAGE

# 3. Build:
$ make -j$(nproc)

# 4. Test with QEMU (no need to install on real hardware):
$ qemu-system-x86_64 \
    -kernel arch/x86/boot/bzImage \
    -initrd /path/to/initramfs.cpio.gz \
    -m 4G \
    -enable-kvm \
    -append "console=ttyS0 kasan.stacktrace=on" \
    -nographic

# 5. For kernel module development:
$ cat > test_mm.c << 'EOF'
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/mm.h>

static int __init test_mm_init(void)
{
    void *p = kmalloc(64, GFP_KERNEL);
    pr_info("kmalloc returned %px (phys: %pa)\n", p, &virt_to_phys(p));
    kfree(p);
    
    struct page *page = alloc_pages(GFP_KERNEL, 0);
    pr_info("alloc_pages: pfn=%lu\n", page_to_pfn(page));
    __free_pages(page, 0);
    
    return 0;
}
static void __exit test_mm_exit(void) {}
module_init(test_mm_init);
module_exit(test_mm_exit);
MODULE_LICENSE("GPL");
EOF
```

---

## 35.2 Hands-On Exercises

```
Exercise 1: Page Fault Analysis
────────────────────────────────
Write a program that:
a) Allocates 1GB with mmap(MAP_ANONYMOUS)
b) Touches every page sequentially
c) Measure time and count page faults with perf

Expected learning:
- Observe demand paging (no pages allocated until touch)
- Measure minor fault overhead (~1µs per fault)
- Compare sequential vs random access patterns

Exercise 2: OOM Killer Behavior
────────────────────────────────
Run inside a cgroup with 256MB limit:
a) Fork multiple children, each allocating 100MB
b) Observe which child gets OOM-killed (check dmesg)
c) Adjust oom_score_adj and repeat

Expected learning:
- Cgroup-local OOM behavior
- OOM score selection algorithm
- oom_score_adj effectiveness

Exercise 3: THP vs Regular Pages
────────────────────────────────
Write a benchmark that:
a) Allocates 1GB and iterates random indices
b) Run with THP enabled vs disabled
c) Measure TLB misses with: perf stat -e dTLB-load-misses

Expected learning:
- THP reduces TLB misses by ~10-100x for large working sets
- Compaction overhead during THP allocation

Exercise 4: SLUB Slab Cache Analysis
────────────────────────────────
a) Create a kernel module with custom kmem_cache
b) Allocate/free 10000 objects
c) Monitor with slabtop and /proc/slabinfo
d) Intentionally leak objects and detect with kmemleak

Exercise 5: KASAN Bug Detection
────────────────────────────────
a) Write a kernel module with an intentional OOB access
b) Boot with KASAN enabled
c) Load module, observe KASAN report in dmesg
d) Fix the bug and verify clean report

Exercise 6: Memory Pressure and Reclaim
────────────────────────────────
a) Fill page cache by reading large files
b) Monitor /proc/meminfo, vmstat, /proc/pressure/memory
c) Trigger memory pressure and observe kswapd behavior
d) Tune vm.swappiness and dirty_ratio, measure effects
```

---

## 35.3 Community and Mailing Lists

```
Kernel Development Community:
┌──────────────────────────────────────────────────────────────────┐
│ Resource                │ Description                            │
├─────────────────────────┼────────────────────────────────────────┤
│ linux-mm@kvack.org      │ Memory management mailing list        │
│                         │ https://lore.kernel.org/linux-mm/     │
│                         │                                        │
│ LKML                    │ Linux Kernel Mailing List (general)   │
│                         │ https://lore.kernel.org/lkml/         │
│                         │                                        │
│ #mm on IRC (OFTC)       │ Real-time kernel MM discussion        │
│                         │                                        │
│ Linux MM wiki           │ https://linux-mm.org                  │
│                         │ Documentation, meeting notes           │
│                         │                                        │
│ Kernel Bugzilla         │ https://bugzilla.kernel.org           │
│                         │ Bug reports and tracking               │
│                         │                                        │
│ kernelnewbies.org       │ Beginner-friendly kernel resources    │
│                         │ First kernel patch guide               │
└─────────────────────────┴────────────────────────────────────────┘

Key MM Maintainers (as of 2024):
  Andrew Morton — MM subsystem maintainer (mm-commits tree)
  Michal Hocko — Memory management, cgroups
  Vlastimil Babka — SLUB, memory management
  Matthew Wilcox — Folios, XArray, page cache
  Liam Howlett — Maple tree, VMA management
  Yu Zhao — MGLRU
```

---

## 35.4 Conferences

```
┌──────────────────────────────────────────────────────────────────┐
│ Conference              │ Focus                                  │
├─────────────────────────┼────────────────────────────────────────┤
│ Linux Plumbers Conf     │ Kernel development, MM microconference│
│ (annual)                │ Deep technical talks by MM developers  │
│                         │                                        │
│ Linux Storage, FS &     │ Memory, storage, and filesystem topics│
│ Memory Management Summit│ (by invitation, annual)                │
│ (LSF/MM/BPF)           │                                        │
│                         │                                        │
│ USENIX ATC/OSDI        │ Academic OS/systems research           │
│                         │ Cutting-edge MM research               │
│                         │                                        │
│ ASPLOS                  │ Architecture + programming languages  │
│                         │ Hardware/software memory topics        │
│                         │                                        │
│ Linux Foundation Events │ Open Source Summit, others             │
│                         │ Kernel training sessions               │
└─────────────────────────┴────────────────────────────────────────┘

Most conference talks are available on YouTube for free.
Search: "Linux Plumbers MM" or "LSF/MM" for deep MM talks.
```

---

## 35.5 Contributing to Linux MM

```
How to contribute to Linux memory management:

1. Start Small
   - Fix a compiler warning in mm/
   - Add a missing __init or __exit annotation
   - Improve a comment or documentation
   
2. Run Tests
   - mm/ has extensive selftests: tools/testing/selftests/mm/
   - Run: make -C tools/testing/selftests/mm/ run_tests
   - Report test failures (may be bugs!)
   
3. Review Code
   - Subscribe to linux-mm@kvack.org
   - Read patches, try to understand them
   - Test patches on your hardware
   
4. Reproduce and Fix Bugs
   - Kernel bugzilla MM component
   - syzbot (syzkaller) finds MM bugs automatically
   - https://syzkaller.appspot.com/upstream
   
5. Submit Patches
   - Follow Documentation/process/submitting-patches.rst
   - CC: linux-mm@kvack.org for MM patches
   - Use scripts/checkpatch.pl before sending
   - Start with patches Andrew Morton reviews

Patch submission workflow:
$ git format-patch -1 HEAD
$ scripts/checkpatch.pl 0001-*.patch
$ scripts/get_maintainer.pl 0001-*.patch
$ git send-email --to=linux-mm@kvack.org 0001-*.patch
```

---

## 35.6 Tools Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────────┐
│ SYSTEM-WIDE MEMORY ANALYSIS                                         │
├───────────────────────┬──────────────────────────────────────────────┤
│ free -h               │ Quick memory/swap overview                   │
│ cat /proc/meminfo     │ Detailed memory statistics                   │
│ vmstat 1              │ Real-time memory activity                    │
│ sar -r 1              │ Memory utilization over time                 │
│ slabtop               │ Kernel slab allocator monitor                │
│ cat /proc/buddyinfo   │ Physical memory fragmentation                │
│ numastat              │ NUMA memory distribution                     │
│ cat /proc/pressure/*  │ Memory/CPU/IO pressure stall info            │
├───────────────────────┴──────────────────────────────────────────────┤
│ PER-PROCESS MEMORY ANALYSIS                                         │
├───────────────────────┬──────────────────────────────────────────────┤
│ cat /proc/PID/maps    │ Process memory map (VMAs)                    │
│ cat /proc/PID/smaps   │ Detailed per-VMA memory info                 │
│ pmap -x PID           │ Process memory regions                       │
│ cat /proc/PID/status  │ VmRSS, VmSize, VmSwap                       │
├───────────────────────┴──────────────────────────────────────────────┤
│ PERFORMANCE PROFILING                                                │
├───────────────────────┬──────────────────────────────────────────────┤
│ perf stat -e page-faults ./app   │ Page fault count                  │
│ perf stat -e dTLB-load-misses    │ TLB miss count                    │
│ perf stat -e cache-misses        │ CPU cache miss count              │
│ perf mem record/report           │ Memory access latency             │
├───────────────────────┴──────────────────────────────────────────────┤
│ DEBUGGING                                                            │
├───────────────────────┬──────────────────────────────────────────────┤
│ valgrind --memcheck   │ Userspace memory error detection             │
│ gcc -fsanitize=address│ ASan: fast memory error detection            │
│ KASAN                 │ Kernel memory error detection                │
│ KFENCE                │ Production kernel memory checking            │
│ kmemleak              │ Kernel memory leak detection                 │
│ slub_debug=FZPU       │ Slab debug (redzone, poisoning)              │
└───────────────────────┴──────────────────────────────────────────────┘
```

---

## Summary

1. **Development setup**: Build debug kernel with KASAN/kmemleak/KFENCE for experimentation.
2. **Hands-on exercises**: Practice page faults, OOM, THP, SLUB, KASAN, and memory pressure.
3. **Community**: linux-mm@kvack.org is the primary forum; LWN.net for articles.
4. **Conferences**: Linux Plumbers, LSF/MM for cutting-edge MM discussions.
5. **Contributing**: Start with small fixes, run selftests, review patches, submit gradually.
6. **Tools**: Master the diagnostic toolkit from simple (`free -h`) to advanced (`perf mem`).

---

*Next: [Chapter 36 — Interview Preparation: 50+ Questions and Answers](Chapter_36_Interview_Questions.md)*
