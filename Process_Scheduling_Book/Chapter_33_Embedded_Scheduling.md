# Chapter 33: Embedded System Scheduling

## Learning Goals
- Understand scheduling challenges unique to embedded Linux
- Learn real-time constraints and how to meet them
- Master deterministic scheduling techniques for automotive, audio, industrial
- Know Android-specific scheduling optimizations

---

## 33.1 Embedded Scheduling Challenges

```
Embedded systems differ from servers/desktops:

  ┌────────────────────┬─────────────────────┬──────────────────────┐
  │ Challenge           │ Server/Desktop       │ Embedded              │
  ├────────────────────┼─────────────────────┼──────────────────────┤
  │ Deadline            │ Best-effort          │ Hard/soft deadlines  │
  │ Power               │ Not primary concern  │ Battery/thermal limit│
  │ CPU                 │ High-performance     │ Often limited cores  │
  │ Memory              │ Abundant (GB+)       │ Limited (MB-GB)      │
  │ Latency require.    │ ~10-100ms OK         │ ~10µs - 10ms critical│
  │ Determinism         │ Statistical average  │ Worst-case bounded   │
  │ Certification       │ Rarely needed        │ Safety-critical (ISO)│
  └────────────────────┴─────────────────────┴──────────────────────┘
```

---

## 33.2 Automotive Linux Scheduling

### Qualcomm SA8155P / SA8295P Platform

```
Typical automotive SoC (e.g., SA8155P):
  8 cores: 4× Kryo Gold (big) + 4× Kryo Silver (little)
  
  CPU Partitioning:
  ┌──────────────────────────────────────────────────────┐
  │ CPU 0-1 (little): Android framework, system services │
  │ CPU 2-3 (little): Background, non-critical tasks     │
  │ CPU 4-5 (big):    Instrument cluster, HMI rendering  │
  │ CPU 6-7 (big):    Camera/ADAS processing             │
  └──────────────────────────────────────────────────────┘

  Scheduling configuration:
    CPU 4-7: isolcpus + nohz_full (isolated for RT)
    Camera pipeline: SCHED_FIFO priority 50-80
    Display rendering: SCHED_FIFO priority 40-60
    Audio: SCHED_DEADLINE (runtime=2ms, period=10ms)
    Android UI: SCHED_NORMAL with EAS
```

### Android Scheduling (AOSP)

```
Android's scheduling stack:

  ┌─────────────────────────────────────────┐
  │  App Layer: Activities, Services         │
  │  → Managed by ActivityManager            │
  ├─────────────────────────────────────────┤
  │  Framework: Priority grouping            │
  │  TOP_APP:     nice -10, big cores       │
  │  FOREGROUND:  nice 0                     │
  │  BACKGROUND:  nice +10, limited cpuset  │
  │  RESTRICTED:  nice +15, throttled       │
  ├─────────────────────────────────────────┤
  │  Kernel: CFS + EAS + cgroups            │
  │  cpuset: pin groups to cores            │
  │  cpu.shares: weight between groups      │
  │  schedtune: utilization clamping        │
  ├─────────────────────────────────────────┤
  │  Hardware: big.LITTLE, EAS energy model │
  └─────────────────────────────────────────┘

  Android-specific scheduling features:
    - SchedTune / uclamp: min/max utilization clamping
    - cpuset partitioning (top-app gets big cores)
    - Binder priority inheritance
    - ADPF (Android Dynamic Performance Framework)
```

### Uclamp (Utilization Clamping)

```
Uclamp constrains PELT util_avg for frequency/placement decisions:

  uclamp_min = 0:    Allow task on least powerful CPU, lowest freq
  uclamp_min = 512:  Guarantee minimum 50% CPU capacity (keep on big core)
  uclamp_max = 1024: Allow maximum capacity
  uclamp_max = 300:  Cap at 30% capacity (power saving)

Usage:
  # Per-task (sched_setattr):
  attr.sched_util_min = 512;  /* Min 50% capacity */
  attr.sched_util_max = 1024; /* No cap */

  # Per-cgroup:
  echo 512 > /sys/fs/cgroup/top-app/cpu.uclamp.min
  echo 1024 > /sys/fs/cgroup/top-app/cpu.uclamp.max
  echo 0 > /sys/fs/cgroup/background/cpu.uclamp.min
  echo 300 > /sys/fs/cgroup/background/cpu.uclamp.max

Impact:
  UI thread: uclamp_min=512 → always on big core → smooth 60fps
  Background: uclamp_max=300 → stays on little core → saves power
```

