# Chapter 26: RTOS Porting

## Learning Goals
- Understand what porting an RTOS to new hardware requires
- Master the FreeRTOS port layer (port.c, portmacro.h)
- Learn BSP (Board Support Package) development
- Know Hardware Abstraction Layer (HAL) design

---

## 1. Porting Overview

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  RTOS Porting = Making kernel run on new CPU/board       │
  │                                                           │
  │  What needs porting:                                     │
  │  ┌─────────────────────────────────────────────────┐    │
  │  │ 1. Context switch (save/restore registers)      │    │
  │  │    → Assembly, architecture-specific             │    │
  │  │ 2. Tick timer (SysTick or GP timer setup)       │    │
  │  │    → Configure timer interrupt at tick rate      │    │
  │  │ 3. Interrupt enable/disable (critical sections) │    │
  │  │    → BASEPRI (Cortex-M) or CPSR (Cortex-A)     │    │
  │  │ 4. Stack initialization (for new tasks)         │    │
  │  │    → Set up initial stack frame for context sw  │    │
  │  │ 5. Startup code (vector table, clock init)      │    │
  │  │    → Board-specific, runs before RTOS starts    │    │
  │  └─────────────────────────────────────────────────┘    │
  │                                                           │
  │  What does NOT need porting (platform-independent):      │
  │  ┌─────────────────────────────────────────────────┐    │
  │  │ - Task management (tasks.c)                      │    │
  │  │ - Queue implementation (queue.c)                 │    │
  │  │ - Timer management (timers.c)                    │    │
  │  │ - Scheduler logic (tasks.c)                      │    │
  │  │ - Event groups (event_groups.c)                  │    │
  │  │ - Memory management (heap_*.c)                   │    │
  │  └─────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. FreeRTOS Port Layer

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  FreeRTOS/Source/portable/                               │
  │  ├── GCC/                                                │
  │  │   ├── ARM_CM0/          ← Cortex-M0 (no BASEPRI)    │
  │  │   │   ├── port.c                                     │
  │  │   │   └── portmacro.h                                │
  │  │   ├── ARM_CM3/          ← Cortex-M3                  │
  │  │   │   ├── port.c                                     │
  │  │   │   └── portmacro.h                                │
  │  │   ├── ARM_CM4F/         ← Cortex-M4 with FPU         │
  │  │   │   ├── port.c                                     │
  │  │   │   └── portmacro.h                                │
  │  │   └── ARM_CM7/r0p1/    ← Cortex-M7 (erratum fix)    │
  │  ├── IAR/                  ← IAR compiler ports         │
  │  └── RVDS/                 ← ARM compiler (Keil)        │
  │                                                           │
  │  Each port provides exactly these functions:             │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ pxPortInitialiseStack()  ← Init task stack   │       │
  │  │ xPortStartScheduler()    ← Start first task  │       │
  │  │ vPortEndScheduler()      ← (rarely used)     │       │
  │  │ vPortYield()             ← Trigger PendSV    │       │
  │  │ vPortEnterCritical()     ← Disable interrupts│       │
  │  │ vPortExitCritical()      ← Restore interrupts│       │
  │  │ xPortPendSVHandler()     ← Context switch ISR│       │
  │  │ xPortSysTickHandler()    ← Tick interrupt     │       │
  │  └──────────────────────────────────────────────┘       │
  └──────────────────────────────────────────────────────────┘
```

```c
/* portmacro.h — architecture-specific definitions */
/* Example: ARM Cortex-M4F (GCC) */

#ifndef PORTMACRO_H
#define PORTMACRO_H

/* Type definitions matching architecture */
#define portCHAR        char
#define portFLOAT       float
#define portDOUBLE      double
#define portLONG        long
#define portSHORT       short
#define portSTACK_TYPE  uint32_t       /* 32-bit stack entries */
#define portBASE_TYPE   long
typedef portSTACK_TYPE  StackType_t;
typedef long            BaseType_t;
typedef unsigned long   UBaseType_t;
typedef uint32_t        TickType_t;
#define portMAX_DELAY   (TickType_t)0xFFFFFFFF

