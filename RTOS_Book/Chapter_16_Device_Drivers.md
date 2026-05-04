# Chapter 16: Device Drivers in RTOS

## Learning Goals
- Understand RTOS device driver architecture patterns
- Learn interrupt-driven vs polled vs DMA driver designs
- Master driver-RTOS integration (ISR + task + IPC)
- Know BSP (Board Support Package) structure
- Compare driver models across FreeRTOS, Zephyr, QNX, VxWorks

---

## 1. RTOS Driver Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Application Layer                                       │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │ Application Tasks                                │     │
  │  │ (vSensorTask, vMotorTask, vCommTask)             │     │
  │  └──────────────────┬──────────────────────────────┘     │
  │                      │ Driver API (uart_write, spi_xfer) │
  │  ┌──────────────────▼──────────────────────────────┐     │
  │  │ Driver Layer                                     │     │
  │  │ ┌──────────────────────────────────────────┐    │     │
  │  │ │ Driver State: FIFOs, config, DMA buffers │    │     │
  │  │ │ ISR Handler: read HW, signal task         │    │     │
  │  │ │ Task Interface: blocking read/write        │    │     │
  │  │ │ Mutex: serialize concurrent access         │    │     │
  │  │ └──────────────────────────────────────────┘    │     │
  │  └──────────────────┬──────────────────────────────┘     │
  │                      │ Register access (MMIO)            │
  │  ┌──────────────────▼──────────────────────────────┐     │
  │  │ Hardware Abstraction Layer (HAL)                 │     │
  │  │ ┌─────────┐ ┌─────────┐ ┌─────────┐            │     │
  │  │ │ UART    │ │ SPI     │ │ I2C     │            │     │
  │  │ │ regs    │ │ regs    │ │ regs    │            │     │
  │  │ └─────────┘ └─────────┘ └─────────┘            │     │
  │  └─────────────────────────────────────────────────┘     │
  │                                                           │
  │  No standard driver model in FreeRTOS (unlike Linux)     │
  │  Zephyr has a device driver model (struct device)        │
  │  QNX has resource managers                               │
  │  VxWorks has VxBus driver framework                      │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. UART Driver (Interrupt-Driven)

