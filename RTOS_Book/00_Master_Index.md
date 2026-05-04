# Real-Time Operating Systems (RTOS) — Complete Guide

## Master Index

```
  RTOS Architecture Overview
  ═══════════════════════════════════════════════════════════════

  ┌─ Application Tasks ─────────────────────────────────────────┐
  │  Task A (High Priority) │ Task B (Med) │ Task C (Low)       │
  │  Periodic: 10ms         │ Event-driven │ Background          │
  └──────────┬──────────────┴──────┬───────┴────────┬───────────┘
             │                     │                │
  ┌──────────▼─────────────────────▼────────────────▼───────────┐
  │                    RTOS Kernel                               │
  │  ┌──────────┐ ┌───────────┐ ┌──────────┐ ┌──────────────┐  │
  │  │Scheduler │ │  IPC /    │ │ Memory   │ │   Timer      │  │
  │  │(Priority │ │  Sync     │ │ Manager  │ │   Manager    │  │
  │  │Preemptive│ │(Mutex/Sem)│ │(Pool/    │ │(Tick/Tickless│  │
  │  │RMS/EDF)  │ │(Queue/Evt)│ │ Static)  │ │ SW Timers)   │  │
  │  └────┬─────┘ └─────┬─────┘ └────┬─────┘ └──────┬───────┘  │
  │       │              │            │              │           │
  │  ┌────▼──────────────▼────────────▼──────────────▼───────┐  │
  │  │              Hardware Abstraction Layer (HAL)          │  │
  │  └──────────────────────────┬────────────────────────────┘  │
  └─────────────────────────────┼───────────────────────────────┘
                                │
  ┌─────────────────────────────▼───────────────────────────────┐
  │                      Hardware                                │
  │  CPU │ Interrupt Controller │ Timers │ Memory │ Peripherals  │
  │  (ARM Cortex-M/R/A, RISC-V, x86, PowerPC)                  │
  └──────────────────────────────────────────────────────────────┘
```

---

## Part I: Foundations (Chapters 1-3)

| # | Chapter | Key Topics |
|---|---------|------------|
| 1 | [Foundations of Real-Time Systems](Chapter_01_Foundations.md) | Hard/soft RT, determinism, latency, jitter, deadlines |
| 2 | [History and Evolution of RTOS](Chapter_02_History_Evolution.md) | Early RT kernels, VxWorks, QNX, FreeRTOS, Zephyr timeline |
| 3 | [RTOS Architecture Overview](Chapter_03_Architecture.md) | Monolithic, microkernel, hybrid designs, kernel components |

## Part II: Hardware & Task Management (Chapters 4-7)

| # | Chapter | Key Topics |
|---|---------|------------|
| 4 | [Hardware Architecture for RTOS](Chapter_04_Hardware.md) | ARM Cortex-M/R, interrupt controllers, timers, MPU |
| 5 | [Task Management](Chapter_05_Task_Management.md) | TCB, task states, lifecycle, stack, task creation |
| 6 | [Real-Time Scheduling](Chapter_06_Scheduling.md) | Preemptive, cooperative, fixed-priority, round-robin |
| 7 | [Real-Time Scheduling Algorithms](Chapter_07_Scheduling_Algorithms.md) | RMS, EDF, DMS, schedulability analysis |

## Part III: Execution & Communication (Chapters 8-12)

| # | Chapter | Key Topics |
|---|---------|------------|
| 8 | [Context Switching](Chapter_08_Context_Switching.md) | Register save/restore, PendSV, stack switching, cost |
| 9 | [Interrupt Handling in RTOS](Chapter_09_Interrupt_Handling.md) | ISR design, latency, NVIC, deferred processing |
| 10 | [Inter-Task Communication](Chapter_10_IPC.md) | Message queues, mailboxes, pipes, event groups |
| 11 | [Synchronization Mechanisms](Chapter_11_Synchronization.md) | Mutex, semaphore, event flags, critical sections |
| 12 | [Priority Inversion](Chapter_12_Priority_Inversion.md) | Problem, inheritance, ceiling protocol, Mars Pathfinder |

## Part IV: Memory & Time (Chapters 13-17)

| # | Chapter | Key Topics |
|---|---------|------------|
| 13 | [Memory Management in RTOS](Chapter_13_Memory_Management.md) | Static, pools, heap strategies, MPU protection |
| 14 | [Real-Time Memory Constraints](Chapter_14_RT_Memory.md) | Fragmentation, WCET impact, stack overflow detection |
| 15 | [Time Management](Chapter_15_Time_Management.md) | System tick, SW timers, HW timers, tickless idle |
| 16 | [Device Drivers in RTOS](Chapter_16_Device_Drivers.md) | Driver architecture, interrupt-driven, DMA, HAL |
| 17 | [RTOS I/O Management](Chapter_17_IO_Management.md) | Buffering, async I/O, device abstraction |

