# Chapter 1: Foundations of Real-Time Systems

## Learning Goals
- Understand what makes a system "real-time"
- Distinguish hard, firm, and soft real-time requirements
- Learn determinism, latency, jitter, and their measurement
- Know real-time performance metrics and certification levels

---

## 1. What is a Real-Time System

```
  Real-Time System Definition
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  A real-time system produces correct results             │
  │  AND meets timing deadlines.                             │
  │                                                           │
  │  Correctness = Logical correctness + Temporal correctness│
  │                                                           │
  │  ┌───────────────────────────────────────────────────┐   │
  │  │                                                   │   │
  │  │  Input Event ──► Processing ──► Output Response   │   │
  │  │       │                              │            │   │
  │  │       │◄─── Response Time (R) ──────►│            │   │
  │  │       │                              │            │   │
  │  │       │◄──────── Deadline (D) ──────────►│        │   │
  │  │                                           │       │   │
  │  │  If R ≤ D  → System CORRECT               │       │   │
  │  │  If R > D  → Deadline MISS (failure)       │       │   │
  │  └───────────────────────────────────────────────────┘   │
  │                                                           │
  │  Key: A fast system is NOT necessarily real-time.         │
  │  A system processing in 1μs average but occasionally     │
  │  taking 100ms is NOT real-time if deadline is 10ms.      │
  │                                                           │
  │  Real-time = GUARANTEED worst-case, not fast average.    │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Hard vs Soft vs Firm Real-Time

```
  Real-Time Classification
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Hard Real-Time:                                         │
  │  - Deadline miss = CATASTROPHIC FAILURE                  │
  │  - System must GUARANTEE 100% deadline compliance        │
  │  - Examples: ABS braking, airbag deployment,             │
  │    pacemaker, flight control, nuclear reactor             │
  │  - Must be formally analyzed (WCET, schedulability)      │
  │                                                           │
  │  Firm Real-Time:                                         │
  │  - Deadline miss = result is USELESS (discard)           │
  │  - No catastrophe, but quality degrades                  │
  │  - Examples: video frame rendering (drop frame),         │
  │    financial trading (stale price useless)                │
  │  - Occasional misses tolerable but undesirable           │
  │                                                           │
  │  Soft Real-Time:                                         │
  │  - Deadline miss = DEGRADED quality, still somewhat      │
  │    useful                                                │
  │  - Examples: audio streaming (glitch), UI responsiveness │
  │    video conferencing (choppy but continues)              │
  │  - Statistical guarantees (99.9% meet deadline)          │
  │                                                           │
  │  Value of result over time:                              │
  │                                                           │
  │  Hard RT    │ Firm RT     │ Soft RT                      │
  │  Value      │ Value       │ Value                        │
  │   │████     │  │████      │  │████╲                      │
  │   │████     │  │████      │  │████ ╲                     │
  │   │████     │  │████      │  │████  ╲                    │
  │   │████──── │  │████──── │  │████   ╲──                 │
  │   └──┤──► t │  └──┤──► t │  └──┤─────► t               │
  │      D      │     D       │     D                        │
  │  (instant   │  (drops to  │  (gradual                    │
  │   zero)     │   zero)     │   decay)                     │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Determinism

```
  Determinism in Computing
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Deterministic system: same input + same state            │
  │  → same output + same timing (bounded)                   │
  │                                                           │
  │  Sources of non-determinism to eliminate:                │
  │                                                           │
  │  ┌─────────────────────────┬───────────────────────────┐ │
  │  │ Source                  │ RTOS Solution              │ │
  │  ├─────────────────────────┼───────────────────────────┤ │
  │  │ Cache misses            │ Lock cache, partition      │ │
  │  │ Virtual memory (TLB)    │ No MMU / pinned pages      │ │
  │  │ Interrupt latency       │ Bounded, prioritized IRQs  │ │
  │  │ Dynamic memory alloc    │ Static alloc / pools        │ │
  │  │ Priority inversion      │ Inheritance / ceiling       │ │
  │  │ Unbounded loops/search  │ O(1) algorithms             │ │
  │  │ Garbage collection      │ No GC (C/C++ only)         │ │
  │  │ Network/I/O delays      │ Bounded timeouts            │ │
  │  │ Kernel preemption delay │ Fully preemptible kernel   │ │
  │  └─────────────────────────┴───────────────────────────┘ │
  │                                                           │
  │  RTOS goal: Bounded WCET (Worst-Case Execution Time)    │
  │  for every operation in the system.                      │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. Latency and Jitter

```
  Latency Types in RTOS
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Interrupt Latency:                                      │
  │  Time from hardware interrupt assertion to first         │
  │  instruction of ISR executing                            │
  │                                                           │
  │  IRQ pin ──► [HW recognition] ──► [Vector fetch]        │
  │          ──► [Context save] ──► [ISR first instruction]  │
  │                                                           │
  │  Typical: 1-12 CPU cycles (Cortex-M: 12 cycles)         │
  │                                                           │
  │  Scheduling Latency:                                     │
  │  Time from event making task ready to task actually      │
  │  running on CPU                                          │
  │                                                           │
  │  Event ──► [ISR runs] ──► [Scheduler invoked]            │
  │        ──► [Context switch] ──► [Task first instruction] │
  │                                                           │
  │  Typical: 1-10 μs (depends on RTOS and CPU)             │
  │                                                           │
  │  Jitter:                                                 │
  │  Variation in latency across multiple invocations        │
  │                                                           │
  │  Iteration 1: Latency = 5 μs                            │
  │  Iteration 2: Latency = 7 μs                            │
  │  Iteration 3: Latency = 4 μs                            │
  │  Iteration 4: Latency = 6 μs                            │
  │                                                           │
  │  Average latency = 5.5 μs                               │
  │  Jitter = max - min = 7 - 4 = 3 μs                     │
  │                                                           │
  │  For hard RT: WORST-CASE latency matters, not average   │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Real-Time Performance Metrics

