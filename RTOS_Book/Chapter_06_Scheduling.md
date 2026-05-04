# Chapter 6: Real-Time Scheduling

## Learning Goals
- Understand preemptive vs cooperative scheduling
- Learn fixed-priority preemptive scheduling mechanics
- Know round-robin and time-slicing implementation
- Understand ready queue management and O(1) scheduler design
- Learn how different RTOS implement their schedulers
- Grasp the concept of idle task and its role

---

## 1. Scheduling Fundamentals

```
  Scheduling Models Comparison
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  COOPERATIVE (non-preemptive):                           │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ TaskA runs │ TaskA yields │ TaskB runs │ yields│      │
  │  │ ▓▓▓▓▓▓▓▓▓▓▓│             │▓▓▓▓▓▓▓▓▓▓▓│       │      │
  │  │ · Task runs until it voluntarily yields         │      │
  │  │ · Simple, no race conditions                    │      │
  │  │ · One task can starve others                    │      │
  │  │ · NOT suitable for hard real-time               │      │
  │  │ · Used in: Arduino loop(), simple event loops  │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  PREEMPTIVE (priority-based):                            │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ Lo │ interrupt │ Hi runs │ Hi done │ Lo resumes│      │
  │  │ ▓▓▓│          │▓▓▓▓▓▓▓▓▓│         │▓▓▓▓▓▓▓▓▓▓│      │
  │  │ · Higher-priority task preempts lower           │      │
  │  │ · Deterministic response to events              │      │
  │  │ · Requires careful synchronization              │      │
  │  │ · Standard for all production RTOS              │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  PREEMPTIVE + TIME-SLICING:                              │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ Same priority tasks: A │ B │ A │ B │ A │ B    │      │
  │  │ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓   │      │
  │  │ · Equal-priority tasks get time slices          │      │
  │  │ · Higher priority always preempts immediately   │      │
  │  │ · Fairness among same-priority tasks            │      │
  │  │ · FreeRTOS: configUSE_TIME_SLICING = 1          │      │
  │  └────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Fixed-Priority Preemptive Scheduling

```
  Preemptive Scheduling Timeline
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Priority 3 (High):    ░░░░░░░▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░  │
  │  Priority 2 (Med):     ░░▓▓▓▓▓░░░░░░░░▓▓▓▓░░░░░░░░░░  │
  │  Priority 1 (Low):     ▓▓░░░░░░░░░░░░░░░░░▓▓▓▓▓▓▓▓▓▓  │
  │  Idle (0):             ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │
  │                                                           │
  │  Time ──────────────────────────────────────────────►     │
  │  t0   t1    t2        t3    t4                           │
  │                                                           │
  │  t0: Low task running (only ready task)                  │
  │  t1: Med task becomes ready → preempts Low               │
  │  t2: High task becomes ready → preempts Med              │
  │  t3: High task blocks/completes → Med resumes            │
  │  t4: Med task blocks/completes → Low resumes             │
  │                                                           │
  │  Rules:                                                  │
  │  1. Highest-priority READY task always runs              │
  │  2. Preemption is immediate upon priority change         │
  │  3. When high-priority task blocks, next highest runs    │
  │  4. Same-priority: FIFO or round-robin (configurable)   │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Scheduler Implementation

```c
/* Simplified FreeRTOS scheduler core (from tasks.c) */

/* Ready lists: one per priority level */
static List_t pxReadyTasksLists[configMAX_PRIORITIES];

/* Points to currently running task's TCB */
TCB_t * volatile pxCurrentTCB = NULL;

/* Track highest ready priority for O(1) lookup */
static volatile UBaseType_t uxTopReadyPriority = 0;

/*
 * taskSELECT_HIGHEST_PRIORITY_TASK():
 * Find highest non-empty ready list and select next task
 */

/* Method 1: Generic (portable) — scan from top */
#define taskSELECT_HIGHEST_PRIORITY_TASK()                  \
{                                                            \
    UBaseType_t uxTopPriority = uxTopReadyPriority;          \
    while (listLIST_IS_EMPTY(&pxReadyTasksLists[uxTopPriority])) \
    {                                                        \
        --uxTopPriority;    /* Scan downward */               \
    }                                                        \
    listGET_OWNER_OF_NEXT_ENTRY(pxCurrentTCB,               \
        &pxReadyTasksLists[uxTopPriority]);                  \
    uxTopReadyPriority = uxTopPriority;                      \
}

/* Method 2: Architecture-optimized — O(1) with CLZ */
/* ARM Cortex-M CLZ (Count Leading Zeros) instruction */
#define taskSELECT_HIGHEST_PRIORITY_TASK()                  \
{                                                            \
    UBaseType_t uxTopPriority;                               \
    /* uxTopReadyPriority is a bitmap: bit N set = priority N has tasks */ \
    portGET_HIGHEST_PRIORITY(uxTopPriority, uxTopReadyPriority); \
    /* Uses: 31 - __clz(bitmap)  → O(1) single instruction */ \
    listGET_OWNER_OF_NEXT_ENTRY(pxCurrentTCB,               \
        &pxReadyTasksLists[uxTopPriority]);                  \
}

/*
 * How the ready bitmap works:
 *
 * uxTopReadyPriority (32-bit bitmap):
 *
 *   Bit:  31 30 29 ... 5 4 3 2 1 0
 *          0  0  0      0 1 0 1 0 1
 *                         ▲   ▲   ▲
 *                         │   │   └─ Priority 0 has ready tasks
 *                         │   └───── Priority 2 has ready tasks
 *                         └───────── Priority 4 has ready tasks
 *
 *   CLZ(0b...10100) = 27 → highest priority = 31 - 27 = 4
 *   Single instruction on ARM! O(1) regardless of number of priorities
 */
```

