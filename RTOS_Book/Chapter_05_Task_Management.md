# Chapter 5: Task Management

## Learning Goals
- Understand task concept and lifecycle in RTOS
- Learn Task Control Block (TCB) internals
- Know task states and transitions
- Implement task creation, deletion, suspension in FreeRTOS
- Understand task stack allocation and overflow detection
- Compare task models across RTOS platforms

---

## 1. Tasks vs Threads vs Processes

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Linux/GPOS:                                             │
  │  ┌──────────────┐      ┌──────────────┐                  │
  │  │   Process A   │      │   Process B   │                 │
  │  │ ┌──────────┐ │      │ ┌──────────┐ │                  │
  │  │ │ Thread 1 │ │      │ │ Thread 1 │ │                  │
  │  │ │ Thread 2 │ │      │ │ Thread 2 │ │                  │
  │  │ └──────────┘ │      │ └──────────┘ │                  │
  │  │  Own VM space │      │  Own VM space │                 │
  │  └──────────────┘      └──────────────┘                  │
  │   Full isolation via MMU                                  │
  │                                                           │
  │  RTOS (Flat memory model):                               │
  │  ┌──────────────────────────────────┐                    │
  │  │        Shared Address Space       │                    │
  │  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐│                    │
  │  │  │Task │ │Task │ │Task │ │Idle ││                    │
  │  │  │  A  │ │  B  │ │  C  │ │Task ││                    │
  │  │  │stack│ │stack│ │stack│ │     ││                    │
  │  │  └─────┘ └─────┘ └─────┘ └─────┘│                    │
  │  │  Each task: own stack, own PC     │                    │
  │  │  Shared: code, globals, heap      │                    │
  │  └──────────────────────────────────┘                    │
  │   No MMU required, minimal overhead                      │
  │                                                           │
  │  Exception (QNX microkernel):                            │
  │  - Processes with separate address spaces (uses MMU)     │
  │  - POSIX threads within each process                     │
  │  - Full isolation like Linux but with RT guarantees      │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Task States

```
  Task State Machine (FreeRTOS model)
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │              vTaskCreate()                                │
  │                   │                                       │
  │                   ▼                                       │
  │              ┌─────────┐                                  │
  │              │  Ready   │◄───────────────────────┐       │
  │              └────┬─────┘                        │       │
  │                   │                               │       │
  │      Scheduler picks   Preempted or              │       │
  │      highest priority   time slice expired        │       │
  │                   │                               │       │
  │                   ▼                               │       │
  │              ┌─────────┐                          │       │
  │              │ Running  │──────────────────────────┘       │
  │              └────┬─────┘                                 │
  │                   │                                       │
  │    ┌──────────────┼──────────────┐                       │
  │    │              │              │                        │
  │    ▼              ▼              ▼                        │
  │ ┌──────┐    ┌──────────┐   ┌───────────┐                │
  │ │Blocked│    │Suspended │   │ Deleted   │                │
  │ │(Wait) │    │(Explicit)│   │(Freed)    │                │
  │ └──┬───┘    └────┬─────┘   └───────────┘                │
  │    │              │                                       │
  │    │ Event/       │ vTaskResume()                         │
  │    │ Timeout      │                                       │
  │    └──────────────┴──────────► Ready                     │
  │                                                           │
  │  Blocked: waiting for queue, semaphore, delay, event     │
  │  Suspended: explicitly suspended, only resume unblocks   │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Task Control Block (TCB)

```c
/* FreeRTOS TCB structure (simplified from tasks.c) */
typedef struct tskTaskControlBlock {
    /* Stack pointer — MUST be first member for context switch */
    volatile StackType_t *pxTopOfStack;

    /* List items for ready/blocked/suspended lists */
    ListItem_t       xStateListItem;
    ListItem_t       xEventListItem;

    /* Task priority (0 = lowest, configMAX_PRIORITIES-1 = highest) */
    UBaseType_t      uxPriority;

    /* Stack memory */
    StackType_t      *pxStack;           /* Base of stack */
    #if (configRECORD_STACK_HIGH_WATER_MARK == 1)
        UBaseType_t  uxStackHighWaterMark;
    #endif

    /* Task name for debug */
    char             pcTaskName[configMAX_TASK_NAME_LEN];

    /* Mutex tracking for priority inheritance */
    #if (configUSE_MUTEXES == 1)
        UBaseType_t  uxBasePriority;     /* Original priority */
        UBaseType_t  uxMutexesHeld;      /* Count of held mutexes */
    #endif

    /* Notification value(s) for task notifications */
    #if (configUSE_TASK_NOTIFICATIONS == 1)
        volatile uint32_t ulNotifiedValue[configTASK_NOTIFICATION_ARRAY_ENTRIES];
        volatile uint8_t  ucNotifyState[configTASK_NOTIFICATION_ARRAY_ENTRIES];
    #endif
} tskTCB;

