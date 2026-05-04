# Chapter 26: Oops and Panic Analysis

## Learning Goals
- Decode kernel Oops messages and call traces
- Understand panic vs oops behavior
- Master addr2line, faddr2line, and decode_stacktrace.sh
- Know register decoding and fault analysis

---

## 1. Kernel Oops Anatomy

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Oops = kernel detected a fault but may continue        │
  │  Panic = fatal error, kernel halts                      │
  │                                                           │
  │  Oops triggers:                                          │
  │  - NULL pointer dereference                              │
  │  - Invalid memory access                                │
  │  - General protection fault (GPF)                       │
  │  - BUG() / BUG_ON() assertion                           │
  │  - WARN() — non-fatal, prints stack but continues       │
  │                                                           │
  │  Oops → Panic escalation:                               │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Oops in interrupt context → always panic │            │
  │  │ Oops in process context → kill process   │            │
  │  │   (unless panic_on_oops=1 → panic)       │            │
  │  │ Oops in init (PID 1) → always panic      │            │
  │  │                                          │            │
  │  │ /proc/sys/kernel/panic_on_oops           │            │
  │  │   0 = kill faulting process, continue    │            │
  │  │   1 = immediate panic                    │            │
  │  │                                          │            │
  │  │ /proc/sys/kernel/panic                   │            │
  │  │   0 = hang on panic                      │            │
  │  │   N = reboot after N seconds             │            │
  │  │  -1 = immediate reboot                   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Full Oops message example:                              │
  │  ┌──────────────────────────────────────────┐            │
  │  │ BUG: unable to handle page fault for     │            │
  │  │   address: ffff888012345008              │            │
  │  │ #PF: supervisor read access in kernel    │            │
  │  │   mode                                   │            │
  │  │ #PF: error_code(0x0000) - not-present    │            │
  │  │   page                                   │            │
  │  │ PGD 4a01067 P4D 4a01067 PUD 4a02067     │            │
  │  │   PMD 0 ← page not mapped              │            │
  │  │                                          │            │
  │  │ Oops: 0000 [#1] PREEMPT SMP             │            │
  │  │   │    │     │                           │            │
  │  │   │    │     └── 1st oops this boot      │            │
  │  │   │    └──────── error code bits:        │            │
  │  │   │               bit 0: 0=not-present   │            │
  │  │   │                      1=protection    │            │
  │  │   │               bit 1: 0=read 1=write  │            │
  │  │   │               bit 2: 0=kernel 1=user │            │
  │  │   └───────────── "Oops" keyword          │            │
  │  │                                          │            │
  │  │ CPU: 2 PID: 1234 Comm: myapp            │            │
  │  │   Tainted: G    W   O   6.1.0            │            │
  │  │   │                                      │            │
  │  │   └── Taint flags:                       │            │
  │  │       G = proprietary module loaded      │            │
  │  │       W = warning occurred previously    │            │
  │  │       O = out-of-tree module loaded      │            │
  │  │       E = unsigned module loaded         │            │
  │  │       (empty = pristine kernel)          │            │
  │  │                                          │            │
  │  │ RIP: 0010:my_driver_read+0x28/0x80       │            │
  │  │       │    │                  │    │      │            │
  │  │       │    │                  │    └ func size│       │
  │  │       │    │                  └ offset    │            │
  │  │       │    └──── function name           │            │
  │  │       └───────── code segment (ring 0)   │            │
  │  │                                          │            │
  │  │ RSP: 0018:ffffc90001234567              │            │
  │  │ RAX: 0000000000000000  RBX: ffff888012345000│        │
  │  │ RCX: 0000000000000008  ... (all regs)   │            │
  │  │                                          │            │
  │  │ Call Trace:                              │            │
  │  │  <TASK>                                  │            │
  │  │  vfs_read+0x9d/0x1b0                    │            │
  │  │  ksys_read+0x62/0xd0                    │            │
  │  │  do_syscall_64+0x3b/0x90               │            │
  │  │  entry_SYSCALL_64_after_hwframe+0x63/0xcd│           │
  │  │  </TASK>                                 │            │
  │  │                                          │            │
  │  │ Code: 48 8b 43 08 48 85 c0 74 12 ...    │            │
  │  │       ← bytes around faulting instruction│            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Decoding Tools

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  addr2line — address to source line:                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # From RIP: my_driver_read+0x28/0x80    │            │
  │  │ # Need: vmlinux or .ko with debug info  │            │
  │  │                                          │            │
  │  │ addr2line -e vmlinux -fip <address>      │            │
  │  │ → my_driver_read at drivers/foo/bar.c:42 │            │
  │  │                                          │            │
  │  │ # For module:                            │            │
  │  │ addr2line -e my_driver.ko -fip 0x28      │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  faddr2line — function+offset to source line:            │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Kernel utility script:                │            │
  │  │ scripts/faddr2line vmlinux \              │            │
  │  │   my_driver_read+0x28/0x80               │            │
  │  │                                          │            │
  │  │ → my_driver_read+0x28/0x80:             │            │
  │  │   my_driver_read at drivers/foo/bar.c:42│            │
  │  │    41: ptr = dev->priv;                 │            │
  │  │   →42: val = ptr->data;  ← THIS LINE   │            │
  │  │    43: return val;                       │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  decode_stacktrace.sh — full Oops decode:                │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Pipe dmesg through:                   │            │
  │  │ dmesg | scripts/decode_stacktrace.sh \    │            │
  │  │   vmlinux /path/to/modules/              │            │
  │  │                                          │            │
  │  │ → Replaces all addresses with           │            │
  │  │   file:line references                  │            │
  │  │                                          │            │
  │  │ # For saved oops:                       │            │
  │  │ cat oops.txt | scripts/                  │            │
  │  │   decode_stacktrace.sh vmlinux           │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  objdump — disassembly around fault:                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ # Disassemble function:                 │            │
  │  │ objdump -dS vmlinux | \                  │            │
  │  │   grep -A 20 'my_driver_read>:'          │            │
  │  │                                          │            │
  │  │ # Or disassemble the Code: bytes:       │            │
  │  │ echo "48 8b 43 08 48 85 c0 74 12" | \    │            │
  │  │   xxd -r -p | \                          │            │
  │  │   objdump -d -b binary -m i386:x86-64 /dev/stdin│   │
  │  │ → mov 0x8(%rbx),%rax    ← faulting insn │            │
  │  │ → test %rax,%rax                         │            │
  │  │ → je <skip>                              │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: Walk through how you would analyze a kernel Oops from a bug report.**
**A:** Step-by-step: (1) **Read the header**: `BUG: unable to handle page fault for address: 0000000000000008` — this is a NULL pointer dereference (address near 0, likely `NULL->field` where field offset is 0x8). (2) **Check RIP**: `RIP: 0010:my_driver_read+0x28/0x80` — the crash is in `my_driver_read`, 0x28 bytes from start of an 0x80-byte function. (3) **Check taint**: `Tainted: G O` means proprietary module + out-of-tree module loaded — might be relevant if the OOT module is the offending one. (4) **Decode source line**: Run `scripts/faddr2line vmlinux my_driver_read+0x28/0x80` → get exact source line. (5) **Read registers**: if `RAX: 0000000000000000` and the disassembly shows `mov 0x8(%rax),%rbx`, then RAX is NULL and we're dereferencing `NULL->field_at_offset_8`. Check which variable maps to RAX — look at the C code and compiler's register allocation. (6) **Read call trace**: bottom-up: `do_syscall_64` → `ksys_read` → `vfs_read` → `my_driver_read` — this was triggered by a read() syscall. (7) **Reproduce**: what user action triggers this? Is the device properly initialized? Race condition? (8) **Fix**: typically a missing NULL check (`if (!ptr) return -EINVAL;`), a use-after-free, or a missing initialization.

---

## Summary

- Oops: fault in process context → kills process (unless panic_on_oops=1)
- Panic: fatal — oops in IRQ context, init crash, explicit panic() call
- Oops contains: RIP (faulting address), registers, call trace, Code bytes, taint flags
- faddr2line: converts function+offset → source:line (most useful tool)
- decode_stacktrace.sh: pipes dmesg, resolves all addresses to source lines
- Error code bits: 0x0=not-present read, 0x2=write, 0x4=user-mode access

---

[Previous: Sparse and Coccinelle ←](Chapter_25_Sparse_Coccinelle.md) | [Next: kdump and crash →](Chapter_27_kdump_crash.md)
