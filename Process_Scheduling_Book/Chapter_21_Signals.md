# Chapter 21: Signals and Process Control

## Learning Goals
- Understand signal delivery, handling, and masking in Linux
- Learn how signals interact with the scheduler
- Master signal implementation in the kernel
- Know real-time signals vs standard signals

---

## 21.1 Signal Fundamentals

```
Signals = asynchronous notifications sent to processes

Standard signals: 1-31 (POSIX)
Real-time signals: 32-64 (POSIX.1b, Linux-specific: 34-64)

Common signals and their default actions:
┌────────┬────────────────┬──────────┬──────────────────────────┐
│ Signal │ Name           │ Default  │ Purpose                  │
├────────┼────────────────┼──────────┼──────────────────────────┤
│ 1      │ SIGHUP         │ Term     │ Terminal hangup          │
│ 2      │ SIGINT         │ Term     │ Ctrl+C                   │
│ 3      │ SIGQUIT        │ Core     │ Ctrl+\ (core dump)       │
│ 6      │ SIGABRT        │ Core     │ abort()                  │
│ 9      │ SIGKILL        │ Term     │ Uncatchable kill         │
│ 11     │ SIGSEGV        │ Core     │ Segmentation fault       │
│ 13     │ SIGPIPE        │ Term     │ Broken pipe              │
│ 14     │ SIGALRM        │ Term     │ Timer alarm              │
│ 15     │ SIGTERM        │ Term     │ Graceful termination     │
│ 17     │ SIGCHLD        │ Ignore   │ Child status changed     │
│ 18     │ SIGCONT        │ Continue │ Resume stopped process   │
│ 19     │ SIGSTOP        │ Stop     │ Uncatchable stop         │
│ 20     │ SIGTSTP        │ Stop     │ Ctrl+Z                   │
└────────┴────────────────┴──────────┴──────────────────────────┘

SIGKILL (9) and SIGSTOP (19): CANNOT be caught, blocked, or ignored
```

---

## 21.2 Signal Handling

```c
#include <signal.h>

/* Modern API: sigaction (preferred over signal()) */
void handler(int signo, siginfo_t *info, void *context)
{
    /* info->si_pid  = sender PID */
    /* info->si_uid  = sender UID */
    /* info->si_code = signal cause (SI_USER, SI_KERNEL, etc.) */
    write(STDOUT_FILENO, "caught!\n", 8);  /* async-signal-safe only! */
}

int main(void)
{
    struct sigaction sa = {
        .sa_sigaction = handler,
        .sa_flags = SA_SIGINFO | SA_RESTART,
    };
    sigemptyset(&sa.sa_mask);
    sigaddset(&sa.sa_mask, SIGTERM);  /* Block SIGTERM during handler */

    sigaction(SIGINT, &sa, NULL);     /* Install handler for SIGINT */

    while (1) pause();  /* Sleep until signal */
}
```

### Signal Mask

```c
sigset_t mask;
sigemptyset(&mask);
sigaddset(&mask, SIGUSR1);
sigaddset(&mask, SIGUSR2);

/* Block signals (they become pending, not lost) */
sigprocmask(SIG_BLOCK, &mask, NULL);    /* Process-wide */
pthread_sigmask(SIG_BLOCK, &mask, NULL); /* Thread-specific */

/* Unblock → pending signals delivered immediately */
sigprocmask(SIG_UNBLOCK, &mask, NULL);

/* Wait for specific signal synchronously */
int sig;
sigwait(&mask, &sig);  /* Blocks until SIGUSR1 or SIGUSR2 */
```

---

## 21.3 Signal Data Structures in Kernel

