# Chapter 25: Performance Optimization

## Learning Goals
- Master latency measurement and reduction techniques
- Learn interrupt optimization and ISR best practices
- Understand memory optimization for constrained systems
- Know compiler optimization flags and their effects on real-time behavior

---

## 1. Latency Analysis and Reduction

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Latency Components (event → response):                  │
  │                                                           │
  │  ┌──────────────────────────────────────────────────┐   │
  │  │ Hardware interrupt latency        │  12 cycles   │   │
  │  │ (pipeline flush + stacking)       │  (Cortex-M4) │   │
  │  ├───────────────────────────────────┼──────────────┤   │
  │  │ ISR prologue (save context)       │  5-20 cycles │   │
  │  ├───────────────────────────────────┼──────────────┤   │
  │  │ ISR body (work + FromISR)         │  variable    │   │
  │  ├───────────────────────────────────┼──────────────┤   │
  │  │ PendSV context switch             │  30-80 cycles│   │
  │  ├───────────────────────────────────┼──────────────┤   │
  │  │ Task restoration + execution      │  5-20 cycles │   │
  │  └───────────────────────────────────┴──────────────┘   │
  │                                                           │
  │  Total worst-case: 50-150 cycles @ 168MHz = 0.3-0.9us  │
  │                                                           │
  │  Latency killers:                                        │
  │  1. Critical sections (BASEPRI mask) — blocks interrupts│
  │  2. Long ISRs (doing work instead of deferring)         │
  │  3. Priority inversion (mutex held by low-prio task)    │
  │  4. Cache misses (code/data not in cache → 10-100x)    │
  │  5. Flash wait states (run hot paths from SRAM/TCM)     │
  └──────────────────────────────────────────────────────────┘
```

```c
/* Measuring task response latency with DWT */
volatile uint32_t isr_timestamp;
volatile uint32_t task_timestamp;

