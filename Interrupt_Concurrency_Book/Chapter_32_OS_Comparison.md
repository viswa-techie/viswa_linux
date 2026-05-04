# Chapter 32: OS Comparison

## Learning Goals
- Compare interrupt handling across Linux, Windows, macOS, QNX, and FreeRTOS
- Understand design trade-offs in each OS's approach
- Know equivalent APIs and concepts across operating systems
- Prepare for cross-platform interview questions

---

## 32.1 Interrupt Handling Comparison

### Interrupt Architecture

```
Feature              │ Linux          │ Windows        │ macOS          │ QNX            │ FreeRTOS
─────────────────────┼────────────────┼────────────────┼────────────────┼────────────────┼──────────
IRQ model            │ Top/bottom     │ ISR + DPC      │ IOKit work     │ Pulse/event    │ ISR + defer
                     │ half           │                │ loop           │ msg passing    │ to task
Handler context      │ Hardirq ctx    │ DIRQL          │ Primary int.   │ Interrupt      │ ISR context
                     │                │                │ filter         │ handler thread │
Deferred work        │ SoftIRQ,       │ DPC (Deferred  │ IOWorkLoop     │ Pulse to       │ Task notify
                     │ Tasklet,       │ Procedure      │ IOCommandGate  │ server thread  │ or deferred
                     │ Workqueue,     │ Call)           │                │                │ interrupt
                     │ Threaded IRQ   │                │                │                │ processing
Threading            │ Optional       │ No (DPC is     │ WorkLoop is    │ Always (msg    │ Deferred to
                     │ (threaded IRQ) │ softirq-like)  │ thread-based   │ passing model) │ task
```

### Detailed Comparison

#### Linux
```
IRQ → Top half (hardirq) → bottom half (softirq/workqueue/threaded IRQ)
  - Most flexible model
  - Choice of deferred mechanisms
  - PREEMPT_RT: all IRQs become threads
  - Open source, highly configurable
```

#### Windows
```
IRQ → ISR at DIRQL → DPC at DISPATCH_LEVEL → Worker thread (passive)

IRQL levels (priority):
  HIGH_LEVEL     ← NMI equivalent
  POWER_LEVEL
  IPI_LEVEL
  CLOCK_LEVEL
  DEVICE_LEVEL   ← Device ISRs (DIRQL)
  DISPATCH_LEVEL ← DPCs, scheduler
  APC_LEVEL      ← Async procedure calls
  PASSIVE_LEVEL  ← Normal thread execution

ISR: Minimal work at DIRQL, schedule DPC
DPC: Deferred Procedure Call (like Linux softirq)
  - Runs at DISPATCH_LEVEL
  - Cannot page fault
  - Serialized per-CPU
Worker thread: For work that needs PASSIVE_LEVEL (can page, sleep)
```

#### macOS (XNU / IOKit)
```
Primary interrupt filter → IOWorkLoop → IOCommandGate

IOKit driver model:
  - Primary interrupt: fast, minimal (like Linux top half)
  - IOFilterInterruptEventSource: check if interrupt is ours
  - IOWorkLoop: single-threaded work loop (like dedicated kthread)
  - IOCommandGate: serialized access to driver state
  - No manual lock management — IOWorkLoop serializes everything
```

#### QNX (Microkernel RTOS)
```
IRQ → Interrupt handler → Pulse → Server thread

Microkernel model:
  1. Hardware IRQ → minimal kernel handler
  2. Handler calls InterruptAttachEvent() → generates pulse/event
  3. Pulse delivered to user-space driver thread via message passing
  4. Driver thread handles interrupt in full POSIX context (can sleep!)
  
Advantages:
  - Drivers run in user space (crash doesn't kill kernel)
  - Full POSIX API available in handler
  - Deterministic: priority-based preemptive scheduling
  
API:
  InterruptAttach()      ← Register ISR (runs in kernel)
  InterruptAttachEvent() ← Register event (deferred to thread)
  InterruptWait()        ← Block until interrupt
  InterruptMask/Unmask() ← Control IRQ
```