---

## 4. Context Switch Trigger Points

```
  When Does a Context Switch Occur?
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  1. TICK INTERRUPT (periodic):                           │
  │     SysTick_Handler → xTaskIncrementTick()               │
  │     - Check if higher-priority task became ready          │
  │     - Check if time-slice expired for current priority   │
  │     - Set PendSV if switch needed                        │
  │                                                           │
  │  2. TASK BLOCKS (voluntary):                             │
  │     xQueueReceive(), xSemaphoreTake(), vTaskDelay()      │
  │     - Current task moves to blocked list                 │
  │     - Scheduler selects next highest-priority ready task │
  │                                                           │
  │  3. ISR MAKES TASK READY (event-driven):                 │
  │     xSemaphoreGiveFromISR(), xQueueSendFromISR()         │
  │     - If unblocked task priority > current task          │
  │     - Sets xHigherPriorityTaskWoken = pdTRUE             │
  │     - portYIELD_FROM_ISR() sets PendSV                   │
  │                                                           │
  │  4. PRIORITY CHANGE:                                     │
  │     vTaskPrioritySet() raises another task's priority    │
  │     - If new priority > current task → immediate switch  │
  │                                                           │
  │  5. TASK RESUME:                                         │
  │     vTaskResume() resumes suspended task                  │
  │     - If resumed task priority > current → switch        │
  │                                                           │
  │  All paths converge to PendSV handler for actual switch: │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ SCB->ICSR |= SCB_ICSR_PENDSVSET_Msk;          │      │
  │  │   (set PendSV pending → fires at lowest prio)  │      │
  │  └────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Idle Task

```c
/*
 * FreeRTOS Idle Task — always exists, runs at priority 0
 * Created automatically by vTaskStartScheduler()
 */

/* Default idle task behavior: */
static void prvIdleTask(void *pvParameters) {
    for (;;) {
        /* 1. Clean up deleted tasks (free TCB/stack memory) */
        prvCheckTasksWaitingTermination();

        /* 2. Yield if other priority-0 tasks exist */
        #if (configIDLE_SHOULD_YIELD == 1)
            if (listCURRENT_LIST_LENGTH(&pxReadyTasksLists[0]) > 1) {
                taskYIELD();
            }
        #endif

        /* 3. Call user idle hook (if configured) */
        #if (configUSE_IDLE_HOOK == 1)
            vApplicationIdleHook();  /* User-defined */
        #endif

        /* 4. Tickless idle: enter low-power mode */
        #if (configUSE_TICKLESS_IDLE != 0)
            /* Calculate expected idle time */
            xExpectedIdleTime = prvGetExpectedIdleTime();
            if (xExpectedIdleTime >= configEXPECTED_IDLE_TIME_BEFORE_SLEEP) {
                /* Suppress tick, enter WFI/WFE sleep */
                portSUPPRESS_TICKS_AND_SLEEP(xExpectedIdleTime);
                /* On wake: compensate tick count for sleep duration */
            }
        #endif
    }
}

/*
 * Tickless Idle Power Savings:
 *
 * Normal mode:    tick tick tick tick tick tick tick
 *                 ▓░▓░▓░▓░▓░▓░▓░  (wake every 1ms)
 *
 * Tickless idle:  tick                    tick
 *                 ▓░░░░░░░░░░░░░░░░░░░░░░▓  (sleep 50ms)
 *                 Power consumption: 10-100x lower during idle
 */
```

---

## 6. FreeRTOS Scheduler Configuration

```c
/* FreeRTOSConfig.h — key scheduling parameters */

/* Preemptive (1) or Cooperative (0) scheduling */
#define configUSE_PREEMPTION                1

/* Enable time-slicing for equal-priority tasks */
#define configUSE_TIME_SLICING              1

/* Number of priority levels (more = more RAM for ready lists) */
#define configMAX_PRIORITIES                32

/* Tick rate: determines scheduling granularity */
#define configTICK_RATE_HZ                  1000  /* 1ms tick */

/* Use optimized O(1) priority selection (ARM CLZ) */
#define configUSE_PORT_OPTIMISED_TASK_SELECTION  1

/* Tickless idle for power savings */
#define configUSE_TICKLESS_IDLE             1
#define configEXPECTED_IDLE_TIME_BEFORE_SLEEP  2  /* Min ticks before sleep */

