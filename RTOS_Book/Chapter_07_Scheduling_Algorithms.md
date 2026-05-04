# Chapter 7: Real-Time Scheduling Algorithms

## Learning Goals
- Master Rate Monotonic Scheduling (RMS) with mathematical analysis
- Understand Earliest Deadline First (EDF) and its optimality
- Learn Deadline Monotonic Scheduling (DMS) for arbitrary deadlines
- Perform schedulability analysis with worked examples
- Compare fixed-priority vs dynamic-priority algorithms
- Know practical considerations and real-world algorithm selection

---

## 1. Rate Monotonic Scheduling (RMS)

```
  RMS: Shorter period → Higher priority (fixed assignment)
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Assumptions (Liu & Layland, 1973):                      │
  │  1. Tasks are periodic with fixed periods                │
  │  2. Deadline = Period (Di = Ti)                          │
  │  3. Tasks are independent (no shared resources)          │
  │  4. Context switch time is zero (theoretical)            │
  │  5. No task self-suspends                                │
  │                                                           │
  │  Priority Assignment Rule:                               │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ Shorter period → Higher priority               │      │
  │  │ If T1 < T2 < T3, then Prio1 > Prio2 > Prio3   │      │
  │  │                                                  │      │
  │  │ This is OPTIMAL among fixed-priority algorithms │      │
  │  │ (if any fixed-priority assignment works, RMS    │      │
  │  │  will also work)                                │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Schedulability Test (sufficient condition):             │
  │                                                           │
  │   n                                                       │
  │   Σ  (Ci/Ti) ≤ n(2^(1/n) - 1) = U_lub                  │
  │  i=1                                                      │
  │                                                           │
  │  Where: Ci = worst-case execution time of task i         │
  │         Ti = period of task i                             │
  │         n  = number of tasks                              │
  │                                                           │
  │  U_lub values:                                           │
  │  ┌─────┬────────────┐                                    │
  │  │  n  │   U_lub    │                                    │
  │  ├─────┼────────────┤                                    │
  │  │  1  │  1.000     │                                    │
  │  │  2  │  0.828     │                                    │
  │  │  3  │  0.780     │                                    │
  │  │  4  │  0.757     │                                    │
  │  │  5  │  0.743     │                                    │
  │  │ ∞   │  ln(2)=0.693│                                   │
  │  └─────┴────────────┘                                    │
  │  Note: U ≤ U_lub is SUFFICIENT but not NECESSARY         │
  │  Some task sets with U > U_lub are still schedulable     │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. RMS Worked Example

```
  Example: Three periodic tasks
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Task    Period(T)   WCET(C)    Utilization(C/T)         │
  │  ────    ─────────   ───────    ──────────────────        │
  │  τ1       10ms        2ms       0.200                    │
  │  τ2       15ms        3ms       0.200                    │
  │  τ3       35ms        5ms       0.143                    │
  │                                                           │
  │  Total Utilization U = 0.200 + 0.200 + 0.143 = 0.543    │
  │  U_lub for n=3 = 0.780                                   │
  │                                                           │
  │  0.543 ≤ 0.780 → Schedulable by RMS ✓                   │
  │                                                           │
  │  Priority assignment (shorter period = higher priority): │
  │  τ1 (T=10) > τ2 (T=15) > τ3 (T=35)                     │
  │                                                           │
  │  Timeline (LCM = 210ms, showing first 35ms):            │
  │                                                           │
  │  Time: 0    5    10   15   20   25   30   35             │
  │        ├────┼────┼────┼────┼────┼────┼────┤             │
  │  τ1:   ▓▓───────▓▓───────▓▓───────▓▓──                  │
  │  τ2:   ──▓▓▓────────▓▓▓─────────▓▓▓──                   │
  │  τ3:   ──────▓▓▓▓▓──────────────────                     │
  │  Idle:       ─    ─    ─                                 │
  │                                                           │
  │  t=0:  τ1 runs (2ms), t=2: τ2 runs (3ms)               │
  │  t=5:  τ3 runs (5ms), t=10: τ1 preempts (deadline)     │
  │  All deadlines met ✓                                      │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Response Time Analysis (Exact Test)