/* TCB Memory Layout (Cortex-M):
 *
 * +-------------------+  ← TCB address (pxCurrentTCB)
 * | pxTopOfStack ptr  |  ← Points into stack below
 * | xStateListItem    |  ← Linked into pxReadyTasksLists[priority]
 * | xEventListItem    |  ← Linked into event wait list
 * | uxPriority        |
 * | pxStack (base)    |  ← Points to allocated stack memory
 * | pcTaskName[16]    |
 * | uxBasePriority    |  ← For priority inheritance
 * | ulNotifiedValue   |
 * +-------------------+
 *
 * Stack (separate allocation):
 * +-------------------+  ← pxStack (stack base, low address)
 * | stack canary      |  ← 0xA5A5A5A5 pattern (overflow detect)
 * | ... free space .. |
 * | saved R4-R11      |  ← Manually pushed on context switch
 * | saved R0-R3       |  ← Auto-pushed by NVIC on exception
 * | saved R12         |
 * | saved LR          |
 * | saved PC          |
 * | saved xPSR        |
 * +-------------------+  ← Top of stack (high address)
 */
```

---

## 4. Task Creation

```c
/* FreeRTOS: Static vs Dynamic task creation */

/**** Dynamic allocation (heap) ****/
TaskHandle_t xSensorTaskHandle;

BaseType_t result = xTaskCreate(
    vSensorTask,              /* Task function */
    "SensorTask",             /* Name (debug only) */
    256,                      /* Stack depth (words, not bytes!) */
    (void *)&sensorConfig,    /* Parameter passed to task */
    3,                        /* Priority (higher = more important) */
    &xSensorTaskHandle        /* Output handle (can be NULL) */
);
/* Returns pdPASS on success, errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY on failure */

/**** Static allocation (compile-time) ****/
static StaticTask_t xSensorTaskBuffer;
static StackType_t  xSensorStack[256];

TaskHandle_t xHandle = xTaskCreateStatic(
    vSensorTask,              /* Task function */
    "SensorTask",             /* Name */
    256,                      /* Stack depth (words) */
    (void *)&sensorConfig,    /* Parameter */
    3,                        /* Priority */
    xSensorStack,             /* Stack buffer (user-provided) */
    &xSensorTaskBuffer        /* TCB buffer (user-provided) */
);
/* Never fails — memory is pre-allocated at compile time */
/* Preferred in safety-critical systems (no malloc at runtime) */

/*** Task function pattern ***/
void vSensorTask(void *pvParameters) {
    SensorConfig_t *config = (SensorConfig_t *)pvParameters;

    /* One-time initialization */
    sensor_hw_init(config->channel);

    /* Infinite loop — tasks must never return */
    for (;;) {
        int16_t reading = sensor_read(config->channel);
        xQueueSend(xSensorQueue, &reading, portMAX_DELAY);
        vTaskDelay(pdMS_TO_TICKS(100));  /* Block for 100ms */
    }
    /* If reached, delete self: vTaskDelete(NULL); */
}
```

---

## 5. Task Stack Sizing and Overflow Detection

```
  Stack Overflow Detection Methods
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Method 1: High Water Mark Check                         │
  │  ┌─────────────────────────────────┐                     │
  │  │ Stack base (low addr)           │                     │
  │  │ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  fill 0xA5  │                     │
  │  │ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓             │                     │
  │  │ ░░░░░░░░░░░░░░░░░  used space  │ ← High water mark  │
  │  │ ░░░░░░░░░░░░░░░░░             │                     │
  │  │ ░░░░░░░░░░░░░░░░░  ← SP       │                     │
  │  │ Stack top (high addr)           │                     │
  │  └─────────────────────────────────┘                     │
  │  uxTaskGetStackHighWaterMark(handle);                    │
  │  Returns: minimum free stack words ever seen             │
  │                                                           │
  │  Method 2: Runtime Check (configCHECK_FOR_STACK_OVERFLOW)│
  │  - Check 1: SP within stack bounds on context switch     │
  │  - Check 2: Verify canary pattern at stack base intact   │
  │  - Calls vApplicationStackOverflowHook() on violation    │
  │                                                           │
  │  Method 3: MPU Guard Region                              │
  │  - MPU region below each stack set to no-access          │
  │  - HardFault/MemManage on overflow                       │
  │  - Hardware protection—catches overflow immediately      │
  └──────────────────────────────────────────────────────────┘
