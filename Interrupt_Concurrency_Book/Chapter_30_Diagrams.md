# Chapter 30: Important Diagrams

## Learning Goals
- Visualize the complete interrupt handling flow
- Map synchronization primitive relationships
- Understand execution context hierarchy
- Reference these diagrams for study and interviews

---

## 30.1 Complete Interrupt Flow (Hardware to Handler)

```
                          HARDWARE EVENT
                               │
                               ▼
              ┌────────────────────────────────┐
              │  Interrupt Controller           │
              │  (APIC / GIC)                   │
              │  ┌──────────────────────────┐   │
              │  │ Priority / Masking       │   │
              │  │ Routing to target CPU    │   │
              │  └──────────┬───────────────┘   │
              └─────────────┼───────────────────┘
                            │ Assert IRQ signal to CPU
                            ▼
              ┌────────────────────────────────┐
              │  CPU                            │
              │  1. Finish current instruction  │
              │  2. Save flags + CS:RIP (x86)   │
              │     or PSTATE + PC (ARM64)      │
              │  3. Disable interrupts          │
              │  4. Jump to vector entry        │
              └──────────────┬─────────────────┘
                             │
                             ▼
              ┌────────────────────────────────┐
              │  Assembly Entry                 │
              │  x86: entry_64.S → asm_common   │
              │  ARM64: entry.S → el1h_64_irq   │
              │  - Save all registers (pt_regs) │
              │  - Switch to IRQ stack (x86)    │
              └──────────────┬─────────────────┘
                             │
                             ▼
              ┌────────────────────────────────┐
              │  C IRQ Entry                    │
              │  x86: common_interrupt()        │
              │  ARM64: el1_interrupt()          │
              │  → handle_arch_irq (GIC)        │
              └──────────────┬─────────────────┘
                             │
                             ▼
              ┌────────────────────────────────┐
              │  IRQ Descriptor Dispatch        │
              │  generic_handle_irq_desc()       │
              │  → desc->handle_irq(desc)        │
              │    (handle_fasteoi_irq, etc.)     │
              └──────────────┬─────────────────┘
                             │
                             ▼
              ┌────────────────────────────────┐
              │  Handler Chain                  │
              │  for each action in chain:      │
              │    action->handler(irq, dev_id)  │
              │    if IRQ_WAKE_THREAD:           │
              │      wake_up_process(thread)     │
              └──────────────┬─────────────────┘
                             │
                             ▼
              ┌────────────────────────────────┐
              │  irq_exit_rcu()                 │
              │  - Decrement irq count          │
              │  - Check pending softirqs       │
              │  - If pending + not in IRQ:     │
              │    → __do_softirq()             │
              └──────────────┬─────────────────┘
                             │
                             ▼
              ┌────────────────────────────────┐
              │  Return from Interrupt          │
              │  - Check need_resched           │
              │  - If returning to user:        │
              │    → signal delivery            │
              │  - Restore registers            │
              │  - iret (x86) / eret (ARM64)    │
              └────────────────────────────────┘
```

---

## 30.2 Top Half / Bottom Half Architecture

```
              INTERRUPT FIRES
                    │
                    ▼
    ┌───────────────────────────────┐
    │         TOP HALF              │  Hard IRQ context
    │  (irq_handler_t)             │  IRQs disabled
    │  ┌─────────────────────────┐ │  Must be fast
    │  │ Read status register    │ │  Cannot sleep
    │  │ ACK interrupt           │ │
    │  │ Save minimal data       │ │
    │  │ Schedule bottom half    │ │
    │  └─────────────────────────┘ │
    └───────────────┬───────────────┘
                    │
        ┌───────────┼───────────┬───────────────┐
        │           │           │               │
        ▼           ▼           ▼               ▼
   ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌──────────────┐
   │ SoftIRQ │ │ Tasklet │ │ Workqueue│ │ Threaded IRQ │
   │         │ │         │ │          │ │              │
   │ Atomic  │ │ Atomic  │ │ Process  │ │ Process      │
   │ context │ │ context │ │ context  │ │ context      │
   │ No sleep│ │ No sleep│ │ Can sleep│ │ Can sleep    │
   │ Fast    │ │ Easy    │ │ Flexible │ │ Integrated   │
   └─────────┘ └─────────┘ └──────────┘ └──────────────┘
        │           │           │               │
        └───────────┴───────────┴───────────────┘
                    │
                    ▼
           BOTTOM HALF COMPLETE
           (data processed, user woken)
```

---

## 30.3 Execution Context Hierarchy

