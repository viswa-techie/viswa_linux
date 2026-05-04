# Chapter 19: eBPF Architecture

## Learning Goals
- Understand eBPF virtual machine and instruction set
- Learn the verifier and safety guarantees
- Master program types and attachment points
- Know JIT compilation and helper functions

---

## 1. eBPF Virtual Machine

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  eBPF = Extended Berkeley Packet Filter                  │
  │  Originally: packet filtering. Now: programmable         │
  │  kernel instrumentation for tracing, networking,        │
  │  security, and more.                                    │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ eBPF VM architecture:                    │            │
  │  │                                          │            │
  │  │ Registers: R0-R10 (64-bit)              │            │
  │  │   R0: return value                       │            │
  │  │   R1-R5: function arguments              │            │
  │  │   R6-R9: callee-saved                    │            │
  │  │   R10: frame pointer (read-only)         │            │
  │  │                                          │            │
  │  │ Instruction format:                      │            │
  │  │   8-byte instructions:                   │            │
  │  │   ┌──────┬───┬───┬────────┬───────────┐  │            │
  │  │   │opcode│dst│src│ offset │ immediate │  │            │
  │  │   │ 8bit │4b │4b │ 16bit  │  32bit    │  │            │
  │  │   └──────┴───┴───┴────────┴───────────┘  │            │
  │  │                                          │            │
  │  │ Instruction classes:                     │            │
  │  │   ALU (32/64): add, sub, mul, div, mod   │            │
  │  │   Memory: ld, st (1/2/4/8 byte)         │            │
  │  │   Branch: jeq, jne, jgt, jlt, call, exit│            │
  │  │   Atomic: xadd, cmpxchg, xchg           │            │
  │  │                                          │            │
  │  │ Stack: 512 bytes (fixed, per program)    │            │
  │  │ No loops (originally); bounded loops ok  │            │
  │  │ with verifiable termination (5.3+)       │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Program lifecycle:                                      │
  │  ┌──────────────────────────────────────────┐            │
  │  │                                          │            │
  │  │ C code ──→ Clang/LLVM ──→ eBPF bytecode │            │
  │  │                              │            │            │
  │  │              ┌───────────────▼──────────┐ │            │
  │  │              │ bpf() syscall           │ │            │
  │  │              │ BPF_PROG_LOAD           │ │            │
  │  │              └───────────────┬──────────┘ │            │
  │  │                              │            │            │
  │  │              ┌───────────────▼──────────┐ │            │
  │  │              │ Verifier                 │ │            │
  │  │              │ - bounds checking        │ │            │
  │  │              │ - no invalid memory      │ │            │
  │  │              │ - guaranteed termination │ │            │
  │  │              │ - type safety            │ │            │
  │  │              └───────────────┬──────────┘ │            │
  │  │                              │            │            │
  │  │              ┌───────────────▼──────────┐ │            │
  │  │              │ JIT Compiler             │ │            │
  │  │              │ eBPF bytecode → native   │ │            │
  │  │              │ x86/ARM64/etc. machine   │ │            │
  │  │              │ code                     │ │            │
  │  │              └───────────────┬──────────┘ │            │
  │  │                              │            │            │
  │  │              ┌───────────────▼──────────┐ │            │
  │  │              │ Attach to hook point:    │ │            │
  │  │              │ kprobe, tracepoint,      │ │            │
  │  │              │ XDP, tc, cgroup, ...     │ │            │
  │  │              └──────────────────────────┘ │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Verifier

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  The verifier: ensures eBPF programs are safe to run    │
  │  in kernel context. Static analysis before execution.   │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Checks performed:                        │            │
  │  │                                          │            │
  │  │ 1. DAG check: no unreachable code,       │            │
  │  │    no backward jumps that form unbounded │            │
  │  │    loops (bounded loops OK since 5.3)    │            │
  │  │                                          │            │
  │  │ 2. Instruction simulation: walks every   │            │
  │  │    possible execution path, tracking     │            │
  │  │    register state (type + value range)   │            │
  │  │                                          │            │
  │  │ 3. Memory safety:                        │            │
  │  │    - No out-of-bounds stack access       │            │
  │  │    - Map value access only after         │            │
  │  │      NULL check                          │            │
  │  │    - Packet data access within bounds    │            │
  │  │    - No use of uninitialized registers   │            │
  │  │                                          │            │
  │  │ 4. Helper function validation:           │            │
  │  │    - Only allowed helpers for program    │            │
  │  │      type                                │            │
  │  │    - Correct argument types              │            │
  │  │                                          │            │
  │  │ 5. Complexity limit:                     │            │
  │  │    - Max 1M verified instructions        │            │
  │  │    - Max 33 tail calls                   │            │
  │  │    - Stack depth <= 512 bytes            │            │
  │  │                                          │            │
  │  │ Result: program is GUARANTEED safe or    │            │
  │  │ rejected with error message              │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. Program Types and Attachment Points

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  ┌─────────────────┬────────────────────────────────────┐│
  │  │ Program Type    │ Use Case                          ││
  │  ├─────────────────┼────────────────────────────────────┤│
  │  │ BPF_PROG_TYPE_  │                                    ││
  │  │ KPROBE          │ Dynamic kernel probing             ││
  │  │ TRACEPOINT      │ Static kernel tracepoints          ││
  │  │ RAW_TRACEPOINT  │ Raw tracepoint (no arg copying)   ││
  │  │ PERF_EVENT      │ perf events (HW/SW counters)      ││
  │  │ TRACING         │ fentry/fexit (ftrace-based)       ││
  │  │ XDP             │ eXpress Data Path (NIC level)     ││
  │  │ SCHED_CLS       │ TC classifier (qdisc)             ││
  │  │ CGROUP_SKB      │ cgroup packet filter              ││
  │  │ SOCKET_FILTER   │ Socket-level packet filter        ││
  │  │ LSM             │ Linux Security Module hooks       ││
  │  │ STRUCT_OPS      │ TCP congestion control, etc.      ││
  │  │ ITER            │ Iterate kernel data structures    ││
  │  │ SYSCALL         │ Attach to syscall entry/exit      ││
  │  └─────────────────┴────────────────────────────────────┘│
  │                                                           │
  │  fentry/fexit (BPF_PROG_TYPE_TRACING):                  │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Newest and most efficient attachment:    │            │
  │  │ - Uses ftrace infrastructure (no INT3)   │            │
  │  │ - Direct access to function arguments    │            │
  │  │   (type-safe via BTF)                    │            │
  │  │ - fexit: access both args AND return val │            │
  │  │ - Lower overhead than kprobe             │            │
  │  │                                          │            │
  │  │ SEC("fentry/do_sys_openat2")             │            │
  │  │ int BPF_PROG(trace_open,                 │            │
  │  │     int dfd,                             │            │
  │  │     struct filename *name,               │            │
  │  │     struct open_how *how) {              │            │
  │  │   bpf_printk("open: %s\n",              │            │
  │  │     name->name);                         │            │
  │  │   return 0;                              │            │
  │  │ }                                        │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does the eBPF verifier ensure safety, and what are its limitations?**