/* Architecture-specific constants */
#define portSTACK_GROWTH        (-1)   /* Stack grows downward */
#define portTICK_PERIOD_MS      ((TickType_t)1000 / configTICK_RATE_HZ)
#define portBYTE_ALIGNMENT      8      /* 8-byte aligned stack */
#define portNVIC_INT_CTRL_REG   (*((volatile uint32_t *)0xE000ED04))
#define portNVIC_PENDSVSET_BIT  (1UL << 28)

/* Critical section: mask interrupts using BASEPRI */
#define portDISABLE_INTERRUPTS()  vPortRaiseBASEPRI()
#define portENABLE_INTERRUPTS()   vPortSetBASEPRI(0)

/* Yield: trigger PendSV exception */
#define portYIELD() do {                            \
    portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT; \
    __DSB(); __ISB();                                \
} while(0)

#define portYIELD_FROM_ISR(x)  portYIELD()

static inline void vPortRaiseBASEPRI(void) {
    uint32_t newBASEPRI = configMAX_SYSCALL_INTERRUPT_PRIORITY;
    __asm volatile (
        "MSR BASEPRI, %0  \n"
        "DSB               \n"
        "ISB               \n"
        :: "r" (newBASEPRI) : "memory"
    );
}

#endif /* PORTMACRO_H */
```

---

## 3. Stack Initialization for New Tasks

```c
/* port.c — pxPortInitialiseStack() */
/* Sets up the stack so context switch "restores" initial state */

/*
 * Initial stack layout (Cortex-M4F, stack grows down):
 *
 *  High address (stack base)
 *  ┌──────────────────────┐
 *  │ xPSR = 0x01000000   │  ← Thumb bit must be set
 *  │ PC = task_function   │  ← Entry point
 *  │ LR = task_exit_error │  ← Error if task returns
 *  │ R12 = 0              │
 *  │ R3  = 0              │  Hardware-saved frame
 *  │ R2  = 0              │  (auto-pushed on exception)
 *  │ R1  = 0              │
 *  │ R0  = pvParameters   │  ← Task parameter
 *  ├──────────────────────┤
 *  │ EXC_RETURN           │  ← 0xFFFFFFFD (return to PSP)
 *  │ S16-S31 (if FPU)     │  ← Lazy: space reserved only
 *  │ R11 = 0              │
 *  │ R10 = 0              │
 *  │ R9  = 0              │  Software-saved frame
 *  │ R8  = 0              │  (manually saved in PendSV)
 *  │ R7  = 0              │
 *  │ R6  = 0              │
 *  │ R5  = 0              │
 *  │ R4  = 0              │
 *  └──────────────────────┘  ← pxTopOfStack (stored in TCB)
 *  Low address
 */

StackType_t *pxPortInitialiseStack(StackType_t *pxTopOfStack,
                                    TaskFunction_t pxCode,
                                    void *pvParameters) {
    /* Simulate hardware-stacked exception frame */
    pxTopOfStack--;
    *pxTopOfStack = 0x01000000;                /* xPSR: Thumb bit */
    pxTopOfStack--;
    *pxTopOfStack = ((StackType_t)pxCode) & 0xFFFFFFFE; /* PC */
    pxTopOfStack--;
    *pxTopOfStack = (StackType_t)prvTaskExitError; /* LR */
    pxTopOfStack -= 5;                         /* R12, R3, R2, R1 */
    *pxTopOfStack = (StackType_t)pvParameters; /* R0 = parameter */

    /* EXC_RETURN for FPU-enabled port */
    pxTopOfStack--;
    *pxTopOfStack = 0xFFFFFFFD;  /* Return to Thread mode, PSP */

    /* Software-saved registers R4-R11 (initially zero) */
    pxTopOfStack -= 8;
    memset(pxTopOfStack, 0, 8 * sizeof(StackType_t));

    return pxTopOfStack;  /* Stored in TCB->pxTopOfStack */
}
```

---

## 4. BSP (Board Support Package)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  BSP Structure for RTOS project:                         │
  │                                                           │
  │  project/                                                │
  │  ├── src/                                                │
  │  │   ├── main.c              ← Application entry        │
  │  │   ├── tasks.c             ← Application tasks        │
  │  │   └── drivers/            ← Peripheral drivers       │
  │  ├── bsp/                                                │
  │  │   ├── startup_stm32f4.s   ← Reset handler, vectors   │
  │  │   ├── system_stm32f4.c    ← SystemInit(), clocks     │
  │  │   ├── stm32f4xx_hal_conf.h← HAL configuration       │
  │  │   └── linker.ld           ← Memory layout            │
  │  ├── FreeRTOS/                                           │
  │  │   ├── Source/                                         │
  │  │   │   ├── tasks.c         ← Kernel (portable)        │
  │  │   │   ├── queue.c                                     │
  │  │   │   └── portable/GCC/ARM_CM4F/                     │
  │  │   │       ├── port.c      ← Port layer               │
  │  │   │       └── portmacro.h                            │
  │  │   └── include/            ← FreeRTOS headers         │
  │  └── FreeRTOSConfig.h        ← Project-specific config  │
  └──────────────────────────────────────────────────────────┘
```

