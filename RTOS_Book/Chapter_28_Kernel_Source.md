# Chapter 28: RTOS Kernel Source Structure

## Learning Goals
- Navigate FreeRTOS kernel source code with confidence
- Understand Zephyr source organization and build artifacts
- Know QNX Neutrino and VxWorks source layout
- Map RTOS concepts to their source file implementations

---

## 1. FreeRTOS Source Structure

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  FreeRTOS/                                               │
  │  ├── Source/                 ← Kernel source (portable)  │
  │  │   ├── tasks.c            ← Task management, scheduler│
  │  │   │                        xTaskCreate, vTaskDelay    │
  │  │   │                        vTaskSwitchContext         │
  │  │   │                        prvIdleTask                │
  │  │   │                        ~4000 lines                │
  │  │   ├── queue.c            ← Queues, semaphores, mutexes│
  │  │   │                        xQueueCreate, xQueueSend  │
  │  │   │                        xSemaphoreCreateMutex     │
  │  │   │                        Priority inheritance here  │
  │  │   │                        ~2500 lines                │
  │  │   ├── list.c             ← Doubly-linked list (core) │
  │  │   │                        vListInsert, vListRemove  │
  │  │   │                        Used by ready/blocked lists│
  │  │   │                        ~200 lines                 │
  │  │   ├── timers.c           ← Software timer daemon task│
  │  │   │                        xTimerCreate, prvTimerTask│
  │  │   │                        Timer command queue        │
  │  │   │                        ~1200 lines                │
  │  │   ├── event_groups.c     ← Event flag groups         │
  │  │   │                        xEventGroupSetBits        │
  │  │   │                        xEventGroupWaitBits       │
  │  │   │                        ~800 lines                 │
  │  │   ├── stream_buffer.c    ← Stream/message buffers    │
  │  │   │                        Single reader/writer      │
  │  │   │                        ~1000 lines                │
  │  │   ├── croutine.c         ← Co-routines (legacy)      │
  │  │   │                                                   │
  │  │   ├── include/           ← Public API headers        │
  │  │   │   ├── FreeRTOS.h     ← Master include            │
  │  │   │   ├── task.h         ← Task API declarations     │
  │  │   │   ├── queue.h        ← Queue/semaphore API       │
  │  │   │   ├── semphr.h       ← Semaphore macros          │
  │  │   │   ├── timers.h       ← Timer API                 │
  │  │   │   ├── event_groups.h                              │
  │  │   │   ├── stream_buffer.h                             │
  │  │   │   ├── list.h         ← List type definitions     │
  │  │   │   ├── portable.h     ← Port layer interface      │
  │  │   │   └── projdefs.h     ← pdTRUE, pdFALSE, pdPASS  │
  │  │   │                                                   │
  │  │   └── portable/          ← Architecture-specific     │
  │  │       ├── GCC/ARM_CM4F/                               │
  │  │       │   ├── port.c     ← Context switch, tick      │
  │  │       │   └── portmacro.h← Types, critical section   │
  │  │       └── MemMang/       ← Memory allocators         │
  │  │           ├── heap_1.c   ← Simple, no free           │
  │  │           ├── heap_2.c   ← Best fit, no coalescing   │
  │  │           ├── heap_3.c   ← Wraps malloc/free         │
  │  │           ├── heap_4.c   ← First fit + coalescing    │
  │  │           └── heap_5.c   ← heap_4 + multiple regions│
  │  │                                                       │
  │  └── FreeRTOSConfig.h       ← Project-specific config   │
  │      (in application directory, not in kernel source)    │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Key FreeRTOS Source Code Walkthrough

