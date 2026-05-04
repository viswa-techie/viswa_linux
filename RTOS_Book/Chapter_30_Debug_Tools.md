# Chapter 30: RTOS Debugging and Monitoring Tools

## Learning Goals
- Know the complete debugging tool ecosystem for RTOS development
- Master commercial and open-source trace analyzers
- Understand runtime monitoring and profiling tools
- Learn post-mortem analysis techniques

---

## 1. Debug Probe Ecosystem

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Hardware Probes:                                        │
  │  ┌───────────────┬──────────────────────────────────┐   │
  │  │ Probe         │ Features                         │   │
  │  ├───────────────┼──────────────────────────────────┤   │
  │  │ Segger J-Link │ Industry standard, fast, RTOS-   │   │
  │  │               │ aware, RTT, SystemView, unlimited│   │
  │  │               │ breakpoints (flash BP). $300-600 │   │
  │  │               │ EDU version: $60                 │   │
  │  ├───────────────┼──────────────────────────────────┤   │
  │  │ ST-Link/V3    │ STM32 only, cheap (on dev board)│   │
  │  │               │ SWD + SWO trace, USB-C           │   │
  │  │               │ Free with Nucleo/Discovery       │   │
  │  ├───────────────┼──────────────────────────────────┤   │
  │  │ CMSIS-DAP     │ Open standard, many vendors      │   │
  │  │ (DAPLink)     │ Drag-and-drop flash, USB HID     │   │
  │  │               │ Slower than J-Link               │   │
  │  ├───────────────┼──────────────────────────────────┤   │
  │  │ Lauterbach    │ Professional trace (ETM)         │   │
  │  │ TRACE32       │ Full instruction trace, RTOS     │   │
  │  │               │ awareness, $5000+                │   │
  │  ├───────────────┼──────────────────────────────────┤   │
  │  │ Black Magic   │ Open-source, GDB server built-in│   │
  │  │ Probe         │ No software needed on host       │   │
  │  └───────────────┴──────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. IDE and Debugger Tools

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  RTOS-Aware Debugging in IDEs:                           │
  │                                                           │
  │  ┌────────────────┬─────────────────────────────────┐   │
  │  │ IDE/Tool       │ RTOS Support                    │   │
  │  ├────────────────┼─────────────────────────────────┤   │
  │  │ Ozone (Segger) │ FreeRTOS, Zephyr, embOS,       │   │
  │  │                │ ThreadX — task view, timeline   │   │
  │  ├────────────────┼─────────────────────────────────┤   │
  │  │ STM32CubeIDE   │ FreeRTOS plugin: task list,    │   │
  │  │ (Eclipse)      │ queue viewer, stack analysis    │   │
  │  ├────────────────┼─────────────────────────────────┤   │
  │  │ VS Code +      │ Cortex-Debug extension: SVD     │   │
  │  │ Cortex-Debug   │ register view, ITM trace,      │   │
  │  │                │ RTOS thread awareness via GDB   │   │
  │  ├────────────────┼─────────────────────────────────┤   │
  │  │ IAR EWARM      │ Built-in RTOS awareness for    │   │
  │  │                │ FreeRTOS, ThreadX, embOS        │   │
  │  ├────────────────┼─────────────────────────────────┤   │
  │  │ Keil MDK       │ Event Recorder, Component Viewer│   │
  │  │ (uVision)      │ System Analyzer timeline       │   │
  │  ├────────────────┼─────────────────────────────────┤   │
  │  │ GDB + OpenOCD  │ FreeRTOS thread awareness      │   │
  │  │ (command line)  │ "info threads" shows tasks     │   │
  │  └────────────────┴─────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Trace Analysis Tools

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌─────────────────┬────────────────────────────────┐   │
  │  │ Tool            │ Purpose                        │   │
  │  ├─────────────────┼────────────────────────────────┤   │
  │  │ Segger SystemView│ Real-time task/ISR timeline   │   │
  │  │                  │ via J-Link RTT. Free.         │   │
  │  ├─────────────────┼────────────────────────────────┤   │
  │  │ Percepio        │ Deep RTOS analysis: task       │   │
  │  │ Tracealyzer     │ timing, CPU load, queue usage, │   │
  │  │                 │ actor/instance view. Commercial│   │
  │  ├─────────────────┼────────────────────────────────┤   │
  │  │ TRACE32         │ Full ETM instruction trace     │   │
  │  │ (Lauterbach)    │ code coverage, profiling       │   │
  │  ├─────────────────┼────────────────────────────────┤   │
  │  │ ARM DS-5 /      │ ETM/ETB trace decode,         │   │
  │  │ Development     │ profiling, code coverage       │   │
  │  │ Studio          │                                │   │
  │  ├─────────────────┼────────────────────────────────┤   │
  │  │ Orbuculum       │ Open-source SWO/ITM/ETM       │   │
  │  │                 │ decoder. Linux/Mac. Free.      │   │
  │  └─────────────────┴────────────────────────────────┘   │
  │                                                           │
  │  Percepio Tracealyzer views:                             │
  │  ┌─────────────────────────────────────────────┐        │
  │  │ 1. Task Trace: execution timeline per task   │        │
  │  │ 2. CPU Load Graph: load over time per task   │        │
  │  │ 3. Response Time: event → task response      │        │
  │  │ 4. Communication Flow: queue/sem interactions│        │
  │  │ 5. Actor Instance: per-object history        │        │
  │  │ 6. User Events: custom markers               │        │
  │  └─────────────────────────────────────────────┘        │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Runtime Monitoring

