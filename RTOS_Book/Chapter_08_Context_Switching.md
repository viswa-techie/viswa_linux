# Chapter 8: Context Switching

## Learning Goals
- Understand what happens during a context switch at the hardware level
- Learn the ARM Cortex-M PendSV context switch mechanism in detail
- Know the difference between hardware-saved and software-saved registers
- Understand context switch timing and optimization
- Compare context switching across RTOS platforms

---

## 1. What Is a Context Switch?

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Context = complete CPU state needed to resume a task:   │
  │                                                           │
  │  ┌─────────────────────────────────────────────────┐     │
  │  │ General Purpose Registers: R0-R12               │     │
  │  │ Stack Pointer: SP (MSP/PSP on Cortex-M)         │     │
  │  │ Program Counter: PC (return address)             │     │
  │  │ Status Register: xPSR (flags, exception number) │     │
  │  │ Link Register: LR (EXC_RETURN on Cortex-M)      │     │
  │  │ FPU Registers: S0-S31 + FPSCR (if FPU enabled) │     │
  │  └─────────────────────────────────────────────────┘     │
  │                                                           │
  │  Context Switch Steps:                                   │
  │  ┌──────┐     ┌──────────────────┐     ┌──────┐         │
  │  │Task A│────►│ Save A's context │────►│Task B│         │
  │  │ runs │     │ to A's stack     │     │ runs │         │
  │  └──────┘     │ Restore B's      │     └──────┘         │
  │               │ context from     │                       │
  │               │ B's stack        │                       │
  │               │ Update pxCurrent │                       │
  │               │ TCB pointer      │                       │
  │               └──────────────────┘                       │
  │                                                           │
  │  Cost:                                                   │
  │  ┌─────────────┬──────────────────────────────────┐     │
  │  │ RTOS        │ Typical context switch time       │     │
  │  ├─────────────┼──────────────────────────────────┤     │
  │  │ FreeRTOS M4 │ ~2-5 μs (no FPU), ~10 μs (FPU) │     │
  │  │ Zephyr M4   │ ~3-5 μs                          │     │
  │  │ QNX A53     │ ~1-5 μs (thread), ~10+ μs (proc) │    │
  │  │ VxWorks     │ ~1-3 μs (optimized)              │     │
  │  │ ThreadX M4  │ ~1-2 μs (fastest commercial)     │     │
  │  └─────────────┴──────────────────────────────────┘     │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. ARM Cortex-M Stack Pointers

```
  Dual Stack Pointer Architecture
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  MSP (Main Stack Pointer):                               │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ · Used by exception handlers (ISRs)             │      │
  │  │ · Used in Thread mode before RTOS starts        │      │
  │  │ · Always used in Handler mode                   │      │
  │  │ · Set via vector table entry 0 (initial SP)     │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  PSP (Process Stack Pointer):                            │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ · Used by RTOS tasks in Thread mode             │      │
  │  │ · Each task has its own PSP value               │      │
  │  │ · RTOS switches PSP on context switch           │      │
  │  │ · Set via CONTROL register bit 1 (SPSEL)       │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Flow:                                                   │
  │                                                           │
  │  ┌──────────┐    Exception    ┌──────────┐               │
  │  │ Thread   │ ──────────────► │ Handler  │               │
  │  │ Mode     │                 │ Mode     │               │
  │  │ Uses PSP │ ◄────────────── │ Uses MSP │               │
  │  └──────────┘    Return       └──────────┘               │
  │                                                           │
  │  CONTROL Register:                                       │
  │  Bit 1 (SPSEL): 0 = use MSP in Thread mode              │
  │                  1 = use PSP in Thread mode              │
  │  Bit 0 (nPRIV): 0 = privileged, 1 = unprivileged        │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Hardware vs Software Context Save

```
  Register Save/Restore on Cortex-M4
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  HARDWARE-SAVED (automatic on exception entry):          │
  │  ┌──────────────────────────────────────────────┐       │
  │  │  PSP (before push) ──────────────────────────│       │
  │  │  +28  xPSR     (Program Status Register)     │       │
  │  │  +24  PC       (Return address)               │       │
  │  │  +20  LR       (Link Register / R14)          │       │
  │  │  +16  R12                                     │       │
  │  │  +12  R3                                      │       │
  │  │   +8  R2                                      │       │
  │  │   +4  R1                                      │       │
  │  │   +0  R0                                      │       │
  │  │  PSP (after push) ◄──── SP points here        │       │
  │  └──────────────────────────────────────────────┘       │
  │  Takes ~12 CPU cycles, done by hardware automatically    │
  │                                                           │
  │  SOFTWARE-SAVED (PendSV handler must push manually):     │
  │  ┌──────────────────────────────────────────────┐       │
  │  │  (hardware frame above)                       │       │
  │  │  R11                                          │       │
  │  │  R10                                          │       │
  │  │  R9                                           │       │
  │  │  R8                                           │       │
  │  │  R7                                           │       │
  │  │  R6                                           │       │
  │  │  R5                                           │       │
  │  │  R4                                           │       │
  │  │  PSP ◄──── stored in TCB->pxTopOfStack        │       │
  │  └──────────────────────────────────────────────┘       │
  │  R4-R11 must be saved/restored by software               │
  │                                                           │
  │  WITH FPU (lazy stacking):                               │
  │  ┌──────────────────────────────────────────────┐       │
  │  │  Additional: S0-S15 + FPSCR (hardware)        │       │
  │  │  Additional: S16-S31 (software, if used)      │       │
  │  │  Total: 26 extra registers = 104 more bytes   │       │
  │  │  Lazy stacking: space reserved but FPU regs   │       │
  │  │  only actually saved if new context uses FPU  │       │
  │  └──────────────────────────────────────────────┘       │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. PendSV Context Switch Handler (FreeRTOS)

