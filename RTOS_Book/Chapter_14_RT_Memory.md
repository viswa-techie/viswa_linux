# Chapter 14: Real-Time Memory Patterns

## Learning Goals
- Understand memory-constrained design patterns for RTOS
- Learn stack memory analysis and optimization
- Know lock-free data structures for RT systems
- Master DMA buffer management patterns
- Understand memory-mapped I/O in RTOS context

---

## 1. Stack Memory Analysis

```c
/*
 * Static stack analysis: determine worst-case stack usage
 * at compile time without running the system
 */

/*
 * Call tree analysis example:
 *
 * vMotorTask()           ← 48 bytes locals
 *   ├── read_encoder()   ← 24 bytes
 *   ├── pid_compute()    ← 64 bytes
 *   │   └── sqrt()       ← 32 bytes (library)
 *   └── set_pwm()        ← 16 bytes
 *
 * Deepest path: vMotorTask → pid_compute → sqrt
 * Stack = 48 + 64 + 32 = 144 bytes (locals only)
 *  + context save: 64 bytes (hw + sw on Cortex-M4)
 *  + ISR nesting: 2 levels × 64 bytes = 128 bytes
 *  = 336 bytes minimum
 *  + 30% margin = 437 → round to 512 bytes (128 words)
 */

/* GCC stack usage analysis */
/* Compile with: -fstack-usage → generates .su files */
/*
 * motor_task.su:
 * motor_task.c:10:6:vMotorTask  48  static
 * motor_task.c:50:6:pid_compute 64  static
 * motor_task.c:80:6:read_encoder 24 static
 *
 * "static" = exact known usage
 * "dynamic" = uses alloca() or VLA (avoid in RTOS!)
 * "bounded" = bounded dynamic allocation
 */

/* Runtime stack monitoring */
void vCheckStacks(void *pvParameters) {
    for (;;) {
        /* Check all tasks' stack high water marks */
        TaskStatus_t taskStatus[10];
        UBaseType_t count = uxTaskGetSystemState(
            taskStatus, 10, NULL);

        for (UBaseType_t i = 0; i < count; i++) {
            UBaseType_t hwm = taskStatus[i].usStackHighWaterMark;
            if (hwm < 20) {  /* Less than 20 words free */
                /* WARNING: task nearly overflowed */
                log_warning("Stack low: %s (free=%u words)",
                    taskStatus[i].pcTaskName, hwm);
            }
        }
        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}
```

---

## 2. Lock-Free Data Structures

```c
/*
 * Lock-free ring buffer — no mutex needed
 * Safe for single-producer, single-consumer (SPSC)
 * Used for ISR-to-task data passing without RTOS API
 */

#define RING_SIZE 256  /* Must be power of 2 */
#define RING_MASK (RING_SIZE - 1)

typedef struct {
    volatile uint32_t head;  /* Written by producer */
    volatile uint32_t tail;  /* Written by consumer */
    uint8_t buffer[RING_SIZE];
} RingBuffer_t;

/* Producer (ISR): */
static inline bool ring_put(RingBuffer_t *rb, uint8_t byte) {
    uint32_t next = (rb->head + 1) & RING_MASK;
    if (next == rb->tail) {
        return false;  /* Full */
    }
    rb->buffer[rb->head] = byte;
    __DMB();  /* Data Memory Barrier — ensure write is visible */
    rb->head = next;
    return true;
}

/* Consumer (Task): */
static inline bool ring_get(RingBuffer_t *rb, uint8_t *byte) {
    if (rb->head == rb->tail) {
        return false;  /* Empty */
    }
    *byte = rb->buffer[rb->tail];
    __DMB();
    rb->tail = (rb->tail + 1) & RING_MASK;
    return true;
}

/*
 * Why this works without locks:
 * 1. head is only written by producer
 * 2. tail is only written by consumer
 * 3. Reads of the other's index only see monotonically
 *    increasing values
 * 4. DMB ensures memory ordering between data write
 *    and index update
 * 5. Power-of-2 size enables & mask instead of % (faster)
 *
 * NOT safe for: multiple producers or multiple consumers
 * For MPMC: use queues with proper locking
 */

/* Lock-free atomic flag (ISR-to-task signaling) */
static volatile uint32_t event_flags = 0;

/* ISR: set flag */
void ISR_Handler(void) {
    __atomic_or_fetch(&event_flags, EVENT_DATA_READY, __ATOMIC_RELEASE);
}

/* Task: check and clear flag */
void vTask(void *p) {
    for (;;) {
        uint32_t flags = __atomic_exchange_n(&event_flags, 0, __ATOMIC_ACQUIRE);
        if (flags & EVENT_DATA_READY) {
            process_data();
        }
        vTaskDelay(1);
    }
}
```