```c
/* Runtime monitoring pattern: shell-based status */

/* Segger RTT: printf without UART, via debug probe */
#include "SEGGER_RTT.h"

void vMonitorTask(void *pvParameters) {
    char buf[512];
    HeapStats_t heap_stats;

    for (;;) {
        /* Task status */
        vTaskList(buf);
        SEGGER_RTT_printf(0, "\n--- Task List ---\n%s\n", buf);

        /* Runtime stats */
        vTaskGetRunTimeStats(buf);
        SEGGER_RTT_printf(0, "--- CPU Usage ---\n%s\n", buf);

        /* Heap statistics (FreeRTOS v10.5+) */
        vPortGetHeapStats(&heap_stats);
        SEGGER_RTT_printf(0, "--- Heap ---\n");
        SEGGER_RTT_printf(0, "Free: %u bytes\n",
                          heap_stats.xAvailableHeapSpaceInBytes);
        SEGGER_RTT_printf(0, "Min ever free: %u bytes\n",
                          heap_stats.xMinimumEverFreeBytesRemaining);
        SEGGER_RTT_printf(0, "Allocs: %u  Frees: %u\n",
                          heap_stats.xNumberOfSuccessfulAllocations,
                          heap_stats.xNumberOfSuccessfulFrees);

        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}

/* Watchpoint-based memory monitoring */
/*
 * DWT Data Watchpoint (Cortex-M3+):
 * - Monitor read/write to specific address
 * - No breakpoint needed (runs at full speed)
 * - Triggers debug event on access
 *
 * Use case: find what corrupts a variable
 */
void dwt_set_watchpoint(uint32_t addr) {
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
    DWT->COMP0 = addr;
    DWT->MASK0 = 0;          /* Exact match */
    DWT->FUNCTION0 = 0x6;    /* Trigger on write access */
    /* Debugger will halt when anything writes to addr */
}
```

---

## 5. Post-Mortem Analysis

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  When device crashes in the field (no debugger):         │
  │                                                           │
  │  1. Crash dump to persistent storage:                    │
  │     Hard fault handler writes registers + stack to       │
  │     flash/EEPROM before reset                            │
  │                                                           │
  │  2. Core dump structure:                                 │
  │     ┌─────────────────────────────────────────┐         │
  │     │ Reset reason (WDT, fault, power-on)     │         │
  │     │ Stacked registers (R0-R3, R12, LR, PC)  │         │
  │     │ Fault status (CFSR, HFSR, BFAR, MMFAR)  │         │
  │     │ Current task name + TCB address           │         │
  │     │ Stack memory (top 256 bytes)              │         │
  │     │ Tick count at crash                        │         │
  │     │ Heap free/used at crash                    │         │
  │     │ Custom: last N events from ring buffer    │         │
  │     └─────────────────────────────────────────┘         │
  │                                                           │
  │  3. Analysis workflow:                                   │
  │     a. Read crash dump from device (UART/USB)            │
  │     b. Use addr2line to map PC/LR to source              │
  │     c. Decode CFSR bits for fault type                   │
  │     d. Check stack for overflow (canary value)           │
  │     e. Review event ring buffer for last operations      │
  └──────────────────────────────────────────────────────────┘
```

```c
/* Crash dump implementation */
typedef struct __attribute__((packed)) {
    uint32_t magic;          /* 0xDEADC0DE = valid dump */
    uint32_t reset_reason;
    uint32_t pc;
    uint32_t lr;
    uint32_t cfsr;
    uint32_t hfsr;
    uint32_t bfar;
    uint32_t mmfar;
    uint32_t tick_count;
    char task_name[16];
    uint32_t stack_dump[64]; /* Top 256 bytes of stack */
} CrashDump_t;

/* Located in battery-backed SRAM or specific flash sector */
__attribute__((section(".crash_dump")))
static CrashDump_t crash_dump;

