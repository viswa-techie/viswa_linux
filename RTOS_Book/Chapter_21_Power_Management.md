# Chapter 21: RTOS Power Management

## Learning Goals
- Understand power management in resource-constrained RTOS systems
- Master tickless idle and low-power modes
- Learn peripheral power gating and clock management
- Know power-aware task design patterns

---

## 1. MCU Power Modes

```
  ARM Cortex-M Power Modes
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Mode          │ CPU │ Periph │ SRAM │ Wake Source       │
  │  ──────────────┼─────┼────────┼──────┼──────────────── │
  │  Run           │ ON  │ ON     │ ON   │ N/A (active)     │
  │  Sleep (WFI)   │ OFF │ ON     │ ON   │ Any interrupt    │
  │  Deep Sleep    │ OFF │ Selective│ ON  │ EXTI, RTC, LPUART│
  │  Standby       │ OFF │ OFF    │ OFF* │ WKUP pin, RTC    │
  │  Shutdown      │ OFF │ OFF    │ OFF  │ WKUP pin only    │
  │                                                           │
  │  *Some MCUs retain backup SRAM in standby               │
  │                                                           │
  │  Power consumption (STM32L4 @ 80MHz example):           │
  │  ┌─────────────┬──────────────────────────┐             │
  │  │ Run          │ ~10 mA                    │             │
  │  │ Sleep        │ ~1 mA                     │             │
  │  │ Stop 2       │ ~1 μA                     │             │
  │  │ Standby      │ ~300 nA                   │             │
  │  │ Shutdown     │ ~30 nA                    │             │
  │  └─────────────┴──────────────────────────┘             │
  │                                                           │
  │  Wake latency:                                           │
  │  Sleep → Run:    ~1 μs (very fast)                       │
  │  Stop  → Run:    ~5-100 μs (PLL restart)                │
  │  Standby → Run:  ~ms (full reboot, SRAM lost)          │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. FreeRTOS Tickless Idle Implementation

```c
/* configUSE_TICKLESS_IDLE = 1 in FreeRTOSConfig.h */

/* FreeRTOS calls this when idle task determines sleep is possible */
void vPortSuppressTicksAndSleep(TickType_t xExpectedIdleTime) {
    /* 1. Calculate how many ticks until next task wakes */
    uint32_t ulTimerCountsForOneTick = (SystemCoreClock / configTICK_RATE_HZ);
    uint32_t ulReloadValue = ulTimerCountsForOneTick * xExpectedIdleTime;

    /* 2. Ensure minimum sleep time is worthwhile */
    if (xExpectedIdleTime < configEXPECTED_IDLE_TIME_BEFORE_SLEEP) {
        return;  /* Too short — not worth the overhead */
    }

    /* 3. Stop SysTick */
    SysTick->CTRL &= ~SysTick_CTRL_ENABLE_Msk;

    /* 4. Reprogram SysTick for extended period */
    SysTick->LOAD = ulReloadValue - 1;
    SysTick->VAL  = 0;
    SysTick->CTRL |= SysTick_CTRL_ENABLE_Msk;

    /* 5. Enter critical section to prevent interrupt race */
    __disable_irq();
    __dsb(0xF);
    __isb(0xF);

    /* 6. Check if a task was unblocked while preparing */
    if (eTaskConfirmSleepModeStatus() != eAbortSleep) {
        /* 7. Enter low-power mode */
        __wfi();  /* Wait For Interrupt — CPU sleeps */
        /* === CPU SLEEPS HERE === */
        /* Wakes on: any enabled interrupt */
    }

    /* 8. Re-enable interrupts */
    __enable_irq();

    /* 9. Read how long we actually slept */
    uint32_t ulSleepCount = ulReloadValue - SysTick->VAL;
    uint32_t ulCompleteTickPeriods = ulSleepCount / ulTimerCountsForOneTick;

    /* 10. Restore normal SysTick operation */
    SysTick->LOAD = ulTimerCountsForOneTick - 1;
    SysTick->VAL  = 0;

    /* 11. Compensate tick count for sleep duration */
    vTaskStepTick(ulCompleteTickPeriods);
}
```

---

## 3. Power-Aware Task Design

```c
/* Peripheral clock gating — disable clocks when not in use */
void vSensorTask(void *pvParameters) {
    for (;;) {
        /* Enable peripheral clock before use */
        RCC->APB1ENR |= RCC_APB1ENR_I2C1EN;
        vTaskDelay(pdMS_TO_TICKS(1));  /* Wait for clock stabilization */

        /* Read sensor */
        int16_t temp = i2c_read_sensor(I2C1, SENSOR_ADDR);
        xQueueSend(xTempQueue, &temp, 0);

        /* Disable peripheral clock when done */
        RCC->APB1ENR &= ~RCC_APB1ENR_I2C1EN;

        /* Sleep until next measurement */
        vTaskDelay(pdMS_TO_TICKS(10000));  /* 10-second interval */
    }
}

/* Zephyr power management */
/* Device runtime PM: automatically manage device power */
/*
 * pm_device_action_run(dev, PM_DEVICE_ACTION_SUSPEND);
 * pm_device_action_run(dev, PM_DEVICE_ACTION_RESUME);
 *
 * Or use device runtime PM for automatic:
 * pm_device_runtime_get(dev);   // Resume + ref count++
 * // Use device
 * pm_device_runtime_put(dev);   // ref count--; suspend if 0
 */
```

---

## Interview Questions

**Q1: How does tickless idle save power?**
**A:** Normal RTOS operation: SysTick fires every 1ms, waking the CPU even when idle. With tickless idle, when the idle task runs and the next task wake-up is N ticks away, the SysTick is reprogrammed for N ticks and the CPU enters WFI (sleep). Instead of 1000 wakeups/second, the CPU might wake only 10 times/second. On wake, the tick count is compensated for the sleep duration (`vTaskStepTick`). Power savings: from ~10mA (run) to ~1μA (stop mode) during idle. Critical detail: must check `eTaskConfirmSleepModeStatus` after disabling interrupts to ensure no task was unblocked during the preparation — if one was, abort sleep and run normally.

**Q2: What are the trade-offs between different low-power modes?**
**A:** Deeper sleep = more power savings but: (1) Longer wake latency (Sleep: ~1μs, Stop: ~100μs, Standby: ~ms). (2) More context lost (Standby loses SRAM, must reinitialize). (3) Fewer wake sources (Standby: only WKUP pin and RTC). Trade-off: choose the deepest mode whose wake latency meets your real-time deadline and whose wake sources cover your event inputs. Example: IoT sensor (measure every 10s): Standby mode okay — save μA, tolerate ms wake time. Motor controller (respond in 10μs): Sleep mode only — can't afford Stop mode's 100μs wake time.

---

## Summary

- MCU power modes: Run → Sleep → Deep Sleep → Standby → Shutdown
- Tickless idle: reprogram SysTick, enter WFI, compensate ticks on wake
- Peripheral clock gating: disable unused peripheral clocks to reduce active power
- Deeper sleep = more savings but longer wake latency and more context lost
- Power-aware design: batch processing, event-driven wake, minimize active time

---

[Previous Chapter: RT Communication Protocols ←](Chapter_20_RT_Protocols.md) | [Next Chapter: Safety and Reliability →](Chapter_22_Safety.md)
