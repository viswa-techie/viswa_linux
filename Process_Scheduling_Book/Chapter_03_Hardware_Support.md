# Chapter 3: Hardware Support for Process Management

## Learning Goals
- Understand CPU features that enable multitasking
- Learn how context switching works at the hardware level
- Grasp the role of timers, interrupts, and privilege modes

---

## 3.1 CPU Context Switching — Hardware Perspective

The CPU has a finite set of registers. When switching from Task A to Task B, the kernel must:
1. **Save** Task A's register state to memory (`task_struct.thread`)
2. **Restore** Task B's register state from memory
3. **Switch** address space (page table pointer)

```
Task A running              Context Switch              Task B running
┌──────────────┐           ┌──────────────┐           ┌──────────────┐
│ PC = 0x4000  │  save     │ A's state    │  restore  │ PC = 0x8000  │
│ SP = 0xFF00  │ ────────► │ saved to     │ ────────► │ SP = 0xEE00  │
│ R0-R30       │           │ A's thread   │           │ R0-R30       │
│ PSTATE       │           │ (in memory)  │           │ PSTATE       │
└──────────────┘           └──────────────┘           └──────────────┘
                                  │
                            Switch page table
                            (TTBR0 on ARM64,
                             CR3 on x86_64)
```

---

## 3.2 Processor Modes (User Mode vs Kernel Mode)

| Mode | ARM64 | x86_64 | Access |
|------|-------|--------|--------|
| **User** | EL0 | Ring 3 | Restricted: no I/O, no privileged instructions |
| **Kernel** | EL1 | Ring 0 | Full: I/O, page tables, interrupts |
| **Hypervisor** | EL2 | Ring -1 (VMX root) | VM management |
| **Firmware** | EL3 (ARM TrustZone) | SMM | Secure boot, firmware |

```
┌─────────────────────────────────────┐
│  User Mode (EL0 / Ring 3)          │
│  Applications run here             │
│  Cannot: access hardware, switch   │
│  tasks, modify page tables         │
├─────────────────────────────────────┤
│  ▲ syscall / exception / IRQ       │
│  ▼ return to user (eret / sysret)  │
├─────────────────────────────────────┤
│  Kernel Mode (EL1 / Ring 0)        │
│  Scheduler, drivers, mm run here   │
│  Can: everything                   │
└─────────────────────────────────────┘
```

Transitions happen via:
- **Syscall** (`svc` on ARM64, `syscall` on x86_64) — deliberate user → kernel
- **Exception** (page fault, divide by zero) — fault during user execution
- **Interrupt** (timer, device) — hardware signals CPU

---

## 3.3 CPU Registers Involved in Context Switching

### ARM64 Registers

```
General purpose: X0-X30 (64-bit), with X29=FP, X30=LR
Stack pointer:   SP_EL0 (user), SP_EL1 (kernel)  
Program counter: PC (saved as ELR_EL1 on exception)
Status:          SPSR_EL1 (saved processor state)
Page table:      TTBR0_EL1 (user), TTBR1_EL1 (kernel)
FPU/SIMD:        V0-V31 (128-bit NEON/FP registers)
```

### x86_64 Registers

```
General purpose: RAX, RBX, RCX, RDX, RSI, RDI, RBP, RSP, R8-R15
Instruction ptr: RIP
Flags:           RFLAGS
Segments:        CS, DS, SS (mostly flat in 64-bit)
Page table:      CR3
FPU/SSE/AVX:     XMM0-XMM15, YMM0-YMM15
Kernel GS:       MSR_GS_BASE (per-CPU data pointer)
```

### What Gets Saved During Context Switch

```c
/* ARM64: arch/arm64/include/asm/processor.h */
struct cpu_context {
    unsigned long x19;    /* Callee-saved registers */
    unsigned long x20;
    unsigned long x21;
    unsigned long x22;
    unsigned long x23;
    unsigned long x24;
    unsigned long x25;
    unsigned long x26;
    unsigned long x27;
    unsigned long x28;
    unsigned long fp;     /* Frame pointer (x29) */
    unsigned long sp;     /* Stack pointer */
    unsigned long pc;     /* Program counter */
};
```

---

## 3.4 Hardware Timers and Scheduling

The scheduler needs periodic interrupts to make preemption decisions.

| Timer | Architecture | Linux Use |
|-------|-------------|-----------|
| **APIC Timer** | x86_64 | Per-CPU local timer, scheduler tick |
| **Generic Timer** | ARM64 | Per-CPU timer, scheduler tick |
| **HPET** | x86_64 | High precision event timer (fallback) |
| **TSC** | x86_64 | Time Stamp Counter (cycle counting) |

```
Timer interrupt flow:
  
  Hardware Timer ──IRQ──► CPU Exception
       │                       │
       │                       ▼
       │              timer_interrupt()
       │                       │
       │                       ▼
       │              scheduler_tick()
       │                       │
       │               ┌───────▼───────┐
       │               │ Update vruntime│
       │               │ Check preempt  │
       │               │ Set TIF_NEED_  │
       │               │ RESCHED if due │
       │               └───────────────┘
       │
  CONFIG_HZ = 100/250/1000
  (tick period = 10ms / 4ms / 1ms)
```

