# Chapter 2: History and Evolution of Interrupt Systems

## Learning Goals
- Trace the evolution of interrupt mechanisms from earliest computers to modern SoCs
- Understand why each generation solved the limitations of the previous one
- See how Linux interrupt handling evolved from simple to the layered architecture used today
- Appreciate the design decisions behind current kernel interrupt infrastructure

---

## 2.1 Early Computer Interrupt Mechanisms

### Before Interrupts: Pure Polling

```
1940s–1950s (ENIAC, UNIVAC):
  ┌──────────────────────────────┐
  │  CPU executes instructions   │
  │  CPU checks I/O device       │←── Programmed I/O
  │  CPU checks I/O device       │    (wasted cycles)
  │  Data ready? Process it      │
  │  CPU checks next device      │
  └──────────────────────────────┘
  No interrupts. No multitasking. One program at a time.
```

### First Interrupts (1950s)

- **UNIVAC I (1951)**: Had a basic "overflow" interrupt for arithmetic
- **IBM 704 (1954)**: I/O channel could signal completion
- **Key insight**: Let the hardware TELL the CPU when it needs attention

```
1950s breakthrough:
  Device ──signal──→ CPU
                      │
                      ▼
                 Save state
                 Jump to handler
                 Process event
                 Restore state
                 Resume program
```

---

## 2.2 Interrupt Handling in Early Operating Systems

### Batch Processing Era (1960s)

- **IBM System/360 (1964)**: Sophisticated interrupt architecture
  - Five classes: Machine Check, I/O, External, Program, Supervisor Call
  - Hardware PSW (Program Status Word) swap for context save
  - Multiple priority levels

```
System/360 Interrupt Priority:
  1. Machine Check    (highest — hardware error)
  2. I/O Interrupt    (device completion)
  3. External         (timer, operator)
  4. Program          (illegal instruction, overflow)
  5. Supervisor Call  (SVC — system call)
```

### Time-Sharing Era (1970s)

- **PDP-11 / UNIX**: Vectored interrupts with priority levels
  - Interrupt vector table at addresses 0–777 (octal)
  - Hardware automatically pushed PC + PSW to stack
  - `spl()` functions to set processor priority level

```c
/* PDP-11 / Early UNIX interrupt priority levels */
spl0()   /* Allow all interrupts */
spl4()   /* Block disk, TTY interrupts */
spl5()   /* Block network */
spl6()   /* Block clock */
spl7()   /* Block ALL interrupts (highest) */
```

This **spl()** (Set Priority Level) concept influenced Linux's `local_irq_disable()` and `spin_lock_irqsave()`.

---

## 2.3 Evolution of Interrupt Controllers

### Generation Timeline

```
Generation    │ Controller   │ Year  │ Capabilities
──────────────┼──────────────┼───────┼──────────────────────────────
1st           │ Simple latch │ 1960s │ 1 IRQ line, no priority
2nd           │ Intel 8259   │ 1979  │ 8 IRQs, cascade to 15, edge
              │  (PIC)       │       │   or level, priority rotate
3rd           │ Intel APIC   │ 1993  │ SMP support, 256 vectors,
              │  (I/O APIC)  │       │   programmable routing
4th           │ ARM GIC      │ 2004  │ Up to 1020 IRQs, security
              │  (v1/v2/v3)  │       │   groups, affinity, LPI
5th           │ MSI/MSI-X    │ 2000s │ No physical lines, memory
              │              │       │   writes, per-queue vectors
```

### Key Evolution: From Wires to Messages

```
Legacy (wire-based):
  Device ──wire──→ PIC ──wire──→ CPU INTR pin
  Problem: limited lines, shared IRQs, routing inflexible

Modern (message-based):
  Device ──memory write──→ Interrupt Controller ──→ CPU
  MSI-X: Each device queue gets own vector (no sharing!)
  
  NVMe SSD example:
    Queue 0 → Vector 33 → CPU 0
    Queue 1 → Vector 34 → CPU 1
    Queue 2 → Vector 35 → CPU 2
    Queue 3 → Vector 36 → CPU 3
    (Perfect per-core I/O parallelism)
```

---

## 2.4 Interrupt Handling Evolution in Unix Systems

### UNIX V6 (1975) — Ken Thompson, Dennis Ritchie

```c
/* Very simple: device vectors to handler directly */
/* PDP-11 vector table: */
Address 60: clock handler
Address 64: TTY handler
Address 70: disk handler

/* Handler did everything in interrupt context */
/* Minimal bottom-half concept via software interrupts */
```

