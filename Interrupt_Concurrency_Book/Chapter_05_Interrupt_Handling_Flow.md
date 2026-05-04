# Chapter 5: Interrupt Handling Flow

## Learning Goals
- Trace an interrupt from hardware signal generation to handler completion
- Understand every step: assertion, propagation, detection, vector lookup, context save, handler, return
- See the complete hardware → kernel flow with both x86 and ARM64 specifics
- Know what happens at each stage and what can go wrong

---

## 5.1 Interrupt Generation by Hardware

```
Step 1: Device has data or event to report

  ┌────────────────────┐
  │   Network Card     │
  │                    │
  │  Packet received   │
  │  into RX buffer    │
  │         │          │
  │  ┌──────▼────────┐ │
  │  │ Set interrupt  │ │
  │  │ status bit     │ │
  │  └──────┬────────┘ │
  │         │          │
  │  Assert IRQ line   │
  │  (or send MSI msg) │
  └─────────┬──────────┘
            │
            ▼
  To Interrupt Controller
```

### Wire-Based (Level/Edge)

```c
/* Level-triggered: Device holds line asserted until ACK'd */
/* Driver must clear the interrupt at the device: */
writel(INT_CLEAR, dev->regs + INT_STATUS);  /* Release IRQ line */

/* Edge-triggered: Device pulses the line */
/* Controller latches the edge. If another edge occurs
   before ACK, it can be lost! */
```

### Message-Signaled (MSI/MSI-X)

```
Device writes a specific value to a specific memory address:
  Address = LAPIC address (x86) or ITS target (ARM GICv3)
  Data    = encodes vector number and destination CPU

No physical wire needed. Each device queue can have its own
vector → no sharing, no routing limitations.
```

---

## 5.2 Interrupt Signal Propagation

```
Step 2: Signal travels from device to interrupt controller

Wire-based path:
  Device ──wire──→ Interrupt Controller input pin
  
  x86:  Device → I/O APIC input pin → system bus → LAPIC
  ARM:  Device → GIC Distributor SPI input → CPU Interface

MSI path:
  Device → PCIe bus → memory-mapped write → controller

  ┌────────┐     ┌────────────┐     ┌──────────┐
  │ Device │────→│ Controller │────→│   CPU    │
  │        │     │ (APIC/GIC) │     │ (vector) │
  └────────┘     └────────────┘     └──────────┘
                      │
              ┌───────┴────────┐
              │ Priority check │
              │ Target CPU sel │
              │ Vector assign  │
              └────────────────┘
```

### Controller Processing

```
Interrupt controller actions:
  1. Receive interrupt request
  2. Check if IRQ is enabled (not masked)
  3. Compare priority against running priority
  4. If higher priority → signal CPU
  5. If lower priority → keep pending until current finishes
  6. For SMP: determine target CPU (affinity, load, etc.)
  7. Present vector number to CPU when acknowledged

ARM GIC flow:
  GICD receives SPI → checks GICD_ISENABLERn (enabled?) →
  checks GICD_IPRIORITYRn (priority) → routes per
  GICD_ITARGETSRn → signals CPU Interface → CPU IF checks
  priority mask → asserts IRQ to CPU → CPU reads IAR →
  gets interrupt ID

x86 APIC flow:
  I/O APIC receives IRQ → reads redirection table entry →
  sends bus message to target LAPIC → LAPIC checks TPR →
  asserts INTR pin → CPU acknowledges → LAPIC provides vector
```

---

## 5.3 CPU Interrupt Detection

```
Step 3: CPU detects the interrupt signal

x86:
  CPU checks INTR pin between instruction boundaries
  IF (Interrupt Flag in EFLAGS) must be set
  If IF=0 (interrupts disabled): IRQ stays pending at LAPIC
  
ARM64:
  CPU checks pending IRQ state between instructions
  PSTATE.I bit must be clear (IRQ unmasked)
  If PSTATE.I=1: IRQ stays pending at GIC CPU Interface

Key point: Interrupts are checked BETWEEN instructions,
not in the middle of one. The current instruction always
completes first (except for some long instructions with
interrupt windows on x86).
```

---

## 5.4 Interrupt Vector Lookup

```
Step 4: CPU determines which handler to call

x86 IDT (Interrupt Descriptor Table):
  Vector from LAPIC → index into IDT
  IDT entry contains: segment selector + handler offset
  
  IDTR register → base address of IDT
  IDT[vector] → gate descriptor → handler address
  
  Example:
    Vector 42 → IDT[42] → common_interrupt
    → vector_irq[42] → do_IRQ() → irq_desc[42] → handler

ARM64 Vector Table (VBAR_EL1):
  Exception type determines offset:
    Sync:   +0x000 / +0x200 / +0x400 / +0x600
    IRQ:    +0x080 / +0x280 / +0x480 / +0x680
    FIQ:    +0x100 / +0x300 / +0x500 / +0x700
    SError: +0x180 / +0x380 / +0x580 / +0x780
  
  IRQ from same EL, using SP_EL1:
    VBAR_EL1 + 0x280 → el1h_irq → gic_handle_irq()
    → reads IAR → gets hardware IRQ ID → irq_domain lookup
    → gets Linux IRQ number → generic_handle_irq()
```

