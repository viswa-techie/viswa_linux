# Chapter 35: Interview Preparation — Process Management & Scheduling

## Learning Goals
- Confidently answer process management questions at any interview level
- Demonstrate deep kernel knowledge with code-level answers
- Handle scenario-based and debugging questions
- Cover embedded, Android, and real-time scheduling interview topics

---

## 35.1 Basic Level Questions

### Q1: What is a process? How is it different from a program?

```
A program is a passive entity — an executable file on disk (ELF binary).
A process is an active entity — a running instance of a program with:
  - Address space (code, data, heap, stack)
  - Kernel state (task_struct)
  - One or more threads of execution
  - Open file descriptors, signal handlers, etc.

Multiple processes can run the same program simultaneously.
Each gets its own task_struct, PID, and address space.

Key kernel structure: struct task_struct (include/linux/sched.h)
  ~700+ fields, ~8 KB per instance
```

### Q2: What are the different process states in Linux?

```
┌───────────────────────────────────────────────────────────────┐
│  State              │ Macro                │ Meaning           │
│─────────────────────┼──────────────────────┼───────────────────│
│  Running            │ TASK_RUNNING (0)     │ On CPU or runqueue│
│  Interruptible Sleep│ TASK_INTERRUPTIBLE(1)│ Waiting, signal OK│
│  Uninterruptible    │ TASK_UNINTERRUPTIBLE │ Waiting, no signal│
│  Stopped            │ TASK_STOPPED         │ SIGSTOP/ptrace    │
│  Traced             │ TASK_TRACED          │ Debugger attached │
│  Zombie             │ EXIT_ZOMBIE          │ Exited, not reaped│
│  Dead               │ EXIT_DEAD            │ Final removal     │
│  Killable           │ TASK_KILLABLE        │ Uninterruptible   │
│                     │                      │ but SIGKILL OK    │
│  Idle               │ TASK_IDLE            │ Idle process sleep│
└───────────────────────────────────────────────────────────────┘

TASK_KILLABLE = TASK_UNINTERRUPTIBLE | TASK_WAKEKILL
Introduced to solve unkillable processes in D state.
```

### Q3: Difference between fork(), vfork(), and clone()?

```
fork():
  - Creates child as copy of parent
  - Copy-on-write (COW) for memory efficiency
  - Child gets new PID, own address space (virtual)
  - Implemented via kernel_clone() with minimal flags

vfork():
  - Child SHARES parent's address space temporarily
  - Parent is BLOCKED until child calls exec() or _exit()
  - Avoids page table copy — faster for exec-only patterns
  - CLONE_VM | CLONE_VFORK flags

clone():
  - Fine-grained control via flags
  - Can share: VM, files, signal handlers, namespace
  - Used to create threads (CLONE_VM | CLONE_FS | CLONE_FILES |
    CLONE_SIGHAND | CLONE_THREAD)
  - Foundation for both fork() and pthread_create()
  - clone3() is the modern version with extensible struct

All three call kernel_clone() internally (kernel/fork.c).
```

### Q4: What is a zombie process? How to prevent it?

```
Zombie: Process that has exited but parent hasn't called wait().
  - Retains task_struct and PID (for exit status)
  - Consumes minimal resources (no memory pages, no CPU)
  - But consumes a PID slot

State: EXIT_ZOMBIE → parent calls wait() → EXIT_DEAD → freed

Prevention:
1. Parent calls wait()/waitpid() to reap child
2. Signal handler: signal(SIGCHLD, SIG_IGN)
   → Kernel auto-reaps (SA_NOCLDWAIT)
3. Double-fork trick: child forks grandchild, child exits
   → Grandchild adopted by init, init always reaps
4. sigaction with SA_NOCLDWAIT flag

Find zombies: ps aux | grep Z
              cat /proc/<pid>/status | grep State
```

### Q5: What is context switching?

```
Context switch = saving state of current process and loading
state of next process so CPU can execute it.

What gets saved/restored:
  1. CPU registers (general purpose, FP/SIMD)
  2. Program counter (instruction pointer)
  3. Stack pointer
  4. Page table base register (switching address space)
  5. FPU/SIMD state (lazy or eager)
  6. TLS (thread-local storage) registers

Cost: 2-5 µs (same mm) to 20-60 µs (different mm + cache cold)

Kernel path:
  schedule() → __schedule() → context_switch()
    ├── switch_mm_irqs_off()    ← address space switch
    └── switch_to()             ← register + stack switch

ARM64: cpu_switch_to (arch/arm64/kernel/entry.S)
x86:   __switch_to_asm (arch/x86/entry/entry_64.S)
```

### Q6: What is the difference between thread and process?

```
                    Process         Thread
─────────────────────────────────────────────────
Address space       Own (separate)  Shared with group
Creation cost       Higher (COW)    Lower (no mm copy)
Context switch      Expensive       Cheaper (same mm)
Communication       IPC needed      Direct memory access
Crash isolation     Independent     Crash kills all
Kernel view         task_struct     task_struct (same!)

Linux key insight: Kernel treats both as task_struct.
A "thread" is a task that shares mm, files, sighand
with other tasks in the same thread_group.

pthread_create() → clone(CLONE_VM | CLONE_FS | 
  CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD)

Linux uses 1:1 threading model (NPTL):
  Each user thread = 1 kernel task_struct
```