```
Priority │ Context      │ Preempts     │ Can Sleep? │ Lock to Use
─────────┼──────────────┼──────────────┼────────────┼─────────────────
Highest  │ NMI          │ Everything   │ No         │ trylock only
         │ Hard IRQ     │ SoftIRQ,     │ No         │ spin_lock()
         │              │ Process      │            │
         │ Soft IRQ     │ Process      │ No         │ spin_lock()
         │              │              │            │ spin_lock_irqsave()
         │              │              │            │   (if shared w/ IRQ)
Lowest   │ Process      │ (preemptible)│ Yes        │ mutex_lock()
         │              │              │            │ spin_lock_bh()
         │              │              │            │ spin_lock_irqsave()

Preemption diagram:
  NMI ─────► can preempt ──► Hard IRQ ──► Soft IRQ ──► Process
                 ▲                            │
                 │         can preempt        │
                 └────────────────────────────┘
```

---

## 30.4 Synchronization Primitives Decision Tree

```
Need to protect shared data?
│
├─ Is it a simple counter?
│   ├─ YES → Is exact real-time value needed?
│   │         ├─ YES → atomic_t / atomic64_t
│   │         └─ NO (stats) → per_cpu counters
│   └─ NO (complex data) ↓
│
├─ Is it read-mostly data?
│   ├─ YES → Can readers sleep?
│   │         ├─ YES → SRCU
│   │         └─ NO  → RCU
│   └─ NO ↓
│
├─ Can the code sleep?
│   ├─ YES → Need reader/writer?
│   │         ├─ YES → rw_semaphore
│   │         └─ NO  → mutex
│   └─ NO (atomic context) ↓
│
├─ Shared with hard IRQ?
│   ├─ YES → spin_lock_irqsave()
│   └─ NO  ↓
│
├─ Shared with softirq?
│   ├─ YES → spin_lock_bh()
│   └─ NO  → spin_lock()
│
└─ Is it a flag/state bit?
    └─ YES → test_and_set_bit() / atomic bitops
```

---

## 30.5 IRQ Subsystem Data Structure Relationships

```
                    ┌──────────────┐
                    │  irq_domain  │  Maps HW IRQ → Linux IRQ
                    │  (per chip)  │
                    └──────┬───────┘
                           │ maps to
                           ▼
  ┌──────────────────────────────────────────────────────┐
  │                    irq_desc                           │
  │  ┌──────────────┐                                    │
  │  │  irq_data    │  IRQ number, HW IRQ, chip pointer  │
  │  │  .irq        │                                    │
  │  │  .hwirq      │                                    │
  │  │  .chip ──────┼─────┐                               │
  │  └──────────────┘     │                               │
  │                        ▼                               │
  │  ┌──────────────────────────┐                         │
  │  │  irq_chip               │  HW operations          │
  │  │  .irq_ack()             │                          │
  │  │  .irq_mask()            │                          │
  │  │  .irq_unmask()          │                          │
  │  │  .irq_set_affinity()    │                          │
  │  └──────────────────────────┘                         │
  │                                                       │
  │  handle_irq ──→ handle_fasteoi_irq()  (flow handler) │
  │                                                       │
  │  action ──→ ┌──────────┐    ┌──────────┐             │
  │             │irqaction  │───→│irqaction  │───→ NULL    │
  │             │.handler   │    │.handler   │  (chain)   │
  │             │.thread_fn │    │.thread_fn │             │
  │             │.dev_id    │    │.dev_id    │             │
  │             │.name="A"  │    │.name="B"  │             │
  │             └──────────┘    └──────────┘             │
  └──────────────────────────────────────────────────────┘
```

---

## 30.6 Lock Internals

### qspinlock Structure

```
32-bit lock word:
  ┌────────┬────────┬────────────────┐
  │ locked │pending │     tail       │
  │  (8)   │  (8)   │     (16)      │
  └────────┴────────┴────────────────┘

  locked  = 1 if lock is held
  pending = 1 if one waiter waiting (fast path)
  tail    = CPU+index of last waiter in MCS queue

States:
  0x00000000 = Unlocked
  0x00000001 = Locked, no waiters
  0x00000101 = Locked, one pending waiter
  0x00XX0001 = Locked, queue of waiters (MCS)
```

### Mutex Three-Path

```
  mutex_lock(&lock)
        │
        ▼
  ┌─────────────────┐
  │ FAST PATH        │ CAS: owner = 0 → current
  │ (single atomic)  │ Success? → DONE (most common case)
  └────────┬─────────┘
           │ Fail
           ▼
  ┌─────────────────┐
  │ OPTIMISTIC SPIN  │ Owner running on CPU?
  │ (MCS osq_lock)   │ YES → spin on MCS queue
  │                   │ Owner releases → try CAS again
  └────────┬──────────┘
           │ Owner slept or we got preempted
           ▼
  ┌─────────────────┐
  │ SLOW PATH        │ Add to wait_list
  │ (sleep)          │ schedule() → sleep
  │                   │ On wake → try CAS
  └──────────────────┘
```

---

## 30.7 RCU Grace Period