---

## 3. DMA Buffer Management

```c
/*
 * DMA double-buffering pattern
 * While DMA fills one buffer, task processes the other
 */

#define DMA_BUF_SIZE 512

/* Place in non-cacheable region or use cache maintenance */
static uint8_t __attribute__((aligned(32)))
    dma_buf_a[DMA_BUF_SIZE];
static uint8_t __attribute__((aligned(32)))
    dma_buf_b[DMA_BUF_SIZE];

static volatile uint8_t *active_buf = dma_buf_a;
static volatile uint8_t *process_buf = dma_buf_b;

void DMA_TransferComplete_IRQHandler(void) {
    BaseType_t xWoken = pdFALSE;

    /* Swap buffers */
    volatile uint8_t *tmp = active_buf;
    active_buf = process_buf;
    process_buf = tmp;

    /* Restart DMA into new active buffer */
    DMA1_Channel1->CMAR = (uint32_t)active_buf;
    DMA1_Channel1->CNDTR = DMA_BUF_SIZE;
    DMA1_Channel1->CCR |= DMA_CCR_EN;

    /* Notify processing task */
    vTaskNotifyGiveFromISR(xDmaTaskHandle, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

void vDmaProcessTask(void *pvParameters) {
    for (;;) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

        /* Cache invalidate if using cached memory (Cortex-M7) */
        SCB_InvalidateDCache_by_Addr((void *)process_buf, DMA_BUF_SIZE);

        /* Process the completed buffer while DMA fills the other */
        process_adc_samples((uint8_t *)process_buf, DMA_BUF_SIZE);
    }
}

/*
 * Cache coherency with DMA (Cortex-M7 with D-cache):
 *
 * Problem: CPU cache and DMA see different memory
 *
 * Solution 1: Place DMA buffers in non-cacheable MPU region
 *   MPU region attribute: TEX=001, C=0, B=0 (uncacheable)
 *
 * Solution 2: Cache maintenance operations
 *   Before DMA TX: SCB_CleanDCache_by_Addr() (flush to RAM)
 *   After DMA RX:  SCB_InvalidateDCache_by_Addr() (discard cache)
 *
 * Solution 3: Use TCM (Tightly Coupled Memory)
 *   Zero-wait-state, not cached, DMA-accessible
 *   Best for performance-critical buffers
 */
```

---

## 4. Memory-Mapped I/O Patterns

```c
/*
 * Accessing hardware registers safely in RTOS
 */

/* Register definition with volatile */
typedef struct {
    volatile uint32_t CR;     /* Control register */
    volatile uint32_t SR;     /* Status register */
    volatile uint32_t DR;     /* Data register */
    volatile uint32_t BRR;    /* Baud rate register */
} UART_TypeDef;

#define UART1  ((UART_TypeDef *)0x40011000)

/* Problem: multiple tasks accessing same peripheral */
/* Solution: peripheral mutex */

static SemaphoreHandle_t xSpiMutex;

int spi_transfer(uint8_t *tx, uint8_t *rx, size_t len) {
    if (xSemaphoreTake(xSpiMutex, pdMS_TO_TICKS(100)) != pdTRUE) {
        return -1;  /* Timeout */
    }

    /* Safe: exclusive access to SPI peripheral */
    SPI1->CR1 |= SPI_CR1_SPE;   /* Enable */
    for (size_t i = 0; i < len; i++) {
        SPI1->DR = tx[i];
        while (!(SPI1->SR & SPI_SR_RXNE));
        rx[i] = SPI1->DR;
    }
    SPI1->CR1 &= ~SPI_CR1_SPE;  /* Disable */

    xSemaphoreGive(xSpiMutex);
    return 0;
}

/*
 * Register access patterns:
 *
 * Read-modify-write (DANGEROUS without protection):
 *   GPIOA->ODR |= (1 << 5);   // Set bit 5
 *   // If ISR modifies ODR between read and write → bug!
 *
 * Safe alternatives:
 *   1. BSRR register (atomic set/reset):
 *      GPIOA->BSRR = (1 << 5);     // Atomic set bit 5
 *      GPIOA->BSRR = (1 << 21);    // Atomic reset bit 5
 *   2. Critical section:
 *      taskENTER_CRITICAL();
 *      GPIOA->ODR |= (1 << 5);
 *      taskEXIT_CRITICAL();
 *   3. Bit-banding (Cortex-M3/M4):
 *      Atomic bit-level access to SRAM and peripheral registers
 */
```

---

## 5. Memory Placement Strategies

