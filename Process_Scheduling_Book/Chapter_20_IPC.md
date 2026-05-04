# Chapter 20: Inter-Process Communication (IPC)

## Learning Goals
- Understand all Linux IPC mechanisms and when to use each
- Master pipes, FIFOs, message queues, shared memory, and sockets
- Learn IPC impact on scheduling (sleep/wake patterns)
- Know signaling mechanisms (covered in depth in Chapter 21)

---

## 20.1 IPC Overview

```
IPC Mechanisms in Linux:

  ┌─────────────────┬──────────┬───────────┬──────────┬────────────┐
  │ Mechanism        │ Related  │ Data flow │ Scope    │ Scheduling │
  │                  │ procs    │           │          │ impact     │
  ├─────────────────┼──────────┼───────────┼──────────┼────────────┤
  │ Pipe             │ Parent↔  │ Byte      │ Same     │ Sleep on   │
  │                  │ child    │ stream    │ ancestry │ empty/full │
  ├─────────────────┼──────────┼───────────┼──────────┼────────────┤
  │ Named pipe(FIFO)│ Any      │ Byte      │ Same host│ Sleep on   │
  │                  │          │ stream    │          │ empty/full │
  ├─────────────────┼──────────┼───────────┼──────────┼────────────┤
  │ Message queue   │ Any      │ Messages  │ Same host│ Sleep on   │
  │ (POSIX/SysV)    │          │ (typed)   │          │ empty/full │
  ├─────────────────┼──────────┼───────────┼──────────┼────────────┤
  │ Shared memory   │ Any      │ Direct    │ Same host│ No sleep   │
  │                  │          │ memory    │          │ (needs sync)│
  ├─────────────────┼──────────┼───────────┼──────────┼────────────┤
  │ Signals          │ Any      │ Async     │ Same host│ Wake target│
  │                  │          │ notify    │          │             │
  ├─────────────────┼──────────┼───────────┼──────────┼────────────┤
  │ Unix sockets    │ Any      │ Stream/   │ Same host│ Sleep on   │
  │                  │          │ datagram  │          │ I/O        │
  ├─────────────────┼──────────┼───────────┼──────────┼────────────┤
  │ TCP/UDP sockets │ Any      │ Stream/   │ Network  │ Sleep on   │
  │                  │          │ datagram  │          │ I/O        │
  ├─────────────────┼──────────┼───────────┼──────────┼────────────┤
  │ eventfd          │ Related  │ Counter   │ Same host│ Wake on    │
  │                  │          │           │          │ write      │
  ├─────────────────┼──────────┼───────────┼──────────┼────────────┤
  │ futex            │ Related  │ Sync only │ Same host│ Sleep/wake │
  │                  │          │           │          │ atomic     │
  └─────────────────┴──────────┴───────────┴──────────┴────────────┘
```

---

## 20.2 Pipes

Unidirectional byte stream between related processes:

```c
/* User space */
#include <unistd.h>

int main(void)
{
    int pipefd[2];   /* [0]=read end, [1]=write end */
    pipe(pipefd);

    pid_t pid = fork();
    if (pid == 0) {
        /* Child: read from pipe */
        close(pipefd[1]);  /* Close write end */
        char buf[128];
        int n = read(pipefd[0], buf, sizeof(buf));  /* BLOCKS if empty */
        write(STDOUT_FILENO, buf, n);
        close(pipefd[0]);
    } else {
        /* Parent: write to pipe */
        close(pipefd[0]);  /* Close read end */
        write(pipefd[1], "Hello\n", 6);  /* BLOCKS if full (64KB default) */
        close(pipefd[1]);
        wait(NULL);
    }
}
```

### Pipe Kernel Implementation

```
Pipe buffer (circular, default 16 pages = 64KB):

  write_pos →  ┌───────────────────────────┐ ← read_pos
               │ DATA DATA DATA ... DATA   │
               └───────────────────────────┘

  Scheduling behavior:
    Reader blocks (TASK_INTERRUPTIBLE) when pipe empty
    Writer blocks (TASK_INTERRUPTIBLE) when pipe full
    write() wakes sleeping readers → try_to_wake_up()
    read() wakes sleeping writers  → try_to_wake_up()

  Kernel path:
    write() → pipe_write() → copy_from_user → wake_up_interruptible()
    read()  → pipe_read()  → copy_to_user   → wake_up_interruptible()
```

### Shell Pipeline Scheduling

```bash
cat large_file | grep pattern | sort | head -10

# Creates 4 processes connected by 3 pipes
# Scheduling interaction:
#   cat writes → pipe1 → wakes grep
#   grep writes → pipe2 → wakes sort
#   Each process blocks when its output pipe is full
#   Back-pressure naturally balances CPU usage

  cat ──pipe──→ grep ──pipe──→ sort ──pipe──→ head
  (produces)    (filters)      (accumulates)   (limits)
```