```
  When U_lub test is inconclusive (U > U_lub but < 1.0)
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Response Time Analysis — EXACT (necessary + sufficient) │
  │                                                           │
  │  For each task i, compute worst-case response time Ri:   │
  │                                                           │
  │  Ri = Ci + Σ ⌈Ri/Tj⌉ × Cj    (for all j with higher    │
  │             j∈hp(i)             priority than i)          │
  │                                                           │
  │  Iterative solution:                                     │
  │  R_i^(0) = Ci                                            │
  │  R_i^(n+1) = Ci + Σ ⌈R_i^(n)/Tj⌉ × Cj                 │
  │  Stop when R_i^(n+1) = R_i^(n) (converged)              │
  │  Task schedulable if Ri ≤ Di (response ≤ deadline)      │
  │                                                           │
  │  Example: τ1(C=3,T=10), τ2(C=3,T=15), τ3(C=5,T=30)    │
  │  U = 0.30 + 0.20 + 0.167 = 0.667                        │
  │  U_lub(3) = 0.780 → PASS (but let's verify exactly)     │
  │                                                           │
  │  R1 (highest priority, no interference):                 │
  │    R1 = C1 = 3ms ≤ T1=10ms ✓                            │
  │                                                           │
  │  R2 (interference from τ1):                              │
  │    R2^(0) = 3                                            │
  │    R2^(1) = 3 + ⌈3/10⌉×3 = 3 + 1×3 = 6                 │
  │    R2^(2) = 3 + ⌈6/10⌉×3 = 3 + 1×3 = 6  (converged)   │
  │    R2 = 6ms ≤ T2=15ms ✓                                 │
  │                                                           │
  │  R3 (interference from τ1 and τ2):                       │
  │    R3^(0) = 5                                            │
  │    R3^(1) = 5 + ⌈5/10⌉×3 + ⌈5/15⌉×3 = 5+3+3 = 11     │
  │    R3^(2) = 5 + ⌈11/10⌉×3 + ⌈11/15⌉×3 = 5+6+3 = 14   │
  │    R3^(3) = 5 + ⌈14/10⌉×3 + ⌈14/15⌉×3 = 5+6+3 = 14   │
  │    R3 = 14ms ≤ T3=30ms ✓ (converged)                    │
  │                                                           │
  │  All tasks schedulable ✓                                  │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Earliest Deadline First (EDF)

```
  EDF: Dynamic priority — task with nearest deadline runs first
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Key Properties:                                         │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ · Optimal dynamic-priority algorithm            │      │
  │  │ · Can achieve 100% CPU utilization              │      │
  │  │ · Schedulability test: U = Σ(Ci/Ti) ≤ 1.0      │      │
  │  │ · Priority changes at every scheduling point    │      │
  │  │ · Works for periodic and aperiodic tasks        │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  EDF vs RMS Utilization Bound:                           │
  │  ┌────────────────────────────────────────────────┐      │
  │  │     0%        69.3%     82.8%    100%          │      │
  │  │     ├──────────┼──────────┼────────┤           │      │
  │  │     │   Both   │ EDF only │ EDF    │           │      │
  │  │     │ schedule │ schedules│ only   │           │      │
  │  │     │          │(RMS may  │        │           │      │
  │  │     │          │ fail)    │        │           │      │
  │  │     ├──────────┴──────────┴────────┤           │      │
  │  │     │ U_lub(RMS,∞)      U_max(EDF) │           │      │
  │  │     └──────────────────────────────┘           │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Example: τ1(C=2,T=5), τ2(C=4,T=7)                     │
  │  U = 0.40 + 0.571 = 0.971 (> 0.828 RMS bound for n=2)  │
  │  RMS: may miss deadlines                                 │
  │  EDF: schedulable (0.971 ≤ 1.0) ✓                       │
  │                                                           │
  │  Timeline:                                               │
  │  Time:  0   1   2   3   4   5   6   7                   │
  │  τ1:    ▓▓──────────▓▓──────────                         │
  │  τ2:    ──▓▓▓▓──────────▓▓▓▓──                           │
  │                                                           │
  │  t=0: τ1 deadline=5, τ2 deadline=7 → τ1 runs (closer dl)│
  │  t=2: τ1 done, τ2 runs                                  │
  │  t=5: τ1 new instance, deadline=10; τ2 deadline=7       │
  │       τ2 has closer deadline → τ2 continues!             │
  │  t=6: τ2 done, τ1 runs                                  │
  │  t=7: τ2 new instance... (dynamic priority adjustment)  │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Deadline Monotonic Scheduling (DMS)