```c
/* Complete UART driver with RTOS integration */

typedef struct {
    UART_TypeDef      *hw;       /* Hardware registers */
    QueueHandle_t      rxQueue;  /* ISR → task data */
    QueueHandle_t      txQueue;  /* Task → ISR data */
    SemaphoreHandle_t  txMutex;  /* Serialize writes */
    SemaphoreHandle_t  txDone;   /* Signal TX complete */
} UartDriver_t;

static UartDriver_t uart1_drv;

/* Initialize driver */
void uart_init(UartDriver_t *drv, UART_TypeDef *hw, uint32_t baud) {
    drv->hw = hw;
    drv->rxQueue = xQueueCreate(128, sizeof(uint8_t));
    drv->txQueue = xQueueCreate(128, sizeof(uint8_t));
    drv->txMutex = xSemaphoreCreateMutex();
    drv->txDone  = xSemaphoreCreateBinary();

    /* Configure hardware */
    hw->BRR = SystemCoreClock / baud;
    hw->CR1 = UART_CR1_UE | UART_CR1_TE | UART_CR1_RE;
    hw->CR1 |= UART_CR1_RXNEIE;  /* Enable RX interrupt */

    NVIC_SetPriority(USART1_IRQn, 6);  /* RTOS-managed priority */
    NVIC_EnableIRQ(USART1_IRQn);
}

/* ISR: minimal hardware interaction */
void USART1_IRQHandler(void) {
    BaseType_t xWoken = pdFALSE;
    UartDriver_t *drv = &uart1_drv;

    /* RX: byte received */
    if (drv->hw->ISR & UART_ISR_RXNE) {
        uint8_t byte = drv->hw->RDR;  /* Read clears flag */
        xQueueSendFromISR(drv->rxQueue, &byte, &xWoken);
    }

    /* TX: transmit register empty, send next byte */
    if (drv->hw->ISR & UART_ISR_TXE) {
        uint8_t byte;
        if (xQueueReceiveFromISR(drv->txQueue, &byte, &xWoken) == pdTRUE) {
            drv->hw->TDR = byte;
        } else {
            drv->hw->CR1 &= ~UART_CR1_TXEIE;  /* No more data, disable TX int */
            xSemaphoreGiveFromISR(drv->txDone, &xWoken);
        }
    }

    portYIELD_FROM_ISR(xWoken);
}

/* API: Blocking read */
int uart_read(UartDriver_t *drv, uint8_t *buf, size_t len, TickType_t timeout) {
    size_t received = 0;
    while (received < len) {
        if (xQueueReceive(drv->rxQueue, &buf[received], timeout) != pdTRUE) {
            break;  /* Timeout */
        }
        received++;
    }
    return received;
}

/* API: Blocking write (thread-safe) */
int uart_write(UartDriver_t *drv, const uint8_t *buf, size_t len, TickType_t timeout) {
    if (xSemaphoreTake(drv->txMutex, timeout) != pdTRUE) {
        return -1;  /* Another task is writing */
    }

    /* Fill TX queue */
    for (size_t i = 0; i < len; i++) {
        xQueueSend(drv->txQueue, &buf[i], portMAX_DELAY);
    }

    /* Start transmission */
    drv->hw->CR1 |= UART_CR1_TXEIE;  /* Enable TX empty interrupt */

    /* Wait for all data to be transmitted */
    xSemaphoreTake(drv->txDone, portMAX_DELAY);

    xSemaphoreGive(drv->txMutex);
    return len;
}
```

---

## 3. Driver Design Patterns

```
  Three Driver I/O Patterns
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  1. POLLED (busy-wait):                                  │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ while (!(UART->SR & UART_SR_RXNE));            │      │
  │  │ data = UART->DR;                                │      │
  │  │                                                  │      │
  │  │ + Simple, no ISR setup                          │      │
  │  │ - Wastes CPU (blocks task entirely)             │      │
  │  │ - Other tasks starved                           │      │
  │  │ - Use only: boot code, debug output, init       │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  2. INTERRUPT-DRIVEN:                                    │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ ISR: reads byte, queues it, signals task        │      │
  │  │ Task: blocks on queue until data available      │      │
  │  │                                                  │      │
  │  │ + CPU free when no data (task blocks)           │      │
  │  │ + Good for low-to-medium data rates             │      │
  │  │ - ISR per byte overhead at high data rates      │      │
  │  │ - Use for: UART, button/GPIO, low-rate SPI      │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  3. DMA-DRIVEN:                                          │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ DMA hardware transfers blocks without CPU       │      │
  │  │ ISR fires on transfer complete (once per block) │      │
  │  │                                                  │      │
  │  │ + Minimal CPU involvement                       │      │
  │  │ + Best for high data rates and large transfers  │      │
  │  │ + Can use double-buffering for continuous flow  │      │
  │  │ - Complex setup (DMA channels, cache coherency) │      │
  │  │ - Use for: ADC sampling, audio, high-speed SPI  │      │
  │  └────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Zephyr Device Driver Model

```c
/* Zephyr: structured driver model with device tree */

/* Device instance from device tree */
const struct device *uart_dev = DEVICE_DT_GET(DT_NODELABEL(uart0));

if (!device_is_ready(uart_dev)) {
    printk("UART not ready\n");
    return;
}

/* Zephyr UART API (unified across all UART implementations) */
struct uart_config cfg = {
    .baudrate = 115200,
    .parity = UART_CFG_PARITY_NONE,
    .stop_bits = UART_CFG_STOP_BITS_1,
    .data_bits = UART_CFG_DATA_BITS_8,
    .flow_ctrl = UART_CFG_FLOW_CTRL_NONE,
};
uart_configure(uart_dev, &cfg);

