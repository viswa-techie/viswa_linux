# Chapter 22: Safety and Reliability

## Learning Goals
- Understand safety standards for RTOS (IEC 61508, ISO 26262, DO-178C)
- Learn safety mechanisms: redundancy, watchdogs, error detection
- Know RTOS certification and qualification requirements
- Master fault-tolerant design patterns

---

## 1. Safety Standards Overview

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Standard       │ Domain        │ Levels                 │
  │  ────────────────┼───────────────┼───────────────────── │
  │  IEC 61508      │ General       │ SIL 1-4               │
  │  ISO 26262      │ Automotive    │ ASIL A-D              │
  │  DO-178C        │ Aviation      │ DAL A-E               │
  │  IEC 62304      │ Medical       │ Class A-C             │
  │  EN 50128       │ Railway       │ SIL 0-4               │
  │                                                           │
  │  RTOS requirements by safety level:                      │
  │  ┌──────────┬──────────────────────────────────────┐    │
  │  │ SIL 1-2  │ Code review, testing, static analysis│    │
  │  │ (ASIL A-B)│ Dynamic allocation allowed with care│    │
  │  ├──────────┼──────────────────────────────────────┤    │
  │  │ SIL 3-4  │ Formal methods, MCDC coverage        │    │
  │  │ (ASIL C-D)│ No dynamic allocation               │    │
  │  │          │ Static stack analysis required        │    │
  │  │          │ Certified RTOS or qualified           │    │
  │  │          │ Watchdog, dual-core lockstep          │    │
  │  └──────────┴──────────────────────────────────────┘    │
  │                                                           │
  │  Pre-certified RTOS:                                     │
  │  ┌─────────────┬───────────────────────────────────┐    │
  │  │ SafeRTOS     │ IEC 61508 SIL 3, ISO 26262 ASIL D│    │
  │  │ QNX Neutrino │ IEC 61508 SIL 3, ISO 26262 ASIL D│    │
  │  │ VxWorks 653  │ DO-178C DAL A (aviation)          │    │
  │  │ INTEGRITY    │ DO-178C DAL A, EAL 6+             │    │
  │  │ PikeOS       │ SIL 4, DAL A, EAL 3+             │    │
  │  │ Zephyr       │ IEC 61508 SIL 3 (ongoing)        │    │
  │  └─────────────┴───────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Safety Mechanisms

