# Chapter 13: Memory Management

## Learning Goals
- Understand RTOS memory management challenges and constraints
- Master FreeRTOS heap allocation schemes (heap_1 through heap_5)
- Learn static vs dynamic memory allocation trade-offs
- Know memory pool (fixed-block) allocators
- Understand memory fragmentation and prevention
- Compare memory management across RTOS platforms

---

## 1. RTOS Memory Constraints

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Why RTOS memory management differs from GPOS:           │
  │                                                           │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ 1. Limited RAM: 16KB - 512KB typical MCU       │      │
  │  │ 2. No MMU/virtual memory (Cortex-M)            │      │
  │  │ 3. Deterministic timing: malloc must be O(1)   │      │
  │  │ 4. No fragmentation tolerance: can't compact   │      │
  │  │ 5. Safety-critical: allocation failure = hazard│      │
  │  │ 6. MISRA/IEC61508: often prohibit dynamic alloc│      │
  │  │ 7. No swap/disk backing                        │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Memory usage in typical RTOS application:               │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │ SRAM (128KB example):                            │     │
  │  │                                                   │     │
  │  │ ┌─────────────┐ 0x2000_0000                      │     │
  │  │ │ .data       │ ~2KB (initialized globals)       │     │
  │  │ ├─────────────┤                                   │     │
  │  │ │ .bss        │ ~4KB (zero-initialized globals)  │     │
  │  │ ├─────────────┤                                   │     │
  │  │ │ RTOS Heap   │ ~64KB (task stacks, queues, etc) │     │
  │  │ │ (ucHeap[])  │                                   │     │
  │  │ ├─────────────┤                                   │     │
  │  │ │ ISR Stack   │ ~2KB (MSP)                       │     │
  │  │ │ (MSP)       │                                   │     │
  │  │ └─────────────┘ 0x2002_0000                      │     │
  │  │                                                   │     │
  │  │ Leftover: ~56KB for application data              │     │
  │  └─────────────────────────────────────────────────┘     │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. FreeRTOS Heap Implementations

```
  FreeRTOS provides 5 heap implementations — choose one at link time
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  heap_1:  Allocate only, NEVER free                      │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ · Simplest: bump pointer allocator              │      │
  │  │ · pvPortMalloc increments pointer, that's it    │      │
  │  │ · vPortFree does nothing (no-op)                │      │
  │  │ · Zero fragmentation risk                       │      │
  │  │ · Deterministic O(1)                            │      │
  │  │ · Use when: all objects created at startup       │      │
  │  │ · MISRA-friendly, safety-critical               │      │
  │  │                                                  │      │
  │  │ ┌──────┬──────┬──────┬────────────────┐        │      │
  │  │ │TCB_A │Stack │Queue │ ← free space → │        │      │
  │  │ └──────┴──────┴──────┴────────────────┘        │      │
  │  │  ▲ next alloc here                              │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  heap_2:  Allocate and free, NO merging                  │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ · Best-fit free list                            │      │
  │  │ · Free blocks NOT merged (fragmentation risk)   │      │
  │  │ · Good when: alloc/free same-size blocks always │      │
  │  │ · O(n) allocation (search free list)            │      │
  │  │ · Deprecated — use heap_4 instead               │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  heap_3:  Wrapper around standard C malloc/free          │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ · Uses compiler's malloc() with thread-safety   │      │
  │  │ · Wraps in vTaskSuspendAll/xTaskResumeAll       │      │
  │  │ · Heap size = linker-configured heap             │      │
  │  │ · Non-deterministic (depends on C library impl) │      │
  │  │ · Use when: using C++ new/delete, need stdlib   │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  heap_4:  Allocate, free, with MERGE (coalesce)          │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ · First-fit with adjacent block merging         │      │
  │  │ · Reduces fragmentation significantly           │      │
  │  │ · Most popular choice for general use           │      │
  │  │ · O(n) allocation (walk free list)              │      │
  │  │ · Supports malloc failed hook                   │      │
  │  │ · Single contiguous heap array                  │      │
  │  │                                                  │      │
  │  │ ┌─────┬ free ┬─────┬ free ┬─────┐              │      │
  │  │ │used │ ░░░░ │used │ ░░░░ │used │ Before free  │      │
  │  │ └─────┴──────┴─────┴──────┴─────┘              │      │
  │  │ ┌─────┬ free ┬ free ░░░░░░─────┐              │      │
  │  │ │used │ ░░░░░░░░░░░░░░░░ │used │ After merge  │      │
  │  │ └─────┴──────────────────┴─────┘              │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  heap_5:  Like heap_4 but MULTIPLE non-contiguous regions│
  │  ┌────────────────────────────────────────────────┐      │
  │  │ · Spans multiple memory regions                 │      │
  │  │ · Example: internal SRAM + external SDRAM       │      │
  │  │ · Must call vPortDefineHeapRegions() at startup │      │
  │  │ · Otherwise same as heap_4 (first-fit+merge)    │      │
  │  │                                                  │      │
  │  │ Region 1 (SRAM):   ┌────────────────┐           │      │
  │  │                     │ heap blocks    │           │      │
  │  │                     └────────────────┘           │      │
  │  │ Region 2 (SDRAM):  ┌────────────────────┐       │      │
  │  │                     │ heap blocks        │       │      │
  │  │                     └────────────────────┘       │      │
  │  └────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Memory Pool (Fixed-Block) Allocator

```c
/*
 * Memory Pool: pre-allocate N blocks of fixed size
 * O(1) alloc and free, ZERO fragmentation
 * Ideal for RTOS where objects are same size (e.g., CAN frames)
 */

