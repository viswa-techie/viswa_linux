# Chapter 4: Process Representation in Linux

## Learning Goals
- Master `task_struct` — the most important data structure in the kernel
- Understand PIDs, namespaces, and the process hierarchy
- See how the kernel tracks every aspect of a running process

---

## 4.1 Process Abstraction in Linux

Every process and thread in Linux is represented by a single structure: **`struct task_struct`** (defined in `include/linux/sched.h`). This is the kernel's complete knowledge about a task.

```
User process "bash" (PID 1234)
         │
         ▼
┌─────────────────────────────────────────────┐
│              struct task_struct              │
│                                             │
│  state           → TASK_RUNNING             │
│  pid             → 1234                     │
│  tgid            → 1234                     │
│  comm[16]        → "bash"                   │
│  *mm             → address space            │
│  *fs             → filesystem info          │
│  *files          → open file table          │
│  *signal         → signal handlers          │
│  thread          → CPU register state       │
│  prio, static_prio → scheduling priority    │
│  se (sched_entity) → CFS scheduling data    │
│  policy          → SCHED_NORMAL             │
│  *parent         → parent task_struct       │
│  children        → list of child tasks      │
│  cpu             → current CPU              │
│  ...             (700+ fields total)        │
└─────────────────────────────────────────────┘
```

---

## 4.2 struct task_struct — Detailed Breakdown

`task_struct` is ~6-8 KB in size and contains everything about a task. Key field groups:

### Identity & State

```c
struct task_struct {
    unsigned int            __state;        /* TASK_RUNNING, etc. */
    pid_t                   pid;            /* Process ID (unique per thread) */
    pid_t                   tgid;           /* Thread Group ID (= PID of main thread) */
    char                    comm[TASK_COMM_LEN]; /* Executable name (16 bytes) */
    unsigned int            flags;          /* PF_KTHREAD, PF_EXITING, etc. */
};
```

### Scheduling

```c
    int                     prio;           /* Dynamic priority (0-139) */
    int                     static_prio;    /* Set by nice value */
    int                     normal_prio;    /* Computed from static_prio */
    unsigned int            rt_priority;    /* RT priority (0-99) */
    unsigned int            policy;         /* SCHED_NORMAL, SCHED_FIFO, etc. */
    struct sched_entity     se;             /* CFS scheduling entity */
    struct sched_rt_entity  rt;             /* RT scheduling entity */
    struct sched_dl_entity  dl;             /* Deadline scheduling entity */
    const struct sched_class *sched_class;  /* stop/dl/rt/fair/idle */
    int                     on_cpu;         /* Currently running on CPU */
    int                     on_rq;          /* On run queue */
```

### Memory

```c
    struct mm_struct        *mm;            /* User address space (NULL for kernel threads) */
    struct mm_struct        *active_mm;     /* Active address space */
```

### File System

```c
    struct fs_struct        *fs;            /* cwd, root directory */
    struct files_struct     *files;         /* Open file descriptor table */
```

### Signals

```c
    struct signal_struct    *signal;        /* Shared signal data */
    struct sighand_struct   *sighand;       /* Signal handlers */
    sigset_t                blocked;        /* Blocked signals */
    struct sigpending       pending;        /* Pending signals */
```

### Process Hierarchy

```c
    struct task_struct      *real_parent;   /* Real parent (fork creator) */
    struct task_struct      *parent;        /* Ptrace parent or real parent */
    struct list_head        children;       /* Child task list */
    struct list_head        sibling;        /* Sibling list linkage */
    struct task_struct      *group_leader;  /* Thread group leader */
```

### CPU Context (Architecture-Specific)

```c
    struct thread_struct    thread;         /* CPU registers, FPU state */
    /* ARM64: struct cpu_context (x19-x28, fp, sp, pc) */
    /* x86_64: sp, ip, fsbase, gsbase, TLS, FPU */
```

---

## 4.3 Kernel Task Descriptors

The kernel accesses `task_struct` via several fast paths:

### current — Getting the Running Task

```c
/* ARM64: stored in register SP_EL0 (repurposed) or TPIDR_EL1 */
static inline struct task_struct *get_current(void)
{
    unsigned long sp_el0;
    asm ("mrs %0, sp_el0" : "=r" (sp_el0));
    return (struct task_struct *)sp_el0;
}
#define current get_current()

/* x86_64: per-CPU variable via GS segment */
DECLARE_PER_CPU(struct task_struct *, current_task);
#define current this_cpu_read_stable(current_task)
```

Usage in kernel code:
```c
/* Everywhere in kernel */
pr_info("Current process: %s (PID %d)\n", current->comm, current->pid);
```

### Kernel Stack

```
┌──────────────────────────┐  High address
│   struct pt_regs         │  ← Saved user registers (syscall entry)
├──────────────────────────┤
│                          │
│   Kernel stack space     │  ← 16KB (ARM64) or 16KB (x86_64)
│   (grows downward)       │
│                          │
├──────────────────────────┤
│   thread_info            │  ← Flags (TIF_NEED_RESCHED, etc.)
└──────────────────────────┘  Low address
```

---

## 4.4 Process Identifiers (PID)

```c
pid_t    pid;    /* Thread ID — unique per task_struct */
pid_t    tgid;   /* Thread Group ID — same for all threads in a process */
```

| Concept | Kernel Field | `getpid()` Returns | `gettid()` Returns |
|---------|-------------|-------------------|-------------------|
| Process ID | `tgid` | tgid | — |
| Thread ID | `pid` | — | pid |

