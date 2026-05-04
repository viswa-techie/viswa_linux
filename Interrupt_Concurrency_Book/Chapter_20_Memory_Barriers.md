# Chapter 20: Memory Barriers and Ordering

## Learning Goals
- Understand why CPUs and compilers reorder memory operations
- Know the Linux kernel barrier API (smp_mb, smp_wmb, smp_rmb)
- Use READ_ONCE / WRITE_ONCE correctly
- Understand acquire/release semantics
- Avoid subtle ordering bugs in lock-free and driver code

---

## 20.1 Why Memory Ordering Matters

Modern CPUs aggressively reorder loads and stores for performance. What you write in C is NOT what the CPU executes.

```
Source code:                 What CPU might actually do:
  x = 1;                      STORE y = 1   ← reordered!
  y = 1;                      STORE x = 1

Why? Store buffer, cache hierarchy, out-of-order execution.
```

### Two Sources of Reordering

```
1. Compiler reordering:
   - GCC/Clang move instructions for optimization
   - Fix: barrier() or volatile/READ_ONCE/WRITE_ONCE

2. CPU reordering:
   - Out-of-order execution, store buffers, invalidation queues
   - Fix: Hardware memory barriers (mfence, dmb, dsb)
   - In kernel: smp_mb(), smp_wmb(), smp_rmb()
```

---

## 20.2 CPU Memory Models

```
Architecture  │ Model              │ Reordering Allowed
──────────────┼────────────────────┼──────────────────────────────
x86/x86_64    │ TSO (Total Store   │ StoreLoad only
              │ Order)             │ (loads can pass earlier stores)
ARM64         │ Weakly ordered     │ All: LoadLoad, LoadStore,
              │                    │      StoreLoad, StoreStore
RISC-V        │ RVWMO (weak)       │ All types
PowerPC       │ Weakly ordered     │ All types
Alpha         │ Very weak          │ All + value speculation

Result: Code correct on x86 may be BROKEN on ARM64!
        Kernel must use portable barrier APIs.
```

### Reordering Types

```
StoreStore: CPU reorders two writes
  STORE A=1; STORE B=1; → CPU may do STORE B=1 first
  Fix: smp_wmb()

LoadLoad: CPU reorders two reads
  LOAD A; LOAD B; → CPU may load B before A
  Fix: smp_rmb()

LoadStore: CPU reorders load before a later store
  LOAD A; STORE B=1; → CPU may store B before loading A
  Fix: smp_mb()

StoreLoad: CPU reorders store before a later load
  STORE A=1; LOAD B; → CPU may load B before storing A
  Fix: smp_mb() (most expensive barrier)
```

---

## 20.3 Compiler Barriers

### barrier()

```c
/* Prevent compiler from reordering across this point */
barrier();

/* Example: */
flags = READY;
barrier();      /* Compiler must emit store of 'flags' before 'data' read */
val = data;
```

### READ_ONCE() and WRITE_ONCE()

```c
/* Prevent compiler optimizations on shared variables: */

/* Without READ_ONCE — compiler may cache in register: */
while (done == 0)   /* May become infinite loop — compiler optimizes to
    ;                   single load + branch */

/* With READ_ONCE — forces re-read from memory each time: */
while (READ_ONCE(done) == 0)
    cpu_relax();

/* WRITE_ONCE — prevents store tearing and coalescing: */
WRITE_ONCE(shared_ptr, new_ptr);
```

### What READ_ONCE / WRITE_ONCE Prevent

```
1. Load tearing:  Reading a value in multiple non-atomic loads
2. Store tearing:  Writing a value in multiple non-atomic stores
3. Load fusing:   Compiler caches value in register, never re-reads
4. Store fusing:  Compiler merges multiple stores into one
5. Invented loads: Compiler reads a value it doesn't need to
6. Invented stores: Compiler writes a value that wasn't in source

Rule: ALL accesses to shared variables (not protected by locks)
      MUST use READ_ONCE() / WRITE_ONCE().
```

---

## 20.4 The Linux Kernel Barrier API

### Full Barrier

```c
smp_mb();     /* Full memory barrier: prevents all reordering across it */
              /* x86: mfence or lock addl $0, (%rsp) */
              /* ARM64: dmb ish */
```

```
  STORE x = 1
  smp_mb()       ← All stores before are visible before all loads/stores after
  LOAD y
  STORE z = 1
```

### Write Barrier

```c
smp_wmb();    /* Write barrier: prevents StoreStore reordering */
              /* x86: nop (TSO provides this) */
              /* ARM64: dmb ishst */
```

```
  STORE x = 1
  smp_wmb()      ← Ensures x=1 is visible before y=1
  STORE y = 1
```

### Read Barrier

```c
smp_rmb();    /* Read barrier: prevents LoadLoad reordering */
              /* x86: nop (TSO provides this) */
              /* ARM64: dmb ishld */
```

```
  LOAD x
  smp_rmb()      ← Ensures x is read before y
  LOAD y
```

### Complete Barrier Family

