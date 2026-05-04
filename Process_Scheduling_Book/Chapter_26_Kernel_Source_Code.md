# Chapter 26: Kernel Source Code for Scheduling

## Learning Goals
- Navigate the scheduler source tree confidently
- Understand key files and their responsibilities
- Read the critical code paths with annotations
- Know how to trace scheduling logic in source

---

## 26.1 Scheduler Source File Map

```
kernel/sched/
├── core.c              ← Main scheduler: schedule(), __schedule(), 
│                          sched_setscheduler(), context_switch()
├── fair.c              ← CFS implementation: update_curr(), 
│                          pick_next_entity(), enqueue/dequeue
├── rt.c                ← RT scheduler: SCHED_FIFO, SCHED_RR
├── deadline.c          ← SCHED_DEADLINE: EDF + CBS
├── idle.c              ← Idle scheduling class
├── stop_task.c         ← Stop scheduling class (migration)
├── sched.h             ← Internal header: struct rq, cfs_rq, rt_rq
├── pelt.c              ← Per-Entity Load Tracking
├── pelt.h              ← PELT inline helpers
├── topology.c          ← Scheduling domains, load balancing
├── stats.h             ← Statistics/tracing helpers
├── autogroup.c         ← Per-TTY autogroup scheduling
├── debug.c             ← /proc/sched_debug output
├── cpufreq.c           ← schedutil governor integration
├── cpufreq_schedutil.c ← CPU frequency scaling
├── cputime.c           ← CPU time accounting
├── loadavg.c           ← System load average calculation
├── clock.c             ← rq clock management
├── swait.c             ← Simple waitqueues
├── wait.c              ← Wait queues
├── wait_bit.c          ← Bit-based wait queues
├── completion.c        ← Completion variables
├── membarrier.c        ← Memory barrier syscall
├── cpupri.c            ← CPU priority tracking (RT)
├── cpudeadline.c       ← CPU deadline tracking (DL)
├── features.h          ← Scheduler feature flags

include/linux/
├── sched.h             ← struct task_struct (scheduling fields)
├── sched/              
│   ├── signal.h        ← signal_struct
│   ├── prio.h          ← Priority definitions
│   ├── rt.h            ← RT-specific helpers
│   ├── deadline.h      ← DL-specific helpers
│   ├── topology.h      ← Topology helpers
│   ├── loadavg.h       ← Load average constants
│   └── task_stack.h    ← Kernel stack helpers

arch/{arm64,x86}/
├── kernel/process.c    ← __switch_to() (arch-specific)
├── include/asm/
│   ├── switch_to.h     ← switch_to() macro
│   └── mmu_context.h   ← switch_mm() (page table switch)
```

---

## 26.2 core.c — The Heart of Scheduling

### schedule() Entry Point

```c
/* kernel/sched/core.c (simplified for readability) */
asmlinkage __visible void __sched schedule(void)
{
    struct task_struct *tsk = current;

    sched_submit_work(tsk);  /* Submit blk-mq work if pending */
    do {
        preempt_disable();
        __schedule(SM_NONE);
        sched_preempt_enable_no_resched();
    } while (need_resched());
    sched_update_worker(tsk);
}

static void __sched notrace __schedule(unsigned int sched_mode)
{
    struct task_struct *prev, *next;
    struct rq *rq;
    int cpu;

    cpu = smp_processor_id();
    rq = cpu_rq(cpu);
    prev = rq->curr;

    /* Update rq clock */
    update_rq_clock(rq);

    /* Dequeue prev if it's going to sleep */
    if (!preempt && prev_state) {
        if (signal_pending_state(prev_state, prev)) {
            WRITE_ONCE(prev->__state, TASK_RUNNING);
        } else {
            deactivate_task(rq, prev, DEQUEUE_SLEEP);
            /* prev removed from runqueue */
        }
    }

    /* Pick next task — walks scheduling class chain */
    next = pick_next_task(rq, prev, &rf);

    if (likely(prev != next)) {
        rq->nr_switches++;
        rq->curr = next;

        /* Count voluntary/involuntary */
        trace_sched_switch(preempt, prev, next, prev_state);

        /* THE ACTUAL CONTEXT SWITCH */
        rq = context_switch(rq, prev, next, &rf);
    }
    /* Now running as 'next' */
}
```

### pick_next_task() — Class Chain Walk

```c
/* kernel/sched/core.c */
static inline struct task_struct *
pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
    /* Optimization: if all tasks are CFS, skip class walk */
    if (likely(!sched_class_above(prev->sched_class, &fair_sched_class) &&
               rq->nr_running == rq->cfs.h_nr_running)) {
        /* Fast path: only CFS tasks */
        return pick_next_task_fair(rq, prev, rf);
    }

    /* Slow path: walk class priority chain */
    for_each_class(class) {            /* stop → dl → rt → fair → idle */
        p = class->pick_next_task(rq);
        if (p)
            return p;
    }
    /* Should never reach here — idle class always returns */
    BUG();
}
```

