# Chapter 8: Threads in Linux

## Learning Goals
- Understand Linux's unified thread model (threads = lightweight processes)
- Differentiate user threads, kernel threads, and POSIX threads
- Know how pthread_create maps to kernel clone()

---

## 8.1 Thread Concept

A thread is an execution context that shares an address space with other threads in the same process.

```
Process (single address space)
┌───────────────────────────────────────┐
│  Shared: code, data, heap, files      │
│                                       │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ │
│  │ Thread 1│ │ Thread 2│ │ Thread 3│ │
│  │ Stack   │ │ Stack   │ │ Stack   │ │
│  │ Regs    │ │ Regs    │ │ Regs    │ │
│  │ TID 100 │ │ TID 101 │ │ TID 102 │ │
│  └─────────┘ └─────────┘ └─────────┘ │
│  TGID = 100 (all threads share this) │
└───────────────────────────────────────┘
```

---

## 8.2 User Threads vs Kernel Threads

| Feature | User Thread | Kernel Thread |
|---------|------------|--------------|
| Scheduled by | User-space library | Kernel scheduler |
| Kernel knows | No (N:1) or Yes (1:1) | Yes |
| Blocking call | Blocks entire process (N:1) | Only blocks that thread |
| Linux model | 1:1 (NPTL) | 1:1 |
| Context switch | Fast (user space only) | Full (kernel involved) |

### Linux Threading Model: 1:1

Every user thread has a corresponding kernel task_struct:

```
User Thread        Kernel
┌──────────┐      ┌──────────────┐
│ pthread 1│ ←──► │ task_struct 1│
│ pthread 2│ ←──► │ task_struct 2│
│ pthread 3│ ←──► │ task_struct 3│
└──────────┘      └──────────────┘

1:1 mapping — kernel schedules each thread individually
```

---

## 8.3 Linux Threading Model — NPTL

**NPTL (Native POSIX Thread Library)** — the standard Linux threading implementation since glibc 2.3.

```
History:
  LinuxThreads (old) → signals broken, getpid() returned different PIDs
  NPTL (2003+)       → proper POSIX semantics, 1:1, uses futex()
```

NPTL uses `clone()` with flags:
```c
clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND |
      CLONE_THREAD | CLONE_SYSVSEM | CLONE_SETTLS |
      CLONE_PARENT_SETTID | CLONE_CHILD_CLEARTID, stack);
```

---

## 8.4 POSIX Threads (pthreads) Implementation

```c
#include <pthread.h>

void *worker(void *arg)
{
    int id = *(int *)arg;
    printf("Thread %d running on CPU %d\n", id, sched_getcpu());
    return NULL;
}

int main(void)
{
    pthread_t threads[4];
    int ids[4];

    for (int i = 0; i < 4; i++) {
        ids[i] = i;
        pthread_create(&threads[i], NULL, worker, &ids[i]);
    }

    for (int i = 0; i < 4; i++)
        pthread_join(threads[i], NULL);

    return 0;
}
```

### pthread_create() Internals

```
pthread_create()
    │
    ├── Allocate thread stack (mmap, default 8MB with guard page)
    ├── Set up TLS (Thread Local Storage)
    │
    ▼
clone(CLONE_VM | CLONE_FILES | ... | CLONE_THREAD, new_stack)
    │
    ▼ (in kernel)
kernel_clone() → copy_process()
    │
    ├── task_struct allocated (separate from parent)
    ├── mm SHARED (CLONE_VM — same address space)
    ├── files SHARED (CLONE_FILES — same fd table)
    ├── signals SHARED (CLONE_SIGHAND)
    ├── TGID = parent's TGID (CLONE_THREAD)
    ├── PID = new unique TID
    │
    ▼
wake_up_new_task() → thread on run queue
```

### Thread vs Process — Kernel View

```c
/* Checking in kernel code */
bool is_thread = (p->tgid != p->pid);
/* Thread: tgid != pid (shares tgid with group leader) */
/* Single-threaded process: tgid == pid */

/* Count threads in a process */
int thread_count = get_nr_threads(current);

/* Iterate threads */
struct task_struct *t;
for_each_thread(current, t) {
    pr_info("Thread: %s TID=%d\n", t->comm, t->pid);
}
```

---

## 8.5 Kernel Threads

Kernel threads run entirely in kernel space (no user address space: `mm = NULL`).

```bash
$ ps aux | grep '\[.*\]'
root     2  0.0  0.0  0  0 ? S  [kthreadd]         # Thread parent
root     3  0.0  0.0  0  0 ? I  [rcu_gp]
root    11  0.0  0.0  0  0 ? S  [migration/0]       # CPU migration
root    12  0.0  0.0  0  0 ? S  [ksoftirqd/0]       # Softirq handling
root    14  0.0  0.0  0  0 ? I  [kworker/0:0]       # Workqueue worker
root    20  0.0  0.0  0  0 ? S  [kswapd0]           # Memory reclaim
root    21  0.0  0.0  0  0 ? S  [kblockd]           # Block device
```

Square brackets `[]` in ps indicate kernel threads (no executable path).

```c
/* Creating a kernel thread */
struct task_struct *kthread_create(int (*fn)(void *data),
                                   void *data,
                                   const char namefmt[], ...);

/* kthread must call kthread_should_stop() and return when true */
int my_thread_fn(void *data)
{
    while (!kthread_should_stop()) {
        /* Do work */
        schedule();  /* Yield CPU */
    }
    return 0;
}

/* Usage */
struct task_struct *t = kthread_run(my_thread_fn, NULL, "my-kthread");
/* To stop: */
kthread_stop(t);
```

---

## Thread Synchronization Summary

| Mechanism | Kernel Space | User Space (pthreads) |
|-----------|-------------|----------------------|
| Mutex | `struct mutex` | `pthread_mutex_t` |
| Spinlock | `spinlock_t` | `pthread_spinlock_t` |
| Condition var | `wait_queue_head_t` | `pthread_cond_t` |
| Barrier | `struct completion` | `pthread_barrier_t` |
| Read-write lock | `rwlock_t` | `pthread_rwlock_t` |

---

## Interview Questions

**Q1: How does Linux implement threads differently from Windows?**
A: Linux: threads are `task_struct` with shared VM/files/signals (lightweight processes). No separate "thread" kernel object. Windows: separate `ETHREAD` structure embedded in `EPROCESS`. Linux's unified approach simplifies the scheduler — it treats all tasks equally.

**Q2: What is the difference between `pthread_create()` and `clone()`?**
A: `pthread_create()` is a user-space library function (NPTL) that: allocates a stack, sets up TLS, then calls `clone()` with thread-specific flags. `clone()` is the syscall that actually creates the new task_struct in the kernel. `pthread_create` is the POSIX-portable API.

**Q3: Why does a kernel thread have `mm = NULL`?**
A: Kernel threads run only in kernel space — they don't need a user address space. However, they borrow `active_mm` from the previously running task (lazy TLB switch) so the kernel page table is valid. This saves the cost of a full address space.

---

*Next: [Chapter 9 — Kernel Threads](Chapter_09_Kernel_Threads.md)*