### Tickless Mode (NO_HZ)

```
CONFIG_NO_HZ_IDLE:  Stop tick when CPU is idle (saves power)
CONFIG_NO_HZ_FULL:  Stop tick when one task running (reduces overhead)

  Traditional:  |tick|tick|tick|tick|tick|tick|tick|  (constant overhead)
  NO_HZ_IDLE:   |tick|tick|----idle, no ticks----|tick|  (saves power)
  NO_HZ_FULL:   |tick|---single task, no ticks---|tick|  (low latency)
```

---

## 3.5 Interrupt-Driven Scheduling

Scheduling decisions happen at specific points — all triggered by kernel entry:

```
1. Timer interrupt → scheduler_tick() → set TIF_NEED_RESCHED
2. Return from interrupt/syscall → check TIF_NEED_RESCHED → schedule()
3. Explicit yield → schedule()
4. Blocking I/O → sleep → schedule()
5. Wake-up of higher-priority task → preempt current

Key check point (ARM64):
  el0_svc (syscall return) → ret_to_user → check TIF_NEED_RESCHED
  el1_irq (kernel IRQ)     → preempt_schedule_irq() if CONFIG_PREEMPT
```

---

## 3.6 CPU Core Architecture and Multitasking

### SMP (Symmetric Multiprocessing)

```
┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
│ CPU0 │  │ CPU1 │  │ CPU2 │  │ CPU3 │
│ L1$  │  │ L1$  │  │ L1$  │  │ L1$  │
└──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘
   │         │         │         │
   └────┬────┘         └────┬────┘
     ┌──▼──┐             ┌──▼──┐
     │ L2$ │             │ L2$ │
     └──┬──┘             └──┬──┘
        │                   │
        └─────────┬─────────┘
              ┌───▼───┐
              │  L3$  │  (shared)
              └───┬───┘
              ┌───▼───┐
              │  DRAM  │
              └───────┘
```

Each CPU has its own **run queue** — the scheduler is per-CPU:

```c
DEFINE_PER_CPU_SHARED_ALIGNED(struct rq, runqueues);

/* Each CPU picks tasks from its own run queue */
/* Load balancer periodically migrates tasks between CPUs */
```

### big.LITTLE / DynamIQ (ARM)

```
┌──────────────┐    ┌──────────────┐
│ big cores    │    │ LITTLE cores │
│ (A76, high   │    │ (A55, power  │
│  performance)│    │  efficient)  │
│ CPU4,5,6,7   │    │ CPU0,1,2,3   │
└──────────────┘    └──────────────┘

Scheduler uses Energy Aware Scheduling (EAS):
  - Background tasks → LITTLE cores (save power)
  - Interactive/compute → big cores (performance)
```

---

## 3.7 Hardware Support for Threads

### Hyper-Threading (SMT — Simultaneous Multi-Threading)

```
Physical Core
┌─────────────────────────┐
│  ┌───────┐  ┌───────┐   │
│  │Thread0│  │Thread1│   │  ← Two logical CPUs
│  │Regs   │  │Regs   │   │  ← Separate register files
│  └───┬───┘  └───┬───┘   │
│      │          │        │
│  ┌───▼──────────▼───┐   │
│  │ Shared execution  │   │  ← Shared ALU, cache, TLB
│  │ units (ALU, FPU)  │   │
│  └──────────────────┘   │
└─────────────────────────┘

Linux sees 2 logical CPUs per physical core.
Scheduler is topology-aware: avoids scheduling on
both SMT siblings when other cores are idle.
```

---

## Interview Questions

**Q1: What happens at the hardware level during a context switch?**
A: 1) Timer interrupt fires. 2) CPU saves PC and status to exception registers. 3) Kernel saves current task's callee-saved registers to `task_struct.thread`. 4) Kernel restores next task's registers. 5) Switches page table (CR3/TTBR0). 6) Returns — CPU now executes next task.

**Q2: Why does Linux use per-CPU run queues instead of a global queue?**
A: A global queue requires a global lock — contention with many CPUs. Per-CPU queues are lock-free for the common case (pick from own queue). Load balancing migrates tasks periodically. This is essential for scaling to 100+ CPU systems.

**Q3: What is CONFIG_HZ and how does it affect scheduling?**
A: `CONFIG_HZ` sets the timer tick frequency: 100 (server, low overhead), 250 (desktop default), 1000 (low latency). Higher HZ = more frequent scheduling decisions = lower latency but more timer interrupt overhead.

**Q4: How does tickless mode (NO_HZ) work?**
A: Instead of interrupting the CPU every tick, the kernel programs the timer for the next event (task wakeup, timer expiry). When idle, no ticks fire — CPU stays in deep sleep. `NO_HZ_FULL` extends this to single-task CPUs, beneficial for HPC and RT workloads.

---

*Next: [Chapter 4 — Process Representation in Linux](Chapter_04_Process_Representation.md)*
