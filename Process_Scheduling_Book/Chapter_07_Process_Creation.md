# Chapter 7: Process Creation Mechanisms

## Learning Goals
- Deep dive into fork(), vfork(), clone(), exec()
- Understand copy-on-write mechanism in detail
- Follow kernel code path for process duplication

---

## 7.1 fork() System Call

Creates a **new process** by duplicating the calling process.

```c
/* User space */
pid_t pid = fork();
if (pid == 0) {
    /* Child process */
    execvp("ls", argv);
} else if (pid > 0) {
    /* Parent process */
    waitpid(pid, &status, 0);
} else {
    perror("fork failed");
}
```

**Kernel path**: `sys_fork()` → `kernel_clone(SIGCHLD)` → `copy_process()`

Return values: child gets 0, parent gets child PID, -1 on error.

---

## 7.2 vfork() System Call

Parent **blocks** until child calls `exec()` or `_exit()`. Child uses parent's address space directly (no COW).

```c
pid_t pid = vfork();
if (pid == 0) {
    execvp("ls", argv);  /* MUST exec or _exit() */
    _exit(1);             /* Never return from child! */
}
/* Parent resumes here after child execs */
```

**Kernel path**: `kernel_clone(CLONE_VFORK | CLONE_VM | SIGCHLD)`

**Why it exists**: On old systems without COW, fork was expensive. Today, vfork is rarely needed — `posix_spawn()` is preferred.

---

## 7.3 clone() System Call

The **generic** process/thread creation primitive.

```c
/* clone() flags control what is shared */
int clone(int (*fn)(void *), void *stack, int flags, void *arg);
```

| Flag | Effect |
|------|--------|
| `CLONE_VM` | Share address space (threads) |
| `CLONE_FILES` | Share file descriptor table |
| `CLONE_FS` | Share filesystem info (cwd, root) |
| `CLONE_SIGHAND` | Share signal handlers |
| `CLONE_THREAD` | Same thread group (same TGID) |
| `CLONE_NEWPID` | New PID namespace |
| `CLONE_NEWNS` | New mount namespace |
| `CLONE_PARENT` | Same parent as caller |
| `CLONE_VFORK` | Parent blocks until child execs |

```
fork()           = clone(SIGCHLD)
pthread_create() = clone(CLONE_VM | CLONE_FILES | CLONE_FS |
                         CLONE_SIGHAND | CLONE_THREAD | ...)
container        = clone(CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET | ...)
```

---

## 7.4 exec() Family of System Calls

Replaces the current process image with a new program.

```c
/* Variants — all call execve() internally */
execl("/bin/ls", "ls", "-la", NULL);       /* List args */
execv("/bin/ls", argv);                     /* Array args */
execlp("ls", "ls", "-la", NULL);           /* Search PATH */
execvp("ls", argv);                         /* Search PATH + array */
execve("/bin/ls", argv, envp);             /* Full control */
execvpe("ls", argv, envp);                  /* PATH + envp */
```

### What exec() Does

```
Before exec:                    After exec:
┌─────────────────┐            ┌─────────────────┐
│ PID: 1234       │            │ PID: 1234       │  (same)
│ Text: parent    │            │ Text: /bin/ls   │  (replaced)
│ Data: parent    │            │ Data: /bin/ls   │  (replaced)
│ Stack: parent   │            │ Stack: new      │  (replaced)
│ Heap: parent    │            │ Heap: empty     │  (reset)
│ FDs: inherited  │            │ FDs: inherited* │  (close-on-exec)
│ PID/PPID: same  │            │ PID/PPID: same  │  (same)
│ Signals: custom │            │ Signals: SIG_DFL│  (reset)
│ cwd: /home/user │            │ cwd: /home/user │  (same)
└─────────────────┘            └─────────────────┘
```

---

## 7.5 Process Duplication — copy_process() Internals

