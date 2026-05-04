# Chapter 1: Foundations of Processes

## Learning Goals
- Understand what a process is and why it exists
- Distinguish between programs and processes
- Grasp concurrency, multitasking, and why scheduling matters
- Set the stage for all subsequent chapters

---

## 1.1 What Is a Process?

A **process** is an instance of a program **in execution**. It is the fundamental unit of work in an operating system.

```
Program (on disk)              Process (in memory)
┌────────────────┐            ┌────────────────────────┐
│  ELF binary    │  exec()    │  Text (code)           │
│  /usr/bin/ls   │ ────────►  │  Data (globals)        │
│  (passive)     │            │  Heap (malloc)         │
└────────────────┘            │  Stack (local vars)    │
                              │  Kernel metadata       │
                              │   (task_struct)        │
                              └────────────────────────┘
```

A process includes:
| Component | Description |
|-----------|-------------|
| **Address space** | Virtual memory: text, data, heap, stack, mmap regions |
| **Execution state** | CPU registers, program counter, stack pointer |
| **Kernel metadata** | `task_struct`: PID, state, priority, open files, signals |
| **Resources** | Open file descriptors, memory mappings, locks, timers |

---

## 1.2 Difference Between Program and Process

| Attribute | Program | Process |
|-----------|---------|---------|
| Nature | Passive (file on disk) | Active (executing) |
| Storage | Disk (ELF, a.out) | Memory (RAM) |
| Count | One → many processes | One per invocation |
| State | None | Running, sleeping, zombie, etc. |
| Resources | None | CPU time, memory, file descriptors |
| Identity | File path | PID, credentials, namespace |

```bash
# One program, multiple processes
$ vim file1.txt &     # Process 1 (PID 1001)
$ vim file2.txt &     # Process 2 (PID 1002)
# Same program (/usr/bin/vim), two distinct processes
```

---

## 1.3 Process Abstraction in Operating Systems

The OS provides the **illusion** that each process has:
- Its own CPU (scheduling)
- Its own memory (virtual memory)
- Its own I/O (file descriptors)

```
          ┌──────────┐  ┌──────────┐  ┌──────────┐
          │ Process A │  │ Process B │  │ Process C │
          │(believes  │  │(believes  │  │(believes  │
          │ it owns   │  │ it owns   │  │ it owns   │
          │ the CPU)  │  │ the CPU)  │  │ the CPU)  │
          └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
                │              │              │
         ┌──────▼──────────────▼──────────────▼──────┐
         │           Kernel Scheduler                 │
         │     (multiplexes CPU among processes)      │
         └──────────────────┬────────────────────────┘
                            │
                    ┌───────▼───────┐
                    │  Physical CPU  │
                    │  (1-N cores)   │
                    └───────────────┘
```

---

## 1.4 Process Lifecycle Overview

Every process follows this lifecycle:

```
                 fork()/clone()
                      │
                      ▼
              ┌───────────────┐
              │   CREATED     │
              │  (task_struct  │
              │   allocated)  │
              └───────┬───────┘
                      │ wake_up_new_task()
                      ▼
              ┌───────────────┐     schedule()     ┌───────────────┐
         ┌───►│   RUNNABLE    │ ──────────────────► │   RUNNING     │
         │    │  (on run queue)│ ◄────────────────── │ (on CPU)      │
         │    └───────┬───────┘     preempt/yield   └───┬───┬───────┘
         │            │                                  │   │
         │  wake_up() │                     sleep/wait   │   │ exit()
         │            │                                  │   │
         │    ┌───────▼───────┐                          │   │
         └────│   SLEEPING    │ ◄────────────────────────┘   │
              │(waiting for   │                              │
              │ event/IO)     │                              │
              └───────────────┘                              │
                                                             ▼
                                                     ┌───────────────┐
                                                     │   ZOMBIE      │
                                                     │(exited, wait  │
                                                     │ for parent)   │
                                                     └───────┬───────┘
                                                             │ wait()/waitpid()
                                                             ▼
                                                     ┌───────────────┐
                                                     │   DEAD        │
                                                     │  (reclaimed)  │
                                                     └───────────────┘
```

---

## 1.5 Processes vs Threads

In Linux, threads and processes are both represented by `task_struct`. The difference is what they **share**.

| Feature | Process | Thread |
|---------|---------|--------|
| Address space | Own (separate) | Shared with parent |
| File descriptors | Own (separate) | Shared |
| PID | Unique | Unique TID, shared TGID |
| Creation | `fork()` | `clone(CLONE_VM \| CLONE_FILES \| ...)` |
| Context switch cost | Higher (TLB flush) | Lower (same address space) |
| Crash isolation | Independent | One thread crash → all threads crash |

```
Process A (PID 100)              Process B (PID 200)
┌─────────────────┐             ┌─────────────────┐
│ Thread 1 (TID 100) │          │ Thread 1 (TID 200) │
│ Thread 2 (TID 101) │          │                   │
│ Thread 3 (TID 102) │          └─────────────────┘
│                     │           ↑ Separate address space
│ Shared: VM, files,  │
│ signal handlers     │
└─────────────────────┘
```

In Linux, `clone()` is used for both — the `flags` parameter decides what is shared:

```c
/* fork() → clone with separate everything */
clone(SIGCHLD, 0);

/* pthread_create() → clone sharing VM, files, signals */
clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND |
      CLONE_THREAD | CLONE_SYSVSEM, stack);
```

---