```
Barrier               │ Prevents              │ SMP  │ UP
──────────────────────┼───────────────────────┼──────┼──────
smp_mb()              │ All reordering        │ Yes  │ barrier()
smp_wmb()             │ StoreStore            │ Yes  │ barrier()
smp_rmb()             │ LoadLoad              │ Yes  │ barrier()
smp_store_release()   │ All before → store    │ Yes  │ barrier()
smp_load_acquire()    │ Load → all after      │ Yes  │ barrier()
mb()                  │ All (always, even UP) │ Yes  │ Yes
wmb()                 │ StoreStore (always)   │ Yes  │ Yes
rmb()                 │ LoadLoad (always)     │ Yes  │ Yes
barrier()             │ Compiler only         │ N/A  │ N/A
```

Note: smp_* variants are no-ops on uniprocessor (UP) builds — they compile to just `barrier()`. Non-smp versions (mb/wmb/rmb) always emit hardware barriers.

---

## 20.5 Acquire and Release Semantics

Modern preferred approach — lighter than full barriers.

```
smp_store_release(&var, val):
  All memory operations BEFORE this store are completed
  before the store becomes visible. (One-way fence ↑)

smp_load_acquire(&var):
  The load completes before any memory operations AFTER it.
  (One-way fence ↓)
```

```
Classic Producer-Consumer:

  Producer:                          Consumer:
  ─────────                          ─────────
  WRITE_ONCE(data, result);          while (!(d = smp_load_acquire(&flag)))
  smp_store_release(&flag, 1);           cpu_relax();
  ↑ data store completes             ↓ flag load completes
    before flag becomes visible         before data is read
                                     val = READ_ONCE(data);  /* Sees result */
```

### Why Prefer Acquire/Release Over Full Barriers?

```
1. More efficient on weak architectures:
   - smp_store_release: ARM64 → stlr (store-release)
   - smp_load_acquire:  ARM64 → ldar (load-acquire)
   - smp_mb:            ARM64 → dmb ish (heavier)

2. Clearer intent:
   - Documents which direction the fence applies
   - Easier for reviewers to verify correctness

3. Compose correctly:
   - Acquire on reader + Release on writer = correctly ordered
```

---

## 20.6 Barrier Pairing Rules

Barriers must be PAIRED between CPUs to be effective:

```
CPU 0                          CPU 1
─────                          ─────
STORE x = 1                   LOAD y (sees 1)
smp_wmb()        ←PAIRED→     smp_rmb()
STORE y = 1                   LOAD x (guaranteed to see 1)
```

```
VALID pairings:
  smp_wmb()  ←→  smp_rmb()     (write barrier pairs with read barrier)
  smp_mb()   ←→  smp_mb()      (full barriers pair with each other)
  smp_store_release() ←→ smp_load_acquire()  (release pairs with acquire)

INVALID:
  smp_wmb() on CPU0 alone without smp_rmb() on CPU1
  → CPU1 may still see stores in wrong order!
```

---

## 20.7 Common Patterns

### Pattern 1: Publishing a Data Structure

```c
/* Writer (CPU 0): */
struct data *new = kmalloc(sizeof(*new), GFP_KERNEL);
new->field1 = val1;
new->field2 = val2;
smp_store_release(&global_ptr, new);
/* Release ensures all fields written before pointer published */

/* Reader (CPU 1): */
struct data *p = smp_load_acquire(&global_ptr);
/* Acquire ensures pointer loaded before accessing fields */
if (p) {
    use(p->field1);    /* Guaranteed to see val1 */
    use(p->field2);    /* Guaranteed to see val2 */
}
```

### Pattern 2: Ring Buffer

```c
struct ring {
    unsigned int head;  /* Writer advances */
    unsigned int tail;  /* Reader advances */
    void *data[SIZE];
};

/* Producer: */
ring->data[ring->head % SIZE] = item;
smp_store_release(&ring->head, ring->head + 1);
/* Data stored before head advances */

/* Consumer: */
unsigned int head = smp_load_acquire(&ring->head);
if (head != ring->tail) {
    item = ring->data[ring->tail % SIZE];
    smp_store_release(&ring->tail, ring->tail + 1);
}
```

### Pattern 3: Flag Signaling

```c
/* Completion signaling without locks: */

/* Thread A: */
do_work();
WRITE_ONCE(result, computed_value);
smp_wmb();                        /* Result before flag */
WRITE_ONCE(done, true);

/* Thread B: */
while (!READ_ONCE(done))
    cpu_relax();
smp_rmb();                        /* Flag before result read */
use(READ_ONCE(result));           /* Sees computed_value */
```

---

## 20.8 Device I/O Barriers

For MMIO (memory-mapped I/O), standard barriers are insufficient. Use I/O barriers:

```c
/* writel/readl have implicit barriers on most architectures */
writel(val, reg);     /* Includes wmb() before and after on ARM */
readl(reg);           /* Includes rmb() */

/* Relaxed variants (no barriers, faster): */
writel_relaxed(val, reg);
readl_relaxed(reg);

/* When using relaxed, add explicit I/O barriers: */
writel_relaxed(cmd, dev->ctrl_reg);
__iowmb();           /* Ensure write reaches device */
status = readl_relaxed(dev->status_reg);
```