```
  Writer: rcu_assign_pointer(ptr, new)
  Writer: synchronize_rcu() --- BLOCKS ---

  CPU 0        CPU 1        CPU 2        Time
  ─────        ─────        ─────        ───→
  rcu_read
  _lock()
  [reading     [idle]       [reading
   old ptr]                  old ptr]
  rcu_read
  _unlock()
  ──QS──       ──QS──                    ← Quiescent states
                             rcu_read
                             _unlock()
                             ──QS──      ← Last CPU reports QS
                                         
  ← ── ── Grace Period Complete ── ── →

  Writer: kfree(old)  ← NOW SAFE
```

---

## 30.8 Interrupt Bottom Half Timeline

```
Time ──────────────────────────────────────────────────────►

  │ IRQ │             │ Soft │            │ Work │
  │fires│             │ IRQ  │            │queue │
  │     │             │      │            │      │
  ▼     ▼             ▼      ▼            ▼      ▼
  ┌─────┐             ┌──────┐            ┌──────┐
  │Top  │             │Bottom│            │Defer │
  │Half │             │Half  │            │Work  │
  │     │             │(soft)│            │      │
  │ ACK │             │      │            │ Long │
  │ Save│             │ Net  │            │ Proc │
  │ Sched│            │ Parse│            │      │
  └──┬──┘             └──┬───┘            └──┬───┘
     │                   │                   │
     │ raise_softirq()   │                   │
     └──────────────────►│                   │
                         │ schedule_work()   │
                         └──────────────────►│
                                             │

  │← hardirq ─→│←─── softirq ──→│←── process ctx ──→│
  │  context    │    context      │    (can sleep)    │
```

---

## 30.9 PREEMPT_RT Interrupt Model

```
Standard Linux:                    PREEMPT_RT:
                                   
  IRQ → hardirq handler            IRQ → minimal hardirq
  (IRQs disabled)                   └→ wake IRQ thread
  └→ raise_softirq()               
     └→ __do_softirq()             IRQ thread (SCHED_FIFO):
        (softirq context)           └→ handler + thread_fn
                                    └→ softirq processing
                                    (fully preemptible!)

  spin_lock:                       spin_lock:
    real spinning                    → rt_mutex (sleeping)
    IRQs may be disabled             IRQs NOT disabled
                                   
  spin_lock_irqsave:               spin_lock_irqsave:
    disable IRQs + spin             → rt_mutex (sleeping)
                                     NO real IRQ disable
                                   
  raw_spin_lock_irqsave:           raw_spin_lock_irqsave:
    (same as standard)               real spin + IRQ disable
```

---

## 30.10 Complete Locking Compatibility Matrix

```
Shared between          │ Process ctx lock    │ IRQ/Softirq lock
────────────────────────┼─────────────────────┼───────────────────
Process ↔ Process       │ mutex_lock()        │ N/A
(can sleep)             │                     │
                        │                     │
Process ↔ Process       │ spin_lock()         │ N/A
(cannot sleep)          │                     │
                        │                     │
Process ↔ SoftIRQ       │ spin_lock_bh()      │ spin_lock()
                        │                     │
Process ↔ HardIRQ       │ spin_lock_irqsave() │ spin_lock()
                        │                     │
SoftIRQ ↔ SoftIRQ       │ N/A                 │ spin_lock()
(same type, serialized) │                     │
                        │                     │
SoftIRQ ↔ HardIRQ       │ N/A                 │ spin_lock()
                        │                     │ (softirq side:
                        │                     │  spin_lock_irqsave)
HardIRQ ↔ HardIRQ       │ N/A                 │ spin_lock()
(nested/different IRQs)  │                     │
```

---

## Interview Questions

1. **Draw the complete interrupt handling flow from hardware to driver handler.**
2. **Draw the decision tree for choosing a synchronization primitive.**
3. **Illustrate the difference between standard Linux and PREEMPT_RT interrupt model.**
4. **Diagram the relationship between irq_desc, irq_chip, and irqaction.**
5. **Show the RCU grace period timeline with multiple readers.**
6. **Draw the top-half / bottom-half architecture.**
7. **Illustrate the qspinlock structure and MCS queue.**
8. **Draw the mutex three-path acquisition flow.**
9. **Show the execution context hierarchy with preemption relationships.**
10. **Create a locking compatibility matrix from memory.**

---

## Summary

This chapter collects the most important diagrams for understanding Linux interrupts and concurrency:
- Complete IRQ flow: hardware → controller → CPU → kernel → handler → softirq → return
- Top/bottom half split: hardirq → softirq/tasklet/workqueue/threaded IRQ
- Context hierarchy: NMI > hardirq > softirq > process
- Lock decision tree: based on context, sleep capability, and data access pattern
- Data structures: irq_desc → irq_chip + irqaction chain
- Lock internals: qspinlock 32-bit word, mutex three-path
- RCU: grace period = all CPUs pass quiescent state
- PREEMPT_RT: everything becomes threaded and preemptible

---

*Next: [Chapter 31 — Definitions and Glossary](Chapter_31_Glossary.md)*
