# Chapter 17: perf Events and PMU

## Learning Goals
- Understand hardware performance counters and PMU architecture
- Learn perf stat, record, report workflows
- Master sampling vs counting modes
- Know software events, tracepoint events, and perf scripting

---

## 1. PMU and Hardware Counters

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  PMU = Performance Monitoring Unit                       │
  │  CPU hardware that counts/samples performance events     │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Hardware counters (per CPU core):        │            │
  │  │                                          │            │
  │  │ ┌──────────────┐  ┌──────────────┐      │            │
  │  │ │ Counter 0    │  │ Counter 1    │      │            │
  │  │ │ cycles       │  │ instructions │      │            │
  │  │ │ 1,234,567,890│  │ 987,654,321  │      │            │
  │  │ └──────────────┘  └──────────────┘      │            │
  │  │ ┌──────────────┐  ┌──────────────┐      │            │
  │  │ │ Counter 2    │  │ Counter 3    │      │            │
  │  │ │ cache-misses │  │ branches     │      │            │
  │  │ │    12,345    │  │ 456,789,012  │      │            │
  │  │ └──────────────┘  └──────────────┘      │            │
  │  │                                          │            │
  │  │ Typical: 4-8 general-purpose counters   │            │
  │  │ + fixed counters (cycles, instructions, │            │
  │  │   ref-cycles on Intel)                  │            │
  │  │                                          │            │
  │  │ PMU generates NMI on counter overflow   │            │
  │  │ → captures instruction pointer (IP)     │            │
  │  │ → that's how sampling works             │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Event types:                                            │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Hardware events:                         │            │
  │  │   cycles, instructions, cache-references,│            │
  │  │   cache-misses, branch-misses,           │            │
  │  │   bus-cycles, stalled-cycles-frontend    │            │
  │  │                                          │            │
  │  │ Software events (kernel counters):       │            │
  │  │   context-switches, cpu-migrations,      │            │
  │  │   page-faults, task-clock, cpu-clock     │            │
  │  │                                          │            │
  │  │ Tracepoint events:                       │            │
  │  │   sched:sched_switch, block:block_rq_*,  │            │
  │  │   syscalls:sys_enter_open, ...           │            │
  │  │                                          │            │
  │  │ Raw PMU events:                          │            │
  │  │   r<hex-event-code> (architecture-specific│           │
  │  │   from CPU manuals)                      │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. perf Tool Usage

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  perf stat — Counting mode (aggregate):                  │
  │  ┌──────────────────────────────────────────┐            │
  │  │ perf stat ./myprogram                    │            │
  │  │                                          │            │
  │  │   1,234,567,890  cycles                  │            │
  │  │     987,654,321  instructions  # 0.80 IPC│            │
  │  │      12,345,678  cache-references        │            │
  │  │       1,234,567  cache-misses   # 10%   │            │
  │  │     234,567,890  branches                │            │
  │  │       5,678,901  branch-misses  # 2.4%  │            │
  │  │                                          │            │
  │  │ perf stat -e cycles,L1-dcache-load-misses \│         │
  │  │   -p <pid> sleep 5                       │            │
  │  │                                          │            │
  │  │ perf stat -e 'syscalls:sys_enter_*' ls   │            │
  │  │ → count system calls by type             │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  perf record — Sampling mode (per-sample data):          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ perf record -g ./myprogram               │            │
  │  │ # -g = record call graphs                │            │
  │  │ # creates perf.data file                 │            │
  │  │                                          │            │
  │  │ perf record -e cache-misses \             │            │
  │  │   -c 10000 -g ./myprogram               │            │
  │  │ # -c 10000 = sample every 10000 events  │            │
  │  │                                          │            │
  │  │ perf record -a -g sleep 30               │            │
  │  │ # -a = system-wide, all CPUs            │            │
  │  │                                          │            │
  │  │ perf record -e sched:sched_switch -a     │            │
  │  │ # trace event sampling                  │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  perf report — Analysis:                                │
  │  ┌──────────────────────────────────────────┐            │
  │  │ perf report                              │            │
  │  │ # Interactive TUI:                       │            │
  │  │ Overhead  Command  Shared Object  Symbol │            │
  │  │  35.21%   myapp    myapp          [.] hot_function│   │
  │  │  12.34%   myapp    libc.so        [.] malloc│       │
  │  │   8.76%   myapp    [kernel]       [k] copy_user│    │
  │  │                                          │            │
  │  │ perf report --stdio # text output        │            │
  │  │ perf report -n      # show sample count  │            │
  │  │ perf report --sort comm,dso              │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Other perf subcommands:                                 │
  │  ┌──────────────────────────────────────────┐            │
  │  │ perf top        — live system profiling  │            │
  │  │ perf list       — list available events  │            │
  │  │ perf annotate   — source-level samples   │            │
  │  │ perf diff       — compare two profiles   │            │
  │  │ perf sched      — scheduler analysis     │            │
  │  │ perf mem        — memory access profiling│            │
  │  │ perf c2c        — cache-to-cache (false  │            │
  │  │                  sharing detection)       │            │
  │  │ perf lock       — lock contention        │            │
  │  │ perf script     — raw event dump for     │            │
  │  │                  post-processing          │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Sampling vs Counting

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌───────────────┬──────────────────────────────────────┐│
  │  │ Counting      │ Sampling                            ││
  │  │ (perf stat)   │ (perf record)                       ││
  │  ├───────────────┼──────────────────────────────────────┤│
  │  │ Counts events │ Records data at regular intervals   ││
  │  │ for duration  │ (every N events)                    ││
  │  │               │                                      ││
  │  │ Output: total │ Output: per-sample data with IP,    ││
  │  │ counts +      │ call chain, timestamp, PID, CPU     ││
  │  │ percentages   │                                      ││
  │  │               │ PMU overflow triggers NMI →          ││
  │  │ Very low      │ kernel records IP + metadata        ││
  │  │ overhead      │                                      ││
  │  │               │ Higher overhead (NMI per sample)    ││
  │  │               │                                      ││
  │  │ Use for:      │ Use for:                            ││
  │  │ Quick check   │ Finding hot functions/lines         ││
  │  │ IPC, miss     │ Flame graphs                        ││
  │  │ rates         │ Call chain analysis                  ││
  │  └───────────────┴──────────────────────────────────────┘│
  │                                                           │
  │  Call graph modes:                                       │
  │  ┌──────────────────────────────────────────┐            │
  │  │ perf record --call-graph fp     # frame pointers  │  │
  │  │   Fastest, needs -fno-omit-frame-pointer │            │
  │  │                                          │            │
  │  │ perf record --call-graph dwarf  # DWARF unwind  │   │
  │  │   Most accurate, stack copies (slower)   │            │
  │  │                                          │            │
  │  │ perf record --call-graph lbr    # Last Branch   │   │
  │  │   Record (Intel HW, no overhead, limited│            │
  │  │   depth ~32 entries)                     │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does perf sampling work at the hardware level?**
