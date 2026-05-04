# Chapter 3: Hardware Architecture of Interrupt Systems

## Learning Goals
- Understand how interrupt signals travel from device to CPU at hardware level
- Know the major interrupt controllers: PIC, APIC, GIC, MSI
- Understand interrupt vectors, priority levels, and routing
- Know how interrupt masking works in hardware
- Connect hardware concepts to Linux kernel abstractions

---

## 3.1 CPU Interrupt Architecture Overview

### x86 Architecture

```
x86 CPU Interrupt Pins:
  ┌──────────────────────────────────┐
  │          x86 CPU Core            │
  │                                  │
  │  INTR pin ←── Maskable IRQs     │
  │  NMI pin  ←── Non-Maskable IRQ  │
  │  INIT pin ←── Reset             │
  │  SMI pin  ←── System Management │
  │                                  │
  │  Internal:                       │
  │    #DE — Divide Error (vector 0) │
  │    #PF — Page Fault (vector 14)  │
  │    #GP — General Protection (13) │
  │    ...256 vector entries in IDT  │
  └──────────────────────────────────┘

IDT (Interrupt Descriptor Table):
  256 entries, each points to handler
  Vectors 0-31:   CPU exceptions (reserved)
  Vectors 32-255: External interrupts (devices, APIC)
```

### ARM64 Architecture

```
ARM64 Exception Model:
  ┌──────────────────────────────────────┐
  │           ARM64 CPU Core             │
  │                                      │
  │  Exception Levels:                   │
  │    EL0: User space                   │
  │    EL1: Kernel (handles most IRQs)   │
  │    EL2: Hypervisor                   │
  │    EL3: Secure Monitor               │
  │                                      │
  │  Exception Vectors (VBAR_EL1):       │
  │    Synchronous (SVC, page fault)     │
  │    IRQ (external interrupt)          │
  │    FIQ (fast interrupt, secure)      │
  │    SError (asynchronous abort)       │
  │                                      │
  │  4 vector tables × 4 entries = 16    │
  │  (current EL SP_EL0, current SP_ELx, │
  │   lower EL AArch64, lower EL AArch32)│
  └──────────────────────────────────────┘
```

---

## 3.2 Interrupt Signals and Lines

### Physical Interrupt Signaling

```
Trigger Types:
                                    
  Level-triggered:                Edge-triggered:
  ───────┐                        ─────┐
         │ (held low/high)             │ (pulse)
         │ while asserted              └──── 
         │                        
  ───────┘                        
  Device holds line until         Device pulses line once.
  CPU acknowledges.               Must not miss the edge!

  Level: Reliable, re-asserts     Edge: Can miss if CPU busy
         until serviced.                 during transition.
         Multiple devices can            Faster, less latency.
         share (all pull line).          MSI works like edge.
```

### Linux IRQ Trigger Configuration

```c
/* Trigger flags for request_irq() / device tree */
IRQF_TRIGGER_RISING   /* Edge: low → high transition */
IRQF_TRIGGER_FALLING  /* Edge: high → low transition */
IRQF_TRIGGER_HIGH     /* Level: active high */
IRQF_TRIGGER_LOW      /* Level: active low */

/* Device tree example: */
/* interrupt-parent = <&gpio>;
   interrupts = <5 IRQ_TYPE_EDGE_RISING>; */
```

---

## 3.3 Interrupt Vectors

```
Interrupt Vector: A number that indexes into a handler table.
The CPU uses the vector to find the correct handler function.

x86 IDT (Interrupt Descriptor Table):
  Vector 0:   #DE — Divide Error
  Vector 1:   #DB — Debug
  Vector 2:   NMI
  Vector 3:   #BP — Breakpoint
  ...
  Vector 13:  #GP — General Protection
  Vector 14:  #PF — Page Fault
  ...
  Vector 32:  First available for devices
  ...
  Vector 255: Last vector

ARM GIC:
  ID 0-15:    SGI (Software Generated Interrupts) — IPI
  ID 16-31:   PPI (Private Peripheral Interrupts) — per-CPU
  ID 32-1019: SPI (Shared Peripheral Interrupts) — devices
  ID 1020+:   Special (LPI in GICv3)

  PPI examples: Timer IRQ (ID 27), PMU (ID 23)
  SPI example:  UART (ID 33+), SPI controller, etc.
```

---

## 3.4 Interrupt Priority Levels

### x86 APIC Priority

