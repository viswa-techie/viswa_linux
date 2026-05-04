# Chapter 1: Foundations of Interrupts and Concurrency

## Learning Goals
- Understand what interrupts are and why every computer system needs them
- Distinguish polling from interrupt-driven I/O and quantify the difference
- Define concurrency vs parallelism with concrete kernel examples
- Learn the core vocabulary used throughout interrupt and concurrency subsystems
- Appreciate why embedded systems depend critically on efficient interrupt handling

---

## 1.1 What Is an Interrupt?

### Beginner Explanation

An interrupt is a **signal to the CPU** that tells it "stop what you're doing and handle this event." Think of it like a phone ringing while you're reading a book — you pause reading, answer the phone, then resume the book.

```
Without interrupts:         With interrupts:
┌─────────────────────┐    ┌─────────────────────┐
│ CPU checks device   │    │ CPU does useful work │
│ CPU checks device   │    │ CPU does useful work │
│ CPU checks device   │    │ ──── INTERRUPT ────  │
│ CPU checks device   │    │ Handle event         │
│ Data ready!         │    │ Resume useful work   │
│ Process data        │    │ CPU does useful work │
└─────────────────────┘    └─────────────────────┘
  Wastes CPU cycles           Efficient!
```

### Kernel-Level Definition

An interrupt is an asynchronous or synchronous event that causes the processor to:

1. **Save** the current execution context (registers, program counter)
2. **Transfer** control to a predefined handler address (interrupt vector)
3. **Execute** the interrupt service routine (ISR)
4. **Restore** the previous context and resume execution

In Linux, the interrupt subsystem manages this through a layered architecture:

```
Hardware Signal
    │
    ▼
Interrupt Controller (PIC / APIC / GIC)
    │
    ▼
CPU Exception Mechanism (IDT on x86 / Vector Table on ARM)
    │
    ▼
Linux Common IRQ Layer  (kernel/irq/)
    │
    ▼
Driver's Interrupt Handler (registered via request_irq())
    │
    ▼
Bottom-Half Processing (softirq / tasklet / workqueue)
```

---

## 1.2 Why Interrupts Are Needed

### The Fundamental Problem

CPUs are extremely fast (GHz). I/O devices are comparatively slow (milliseconds for disk, microseconds for network). Without interrupts, the CPU would waste billions of cycles waiting.

```
CPU speed:    ~3 GHz = 3,000,000,000 cycles/second
Disk latency: ~5 ms  = 15,000,000 wasted cycles per I/O
Network:      ~50 µs = 150,000 wasted cycles per packet
UART byte:    ~87 µs at 115200 baud = 261,000 wasted cycles
```

### What Interrupts Enable

| Capability | Without Interrupts | With Interrupts |
|---|---|---|
| I/O handling | Busy-wait (polling) | Asynchronous notification |
| Multitasking | Cooperative only | Preemptive scheduling |
| Real-time response | Unpredictable | Bounded latency |
| Power management | CPU always active | CPU can sleep (WFI/HLT) |
| Error handling | Periodic checking | Immediate notification |

### In Embedded Systems

Interrupts are especially critical because:
- **Limited CPU power** — cannot afford to waste cycles polling
- **Real-time requirements** — sensor data, motor control, safety
- **Power constraints** — CPU must sleep between events
- **Multiple peripherals** — UART, SPI, I2C, GPIO, DMA, timers all need attention

---

## 1.3 Polling vs Interrupt-Driven I/O

### Polling (Programmed I/O)

```c
/* Polling: CPU repeatedly checks device status register */
while (1) {
    status = readl(dev->base + STATUS_REG);
    if (status & DATA_READY) {
        data = readl(dev->base + DATA_REG);
        process(data);
    }
    /* CPU burns cycles even when no data is ready */
}
```

### Interrupt-Driven I/O

```c
/* Interrupt: CPU notified only when data arrives */
static irqreturn_t my_handler(int irq, void *dev_id)
{
    struct my_device *dev = dev_id;
    u32 data = readl(dev->base + DATA_REG);
    process(data);
    return IRQ_HANDLED;
}

/* Registration — CPU sleeps or does other work until interrupted */
request_irq(dev->irq, my_handler, IRQF_SHARED, "mydev", dev);
```

### Comparison Table

```
┌─────────────────┬───────────────────┬───────────────────────┐
│ Criterion       │ Polling           │ Interrupt-Driven      │
├─────────────────┼───────────────────┼───────────────────────┤
│ CPU utilization │ Poor (busy-wait)  │ Excellent (sleep OK)  │
│ Latency         │ Depends on rate   │ Hardware-speed         │
│ Predictability  │ Bounded if fast   │ Depends on system load│
│ Complexity      │ Simple            │ Context-save overhead │
│ Power usage     │ High              │ Low (idle possible)   │
│ Throughput      │ Can be high*      │ Overhead per interrupt│
│ Best for        │ Very fast devices │ Most real scenarios   │
│ Used where      │ NVMe (hybrid)     │ UART, GPIO, network   │
└─────────────────┴───────────────────┴───────────────────────┘

* High-speed networking/storage uses hybrid: NAPI polls after
  first interrupt to batch packets, then switches back to
  interrupt mode when traffic drops.
```