```
  DMS: Fixed priority, shorter DEADLINE → higher priority
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  When Deadline ≠ Period (Di ≤ Ti):                       │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ RMS assigns priority by period                  │      │
  │  │ DMS assigns priority by deadline                │      │
  │  │ DMS is optimal among fixed-priority when D ≤ T  │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Example:                                                │
  │  Task    Period(T)  Deadline(D)  WCET(C)                 │
  │  τ1       10ms       8ms         2ms                     │
  │  τ2       12ms       6ms         3ms                     │
  │                                                           │
  │  RMS priority: τ1 > τ2 (T1=10 < T2=12)                  │
  │  DMS priority: τ2 > τ1 (D2=6 < D1=8)  ← different!     │
  │                                                           │
  │  With DMS:                                               │
  │  τ2 (D=6) runs first → finishes at t=3                  │
  │  τ1 (D=8) runs next  → finishes at t=5                  │
  │  Both meet deadlines ✓                                    │
  │                                                           │
  │  With RMS:                                               │
  │  τ1 (T=10) runs first → finishes at t=2                 │
  │  τ2 (T=12) runs next  → finishes at t=5                 │
  │  Both meet deadlines ✓ (in this case both work)          │
  │                                                           │
  │  DMS = RMS when D = T (reduces to same assignment)       │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. Algorithm Comparison

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Feature        │ RMS         │ EDF         │ DMS        │
  │  ────────────────┼─────────────┼─────────────┼──────────│
  │  Priority type  │ Fixed       │ Dynamic     │ Fixed     │
  │  Assignment     │ By period   │ By deadline │ By D.line │
  │  Optimality     │ Fixed-prio  │ Overall     │ Fixed-prio│
  │                  │ (when D=T)  │ optimal     │ (when D≤T)│
  │  Max utilization│ ~69% (n→∞)  │ 100%        │ ~69%     │
  │  Overload       │ Low prio    │ Unpredictable│ Low prio │
  │  behavior       │ misses first│ (domino)    │ miss first│
  │  Implementation │ Simple      │ Complex     │ Simple   │
  │  Overhead       │ Low         │ Higher      │ Low      │
  │  D ≠ T support │ No          │ Yes         │ Yes      │
  │  RTOS support   │ All (manual)│ Few (rare)  │ Manual   │
  │  Certification  │ Easy        │ Hard        │ Easy     │
  │                                                           │
  │  Why RMS/fixed-priority dominates in practice:           │
  │  1. ALL commercial RTOS support fixed-priority natively  │
  │  2. EDF overhead: re-sorting deadline queue each event   │
  │  3. EDF overload: domino effect (all tasks miss)         │
  │  4. RMS overload: only lowest-priority tasks miss        │
  │  5. RMS analyzable for certification (DO-178C, ISO26262)│
  │  6. 69% bound is sufficient for most real systems        │
  │  7. Response Time Analysis extends RMS to exact test     │
  └──────────────────────────────────────────────────────────┘
```

---

## 7. Practical Scheduling in Real RTOS