### BSD (1980s) — Introduced `spl()` Discipline

```c
/* BSD formalized interrupt priority levels */
/* Rule: raise spl before accessing shared data */
int s = splbio();      /* Block disk interrupts */
modify_buffer_cache();
splx(s);               /* Restore previous level */

/* This is the ancestor of Linux's:
   spin_lock_irqsave(&lock, flags);
   ...
   spin_unlock_irqrestore(&lock, flags);
*/
```

### SVR4 / Solaris (1990s)

- Interrupt threads (each IRQ handler in its own thread)
- Priority-based: interrupt threads higher priority than normal
- Influenced Linux's **threaded interrupts** (PREEMPT_RT)

---

## 2.5 Evolution of Interrupt Architecture in Linux

### Linux 0.01 — 1.x (1991–1995): Simple and Direct

```
- Hardcoded interrupt vector table (x86 IDT)
- All handling in IRQ context (top half only)
- Bottom halves: 32-slot "bh" array (bh_base[32])
- Global cli/sti for synchronization
- Single processor only

/* Linux 1.x bottom half */
static void (*bh_base[32])(void);  /* 32 bottom-half slots */
mark_bh(TIMER_BH);                 /* Schedule bottom half */
```

### Linux 2.0–2.2 (1996–1999): SMP Arrives

```
- SMP support added → global IRQ lock (big kernel lock)
- cli()/sti() became per-CPU
- Bottom halves still global (ran on CPU 0 only)
- Major scaling bottleneck on SMP

Problem:
  CPU 0: handles ALL bottom halves
  CPU 1: handles ALL bottom halves  ← NO! Only CPU 0
  Result: SMP scaling terrible
```

### Linux 2.4 (2001): SoftIRQ Revolution

```
Key changes:
  - SoftIRQs replaced old bottom halves
  - Each CPU can run softirqs independently (SMP-scalable!)
  - Tasklets built on top of softirqs
  - ksoftirqd threads for overflow

  Old BH:                    New SoftIRQ:
  ┌─────────────────┐       ┌─────────────────┐
  │ 32 slots         │       │ Per-CPU vectors  │
  │ Global lock      │       │ No global lock   │
  │ Runs on CPU 0    │       │ Runs on ANY CPU  │
  └─────────────────┘       └─────────────────┘
```

### Linux 2.6 (2003–2011): Modern IRQ Framework

```
Major additions:
  - Generic IRQ layer (kernel/irq/) — Ingo Molnár, Thomas Gleixner
  - irq_chip abstraction (platform-independent IRQ ops)
  - irq_desc per-IRQ descriptor
  - Workqueue system (process context deferred work)
  - request_threaded_irq() — threaded interrupt handlers
  - irq_domain — hierarchical IRQ mapping (DT-based)

  Before genirq:              After genirq:
  Each arch had own           Common framework:
  interrupt code              ┌─────────────────────┐
  (duplicated!)               │  Generic IRQ Layer   │
                              │  ┌────────────────┐  │
                              │  │ irq_chip ops   │  │
                              │  │ (per-controller)│  │
                              │  └────────────────┘  │
                              └─────────────────────┘
```

### Linux 3.x–4.x (2011–2019): IRQ Domains & Stacked Controllers

```
- IRQ domains: map hardware IRQs to Linux IRQ numbers
  (essential for device tree and hierarchical controllers)
  
  Hardware IRQ → Parent Controller → Child Controller → Linux IRQ
  
  Example:
    GPIO pin 5 → GPIO controller (child domain)
      → GIC SPI 42 (parent domain)
        → Linux IRQ 137

- Managed interrupts (devm_request_irq)
- IRQ affinity improvements
- PREEMPT_RT patches maturing (force-threaded IRQs)
```

### Linux 5.x–6.x (2019–Present): Modern State