### DMA Ordering

```c
/* Before starting DMA: ensure CPU writes are visible to device */
dma_wmb();    /* Lightweight DMA write barrier */

/* After DMA completes: ensure device writes visible to CPU */
dma_rmb();    /* Lightweight DMA read barrier */

/* Example: DMA descriptor ring */
desc->addr = dma_addr;
desc->len = buffer_len;
dma_wmb();              /* Fields written before ownership flag */
desc->flags = DESC_OWN; /* Tell hardware it owns this descriptor */
```

---

## 20.9 Barriers in Locking Primitives

Locks implicitly contain barriers:

```
spin_lock():
  ┌───────────────────┐
  │ ACQUIRE semantics │  ← All memory ops after lock acquisition
  └───────────────────┘     cannot move before the lock

  /* Critical section: fully ordered */

  ┌───────────────────┐
  │ RELEASE semantics │  ← All memory ops before unlock
  └───────────────────┘     cannot move after the unlock
spin_unlock():

Result: If you hold a lock, you do NOT need additional barriers
        for data protected by that lock.
```

```
Which primitives have implicit barriers:
  spin_lock/unlock          → acquire/release
  mutex_lock/unlock         → acquire/release
  down/up (semaphore)       → acquire/release
  atomic_dec_and_test()     → full barrier
  test_and_set_bit()        → full barrier
  smp_store_release()       → release
  smp_load_acquire()        → acquire
```

---

## 20.10 Debugging Ordering Bugs

Ordering bugs are insidious — they may only manifest on weak architectures under heavy load.

### Tools

```
1. KCSAN (Kernel Concurrency Sanitizer):
   - Detects data races at runtime
   - CONFIG_KCSAN=y
   - Reports missing READ_ONCE/WRITE_ONCE and barriers

2. KTSAN (Kernel Thread Sanitizer):
   - Detects ordering violations
   - Experimental

3. Litmus tests (tools/memory-model/):
   - Formal verification of memory ordering
   - herd7 tool simulates barrier correctness
   - Example: tools/memory-model/litmus-tests/

4. Sparse (__rcu annotation):
   - Flags accesses to RCU-protected data without proper API
```

### Common Bugs

```
Bug                               │ Fix
──────────────────────────────────┼──────────────────────
Polling without READ_ONCE         │ while(READ_ONCE(flag))
Missing smp_wmb between data/flag │ Add smp_wmb() or use
                                  │ smp_store_release()
Using mb() instead of smp_mb()    │ Use smp_mb() (lighter on UP)
Barrier without matching pair     │ Pair wmb←→rmb on both CPUs
MMIO without writel (using raw)   │ Use writel/readl
DMA descriptor without dma_wmb    │ Add dma_wmb() before ownership
```

---

## Kernel Source References

```
Memory model:
  tools/memory-model/              ← Formal memory model & litmus tests
  Documentation/memory-barriers.txt← THE definitive reference (1000+ lines)
  include/linux/compiler.h         ← barrier(), READ_ONCE, WRITE_ONCE
  include/asm-generic/barrier.h    ← Generic barrier definitions
  arch/x86/include/asm/barrier.h   ← x86 barriers
  arch/arm64/include/asm/barrier.h ← ARM64 barriers

KCSAN:
  kernel/kcsan/                    ← Concurrency sanitizer
```

---

## Interview Questions

1. **Why do CPUs reorder memory operations? What hardware features cause it?**
2. **What is the x86 TSO model? Which reordering does x86 allow?**
3. **Explain the difference between smp_mb(), smp_wmb(), and smp_rmb().**
4. **Why must barriers be paired between CPUs?**
5. **What is READ_ONCE()? What optimizations does it prevent?**
6. **Explain smp_store_release() and smp_load_acquire(). Why prefer them?**
7. **Does code correct on x86 guarantee correctness on ARM64? Why not?**
8. **What barriers are implicit in spin_lock/spin_unlock?**
9. **What is dma_wmb()? When is it needed?**
10. **How does KCSAN help detect data races?**
11. **Write a correct lock-free producer-consumer using barriers.**

---

## Summary

- CPUs reorder loads/stores for performance; compilers reorder for optimization
- READ_ONCE/WRITE_ONCE: prevent compiler from caching/tearing shared variables
- smp_wmb: ensures write ordering; smp_rmb: read ordering; smp_mb: full ordering
- Barriers must be PAIRED between CPUs (wmb on writer ↔ rmb on reader)
- smp_store_release + smp_load_acquire: modern, efficient, preferred over full barriers
- Locks contain implicit acquire/release barriers — no extra barriers inside critical sections
- Code correct on x86 may break on ARM64 — always use proper barrier APIs
- Documentation/memory-barriers.txt is the kernel's definitive reference

---

*Next: [Chapter 21 — Locking in IRQ Context](Chapter_21_Locking_IRQ_Context.md)*
