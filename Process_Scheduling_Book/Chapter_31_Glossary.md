# Chapter 31: Glossary and Definitions

## Learning Goals
- Quick reference for all process management and scheduling terminology
- Precise definitions with kernel context
- Cross-references to relevant chapters

---

## A

**Active Balancing**: Forced migration of tasks from overloaded CPUs. Triggered when periodic load balancing finds significant imbalance. (Ch 16)

**Affinity (CPU)**: Bitmask of CPUs a task is allowed to run on. Set via `sched_setaffinity()` or `taskset`. Restricts scheduler's CPU choices. (Ch 16)

**ASID (Address Space ID)**: Hardware tag in TLB entries (ARM64) that identifies which address space an entry belongs to. Avoids TLB flush on context switch. x86 equivalent: PCID. (Ch 10, 25)

**Atomic Operation**: CPU instruction that completes without interruption (e.g., `atomic_inc`, `cmpxchg`). Used for lock-free synchronization. (Ch 19)

**Autogroup**: Automatic per-TTY task grouping (`CONFIG_SCHED_AUTOGROUP`). Groups tasks by terminal session for better desktop responsiveness. (Ch 17)

---

## B

**Bandwidth Control**: CFS mechanism to limit CPU time per cgroup. Defined by `cpu.cfs_quota_us` / `cpu.cfs_period_us`. Tasks throttled when quota exhausted. (Ch 15, 23)

**Big.LITTLE**: ARM heterogeneous CPU architecture with power-efficient (little) and high-performance (big) cores. Managed by EAS in Linux. (Ch 16)

**Binder**: Android IPC mechanism with priority inheritance. Used for cross-process calls between apps and system services. (Ch 20)

---

## C

**CBS (Constant Bandwidth Server)**: Algorithm used by SCHED_DEADLINE to limit task CPU consumption to declared bandwidth (runtime/period). (Ch 14)

**CFS (Completely Fair Scheduler)**: Default Linux scheduler for normal tasks (SCHED_NORMAL). Uses vruntime and red-black tree. Introduced in 2.6.23. (Ch 13)

**cgroup**: Control group — kernel mechanism for organizing processes into hierarchical groups and applying resource limits (CPU, memory, I/O). (Ch 15, 24)

**cond_resched()**: Voluntary preemption point. Kernel function that checks TIF_NEED_RESCHED and calls schedule() if set. Used in long kernel paths. (Ch 18)

**Context Switch**: Saving one task's CPU state and restoring another's. Includes register save/restore, page table switch, and TLB management. Cost: 2-60µs. (Ch 10)

**Copy-on-Write (COW)**: Memory optimization for fork(). Parent and child share pages read-only. Pages copied only when written. (Ch 7)

**Core Dump**: Memory image written to file when process receives fatal signal (SIGSEGV, SIGABRT, etc.). Used for post-mortem debugging. (Ch 21)

**CPU Migration**: Moving a task from one CPU's run queue to another for load balancing. Costs cache warmth. (Ch 16)

**current**: Macro that returns pointer to currently executing task's `task_struct`. ARM64: from SP_EL0. x86: from per-CPU GS segment. (Ch 4)

---

## D

**D State**: TASK_UNINTERRUPTIBLE — process sleeping in kernel, not interruptible by signals. Often waiting for I/O. Visible as 'D' in ps. (Ch 5)

**Deadline Scheduling**: SCHED_DEADLINE policy using EDF algorithm. Tasks specify runtime, deadline, and period. Highest priority class after stop. (Ch 14)

**Deferred Work**: Kernel mechanisms for postponing work: softirqs, tasklets, workqueues, threaded IRQs. (Ch 9)

---

## E

**EAS (Energy Aware Scheduling)**: Scheduler extension for heterogeneous CPUs. Places tasks on the most energy-efficient CPU that meets performance needs. (Ch 16)

**EDF (Earliest Deadline First)**: Scheduling algorithm that always runs the task with the nearest absolute deadline. Used by SCHED_DEADLINE. (Ch 14)

**EEVDF (Earliest Eligible Virtual Deadline First)**: Replacement for CFS's pick-next logic in Linux 6.6+. Adds virtual deadlines for better latency fairness. (Ch 13)

**Effective Priority (prio)**: The actual scheduling priority used by the kernel. May differ from normal_prio during priority inheritance. (Ch 17)

---

## F

**Fair Scheduling**: Proportional CPU sharing where tasks get CPU time proportional to their weight. CFS implements this via vruntime. (Ch 13)

**FIFO (SCHED_FIFO)**: Real-time scheduling policy. Highest priority runs until it blocks, yields, or is preempted by higher priority. No timeslice. (Ch 14)

**ftrace**: Linux kernel tracing framework. Traces function calls, scheduling events, interrupts. Interface: `/sys/kernel/debug/tracing/`. (Ch 28)

