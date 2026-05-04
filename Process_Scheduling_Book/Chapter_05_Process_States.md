# Chapter 5: Process States

## Learning Goals
- Understand every state a Linux process can be in
- Map kernel constants to observable behavior
- Debug processes in unexpected states

---

## 5.1 Process State Overview

```c
/* include/linux/sched.h — state field values */
#define TASK_RUNNING            0x00000000
#define TASK_INTERRUPTIBLE      0x00000001
#define TASK_UNINTERRUPTIBLE    0x00000002
#define __TASK_STOPPED          0x00000004
#define __TASK_TRACED           0x00000008
#define EXIT_DEAD               0x00000010
#define EXIT_ZOMBIE             0x00000020
#define TASK_IDLE               0x00000402
```

## 5.1 Running State (R)

The task is **currently executing on a CPU**.

```c
task->__state = TASK_RUNNING;
task->on_cpu  = 1;   /* Actually on CPU right now */
```

Only **one task per CPU** can be in RUNNING+on_cpu state at any moment.

## 5.2 Runnable State (R)

The task is ready to run but **waiting for CPU time** on the run queue.

```c
task->__state = TASK_RUNNING;   /* Same constant! */
task->on_cpu  = 0;              /* Not on CPU — on run queue */
task->on_rq   = 1;              /* On the run queue */
```

Note: Linux uses the same `TASK_RUNNING` for both "running" and "runnable". The distinction is `on_cpu`.

## 5.3–5.4 Sleeping States (S, D)

### Interruptible Sleep (S)

Task waits for an event **but can be woken by a signal**.

```c
set_current_state(TASK_INTERRUPTIBLE);
schedule();   /* Remove from run queue, go to sleep */

/* Woken by: wait_event completes, OR signal delivered */
if (signal_pending(current))
    return -EINTR;   /* Signal interrupted the wait */
```

Typical: `read()` waiting for data, `sleep()`, `poll()`

### Uninterruptible Sleep (D)

Task waits for an event **and cannot be interrupted by signals**.

```c
set_current_state(TASK_UNINTERRUPTIBLE);
schedule();
/* Only woken by explicit wake_up() — signals ignored */
```

Typical: waiting for disk I/O, NFS mount. **Cannot be killed** — the dreaded "D state" processes.

```bash
# D-state process is unkillable:
$ kill -9 <pid>    # Has no effect!
# Must fix the underlying issue (disk I/O, NFS timeout)
```

## 5.5 Zombie State (Z)

Task has **exited but parent hasn't called `wait()`** yet.

```c
task->exit_state = EXIT_ZOMBIE;
/* task_struct still exists (holds exit code)
   but all resources (mm, files) already freed */
```

```
$ ps aux | grep Z
USER  PID  STAT  COMMAND
root  500  Z     [defunct]
```

**Zombie consumes**: only the `task_struct` (~6KB) + PID slot. Fix: parent must `wait()`, or reparent to init.

## 5.6 Stopped State (T)

Task is **stopped by a signal** (won't run until continued).

```c
task->__state = __TASK_STOPPED;
/* Triggered by SIGSTOP, SIGTSTP (Ctrl+Z), SIGTTIN, SIGTTOU */
/* Resumed by SIGCONT */
```

```bash
$ sleep 100 &
[1] 5000
$ kill -STOP 5000    # Process stops
$ ps -o pid,stat,comm -p 5000
  PID STAT COMMAND
 5000 T    sleep
$ kill -CONT 5000    # Process resumes
```

Also: `__TASK_TRACED` — stopped under ptrace (debugger attached).

---

## State Transition Diagram

```
                   fork()/clone()
                        │
                        ▼
                 ┌──────────────┐
                 │  TASK_RUNNING │
                 │  (Runnable)   │◄─────────────────────────┐
                 └──────┬───────┘         wake_up()         │
                        │                                    │
            schedule()  │                                    │
            picks task  │                                    │
                        ▼                                    │
                 ┌──────────────┐                    ┌───────┴──────┐
                 │  TASK_RUNNING │ ──sleep/block()──►│  TASK_       │
                 │  (Running)    │                   │  INTERRUPTIBLE│
                 └──┬───┬───────┘                    │  (S state)   │
                    │   │                            └──────────────┘
                    │   │                                    │
                    │   │ sleep/block()               signal │
                    │   │ (no signal)                        │
                    │   ▼                                    ▼
                    │  ┌──────────────┐              return -EINTR
                    │  │  TASK_       │
                    │  │  UNINTERRUPT │
                    │  │  IBLE (D)    │
                    │  └──────────────┘
                    │
           SIGSTOP  │                                exit()
            ┌───────┘                                  │
            ▼                                          ▼
     ┌──────────────┐                          ┌──────────────┐
     │  __TASK_     │                          │  EXIT_ZOMBIE │
     │  STOPPED (T) │                          │  (Z state)   │
     └──────────────┘                          └──────┬───────┘
            │                                         │ wait()
            │ SIGCONT                                  ▼
            └──────► back to RUNNABLE          ┌──────────────┐
                                               │  EXIT_DEAD   │
                                               │  (reclaimed) │
                                               └──────────────┘
```

---

## Viewing Process States

```bash
# ps STAT column codes
$ ps aux
USER   PID  %CPU %MEM STAT   COMMAND
root     1   0.0  0.1 Ss     /sbin/init          # S=sleeping, s=session leader
root     2   0.0  0.0 S      [kthreadd]           # S=interruptible sleep
root    15   0.0  0.0 S<     [ksoftirqd/0]        # S<=high priority
user  1000   5.0  2.0 R      /usr/bin/gcc          # R=running/runnable
user  1001   0.0  0.0 T      vim                   # T=stopped
user  1002   0.0  0.0 Z      [defunct]             # Z=zombie
root   500   0.0  0.0 D      [nfs_mount]           # D=uninterruptible

# STAT modifiers:
# s = session leader
# l = multi-threaded
# + = foreground process group
# < = high priority
# N = low priority (nice > 0)
```

```bash
# Detailed state from /proc
cat /proc/<pid>/status | grep State
# State:  S (sleeping)

cat /proc/<pid>/stat | awk '{print $3}'
# R (running), S (sleeping), D (disk sleep), T (stopped), Z (zombie)
```

---

## Interview Questions

**Q1: What is the difference between S and D states?**
A: S (interruptible sleep) — task waits for an event but can be woken by signals (returns -EINTR). D (uninterruptible sleep) — task cannot be interrupted; even SIGKILL is ignored. D state is used for critical kernel operations (disk I/O) where interruption would cause data corruption.

**Q2: How do you debug a process stuck in D state?**
A: 1) `cat /proc/<pid>/wchan` — shows which kernel function it's blocked in. 2) `cat /proc/<pid>/stack` — shows kernel stack trace. 3) Common causes: NFS timeout, hung disk I/O, broken driver. 4) Fix the underlying resource issue; you cannot kill a D-state process.

**Q3: What happens if a parent never calls wait() for a zombie?**
A: The zombie stays forever, consuming a `task_struct` and PID slot. If many zombies accumulate, PID space exhaustion occurs. Fix: the parent must be fixed to call `wait()`/`waitpid()`, or if the parent exits, init takes over and reaps them.

**Q4: Why does Linux use the same TASK_RUNNING for both running and runnable?**
A: Because from the scheduler's perspective, both states mean "ready to execute." The `on_cpu` flag distinguishes the two. This simplifies state management — transitions between running and runnable happen frequently and don't need a state change.

---

*Next: [Chapter 6 — Process Lifecycle](Chapter_06_Process_Lifecycle.md)*