#### FreeRTOS
```
IRQ → ISR (hardware) → Task notification/queue → Task

Simple model:
  1. ISR runs in hardware interrupt context
  2. ISR sends notification or queue item to a task
  3. Task wakes up and processes (portYIELD_FROM_ISR)
  
API:
  xTaskNotifyFromISR()           ← Notify task from ISR
  xQueueSendFromISR()            ← Send to queue from ISR
  xSemaphoreGiveFromISR()        ← Give semaphore from ISR
  portYIELD_FROM_ISR(xHigherPriorityTaskWoken)
  
Rules:
  - ISR must use *FromISR() variants
  - Cannot call blocking APIs from ISR
  - configMAX_SYSCALL_INTERRUPT_PRIORITY
```

---

## 32.2 Synchronization Primitives Comparison

```
Concept           │ Linux            │ Windows          │ QNX             │ FreeRTOS
──────────────────┼──────────────────┼──────────────────┼─────────────────┼──────────
Spinlock          │ spin_lock()      │ KeAcquireSpin    │ SpinLock (SMP)  │ taskENTER
                  │ raw_spin_lock()  │ Lock()           │                 │ _CRITICAL()
Mutex             │ mutex_lock()     │ ExAcquireFast    │ pthread_mutex   │ xSemaphore
                  │                  │ Mutex()          │ _lock()         │ CreateMutex()
Semaphore         │ down()           │ KeWaitFor        │ sem_wait()      │ xSemaphore
                  │                  │ SingleObject()   │ (POSIX)         │ CreateCounting()
RW lock           │ rw_semaphore     │ ExAcquire        │ pthread_rwlock  │ N/A
                  │ (sleeping)       │ ResourceShared   │ _rdlock()       │
                  │ rwlock_t (spin)  │ Lite()           │                 │
RCU               │ rcu_read_lock()  │ N/A (some        │ N/A             │ N/A
                  │                  │  lock-free data  │                 │
                  │                  │  structures)     │                 │
Atomic ops        │ atomic_t         │ Interlocked*()   │ atomic_*()      │ Atomic ops
                  │ atomic_inc()     │ InterlockedInc   │ (C11 atomics)   │ (port-specific)
                  │                  │ rement()         │                 │
Completion/Event  │ completion      │ KEVENT           │ pthread_cond    │ xTaskNotify
                  │ complete()       │ KeSetEvent()     │ _signal()       │ Wait()
Memory barrier    │ smp_mb()         │ MemoryBarrier()  │ __sync_synch    │ (compiler
                  │ smp_wmb()        │ KeMemoryBarrier()│ ronize()        │  barriers)
IRQ disable       │ local_irq_save() │ KeRaiseIrql()    │ InterruptMask() │ portDISABLE
                  │                  │ → DISPATCH_LEVEL │                 │ _INTERRUPTS()
```

---

## 32.3 Deferred Work Comparison

```
Linux                │ Windows          │ macOS            │ QNX
─────────────────────┼──────────────────┼──────────────────┼──────────────
SoftIRQ              │ DPC              │ N/A              │ N/A
(10 types, per-CPU)  │ (DISPATCH_LEVEL) │                  │
                     │                  │                  │
Tasklet              │ DPC              │ N/A              │ N/A
(softirq-based)      │                  │                  │
                     │                  │                  │
Workqueue            │ System thread    │ IOWorkLoop       │ Thread pool
(kworker threads)    │ pool             │                  │ (pthread)
                     │ IoQueueWorkItem()│                  │
                     │                  │                  │
Threaded IRQ         │ IoConnectInter   │ IOInterrupt      │ InterruptAttach
(kernel thread)      │ ruptEx()         │ EventSource      │ Event() + thread
                     │ + worker thread  │ + IOWorkLoop     │
```

---

## 32.4 Real-Time Capabilities

```
Feature                │ Linux           │ Linux RT        │ QNX             │ FreeRTOS
───────────────────────┼─────────────────┼─────────────────┼─────────────────┼──────────
Scheduling             │ CFS + RT class  │ SCHED_FIFO/RR   │ Priority-based  │ Priority-
                       │                 │ (preemptive)    │ preemptive      │ based
Worst-case latency     │ ~100µs-1ms      │ ~5-50µs         │ ~1-10µs         │ ~1-5µs
Priority inversion     │ rt_mutex (PI)   │ rt_mutex (PI)   │ PI built-in     │ Priority
protection             │ for mutexes     │ for all locks   │ for all locks   │ inheritance
IRQ determinism        │ Variable        │ Bounded (forced  │ Very bounded    │ Bounded
                       │                 │  threading)     │ (user-space)    │ (minimal ISR)
Kernel preemption      │ CONFIG_PREEMPT  │ Full preempt    │ Fully preempt   │ Full preempt
                       │ (voluntary to   │ (everything is  │ (microkernel)   │ (cooperative
                       │  full to RT)    │ preemptible)    │                 │  optional)
Certification          │ No formal cert  │ No formal cert  │ IEC 61508       │ IEC 61508
                       │                 │                 │ ISO 26262       │ (SafeRTOS)
                       │                 │                 │ DO-178C         │
```

