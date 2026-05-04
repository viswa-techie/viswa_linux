# Chapter 10: Context Switching

## Learning Goals
- Understand the full context switch path in Linux
- Know exactly what hardware state is saved and restored
- Measure and minimize context switch overhead

---

## 10.1 Context Switch Concept

A **context switch** replaces the currently executing task with another one.

```
CPU running Task A          Context Switch           CPU running Task B
┌──────────────────┐      ┌──────────────────┐     ┌──────────────────┐
│ A's registers    │ save │ A's state → mem  │ rstr│ B's registers    │
│ A's stack ptr    │─────►│ B's state ← mem  │────►│ B's stack ptr    │
│ A's page table   │      │ Switch page table│     │ B's page table   │
│ A's FPU state    │      │ Flush TLB (maybe)│     │ B's FPU state    │
└──────────────────┘      └──────────────────┘     └──────────────────┘
         ~1-5 µs on modern hardware
```

---

## 10.2 Process Context vs Interrupt Context

| Property | Process Context | Interrupt Context |
|----------|----------------|-------------------|
| Can sleep | Yes | **No** |
| Has `current` | Yes (valid task_struct) | Yes (interrupted task) |
| Can access user memory | Yes (copy_to/from_user) | **No** |
| Preemptible | Yes (if CONFIG_PREEMPT) | **No** |
| Example | Syscall handler, kthread | IRQ handler, softirq |

---

## 10.3 Kernel Context Switch Path

```
schedule()                              [kernel/sched/core.c]
    │
    ▼
__schedule(SM_NONE)
    │
    ├── Pick next task:
    │   pick_next_task(rq, prev)
    │       ├── Check stop_sched_class
    │       ├── Check dl_sched_class (deadline)
    │       ├── Check rt_sched_class (FIFO/RR)
    │       ├── Check fair_sched_class (CFS)  ← most tasks end up here
    │       └── Check idle_sched_class
    │
    ├── If prev == next → return (no switch needed)
    │
    ▼
context_switch(rq, prev, next)
    │
    ├── prepare_task_switch(rq, prev, next)
    │
    ├── Switch address space:
    │   switch_mm_irqs_off(prev->active_mm, next->mm, next)
    │       ├── ARM64: Write TTBR0_EL1 (user page table base)
    │       ├── x86_64: Write CR3 (page table base)
    │       └── ASID/PCID: avoid full TLB flush if supported
    │
    ├── Switch register state:
    │   switch_to(prev, next, prev)
    │       │
    │       ▼ (architecture-specific)
    │   __switch_to(prev, next)     [arch/arm64/kernel/process.c]
    │       ├── Save callee-saved regs (x19-x28, fp, sp, pc) to prev->thread
    │       ├── Restore callee-saved regs from next->thread
    │       ├── Switch kernel stack pointer
    │       ├── Update current (SP_EL0 or per-CPU)
    │       └── Return — now executing as 'next' task
    │
    └── finish_task_switch(prev)
            ├── Complete any deferred cleanup
            └── If prev was dying → release task_struct
```

---

## 10.4 Saving and Restoring CPU State — ARM64

```c
/* arch/arm64/kernel/entry.S — simplified */
SYM_FUNC_START(cpu_switch_to)
    /* Save current (prev) task registers */
    mov     x10, #THREAD_CPU_CONTEXT
    add     x8, x0, x10        /* x0 = prev task_struct */
    stp     x19, x20, [x8], #16
    stp     x21, x22, [x8], #16
    stp     x23, x24, [x8], #16
    stp     x25, x26, [x8], #16
    stp     x27, x28, [x8], #16
    stp     x29, lr,  [x8], #16  /* Save FP and return address */
    mov     x9, sp
    str     x9, [x8]              /* Save stack pointer */

    /* Restore next task registers */
    add     x8, x1, x10        /* x1 = next task_struct */
    ldp     x19, x20, [x8], #16
    ldp     x21, x22, [x8], #16
    ldp     x23, x24, [x8], #16
    ldp     x25, x26, [x8], #16
    ldp     x27, x28, [x8], #16
    ldp     x29, lr,  [x8], #16
    ldr     x9, [x8]
    mov     sp, x9                /* Restore stack pointer */

    /* Update current task pointer */
    msr     sp_el0, x1

    ret                           /* Return to next task's saved LR */
SYM_FUNC_END(cpu_switch_to)
```