```
  Placing Data in Optimal Memory Regions
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Memory Region          Speed      Use For               │
  │  ──────────────────────  ─────────  ──────────────────── │
  │  ITCM (Instruction TCM) 0 wait     Critical code (ISR)  │
  │  DTCM (Data TCM)         0 wait     Task stacks, hot data│
  │  SRAM (internal)         0-1 wait   General data, heap   │
  │  SDRAM (external)        3-10 wait  Large buffers, images│
  │  Flash                   0-2 wait   Code, const data     │
  │  CCM (Core-Coupled Mem)  0 wait     Stack, DMA-excluded  │
  │                                                           │
  │  Linker script placement:                                │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ /* Place ISR stack in DTCM for fastest access */│      │
  │  │ .isr_stack (NOLOAD) : {                         │      │
  │  │     . = ALIGN(8);                               │      │
  │  │     *(.isr_stack)                               │      │
  │  │ } > DTCM                                        │      │
  │  │                                                  │      │
  │  │ /* Place DMA buffers in non-cacheable SRAM */    │      │
  │  │ .dma_buffers (NOLOAD) : {                       │      │
  │  │     . = ALIGN(32);                              │      │
  │  │     *(.dma_buffers)                             │      │
  │  │ } > SRAM_NOCACHE                                │      │
  │  │                                                  │      │
  │  │ /* Large data in external SDRAM */               │      │
  │  │ .ext_data : {                                   │      │
  │  │     *(.ext_data)                                │      │
  │  │ } > SDRAM                                       │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  GCC Section attributes:                                 │
  │  __attribute__((section(".dtcm")))                       │
  │      static uint32_t fast_data;                          │
  │  __attribute__((section(".dma_buffers")))                 │
  │      static uint8_t dma_buf[1024];                       │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How would you manage DMA buffers on a Cortex-M7 with data cache?**
**A:** Three approaches: (1) Place DMA buffers in non-cacheable MPU region — configure MPU region with TEX=001, C=0, B=0. Simplest but slower CPU access. (2) Use cache maintenance — before DMA TX: `SCB_CleanDCache_by_Addr()` to flush dirty cache lines to RAM so DMA reads correct data. After DMA RX: `SCB_InvalidateDCache_by_Addr()` to discard stale cache entries so CPU reads new DMA data. Buffers must be 32-byte aligned (cache line size). (3) Place in DTCM — zero-wait-state, not cached, DMA-accessible. Best performance but limited size. The cache maintenance approach is most flexible but error-prone — forgetting one operation causes intermittent data corruption that's very hard to debug.

**Q2: Explain why lock-free ring buffers work without mutexes for SPSC.**
**A:** Single-producer single-consumer (SPSC) ring buffer works without locks because: (1) head index is only written by the producer, (2) tail index is only written by the consumer, (3) each side reads the other's index but never writes it, (4) index updates are naturally atomic (single 32-bit write on ARM). (5) Memory barriers (DMB) ensure the data write is visible before the index update. The consumer sees a consistent snapshot: either the old data (index not yet updated) or the new data (index updated after data). There's no intermediate state where the index is updated but data isn't written. This doesn't work for multiple producers/consumers because concurrent head (or tail) updates would race without atomic compare-and-swap.

**Q3: What are the trade-offs of placing task stacks in TCM vs SRAM?**
**A:** TCM (Tightly Coupled Memory): 0 wait-state, deterministic access time (no cache hit/miss variability), excellent for real-time tasks where context switch latency matters. But TCM is typically small (64-128KB) — can't fit many stacks. Not cached (no coherency issues with DMA). SRAM: larger (up to 512KB+), may have 1+ wait states, suitable for most tasks. If cached (Cortex-M7), context switch time becomes variable due to cache misses when restoring another task's stack. Strategy: place highest-priority task stacks in TCM, others in SRAM. ISR stack (MSP) should always be in TCM for minimum interrupt latency.

---

## Summary

- Stack analysis: static (GCC -fstack-usage) + runtime (high water mark monitoring)
- Lock-free ring buffers: SPSC only, O(1), no RTOS API overhead, DMB for ordering
- DMA double-buffering: swap buffers on transfer complete, process while DMA fills
- Cache coherency: non-cacheable MPU region, cache maintenance, or TCM placement
- Memory-mapped I/O: volatile registers, peripheral mutex, atomic BSRR for GPIO
- Memory placement: TCM for time-critical, SRAM for general, SDRAM for large buffers
- Linker script + section attributes control memory placement

---

[Previous Chapter: Memory Management ←](Chapter_13_Memory_Management.md) | [Next Chapter: Time Management →](Chapter_15_Time_Management.md)