```c
/* include/linux/sched/signal.h */
struct signal_struct {        /* Shared by all threads in process */
    refcount_t              sigcnt;
    struct sigpending       shared_pending;  /* Process-wide pending signals */
    struct task_struct      *curr_target;    /* Thread being targeted */
    int                     group_exit_code;
    int                     group_stop_count;
    /* ... timer, rlimit, accounting fields ... */
};

struct sighand_struct {       /* Signal handlers (shared by threads) */
    refcount_t              count;
    struct k_sigaction      action[_NSIG];   /* 64 signal actions */
    spinlock_t              siglock;
};

/* Per-thread signal state (in task_struct): */
struct task_struct {
    struct sigpending       pending;         /* Thread-specific pending */
    sigset_t                blocked;         /* Blocked signal mask */
    sigset_t                real_blocked;    /* Saved mask */
    struct sighand_struct   *sighand;        /* Shared handlers */
    struct signal_struct    *signal;         /* Shared signal info */
};
```

```
Signal data layout:

  Process (TGID = 100)
  ├── signal_struct (shared)
  │     ├── shared_pending: [SIGTERM pending]
  │     └── curr_target → Thread B
  ├── sighand_struct (shared)
  │     └── action[SIGTERM] = my_handler
  │
  ├── Thread A (PID 100)
  │     ├── pending: [SIGUSR1 pending]
  │     └── blocked: {SIGPIPE}
  ├── Thread B (PID 101)
  │     ├── pending: []
  │     └── blocked: {}
  └── Thread C (PID 102)
        ├── pending: []
        └── blocked: {SIGUSR1, SIGUSR2}
```

---

## 21.4 Signal Delivery — Kernel Path

```
Signal sending (kill/tkill/tgkill):

  sys_kill(pid, SIGTERM)
    → kill_something_info()
      → group_send_sig_info()
        → __send_signal()
          → sigaddset(&pending->signal, sig)
          → complete_signal()
            → Find eligible thread:
              1. Main thread (if not blocked)
              2. Any thread with signal unblocked
            → signal_wake_up(target)
              → set TIF_SIGPENDING on target
              → wake_up_state(target, TASK_INTERRUPTIBLE)
```

### Signal Delivery on Return to User Space

```
Signals are NOT delivered immediately!
They are delivered when the target returns to user space:

  Interrupt/syscall return path:
    exit_to_user_mode() or ret_to_user:
      → Check TIF_SIGPENDING
      → If set: do_signal()
        → get_signal()          /* Dequeue pending signal */
        → handle_signal()       /* Set up signal frame on user stack */
          → Push registers, return address onto user stack
          → Set PC to signal handler address
          → Return to user space → handler executes
          → Handler returns → sigreturn syscall → restore original context

  Stack frame for signal handler:
  ┌─────────────────────────────┐ High address
  │ Original user stack          │
  ├─────────────────────────────┤
  │ Signal frame (sigframe):     │
  │   - Saved registers          │
  │   - siginfo_t               │
  │   - ucontext_t              │
  │   - Return address (rt_sigreturn) │
  ├─────────────────────────────┤
  │ Signal handler stack frame   │
  └─────────────────────────────┘ Low address (SP)
```

---

## 21.5 Signals and Scheduling Interaction

```
Key interactions between signals and scheduler:

1. SIGSTOP/SIGTSTP → Process stops (TASK_STOPPED)
   do_signal_stop() → set_current_state(TASK_STOPPED) → schedule()
   
   All threads stop via group_stop mechanism

2. SIGCONT → Resume stopped process
   signal_wake_up() → wake_up_state(task, __TASK_STOPPED)
   Task returns to TASK_RUNNING

3. Signal wakes TASK_INTERRUPTIBLE sleepers:
   Task sleeping in read()/poll()/wait():
     signal_wake_up() → set TIF_SIGPENDING → wake up
     Syscall returns -EINTR or -ERESTARTSYS
     Signal delivered on return to user space

4. SIGKILL forces IMMEDIATE wakeup:
   Even wakes TASK_UNINTERRUPTIBLE in some paths
   recalc_sigpending() → fatal_signal_pending() → do fast exit

5. TASK_KILLABLE = TASK_UNINTERRUPTIBLE | TASK_WAKEKILL:
   Only wakeable by SIGKILL (not other signals)
   Used for critical I/O that shouldn't be interrupted
   But responds to kill -9
```

---

## 21.6 Real-Time Signals (32-64)