/* Simple memory pool implementation */
#define POOL_BLOCK_SIZE  64
#define POOL_BLOCK_COUNT 32

typedef struct MemPool {
    uint8_t  memory[POOL_BLOCK_COUNT][POOL_BLOCK_SIZE];
    uint8_t *freeList[POOL_BLOCK_COUNT];
    volatile uint32_t freeCount;
    SemaphoreHandle_t xMutex;
} MemPool_t;

static MemPool_t pool;

void pool_init(MemPool_t *p) {
    p->freeCount = POOL_BLOCK_COUNT;
    for (int i = 0; i < POOL_BLOCK_COUNT; i++) {
        p->freeList[i] = p->memory[i];
    }
    p->xMutex = xSemaphoreCreateMutex();
}

void *pool_alloc(MemPool_t *p) {
    void *block = NULL;
    xSemaphoreTake(p->xMutex, portMAX_DELAY);
    if (p->freeCount > 0) {
        block = p->freeList[--p->freeCount];
    }
    xSemaphoreGive(p->xMutex);
    return block;  /* NULL if pool exhausted */
}

void pool_free(MemPool_t *p, void *block) {
    xSemaphoreTake(p->xMutex, portMAX_DELAY);
    p->freeList[p->freeCount++] = (uint8_t *)block;
    xSemaphoreGive(p->xMutex);
}

/*
 * Memory Pool Advantages:
 * ┌────────────────────────────────────────────────┐
 * │ + O(1) allocation and deallocation             │
 * │ + Zero fragmentation (all blocks same size)    │
 * │ + Deterministic (constant time, always)        │
 * │ + Bounded memory usage (N × block_size)        │
 * │ + MISRA/safety compliant                       │
 * │ - Wastes memory if blocks vary in size         │
 * │ - Fixed at compile time (inflexible)           │
 * │ - Need separate pools for each block size      │
 * └────────────────────────────────────────────────┘
 *
 * Zephyr: k_mem_slab (built-in memory pool)
 * VxWorks: memPartLib with fixed partitions
 */

/* Zephyr k_mem_slab */
K_MEM_SLAB_DEFINE(can_frame_pool, sizeof(CanFrame_t), 32, 4);

void *frame;
if (k_mem_slab_alloc(&can_frame_pool, &frame, K_MSEC(100)) == 0) {
    /* Use frame */
    k_mem_slab_free(&can_frame_pool, &frame);
}
```

---

## 4. Static vs Dynamic Allocation

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Static Allocation (compile-time):                       │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ All memory determined at build time             │      │
  │  │ No malloc, no heap, no runtime allocation       │      │
  │  │ FreeRTOS: configSUPPORT_DYNAMIC_ALLOCATION = 0  │      │
  │  │           configSUPPORT_STATIC_ALLOCATION = 1   │      │
  │  │ Use xTaskCreateStatic, xQueueCreateStatic, etc. │      │
  │  │                                                  │      │
  │  │ Advantages:                                     │      │
  │  │ · Cannot fail at runtime (no out-of-memory)     │      │
  │  │ · Linker verifies total fits in RAM             │      │
  │  │ · MISRA Rule 21.3 compliant (no stdlib malloc)  │      │
  │  │ · Required for IEC 61508 SIL 3/4, DO-178C       │      │
  │  │ · Easy to analyze memory usage (map file)       │      │
  │  │                                                  │      │
  │  │ Disadvantages:                                   │      │
  │  │ · Inflexible: can't adapt to runtime conditions │      │
  │  │ · May waste memory (worst-case sizing)          │      │
  │  │ · More boilerplate code                         │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Dynamic Allocation (runtime):                           │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ Memory allocated from heap at runtime           │      │
  │  │ FreeRTOS: pvPortMalloc / vPortFree              │      │
  │  │                                                  │      │
  │  │ Advantages:                                     │      │
  │  │ · Flexible: allocate only what's needed         │      │
  │  │ · Simpler code (no static buffers)              │      │
  │  │ · Supports runtime-determined configurations    │      │
  │  │                                                  │      │
  │  │ Disadvantages:                                   │      │
  │  │ · Can fail at runtime (out of memory)           │      │
  │  │ · Fragmentation over time                       │      │
  │  │ · Non-deterministic (heap_3, heap_4: O(n))      │      │
  │  │ · Not MISRA compliant                           │      │
  │  │ · Memory leaks if not carefully managed          │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Best practice: allocate everything at init, never free  │
  │  (heap_1 philosophy, even with heap_4)                   │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Fragmentation

```
  Memory Fragmentation in RTOS
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  External Fragmentation:                                 │
  │  Total free memory sufficient, but no single block large │
  │  enough for the request.                                 │
  │                                                           │
  │  Before: ┌─────┬ free ┬─────┬ free ┬─────┬ free ┐      │
  │          │ 32B │ 16B  │ 64B │ 16B  │ 32B │ 16B  │      │
  │          └─────┴──────┴─────┴──────┴─────┴──────┘      │
  │  Free total: 48 bytes. Request: 32 bytes. FAILS!        │
  │  (largest contiguous free block = 16 bytes)              │
  │                                                           │
  │  Internal Fragmentation:                                 │
  │  Allocated block larger than needed (alignment/minimum)  │
  │                                                           │
  │  Request: 5 bytes → Allocated: 8 bytes (aligned)        │
  │  Wasted: 3 bytes per allocation                          │
  │                                                           │
  │  Anti-fragmentation strategies:                          │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ 1. Memory pools: fixed-size blocks (zero frag) │      │
  │  │ 2. heap_4/heap_5: adjacent block merging        │      │
  │  │ 3. Allocate at init only, never free            │      │
  │  │ 4. Use similar-sized allocations                │      │
  │  │ 5. Static allocation (no heap at all)           │      │
  │  │ 6. Separate heaps for different object sizes    │      │
  │  └────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. Memory Management Comparison