**Futex**: Fast user-space mutex. Uncontended case handled in user space (no syscall). Contended case uses kernel for sleep/wake. (Ch 20)

---

## G

**Grace Period (RCU)**: Time after RCU pointer update during which all pre-existing RCU read-side critical sections must complete. After grace period, old data can be freed. (Ch 19)

**Group Scheduling**: CFS feature where scheduling entities can represent task groups (cgroups). Groups compete for CPU as single entities. (Ch 15)

---

## H

**Huge Pages**: Memory pages larger than default 4KB (2MB or 1GB). Reduce TLB misses and page table overhead. (Ch 25)

---

## I

**Idle Class**: Lowest priority scheduling class. Runs swapper/idle thread when no other task is runnable. (Ch 11)

**IPC (Inter-Process Communication)**: Mechanisms for processes to exchange data: pipes, message queues, shared memory, sockets, signals. (Ch 20)

---

## J

**Jiffies**: Kernel tick counter. Incremented every 1/HZ seconds. Used for timekeeping and timeout management. (Ch 3)

**Job Control**: Terminal-based process management. Foreground/background groups, signals (SIGTSTP, SIGCONT), controlled by shell. (Ch 22)

---

## K

**Kernel Stack**: Per-task stack used in kernel context. Typically 8KB or 16KB. Contains saved registers, function call frames, pt_regs. (Ch 4)

**Kernel Thread**: Task with mm=NULL that runs only in kernel space. Created via kthread_create()/kthread_run(). Examples: kswapd, kworker, ksoftirqd. (Ch 9)

**kthread**: See Kernel Thread. (Ch 9)

---

## L

**Load Average**: System-wide metric (1, 5, 15 min). Exponentially decaying average of runnable + uninterruptible tasks. Visible in `/proc/loadavg`. (Ch 24)

**Load Balancing**: Periodic redistribution of tasks across CPUs. Happens at each scheduling domain level (SMT, MC, NUMA). (Ch 16)

**Lockdep**: Kernel lock dependency checker (`CONFIG_LOCKDEP`). Detects potential deadlocks at runtime by tracking lock acquisition order. (Ch 19)

---

## M

**Migration Thread**: Per-CPU kernel thread (migration/N) at highest RT priority. Handles forced task migration for CPU hotplug and active balancing. (Ch 16)

**min_vruntime**: Monotonically increasing floor in CFS run queue. Used as anchor for new task placement and cross-CPU migration. (Ch 13, 15)

**Mutex**: Sleeping lock for process context. Allows only one holder. Sleeps when contended (unlike spinlocks which spin). (Ch 19)

---

## N

**Nice Value**: User-space priority for normal tasks. Range: -20 (highest) to +19 (lowest). Maps to kernel priority 100-139 and CFS weight. (Ch 17)

**NO_HZ**: Tickless kernel mode. NO_HZ_IDLE: stop ticks on idle CPUs. NO_HZ_FULL: stop ticks even with one task running. (Ch 25)

**Normal Priority (normal_prio)**: Base effective priority computed from static_prio (normal tasks) or rt_priority (RT tasks). Equals prio unless PI-boosted. (Ch 17)

**NPTL (Native POSIX Thread Library)**: Linux's 1:1 threading model. Each pthread maps to one kernel task (clone with CLONE_THREAD). (Ch 8)

**NUMA (Non-Uniform Memory Access)**: Architecture where memory access time depends on CPU-memory distance. Scheduling tries to keep tasks near their memory. (Ch 16)

---

## O

**O(1) Scheduler**: Linux scheduler from 2.6.0-2.6.22. Used priority arrays with O(1) task selection. Replaced by CFS due to heuristic issues. (Ch 2)

**Orphan Process**: Process whose parent has exited. Reparented to init (PID 1) or nearest subreaper. (Ch 4, 6)

---

## P

**PCID (Process Context ID)**: x86 TLB tag (12-bit). Same concept as ARM64 ASID. Avoids TLB flush on context switch. (Ch 10, 25)

**PELT (Per-Entity Load Tracking)**: Scheduler mechanism that tracks CPU utilization per scheduling entity with exponential decay (~32ms half-life). (Ch 15)

**Preemption**: OS forcibly taking CPU from running task. User preemption always enabled. Kernel preemption depends on CONFIG_PREEMPT model. (Ch 18)

**PREEMPT_RT**: Real-time preemption patchset. Converts spinlocks to RT mutexes, threads all IRQs, achieves ~50-100µs worst-case latency. (Ch 14, 18)

**Priority Inheritance (PI)**: Mechanism where lock-holding low-priority task temporarily inherits higher-priority waiter's priority. Prevents priority inversion. (Ch 17, 19)

**Priority Inversion**: When medium-priority task indirectly blocks high-priority task by preempting low-priority lock holder. Solved by PI. (Ch 17)

