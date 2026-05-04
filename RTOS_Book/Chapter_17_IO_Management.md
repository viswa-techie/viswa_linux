# Chapter 17: I/O Management

## Learning Goals
- Understand I/O subsystem architecture in RTOS
- Master peripheral interfaces: SPI, I2C, GPIO, ADC
- Learn I/O scheduling and priority-aware access
- Know bus arbitration and shared peripheral management
- Understand sensor fusion and data pipeline patterns

---

## 1. Peripheral Interface Overview

```
  Common Embedded Peripherals
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Bus        │ Speed      │ Topology  │ Use Cases         │
  │  ───────────┼────────────┼───────────┼────────────────── │
  │  GPIO       │ N/A        │ Pin-level │ LEDs, buttons,    │
  │             │            │           │ chip select        │
  │  UART       │ ~1Mbps     │ Point-to- │ Debug console,    │
  │             │            │ point     │ GPS, BT module    │
  │  SPI        │ ~50MHz+    │ Master +  │ Flash, display,   │
  │             │            │ multi-slave│ ADC, sensors      │
  │  I2C        │ 100K-3.4M  │ Multi-    │ Sensors, EEPROM,  │
  │             │            │ master bus│ RTC, IO expanders  │
  │  CAN        │ 1Mbps      │ Bus       │ Automotive, indust│
  │  ADC        │ ~1MSPS     │ Internal  │ Analog sensors,   │
  │             │            │           │ voltage monitoring │
  │  PWM        │ ~kHz-MHz   │ Output    │ Motors, LEDs,     │
  │             │            │           │ servo control      │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. SPI Driver with RTOS

```c
/* SPI bus shared between multiple devices (tasks) */

typedef struct {
    SPI_TypeDef       *hw;
    SemaphoreHandle_t  busMutex;      /* Exclusive bus access */
    SemaphoreHandle_t  xferDone;      /* DMA/ISR completion signal */
    DMA_Channel_TypeDef *txDma;
    DMA_Channel_TypeDef *rxDma;
} SpiBus_t;

typedef struct {
    SpiBus_t          *bus;
    GPIO_TypeDef      *csPort;
    uint16_t           csPin;
    uint32_t           clockDiv;      /* Per-device clock config */
    uint8_t            cpol, cpha;    /* Per-device SPI mode */
} SpiDevice_t;

/* Thread-safe SPI transfer */
int spi_transfer(SpiDevice_t *dev, const uint8_t *tx,
                 uint8_t *rx, size_t len, TickType_t timeout) {

    /* Lock the bus (blocks if another task is using it) */
    if (xSemaphoreTake(dev->bus->busMutex, timeout) != pdTRUE) {
        return -1;  /* Bus busy */
    }

    /* Configure bus for THIS device's settings */
    SPI_TypeDef *spi = dev->bus->hw;
    uint32_t cr1 = SPI_CR1_MSTR | SPI_CR1_SSM | SPI_CR1_SSI
                 | dev->clockDiv;
    if (dev->cpol) cr1 |= SPI_CR1_CPOL;
    if (dev->cpha) cr1 |= SPI_CR1_CPHA;
    spi->CR1 = cr1;

    /* Assert chip select */
    dev->csPort->BSRR = (uint32_t)dev->csPin << 16;  /* Low */

    /* Start DMA transfer */
    dma_setup_tx(dev->bus->txDma, tx, len);
    dma_setup_rx(dev->bus->rxDma, rx, len);
    spi->CR1 |= SPI_CR1_SPE;  /* Enable SPI */

    /* Wait for completion */
    xSemaphoreTake(dev->bus->xferDone, portMAX_DELAY);

    /* Deassert chip select */
    dev->csPort->BSRR = dev->csPin;  /* High */

    spi->CR1 &= ~SPI_CR1_SPE;

    /* Release bus for other tasks */
    xSemaphoreGive(dev->bus->busMutex);
    return 0;
}

/*
 * Bus sharing pattern:
 *
 * Task A (Sensor, prio 3):  [ mutex ] ── SPI xfer ── [ release ]
 * Task B (Flash, prio 2):   wait─────── [ mutex ] ── SPI xfer ── [ release ]
 * Task C (Display, prio 1): wait──────────────────── [ mutex ] ── SPI xfer
 *
 * Priority inheritance on busMutex prevents inversion:
 * If high-prio task needs bus while low-prio holds it,
 * low-prio is boosted to complete quickly.
 */
```

---

## 3. I2C with Multi-Master Consideration

```c
/* I2C driver with retry logic for multi-master bus */

typedef struct {
    I2C_TypeDef       *hw;
    SemaphoreHandle_t  busMutex;
    SemaphoreHandle_t  xferDone;
    volatile int       error;
} I2cBus_t;

