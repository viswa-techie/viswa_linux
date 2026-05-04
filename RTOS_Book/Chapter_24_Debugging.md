# Chapter 24: RTOS Debugging

## Learning Goals
- Master JTAG/SWD debugging for embedded RTOS systems
- Learn RTOS-aware debugging: task lists, stack inspection, queue state
- Understand trace debugging: SWO, ITM, ETM, SystemView
- Know printf debugging trade-offs and semihosting

---

## 1. Debug Interface Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Host PC                    Target MCU                   │
  │  ┌──────────┐              ┌──────────────────┐         │
  │  │ GDB      │◄── TCP ────►│ Debug Access Port │         │
  │  │ (or IDE) │              │ (DAP)            │         │
  │  └──────────┘              └────────┬─────────┘         │
  │       │                             │                    │
  │  ┌──────────┐              ┌────────▼─────────┐         │
  │  │ OpenOCD  │◄── USB ────►│ JTAG/SWD Probe   │         │
  │  │ J-Link   │              │ (J-Link, ST-Link)│         │
  │  │ pyOCD    │              └──────────────────┘         │
  │  └──────────┘                                            │
  │                                                           │
  │  JTAG vs SWD:                                            │
  │  ┌──────────┬──────────────┬──────────────────┐         │
  │  │          │ JTAG         │ SWD              │         │
  │  ├──────────┼──────────────┼──────────────────┤         │
  │  │ Pins     │ 4 (TCK,TMS, │ 2 (SWDIO, SWCLK)│         │
  │  │          │  TDI,TDO)   │                  │         │
  │  │ Speed    │ Up to 50MHz │ Up to 50MHz      │         │
  │  │ Daisy    │ Yes (chain) │ No               │         │
  │  │ Trace    │ Separate ETM│ SWO (1 pin)      │         │
  │  │ ARM only │ No (generic)│ Yes (ARM only)   │         │
  │  │ Use case │ Multi-core  │ Single Cortex-M  │         │
  │  └──────────┴──────────────┴──────────────────┘         │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. RTOS-Aware Debugging

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Standard debugger sees: one thread of execution         │
  │  RTOS-aware debugger sees: ALL tasks, their states,      │
  │  stacks, owned mutexes, pending queues                   │
  │                                                           │
  │  RTOS-aware features:                                    │
  │  ┌─────────────────────────────────────────────┐        │
  │  │ 1. Task list with states (Ready/Blocked/etc)│        │
  │  │ 2. Stack usage per task (HWM = High Water)  │        │
  │  │ 3. Queue contents and waiting tasks          │        │
  │  │ 4. Mutex ownership chain                     │        │
  │  │ 5. Timer list (active software timers)       │        │
  │  │ 6. Heap usage statistics                     │        │
  │  │ 7. Switch to any task's stack frame          │        │
  │  └─────────────────────────────────────────────┘        │
  │                                                           │
  │  Setup requires:                                         │
  │  - RTOS plugin for debugger (J-Link GDB Server,         │
  │    OpenOCD FreeRTOS thread awareness)                    │
  │  - configUSE_TRACE_FACILITY = 1 in FreeRTOSConfig.h     │
  │  - configRECORD_STACK_HIGH_ADDRESS = 1                   │
  │  - Symbol information in ELF (not stripped)              │
  └──────────────────────────────────────────────────────────┘
```

```c
/* FreeRTOS runtime stats for debugging */
/* FreeRTOSConfig.h */
#define configUSE_TRACE_FACILITY             1
#define configUSE_STATS_FORMATTING_FUNCTIONS 1
#define configGENERATE_RUN_TIME_STATS        1
#define portCONFIGURE_TIMER_FOR_RUN_TIME_STATS() init_runtime_timer()
#define portGET_RUN_TIME_COUNTER_VALUE()         get_runtime_timer()