| Feature | FreeRTOS | Zephyr | QNX | VxWorks |
|---------|----------|--------|-----|---------|
| **Heap** | 5 schemes (heap_1-5) | System heap + k_heap | Process heap (libc) | memPartLib |
| **Memory pool** | Manual/heap_2 | k_mem_slab | Fixed pool API | memPartLib |
| **Static alloc** | CreateStatic APIs | Default (K_xxx_DEFINE) | N/A | N/A |
| **MPU protection** | Optional | Optional | MMU (full VM) | Optional/RTP |
| **Deterministic** | heap_1 (O(1)) | k_mem_slab (O(1)) | No (libc) | Partition (O(1)) |
| **MISRA compliant** | heap_1 + static | Static macros | N/A | Restricted |

---

## Interview Questions

**Q1: Compare FreeRTOS heap_1, heap_4, and heap_5. When would you use each?**
**A:** heap_1: bump allocator, allocate-only (no free). O(1), zero fragmentation. Use for: safety-critical systems where all objects are created at startup and never deleted. MISRA-friendly. heap_4: first-fit with adjacent block merging. Most general-purpose. Handles alloc/free patterns well. Reduces fragmentation via coalescing. Use for: general development, prototyping, systems that create/delete tasks dynamically. heap_5: same as heap_4 but spans multiple non-contiguous memory regions (e.g., internal SRAM + external SDRAM). Use for: MCUs with multiple RAM banks, systems with external memory. All three: determinism decreases from heap_1 (always O(1)) to heap_4/5 (O(n) free list traversal, though bounded by total heap size).

**Q2: Why do safety standards like MISRA and IEC 61508 prohibit or restrict dynamic memory allocation?**
**A:** (1) Runtime failure: malloc can return NULL at unpredictable times — in safety-critical code, every failure path must be analyzed and handled. (2) Fragmentation: over long runtimes (years in automotive/industrial), fragmentation can cause allocation failure even with sufficient total memory. (3) Non-deterministic timing: allocation time varies with heap state — violates worst-case execution time guarantees. (4) Memory leaks: if a free is missed, memory is lost permanently — no OS reclamation on MCU. (5) Difficult to verify: static analysis tools cannot easily prove memory safety with dynamic allocation. Alternative: pre-allocate everything statically, use memory pools, or allocate at init and never free (heap_1 pattern).

**Q3: Explain memory pools and when they are superior to general heap allocation.**
**A:** Memory pool pre-allocates N blocks of identical fixed size. Allocation = pop from free list (O(1)). Free = push to free list (O(1)). Zero fragmentation since all blocks are the same size. Superior when: (1) many same-sized objects are allocated/freed frequently (CAN message buffers, network packets, sensor readings), (2) deterministic timing is required (O(1) guaranteed), (3) safety-critical systems need bounded memory usage, (4) long-running systems where fragmentation would accumulate. Trade-off: wastes memory if actual data sizes vary significantly. Solution: multiple pools for different size classes — similar to Linux's slab allocator.

---

## Summary

- RTOS memory: limited RAM, no virtual memory, must be deterministic
- FreeRTOS heap_1 (allocate-only), heap_4 (alloc+free+merge), heap_5 (multi-region)
- Static allocation: compile-time, never fails, MISRA/safety compliant
- Memory pools: fixed-size blocks, O(1), zero fragmentation — ideal for RTOS
- Fragmentation: biggest risk with dynamic allocation on long-running systems
- Safety standards prohibit/restrict malloc: use static or pool allocators
- Best practice: allocate at init, never free (even with heap_4)

---

[Previous Chapter: Priority Inversion ←](Chapter_12_Priority_Inversion.md) | [Next Chapter: Real-Time Memory Patterns →](Chapter_14_RT_Memory.md)
