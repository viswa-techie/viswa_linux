# Chapter 15: Time Management

## Learning Goals
- Understand RTOS tick system and time representation
- Master software timers (one-shot and periodic)
- Learn high-resolution timing beyond the tick resolution
- Know tickless idle implementation details
- Understand watchdog timers in RTOS context
- Compare time management across RTOS platforms

---

## 1. RTOS Tick System

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Tick: periodic interrupt that drives RTOS time keeping  │
  │                                                           │
  │  SysTick (Cortex-M) → SysTick_Handler → xTaskIncrementTick │
  │                                                           │
  │  ──┬────┬────┬────┬────┬────┬────┬────┬────┬────┬──     │
  │    tick tick tick tick tick tick tick tick tick tick        │
  │    0    1    2    3    4    5    6    7    8    9          │
  │    |←── 1ms ──→|  (configTICK_RATE_HZ = 1000)           │
  │                                                           │
  │  What happens each tick:                                 │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ 1. Increment xTickCount                        │      │
  │  │ 2. Check delayed task list for expired timeouts│      │
  │  │    → Move expired tasks to Ready list          │      │
  │  │ 3. Check time-slice for current priority       │      │
  │  │    → If expired, rotate to next same-prio task │      │
  │  │ 4. Call xApplicationTickHook() if configured   │      │
  │  │ 5. Set PendSV if context switch needed         │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Tick rate trade-offs:                                   │
  │  ┌───────────┬──────────────┬────────────────────┐      │
  │  │ Rate      │ Resolution   │ Overhead            │      │
  │  ├───────────┼──────────────┼────────────────────┤      │
  │  │ 100 Hz    │ 10ms         │ Very low (~0.1%)   │      │
  │  │ 1000 Hz   │ 1ms          │ Low (~1%)          │      │
  │  │ 10000 Hz  │ 100μs        │ Significant (~5%) │      │
  │  └───────────┴──────────────┴────────────────────┘      │
  │  Higher rate = better timing resolution but more         │
  │  CPU time spent in tick interrupt handler                │
  │                                                           │
  │  TickType_t: 16-bit or 32-bit (configUSE_16_BIT_TICKS)  │
  │  32-bit @ 1000Hz: wraps after 49.7 days                 │
  │  16-bit @ 1000Hz: wraps after 65.5 seconds!             │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Task Delays and Periodic Execution

```c
/* vTaskDelay: relative delay */
void vTask(void *pvParameters) {
    for (;;) {
        do_work();  /* Takes variable time */
        vTaskDelay(pdMS_TO_TICKS(100));
        /* Period = work_time + 100ms (variable!) */
    }
}

/* vTaskDelayUntil: absolute periodic execution */
void vPeriodicTask(void *pvParameters) {
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;) {
        do_work();  /* Takes variable time */
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(100));
        /* Period = exactly 100ms (constant) */
        /* xLastWakeTime auto-incremented by 100 ticks */
    }
}

/*
 * Timing diagram comparison:
 *
 * vTaskDelay(100ms):
 * |--work(5ms)--|---100ms delay---|--work(8ms)--|---100ms delay---|
 * |<----105ms period---->|        |<----108ms period---->|
 * Variable period!
 *
 * vTaskDelayUntil(100ms):
 * |--work(5ms)--|--95ms wait--|--work(8ms)--|--92ms wait--|
 * |<------100ms period------>|<------100ms period------>|
 * Constant period!
 *
 * What if work takes LONGER than period?
 * vTaskDelayUntil detects this and returns immediately
 * (no negative delay), but the next period is still
 * measured from the original time — it will "catch up"
 */

/* Zephyr equivalent */
void periodic_thread(void) {
    int64_t next = k_uptime_get();
    while (1) {
        do_work();
        next += 100;  /* 100ms period */
        k_sleep(K_TIMEOUT_ABS_MS(next));
    }
}
```

---

## 3. Software Timers