---

## 33.3 Audio/Video Real-Time Scheduling

### Audio Pipeline

```
Audio requirement:
  Buffer size: 256 samples @ 48kHz = 5.33ms deadline
  Processing time: ~1-2ms per buffer
  
  MUST deliver audio buffer every 5.33ms or audible glitch!

Scheduling setup:
  Audio thread: SCHED_FIFO priority 70
  OR: SCHED_DEADLINE (runtime=2ms, deadline=5ms, period=5ms)

  # Using SCHED_DEADLINE for audio
  struct sched_attr attr = {
      .sched_policy   = SCHED_DEADLINE,
      .sched_runtime  =  2000000,  /* 2ms */
      .sched_deadline =  5000000,  /* 5ms */
      .sched_period   =  5000000,  /* 5ms */
  };

Timeline:
  |--process--|--idle--|--process--|--idle--|
  0          2ms     5ms       7ms      10ms
  |<-deadline 5ms->|  |<-deadline 5ms->|
```

### Camera Pipeline (Automotive)

```
Camera requirements (surround view / ADAS):
  30fps → 33ms per frame
  4 cameras, 1920×1080 each
  
  Pipeline stages (each a thread):
    1. ISP capture: SCHED_FIFO prio 80 (highest, hardware-driven)
    2. Dewarping:   SCHED_FIFO prio 60 (GPU-accelerated)
    3. Stitching:   SCHED_FIFO prio 50 (CPU-intensive)
    4. Display:     SCHED_FIFO prio 70 (must meet vsync)
    
  CPU affinity:
    ISP/Display threads → isolated big cores (CPU 6-7)
    Processing threads → big cores (CPU 4-5)
    Android UI → little cores (CPU 0-3)
```

---

## 33.4 PREEMPT_RT for Embedded

```
PREEMPT_RT configuration for embedded:

  Kernel config:
    CONFIG_PREEMPT_RT=y
    CONFIG_HZ=1000          # Fine-grained ticks
    CONFIG_NO_HZ_FULL=y     # Tickless on isolated CPUs
    CONFIG_HIGH_RES_TIMERS=y

  Boot parameters:
    isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3
    # Isolate CPUs 2,3 for RT work
    # No timer ticks on those CPUs
    # No RCU callbacks on those CPUs

  RT thread setup:
    mlockall(MCL_CURRENT | MCL_FUTURE);  /* Lock all pages in RAM */
    /* Avoid page faults during RT execution */
    
    /* Set SCHED_FIFO */
    struct sched_param param = { .sched_priority = 80 };
    sched_setscheduler(0, SCHED_FIFO, &param);
    
    /* Pin to isolated CPU */
    cpu_set_t mask;
    CPU_ZERO(&mask);
    CPU_SET(2, &mask);
    sched_setaffinity(0, sizeof(mask), &mask);
```

### Measuring RT Latency

```bash
# cyclictest: standard RT latency benchmark
cyclictest -m -S -p 98 -D 5m -h 400 -q
# -m: lock memory
# -S: SMP mode (one thread per CPU)
# -p 98: SCHED_FIFO priority 98
# -D 5m: run for 5 minutes
# -h 400: histogram up to 400µs

# Results:
# T: 0 Min:  1 Act: 3 Avg: 5 Max: 35    ← CPU 0
# T: 1 Min:  1 Act: 2 Avg: 4 Max: 28    ← CPU 1
# T: 2 Min:  1 Act: 2 Avg: 3 Max: 12    ← CPU 2 (isolated)
# T: 3 Min:  1 Act: 2 Avg: 3 Max: 10    ← CPU 3 (isolated)

# Isolated CPUs show much lower max latency!

# hwlatdetect: detect hardware-induced latency
hwlatdetect --duration=60
# Reports SMI, NMI, or other hardware latency spikes
```

---

## 33.5 Power-Aware Scheduling for Mobile

```
Energy Aware Scheduling (EAS) + schedutil:

  Task arrives (util_avg = 200):
    1. EAS checks energy model:
       Little core (cap=400): can handle, power=50mW
       Big core (cap=1024):   can handle, power=200mW
       → Place on little core (saves 150mW)

    2. schedutil sets CPU frequency:
       Required freq = util_avg / capacity × max_freq
       = 200/400 × 1.8GHz = 900MHz
       
    3. If utilization increases (util_avg → 500):
       Little core capacity exceeded (400 < 500)
       → Migrate to big core (misfit migration)
       → schedutil adjusts big core frequency

  Power optimization techniques:
    - uclamp_max: cap background task utilization
    - cpufreq governor: schedutil (integrated with scheduler)
    - CPU idle: cpuidle selects deepest sleep when idle
    - Core parking: keep unused cores in deep idle
```

