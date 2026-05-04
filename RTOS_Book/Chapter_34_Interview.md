# Chapter 34: Interview Preparation

## Learning Goals
- Master comprehensive RTOS interview questions across all topics
- Practice system design for real-time embedded systems
- Prepare for whiteboard coding of RTOS primitives
- Know common misconceptions and tricky questions

---

## 1. Fundamentals (Chapters 1-4)

**Q1: What is the difference between hard real-time and soft real-time?**
**A:** Hard real-time: missing a deadline is a system failure. The system is incorrect regardless of the quality of the result. Examples: ABS braking (5ms), pacemaker pulse (precise timing), airbag deployment (<10ms). Soft real-time: missing a deadline degrades quality but isn't catastrophic. The value of the result decreases after the deadline. Examples: video streaming (frame drop), audio playback (buffer underrun). Key distinction: hard RT systems need worst-case analysis (WCET), formal schedulability proof. Soft RT systems use average-case design with statistical guarantees.

**Q2: What makes an RTOS deterministic compared to a GPOS?**
**A:** Determinism means bounded, predictable timing: (1) Every RTOS API call has a documented worst-case execution time (WCET) — `xQueueSend()` takes at most N cycles regardless of system state. (2) Interrupt latency is bounded — Cortex-M: 12 cycles hardware + known software overhead. (3) Scheduler is O(1) — constant time to find next task regardless of task count (bitmap + CLZ). (4) No virtual memory/page faults — flat memory model, no unpredictable disk I/O. (5) Priority-based scheduling — highest priority task always preempts, no fairness delays. A GPOS (Linux) optimizes for throughput and fairness; an RTOS optimizes for worst-case response time.

**Q3: Explain the RTOS architecture differences: monolithic vs microkernel.**
**A:** Monolithic (FreeRTOS, VxWorks): everything in one address space — kernel, drivers, application. Fast (no context switch for system calls), simple, small. Risk: any bug crashes everything. Microkernel (QNX): minimal kernel (IPC, scheduling, memory management only). Drivers and services run as user-space processes. Slower (IPC overhead for every service call) but fault-isolated — crashed driver gets restarted without rebooting. QNX Neutrino microkernel is ~60KB. Choice depends on: size constraints → monolithic; safety/reliability → microkernel.

---

## 2. Task Management and Scheduling (Chapters 5-7)

**Q4: What is stored in a Task Control Block (TCB)?**
**A:** TCB contains all per-task state: (1) `pxTopOfStack` — pointer to saved stack position (MUST be first field for context switch assembly). (2) `xStateListItem` — links task into ready/blocked/suspended list. (3) `xEventListItem` — links task into event waiting list (queue/semaphore). (4) `uxPriority` — task priority (0 = idle, higher = more urgent). (5) `pxStack` — pointer to allocated stack base (for overflow detection). (6) `pcTaskName` — debug identification string. (7) `uxBasePriority` — original priority before inheritance. (8) `uxMutexesHeld` — count of mutexes held (for priority inheritance). The TCB is never directly accessed by application code — accessed via `TaskHandle_t` opaque handle.

**Q5: Explain Rate Monotonic Scheduling and its utilization bound.**
**A:** RMS (Liu & Layland, 1973): assign priorities based on period — shorter period = higher priority. Optimal among fixed-priority algorithms for independent, periodic tasks. Utilization bound: a task set is guaranteed schedulable if total utilization $U = \sum_{i=1}^{n} \frac{C_i}{T_i} \leq n(2^{1/n} - 1)$. For n=1: 100%, n=2: 82.8%, n→∞: ln(2) ≈ 69.3%. Example: Task A (C=1ms, T=4ms, U=25%), Task B (C=2ms, T=6ms, U=33.3%). Total U=58.3% ≤ 82.8% (n=2), so RMS guarantees schedulability. Note: this is a sufficient condition — some task sets with U > bound are still schedulable (requires Response Time Analysis to verify).