int i2c_write_reg(I2cBus_t *bus, uint8_t addr, uint8_t reg,
                  const uint8_t *data, size_t len) {
    int retries = 3;

    xSemaphoreTake(bus->busMutex, portMAX_DELAY);

    while (retries-- > 0) {
        bus->error = 0;

        /* Generate START + address + write */
        bus->hw->CR2 = (addr << 1) | I2C_CR2_START
                     | ((len + 1) << I2C_CR2_NBYTES_Pos);

        /* Send register address */
        while (!(bus->hw->ISR & I2C_ISR_TXIS));
        bus->hw->TXDR = reg;

        /* Send data bytes */
        for (size_t i = 0; i < len; i++) {
            while (!(bus->hw->ISR & I2C_ISR_TXIS)) {
                if (bus->hw->ISR & I2C_ISR_NACKF) {
                    bus->hw->ICR = I2C_ICR_NACKCF;
                    bus->error = -1;
                    break;
                }
            }
            if (bus->error) break;
            bus->hw->TXDR = data[i];
        }

        /* Generate STOP */
        bus->hw->CR2 |= I2C_CR2_STOP;
        while (bus->hw->ISR & I2C_ISR_BUSY);

        if (bus->error == 0) break;  /* Success */

        /* Bus error — wait and retry */
        vTaskDelay(pdMS_TO_TICKS(1));
    }

    xSemaphoreGive(bus->busMutex);
    return bus->error;
}
```

---

## 4. ADC with RTOS Integration

```c
/* ADC driver: DMA + task notification pattern */

#define ADC_CHANNELS    4
#define ADC_SAMPLES     16  /* Oversampling count */

static volatile uint16_t adc_dma_buf[ADC_CHANNELS * ADC_SAMPLES]
    __attribute__((aligned(4)));

static TaskHandle_t xAdcTaskHandle;

void adc_init(void) {
    /* Configure ADC for scan mode + DMA circular */
    ADC1->CFGR = ADC_CFGR_CONT | ADC_CFGR_DMA_Circular;
    ADC1->SQR1 = (ADC_CHANNELS - 1) << ADC_SQR1_L_Pos;
    /* Configure channels, sampling time... */

    /* DMA circular mode: continuous conversion */
    DMA1_Channel1->CPAR = (uint32_t)&ADC1->DR;
    DMA1_Channel1->CMAR = (uint32_t)adc_dma_buf;
    DMA1_Channel1->CNDTR = ADC_CHANNELS * ADC_SAMPLES;
    DMA1_Channel1->CCR = DMA_CCR_MINC | DMA_CCR_MSIZE_16
                       | DMA_CCR_PSIZE_16 | DMA_CCR_CIRC
                       | DMA_CCR_TCIE | DMA_CCR_EN;

    NVIC_EnableIRQ(DMA1_Channel1_IRQn);
    ADC1->CR |= ADC_CR_ADSTART;
}