---

## 35.2 Intermediate Level Questions

### Q7: Explain the Completely Fair Scheduler (CFS).

```
CFS Goal: Give each task a fair share of CPU proportional
to its weight (derived from nice value).

Core concept: Virtual Runtime (vruntime)
  vruntime += actual_runtime × (NICE_0_LOAD / task_weight)

  Higher weight (lower nice) → vruntime grows SLOWER
  → Gets MORE actual CPU time

Data structure: Red-Black tree ordered by vruntime
  ┌─────────────────────────────────┐
  │     Pick task with LOWEST       │
  │     vruntime (leftmost node)    │
  │            [5ms]                │
  │           /     \               │
  │        [3ms]   [8ms]  ← rb_leftmost cached
  │        /                        │
  │     [1ms] ← NEXT TO RUN        │
  └─────────────────────────────────┘

Timeslice: Not fixed! Calculated as:
  slice = sched_latency × (task_weight / total_weight_of_runqueue)

Tunables:
  /proc/sys/kernel/sched_latency_ns        = 6ms (default)
  /proc/sys/kernel/sched_min_granularity_ns = 0.75ms
  /proc/sys/kernel/sched_wakeup_granularity_ns = 1ms

Linux 6.6+: EEVDF (Earliest Eligible Virtual Deadline First)
  Adds virtual deadline to improve latency fairness.
  Tasks with shorter requests get lower latency.
```

### Q8: What is priority inversion? How does Linux solve it?

```
Priority Inversion:
  High-priority task H blocked on lock held by low-priority L.
  Medium-priority task M preempts L.
  Result: H waits for M (effective priority flip!)

  Timeline:
    L grabs mutex → M preempts L → H arrives, needs mutex
    H blocked on L, but L can't run because M is using CPU
    H effectively has LOWER priority than M!

Solutions in Linux:

1. Priority Inheritance (PI) — rt_mutex
   When H blocks on mutex held by L:
     L's priority is temporarily BOOSTED to H's priority
     L runs, releases mutex, priority restored
     H gets the mutex immediately

   Kernel implementation: kernel/locking/rtmutex.c
   rt_mutex_adjust_prio() walks the lock chain

2. PREEMPT_RT: ALL spinlocks become rt_mutex
   → PI everywhere, not just for rt_mutex users

Mars Pathfinder (1997): Famous PI bug
  Watchdog timer kept resetting due to PI on VxWorks.
  Fixed by enabling PI protocol.
```

### Q9: Explain Linux scheduling classes and their priority.

```
Linux uses a hierarchy of scheduling classes:

  Highest priority
    │
    ▼
  ┌─────────────────────┐
  │  stop_sched_class    │ ← Migration/stop tasks (highest)
  ├─────────────────────┤
  │  dl_sched_class      │ ← SCHED_DEADLINE (EDF)
  ├─────────────────────┤
  │  rt_sched_class      │ ← SCHED_FIFO, SCHED_RR
  ├─────────────────────┤
  │  fair_sched_class    │ ← SCHED_NORMAL, SCHED_BATCH
  ├─────────────────────┤
  │  idle_sched_class    │ ← SCHED_IDLE, per-CPU idle
  └─────────────────────┘
    │
    ▼
  Lowest priority

Each class implements struct sched_class with callbacks:
  .enqueue_task, .dequeue_task, .pick_next_task,
  .put_prev_task, .task_tick, .check_preempt_curr

pick_next_task() iterates classes top-down:
  for_each_class(class) {
      p = class->pick_next_task(rq);
      if (p) return p;
  }

Priority ranges:
  0       = stop/deadline (internally)
  1-99    = RT priorities (SCHED_FIFO/RR)
  100-139 = Normal priorities (nice -20 to +19)
```

### Q10: What is preemption? What are Linux's preemption models?

```
Preemption = involuntarily taking CPU from a running task
to give it to a higher-priority task.

Linux preemption models (Kconfig):

1. PREEMPT_NONE
   - No kernel preemption
   - Only preempt at return to user space
   - Best throughput (servers, HPC)

2. PREEMPT_VOLUNTARY
   - Adds explicit preemption points: might_sleep()
   - ~200 check points in kernel
   - Compromise: decent latency, good throughput
   - Desktop/general use

3. PREEMPT (full kernel preemption)
   - Preempt anywhere except when preempt_count > 0
   - (spinlock held, IRQ disabled, BH disabled)
   - Best latency for non-RT
   - Low-latency audio, embedded

4. PREEMPT_RT (Real-Time patch)
   - spinlock → rt_mutex (preemptible)
   - IRQ handlers → threaded (preemptible)
   - Worst-case latency: ~10-50 µs
   - Hard real-time applications

Key mechanism: TIF_NEED_RESCHED flag
  Set by: scheduler_tick(), try_to_wake_up()
  Checked at: return from interrupt, preemption points,
              spin_unlock (if PREEMPT)
```

### Q11: How does load balancing work on multi-core Linux?

