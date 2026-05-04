# Chapter 4: Hardware Architecture for RTOS

## Learning Goals
- Understand CPU architectures used in embedded RTOS systems
- Learn interrupt controller architecture (NVIC, GIC)
- Know memory architecture: Flash, SRAM, MPU
- Understand hardware timers and their RTOS role
- Learn peripheral interfaces commonly managed by RTOS

---

## 1. CPU Architectures for RTOS

```
  ARM Processor Family for RTOS
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Cortex-M (Microcontroller Profile):                     │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │ M0/M0+: Ultra-low power, simple pipeline         │    │
  │  │         32-bit, Thumb-2, no MPU, no FPU           │    │
  │  │         Use: sensors, wearables                   │    │
  │  │                                                    │    │
  │  │ M3:     3-stage pipeline, hardware divide          │    │
  │  │         Optional MPU, Thumb-2 only                │    │
  │  │         Use: industrial control, motor drive       │    │
  │  │                                                    │    │
  │  │ M4:     M3 + DSP + optional FPU (single precision)│    │
  │  │         Use: audio, motor control, IoT gateways   │    │
  │  │                                                    │    │
  │  │ M7:     6-stage pipeline, dual-issue, FPU (DP)    │    │
  │  │         Instruction+data cache, TCM               │    │
  │  │         Use: high-perf embedded, automotive       │    │
  │  │                                                    │    │
  │  │ M33:    M3 + TrustZone security + DSP             │    │
  │  │         Secure/Non-secure worlds                   │    │
  │  │         Use: IoT security, smart meters           │    │
  │  └──────────────────────────────────────────────────┘    │
  │                                                           │
  │  Cortex-R (Real-time Profile):                           │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │ R4/R5:  Deterministic, dual-core lockstep         │    │
  │  │         TCM, ECC memory, low-latency interrupts   │    │
  │  │         Use: automotive (AUTOSAR), storage, 5G    │    │
  │  │                                                    │    │
  │  │ R52:    ARM v8-R, virtualization support           │    │
  │  │         Hypervisor for multiple RTOS instances     │    │
  │  │         Use: next-gen automotive, safety systems  │    │
  │  └──────────────────────────────────────────────────┘    │
  │                                                           │
  │  Cortex-A (Application Profile):                         │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │ A53/A55/A78: Full MMU, Linux-capable              │    │
  │  │ Used for QNX, VxWorks on MPU-class processors     │    │
  │  │ AMP: A-core runs Linux, M/R-core runs RTOS       │    │
  │  └──────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Interrupt Controller (NVIC)

```
  ARM Cortex-M NVIC Architecture
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Nested Vectored Interrupt Controller (NVIC):            │
  │                                                           │
  │  ┌── Interrupt Sources ────┐    ┌── NVIC ──────────┐    │
  │  │ IRQ0: UART RX           │──► │                   │    │
  │  │ IRQ1: Timer0             │──► │ Priority decode  │    │
  │  │ IRQ2: SPI complete       │──► │ (preemption +    │──► CPU │
  │  │ IRQ3: GPIO edge          │──► │  sub-priority)   │    │
  │  │ ...                      │──► │                   │    │
  │  │ IRQ239: (up to 240)      │──► │ Tail-chaining    │    │
  │  │                          │    │ Late arrival      │    │
  │  │ NMI: Non-maskable        │──► │ (optimization)   │    │
  │  │ SysTick: System timer    │──► │                   │    │
  │  │ PendSV: Context switch   │──► │                   │    │
  │  │ SVCall: System call      │──► │                   │    │
  │  └─────────────────────────┘    └───────────────────┘    │
  │                                                           │
  │  Key NVIC Features for RTOS:                             │
  │                                                           │
  │  1. Automatic context save: pushes R0-R3,R12,LR,PC,xPSR│
  │     on interrupt entry (12 CPU cycles on Cortex-M4)      │
  │                                                           │
  │  2. Tail-chaining: if higher-priority IRQ pending when   │
  │     current ISR finishes, enter next ISR without full     │
  │     unstacking/restacking (6 cycles vs 12+12)            │
  │                                                           │
  │  3. Late arrival: if higher-priority IRQ arrives during  │
  │     stacking of lower-priority IRQ, redirect to higher   │
  │                                                           │
  │  4. Priority grouping: preemption priority + sub-priority│
  │     NVIC_SetPriorityGrouping() splits 8 bits             │
  │                                                           │
  │  5. PendSV: lowest-priority exception, used by RTOS     │
  │     for context switching (deferred from ISR)            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Memory Architecture