**Q6: What is the difference between cooperative and preemptive scheduling?**
**A:** Cooperative: running task keeps CPU until it voluntarily yields (`taskYIELD()`) or blocks (delay, queue wait). Simple, no race conditions with shared data, but one task can starve others. Preemptive: scheduler can forcibly switch tasks based on events — tick interrupt (time-slice), higher priority task unblocked, ISR wakes higher-prio task. More responsive but requires synchronization (mutexes/critical sections) for shared data. FreeRTOS supports all three: cooperative (`configUSE_PREEMPTION=0`), preemptive (`configUSE_PREEMPTION=1, configUSE_TIME_SLICING=0`), preemptive with time-slicing (both = 1, default).

---

## 3. Context Switching and Interrupts (Chapters 8-9)

**Q7: Describe the complete context switch process on Cortex-M4.**
**A:** Context switch happens in PendSV handler (lowest priority exception): (1) Hardware auto-pushes to current task's PSP: xPSR, PC, LR, R12, R3-R0 (8 registers, 32 bytes). If FPU active: also S0-S15, FPSCR (auto by lazy stacking). (2) PendSV ISR (software): save R4-R11 to PSP manually (`STMDB R0!, {R4-R11}`). If FPU context: also S16-S31. (3) Save PSP value to `pxCurrentTCB->pxTopOfStack`. (4) Call `vTaskSwitchContext()` — selects next highest-priority ready task, updates `pxCurrentTCB`. (5) Load new PSP from `pxCurrentTCB->pxTopOfStack`. (6) Pop R4-R11 from new task's stack (`LDMIA R0!, {R4-R11}`). (7) Return via BX LR (EXC_RETURN) — hardware auto-pops R0-R3, R12, LR, PC, xPSR from PSP. New task resumes exactly where it left off. Total cost: ~30-80 cycles depending on FPU state.

**Q8: Why must FreeRTOS ISRs use "FromISR" API variants?**
**A:** FromISR variants differ in three critical ways: (1) They never block — `xQueueSendFromISR()` returns immediately if queue full (errQUEUE_FULL), whereas `xQueueSend()` blocks the calling task. ISRs cannot block because they don't have a task context to suspend. (2) They use a different critical section mechanism — ISRs use interrupt masking (BASEPRI), not the task-level `taskENTER_CRITICAL()` nesting counter. (3) They return `xHigherPriorityTaskWoken` — tells the ISR whether a higher-priority task was unblocked, so it can call `portYIELD_FROM_ISR()` to trigger an immediate context switch via PendSV instead of waiting for the next tick. Using the non-FromISR version in an ISR would attempt to use task-level scheduler operations, causing undefined behavior or hard faults.

---

## 4. IPC and Synchronization (Chapters 10-12)

**Q9: Explain priority inversion and the Mars Pathfinder bug.**
**A:** Priority inversion: high-priority task is blocked waiting for a resource held by a low-priority task, while a medium-priority task runs (preventing the low-priority task from releasing the resource). This unbounded delay violates real-time guarantees. Mars Pathfinder (1997): VxWorks on RAD6000 CPU. High-priority bus management task needed mutex held by low-priority meteorological task. Medium-priority communication task preempted the low-priority task. Result: bus management missed deadline → watchdog timeout → system reset (losing data). Fix: engineers uploaded a patch from Earth enabling VxWorks priority inheritance on that mutex (`SEM_INVERSION_SAFE` flag). With priority inheritance, the low-priority task temporarily inherits the high-priority task's priority, preventing the medium-priority task from preempting it, ensuring the mutex is released quickly.

**Q10: When would you use a queue vs a task notification vs an event group?**
**A:** Queue: use when transferring data between tasks/ISRs. Copy-by-value, FIFO ordering, multiple senders/receivers supported. Overhead: ~100 bytes per queue + item storage. Best for: sensor data pipeline, command processing. Task notification: use for lightweight signaling to a specific task. Acts as binary/counting semaphore or lightweight mailbox. Overhead: zero — uses existing TCB field. 45% faster than semaphore. Limitation: only one receiver (the notified task), notification value is single uint32_t. Best for: ISR-to-task signaling, simple event notification. Event group: use when a task must wait for a combination of conditions (AND/OR of flags). 24 event bits. Can synchronize multiple tasks (rendezvous). Best for: "wait until sensor ready AND calibration done AND network connected."

---