```
x86 TPR (Task Priority Register) and vector-based priority:
  Priority = Vector / 16  (vector 32-255 → priority 2-15)
  Higher vector number → higher priority
  CPU only accepts IRQs with priority > current TPR

  Priority Level │ Vector Range │ Typical Use
  ───────────────┼──────────────┼────────────────
  15             │ 240-255      │ Highest (NMI-like)
  14             │ 224-239      │ IPI, performance
  ...            │ ...          │ ...
  3              │ 48-63        │ Device IRQs
  2              │ 32-47        │ Device IRQs
  1              │ 16-31        │ Reserved
  0              │ 0-15         │ Exceptions
```

### ARM GIC Priority

```
GIC supports 8-bit priority (0-255, lower = higher):
  0x00: Highest priority
  0xFF: Lowest priority
  Often only 5 bits implemented (32 levels)

  GIC Priority Registers (GICD_IPRIORITYRn):
    Each interrupt gets a priority byte
    CPU interface compares running priority vs pending

  Group 0: Secure (FIQ) — typically highest
  Group 1: Non-Secure (IRQ) — OS interrupts
```

---

## 3.5 Interrupt Controllers — Overview

```
Device → Interrupt Controller → CPU

Role of the controller:
  1. Collect interrupt requests from multiple sources
  2. Prioritize and arbitrate between pending IRQs
  3. Route interrupts to appropriate CPU(s)
  4. Provide vector number to CPU
  5. Handle masking/unmasking per-source
  6. Track in-service interrupts (EOI mechanism)
```

---

## 3.6 Programmable Interrupt Controller (PIC) — Intel 8259

```
Legacy x86 interrupt architecture (ISA era):

  ┌──────────────────────────────────────────┐
  │              Cascaded 8259 PICs          │
  │                                          │
  │  Master PIC (8259A)    Slave PIC (8259A) │
  │  IRQ 0: Timer          IRQ 8:  RTC      │
  │  IRQ 1: Keyboard       IRQ 9:  (ACPI)   │
  │  IRQ 2: ──cascade──→   IRQ 10: (free)   │
  │  IRQ 3: COM2           IRQ 11: (free)   │
  │  IRQ 4: COM1           IRQ 12: PS/2 Mse │
  │  IRQ 5: LPT2/Sound     IRQ 13: FPU      │
  │  IRQ 6: Floppy         IRQ 14: IDE Pri  │
  │  IRQ 7: LPT1           IRQ 15: IDE Sec  │
  │                                          │
  │  Total: 15 usable IRQs (IRQ 2 = cascade)│
  └──────────────────────────────────────────┘

Limitations:
  - Only 15 IRQ lines (severe shortage for modern devices)
  - Fixed priority (IRQ 0 highest)
  - No SMP support (single CPU only)
  - Edge-triggered issues (missed interrupts)
  - Shared IRQs problematic
```

---

## 3.7 Advanced Programmable Interrupt Controller (APIC)

```
Modern x86 uses APIC system:

  ┌─────────┐  ┌─────────┐
  │ Device A │  │ Device B │
  └────┬─────┘  └────┬─────┘
       │              │
  ┌────▼──────────────▼────┐
  │      I/O APIC          │  ← One per chipset (24+ inputs)
  │  (Interrupt routing)   │
  └────────────┬───────────┘
               │ (bus message)
    ┌──────────┼──────────┐
    │          │          │
  ┌─▼──┐   ┌──▼──┐   ┌──▼──┐
  │LAPIC│   │LAPIC│   │LAPIC│  ← One per CPU core
  │CPU 0│   │CPU 1│   │CPU 2│
  └─────┘   └─────┘   └─────┘

I/O APIC:
  - 24+ redirection entries
  - Each entry specifies: destination CPU(s), vector, trigger
  - Programmable routing: fixed, lowest-priority, etc.

Local APIC (LAPIC):
  - One per CPU core
  - Handles local timer, IPI, performance counters
  - IRQ priority via TPR (Task Priority Register)
  - EOI (End-of-Interrupt) acknowledgement

Key registers:
  I/O APIC: IOREDTBL[0..23]  (redirection table)
  LAPIC: TPR, PPR, ISR, IRR, EOI, ICR (IPI)
```

---

## 3.8 Generic Interrupt Controller (GIC) — ARM

### GIC Architecture