```
Load balancing ensures tasks are spread across CPUs.

Architecture: Scheduling Domains (hierarchical)
  ┌────────────────────────────────────────┐
  │              NUMA Domain               │
  │  ┌──────────────┐  ┌──────────────┐   │
  │  │  MC Domain   │  │  MC Domain   │   │
  │  │ ┌────┐┌────┐ │  │ ┌────┐┌────┐ │   │
  │  │ │CPU0││CPU1│ │  │ │CPU2││CPU3│ │   │
  │  │ └────┘└────┘ │  │ └────┘└────┘ │   │
  │  └──────────────┘  └──────────────┘   │
  └────────────────────────────────────────┘

Triggers:
  1. Periodic (scheduler_tick → trigger_load_balance)
     Every sd->balance_interval (typically 4-64ms)
  2. Idle (newidle_balance — CPU runs out of work)
  3. Fork/Exec (place new task on least loaded CPU)
  4. Wake-up (select_task_rq_fair → select_idle_sibling)

Algorithm:
  find_busiest_group() → find_busiest_queue()
    → detach_tasks() → attach_tasks()

PELT (Per-Entity Load Tracking):
  Each entity tracks: load_avg, util_avg, runnable_avg
  Geometric decay: y = 1/2^(1/32ms) ≈ 0.978
  Used for accurate load comparison between CPUs

EAS (Energy Aware Scheduling) on ARM big.LITTLE:
  find_energy_efficient_cpu() considers power cost
  Prefers efficient (LITTLE) CPUs for light tasks
```

### Q12: What happens when you call fork()?

```
User space: fork() → glibc wrapper → syscall

Kernel path (kernel/fork.c):
  sys_fork()
    → kernel_clone(SIGCHLD, 0, 0, NULL, NULL)
      → copy_process()
          ├── dup_task_struct()      ← allocate new task_struct
          ├── copy_creds()           ← copy credentials
          ├── sched_fork()           ← initialize scheduling
          │   ├── Set state = TASK_NEW
          │   ├── Set prio = parent's normal_prio
          │   ├── Assign fair_sched_class
          │   └── init sched_entity (se.vruntime = 0)
          ├── copy_files()           ← dup file descriptor table
          ├── copy_mm()              ← copy page tables (COW)
          │   └── dup_mm() → dup_mmap()
          ├── copy_thread()          ← set child's kernel stack
          │   └── pt_regs: set return value = 0 for child
          ├── alloc_pid()            ← assign new PID
          └── return child task_struct
      → wake_up_new_task(child)
          ├── activate_task()        ← enqueue on runqueue
          ├── place_entity()         ← set initial vruntime
          │   └── vruntime = cfs_rq->min_vruntime (approx)
          └── check_preempt_curr()   ← may preempt parent

Return:
  Parent: returns child's PID
  Child:  returns 0 (set in copy_thread → pt_regs)
```

### Q13: Explain SCHED_DEADLINE.

```
SCHED_DEADLINE implements Earliest Deadline First (EDF)
with Constant Bandwidth Server (CBS) for bandwidth isolation.

Three parameters per task:
  Runtime  (worst-case execution time per period)
  Deadline (relative deadline within period)  
  Period   (how often the task activates)

  Constraint: Runtime ≤ Deadline ≤ Period

Example: Audio processing every 5ms needs 1ms of CPU:
  sched_attr.sched_runtime  = 1,000,000   (1ms)
  sched_attr.sched_deadline = 5,000,000   (5ms)
  sched_attr.sched_period   = 5,000,000   (5ms)

Admission control:
  Σ(Runtime_i / Period_i) ≤ total CPU capacity
  Rejected if overcommitted → EBUSY

Priority: HIGHER than SCHED_FIFO/RR
  dl_sched_class sits above rt_sched_class

Data structure: dl_rq uses RB tree ordered by absolute deadline
  Earliest deadline → runs first

CBS: If task exceeds runtime budget in current period,
     deadline is pushed to next period (throttled).
     Prevents one task from starving others.
```

---

## 35.3 Advanced Level Questions

### Q14: Walk through __schedule() step by step.

```c
/* kernel/sched/core.c — simplified */
static void __schedule(unsigned int sched_mode)
{
    struct rq *rq;
    struct task_struct *prev, *next;
    unsigned long *switch_count;

    /* 1. Get current CPU's runqueue and current task */
    rq = cpu_rq(smp_processor_id());
    prev = rq->curr;

    /* 2. Handle previous task's state */
    if (!preempt && prev_state) {
        if (signal_pending_state(prev_state, prev)) {
            WRITE_ONCE(prev->__state, TASK_RUNNING);
        } else {
            deactivate_task(rq, prev, DEQUEUE_SLEEP);
            /* task removed from runqueue */
        }
    }

    /* 3. Pick next task using scheduling class hierarchy */
    next = pick_next_task(rq, prev, &rf);
    /*
     * Iterates: stop → dl → rt → fair → idle
     * Fast path: if all tasks are CFS, skip hierarchy
     * pick_next_task_fair() → pick_next_entity()
     *   → __pick_first_entity() (leftmost in RB tree)
     */

    /* 4. If different task selected, do context switch */
    if (likely(prev != next)) {
        rq->nr_switches++;
        RCU_INIT_POINTER(rq->curr, next);

        /* 5. Context switch: save prev, load next */
        context_switch(rq, prev, next, &rf);
        /*
         * switch_mm_irqs_off() → load next->mm page tables
         * switch_to(prev, next, prev) → save/restore registers
         * After switch_to, we ARE next — running on next's stack
         */
    }
    /* If prev == next, just return (no switch needed) */
}
```