```
  Typical MCU Memory Map (Cortex-M)
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Address         │ Region          │ Content              │
  │  ─────────────────┼─────────────────┼────────────────────│
  │  0x0000_0000     │ Flash (Code)    │ Vector table        │
  │  0x0000_0004     │                 │ Reset handler       │
  │  ...             │                 │ .text (code)        │
  │  0x000x_xxxx     │                 │ .rodata (const)     │
  │                   │                 │ 256KB - 2MB typical │
  │                   │                 │                     │
  │  0x2000_0000     │ SRAM            │ .data (initialized) │
  │  ...             │                 │ .bss (zeroed)       │
  │  ...             │                 │ Heap                │
  │  ...             │                 │ Task stacks         │
  │  0x2000_xxxx     │                 │ Main stack (MSP)    │
  │                   │                 │ 64KB - 512KB typical│
  │                   │                 │                     │
  │  0x4000_0000     │ Peripherals     │ UART, SPI, I2C,    │
  │  ...             │                 │ Timer, GPIO, ADC    │
  │                   │                 │ Memory-mapped I/O   │
  │                   │                 │                     │
  │  0xE000_0000     │ System          │ NVIC, SysTick,     │
  │  ...             │                 │ SCB, MPU, Debug     │
  │  0xE000_E010     │                 │ SysTick registers   │
  │  0xE000_E100     │                 │ NVIC registers      │
  │  0xE000_ED00     │                 │ SCB registers       │
  │  0xE000_ED90     │                 │ MPU registers       │
  └──────────────────────────────────────────────────────────┘

  Task Stack Layout in SRAM:
  ┌──────────────────────────────────────────────────────────┐
  │  Low Address                                             │
  │  ┌────────────────┐                                      │
  │  │ Task A stack   │ ← Stack grows downward               │
  │  │ (1024 bytes)   │                                      │
  │  ├────────────────┤ ← Stack canary / guard pattern       │
  │  │ Task B stack   │                                      │
  │  │ (512 bytes)    │                                      │
  │  ├────────────────┤                                      │
  │  │ Task C stack   │                                      │
  │  │ (2048 bytes)   │                                      │
  │  ├────────────────┤                                      │
  │  │ Heap           │                                      │
  │  ├────────────────┤                                      │
  │  │ .bss / .data   │                                      │
  │  └────────────────┘                                      │
  │  High Address                                            │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. MPU (Memory Protection Unit)

```c
/* MPU configuration for RTOS task isolation */

/* Cortex-M MPU: 8-16 regions, no address translation
 * (unlike MMU — MPU only provides protection, not virtual memory)
 */

/* Region 0: Flash (code) — read-only, executable */
MPU->RBAR = 0x00000000 | MPU_RBAR_VALID | 0;
MPU->RASR = MPU_RASR_ENABLE
          | (17 << MPU_RASR_SIZE_Pos)  /* 256KB */
          | MPU_RASR_AP_RO_RO          /* Read-only */
          | MPU_RASR_XN_NO;            /* Executable */

/* Region 1: Task A stack — read/write, no execute */
MPU->RBAR = 0x20001000 | MPU_RBAR_VALID | 1;
MPU->RASR = MPU_RASR_ENABLE
          | (9 << MPU_RASR_SIZE_Pos)   /* 1KB */
          | MPU_RASR_AP_RW_RW          /* Read/Write */
          | MPU_RASR_XN_YES;           /* No execute */

/* Region 2: Peripherals — read/write, device memory */
MPU->RBAR = 0x40000000 | MPU_RBAR_VALID | 2;
MPU->RASR = MPU_RASR_ENABLE
          | (28 << MPU_RASR_SIZE_Pos)  /* 512MB */
          | MPU_RASR_AP_RW_RW
          | MPU_RASR_XN_YES
          | MPU_RASR_TEX_DEVICE;       /* Device memory */