void DMA1_Channel1_IRQHandler(void) {
    BaseType_t xWoken = pdFALSE;
    DMA1->IFCR = DMA_IFCR_CTCIF1;
    vTaskNotifyGiveFromISR(xAdcTaskHandle, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

void vAdcProcessTask(void *pvParameters) {
    xAdcTaskHandle = xTaskGetCurrentTaskHandle();

    for (;;) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

        /* Average oversampled readings per channel */
        uint32_t averages[ADC_CHANNELS] = {0};
        for (int s = 0; s < ADC_SAMPLES; s++) {
            for (int ch = 0; ch < ADC_CHANNELS; ch++) {
                averages[ch] += adc_dma_buf[s * ADC_CHANNELS + ch];
            }
        }
        for (int ch = 0; ch < ADC_CHANNELS; ch++) {
            averages[ch] /= ADC_SAMPLES;
            xQueueOverwrite(xAdcQueues[ch], &averages[ch]);
        }
    }
}
```

---

## 5. GPIO and Interrupt-Driven Input

```c
/* Button debounce with RTOS software timer */

static TimerHandle_t xDebounceTimer;
static volatile uint8_t button_state = 0;

void button_init(void) {
    /* Configure GPIO as input with pull-up, falling edge interrupt */
    GPIOA->MODER &= ~GPIO_MODER_MODER0;      /* Input mode */
    GPIOA->PUPDR |= GPIO_PUPDR_PUPDR0_0;      /* Pull-up */

    EXTI->IMR |= EXTI_IMR_IM0;                /* Unmask line 0 */
    EXTI->FTSR |= EXTI_FTSR_FT0;              /* Falling edge */
    NVIC_SetPriority(EXTI0_IRQn, 6);
    NVIC_EnableIRQ(EXTI0_IRQn);

    xDebounceTimer = xTimerCreate("Debounce",
        pdMS_TO_TICKS(50), pdFALSE, NULL, vDebounceCallback);
}

void EXTI0_IRQHandler(void) {
    BaseType_t xWoken = pdFALSE;
    EXTI->PR = EXTI_PR_PIF0;  /* Clear pending */

    /* Disable further interrupts during debounce */
    EXTI->IMR &= ~EXTI_IMR_IM0;

    /* Start debounce timer (50ms) */
    xTimerResetFromISR(xDebounceTimer, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

void vDebounceCallback(TimerHandle_t xTimer) {
    /* Read stable button state after debounce period */
    if (!(GPIOA->IDR & GPIO_IDR_ID0)) {
        button_state = 1;
        /* Notify application task */
        xTaskNotifyGive(xAppTaskHandle);
    }
    /* Re-enable interrupt */
    EXTI->IMR |= EXTI_IMR_IM0;
}
```

---

## 6. Sensor Data Pipeline

```
  Multi-Stage Data Pipeline Pattern
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Stage 1: Acquisition (highest priority)                 │
  │  ┌───────────┐    ┌───────────┐    ┌───────────┐       │
  │  │ ADC DMA   │    │ IMU SPI   │    │ GPS UART  │       │
  │  │ ISR → Q   │    │ ISR → Q   │    │ ISR → Q   │       │
  │  └─────┬─────┘    └─────┬─────┘    └─────┬─────┘       │
  │        ▼                ▼                ▼               │
  │                                                           │
  │  Stage 2: Processing (medium priority)                   │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ Sensor Fusion Task                            │       │
  │  │ - Read from all sensor queues                 │       │
  │  │ - Kalman filter / complementary filter        │       │
  │  │ - Output: fused orientation/position          │       │
  │  └───────────────┬──────────────────────────────┘       │
  │                   ▼                                       │
  │                                                           │
  │  Stage 3: Decision (medium priority)                     │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ Control Task                                  │       │
  │  │ - PID controller                              │       │
  │  │ - Generate actuator commands                  │       │
  │  └───────────────┬──────────────────────────────┘       │
  │                   ▼                                       │
  │                                                           │
  │  Stage 4: Output (high priority)                         │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ Actuator Task                                 │       │
  │  │ - PWM output (motors)                         │       │
  │  │ - CAN message output                          │       │
  │  └──────────────────────────────────────────────┘       │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How would you share an SPI bus between multiple RTOS tasks safely?**
**A:** Use a mutex to protect the entire bus transaction (chip select assert → transfer → chip select deassert). Each SPI device has its own CS pin and may require different clock speed, polarity, and phase. On acquiring the mutex: (1) reconfigure SPI peripheral for this device's settings, (2) assert CS, (3) perform transfer (interrupt or DMA driven), (4) deassert CS, (5) release mutex. Use a mutex (not semaphore) so priority inheritance protects against inversion — if a high-priority task needs the bus while a low-priority task holds it, the low-priority task is boosted. Critical: never assert CS of device A while device B's CS is still low.

**Q2: Explain hardware debouncing vs software debouncing for buttons.**
**A:** Hardware: RC filter (resistor + capacitor) on the GPIO line smooths the bounce. Advantage: no CPU time, works without RTOS. Disadvantage: added components, fixed debounce time. Software: on GPIO interrupt, disable further interrupts and start a timer (e.g., 20-50ms). When timer expires, read the stable pin state. In RTOS: use a software timer (`xTimerReset` from ISR). Advantage: adjustable timing, no extra components. The ISR-timer pattern is common: ISR fires on edge → starts debounce timer → timer callback reads stable state and re-enables interrupt. Never busy-wait in ISR for debounce.

**Q3: Design a sensor data pipeline for a drone flight controller.**
**A:** Four-stage pipeline: (1) Acquisition — IMU (SPI DMA, 1kHz), barometer (I2C, 50Hz), GPS (UART, 10Hz), each with ISR placing data into dedicated queues. (2) Sensor fusion — Kalman filter task reads all sensor queues, outputs attitude/position estimate at IMU rate (1kHz). Priority set for consistent timing. (3) Control — PID controller task consumes fused data, computes motor commands at 500Hz. (4) Output — motor task writes PWM values, telemetry task sends data over radio. Priorities: acquisition ISRs highest, motor output and control high, sensor fusion medium, telemetry lowest. Each stage connected by queues for decoupling and buffering.

---

## Summary

- Peripheral drivers use ISR + queue + mutex pattern for RTOS integration
- SPI/I2C bus sharing: mutex protects entire transaction including CS management
- DMA for ADC: continuous conversion + circular DMA + notification on completion
- GPIO debounce: ISR disables interrupt, starts timer, timer callback reads stable state
- Sensor data pipelines: staged architecture with priority-ordered tasks and queue coupling
- Per-device configuration: reconfigure shared bus peripheral before each device access

---

[Previous Chapter: Device Drivers ←](Chapter_16_Device_Drivers.md) | [Next Chapter: Networking →](Chapter_18_Networking.md)