```c
/* FreeRTOS: Manual RMS-style priority assignment */

/* System design: assign priorities by period */
#define PRIO_MOTOR_CONTROL    5   /* Period: 1ms (highest freq → highest prio) */
#define PRIO_SENSOR_READ      4   /* Period: 5ms */
#define PRIO_CAN_HANDLER      3   /* Period: 10ms */
#define PRIO_DISPLAY_UPDATE   2   /* Period: 50ms */
#define PRIO_LOGGING          1   /* Period: 1000ms */

/* Verify schedulability before deploying */
/*
 * Task              C(ms)   T(ms)   U = C/T
 * Motor Control      0.2     1       0.200
 * Sensor Read        0.5     5       0.100
 * CAN Handler        1.0    10       0.100
 * Display Update     5.0    50       0.100
 * Logging            2.0  1000       0.002
 *                                    ──────
 * Total U                             0.502
 *
 * U_lub(5) = 0.743
 * 0.502 ≤ 0.743 → Schedulable ✓
 * CPU is 50.2% utilized, leaving 49.8% headroom
 */

xTaskCreate(vMotorControl, "Motor",  128, NULL, PRIO_MOTOR_CONTROL, NULL);
xTaskCreate(vSensorRead,   "Sensor", 256, NULL, PRIO_SENSOR_READ,   NULL);
xTaskCreate(vCANHandler,   "CAN",    256, NULL, PRIO_CAN_HANDLER,   NULL);
xTaskCreate(vDisplayUpdate,"Disp",   512, NULL, PRIO_DISPLAY_UPDATE, NULL);
xTaskCreate(vLogging,      "Log",    512, NULL, PRIO_LOGGING,        NULL);

/* Each task uses vTaskDelayUntil for periodic execution */
void vMotorControl(void *pvParameters) {
    TickType_t xLastWakeTime = xTaskGetTickCount();
    for (;;) {
        /* Read encoder, compute PID, set PWM */
        motor_pid_step();
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(1));
    }
}
```

```
  Zephyr Scheduling Policies
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Cooperative threads (negative priority):                │
  │  - Cannot be preempted by scheduler                      │
  │  - Only preempted by ISRs                                │
  │  - Must explicitly yield: k_yield() or k_sleep()        │
  │  - Priority: -1 to -(CONFIG_NUM_COOP_PRIORITIES)        │
  │                                                           │
  │  Preemptive threads (non-negative priority):             │
  │  - Can be preempted by higher-priority threads           │
  │  - Time-sliced among equal priority (if enabled)         │
  │  - Priority: 0 to (CONFIG_NUM_PREEMPT_PRIORITIES - 1)   │
  │                                                           │
  │  Priority number:  ... -2  -1  0  1  2  3 ...           │
  │                    ◄── cooperative │ preemptive ──►       │
  │                    higher priority │ lower priority       │
  │                                                           │
  │  Meta-IRQ threads: can preempt cooperative threads       │
  │  (bridge between ISR and thread context)                 │
  └──────────────────────────────────────────────────────────┘
```

---

## 8. Aperiodic Task Handling

