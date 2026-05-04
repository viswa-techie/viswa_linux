# Chapter 9: Interrupt Handling in RTOS

## Learning Goals
- Understand ISR design rules in RTOS environments
- Learn deferred interrupt processing patterns
- Know ISR-safe API variants and why they exist
- Understand interrupt latency components and measurement
- Learn interrupt nesting and priority management
- Compare interrupt handling across RTOS platforms

---

## 1. ISR Constraints in RTOS

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ISR Rules — What you CANNOT do in an ISR:               │
  │                                                           │
  │  ╔══════════════════════════════════════════════════════╗ │
  │  ║ 1. NO blocking calls (xQueueReceive with timeout)   ║ │
  │  ║ 2. NO mutex take (xSemaphoreTake on mutex)          ║ │
  │  ║ 3. NO task delay (vTaskDelay, vTaskDelayUntil)      ║ │
  │  ║ 4. NO memory allocation (pvPortMalloc) — usually    ║ │
  │  ║ 5. NO floating point (unless FPU context managed)   ║ │
  │  ║ 6. NO printf or heavy processing                    ║ │
  │  ║ 7. Keep ISR as SHORT as possible                    ║ │
  │  ╚══════════════════════════════════════════════════════╝ │
  │                                                           │
  │  Why: ISRs run in Handler mode (MSP), above all task     │
  │  priorities. Blocking would deadlock the system —         │
  │  no task can run to unblock the ISR.                      │
  │                                                           │
  │  ISR-safe API pattern (FreeRTOS):                        │
  │  ┌────────────────────┬─────────────────────────┐        │
  │  │ Task API           │ ISR-safe API              │       │
  │  ├────────────────────┼─────────────────────────┤        │
  │  │ xQueueSend()       │ xQueueSendFromISR()       │       │
  │  │ xSemaphoreGive()   │ xSemaphoreGiveFromISR()   │       │
  │  │ xTaskNotifyGive()  │ vTaskNotifyGiveFromISR()   │       │
  │  │ xEventGroupSetBits │ xEventGroupSetBitsFromISR │       │
  │  │ xStreamBufferSend  │ xStreamBufferSendFromISR  │       │
  │  └────────────────────┴─────────────────────────┘        │
  │                                                           │
  │  FromISR() functions:                                    │
  │  - Never block (return immediately if operation fails)   │
  │  - Take pxHigherPriorityTaskWoken parameter              │
  │  - Don't manipulate scheduler directly (deferred)        │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Deferred Interrupt Processing

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Pattern: Split interrupt into ISR (top half) +          │
  │           Task (bottom half / deferred handler)          │
  │                                                           │
  │  ┌──────┐   signal   ┌──────────────────┐               │
  │  │ ISR  │──────────►│ Handler Task      │               │
  │  │(fast)│            │ (does real work)  │               │
  │  └──────┘            └──────────────────┘               │
  │                                                           │
  │  ISR: read hardware, clear interrupt, signal task        │
  │  Task: process data, update state, respond               │
  │                                                           │
  │  Signaling mechanisms:                                   │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ 1. Binary Semaphore    — simple wake-up        │      │
  │  │ 2. Task Notification   — fastest, lightweight  │      │
  │  │ 3. Queue               — pass data with signal │      │
  │  │ 4. Event Group Bits    — multiple event flags   │      │
  │  │ 5. Stream/Message Buf  — data stream from ISR  │      │
  │  └────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────┘
```

```c
/* Deferred interrupt processing example */

/* ISR — runs in Handler mode, must be fast */
void UART_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    uint8_t byte;

    /* Read received byte from hardware */
    if (UART1->ISR & UART_ISR_RXNE) {
        byte = UART1->RDR;  /* Read clears interrupt flag */

        /* Send byte to queue — ISR-safe API */
        xQueueSendFromISR(xUartRxQueue, &byte, &xHigherPriorityTaskWoken);
    }

    /* Request context switch if higher-priority task was unblocked */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
    /*
     * portYIELD_FROM_ISR() expands to:
     *   if (xHigherPriorityTaskWoken == pdTRUE) {
     *       portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;
     *   }
     * Sets PendSV pending → context switch after ISR chain completes
     */
}

/* Handler task — runs in Thread mode, can use full API */
void vUartHandlerTask(void *pvParameters) {
    uint8_t rxByte;
    for (;;) {
        /* Block until ISR sends data */
        if (xQueueReceive(xUartRxQueue, &rxByte, portMAX_DELAY) == pdPASS) {
            /* Process received byte — can take as long as needed */
            protocol_parse(rxByte);
            if (protocol_frame_complete()) {
                process_command(protocol_get_frame());
            }
        }
    }
}