```c
/* tasks.c: Core data structures */

/* Ready task lists — one per priority level */
static List_t pxReadyTasksLists[configMAX_PRIORITIES];

/* Delayed task lists (double-buffered for overflow) */
static List_t xDelayedTaskList1;
static List_t xDelayedTaskList2;
static List_t *volatile pxDelayedTaskList;
static List_t *volatile pxOverflowDelayedTaskList;

/* Currently running task */
TCB_t *volatile pxCurrentTCB = NULL;

/* Scheduler state */
static volatile BaseType_t xSchedulerRunning = pdFALSE;
static volatile UBaseType_t uxCurrentNumberOfTasks = 0;
static volatile TickType_t xTickCount = 0;
static volatile UBaseType_t uxTopReadyPriority = 0;

/* ------------------------------------------------------- */

/* tasks.c: xTaskCreate() — simplified flow */
BaseType_t xTaskCreate(TaskFunction_t pxTaskCode,
                        const char *pcName,
                        uint16_t usStackDepth,
                        void *pvParameters,
                        UBaseType_t uxPriority,
                        TaskHandle_t *pxCreatedTask) {
    TCB_t *pxNewTCB;
    StackType_t *pxStack;

    /* 1. Allocate stack */
    pxStack = pvPortMalloc(usStackDepth * sizeof(StackType_t));

    /* 2. Allocate TCB */
    pxNewTCB = pvPortMalloc(sizeof(TCB_t));

    /* 3. Initialize TCB fields */
    pxNewTCB->pxStack = pxStack;
    pxNewTCB->uxPriority = uxPriority;
    strncpy(pxNewTCB->pcTaskName, pcName, configMAX_TASK_NAME_LEN);

    /* 4. Initialize stack frame (port-specific) */
    pxNewTCB->pxTopOfStack = pxPortInitialiseStack(
        pxStack + usStackDepth - 1,
        pxTaskCode,
        pvParameters
    );

    /* 5. Add to ready list */
    taskENTER_CRITICAL();
    uxCurrentNumberOfTasks++;
    prvAddTaskToReadyList(pxNewTCB);

    /* 6. Yield if new task has higher priority */
    if (pxNewTCB->uxPriority > pxCurrentTCB->uxPriority) {
        taskYIELD_IF_USING_PREEMPTION();
    }
    taskEXIT_CRITICAL();

    return pdPASS;
}

/* ------------------------------------------------------- */

/* tasks.c: vTaskSwitchContext() — called from PendSV */
void vTaskSwitchContext(void) {
    /* Find highest priority ready task */
    taskSELECT_HIGHEST_PRIORITY_TASK();
    /* pxCurrentTCB now points to next task to run */
}

/* Macro: O(1) selection using CLZ (Count Leading Zeros) */
#define taskSELECT_HIGHEST_PRIORITY_TASK() do {            \
    UBaseType_t uxTopPriority;                              \
    portGET_HIGHEST_PRIORITY(uxTopPriority,                 \
                             uxTopReadyPriority);           \
    listGET_OWNER_OF_NEXT_ENTRY(pxCurrentTCB,              \
        &(pxReadyTasksLists[uxTopPriority]));              \
} while(0)

/* portGET_HIGHEST_PRIORITY uses CLZ on Cortex-M: */
/* 31 - __builtin_clz(uxReadyPriorities) → highest set bit */
```

---

## 3. Zephyr Source Organization

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  zephyr/                                                 │
  │  ├── kernel/                ← Kernel core                │
  │  │   ├── sched.c           ← Scheduler                  │
  │  │   ├── thread.c          ← Thread management          │
  │  │   ├── sem.c             ← Semaphores                 │
  │  │   ├── mutex.c           ← Mutexes (priority inherit) │
  │  │   ├── msg_q.c           ← Message queues             │
  │  │   ├── mailbox.c         ← Mailboxes                  │
  │  │   ├── pipe.c            ← Pipes                      │
  │  │   ├── timer.c           ← Kernel timers              │
  │  │   ├── mem_slab.c        ← Fixed-size memory blocks   │
  │  │   ├── mempool.c         ← Variable-size memory pool  │
  │  │   └── idle.c            ← Idle thread + PM           │
  │  │                                                       │
  │  ├── arch/                  ← Architecture support       │
  │  │   ├── arm/              ← ARM-specific               │
  │  │   │   ├── core/aarch32/ ← Cortex-M/R                 │
  │  │   │   │   ├── swap_helper.S ← Context switch asm    │
  │  │   │   │   ├── isr_wrapper.S ← ISR wrapper           │
  │  │   │   │   ├── fault.c       ← Fault handlers        │
  │  │   │   │   └── mpu/         ← MPU driver             │
  │  │   │   └── core/aarch64/ ← Cortex-A (64-bit)         │
  │  │   ├── x86/              ← x86 support                │
  │  │   └── riscv/            ← RISC-V support             │
  │  │                                                       │
  │  ├── drivers/               ← Device drivers            │
  │  │   ├── serial/           ← UART drivers               │
  │  │   ├── gpio/             ← GPIO drivers               │
  │  │   ├── spi/              ← SPI drivers                │
  │  │   ├── i2c/              ← I2C drivers                │
  │  │   ├── sensor/           ← Sensor subsystem           │
  │  │   └── timer/            ← Timer drivers              │
  │  │                                                       │
  │  ├── boards/                ← Board definitions         │
  │  │   └── arm/                                            │
  │  │       ├── nucleo_f411re/ ← DTS + defconfig           │
  │  │       └── nrf52840dk/   ← Nordic DK board            │
  │  │                                                       │
  │  ├── soc/                   ← SoC definitions           │
  │  │   └── arm/                                            │
  │  │       ├── st_stm32/     ← STM32 family               │
  │  │       └── nordic_nrf/   ← Nordic nRF family          │
  │  │                                                       │
  │  ├── subsys/                ← Subsystems                 │
  │  │   ├── bluetooth/        ← BLE stack                  │
  │  │   ├── net/              ← Networking stack           │
  │  │   ├── fs/               ← File systems              │
  │  │   ├── logging/          ← Logging framework          │
  │  │   └── shell/            ← Interactive shell          │
  │  │                                                       │
  │  ├── lib/                   ← Libraries                  │
  │  │   ├── libc/             ← Minimal C library          │
  │  │   └── os/               ← OS utilities               │
  │  │                                                       │
  │  ├── include/               ← Public headers             │
  │  │   └── zephyr/                                         │
  │  │       ├── kernel.h      ← Main kernel API            │
  │  │       ├── device.h      ← Device driver API          │
  │  │       └── devicetree.h  ← Device tree accessors      │
  │  │                                                       │
  │  ├── dts/                   ← Device tree sources        │
  │  │   └── arm/              ← ARM SoC DTS files          │
  │  │       └── st/           ← STM32 DTS                  │
  │  │                                                       │
  │  ├── scripts/               ← Build scripts              │
  │  │   └── west_commands/    ← West tool plugins          │
  │  │                                                       │
  │  └── Kconfig                ← Root Kconfig               │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. FreeRTOS vs Zephyr Source Comparison

