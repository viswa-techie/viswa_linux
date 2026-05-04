# Chapter 3: Hardware Architecture Overview

## Learning Goals
- Understand CPU architecture fundamentals relevant to kernel development
- Grasp processor privilege levels and their role in OS design
- Understand hardware memory architecture (MMU, TLB, caches)
- Know how hardware interrupt systems work
- Understand multiprocessing hardware and timers

---

## 3.1 CPU Architecture Overview

The CPU is the kernel's primary interface to the hardware. A kernel developer must understand the CPU's execution model, register set, and special features.

```
Modern CPU Block Diagram (ARM Cortex-A series / x86):

┌────────────────────────────────────────────────────────────┐
│                         CPU Core                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Fetch → Decode → Execute → Memory → Writeback       │  │
│  │         (Pipeline stages)                             │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐   │
│  │   Integer    │  │  Floating    │  │    SIMD/NEON   │   │
│  │    ALU       │  │  Point Unit  │  │    (Vector)    │   │
│  └──────────────┘  └──────────────┘  └────────────────┘   │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐   │
│  │  Register    │  │    MMU       │  │     TLB        │   │
│  │  File        │  │  (Paging)   │  │  (Translation  │   │
│  │  (GPR/FP/SP) │  │             │  │   Lookaside)   │   │
│  └──────────────┘  └──────────────┘  └────────────────┘   │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐                       │
│  │  L1 I-Cache  │  │  L1 D-Cache  │  ← Per-core          │
│  │  (32-64KB)   │  │  (32-64KB)   │                       │
│  └──────────────┘  └──────────────┘                       │
│                                                            │
│  ┌──────────────────────────────────┐                      │
│  │          L2 Cache (256KB-1MB)    │  ← Per-core or shared│
│  └──────────────────────────────────┘                      │
├────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────┐  │
│  │              L3 Cache (2-32MB)  ← Shared across cores│  │
│  └──────────────────────────────────────────────────────┘  │
├────────────────────────────────────────────────────────────┤
│                    Memory Controller                        │
│                    ↓              ↓                          │
│               DDR4/5 DRAM    DDR4/5 DRAM                   │
└────────────────────────────────────────────────────────────┘
```

Key CPU registers the kernel uses:

| Register (ARM64) | Register (x86_64) | Kernel Use |
|-------------------|--------------------|----|
| `PC` (Program Counter) | `RIP` | Current instruction address |
| `SP` (Stack Pointer) | `RSP` | Kernel/user stack pointer |
| `LR` (Link Register) | (on stack) | Return address |
| `X0-X7` | `RDI, RSI, RDX, RCX, R8, R9` | System call arguments |
| `PSTATE` | `RFLAGS` | Condition flags, interrupt mask |
| `TTBR0/TTBR1` | `CR3` | Page table base address |
| `VBAR_EL1` | `IDTR` | Exception/interrupt vector base |
| `TPIDR_EL0/EL1` | `FS`/`GS` base | Thread-local storage, per-CPU data |

---

## 3.2 Processor Privilege Levels

Modern CPUs enforce privilege levels to prevent user code from executing dangerous instructions or accessing kernel memory.

```
x86_64 Privilege Rings:

        Ring 0 (Kernel Mode)
       ┌─────────────────┐
       │  Full access:    │
       │  - All memory    │
       │  - I/O ports     │
       │  - CR registers  │   ← Linux kernel runs here
       │  - Interrupt     │
       │    control       │
       └────────┬────────┘
                │ (syscall / sysret)
       ┌────────▼────────┐
       │  Ring 3           │
       │  (User Mode)      │
       │  - Own memory     │   ← Applications run here
       │  - No I/O ports   │
       │  - No privileged  │
       │    instructions   │
       └──────────────────┘

Note: Rings 1 & 2 are unused in Linux


ARM64 Exception Levels:

        EL3 — Secure Monitor
       ┌─────────────────────┐
       │ ARM Trusted Firmware │  ← Secure world management
       │ (ATF / TF-A)        │
       └─────────┬───────────┘
                 │
        EL2 — Hypervisor
       ┌─────────▼───────────┐
       │ KVM / Xen            │  ← Virtualization
       └─────────┬───────────┘
                 │
        EL1 — Kernel
       ┌─────────▼───────────┐
       │ Linux Kernel         │  ← OS kernel
       └─────────┬───────────┘
                 │
        EL0 — User
       ┌─────────▼───────────┐
       │ Applications         │  ← User programs
       └─────────────────────┘
```

Privileged instructions (only in kernel mode):

