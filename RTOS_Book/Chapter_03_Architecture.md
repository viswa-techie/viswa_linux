# Chapter 3: RTOS Architecture Overview

## Learning Goals
- Understand monolithic, microkernel, and hybrid RTOS architectures
- Know the tradeoffs of each architecture for real-time performance
- Learn core RTOS kernel components and their roles
- Compare architectures across FreeRTOS, QNX, VxWorks, and Zephyr

---

## 1. Monolithic RTOS Kernel

```
  Monolithic RTOS Architecture (VxWorks, FreeRTOS)
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌── User / Application Space ───────────────────────┐   │
  │  │  Task A │ Task B │ Task C │ ... │ Task N           │   │
  │  └──────────────────────┬────────────────────────────┘   │
  │                         │ Direct function calls          │
  │  ┌──────────────────────▼────────────────────────────┐   │
  │  │            RTOS Kernel (single address space)      │   │
  │  │                                                     │   │
  │  │  ┌──────────┐ ┌─────────┐ ┌──────────┐            │   │
  │  │  │Scheduler │ │  IPC    │ │ Memory   │            │   │
  │  │  │          │ │(Queues, │ │ Manager  │            │   │
  │  │  │          │ │ Mutex)  │ │          │            │   │
  │  │  └──────────┘ └─────────┘ └──────────┘            │   │
  │  │  ┌──────────┐ ┌─────────┐ ┌──────────┐            │   │
  │  │  │ Timer    │ │ Device  │ │ Network  │            │   │
  │  │  │ Manager  │ │ Drivers │ │ Stack    │            │   │
  │  │  └──────────┘ └─────────┘ └──────────┘            │   │
  │  └────────────────────────────────────────────────────┘   │
  │                         │                                 │
  │  ┌──────────────────────▼────────────────────────────┐   │
  │  │                   Hardware                         │   │
  │  └────────────────────────────────────────────────────┘   │
  │                                                           │
  │  Pros:                                                   │
  │  + Fast: no mode switches, direct function calls         │
  │  + Low latency: no IPC overhead between components       │
  │  + Simple: everything in one binary                      │
  │                                                           │
  │  Cons:                                                   │
  │  - No memory protection between tasks (typically)        │
  │  - Driver bug crashes entire system                      │
  │  - Larger trusted computing base                         │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Microkernel RTOS Architecture

```
  Microkernel Architecture (QNX Neutrino)
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌── User Space Processes ───────────────────────────┐   │
  │  │                                                     │   │
  │  │  ┌──────────┐ ┌──────────┐ ┌──────────────────┐   │   │
  │  │  │  App 1   │ │  App 2   │ │  System Services  │   │   │
  │  │  └────┬─────┘ └────┬─────┘ │  ┌─────────────┐ │   │   │
  │  │       │             │       │  │ File System │ │   │   │
  │  │       │             │       │  │ Network Stk │ │   │   │
  │  │       │             │       │  │ Device Drvrs│ │   │   │
  │  │       │             │       │  │ USB Stack   │ │   │   │
  │  │       │             │       │  └─────────────┘ │   │   │
  │  │       │             │       └──────────┬───────┘   │   │
  │  └───────┼─────────────┼──────────────────┼───────────┘   │
  │          │ Message      │ Message          │ Message       │
  │          │ passing      │ passing          │ passing       │
  │  ┌───────▼─────────────▼──────────────────▼───────────┐   │
  │  │          Microkernel (minimal)                      │   │
  │  │                                                     │   │
  │  │  Only:                                              │   │
  │  │  - Thread scheduling                                │   │
  │  │  - Message passing (IPC)                            │   │
  │  │  - Interrupt routing                                │   │
  │  │  - Timer management                                 │   │
  │  │                                                     │   │
  │  │  Size: ~100KB (QNX Neutrino microkernel)            │   │
  │  └────────────────────────────────────────────────────┘   │
  │                                                           │
  │  Pros:                                                   │
  │  + Memory protection: driver crash doesn't kill system   │
  │  + Self-healing: restart crashed service transparently   │
  │  + Small TCB (Trusted Computing Base): easier to certify │
  │  + Clean separation of concerns                          │
  │                                                           │
  │  Cons:                                                   │
  │  - IPC overhead: message passing slower than function call│
  │  - More complex system design                            │
  │  - Higher memory footprint (MMU required)                │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Hybrid RTOS Designs