/*
 * RTOS reconfigures MPU on each context switch:
 * - Update task stack region to current task's stack
 * - Optionally restrict peripheral access per task
 * - Fault on access violation → MemManage_Handler
 */
```

---

## 5. Hardware Timers

```
  Timers Used by RTOS
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  SysTick Timer (ARM Cortex-M system timer):              │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ 24-bit down counter, auto-reload                │      │
  │  │ Generates periodic interrupt for RTOS tick      │      │
  │  │                                                  │      │
  │  │ SysTick->LOAD = (SystemCoreClock / 1000) - 1;   │      │
  │  │ /* 1ms tick at given clock frequency */           │      │
  │  │                                                  │      │
  │  │ SysTick_Handler() {                              │      │
  │  │     xTaskIncrementTick();  /* FreeRTOS */        │      │
  │  │     /* Check if context switch needed */         │      │
  │  │     portYIELD_FROM_ISR();                        │      │
  │  │ }                                                │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  General Purpose Timers (TIMx):                          │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ 16/32-bit, multiple channels                    │      │
  │  │ Used for:                                       │      │
  │  │ - PWM generation (motor control)                │      │
  │  │ - Input capture (pulse measurement)             │      │
  │  │ - One-shot delays                               │      │
  │  │ - Higher-resolution timing than SysTick         │      │
  │  │ - Tickless idle: reload with sleep duration     │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  RTC (Real-Time Clock):                                  │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ Battery-backed calendar/time                    │      │
  │  │ Wakeup from deep sleep modes                    │      │
  │  │ Low-power alarm for scheduled wakeups           │      │
  │  └────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Why is PendSV used for context switching instead of SysTick on Cortex-M?**
**A:** PendSV is set to the lowest exception priority, ensuring context switches happen only when no other ISR is running. If context switch happened in SysTick (which has configurable priority), it could delay higher-priority ISR processing. Pattern: SysTick determines a switch is needed, sets PendSV pending (`SCB->ICSR |= SCB_ICSR_PENDSVSET`), SysTick returns, all pending ISRs complete, then PendSV fires at lowest priority to perform the actual context switch. This guarantees ISR latency isn't affected by scheduling.

**Q2: What is the difference between MPU and MMU?**
**A:** MPU (Memory Protection Unit): provides access control (R/W/X permissions) for memory regions but NO virtual address translation. All addresses are physical. Found in Cortex-M/R MCUs. Typically 8-16 regions. Used by RTOS for task stack protection. MMU (Memory Management Unit): provides BOTH virtual-to-physical address translation (page tables) AND access control. Found in Cortex-A processors. Required for microkernels (QNX) and full OSes (Linux). Enables process isolation with separate address spaces. RTOS on MCU uses MPU; RTOS on MPU-class uses MMU.

**Q3: How does the NVIC's tail-chaining optimize interrupt handling?**
**A:** When an ISR completes and another IRQ is pending, tail-chaining skips the full unstack/restack of registers. Instead of 12 cycles (unstack) + 12 cycles (restack) = 24 cycles, tail-chaining takes only 6 cycles to switch to the next ISR. This is critical for RTOS where multiple interrupts (timer, UART, DMA) may fire in rapid succession. Combined with late-arrival (redirect stacking to higher-priority IRQ that arrives during stacking), NVIC ensures deterministic, low-latency interrupt handling.

---

## Summary

- ARM Cortex-M (MCU): primary RTOS target, no MMU, optional MPU, NVIC for interrupts
- ARM Cortex-R (real-time): deterministic, lockstep, TCM, ECC — automotive/safety
- NVIC: nested priorities, automatic context save, tail-chaining, PendSV for context switch
- MCU memory: Flash (code) + SRAM (stacks, heap, data) + peripheral registers
- MPU provides protection without virtual memory (8-16 regions, reconfigured per task)
- SysTick drives RTOS tick; general-purpose timers for PWM/capture/tickless idle
- PendSV at lowest priority ensures context switching doesn't delay ISRs

---

[Previous Chapter: RTOS Architecture ←](Chapter_03_Architecture.md) | [Next Chapter: Task Management →](Chapter_05_Task_Management.md)