void save_crash_dump(uint32_t *stack_frame) {
    crash_dump.magic = 0xDEADC0DE;
    crash_dump.pc = stack_frame[6];
    crash_dump.lr = stack_frame[5];
    crash_dump.cfsr = SCB->CFSR;
    crash_dump.hfsr = SCB->HFSR;
    crash_dump.bfar = SCB->BFAR;
    crash_dump.mmfar = SCB->MMFAR;
    crash_dump.tick_count = xTaskGetTickCountFromISR();

    if (pxCurrentTCB) {
        strncpy(crash_dump.task_name,
                pxCurrentTCB->pcTaskName, 15);
    }

    /* Copy top of stack */
    memcpy(crash_dump.stack_dump, stack_frame,
           sizeof(crash_dump.stack_dump));

    __NVIC_SystemReset();
}

/* On next boot, check for crash dump */
void check_crash_dump(void) {
    if (crash_dump.magic == 0xDEADC0DE) {
        printf("Previous crash: PC=0x%08lX LR=0x%08lX Task=%s\n",
               crash_dump.pc, crash_dump.lr, crash_dump.task_name);
        /* Upload crash dump to server if connected */
        crash_dump.magic = 0;  /* Clear after reading */
    }
}
```

---

## 6. Tool Selection Guide

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Project Stage      │ Recommended Tools                  │
  │  ───────────────────┼──────────────────────────────────  │
  │  Prototyping        │ ST-Link/V3 + STM32CubeIDE         │
  │                     │ printf via UART/SWO                │
  │  ───────────────────┼──────────────────────────────────  │
  │  Development        │ J-Link + VS Code (Cortex-Debug)   │
  │                     │ SystemView for task timing         │
  │                     │ RTT for printf                     │
  │  ───────────────────┼──────────────────────────────────  │
  │  Integration test   │ J-Link + Tracealyzer              │
  │                     │ Logic analyzer for protocols      │
  │                     │ Oscilloscope for timing            │
  │  ───────────────────┼──────────────────────────────────  │
  │  Production debug   │ Crash dump + post-mortem           │
  │                     │ Event ring buffer logging          │
  │                     │ Watchdog reset counter             │
  │  ───────────────────┼──────────────────────────────────  │
  │  Safety-critical    │ Lauterbach TRACE32 (ETM trace)    │
  │  certification      │ Code coverage analysis             │
  │                     │ Static analysis (PC-lint, Polyspace)│
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: What tools would you use to debug a timing issue in an RTOS system?**
**A:** Layered approach: (1) **First pass — GPIO toggle + oscilloscope**: toggle a GPIO pin at task entry/exit. Oscilloscope shows exact timing, jitter, and period. Zero software overhead. (2) **Detailed analysis — SystemView**: instrument FreeRTOS trace hooks, view timeline of all tasks/ISRs. Identifies which task/ISR is causing delays, preemptions, or priority inversions. Connects via J-Link RTT (~1us overhead per event). (3) **Precise measurement — DWT cycle counter**: read `DWT->CYCCNT` at critical points. Gives cycle-accurate timing (1/168MHz = 6ns resolution). Log min/max/average to detect worst-case scenarios. (4) **Full instruction trace — ETM** (if available): reconstruct complete execution path. Find where CPU spent time. Requires Lauterbach TRACE32 or ARM DS-5. Expensive but zero overhead. (5) **Percepio Tracealyzer**: commercial tool with response time analysis — measures time from event (ISR) to task response, plots distribution over time.

**Q2: How do you implement post-mortem crash analysis for a deployed RTOS device?**
**A:** (1) Hard fault handler saves crash state to persistent memory (battery-backed SRAM, dedicated flash sector, or EEPROM): stacked registers (PC = faulting instruction, LR = caller), fault status registers (CFSR, HFSR, BFAR), current task name from TCB, tick count, top of stack memory, heap statistics. (2) Use a magic number (e.g., 0xDEADC0DE) to mark valid dump. (3) On next boot, check for valid crash dump — upload to backend server if connectivity exists, or display via diagnostic UART. (4) For persistent logging: maintain a circular event buffer (last 100 events with timestamps). Store in non-volatile memory before reset. This provides context leading up to the crash. (5) Analysis: use `arm-none-eabi-addr2line -e firmware.elf <PC>` to map faulting address to source file:line. Decode CFSR bits to identify fault type (bus fault, memory violation, usage fault).

---

## Summary

- Debug probes: J-Link (best general), ST-Link (STM32-specific), Lauterbach (professional ETM trace)
- IDEs: STM32CubeIDE, VS Code + Cortex-Debug, Ozone, IAR, Keil — all support RTOS awareness
- Trace tools: SystemView (free, J-Link RTT), Tracealyzer (commercial, deep analysis), TRACE32 (ETM)
- Runtime monitoring: vTaskList(), vPortGetHeapStats(), RTT printf, DWT watchpoints
- Post-mortem: crash dump to persistent storage, addr2line for PC→source mapping
- Tool progression: GPIO toggle → SystemView → Tracealyzer → ETM (cost and detail increase)

---

[Previous Chapter: System Flow ←](Chapter_29_System_Flow.md) | [Next Chapter: RTOS in Domains →](Chapter_31_Domains.md)
