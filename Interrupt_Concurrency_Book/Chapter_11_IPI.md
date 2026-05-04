# Chapter 11: Inter-Processor Interrupts (IPI)

## Learning Goals
- Understand why CPUs need to interrupt each other in SMP systems
- Know the major IPI use cases: TLB shootdown, rescheduling, function calls, stop
- See IPI implementation on x86 (LAPIC) and ARM (GIC SGI)
- Understand IPI performance implications and RT considerations

---

## 11.1 Multiprocessor Systems Overview

```
SMP (Symmetric Multi-Processing):
  All CPUs are equal — any can run any task, handle any IRQ.
  CPUs share memory but have private caches.

  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
  │ CPU0 │  │ CPU1 │  │ CPU2 │  │ CPU3 │
  │ L1$  │  │ L1$  │  │ L1$  │  │ L1$  │
  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘
     └────┬─────┴─────────┴────┬────┘
          │    Shared L2/L3    │
          │    Main Memory     │
          └────────────────────┘

Problem: CPUs need to coordinate about shared state.
  CPU 0 unmaps a page → CPU 1's TLB still has old mapping!
  CPU 0 wakes a task → CPU 2 needs to reschedule!
  
Solution: Inter-Processor Interrupts (IPI)
  One CPU sends an interrupt TO another CPU.
```

---

## 11.2 IPI Concept

```
IPI = one CPU interrupting another CPU for coordination.

NOT device-driven. Purely software-directed, CPU-to-CPU.

  CPU 0 (sender)                CPU 2 (target)
  ──────────────                ────────────────
  Need CPU 2 to flush TLB
       │
       ▼
  Write to IPI register ──────→ IRQ arrives on CPU 2
  (LAPIC ICR on x86,            │
   GICD_SGIR on ARM)            ▼
                              IPI handler executes
                              (flush TLB entries)
                                │
                                ▼
                              Resume normal execution
```

### Hardware Mechanism

```
x86: Local APIC ICR (Interrupt Command Register)
  Write destination APIC ID + vector to ICR
  → Bus message to target LAPIC → target CPU interrupted

ARM: GIC SGI (Software Generated Interrupt)
  Write to GICD_SGIR register:
    Target CPU list + SGI ID (0-15)
  → GIC delivers SGI to target CPU(s)
  IDs 0-15 reserved for SGI (16 possible IPI types)
```

---

## 11.3 IPI Usage in Linux Kernel

### IPI Types and Functions

```
Use Case              │ x86 Vector/Function        │ ARM SGI ID
──────────────────────┼────────────────────────────┼───────────
Reschedule            │ RESCHEDULE_VECTOR          │ SGI 7
                      │ smp_send_reschedule()      │
──────────────────────┼────────────────────────────┼───────────
Function call         │ CALL_FUNCTION_VECTOR       │ SGI 1
                      │ smp_call_function()        │
                      │ smp_call_function_single() │
──────────────────────┼────────────────────────────┼───────────
TLB shootdown         │ (part of function call)    │ SGI (via
                      │ flush_tlb_others()         │ function call)
──────────────────────┼────────────────────────────┼───────────
Stop CPU              │ smp_send_stop()            │ SGI 0
                      │ (for panic/reboot)         │
──────────────────────┼────────────────────────────┼───────────
IRQ work              │ IRQ_WORK_VECTOR            │ SGI 2
                      │ irq_work_queue_on()        │
```

### Reschedule IPI

```c
/* When the scheduler decides a task should run on another CPU: */
/* kernel/sched/core.c */
void resched_curr(struct rq *rq)
{
    /* If target CPU != current CPU */
    if (cpu != smp_processor_id())
        smp_send_reschedule(cpu);
        /* Sends IPI → target CPU's scheduler_ipi() handler
           → Sets TIF_NEED_RESCHED → preemption check → schedule() */
}

/* Flow:
   CPU 0: wake_up_process(task) → task should run on CPU 2
     → smp_send_reschedule(CPU 2) → IPI to CPU 2
   CPU 2: receives IPI → scheduler_ipi()
     → set TIF_NEED_RESCHED → return from interrupt
     → preemption check → schedule() → runs the woken task
*/
```

### Function Call IPI

```c
/* Execute a function on all other CPUs: */
smp_call_function(func, info, wait);
/* func: function to execute on each remote CPU
   info: argument passed to func
   wait: 1 = wait for all CPUs to complete, 0 = fire and forget */

/* Execute on a specific CPU: */
smp_call_function_single(cpu, func, info, wait);

/* Execute on specific CPUs (mask): */
smp_call_function_many(&cpumask, func, info, wait);

/* Example: Read a register on all CPUs */
static void read_msr_on_cpu(void *info)
{
    u64 *val = info;
    *val = rdmsr(MSR_IA32_TSC);
}

u64 tsc_values[NR_CPUS];
smp_call_function(read_msr_on_cpu, tsc_values, 1);
```

---

## 11.4 TLB Shootdown via IPI