```c
/* FreeRTOS Software Timers */
/* Run by timer daemon task (prvTimerTask), NOT by ISR */

/* One-shot timer: fires once */
TimerHandle_t xOneShotTimer = xTimerCreate(
    "Timeout",                    /* Name */
    pdMS_TO_TICKS(5000),          /* Period: 5 seconds */
    pdFALSE,                      /* One-shot (not auto-reload) */
    (void *)0,                    /* Timer ID */
    vTimeoutCallback              /* Callback function */
);

/* Periodic timer: fires repeatedly */
TimerHandle_t xPeriodicTimer = xTimerCreate(
    "Heartbeat",
    pdMS_TO_TICKS(1000),          /* Every 1 second */
    pdTRUE,                       /* Auto-reload (periodic) */
    (void *)1,
    vHeartbeatCallback
);

/* Start timers */
xTimerStart(xOneShotTimer, 0);    /* Start immediately */
xTimerStart(xPeriodicTimer, 0);

/* Timer callbacks — run in timer daemon task context */
void vTimeoutCallback(TimerHandle_t xTimer) {
    /* WARNING: runs in timer task context, NOT the creating task */
    /* Use with care: don't block for long, shares timer task */
    uint32_t timerId = (uint32_t)pvTimerGetTimerID(xTimer);
    handle_timeout(timerId);
}

void vHeartbeatCallback(TimerHandle_t xTimer) {
    toggle_led();
    send_heartbeat_can_message();
}

/* Timer API from ISR */
void Button_IRQHandler(void) {
    BaseType_t xWoken = pdFALSE;
    /* Reset the timeout timer (restart countdown) */
    xTimerResetFromISR(xOneShotTimer, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

/*
 * Timer daemon task internals:
 *
 * ┌────────────────┐     ┌──────────────────────────┐
 * │ Timer Command  │     │ Timer Daemon Task         │
 * │ Queue          │────►│ (prvTimerTask)             │
 * │ (xTimerQueue)  │     │ Priority: configTIMER_TASK_PRIORITY │
 * └────────────────┘     │ Stack: configTIMER_TASK_STACK_DEPTH │
 *                         │                            │
 *  xTimerStart() ──►      │ Checks: sorted timer list │
 *  xTimerStop()  ──►      │ Fires: expired callbacks  │
 *  xTimerReset() ──►      │ One callback at a time     │
 *                         └──────────────────────────┘
 *
 * Timer commands are sent via queue (deferred execution)
 * Callbacks run serially — long callback delays others!
 * Timer resolution limited by tick rate
 */
```

---

## 4. High-Resolution Timing

```c
/*
 * When tick resolution (1ms) is not enough
 * Use hardware timers for sub-millisecond precision
 */

/* DWT Cycle Counter (Cortex-M3/M4/M7) — CPU cycle resolution */
/* Available without any timer peripheral */

static inline void dwt_init(void) {
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
    DWT->CYCCNT = 0;
    DWT->CTRL |= DWT_CTRL_CYCCNTENA_Msk;
}

static inline uint32_t dwt_get_cycles(void) {
    return DWT->CYCCNT;
}

/* Measure execution time with cycle-level precision */
uint32_t start = dwt_get_cycles();
critical_function();
uint32_t elapsed = dwt_get_cycles() - start;
/* elapsed = CPU cycles. At 168MHz: 1 cycle = 5.95ns */
float time_us = (float)elapsed / (SystemCoreClock / 1000000);

/* Hardware timer for precise delays */
/* Use TIM with one-shot mode for exact microsecond delays */
void delay_us(uint32_t us) {
    TIM2->CNT = 0;
    TIM2->ARR = us - 1;  /* Timer pre-scaled to 1MHz (1μs/tick) */
    TIM2->CR1 |= TIM_CR1_CEN | TIM_CR1_OPM;  /* One-pulse mode */
    while (TIM2->CR1 & TIM_CR1_CEN);  /* Wait for completion */
    /* WARNING: this busy-waits! Only for very short delays */
}

/* Combining tick time with hardware timer for high-res timestamp */
uint64_t get_timestamp_us(void) {
    /* Atomic read of tick + systick counter */
    uint32_t ticks, systick_val;
    do {
        ticks = xTaskGetTickCount();
        systick_val = SysTick->VAL;
        /* Re-read ticks to detect wraparound during read */
    } while (ticks != xTaskGetTickCount());

    uint32_t elapsed_in_tick = SysTick->LOAD - systick_val;
    uint32_t us_per_tick = 1000000 / configTICK_RATE_HZ;
    uint32_t sub_tick_us = (elapsed_in_tick * us_per_tick) / SysTick->LOAD;

    return (uint64_t)ticks * us_per_tick + sub_tick_us;
}
```

---

## 5. Watchdog Timers

```c
/*
 * Watchdog: hardware timer that resets system if not "fed"
 * Catches: infinite loops, deadlocks, task starvation
 */

/* Independent Watchdog (IWDG) — runs on separate clock */
void iwdg_init(uint32_t timeout_ms) {
    IWDG->KR = 0x5555;   /* Unlock registers */
    IWDG->PR = 4;        /* Prescaler /64 */
    IWDG->RLR = (timeout_ms * 40) / 64;  /* Reload value */
    IWDG->KR = 0xCCCC;   /* Start watchdog */
}

void iwdg_feed(void) {
    IWDG->KR = 0xAAAA;   /* Reload counter */
}

/*
 * RTOS watchdog patterns:
 *
 * Pattern 1: Single watchdog task
 * ┌────────────────────────────────────────────────┐
 * │ Each task reports "alive" to watchdog task      │
 * │ Watchdog task feeds IWDG only if ALL tasks     │
 * │ have reported within deadline                   │
 * └────────────────────────────────────────────────┘
 */

#define NUM_MONITORED_TASKS 4
static volatile uint32_t alive_flags = 0;

/* Each task calls this periodically */
void task_report_alive(uint32_t task_id) {
    taskENTER_CRITICAL();
    alive_flags |= (1 << task_id);
    taskEXIT_CRITICAL();
}

/* Watchdog supervisor task */
void vWatchdogTask(void *pvParameters) {
    for (;;) {
        vTaskDelay(pdMS_TO_TICKS(500));  /* Check every 500ms */

        uint32_t expected = (1 << NUM_MONITORED_TASKS) - 1;
        if ((alive_flags & expected) == expected) {
            iwdg_feed();  /* All tasks alive → feed watchdog */
            taskENTER_CRITICAL();
            alive_flags = 0;  /* Reset for next check cycle */
            taskEXIT_CRITICAL();
        }
        /* If any task didn't report: DON'T feed → reset! */
    }
}

/*
 * Pattern 2: Window watchdog (WWDG)
 * Must be fed within a specific time WINDOW
 * Too early = reset. Too late = reset.
 * Catches: tasks running too fast (tight loops)
 *
 *   ──┬────────────────────────────────────┬──
 *     │ too early   │  feed window  │ too   │
 *     │ = reset     │  (must feed   │ late  │
 *     │             │   here)       │ =reset│
 *   ──┴─────────────┴───────────────┴──────┴──
 */
```

