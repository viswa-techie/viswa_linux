# Chapter 11: Synchronization Primitives

## Learning Goals
- Master mutex, binary semaphore, counting semaphore, and their differences
- Understand recursive mutexes and deadlock avoidance
- Learn spinlocks and their RTOS role (SMP)
- Know reader-writer locks and condition variables
- Understand critical sections and interrupt masking
- Compare synchronization across RTOS platforms

---

## 1. Mutex vs Semaphore

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  MUTEX (Mutual Exclusion):                               │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ · Protects shared resource (ownership concept)  │      │
  │  │ · Only the OWNER can release (unlock)           │      │
  │  │ · Supports priority inheritance                 │      │
  │  │ · Can be recursive (same task locks multiple)   │      │
  │  │ · Use for: protecting data structures, HW regs  │      │
  │  │ · Analogy: bathroom key — only holder returns it│      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  BINARY SEMAPHORE:                                       │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ · Signaling mechanism (no ownership)            │      │
  │  │ · ANY task/ISR can give (signal)                │      │
  │  │ · NO priority inheritance                       │      │
  │  │ · Value: 0 or 1                                 │      │
  │  │ · Use for: ISR-to-task signaling, event flags   │      │
  │  │ · Analogy: traffic light — anyone can change it │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  COUNTING SEMAPHORE:                                     │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ · Value: 0 to N (counts available resources)    │      │
  │  │ · Give = increment, Take = decrement            │      │
  │  │ · Blocks when count reaches 0                   │      │
  │  │ · Use for: resource pools, event counting       │      │
  │  │ · Example: 3 UART ports, sem count = 3          │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  Common MISTAKE:                                         │
  │  Using binary semaphore for mutual exclusion             │
  │  → No priority inheritance → priority inversion risk!    │
  │  ALWAYS use mutex for protecting shared resources.       │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Mutex Implementation

```c
/* FreeRTOS Mutex */

/* Create a mutex */
SemaphoreHandle_t xMutex = xSemaphoreCreateMutex();
/* Starts in "given" state (available) */

/* Create a recursive mutex (same task can lock multiple times) */
SemaphoreHandle_t xRecMutex = xSemaphoreCreateRecursiveMutex();

/* Lock (take) a mutex */
void vSharedResourceTask(void *pvParameters) {
    for (;;) {
        /* Attempt to take mutex with 100ms timeout */
        if (xSemaphoreTake(xMutex, pdMS_TO_TICKS(100)) == pdTRUE) {
            /* === CRITICAL SECTION START === */
            /* Safe to access shared resource */
            shared_buffer[idx++] = new_data;
            update_global_state();
            /* === CRITICAL SECTION END === */

            xSemaphoreGive(xMutex);  /* Release mutex */
        } else {
            /* Could not obtain mutex within 100ms */
            handle_timeout();
        }
    }
}

/*
 * Mutex internals (FreeRTOS):
 *
 * When task takes a mutex:
 * 1. If available: task becomes owner, pxMutexHolder = pxCurrentTCB
 * 2. If held by another: requesting task blocks on xTasksWaitingToReceive
 *    AND if requester has higher priority → priority inheritance triggered
 *
 * Priority Inheritance:
 * ┌─────────────────────────────────────────────────┐
 * │ Task H (prio 5) wants mutex held by Task L (prio 1) │
 * │ → Task L temporarily raised to priority 5        │
 * │ → Task L runs at prio 5 until it releases mutex  │
 * │ → Task L reverts to original priority 1           │
 * │ → Task H gets mutex and runs                      │
 * └─────────────────────────────────────────────────┘
 *
 * Stored in TCB:
 *   uxBasePriority: original priority (never changes)
 *   uxPriority:     current (possibly inherited) priority
 *   uxMutexesHeld:  count of mutexes held by this task
 */

/* Recursive mutex: same task can take multiple times */
void vNestedLockFunction(void) {
    xSemaphoreTakeRecursive(xRecMutex, portMAX_DELAY);
    /* Do work... */
    helper_function();  /* This also takes the same mutex */
    xSemaphoreGiveRecursive(xRecMutex);
}

void helper_function(void) {
    xSemaphoreTakeRecursive(xRecMutex, portMAX_DELAY); /* OK! Same task */
    /* Access shared resource */
    xSemaphoreGiveRecursive(xRecMutex);
}
/* Each Take must have matching Give — internal counter tracks depth */
```

---

## 3. Critical Sections