### The Problem

```
CPU 0 unmaps a page (e.g., munmap, page migration):
  CPU 0's page table updated ✓
  CPU 0's TLB flushed ✓
  CPU 1's TLB still has OLD mapping!
  CPU 2's TLB still has OLD mapping!
  
  If CPU 1 accesses the unmapped page using stale TLB:
    → Accesses freed/remapped memory → CORRUPTION!
  
  Solution: CPU 0 sends IPI to CPU 1, 2 → "flush your TLB"
  This is called "TLB shootdown."
```

### TLB Shootdown Flow

```
  CPU 0                     CPU 1              CPU 2
  ─────                     ─────              ─────
  munmap(addr)
  update page table
  flush local TLB
  send IPI to CPU 1,2 ────→ IPI arrives        IPI arrives
                            │                   │
                            ▼                   ▼
                         flush TLB            flush TLB
                         for addr range       for addr range
                            │                   │
                            ▼                   ▼
                         ACK ──────────────→ CPU 0 continues
                                             (all CPUs consistent)
```

### Linux Implementation

```c
/* mm/tlb.c — simplified TLB shootdown */
void flush_tlb_mm_range(struct mm_struct *mm,
                        unsigned long start, unsigned long end)
{
    /* Determine which CPUs are running this mm */
    cpumask_t cpumask;
    cpumask_and(&cpumask, mm_cpumask(mm), cpu_online_mask);
    
    /* Flush local TLB */
    local_flush_tlb_range(start, end);
    
    /* Send IPI to other CPUs running this mm */
    if (!cpumask_empty(&cpumask))
        smp_call_function_many(&cpumask, flush_tlb_func, info, 1);
        /* wait=1: must wait for all CPUs to flush */
}

/* x86 optimization: PCID/INVPCID reduces shootdown frequency */
/* ARM: ASID per-process TLB tags also reduce shootdowns */
```

### Performance Impact

```
TLB shootdown cost:
  IPI send + propagation:     ~1-5 µs
  Remote TLB flush:           ~0.5-2 µs per CPU
  Synchronization overhead:   ~1-3 µs
  
  Total for 4 CPUs: ~5-15 µs per shootdown

Heavy mmap/munmap workloads → many shootdowns → performance impact
  Mitigations:
    - Batch TLB flushes (flush_tlb_batched)
    - PCID (x86) / ASID (ARM) reduce flush scope
    - Lazy TLB: don't flush if CPU not running the mm
    - Huge pages: fewer page table entries to invalidate
```

---

## IPI on PREEMPT_RT

```
IPIs always run in hardirq context (IRQF_NO_THREAD):
  - Cannot be deferred or threaded
  - Must be fast (no sleeping)
  - TLB shootdowns unavoidable — source of RT latency!

RT mitigation:
  - Minimize shared memory between RT and non-RT tasks
  - Use per-CPU data where possible (no TLB shootdown needed)
  - Isolate RT CPUs to reduce IPI frequency
  - Batch TLB operations on non-RT CPUs
```

---

## Kernel Source References

```
IPI infrastructure:
  arch/x86/kernel/smp.c          ← x86 IPI send/receive
  arch/arm64/kernel/smp.c        ← ARM64 IPI implementation
  kernel/smp.c                   ← smp_call_function, generic IPI

TLB shootdown:
  arch/x86/mm/tlb.c              ← x86 TLB flush + IPI
  arch/arm64/mm/context.c        ← ARM64 ASID management
  mm/tlb.c                       ← Generic TLB batching

Reschedule IPI:
  kernel/sched/core.c            ← resched_curr, smp_send_reschedule
```

---

## Interview Questions

1. **What is an IPI and why is it needed in SMP systems?**
2. **Name four uses of IPI in the Linux kernel.**
3. **What is a TLB shootdown? Walk through the complete flow.**
4. **How does smp_call_function() work internally?**
5. **What hardware mechanism sends IPIs on x86 (LAPIC) and ARM (GIC)?**
6. **Why are IPIs always handled in hardirq context, even on PREEMPT_RT?**
7. **What is the performance cost of TLB shootdowns?**
8. **How do PCID (x86) and ASID (ARM) reduce TLB shootdown frequency?**
9. **What happens during a reschedule IPI?**
10. **How would excessive IPI traffic affect system performance?**
11. **What is lazy TLB mode and how does it help?**
12. **Can IPI cause deadlocks? Under what circumstances?**

---

## Summary

- IPIs enable CPU-to-CPU coordination in SMP systems — essential for consistency
- Four main uses: rescheduling, remote function execution, TLB shootdown, CPU stop
- TLB shootdown is the most performance-critical IPI — ensures memory mapping consistency
- IPIs run in hardirq context (fast, not threadable) — a source of RT latency
- Modern hardware (PCID/ASID) reduces the frequency of full TLB flushes

---

*Next: [Chapter 12 — Timer Interrupts](Chapter_12_Timer_Interrupts.md)*
