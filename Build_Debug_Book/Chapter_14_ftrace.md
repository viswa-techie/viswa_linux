# Chapter 14: ftrace Architecture

## Learning Goals
- Understand ftrace function tracing mechanisms (mcount/fentry)
- Learn tracefs interface and trace-cmd usage
- Master function_graph tracer for call flow visualization
- Know ftrace trampoline and dynamic patching

---

## 1. ftrace Overview

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ftrace = Function Tracer (but does much more)          │
  │  The kernel's built-in tracing framework                │
  │                                                           │
  │  Key concepts:                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. Every kernel function has a hook      │            │
  │  │    point at entry (compiled with -pg     │            │
  │  │    or -mfentry)                          │            │
  │  │                                          │            │
  │  │ 2. By default: hooks are NOPs            │            │
  │  │    (almost zero overhead)                │            │
  │  │                                          │            │
  │  │ 3. When tracing enabled: NOPs replaced   │            │
  │  │    with calls to tracer functions        │            │
  │  │    (dynamic code patching!)              │            │
  │  │                                          │            │
  │  │ 4. Trace data written to per-CPU ring    │            │
  │  │    buffers                               │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Function entry instrumentation:                         │
  │  ┌──────────────────────────────────────────┐            │
  │  │ GCC -pg: inserts call to mcount          │            │
  │  │   at function PROLOGUE                   │            │
  │  │                                          │            │
  │  │ GCC -mfentry: inserts call to __fentry__ │            │
  │  │   BEFORE function prologue               │            │
  │  │   (x86, better — frame pointer intact)   │            │
  │  │                                          │            │
  │  │ At boot: all mcount/__fentry__ calls     │            │
  │  │ patched to NOP (5-byte NOP on x86)       │            │
  │  │ List of patchable sites stored in        │            │
  │  │ __mcount_loc section (thousands entries) │            │
  │  │                                          │            │
  │  │ When tracer activates:                   │            │
  │  │ NOP patched back to CALL <trampoline>    │            │
  │  │ (text_poke / aarch64_insn_patch)        │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. tracefs Interface

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  /sys/kernel/tracing/ (or /sys/kernel/debug/tracing/)   │
  │                                                           │
  │  Key files:                                              │
  │  ┌──────────────────────────────────────────┐            │
  │  │ current_tracer     — select tracer       │            │
  │  │ trace              — human-readable output│           │
  │  │ trace_pipe         — streaming output     │            │
  │  │ tracing_on         — 0/1 enable/disable  │            │
  │  │ available_tracers  — list tracers        │            │
  │  │ set_ftrace_filter  — trace only these fns│            │
  │  │ set_ftrace_notrace — exclude these fns   │            │
  │  │ set_ftrace_pid     — trace only this PID │            │
  │  │ buffer_size_kb     — per-CPU buffer size │            │
  │  │ available_filter_functions — all hookable│            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Basic function tracing:                                 │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Set tracer:                            │            │
  │  │ echo function > current_tracer           │            │
  │  │                                          │            │
  │  │ # Filter to specific functions:          │            │
  │  │ echo 'schedule*' > set_ftrace_filter     │            │
  │  │                                          │            │
  │  │ # Enable and read:                       │            │
  │  │ echo 1 > tracing_on                      │            │
  │  │ cat trace_pipe                            │            │
  │  │                                          │            │
  │  │ Output:                                  │            │
  │  │ # tracer: function                       │            │
  │  │ #  TASK-PID  CPU#  TIMESTAMP  FUNCTION   │            │
  │  │ bash-1234  [001]  123.456:  schedule     │            │
  │  │ kworker-56 [003]  123.457:  schedule     │            │
  │  │                                          │            │
  │  │ # Disable:                               │            │
  │  │ echo nop > current_tracer                │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  function_graph tracer:                                  │
  │  ┌──────────────────────────────────────────┐            │
  │  │ echo function_graph > current_tracer     │            │
  │  │                                          │            │
  │  │ Output (shows call depth + return time): │            │
  │  │  1)               | do_sys_openat2() {   │            │
  │  │  1)               |   getname() {        │            │
  │  │  1)   0.241 us    |     kmem_cache_alloc()│           │
  │  │  1)   0.753 us    |   }                  │            │
  │  │  1)               |   do_filp_open() {   │            │
  │  │  1)               |     path_openat() {  │            │
  │  │  1)   0.152 us    |       alloc_empty_file()│        │
  │  │  1) + 12.345 us   |     }                │            │
  │  │  1) + 13.078 us   |   }                  │            │
  │  │  1) + 14.234 us   | }                    │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. trace-cmd Tool

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  trace-cmd: userspace tool for ftrace                   │
  │  (easier than writing to tracefs directly)              │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Record function calls:                │            │
  │  │ trace-cmd record -p function_graph \     │            │
  │  │   -g do_sys_openat2 -F ./myapp          │            │
  │  │                                          │            │
  │  │ # Record with events:                   │            │
  │  │ trace-cmd record -e sched -e irq \       │            │
  │  │   -F ./myapp                            │            │
  │  │                                          │            │
  │  │ # Report:                               │            │
  │  │ trace-cmd report                         │            │
  │  │ trace-cmd report --cpu 0                 │            │
  │  │                                          │            │
  │  │ # List available events:                │            │
  │  │ trace-cmd list -e                        │            │
  │  │                                          │            │
  │  │ # Function profiling:                   │            │
  │  │ trace-cmd record -p function_graph \     │            │
  │  │   --max-graph-depth 3 sleep 10           │            │
  │  │ trace-cmd report --profile              │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does ftrace achieve near-zero overhead when tracing is disabled?**
