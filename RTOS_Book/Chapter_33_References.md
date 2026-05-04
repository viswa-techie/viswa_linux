# Chapter 33: Documentation and References

## Learning Goals
- Know essential RTOS documentation and specifications
- Access key reference materials for each major RTOS
- Understand academic foundations and seminal papers
- Build a reference library for RTOS development

---

## 1. Official RTOS Documentation

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  FreeRTOS:                                               │
  │  ├── freertos.org/Documentation                          │
  │  │   └── API Reference (task, queue, semaphore, timer)  │
  │  ├── freertos.org/FreeRTOS-Coding-Standard               │
  │  ├── "Mastering the FreeRTOS Real Time Kernel"           │
  │  │   by Richard Barry (free PDF)                         │
  │  ├── Source code: github.com/FreeRTOS/FreeRTOS-Kernel   │
  │  └── FreeRTOS+TCP, FreeRTOS+CLI, FreeRTOS+FAT           │
  │                                                           │
  │  Zephyr:                                                 │
  │  ├── docs.zephyrproject.org                              │
  │  │   ├── Kernel Services                                │
  │  │   ├── Device Driver Model                            │
  │  │   ├── Device Tree Usage                              │
  │  │   └── Board Porting Guide                            │
  │  ├── Source: github.com/zephyrproject-rtos/zephyr        │
  │  └── Supported boards list: 400+ boards                 │
  │                                                           │
  │  QNX:                                                    │
  │  ├── qnx.com/developers                                 │
  │  │   ├── System Architecture Guide                      │
  │  │   ├── Programmer's Guide                             │
  │  │   └── Neutrino Microkernel (procnto) Reference       │
  │  └── QNX Momentics IDE documentation                    │
  │                                                           │
  │  VxWorks:                                                │
  │  ├── windriver.com/documentation                         │
  │  │   ├── VxWorks Programmer's Guide                     │
  │  │   ├── Kernel API Reference                           │
  │  │   └── BSP Developer's Guide                          │
  │  └── Wind River Workbench help                          │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Standards and Specifications

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Safety Standards:                                       │
  │  ├── IEC 61508: Functional safety (general)             │
  │  ├── ISO 26262: Road vehicles functional safety         │
  │  ├── DO-178C: Software for airborne systems             │
  │  ├── IEC 62304: Medical device software                 │
  │  └── EN 50128: Railway software                         │
  │                                                           │
  │  RTOS Standards:                                         │
  │  ├── OSEK/VDX: Automotive OS standard (basis of AUTOSAR)│
  │  ├── ARINC 653: Avionics partitioned OS                 │
  │  ├── POSIX 1003.13: RT profiles for embedded            │
  │  └── CMSIS-RTOS v2: ARM's RTOS abstraction API         │
  │                                                           │
  │  Communication Standards:                                │
  │  ├── IEEE 802.1Qbv: Time-Sensitive Networking (TSN)     │
  │  ├── IEC 61158: Fieldbus (PROFINET, EtherCAT)           │
  │  ├── ISO 11898: CAN bus protocol                         │
  │  └── SAE J1939: CAN for commercial vehicles             │
  │                                                           │
  │  Coding Standards:                                       │
  │  ├── MISRA C: Guidelines for safety-critical C code     │
  │  ├── CERT C: Secure C coding standard                   │
  │  └── BARR-C: Embedded C coding standard                 │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Seminal Papers and Theory

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Foundational Papers:                                    │
  │                                                           │
  │  1. Liu & Layland (1973)                                 │
  │     "Scheduling Algorithms for Multiprogramming in a     │
  │      Hard-Real-Time Environment"                         │
  │     → RMS, EDF, utilization bound U = n(2^(1/n) - 1)   │
  │     → Most cited paper in real-time systems              │
  │                                                           │
  │  2. Sha, Rajkumar, Lehoczky (1990)                      │
  │     "Priority Inheritance Protocols"                     │
  │     → Priority Inheritance Protocol (PIP)                │
  │     → Priority Ceiling Protocol (PCP)                   │
  │     → Solves unbounded priority inversion               │
  │                                                           │
  │  3. Joseph & Pandya (1986)                               │
  │     "Finding Response Times in a Real-Time System"       │
  │     → Response Time Analysis (exact schedulability test) │
  │     → Iterative formula: Ri = Ci + Σ⌈Ri/Tj⌉ × Cj      │
  │                                                           │
  │  4. Coffman et al. (1971)                                │
  │     "System Deadlocks"                                   │
  │     → Four necessary conditions for deadlock             │
  │     → Mutual exclusion, hold-and-wait, no preemption,   │
  │       circular wait                                      │
  │                                                           │
  │  5. Buttazzo (2011)                                      │
  │     "Hard Real-Time Computing Systems"                   │
  │     → Comprehensive textbook on RT theory                │
  │     → EDF, server algorithms, resource access protocols │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Books

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Recommended Reading (by topic):                         │
  │                                                           │
  │  FreeRTOS-specific:                                      │
  │  1. "Mastering the FreeRTOS Real Time Kernel"            │
  │     Richard Barry (free, official guide)                 │
  │  2. "FreeRTOS Reference Manual" (API documentation)     │
  │                                                           │
  │  RTOS Theory:                                            │
  │  3. "Hard Real-Time Computing Systems" — G. Buttazzo    │
  │     (comprehensive theory: scheduling, resource access) │
  │  4. "Real-Time Systems" — Jane Liu                      │
  │     (scheduling algorithms, analysis techniques)        │
  │                                                           │
  │  Embedded Systems:                                       │
  │  5. "Making Embedded Systems" — Elecia White            │
  │     (practical: from hardware to firmware)              │
  │  6. "The Definitive Guide to ARM Cortex-M3/M4"          │
  │     — Joseph Yiu (ARM architecture reference)           │
  │                                                           │
  │  Safety-Critical:                                        │
  │  7. "Safety-Critical Systems Handbook" — D. Smith       │
  │  8. MISRA C:2012 (coding guidelines document)           │
  │                                                           │
  │  Operating Systems (foundational):                       │
  │  9. "Operating System Concepts" — Silberschatz          │
  │     (general OS: processes, memory, scheduling)         │
  │  10. "Operating Systems: Three Easy Pieces" — Arpaci    │
  │      (free online, excellent teaching)                  │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. ARM Architecture References

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ARM Documentation (developer.arm.com):                  │
  │                                                           │
  │  Architecture Reference Manuals (ARM ARM):               │
  │  ├── ARMv7-M: Cortex-M3/M4/M7 instruction set          │
  │  ├── ARMv8-M: Cortex-M23/M33/M55 (TrustZone-M)        │
  │  └── ARMv7-R: Cortex-R4/R5 (safety, lockstep)          │
  │                                                           │
  │  Technical Reference Manuals (TRM):                      │
  │  ├── Cortex-M4 TRM: pipeline, MPU, FPU details          │
  │  ├── NVIC: Nested Vectored Interrupt Controller         │
  │  └── CoreSight: ETM, ITM, DWT trace components         │
  │                                                           │
  │  CMSIS (Cortex Microcontroller Software Interface Std):  │
  │  ├── CMSIS-Core: register access macros (SCB, NVIC)     │
  │  ├── CMSIS-RTOS v2: standardized RTOS API               │
  │  ├── CMSIS-DSP: signal processing library               │
  │  └── CMSIS-SVD: register description for debuggers      │
  │                                                           │
  │  MCU Vendor Documentation:                               │
  │  ├── STM32: Reference Manual (RM0090 for STM32F4)       │
  │  │   → Peripheral registers, clock tree, pin mapping    │
  │  ├── NXP: User Manual per MCU family                    │
  │  └── Nordic: nRF5 SDK + Infocenter documentation        │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. Online Resources

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Community and Learning:                                 │
  │  ├── freertos.org/FreeRTOS-quick-start-guide            │
  │  ├── digikey.com RTOS tutorial series (Shawn Hymel)     │
  │  ├── embedded.fm (podcast on embedded systems)          │
  │  └── reddit.com/r/embedded                               │
  │                                                           │
  │  Source Code:                                            │
  │  ├── github.com/FreeRTOS/FreeRTOS-Kernel                │
  │  ├── github.com/zephyrproject-rtos/zephyr               │
  │  ├── github.com/ARM-software/CMSIS_5                    │
  │  └── github.com/STMicroelectronics/STM32CubeF4          │
  │                                                           │
  │  Tools:                                                  │
  │  ├── segger.com (J-Link, SystemView, Ozone, RTT)       │
  │  ├── percepio.com (Tracealyzer)                         │
  │  ├── openocd.org (open-source debug server)             │
  │  └── arm-none-eabi-gcc (GNU ARM toolchain)              │
  └──────────────────────────────────────────────────────────┘
```

---

## Summary

- Official docs: FreeRTOS (freertos.org), Zephyr (docs.zephyrproject.org), QNX, VxWorks
- Key standards: IEC 61508 (safety), ISO 26262 (auto), DO-178C (avionics), MISRA C (coding)
- Seminal papers: Liu & Layland 1973 (RMS/EDF), Sha 1990 (priority inheritance)
- Essential books: "Mastering FreeRTOS" (free), "Hard RT Computing Systems" (Buttazzo), "Definitive Guide to ARM Cortex-M" (Yiu)
- ARM docs: Architecture Reference Manual, CMSIS, vendor reference manuals
- Open source: FreeRTOS Kernel, Zephyr, CMSIS on GitHub

---

[Previous Chapter: RTOS vs Linux ←](Chapter_32_RTOS_vs_Linux.md) | [Next Chapter: Interview Preparation →](Chapter_34_Interview.md)
