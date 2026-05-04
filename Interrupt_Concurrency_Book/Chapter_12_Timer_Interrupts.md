# Chapter 12: Timer Interrupts

## Learning Goals
- Understand how Linux uses timer interrupts for scheduling and timekeeping
- Know the classic timer (timer_list) and high-resolution timer (hrtimer) APIs
- Understand periodic ticks, tickless kernels (NO_HZ), and dynamic ticks
- Write driver code using kernel timers and hrtimers

---

## 12.1 System Timers

### Timer Hardware

```
x86 Timer Sources:
  PIT (8254):   1.193182 MHz, IRQ 0, legacy (rarely used now)
  HPET:         ≥10 MHz, memory-mapped, per-comparator IRQs
  LAPIC Timer:  Per-CPU, TSC-derived, most commonly used today
  TSC:          CPU timestamp counter (read-only, no interrupt)
  TSC Deadline: LAPIC mode that fires at exact TSC value

ARM Timer Sources:
  Generic Timer: Per-CPU, PPI ID 27/30, architected
  SoC timers:    Vendor-specific (e.g., Qualcomm QTIMER)

Linux clocksource/clockevent framework:
  clocksource: for reading time (TSC, arch_timer)
  clockevent:  for scheduling interrupts (LAPIC, Generic Timer)
```

### Timer Interrupt Role

```
Timer tick fires → timer interrupt handler:

  scheduler_tick()
      │
      ├── Update jiffies (jiffies_64++)
      ├── update_process_times()
      │   ├── Account CPU time (user/system)
      │   ├── run_local_timers() → raise TIMER_SOFTIRQ
      │   └── scheduler_tick() → CFS vruntime update
      ├── profile_tick() → kernel profiling
      └── Check POSIX timers
      
  TIMER_SOFTIRQ → __run_timers():
      Process expired timer_list timers
      
  HRTIMER_SOFTIRQ / hardirq:
      Process expired hrtimers
```

---

## 12.2 Periodic Timer Interrupts

### CONFIG_HZ

```
CONFIG_HZ determines the base tick frequency:

  CONFIG_HZ │ Period   │ Best For
  ──────────┼──────────┼─────────────────
  100       │ 10 ms    │ Servers (throughput)
  250       │ 4 ms     │ General desktop
  300       │ 3.33 ms  │ Video playback
  1000      │ 1 ms     │ Low-latency, RT

Higher HZ = better timer resolution but more overhead.
jiffies increments once per tick.

jiffies_to_msecs(j) = j * 1000 / HZ
msecs_to_jiffies(ms) = ms * HZ / 1000
```

### Classic Timer API (timer_list)

```c
#include <linux/timer.h>

struct timer_list my_timer;

/* Timer callback (runs in softirq context — cannot sleep!) */
static void my_timer_callback(struct timer_list *t)
{
    struct my_device *dev = from_timer(dev, t, timer);
    
    /* Do periodic work */
    check_device_status(dev);
    
    /* Re-arm for next period */
    mod_timer(&dev->timer, jiffies + msecs_to_jiffies(100));
}

/* Setup */
timer_setup(&my_timer, my_timer_callback, 0);

/* Start: fire after 100ms */
mod_timer(&my_timer, jiffies + msecs_to_jiffies(100));

/* Modify timeout */
mod_timer(&my_timer, jiffies + HZ);  /* 1 second from now */

/* Cancel */
del_timer(&my_timer);        /* May still be running on another CPU! */
del_timer_sync(&my_timer);   /* Wait for completion — SAFE */

/* Resolution: 1/HZ (1ms at HZ=1000, 4ms at HZ=250)
   Cannot be more precise than jiffies resolution.
   For sub-jiffie precision → use hrtimers */
```

---

## 12.3 High Resolution Timers (hrtimers)

### Concept