/* Interrupt-driven API */
uart_irq_callback_user_data_set(uart_dev, uart_cb, NULL);
uart_irq_rx_enable(uart_dev);

static void uart_cb(const struct device *dev, void *user_data) {
    uint8_t byte;
    while (uart_irq_update(dev) && uart_irq_rx_ready(dev)) {
        uart_fifo_read(dev, &byte, 1);
        k_msgq_put(&uart_msgq, &byte, K_NO_WAIT);
    }
}

/*
 * Zephyr driver architecture:
 *
 * ┌─────────────────────────────────────────────┐
 * │ Application                                  │
 * │   uart_poll_out(dev, byte);                  │
 * ├─────────────────────────────────────────────┤
 * │ Device API (struct uart_driver_api)          │
 * │   .poll_in, .poll_out, .configure            │
 * │   .irq_tx_enable, .fifo_read, etc.          │
 * ├─────────────────────────────────────────────┤
 * │ Driver Implementation (e.g., uart_stm32.c)  │
 * │   Implements uart_driver_api functions       │
 * │   Accesses hardware registers                │
 * ├─────────────────────────────────────────────┤
 * │ Device Tree (devicetree binding)             │
 * │   Provides: base address, IRQ, clock, pins  │
 * └─────────────────────────────────────────────┘
 *
 * struct device {
 *     const char *name;
 *     const void *config;    // DT-generated config
 *     const void *api;       // Driver API struct
 *     void *data;            // Runtime state
 * };
 */