---

## 32.5 Design Philosophy Comparison

```
Linux:
  - Monolithic kernel with modules
  - Interrupt handling optimized for throughput
  - Most flexible: many mechanisms to choose from
  - PREEMPT_RT add-on for real-time
  - Community-driven, open source

Windows:
  - Hybrid kernel (microkernel heritage, monolithic practice)
  - IRQL-based priority system (clear hierarchy)
  - DPC mechanism for deferred work
  - Less choice but well-documented patterns
  - Proprietary, certified (automotive with QNX-like additions)

macOS (XNU):
  - Mach microkernel + BSD layer
  - IOKit framework abstracts interrupt handling
  - Developer-friendly but less control
  - Single-threaded work loop per driver (simple)

QNX:
  - True microkernel — drivers in user space
  - Message passing for everything
  - Best determinism and fault isolation
  - Industry standard for safety-critical (automotive, medical)

FreeRTOS:
  - Minimal RTOS for microcontrollers
  - Simple ISR + task model
  - Tiny footprint (6-12KB ROM)
  - No MMU, no virtual memory
  - SafeRTOS variant for certification
```

---

## 32.6 Equivalent APIs Quick Reference

```
Task                        │ Linux                │ Windows              │ QNX
────────────────────────────┼──────────────────────┼──────────────────────┼──────────────
Register IRQ handler        │ request_irq()        │ IoConnectInterrupt() │ InterruptAttach()
Free IRQ                    │ free_irq()           │ IoDisconnectIntr()   │ InterruptDetach()
Disable IRQ (local)         │ local_irq_disable()  │ KeRaiseIrql()        │ InterruptDisable()
Enable IRQ (local)          │ local_irq_enable()   │ KeLowerIrql()        │ InterruptEnable()
Schedule deferred work      │ schedule_work()      │ KeInsertQueueDpc()   │ MsgSendPulse()
Spinlock acquire            │ spin_lock()          │ KeAcquireSpinLock()  │ SpinLock()
Mutex acquire               │ mutex_lock()         │ ExAcquireFastMutex() │ pthread_mutex_lock()
Sleep/wait                  │ wait_event()         │ KeWaitForSingle()    │ pthread_cond_wait()
Atomic increment            │ atomic_inc()         │ InterlockedIncrement │ atomic_add_value()
Timer setup                 │ timer_setup()        │ KeInitializeTimer()  │ timer_create()
```

---

## Interview Questions

1. **Compare Linux interrupt handling with Windows ISR/DPC model.**
2. **How does QNX handle interrupts differently from Linux? Why?**
3. **What is the Windows IRQL system? Map it to Linux contexts.**
4. **Compare Linux workqueue with Windows system thread pool.**
5. **Why doesn't FreeRTOS need softirqs or workqueues?**
6. **What is IOWorkLoop in macOS? How does it compare to Linux threaded IRQ?**
7. **Which OS provides the best real-time interrupt determinism? Why?**
8. **Compare RCU (Linux) with any equivalent in Windows or QNX.**
9. **How does QNX's microkernel design improve fault isolation for drivers?**
10. **If porting a Linux driver to QNX, what changes in interrupt handling?**

---

## Summary

- Linux: most flexible (softirq/tasklet/workqueue/threaded IRQ), throughput-optimized
- Windows: structured IRQL hierarchy, DPC for deferred work, well-documented
- macOS: IOKit work loop abstracts everything, developer-friendly
- QNX: microkernel, drivers in user space, message-passing, best for safety-critical
- FreeRTOS: minimal ISR + task model, microcontroller-focused, tiny footprint
- Real-time: QNX > FreeRTOS > Linux PREEMPT_RT > Standard Linux > Windows/macOS
- Each OS makes different trade-offs between flexibility, safety, latency, and throughput

---

*Next: [Chapter 33 — Real-Time Linux (PREEMPT_RT)](Chapter_33_Real_Time_Linux.md)*