```c
/* ARM Cortex-M4 PendSV handler — the heart of RTOS context switching */
/* From FreeRTOS port.c (GCC/ARM_CM4F) */

__attribute__((naked)) void xPortPendSVHandler(void) {
    __asm volatile(
    "   mrs r0, psp                    \n" /* R0 = current task's PSP */
    "   isb                            \n" /* Instruction sync barrier */
    "                                  \n"
    "   ldr r3, =pxCurrentTCB          \n" /* R3 = &pxCurrentTCB */
    "   ldr r2, [r3]                   \n" /* R2 = pxCurrentTCB (current TCB) */
    "                                  \n"
    /* Save FPU context if task used FPU */
    "   tst r14, #0x10                 \n" /* Test EXC_RETURN bit 4 */
    "   it eq                          \n" /* If 0, FPU was used */
    "   vstmdbeq r0!, {s16-s31}        \n" /* Save S16-S31 (S0-S15 already hw-saved) */
    "                                  \n"
    /* Save R4-R11 to current task's stack */
    "   stmdb r0!, {r4-r11, r14}       \n" /* Push R4-R11 + EXC_RETURN */
    "                                  \n"
    /* Save current PSP to current TCB */
    "   str r0, [r2]                   \n" /* pxCurrentTCB->pxTopOfStack = PSP */
    "                                  \n"
    /* ---- Current task context fully saved ---- */
    "                                  \n"
    "   stmdb sp!, {r0, r3}            \n" /* Preserve R0, R3 on MSP */
    "   mov r0, %0                     \n" /* R0 = configMAX_SYSCALL_INTERRUPT_PRIORITY */
    "   msr basepri, r0                \n" /* Disable interrupts (critical section) */
    "   dsb                            \n"
    "   isb                            \n"
    "   bl vTaskSwitchContext           \n" /* Call C function to select next task */
    "   mov r0, #0                     \n" /* pxCurrentTCB now points to new task */
    "   msr basepri, r0                \n" /* Re-enable interrupts */
    "   ldmia sp!, {r0, r3}            \n" /* Restore R0, R3 */
    "                                  \n"
    /* ---- Now restore new task's context ---- */
    "                                  \n"
    "   ldr r1, [r3]                   \n" /* R1 = new pxCurrentTCB */
    "   ldr r0, [r1]                   \n" /* R0 = new task's pxTopOfStack */
    "                                  \n"
    /* Restore R4-R11 from new task's stack */
    "   ldmia r0!, {r4-r11, r14}       \n" /* Pop R4-R11 + EXC_RETURN */
    "                                  \n"
    /* Restore FPU context if new task used FPU */
    "   tst r14, #0x10                 \n"
    "   it eq                          \n"
    "   vldmiaeq r0!, {s16-s31}        \n" /* Restore S16-S31 */
    "                                  \n"
    "   msr psp, r0                    \n" /* Set PSP to new task's stack */
    "   isb                            \n"
    "   bx r14                         \n" /* Return — hardware restores R0-R3,R12,LR,PC,xPSR */
    ::"i"(configMAX_SYSCALL_INTERRUPT_PRIORITY)
    );
}

/*
 * Complete context switch flow:
 *
 * 1. PendSV fires (lowest priority exception)
 * 2. Hardware auto-saves R0-R3, R12, LR, PC, xPSR to task's PSP
 *    (and S0-S15 if FPU used, with lazy stacking)
 * 3. CPU switches to MSP (Handler mode)
 * 4. PendSV handler reads PSP (current task's stack position)
 * 5. Software saves R4-R11 + EXC_RETURN to current task's stack
 * 6. Saves updated PSP to current TCB->pxTopOfStack
 * 7. Calls vTaskSwitchContext() → updates pxCurrentTCB to next task
 * 8. Loads new task's pxTopOfStack
 * 9. Software restores R4-R11 + EXC_RETURN from new task's stack
 * 10. Sets PSP to new task's stack position
 * 11. BX LR → hardware auto-restores R0-R3, R12, LR, PC, xPSR
 * 12. New task resumes from where it was suspended
 */
```