```

---

## 5. QNX Resource Managers

```
  QNX Driver Model: Resource Manager
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  In QNX, drivers run as USER-SPACE processes             │
  │  They register a path in the filesystem namespace        │
  │                                                           │
  │  Application: fd = open("/dev/ser1", O_RDWR);            │
  │               read(fd, buf, len);                         │
  │               write(fd, data, len);                       │
  │               close(fd);                                  │
  │                                                           │
  │  ┌──────────┐  MsgSend  ┌──────────────────┐            │
  │  │ App      │──────────►│ Resource Manager  │            │
  │  │ process  │◄──────────│ (driver process)  │            │
  │  └──────────┘  MsgReply └────────┬─────────┘            │
  │                                   │                       │
  │                           ┌───────▼──────────┐           │
  │                           │ Hardware (mmap'd) │           │
  │                           └──────────────────┘           │
  │                                                           │
  │  Advantages:                                             │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ · Driver crash doesn't crash the system        │      │
  │  │ · Standard POSIX API (open/read/write/ioctl)   │      │
  │  │ · Can restart crashed driver without reboot    │      │
  │  │ · Memory protection (separate address space)   │      │
  │  │ · Same API for local and network-transparent   │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Trade-off: message passing overhead (~1-3μs per call)   │
  │  vs direct function call in monolithic RTOS (~ns)        │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. BSP (Board Support Package)

```
  BSP Structure
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  BSP provides hardware-specific initialization:          │
  │                                                           │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ 1. Startup code (Reset_Handler)                │      │
  │  │    - Initialize stack pointer                   │      │
  │  │    - Copy .data from Flash to SRAM              │      │
  │  │    - Zero .bss section                          │      │
  │  │    - Initialize FPU, cache                      │      │
  │  │    - Call SystemInit() then main()              │      │
  │  │                                                  │      │
  │  │ 2. Clock configuration                          │      │
  │  │    - PLL setup, bus clock dividers               │      │
  │  │    - Peripheral clock enables                   │      │
  │  │                                                  │      │
  │  │ 3. Interrupt vector table                       │      │
  │  │    - Exception and IRQ handler addresses         │      │
  │  │    - Default handlers for unused interrupts     │      │
  │  │                                                  │      │
  │  │ 4. Linker script                                │      │
  │  │    - Memory regions (Flash, SRAM, TCM, etc.)    │      │
  │  │    - Section placement                          │      │
  │  │                                                  │      │
  │  │ 5. Low-level drivers                            │      │
  │  │    - GPIO, UART (debug console), clock          │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  FreeRTOS BSP files:                                     │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ startup_stm32f4xx.s    — vector table + startup│      │
  │  │ system_stm32f4xx.c     — SystemInit, clock setup│     │
  │  │ stm32f4xx.ld           — linker script          │      │
  │  │ FreeRTOSConfig.h       — RTOS configuration     │      │
  │  │ port.c                 — Cortex-M4 port layer   │      │
  │  │ portmacro.h            — port-specific macros   │      │
  │  └────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Design a UART driver for an RTOS. What are the key components?**
**A:** Key components: (1) Hardware abstraction — register definitions, clock enable, pin configuration. (2) RX path — ISR reads received byte from data register, sends to RX queue via `xQueueSendFromISR()`. Task blocks on queue with `xQueueReceive()`. (3) TX path — task puts bytes into TX queue, enables TX empty interrupt. ISR pulls bytes from queue and writes to data register. When queue empty, disable TX interrupt and signal completion. (4) Thread safety — TX mutex prevents concurrent writes from multiple tasks. (5) Configuration — baud rate, parity, stop bits, flow control. (6) Error handling — overrun detection, framing errors, parity errors in ISR. (7) Buffer sizing — RX queue sized for burst tolerance, TX queue for write buffering. The ISR must be minimal: read/write hardware register + queue operation + yield check.

**Q2: Compare driver models in FreeRTOS, Zephyr, and QNX.**
**A:** FreeRTOS: no standard driver model. Drivers are ad-hoc C modules. This is flexible but means no portability between hardware. Zephyr: structured driver model with device tree (DT) for hardware description, `struct device` for runtime, and standardized API structs (`uart_driver_api`, `spi_driver_api`). Drivers are board-independent; hardware specifics come from DT. QNX: resource manager pattern — drivers run as separate user-space processes accessed via POSIX API (open/read/write). Driver crash doesn't affect kernel. Most isolated but highest overhead per I/O call due to message passing. VxWorks: VxBus framework — standardized bus-based driver model with auto-discovery. Trade-off spectrum: FreeRTOS (simplest, least portable) → Zephyr (structured, portable) → QNX (fully isolated, most overhead).

**Q3: When would you use DMA-driven I/O instead of interrupt-driven?**
**A:** Use DMA when: (1) High data rates — UART at 1Mbps+ or SPI at 10MHz+ where per-byte ISR overhead is too high. (2) Continuous data streams — ADC sampling, audio capture, where data arrives constantly. (3) Large block transfers — reading sensor arrays, LCD frame buffers, flash programming. (4) CPU must do other work — DMA transfers data independently while CPU processes previous data (double-buffering). Don't use DMA when: (1) Low data rates where ISR overhead is negligible. (2) Variable-length, event-driven data (single bytes). (3) Simple I2C or low-speed SPI where setup complexity outweighs benefit. DMA complications: cache coherency (Cortex-M7), channel contention, alignment requirements, error handling.

---

## Summary

- RTOS drivers: HAL + ISR + task + IPC (queue/semaphore) + mutex for thread safety
- Polled I/O: simple but wastes CPU — only for boot/debug
- Interrupt-driven: ISR per event, good for low-medium data rates
- DMA-driven: hardware transfers blocks, ISR on completion — best for high-speed
- Zephyr: structured driver model with device tree and standardized APIs
- QNX: drivers as user-space processes (resource managers), fault-isolated
- BSP: startup code, clock config, vector table, linker script, port layer

---

[Previous Chapter: Time Management ←](Chapter_15_Time_Management.md) | [Next Chapter: I/O Management →](Chapter_17_IO_Management.md)
