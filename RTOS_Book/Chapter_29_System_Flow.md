# Chapter 29: RTOS System Flow

## Learning Goals
- Understand the complete RTOS boot and initialization sequence
- Trace execution from hardware reset to task execution
- Know scheduler startup internals and first task selection
- Master shutdown and restart patterns

---

## 1. Complete Boot Sequence

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Hardware Reset                                          │
  │       │                                                   │
  │       ▼                                                   │
  │  ┌─────────────────────┐                                │
  │  │ 1. Load MSP from    │  Vector table[0] = _estack     │
  │  │    address 0x0      │  (top of SRAM)                  │
  │  └─────────┬───────────┘                                │
  │            │                                              │
  │            ▼                                              │
  │  ┌─────────────────────┐                                │
  │  │ 2. Load PC from     │  Vector table[1] = Reset_Handler│
  │  │    address 0x4      │  (entry point)                  │
  │  └─────────┬───────────┘                                │
  │            │                                              │
  │            ▼                                              │
  │  ┌─────────────────────┐                                │
  │  │ 3. Reset_Handler()  │                                │
  │  │  • Copy .data       │  Flash → SRAM                   │
  │  │  • Zero .bss        │  Zero-initialize               │
  │  │  • Call SystemInit() │  Configure clocks (PLL, etc)   │
  │  │  • Call main()      │                                 │
  │  └─────────┬───────────┘                                │
  │            │                                              │
  │            ▼                                              │
  │  ┌─────────────────────┐                                │
  │  │ 4. main()           │                                │
  │  │  • HAL_Init()       │  Initialize HAL tick            │
  │  │  • Configure_Clocks │  Full clock tree setup          │
  │  │  • Init peripherals │  GPIO, UART, SPI, etc.          │
  │  │  • Create tasks     │  xTaskCreate() for each task    │
  │  │  • Create queues    │  xQueueCreate(), semaphores     │
  │  │  • vTaskStartScheduler()                              │
  │  └─────────┬───────────┘                                │
  │            │                                              │
  │            ▼  (never returns if scheduler works)         │
  │  ┌─────────────────────┐                                │
  │  │ 5. vTaskStartScheduler()                             │
  │  │  • Create Idle task │  Priority 0 (lowest)            │
  │  │  • Create Timer task│  If timers enabled              │
  │  │  • Setup SysTick    │  configTICK_RATE_HZ             │
  │  │  • Set PendSV/      │                                 │
  │  │    SysTick priority │  Lowest NVIC priority           │
  │  │  • Call SVC #0      │  Start first task               │
  │  └─────────┬───────────┘                                │
  │            │                                              │
  │            ▼                                              │
  │  ┌─────────────────────┐                                │
  │  │ 6. SVC_Handler      │                                │
  │  │  • Load PSP from    │  First task's pxTopOfStack      │
  │  │    highest-prio TCB │                                 │
  │  │  • Pop R4-R11       │  Restore software context       │
  │  │  • Set PSP          │  Switch to Process Stack        │
  │  │  • BX LR (0xFFFFFFFD)  Return to thread mode         │
  │  └─────────┬───────────┘                                │
  │            │                                              │
  │            ▼                                              │
  │  ┌─────────────────────┐                                │
  │  │ 7. First task runs! │  Highest priority task          │
  │  │    Normal operation │  Scheduler handles preemption   │
  │  └─────────────────────┘                                │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Detailed Scheduler Startup