/* Idle task should yield to other priority-0 tasks */
#define configIDLE_SHOULD_YIELD             1

/*
 * Scheduling behavior matrix:
 *
 * configUSE_PREEMPTION | configUSE_TIME_SLICING | Behavior
 * ─────────────────────┼────────────────────────┼──────────────────
 *          1           │           1            │ Preemptive + RR
 *          1           │           0            │ Preemptive, no RR
 *          0           │           X            │ Cooperative only
 *
 * Preemptive + RR: Most common. Higher prio preempts,
 *   same-prio tasks get equal time slices.
 * Preemptive no RR: Higher prio preempts,
 *   same-prio tasks run until they block (no forced rotation).
 * Cooperative: Task runs until it calls taskYIELD() or blocks.
 */
```

---

## 7. Scheduler Comparison Across RTOS

| Feature | FreeRTOS | Zephyr | QNX Neutrino | VxWorks |
|---------|----------|--------|-------------|---------|
| **Default policy** | Fixed priority preemptive | Priority preemptive | POSIX (FIFO/RR/Sporadic) | Priority preemptive |
| **Priority count** | Configurable (typ 32) | Configurable (typ 32) | 64 (0=highest) | 256 (0=highest) |
| **O(1) selection** | Yes (CLZ bitmap) | Yes (bitmap) | Yes | Yes |
| **Time slicing** | Configurable | Configurable | SCHED_RR | Configurable |
| **Dynamic priority** | Manual only | Manual only | Priority inheritance, sporadic | windPrioritySet |
| **Multi-core** | SMP (v11+) | SMP | SMP/BMP | SMP/AMP |
| **Tickless** | Configurable | Yes | Yes | Yes (since 7.x) |
| **POSIX** | Only with wrapper | Partial support | Full POSIX | Full POSIX |

---

## Interview Questions

**Q1: What are the advantages and disadvantages of preemptive vs cooperative scheduling?**
**A:** Preemptive: + deterministic response to high-priority events, + no task can monopolize CPU, + essential for hard real-time. − requires synchronization (mutexes/semaphores), − context switch overhead, − priority inversion risk, − harder to debug. Cooperative: + no race conditions (task controls when it yields), + simpler code, + lower overhead (no forced context switches), + no synchronization needed for shared data. − one task can starve everything, − response time depends on when others yield, − not suitable for hard real-time.

**Q2: How does FreeRTOS achieve O(1) task selection on ARM?**
**A:** FreeRTOS maintains a bitmap where bit N is set when priority N has at least one ready task. To find the highest priority, it uses the ARM CLZ (Count Leading Zeros) instruction: `highest = 31 - CLZ(bitmap)`. This is a single hardware instruction that executes in 1 cycle. It then indexes into `pxReadyTasksLists[highest]` and gets the next task via `listGET_OWNER_OF_NEXT_ENTRY` (round-robin among same-priority). Total: O(1) regardless of number of priorities or tasks. Limited to 32 priorities when using bitmap method.

**Q3: Why does FreeRTOS use PendSV for context switching instead of doing it directly in SysTick?**
**A:** PendSV is configured as the lowest-priority exception. After SysTick determines a context switch is needed, it pends PendSV. All higher-priority ISRs complete first, then PendSV fires. This ensures: (1) ISR latency is not increased by context switch code, (2) no nested-ISR complications during context save/restore, (3) tail-chaining works properly. If context switch happened in SysTick (mid-priority), it could delay other pending ISRs. The pattern: SysTick → set PendSV pending → exit SysTick → other ISRs run if pending → PendSV runs last → context switch.

**Q4: Explain tickless idle mode. When would you NOT use it?**
**A:** Tickless idle suppresses periodic tick interrupts during idle periods. Instead of waking every 1ms for the tick, the timer is reprogrammed to fire only when the next task is due. On wake, the tick count is compensated for the sleep duration. Benefits: dramatic power reduction (10-100x less idle power). When NOT to use: (1) when accurate free-running tick count is needed for timestamps, (2) when timer reprogramming latency is unacceptable, (3) when the system is rarely idle (overhead of constant timer reconfiguration outweighs savings), (4) when peripherals require continuous polling that prevents sleep.

---

## Summary

- Preemptive scheduling: highest-priority ready task always runs, immediate preemption
- Cooperative: task runs until it yields — simpler but not real-time suitable
- FreeRTOS uses bitmap + CLZ for O(1) priority selection on ARM
- Context switch triggers: tick (time-slice), block, ISR unblock, priority change
- PendSV handler performs actual context switch at lowest exception priority
- Idle task: cleanup, power management, tickless idle for ultra-low-power
- Time-slicing gives round-robin among same-priority tasks
- All production RTOS use fixed-priority preemptive as default policy

---

[Previous Chapter: Task Management ←](Chapter_05_Task_Management.md) | [Next Chapter: Scheduling Algorithms →](Chapter_07_Scheduling_Algorithms.md)
