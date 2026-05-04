# Chapter 22: Process Groups and Sessions

## Learning Goals
- Understand process groups, sessions, and controlling terminals
- Learn job control mechanics (foreground/background)
- Master session leader and process group leader roles
- Know how orphan process groups are handled

---

## 22.1 Process Hierarchy

```
Linux process organization (3 levels):

  Session (SID)
  └── Process Group (PGID)
      └── Process (PID)
          └── Thread (tid, same TGID)

Example: user logs in via SSH

  Session 1000 (leader: bash, SID=1000)
  ├── Process Group 1000 (leader: bash, PGID=1000)
  │   └── bash (PID=1000) [foreground]
  │
  ├── Process Group 1001 (leader: vim, PGID=1001)
  │   └── vim (PID=1001) [foreground — replaced bash's group]
  │
  ├── Process Group 1005 (pipeline)
  │   ├── cat (PID=1005, PGID=1005)
  │   ├── grep (PID=1006, PGID=1005)
  │   └── sort (PID=1007, PGID=1005)
  │
  └── Process Group 1010 (background job)
      └── make -j4 (PID=1010, PGID=1010) [background]
          ├── cc (PID=1011, PGID=1010)
          ├── cc (PID=1012, PGID=1010)
          └── cc (PID=1013, PGID=1010)
```

---

## 22.2 Key Concepts

### Session

```c
/* Create new session (calling process becomes session leader) */
pid_t sid = setsid();

/* A session:
 *   - Has ONE session leader (the process that called setsid())
 *   - May have ONE controlling terminal
 *   - Contains multiple process groups
 *   - Session leader PID = Session ID (SID)
 *
 * Lifetime: until session leader exits AND all processes leave
 */
```

### Process Group

```c
/* Create new process group / join existing */
setpgid(pid, pgid);

/* If pid == pgid → pid becomes process group leader */
/* If pid == 0 → use calling process's PID */
/* If pgid == 0 → use pid as PGID */

pid_t pgid = getpgrp();  /* Get own PGID */
pid_t pgid = getpgid(pid);  /* Get process's PGID */

/* A process group:
 *   - Has ONE process group leader (PGID = leader's PID)
 *   - Contains multiple processes
 *   - Exists as long as any member process exists
 *   - Shell creates new group for each pipeline/job
 */
```

### Controlling Terminal

```
Each session may have ONE controlling terminal:

  Controlling terminal (e.g., /dev/pts/0)
  ├── Foreground process group → receives keyboard signals (^C, ^Z)
  └── Background process groups → SIGTTIN/SIGTTOU on terminal I/O

  Session leader = "controlling process"
  Only the foreground group can read from terminal
  Background groups get SIGTTIN (stopped) on read attempt

  tcsetpgrp(fd, pgid);  /* Set foreground process group */
  tcgetpgrp(fd);         /* Get foreground process group */
```

---

## 22.3 Kernel Data Structures

```c
/* include/linux/sched.h — relevant fields */
struct task_struct {
    pid_t                   pid;          /* Thread ID */
    pid_t                   tgid;         /* Process ID */
    
    struct task_struct      *group_leader; /* Thread group leader */
    struct signal_struct    *signal;       /* Shared signal info */
};

/* include/linux/sched/signal.h */
struct signal_struct {
    struct pid              *pids[PIDTYPE_MAX];
    /* PIDTYPE_PID, PIDTYPE_TGID, PIDTYPE_PGID, PIDTYPE_SID */

    struct pid              *tty_old_pgrp;  /* Previous foreground group */
    struct tty_struct       *tty;           /* Controlling terminal */
    
    int                     leader;         /* Is session leader? */
    /* ... */
};

/* PID namespace structure */
struct pid {
    refcount_t              count;
    unsigned int            level;
    spinlock_t              lock;
    struct hlist_head       tasks[PIDTYPE_MAX]; /* Linked lists of tasks */
    /* tasks[PIDTYPE_PGID] = all processes in this process group */
    /* tasks[PIDTYPE_SID]  = all processes in this session */
};
```

---

## 22.4 Job Control in Detail

```bash
# Interactive shell session:

$ cat file.txt | grep "pattern" | wc -l
#   Shell creates process group for pipeline
#   PGID = cat's PID (first process in pipeline)
#   All 3 processes: same PGID
#   This is the FOREGROUND group

$ make -j4 &
#   Shell creates new process group for make
#   PGID = make's PID
#   This is a BACKGROUND group

$ jobs
[1]+  Running   make -j4 &

$ fg %1
#   Shell calls tcsetpgrp() to make job 1's group foreground
#   Sends SIGCONT if stopped
#   Shell suspends itself (waits)

^Z
#   Terminal driver sends SIGTSTP to foreground group
#   All processes in group stop (TASK_STOPPED)

$ bg %1
#   Shell sends SIGCONT to the process group
#   Keeps it as background (doesn't tcsetpgrp)
```

### Signal Flow for Job Control

```
Ctrl+C (SIGINT):
  Terminal driver → foreground pgid → all processes in that group
  
  tty_signal_session_leader()
    → kill_pgrp(tty->pgrp, SIGINT, 1)
      → __kill_pgrp_info()
        → for each task in pgrp:
            group_send_sig_info(SIGINT, task)

Ctrl+Z (SIGTSTP):
  Same path but SIGTSTP
  Receiving processes → do_signal_stop() → TASK_STOPPED → schedule()

Background process reads terminal:
  → kernel sends SIGTTIN to the process group
  → Group stops until brought to foreground
```