/* Print task status from a debug task */
void vDebugTask(void *pvParameters) {
    char buf[2048];
    for (;;) {
        /* Format: Name | State | Prio | Stack(HWM) | TaskNum */
        vTaskList(buf);
        printf("Task List:\n%s\n", buf);

        /* Format: Name | AbsTime | %%Time */
        vTaskGetRunTimeStats(buf);
        printf("Runtime Stats:\n%s\n", buf);

        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}

/*
 * Example output:
 *
 * Task List:
 * Name          State  Prio  Stack  Num
 * SensorTask    R      3     128    1
 * MotorTask     B      4     96     2
 * CommTask      B      2     256    3
 * IDLE          R      0     64     4
 *
 * Runtime Stats:
 * Name          Abs Time   % Time
 * SensorTask    45000      15%
 * MotorTask     12000      4%
 * CommTask      30000      10%
 * IDLE          213000     71%
 */
```

---

## 3. Trace Debugging

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ARM CoreSight Trace Components:                         │
  │                                                           │
  │  ┌──────────┐   ┌──────────┐   ┌──────────┐           │
  │  │ ITM      │   │ DWT      │   │ ETM      │           │
  │  │ Instrumen│   │ Data     │   │ Embedded │           │
  │  │ Trace    │   │ Watchpt  │   │ Trace    │           │
  │  │ Macrocell│   │ & Trace  │   │ Macrocell│           │
  │  └────┬─────┘   └────┬─────┘   └────┬─────┘           │
  │       │               │               │                  │
  │       └───────────────┼───────────────┘                  │
  │                       ▼                                   │
  │               ┌──────────────┐                           │
  │               │ TPIU         │  Trace Port Interface    │
  │               │ (formatter)  │                           │
  │               └──────┬───────┘                           │
  │                      │                                    │
  │            ┌─────────┴──────────┐                        │
  │            ▼                    ▼                         │
  │      ┌──────────┐       ┌──────────┐                    │
  │      │ SWO pin  │       │ Trace    │                    │
  │      │ (1 wire) │       │ Port     │                    │
  │      │ ~2Mbit/s │       │ (4 pins) │                    │
  │      │ Low BW   │       │ ~1Gbit/s │                    │
  │      └──────────┘       └──────────┘                    │
  │                                                           │
  │  ITM: printf-style trace (low overhead, <1 cycle)       │
  │  DWT: data watchpoints, cycle counter, exception trace  │
  │  ETM: full instruction trace (every instruction logged) │
  └──────────────────────────────────────────────────────────┘
```

```c
/* ITM stimulus port for trace output (non-blocking) */
/* Much better than UART printf — negligible CPU overhead */

#define ITM_STIM_PORT0  (*(volatile uint32_t *)0xE0000000)
#define ITM_TER         (*(volatile uint32_t *)0xE0000E00)
#define ITM_TCR         (*(volatile uint32_t *)0xE0000E80)

void itm_putchar(char c) {
    if ((ITM_TCR & 1) && (ITM_TER & 1)) {
        while (ITM_STIM_PORT0 == 0);  /* Wait for FIFO ready */
        ITM_STIM_PORT0 = c;
    }
}

/* CMSIS standard: ITM_SendChar() does the same */
int itm_printf(const char *fmt, ...) {
    char buf[128];
    va_list args;
    va_start(args, fmt);
    int len = vsnprintf(buf, sizeof(buf), fmt, args);
    va_end(args);
    for (int i = 0; i < len; i++) itm_putchar(buf[i]);
    return len;
}

/* DWT cycle counter for precise timing */
void dwt_init(void) {
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
    DWT->CYCCNT = 0;
    DWT->CTRL |= DWT_CTRL_CYCCNTENA_Msk;
}

uint32_t dwt_get_cycles(void) {
    return DWT->CYCCNT;
}

/* Measure ISR latency */
void EXTI0_IRQHandler(void) {
    uint32_t entry = DWT->CYCCNT;
    /* ... handle interrupt ... */
    uint32_t exit = DWT->CYCCNT;
    uint32_t cycles = exit - entry;
    /* Store for analysis: cycles / SystemCoreClock = time */
}
```

---

## 4. Segger SystemView

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  SystemView: real-time RTOS visualization tool           │
  │                                                           │
  │  ┌─────────────────────────────────────────────────┐    │
  │  │ Timeline View:                                   │    │
  │  │                                                   │    │
  │  │ Task A  ████░░░░████░░░░████░░░░████             │    │
  │  │ Task B  ░░░░████░░░░░░░░░░░░████░░░░             │    │
  │  │ Task C  ░░░░░░░░░░░░████░░░░░░░░░░░░             │    │
  │  │ ISR     │▌░░░│▌░░░│▌░░░│▌░░░│▌░░░│▌             │    │
  │  │ Idle    ░░░░░░░░░░░░░░░░░░░░░░░░░░░░             │    │
  │  │         0ms    5ms    10ms   15ms   20ms         │    │
  │  └─────────────────────────────────────────────────┘    │
  │                                                           │
  │  Setup:                                                  │
  │  1. Add SEGGER_SYSVIEW files to project                 │
  │  2. Configure SYSVIEW_CONF.h with RTOS hooks            │
  │  3. Instrument FreeRTOS: traceTASK_SWITCHED_IN/OUT      │
  │  4. Connect via J-Link RTT (Real-Time Transfer)         │
  │  5. View in SystemView PC application                   │
  │                                                           │
  │  Metrics visible:                                        │
  │  - Task execution time, preemptions, blocking            │
  │  - ISR entry/exit timing and nesting                     │
  │  - Queue/semaphore/mutex operations                      │
  │  - CPU load per task                                     │
  │  - Custom markers and log messages                       │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Common RTOS Debugging Scenarios

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Problem              │ Debug Approach                   │
  │  ─────────────────────┼───────────────────────────────── │
  │  Hard fault           │ Decode CFSR/HFSR/MMFAR/BFAR     │
  │                       │ Check stacked PC for fault addr  │
  │                       │ Examine task stack for corruption│
  │  ─────────────────────┼───────────────────────────────── │
  │  Stack overflow       │ Enable configCHECK_FOR_STACK_OVF │
  │                       │ Check HWM in vTaskList output    │
  │                       │ Enable MPU stack guard region    │
  │  ─────────────────────┼───────────────────────────────── │
  │  Deadlock             │ Inspect mutex owners in debugger │
  │                       │ Check all Blocked tasks and what │
  │                       │ resources they are waiting for   │
  │  ─────────────────────┼───────────────────────────────── │
  │  Priority inversion   │ SystemView timeline — see low    │
  │                       │ prio task holding resource while │
  │                       │ high prio task is Blocked        │
  │  ─────────────────────┼───────────────────────────────── │
  │  Missed deadline      │ DWT timestamp at task start/end  │
  │                       │ Compare against required period  │
  │  ─────────────────────┼───────────────────────────────── │
  │  Memory corruption    │ Enable MPU regions per task      │
  │                       │ Use heap canaries (heap_5 ext)   │
  │                       │ Watch memory breakpoint on addr  │
  │  ─────────────────────┼───────────────────────────────── │
  │  ISR too long         │ DWT cycle count in ISR           │
  │                       │ Defer work to task (FromISR API) │
  └──────────────────────────────────────────────────────────┘
```

```c
/* Hard Fault handler for Cortex-M: extract fault info */
void HardFault_Handler(void) {
    __asm volatile (
        "TST   LR, #4          \n"  /* Check EXC_RETURN bit 2 */
        "ITE   EQ               \n"
        "MRSEQ R0, MSP          \n"  /* Fault from MSP */
        "MRSNE R0, PSP          \n"  /* Fault from PSP (task) */
        "B     hard_fault_diag  \n"
    );
}

void hard_fault_diag(uint32_t *stack_frame) {
    /* Stacked registers from exception entry */
    volatile uint32_t r0  = stack_frame[0];
    volatile uint32_t r1  = stack_frame[1];
    volatile uint32_t r2  = stack_frame[2];
    volatile uint32_t r3  = stack_frame[3];
    volatile uint32_t r12 = stack_frame[4];
    volatile uint32_t lr  = stack_frame[5];  /* Return address */
    volatile uint32_t pc  = stack_frame[6];  /* Faulting instruction */
    volatile uint32_t psr = stack_frame[7];

    /* Fault status registers */
    volatile uint32_t cfsr = SCB->CFSR;  /* Combined FSR */
    volatile uint32_t hfsr = SCB->HFSR;  /* Hard Fault SR */
    volatile uint32_t mmfar = SCB->MMFAR; /* MemManage addr */
    volatile uint32_t bfar = SCB->BFAR;   /* BusFault addr */

    /* Log and halt — PC shows the faulting instruction */
    printf("HARD FAULT at PC=0x%08lX LR=0x%08lX\n", pc, lr);
    printf("CFSR=0x%08lX HFSR=0x%08lX\n", cfsr, hfsr);
    if (cfsr & SCB_CFSR_MMARVALID_Msk)
        printf("MemManage fault at 0x%08lX\n", mmfar);
    if (cfsr & SCB_CFSR_BFARVALID_Msk)
        printf("BusFault at 0x%08lX\n", bfar);

    for (;;);  /* Halt for debugger */
}
```

---

## 6. Printf Debugging vs Trace

```
  ┌──────────┬─────────────────┬────────────────────────────┐
  │ Method   │ Overhead        │ Use Case                   │
  ├──────────┼─────────────────┼────────────────────────────┤
  │ UART     │ ~1ms per line   │ Slow debug, no timing      │
  │ printf   │ Blocks CPU      │ critical code              │
  ├──────────┼─────────────────┼────────────────────────────┤
  │ ITM/SWO  │ ~10 cycles/byte │ Real-time trace, timing OK │
  │          │ Non-blocking    │ Use for ISR/task tracing   │
  ├──────────┼─────────────────┼────────────────────────────┤
  │ RTT      │ ~1us per msg    │ Fastest printf alternative │
  │ (Segger) │ RAM buffer      │ No pin needed (via J-Link) │
  ├──────────┼─────────────────┼────────────────────────────┤
  │ GPIO     │ ~10ns toggle    │ Scope trigger, ISR timing  │
  │ toggle   │                 │ Measure with oscilloscope  │
  ├──────────┼─────────────────┼────────────────────────────┤
  │ ETM      │ Zero CPU        │ Full instruction trace     │
  │          │ (hardware)      │ Needs trace probe ($$$$)   │
  └──────────┴─────────────────┴────────────────────────────┘
```

---

## Interview Questions

**Q1: How do you debug a hard fault on a Cortex-M running FreeRTOS?**
**A:** (1) Hard fault handler extracts the stacked exception frame from PSP (if fault from task context, determined by EXC_RETURN bit 2). The stacked PC register points to the faulting instruction. (2) Read SCB->CFSR to determine fault type: UsageFault (undefined instruction, unaligned access, divide by zero), BusFault (invalid bus access, stacking error), MemManageFault (MPU violation). (3) If BFARVALID/MMARVALID bits are set, SCB->BFAR/MMFAR gives the faulting address. (4) Use `addr2line -e firmware.elf <PC value>` to map PC to source file:line. (5) Common root causes: stack overflow (PC in random location), null pointer dereference (BFAR=0x0000000X), writing to flash without unlock, accessing peripheral without clock enabled. (6) Enable `configCHECK_FOR_STACK_OVERFLOW` to catch stack overflows before they cause hard faults.

**Q2: What is Segger SystemView and how does it help RTOS debugging?**
**A:** SystemView is a real-time recording and visualization tool for RTOS behavior. It instruments FreeRTOS trace hooks (`traceTASK_SWITCHED_IN`, `traceTASK_SWITCHED_OUT`, `traceQUEUE_SEND`, etc.) to record events with timestamps into a RAM ring buffer. Data is streamed to the host PC via J-Link RTT (Real-Time Transfer) — a RAM-based bidirectional channel that requires only a debug probe, no extra pins. The PC application shows: (1) Timeline: visual bar chart of which task/ISR ran when, with preemptions visible. (2) CPU load per task. (3) ISR entry/exit timing and nesting. (4) IPC operations (queue send/receive, mutex take/give). This makes priority inversion, deadline misses, and CPU hogging immediately visible. Overhead: ~1us per event, configurable buffer size, events are dropped if buffer overflows (not blocking).

---

## Summary

- JTAG: 4-pin, multi-core/daisy-chain; SWD: 2-pin, ARM only, simpler
- RTOS-aware debugging: task list, stack HWM, queue/mutex state visible in debugger
- Trace: ITM (printf via SWO, low overhead), DWT (cycle counter, watchpoints), ETM (full instruction trace)
- SystemView: timeline visualization of task/ISR execution, CPU load, IPC via J-Link RTT
- Hard fault diagnosis: stacked PC + CFSR/HFSR/BFAR/MMFAR → `addr2line` → root cause
- Printf alternatives ranked by overhead: GPIO toggle < RTT < ITM/SWO < UART printf

---

[Previous Chapter: Security ←](Chapter_23_Security.md) | [Next Chapter: Performance Optimization →](Chapter_25_Performance.md)
