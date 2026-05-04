# Chapter 28: Kernel Tracing for Scheduling

## Learning Goals
- Master ftrace for tracing scheduler events
- Learn tracepoints relevant to scheduling
- Use trace-cmd and perf for latency analysis
- Understand Android's systrace/perfetto for scheduling analysis

---

## 28.1 ftrace Fundamentals

```bash
# ftrace lives in: /sys/kernel/debug/tracing/
# (or /sys/kernel/tracing/ on some systems)

# Available tracers:
cat /sys/kernel/debug/tracing/available_tracers
# function function_graph nop

# Available scheduling events:
ls /sys/kernel/debug/tracing/events/sched/
# sched_switch       sched_wakeup      sched_wakeup_new
# sched_migrate_task sched_process_fork sched_process_exit
# sched_process_exec sched_stat_wait   sched_stat_sleep
# sched_stat_runtime sched_pi_setprio  sched_process_free
```

---

## 28.2 Tracing Context Switches (sched_switch)

```bash
# Enable sched_switch tracepoint
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# Let it run for a few seconds
sleep 2

# Read trace
cat /sys/kernel/debug/tracing/trace | head -30

# Output format:
#          <idle>-0   [001] d.. 1000.001: sched_switch: prev_comm=swapper/1
#    prev_pid=0 prev_prio=120 prev_state=S ==> next_comm=my_app
#    next_pid=1234 next_prio=120
#
#  Fields:
#    prev_comm/pid/prio/state — task being switched OUT
#    next_comm/pid/prio       — task being switched IN
#    [001]                    — CPU number
#    d..                      — flags: d=irqs disabled, .=preempt ok

# Disable tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo 0 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
```

---

## 28.3 Tracing Wakeups (sched_wakeup)

```bash
# Enable wakeup tracing
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_wakeup/enable
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_wakeup_new/enable

# Output:
# kworker/0:1-42 [000] d.. 1000.500: sched_wakeup: comm=my_app
#    pid=1234 prio=120 target_cpu=002
#
# Interpretation:
#   kworker (PID 42) on CPU 0 woke up my_app (PID 1234)
#   my_app will run on CPU 2 (target_cpu)
#   prio=120 → nice 0

# Combine sched_switch + sched_wakeup for full picture:
# wakeup timestamp → switch timestamp = SCHEDULING LATENCY
```

---

## 28.4 Scheduling Statistics Tracepoints

```bash
# sched_stat_wait — time spent waiting on runqueue
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_stat_wait/enable

# Output:
# my_app-1234 [002] 1001.000: sched_stat_wait: comm=my_app pid=1234
#    delay=50000 [ns]
#    → Task waited 50µs on runqueue before getting CPU

# sched_stat_sleep — time spent sleeping
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_stat_sleep/enable
# delay=5000000000 [ns] → slept for 5 seconds

# sched_stat_runtime — actual CPU runtime
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_stat_runtime/enable
# runtime=2000000 [ns] vruntime=98765432100 [ns]
```

---

## 28.5 trace-cmd — Convenient ftrace Frontend

```bash
# Record scheduling events (better than manual ftrace)
trace-cmd record -e sched sleep 5

# Analyze the recording
trace-cmd report | head -50

# Filter for specific process
trace-cmd report -F 'sched_switch: next_comm ~ "my_app"'

# Record specific events only
trace-cmd record -e sched_switch -e sched_wakeup -e sched_migrate_task sleep 5

# Record with function tracing
trace-cmd record -p function_graph -g scheduler_tick sleep 2
trace-cmd report

# Output:
#  0)               |  scheduler_tick() {
#  0)               |    task_tick_fair() {
#  0)               |      update_curr() {
#  0)   0.500 us    |        calc_delta_fair();
#  0)   0.300 us    |        update_min_vruntime();
#  0)   1.500 us    |      }
#  0)   0.200 us    |      check_preempt_tick();
#  0)   3.000 us    |    }
#  0)   4.000 us    |  }
```