```
hrtimers provide nanosecond-resolution timers,
independent of CONFIG_HZ jiffies tick.

  timer_list:  resolution = 1/HZ (1-10 ms)
  hrtimer:     resolution = hardware timer (nanoseconds)

  Used by: POSIX timers, nanosleep, scheduling, SCHED_DEADLINE

hrtimer uses red-black tree ordered by expiry time.
Hardware programmed to interrupt at next expiry.
```

### hrtimer API

```c
#include <linux/hrtimer.h>

struct my_device {
    struct hrtimer hr_timer;
    ktime_t period;
};

/* Callback — runs in hardirq or softirq context depending on mode */
static enum hrtimer_restart my_hrtimer_callback(struct hrtimer *timer)
{
    struct my_device *dev = container_of(timer, struct my_device, hr_timer);
    
    /* Do precise periodic work */
    sample_sensor(dev);
    
    /* Re-arm: forward timer by period */
    hrtimer_forward_now(timer, dev->period);
    return HRTIMER_RESTART;  /* Automatically re-arm */
    /* Or HRTIMER_NORESTART to stop */
}

/* Setup */
dev->period = ktime_set(0, 1000000);  /* 1ms = 1,000,000 ns */
hrtimer_init(&dev->hr_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
dev->hr_timer.function = my_hrtimer_callback;

/* Start */
hrtimer_start(&dev->hr_timer, dev->period, HRTIMER_MODE_REL);

/* Cancel */
hrtimer_cancel(&dev->hr_timer);  /* Wait for callback to finish */

/* Check if active */
if (hrtimer_active(&dev->hr_timer))
    pr_info("Timer is running\n");
```

### hrtimer Internals

```
  Per-CPU hrtimer bases (one per clock type):
  ┌─────────────────────────────────────────┐
  │  hrtimer_cpu_base (per-CPU)             │
  │  ├── clock_base[CLOCK_MONOTONIC]        │
  │  │   └── RB-tree of hrtimers           │
  │  │       [5ms] → [12ms] → [100ms]      │
  │  │        ↑ next to expire              │
  │  ├── clock_base[CLOCK_REALTIME]         │
  │  │   └── RB-tree...                    │
  │  └── clock_base[CLOCK_BOOTTIME]         │
  │      └── RB-tree...                    │
  └─────────────────────────────────────────┘

  Hardware timer programmed to fire at:
    earliest expiry across all bases = "next_event"
  
  When hardware timer fires:
    hrtimer_interrupt() → __hrtimer_run_queues()
    → Execute all expired callbacks
    → Reprogram hardware for next earliest expiry
```

---

## 12.4 Tickless Kernel (NO_HZ)

### The Problem with Periodic Ticks

```
Traditional: Timer fires every 1/HZ seconds, whether needed or not.

  ──┬──┬──┬──┬──┬──┬──┬──┬──→ time
    │  │  │  │  │  │  │  │    (HZ=1000: 1000 IRQs/second/CPU)
  tick tick tick tick tick...

  When CPU is idle: 1000 wakeups/second → wasted power!
  When running single task: overhead with no benefit!
```

### NO_HZ Modes

```
1. CONFIG_NO_HZ_IDLE (default for most kernels):
   Stop ticks when CPU is idle.
   
   ──┬──┬──┬──────────────────┬──┬──→
     │  │  │   CPU idle        │  │
   tick tick (no ticks!)     tick tick
   
   Saves power: CPU stays in deep C-states.

2. CONFIG_NO_HZ_FULL (adaptive ticks):
   Stop ticks even when ONE task is running.
   
   ──┬──────────────────────────────→
     │                              
   tick (single userspace task runs without ticks!)
   
   Benefit: No overhead for CPU-bound tasks.
   Requirement: nohz_full=<cpulist> boot parameter
   Used for: HPC, RT, latency-sensitive workloads

3. CONFIG_HZ_PERIODIC (legacy):
   Always tick. No power savings.
```

### NO_HZ_FULL Detail