---

## 20.3 Named Pipes (FIFOs)

```bash
# Create named pipe
mkfifo /tmp/my_fifo

# Writer (blocks until reader opens):
echo "data" > /tmp/my_fifo

# Reader (blocks until writer opens):
cat /tmp/my_fifo
```

```c
/* Kernel: fifo_open in fs/pipe.c */
/* Opening FIFO blocks until BOTH reader and writer exist */
/* Same pipe implementation underneath */
```

---

## 20.4 POSIX Message Queues

```c
#include <mqueue.h>
#include <fcntl.h>

/* Sender */
void sender(void)
{
    struct mq_attr attr = {
        .mq_maxmsg = 10,      /* Max messages in queue */
        .mq_msgsize = 256,    /* Max message size */
    };
    mqd_t mq = mq_open("/myqueue", O_CREAT | O_WRONLY, 0644, &attr);
    
    mq_send(mq, "Hello", 5, 1);  /* priority 1 — BLOCKS if full */
    mq_close(mq);
}

/* Receiver */
void receiver(void)
{
    mqd_t mq = mq_open("/myqueue", O_RDONLY);
    char buf[256];
    unsigned int prio;
    
    ssize_t n = mq_receive(mq, buf, 256, &prio);  /* BLOCKS if empty */
    printf("Received: %.*s (prio %u)\n", (int)n, buf, prio);
    mq_close(mq);
    mq_unlink("/myqueue");
}
```

### Scheduling Impact

```
Message queue sleep/wake:

  mq_send() to full queue:
    → sender sleeps (TASK_INTERRUPTIBLE)
    → woken when receiver calls mq_receive()

  mq_receive() from empty queue:
    → receiver sleeps (TASK_INTERRUPTIBLE)
    → woken when sender calls mq_send()

  Messages delivered by PRIORITY (highest first)
  Within same priority: FIFO order
```

---

## 20.5 Shared Memory

Fastest IPC — direct memory access, no kernel copy:

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>

/* POSIX shared memory */
/* Process A: create and write */
int fd = shm_open("/myshm", O_CREAT | O_RDWR, 0644);
ftruncate(fd, 4096);
void *ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
strcpy(ptr, "Shared data!");
/* Need external sync (semaphore/futex) to notify B */

/* Process B: read */
int fd = shm_open("/myshm", O_RDONLY, 0);
void *ptr = mmap(NULL, 4096, PROT_READ, MAP_SHARED, fd, 0);
printf("%s\n", (char *)ptr);  /* Reads "Shared data!" */
```

### Scheduling Impact

```
Shared memory has NO built-in scheduling interaction!

  Unlike pipes/messages: no automatic sleep/wake
  Processes must use EXPLICIT synchronization:
    - Semaphore (sem_wait/sem_post)
    - Futex (fast user-space mutex)
    - Atomic variables with futex_wait/futex_wake

  Advantage: ZERO-COPY — fastest possible data transfer
  Disadvantage: Manual synchronization — error-prone
```

---

## 20.6 Futex (Fast User-Space Mutex)

```
Futex: hybrid user-space / kernel synchronization

  Uncontended case (fast path):
    Atomic compare-and-swap in USER SPACE — no syscall!
    
  Contended case (slow path):
    futex() syscall → kernel puts task to sleep

  Used by: pthread_mutex, sem_wait, condition variables
```

```c
/* Conceptual futex usage */
#include <linux/futex.h>
#include <sys/syscall.h>

int futex_val = 0;  /* Shared between processes (in shared memory) */

/* Lock: */
while (1) {
    if (__sync_bool_compare_and_swap(&futex_val, 0, 1))
        break;  /* Got lock — no syscall needed! (fast path) */
    
    /* Contended — sleep in kernel */
    syscall(SYS_futex, &futex_val, FUTEX_WAIT, 1, NULL, NULL, 0);
}

/* Unlock: */
futex_val = 0;
syscall(SYS_futex, &futex_val, FUTEX_WAKE, 1, NULL, NULL, 0);
```

### Futex and Scheduling

```
Futex kernel path:

  FUTEX_WAIT:
    1. Check if *uaddr == expected_val (atomic)
    2. If yes → add task to futex hash queue → sleep (TASK_INTERRUPTIBLE)
    3. If no → return immediately (value changed, retry)

  FUTEX_WAKE:
    1. Find waiters in futex hash queue for this address
    2. Wake up 'n' waiters → try_to_wake_up()
    3. Woken tasks compete for the futex in user space

  No context switch on uncontended path!
  Only enters kernel on contention → minimal scheduling overhead
```

---

## 20.7 Unix Domain Sockets

```c
/* Superior to pipes: bidirectional, multi-client, credential passing */
#include <sys/socket.h>
#include <sys/un.h>