### Q15: Explain PELT (Per-Entity Load Tracking) in detail.

```
PELT tracks three signals for each scheduling entity:

1. load_avg    → demand (weight × runnable fraction)
2. util_avg    → utilization (fraction of CPU used)  
3. runnable_avg → runnable time fraction

Mathematical model:
  Geometric series with half-life ≈ 32ms

  load_sum = Σ (load_i × y^i)
  where y = (2^32 - 89)/(2^32) ≈ 0.97857...
  and each period = 1024 µs (≈ 1ms)

  load_avg = load_sum / divider
  divider = LOAD_AVG_MAX - 1024 + running_period

When task is running:
  util_sum += delta_time   (capped at 1024 per period)
  util_avg = util_sum / LOAD_AVG_MAX

When task is sleeping:
  All signals decay: signal *= y^(periods_sleeping)

Why PELT over instantaneous:
  - Smoother: avoids oscillation on bursty workloads
  - Hierarchical: entities aggregate up (task → cfs_rq → rq)
  - Predictive: captures recent history, not just now
  - Used by: load_balance(), find_energy_efficient_cpu(),
             schedutil governor (CPU frequency selection)

Update path:
  update_curr() → __update_load_avg_se() → 
    ___update_load_sum() + ___update_load_avg()
```

### Q16: How does try_to_wake_up() work?

```c
/* Simplified path of try_to_wake_up() */

int try_to_wake_up(struct task_struct *p, 
                   unsigned int state, int wake_flags)
{
    /* 1. Check if task is in the expected sleep state */
    if (!(READ_ONCE(p->__state) & state))
        return 0;  /* Not sleeping or wrong state */

    /* 2. Mark task as TASK_WAKING (prevent races) */
    WRITE_ONCE(p->__state, TASK_WAKING);

    /* 3. Select target CPU for wakeup */
    cpu = select_task_rq(p, p->wake_cpu, wake_flags);
    /*
     * For CFS: select_task_rq_fair()
     *   → want_affine? select_idle_sibling()
     *   → find_idlest_cpu() if no affine idle
     * For RT: select_task_rq_rt()
     *   → find lowest-priority CPU to push to
     */

    /* 4. If target CPU != current CPU, migrate */
    if (cpu != task_cpu(p)) {
        set_task_cpu(p, cpu);
        /* Update PELT: migrate_task_rq_fair() */
    }

    /* 5. Enqueue task on target CPU's runqueue */
    ttwu_queue(p, cpu, wake_flags);
    /* → activate_task() → enqueue_task() */
    
    /* 6. Check if woken task should preempt current */
    check_preempt_curr(rq, p, wake_flags);
    /* For CFS: if p->vruntime < curr->vruntime - gran
     *   → set TIF_NEED_RESCHED on curr
     */

    /* 7. Set task to TASK_RUNNING */
    WRITE_ONCE(p->__state, TASK_RUNNING);

    return 1;
}
```

### Q17: Explain the scheduling domain hierarchy and its purpose.

```
Scheduling domains model CPU topology for load balancing:

Physical topology example (2-socket, 4-core, 2-HT each):
  Socket 0                    Socket 1
  ┌─────────────────────┐    ┌─────────────────────┐
  │ Core0   Core1       │    │ Core4   Core5       │
  │ ┌─┬─┐  ┌─┬─┐       │    │ ┌─┬─┐  ┌─┬─┐       │
  │ │0│1│  │2│3│ Core2  │    │ │8│9│  │A│B│ Core6  │
  │ └─┴─┘  └─┴─┘ ...   │    │ └─┴─┘  └─┴─┘ ...   │
  └─────────────────────┘    └─────────────────────┘

Domain hierarchy (bottom-up):
  Level 0: SMT domain   (HT siblings — CPU 0,1)
  Level 1: MC domain    (cores sharing LLC — all socket CPUs)
  Level 2: NUMA domain  (cross-socket)

Each level has:
  struct sched_domain {
      .balance_interval    /* how often to balance */
      .imbalance_pct       /* threshold to migrate */
      .flags               /* SD_BALANCE_WAKE, etc. */
      .groups              /* sched_group ring */
  };

Why hierarchical?
  - SMT: balance every 2ms (cheap — shared cache)
  - MC:  balance every 4ms (moderate cost)
  - NUMA: balance every 64ms (expensive — remote memory)

Key principle: prefer local balancing, escalate only when
local domain is balanced but system is not.
```

### Q18: What is RCU and why is it important for scheduling?

```
RCU (Read-Copy-Update): Lock-free synchronization mechanism.
  - Readers: ZERO overhead (no locks, no atomics)
  - Writers: Create new version, wait for readers to finish,
             then free old version

Critical for scheduler performance because:
  1. Reading task lists must be fast (ps, /proc, signals)
  2. Scheduling decisions shouldn't block on reader locks
  3. Hot paths (schedule, wakeup) need minimal overhead

RCU in scheduler:
  - task->rcu used for safe task_struct access
  - rcu_read_lock() protects task list traversal
  - Task freeing delayed: call_rcu(&task->rcu, delayed_free)
  
  Read-side:        Write-side:
  rcu_read_lock()   Modify data
  access data       synchronize_rcu() — wait for readers
  rcu_read_unlock() Free old data

Grace period:
  ┌──────────────────────────────────────────────┐
  │ Writer     Reader1    Reader2    Writer      │
  │ updates    in RCU     finishes   frees old   │
  │ ──┤        ╠═══╣               ├──           │
  │   ├────── grace period ────────┤             │
  └──────────────────────────────────────────────┘

Context switches are quiescent states for RCU —
  schedule() → rcu_note_context_switch()
  This is why context switches help RCU make progress.
```

