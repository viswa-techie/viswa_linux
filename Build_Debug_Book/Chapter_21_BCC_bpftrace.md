# Chapter 21: BCC and bpftrace

## Learning Goals
- Understand BCC (BPF Compiler Collection) tools and Python API
- Learn bpftrace one-liner syntax and AWK-like language
- Master common BCC tools for production tracing
- Know when to use BCC vs bpftrace vs libbpf

---

## 1. BCC Tools

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  BCC = BPF Compiler Collection                           │
  │  Pre-built tools + Python/C++ API for eBPF programming  │
  │                                                           │
  │  Key tools (from Brendan Gregg et al.):                  │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Process/CPU:                             │            │
  │  │   execsnoop  — trace new process exec    │            │
  │  │   exitsnoop  — trace process exits       │            │
  │  │   runqlat    — scheduler run queue latency│           │
  │  │   runqlen    — run queue length histogram│            │
  │  │   cpudist    — on/off-CPU time distrib.  │            │
  │  │   offcputime — off-CPU stack traces      │            │
  │  │   profile    — CPU profiler (flame graph)│            │
  │  │                                          │            │
  │  │ I/O and filesystem:                      │            │
  │  │   biolatency — block I/O latency histogram│          │
  │  │   biosnoop   — trace block I/O with latency│         │
  │  │   ext4slower — slow ext4 operations      │            │
  │  │   filetop    — file I/O by process       │            │
  │  │   opensnoop  — trace open() syscalls     │            │
  │  │   filelife   — trace file create/delete  │            │
  │  │                                          │            │
  │  │ Networking:                              │            │
  │  │   tcpconnect — trace TCP connections     │            │
  │  │   tcpaccept  — trace TCP accepts         │            │
  │  │   tcplife    — TCP session lifespans     │            │
  │  │   tcpretrans — TCP retransmissions       │            │
  │  │                                          │            │
  │  │ Memory:                                  │            │
  │  │   memleak    — memory leak detector      │            │
  │  │   cachestat  — page cache hit/miss       │            │
  │  │   slabratetop — SLAB allocator rates     │            │
  │  │                                          │            │
  │  │ General:                                 │            │
  │  │   trace      — custom function tracing   │            │
  │  │   argdist    — argument distribution     │            │
  │  │   funccount  — function call counting    │            │
  │  │   stackcount — stack trace counting      │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  BCC Python example:                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ from bcc import BPF                      │            │
  │  │                                          │            │
  │  │ prog = """                               │            │
  │  │ BPF_HASH(counts, u32, u64);              │            │
  │  │ int trace_open(struct pt_regs *ctx) {    │            │
  │  │   u32 pid = bpf_get_current_pid_tgid()   │            │
  │  │     >> 32;                               │            │
  │  │   u64 *val, zero = 0;                    │            │
  │  │   val = counts.lookup_or_try_init(       │            │
  │  │     &pid, &zero);                        │            │
  │  │   if (val) (*val)++;                     │            │
  │  │   return 0;                              │            │
  │  │ }                                        │            │
  │  │ """                                      │            │
  │  │ b = BPF(text=prog)                       │            │
  │  │ b.attach_kprobe(event="do_sys_openat2",  │            │
  │  │   fn_name="trace_open")                  │            │
  │  │ while True:                              │            │
  │  │   for k, v in b["counts"].items():       │            │
  │  │     print(f"PID {k.value}: {v.value}")   │            │
  │  │   b["counts"].clear()                    │            │
  │  │   sleep(1)                               │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. bpftrace

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  bpftrace: high-level tracing language (AWK-like)       │
  │  One-liners for quick kernel investigation              │
  │                                                           │
  │  Syntax:  probe /filter/ { action }                     │
  │                                                           │
  │  Probes:                                                 │
  │  ┌──────────────────────────────────────────┐            │
  │  │ kprobe:func_name      — kernel func entry│            │
  │  │ kretprobe:func_name   — kernel func return│           │
  │  │ tracepoint:cat:name   — static tracepoint│            │
  │  │ usdt:binary:probe     — userspace USDT   │            │
  │  │ uprobe:binary:func    — userspace func   │            │
  │  │ profile:hz:99         — CPU sampling     │            │
  │  │ interval:s:1          — periodic timer   │            │
  │  │ BEGIN / END            — program start/end│           │
  │  │ software:faults:1     — SW events        │            │
  │  │ hardware:cache-misses:1000 — HW events   │            │
  │  │ fentry:func / fexit:func — ftrace-based  │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Essential one-liners:                                   │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Trace new processes:                  │            │
  │  │ bpftrace -e 'tracepoint:syscalls:        │            │
  │  │   sys_enter_execve {                     │            │
  │  │   printf("%s %s\n", comm, str(args.      │            │
  │  │     filename)); }'                       │            │
  │  │                                          │            │
  │  │ # Count syscalls by name:               │            │
  │  │ bpftrace -e 'tracepoint:raw_syscalls:    │            │
  │  │   sys_enter { @[ksym(*(kaddr(             │            │
  │  │   "sys_call_table") + args.id * 8))] =   │            │
  │  │   count(); }'                            │            │
  │  │                                          │            │
  │  │ # Block I/O latency histogram:          │            │
  │  │ bpftrace -e 'kprobe:blk_account_io_start│            │
  │  │   { @start[arg0] = nsecs; }              │            │
  │  │   kprobe:blk_account_io_done             │            │
  │  │   /@start[arg0]/ {                       │            │
  │  │   @usecs = hist((nsecs - @start[arg0])   │            │
  │  │     / 1000);                             │            │
  │  │   delete(@start[arg0]); }'               │            │
  │  │                                          │            │
  │  │ # Scheduler latency:                    │            │
  │  │ bpftrace -e 'tracepoint:sched:           │            │
  │  │   sched_wakeup { @[tid] = nsecs; }       │            │
  │  │   tracepoint:sched:sched_switch          │            │
  │  │   /@[tid]/ { @usecs = hist((nsecs -      │            │
  │  │     @[tid]) / 1000);                     │            │
  │  │   delete(@[tid]); }'                     │            │
  │  │                                          │            │
  │  │ # CPU profiling (flame graph):          │            │
  │  │ bpftrace -e 'profile:hz:99 {            │            │
  │  │   @[kstack] = count(); }'               │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Built-in variables:                                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ pid, tid, uid, gid, comm, curtask,       │            │
  │  │ nsecs, elapsed, kstack, ustack, cpu,     │            │
  │  │ args (tracepoint fields), retval,        │            │
  │  │ arg0-arg5 (kprobe arguments)             │            │
  │  │                                          │            │
  │  │ Map functions: count(), sum(), avg(),    │            │
  │  │   min(), max(), hist(), lhist(), stats() │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Tool Selection