---

## 28.6 perf sched — Scheduler-Specific Analysis

```bash
# Record all scheduling events
perf sched record -- sleep 10
# or for a specific command:
perf sched record -- ./my_app

# Scheduling latency summary
perf sched latency
#  Task                  | Runtime ms | Switches | Avg delay ms | Max delay ms
# ────────────────────────────────────────────────────────────────────────────
# audio_thread:200       |    500.00  |     5000 |        0.050 |        2.500
# my_app:1234            |   5000.00  |     1500 |        0.100 |       15.000
# kworker/0:42           |     50.00  |      200 |        0.010 |        0.500
# ────────────────────────────────────────────────────────────────────────────
# TOTAL:                 |   7000.00  |     8000 |        0.070 |       15.000

# Max delay shows worst-case scheduling latency!

# CPU usage timeline (text visualization)
perf sched map
# *A0  B0  .   .       A = my_app, B = bash, . = idle
# *A0  .   B0  .       Numbers = CPU ID
#  .   .  *A0  .
# Asterisk = currently running

# Detailed time history
perf sched timehist
# time       cpu task                wait   sch delay  run time
# 1000.001   [0] my_app:1234         5ms     0.05ms     2.00ms
# 1002.005   [0] <idle>                       -         1.99ms
# 1004.000   [0] bash:500            10ms    0.02ms     0.50ms

# Script for custom analysis
perf sched script
# raw trace events, parseable for custom tools
```

---

## 28.7 Wakeup Latency Tracing

```bash
# ftrace wakeup latency tracer — measures worst-case
echo wakeup > /sys/kernel/debug/tracing/current_tracer

# For RT tasks specifically:
echo wakeup_rt > /sys/kernel/debug/tracing/current_tracer

# Run workload, then check:
cat /sys/kernel/debug/tracing/tracing_max_latency
# 150  (microseconds) ← worst-case wakeup-to-running latency

# Reset max:
echo 0 > /sys/kernel/debug/tracing/tracing_max_latency

# View the trace that caused max latency:
cat /sys/kernel/debug/tracing/trace
# Shows full call chain from wakeup to actual scheduling
```

---

## 28.8 Function Graph Tracing

```bash
# Trace specific scheduler functions
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo schedule > /sys/kernel/debug/tracing/set_graph_function
echo __schedule >> /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on

# View call graph:
cat /sys/kernel/debug/tracing/trace
#  0)               |  __schedule() {
#  0)   0.300 us    |    rcu_note_context_switch();
#  0)               |    pick_next_task_fair() {
#  0)               |      update_curr() {
#  0)   0.200 us    |        calc_delta_fair();
#  0)   0.700 us    |      }
#  0)   0.300 us    |      __pick_first_entity();
#  0)   1.500 us    |    }
#  0)               |    context_switch() {
#  0)   0.200 us    |      switch_mm_irqs_off();
#  0)   0.500 us    |      switch_to();
#  0)   1.000 us    |    }
#  0)   5.000 us    |  }
```

---

## 28.9 Android Perfetto / Systrace

```bash
# Android perfetto (modern replacement for systrace)
# Record scheduling events on Android:
adb shell perfetto \
  -c - --txt \
  -o /data/misc/perfetto-traces/trace.pftrace <<EOF
buffers: {
  size_kb: 32768
}
data_sources: {
  config {
    name: "linux.ftrace"
    ftrace_config {
      ftrace_events: "sched/sched_switch"
      ftrace_events: "sched/sched_wakeup"
      ftrace_events: "sched/sched_blocked_reason"
      ftrace_events: "power/cpu_frequency"
      ftrace_events: "power/cpu_idle"
    }
  }
}
duration_ms: 10000
EOF

# Pull trace and open in Perfetto UI (ui.perfetto.dev)
adb pull /data/misc/perfetto-traces/trace.pftrace

# Legacy systrace (still works):
python systrace.py -o trace.html sched freq idle am wm
# Opens in Chrome for visual timeline analysis

# Key things to look for in Perfetto:
#   1. Thread state colors: Running (green), Runnable (blue), 
#      Sleeping (gray), Uninterruptible (orange)
#   2. Scheduling latency: gap between Runnable → Running
#   3. CPU frequency changes during critical path
#   4. Binder transactions and their latency
```