### context_switch()

```c
/* kernel/sched/core.c */
static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
               struct task_struct *next, struct rq_flags *rf)
{
    prepare_task_switch(rq, prev, next);

    /* Switch memory mapping */
    if (!next->mm) {
        /* Kernel thread — borrow prev's active_mm */
        next->active_mm = prev->active_mm;
        enter_lazy_tlb(prev->active_mm, next);
    } else {
        /* User process — switch page tables */
        switch_mm_irqs_off(prev->active_mm, next->mm, next);
    }

    /* Switch register state + kernel stack */
    switch_to(prev, next, prev);
    barrier();

    return finish_task_switch(prev);
}
```

---

## 26.3 fair.c — CFS Implementation

### Key Functions

```c
/* kernel/sched/fair.c — function summary */

/* === Task lifecycle on CFS runqueue === */
enqueue_entity(cfs_rq, se, flags)    /* Add task to RB tree */
dequeue_entity(cfs_rq, se, flags)    /* Remove from RB tree */
update_curr(cfs_rq)                  /* Update vruntime of running task */
pick_next_entity(cfs_rq)             /* Select leftmost RB node */
put_prev_entity(cfs_rq, prev)        /* Return previous task to tree */
set_next_entity(cfs_rq, se)          /* Set new current entity */

/* === Global CFS operations === */
task_fork_fair(p)                    /* New task placement */
task_tick_fair(rq, curr, queued)     /* Timer tick handler */
check_preempt_wakeup(rq, p, flags)  /* Should waking task preempt? */
select_task_rq_fair(p, prev_cpu, wake_flags) /* CPU selection on wake */

/* === Load balancing (major functions) === */
load_balance(this_cpu, this_rq, sd, idle) /* Main balance function */
find_busiest_group(env, bal)         /* Find overloaded group */
find_busiest_queue(env, group)       /* Find overloaded CPU */
detach_tasks(env)                    /* Remove tasks from busy CPU */
attach_tasks(env)                    /* Add tasks to idle CPU */
newidle_balance(this_rq, rf)         /* Balance when going idle */
```

### update_curr() — The Heartbeat of CFS

```c
static void update_curr(struct cfs_rq *cfs_rq)
{
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_clock_task(rq_of(cfs_rq));
    u64 delta_exec;

    if (unlikely(!curr))
        return;

    delta_exec = now - curr->exec_start;
    if (unlikely((s64)delta_exec <= 0))
        return;

    curr->exec_start = now;

    /* Track total execution time */
    curr->sum_exec_runtime += delta_exec;
    schedstat_add(cfs_rq->exec_clock, delta_exec);

    /* Update virtual runtime (weighted) */
    curr->vruntime += calc_delta_fair(delta_exec, curr);

    /* Update minimum vruntime floor */
    update_min_vruntime(cfs_rq);

    /* Update deadline for EEVDF (6.6+) */
    update_deadline(cfs_rq, curr);

    /* Account for CFS bandwidth throttling */
    account_cfs_rq_runtime(cfs_rq, delta_exec);
}
```

---

## 26.4 rt.c — Real-Time Scheduler

```c
/* kernel/sched/rt.c — key functions */

/* Pick highest priority RT task: O(1) */
static struct task_struct *pick_next_task_rt(struct rq *rq)
{
    struct rt_rq *rt_rq = &rq->rt;
    struct sched_rt_entity *rt_se;
    struct rt_prio_array *array = &rt_rq->active;
    int idx;

    if (!rt_rq->rt_nr_running)
        return NULL;

    idx = sched_find_first_bit(array->bitmap);  /* O(1) */
    rt_se = list_entry(array->queue[idx].next, ...);
    return rt_task_of(rt_se);
}

/* Timer tick for SCHED_RR */
static void task_tick_rt(struct rq *rq, struct task_struct *p, int queued)
{
    if (p->policy != SCHED_RR)
        return;  /* SCHED_FIFO: no timeslice */

    if (--p->rt.time_slice)
        return;  /* Still has time left */

    /* Timeslice expired — move to end of priority queue */
    p->rt.time_slice = sched_rr_timeslice;
    requeue_task_rt(rq, p, 0);
    set_tsk_need_resched(p);
}
```

---

## 26.5 Reading Scheduler Code — Tips