/* Server */
int server_fd = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr = { .sun_family = AF_UNIX, .sun_path = "/tmp/my.sock" };
bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
listen(server_fd, 5);

int client_fd = accept(server_fd, NULL, NULL);  /* BLOCKS until client connects */
char buf[128];
read(client_fd, buf, sizeof(buf));               /* BLOCKS until data arrives */

/* Client */
int fd = socket(AF_UNIX, SOCK_STREAM, 0);
connect(fd, (struct sockaddr *)&addr, sizeof(addr));
write(fd, "Hello", 5);
```

### Socket Scheduling

```
Socket operations and scheduling:

  accept() on empty queue → sleep (TASK_INTERRUPTIBLE)
    Woken by: incoming connection → try_to_wake_up()

  read() on empty socket buffer → sleep
    Woken by: data arrival (write from peer)

  write() to full socket buffer → sleep
    Woken by: reader consuming data (buffer space available)

  select/poll/epoll: sleep until ANY fd is ready
    Single task monitors multiple connections efficiently
    Woken by: any monitored fd becoming ready
```

---

## 20.8 Android Binder — IPC for Mobile

```
Android uses Binder for cross-process communication:

  Features:
    - Synchronous RPC (caller blocks until callee returns)
    - Object-oriented (pass object references across processes)
    - Priority inheritance (caller's priority passes to callee)
    - Death notifications

  Scheduling impact:
    Client calls binder_transaction() → sleeps
    Binder driver wakes target process's binder thread
    Target thread inherits client's priority (avoiding inversion)
    Reply wakes client → client continues

  Priority inheritance in Binder:
    UI thread (prio 120) calls system_server (prio 130)
    → system_server thread temporarily runs at prio 120
    → Prevents jank from priority inversion
```

---

## 20.9 IPC Performance Comparison

```
Benchmark: transferring 1MB between two processes (rough numbers):

  Mechanism          │ Latency (µs) │ Throughput │ Context switches
  ───────────────────┼──────────────┼────────────┼─────────────────
  Shared memory      │     0.1      │  ~10 GB/s  │ 0 (needs sync)
  pipe               │    ~2-5      │  ~3 GB/s   │ 2 (read+write)
  Unix socket        │    ~3-8      │  ~2 GB/s   │ 2+
  POSIX msg queue    │    ~5-10     │  ~1 GB/s   │ 2
  TCP socket (local) │   ~10-20     │  ~1 GB/s   │ 2+
  Binder (Android)   │   ~10-50     │  ~500 MB/s │ 2

  Shared memory is fastest (zero copy) but needs external sync
  Pipes are best for simple producer-consumer
  Unix sockets most versatile for local IPC
```

---

## IPC Scheduling Patterns

```
Common patterns and their scheduling behavior:

1. Producer-Consumer (pipe/msgq):
   Producer writes → Consumer wakes → reads → sleeps
   Natural back-pressure via blocking writes

2. Request-Reply (socket/binder):
   Client sends → sleeps → Server wakes → processes → replies → Client wakes
   Synchronous: at most one active at a time per request

3. Shared Memory + Futex:
   Writer modifies data → futex_wake → Reader wakes → reads
   Lowest latency, most complex to program correctly

4. Event-driven (epoll):
   Single thread monitors many fds → sleeps in epoll_wait
   Woken when any fd is ready → processes all ready fds → back to sleep
   Minimizes thread count and context switches
```

---

## Interview Questions

**Q1: Why is shared memory the fastest IPC but also the most dangerous?**
A: Shared memory requires zero data copying — processes directly read/write the same physical pages via mapped virtual addresses. But: 1) No built-in synchronization — races are easy. 2) No ordering guarantees — needs memory barriers. 3) No access control once mapped — any process with the mapping can corrupt data. 4) Hard to debug — race conditions only manifest under specific timing.

**Q2: How does a pipe cause a context switch?**
A: When a pipe reader reads from an empty pipe, it calls `pipe_read()` which puts the task to sleep (TASK_INTERRUPTIBLE) and calls `schedule()` — context switch #1. When the writer calls `pipe_write()`, it places data in the pipe buffer and calls `wake_up_interruptible()` on the reader, which calls `try_to_wake_up()`. If the reader has higher priority (or is the only runnable task), the current task is preempted — context switch #2.

**Q3: How does Android Binder avoid priority inversion?**
A: Binder implements priority inheritance natively. When a high-priority client (e.g., UI thread at priority 120) makes a binder call to a lower-priority server thread (e.g., system_server at 130), the binder driver boosts the server thread to the client's priority for the duration of the transaction. This ensures the UI-critical work in the server isn't preempted by medium-priority tasks.

---

*Next: [Chapter 21 — Signals and Process Control](Chapter_21_Signals.md)*