```c
/* Dual-core lockstep: two cores run same code in sync */
/*
 * ARM Cortex-R5 lockstep mode:
 * ┌────────┐     ┌────────┐
 * │ Core 0 │     │ Core 1 │  (runs same instructions)
 * │ Output ├────►│ Compare├──► Match? → Continue
 * └────────┘     └────────┘   Mismatch? → Fault!
 *
 * Hardware detects: bit flips, transient faults, silicon errors
 * Used in: automotive (AUTOSAR ASIL D), industrial SIL 3
 */

/* Stack canary and runtime checks */
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
    /* Log fault and enter safe state */
    safety_log_fault(FAULT_STACK_OVERFLOW, pcTaskName);
    enter_safe_state();  /* Shut down actuators, signal fault */
}

/* Memory error detection with ECC */
/*
 * ECC (Error Correcting Code) RAM:
 * - SECDED: Single-bit Error Correct, Double-bit Error Detect
 * - Hardware corrects 1-bit errors transparently
 * - Hardware detects 2-bit errors → NMI/BusFault
 * - Required for ASIL C/D and SIL 3/4
 * - Available on: Cortex-R, some Cortex-M (STM32H7)
 */

void BusFault_Handler(void) {
    /* ECC double-bit error or access violation */
    uint32_t cfsr = SCB->CFSR;
    safety_log_fault(FAULT_MEMORY_ECC, cfsr);
    enter_safe_state();
}

/* Alive monitoring pattern (functional watchdog) */
typedef struct {
    uint32_t expected_sequence;
    uint32_t actual_sequence;
    uint32_t deadline_ticks;
    TickType_t last_report;
} AliveMonitor_t;

static AliveMonitor_t monitors[NUM_SAFETY_TASKS];

void safety_report_alive(uint32_t task_id, uint32_t sequence) {
    monitors[task_id].actual_sequence = sequence;
    monitors[task_id].last_report = xTaskGetTickCount();
}

void vSafetyMonitorTask(void *pvParameters) {
    for (;;) {
        for (int i = 0; i < NUM_SAFETY_TASKS; i++) {
            TickType_t elapsed = xTaskGetTickCount() - monitors[i].last_report;

            if (elapsed > monitors[i].deadline_ticks) {
                safety_log_fault(FAULT_TASK_TIMEOUT, i);
                enter_safe_state();
            }
            if (monitors[i].actual_sequence != monitors[i].expected_sequence) {
                safety_log_fault(FAULT_WRONG_SEQUENCE, i);
                enter_safe_state();
            }
            monitors[i].expected_sequence++;
        }
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

---

## 3. SafeRTOS vs FreeRTOS

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  FreeRTOS: open source, MIT license, not safety-certified│
  │  SafeRTOS: derived from FreeRTOS, pre-certified          │
  │                                                           │
  │  ┌──────────────┬──────────────┬────────────────┐       │
  │  │              │ FreeRTOS     │ SafeRTOS       │       │
  │  ├──────────────┼──────────────┼────────────────┤       │
  │  │ License      │ MIT (free)   │ Commercial     │       │
  │  │ Certification│ None         │ SIL 3, ASIL D  │       │
  │  │ API          │ Same         │ Same (subset)  │       │
  │  │ Error checks │ configASSERT │ Validated      │       │
  │  │ Source       │ Open         │ Provided with  │       │
  │  │              │              │ safety manual  │       │
  │  │ Config       │ FreeRTOSConfig│ Static only   │       │
  │  │ Dynamic alloc│ Yes          │ No (static)   │       │
  │  │ Testing      │ Community    │ MCDC coverage │       │
  │  │ Documentation│ Community    │ Safety manual  │       │
  │  └──────────────┴──────────────┴────────────────┘       │
  │                                                           │
  │  SafeRTOS removes: dynamic allocation, configurable      │
  │  features, and adds: parameter validation on every API   │
  │  call, proven bounded execution time per API call.       │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Fault-Tolerant Design Patterns

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  1. Defensive programming:                               │
  │     - Validate all inputs at module boundaries           │
  │     - Assert on invariants                               │
  │     - Check return values of all RTOS API calls          │
  │                                                           │
  │  2. Fail-safe state:                                     │
  │     - Define safe output states for all actuators        │
  │     - On fault: disable motors, close valves, brake      │
  │     - Safe state must be achievable from any error       │
  │                                                           │
  │  3. Redundancy patterns:                                 │
  │     - Dual-channel: two independent paths, compare       │
  │     - Triple Modular Redundancy (TMR): majority vote     │
  │     - Diverse redundancy: different algorithms/HW        │
  │                                                           │
  │  4. Error recovery:                                      │
  │     - Restart failed task (if isolation available)        │
  │     - QNX: process restart without OS reboot             │
  │     - Checkpoint/restore for state recovery               │
  │                                                           │
  │  5. Temporal monitoring:                                 │
  │     - Deadline monitoring per task                        │
  │     - Execution time monitoring (overrun detection)      │
  │     - Sequence monitoring (correct execution order)      │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: What makes SafeRTOS different from FreeRTOS for safety-critical systems?**
**A:** SafeRTOS is derived from FreeRTOS design but independently implemented to meet IEC 61508 SIL 3. Key differences: (1) All API calls validate parameters and return error codes (FreeRTOS uses `configASSERT` which is debug-only). (2) No dynamic memory allocation — all objects created statically. (3) Bounded, documented worst-case execution time for every API call. (4) MC/DC test coverage with full traceability. (5) Supplied with safety manual, design documentation, test reports. (6) Certified by TUV SUD. (7) Reduced, fixed-configuration kernel (no runtime configuration). FreeRTOS is used during development and then replaced with SafeRTOS for production in safety-critical systems — the API is compatible.

**Q2: How does a safety monitor detect task failures in an RTOS?**
**A:** A safety monitor task runs at highest priority and checks three things: (1) Liveness — each monitored task must call `report_alive()` within its deadline period. If not, the task is stuck (deadlock, infinite loop, or crash). (2) Sequence — each task reports a sequence number. The monitor verifies the sequence matches expected order, detecting execution flow errors. (3) Timing — execution time monitoring detects tasks that overrun their WCET. On any violation: feed the hardware watchdog is withheld (causing reset), or the system enters a pre-defined safe state (actuators to known-safe outputs). This pattern catches: deadlocks, stack overflows (if they corrupt other memory), priority inversions that cause deadline misses, and hardware faults that corrupt task execution.

---

## Summary

- Safety standards: IEC 61508 (SIL), ISO 26262 (ASIL), DO-178C (DAL) define RTOS requirements
- SafeRTOS: pre-certified (SIL 3, ASIL D), static allocation only, validated API
- Safety mechanisms: lockstep cores, ECC RAM, watchdogs, stack canaries, MPU
- Fault-tolerant design: fail-safe state, redundancy, temporal monitoring
- Higher safety levels require: formal methods, MCDC coverage, no dynamic allocation, certified RTOS

---

[Previous Chapter: Power Management ←](Chapter_21_Power_Management.md) | [Next Chapter: Security →](Chapter_23_Security.md)