```c
/* kernel/fork.c — simplified copy_process() */
static struct task_struct *copy_process(...)
{
    struct task_struct *p;
    int retval;

    /* 1. Allocate task_struct and kernel stack */
    p = dup_task_struct(current);

    /* 2. Initialize scheduling */
    retval = sched_fork(clone_flags, p);
    /* Sets p->prio, p->sched_class, initial vruntime */

    /* 3. Copy/share resources based on clone flags */
    retval = copy_files(clone_flags, p);   /* CLONE_FILES → share */
    retval = copy_fs(clone_flags, p);      /* CLONE_FS → share */
    retval = copy_sighand(clone_flags, p); /* CLONE_SIGHAND → share */
    retval = copy_signal(clone_flags, p);  /* CLONE_THREAD → share */
    retval = copy_mm(clone_flags, p);      /* CLONE_VM → share */
    retval = copy_namespaces(clone_flags, p);
    retval = copy_thread(p, args);         /* Set up kernel stack */

    /* 4. Allocate PID */
    p->pid = pid_nr(alloc_pid(p->nsproxy->pid_ns_for_children, ...));

    /* 5. Set parent-child links */
    p->real_parent = current;
    list_add_tail(&p->sibling, &current->children);

    return p;
}
```

---

## 7.6 Copy-on-Write (COW) During fork()

```
Before fork:
  Parent virtual page → Physical page P1 [R/W]

After fork (COW):
  Parent virtual page → Physical page P1 [R/O]  ← Read-Only
  Child virtual page  → Physical page P1 [R/O]  ← Same physical page!
  refcount(P1) = 2

On write (by either parent or child):
  1. Page fault → handle_mm_fault() → do_wp_page()
  2. Allocate new page P2
  3. Copy P1 → P2
  4. Writer gets P2 [R/W]
  5. Other keeps P1, refcount decremented
```

```
Step by step:
                                                       
 fork()                Parent writes               
 ┌──────┐            ┌──────┐  ┌──────┐           
 │Parent│──P1(RO)──►│P1    │  │P1    │◄──Child    
 │Child │──P1(RO)──►│(shared)  │(RO)  │           
 └──────┘            └──────┘  └──────┘           
                                 ▲                 
                     ┌──────┐   │                  
                     │P2    │   │                  
                     │(new) │   Child still uses P1
                     └──────┘                      
                       ▲                           
                     Parent now has P2 (R/W)       
```

### Why COW Matters

```
Without COW: fork() copies ALL memory → slow for large processes
With COW:    fork() only copies page tables → fast
             Only pages that are written get duplicated
             fork+exec: almost zero copying (exec replaces everything)
```

```c
/* kernel/fork.c — copy_mm with CLONE_VM */
static int copy_mm(unsigned long clone_flags, struct task_struct *tsk)
{
    struct mm_struct *mm;

    if (clone_flags & CLONE_VM) {
        /* Thread: share parent's mm */
        mmget(current->mm);
        tsk->mm = current->mm;
        return 0;
    }

    /* Process: duplicate mm with COW */
    mm = dup_mm(tsk, current->mm);
    tsk->mm = mm;
    return 0;
}
```

---

## Interview Questions

**Q1: Explain the complete path from `fork()` to a running child process.**
A: `fork()` → `sys_fork()` → `kernel_clone()` → `copy_process()` (allocate task_struct, copy/share resources per flags, set up scheduling, allocate PID) → `wake_up_new_task()` (enqueue child on run queue, check preemption). Child returns 0 from fork; parent returns child's PID.

**Q2: Why is fork() efficient despite duplicating the entire process?**
A: Copy-on-write (COW). After fork, parent and child share all physical pages (marked read-only). Pages are only copied when written. Since most forks are followed by exec() (which discards all pages), the actual copying is minimal.

**Q3: How does the kernel make fork() return different values to parent and child?**
A: `copy_thread()` sets up the child's kernel stack so that when it's scheduled for the first time, it returns 0 (via `ret_from_fork` assembly code). The parent's kernel_clone() returns the child's PID normally. The two task_structs have separate kernel stacks with different return values.

**Q4: What clone() flags does pthread_create() use?**
A: `CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD | CLONE_SYSVSEM | CLONE_SETTLS | CLONE_PARENT_SETTID | CLONE_CHILD_CLEARTID`. This shares VM, files, signals, and creates a thread within the same thread group (same TGID).

---

*Next: [Chapter 8 — Threads in Linux](Chapter_08_Threads.md)*