### What's NOT Saved in `cpu_switch_to()`

- **Caller-saved registers** (x0-x18): saved on stack by calling convention
- **FPU/NEON state** (V0-V31): saved lazily on first FPU use after switch
- **User registers** (in pt_regs): saved on kernel entry (syscall/exception)

---

## 10.5 Context Switch Overhead

### Components of Overhead

```
Direct costs:
  ├── Save/restore registers:     ~50-200 ns
  ├── Switch page table (CR3/TTBR0): ~100-500 ns
  └── TLB flush (if needed):       ~0-2000 ns

Indirect costs (cache pollution):
  ├── L1 cache cold:               ~1000-5000 ns
  ├── L2 cache cold:               ~5000-20000 ns
  └── TLB refill (page walks):     ~1000-10000 ns

Total: ~1-25 µs depending on hardware and working set
```

### Measuring Context Switch Cost

```bash
# Using perf
perf stat -e context-switches,cpu-migrations -a sleep 10

# Using vmstat
vmstat 1
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs ← context switches
#  1  0      0 500000  50000 200000    0    0     0     0  500 2000

# Micro-benchmark
perf bench sched messaging
perf bench sched pipe
```

### Thread vs Process Context Switch

```
Thread switch (same process):
  ├── Save/restore registers: same
  ├── Switch page table: NOT needed (same mm)  ← BIG saving
  └── TLB flush: NOT needed
  Total: ~1-5 µs

Process switch (different process):
  ├── Save/restore registers: same
  ├── Switch page table: YES (CR3/TTBR0 write)
  └── TLB: ASID/PCID avoids full flush on modern CPUs
  Total: ~3-25 µs
```

---

## FPU State Management

```c
/* Lazy FPU switching (modern approach):
   Don't save/restore FPU on every context switch.
   Instead, save FPU state only when the next task uses FPU. */

/* ARM64: fpsimd_thread_switch() */
void fpsimd_thread_switch(struct task_struct *next)
{
    if (current->thread.fpsimd_cpu == smp_processor_id())
        /* FPU state is in registers — save to memory */
        fpsimd_save_state(&current->thread.fpsimd_state);

    /* Mark: next task's FPU state needs restore on first use */
}
```

---

## Interview Questions

**Q1: Walk through what happens during a context switch in Linux.**
A: 1) `schedule()` calls `pick_next_task()` to find the best task. 2) If different from current: `context_switch()`. 3) `switch_mm()` changes the page table base register (TTBR0/CR3). 4) `switch_to()` saves current callee-saved registers, restores next's registers, switches kernel stack. 5) `finish_task_switch()` does cleanup. CPU now executes the next task.

**Q2: Why is a thread context switch cheaper than a process switch?**
A: Threads share the same address space (mm_struct), so `switch_mm()` is skipped — no page table switch, no TLB flush. Only registers are switched. This saves ~1-20 µs of TLB refill cost per switch.

**Q3: What is ASID/PCID and how does it reduce context switch cost?**
A: ASID (ARM) / PCID (x86) tags TLB entries with an address space identifier. On context switch, the new page table base + ASID is loaded, but old TLB entries from other ASIDs remain valid. No flush needed — only entries with the new ASID are used. This avoids the expensive TLB refill cost.

**Q4: How many context switches per second is normal?**
A: For a desktop: 1000-5000/s. For a busy server: 10000-100000/s. For a real-time system: minimize to <1000/s. Very high cs rates (>100K) suggest excessive contention or too many runnable tasks. Check with `vmstat` or `perf stat -e context-switches`.

---

*Next: [Chapter 11 — Scheduler Architecture in Linux](Chapter_11_Scheduler_Architecture.md)*