---

## 5. EXC_RETURN and Stack Frame

```
  EXC_RETURN Values (stored in LR on exception entry)
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  LR Value         │ Meaning                              │
  │  ──────────────────┼────────────────────────────────────  │
  │  0xFFFFFFF1       │ Return to Handler mode, use MSP      │
  │  0xFFFFFFF9       │ Return to Thread mode, use MSP       │
  │  0xFFFFFFFD       │ Return to Thread mode, use PSP  ←    │
  │  0xFFFFFFE1       │ Handler mode, MSP, FPU frame         │
  │  0xFFFFFFE9       │ Thread mode, MSP, FPU frame          │
  │  0xFFFFFFED       │ Thread mode, PSP, FPU frame     ←    │
  │                                                           │
  │  ← RTOS tasks use these (PSP in Thread mode)             │
  │                                                           │
  │  Bit 4 of EXC_RETURN:                                   │
  │  1 = basic frame (no FPU context on stack)               │
  │  0 = extended frame (FPU context on stack)               │
  │                                                           │
  │  This is why PendSV handler checks: tst r14, #0x10      │
  │  If bit 4 is 0 → save/restore S16-S31                   │
  └──────────────────────────────────────────────────────────┘

  Stack Frame on Context Switch (without FPU):
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌── Software saved (PendSV) ──┐                        │
  │  │ EXC_RETURN (R14)             │ ← pxTopOfStack         │
  │  │ R4                           │                        │
  │  │ R5                           │                        │
  │  │ R6                           │                        │
  │  │ R7                           │                        │
  │  │ R8                           │                        │
  │  │ R9                           │                        │
  │  │ R10                          │                        │
  │  │ R11                          │                        │
  │  ├── Hardware saved (NVIC) ─────┤                        │
  │  │ R0                           │                        │
  │  │ R1                           │                        │
  │  │ R2                           │                        │
  │  │ R3                           │                        │
  │  │ R12                          │                        │
  │  │ LR (R14)                     │                        │
  │  │ PC (R15) — return address    │                        │
  │  │ xPSR                         │                        │
  │  └─────────────────────────────┘ ← original PSP          │
  │                                                           │
  │  Total: 17 words × 4 bytes = 68 bytes per context switch │
  │  With FPU: + 34 words = 136 bytes additional             │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. Context Switch in Other RTOS

```c
/* Zephyr context switch (arch/arm/core/cortex_m/swap.c) — similar pattern */
/*
 * Zephyr uses _thread.callee_saved to store R4-R11
 * rather than pushing to stack (different design choice)
 *
 * struct _callee_saved {
 *     uint32_t r4, r5, r6, r7, r8, r9, r10, r11;
 *     uint32_t psp;       // Process Stack Pointer
 * };
 *
 * This means context is saved to TCB struct, not to task stack.
 * Advantage: stack usage more predictable
 * Disadvantage: slightly more memory per thread struct
 */

/* QNX Neutrino (Cortex-A with MMU):
 *
 * Context switch includes:
 * - Full register save/restore (R0-R15 + CPSR)
 * - TLB/ASID management (if switching processes)
 * - Cache considerations (may need flush on process switch)
 * - Thread switch within same process: fast (~1-3 μs)
 * - Process switch: slower (~5-15 μs) due to MMU update
 *
 * QNX minimizes process switches by using threads heavily
 */

/* VxWorks task switch:
 *
 * windExit() → reschedule() → switch context
 * - Uses optimized assembly per architecture
 * - Supports both kernel tasks (shared) and RTPs (isolated)
 * - RTP switch includes MMU context switch
 * - Kernel task switch: ~1-2 μs
 */