```c
/* FreeRTOS Critical Section Methods */

/* Method 1: taskENTER_CRITICAL / taskEXIT_CRITICAL */
/* Disables interrupts up to configMAX_SYSCALL_INTERRUPT_PRIORITY */
taskENTER_CRITICAL();
{
    /* Interrupts at/below RTOS priority: DISABLED */
    /* Interrupts above RTOS priority: STILL ENABLED */
    /* Scheduler: DISABLED (no preemption) */
    shared_counter++;
    shared_flag = 1;
}
taskEXIT_CRITICAL();
/* Nestable: maintains count, only re-enables on final EXIT */

/* Method 2: taskENTER_CRITICAL_FROM_ISR */
/* For use inside ISR — saves/restores BASEPRI */
void SomeISR(void) {
    UBaseType_t uxSavedStatus = taskENTER_CRITICAL_FROM_ISR();
    /* Protected code */
    taskEXIT_CRITICAL_FROM_ISR(uxSavedStatus);
}

/* Method 3: Suspend scheduler (no interrupt disable) */
vTaskSuspendAll();
{
    /* Interrupts: STILL ENABLED (ISRs can fire) */
    /* Scheduler: DISABLED (no context switch) */
    /* Use when: operation takes too long to disable interrupts */
    process_large_data_structure();
}
xTaskResumeAll();  /* Returns pdTRUE if context switch needed */

/*
 * Comparison:
 * ┌──────────────────┬────────────┬───────────┬──────────────┐
 * │ Method           │ Interrupts │ Scheduler │ Use When     │
 * ├──────────────────┼────────────┼───────────┼──────────────┤
 * │ ENTER_CRITICAL   │ Masked     │ Disabled  │ Very short   │
 * │                  │ (BASEPRI)  │           │ operations   │
 * ├──────────────────┼────────────┼───────────┼──────────────┤
 * │ SuspendAll       │ Enabled    │ Disabled  │ Longer ops   │
 * │                  │            │           │ (need ISRs)  │
 * ├──────────────────┼────────────┼───────────┼──────────────┤
 * │ Mutex            │ Enabled    │ Enabled   │ Shared data  │
 * │                  │            │           │ between tasks│
 * └──────────────────┴────────────┴───────────┴──────────────┘
 *
 * Rule: use the LEAST restrictive method that is sufficient
 */
```

---

## 4. Deadlock

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Deadlock: Two or more tasks waiting for each other      │
  │                                                           │
  │  ┌──────┐  holds MutexA   ┌──────┐                      │
  │  │Task A│────────────────►│MutexA│ (locked by A)         │
  │  │      │  wants MutexB   │      │                       │
  │  │      │─────┐           └──────┘                       │
  │  └──────┘     │                                          │
  │               │ ← DEADLOCK: A waits for B,               │
  │               │              B waits for A                │
  │  ┌──────┐     │           ┌──────┐                       │
  │  │Task B│─────┘           │MutexB│ (locked by B)         │
  │  │      │────────────────►│      │                       │
  │  │      │  holds MutexB   └──────┘                       │
  │  │      │  wants MutexA                                  │
  │  └──────┘                                                │
  │                                                           │
  │  Necessary conditions (Coffman, 1971):                   │
  │  1. Mutual exclusion: resource held exclusively          │
  │  2. Hold and wait: hold one, request another             │
  │  3. No preemption: can't force release                   │
  │  4. Circular wait: A→B→...→A                             │
  │                                                           │
  │  Prevention strategies:                                  │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ 1. Lock ordering: always acquire MutexA before  │      │
  │  │    MutexB (breaks circular wait)                │      │
  │  │ 2. Timeout: xSemaphoreTake(mutex, timeout)      │      │
  │  │    Release held mutexes on timeout              │      │
  │  │ 3. Try-lock: attempt without blocking           │      │
  │  │ 4. Single lock: protect all shared resources    │      │
  │  │    with one mutex (coarse but safe)             │      │
  │  │ 5. Avoid nested locks when possible             │      │
  │  └────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. Spinlocks (SMP Systems)

