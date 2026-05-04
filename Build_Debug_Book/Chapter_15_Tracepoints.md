# Chapter 15: Tracepoints and Events

## Learning Goals
- Understand static tracepoints and TRACE_EVENT macro
- Learn event format, filters, and triggers
- Master histogram tracing (hist triggers)
- Know the tracepoint infrastructure in kernel code

---

## 1. Static Tracepoints

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Tracepoints: predefined instrumentation points         │
  │  in kernel code. Static, stable API.                    │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ How they work:                           │            │
  │  │                                          │            │
  │  │ 1. Developer places trace_*() call       │            │
  │  │    in kernel code:                       │            │
  │  │    trace_sched_switch(prev, next);       │            │
  │  │                                          │            │
  │  │ 2. When disabled: single branch check    │            │
  │  │    (static key — NOP or always-not-taken)│            │
  │  │                                          │            │
  │  │ 3. When enabled: calls registered        │            │
  │  │    probe functions (tracers, perf, eBPF) │            │
  │  │                                          │            │
  │  │ Cost when disabled: ~0 (static key)      │            │
  │  │ Cost when enabled: function call per     │            │
  │  │   registered callback                    │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  TRACE_EVENT macro (defining a tracepoint):              │
  │  ┌──────────────────────────────────────────┐            │
  │  │ TRACE_EVENT(sched_switch,                │            │
  │  │   TP_PROTO(                              │            │
  │  │     bool preempt,                        │            │
  │  │     struct task_struct *prev,             │            │
  │  │     struct task_struct *next),            │            │
  │  │   TP_ARGS(preempt, prev, next),          │            │
  │  │   TP_STRUCT__entry(                      │            │
  │  │     __array(char, prev_comm, TASK_COMM_LEN)│          │
  │  │     __field(pid_t, prev_pid)             │            │
  │  │     __field(int, prev_prio)              │            │
  │  │     __field(long, prev_state)            │            │
  │  │     __array(char, next_comm, TASK_COMM_LEN)│          │
  │  │     __field(pid_t, next_pid)             │            │
  │  │     __field(int, next_prio)),            │            │
  │  │   TP_fast_assign(                        │            │
  │  │     memcpy(__entry->prev_comm, prev->comm,│           │
  │  │            TASK_COMM_LEN);               │            │
  │  │     __entry->prev_pid = prev->pid;       │            │
  │  │     __entry->prev_prio = prev->prio;     │            │
  │  │     __entry->prev_state = prev->state;   │            │
  │  │     memcpy(__entry->next_comm, next->comm,│           │
  │  │            TASK_COMM_LEN);               │            │
  │  │     __entry->next_pid = next->pid;       │            │
  │  │     __entry->next_prio = next->prio;),   │            │
  │  │   TP_printk("prev=(%s:%d:%d:%s) ==> "   │            │
  │  │     "next=(%s:%d:%d)",                   │            │
  │  │     __entry->prev_comm, __entry->prev_pid,│           │
  │  │     __entry->prev_prio,                  │            │
  │  │     __entry->prev_state ?                │            │
  │  │       __print_flags(prev_state, "|", ...) │           │
  │  │       : "R",                             │            │
  │  │     __entry->next_comm, __entry->next_pid,│           │
  │  │     __entry->next_prio)                  │            │
  │  │ );                                       │            │
  │  │                                          │            │
  │  │ This single macro generates:             │            │
  │  │ - trace_sched_switch() function          │            │
  │  │ - Register/unregister hooks              │            │
  │  │ - Binary format for ring buffer          │            │
  │  │ - Event format file in tracefs           │            │
  │  │ - perf integration                       │            │
  │  │ - eBPF attachment point                  │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Event Tracing via tracefs

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  /sys/kernel/tracing/events/                             │
  │  ├── sched/                                              │
  │  │   ├── sched_switch/                                   │
  │  │   │   ├── enable        — 0/1                        │
  │  │   │   ├── format        — event field layout         │
  │  │   │   ├── filter        — per-event filter           │
  │  │   │   ├── trigger       — action on event fire       │
  │  │   │   └── id            — numeric event ID           │
  │  │   ├── sched_wakeup/                                  │
  │  │   └── enable            — enable all sched events    │
  │  ├── irq/                                               │
  │  ├── block/                                              │
  │  └── enable                — enable ALL events          │
  │                                                           │
  │  Basic event tracing:                                    │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Enable one event:                     │            │
  │  │ echo 1 > events/sched/sched_switch/enable│            │
  │  │                                          │            │
  │  │ # Enable entire subsystem:              │            │
  │  │ echo 1 > events/sched/enable             │            │
  │  │                                          │            │
  │  │ # Read results:                         │            │
  │  │ cat trace_pipe                            │            │
  │  │                                          │            │
  │  │ # View event format:                    │            │
  │  │ cat events/sched/sched_switch/format      │            │
  │  │ → shows all fields with their types     │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Event filters:                                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Only trace events matching condition: │            │
  │  │ echo 'next_pid == 1234' > \              │            │
  │  │   events/sched/sched_switch/filter       │            │
  │  │                                          │            │
  │  │ echo 'prev_prio < 100' > \               │            │
  │  │   events/sched/sched_switch/filter       │            │
  │  │                                          │            │
  │  │ echo 'next_comm ~ "kworker*"' > \        │            │
  │  │   events/sched/sched_switch/filter       │            │
  │  │                                          │            │
  │  │ # Multiple conditions:                  │            │
  │  │ echo 'next_pid > 0 && prev_state == 1' > \│           │
  │  │   events/sched/sched_switch/filter       │            │
  │  │                                          │            │
  │  │ # Clear filter:                         │            │
  │  │ echo 0 > events/sched/sched_switch/filter│            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Event triggers:                                         │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Stacktrace on event:                  │            │
  │  │ echo stacktrace > \                      │            │
  │  │   events/sched/sched_switch/trigger      │            │
  │  │                                          │            │
  │  │ # Enable another event on trigger:      │            │
  │  │ echo 'enable_event:irq:irq_handler_entry' > \│       │
  │  │   events/sched/sched_switch/trigger      │            │
  │  │                                          │            │
  │  │ # Snapshot on trigger:                  │            │
  │  │ echo snapshot > \                        │            │
  │  │   events/sched/sched_switch/trigger      │            │
  │  │                                          │            │
  │  │ # Traceoff (stop tracing) on event:     │            │
  │  │ echo traceoff > \                        │            │
  │  │   events/kmem/oom_score_adj_update/trigger│           │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Histogram triggers:                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Count context switches per task:      │            │
  │  │ echo 'hist:key=next_comm:val=hitcount:  │            │
  │  │   sort=hitcount.descending' > \          │            │
  │  │   events/sched/sched_switch/trigger      │            │
  │  │                                          │            │
  │  │ cat events/sched/sched_switch/hist       │            │
  │  │ → { next_comm: bash   } hitcount: 1234  │            │
  │  │ → { next_comm: chrome } hitcount:  567  │            │
  │  │                                          │            │
  │  │ # Latency histogram (wakeup → run):     │            │
  │  │ echo 'hist:key=pid:ts0=common_timestamp │            │
  │  │   .usecs' > events/sched/sched_wakeup/   │            │
  │  │   trigger                                │            │
  │  │ echo 'hist:key=prev_pid:                │            │
  │  │   val=$ts0-common_timestamp.usecs:       │            │
  │  │   sort=val' > events/sched/sched_switch/ │            │
  │  │   trigger                                │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: What is the TRACE_EVENT macro and why is it important?**