---

## 33.6 Industrial Control Scheduling

```
PLC-like control loop on embedded Linux:

  Control loop: read sensor → compute → actuate → repeat

  Requirements:
    Period: 1ms (1kHz control loop)
    Jitter: < 50µs
    
  Setup:
    PREEMPT_RT kernel
    SCHED_DEADLINE (runtime=200µs, period=1ms)
    Isolated CPU, mlockall, pre-faulted stack
    
  Timing:
    |--read--|--compute--|--actuate--|--idle--|
    0       50µs       150µs       200µs     1ms
    |<─────── period = 1ms ────────────────>|

  Anti-patterns to AVOID:
    ✗ malloc/free in control loop (non-deterministic)
    ✗ printf/logging (can block on I/O)
    ✗ File I/O in hot path
    ✗ Dynamic memory mapping
    ✗ Non-RT system calls (select, poll without timeout)
    
  Best practices:
    ✓ Pre-allocate all memory before entering RT loop
    ✓ Use lock-free data structures for communication
    ✓ Use real-time safe logging (shared memory ring buffer)
    ✓ Validate worst-case with cyclictest under stress
    ✓ Monitor with ftrace (minimal overhead)
```

---

## 33.7 Embedded Scheduling Checklist

```
For any embedded Linux RT application:

□ Kernel configuration:
  □ CONFIG_PREEMPT or CONFIG_PREEMPT_RT
  □ CONFIG_HIGH_RES_TIMERS=y
  □ CONFIG_NO_HZ_FULL for isolated CPUs

□ CPU isolation:
  □ isolcpus= boot parameter
  □ nohz_full= for tickless RT cores
  □ rcu_nocbs= to keep RCU off RT cores
  □ irqaffinity= for non-RT cores

□ Application setup:
  □ mlockall(MCL_CURRENT | MCL_FUTURE)
  □ Pre-fault stack (alloca or dummy access)
  □ Set SCHED_FIFO/DEADLINE appropriately
  □ Pin to isolated CPU (sched_setaffinity)

□ Avoid non-deterministic operations:
  □ No malloc/free in critical path
  □ No file I/O in critical path
  □ No unbounded loops
  □ No blocking on non-RT locks

□ Validation:
  □ Run cyclictest under CPU/IO/memory stress
  □ Verify max latency meets deadline
  □ Monitor with ftrace/perf in production
```

---

## Interview Questions

**Q1: How would you design the scheduling for an automotive instrument cluster running on Linux?**
A: 1) Use PREEMPT_RT kernel for deterministic latency. 2) Isolate 2 big cores (isolcpus + nohz_full) for display rendering. 3) Display compositor thread: SCHED_FIFO priority 70, pinned to isolated cores. 4) CAN bus reader: SCHED_DEADLINE (runtime=500µs, period=10ms). 5) Android UI (non-safety): SCHED_NORMAL on non-isolated small cores. 6) mlockall for all RT processes to avoid page faults. 7) Validate with cyclictest under worst-case load (max latency < 1ms for 60fps).

**Q2: Why is `mlockall()` critical for real-time applications?**
A: Without mlockall, page faults can occur when accessing code or data not yet paged in. A major page fault requires disk I/O (or swap) — taking 1-10ms, violating most RT deadlines. mlockall locks all current and future pages in physical RAM, guaranteeing no page faults. Combined with pre-faulting the stack (touching all stack pages up-front), this eliminates one of the biggest sources of non-deterministic latency.

**Q3: Compare `uclamp_min` vs `SCHED_FIFO` for ensuring an Android app's UI thread gets enough CPU.**
A: `uclamp_min` sets a minimum CPU capacity hint (e.g., 512 → guaranteed big core), working within CFS fairness. The task still shares CPU proportionally but the scheduler ensures it runs on a capable core at sufficient frequency. `SCHED_FIFO` gives absolute priority over all normal tasks — RT threads preempt everything. For Android UI: uclamp_min is preferred because it works with EAS for power efficiency and doesn't require root/capabilities. SCHED_FIFO is overkill for UI and risks starving system services.

---

*Next: [Chapter 34 — Documentation and References](Chapter_34_References.md)*