```c
/*
 * Spinlocks: busy-wait synchronization for multi-core RTOS
 *
 * On single-core RTOS: NOT needed (disable interrupts suffices)
 * On SMP RTOS: needed to synchronize between CPU cores
 *
 * FreeRTOS SMP (v11+), Zephyr SMP, VxWorks SMP use spinlocks
 */

/* Zephyr spinlock (SMP-safe) */
static struct k_spinlock my_lock;

void smp_safe_function(void) {
    k_spinlock_key_t key = k_spin_lock(&my_lock);
    /* Critical section — no other core can enter */
    /* Local core interrupts also disabled */
    shared_counter++;
    k_spin_unlock(&my_lock, key);
}

/*
 * How a spinlock works on ARM (LDREX/STREX):
 *
 * spin_lock:
 *     LDREX  R1, [R0]           ; Load exclusive (R0 = lock addr)
 *     CMP    R1, #0             ; Is lock free?
 *     BNE    spin_lock          ; No → spin (retry)
 *     MOV    R1, #1
 *     STREX  R2, R1, [R0]       ; Try to store 1 (acquire)
 *     CMP    R2, #0             ; STREX succeeded?
 *     BNE    spin_lock          ; No → retry
 *     DMB                        ; Data Memory Barrier
 *     BX     LR                 ; Lock acquired
 *
 * spin_unlock:
 *     MOV    R1, #0
 *     DMB
 *     STR    R1, [R0]           ; Release lock
 *     BX     LR
 *
 * LDREX/STREX: atomic compare-and-swap using CPU exclusive monitor
 * DMB: ensures memory ordering across cores (cache coherency)
 *
 * Spinlock rules:
 * 1. Hold for VERY short time (no blocking calls!)
 * 2. Disable local interrupts while holding
 * 3. Never sleep/block while holding a spinlock
 * 4. Use mutex for longer critical sections
 */

/* FreeRTOS SMP: port-specific spinlock */
/* portENTER_CRITICAL() on SMP uses spinlock + interrupt disable */
/* Different from UP (uniprocessor) where only interrupt disable */
```

---

## 6. Reader-Writer Locks and Condition Variables

```c
/* Reader-Writer Lock (not in FreeRTOS, available in QNX/POSIX) */

/*
 * Multiple readers can hold the lock simultaneously
 * Only one writer at a time, exclusive of readers
 *
 * Use when: reads >> writes (sensor data, config tables)
 */

/* QNX/POSIX reader-writer lock */
pthread_rwlock_t rwlock;
pthread_rwlock_init(&rwlock, NULL);

/* Reader: */
pthread_rwlock_rdlock(&rwlock);
value = shared_config.parameter;  /* Multiple readers OK */
pthread_rwlock_unlock(&rwlock);

/* Writer: */
pthread_rwlock_wrlock(&rwlock);
shared_config.parameter = new_value;  /* Exclusive access */
pthread_rwlock_unlock(&rwlock);

/* Implementing RW lock with FreeRTOS primitives: */
typedef struct {
    SemaphoreHandle_t xWriteMutex;  /* Exclusive writer access */
    SemaphoreHandle_t xReaderMutex; /* Protects reader_count */
    volatile int reader_count;
} RWLock_t;

void rw_read_lock(RWLock_t *rw) {
    xSemaphoreTake(rw->xReaderMutex, portMAX_DELAY);
    rw->reader_count++;
    if (rw->reader_count == 1) {
        xSemaphoreTake(rw->xWriteMutex, portMAX_DELAY); /* First reader blocks writers */
    }
    xSemaphoreGive(rw->xReaderMutex);
}

void rw_read_unlock(RWLock_t *rw) {
    xSemaphoreTake(rw->xReaderMutex, portMAX_DELAY);
    rw->reader_count--;
    if (rw->reader_count == 0) {
        xSemaphoreGive(rw->xWriteMutex); /* Last reader allows writers */
    }
    xSemaphoreGive(rw->xReaderMutex);
}

/* Condition Variables (POSIX / QNX) */
pthread_mutex_t mutex;
pthread_cond_t cond;

/* Waiter: */
pthread_mutex_lock(&mutex);
while (!condition_met) {
    pthread_cond_wait(&cond, &mutex);  /* Atomically unlock + wait */
}
/* Process condition */
pthread_mutex_unlock(&mutex);

/* Signaler: */
pthread_mutex_lock(&mutex);
condition_met = 1;
pthread_cond_signal(&cond);  /* Wake one waiter */
/* pthread_cond_broadcast(&cond); */  /* Wake ALL waiters */
pthread_mutex_unlock(&mutex);
```

---

## 7. Synchronization Comparison