```

```c
/* Stack sizing guidelines */
/*
 * Base requirement for context save: ~64 bytes (Cortex-M4)
 * + Local variables
 * + Function call depth (each call: 4-8 bytes return address + locals)
 * + ISR nesting (each level: ~64 bytes auto-saved)
 * + Library calls (printf can use 1KB+!)
 *
 * Rule of thumb:
 * - Simple task: 128-256 words (512-1024 bytes)
 * - Moderate task: 256-512 words
 * - Complex task (printf, deep calls): 512-1024 words
 * - Start large, measure with high water mark, then reduce
 */

void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
    /* Called when stack overflow detected */
    /* DO NOT use printf or any stack-heavy function here */
    __disable_irq();
    /* Log task name to UART via direct register access */
    volatile char *name = pcTaskName;
    /* Trigger breakpoint or reset */
    __BKPT(0);
    for (;;) {} /* Halt */
}
```

---

## 6. Task Management Operations

```c
/* Task lifecycle operations in FreeRTOS */

/* Suspend a task (remove from scheduling) */
vTaskSuspend(xSensorTaskHandle);     /* Suspend specific task */
vTaskSuspend(NULL);                  /* Suspend calling task */

/* Resume a task */
vTaskResume(xSensorTaskHandle);      /* Resume from task context */
xTaskResumeFromISR(xSensorTaskHandle); /* Resume from ISR */

/* Delete a task */
vTaskDelete(xSensorTaskHandle);      /* Delete specific task */
vTaskDelete(NULL);                   /* Delete calling task (self) */
/* Warning: dynamically allocated stack/TCB freed by idle task */

/* Change priority at runtime */
vTaskPrioritySet(xSensorTaskHandle, 5);  /* Raise priority */
UBaseType_t prio = uxTaskPriorityGet(xSensorTaskHandle);

/* Delay (block) a task */
vTaskDelay(pdMS_TO_TICKS(100));      /* Relative delay: 100ms from now */
vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(100)); /* Absolute periodic */

/*
 * vTaskDelay vs vTaskDelayUntil:
 *
 * vTaskDelay(100ms):
 *   |--exec--|----100ms----|--exec--|----100ms----|
 *   Total period = exec_time + 100ms (variable!)
 *
 * vTaskDelayUntil(100ms):
 *   |--exec--|---wait---|--exec--|---wait---|
 *   |<-----100ms------>|<-----100ms------>|
 *   Total period = exactly 100ms (constant, for periodic tasks)
 */
```

---

## 7. RTOS Task Lists (Scheduler Internals)

```
  FreeRTOS Ready List Architecture
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  pxReadyTasksLists[configMAX_PRIORITIES]:                │
  │                                                           │
  │  Priority 4: ─── [TaskA] ──── [TaskD] ──── (round-robin)│
  │  Priority 3: ─── [TaskB] ────                            │
  │  Priority 2: ─── (empty) ────                            │
  │  Priority 1: ─── [TaskC] ──── [TaskE] ────              │
  │  Priority 0: ─── [IdleTask] ────                         │
  │                                                           │
  │  Scheduler always runs highest non-empty priority list   │
  │  Multiple tasks at same priority → round-robin time-slice│
  │                                                           │
  │  Other lists:                                            │
  │  ┌──────────────────────────────────────────────────┐    │
  │  │ xDelayedTaskList:     tasks blocked with timeout  │    │
  │  │ xPendingReadyList:    tasks unblocked from ISR    │    │
  │  │ xSuspendedTaskList:   explicitly suspended tasks  │    │
  │  │ xTasksWaitingTermination: deleted, awaiting cleanup│   │
  │  └──────────────────────────────────────────────────┘    │
  │                                                           │
  │  pxCurrentTCB: points to TCB of currently running task   │
  └──────────────────────────────────────────────────────────┘
