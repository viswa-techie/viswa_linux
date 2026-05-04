# Chapter 18: Perfetto and Trace Visualization

## Learning Goals
- Understand Perfetto architecture and ftrace integration
- Learn trace_processor and SQL-based analysis
- Master Chrome trace format and visualization tools
- Know Perfetto on Android (atrace, traced)

---

## 1. Perfetto Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Perfetto: production-grade tracing framework           │
  │  (Google, open source, used in Android/Chrome/Linux)    │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Architecture:                            │            │
  │  │                                          │            │
  │  │ ┌─────────────┐  ┌─────────────────────┐│            │
  │  │ │ Data Sources │  │ traced (daemon)     ││            │
  │  │ │             │──→│ Central trace      ││            │
  │  │ │ - ftrace    │  │ service:            ││            │
  │  │ │ - /proc     │  │ - shared memory     ││            │
  │  │ │ - atrace    │  │   buffers           ││            │
  │  │ │ - GPU       │  │ - page-aligned      ││            │
  │  │ │ - userspace │  │   chunks            ││            │
  │  │ │   SDK       │  │ - protobuf format   ││            │
  │  │ └─────────────┘  └────────┬────────────┘│            │
  │  │                           │              │            │
  │  │                    ┌──────▼──────┐       │            │
  │  │                    │ .perfetto-  │       │            │
  │  │                    │ trace file  │       │            │
  │  │                    └──────┬──────┘       │            │
  │  │                           │              │            │
  │  │             ┌─────────────┼──────────┐   │            │
  │  │             ▼             ▼          ▼   │            │
  │  │     ┌────────────┐ ┌──────────┐ ┌─────┐│            │
  │  │     │ Perfetto UI│ │trace_    │ │SQL  ││            │
  │  │     │(web viewer)│ │processor │ │query││            │
  │  │     └────────────┘ └──────────┘ └─────┘│            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Perfetto + ftrace:                                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Perfetto reads ftrace events via         │            │
  │  │ /sys/kernel/tracing/trace_pipe_raw       │            │
  │  │ (binary format, much faster than trace_pipe)│        │
  │  │                                          │            │
  │  │ Configuration (protobuf text format):    │            │
  │  │ data_sources {                           │            │
  │  │   config {                               │            │
  │  │     name: "linux.ftrace"                 │            │
  │  │     ftrace_config {                      │            │
  │  │       ftrace_events: "sched/sched_switch"│            │
  │  │       ftrace_events: "sched/sched_wakeup"│            │
  │  │       ftrace_events: "power/cpu_frequency"│           │
  │  │       ftrace_events: "irq/irq_handler_*" │            │
  │  │       buffer_size_kb: 16384              │            │
  │  │     }                                    │            │
  │  │   }                                      │            │
  │  │ }                                        │            │
  │  │ duration_ms: 10000                       │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. trace_processor and SQL

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  trace_processor: SQLite-based trace analysis engine     │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Load trace:                            │            │
  │  │ trace_processor --interactive trace.perfetto│         │
  │  │                                          │            │
  │  │ # SQL query: scheduling latency          │            │
  │  │ SELECT ts, dur, thread.name, cpu         │            │
  │  │ FROM sched_slice                         │            │
  │  │ JOIN thread USING (utid)                 │            │
  │  │ WHERE thread.name = 'RenderThread'       │            │
  │  │ ORDER BY dur DESC LIMIT 10;              │            │
  │  │                                          │            │
  │  │ # CPU frequency changes:                │            │
  │  │ SELECT ts, cpu, value                    │            │
  │  │ FROM counter                             │            │
  │  │ JOIN cpu_counter_track ON                │            │
  │  │   counter.track_id = cpu_counter_track.id│            │
  │  │ WHERE cpu_counter_track.name =           │            │
  │  │   'cpufreq';                             │            │
  │  │                                          │            │
  │  │ # Wakeup chain analysis:                │            │
  │  │ SELECT                                   │            │
  │  │   waker.name AS waker,                   │            │
  │  │   wakee.name AS wakee,                   │            │
  │  │   COUNT(*) AS count                      │            │
  │  │ FROM sched_waking                        │            │
  │  │ JOIN thread AS waker ON waker_utid = waker.utid│     │
  │  │ JOIN thread AS wakee ON utid = wakee.utid│            │
  │  │ GROUP BY waker, wakee                    │            │
  │  │ ORDER BY count DESC;                     │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Android-specific usage:                                 │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Record on Android device:             │            │
  │  │ adb shell perfetto \                     │            │
  │  │   -c /data/misc/perfetto-configs/cfg.txt \│           │
  │  │   -o /data/misc/perfetto-traces/trace    │            │
  │  │                                          │            │
  │  │ # Or use record_android_trace script:   │            │
  │  │ tools/record_android_trace -t 10s \      │            │
  │  │   -b 32mb sched freq idle               │            │
  │  │                                          │            │
  │  │ # Categories (atrace):                  │            │
  │  │ sched, freq, idle, am, wm, gfx, view,   │            │
  │  │ hal, input, binder_driver, ...           │            │
  │  │                                          │            │
  │  │ # View: open ui.perfetto.dev             │            │
  │  │ # Drag and drop trace file               │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Visualization Tools

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌───────────────┬──────────────────────────────────────┐│
  │  │ Tool          │ Best For                             ││
  │  ├───────────────┼──────────────────────────────────────┤│
  │  │ Perfetto UI   │ Android traces, ftrace events,      ││
  │  │ (web-based)   │ SQL queries, timeline view           ││
  │  ├───────────────┼──────────────────────────────────────┤│
  │  │ KernelShark   │ ftrace/trace-cmd data, scheduler    ││
  │  │               │ analysis, CPU timeline               ││
  │  ├───────────────┼──────────────────────────────────────┤│
  │  │ Flame Graphs  │ perf profiles, CPU hotspot analysis  ││
  │  │ (Brendan Gregg)│ differential comparison             ││
  │  ├───────────────┼──────────────────────────────────────┤│
  │  │ perf report   │ Quick interactive profiling, TUI     ││
  │  │ TUI           │ source annotation                    ││
  │  ├───────────────┼──────────────────────────────────────┤│
  │  │ Trace Compass │ CTF traces, LTTng, complex state    ││
  │  │ (Eclipse)     │ machine analysis                     ││
  │  └───────────────┴──────────────────────────────────────┘│
  │                                                           │
  │  Flame graph generation:                                 │
  │  ┌──────────────────────────────────────────┐            │
  │  │ perf record -g -a sleep 30               │            │
  │  │ perf script > out.perf                    │            │
  │  │ stackcollapse-perf.pl out.perf > folded   │            │
  │  │ flamegraph.pl folded > flame.svg          │            │
  │  │                                          │            │
  │  │ Differential flame graph:                │            │
  │  │ difffolded.pl folded1 folded2 | \         │            │
  │  │   flamegraph.pl > diff.svg               │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does Perfetto differ from ftrace and when would you use each?**