**A:** The verifier performs static analysis via abstract interpretation. It simulates execution of every possible path through the program, tracking register states as `(type, value_range)` tuples. For example, after `r1 = bpf_map_lookup_elem(...)`, r1 has type `PTR_TO_MAP_VALUE_OR_NULL`. The verifier requires a null check before dereferencing — `if (!r1) return 0;` — after which r1's type changes to `PTR_TO_MAP_VALUE` with known bounds. Key safety guarantees: (1) No unbounded loops — either no backward edges (original) or provably bounded iterations (5.3+). (2) No invalid memory access — all pointer arithmetic checked against type bounds, no arbitrary kernel memory reads. (3) No uninitialized data leaks — registers must be initialized before use. (4) Helper function contracts — each helper has defined argument types and return types the verifier enforces. Limitations: (1) **False rejections**: the verifier is conservative — some safe programs are rejected because the verifier can't prove safety (e.g., complex pointer arithmetic). (2) **Complexity limit**: programs > 1M verified instructions are rejected (needed for termination of the verifier itself). (3) **State explosion**: paths with many branches cause exponential states — mitigated by state pruning. (4) **BTF dependency**: modern features (CO-RE, fentry) require BTF (BPF Type Format) in the kernel, which isn't always available on older/embedded kernels. (5) **Privileged by default**: loading eBPF programs requires CAP_BPF (or CAP_SYS_ADMIN on older kernels), though `BPF_PROG_TYPE_SOCKET_FILTER` and `BPF_PROG_TYPE_CGROUP_*` can be unprivileged.

**Q2: Compare kprobes vs fentry/fexit for eBPF tracing.**
**A:** **kprobes-based eBPF**: attaches via `SEC("kprobe/func_name")`. The kprobe fires (INT3 breakpoint on x86), the kernel BPF trampoline calls the eBPF program with a `struct pt_regs *` context — you access arguments through registers (`PT_REGS_PARM1(ctx)` etc.). Overhead: INT3 trap + register save/restore + eBPF program execution. Works on any kernel function. Not type-safe — you cast register values manually. **fentry/fexit-based eBPF** (`SEC("fentry/func_name")`): uses ftrace infrastructure (NOP → CALL trampoline). The eBPF program receives typed arguments directly — verified against BTF (BPF Type Format) information. fexit additionally provides the return value. Overhead: much lower — no INT3, no single-stepping, just a function call through the ftrace trampoline. Advantages of fentry/fexit: (1) 5-10x lower overhead. (2) Type-safe: verifier knows argument types from BTF, catches type errors at load time. (3) fexit gives both args and return value (kretprobe only gives return value, not original args). (4) Simpler code: `BPF_PROG(name, arg1_type arg1, arg2_type arg2)` vs manual `PT_REGS_PARM` macros. Disadvantage: requires BTF-enabled kernel (CONFIG_DEBUG_INFO_BTF=y), only works at function entry/exit (not arbitrary instructions like general kprobes).

---

## Summary

- eBPF: in-kernel virtual machine — 11 registers (R0-R10), 64-bit, 512B stack
- Lifecycle: C → Clang → bytecode → bpf() syscall → verifier → JIT → attach
- Verifier: static analysis — memory safety, bounds checking, guaranteed termination
- JIT: eBPF bytecode → native machine code (x86, ARM64, etc.) for near-native speed
- Program types: kprobe, tracepoint, XDP, TC, cgroup, LSM, fentry/fexit, struct_ops
- fentry/fexit: lowest-overhead tracing via ftrace + BTF type safety

---

[Previous: Perfetto ←](Chapter_18_Perfetto.md) | [Next: eBPF Maps and Helpers →](Chapter_20_eBPF_Maps.md)