---

## 5.5 Interrupt Handler Execution

### Complete x86 Path

```
Hardware IRQ on x86_64:

  1. APIC asserts INTR
  2. CPU finishes current instruction
  3. CPU pushes: SS, RSP, RFLAGS, CS, RIP (on interrupt stack)
  4. CPU clears IF (disables further interrupts)
  5. CPU loads new CS:RIP from IDT[vector]
  ────── now in kernel ──────
  6. entry code (arch/x86/entry/entry_64.S):
     - Save remaining registers (push all GPRs)
     - Switch to interrupt stack if needed
  7. call do_IRQ(regs)
  8. do_IRQ():
     - irq_enter() (track IRQ context, update accounting)
     - generic_handle_irq(irq)
       → irq_desc[irq]->handle_irq()  (flow handler)
         → handle_level_irq() or handle_edge_irq()
           → irq_desc->action->handler()  (YOUR driver handler!)
     - irq_exit()
       → if softirqs pending: __do_softirq()
  9. Restore registers, IRETQ → back to interrupted code
```

### Complete ARM64 Path

```
Hardware IRQ on ARM64:

  1. GIC signals CPU via IRQ line
  2. CPU finishes current instruction
  3. CPU saves PSTATE → SPSR_EL1, PC → ELR_EL1
  4. CPU masks interrupts (PSTATE.I = 1)
  5. CPU jumps to VBAR_EL1 + offset (IRQ vector)
  ────── now in kernel ──────
  6. entry code (arch/arm64/kernel/entry.S):
     kernel_entry:
       - Save x0-x30, SP, ELR, SPSR to pt_regs on stack
       - Ensure stack alignment
  7. Call handler:
     el1h_64_irq_handler / el0_64_irq_handler
       → gic_handle_irq()
         → Read GICC_IAR (acknowledge, get IRQ ID)
         → irq_domain_translate → Linux IRQ number
         → generic_handle_domain_irq()
           → irq_desc[linux_irq]->handle_irq()
             → handle_fasteoi_irq()
               → action->handler()  (YOUR driver handler!)
             → irq_chip->irq_eoi()  (write GICC_EOIR)
  8. irq_exit():
     → Process pending softirqs if needed
  9. kernel_exit:
     - Restore registers from pt_regs
     - ERET → return to interrupted code
```

---

## 5.6 Interrupt Completion and Return

```
After handler returns:

  1. irq_exit() is called:
     ├── Decrement preempt_count (hardirq section)
     ├── Check if softirqs are pending
     │   └── If yes: invoke_softirq() → __do_softirq()
     ├── Check if rescheduling needed (TIF_NEED_RESCHED)
     └── (handled at return-from-interrupt check point)

  2. Return-from-interrupt:
     x86: ret_from_intr → check preempt → IRETQ
     ARM: kernel_exit → check preempt → ERET

  3. Preemption check (if CONFIG_PREEMPT):
     If returning to kernel code and TIF_NEED_RESCHED set:
       → preempt_schedule_irq() → schedule()

  4. If returning to user space:
     Check for: signals, rescheduling, tracing
     → do_signal() if pending
     → schedule() if TIF_NEED_RESCHED

  5. Restore all registers → resume interrupted code
```

### Complete Flow Summary