**A:** **ftrace** is the kernel's built-in tracing framework — it provides the data sources (function tracing, events, kprobes) and the tracefs interface. It's always available, requires no daemon, and is excellent for quick kernel debugging. **Perfetto** is a user-space tracing service that reads data from ftrace (via `trace_pipe_raw`) and other sources, stores it in an efficient protobuf format with compression, and provides powerful analysis tools (SQL queries via trace_processor, timeline UI). Key differences: (1) **Data format**: ftrace uses plain text or binary per-CPU ring buffers; Perfetto uses protobuf with optimized encoding (10-50x smaller files). (2) **Multi-source**: Perfetto correlates kernel events (ftrace), userspace events (atrace/SDK), GPU events, memory counters, and process stats in a single timeline. ftrace is kernel-only. (3) **Analysis**: ftrace analysis is grep/awk on text; Perfetto provides SQL — `SELECT dur, name FROM slice WHERE dur > 16000000` (find frames > 16ms). (4) **Overhead**: ftrace is lower overhead (no daemon, no protobuf serialization). Perfetto adds a traced daemon but uses shared memory and batching to minimize impact. **Use ftrace** for: quick kernel debugging, function tracing on embedded systems without Perfetto, development-time investigation. **Use Perfetto** for: production tracing, Android system tracing (it's the standard tool), cross-layer analysis (kernel + framework + app), and when you need SQL queries or the timeline UI.

**Q2: What is the Chrome Trace Format and how do different tools interoperate?**
**A:** The Chrome Trace Format (also called Trace Event Format, or catapult format) is a JSON-based trace format originally from Chrome's `chrome://tracing`. It represents events as JSON objects with fields: `name` (event name), `cat` (category), `ph` (phase: B=begin, E=end, X=complete, i=instant), `ts` (timestamp in microseconds), `pid`, `tid`, `args` (metadata). This became a de facto interchange format: systrace (Android's older tool) outputs it, Perfetto can export it, and any tool can generate it for visualization in Perfetto UI or chrome://tracing. Perfetto's native format is protobuf (`.perfetto-trace` or `.proto`), which is much more compact. Perfetto UI reads both formats. The interop ecosystem: ftrace raw → trace-cmd → KernelShark (native viewer) or → Perfetto (via trace_to_text converter). perf.data → perf script → Flame graphs. ftrace raw → Perfetto (direct read) → .perfetto-trace → trace_processor SQL or Perfetto UI. The key insight is that ftrace events are the foundational data source — Perfetto, trace-cmd, perf, and eBPF all consume them through different interfaces.

---

## Summary

- Perfetto: production tracing service (daemon, shared memory, protobuf format)
- Reads ftrace via trace_pipe_raw + atrace + userspace SDK + GPU/memory counters
- trace_processor: SQLite-based analysis engine — SQL queries on trace data
- Perfetto UI (ui.perfetto.dev): web-based timeline visualization
- Android: `perfetto` CLI or record_android_trace script; atrace categories
- Flame graphs: perf script → stackcollapse-perf.pl → flamegraph.pl
- KernelShark: GUI for trace-cmd/ftrace data; Trace Compass: Eclipse-based for LTTng

---

[Previous: perf Events and PMU ←](Chapter_17_perf.md) | [Next: eBPF Architecture →](Chapter_19_eBPF.md)
