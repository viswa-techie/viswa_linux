# Chapter 27: Kernel Tracing for Interrupts and Concurrency

## Learning Goals
- Use ftrace to trace interrupt handling and latency
- Use trace-cmd and perf for interrupt analysis
- Trace lock contention and softirq execution
- Measure worst-case latency for RT systems
- Use eBPF/bpftrace for custom interrupt tracing

---

## 27.1 Ftrace Overview

Ftrace is the kernel's built-in tracer. It lives in `/sys/kernel/debug/tracing/`.

```bash
# Mount debugfs (if not already mounted):
mount -t debugfs debugfs /sys/kernel/debug

# Key files:
/sys/kernel/debug/tracing/
  available_tracers        # List available tracers
  current_tracer           # Active tracer
  tracing_on               # 1=tracing, 0=stopped
  trace                    # Trace output (text)
  trace_pipe               # Streaming trace output
  set_ftrace_filter        # Function filter
  available_events         # All tracepoints
  events/                  # Per-subsystem event controls
```

---

## 27.2 Tracing Interrupt Events

### IRQ Events

```bash
# Enable IRQ entry/exit tracing:
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable

# Enable softirq events:
echo 1 > /sys/kernel/debug/tracing/events/irq/softirq_entry/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/softirq_exit/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/softirq_raise/enable

# Start tracing:
echo 1 > /sys/kernel/debug/tracing/tracing_on

# View trace:
cat /sys/kernel/debug/tracing/trace
```

### Sample Output

```
# tracer: nop
#              _-----=> irqs-off
#             / _----=> need-resched
#            | / _---=> hardirq/softirq
#            || / _--=> preempt-depth
#            ||| /     delay
#  TASK-PID  ||||   TIMESTAMP  FUNCTION
#     | |    ||||      |         |
  <idle>-0   d..1  1234.567890: irq_handler_entry: irq=25 name=nvme0q1
  <idle>-0   d..1  1234.567892: irq_handler_exit: irq=25 ret=handled
  <idle>-0   ..s1  1234.567893: softirq_entry: vec=9 [action=RCU]
  <idle>-0   ..s1  1234.567894: softirq_exit: vec=9 [action=RCU]
```

### Reading the Flags

```
Flags column (d..1):
  d = interrupts disabled (. = enabled)
  . = need_resched not set (N = set)
  . = not in softirq (s = in softirq, h = in hardirq, H = both)
  1 = preempt_depth

Timestamps: seconds.microseconds (configurable to nanoseconds)
```

---

## 27.3 Latency Tracers

### irqsoff — Longest IRQ-disabled Section

```bash
echo irqsoff > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# Run workload...
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

```
# tracer: irqsoff
# irqsoff latency trace v1.1.5 on 6.1.0
# latency: 245 us, #4/4, CPU#2 | (M:preempt VP:0, KP:0, SP:0 HP:0)
#    -----------------
#    | task: kworker/2:1-1234 (uid:0 nice:0 policy:0 rt_prio:0)
#    -----------------
#  => started at: _raw_spin_lock_irqsave
#  => ended at:   _raw_spin_unlock_irqrestore
#
#                  _------=> CPU#
#                 / _-----=> irqs-off
#                | / _----=> need-resched
#                || /
#  cmd     pid   |||   time  |   caller
#     \   /      |||    \    |   /
  kworker-1234   2d..  0us  : _raw_spin_lock_irqsave
  kworker-1234   2d..  45us : process_data
  kworker-1234   2d.. 245us : _raw_spin_unlock_irqrestore
```

### preemptoff — Longest Preemption-disabled Section

```bash
echo preemptoff > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

### preemptirqsoff — Combined (Worst Overall Latency)

```bash
echo preemptirqsoff > /sys/kernel/debug/tracing/current_tracer
```

### Wakeup Latency (RT Tasks)

```bash
echo wakeup_rt > /sys/kernel/debug/tracing/current_tracer
# Measures worst-case scheduling latency for RT tasks
```

---

## 27.4 Function Tracing

