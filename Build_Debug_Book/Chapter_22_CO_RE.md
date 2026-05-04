# Chapter 22: CO-RE and libbpf

## Learning Goals
- Understand CO-RE (Compile Once, Run Everywhere) concept
- Learn BTF (BPF Type Format) and its role in portability
- Master libbpf skeleton workflow
- Know BPF_CORE_READ macros and field relocations

---

## 1. The Portability Problem and CO-RE

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  The problem:                                            │
  │  ┌──────────────────────────────────────────┐            │
  │  │ eBPF programs access kernel structs:     │            │
  │  │                                          │            │
  │  │ struct task_struct *task = ...;           │            │
  │  │ pid_t pid = task->pid;  // offset 0x5b0? │            │
  │  │                         // or 0x5c0?     │            │
  │  │                         // or 0x490?     │            │
  │  │                                          │            │
  │  │ Struct layout changes across kernel      │            │
  │  │ versions: fields added, removed,         │            │
  │  │ reordered, renamed.                      │            │
  │  │                                          │            │
  │  │ BCC solution: recompile on-target with   │            │
  │  │ local headers → works but needs LLVM     │            │
  │  │                                          │            │
  │  │ CO-RE solution: compile once, adjust     │            │
  │  │ offsets at load time using BTF           │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  BTF = BPF Type Format:                                  │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Compact type information embedded in     │            │
  │  │ kernel (CONFIG_DEBUG_INFO_BTF=y):        │            │
  │  │                                          │            │
  │  │ /sys/kernel/btf/vmlinux                  │            │
  │  │ → full struct definitions for running    │            │
  │  │   kernel (all types, fields, offsets)    │            │
  │  │                                          │            │
  │  │ Size: ~3-5 MB (vs ~500 MB for DWARF)    │            │
  │  │                                          │            │
  │  │ Contains:                                │            │
  │  │ - struct/union/enum definitions          │            │
  │  │ - Field names, types, offsets, sizes     │            │
  │  │ - Function prototypes                    │            │
  │  │ - Typedef chains                         │            │
  │  │                                          │            │
  │  │ Generate vmlinux.h (all kernel types):   │            │
  │  │ bpftool btf dump file /sys/kernel/btf/   │            │
  │  │   vmlinux format c > vmlinux.h           │            │
  │  │ → one header replaces ALL kernel headers │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  CO-RE relocation:                                       │
  │  ┌──────────────────────────────────────────┐            │
  │  │                                          │            │
  │  │ Compile time (on dev machine):           │            │
  │  │   Clang compiles C with #include "vmlinux.h"│        │
  │  │   Records field access as relocations:   │            │
  │  │   "accessing field 'pid' (type 'task_struct')"│      │
  │  │   Stores in .BTF.ext section of ELF      │            │
  │  │                                          │            │
  │  │ Load time (on target machine):           │            │
  │  │   libbpf reads target's /sys/kernel/btf/vmlinux│     │
  │  │   For each relocation:                   │            │
  │  │     - Find 'task_struct' in target BTF   │            │
  │  │     - Find field 'pid' → get offset      │            │
  │  │     - Patch eBPF bytecode instruction    │            │
  │  │       with correct offset                │            │
  │  │                                          │            │
  │  │ Result: same .o file works on 5.4,       │            │
  │  │   5.10, 5.15, 6.1, ... (if field exists) │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. libbpf Skeleton Workflow

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Development workflow:                                   │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. Write eBPF program (myprobe.bpf.c):  │            │
  │  │                                          │            │
  │  │ #include "vmlinux.h"                     │            │
  │  │ #include <bpf/bpf_helpers.h>             │            │
  │  │ #include <bpf/bpf_tracing.h>             │            │
  │  │ #include <bpf/bpf_core_read.h>           │            │
  │  │                                          │            │
  │  │ struct {                                 │            │
  │  │   __uint(type, BPF_MAP_TYPE_RINGBUF);    │            │
  │  │   __uint(max_entries, 256 * 1024);       │            │
  │  │ } events SEC(".maps");                   │            │
  │  │                                          │            │
  │  │ SEC("tp/sched/sched_process_exec")       │            │
  │  │ int handle_exec(struct trace_event_raw_  │            │
  │  │   sched_process_exec *ctx) {             │            │
  │  │   struct task_struct *task =              │            │
  │  │     (void *)bpf_get_current_task();      │            │
  │  │   pid_t pid = BPF_CORE_READ(task, pid);  │            │
  │  │   // ... emit to ringbuf                 │            │
  │  │   return 0;                              │            │
  │  │ }                                        │            │
  │  │ char LICENSE[] SEC("license") = "GPL";   │            │
  │  │                                          │            │
  │  │ 2. Compile to BPF object:               │            │
  │  │ clang -g -O2 -target bpf \               │            │
  │  │   -D__TARGET_ARCH_x86 \                  │            │
  │  │   -c myprobe.bpf.c -o myprobe.bpf.o     │            │
  │  │                                          │            │
  │  │ 3. Generate skeleton header:             │            │
  │  │ bpftool gen skeleton myprobe.bpf.o > \   │            │
  │  │   myprobe.skel.h                         │            │
  │  │                                          │            │
  │  │ 4. Write userspace loader (myprobe.c):   │            │
  │  │ #include "myprobe.skel.h"                │            │
  │  │                                          │            │
  │  │ struct myprobe_bpf *skel =               │            │
  │  │   myprobe_bpf__open_and_load();          │            │
  │  │ myprobe_bpf__attach(skel);               │            │
  │  │ // ... poll ring buffer                  │            │
  │  │ myprobe_bpf__destroy(skel);              │            │
  │  │                                          │            │
  │  │ 5. Compile userspace:                   │            │
  │  │ gcc -o myprobe myprobe.c -lbpf -lelf -lz│            │
  │  │                                          │            │
  │  │ 6. Deploy single binary — runs on any   │            │
  │  │    BTF-enabled kernel!                   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  BPF_CORE_READ macros:                                   │
  │  ┌──────────────────────────────────────────┐            │
  │  │ BPF_CORE_READ(task, pid)                 │            │
  │  │ → bpf_probe_read_kernel(&pid,            │            │
  │  │     sizeof(pid),                         │            │
  │  │     &task->pid /* CO-RE relocated */)    │            │
  │  │                                          │            │
  │  │ /* Chained reads (following pointers): */│            │
  │  │ BPF_CORE_READ(task, mm, pgd, pgd)        │            │
  │  │ → reads task->mm->pgd->pgd safely       │            │
  │  │   (each dereference is a separate        │            │
  │  │    bpf_probe_read_kernel call)           │            │
  │  │                                          │            │
  │  │ /* Field existence check: */             │            │
  │  │ if (bpf_core_field_exists(               │            │
  │  │       task->thread_pid)) {               │            │
  │  │   // only on kernels that have this field│            │
  │  │ }                                        │            │
  │  │                                          │            │
  │  │ /* Type size check: */                   │            │
  │  │ if (bpf_core_type_size(                  │            │
  │  │       struct task_struct) > 2048) { ... } │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does CO-RE achieve cross-kernel portability and what are its limitations?**