### Hybrid: NAPI Model (Linux Networking)

```
Normal mode:
  Packet arrives → IRQ fires → driver handler called

NAPI (high throughput):
  First packet → IRQ fires → disable IRQs → poll for more packets
  ↓ (batch processing, no interrupt per packet)
  When queue empty → re-enable IRQs → back to interrupt mode

  ┌─────────┐     ┌──────┐     ┌──────────┐     ┌─────────┐
  │ IRQ     │────→│ Mask │────→│ Poll loop│────→│ Unmask  │
  │ fires   │     │ IRQ  │     │ (softirq)│     │ IRQ     │
  └─────────┘     └──────┘     └──────────┘     └─────────┘
```

---

## 1.4 Concurrency in Operating Systems

### Definition

**Concurrency** means multiple tasks make progress within overlapping time periods. They don't necessarily execute at the exact same instant — they interleave.

### Sources of Concurrency in the Linux Kernel

```
1. True parallelism     — Multiple CPUs executing different code
2. Preemption           — Scheduler switches tasks on same CPU
3. Interrupt context    — IRQ handler interrupts running code
4. Softirq/Tasklet      — Deferred work runs at unpredictable times
5. Workqueues           — Kernel threads executing driver work
6. User-space threads   — Multiple threads in same process
```

### Why the Kernel Must Handle Concurrency

```
  CPU 0                    CPU 1
  ─────                    ─────
  read shared_counter      read shared_counter   ← RACE!
  increment                increment
  write shared_counter     write shared_counter  ← Lost update!

  Expected: counter += 2
  Actual:   counter += 1  (lost update)
```

Every shared data structure in the kernel — from the page cache to device registers — must be protected against concurrent access.

---

## 1.5 Parallelism vs Concurrency

```
Concurrency (interleaving on 1 CPU):
  Task A: ████░░░░████░░░░████
  Task B: ░░░░████░░░░████░░░░
  Time →  ─────────────────────
  Only one runs at a time, but both make progress.

Parallelism (simultaneous on N CPUs):
  CPU 0 Task A: ████████████████████
  CPU 1 Task B: ████████████████████
  Time →        ─────────────────────
  Both truly execute at the same instant.
```

| Aspect | Concurrency | Parallelism |
|--------|-------------|-------------|
| CPUs needed | 1+ | 2+ (always) |
| Tasks run simultaneously? | No (interleaved) | Yes |
| Needs synchronization? | Yes (preemption) | Yes (shared data) |
| Example | Preemptive scheduling | SMP kernel |
| Linux concern | CONFIG_PREEMPT | CONFIG_SMP |

### Key Insight for Kernel Developers

Even on a **uniprocessor** system with `CONFIG_PREEMPT`, you need locking because:
- Interrupt handlers can preempt process context
- Softirqs can preempt process context
- Schedule points can switch tasks mid-operation

On **SMP**, true parallelism adds another dimension — two CPUs can literally execute the same function at the same time.

---

## 1.6 Importance of Interrupts in Embedded Systems

### Embedded Interrupt Landscape

```
Typical Embedded SoC (e.g., SA8155P / i.MX8 / STM32):

  ┌─────────────────────────────────────────────────┐
  │                    SoC                           │
  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐       │
  │  │ CPU0 │  │ CPU1 │  │ CPU2 │  │ CPU3 │       │
  │  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘       │
  │     └────┬─────┴─────────┴────┬────┘           │
  │          │    GIC / NVIC      │                 │
  │          │  (IRQ Controller)  │                 │
  │          └────────┬───────────┘                 │
  │    ┌──────┬───────┼───────┬──────┬──────┐      │
  │    │      │       │       │      │      │      │
  │  UART   SPI    I2C     GPIO   DMA   Timer     │
  │  IRQ    IRQ    IRQ     IRQ    IRQ    IRQ      │
  │    │      │       │       │      │      │      │
  │  Serial  Flash  Sensor  Button Memory  Tick   │
  └─────────────────────────────────────────────────┘
```

### Why Each Interrupt Source Matters