```c
/* tasks.c: vTaskStartScheduler() — simplified */
void vTaskStartScheduler(void) {
    /* Create idle task at priority 0 */
    xTaskCreate(prvIdleTask, "IDLE",
                configMINIMAL_STACK_SIZE, NULL,
                tskIDLE_PRIORITY, &xIdleTaskHandle);

    /* Create timer daemon task if configured */
    #if (configUSE_TIMERS == 1)
    xTimerCreateTimerTask();
    #endif

    /* Disable interrupts (will be enabled when first task runs) */
    portDISABLE_INTERRUPTS();

    xSchedulerRunning = pdTRUE;
    xTickCount = 0;

    /* Port-specific: configure tick timer and start first task */
    /* This function DOES NOT RETURN */
    xPortStartScheduler();
}

/* port.c: xPortStartScheduler() for Cortex-M */
BaseType_t xPortStartScheduler(void) {
    /* Set PendSV and SysTick to lowest priority */
    *(portNVIC_SHPR3_REG) |= portNVIC_PENDSV_PRI;
    *(portNVIC_SHPR3_REG) |= portNVIC_SYSTICK_PRI;

    /* Configure SysTick timer */
    portNVIC_SYSTICK_LOAD_REG = (configCPU_CLOCK_HZ /
                                  configTICK_RATE_HZ) - 1;
    portNVIC_SYSTICK_CTRL_REG = portNVIC_SYSTICK_CLK_BIT |
                                 portNVIC_SYSTICK_INT_BIT |
                                 portNVIC_SYSTICK_ENABLE_BIT;

    /* Start first task via SVC exception */
    prvStartFirstTask();  /* Triggers SVC #0 */

    /* Should never reach here */
    return 0;
}
```

---

## 3. Tick Processing Flow

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  SysTick fires every 1ms (configTICK_RATE_HZ = 1000)   │
  │       │                                                   │
  │       ▼                                                   │
  │  ┌────────────────────────┐                              │
  │  │ SysTick_Handler:       │                              │
  │  │  portDISABLE_INTERRUPTS│  Mask below threshold        │
  │  │  xTaskIncrementTick()  │  Core tick processing        │
  │  │  portENABLE_INTERRUPTS │                              │
  │  │  if(switch needed)     │                              │
  │  │    portNVIC_PENDSVSET  │  Trigger PendSV              │
  │  └────────┬───────────────┘                              │
  │           │                                               │
  │           ▼                                               │
  │  ┌────────────────────────┐                              │
  │  │ xTaskIncrementTick():   │                              │
  │  │  xTickCount++           │  Advance tick               │
  │  │                          │                              │
  │  │  Check delayed list:    │                              │
  │  │  while (task at head    │                              │
  │  │    has expired delay) { │                              │
  │  │    Remove from delayed  │                              │
  │  │    Add to ready list    │                              │
  │  │    if (prio > current)  │                              │
  │  │      return pdTRUE      │  → context switch needed    │
  │  │  }                      │                              │
  │  │                          │                              │
  │  │  If time-slicing:       │                              │
  │  │  if (other tasks at     │                              │
  │  │    same priority exist) │                              │
  │  │      return pdTRUE      │  → round-robin switch       │
  │  └────────────────────────┘                              │
  │                                                           │
  │  PendSV runs at lowest priority:                         │
  │  - Ensures all ISRs complete first                       │
  │  - Then performs context switch                          │
  │  - Saves current task registers, loads next task         │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Task Lifecycle Flow

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  xTaskCreate()                                           │
  │       │                                                   │
  │       ▼                                                   │
  │  ┌─────────┐      ┌─────────┐      ┌─────────┐         │
  │  │  Ready  │─────►│ Running │─────►│ Blocked │         │
  │  │         │◄─────│         │      │ (delay/ │         │
  │  │         │      │         │      │  queue/ │         │
  │  │         │      │         │      │  sema)  │         │
  │  │         │◄─────────────────────│         │         │
  │  └────┬────┘      └─────────┘      └─────────┘         │
  │       │                                                   │
  │       │ vTaskSuspend()    vTaskResume()                   │
  │       ▼                       │                           │
  │  ┌─────────┐                  │                           │
  │  │Suspended│──────────────────┘                           │
  │  └─────────┘                                              │
  │                                                           │
  │  vTaskDelete()  (from any state)                         │
  │       │                                                   │
  │       ▼                                                   │
  │  ┌─────────┐                                             │
  │  │ Deleted │  TCB/stack freed by Idle task               │
  │  └─────────┘  (cleanup deferred to idle)                 │
  │                                                           │
  │  Common patterns:                                        │
  │  1. Periodic task:                                       │
  │     Ready → Running → Blocked (vTaskDelayUntil) → Ready │
  │  2. Event-driven task:                                   │
  │     Ready → Running → Blocked (xQueueReceive) → Ready   │
  │  3. One-shot task:                                       │
  │     Ready → Running → Deleted (vTaskDelete(NULL))        │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. System Shutdown and Restart