```
  Handling Non-Periodic Tasks in RT Systems
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Background Server:                                      │
  │  - Run aperiodic tasks at lowest priority                │
  │  - Simple but poor response time                         │
  │                                                           │
  │  Polling Server:                                         │
  │  - Periodic server task with budget (Cs, Ts)             │
  │  - Checks for aperiodic work each period                 │
  │  - If no work, budget is lost (not carried over)         │
  │  - Analyzable with RMS (treat as periodic task)          │
  │                                                           │
  │  Sporadic Server:                                        │
  │  - Budget replenished only when consumed                 │
  │  - Better responsiveness than polling                    │
  │  - Preserves schedulability of periodic tasks            │
  │  - QNX supports SCHED_SPORADIC natively                  │
  │                                                           │
  │  ┌─── Periodic Tasks ────┬─── Server ─────────────┐     │
  │  │ τ1, τ2, τ3 (RMS)     │ Budget = 1ms            │     │
  │  │ Fixed periods         │ Period = 10ms           │     │
  │  │                       │ Handles: button press,  │     │
  │  │                       │ network packet, etc.    │     │
  │  └───────────────────────┴─────────────────────────┘     │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Explain the RMS schedulability test. What does "sufficient but not necessary" mean?**
**A:** The RMS utilization bound test: U = Σ(Ci/Ti) ≤ n(2^(1/n) - 1). If total utilization U is at or below this bound, the task set is GUARANTEED schedulable with RMS. "Sufficient but not necessary" means: if U ≤ U_lub, definitely schedulable (no false negatives). But if U > U_lub, the test is inconclusive — the task set MAY still be schedulable (possible false positives for "not schedulable" conclusion). To get an exact answer when U > U_lub, use Response Time Analysis: compute Ri for each task iteratively and check Ri ≤ Di. For n→∞, U_lub = ln(2) ≈ 0.693, meaning some task sets using >69.3% CPU are still RMS-schedulable.

**Q2: Why does EDF achieve 100% utilization but is rarely used in practice?**
**A:** EDF is theoretically optimal — if U ≤ 1.0, it can schedule the task set. 100% utilization means no CPU waste. But practical issues: (1) Overload behavior — when U > 1.0 (e.g., task overruns), EDF can cause ALL tasks to miss deadlines (domino effect) because deadline-closest order constantly changes. RMS under overload only misses low-priority deadlines — predictable degradation. (2) Implementation overhead — EDF requires maintaining a sorted deadline queue, re-sorting on every event. Fixed-priority uses a bitmap (O(1) on ARM). (3) Analysis complexity — EDF is harder to analyze for certification (DO-178C, ISO 26262). (4) RTOS support — virtually no commercial RTOS implements EDF natively. (5) In practice, systems rarely need >69% utilization, so RMS suffices.

**Q3: A system has tasks: τ1(C=1ms,T=4ms), τ2(C=2ms,T=6ms), τ3(C=3ms,T=12ms). Is it RMS schedulable?**
**A:** Step 1: Calculate utilization. U = 1/4 + 2/6 + 3/12 = 0.250 + 0.333 + 0.250 = 0.833. Step 2: Compare with U_lub for n=3: U_lub = 3(2^(1/3) - 1) = 0.780. Since 0.833 > 0.780, the U_lub test is INCONCLUSIVE. Step 3: Use Response Time Analysis. R1 = C1 = 1ms ≤ 4ms ✓. R2: R2^0=2, R2^1=2+⌈2/4⌉×1=3, R2^2=2+⌈3/4⌉×1=3 → R2=3ms ≤ 6ms ✓. R3: R3^0=3, R3^1=3+⌈3/4⌉×1+⌈3/6⌉×2=3+1+2=6, R3^2=3+⌈6/4⌉×1+⌈6/6⌉×2=3+2+2=7, R3^3=3+⌈7/4⌉×1+⌈7/6⌉×2=3+2+4=9, R3^4=3+⌈9/4⌉×1+⌈9/6⌉×2=3+3+4=10, R3^5=3+⌈10/4⌉×1+⌈10/6⌉×2=3+3+4=10 → R3=10ms ≤ 12ms ✓. All schedulable despite U > U_lub.

**Q4: What is the Mars Pathfinder scheduling problem and what algorithm concepts does it illustrate?**
**A:** Mars Pathfinder (1997) experienced system resets due to priority inversion. A low-priority meteorological task held a shared mutex, was preempted by a medium-priority communication task, while the high-priority bus management task waited for the mutex. The medium task ran for extended periods, preventing the low task from releasing the mutex, causing the high-priority task to miss its watchdog deadline → system reset. Fix: enabling priority inheritance in VxWorks (uploaded from Earth). This illustrates: (1) fixed-priority scheduling alone isn't sufficient — resource sharing must be considered, (2) priority inheritance protocol — temporarily raise holder's priority to waiter's level, (3) the gap between theoretical scheduling analysis (which assumes independent tasks) and real systems with shared resources.

---

## Summary

- RMS: fixed priority by period, optimal among fixed-priority (when D=T), U_lub ≈ 69.3% for n→∞
- EDF: dynamic priority by deadline, optimal overall, 100% utilization, but impractical overload behavior
- DMS: fixed priority by deadline, optimal when D ≤ T, generalizes RMS
- Response Time Analysis: exact test — compute Ri iteratively, check Ri ≤ Di
- RMS U_lub test is sufficient but not necessary; RTA is both sufficient and necessary
- EDF rarely used in practice: overload domino effect, implementation complexity, certification difficulty
- Aperiodic tasks handled by polling/sporadic servers within periodic framework
- All commercial RTOS use fixed-priority; RMS-style assignment is standard practice

---

[Previous Chapter: Real-Time Scheduling ←](Chapter_06_Scheduling.md) | [Next Chapter: Context Switching →](Chapter_08_Context_Switching.md)