```c
/* startup_stm32f4.s — minimal startup (simplified) */

.section .isr_vector, "a"
.word _estack           /* Initial stack pointer (top of RAM) */
.word Reset_Handler     /* Reset vector */
.word NMI_Handler
.word HardFault_Handler
.word MemManage_Handler
.word BusFault_Handler
.word UsageFault_Handler
.word 0, 0, 0, 0       /* Reserved */
.word SVC_Handler       /* FreeRTOS: vPortSVCHandler */
.word 0, 0
.word PendSV_Handler    /* FreeRTOS: xPortPendSVHandler */
.word SysTick_Handler   /* FreeRTOS: xPortSysTickHandler */
/* ... IRQ handlers ... */

.section .text
Reset_Handler:
    /* Copy .data from flash to SRAM */
    ldr r0, =_sdata
    ldr r1, =_edata
    ldr r2, =_sidata
copy_loop:
    cmp r0, r1
    bge copy_done
    ldr r3, [r2], #4
    str r3, [r0], #4
    b copy_loop
copy_done:

    /* Zero .bss */
    ldr r0, =_sbss
    ldr r1, =_ebss
    mov r2, #0
zero_loop:
    cmp r0, r1
    bge zero_done
    str r2, [r0], #4
    b zero_loop
zero_done:

    /* Call SystemInit (clock config) then main */
    bl SystemInit
    bl main
    b .  /* Should never reach here */
```

---

## 5. Hardware Abstraction Layer

```c
/*
 * HAL provides hardware-independent API:
 *
 *  Application
 *      │
 *  ┌───▼─────────────────┐
 *  │ HAL API              │  hal_gpio_write(pin, HIGH)
 *  │ (portable interface) │  hal_uart_send(port, data, len)
 *  └───┬─────────────────┘
 *      │
 *  ┌───▼─────────────────┐
 *  │ HAL Implementation   │  Platform-specific register access
 *  │ (per MCU family)     │  STM32, NXP, Nordic, TI
 *  └───┬─────────────────┘
 *      │
 *  ┌───▼─────────────────┐
 *  │ Hardware Registers   │  Memory-mapped I/O
 *  └─────────────────────┘
 */

/* HAL interface (header) */
typedef enum { HAL_PIN_LOW = 0, HAL_PIN_HIGH = 1 } hal_pin_state_t;

void hal_gpio_init(uint32_t port, uint32_t pin, uint32_t mode);
void hal_gpio_write(uint32_t port, uint32_t pin, hal_pin_state_t state);
hal_pin_state_t hal_gpio_read(uint32_t port, uint32_t pin);

/* HAL implementation for STM32F4 */
void hal_gpio_write(uint32_t port, uint32_t pin, hal_pin_state_t state) {
    GPIO_TypeDef *gpio = (GPIO_TypeDef *)(GPIOA_BASE + port * 0x400);
    if (state == HAL_PIN_HIGH) {
        gpio->BSRR = (1U << pin);        /* Set via BSRR (atomic) */
    } else {
        gpio->BSRR = (1U << (pin + 16)); /* Reset via BSRR high half */
    }
}

/* HAL implementation for NXP LPC (same API, different registers) */
void hal_gpio_write(uint32_t port, uint32_t pin, hal_pin_state_t state) {
    if (state == HAL_PIN_HIGH) {
        LPC_GPIO->SET[port] = (1U << pin);
    } else {
        LPC_GPIO->CLR[port] = (1U << pin);
    }
}
```