/* Using Task Notification (faster than semaphore) */
void DMA_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    /* Clear DMA transfer complete flag */
    DMA1->IFCR = DMA_IFCR_CTCIF1;

    /* Notify handler task — fastest IPC mechanism */
    vTaskNotifyGiveFromISR(xDmaTaskHandle, &xHigherPriorityTaskWoken);

    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

void vDmaHandlerTask(void *pvParameters) {
    for (;;) {
        /* Block until DMA ISR notifies us */
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);  /* Clear on exit */

        /* Process DMA transfer result */
        process_dma_buffer(dma_buffer, DMA_BUFFER_SIZE);
    }
}
```

---

## 3. Interrupt Latency Analysis

```
  Interrupt Latency Components
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Total Interrupt Latency =                               │
  │    Hardware latency + Software latency + Scheduling delay│
  │                                                           │
  │  ┌─ HW interrupt occurs                                 │
  │  │                                                       │
  │  ├─ Pipeline flush + exception entry    (~12 cycles)    │
  │  │  (hardware saves R0-R3,R12,LR,PC,xPSR)              │
  │  │                                                       │
  │  ├─ Vector fetch + ISR entry            (~5 cycles)     │
  │  │                                                       │
  │  ├─ ISR prologue (if any)               (~2 cycles)     │
  │  │                                                       │
  │  ├─ [If interrupt was masked by         (variable)      │
  │  │   RTOS critical section: BASEPRI]                    │
  │  │                                                       │
  │  ├─ ISR code executes                   (application)   │
  │  │                                                       │
  │  ├─ ISR epilogue + exception return     (~12 cycles)    │
  │  │                                                       │
  │  ├─ PendSV (if context switch needed)   (~30-50 cycles) │
  │  │                                                       │
  │  └─ Handler task resumes                                │
  │                                                           │
  │  Critical section impact on latency:                     │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ FreeRTOS uses BASEPRI (not PRIMASK):            │      │
  │  │                                                  │      │
  │  │ BASEPRI = configMAX_SYSCALL_INTERRUPT_PRIORITY   │      │
  │  │                                                  │      │
  │  │ Interrupts at or below this priority: BLOCKED   │      │
  │  │ Interrupts ABOVE this priority: ALWAYS EXECUTE  │      │
  │  │                                                  │      │
  │  │ This means: ultra-high-priority interrupts      │      │
  │  │ (motor control, safety) are never disabled!     │      │
  │  │ They can't use FreeRTOS API though.             │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Typical worst-case latencies:                           │
  │  ┌─────────────┬──────────────────────────────┐         │
  │  │ Component    │ Cortex-M4 @ 168MHz           │         │
  │  ├─────────────┼──────────────────────────────┤         │
  │  │ HW entry     │ ~12 cycles = 71 ns            │         │
  │  │ Vector fetch │ ~5 cycles = 30 ns             │         │
  │  │ RTOS block   │ 0 - ~500 ns (critical section)│        │
  │  │ ISR body     │ Application dependent          │         │
  │  │ Context sw   │ ~50 cycles = 300 ns           │         │
  │  │ Total to task│ ~2-5 μs typical worst-case     │         │
  │  └─────────────┴──────────────────────────────┘         │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Interrupt Priority vs Task Priority

```
  Priority Space (Cortex-M with RTOS)
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Higher    ┌─────────────────────────────────┐           │
  │  priority  │ HardFault, NMI, Reset           │           │
  │    ▲       │ (always execute, cannot disable) │           │
  │    │       ├─────────────────────────────────┤           │
  │    │       │ High-priority HW interrupts      │           │
  │    │       │ (above configMAX_SYSCALL_INT_PRIO)│          │
  │    │       │ CANNOT use FreeRTOS API!          │           │
  │    │       │ (e.g., safety watchdog, motor PWM)│          │
  │    │       ├─────────────────────────────────┤           │
  │    │       │ configMAX_SYSCALL_INTERRUPT_PRIO │ ← BASEPRI│
  │    │       ├─────────────────────────────────┤           │
  │    │       │ RTOS-managed HW interrupts       │           │
  │    │       │ CAN use FromISR() API            │           │
  │    │       │ (UART, SPI, DMA, ADC, etc.)      │           │
  │    │       ├─────────────────────────────────┤           │
  │    │       │ SysTick (configKERNEL_INT_PRIO)  │           │
  │    │       ├─────────────────────────────────┤           │
  │    │       │ PendSV (lowest HW priority)      │           │
  │    │       ╞═════════════════════════════════╡           │
  │    │       │ Task Priority N (highest)        │           │
  │    │       │ Task Priority N-1                │           │
  │    │       │ ...                              │           │
  │    │       │ Task Priority 1                  │           │
  │    │       │ Task Priority 0 (Idle task)      │           │
  │  Lower    └─────────────────────────────────┘           │
  │  priority                                                 │
  │                                                           │
  │  KEY INSIGHT: ANY interrupt preempts ALL tasks.           │
  │  Interrupt priorities and task priorities are separate    │
  │  domains. The lowest-priority interrupt (PendSV) still   │
  │  preempts the highest-priority task.                     │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Interrupt Nesting

```c
/* Cortex-M NVIC supports automatic interrupt nesting */