| Category | x86_64 | ARM64 | Purpose |
|----------|--------|-------|---------|
| Page table | Write to `CR3` | Write to `TTBR0/1` | Switch address space |
| Interrupts | `cli` / `sti` | `MSR DAIFSet/Clr` | Disable/enable interrupts |
| I/O | `in` / `out` | N/A (MMIO only) | Access I/O ports |
| Halt | `hlt` | `wfi` | Wait for interrupt (idle) |
| TLB | `invlpg` | `tlbi` | Flush TLB entry |
| Cache | `wbinvd` | `dc civac` | Cache maintenance |

---

## 3.3 Hardware Memory Architecture

The kernel manages memory through the **Memory Management Unit (MMU)**, which translates virtual addresses to physical addresses.

```
Virtual → Physical Address Translation:

CPU generates                        Physical
Virtual Address                      Address
    │                                   │
    ▼                                   ▼
┌──────────┐    ┌──────────┐    ┌──────────────┐
│   TLB    │───►│  Cache   │    │ Physical RAM  │
│ (fast    │hit │  (L1/L2) │    │              │
│  lookup) │    └──────────┘    └──────────────┘
└────┬─────┘         ▲
     │miss           │
     ▼               │
┌──────────┐         │
│   MMU    │─────────┘
│ (walks   │    Translated physical
│  page    │    address used for
│  tables) │    cache/memory access
└──────────┘

Page Table Walk (4-level, x86_64):

Virtual Address (48-bit used):
┌────────┬────────┬────────┬────────┬──────────────┐
│ PGD    │ PUD    │ PMD    │ PTE    │ Page Offset  │
│(9 bits)│(9 bits)│(9 bits)│(9 bits)│  (12 bits)   │
└───┬────┴───┬────┴───┬────┴───┬────┴──────────────┘
    │        │        │        │
    ▼        ▼        ▼        ▼
  CR3 → PGD → PUD → PMD → PTE → Physical Page + Offset
  (one walk = up to 4 memory accesses → TLB is critical!)
```

Page sizes and their use:

| Page Size | Name | Use Case |
|-----------|------|----------|
| 4 KB | Normal page | Default for most allocations |
| 2 MB | Huge page (x86) / Large page | Performance-critical apps, THP |
| 1 GB | Gigantic page (x86) | Large databases, VMs |
| 16 KB | Configurable (ARM64) | Android default (kernel 6.x) |
| 64 KB | Configurable (ARM64) | Some server/embedded configs |

---

## 3.4 Hardware Interrupt Systems

Interrupts are the mechanism by which hardware notifies the CPU that an event needs attention. The kernel's interrupt handling is critical to system performance.

```
Interrupt Flow: Hardware → CPU → Kernel

         Device (UART, NIC, Timer)
              │
              │ IRQ signal (electrical)
              ▼
     ┌─────────────────┐
     │  Interrupt       │
     │  Controller      │     x86: APIC (Local + IO-APIC)
     │  (GIC / APIC)    │     ARM: GIC (Generic Interrupt Controller)
     └────────┬─────────┘
              │ Interrupt delivered to CPU
              ▼
     ┌─────────────────┐
     │      CPU         │
     │  1. Save state   │ (PC, PSTATE/RFLAGS)
     │  2. Switch to    │ kernel mode (if from user)
     │  3. Jump to      │ interrupt vector
     └────────┬─────────┘
              │
              ▼
     ┌─────────────────┐
     │  Kernel ISR      │
     │  (Interrupt      │ Registered via request_irq()
     │   Service        │
     │   Routine)       │
     └─────────────────┘
```

Interrupt types:

| Type | Source | Example | Handling |
|------|--------|---------|----------|
| **Hardware IRQ** | External device | Timer, NIC, UART | Top-half + bottom-half |
| **Software interrupt** | CPU instruction | `int 0x80`, `svc` | System call entry |
| **Exception** | CPU error/event | Page fault, div-by-0 | Exception handler |
| **NMI** | Critical hardware | Watchdog, memory error | Non-maskable, always handled |
| **IPI** | Another CPU | TLB shootdown, reschedule | Inter-processor communication |