## 1.6 Concurrency in Process Execution

**Concurrency**: Multiple processes making progress (interleaved on one CPU or parallel on multiple CPUs).

```
Single CPU (concurrency via time-slicing):
  Time ──────────────────────────────────►
  CPU:  [A][A][B][B][A][C][C][B][A][C]...
        Process A, B, C interleaved

Multi-CPU (true parallelism):
  CPU 0: [A][A][A][A][A]...
  CPU 1: [B][B][B][B][B]...
  CPU 2: [C][C][C][C][C]...
```

Challenges of concurrency:
- **Race conditions**: Two processes modify shared data simultaneously
- **Deadlocks**: Circular wait on resources
- **Starvation**: A process never gets CPU time
- **Priority inversion**: Low-priority task blocks high-priority task

---

## 1.7 Multitasking Concepts

| Type | Description | Example |
|------|-------------|---------|
| **Cooperative** | Tasks voluntarily yield CPU | Windows 3.1, classic Mac OS |
| **Preemptive** | Kernel forcibly switches tasks | Linux, Windows NT+, macOS |
| **Real-time** | Tasks have hard deadlines | QNX, PREEMPT_RT Linux |

Linux uses **preemptive multitasking**:
- The kernel timer interrupt fires (typically every 1-4 ms, configurable via `CONFIG_HZ`)
- The scheduler evaluates whether to switch to a higher-priority or more deserving task
- The running task cannot monopolize the CPU (unless it's a real-time task with `SCHED_FIFO`)

```c
/* Timer interrupt triggers scheduler tick */
void scheduler_tick(void)
{
    struct task_struct *curr = current;
    struct rq *rq = this_rq();

    /* Update runtime statistics */
    curr->sched_class->task_tick(rq, curr, 0);

    /* Check if preemption needed */
    if (need_resched())
        set_tsk_need_resched(curr);
}
```

---

## 1.8 Importance of Scheduling in Operating Systems

Without a scheduler, only one process could run at a time. The scheduler is critical for:

| Goal | How Scheduler Achieves It |
|------|--------------------------|
| **Fairness** | CFS gives each task proportional CPU time |
| **Responsiveness** | Interactive tasks get low latency |
| **Throughput** | Batch tasks get maximum CPU utilization |
| **Real-time guarantees** | RT scheduler ensures deadline compliance |
| **Energy efficiency** | Idle CPU cores enter low-power states |
| **SMP scalability** | Load balancing across cores |

```
Scheduling Impact on User Experience:
                                                    
  Without good scheduler:     With good scheduler:  
  ┌──────────────────────┐    ┌──────────────────────┐
  │ Mouse: ........lag    │    │ Mouse: instant ✓     │
  │ Video: stutter        │    │ Video: smooth ✓      │
  │ Build: variable       │    │ Build: fast ✓        │
  │ UI: freezes           │    │ UI: responsive ✓     │
  └──────────────────────┘    └──────────────────────┘
```

---

## Kernel Source References

| File | Content |
|------|---------|
| `include/linux/sched.h` | `task_struct` definition — the process descriptor |
| `kernel/fork.c` | Process creation (`kernel_clone()`, `copy_process()`) |
| `kernel/sched/core.c` | Scheduler core (`schedule()`, `scheduler_tick()`) |
| `kernel/exit.c` | Process exit and zombie handling |
| `fs/proc/base.c` | `/proc/[pid]/` interface |

---

## OS Comparison

| Concept | Linux | Windows | QNX | macOS (XNU) |
|---------|-------|---------|-----|-------------|
| Process descriptor | `task_struct` | `EPROCESS` | `PROCESS` | `proc` + `task` (Mach) |
| Thread descriptor | `task_struct` (shared VM) | `ETHREAD` | `THREAD` | `thread` (Mach) |
| Minimum entity | Thread = lightweight process | Thread | Thread | Mach thread |
| Idle task | PID 0 (`swapper`) | System Idle Process | idle thread | kernel_task |

---

## Interview Questions

**Q1: What is the difference between a process and a program?**
A: A program is a passive file on disk (ELF binary). A process is an active instance of that program in memory — with its own address space, CPU state, PID, and resources. One program can have many processes.

**Q2: Why does Linux represent threads as `task_struct` rather than a separate structure?**
A: Linux uses a unified `task_struct` for both processes and threads. The `clone()` flags control what's shared (VM, files, signals). This simplifies the kernel — the scheduler treats all tasks uniformly, and threads are just processes that share more resources.

**Q3: What is the difference between concurrency and parallelism?**
A: Concurrency: multiple tasks making progress (possibly interleaved on one CPU). Parallelism: multiple tasks executing simultaneously on different CPUs. Concurrency is a software concept; parallelism requires hardware support (multiple cores).

**Q4: Why is preemptive multitasking better than cooperative?**
A: In cooperative multitasking, a buggy program that doesn't yield freezes the entire system. Preemptive multitasking lets the kernel forcibly switch tasks (via timer interrupt), ensuring no single process can monopolize the CPU.

---

## Summary

- A process is a program in execution with its own address space, CPU state, and resources
- Linux represents both processes and threads as `task_struct` — the distinction is what resources are shared
- The scheduler multiplexes the CPU among processes, providing fairness, responsiveness, and throughput
- Preemptive multitasking ensures the kernel maintains control regardless of process behavior

---

*Next: [Chapter 2 — History and Evolution of Process Management](Chapter_02_History_and_Evolution.md)*