```
Interrupt Source │ Typical Latency Requirement │ Consequence of Miss
────────────────┼────────────────────────────┼─────────────────────
Timer           │ < 10 µs                    │ Scheduling failure
GPIO (button)   │ < 1 ms                     │ Missed user input
UART RX         │ < 87 µs (115200 baud)      │ Data overrun
SPI/I2C done    │ < 100 µs                   │ Transfer timeout
DMA complete    │ < 50 µs                    │ Buffer overrun
Network packet  │ < 100 µs                   │ Packet drop
Watchdog        │ < period                    │ System reset
```

### Real-Time Considerations

In safety-critical embedded systems (automotive ADAS, industrial control):
- Interrupt latency must be **bounded** and **deterministic**
- PREEMPT_RT kernel converts hard IRQs to threads for priority management
- Missed deadlines can cause physical harm (braking, motor control)

---

## 1.7 Basic Terminology

```
┌───────────────────────────┬─────────────────────────────────────┐
│ Term                      │ Definition                          │
├───────────────────────────┼─────────────────────────────────────┤
│ IRQ (Interrupt Request)   │ Hardware signal requesting CPU      │
│                           │ attention                           │
│ ISR (Interrupt Service    │ The function that handles an IRQ    │
│      Routine)             │                                     │
│ IRQ Number                │ Software identifier for an IRQ line │
│ Interrupt Vector          │ Index into handler table (IDT/VBAR) │
│ Interrupt Context         │ Execution state during IRQ handling │
│                           │ (cannot sleep, limited stack)       │
│ Process Context           │ Normal execution (can sleep, full   │
│                           │ kernel services available)          │
│ Top Half                  │ Fast, minimal IRQ handler           │
│ Bottom Half               │ Deferred work (softirq/tasklet/wq) │
│ Preemption                │ Involuntary task switch by scheduler│
│ Race Condition            │ Bug from unsynchronized shared data │
│ Critical Section          │ Code region accessing shared data   │
│ Spinlock                  │ Busy-wait lock (IRQ context safe)   │
│ Mutex                     │ Sleeping lock (process context only)│
│ Atomic Operation          │ Indivisible read-modify-write       │
│ Memory Barrier            │ Instruction ordering guarantee      │
│ RCU                       │ Read-Copy-Update: lock-free reads   │
│ SMP                       │ Symmetric Multi-Processing          │
│ NAPI                      │ New API — hybrid IRQ+poll for net   │
│ DMA                       │ Direct Memory Access (no CPU copy)  │
│ GIC                       │ Generic Interrupt Controller (ARM)  │
│ APIC                      │ Advanced PIC (x86)                  │
│ NMI                       │ Non-Maskable Interrupt              │
│ IPI                       │ Inter-Processor Interrupt           │
└───────────────────────────┴─────────────────────────────────────┘
```

---

## Kernel Source References

```
Key files for the interrupt subsystem entry point:

include/linux/interrupt.h     ← Core interrupt API (request_irq, etc.)
include/linux/irq.h           ← irq_chip, irq_data, flow handlers
include/linux/irqdesc.h       ← struct irq_desc
kernel/irq/manage.c           ← request_irq, free_irq implementation
kernel/irq/chip.c             ← irq_chip callbacks
kernel/irq/handle.c           ← generic_handle_irq()
arch/arm64/kernel/entry.S     ← ARM64 exception entry
arch/x86/kernel/irq.c         ← x86 IRQ entry
```

---

## Interview Questions

1. **What is an interrupt and why does a CPU need interrupts?**
2. **Compare polling vs interrupt-driven I/O. When would you choose polling?**
3. **What is the difference between concurrency and parallelism?**
4. **Can race conditions happen on a uniprocessor system? Explain.**
5. **What is interrupt context? What restrictions does it impose?**
6. **Explain the difference between top half and bottom half.**
7. **What is NAPI and why does Linux networking use it?**
8. **Name three sources of concurrency in the Linux kernel.**
9. **What happens if an interrupt handler takes too long to execute?**
10. **Why are interrupts critical for embedded real-time systems?**
11. **What is an interrupt vector? How does the CPU use it?**
12. **Explain the concept of interrupt latency.**
13. **What is NMI? Give an example use case.**
14. **Why can't you call `kmalloc(GFP_KERNEL)` in interrupt context?**
15. **What is the relationship between interrupts and the Linux scheduler?**

---

## Summary

- Interrupts are the fundamental mechanism for hardware-software communication
- They enable efficient CPU usage, preemptive multitasking, and real-time response
- Polling wastes CPU; interrupts notify asynchronously (hybrid NAPI for throughput)
- Concurrency arises from SMP, preemption, and interrupt contexts — all require synchronization
- Embedded systems depend heavily on fast, deterministic interrupt handling
- The Linux interrupt subsystem is layered: hardware → controller → kernel IRQ core → driver

---

*Next: [Chapter 2 — History and Evolution of Interrupt Systems](Chapter_02_History_and_Evolution.md)*