```
  ┌──────────────┬──────────────────┬────────────────────────┐
  │ Aspect       │ FreeRTOS         │ Zephyr                 │
  ├──────────────┼──────────────────┼────────────────────────┤
  │ Kernel size  │ ~9000 lines      │ ~30000 lines           │
  │ (core only)  │ (6 .c files)     │ (kernel/ directory)    │
  ├──────────────┼──────────────────┼────────────────────────┤
  │ Config       │ FreeRTOSConfig.h │ Kconfig + DTS          │
  │              │ (#define macros) │ (menuconfig GUI)       │
  ├──────────────┼──────────────────┼────────────────────────┤
  │ Drivers      │ User-written     │ Built-in (300+ drivers)│
  ├──────────────┼──────────────────┼────────────────────────┤
  │ Build system │ Any (Make/CMake) │ West + CMake (required)│
  ├──────────────┼──────────────────┼────────────────────────┤
  │ Port layer   │ 2 files per arch │ arch/ directory tree   │
  │              │ (port.c/h)       │ (400+ files)           │
  ├──────────────┼──────────────────┼────────────────────────┤
  │ Networking   │ Separate library │ Built-in (subsys/net/) │
  │              │ (FreeRTOS+TCP)   │                        │
  ├──────────────┼──────────────────┼────────────────────────┤
  │ Bluetooth    │ Not included     │ Full BLE stack included│
  ├──────────────┼──────────────────┼────────────────────────┤
  │ File download│ ~500KB           │ ~100MB (full tree)     │
  └──────────────┴──────────────────┴────────────────────────┘
```

---