---

## 22.5 Session Creation (setsid)

```
When a daemon starts:

  1. fork() — child continues, parent exits
  2. setsid() — child becomes:
     - Session leader (SID = PID)
     - Process group leader (PGID = PID)
     - Detached from controlling terminal
  3. fork() again — prevents reacquiring terminal
  4. chdir("/") — don't hold directory references
  5. Close/redirect stdin/stdout/stderr → /dev/null

  Classic daemon pattern:
    Original process (PID 100, SID=50)
      fork() → child (PID 101, SID=50)
        setsid() → (PID 101, SID=101, PGID=101, no tty)
          fork() → grandchild (PID 102, SID=101, PGID=101)
            ← This is the daemon (not session leader)
```

```c
/* kernel/sys.c — setsid implementation (simplified) */
SYSCALL_DEFINE0(setsid)
{
    struct task_struct *p = current;
    pid_t session = task_pid_vnr(p);

    /* Can't be process group leader already */
    if (p->signal->leader)
        return -EPERM;
    /* Can't if existing process group has same PGID as our PID */
    if (pid_task(find_pid_ns(session, ns), PIDTYPE_PGID))
        return -EPERM;

    p->signal->leader = 1;
    set_special_pids(session);  /* Set PGID and SID */
    p->signal->tty = NULL;     /* Detach controlling terminal */
    return session;
}
```

---

## 22.6 Orphan Process Groups

```
Orphan process group: group where NO member has a parent
in a DIFFERENT process group within the SAME session.

Why it matters:
  If the controlling process dies, stopped children might
  never be continued (no shell to "fg" them).

Kernel handling (exit_notify → kill_orphaned_pgrp):
  1. Detect orphaned group when parent exits
  2. If group has stopped members:
     → Send SIGHUP then SIGCONT to entire group
     → SIGHUP default = terminate (but can be caught)
     → SIGCONT = resume (so handler can clean up)

  Shell exits while job is stopped:
    Shell (PGID=100) exits
      → Job group (PGID=200) becomes orphaned
      → Kernel sends SIGHUP + SIGCONT to group
      → Default: all processes terminate
```

---

## 22.7 Process Groups and Scheduling

```
Process group operations that affect scheduling:

1. SIGSTOP/SIGTSTP to process group:
   → All members enter TASK_STOPPED
   → Removed from run queues → schedule() called
   → CPU freed for other tasks

2. SIGCONT to process group:
   → All stopped members → TASK_RUNNING
   → Added back to run queues
   → May preempt current task if higher priority

3. Process group kill (kill -PGID):
   → Each member gets signal
   → Fast path for batch termination

4. cgroup cpu controller:
   → Process groups can map to cgroups
   → CFS bandwidth limits apply to group
   → Throttling affects all group members
```

---

## 22.8 Viewing Process Groups

```bash
# ps with session and group info
ps -eo pid,ppid,pgid,sid,tty,stat,comm
  PID  PPID  PGID   SID TT       STAT COMMAND
    1     0     1     1 ?        Ss   systemd
  500   497   500   500 pts/0    Ss   bash
  600   500   600   500 pts/0    R+   vim       # + = foreground
  700   500   700   500 pts/0    T    sleep     # T = stopped
  800   500   800   500 pts/0    S    make      # Background

# ps forest view
ps -ejH
  PID  PGID   SID COMMAND
  500   500   500 bash              ← session leader
  600   600   500   vim             ← foreground group
  700   700   500   sleep           ← stopped background
  800   800   500   make            ← running background
  801   800   500     cc            ←   child of make (same PGID)

# /proc filesystem
cat /proc/<pid>/stat    # Field 5 = PGID, Field 6 = SID
cat /proc/<pid>/status  # Contains Tgid, Ngid, Pid, PPid, TracerPid
ls -la /proc/<pid>/fd/0 # → /dev/pts/N (controlling terminal)
```

---

## Interview Questions

**Q1: Why does a daemon call `setsid()` and then `fork()` again?**
A: `setsid()` creates a new session and detaches from the controlling terminal. The second `fork()` ensures the daemon is NOT a session leader — only session leaders can acquire controlling terminals (by opening a terminal device). The parent (session leader) exits; the grandchild continues as a non-session-leader, guaranteeing it can never accidentally reacquire a terminal.

**Q2: What happens to background jobs if the terminal closes?**
A: When the terminal disconnects: 1) The kernel sends SIGHUP to the session leader (shell). 2) The shell (if bash/zsh) sends SIGHUP to all its jobs. 3) Processes that don't handle SIGHUP terminate. Use `nohup` or `disown` to survive terminal closure. `tmux`/`screen` work by creating a pseudo-terminal that persists independently.

**Q3: How does `Ctrl+C` know which processes to signal?**
A: The terminal driver maintains the foreground process group ID (set by `tcsetpgrp()`). When Ctrl+C is pressed, the terminal's line discipline calls `kill_pgrp()` with the foreground PGID and SIGINT. The kernel iterates all processes whose PGID matches and sends SIGINT to each. Only the foreground group is affected — background groups are unaffected.

---

*Next: [Chapter 23 — Scheduling Policies](Chapter_23_Scheduling_Policies.md)*
