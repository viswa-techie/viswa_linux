# Chapter 29: Flow Diagrams

## Learning Goals
- Visualize complete process lifecycle flows
- Understand scheduler decision paths end-to-end
- See context switch flow from trigger to completion
- Map fork → exec → exit flows in detail

---

## 29.1 Process Creation Flow (fork)

```
User calls fork()
        │
        ▼
    sys_fork() / sys_clone()
        │
        ▼
    kernel_clone()
        │
        ▼
    copy_process() ──────────────────────────────────
        │                                             │
        ├── dup_task_struct()                          │
        │     Allocate new task_struct + kernel stack  │
        │                                             │
        ├── sched_fork(clone_flags, p)                │
        │     Set p->prio, p->sched_class             │
        │     Initialize sched_entity                 │
        │     p->on_rq = 0 (not yet enqueued)         │
        │                                             │
        ├── copy_files()   ← share or dup fd table    │
        ├── copy_fs()      ← share or dup fs info     │
        ├── copy_sighand() ← share or dup handlers    │
        ├── copy_signal()  ← new signal_struct        │
        ├── copy_mm()      ← COW page tables          │
        │                                             │
        ├── alloc_pid()    ← get new PID              │
        │                                             │
        └── Return p (new task_struct)                │
        │                                          ────
        ▼
    wake_up_new_task(p)
        │
        ├── activate_task(rq, p)
        │     enqueue_task → sched_class->enqueue_task()
        │     Insert into CFS RB tree / RT priority array
        │
        └── check_preempt_curr(rq, p)
              If child should preempt parent
                → set TIF_NEED_RESCHED on parent
        │
        ▼
    Return to parent (or child if sched_child_runs_first)
```

---

## 29.2 Process Execution Flow (exec)

```
User calls execve("/bin/program", argv, envp)
        │
        ▼
    sys_execve()
        │
        ▼
    do_execveat_common()
        │
        ├── Open executable file
        │
        ├── bprm_fill(bprm) ← Read first 256 bytes (magic number)
        │
        ├── search_binary_handler(bprm)
        │     Try each registered format:
        │       - load_elf_binary()  ← most common
        │       - load_script()     ← #! interpreter
        │       - load_misc_binary() ← binfmt_misc
        │
        ▼
    load_elf_binary()
        │
        ├── flush_old_exec()
        │     ├── de_thread()       ← Kill other threads
        │     ├── exec_mmap()       ← Replace address space
        │     │     └── mm_release() → mmput(old_mm) → free old pages
        │     └── flush_signal_handlers() ← Reset caught signals
        │
        ├── setup_new_exec()
        │     ├── Set new comm (program name)
        │     ├── arch_setup_new_exec() ← Arch-specific (clear ADDR_LIMIT)
        │     └── Set new credentials if setuid
        │
        ├── Map ELF segments into new address space
        │     .text  → executable, read-only
        │     .data  → read-write
        │     .bss   → zero-initialized
        │     stack  → user stack with argv, envp
        │
        ├── Set up new entry point
        │     START_THREAD(regs, elf_entry, new_sp)
        │
        └── Return to user space → starts executing new program
```

---

## 29.3 Process Termination Flow

```
Process calls exit(status) or receives fatal signal
        │
        ▼
    do_exit(code)
        │
        ├── exit_signals(tsk)        ← Set PF_EXITING
        │
        ├── exit_mm(tsk)             ← Release address space
        │     mmput(mm) → if last user → free all pages
        │
        ├── exit_sem(tsk)            ← Release SysV semaphores
        ├── exit_shm(tsk)            ← Release shared memory
        ├── exit_files(tsk)          ← Close all file descriptors
        ├── exit_fs(tsk)             ← Release fs_struct
        │
        ├── exit_notify(tsk)
        │     ├── forget_original_parent()
        │     │     Reparent children to init/subreaper
        │     │
        │     ├── If parent wants SIGCHLD:
        │     │     do_notify_parent(tsk, SIGCHLD)
        │     │     tsk->exit_state = EXIT_ZOMBIE  ─────┐
        │     │                                          │
        │     └── If parent ignores SIGCHLD:             │
        │           tsk->exit_state = EXIT_DEAD          │
        │           release_task() → free immediately    │
        │                                                │
        ├── Schedule away (TASK_DEAD)                    │
        │     tsk->__state = TASK_DEAD                   │
        │     do_task_dead()                              │
        │       → __schedule()  ← NEVER returns          │
        │                                                │
        └── ────────── ZOMBIE STATE ──────────────────   │
              Waiting for parent to call wait()    ◄─────┘
              │
              ▼
            Parent calls wait4() / waitpid()
              → wait_task_zombie()
              → Collect exit_code
              → release_task(zombie)
                ├── __exit_signal()    ← Account to parent
                ├── __unhash_process() ← Remove from PID hash
                ├── delayed_put_task_struct()
                │     → free task_struct + kernel stack
                └── Process fully gone
```