**Process Group**: Collection of related processes (e.g., shell pipeline). Share PGID. Receive signals together for job control. (Ch 22)

---

## R

**RCU (Read-Copy-Update)**: Scalable synchronization for read-mostly data. Readers have zero overhead. Writers copy, update pointer, wait for grace period. (Ch 19)

**Red-Black Tree**: Self-balancing binary search tree used by CFS. Tasks keyed by vruntime. O(log n) insert/remove, O(1) min (cached leftmost). (Ch 13)

**Runqueue (rq)**: Per-CPU data structure containing all scheduling state. Holds cfs_rq, rt_rq, dl_rq, current task, idle task. (Ch 11, 15)

**Round Robin (SCHED_RR)**: RT scheduling policy. Same as SCHED_FIFO but with timeslice for tasks at same priority level. Default: 100ms. (Ch 14)

---

## S

**SCHED_BATCH**: CFS policy for CPU-bound batch jobs. Same as SCHED_NORMAL but without wakeup preemption bonus. (Ch 23)

**SCHED_DEADLINE**: Deadline scheduling policy using EDF+CBS. Parameters: runtime, deadline, period. (Ch 14, 23)

**SCHED_FIFO**: Real-time FIFO policy. No timeslice. Runs until blocks, yields, or preempted by higher priority. (Ch 14, 23)

**SCHED_IDLE**: Lowest priority CFS policy. Tasks run only when no SCHED_NORMAL/BATCH tasks want CPU. (Ch 23)

**SCHED_NORMAL (SCHED_OTHER)**: Default scheduling policy. Uses CFS with proportional fair sharing based on nice/weight. (Ch 23)

**SCHED_RR**: Real-time round-robin policy. Same as SCHED_FIFO with time-slicing at same priority. (Ch 14, 23)

**Scheduling Class**: Modular scheduler component. Chain: stop → dl → rt → fair → idle. Each implements struct sched_class. (Ch 11)

**Scheduling Domain**: Hierarchical representation of CPU topology for load balancing. Levels: SMT, MC (multi-core), NUMA. (Ch 15, 16)

**sched_entity**: Per-task (or per-group) CFS scheduling data. Contains vruntime, load weight, RB tree node, PELT averages. (Ch 15)

**Session**: Group of process groups sharing a controlling terminal. Created by setsid(). Session leader: the process that called setsid(). (Ch 22)

**Signal**: Asynchronous notification to a process. Standard (1-31) and real-time (32-64). Delivered on return to user space. (Ch 21)

**Spinlock**: Busy-wait lock. CPU spins until lock available. Used for short critical sections, valid in any context. (Ch 19)

**Static Priority (static_prio)**: Priority set by nice value. Range 100-139. Doesn't change due to PI or scheduling. (Ch 17)

**Stop Class**: Highest priority scheduling class (internal). Used for migration threads and stop_machine callbacks. (Ch 11)

---

## T

**task_struct**: The fundamental process descriptor in Linux. Contains all process state: scheduling, memory, files, signals, hierarchy (~6-8KB). (Ch 4)

**TGID (Thread Group ID)**: Shared by all threads in a process. TGID = main thread's PID. getpid() returns TGID. (Ch 4)

**TIF_NEED_RESCHED**: Thread flag indicating rescheduling should happen at next opportunity. Set by scheduler, checked on interrupt/syscall return. (Ch 18)

**TIF_SIGPENDING**: Thread flag indicating pending signals. Checked on return to user space. (Ch 21)

**TLB (Translation Lookaside Buffer)**: CPU cache for virtual→physical address translations. Flushed or tagged on context switch. (Ch 10, 25)

**Throttling**: CPU bandwidth enforcement. CFS bandwidth control pauses (throttles) tasks/groups that exceed their quota. RT throttling reserves CPU for normal tasks. (Ch 14, 15)

---

## V

**vruntime (Virtual Runtime)**: CFS metric tracking weighted CPU time consumed. Lower weight (higher nice) accumulates vruntime faster → gets less CPU. (Ch 13)

**Voluntary Context Switch (nvcsw)**: Context switch where the task chose to sleep (e.g., I/O wait, lock). Counts in task_struct->nvcsw. (Ch 24)

---

## W

**Wait Queue**: Kernel mechanism for sleeping tasks waiting for events. Tasks added to wait queue, woken when event occurs. (Ch 19)

**Weight**: CFS task load based on nice value. Nice 0 = 1024. Each nice level ≈ 10% CPU difference. (Ch 13, 17)

**Workqueue**: Kernel mechanism for deferring work to kernel thread context. Uses kworker threads from thread pool. (Ch 9)

---

## Z

**Zombie (Z)**: Process that has exited but whose exit status hasn't been collected by parent via wait(). task_struct retained until wait(). (Ch 5, 6)

---

*Next: [Chapter 32 — OS Comparison](Chapter_32_OS_Comparison.md)*