/*
 * Example: Three interrupt sources with priorities
 *
 * Timer (priority 2, medium):
 *   Running ─────────→ UART preempts ──→ Timer resumes
 *
 * UART (priority 1, high):
 *   ─────────────────→ Runs ──────────→
 *
 * ADC (priority 3, low):
 *   ─── Timer preempts ──────────────→ ADC resumes
 *
 * Timeline:
 *   Time: ──────────────────────────────────────────→
 *   ADC:  ▓▓▓▓                              ▓▓▓▓▓▓▓
 *   Timer:     ▓▓▓▓▓▓              ▓▓▓▓▓▓▓▓▓
 *   UART:              ▓▓▓▓▓▓▓▓▓▓▓
 *
 *   Nesting depth: 3 levels (ADC → Timer → UART)
 *   Each level uses MSP stack space (~32-36 bytes)
 */

/* configuring NVIC priorities for RTOS */
void interrupt_setup(void) {
    /* Set priority grouping: 4 bits preemption, 0 sub-priority */
    NVIC_SetPriorityGrouping(0);

    /* Interrupts ABOVE this are not RTOS-managed */
    /* configMAX_SYSCALL_INTERRUPT_PRIORITY = 5 (shifted) */

    /* Safety-critical: priority 2 — never delayed by RTOS */
    NVIC_SetPriority(MOTOR_FAULT_IRQn, 2);
    /* CANNOT call any FreeRTOS ...FromISR() function! */

    /* RTOS-managed interrupts: priority 5-15 */
    NVIC_SetPriority(UART1_IRQn, 5);    /* Can use FromISR API */
    NVIC_SetPriority(SPI1_IRQn, 6);
    NVIC_SetPriority(DMA1_IRQn, 7);
    NVIC_SetPriority(ADC_IRQn, 8);

    /* SysTick and PendSV: lowest priority */
    NVIC_SetPriority(SysTick_IRQn, 15);
    NVIC_SetPriority(PendSV_IRQn, 15);
}

/*
 * Common bug: ARM Cortex-M uses LOWER numbers for HIGHER priority!
 *
 * Priority 0 = HIGHEST (NMI level)
 * Priority 15 = LOWEST
 *
 * FreeRTOS convention: higher number = higher task priority
 *
 * This causes confusion! NVIC priorities are inverted from
 * FreeRTOS task priorities.
 */
```

---

## 6. Direct-to-Task Notifications (Fastest ISR-to-Task)

```c
/*
 * Task Notifications: lightweight alternative to semaphore/queue
 * Each task has a built-in 32-bit notification value
 * ~45% faster than binary semaphore, uses no extra RAM
 */

/* ISR: notify task with value */
void CAN_RX_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    uint32_t canId = CAN1->sFIFOMailBox[0].RIR >> 21;

    /* Send notification with CAN ID as value */
    xTaskNotifyFromISR(
        xCanTaskHandle,
        canId,                           /* Notification value */
        eSetValueWithOverwrite,          /* Action: overwrite */
        &xHigherPriorityTaskWoken
    );

    /* Or: bit-set for event flags pattern */
    xTaskNotifyFromISR(
        xCanTaskHandle,
        CAN_RX_EVENT_BIT | CAN_ID_MASK(canId),
        eSetBits,                        /* OR bits into value */
        &xHigherPriorityTaskWoken
    );

    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

/* Task: wait for notification */
void vCanHandlerTask(void *pvParameters) {
    uint32_t notificationValue;
    for (;;) {
        /* Block until notified, read value, clear on exit */
        xTaskNotifyWait(
            0x00,                    /* Don't clear on entry */
            0xFFFFFFFF,              /* Clear all bits on exit */
            &notificationValue,      /* Received value */
            portMAX_DELAY            /* Wait forever */
        );

        uint32_t canId = notificationValue & 0x7FF;
        process_can_message(canId);
    }
}