```
  ┌──────────────────────────────────────────────────────────┐
  │  ┌─────────────┬────────────────────────────────────────┐│
  │  │ Tool        │ Best For                              ││
  │  ├─────────────┼────────────────────────────────────────┤│
  │  │ bpftrace    │ Quick one-liners, ad-hoc exploration, ││
  │  │             │ prototyping, interactive debugging     ││
  │  ├─────────────┼────────────────────────────────────────┤│
  │  │ BCC tools   │ Pre-built production tools (just run  ││
  │  │             │ execsnoop, biolatency, etc.)           ││
  │  ├─────────────┼────────────────────────────────────────┤│
  │  │ BCC Python  │ Custom tools with rich output, data   ││
  │  │             │ processing, complex logic              ││
  │  ├─────────────┼────────────────────────────────────────┤│
  │  │ libbpf/CO-RE│ Production deployment, portable      ││
  │  │             │ across kernels, compiled once, no     ││
  │  │             │ runtime compilation overhead           ││
  │  └─────────────┴────────────────────────────────────────┘│
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How would you use BCC/bpftrace to diagnose a production latency issue?**
**A:** Systematic approach: (1) **CPU or off-CPU?** Run `bpftrace -e 'profile:hz:99 { @[kstack] = count(); }'` for on-CPU profiling. Simultaneously `offcputime-bpfcc -K -p <pid>` for off-CPU stacks. If the process is mostly on-CPU: hot function in CPU profile. If mostly off-CPU: it's blocking on something (I/O, lock, sleep). (2) **Scheduler latency?** `runqlat` — shows distribution of time tasks wait in the run queue. If tail latencies are high, CPUs may be oversubscribed. (3) **I/O latency?** `biolatency` — block I/O latency histogram. `biosnoop` — per-I/O latency with device/offset. `ext4slower 10` — ext4 operations taking > 10ms. (4) **Network latency?** `tcpretrans` — retransmissions. `tcplife` — connection durations. Custom bpftrace probing specific kernel functions. (5) **Lock contention?** `bpftrace -e 'kprobe:mutex_lock { @[kstack] = count(); }'` — where locks are acquired. (6) **Specific function latency?** `funclatency -p <pid> functionname` — latency distribution for a specific function. The key advantage of eBPF for production: these tools run with minimal overhead (a few percent) and no kernel recompilation. You can deploy them on live production systems safely.

**Q2: What are the tradeoffs between BCC and libbpf for eBPF development?**
**A:** **BCC**: compiles eBPF C code at runtime using embedded LLVM/Clang. Advantages: (1) Easy Python API for rapid development. (2) Rewrites C code to match running kernel headers (adapts to struct layout changes). (3) Huge library of ready-made tools. Disadvantages: (1) Requires LLVM/Clang installed on target (100+ MB). (2) Compilation at startup (1-5 seconds delay). (3) High memory usage (LLVM in memory). (4) Fragile: depends on exact kernel headers being installed. **libbpf (CO-RE)**: compiles eBPF once with Clang, produces a portable `.o` file. At load time, libbpf uses BTF (BPF Type Format) information to relocate field accesses for the running kernel. Advantages: (1) No Clang/LLVM on target — just the compiled binary. (2) Instant startup (no compilation). (3) Low memory footprint. (4) Portable across kernel versions (CO-RE = Compile Once, Run Everywhere). (5) Better verifier diagnostics (BTF-aware). Disadvantages: (1) More complex development (C skeleton, manual map management). (2) Requires BTF-enabled kernel (CONFIG_DEBUG_INFO_BTF=y). (3) Less forgiving — must handle struct changes via `bpf_core_read()` and `BPF_CORE_READ()` macros. **Recommendation**: Use BCC tools/bpftrace for ad-hoc debugging and prototyping. Use libbpf/CO-RE for production tooling, embedded systems, and distributed deployment.

---

## Summary

- BCC: Python/C++ API + 100+ pre-built tools (execsnoop, biolatency, tcpconnect, ...)
- bpftrace: AWK-like language for eBPF one-liners; probes: kprobe, tracepoint, profile, etc.
- bpftrace maps: @name[key] = count()/hist()/sum() — in-kernel aggregation
- BCC tools run on live production with minimal overhead (few percent)
- Tool selection: bpftrace for quick investigation, BCC tools for standard monitoring, libbpf for production deployment

---

[Previous: eBPF Maps and Helpers ←](Chapter_20_eBPF_Maps.md) | [Next: CO-RE and libbpf →](Chapter_22_CO_RE.md)