### Trace Specific Functions

```bash
echo function > /sys/kernel/debug/tracing/current_tracer

# Only trace IRQ-related functions:
echo 'do_IRQ' > /sys/kernel/debug/tracing/set_ftrace_filter
echo '__do_softirq' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'handle_irq_event' >> /sys/kernel/debug/tracing/set_ftrace_filter

echo 1 > /sys/kernel/debug/tracing/tracing_on
```

### Function Graph Tracer

```bash
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 'handle_irq_event_percpu' > /sys/kernel/debug/tracing/set_graph_function

cat /sys/kernel/debug/tracing/trace
```

```
 2)               |  handle_irq_event_percpu() {
 2)               |    my_irq_handler() {
 2)   0.345 us    |      readl();
 2)   0.123 us    |      writel();
 2)   0.098 us    |      wake_up_interruptible();
 2)   2.456 us    |    }
 2)   3.012 us    |  }
```

---

## 27.5 trace-cmd (User-Space Tool)

```bash
# Install:
apt install trace-cmd   # or: yum install trace-cmd

# Record IRQ events:
trace-cmd record -e irq -e softirq -e workqueue

# Record with function graph for specific function:
trace-cmd record -p function_graph -g handle_irq_event

# View recording:
trace-cmd report | head -50

# Record IRQ latency:
trace-cmd record -p irqsoff

# Filter by CPU:
trace-cmd record -e irq -C 0

# Save to file for later analysis:
trace-cmd record -e irq -o irq_trace.dat
trace-cmd report -i irq_trace.dat
```

### KernelShark (GUI)

```bash
# Visual trace analysis:
kernelshark irq_trace.dat
# Shows timeline of IRQs, scheduling, wakeups
```

---

## 27.6 perf for Interrupt Analysis

```bash
# Count interrupts per second:
perf stat -e irq:irq_handler_entry -a sleep 5

# Record IRQ events with callchain:
perf record -e irq:irq_handler_entry -ag sleep 10
perf report

# Lock contention analysis:
perf lock record -- sleep 10
perf lock report

# Lock contention with callchains:
perf lock contention -a -- sleep 5

# Hardware performance counters:
perf stat -e cache-misses,cache-references,instructions sleep 10
```

---

## 27.7 eBPF and bpftrace

### bpftrace One-Liners

```bash
# Count IRQs by handler name:
bpftrace -e 'tracepoint:irq:irq_handler_entry { @[args->name] = count(); }'

# Measure IRQ handler duration:
bpftrace -e '
tracepoint:irq:irq_handler_entry { @start[tid] = nsecs; }
tracepoint:irq:irq_handler_exit /@start[tid]/ {
    @latency_us = hist((nsecs - @start[tid]) / 1000);
    delete(@start[tid]);
}'

# Softirq latency histogram:
bpftrace -e '
tracepoint:irq:softirq_entry { @start[args->vec] = nsecs; }
tracepoint:irq:softirq_exit /@start[args->vec]/ {
    @us[args->vec] = hist((nsecs - @start[args->vec]) / 1000);
    delete(@start[args->vec]);
}'

# Trace spin_lock contention:
bpftrace -e '
kprobe:_raw_spin_lock { @start[tid] = nsecs; }
kretprobe:_raw_spin_lock /@start[tid]/ {
    @lock_hold_ns = hist(nsecs - @start[tid]);
    delete(@start[tid]);
}'

# Count hardirq/softirq time per CPU:
bpftrace -e '
tracepoint:irq:irq_handler_entry { @irq_start[cpu] = nsecs; }
tracepoint:irq:irq_handler_exit /@irq_start[cpu]/ {
    @irq_time_us[cpu] = sum((nsecs - @irq_start[cpu]) / 1000);
    delete(@irq_start[cpu]);
}'
```

---

## 27.8 Tracing Lock Contention

### Lock Tracepoints