```

---

## 7. Context Switch Optimization Techniques

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  1. Lazy FPU Context Save (ARM):                         │
  │     - Reserve stack space for FPU regs on exception      │
  │     - Don't actually save until another task uses FPU    │
  │     - If no FPU use, save overhead = 0                   │
  │     - LSPACT bit in FPCCR controls this                  │
  │                                                           │
  │  2. TCM (Tightly Coupled Memory):                        │
  │     - Place task stacks in TCM for fastest access        │
  │     - Single-cycle access vs multi-cycle SRAM             │
  │     - Critical for sub-microsecond switches               │
  │                                                           │
  │  3. Minimal register save:                               │
  │     - Only save registers that are actually used          │
  │     - Compiler can be instructed to limit register use   │
  │     - ThreadX uses this for ~1μs switches                │
  │                                                           │
  │  4. Avoid cache thrashing:                               │
  │     - On cached cores (Cortex-A, Cortex-M7)             │
  │     - Pin hot task stacks in cache if possible           │
  │     - Cache miss during restore → unpredictable latency  │
  │                                                           │
  │  5. Compiler-assisted:                                   │
  │     - Use __attribute__((naked)) for switch handler      │
  │     - Prevents compiler from adding prologue/epilogue    │
  │     - Full control over register usage                   │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Walk through the complete PendSV context switch on Cortex-M4.**
**A:** (1) SysTick or API call determines switch is needed, pends PendSV via `SCB->ICSR |= PENDSVSET`. (2) All higher-priority ISRs complete. (3) PendSV fires — hardware automatically pushes R0-R3, R12, LR, PC, xPSR onto current task's PSP (12 cycles). (4) PendSV handler reads PSP via `MRS R0, PSP`. (5) Tests EXC_RETURN bit 4 — if FPU was used, saves S16-S31. (6) Pushes R4-R11 + EXC_RETURN to task stack via `STMDB`. (7) Stores updated PSP in current TCB's `pxTopOfStack`. (8) Calls `vTaskSwitchContext()` which updates `pxCurrentTCB` to next task. (9) Loads new task's `pxTopOfStack`. (10) Pops R4-R11 + EXC_RETURN via `LDMIA`. (11) If FPU frame, restores S16-S31. (12) Sets PSP to new stack via `MSR PSP, R0`. (13) `BX LR` — hardware auto-restores R0-R3, R12, LR, PC, xPSR from new PSP. New task resumes.

**Q2: Why does Cortex-M have two stack pointers (MSP and PSP)?**
**A:** Separation provides several benefits: (1) ISR stack isolation — MSP is used for all exception handlers, preventing ISR stack usage from depleting task stacks. (2) Predictable ISR stack — MSP size can be set based on worst-case ISR nesting, independent of task count. (3) MPU protection — each task's PSP stack region can be MPU-protected; MSP region is separate. (4) Fault containment — if a task overflows its stack (PSP), it doesn't corrupt ISR stack (MSP) or other tasks. (5) Simpler context switch — on exception entry, hardware automatically switches from PSP to MSP; on return, switches back. The RTOS only needs to update PSP between tasks.

**Q3: What is lazy FPU context saving and why is it important?**
**A:** On Cortex-M4F, when an exception occurs and FPU is enabled, the hardware reserves 68 bytes of stack space for FPU registers (S0-S15 + FPSCR) but doesn't actually write them. The LSPACT (Lazy State Preservation Active) bit is set. If the exception handler (or new task) accesses an FPU register, ONLY THEN does the hardware save the caller's FPU state and clear LSPACT. If no FPU access occurs, the save is skipped entirely — just the stack space was reserved. Benefit: tasks that don't use FPU pay zero overhead for FPU context save. In a system where only 1 of 10 tasks uses FPU, this saves significant context switch time for the other 9 tasks.

---

## Summary

- Context switch = save current task's CPU state + restore next task's state
- Cortex-M: hardware saves R0-R3, R12, LR, PC, xPSR (8 regs); software saves R4-R11 (8 regs)
- PendSV handler at lowest priority performs actual switch via assembly
- MSP for ISRs, PSP for tasks — isolation and predictability
- EXC_RETURN value tells hardware which stack and frame type to use
- Lazy FPU stacking avoids saving S0-S31 unless FPU is actually used
- Typical switch time: 2-10μs depending on FPU and RTOS
- Context saved on task stack (FreeRTOS) or in TCB struct (Zephyr)

---

[Previous Chapter: Scheduling Algorithms ←](Chapter_07_Scheduling_Algorithms.md) | [Next Chapter: Interrupt Handling →](Chapter_09_Interrupt_Handling.md)