**A:** The kernel is compiled with `-pg` (GCC) or `-mfentry` (GCC 4.6+/Clang), which inserts a call instruction (to `mcount` or `__fentry__`) at the beginning of every function. During kernel boot, the `ftrace_init()` function walks the `__mcount_loc` section — a table of addresses of all these call instructions — and patches each `CALL mcount` to a 5-byte NOP (on x86: `0F 1F 44 00 00`). On ARM64, it's patched to a NOP instruction. With NOPs in place, there is literally zero overhead — the CPU executes the NOP (1 cycle) and continues. When tracing is enabled for specific functions, only those functions' NOPs are patched back to `CALL <ftrace_trampoline>`. The trampoline saves registers, calls the tracer callback (which records to the per-CPU ring buffer), restores registers, and returns. The patching uses architecture-specific safe code modification: x86 uses `text_poke_bp()` (breakpoint-based safe patching for SMP), ARM64 uses `aarch64_insn_patch_text()`. This dynamic binary patching approach is what makes ftrace unique — it's compiled into every function but costs nothing until activated.

**Q2: What is the function_graph tracer and how does it capture both entry and exit?**
**A:** The function_graph tracer records function entry AND exit, showing call depth and execution duration. Entry is captured the same way as the regular function tracer (NOP → CALL trampoline). Exit is captured using a return address hijack: when the trampoline runs at function entry, it saves the original return address (from the stack) and replaces it with the address of a special `return_to_handler` trampoline. When the function returns, it returns to `return_to_handler` instead of the real caller. `return_to_handler` records the function exit event (with timestamp), retrieves the saved real return address from a per-task shadow stack, and jumps there. This gives the tracer both entry and exit timestamps, enabling duration calculation and nested call visualization (the indented output showing `{ }` blocks). The per-task return address stack has a fixed depth (typically 50), limiting the maximum nesting depth tracked. If exceeded, tracing for that call chain is lost. The overhead is higher than the basic function tracer (two trampoline calls per function + return address manipulation), but still manageable for targeted tracing with `set_graph_function` to limit which top-level functions are traced.

---

## Summary

- ftrace: built-in kernel tracer; every function instrumented with NOP (zero overhead)
- Dynamic patching: NOPs → CALL trampoline when tracing enabled
- tracefs: /sys/kernel/tracing/ — current_tracer, trace, set_ftrace_filter
- function tracer: records function entries; function_graph: entries + exits + duration
- trace-cmd: userspace tool — record, report, profile; easier than raw tracefs
- function_graph uses return address hijacking for exit capture
- Per-CPU ring buffers: lockless, minimal contention

---

[Previous: dmesg and Logging ←](Chapter_13_dmesg_Logging.md) | [Next: Tracepoints and Events →](Chapter_15_Tracepoints.md)