```bash
# Available lock events:
ls /sys/kernel/debug/tracing/events/lock/
# lock_acquire, lock_release, lock_contended, lock_acquired

# Enable lock contention tracing:
echo 1 > /sys/kernel/debug/tracing/events/lock/lock_contended/enable
echo 1 > /sys/kernel/debug/tracing/events/lock/lock_acquired/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

### Mutex/Spinlock Contention

```bash
# With perf:
perf lock record -a -- sleep 10
perf lock report --sort acquired,contended,avg_wait

# Output:
#                   Name   acquired  contended  avg wait (ns)
#  &dev->lock          12345       234          1234
#  &rq->lock           98765      1234          567
```

---

## 27.9 Cyclictest for RT Latency

```bash
# Measure worst-case interrupt + scheduling latency:
cyclictest -p 80 -t 4 -n -m -l 100000

# Output:
# T: 0 ( 1234) P:80 I:1000 C: 100000 Min:      2 Act:    5 Avg:    4 Max:   45
# T: 1 ( 1235) P:80 I:1500 C: 100000 Min:      2 Act:    6 Avg:    5 Max:   52

# With system stress:
stress-ng --cpu 4 --io 2 &
cyclictest -p 99 -t 4 -n -m -l 1000000 -q
```

```
Fields:
  T: thread  P: priority  I: interval (us)  C: count
  Min: minimum latency (us)
  Avg: average latency (us)
  Max: maximum latency (us)

Targets:
  Desktop Linux:  Max < 500us typical
  PREEMPT_RT:     Max < 50us expected
  Hard RT:        Max < 10us required
```

---

## 27.10 Quick Reference: Tracing Commands

```
Task                              │ Command
──────────────────────────────────┼─────────────────────────────────
See all IRQ counts                │ cat /proc/interrupts
See softirq counts                │ cat /proc/softirqs
Trace IRQ events                  │ trace-cmd record -e irq
Trace IRQ handler duration        │ bpftrace (handler entry/exit)
Find longest IRQ-disabled section │ echo irqsoff > current_tracer
Find wakeup latency               │ echo wakeup_rt > current_tracer
Function graph of IRQ handling    │ function_graph + set_graph_function
Lock contention                   │ perf lock record
Measure RT latency                │ cyclictest -p 80 -t 4 -n -m
Trace specific driver IRQ         │ echo "module mydrv +p" > dynamic_debug
Visual analysis                   │ trace-cmd record; kernelshark
```

---

## Kernel Source References

```
Ftrace:
  kernel/trace/                         ← Ftrace infrastructure
  kernel/trace/trace_irqsoff.c          ← irqsoff tracer
  kernel/trace/trace_sched_wakeup.c     ← wakeup tracer
  include/trace/events/irq.h            ← IRQ tracepoints
  include/trace/events/lock.h           ← Lock tracepoints

Tools:
  tools/perf/                           ← perf source
  tools/tracing/                        ← Trace tools
```

---

## Interview Questions

1. **How do you use ftrace to measure IRQ latency?**
2. **What tracepoints are available for interrupt analysis?**
3. **Explain the irqsoff tracer. What does it measure?**
4. **How do you trace softirq execution time?**
5. **What is cyclictest? How do you interpret its output?**
6. **How does perf lock help analyze lock contention?**
7. **Write a bpftrace script to histogram IRQ handler duration.**
8. **What does the function_graph tracer show? How do you filter it?**
9. **How do you trace a specific driver's interrupts?**
10. **What is KernelShark? When would you use it?**

---

## Summary

- ftrace: kernel's built-in tracer — irqsoff, preemptoff, function_graph
- /proc/interrupts + /proc/softirqs: first-line IRQ monitoring
- trace-cmd: user-space tool for easy ftrace management
- perf: hardware counters, lock contention, IRQ event counting
- bpftrace: programmable tracing for custom interrupt analysis
- cyclictest: RT latency measurement (scheduling + interrupt)
- Tracepoints: kernel events for IRQs, softirqs, locks, workqueues
- KernelShark: visual timeline analysis of trace data

---

*Next: [Chapter 28 — Kernel Source Code for Interrupts](Chapter_28_Kernel_Source.md)*