---

## 35.4 Scenario-Based Questions

### Q19: A process is stuck in D state (uninterruptible sleep). How do you debug it?

```
D state = TASK_UNINTERRUPTIBLE — cannot be killed with SIGKILL

Step 1: Identify the process
  ps aux | grep " D"
  cat /proc/<pid>/status    # State: D (disk sleep)
  cat /proc/<pid>/wchan     # Kernel function where blocked

Step 2: Get kernel stack trace
  cat /proc/<pid>/stack
  # Example output:
  [<0>] nfs_wait_on_inode+0x28/0x40
  [<0>] nfs_file_write+0x1a0/0x210
  [<0>] vfs_write+0xb8/0x2a0
  # → Stuck waiting on NFS I/O

Step 3: Check what it's waiting for
  Common causes:
  - Disk I/O (storage failure, full queue)
  - NFS/network filesystem (server unreachable)
  - Device driver bug (missing wakeup)
  - Kernel mutex held by crashed module

Step 4: System-wide check
  echo w > /proc/sysrq-trigger   # Show blocked tasks
  dmesg | tail                    # Kernel messages
  iostat -x 1                     # I/O statistics

Step 5: Resolution
  - If NFS: check network/server, umount -f / umount -l
  - If disk: check dmesg for I/O errors
  - If driver bug: only reboot can clear
  - Modern alternative: TASK_KILLABLE (D state + SIGKILL OK)
```

### Q20: System is experiencing high context switch rate. Diagnose and fix.

```
Step 1: Measure
  vmstat 1          # "cs" column = context switches/sec
  pidstat -w 1      # Per-process switches (voluntary + involuntary)
  perf stat -a -- sleep 5   # System-wide stats

  Typical rates:
    1K-10K/s  = normal desktop
    10K-50K/s = busy server
    >100K/s   = investigate

Step 2: Identify source
  # Find processes causing most switches:
  pidstat -w 1 | sort -k5 -rn | head

  # voluntary (cswch/s) = I/O waits, sleeps, futex
  # involuntary (nvcswch/s) = preemption, time quantum expired

Step 3: Analyze with perf
  perf sched record -- sleep 10
  perf sched latency --sort max
  perf sched map

Step 4: Common causes and fixes
  ┌──────────────────────────┬──────────────────────────┐
  │ Cause                    │ Fix                      │
  ├──────────────────────────┼──────────────────────────┤
  │ Too many threads         │ Use thread pools         │
  │ Spin-then-sleep locks    │ Tune spinlock backoff    │
  │ Producer-consumer churn  │ Batch work items         │
  │ Small time slices        │ Increase granularity     │
  │ Excessive polling        │ Use epoll/io_uring       │
  │ Lock contention          │ Reduce critical sections │
  │ RT tasks thrashing       │ Check priority design    │
  └──────────────────────────┴──────────────────────────┘

Step 5: Tune scheduler
  # Increase minimum granularity (reduce switches):
  echo 1500000 > /proc/sys/kernel/sched_min_granularity_ns
  
  # Or use SCHED_BATCH for throughput-oriented tasks:
  chrt -b 0 ./my_program
```

### Q21: An RT audio application has occasional glitches. How to investigate?

```
Audio glitch = processing deadline missed (e.g., 5ms buffer)

Step 1: Verify RT configuration
  uname -a                         # Check for -rt kernel
  cat /sys/kernel/realtime         # 1 = PREEMPT_RT
  chrt -p <pid>                    # Verify SCHED_FIFO/DEADLINE
  cat /proc/sys/kernel/sched_rt_runtime_us  # RT throttling

Step 2: Measure latency
  cyclictest -t1 -p 80 -n -i 1000 -l 10000
  # Worst-case latency should be < audio buffer period

Step 3: Trace scheduling events
  trace-cmd record -e sched_switch -e sched_wakeup \
    -P <audio_pid> sleep 10
  trace-cmd report | grep -A1 <audio_pid>

  # Look for: long gaps between wakeup → sched_switch
  # This is scheduling latency

Step 4: Common causes
  a) SMI (System Management Interrupt) — BIOS steals CPU
     hwlatdetect --duration=60
  
  b) IRQ handlers too long
     cat /proc/interrupts   # Check IRQ counts
     # With PREEMPT_RT, IRQs are threaded: check priorities
  
  c) RT throttling stealing time
     # Default: RT gets 950ms per 1000ms
     echo -1 > /proc/sys/kernel/sched_rt_runtime_us  # Disable
     # WARNING: RT bug can lock system
  
  d) CPU frequency scaling delays
     cpupower frequency-set -g performance
  
  e) Page faults
     mlockall(MCL_CURRENT | MCL_FUTURE)  # Lock all pages
  
  f) Priority inversion
     Use SCHED_DEADLINE instead of SCHED_FIFO
     Or ensure PI mutexes (pthread_mutexattr_setprotocol)
```