---

## 6. Time Management Comparison

| Feature | FreeRTOS | Zephyr | QNX | VxWorks |
|---------|----------|--------|-----|---------|
| **Tick source** | SysTick / TIM | SysTick / custom | Timer interrupt | System clock |
| **Tick type** | TickType_t (32/16) | int64_t (ms or ticks) | struct timespec | UINT32 |
| **Resolution** | configTICK_RATE_HZ | CONFIG_SYS_CLOCK_TICKS_PER_SEC | nanoseconds | sysClkRateGet() |
| **SW timers** | Timer daemon task | k_timer (ISR callback!) | POSIX timer_create | wdCreate |
| **Periodic** | vTaskDelayUntil | k_timer_start | timer_settime | taskDelay |
| **Tickless** | configUSE_TICKLESS_IDLE | Always (if idle) | Dynamic tick | VxWorks 7+ |
| **High-res** | DWT + SysTick | k_cycle_get_32() | ClockCycles() | sysTimestamp() |

---

## Interview Questions

**Q1: What is the difference between vTaskDelay and vTaskDelayUntil?**
**A:** `vTaskDelay(100)` blocks for 100 ticks RELATIVE to when it's called. Period = execution_time + delay_time (variable). `vTaskDelayUntil(&lastWake, 100)` blocks until an ABSOLUTE tick count, computed from the last wake time. Period = exactly 100 ticks (constant), regardless of execution time. Use `vTaskDelayUntil` for periodic tasks (sensor sampling, control loops) where consistent timing matters. Internal: `vTaskDelayUntil` increments `lastWake` by the period and calculates how many ticks to actually delay. If execution overran the period, it returns immediately without negative delay.

**Q2: How do FreeRTOS software timers work internally?**
**A:** A dedicated timer daemon task (`prvTimerTask`) runs at `configTIMER_TASK_PRIORITY`. Timer API calls (`xTimerStart`, `xTimerStop`, `xTimerReset`) send command messages to a timer command queue (`xTimerQueue`). The daemon task processes these commands and maintains a sorted list of active timers ordered by expiry time. It blocks on the queue with a timeout equal to the time until the next timer expires. When a timer expires, the daemon calls the timer's callback function in its own task context — NOT in ISR context. Implication: timer callbacks run serially, so a long callback delays all other timers. Timer resolution is limited to tick rate. Set timer task priority high if timer accuracy matters.

**Q3: How would you achieve sub-microsecond timing on Cortex-M4?**
**A:** Use the DWT (Data Watchpoint and Trace) cycle counter: `DWT->CYCCNT` counts CPU cycles at core clock frequency. At 168MHz, resolution = ~6ns. Enable via `CoreDebug->DEMCR |= TRCENA; DWT->CTRL |= CYCCNTENA`. For timestamps: combine `xTaskGetTickCount()` (ms) with `SysTick->VAL` (sub-tick fraction) — read atomically to avoid race between tick increment and SysTick reload. For precise delays: use a hardware timer (TIM2-TIM5) prescaled to 1MHz in one-pulse mode — but this busy-waits. For scheduling with sub-tick precision: use hardware timer compare interrupt that fires at the exact needed microsecond, rather than relying on the RTOS tick.

---

## Summary

- Tick system: periodic interrupt (typ 1ms) drives time keeping, delays, timeouts
- vTaskDelayUntil for periodic tasks, vTaskDelay for relative delays
- Software timers: run by daemon task, not ISR; limited to tick resolution
- High-resolution: DWT cycle counter (ns), hardware timers (μs), SysTick fraction
- Watchdog: hardware reset if tasks miss deadlines; supervisor pattern monitors all tasks
- Tickless idle: suppress ticks during idle, compensate tick count on wake
- 32-bit tick @ 1000Hz wraps in ~49 days; use 64-bit for long-running systems

---

[Previous Chapter: Real-Time Memory Patterns ←](Chapter_14_RT_Memory.md) | [Next Chapter: Device Drivers →](Chapter_16_Device_Drivers.md)
