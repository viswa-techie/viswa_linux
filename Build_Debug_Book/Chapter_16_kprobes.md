# Chapter 16: kprobes and kretprobes

## Learning Goals
- Understand kprobes dynamic instrumentation mechanism
- Learn kretprobes for function return tracing
- Master kernel breakpoint and trampoline internals
- Know ftrace-based kprobes optimization

---

## 1. kprobes Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  kprobes = Dynamic instrumentation — probe ANY           │
  │  kernel instruction at runtime (no recompile)           │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ How kprobes work:                        │            │
  │  │                                          │            │
  │  │ 1. User registers probe at address       │            │
  │  │    (e.g., entry of do_sys_open)          │            │
  │  │                                          │            │
  │  │ 2. Kernel saves original instruction     │            │
  │  │    and replaces with INT3 (breakpoint)   │            │
  │  │    on x86, or BRK on ARM64              │            │
  │  │                                          │            │
  │  │ 3. When CPU hits INT3:                   │            │
  │  │    a) Trap handler fires                 │            │
  │  │    b) pre_handler() callback runs        │            │
  │  │       (user's probe function)            │            │
  │  │    c) Original instruction executed      │            │
  │  │       via single-step or out-of-line copy│            │
  │  │    d) post_handler() callback runs       │            │
  │  │    e) Execution continues normally       │            │
  │  │                                          │            │
  │  │ 4. Probe can inspect/modify registers    │            │
  │  │    (struct pt_regs)                      │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Mechanism detail:                                       │
  │  ┌──────────────────────────────────────────┐            │
  │  │                                          │            │
  │  │ BEFORE:                                  │            │
  │  │ do_sys_open:                             │            │
  │  │   push rbp         ← original insn      │            │
  │  │   mov rbp, rsp                           │            │
  │  │   ...                                    │            │
  │  │                                          │            │
  │  │ AFTER probe registered:                  │            │
  │  │ do_sys_open:                             │            │
  │  │   INT3             ← breakpoint          │            │
  │  │   mov rbp, rsp     (rest unchanged)      │            │
  │  │   ...                                    │            │
  │  │                                          │            │
  │  │ Original insn saved in:                  │            │
  │  │   kprobe->ainsn.insn (copy buffer)       │            │
  │  │   Single-stepped out-of-line             │            │
  │  │                                          │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. kretprobes — Return Probes

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  kretprobe: probe function RETURN (capture return value) │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Mechanism (same as function_graph):      │            │
  │  │                                          │            │
  │  │ 1. kprobe placed at function entry       │            │
  │  │                                          │            │
  │  │ 2. entry handler:                        │            │
  │  │    - Save real return address from stack  │            │
  │  │    - Replace with kretprobe_trampoline   │            │
  │  │                                          │            │
  │  │ 3. Function executes normally            │            │
  │  │                                          │            │
  │  │ 4. Function returns to trampoline:       │            │
  │  │    - ret_handler() callback runs         │            │
  │  │      (access return value in regs->ax)   │            │
  │  │    - Restore real return address         │            │
  │  │    - Jump to real caller                 │            │
  │  │                                          │            │
  │  │ Pool of return instances (maxactive):    │            │
  │  │ handles recursive/concurrent calls       │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Using kprobes via tracefs (without code):               │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Register kprobe event:                │            │
  │  │ echo 'p:myprobe do_sys_openat2 \         │            │
  │  │   dfd=%di:s32 \                          │            │
  │  │   filename=+0(%si):string \              │            │
  │  │   flags=%dx:x32 \                       │            │
  │  │   mode=%cx:x32' > \                      │            │
  │  │   /sys/kernel/tracing/kprobe_events      │            │
  │  │                                          │            │
  │  │ # Register kretprobe event:             │            │
  │  │ echo 'r:myretprobe do_sys_openat2 \      │            │
  │  │   ret=$retval:s64' > \                   │            │
  │  │   /sys/kernel/tracing/kprobe_events      │            │
  │  │                                          │            │
  │  │ # Enable:                               │            │
  │  │ echo 1 > events/kprobes/myprobe/enable   │            │
  │  │ echo 1 > events/kprobes/myretprobe/enable│            │
  │  │                                          │            │
  │  │ # Read:                                 │            │
  │  │ cat trace_pipe                            │            │
  │  │ → myprobe: (do_sys_openat2+0x0)          │            │
  │  │   dfd=-100 filename="/etc/passwd"        │            │
  │  │   flags=0x0 mode=0x0                     │            │
  │  │ → myretprobe: (do_sys_openat2+0x0)       │            │
  │  │   ret=3                                  │            │
  │  │                                          │            │
  │  │ # Remove:                               │            │
  │  │ echo '-:myprobe' >> kprobe_events         │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  ftrace-based kprobes (optimization):                    │
  │  ┌──────────────────────────────────────────┐            │
  │  │ If probe at function entry AND function  │            │
  │  │ has ftrace hook (mcount/fentry nop):     │            │
  │  │                                          │            │
  │  │ Instead of INT3 → uses ftrace mechanism  │            │
  │  │ (NOP → CALL trampoline)                  │            │
  │  │                                          │            │
  │  │ Benefits:                                │            │
  │  │ - No single-stepping (faster)            │            │
  │  │ - No INT3 trap overhead                  │            │
  │  │ - Same probe functionality               │            │
  │  │                                          │            │
  │  │ Check: kprobe->flags & KPROBE_FLAG_FTRACE│            │
  │  │ CONFIG_KPROBES_ON_FTRACE=y              │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: What are the limitations and risks of kprobes?**