**A:** (1) The user requests sampling on an event (e.g., `perf record -e cycles -c 100000`). The kernel programs a PMU counter to count `cycles` and sets an overflow threshold of 100000. (2) The CPU core counts cycles in hardware. When the counter reaches 100000, it overflows and generates a PMI (Performance Monitoring Interrupt) — typically delivered as an NMI (Non-Maskable Interrupt) for precision. (3) The NMI handler in the kernel (`perf_event_nmi_handler`) captures the interrupted context: the instruction pointer (IP/RIP), stack pointer, process ID, CPU number, and timestamp. If call graph recording is enabled, it also captures the call chain (via frame pointers, DWARF unwinding, or LBR). (4) This sample data is written to a per-CPU ring buffer (perf mmap buffer) that the `perf` userspace tool reads. (5) The counter is reset and counting resumes. Because the counter has a fixed overflow value, higher-frequency events are sampled proportionally more often — giving a statistical profile of where CPU time is spent. The `-F <freq>` option auto-tunes the overflow value to achieve a target sample rate (e.g., 1000 Hz). Skid: on out-of-order CPUs, the IP captured may be a few instructions past the actual event — this is "sampling skid." Intel PEBS (Precise Event-Based Sampling) reduces skid to near-zero by recording the IP at the exact instruction boundary.

**Q2: How would you use perf to diagnose a performance regression?**
**A:** (1) **Quantify**: `perf stat ./workload` on both old and new versions — compare IPC (instructions per cycle), cache-miss rates, branch-miss rates. If IPC dropped significantly, it's likely a microarchitectural issue (more stalls). (2) **Profile**: `perf record -g ./workload` on both → `perf report` to identify changed hot functions. `perf diff old.data new.data` directly shows which functions gained/lost overhead. (3) **Annotate**: `perf annotate hot_function` shows per-instruction sample counts — pinpoints the exact line/instruction causing the regression. (4) **Cache analysis**: if cache-misses increased: `perf record -e cache-misses -g ./workload` → `perf report` shows which functions cause cache misses. `perf c2c record ./workload` → `perf c2c report` detects false sharing. `perf mem record ./workload` shows memory access patterns. (5) **Scheduler**: if the issue is scheduling-related: `perf sched record ./workload` → `perf sched latency` shows wakeup latencies, `perf sched map` shows CPU migration patterns. (6) **Flame graphs**: `perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg` — visual comparison of CPU profiles. The differential flame graph (comparing old vs new) immediately highlights what changed.

---

## Summary

- PMU: CPU hardware counters (4-8 general, 3 fixed on Intel) count events in hardware
- perf stat: counting mode — aggregate totals (IPC, miss rates, quick assessment)
- perf record: sampling mode — PMU overflow generates NMI, captures IP + call chain
- perf report: TUI analysis; perf annotate: source/assembly-level profiling
- Call graphs: frame pointers (fast), DWARF (accurate), LBR (Intel HW, zero-overhead)
- Event types: hardware (cycles, cache), software (context-switches), tracepoints, raw PMU

---

[Previous: kprobes ←](Chapter_16_kprobes.md) | [Next: Perfetto and Trace Visualization →](Chapter_18_Perfetto.md)
