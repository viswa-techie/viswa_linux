# Chapter 9: Kernel Threads

## Learning Goals
- Understand kernel thread architecture and purpose
- Master the kthread API
- Know common kernel threads and their roles

---

## 9.1 Kernel Thread Architecture

Kernel threads are tasks that run entirely in kernel space with no user-space component.

```
User Process                    Kernel Thread
┌───────────────────┐          ┌───────────────────┐
│ User space code   │          │ (no user space)    │
│ + kernel space    │          │ mm = NULL           │
│ mm → user pages   │          │ active_mm → borrowed│
│ files → user's    │          │ files = NULL        │
│ PID visible in ps │          │ [name] in ps        │
└───────────────────┘          └───────────────────┘
```

Key properties:
- `mm = NULL` — no user address space
- Runs in kernel context — can call any kernel function
- Cannot access user-space memory (`copy_from_user()` would fail)
- Created as children of `kthreadd` (PID 2)

---

## 9.2 Kernel Worker Threads

### Important Kernel Threads

| Thread | Purpose |
|--------|---------|
| `[kthreadd]` | Parent of all kernel threads (PID 2) |
| `[ksoftirqd/N]` | Process softirqs on CPU N |
| `[migration/N]` | Handle task migration to CPU N |
| `[kworker/N:X]` | Workqueue worker on CPU N, instance X |
| `[kswapd0]` | Memory reclaim (page swap) |
| `[khugepaged]` | Transparent huge page compaction |
| `[kcompactd0]` | Memory compaction |
| `[rcu_preempt]` | RCU grace period management |
| `[irq/N-name]` | Threaded IRQ handler |
| `[kblockd]` | Block device work |
| `[oom_reaper]` | Kill and reclaim from OOM victims |

### Workqueue Workers

```
Work item submitted:  schedule_work(&my_work);
         │
         ▼
  ┌──────────────────┐
  │  Workqueue        │
  │  (per-CPU pool)   │
  │                   │
  │  [kworker/0:0] ◄── picks work item
  │  [kworker/0:1]    │
  │  [kworker/1:0]    │
  └──────────────────┘
```

---

## 9.3 kthread API

### Creating Kernel Threads

```c
#include <linux/kthread.h>

/* Method 1: Create + start in one step */
struct task_struct *task = kthread_run(my_func, data, "my-thread-%d", id);

/* Method 2: Create then wake separately */
struct task_struct *task = kthread_create(my_func, data, "my-thread");
if (!IS_ERR(task))
    wake_up_process(task);
```

### Thread Function Pattern

```c
static int my_kthread_fn(void *data)
{
    struct my_device *dev = data;

    /* Optional: set scheduling policy */
    /* sched_set_fifo(current);  for RT priority */

    while (!kthread_should_stop()) {
        /* Wait for work */
        wait_event_interruptible(dev->wq, dev->data_ready ||
                                 kthread_should_stop());

        if (kthread_should_stop())
            break;

        /* Process data */
        process_data(dev);
        dev->data_ready = false;
    }

    return 0;  /* Thread exit code */
}
```

### Stopping Kernel Threads

```c
/* From another context (e.g., driver remove): */
kthread_stop(task);
/* 1. Sets kthread_should_stop() = true for the target thread
   2. Wakes the thread if sleeping
   3. Waits for the thread to exit
   4. Returns the thread's exit code */
```

### Per-CPU Kernel Threads

```c
/* Create a thread bound to each CPU */
for_each_online_cpu(cpu) {
    struct task_struct *t;
    t = kthread_create(worker_fn, per_cpu_ptr(data, cpu),
                       "worker/%d", cpu);
    kthread_bind(t, cpu);  /* Pin to specific CPU */
    wake_up_process(t);
}
```

---

## 9.4 Kernel Thread Lifecycle

```
kthread_create()
    │
    ▼
kthreadd kernel thread (PID 2)
    │
    ├── Allocate task_struct via kernel_clone()
    ├── Set mm = NULL (no user space)
    ├── Set function pointer and data
    │
    ▼
wake_up_process()
    │
    ▼
Thread runs my_func(data)
    │
    ├── while (!kthread_should_stop()) {
    │       do_work();
    │       schedule();
    │   }
    │
    ▼
kthread_stop() called from another context
    │
    ├── Sets stop flag
    ├── Wakes thread
    │
    ▼
Thread sees kthread_should_stop() = true
    │
    ├── Returns from my_func()
    │
    ▼
do_exit() → thread terminated
kthread_stop() returns exit code to caller
```

---

## Complete Kernel Thread Example

```c
#include <linux/module.h>
#include <linux/kthread.h>
#include <linux/delay.h>

static struct task_struct *my_thread;
static int counter;

static int thread_fn(void *data)
{
    pr_info("kthread started\n");

    while (!kthread_should_stop()) {
        counter++;
        pr_info("kthread: counter = %d\n", counter);
        msleep(1000);  /* Sleep 1 second */
    }

    pr_info("kthread stopping\n");
    return counter;
}

static int __init my_init(void)
{
    my_thread = kthread_run(thread_fn, NULL, "my-counter");
    if (IS_ERR(my_thread)) {
        pr_err("Failed to create kthread\n");
        return PTR_ERR(my_thread);
    }
    return 0;
}

static void __exit my_exit(void)
{
    int ret = kthread_stop(my_thread);
    pr_info("kthread returned: %d\n", ret);
}

module_init(my_init);
module_exit(my_exit);
MODULE_LICENSE("GPL");
```

---

## Interview Questions

**Q1: What is kthreadd and why is it needed?**
A: `kthreadd` (PID 2) is the parent of all kernel threads. When you call `kthread_create()`, the request is queued and `kthreadd` creates the thread via `kernel_clone()`. This is because thread creation needs to happen in process context (cannot allocate memory in IRQ context), and `kthreadd` provides a clean context for this.

**Q2: How do you make a kernel thread run with real-time priority?**
A: Call `sched_set_fifo(current)` inside the thread function. This sets `SCHED_FIFO` with priority 50. For a specific priority: `sched_set_fifo_low(current)` (priority 1). These replaced the deprecated `sched_setscheduler()` call.

**Q3: What happens if a kernel thread doesn't check `kthread_should_stop()`?**
A: `kthread_stop()` will hang forever because it waits for the thread to exit. The thread must cooperatively check the stop flag and return. There's no forced termination for kernel threads.

---

*Next: [Chapter 10 — Context Switching](Chapter_10_Context_Switching.md)*