```
ARM systems use GIC (v1/v2/v3/v4):

  ┌──────────────────────────────────────────────────────┐
  │                    GIC Architecture                   │
  │                                                       │
  │  Peripherals (SPI)                                    │
  │  ┌─────┐ ┌─────┐ ┌─────┐                            │
  │  │UART │ │SPI  │ │DMA  │    Per-CPU (PPI)            │
  │  └──┬──┘ └──┬──┘ └──┬──┘    ┌──────┐  ┌──────┐      │
  │     │       │       │       │Timer │  │PMU   │      │
  │     └───────┼───────┘       └──┬───┘  └──┬───┘      │
  │             │                   │         │           │
  │  ┌──────────▼───────────────────▼─────────▼────────┐ │
  │  │              GIC Distributor (GICD)              │ │
  │  │  - SPI routing & priority                       │ │
  │  │  - Enable/disable per-IRQ                       │ │
  │  │  - Target CPU selection                         │ │
  │  │  - Group 0 (secure) / Group 1 (non-secure)     │ │
  │  └────────────┬──────────────────┬─────────────────┘ │
  │               │                  │                    │
  │  ┌────────────▼─────┐ ┌─────────▼──────────┐        │
  │  │ CPU Interface 0  │ │ CPU Interface 1    │        │
  │  │ (GICC)           │ │ (GICC)             │        │
  │  │ - Priority mask  │ │ - Acknowledge      │        │
  │  │ - Ack / EOI      │ │ - Running priority │        │
  │  └────────┬─────────┘ └────────┬───────────┘        │
  │           │                    │                      │
  │      ┌────▼────┐         ┌────▼────┐                 │
  │      │  CPU 0  │         │  CPU 1  │                 │
  │      └─────────┘         └─────────┘                 │
  └──────────────────────────────────────────────────────┘
```

### GIC Versions

```
Version │ Features
────────┼──────────────────────────────────────────────
GICv1   │ Basic distributor + CPU interface
GICv2   │ Virtualization support, security groups
GICv3   │ System register access (no MMIO for CPU IF),
        │ affinity routing, LPI (Locality-specific
        │ Peripheral Interrupts), ITS (Interrupt
        │ Translation Service for MSI)
GICv4   │ Direct virtual interrupt injection
        │ (passthrough for VM guests)

GICv3 (used in modern ARM servers/automotive):
  - Up to 1020 SPIs + 8192+ LPIs
  - ITS: translates MSI writes → IRQ routing
  - Affinity: 4-level (Aff3.Aff2.Aff1.Aff0 = cluster.core)
  - Security: Group 0 (EL3/FIQ), Group 1S (secure OS), Group 1NS
```

---

## 3.9 Interrupt Routing in Modern Systems

### x86 Routing Path

```
Device → I/O APIC → Destination LAPIC → CPU

Routing modes (I/O APIC redirection table):
  - Fixed: Always goes to specified CPU(s)
  - Lowest Priority: Goes to CPU with lowest TPR
  - Physical: Specific APIC ID
  - Logical: Group of CPUs (bitmask)

MSI/MSI-X (PCI Express):
  Device writes to special memory address → LAPIC directly
  ┌────────┐     memory write     ┌─────────┐
  │ PCIe   │ ───────────────────→ │  LAPIC  │
  │ Device │  (address encodes    │  CPU N  │
  └────────┘   CPU + vector)      └─────────┘
  No I/O APIC needed! Each queue gets its own vector.
```

### ARM Routing Path

```
Device → GIC Distributor → GIC CPU Interface → CPU

SPI routing:
  GICD_ITARGETSRn: bitmask of target CPUs (GICv2)
  GICD_IROUTERn:   affinity routing (GICv3)

  Example: Route UART IRQ to CPU 2
    GICv2: GICD_ITARGETSR[33] = 0x04  (bit 2 = CPU 2)
    GICv3: GICD_IROUTER[33]   = Aff0=2

LPI routing (GICv3): Message-based, similar to MSI-X
  Device → ITS (Interrupt Translation Service) → CPU
  Scalable to thousands of interrupt sources
```

---

## 3.10 Hardware Interrupt Masking Mechanisms

### CPU-Level Masking

```c
/* x86: CLI/STI instructions */
cli();  /* Clear interrupt flag — disable all maskable IRQs */
sti();  /* Set interrupt flag — enable maskable IRQs */
/* NMI cannot be masked this way */

/* ARM64: DAIF register bits */
msr daifset, #2  /* Mask IRQ (bit I) */
msr daifclr, #2  /* Unmask IRQ */
/* D=Debug, A=SError, I=IRQ, F=FIQ */

/* Linux abstractions: */
local_irq_disable();     /* CLI / mask IRQ */
local_irq_enable();      /* STI / unmask IRQ */
local_irq_save(flags);   /* Save current state + disable */
local_irq_restore(flags); /* Restore saved state */
```

### Controller-Level Masking