```c
/* Graceful shutdown pattern */
typedef enum {
    SYSTEM_RUNNING,
    SYSTEM_SHUTTING_DOWN,
    SYSTEM_HALTED
} SystemState_t;

static volatile SystemState_t system_state = SYSTEM_RUNNING;

void vShutdownTask(void *pvParameters) {
    /* Wait for shutdown command */
    ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

    system_state = SYSTEM_SHUTTING_DOWN;

    /* 1. Stop accepting new work */
    /* 2. Wait for in-progress operations to complete */
    vTaskDelay(pdMS_TO_TICKS(100));

    /* 3. Save state to non-volatile storage */
    save_persistent_data();

    /* 4. Disable peripherals safely */
    disable_motors();      /* Safe state for actuators */
    disable_comm();        /* Close connections gracefully */

    /* 5. System reset */
    __NVIC_SystemReset();  /* CMSIS: triggers reset */
}

/*
 * NVIC_SystemReset():
 * - Writes to SCB->AIRCR with SYSRESETREQ bit
 * - Resets entire MCU (CPU + peripherals)
 * - Execution restarts from Reset_Handler
 * - RAM content may be preserved (depends on reset type)
 *
 * Watchdog reset:
 * - If software hangs, hardware watchdog resets system
 * - Same effect as NVIC_SystemReset
 * - Can store reset reason for post-mortem analysis
 */
```

---

## Interview Questions

**Q1: Describe the complete boot sequence from power-on to first RTOS task running.**
**A:** (1) Hardware loads MSP from vector table address 0x0 (initial stack pointer = top of SRAM) and PC from address 0x4 (Reset_Handler entry point). (2) Reset_Handler: copies `.data` section from Flash to SRAM (initialized globals), zeros `.bss` section, calls `SystemInit()` to configure PLL and system clock, then calls `main()`. (3) `main()`: initializes HAL and peripherals (GPIO, UART, SPI), creates RTOS tasks with `xTaskCreate()`, creates IPC objects (queues, semaphores), calls `vTaskStartScheduler()`. (4) `vTaskStartScheduler()`: creates idle task (priority 0) and timer daemon task, configures SysTick timer at `configTICK_RATE_HZ`, sets PendSV and SysTick to lowest NVIC priority, triggers SVC #0 to start first task. (5) SVC handler: loads `pxCurrentTCB->pxTopOfStack` (highest priority task), pops R4-R11, sets PSP, returns via EXC_RETURN (0xFFFFFFFD) to thread mode. (6) First task begins executing — scheduler is now operational, SysTick drives preemptive scheduling.

**Q2: Why does FreeRTOS use SVC to start the first task instead of just jumping to it?**
**A:** The first task needs to run in Thread mode with PSP (Process Stack Pointer), but startup code runs in Thread mode with MSP (Main Stack Pointer) — and there's no direct way to switch from Thread/MSP to Thread/PSP. The SVC exception provides a clean transition: (1) SVC triggers Handler mode (which has full control). (2) SVC handler loads the first task's stack into PSP. (3) SVC handler returns with EXC_RETURN value 0xFFFFFFFD, which tells the hardware to return to Thread mode using PSP. This correctly establishes the execution context: first task runs in Thread mode + PSP, and future interrupts will stack onto PSP (task stack). MSP is now free for interrupt handler stacking. This is the only clean way to set up the dual-stack architecture that FreeRTOS relies on for per-task stacks.

---

## Summary

- Boot: Reset → copy .data → zero .bss → SystemInit → main → vTaskStartScheduler
- Scheduler start: create idle/timer tasks → configure SysTick → SVC #0 → first task runs
- Tick flow: SysTick → increment tick → check delayed list → trigger PendSV if switch needed
- PendSV runs at lowest priority: ensures all ISRs complete before context switch
- Task lifecycle: Ready → Running → Blocked → Ready (periodic) or Deleted (one-shot)
- SVC used for first task: only way to transition Thread/MSP → Thread/PSP correctly

---

[Previous Chapter: Kernel Source Structure ←](Chapter_28_Kernel_Source.md) | [Next Chapter: Debug Tools →](Chapter_30_Debug_Tools.md)