```bash
# Enable full tickless on CPUs 2-3:
# Kernel command line: nohz_full=2,3

# When CPU 2 has exactly ONE runnable task:
#   Timer ticks STOP on CPU 2
#   CPU 2 runs without interruption
#   If another task becomes runnable on CPU 2:
#     Ticks restart (scheduler needs them)

# Check status:
$ cat /sys/devices/system/cpu/nohz_full
2-3

# Requirements:
# - At least one housekeeping CPU (CPU 0 usually)
# - RCU callbacks offloaded (rcu_nocbs=2,3)
# - No timers queued on those CPUs
```

---

## Timer Subsystem Architecture

```
                     User Space
                ─────────────────
  nanosleep()  →  hrtimer_nanosleep()
  timer_create →  posix_timer → hrtimer
  setitimer    →  itimer → alarm signal
                ─────────────────
  
  ┌──────────────────────────────────────┐
  │         Timer Subsystem              │
  │                                       │
  │  ┌─────────────┐  ┌───────────────┐  │
  │  │ timer_list   │  │ hrtimer       │  │
  │  │ (jiffies     │  │ (nanosecond   │  │
  │  │  resolution) │  │  resolution)  │  │
  │  │ Hash wheel   │  │ RB-tree       │  │
  │  └──────┬──────┘  └──────┬────────┘  │
  │         │                │            │
  │  TIMER_SOFTIRQ    Hardware timer     │
  │  (from tick)       (LAPIC/Generic)   │
  └──────────────────────────────────────┘
               │
  ┌────────────▼─────────────────────────┐
  │     clocksource / clockevent         │
  │  TSC, LAPIC Timer, Generic Timer     │
  └──────────────────────────────────────┘
```

---

## Kernel Source References

```
Timer infrastructure:
  kernel/time/timer.c          ← timer_list (classic timers)
  kernel/time/hrtimer.c        ← High-resolution timers
  kernel/time/tick-common.c    ← Periodic tick handling
  kernel/time/tick-sched.c     ← Dynamic tick (NO_HZ)
  kernel/time/tick-broadcast.c ← Tick broadcast (deep idle)
  kernel/time/clocksource.c    ← Clock source management
  kernel/time/clockevents.c    ← Clock event (timer HW) mgmt

Key includes:
  include/linux/timer.h        ← timer_list API
  include/linux/hrtimer.h      ← hrtimer API
  include/linux/jiffies.h      ← Jiffies conversion macros
```

---

## Interview Questions

1. **What is the difference between timer_list and hrtimer?**
2. **What does CONFIG_HZ control? What are common values?**
3. **Explain NO_HZ_IDLE and NO_HZ_FULL. When would you use each?**
4. **What is jiffies? How does it relate to CONFIG_HZ?**
5. **Write code to create an hrtimer that fires every 500µs.**
6. **What happens in scheduler_tick()? List the major actions.**
7. **Why is del_timer_sync() needed instead of del_timer()?**
8. **What callback return value re-arms an hrtimer?**
9. **What hardware timer does Linux typically use on modern x86?**
10. **How does the tickless kernel save power?**
11. **What is tick broadcast and when is it needed?**
12. **Compare hrtimer resolution vs timer_list resolution. Give examples.**

---

## Summary

- Timer interrupts drive scheduling, accounting, and timekeeping in the kernel
- timer_list: jiffies-resolution (1-10ms), hash-wheel, for coarse timeouts
- hrtimer: nanosecond-resolution, RB-tree, for precise timing needs
- NO_HZ_IDLE stops ticks when idle (power savings); NO_HZ_FULL stops ticks for single-task CPUs
- Modern kernels rely on hrtimers for scheduling (SCHED_DEADLINE, nanosleep)
- Understanding timer architecture is essential for RT design and power optimization

---

*Next: [Chapter 13 — Concurrency in Operating Systems](Chapter_13_Concurrency_Concepts.md)*