```
Linux Interrupt Processing Model:

┌──────────────────────────────────────────────────────────┐
│                    TOP HALF (hardirq)                     │
│                                                          │
│  - Runs with interrupts disabled (on this line)          │
│  - Must be FAST (microseconds)                           │
│  - Cannot sleep, cannot allocate with GFP_KERNEL         │
│  - Acknowledge hardware, read urgent data                │
│  - Schedule bottom half                                  │
└──────────────────────┬───────────────────────────────────┘
                       │ Schedule deferred work
                       ▼
┌──────────────────────────────────────────────────────────┐
│                  BOTTOM HALF (deferred)                   │
│                                                          │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐   │
│  │  Softirq    │  │  Tasklet     │  │  Workqueue    │   │
│  │  (per-CPU,  │  │  (single CPU │  │  (process ctx │   │
│  │   highest   │  │   atomic,    │  │   CAN sleep,  │   │
│  │   priority) │  │   no sleep)  │  │   schedule)   │   │
│  └─────────────┘  └──────────────┘  └───────────────┘   │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │  Threaded IRQ   (irqreturn_t handler in kthread) │    │
│  │  (process context, CAN sleep, modern preferred)  │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

---

## 3.5 CPU Cores and Multiprocessing

Modern systems have multiple CPU cores. The kernel must manage scheduling, memory coherency, and inter-processor coordination.

```
SMP (Symmetric Multiprocessing) Architecture:

┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│  Core 0  │  │  Core 1  │  │  Core 2  │  │  Core 3  │
│  L1 I/D  │  │  L1 I/D  │  │  L1 I/D  │  │  L1 I/D  │
│  L2      │  │  L2      │  │  L2      │  │  L2      │
└────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │             │
     └──────┬──────┘──────┬──────┘─────────────┘
            │             │
      ┌─────▼─────────────▼──────┐
      │     Shared L3 Cache       │  ← Cache coherency protocol
      │     (MESI / MOESI)        │     keeps caches consistent
      └────────────┬──────────────┘
                   │
      ┌────────────▼──────────────┐
      │     Memory Controller      │
      │     (DDR4/5 DRAM)         │
      └───────────────────────────┘


big.LITTLE / DynamIQ (ARM - common in mobile/automotive):

┌─────────────────────────────────────────────────────────┐
│  big cluster (performance)    LITTLE cluster (efficient) │
│  ┌───────┐  ┌───────┐       ┌───────┐  ┌───────┐      │
│  │ A78   │  │ A78   │       │ A55   │  │ A55   │      │
│  │ Core  │  │ Core  │       │ Core  │  │ Core  │      │
│  └───┬───┘  └───┬───┘       └───┬───┘  └───┬───┘      │
│      └─────┬────┘               └─────┬────┘            │
│            │                          │                  │
│      ┌─────▼─────┐             ┌──────▼─────┐          │
│      │  L3 (big) │             │ L3 (LITTLE)│          │
│      └─────┬─────┘             └──────┬─────┘          │
│            └──────────────┬───────────┘                  │
│                    ┌──────▼─────┐                        │
│                    │   CCI/DSU  │  ← Cache coherent     │
│                    │   (NoC)    │     interconnect       │
│                    └──────┬─────┘                        │
│                    ┌──────▼─────┐                        │
│                    │  Memory    │                        │
│                    └────────────┘                        │
└─────────────────────────────────────────────────────────┘
```

---

## 3.6 Hardware Timers

Timers are essential for scheduling, timeouts, and timekeeping. The kernel relies on hardware timers for its tick and high-resolution timing.

| Timer | Architecture | Resolution | Kernel Use |
|-------|-------------|------------|------------|
| **PIT** (8254) | x86 (legacy) | ~1.19 MHz | Legacy tick source |
| **HPET** | x86 | 10+ MHz | High-precision timer |
| **TSC** | x86 | CPU frequency | Cycle counter, `ktime_get()` |
| **Local APIC Timer** | x86 SMP | CPU frequency | Per-CPU tick, scheduling |
| **ARM Generic Timer** | ARM/ARM64 | Configurable | System tick, scheduling |
| **ARM Arch Timer** | ARM64 | Typically 19.2 MHz | `cntvct_el0`, timestamps |
| **Watchdog Timer** | All | Hardware-specific | System hang detection |

```c
/* How the kernel uses timers */

/* 1. System tick — drives scheduler, timers, jiffies */
void tick_handle_periodic(struct clock_event_device *dev)
{
    tick_periodic(smp_processor_id());
}

/* 2. High-resolution timers — nanosecond precision */
ktime_t now = ktime_get();  /* Uses TSC/Arch Timer */

/* 3. Clocksource — monotonic time reference */
struct clocksource tsc_cs = {
    .name   = "tsc",
    .rating = 300,  /* higher = preferred */
    .read   = read_tsc,
    .mask   = CLOCKSOURCE_MASK(64),
};
```

---

## 3.7 Hardware Device Interfaces

The kernel communicates with hardware through several mechanisms:

```
CPU ←→ Device Communication Methods:

1. MMIO (Memory-Mapped I/O):
   ┌──────┐    Memory Bus    ┌──────────┐
   │ CPU  │ ──────────────── │ Device   │
   │      │  read/write to   │ Registers│
   └──────┘  0x40010000      └──────────┘
   Device registers appear in the memory address space

2. Port I/O (x86 only):
   ┌──────┐    I/O Bus       ┌──────────┐
   │ CPU  │ ──────────────── │ Device   │
   │      │  in/out to       │ Registers│
   └──────┘  port 0x3F8      └──────────┘
   Separate address space, accessed via in/out instructions

3. DMA (Direct Memory Access):
   ┌──────┐                  ┌──────────┐
   │ CPU  │ sets up DMA      │ DMA      │
   │      │ ───────────────► │ Engine   │
   └──────┘ (desc + addr)    └────┬─────┘
                                   │ Transfers data
                              ┌────▼─────┐
                              │   RAM    │
                              └──────────┘
   CPU is free while DMA moves data between device and memory
```

Bus architectures:

| Bus | Speed | Use Case | Discovery |
|-----|-------|----------|-----------|
| **PCI/PCIe** | 2.5-64 GT/s | GPU, NIC, NVMe, USB controller | Auto-enumerated |
| **USB** | 12-20 Gbps | Peripherals, storage, HID | Hot-pluggable, enumerated |
| **I2C** | 100-3400 kHz | Sensors, PMIC, EEPROM, touchscreen | Platform / DT described |
| **SPI** | 1-100+ MHz | Flash, display, fingerprint | Platform / DT described |
| **UART** | 9600-12 Mbps | Debug console, GPS, Bluetooth | Platform / DT described |
| **AMBA/AXI** | SoC internal | SoC peripherals | Platform / DT described |
| **CAN** | 1-8 Mbps | Automotive networking | Platform / DT described |

---

## Kernel Source References

| File | Content |
|------|---------|
| `arch/arm64/include/asm/sysreg.h` | ARM64 system register definitions |
| `arch/x86/include/asm/processor.h` | x86 CPU data structures |
| `arch/arm64/mm/mmu.c` | ARM64 MMU setup |
| `arch/x86/kernel/apic/` | x86 APIC interrupt controller |
| `drivers/irqchip/irq-gic-v3.c` | ARM GICv3 interrupt controller |
| `kernel/time/clocksource.c` | Clock source management |

---

## Interview Questions

**Q1: What is the difference between MMIO and Port I/O?**
A: MMIO maps device registers into the physical address space — accessed via normal load/store or `readl()`/`writel()`. Port I/O (x86 only) uses a separate I/O address space accessed via `inb()`/`outb()` instructions. ARM uses MMIO exclusively. MMIO is dominant in modern hardware.

**Q2: Why is the TLB important for kernel performance?**
A: A page table walk requires up to 4 memory accesses (on 4-level paging). The TLB caches virtual→physical translations, making most address translations a single-cycle lookup. TLB misses significantly impact performance, especially during context switches that may flush the TLB.

**Q3: What is cache coherency and why does the kernel care?**
A: In SMP systems, each core has its own L1/L2 cache. Cache coherency protocols (MESI/MOESI) ensure all cores see consistent data. The kernel must understand this for correct locking, memory barriers, and per-CPU data — otherwise one core might read stale data written by another.

**Q4: Explain the top-half / bottom-half interrupt model.**
A: Top-half (hardirq handler) runs with interrupts disabled — it must be fast, just acknowledge hardware and save critical data. Bottom-half (softirq, tasklet, workqueue, threaded IRQ) runs later with interrupts enabled for longer processing. This minimizes interrupt latency while still handling complex work.

---

## Summary

- The kernel directly interacts with CPU features: registers, privilege levels, MMU, caches, timers
- Privilege levels (Ring 0/EL1 vs Ring 3/EL0) enforce kernel/user separation in hardware
- The MMU translates virtual to physical addresses; TLB caching is critical for performance
- Hardware interrupts use a controller (GIC/APIC) to notify the CPU; kernel uses top-half/bottom-half model
- SMP systems require cache coherency; big.LITTLE adds heterogeneous core management
- Hardware timers drive scheduling, timekeeping, and watchdog functionality
- Device communication uses MMIO (universal), Port I/O (x86 legacy), and DMA (bulk transfers)

---

*Next: [Chapter 4 — Linux Kernel Architecture Overview](Chapter_04_Kernel_Architecture_Overview.md)*
