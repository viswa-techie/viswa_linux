# Chapter 32: OS Comparison — Scheduling Across Operating Systems

## Learning Goals
- Compare Linux scheduling with Windows, macOS, QNX, FreeRTOS, and Zephyr
- Understand different design philosophies and trade-offs
- Know how real-time OSes differ from general-purpose kernels

---

## 32.1 Scheduler Comparison Table

```
┌────────────┬──────────────┬──────────────┬─────────────┬────────────┬────────────┐
│ Feature     │ Linux         │ Windows       │ macOS/XNU    │ QNX         │ FreeRTOS    │
├────────────┼──────────────┼──────────────┼─────────────┼────────────┼────────────┤
│ Scheduler   │ CFS (fair)   │ MLFQ         │ Mach decay  │ Partitioned│ Priority   │
│ algorithm   │ + RT classes │ + MMCSS      │ + priority  │ fixed-prio │ preemptive │
│             │              │              │ + QoS bands │            │            │
├────────────┼──────────────┼──────────────┼─────────────┼────────────┼────────────┤
│ Normal      │ Proportional │ Dynamic      │ Mach time-  │ Fixed      │ N/A        │
│ tasks       │ fair sharing │ priority     │ decay bands │ priority   │ (all tasks │
│             │ (vruntime)   │ (boost/decay)│ (QoS classes│ FIFO/RR    │ are "RT")  │
│             │              │              │  map to Mach│            │            │
│             │              │              │  priorities)│            │            │
├────────────┼──────────────┼──────────────┼─────────────┼────────────┼────────────┤
│ RT support  │ SCHED_FIFO   │ Real-time    │ Real-time   │ All tasks  │ All tasks  │
│             │ SCHED_RR     │ priority     │ threads     │ are RT     │ are RT     │
│             │ SCHED_DL     │ class        │ (prio 80-97)│ by design  │ by design  │
├────────────┼──────────────┼──────────────┼─────────────┼────────────┼────────────┤
│ Priority    │ 0-139        │ 0-31         │ 0-127       │ 0-255      │ 0-N config │
│ levels      │ (lower=higher│ (higher=     │ (Mach prio) │ (0=highest │ (0=highest │
│             │  priority)   │  higher prio)│             │  in usage) │  often)    │
├────────────┼──────────────┼──────────────┼─────────────┼────────────┼────────────┤
│ Timeslice   │ Dynamic      │ 20ms default │ Mach quantum│ No defaults│ Tick-based │
│             │ (calculated  │ (varies by   │ (10-200ms   │ (configura-│ or tickless│
│             │  from weight)│  class)      │  by band)   │  ble)      │            │
├────────────┼──────────────┼──────────────┼─────────────┼────────────┼────────────┤
│ Preemption  │ Configurable │ Full kernel  │ Full kernel │ Full       │ Full       │
│             │ (NONE/VOL/   │ preemption   │ preemption  │ preemption │ preemption │
│             │  FULL/RT)    │              │             │ always     │ always     │
├────────────┼──────────────┼──────────────┼─────────────┼────────────┼────────────┤
│ Hard RT     │ PREEMPT_RT   │ No           │ No          │ YES (POSIX │ YES        │
│ capable?    │ (patchset)   │              │             │  certified)│ (by design)│
├────────────┼──────────────┼──────────────┼─────────────┼────────────┼────────────┤
│ SMP support │ Yes (complex │ Yes          │ Yes         │ Yes (BMP   │ Limited    │
│             │  domains)    │              │             │  + SMP)    │ (SMP in    │
│             │              │              │             │            │  newer)    │
├────────────┼──────────────┼──────────────┼─────────────┼────────────┼────────────┤
│ Worst-case  │ ~1ms (stock) │ ~5-15ms      │ ~5-10ms     │ ~5-50µs   │ ~1-10µs   │
│ latency     │ ~50µs (RT)   │              │             │            │            │
└────────────┴──────────────┴──────────────┴─────────────┴────────────┴────────────┘
```

---

## 32.2 Windows Scheduling

```
Windows uses a Multilevel Feedback Queue with 32 priority levels:

  Priority classes (user-visible):
    Real-time:    24-31 (only admin)
    High:         13
    Above Normal: 10
    Normal:       8
    Below Normal: 6
    Idle:         4

  Thread priorities within class:
    Time-critical, Highest, Above-normal, Normal,
    Below-normal, Lowest, Idle

  Dynamic boosting:
    - I/O completion → temporary boost
    - Foreground window → +2 quantum priority
    - GUI thread → quantum doubling
    - Priority decay: boost decreases each quantum

  MMCSS (Multimedia Class Scheduler Service):
    Guarantees 80% CPU to multimedia threads
    Used by audio/video applications
    Similar concept to Linux SCHED_FIFO but managed by service

Key differences from Linux:
  ✓ Windows has NO equivalent of CFS's vruntime fairness
  ✓ Windows uses heuristic boosting (more like Linux O(1) scheduler)
  ✓ Windows foreground priority boosting = hardcoded
  ✓ No equivalent of SCHED_DEADLINE (no EDF)
```