## 5. Memory Management (Chapters 13-14)

**Q11: Compare FreeRTOS heap implementations (heap_1 through heap_5).**
**A:** heap_1: simplest, allocate-only (no free). Deterministic O(1). Use for safety-critical where objects are created once at startup. heap_2: best-fit, free supported, no coalescing. Fast allocation but fragments over time. Use when allocated blocks are always same size. heap_3: wraps standard malloc/free with RTOS mutex. Uses whatever the toolchain provides. Non-deterministic. Only use if you need C library compatibility. heap_4: first-fit with adjacent block coalescing. Best general-purpose allocator. O(n) allocation (n = number of free blocks). Most commonly used. heap_5: extends heap_4 to span non-contiguous memory regions (e.g., internal SRAM + external SDRAM, or SRAM + CCM). Same algorithm as heap_4. For safety-critical (ASIL C/D, SIL 3): use only static allocation (`xTaskCreateStatic`) — no heap at all.

**Q12: What is a lock-free ring buffer and when would you use one in RTOS?**
**A:** A lock-free SPSC (Single Producer, Single Consumer) ring buffer uses separate read and write indices with memory barriers — no mutex needed. Producer writes at `write_idx` and advances it; consumer reads at `read_idx` and advances it. Memory barrier (`__DMB()`) ensures index updates are visible after data writes. Use cases: (1) ISR → task data transfer (ISR can't take mutex). (2) High-frequency ADC sampling where mutex overhead is unacceptable. (3) Inter-core communication in multi-core systems. Key constraint: exactly one writer and one reader — if multiple writers/readers are needed, use a queue with mutex.

---

## 6. System Design Questions

**Q13: Design a motor control system with RTOS.**
**A:**
```
Tasks (by priority, highest first):
1. Safety Monitor (P=5, 1ms): check limits, emergency stop
2. Current Control (P=4, 100us): PI loop for motor current
3. Speed Control (P=3, 1ms): PI/PID loop for speed
4. Position Control (P=2, 5ms): trajectory planning
5. Communication (P=1, 10ms): CAN/Ethernet command interface
6. Diagnostic (P=0, 100ms): logging, temperature monitoring

IPC:
- ISR → Current Control: task notification (ADC complete)
- Speed → Current: queue (setpoint)
- Position → Speed: queue (setpoint)
- Communication → Position: queue (trajectory commands)
- Safety Monitor: checks all shared state, can disable PWM

Memory:
- Static allocation only (safety-critical)
- DMA double-buffer for ADC samples
- Shared setpoints protected by critical sections (short)

Timing verification:
- RMS analysis: verify U ≤ bound for all periodic tasks
- WCET measurement with DWT cycle counter
- SystemView validation of timing in integration testing
```

**Q14: Design an IoT sensor node with FreeRTOS.**
**A:**
```
Tasks:
1. Sensor Read (P=3, 1s): I2C/SPI sensor sampling
2. Data Process (P=2): filtering, threshold detection
3. MQTT Publish (P=1): send data to cloud when ready
4. OTA Check (P=0, 1hr): check for firmware updates

Architecture:
- Sensor → Queue → Process → Queue → MQTT Publish
- Event group: WIFI_CONNECTED | MQTT_CONNECTED | SENSOR_READY
- MQTT task waits for all three flags before publishing

Power management:
- Tickless idle: sleep between sensor readings
- WiFi power save: connect → publish → disconnect
- Target: 1 year on 2xAA batteries
- Wake schedule: sample every 60s, publish every 5min

Security:
- TLS 1.3 for MQTT (mbedTLS, PSK or certificate)
- Secure boot with MCUboot
- Encrypted credentials in secure element (ATECC608)
```

---

## 7. Tricky Questions

**Q15: Can you use malloc() in an RTOS ISR?**
**A:** Absolutely not. malloc() is non-reentrant (uses global heap state), may block (some implementations use locks), and has non-deterministic execution time. In ISR context: (1) Cannot block or be preempted by scheduler. (2) Must complete in bounded time. (3) Cannot use standard library functions that aren't reentrant. Even FreeRTOS `pvPortMalloc()` should not be called from ISR — it uses critical sections that aren't ISR-safe in all ports. Pre-allocate all memory before scheduler starts, or use memory pools (fixed-size block allocation with O(1) time).

**Q16: What happens if a task never blocks or yields in a preemptive RTOS?**
**A:** If the task has the highest priority: it runs forever, starving all lower-priority tasks including idle. The watchdog (if fed only by idle task) will trigger a reset. If the task has a lower priority: it runs when no higher-priority task is ready but prevents all equal/lower-priority tasks from running. SysTick interrupt still fires, so the tick counter advances and higher-priority tasks can preempt it. With time-slicing enabled: tasks at the same priority get round-robin'd at each tick, so other equal-priority tasks still get CPU time.

**Q17: Why is `volatile` important in RTOS shared variables?**
**A:** `volatile` tells the compiler the variable can change outside the current execution context — by another task (after context switch), by an ISR, or by DMA. Without volatile, the compiler may: (1) Cache the variable in a register and never re-read from memory. (2) Reorder reads/writes for optimization. (3) Eliminate "redundant" reads in a loop. Example: `while(flag == 0);` — compiler optimizes to `if(flag == 0) while(1);` because it sees no assignment to `flag` in loop body. With `volatile uint32_t flag;`, compiler generates a memory read on every loop iteration. Note: volatile alone is NOT sufficient for thread-safety — it doesn't provide atomicity or memory ordering. Use mutexes or critical sections for compound operations.

---

## 8. Quick Reference: Key Numbers

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Numbers Every RTOS Developer Should Know:               │
  │                                                           │
  │  Cortex-M4 @ 168MHz:                                    │
  │  • 1 clock cycle = 5.95 ns                              │
  │  • Interrupt latency (hardware): 12 cycles = 71 ns      │
  │  • Context switch: ~30-80 cycles = 180-480 ns           │
  │  • Context switch with FPU: ~80-150 cycles              │
  │  • Cache miss penalty (M7): ~20 cycles                  │
  │  • Flash read (0 wait): 30 MHz max                      │
  │  • Flash read (5 wait): 168 MHz via ART accelerator     │
  │                                                           │
  │  FreeRTOS memory usage:                                  │
  │  • Kernel code: ~5-10 KB Flash                          │
  │  • Per task: TCB (~100 bytes) + stack (128-2048 bytes)  │
  │  • Per queue: ~76 bytes + (item_size * length)          │
  │  • Per timer: ~44 bytes                                 │
  │  • Per event group: ~28 bytes                           │
  │                                                           │
  │  Scheduling:                                             │
  │  • RMS utilization bound (n=3): 78.0%                   │
  │  • RMS utilization bound (n→∞): 69.3%                   │
  │  • EDF utilization bound: 100% (optimal)                │
  │  • Typical tick rate: 1000 Hz (1ms)                     │
  │  • Tick ISR overhead: ~1-5 us                           │
  │                                                           │
  │  Comparison (round numbers):                             │
  │  • RTOS context switch: ~1 us                           │
  │  • Linux context switch: ~5-50 us                       │
  │  • RTOS interrupt latency: < 1 us                       │
  │  • Linux interrupt latency: 10-100 us                   │
  │  • Linux RT_PREEMPT: 20-80 us worst case               │
  │  • Xenomai: 5-15 us worst case                          │
  └──────────────────────────────────────────────────────────┘
```

---

## Summary

- Master fundamentals: hard vs soft RT, determinism, RTOS vs GPOS
- Know scheduling: RMS (U bound = n(2^(1/n)-1)), priority-based preemptive, cooperative
- Context switch: HW saves R0-R3/R12/LR/PC/xPSR, SW saves R4-R11, PendSV at lowest priority
- Synchronization: priority inversion → inheritance/ceiling, Mars Pathfinder case study
- Memory: heap_1-5 trade-offs, static for safety-critical, lock-free for ISR communication
- System design: prioritize by urgency, RMS verify, use queues for data, notifications for signals
- Key numbers: context switch ~1us, IRQ latency ~71ns, FreeRTOS kernel ~6KB flash

---

[Previous Chapter: References ←](Chapter_33_References.md) | [Back to Index →](00_Master_Index.md)