## Part V: Networking & Communication (Chapters 18-22)

| # | Chapter | Key Topics |
|---|---------|------------|
| 18 | [RTOS Networking](Chapter_18_Networking.md) | lwIP, TCP/IP stacks, socket API, network drivers |
| 19 | [File Systems in RTOS](Chapter_19_File_Systems.md) | FAT, LittleFS, SPIFFS, flash wear leveling |
| 20 | [Real-Time Communication Protocols](Chapter_20_RT_Protocols.md) | CAN bus, Automotive Ethernet, EtherCAT, MQTT |
| 21 | [Power Management in RTOS](Chapter_21_Power_Management.md) | Sleep modes, tickless idle, power-aware scheduling |
| 22 | [Safety and Reliability](Chapter_22_Safety_Reliability.md) | Watchdog, fault tolerance, FMEA, IEC 61508, ISO 26262 |

## Part VI: Security & Development (Chapters 23-28)

| # | Chapter | Key Topics |
|---|---------|------------|
| 23 | [Security in RTOS](Chapter_23_Security.md) | Secure boot, TrustZone, crypto, firmware update |
| 24 | [RTOS Debugging](Chapter_24_Debugging.md) | JTAG, SWD, ITM trace, Segger SystemView |
| 25 | [Performance Optimization](Chapter_25_Performance.md) | Latency, ISR optimization, memory tuning, WCET |
| 26 | [RTOS Porting](Chapter_26_Porting.md) | BSP, HAL, arch-specific code, startup sequence |
| 27 | [RTOS Build Systems](Chapter_27_Build_Systems.md) | Cross-compilation, CMake, Kconfig, firmware images |
| 28 | [RTOS Kernel Source Structure](Chapter_28_Source_Structure.md) | FreeRTOS/Zephyr/QNX source layout, key files |

## Part VII: Reference & Interview (Chapters 29-34)

| # | Chapter | Key Topics |
|---|---------|------------|
| 29 | [RTOS System Flow Diagrams](Chapter_29_System_Flow.md) | Boot, scheduler, ISR, task lifecycle flows |
| 30 | [RTOS Debugging and Monitoring Tools](Chapter_30_Debug_Tools.md) | JTAG, Tracealyzer, SystemView, OpenOCD |
| 31 | [RTOS in Different Domains](Chapter_31_Domains.md) | Automotive, aerospace, industrial, medical, IoT |
| 32 | [RTOS vs Linux Comparison](Chapter_32_RTOS_vs_Linux.md) | Scheduling, memory, determinism, when to use which |
| 33 | [Documentation and References](Chapter_33_References.md) | Specs, books, vendor docs, standards |
| 34 | [Interview Preparation](Chapter_34_Interview_Prep.md) | Top questions, scheduling scenarios, code problems |

---

## Reading Paths

**Embedded Developer Path:**
Ch 1 → 4 → 5 → 6 → 8 → 9 → 11 → 13 → 16 → 26

**Scheduling Deep Dive:**
Ch 1 → 6 → 7 → 8 → 12 → 25

**Automotive RTOS Path:**
Ch 1 → 3 → 9 → 11 → 12 → 20 → 22 → 31

**Interview Fast Track:**
Ch 1 → 5 → 6 → 7 → 9 → 11 → 12 → 32 → 34

---

## RTOS Comparison Quick Reference

```
  ┌──────────┬──────────┬──────────┬──────────┬──────────┐
  │ Feature  │ FreeRTOS │ Zephyr   │ QNX      │ VxWorks  │
  ├──────────┼──────────┼──────────┼──────────┼──────────┤
  │ Type     │ Kernel   │ Full RTOS│ Microknl │ Monolith │
  │ License  │ MIT      │ Apache2  │ Commercl │ Commercl │
  │ Targets  │ MCU      │ MCU+MPU  │ MPU/SoC  │ MPU/SoC  │
  │ Min RAM  │ ~4 KB    │ ~8 KB    │ ~512 KB  │ ~256 KB  │
  │ Scheduling│Preemptive│Preemptive│Preemptive│Preemptive│
  │ POSIX    │ Partial  │ Partial  │ Full     │ Full     │
  │ Safety   │ Certifbl │ Certifbl │IEC61508  │DO-178B   │
  │ Domains  │ IoT,Embdd│IoT,Indstl│Auto,Aero│Aero,Def  │
  └──────────┴──────────┴──────────┴──────────┴──────────┘
```