---

## 32.3 macOS / XNU Scheduling

```
XNU = hybrid kernel (Mach microkernel + BSD)

  Mach scheduling:
    128 priority levels (0-127)
    4 scheduling bands/QoS levels (mapped to Mach priorities):
    
    QoS Class          │ Mach Priority │ Use
    ──────────────────────────────────────────
    User Interactive    │ 47            │ UI thread
    User Initiated      │ 37            │ User-triggered work
    Default/Utility     │ 31/20         │ General/long-running
    Background          │ 4             │ Non-urgent work
    Maintenance         │ 4             │ System maintenance

  Time decay scheduling:
    Mach uses "time-decay" for normal threads:
      Active threads' priority decays over time
      Sleeping threads' priority rises
      Similar effect to CFS vruntime

  Grand Central Dispatch (GCD):
    User-space thread pool managed by kernel
    QoS classes propagate through the system
    Reduces thread creation overhead

  Real-time threads:
    Priority 80-97 (real-time band)
    No timeslice preemption
    Similar to Linux SCHED_FIFO

Key differences from Linux:
  ✓ QoS-based scheduling (high level, propagated automatically)
  ✓ Mach IPC + thread migration (Mach-specific)
  ✓ GCD replaces explicit threading in many cases
  ✓ No SCHED_DEADLINE equivalent
  ✓ Binary-only kernel (no tuning like Linux sched parameters)
```

---

## 32.4 QNX — Hard Real-Time Microkernel

```
QNX is a POSIX-certified hard real-time microkernel OS:

  Architecture:
    Microkernel handles: scheduling, IPC, interrupt dispatching
    Everything else: drivers, filesystems, network → user-space servers
    
  Scheduling:
    256 priority levels (0-255)
    Policies: SCHED_FIFO, SCHED_RR, SCHED_SPORADIC
    ALL tasks are effectively "real-time" — deterministic behavior

  SCHED_SPORADIC (unique to QNX):
    Budget-based: task gets 'C' microseconds every 'T' microseconds
    Prevents RT tasks from monopolizing CPU
    Similar concept to Linux SCHED_DEADLINE but older

  Key guarantees:
    Context switch: ~1-5µs
    Interrupt latency: ~1-5µs
    Deterministic: bounded worst-case latency

  Why automotive/medical uses QNX:
    - POSIX certified (code portability)
    - Safety certified (IEC 61508, ISO 26262)
    - Deterministic scheduling (guaranteed latency bounds)
    - Microkernel: driver crash doesn't affect kernel

Comparison with Linux:
  QNX advantage: Deterministic, certified, microkernel isolation
  Linux advantage: Wider hardware support, CFS fairness, community
  Linux PREEMPT_RT: Approaches QNX latency but no safety certification
```

---

## 32.5 FreeRTOS — Embedded RTOS

```
FreeRTOS is a lightweight RTOS for microcontrollers:

  Target: ARM Cortex-M, RISC-V, small processors
  Memory: ~6-12 KB ROM, 0.5-2 KB RAM for kernel

  Scheduling:
    - Fixed-priority preemptive (default)
    - Round-robin at same priority (configurable)
    - Cooperative scheduling mode (optional)
    - configMAX_PRIORITIES: typically 5-56 levels

  No dynamic priority adjustment (no CFS, no boost)
  No load balancing (usually single-core)
  No virtual memory (no address spaces, no COW)

  Task states:
    Running → Ready → Blocked → Suspended
    Simpler than Linux's 7+ states

  IPC:
    Queues (message passing)
    Semaphores (binary, counting)
    Mutexes (with priority inheritance!)
    Event groups (bit flags)
    Stream/Message buffers

Key differences from Linux:
  ✓ Much simpler (~9000 lines of core code vs millions)
  ✓ No memory protection (tasks share address space)
  ✓ Static allocation common (no malloc in critical paths)
  ✓ Deterministic timing (simple scheduler, no complex heuristics)
  ✓ No filesystem, networking built-in (add-on modules)
```

---

## 32.6 Zephyr RTOS