**A:** CO-RE works through three components: (1) **vmlinux.h** (compile-time): generated from a reference kernel's BTF, provides all struct definitions. The eBPF program is compiled against this. (2) **.BTF.ext relocations** (in the ELF object): When Clang compiles a field access like `task->pid`, it records a relocation entry: "this instruction accesses field `pid` of type `task_struct`" — storing the field name and type name, not the offset. (3) **libbpf relocation engine** (load-time): reads the target kernel's `/sys/kernel/btf/vmlinux`, finds `task_struct`, finds field `pid`, gets the actual offset on this kernel, and patches the eBPF bytecode instruction's offset operand. Supported relocations: field offset, field existence (`bpf_core_field_exists`), field size, type existence, type size, enum value. Limitations: (1) **BTF required**: target kernel must have `CONFIG_DEBUG_INFO_BTF=y` (available since 5.2, default in many distros since 5.10+). (2) **Field must exist by name**: if a field was renamed (e.g., `state` → `__state` in 5.14), you need `bpf_core_field_exists()` checks for both names. (3) **Semantic changes**: if a field's meaning changed (same name, different semantics), CO-RE can't detect that. (4) **Struct reorganization**: if a field was moved to a sub-struct, the type+field path changes — need conditional logic. (5) **No kernel header access**: can't use `#define` constants from kernel headers (need vmlinux.h or manual definitions).

---

## Summary

- CO-RE: Compile Once, Run Everywhere — portable eBPF programs across kernel versions
- BTF: compact type info (/sys/kernel/btf/vmlinux, ~3-5MB) — struct layouts for running kernel
- vmlinux.h: auto-generated header from BTF containing all kernel type definitions
- libbpf: load-time relocation engine — patches field offsets in eBPF bytecode
- Skeleton: bpftool gen skeleton → auto-generated open/load/attach/destroy API
- BPF_CORE_READ: safe pointer chasing with CO-RE relocations

---

[Previous: BCC and bpftrace ←](Chapter_21_BCC_bpftrace.md) | [Next: KASAN →](Chapter_23_KASAN.md)