---

## 29.4 Complete Scheduler Decision Flow

```
    ┌──────────────────────────────────────────────────────────┐
    │                  SCHEDULE TRIGGERS                        │
    │                                                          │
    │  Timer tick → scheduler_tick()                           │
    │  Wakeup    → try_to_wake_up() → check_preempt_curr()   │
    │  Yield     → sched_yield()                               │
    │  Sleep     → schedule() from wait_event etc.             │
    │  Preempt   → preempt_schedule() (CONFIG_PREEMPT)        │
    │                                                          │
    │  All paths → set TIF_NEED_RESCHED                        │
    └───────────────────────┬──────────────────────────────────┘
                            │
                            ▼
    ┌──────────────────────────────────────────────────────────┐
    │                    __schedule()                           │
    │                                                          │
    │  1. rq = this_rq(); prev = rq->curr;                    │
    │                                                          │
    │  2. If prev is sleeping → deactivate_task(prev)          │
    │     (remove from runqueue)                               │
    │                                                          │
    │  3. pick_next_task(rq):                                  │
    │     ┌─ Stop class: migration thread? ─── YES → return   │
    │     ├─ DL class: deadline task?       ─── YES → return   │
    │     ├─ RT class: RT task (FIFO/RR)?   ─── YES → return   │
    │     ├─ Fair class: CFS task?          ─── YES → return   │
    │     └─ Idle class: idle thread        ─── return         │
    │                                                          │
    │  4. If prev != next:                                     │
    │     context_switch(rq, prev, next)                       │
    │     ├── switch_mm()  → change page tables                │
    │     └── switch_to()  → change registers + stack          │
    │                                                          │
    │  5. Now running as 'next'                                │
    │     finish_task_switch(prev)                              │
    │     → Clean up prev if it exited (TASK_DEAD)             │
    └──────────────────────────────────────────────────────────┘
```

---

## 29.5 CFS Pick-Next Decision

```
    pick_next_task_fair(rq)
        │
        ▼
    update_curr(cfs_rq)
    ← Update current task's vruntime
        │
        ▼
    put_prev_entity(cfs_rq, prev)
    ← Put current back in RB tree
        │
        ▼
    pick_next_entity(cfs_rq)
        │
        ├── Get leftmost = __pick_first_entity()
        │     (smallest vruntime in RB tree → O(1) cached)
        │
        ├── Compare with curr:
        │     If curr->vruntime < leftmost->vruntime
        │       → Keep curr (cache warmth bonus)
        │
        ├── Check buddy hints (skip/last/next):
        │     Skip: avoid this entity (yield)
        │     Last: prefer this (just ran, cache warm)
        │     Next: prefer this (just woken, interactive)
        │
        └── Return best entity
        │
        ▼
    set_next_entity(cfs_rq, next)
    ← Remove from RB tree, set as current
        │
        ▼
    Return task_struct to __schedule()
```

---

## 29.6 Try-to-Wake-Up Flow

```
    try_to_wake_up(p, state, wake_flags)
        │
        ▼
    Is p->__state & state?
    ├── NO  → Return 0 (not in expected state)
    └── YES ↓
        │
        ▼
    Select target CPU:
    select_task_rq(p, p->wake_cpu, wake_flags)
        │
        ├── sched_class->select_task_rq()
        │     For CFS: select_task_rq_fair()
        │       ├── Want affine? → select_idle_sibling()
        │       │     Prefer: prev_cpu → target → idle LLC sibling
        │       └── Slow path: find_idlest_group/cpu
        │
        └── Return target CPU
        │
        ▼
    ttwu_queue(p, cpu)
        │
        ├── If same CPU: ttwu_do_activate() directly
        └── If remote CPU: queue work on target CPU's rq
        │
        ▼
    ttwu_do_activate(rq, p)
        │
        ├── activate_task(rq, p)
        │     enqueue_task(rq, p)
        │       → sched_class->enqueue_task()
        │       → CFS: insert into RB tree by vruntime
        │       → RT: insert into priority array
        │
        ├── ttwu_do_wakeup(rq, p)
        │     p->__state = TASK_RUNNING
        │
        └── check_preempt_curr(rq, p)
              sched_class->check_preempt_curr()
              → CFS: if p->vruntime < curr->vruntime - gran
                  → resched_curr() → TIF_NEED_RESCHED
              → RT: if p->prio < curr->prio
                  → resched_curr()
```