```
Zephyr is a modern open-source RTOS (Linux Foundation):

  Scheduling:
    - Multi-queue priority-based preemptive
    - Cooperative threads (don't yield until explicitly)
    - Preemptive threads (preempted by higher priority)
    - EDF scheduling support (like Linux SCHED_DEADLINE)
    - Meta-IRQ priority class (like Linux stop class)

  Unique features:
    - Unified kernel (not micro/mono distinction)
    - Thread-level memory protection (ARM MPU)
    - Native POSIX compatibility layer
    - Built-in Bluetooth, networking, USB stacks

  Priority levels: configurable (typically 15-40)
  Interrupt latency: ~1-5µs on ARM Cortex-M

Comparison with Linux:
  Zephyr: Small footprint (8KB+), deterministic, IoT-focused
  Linux: Full-featured, scalable, rich ecosystem
  Zephyr is to FreeRTOS what Linux is to Unix — modern reimagining
```

---

## 32.7 Scheduling Philosophy Comparison

```
                        Fairness ──────────── Determinism
                            │                      │
                   Linux CFS │                      │ QNX
                            │                      │ FreeRTOS
                            │                      │ Zephyr
                   Windows  │                      │
                   macOS    │                      │
                            │                      │
                 Linux RT ──┼──────────────────────│
                 (PREEMPT_RT)     middle ground     │

  General-purpose OS (Linux, Windows, macOS):
    Optimize for THROUGHPUT and FAIRNESS
    Best-effort latency (usually good enough)
    Dynamic priority adjustments
    
  Hard RTOS (QNX, FreeRTOS, Zephyr):
    Optimize for DETERMINISM and bounded LATENCY
    Guaranteed worst-case response times
    Fixed priority, minimal overhead
    
  Linux PREEMPT_RT:
    Bridges the gap — general-purpose + deterministic
    Not certified for safety-critical (yet)
    Close to RTOS latency with Linux ecosystem
```

---

## 32.8 Process/Task Model Comparison

```
┌──────────────┬────────────┬────────────┬──────────┬────────────┐
│ Feature       │ Linux       │ Windows     │ QNX       │ FreeRTOS    │
├──────────────┼────────────┼────────────┼──────────┼────────────┤
│ Process model│ fork/exec  │ CreateProc │ posix_    │ N/A (tasks │
│              │ clone()    │ ess()      │ spawn()  │ only)      │
├──────────────┼────────────┼────────────┼──────────┼────────────┤
│ Threads      │ 1:1 (NPTL)│ 1:1        │ 1:1      │ Tasks ARE  │
│              │ clone+flags│            │ POSIX    │ threads    │
├──────────────┼────────────┼────────────┼──────────┼────────────┤
│ Address space│ Per-process│ Per-process│Per-process│ Shared     │
│              │ (COW fork) │            │           │ (flat)     │
├──────────────┼────────────┼────────────┼──────────┼────────────┤
│ IPC          │ pipe/shm/  │ Named pipe│ Message  │ Queue/Sema │
│              │ socket/mq │ mailslot  │ passing  │ /Mutex     │
│              │ binder(And)│ COM/RPC   │ (kernel) │            │
├──────────────┼────────────┼────────────┼──────────┼────────────┤
│ Signals      │ POSIX full │ Limited   │ POSIX    │ Event      │
│              │ (1-64)    │ (SEH, APC)│ signals  │ groups     │
└──────────────┴────────────┴────────────┴──────────┴────────────┘
```

---

## Interview Questions

**Q1: Why might an automotive company choose QNX over Linux for their instrument cluster?**
A: Safety certification (ISO 26262 ASIL-D), deterministic scheduling (guaranteed <10µs latency), microkernel isolation (driver crash doesn't affect kernel), and POSIX compliance for portability. Linux with PREEMPT_RT approaches the latency but lacks safety certification. However, Android Automotive (Linux-based) is gaining for infotainment while QNX handles safety-critical displays.

**Q2: How does Windows' scheduling compare to Linux CFS for a desktop workload?**
A: Windows uses MLFQ with heuristic priority boosting — foreground windows get quantum extension and priority boost, I/O completion triggers temporary priority increase. This gives "snappy" interactive feel. Linux CFS achieves similar results differently: sleeping tasks accumulate "credit" (lower vruntime), so they run sooner on wakeup. CFS is mathematically fair; Windows relies on heuristics. For desktops, both work well, but CFS handles edge cases (many equal-priority tasks) more predictably.

**Q3: Could Linux ever replace QNX in safety-critical automotive systems?**
A: Currently no, because Linux lacks formal safety certification (ISO 26262). PREEMPT_RT improves determinism but the huge kernel codebase makes formal verification impractical. Approaches being explored: ELISA project (Linux safety qualification), hypervisor-based partitioning (Linux for non-critical + QNX/RTOS for critical), and automotive Linux distributions with restricted configurations. The gap is narrowing but formal certification remains a significant barrier.

---

*Next: [Chapter 33 — Embedded System Scheduling](Chapter_33_Embedded_Scheduling.md)*