## 5. VxWorks and QNX Source Layout

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  VxWorks (commercial, source available with license):    │
  │  vxworks/                                                │
  │  ├── target/                                             │
  │  │   ├── h/            ← Headers                        │
  │  │   ├── src/                                            │
  │  │   │   ├── wind/     ← Kernel (taskLib, semLib, etc.) │
  │  │   │   ├── os/       ← OS services                    │
  │  │   │   └── drv/      ← Drivers                        │
  │  │   ├── config/       ← BSP configurations             │
  │  │   └── lib/          ← Compiled libraries             │
  │  └── host/             ← Development tools               │
  │                                                           │
  │  QNX Neutrino (microkernel):                             │
  │  qnx/                                                    │
  │  ├── services/                                           │
  │  │   └── system/                                         │
  │  │       └── ker/      ← Microkernel source             │
  │  │           ├── nano_sched.c  ← Scheduler              │
  │  │           ├── nano_signal.c ← Signals                │
  │  │           ├── nano_sync.c   ← Sync primitives        │
  │  │           └── nano_xfer.c   ← Message passing        │
  │  ├── lib/                                                │
  │  │   └── c/            ← C library                      │
  │  └── hardware/                                           │
  │      └── startup/      ← BSP startup                    │
  │                                                           │
  │  QNX microkernel: only ~60KB binary                     │
  │  All services (filesystem, network, drivers) are         │
  │  separate user-space processes (resource managers)       │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. Reading RTOS Source Effectively

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  How to navigate FreeRTOS source (recommended path):     │
  │                                                           │
  │  1. Start with list.c (200 lines)                        │
  │     - Understand List_t, ListItem_t                      │
  │     - vListInsert() sorts by xItemValue                  │
  │     - All kernel data structures use these lists         │
  │                                                           │
  │  2. Read tasks.c: xTaskCreate()                          │
  │     - How TCB is allocated and initialized               │
  │     - How task is added to pxReadyTasksLists[]           │
  │                                                           │
  │  3. Read tasks.c: vTaskSwitchContext()                   │
  │     - taskSELECT_HIGHEST_PRIORITY_TASK macro             │
  │     - How pxCurrentTCB is updated                        │
  │                                                           │
  │  4. Read port.c: xPortPendSVHandler                      │
  │     - Assembly context switch                            │
  │     - Calls vTaskSwitchContext() in the middle           │
  │                                                           │
  │  5. Read queue.c: xQueueGenericSend()                    │
  │     - How data is copied into queue buffer               │
  │     - How blocked tasks are woken up                     │
  │     - How priority inheritance works for mutexes         │
  │                                                           │
  │  6. Read timers.c: prvTimerTask()                        │
  │     - Timer daemon task and command queue                │
  │     - How expired timers are processed                   │
  │                                                           │
  │  Total kernel: ~9000 lines — fully readable in a day    │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Walk through the FreeRTOS source files and explain what each one does.**
**A:** FreeRTOS kernel has 6 core source files: (1) **list.c** (~200 lines): implements sorted doubly-linked lists — the fundamental data structure. Ready lists, delayed lists, and blocked lists all use `List_t`. `vListInsert()` inserts sorted by `xItemValue` (used for priority/delay ticks). (2) **tasks.c** (~4000 lines): task management and scheduler. Contains `xTaskCreate()`, `vTaskDelay()`, `vTaskSwitchContext()`, the idle task, and the ready/delayed lists. The scheduler uses `uxTopReadyPriority` bitmap + CLZ for O(1) task selection. (3) **queue.c** (~2500 lines): implements queues (copy-by-value), binary/counting semaphores, and mutexes — all built on the same internal structure. Priority inheritance for mutexes is in `xQueueGenericSend()`/`xQueueGenericReceive()`. (4) **timers.c** (~1200 lines): software timer daemon task that processes timer commands from a queue and calls callback functions when timers expire. (5) **event_groups.c** (~800 lines): event flag groups with AND/OR wait combinations. (6) **stream_buffer.c** (~1000 lines): byte-stream and message buffers for single-reader/single-writer communication.

**Q2: How does the Zephyr source organization differ from FreeRTOS?**
**A:** FreeRTOS is minimal (9000 lines, 6 files) — just the kernel. Application provides drivers, network stack, and build system. Zephyr is a full embedded OS (~100MB source tree) with kernel, 300+ drivers, BLE stack, networking, file systems, logging, and shell — all in-tree. Key differences: (1) Config: FreeRTOS uses `#define` in FreeRTOSConfig.h; Zephyr uses Kconfig (menuconfig GUI) + Device Tree (hardware description). (2) Drivers: FreeRTOS has none; Zephyr has `drivers/` directory with implementations for UART, SPI, I2C, GPIO, sensors, etc. (3) Build: FreeRTOS works with any build system; Zephyr requires `west` + CMake. (4) Architecture support: FreeRTOS has 2 files per port; Zephyr has entire `arch/` directory tree with fault handlers, MPU drivers, context switch code. Zephyr is better for complex projects (BLE + networking); FreeRTOS is better for minimal, resource-constrained systems.

---

## Summary

- FreeRTOS: 6 core files (~9000 lines), fully readable — list.c → tasks.c → queue.c → timers.c
- Key file: tasks.c contains scheduler, task creation, context switch selection, idle task
- queue.c: unified implementation for queues, semaphores, and mutexes
- Zephyr: full OS tree (kernel/ + arch/ + drivers/ + subsys/) with Kconfig + DTS
- VxWorks: wind/ kernel with semLib/taskLib, commercial BSP model
- QNX: ~60KB microkernel (nano_*.c), all services as user-space resource managers
- Source reading path: list.c first (smallest, foundation), then tasks.c, queue.c

---

[Previous Chapter: Build Systems ←](Chapter_27_Build_Systems.md) | [Next Chapter: System Flow →](Chapter_29_System_Flow.md)