| Primitive | FreeRTOS | Zephyr | QNX | VxWorks |
|-----------|----------|--------|-----|---------|
| **Mutex** | `xSemaphoreCreateMutex()` | `k_mutex_lock()` | `pthread_mutex_lock()` | `semMCreate()` |
| **Binary Sem** | `xSemaphoreCreateBinary()` | `k_sem_init(,0,1)` | `sem_init(,0,0)` | `semBCreate()` |
| **Counting Sem** | `xSemaphoreCreateCounting()` | `k_sem_init(,0,N)` | `sem_init(,0,N)` | `semCCreate()` |
| **Priority Inh.** | Mutex only | Mutex (configurable) | PTHREAD_PRIO_INHERIT | Yes |
| **Recursive Mutex** | `CreateRecursiveMutex()` | `k_mutex` (default recursive) | PTHREAD_MUTEX_RECURSIVE | semMCreate(SEM_INVERSION_SAFE) |
| **RW Lock** | Manual | `k_rwlock` (planned) | `pthread_rwlock` | `rwlock_init()` |
| **Spinlock** | SMP port | `k_spinlock` | SpinLock API | spinLockISR |
| **Condition Var** | Not native | `k_condvar` | `pthread_cond` | `condVarCreate()` |

---

## Interview Questions

**Q1: What is the difference between a mutex and a binary semaphore? When would you use each?**
**A:** Mutex has OWNERSHIP — only the task that locked it can unlock it. Binary semaphore has no ownership — any task/ISR can give it. Mutex supports priority inheritance — if a high-priority task waits for a mutex held by a low-priority task, the low-priority task's priority is temporarily raised. Binary semaphore does NOT do this. Use mutex for: protecting shared resources (data structures, hardware registers, files). Use binary semaphore for: signaling between tasks or ISR-to-task notification (event occurred). Common mistake: using binary semaphore to protect shared data — works functionally but risks priority inversion because there's no inheritance mechanism.

**Q2: How does priority inheritance work in FreeRTOS mutexes?**
**A:** When Task H (high priority) calls `xSemaphoreTake()` on a mutex held by Task L (low priority): (1) FreeRTOS checks `pxMutexHolder` — it's Task L. (2) Task L's `uxPriority` is raised to Task H's priority. (3) Task L is moved to the appropriate ready list for its new priority. (4) Task L runs at elevated priority until it calls `xSemaphoreGive()`. (5) On give: Task L's `uxPriority` is restored to `uxBasePriority`. (6) Task H is unblocked and runs (it's the highest priority again). This prevents unbounded priority inversion where a medium-priority task could preempt Task L indefinitely, blocking Task H. Limitation: FreeRTOS only supports single-level inheritance (not transitive across chains of mutexes).

**Q3: Explain deadlock. How would you prevent it in an RTOS design?**
**A:** Deadlock occurs when two or more tasks are permanently blocked, each waiting for a resource held by the other. All four Coffman conditions must hold: mutual exclusion, hold-and-wait, no preemption, circular wait. Prevention strategies for RTOS: (1) Lock ordering — define a global order for all mutexes; always acquire in ascending order (breaks circular wait). (2) Timeouts — use `xSemaphoreTake(mutex, timeout)` instead of blocking forever; on timeout, release all held mutexes and retry. (3) Single lock — use one mutex for all related shared resources (coarse-grained but deadlock-free). (4) Try-lock pattern — attempt second lock without blocking; if fail, release first lock, back off, retry. (5) Design review — minimize shared state; prefer message-passing (queues) over shared memory.

**Q4: When should you use taskENTER_CRITICAL() vs a mutex?**
**A:** `taskENTER_CRITICAL()` disables RTOS-managed interrupts and prevents context switching. It's suitable for very short operations (a few instructions) that must be atomic. It blocks ALL tasks and RTOS-managed ISRs, so it affects system-wide responsiveness. Use it for: accessing shared variables written by ISRs, updating linked lists accessed from ISR context, any ISR-task shared state. Mutex only prevents other tasks from entering the protected section — ISRs still fire, other tasks still run (just can't take the same mutex). Use mutex for: longer operations, task-only sharing, when interrupt latency must be preserved. Rule of thumb: if the critical section is >1-2μs, use a mutex. If it's ISR-shared data that takes a few cycles, use ENTER_CRITICAL.

---

## Summary

- Mutex: ownership + priority inheritance — use for protecting shared resources
- Binary semaphore: signaling — use for ISR-to-task notification
- Counting semaphore: resource counting — N available resources
- Critical sections: taskENTER_CRITICAL for very short, ISR-shared data
- Suspend scheduler: interrupt-safe but prevents context switching
- Deadlock: prevent with lock ordering, timeouts, or design simplification
- Spinlocks: SMP only — busy-wait with LDREX/STREX, very short hold times
- Recursive mutex: same task can lock multiple times (for nested calls)

---

[Previous Chapter: Inter-Task Communication ←](Chapter_10_IPC.md) | [Next Chapter: Priority Inversion →](Chapter_12_Priority_Inversion.md)