---

## 29.7 Context Switch Detailed Flow

```
    context_switch(rq, prev, next)
        │
        ▼
    prepare_task_switch(rq, prev, next)
    ← Fire trace events, arch prepare
        │
        ▼
    Is next a kernel thread (next->mm == NULL)?
    ├── YES: enter_lazy_tlb(prev->active_mm, next)
    │        next->active_mm = prev->active_mm  (borrow)
    │        No page table switch needed
    │
    └── NO:  switch_mm_irqs_off(prev->active_mm, next->mm, next)
             │
             ├── ARM64: Write next->mm->pgd to TTBR0_EL1
             │          Set ASID in TTBR0
             │          isb  (instruction synchronization barrier)
             │
             └── x86_64: Write next->mm->pgd to CR3
                         Set PCID bits to avoid TLB flush
        │
        ▼
    switch_to(prev, next, prev)
        │
        ├── Save prev's callee-saved registers to kernel stack
        │     ARM64: stp x19-x28, fp, lr, sp
        │     x86_64: push rbx, rbp, r12-r15
        │
        ├── Switch kernel stack pointer
        │     ARM64: mov sp, next->thread.cpu_context.sp
        │     x86_64: mov rsp, next->thread.sp
        │
        ├── Restore next's callee-saved registers
        │     ARM64: ldp x19-x28, fp, lr, sp
        │     x86_64: pop r15-r12, rbp, rbx
        │
        └── Return (to next's saved return address)
             ← Now executing as 'next' process!
        │
        ▼
    finish_task_switch(prev)
        │
        ├── If prev is TASK_DEAD → put_task_struct(prev) (free)
        ├── Fire trace_sched_switch() event
        └── Re-enable preemption
```

---

## 29.8 Timer Tick → Scheduler Interaction

```
    Hardware timer interrupt fires
        │
        ▼
    tick_handle_periodic() / tick_sched_handle()
        │
        ▼
    update_process_times(user_mode(regs))
        │
        ├── account_process_tick()
        │     Update current->utime or current->stime
        │
        └── scheduler_tick()
              │
              ├── curr->sched_class->task_tick(rq, curr)
              │     │
              │     ├── CFS: task_tick_fair()
              │     │     update_curr() → update vruntime
              │     │     entity_tick() → check_preempt_tick()
              │     │       If ran > ideal_runtime:
              │     │         set TIF_NEED_RESCHED
              │     │
              │     └── RT: task_tick_rt()
              │           If SCHED_RR && timeslice expired:
              │             requeue + set TIF_NEED_RESCHED
              │
              ├── trigger_load_balance(rq)
              │     If time for periodic rebalancing:
              │       raise SCHED_SOFTIRQ → run_rebalance_domains()
              │
              └── calc_global_load_tick()
                    Update system load averages
        │
        ▼
    irq_exit()
        │
        ▼
    Check TIF_NEED_RESCHED
    ├── Set   → preempt_schedule_irq() → __schedule()
    └── Clear → return to interrupted context
```

---

## 29.9 Signal Delivery Flow

```
    Signal sent (kill/tkill/signal)
        │
        ▼
    __send_signal(sig, info, task)
        │
        ├── Allocate sigqueue if RT signal
        ├── sigaddset(&pending->signal, sig)
        └── complete_signal(sig, task)
              │
              ├── Find target thread (unblocked for this signal)
              ├── signal_wake_up(t, sig == SIGKILL)
              │     set TIF_SIGPENDING
              │     wake_up_state(t, TASK_INTERRUPTIBLE)
              │       → try_to_wake_up() → enqueue → check preempt
              └── Return
        │
        ═══════ Time passes... ═══════
        │
        ▼
    Target returns to user space
    (syscall return / interrupt return)
        │
        ▼
    exit_to_user_mode_prepare()
        │
        ├── Check TIF_SIGPENDING
        └── do_signal(regs)
              │
              ├── get_signal() → dequeue signal from pending
              │
              ├── Default action?
              │     SIGKILL → do_group_exit()
              │     SIGSTOP → do_signal_stop() → TASK_STOPPED
              │     SIG_DFL → term/core/ignore
              │
              └── Custom handler?
                    handle_signal(sig, ka, info, regs)
                      Setup sigframe on user stack
                      Redirect IP to handler
                      Return to user → handler runs
                      Handler returns → sigreturn → restore
```

---

*Next: [Chapter 30 — Important Diagrams](Chapter_30_Important_Diagrams.md)*