```
Process "firefox" (TGID 500):
  Main thread:   task_struct { pid=500, tgid=500 }
  Worker thread:  task_struct { pid=501, tgid=500 }
  Render thread:  task_struct { pid=502, tgid=500 }
  
  getpid() from any thread → 500 (returns tgid)
  gettid() from worker    → 501 (returns pid)
```

### PID Allocation

```c
/* kernel/pid.c — PID allocation with IDR (radix tree) */
struct pid *alloc_pid(struct pid_namespace *ns, ...);

/* Default PID range: 1 to 32768 (configurable) */
/* cat /proc/sys/kernel/pid_max → 32768 (can increase to 4194304) */
```

---

## 4.5 Process Namespaces

Namespaces allow multiple independent views of system resources.

```
Host PID Namespace:
  PID 1 (init), PID 100 (sshd), PID 200 (container-runtime)

Container PID Namespace:
  ┌─────────────────────────────────┐
  │ PID 1 (container init)          │  ← Host sees this as PID 300
  │ PID 2 (app)                     │  ← Host sees this as PID 301
  │ PID 3 (worker)                  │  ← Host sees this as PID 302
  └─────────────────────────────────┘
```

```c
/* A task can have different PIDs in different namespaces */
struct pid {
    refcount_t count;
    unsigned int level;              /* Namespace depth */
    struct upid numbers[];           /* PID per namespace level */
};

struct upid {
    int nr;                          /* PID value in this namespace */
    struct pid_namespace *ns;        /* The namespace */
};
```

| Namespace | Resource Isolated |
|-----------|------------------|
| PID | Process IDs |
| Mount | Filesystem mounts |
| Network | Network stack |
| UTS | Hostname |
| IPC | System V IPC |
| User | UID/GID mappings |
| Cgroup | Cgroup root |
| Time | Clock offsets (Linux 5.6+) |

---

## 4.6 Process Hierarchy and Parent-Child Relationships

```
PID 0 (swapper/idle) — per-CPU idle task
  │
  └── PID 1 (init/systemd) — first user process
       │
       ├── PID 100 (sshd)
       │    │
       │    ├── PID 200 (bash)
       │    │    │
       │    │    └── PID 300 (vim)
       │    │
       │    └── PID 201 (bash)
       │
       ├── PID 101 (cron)
       │
       └── PID 102 (NetworkManager)
  │
  └── PID 2 (kthreadd) — parent of all kernel threads
       │
       ├── [ksoftirqd/0]
       ├── [kworker/0:0]
       ├── [migration/0]
       └── [rcu_preempt]
```

```c
/* Traversing the process tree */
struct task_struct *task;

/* Iterate children */
list_for_each_entry(task, &current->children, sibling) {
    pr_info("Child: %s (PID %d)\n", task->comm, task->pid);
}

/* Walk up to init */
for (task = current; task != &init_task; task = task->real_parent)
    pr_info("Ancestor: %s (PID %d)\n", task->comm, task->pid);
```

### Orphan and Zombie Handling

```
Parent exits before child:
  Child becomes "orphan" → reparented to init (PID 1) or subreaper
  init calls wait() on orphans → zombie cleaned up

Child exits before parent calls wait():
  Child becomes "zombie" (Z state) → task_struct still exists
  Only freed when parent calls waitpid()
```

```c
/* Reparenting orphans: kernel/exit.c */
static void forget_original_parent(struct task_struct *father, ...)
{
    struct task_struct *p, *reaper;

    reaper = find_child_reaper(father, ...);
    list_for_each_entry(p, &father->children, sibling) {
        p->real_parent = reaper;
        /* ... */
    }
}
```

---

## Kernel Source References

| File | Content |
|------|---------|
| `include/linux/sched.h` | `task_struct` definition |
| `include/linux/sched/task.h` | Task management helpers |
| `include/linux/pid.h` | PID structures |
| `kernel/pid.c` | PID allocation |
| `kernel/fork.c` | `copy_process()` — task_struct creation |
| `kernel/nsproxy.c` | Namespace management |
| `fs/proc/base.c` | `/proc/[pid]/` entries |

---

## Interview Questions

**Q1: What is `task_struct` and approximately how large is it?**
A: `task_struct` is the kernel's complete descriptor for a process/thread. It's ~6-8 KB and contains: state, PID, scheduling info, memory descriptor (mm), file table, signal handlers, namespaces, cgroup pointers, CPU register state, and 700+ fields total. Defined in `include/linux/sched.h`.

**Q2: What is the difference between `pid` and `tgid` in `task_struct`?**
A: `pid` is the unique thread ID (what `gettid()` returns). `tgid` is the thread group ID (what `getpid()` returns) — it's the PID of the main thread. All threads in a process share the same `tgid`. This allows POSIX semantics: `getpid()` returns the same value for all threads.

**Q3: How does the kernel access the current task's `task_struct`?**
A: Via the `current` macro. On ARM64, it reads `SP_EL0` register (repurposed to hold `task_struct *`). On x86_64, it reads a per-CPU variable via the GS segment. This is O(1) — a single register read or memory access.

**Q4: What happens when a parent process exits but its child is still running?**
A: The child becomes an orphan. The kernel reparents it to `init` (PID 1) or the nearest ancestor that set `PR_SET_CHILD_SUBREAPER`. `init` periodically calls `wait()` to reap zombies from orphaned children.

---

*Next: [Chapter 5 — Process States](Chapter_05_Process_States.md)*