### Q22: Design a scheduling strategy for an embedded automotive system.

```
Requirements:
  - Safety-critical (ASIL-B/D): airbag, braking → hard RT
  - Infotainment: media, navigation → soft RT
  - Telemetry: logging, OTA → best-effort

Architecture (e.g., SA8155P with 8 cores):

  CPU Partitioning:
  ┌─────────────────────────────────────────────┐
  │ CPU 0-1: Safety (isolated)                  │
  │   isolcpus=0,1  nohz_full=0,1              │
  │   SCHED_DEADLINE for sensor/actuator loops  │
  │   PREEMPT_RT kernel                         │
  │   mlockall + preallocated memory            │
  ├─────────────────────────────────────────────┤
  │ CPU 2-3: Instrument Cluster                 │
  │   SCHED_FIFO prio 50-70 for rendering      │
  │   CPU affinity pinned                       │
  │   GPU deadlines managed                     │
  ├─────────────────────────────────────────────┤
  │ CPU 4-7: Infotainment + General             │
  │   SCHED_NORMAL (CFS) for most tasks         │
  │   cgroup CPU bandwidth for isolation        │
  │   Android scheduling (EAS)                  │
  └─────────────────────────────────────────────┘

  Key configurations:
    # Isolate safety CPUs:
    isolcpus=0,1 nohz_full=0,1 rcu_nocbs=0,1
    
    # Safety task setup:
    struct sched_attr attr = {
        .sched_policy   = SCHED_DEADLINE,
        .sched_runtime  = 500000,    // 0.5ms
        .sched_deadline = 2000000,   // 2ms
        .sched_period   = 5000000,   // 5ms
    };
    sched_setattr(0, &attr, 0);
    
    # Memory locking:
    mlockall(MCL_CURRENT | MCL_FUTURE);
    
    # Disable RT throttling on safety CPUs:
    echo -1 > /sys/fs/cgroup/cpu/safety/cpu.rt_runtime_us
```

---

## 35.5 Kernel Internals — Deep Questions

### Q23: How does copy-on-write (COW) work with fork()?

```
When fork() creates a child:
  1. copy_mm() → dup_mm() → dup_mmap()
  2. Parent AND child page table entries point to SAME
     physical pages
  3. Both PTEs marked READ-ONLY (even if originally writable)
  4. Reference count on each page frame incremented

When either process WRITES to a COW page:
  1. Page fault (write to read-only page)
  2. Fault handler: do_wp_page() (arch/mm/memory.c)
  3. Check: is this a COW page? (page refcount > 1)
  4. YES → allocate new page, copy content, update PTE
     The writing process gets private copy
     Other process keeps original
  5. Mark new PTE writable, decrement refcount on original

Why COW?
  - fork() + exec() pattern: child immediately replaces
    address space, so copying pages would be wasted
  - Even fork() without exec: most pages are read-only
    (code, read-only data), only modified pages are copied
  
  Cost reduction:
    Without COW: fork() copies entire address space (~ms)
    With COW: fork() copies only page tables (~µs)
```

### Q24: Explain the kernel stack and how it relates to scheduling.

```
Each task has its own kernel stack:

  Allocation: alloc_thread_stack_node() during fork
  Size: typically 2 pages (8KB on x86, 16KB on ARM64)
        CONFIG_THREAD_SIZE

  Layout (ARM64):
  ┌───────────────────────┐ ← stack top (high address)
  │     struct pt_regs    │ ← saved user registers (on syscall)
  ├───────────────────────┤
  │                       │ 
  │   Kernel call stack   │ ← grows downward
  │       (local vars,    │
  │        return addrs)  │
  │                       │
  ├───────────────────────┤
  │   Stack canary/guard  │
  ├───────────────────────┤
  │   thread_info         │ ← TIF flags (TIF_NEED_RESCHED etc.)
  └───────────────────────┘ ← stack bottom

Context switch saves/restores:
  switch_to(prev, next):
    1. Save prev's SP → prev->thread.cpu_context.sp
    2. Save prev's PC → prev->thread.cpu_context.pc  
    3. Load next's SP ← next->thread.cpu_context.sp
    4. Load next's PC ← next->thread.cpu_context.pc
    5. We're now on next's kernel stack!

Stack overflow protection:
  - VMAP_STACK: Guard page at bottom (detects overflow)
  - CONFIG_STACKPROTECTOR: Canary value check
  - KASAN: Runtime memory error detection
```

### Q25: How does the scheduler handle CPU hotplug?

```
CPU hotplug: bringing CPUs online/offline at runtime.

Taking CPU offline:
  echo 0 > /sys/devices/system/cpu/cpu3/online

Scheduler path:
  1. cpu_down() → takedown_cpu()
  2. sched_cpu_deactivate()
     - Set CPU inactive in cpu_active_mask
     - migrate_tasks(): move ALL tasks off this CPU
       → For each task on CPU's runqueue:
         select_fallback_rq() to find new CPU
         set_task_cpu(p, new_cpu)
         → Tasks with cpu affinity ONLY to this CPU:
           Temporarily allow wider affinity
  3. stop_machine(): halt the CPU
  4. Scheduling domains rebuilt to exclude offline CPU

Bringing CPU online:
  echo 1 > /sys/devices/system/cpu/cpu3/online
  
  1. cpu_up() → bringup_cpu()
  2. sched_cpu_activate()
     - Add CPU to cpu_active_mask
     - Rebuild scheduling domains
     - CPU starts idle — load balancing will migrate tasks

Critical for:
  - Power management (disable unused CPUs)
  - Thermal throttling (offline hot cores)
  - Hardware error isolation
  - Live kernel patching
```