```
  Hybrid Approaches
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  1. Zephyr (kernel + optional subsystems):               │
  │  ┌────────────────────────────────────────────────┐      │
  │  │  Application                                    │      │
  │  │  ┌──────────────────────────────────────────┐  │      │
  │  │  │ Subsystems (compiled in as needed):       │  │      │
  │  │  │ Bluetooth, WiFi, USB, FS, Shell, Logging │  │      │
  │  │  └──────────────────────────────────────────┘  │      │
  │  │  ┌──────────────────────────────────────────┐  │      │
  │  │  │ Kernel: scheduler, IPC, memory, timers   │  │      │
  │  │  └──────────────────────────────────────────┘  │      │
  │  │  ┌──────────────────────────────────────────┐  │      │
  │  │  │ HAL / Device Drivers / Arch code          │  │      │
  │  │  └──────────────────────────────────────────┘  │      │
  │  └────────────────────────────────────────────────┘      │
  │  → Monolithic binary, but modular (select via Kconfig)   │
  │  → Optional user mode with MPU protection                │
  │                                                           │
  │  2. Asymmetric Multiprocessing (AMP):                    │
  │  ┌──────────────────────────────────────────────┐        │
  │  │  Core 0 (Cortex-A)  │  Core 1 (Cortex-M/R)  │        │
  │  │  ┌─────────────┐    │  ┌──────────────┐      │        │
  │  │  │  Linux       │    │  │  FreeRTOS    │      │        │
  │  │  │  (UI, net,   │◄──►│  │  (RT control,│      │        │
  │  │  │   storage)   │ IPC│  │   motor,     │      │        │
  │  │  └─────────────┘    │  │   safety)     │      │        │
  │  │                     │  └──────────────┘      │        │
  │  └──────────────────────────────────────────────┘        │
  │  → Linux for complex tasks, RTOS for deterministic tasks │
  │  → IPC via shared memory + OpenAMP/RPMsg                 │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. RTOS Kernel Components

```
  Core Kernel Components
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Component        │ Role                                  │
  │  ──────────────────┼─────────────────────────────────────│
  │  Scheduler        │ Selects next task to run based on    │
  │                   │ priority. O(1) for determinism.      │
  │                   │ Preemptive in most RTOSes.           │
  │                   │                                      │
  │  Task Manager     │ Creates, deletes, suspends tasks.    │
  │                   │ Manages TCB (Task Control Block).    │
  │                   │                                      │
  │  IPC              │ Queues, mailboxes, event groups,     │
  │                   │ message passing between tasks.       │
  │                   │                                      │
  │  Synchronization  │ Mutexes, semaphores, event flags,    │
  │                   │ critical sections for shared access. │
  │                   │                                      │
  │  Timer Manager    │ System tick, software timers,        │
  │                   │ tickless idle support.               │
  │                   │                                      │
  │  Memory Manager   │ Static allocation, memory pools,     │
  │                   │ optional heap with deterministic     │
  │                   │ allocation time.                     │
  │                   │                                      │
  │  Interrupt Manager│ ISR registration, priority config,   │
  │                   │ deferred processing (bottom halves). │
  │                   │                                      │
  │  HAL              │ Hardware Abstraction Layer —          │
  │                   │ arch-specific code isolated here.    │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Architecture Comparison

```
  Architecture Decision Matrix
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │                   Monolithic  Microkernel  Hybrid         │
  │  ──────────────── ─────────── ─────────── ─────────      │
  │  IPC speed        Fast        Slower       Medium        │
  │  Isolation        None        Full (MMU)   Optional(MPU) │
  │  Fault tolerance  Low         High         Medium        │
  │  Complexity       Low         High         Medium        │
  │  Memory footprint Smallest    Largest      Medium        │
  │  Certification    Harder*     Easier**     Medium        │
  │  Boot time        Fastest     Slowest      Medium        │
  │  Target MCU       Yes         No(needs MMU)Usually MCU   │
  │                                                           │
  │  * Larger TCB makes certification more expensive         │
  │  ** Smaller kernel = less code to certify                │
  │                                                           │
  │  Examples:                                               │
  │  Monolithic: FreeRTOS, VxWorks, ThreadX                  │
  │  Microkernel: QNX Neutrino, seL4, INTEGRITY              │
  │  Hybrid: Zephyr, NuttX                                   │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: What are the advantages of a microkernel RTOS over a monolithic one?**
**A:** Microkernel advantages: (1) Memory isolation — drivers and services run in separate address spaces, so a driver crash doesn't take down the system. (2) Self-healing — crashed services can be automatically restarted without rebooting. (3) Smaller TCB (Trusted Computing Base) — less code to certify for safety standards (IEC 61508, ISO 26262). (4) Clean modularity — services communicate via message passing. Disadvantage: IPC overhead adds latency (message pass vs function call), and requires an MMU, making it unsuitable for small MCUs.

**Q2: Why is FreeRTOS considered monolithic even though it's so small?**
**A:** FreeRTOS is monolithic because all kernel components (scheduler, IPC, memory management) and application tasks run in a single address space with no memory protection between them. A bug in any task can corrupt kernel data. Despite being only ~9000 lines and 4KB RAM, the architecture is monolithic — a task calling `xQueueSend()` directly invokes kernel code without any mode switch or address space boundary. This is by design for MCUs that lack an MMU.

**Q3: When would you choose an AMP (Linux + RTOS) design?**
**A:** AMP is ideal when a system needs both rich OS features AND hard real-time guarantees: (1) Automotive IVI: Linux runs infotainment UI, RTOS handles vehicle control. (2) Industrial: Linux runs HMI/networking, RTOS controls motors/sensors. (3) Robotics: Linux runs computer vision/planning, RTOS controls actuators with microsecond precision. Communication via shared memory (OpenAMP/RPMsg). The Linux core handles complex, non-deterministic tasks while the RTOS core guarantees timing.

---

## Summary

- Monolithic RTOS (FreeRTOS, VxWorks): single address space, fast, no isolation, smallest footprint
- Microkernel RTOS (QNX): isolated services, self-healing, requires MMU, IPC overhead
- Hybrid: combine monolithic simplicity with optional protection (Zephyr MPU) or AMP (Linux+RTOS)
- Core components: scheduler, task manager, IPC, synchronization, timers, memory, interrupt manager, HAL
- Architecture choice depends on: target hardware (MCU vs MPU), safety requirements, latency budget, complexity

---

[Previous Chapter: History and Evolution ←](Chapter_02_History_Evolution.md) | [Next Chapter: Hardware Architecture →](Chapter_04_Hardware.md)