```
Standard signals vs Real-time signals:

  Standard (1-31):
    - NOT queued (multiple sends = one delivery)
    - No ordering guarantee
    - No data payload
    - Fixed semantics (SIGTERM, SIGINT, etc.)

  Real-time (32-64):
    - QUEUED (each send = one delivery)
    - Delivered in order (lowest number first)
    - Carry data payload (sigqueue)
    - No predefined meaning (application-defined)
```

```c
/* Sending real-time signal with data */
#include <signal.h>

/* Sender */
union sigval val;
val.sival_int = 42;  /* or sival_ptr for pointer */
sigqueue(target_pid, SIGRTMIN + 3, val);

/* Receiver */
void handler(int sig, siginfo_t *info, void *ctx)
{
    printf("RT signal %d, data=%d, from pid=%d\n",
           sig, info->si_value.sival_int, info->si_pid);
}

/* Install with SA_SIGINFO to receive siginfo_t */
struct sigaction sa = {
    .sa_sigaction = handler,
    .sa_flags = SA_SIGINFO,
};
sigaction(SIGRTMIN + 3, &sa, NULL);
```

---

## 21.7 Signal-Related Process Control

### Process Group Signals

```bash
# Send signal to entire process group
kill -SIGTERM -<pgid>       # dash prefix = process group

# Ctrl+C sends SIGINT to foreground process group
# Ctrl+Z sends SIGTSTP to foreground process group
```

### Job Control Flow

```
Terminal job control:

  $ sleep 100 &        ← Background: PGID set, no SIGTTOU
  [1] 1234

  $ fg %1              ← Foreground: SIGCONT if stopped
  sleep 100
  ^Z                   ← SIGTSTP → all threads stop
  [1]+  Stopped

  $ bg %1              ← SIGCONT → resume in background
  [1]+ sleep 100 &

  $ kill %1            ← SIGTERM to process group
  [1]+  Terminated
```

---

## 21.8 Fatal Signal Handling (Core Dump)

```
Signals that cause core dump: SIGQUIT, SIGILL, SIGABRT, SIGFPE, SIGSEGV, SIGBUS

  Kernel path:
    do_coredump()
      → Fill core file with:
        - ELF header
        - Program headers (memory mappings)
        - Register state (all threads)
        - /proc/pid/maps equivalent
        - Signal information
      → Write to: core, core.<pid>, or pipe to core_pattern

  /proc/sys/kernel/core_pattern examples:
    core                           # Simple file
    /tmp/core.%e.%p.%t            # Executable.pid.time
    |/usr/bin/systemd-coredump    # Pipe to coredump handler
```

---

## Interview Questions

**Q1: Why are signals delivered on return to user space, not immediately?**
A: Executing a signal handler requires modifying the user-space stack and registers. This can only be done safely at well-defined points (syscall/interrupt return) where the full user context is available on the kernel stack. Delivering mid-kernel-execution would corrupt kernel state. The TIF_SIGPENDING flag ensures the next return-to-user checks for pending signals.

**Q2: What happens when you send SIGKILL to a process in TASK_UNINTERRUPTIBLE (D) state?**
A: SIGKILL is set as pending, but TASK_UNINTERRUPTIBLE tasks CANNOT be woken by signals — they're waiting for an event that must complete (e.g., disk I/O). The kill takes effect when the I/O completes and the task returns to a signal check point. This is why `kill -9` doesn't work on some zombie/D-state processes. TASK_KILLABLE (introduced in 2.6.25) addresses this — it's UNINTERRUPTIBLE but responds to SIGKILL.

**Q3: In a multi-threaded process, which thread receives a process-directed signal?**
A: The kernel selects the first thread that: 1) hasn't blocked the signal, and 2) is preferably the "current target" (last thread that received a signal). From `complete_signal()`: it tries the main thread first, then walks the thread list. This is why applications typically block signals in worker threads and handle them in a dedicated signal-handling thread using `sigwait()`.

---

*Next: [Chapter 22 — Process Groups and Sessions](Chapter_22_Process_Groups_Sessions.md)*