```
Per-IRQ masking at the controller:

x86 APIC:
  I/O APIC: mask bit in redirection entry
  LAPIC: ISR/IRR/TMR registers track state

ARM GIC:
  GICD_ISENABLERn: Set-enable (write 1 to enable)
  GICD_ICENABLERn: Clear-enable (write 1 to disable)
  Per-IRQ granularity

Linux kernel wraps this:
  irq_chip->irq_mask(irq_data)   → mask specific IRQ
  irq_chip->irq_unmask(irq_data) → unmask specific IRQ
  irq_chip->irq_ack(irq_data)    → acknowledge IRQ
  irq_chip->irq_eoi(irq_data)    → end of interrupt
```

### Masking Hierarchy

```
  Level           │ Granularity    │ Mechanism
  ────────────────┼────────────────┼─────────────────────
  CPU flags       │ ALL IRQs       │ CLI/STI, DAIF
  LAPIC TPR       │ Priority-based │ Mask below threshold
  Controller mask │ Per-IRQ        │ GICD_ICENABLER, APIC mask
  Device register │ Per-source     │ Device interrupt enable bit
  
  Priority of masking (widest → narrowest):
  CPU disable > Controller mask > Device mask

  Linux recommendation:
    Use spin_lock_irqsave() instead of raw cli/sti
    Let the IRQ framework handle controller masking
```

---

## Hardware → Linux Mapping

```
Hardware Concept          │ Linux Abstraction
──────────────────────────┼─────────────────────────
Interrupt Controller      │ struct irq_chip
IRQ Line/Source           │ struct irq_desc (per IRQ)
Controller registers      │ irq_chip callbacks
Vector table              │ IDT (x86) / vector_table (ARM)
CPU masking               │ local_irq_disable/enable()
Per-IRQ masking           │ disable_irq() / enable_irq()
Priority                  │ irq_set_affinity_hint()
Routing                   │ irq_set_affinity()
IRQ acknowledge           │ irq_chip->irq_ack()
End of interrupt          │ irq_chip->irq_eoi()
Hardware IRQ number       │ irq_domain mapping → Linux IRQ
```

---

## Kernel Source References

```
x86 APIC:
  arch/x86/kernel/apic/      ← LAPIC and I/O APIC drivers
  arch/x86/kernel/irq.c      ← x86 IRQ entry handling
  arch/x86/include/asm/apic.h

ARM GIC:
  drivers/irqchip/irq-gic.c      ← GICv2 driver
  drivers/irqchip/irq-gic-v3.c   ← GICv3 driver
  drivers/irqchip/irq-gic-v3-its.c ← ITS for LPI/MSI

Generic IRQ:
  kernel/irq/chip.c          ← irq_chip framework
  include/linux/irq.h        ← struct irq_chip definition
  kernel/irq/irqdomain.c     ← Hardware → Linux IRQ mapping
```

---

## Interview Questions

1. **What is the difference between PIC, APIC, and GIC?**
2. **Explain level-triggered vs edge-triggered interrupts. Which is safer for shared IRQs?**
3. **What happens at the hardware level when a device asserts an interrupt?**
4. **How does the x86 APIC route an interrupt to a specific CPU?**
5. **What are SPI, PPI, and SGI in the ARM GIC?**
6. **What is MSI-X and why is it better than wire-based IRQs?**
7. **How does the CPU's interrupt flag (IF on x86, I in DAIF on ARM) affect interrupt delivery?**
8. **What is the interrupt vector table? Where is it stored on x86 vs ARM?**
9. **Explain EOI (End of Interrupt). What happens if a driver forgets to send EOI?**
10. **What is the GIC ITS and why is it needed for GICv3?**
11. **How does NMI differ from regular maskable interrupts at the hardware level?**
12. **What is the TPR (Task Priority Register) in the LAPIC?**
13. **Compare interrupt routing in x86 I/O APIC vs ARM GIC Distributor.**
14. **How does `local_irq_save()` differ from `disable_irq()`?**
15. **What are IRQ domains in Linux and what hardware problem do they solve?**

---

## Summary

- CPUs accept interrupts through dedicated pins/signals; vectors index into handler tables
- Interrupt controllers (PIC → APIC → GIC) manage routing, priority, and masking
- Modern systems use message-signaled interrupts (MSI-X) — no physical wires needed
- ARM GIC provides SPI/PPI/SGI/LPI with security groups and virtualization support
- Hardware masking exists at CPU level (global), controller level (per-IRQ), and device level
- Linux's irq_chip and irq_domain abstract all hardware differences into a common framework

---

*Next: [Chapter 4 — Types of Interrupts](Chapter_04_Types_of_Interrupts.md)*