```

---

## 8. Task Models Across RTOS Platforms

| Feature | FreeRTOS | Zephyr | QNX Neutrino | VxWorks |
|---------|----------|--------|-------------|---------|
| **Unit** | Task | Thread | POSIX Thread | Task |
| **Address space** | Shared (flat) | Shared (optional MPU) | Per-process (MMU) | Shared or RTP |
| **Creation API** | `xTaskCreate()` | `k_thread_create()` | `pthread_create()` | `taskSpawn()` |
| **Priority** | 0=lowest | 0=highest (configurable) | 0-63 (0=highest) | 0-255 (0=highest) |
| **Stack** | User-specified | User or pool | Per-thread | User-specified |
| **Static alloc** | `xTaskCreateStatic()` | Default | N/A | N/A |
| **Max tasks** | Limited by RAM | Limited by RAM | System limit | Limited by RAM |
| **Isolation** | None/MPU | None/MPU | Full MMU | None or RTP (MMU) |

---

## Interview Questions

**Q1: Why must an RTOS task function never return?**
**A:** An RTOS task is not called like a regular function with a valid return address on the stack. The "return address" on the stack is typically set to a cleanup function or is invalid. If a task function returns, execution will jump to an undefined address, causing a HardFault or unpredictable behavior. Proper patterns: (1) infinite `for(;;)` loop with blocking calls, (2) call `vTaskDelete(NULL)` before returning to properly clean up. In FreeRTOS, if the task does return, the port-specific wrapper calls `vTaskDelete(NULL)` as a safety net on some ports.

**Q2: What is the difference between vTaskDelay and vTaskDelayUntil?**
**A:** `vTaskDelay(ticks)` delays the task for a RELATIVE number of ticks from the moment it's called. The actual period between executions = execution_time + delay_time, which varies. `vTaskDelayUntil(&lastWake, ticks)` delays until an ABSOLUTE tick count, compensating for execution time. This gives a constant period regardless of how long the task code takes to execute. Use `vTaskDelayUntil` for periodic tasks (sensor sampling, control loops) where consistent timing matters.

**Q3: How would you determine the correct stack size for an RTOS task?**
**A:** (1) Start with a generous allocation (e.g., 1024 words). (2) Run the task through all code paths including worst-case scenarios. (3) Use `uxTaskGetStackHighWaterMark()` to find minimum free stack ever seen. (4) Set final stack size = actual usage + 20-30% safety margin. Must account for: ISR nesting (auto-saved context per level), deepest function call chain, largest local variables, library function usage (avoid printf in production — can use 1KB+). In safety-critical systems, use static analysis tools to calculate worst-case stack depth.

**Q4: Explain static vs dynamic task allocation. When would you use each?**
**A:** Dynamic (`xTaskCreate`): TCB and stack allocated from heap at runtime. Simpler code, but: heap fragmentation risk, allocation can fail at runtime, not suitable for safety-critical. Static (`xTaskCreateStatic`): TCB and stack are pre-allocated arrays, passed to creation function. Cannot fail at runtime, no heap dependency, deterministic. Use static for: MISRA compliance, safety-critical (IEC 61508, ISO 26262), medical devices, any system where runtime allocation failure is unacceptable. Use dynamic for: prototyping, non-critical systems, when task count varies at runtime.

---

## Summary

- RTOS tasks: independent execution units with own stack and program counter, shared address space
- Task states: Ready → Running → Blocked/Suspended → Ready (cyclic)
- TCB holds stack pointer (first field), priority, list items, notification values
- `xTaskCreate` (dynamic/heap) vs `xTaskCreateStatic` (static/compile-time)
- Tasks must never return — use infinite loop with blocking calls
- Stack overflow detection: canary pattern, high water mark, MPU guard regions
- `vTaskDelayUntil` for periodic tasks, `vTaskDelay` for relative delays
- Scheduler uses priority-indexed ready lists; highest non-empty priority runs first

---

[Previous Chapter: Hardware Architecture ←](Chapter_04_Hardware.md) | [Next Chapter: Real-Time Scheduling →](Chapter_06_Scheduling.md)