```
  Key RTOS Performance Metrics
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Metric              │ Description          │ Typical     │
  │  ─────────────────────┼──────────────────────┼───────────│
  │  Interrupt latency   │ IRQ → ISR start      │ 1-12 cyc  │
  │  Context switch time │ Task A → Task B      │ 1-5 μs    │
  │  Task response time  │ Event → Task action  │ 2-20 μs   │
  │  Scheduler overhead  │ Priority resolution  │ O(1) time │
  │  WCET               │ Max execution time    │ Analyzed  │
  │  Deadline miss ratio │ % missed deadlines   │ 0% (hard) │
  │  CPU utilization     │ Busy / total time    │ ≤69% (RMS)│
  │  Jitter              │ Latency variation    │ <1 μs     │
  │  Kernel footprint    │ Code + data size     │ 4-64 KB   │
  │  Tick resolution     │ Timer granularity    │ 1-10 ms   │
  └──────────────────────────────────────────────────────────┘
  │                                                           │
  │  CPU Utilization Bounds (schedulability):                │
  │                                                           │
  │  RMS (Rate Monotonic Scheduling):                        │
  │  U = Σ(Ci/Ti) ≤ n(2^(1/n) - 1)                         │
  │                                                           │
  │  n=1: U ≤ 1.000  (100%)                                 │
  │  n=2: U ≤ 0.828  (82.8%)                                │
  │  n=3: U ≤ 0.780  (78.0%)                                │
  │  n→∞: U ≤ 0.693  (69.3%) — the "69% rule"              │
  │                                                           │
  │  EDF (Earliest Deadline First):                          │
  │  U = Σ(Ci/Ti) ≤ 1.000  (100% utilization possible)     │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. RTOS vs General-Purpose OS

```
  RTOS vs GPOS Comparison
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌──────────────┬── RTOS ──────┬── GPOS (Linux) ──────┐ │
  │  │ Priority     │ Correctness  │ Throughput/fairness   │ │
  │  │ Scheduling   │ Deterministic│ Best-effort (CFS)     │ │
  │  │ Latency      │ Bounded μs   │ Unbounded ms range    │ │
  │  │ Memory       │ Static/pools │ Virtual + dynamic     │ │
  │  │ Footprint    │ 4KB - 256KB  │ Megabytes - Gigabytes │ │
  │  │ Preemption   │ Fully preempt│ Preempt points        │ │
  │  │ Interrupts   │ Fast, nested │ Top/bottom halves     │ │
  │  │ Resource     │ Deterministic│ Best-effort           │ │
  │  │   access     │ locking      │                       │ │
  │  │ Applications │ Single       │ Many concurrent       │ │
  │  │              │ embedded app │ apps + users           │ │
  │  │ Boot time    │ Milliseconds │ Seconds               │ │
  │  │ MMU          │ Optional/MPU │ Required (MMU)        │ │
  │  └──────────────┴──────────────┴───────────────────────┘ │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: What makes a system "real-time"?**
**A:** A real-time system must produce correct results AND meet timing deadlines. The key is **guaranteed worst-case timing**, not fast average performance. A system processing in 1μs average but occasionally taking 100ms is NOT real-time if the deadline is 10ms. Real-time means the system's temporal behavior is predictable and bounded.

**Q2: Explain the difference between hard and soft real-time with examples.**
**A:** Hard real-time: a deadline miss causes catastrophic failure — examples include ABS braking (must activate within 5ms or crash), airbag deployment (must fire within 10ms), pacemaker pulses. Formally analyzed with WCET. Soft real-time: deadline misses degrade quality but remain partially useful — examples include audio playback (glitches but continues), video streaming (dropped frames), UI responsiveness. Statistical guarantees (99.9%) are acceptable.

**Q3: What is jitter and why is it problematic?**
**A:** Jitter is the variation in latency across successive invocations. If a periodic task should run every 10ms but actually runs at 10ms, 11ms, 9.5ms, 10.5ms — the jitter is 1.5ms (max-min variation). Jitter is problematic in: motor control (uneven motion), audio processing (clicks/pops), communication protocols (timing violations), sensor sampling (aliasing). RTOS minimizes jitter through deterministic scheduling and bounded interrupt latency.

---

## Summary

- Real-time = correct results + timing guarantees (WCET, not average)
- Hard RT: deadline miss = catastrophe; Soft RT: deadline miss = degraded quality
- Determinism requires eliminating unpredictable delays: cache, VM, dynamic alloc, priority inversion
- Interrupt latency: IRQ → ISR start (cycles); scheduling latency: event → task running (μs)
- Jitter = variation in latency; must be minimized for periodic tasks
- RMS utilization bound: 69.3% for large task sets; EDF can use 100%
- RTOS vs GPOS: determinism vs throughput, bounded μs vs unbounded ms, KB vs GB

---

[Next Chapter: History and Evolution of RTOS →](Chapter_02_History_Evolution.md)