**A:** Limitations: (1) **Cannot probe certain functions**: `kprobe_handler` itself, NMI handlers, and functions called from kprobe infrastructure (would cause infinite recursion). The kernel has a blacklist (`/sys/kernel/debug/kprobes/blacklist`). (2) **Architecture dependent**: relies on breakpoint instructions (INT3/BRK) and single-stepping; not available on all architectures. (3) **Not inline-safe**: can only probe instruction boundaries — you can't probe the middle of a multi-byte instruction (would corrupt the instruction stream). (4) **Performance overhead**: INT3 trap + single-step is expensive (~1-5μs per probe hit on x86). For high-frequency functions, this matters. ftrace-based optimization helps for function entry probes. (5) **Not ABI-stable**: probes are set by kernel symbol+offset, which can change across kernel versions. (6) **Maxactive limit for kretprobes**: the pool of return instances has a fixed size; if exhausted (too many concurrent calls), probes are missed. Risks: (1) Incorrect probe handler can crash the kernel (runs in atomic context, must not sleep). (2) Probe on a hot path can cause severe performance degradation. (3) Modifying registers in the handler can cause unpredictable behavior if not done carefully.

**Q2: Compare kprobes, tracepoints, and ftrace for kernel instrumentation.**
**A:** **Tracepoints (static)**: predefined by kernel developers in source code. Advantages: stable (maintained across versions), efficient (static key NOP), type-safe (TRACE_EVENT defines field types), visible in tracefs. Disadvantages: only where developers put them — can't add new ones without kernel changes. **ftrace (function tracer)**: traces function entry/exit using compiler-inserted hooks. Advantages: covers every function, function_graph shows call trees. Disadvantages: only entry/exit (no instruction-level), no argument access beyond registers. **kprobes (dynamic)**: probe any instruction at runtime. Advantages: most flexible — any address, access registers, modify execution. Can probe places without tracepoints. Disadvantages: fragile (breaks across kernel versions), more overhead (INT3), must handle architecture details (register conventions). **Usage guidelines**: Use tracepoints first — they're stable and efficient. Use ftrace function when you need call flow analysis. Use kprobes when tracepoints don't exist for your instrumentation point. In practice, eBPF programs attach to all three: tracepoints (most common), kprobes (when needed), and ftrace (via fentry/fexit BPF programs for lowest overhead dynamic probing).

---

## Summary

- kprobes: dynamic instrumentation — INT3 breakpoint at any kernel instruction
- pre_handler runs before, post_handler after the probed instruction
- kretprobes: return address hijack to probe function exit + capture return value
- tracefs interface: write to kprobe_events to create probes without C code
- ftrace-based optimization: function entry probes use NOP→CALL instead of INT3
- Blacklist: certain functions cannot be probed (kprobe internals, NMI handlers)

---

[Previous: Tracepoints and Events ←](Chapter_15_Tracepoints.md) | [Next: perf Events and PMU →](Chapter_17_perf.md)