**A:** TRACE_EVENT is a single macro that defines a complete tracepoint — its function prototype, data fields, assignment logic, and human-readable output format. From one TRACE_EVENT definition, the preprocessor generates: (1) The `trace_<name>()` inline function that kernel code calls — includes a static key check (zero-cost when disabled). (2) The `struct trace_event_raw_<name>` — the binary data structure stored in the ring buffer. (3) The `events/<subsystem>/<name>/format` file in tracefs — tells tools like `perf` and `trace-cmd` how to parse the binary data. (4) Registration functions for attaching callbacks (ftrace, perf, eBPF). (5) The `TP_printk` function for human-readable output in `trace` file. This is important because it provides a stable ABI: the format file describes the binary layout, so tools can parse events without kernel headers. It also ensures consistency — a single definition keeps the binary format, human output, and registration all in sync. Tracepoints defined with TRACE_EVENT are used by ftrace events, perf events, and eBPF programs, making them the universal kernel instrumentation point. There are thousands of tracepoints in the kernel (>2000 in a typical config), covering scheduling, I/O, networking, memory, and every major subsystem.

**Q2: How would you trace scheduler latency using ftrace events?**
**A:** To measure wakeup-to-schedule latency (time between a task being woken up and actually running): (1) **Using the built-in wakeup tracer**: `echo wakeup > current_tracer` — traces the highest-priority task's wakeup latency. Or `echo wakeup_rt > current_tracer` for RT tasks only. The tracer records the maximum latency seen and stores the trace that caused it. (2) **Using events + hist triggers** (more flexible): Enable `sched_wakeup` and `sched_switch` events. Create a synthetic event from the two: `echo 'hist:keys=pid:ts0=common_timestamp.usecs' > events/sched/sched_wakeup/trigger` (saves wakeup timestamp keyed by PID). `echo 'hist:keys=next_pid:wakeup_lat=common_timestamp.usecs-$ts0:sort=wakeup_lat' > events/sched/sched_switch/trigger` (computes delta at switch time). Read `events/sched/sched_switch/hist` for per-task latency distribution. (3) **Using trace-cmd**: `trace-cmd record -e sched_wakeup -e sched_switch -F ./myapp` → `trace-cmd report` → post-process timestamps.

---

## Summary

- Tracepoints: static instrumentation in kernel code; zero cost when disabled (static keys)
- TRACE_EVENT macro: generates trace function + binary format + format file + registration
- tracefs events: /sys/kernel/tracing/events/ — enable, format, filter, trigger per event
- Filters: field-based (==, !=, <, >, ~glob); triggers: stacktrace, snapshot, traceoff
- Histogram triggers: in-kernel aggregation — count, latency distribution without userspace
- Thousands of tracepoints across all kernel subsystems (sched, irq, block, net, kmem, ...)

---

[Previous: ftrace Architecture ←](Chapter_14_ftrace.md) | [Next: kprobes and kretprobes →](Chapter_16_kprobes.md)