---

## 6. Zephyr Porting Model

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Zephyr uses Device Tree + Kconfig for porting:         │
  │                                                           │
  │  boards/arm/my_board/                                    │
  │  ├── my_board.dts           ← Device Tree Source        │
  │  ├── my_board_defconfig     ← Kconfig defaults          │
  │  ├── board.cmake            ← Build/flash settings      │
  │  └── Kconfig.board          ← Board menu entry          │
  │                                                           │
  │  soc/arm/my_soc/                                         │
  │  ├── soc.c                  ← SoC init code             │
  │  ├── soc.h                  ← SoC-specific defines      │
  │  ├── Kconfig.soc            ← SoC Kconfig               │
  │  └── CMakeLists.txt                                      │
  │                                                           │
  │  Device Tree (my_board.dts):                             │
  │  / {                                                     │
  │    model = "My Custom Board";                            │
  │    chosen {                                              │
  │      zephyr,console = &uart0;                            │
  │      zephyr,sram = &sram0;                               │
  │      zephyr,flash = &flash0;                             │
  │    };                                                     │
  │    soc {                                                  │
  │      uart0: uart@40011000 {                              │
  │        compatible = "st,stm32-uart";                     │
  │        reg = <0x40011000 0x400>;                         │
  │        interrupts = <37 0>;                              │
  │        status = "okay";                                  │
  │      };                                                   │
  │    };                                                     │
  │  };                                                       │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: What files need to be created or modified when porting FreeRTOS to a new Cortex-M MCU?**
**A:** Two port-layer files and one config file: (1) **portmacro.h**: architecture-specific type definitions (`StackType_t`, `TickType_t`), stack growth direction, byte alignment, critical section macros (BASEPRI for Cortex-M3+, PRIMASK for Cortex-M0), yield macro (PendSV trigger). (2) **port.c**: `pxPortInitialiseStack()` — build initial stack frame matching hardware exception entry format, `xPortStartScheduler()` — configure SysTick, set PendSV/SysTick to lowest priority, start first task via SVC, `PendSV_Handler` — assembly for context switch (save R4-R11, load next task's stack), `SysTick_Handler` — increment tick, check for context switch. (3) **FreeRTOSConfig.h**: set `configCPU_CLOCK_HZ`, `configTICK_RATE_HZ`, stack sizes, `configMAX_SYSCALL_INTERRUPT_PRIORITY`, memory model. Additionally: BSP startup code (vector table, clock init), linker script (memory regions for the specific MCU).

**Q2: What is the difference between a BSP and a HAL?**
**A:** BSP (Board Support Package) is board-specific: startup code, vector table, clock configuration, linker script, board-level pin assignments (which UART is on which pins). It handles the specific board's hardware layout and runs before the RTOS. HAL (Hardware Abstraction Layer) is a portable API layer: provides hardware-independent functions (`hal_gpio_write()`, `hal_uart_send()`) that internally map to MCU-specific register access. HAL allows application code to be portable across MCU families. BSP = "what hardware is on this board", HAL = "how to talk to that hardware through a common API". In practice: BSP calls `SystemInit()` to set up clocks, HAL provides `hal_gpio_write()` that translates to STM32 BSRR register or NXP GPIO->SET register depending on target.

---

## Summary

- Porting RTOS requires: context switch assembly, tick timer setup, critical section mechanism, stack initialization
- FreeRTOS port: portmacro.h (types, macros) + port.c (PendSV handler, stack init, SysTick)
- BSP: startup code, vector table, clock config, linker script — board-specific
- HAL: portable API over hardware registers — MCU-family-specific implementation
- Zephyr: Device Tree + Kconfig model — board definitions in DTS files
- FreeRTOS kernel code (tasks.c, queue.c) is platform-independent — never modified during porting

---

[Previous Chapter: Performance Optimization ←](Chapter_25_Performance.md) | [Next Chapter: Build Systems →](Chapter_27_Build_Systems.md)