```
Current architecture:
  ┌─────────────────────────────────────────────────┐
  │              Linux IRQ Subsystem (6.x)          │
  │                                                  │
  │  ┌──────────────┐  ┌──────────────────────────┐ │
  │  │ irq_domain   │  │ Interrupt Threading      │ │
  │  │ (hierarchical│  │ (PREEMPT_RT default)     │ │
  │  │  mapping)    │  │                          │ │
  │  └──────────────┘  └──────────────────────────┘ │
  │  ┌──────────────┐  ┌──────────────────────────┐ │
  │  │ irq_chip     │  │ MSI/MSI-X support        │ │
  │  │ (per-ctrlr   │  │ (PCIe message-signaled)  │ │
  │  │  operations) │  │                          │ │
  │  └──────────────┘  └──────────────────────────┘ │
  │  ┌──────────────┐  ┌──────────────────────────┐ │
  │  │ SoftIRQ +    │  │ Workqueue (WQ_UNBOUND,   │ │
  │  │ Tasklet      │  │  system, per-CPU)        │ │
  │  └──────────────┘  └──────────────────────────┘ │
  │  ┌──────────────────────────────────────────┐   │
  │  │ Tracing: ftrace irq events, /proc/irqs   │   │
  │  └──────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────┘

Key 6.x additions:
  - PREEMPT_RT merged into mainline (6.12+)
  - Improved IRQ time accounting
  - Better NUMA-aware IRQ affinity
  - IRQ matrix allocator for x86 vector management
```

---

## Historical Timeline Summary

```
Year  │ Milestone
──────┼──────────────────────────────────────────────
1951  │ UNIVAC overflow interrupt (first interrupt)
1964  │ IBM System/360 — 5-level interrupt system
1975  │ UNIX V6 — vectored interrupts, spl() levels
1979  │ Intel 8259 PIC — standard PC interrupt controller
1991  │ Linux 0.01 — x86 IDT, simple handlers
1993  │ Intel I/O APIC — SMP interrupt routing
1996  │ Linux 2.0 — SMP support, global IRQ lock
2001  │ Linux 2.4 — SoftIRQs replace bottom halves
2003  │ Linux 2.6 — Generic IRQ layer, workqueues
2004  │ ARM GICv1 — Generic Interrupt Controller
2009  │ request_threaded_irq() — threaded handlers
2010  │ irq_domain — hierarchical IRQ mapping
2019  │ Linux 5.x — MSI-X, managed IRQs mature
2024  │ Linux 6.12 — PREEMPT_RT merged into mainline
```

---

## Comparison with Other OS Evolution

```
OS         │ Interrupt Evolution Path
───────────┼───────────────────────────────────────────
Linux      │ BH → SoftIRQ → Tasklet → Workqueue → Threaded IRQ
Windows    │ ISR → DPC (Deferred Procedure Call) → IPI-based
           │ IRQL-based priority masking (PASSIVE→DISPATCH→DEVICE→HIGH)
macOS/XNU  │ Mach exception ports → IOKit interrupt event sources
           │ Work loops for deferred processing
QNX        │ ISR minimal → Interrupt thread (pulse) → message passing
           │ Deterministic from the start (microkernel)
```

---

## Kernel Source References

```
The generic IRQ framework was developed primarily by:
  - Ingo Molnár
  - Thomas Gleixner

Key commit history:
  git log --oneline kernel/irq/
  
Key files:
  kernel/irq/manage.c     ← IRQ registration (request_irq)
  kernel/irq/chip.c       ← irq_chip operations framework
  kernel/irq/irqdomain.c  ← Hierarchical IRQ domain mapping
  kernel/irq/handle.c     ← generic_handle_irq()
  kernel/softirq.c        ← SoftIRQ/Tasklet implementation
```

---

## Interview Questions

1. **What was the original Linux "bottom half" mechanism and why was it replaced?**
2. **How did SMP support change Linux interrupt handling?**
3. **What is the generic IRQ layer and what problem does it solve?**
4. **Explain the evolution from PIC to APIC to GIC.**
5. **What is an IRQ domain and why was it introduced?**
6. **How did BSD's `spl()` mechanism influence Linux locking?**
7. **What is MSI-X and how does it differ from traditional wire-based interrupts?**
8. **Why did Linux 2.4 introduce SoftIRQs?**
9. **What is request_threaded_irq() and when was it added?**
10. **How does PREEMPT_RT change interrupt handling?**
11. **Compare the evolution path of deferred work in Linux vs Windows (DPC).**
12. **What is the significance of PREEMPT_RT being merged into mainline?**

---

## Summary

- Interrupt mechanisms evolved from simple hardware signals to sophisticated multi-level, SMP-aware, message-signaled systems
- Linux progressed from a simple x86-only handler model to a generic, hierarchical, thread-aware framework
- Each evolution step solved a scalability, portability, or determinism limitation
- Modern Linux combines hardware abstraction (irq_chip/irq_domain), efficient deferred work (softirq/workqueue), and RT capability (threaded IRQs)

---

*Next: [Chapter 3 — Hardware Architecture of Interrupt Systems](Chapter_03_Hardware_Architecture.md)*