```
Navigating kernel/sched/:

1. Start with __schedule() in core.c
   → This is the ONE function called on every context switch
   → Follow: pick_next_task → class->pick_next_task → context_switch

2. For CFS specifics:
   → Start with task_tick_fair() (called every tick)
   → Then check_preempt_wakeup() (called on task wake)
   → Then update_curr() (called from both above)

3. For load balancing:
   → Start with trigger_load_balance() (called from scheduler_tick)
   → Follow load_balance() → find_busiest_group() → detach_tasks()
   → Also: select_task_rq_fair() for task placement on wake/fork

4. For RT:
   → pick_next_task_rt() is simple to understand
   → push_rt_task() / pull_rt_task() for SMP balancing

5. Grep patterns for exploring:
   grep -n "static.*sched_class" kernel/sched/fair.c  # Class operations
   grep -n "DEFINE_SCHED_CLASS" kernel/sched/*.c        # Class definitions
   grep -rn "set_tsk_need_resched" kernel/sched/       # Preemption triggers

6. Important macros:
   for_each_class(class)     — iterate scheduling classes
   for_each_domain(cpu, sd)  — iterate scheduling domains
   rq_of(cfs_rq)             — get rq from cfs_rq
   task_rq(p)                — get task's rq
   cpu_rq(cpu)               — get CPU's rq
   task_of(se)               — get task from sched_entity
```

---

## 26.6 Scheduling Class Registration

```c
/* kernel/sched/fair.c */
DEFINE_SCHED_CLASS(fair) = {
    .enqueue_task       = enqueue_task_fair,
    .dequeue_task       = dequeue_task_fair,
    .yield_task         = yield_task_fair,
    .check_preempt_curr = check_preempt_wakeup,
    .pick_next_task     = __pick_next_task_fair,
    .put_prev_task      = put_prev_task_fair,
    .set_next_task      = set_next_task_fair,
    .select_task_rq     = select_task_rq_fair,
    .task_tick          = task_tick_fair,
    .task_fork          = task_fork_fair,
    .task_dead          = task_dead_fair,
    .balance            = balance_fair,
    .switched_from      = switched_from_fair,
    .switched_to        = switched_to_fair,
    .prio_changed       = prio_changed_fair,
    .update_curr        = update_curr_fair,
#ifdef CONFIG_FAIR_GROUP_SCHED
    .task_change_group  = task_change_group_fair,
#endif
};

/* Class priority chain (compile-time linked): */
/* stop_sched_class → dl_sched_class → rt_sched_class → 
 * fair_sched_class → idle_sched_class */
```

---

## 26.7 Source Cross-Reference

```
Common call chains to trace:

Timer tick path:
  tick_handle_periodic() → tick_periodic() → scheduler_tick()
    → task_tick() → sched_class->task_tick() → task_tick_fair()
      → update_curr() → entity_tick() → check need_resched

Wakeup path:
  try_to_wake_up(p) → ttwu_queue(p, cpu)
    → activate_task(rq, p) → enqueue_task() → sched_class->enqueue_task()
    → check_preempt_curr() → sched_class->check_preempt_curr()
      → resched_curr() → set TIF_NEED_RESCHED

Fork path:
  kernel_clone() → copy_process() → sched_fork(p)
    → p->sched_class->task_fork(p) → task_fork_fair()
    → wake_up_new_task(p) → activate_task() → check_preempt_curr()

Syscall: sched_setscheduler:
  __sched_setscheduler() → __setscheduler()
    → Change policy, priority, sched_class
    → check_class_changed() → switched_to() callback
```

---

## Interview Questions

**Q1: Where in the kernel source would you look to understand why a CFS task is being preempted?**
A: Two paths cause CFS preemption: 1) `check_preempt_wakeup()` in `kernel/sched/fair.c` — called when a higher-vruntime task wakes a lower-vruntime task. Checks if woken task's vruntime is lower than current by at least `sched_wakeup_granularity`. 2) `entity_tick()` → `check_preempt_tick()` in fair.c — called on timer tick. Checks if current has run longer than its ideal weighted share and if leftmost task's vruntime is lower. Both call `resched_curr()` to set TIF_NEED_RESCHED.

**Q2: How is the scheduling class chain implemented?**
A: Classes are defined with `DEFINE_SCHED_CLASS()` macro which creates a struct linked at compile time. The `for_each_class(class)` macro walks from highest (stop) to lowest (idle). Each class is a `struct sched_class` with function pointers (vtable pattern). `pick_next_task()` iterates classes and calls each `pick_next_task` method until one returns a non-NULL task. This guarantees higher-class tasks always run first.

**Q3: How would you add a new scheduling class to Linux?**
A: 1) Create a new file `kernel/sched/myclass.c`. 2) Define `struct sched_class myclass_sched_class` with all required function pointers. 3) Insert it in the class chain (e.g., between fair and idle). 4) Add a new `SCHED_MYCLASS` policy constant. 5) Update `__setscheduler_class()` in core.c to map the policy. 6) Implement all required callbacks: enqueue, dequeue, pick_next, task_tick, check_preempt, etc. The modular design makes extension straightforward.

---

*Next: [Chapter 27 — Process Debugging Tools](Chapter_27_Process_Debugging_Tools.md)*