---

## 28.10 Custom Tracepoints with BPF

```bash
# Using bpftrace for custom scheduling analysis

# Count context switches per process
bpftrace -e 'tracepoint:sched:sched_switch { @[args->next_comm] = count(); }'

# Histogram of scheduling latency
bpftrace -e '
tracepoint:sched:sched_wakeup { @start[args->pid] = nsecs; }
tracepoint:sched:sched_switch /args->prev_pid/ {
    if (@start[args->next_pid]) {
        @latency = hist(nsecs - @start[args->next_pid]);
        delete(@start[args->next_pid]);
    }
}'

# Track which process wakes which
bpftrace -e '
tracepoint:sched:sched_wakeup {
    printf("%-16s (pid %d) woke %-16s (pid %d)\n",
           comm, pid, args->comm, args->pid);
}'

# Monitor runqueue length per CPU
bpftrace -e '
tracepoint:sched:sched_switch {
    @rqlen[cpu] = hist(curtask->se.cfs_rq->nr_running);
}'
```

---

## 28.11 Tracing Cheat Sheet

```
┌────────────────────────┬──────────────────────────────────────┐
│ Want to know            │ Tracing command                      │
├────────────────────────┼──────────────────────────────────────┤
│ Context switch rate     │ perf stat -e context-switches       │
│ Which tasks switch      │ perf sched record + perf sched map  │
│ Scheduling latency      │ perf sched latency                  │
│ Worst-case RT latency   │ ftrace wakeup_rt tracer             │
│ Why task was preempted   │ trace sched_switch prev_state       │
│ Who woke a task          │ trace sched_wakeup                  │
│ CPU migration events    │ trace sched_migrate_task             │
│ Scheduler function time │ function_graph tracer on __schedule │
│ Per-task wait time      │ trace sched_stat_wait                │
│ Android frame drops     │ perfetto with sched + gfx events     │
│ Custom analysis          │ bpftrace with sched tracepoints     │
└────────────────────────┴──────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How do you measure scheduling latency for a real-time audio application?**
A: 1) Use `perf sched record` while the audio app runs, then `perf sched latency --sort max` to find the worst-case delay. 2) Use ftrace's `wakeup_rt` tracer to find the maximum wakeup-to-running latency. 3) Use `cyclictest` (rt-tests package) as a dedicated RT latency benchmark. 4) On Android, use Perfetto with sched events and look for gaps between "Runnable" (blue) and "Running" (green) states for the audio thread. The audio buffer size divided by sample rate gives the deadline — latency must stay below it.

**Q2: What does a `prev_state=R+` in sched_switch mean?**
A: `prev_state=R+` means the previous task was in TASK_RUNNING state when switched out, and the `+` indicates it was preempted (involuntary switch). `R` = still runnable (wants CPU). This means a higher-priority task arrived, or the timer tick determined the current task's timeslice/vruntime was exceeded. Compare with `prev_state=S` (voluntary sleep) or `prev_state=D` (uninterruptible block).

**Q3: How would you use bpftrace to find which tasks are causing the most scheduling delays?**
A: Use the wakeup-to-switch latency pattern: record timestamp at `sched_wakeup` for each PID, then compute the delta at `sched_switch` when that PID runs. Store results in a histogram keyed by task name. Tasks with long tail latency are suffering scheduling delays. Additionally, track the `prev_comm` at each switch to identify which tasks are holding the CPU when delayed tasks are waiting — these "blockers" may need lower priority or different CPU affinity.

---

*Next: [Chapter 29 — Flow Diagrams](Chapter_29_Flow_Diagrams.md)*