---

## 35.6 Quick-Fire Questions (Rapid Review)

```
Q: What's the default scheduling policy for normal processes?
A: SCHED_NORMAL (SCHED_OTHER in POSIX terms), nice 0

Q: What's the time complexity of CFS task selection?
A: O(1) — leftmost node is cached (rb_leftmost)

Q: What register holds the page table base on ARM64?
A: TTBR0_EL1 (user) and TTBR1_EL1 (kernel)

Q: How many priority levels does Linux have?
A: 140 total. 0-99 RT, 100-139 normal (nice -20 to +19)
   Internally MAX_PRIO = 140

Q: What's PID 1?
A: init (systemd typically). Adopts orphan processes.
   Special: immune to signals it doesn't handle.

Q: Can a SCHED_NORMAL task preempt a SCHED_FIFO task?
A: No. RT class always preempts fair class.
   SCHED_DEADLINE preempts both.

Q: What's the difference between nice and priority?
A: nice = user-facing (-20 to +19)
   priority = kernel internal (100-139 for normal)
   static_prio = MAX_RT_PRIO + nice + 20 = 120 + nice

Q: What does preempt_count track?
A: Nested non-preemptible sections:
   Bits 0-7: preemption disable count
   Bits 8-15: softirq disable count  
   Bits 16-19: hardirq nesting count
   Bit 21: NMI flag
   preempt_count > 0 → cannot preempt

Q: What syscall creates both processes and threads?
A: clone() / clone3(). Flags determine sharing level.

Q: What's an idle task?
A: Per-CPU special task (PID 0, per-cpu swapper).
   Runs when no other tasks available.
   Executes power-saving instructions (HLT/WFI).

Q: What loads a new program into a process?
A: execve() → do_execveat_common() → load_elf_binary()
   Replaces: code/data/heap/stack/signals. Keeps: PID, FDs.

Q: What's the difference between voluntary and involuntary
   context switches?
A: Voluntary: task sleeps (I/O, mutex, futex)
   Involuntary: task preempted (higher priority, time expired)
   Check: /proc/<pid>/status → voluntary/nonvoluntary

Q: What's cgroup CPU bandwidth control?
A: cpu.cfs_quota_us / cpu.cfs_period_us
   E.g., 50000/100000 = 50% CPU cap per period
   cgroup v2: cpu.max "50000 100000"

Q: How does the kernel avoid the thundering herd problem?
A: exclusive wake: wake_up() vs wake_up_all()
   WQ_FLAG_EXCLUSIVE on wait queue entries
   epoll: EPOLLEXCLUSIVE flag

Q: What's cpu affinity?
A: Restricting which CPUs a task can run on.
   sched_setaffinity(), taskset, cpuset cgroup
   task_struct->cpus_mask bit field
```

---

## 35.7 System Design Interview Questions

### Q26: Design a thread pool scheduler.

```
Requirements:
  - Fixed number of worker threads
  - Tasks submitted to a queue
  - Workers pull and execute tasks
  - Handle priority, cancellation, shutdown

Design:
  ┌──────────────┐     ┌─────────────────┐
  │   Submit      │────→│  Priority Queue  │
  │   Task()      │     │  (min-heap by    │
  └──────────────┘     │   priority)      │
                        └────────┬────────┘
                                 │
                  ┌──────────────┼──────────────┐
                  ▼              ▼              ▼
            ┌──────────┐  ┌──────────┐  ┌──────────┐
            │ Worker 0 │  │ Worker 1 │  │ Worker 2 │
            │(pthread) │  │(pthread) │  │(pthread) │
            └──────────┘  └──────────┘  └──────────┘

Key implementation choices:
  - Use pthread_cond for signaling (not busy-wait)
  - Work stealing: idle worker steals from busy worker's
    local queue (like kernel's newidle_balance)
  - CPU affinity: pin workers to cores for cache locality
  - SCHED_FIFO for latency-critical pools

  // Worker loop pseudocode:
  while (!pool->shutdown) {
      pthread_mutex_lock(&pool->lock);
      while (queue_empty(&pool->queue) && !pool->shutdown)
          pthread_cond_wait(&pool->cond, &pool->lock);
      task = queue_dequeue(&pool->queue);
      pthread_mutex_unlock(&pool->lock);
      if (task) task->func(task->arg);
  }

Linux kernel example: workqueue (kernel/workqueue.c)
  kworker threads, per-CPU pools, unbound pools
```

### Q27: Compare process scheduling in a hypervisor vs. bare metal.