/*
 * Task Notification Actions:
 *
 * eSetBits:              OR bits into notification value
 * eIncrement:            Increment notification value (like counting semaphore)
 * eSetValueWithOverwrite: Set value, overwriting any pending
 * eSetValueWithoutOverwrite: Set value only if no pending notification
 * eNoAction:             Just notify, don't modify value
 *
 * Limitation: only ONE sender can notify a given task this way
 * (each task has only one notification value — or array since v10.4)
 */
```

---

## 7. Interrupt Handling Across RTOS

| Feature | FreeRTOS | Zephyr | QNX | VxWorks |
|---------|----------|--------|-----|---------|
| **ISR model** | Direct ISR + FromISR API | ISR + work queue | Interrupt event + pulse | ISR + deferred ISR |
| **API in ISR** | Separate FromISR() variants | Same API (k_sem_give) | Minimal (MsgSendPulse) | Same API (semGive) |
| **Deferred processing** | Task + semaphore/notification | Work queue (k_work) | Interrupt thread | netTask / custom |
| **ISR stack** | MSP (shared) | Separate ISR stack | Per-CPU ISR stack | Interrupt stack |
| **Priority masking** | BASEPRI | BASEPRI or PRIMASK | GIC masking | intLock/intUnlock |
| **Nesting** | Yes (NVIC auto) | Yes (configurable) | Yes | Yes |

---

## Interview Questions

**Q1: Why does FreeRTOS have separate FromISR() API functions?**
**A:** FreeRTOS ISR-safe functions differ from task functions in critical ways: (1) They never block — if a queue is full, `xQueueSendFromISR()` returns `errQUEUE_FULL` immediately instead of blocking. Blocking in ISR would deadlock the system since no task can run to make space. (2) They use `pxHigherPriorityTaskWoken` instead of directly triggering context switch — the switch is deferred to PendSV after the ISR chain completes. (3) They use a different critical section mechanism — ISR functions save/restore BASEPRI instead of suspending the scheduler. (4) They skip certain checks (like mutex priority inheritance) that aren't applicable in ISR context. Having separate functions catches errors at compile time — calling `xQueueSend()` from ISR causes assertion failure in debug mode.

**Q2: What is BASEPRI and why does FreeRTOS use it instead of PRIMASK?**
**A:** BASEPRI is a Cortex-M register that masks interrupts at or below a specified priority level. PRIMASK disables ALL maskable interrupts (equivalent to BASEPRI=0). FreeRTOS sets BASEPRI to `configMAX_SYSCALL_INTERRUPT_PRIORITY` during critical sections. This means: interrupts with priority ABOVE this threshold (lower number on ARM) continue to execute even during RTOS critical sections. This is crucial for safety: a motor fault interrupt or safety watchdog can always respond, maintaining hard real-time guarantees for critical hardware. The trade-off: these high-priority ISRs cannot use any FreeRTOS API (no FromISR calls), because the kernel data structures may be in an inconsistent state during the critical section.

**Q3: Explain the deferred interrupt processing pattern and its benefits.**
**A:** The ISR (top half) does only the minimum: read hardware register, clear interrupt flag, signal a handler task. The handler task (bottom half) does the actual processing. Benefits: (1) ISR runs fast — microseconds, not milliseconds — minimizing interrupt latency for other sources. (2) Handler task can use full RTOS API — blocking calls, mutex, memory allocation. (3) Handler task participates in scheduling — its priority can be set relative to other tasks. (4) Testable — task code can be unit tested without hardware. (5) Timing predictable — ISR time is bounded, processing time is scheduled. (6) Debug-friendly — task code can use printf, breakpoints. The signaling mechanism choice matters: task notification is fastest (~45% faster than binary semaphore), queue passes data, event groups handle multiple events.

---

## Summary

- ISRs must be short: read hardware, clear flag, signal task, return
- FreeRTOS: separate FromISR() API — never blocks, defers context switch
- Deferred processing: ISR signals handler task via notification/semaphore/queue
- Task notifications: fastest ISR-to-task mechanism (~45% faster than semaphore)
- BASEPRI allows ultra-high-priority ISRs to run during RTOS critical sections
- Interrupt priorities (HW) and task priorities (SW) are separate domains
- ANY interrupt preempts ALL tasks — PendSV is the bridge between the two
- Interrupt nesting: NVIC handles automatically, each level costs ~32 bytes MSP

---

[Previous Chapter: Context Switching ←](Chapter_08_Context_Switching.md) | [Next Chapter: Inter-Task Communication →](Chapter_10_IPC.md)