```
┌──────────────────────────────────────────────────────────────┐
│                    COMPLETE IRQ FLOW                         │
│                                                              │
│  Device asserts IRQ                                          │
│       │                                                      │
│       ▼                                                      │
│  IRQ Controller (APIC/GIC)                                   │
│  ├── Priority check                                          │
│  ├── Route to target CPU                                     │
│  └── Assert CPU interrupt signal                             │
│       │                                                      │
│       ▼                                                      │
│  CPU detects IRQ (between instructions)                      │
│  ├── Save minimal context (PC, flags)                        │
│  ├── Disable interrupts                                      │
│  └── Jump to vector table entry                              │
│       │                                                      │
│       ▼                                                      │
│  Entry asm code (entry.S)                                    │
│  ├── Save ALL registers to stack (pt_regs)                   │
│  └── Call C handler                                          │
│       │                                                      │
│       ▼                                                      │
│  irq_enter() — enter hardirq context                         │
│       │                                                      │
│       ▼                                                      │
│  generic_handle_irq(irq)                                     │
│  ├── irq_desc->handle_irq()  [flow handler]                 │
│  │   └── handle_level_irq / handle_edge_irq / handle_fasteoi│
│  │       ├── irq_chip->irq_ack()                            │
│  │       ├── action->handler()  ← YOUR DRIVER CODE          │
│  │       │   └── Returns IRQ_HANDLED / IRQ_NONE / IRQ_WAKE  │
│  │       └── irq_chip->irq_eoi()                            │
│  └── Handle shared IRQs: iterate action list                 │
│       │                                                      │
│       ▼                                                      │
│  irq_exit() — exit hardirq context                           │
│  ├── Softirq processing if pending                           │
│  └── Decrement preempt_count                                 │
│       │                                                      │
│       ▼                                                      │
│  Return-from-interrupt                                       │
│  ├── To kernel: check preempt_schedule_irq()                │
│  └── To user: check signals, reschedule                      │
│       │                                                      │
│       ▼                                                      │
│  Restore registers → IRETQ/ERET                              │
│  Resume interrupted code                                     │
└──────────────────────────────────────────────────────────────┘
```

---

## Timing Breakdown

```
Typical interrupt latency on modern hardware:

Phase                       │ Time (approx)
────────────────────────────┼──────────────
Device asserts IRQ          │ 0
Controller propagation      │ ~0.1-1 µs
CPU detection + context save│ ~0.1-0.5 µs
Kernel entry code           │ ~0.2-0.5 µs
IRQ framework (irq_desc)    │ ~0.1-0.3 µs
Driver handler execution    │ 0.1-10 µs (varies!)
EOI + exit                  │ ~0.1-0.3 µs
Return to interrupted code  │ ~0.2-0.5 µs
────────────────────────────┼──────────────
Total (best case):          │ ~1-3 µs
Total (typical device):     │ ~3-15 µs

PREEMPT_RT (threaded IRQ):
  Hard IRQ + wake thread:   │ ~1-5 µs
  Thread scheduled + runs:  │ ~5-50 µs
  (Higher latency but deterministic)
```

---

## Kernel Source References

```
x86 entry:
  arch/x86/entry/entry_64.S      ← IRQ entry/exit asm
  arch/x86/kernel/irq.c          ← do_IRQ, vector dispatch

ARM64 entry:
  arch/arm64/kernel/entry.S       ← Exception vectors, context save
  arch/arm64/kernel/irq.c         ← ARM64 IRQ handling

Generic IRQ:
  kernel/irq/handle.c             ← generic_handle_irq()
  kernel/irq/chip.c               ← handle_level_irq, handle_edge_irq
  kernel/irq/manage.c             ← IRQ registration, threading

Flow handlers:
  handle_level_irq()              ← Level-triggered flow
  handle_edge_irq()               ← Edge-triggered flow
  handle_fasteoi_irq()            ← Modern APIC/GIC fast EOI
  handle_percpu_devid_irq()       ← Per-CPU IRQs (timer, IPI)
```

---

## Interview Questions

1. **Walk through the complete path of a hardware interrupt from device to handler return.**
2. **What does the CPU do automatically when an interrupt arrives (before any kernel code runs)?**
3. **What is the difference between handle_level_irq and handle_edge_irq?**
4. **What happens during irq_enter() and irq_exit()?**
5. **Why are softirqs processed during irq_exit()?**
6. **What is handle_fasteoi_irq and when is it used?**
7. **How does ARM64 determine which exception vector to jump to?**
8. **What is the IDT on x86? How many entries does it have?**
9. **What registers does the CPU automatically save on x86 when an interrupt occurs?**
10. **What is the interrupt stack and why is it separate from the process stack?**
11. **When does preemption happen after an interrupt return?**
12. **What happens if a driver handler returns IRQ_NONE for a shared IRQ?**
13. **How is EOI (End of Interrupt) handled differently for level vs edge triggers?**
14. **What is the latency from IRQ assertion to handler execution on typical hardware?**
15. **Explain the IRQF_ONESHOT flag and its role in the interrupt flow.**

---

## Summary

- An interrupt travels: device → controller → CPU → vector lookup → entry code → kernel handler → exit → resume
- The CPU automatically saves minimal context; the entry code saves the rest
- The generic IRQ layer dispatches through irq_desc → flow handler → action list → driver handler
- irq_exit() is the bridge to softirq processing (bottom halves)
- Return-from-interrupt is the preemption check point for the scheduler
- Understanding this flow end-to-end is essential for writing correct interrupt handlers and debugging latency issues

---

*Next: [Chapter 6 — Interrupt Handling in Linux Kernel](Chapter_06_Linux_Interrupt_Handling.md)*