```
Bare Metal:
  Hardware → Linux Kernel → Processes
  Scheduler has full control of physical CPUs
  Direct PELT, cpufreq, C-states

Hypervisor (Type 1 — KVM/Xen):
  Hardware → Hypervisor → VMs (each with own kernel)
  
  Two levels of scheduling:
  ┌─────────────────────────────────────┐
  │  VM Guest Level                     │
  │  Guest kernel schedules processes   │
  │  vCPU = kernel thread on host       │
  └────────────────┬────────────────────┘
                   │ vCPU = host thread
  ┌────────────────▼────────────────────┐
  │  Host Level (KVM)                   │
  │  Host kernel schedules vCPUs        │
  │  CFS treats vCPU as regular task    │
  └─────────────────────────────────────┘

Challenges:
  1. Lock holder preemption: Guest holding spinlock,
     host preempts vCPU → all other vCPUs spin
     Solution: PV spinlocks (paravirt), kick notification
  
  2. Timer accuracy: Guest timer ticks may be delayed
     Solution: stolen time accounting (steal in /proc/stat)
  
  3. NUMA awareness: vCPU-to-pCPU mapping matters
     Solution: Pin vCPUs to NUMA nodes
  
  4. Overcommit: More vCPUs than pCPUs
     Solution: Credit-based scheduling (Xen), KVM uses CFS
```

---

## 35.8 Comparison Table — Key Concepts

```
┌─────────────────────┬────────────────┬────────────────────────┐
│ Concept             │ Key Function   │ File                   │
├─────────────────────┼────────────────┼────────────────────────┤
│ Fork process        │ kernel_clone() │ kernel/fork.c          │
│ Schedule            │ __schedule()   │ kernel/sched/core.c    │
│ CFS pick next       │ pick_next_entity│ kernel/sched/fair.c   │
│ Wake up task        │ try_to_wake_up │ kernel/sched/core.c    │
│ Context switch      │ context_switch │ kernel/sched/core.c    │
│ Load balance        │ load_balance() │ kernel/sched/fair.c    │
│ Set scheduler       │ __sched_setscheduler│ kernel/sched/core.c│
│ Exit process        │ do_exit()      │ kernel/exit.c          │
│ Send signal         │ do_send_sig_info│ kernel/signal.c       │
│ Exec program        │ do_execveat    │ fs/exec.c              │
│ Alloc PID           │ alloc_pid()    │ kernel/pid.c           │
│ Wait for child      │ do_wait()      │ kernel/exit.c          │
│ Timer tick          │ scheduler_tick │ kernel/sched/core.c    │
│ vruntime update     │ update_curr()  │ kernel/sched/fair.c    │
│ Migrate task        │ set_task_cpu() │ kernel/sched/core.c    │
└─────────────────────┴────────────────┴────────────────────────┘
```

---

## 35.9 Common Mistakes in Interviews

```
1. "Nice values ARE priorities"
   WRONG: Nice is user-facing (-20 to +19).
   Priority is internal (0-139). They're related but different.

2. "CFS uses time slices"
   PARTIALLY WRONG: CFS calculates proportional time
   allocations, but selection is by vruntime, not quantum.

3. "Threads are lighter than processes"
   NUANCED: In Linux, both are task_struct. Thread creation
   is faster (no mm copy), but scheduling cost is similar—
   context switch between same-mm tasks only saves switch_mm.

4. "SCHED_FIFO has no timeslice"
   CORRECT! But SCHED_RR does (default 100ms).
   Know the difference.

5. "Priority inversion only affects RT"
   WRONG: It affects any system with priorities and shared
   locks. CFS tasks can experience it too.

6. "fork() copies all memory"
   WRONG: COW means only page tables are copied.
   Physical pages shared until written.

7. "RT priority 99 is highest"
   CORRECT for RT class. But SCHED_DEADLINE has even
   higher priority (dl_sched_class above rt_sched_class).

8. "Context switch saves all registers"
   NUANCED: Only callee-saved registers. Caller-saved
   registers were already on the stack by the calling code.
```

---

## 35.10 Study Plan — 4-Week Schedule

```
Week 1: Foundations (Chapters 1-10)
  - Process concept, states, lifecycle
  - fork/exec/exit deep understanding
  - Threads vs processes
  - Context switching mechanics
  → Practice: Write fork/exec programs, trace with strace

Week 2: Scheduling Core (Chapters 11-18)
  - CFS: vruntime, red-black tree
  - RT scheduling: FIFO, RR, DEADLINE
  - Data structures: rq, sched_entity, sched_class
  - Preemption models
  → Practice: Use chrt, perf sched, read kernel source

Week 3: Advanced Topics (Chapters 19-28)
  - Synchronization, IPC, signals
  - Load balancing, PELT, NUMA
  - Process debugging and tracing tools
  → Practice: Build and trace RT applications

Week 4: Mastery (Chapters 29-35)
  - Flow diagrams and state machines
  - OS comparisons
  - Interview questions — mock sessions
  → Practice: Walk through __schedule() from memory,
    explain try_to_wake_up(), design scheduling solutions
```

---

## Summary

This chapter consolidates the key interview topics across all 34 previous chapters. Mastering these questions requires understanding:
- The kernel data structures (task_struct, sched_entity, struct rq)
- The code paths (fork, schedule, wakeup, exit)
- The design decisions (CFS fairness, class hierarchy, COW)
- Practical debugging skills (perf, ftrace, /proc)

**Key interview tip**: Always ground your answers in concrete kernel code.
Don't just explain concepts abstractly — reference functions, files, and
data structure fields. This demonstrates genuine kernel knowledge.

---

*This is the final chapter. Return to [Master Index](00_Master_Index.md) for navigation.*