void EXTI0_IRQHandler(void) {
    isr_timestamp = DWT->CYCCNT;
    EXTI->PR = EXTI_PR_PR0;  /* Clear pending */

    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    vTaskNotifyGiveFromISR(hResponseTask, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

void vResponseTask(void *pvParameters) {
    for (;;) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
        task_timestamp = DWT->CYCCNT;

        uint32_t latency_cycles = task_timestamp - isr_timestamp;
        float latency_us = (float)latency_cycles / (SystemCoreClock / 1000000);

        /* Track min/max/avg for analysis */
        update_latency_stats(latency_us);
    }
}
```

---

## 2. Interrupt Optimization

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ISR Best Practices:                                     │
  │                                                           │
  │  BAD ISR (blocks for too long):                          │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ void UART_IRQHandler(void) {                 │       │
  │  │     char c = UART->DR;                       │       │
  │  │     process_command(c);  // ← TOO SLOW!     │       │
  │  │     update_display();    // ← NEVER IN ISR! │       │
  │  │     log_to_flash(c);    // ← BLOCKS!        │       │
  │  │ }                                            │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                           │
  │  GOOD ISR (minimal work, defer to task):                 │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ void UART_IRQHandler(void) {                 │       │
  │  │     char c = UART->DR;                       │       │
  │  │     xQueueSendFromISR(rxQueue, &c, &woken);  │       │
  │  │     portYIELD_FROM_ISR(woken);               │       │
  │  │ }                                            │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                           │
  │  ISR Time Budget:                                        │
  │  - Hard real-time: < 1us (< 168 cycles @ 168MHz)       │
  │  - Typical target: < 5us                                │
  │  - Maximum acceptable: < 50us                           │
  │  - Anything longer → defer to task                      │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Memory Optimization

```c
/*
 * Code size reduction techniques:
 *
 * 1. Compiler optimization flags:
 *    -Os  : Optimize for size (usually best for MCU)
 *    -Oz  : Clang aggressive size optimization
 *    -flto: Link-Time Optimization (removes unused code)
 *    --gc-sections + -ffunction-sections -fdata-sections:
 *      Linker removes unused functions/data
 *
 * 2. ARM Thumb vs ARM instructions:
 *    -mthumb: 16-bit instructions (30-40% smaller)
 *    Cortex-M always uses Thumb-2
 *
 * 3. Compiler intrinsics instead of stdlib:
 *    __builtin_memcpy, __builtin_clz → no library linkage
 */

/* Stack size optimization */
/*
 * Stack usage analysis:
 * 1. GCC: -fstack-usage → generates .su files
 *    Each function reports max stack usage
 *
 * 2. Static analysis: worst-case call chain
 *    main() → task_func() → process() → parse()
 *    32      + 128        + 64       + 256 = 480 bytes
 *    Add 20% margin + ISR stacking = 576 bytes → round to 640
 *
 * 3. Runtime: paint stack with 0xDEADBEEF, check HWM
 */

/* FreeRTOS: check stack high water mark at runtime */
void vMonitorStacks(void *pvParameters) {
    for (;;) {
        UBaseType_t hwm;

        hwm = uxTaskGetStackHighWaterMark(hSensorTask);
        if (hwm < 32) {  /* Less than 32 words remaining */
            printf("WARNING: SensorTask stack low: %lu words\n", hwm);
        }

        hwm = uxTaskGetStackHighWaterMark(hMotorTask);
        if (hwm < 32) {
            printf("WARNING: MotorTask stack low: %lu words\n", hwm);
        }

        vTaskDelay(pdMS_TO_TICKS(10000));
    }
}

/* RAM usage optimization */
/*
 * 1. Use smallest data types:
 *    uint8_t counter;           // not uint32_t
 *    int16_t temperature;       // not int32_t
 *
 * 2. Pack structures:
 *    __attribute__((packed)) — saves padding bytes
 *    But: unaligned access penalty on some architectures
 *
 * 3. Const data in flash (not copied to RAM):
 *    const char msg[] = "Hello"; // stays in flash
 *    static const uint16_t lut[256] = {...}; // flash LUT
 *
 * 4. Overlay buffers (reuse same RAM for different phases):
 *    union { uint8_t rx_buf[512]; uint8_t tx_buf[512]; };
 *    Only one used at a time
 */
```

---

## 4. Cache Optimization for Cortex-M7/A

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Cortex-M7: I-Cache (16KB) + D-Cache (16KB)             │
  │                                                           │
  │  Cache miss penalty: ~20 cycles (vs 0-1 for hit)        │
  │                                                           │
  │  Optimization strategies:                                │
  │                                                           │
  │  1. Place hot code in ITCM (zero wait state):           │
  │     __attribute__((section(".itcm"))) void fast_isr() {} │
  │     Linker copies from flash to ITCM on startup         │
  │                                                           │
  │  2. Place hot data in DTCM (zero wait state):           │
  │     __attribute__((section(".dtcm"))) uint32_t buf[256];│
  │     Perfect for DMA buffers, lookup tables              │
  │                                                           │
  │  3. Cache coherency with DMA:                           │
  │     DMA reads from RAM, not cache → stale data          │
  │     Solution:                                            │
  │     SCB_CleanDCache_by_Addr(buf, size);  // before DMA TX│
  │     SCB_InvalidateDCache_by_Addr(buf, size); // after RX│
  │     Or: place DMA buffers in non-cacheable region       │
  │                                                           │
  │  4. Data locality:                                      │
  │     Keep related data in same cache line (32 bytes)     │
  │     Align critical structures to cache line boundary    │
  │     __attribute__((aligned(32))) struct { ... } sensor; │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Compiler Optimization Levels

```
  ┌───────┬──────────────────────────────┬───────────────────┐
  │ Flag  │ Effect                       │ RTOS Impact       │
  ├───────┼──────────────────────────────┼───────────────────┤
  │ -O0   │ No optimization              │ Debuggable, slow  │
  │       │                              │ Large code size   │
  ├───────┼──────────────────────────────┼───────────────────┤
  │ -O1   │ Basic optimizations          │ Good for debug    │
  │       │ (reduced code, some inline)  │ Mostly debuggable │
  ├───────┼──────────────────────────────┼───────────────────┤
  │ -O2   │ Full optimizations           │ Production builds │
  │       │ (loop unroll, inline, LICM)  │ Faster, may alter │
  │       │                              │ timing behavior   │
  ├───────┼──────────────────────────────┼───────────────────┤
  │ -Os   │ Optimize for size            │ Best for MCU      │
  │       │ (like -O2, less inlining)    │ Fits in flash     │
  ├───────┼──────────────────────────────┼───────────────────┤
  │ -O3   │ Aggressive (vectorize,       │ Rarely used on MCU│
  │       │  heavy inline)               │ Code bloat risk   │
  ├───────┼──────────────────────────────┼───────────────────┤
  │ -Ofast│ -O3 + fast-math             │ DANGEROUS: breaks │
  │       │ (non-IEEE float)             │ NaN/Inf checks    │
  ├───────┼──────────────────────────────┼───────────────────┤
  │ -flto │ Link-Time Optimization       │ Cross-file inline │
  │       │ Whole-program analysis        │ Best dead-code    │
  │       │                              │ removal           │
  └───────┴──────────────────────────────┴───────────────────┘

  WARNING: Optimization can reorder code and eliminate
  "unnecessary" reads. Use volatile for hardware registers
  and shared variables. Use memory barriers for DMA/multi-core.
```

---

## 6. CPU Load Monitoring

```c
/* FreeRTOS CPU load measurement using idle hook */
static volatile uint32_t idle_counter;
static volatile uint32_t total_counter;
static volatile float cpu_load_percent;

/* Called every tick */
void vApplicationTickHook(void) {
    total_counter++;
}

/* Called continuously when no task is ready */
void vApplicationIdleHook(void) {
    idle_counter++;
}

/* Calculate CPU load periodically */
void vLoadMonitorTask(void *pvParameters) {
    for (;;) {
        vTaskDelay(pdMS_TO_TICKS(1000));

        /* CPU load = 1 - (idle_time / total_time) */
        if (total_counter > 0) {
            cpu_load_percent = 100.0f *
                (1.0f - (float)idle_counter / (float)total_counter);
        }
        printf("CPU Load: %.1f%%\n", cpu_load_percent);

        idle_counter = 0;
        total_counter = 0;
    }
}

/*
 * More accurate: use DWT cycle counter
 * Measure cycles spent in idle task vs total cycles
 * DWT->CYCCNT gives 32-bit cycle count (wraps at ~25s @ 168MHz)
 */
```

---

## Interview Questions

**Q1: How do you reduce interrupt latency in an RTOS system?**
**A:** Interrupt latency = time from hardware event to ISR execution. Key reductions: (1) Minimize critical section duration — use `taskENTER_CRITICAL` only around 2-3 instructions, never around I/O or blocking calls. (2) Use BASEPRI instead of PRIMASK — BASEPRI masks only below a threshold, keeping higher-priority ISRs responsive (FreeRTOS uses `configMAX_SYSCALL_INTERRUPT_PRIORITY`). (3) Keep ISRs short — do minimum work (read register, clear flag, queue event), defer processing to tasks via `xTaskNotifyFromISR()` or `xQueueSendFromISR()`. (4) Place ISR code in ITCM/SRAM (zero wait state) instead of flash. (5) Tail-chaining and late-arrival on Cortex-M reduce back-to-back ISR overhead (6 cycles vs 12). (6) Avoid shared resources in ISRs — use lock-free queues or task notifications instead of mutexes (which can't be used in ISRs).

**Q2: How do you optimize code size on a memory-constrained MCU?**
**A:** (1) Compiler: `-Os` (size-optimized), `-flto` (link-time optimization removes unused code across files), `-ffunction-sections -fdata-sections` with `--gc-sections` (linker removes unreferenced sections). (2) Use Thumb-2 instructions (`-mthumb`) — 30-40% smaller than ARM. (3) Minimize FreeRTOS: disable unused features in FreeRTOSConfig.h (`configUSE_MUTEXES 0`, `configUSE_COUNTING_SEMAPHORES 0`, `configUSE_TRACE_FACILITY 0`). (4) Replace printf/sprintf with minimal versions (nano newlib: `--specs=nano.specs`). (5) Use `const` for lookup tables — stays in flash, not copied to RAM. (6) Avoid C++ exceptions and RTTI (`-fno-exceptions -fno-rtti`). (7) Use `arm-none-eabi-size` to analyze text/data/bss per object file, find largest contributors.

---

## Summary

- Interrupt latency on Cortex-M4: ~12 cycles hardware + ISR + context switch = 0.3-1us @ 168MHz
- ISR optimization: keep under 5us, defer heavy work to tasks via FromISR APIs
- Memory: `-Os -flto --gc-sections`, Thumb-2, const data in flash, minimal stack sizes
- Cache: place hot code in ITCM, data in DTCM, manage DMA coherency explicitly
- CPU load: idle hook counter method or DWT cycle counter for precision
- Compiler flags affect timing — always profile after changing optimization level

---

[Previous Chapter: Debugging ←](Chapter_24_Debugging.md) | [Next Chapter: Porting →](Chapter_26_Porting.md)
