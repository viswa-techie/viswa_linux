# Chapter 28: Kernel Source Code for Interrupts

## Learning Goals
- Navigate key kernel source files for interrupt and locking subsystems
- Understand the architecture of kernel/irq/, kernel/locking/, kernel/softirq.c
- Read critical data structures and functions in their source context
- Build a mental map of the interrupt code path from source

---

## 28.1 Source Tree Map

```
linux/
├── kernel/
│   ├── irq/                    ← Core IRQ subsystem
│   │   ├── manage.c            ← request_irq, free_irq, IRQ threading
│   │   ├── handle.c            ← Generic IRQ handling flow
│   │   ├── chip.c              ← irq_chip operations
│   │   ├── irqdesc.c           ← irq_desc allocation & management
│   │   ├── irqdomain.c         ← IRQ domain (hwirq → Linux IRQ mapping)
│   │   ├── proc.c              ← /proc/interrupts
│   │   ├── resend.c            ← IRQ resend logic
│   │   ├── affinity.c          ← IRQ affinity management
│   │   ├── msi.c               ← MSI/MSI-X support
│   │   └── debugfs.c           ← Debug filesystem entries
│   │
│   ├── softirq.c               ← SoftIRQ, tasklet, ksoftirqd
│   ├── workqueue.c             ← Workqueue subsystem
│   │
│   ├── locking/
│   │   ├── spinlock.c          ← Spinlock implementation
│   │   ├── mutex.c             ← Mutex (3-path) implementation
│   │   ├── rtmutex.c           ← RT mutex (priority inheritance)
│   │   ├── rwsem.c             ← Reader-writer semaphore
│   │   ├── semaphore.c         ← Counting semaphore
│   │   ├── lockdep.c           ← Lock dependency validator
│   │   ├── qspinlock.c         ← Queue spinlock (MCS-based)
│   │   └── percpu-rwsem.c      ← Per-CPU reader-writer semaphore
│   │
│   ├── rcu/
│   │   ├── tree.c              ← Tree RCU (main implementation)
│   │   ├── update.c            ← RCU API wrappers
│   │   ├── srcutree.c          ← SRCU implementation
│   │   └── rcu_segcblist.c     ← RCU callback list management
│   │
│   ├── sched/
│   │   └── core.c              ← Scheduler (IRQ return preemption)
│   │
│   ├── time/
│   │   ├── timer.c             ← timer_list (classic timers)
│   │   ├── hrtimer.c           ← High-resolution timers
│   │   └── tick-common.c       ← Tick management
│   │
│   └── watchdog.c              ← Soft/hard lockup detection
│
├── include/
│   ├── linux/
│   │   ├── interrupt.h         ← IRQ API (request_irq, IRQF flags)
│   │   ├── irq.h               ← irq_desc, irq_data, irq_chip
│   │   ├── irqdomain.h         ← IRQ domain API
│   │   ├── spinlock.h          ← Spinlock API
│   │   ├── mutex.h             ← Mutex API
│   │   ├── rwsem.h             ← rw_semaphore API
│   │   ├── semaphore.h         ← Semaphore API
│   │   ├── rcupdate.h          ← RCU API
│   │   ├── rculist.h           ← RCU-protected list API
│   │   ├── workqueue.h         ← Workqueue API
│   │   ├── atomic.h            ← Atomic operations API
│   │   ├── percpu.h            ← Per-CPU API
│   │   ├── completion.h        ← Completion API
│   │   └── preempt.h           ← Preemption control
│   │
│   └── asm-generic/
│       ├── hardirq.h           ← IRQ count macros (in_irq, in_softirq)
│       └── irq.h               ← Generic IRQ defs
│
├── arch/
│   ├── x86/
│   │   ├── kernel/
│   │   │   ├── irq.c           ← x86 IRQ entry
│   │   │   ├── apic/           ← APIC driver
│   │   │   └── idt.c           ← IDT setup
│   │   ├── entry/
│   │   │   └── entry_64.S      ← Low-level IRQ entry (assembly)
│   │   └── include/asm/
│   │       ├── irq.h
│   │       ├── apic.h
│   │       └── idtentry.h
│   │
│   └── arm64/
│       ├── kernel/
│       │   ├── irq.c           ← ARM64 IRQ handling
│       │   └── entry.S         ← Exception vector table (assembly)
│       ├── include/asm/
│       │   └── irq.h
│       └── mm/
│           └── fault.c         ← Page fault handler
│
└── drivers/irqchip/
    ├── irq-gic.c               ← ARM GIC v1/v2
    ├── irq-gic-v3.c            ← ARM GIC v3
    └── irq-gic-v3-its.c        ← GIC ITS (message-based)
```

---

## 28.2 Key Data Structures in Source

### struct irq_desc (include/linux/irqdesc.h)

```c
struct irq_desc {
    struct irq_common_data  irq_common_data;
    struct irq_data         irq_data;
    unsigned int __percpu   *kstat_irqs;      /* Per-CPU IRQ counts */
    irq_flow_handler_t      handle_irq;       /* High-level flow handler */
    struct irqaction        *action;           /* Handler chain */
    unsigned int            status_use_accessors;
    unsigned int            core_internal_state__do_not_mess_with_it;
    unsigned int            depth;            /* Disable depth */
    unsigned int            wake_depth;       /* Wake-up depth */
    unsigned int            tot_count;
    unsigned int            irq_count;
    unsigned long           last_unhandled;
    unsigned int            irqs_unhandled;
    raw_spinlock_t          lock;             /* Per-descriptor lock */
    struct cpumask          *percpu_enabled;
    const struct cpumask    *percpu_affinity;
    const struct cpumask    *affinity_hint;
    unsigned long           threads_handled;
    int                     threads_handled_last;
    wait_queue_head_t       wait_for_threads;
    struct proc_dir_entry   *dir;            /* /proc/irq/N/ */
    const char              *name;
};
```

### struct irqaction (include/linux/interrupt.h)

```c
struct irqaction {
    irq_handler_t           handler;         /* Primary ISR */
    void                   *dev_id;          /* Device data */
    void __percpu          *percpu_dev_id;
    struct irqaction       *next;            /* Shared IRQ chain */
    irq_handler_t           thread_fn;       /* Thread handler */
    struct task_struct     *thread;           /* IRQ thread */
    struct irqaction       *secondary;       /* Secondary action */
    unsigned int            irq;
    unsigned int            flags;           /* IRQF_* */
    unsigned long           thread_flags;
    unsigned long           thread_mask;
    const char             *name;
};
```

---

## 28.3 Critical Code Paths

### request_irq() Flow

```
request_irq()                     [include/linux/interrupt.h]
  └─ request_threaded_irq()       [kernel/irq/manage.c]
       ├─ Allocate irqaction
       ├─ Set handler, thread_fn, flags, dev_id
       ├─ If thread_fn: setup_irq_thread()
       │    └─ kthread_create(irq_thread, ...)
       └─ __setup_irq()
            ├─ Validate flags (shared IRQ compatibility)
            ├─ Attach to irq_desc->action chain
            ├─ Enable IRQ if first action
            └─ If threaded: wake_up_process(thread)
```

### IRQ Entry (x86)

```
Hardware interrupt
  └─ CPU looks up IDT[vector]
       └─ asm_common_interrupt     [arch/x86/entry/entry_64.S]
            └─ common_interrupt()   [arch/x86/kernel/irq.c]
                 └─ handle_irq()
                      └─ generic_handle_irq_desc(desc)
                           └─ desc->handle_irq(desc)
                                └─ handle_fasteoi_irq()  [kernel/irq/chip.c]
                                     └─ handle_irq_event()
                                          └─ handle_irq_event_percpu()
                                               └─ action->handler(irq, dev_id)
```

### IRQ Entry (ARM64)

```
Exception at EL1 (vectors)
  └─ el1h_64_irq               [arch/arm64/kernel/entry.S]
       └─ el1_interrupt()       [arch/arm64/kernel/entry-common.c]
            └─ handle_arch_irq  (= gic_handle_irq)
                 └─ gic_handle_irq()   [drivers/irqchip/irq-gic-v3.c]
                      └─ Read IAR register (get interrupt ID)
                      └─ generic_handle_domain_irq()
                           └─ generic_handle_irq_desc(desc)
                                └─ desc->handle_irq(desc)
```

### Softirq Execution Path

```
irq_exit_rcu()                    [kernel/softirq.c]
  └─ __irq_exit_rcu()
       └─ if (pending softirqs && !in_interrupt())
            └─ invoke_softirq()
                 └─ __do_softirq()
                      ├─ Loop: Execute pending softirq vectors
                      ├─ Limit: 10 restarts or 2ms time
                      ├─ If still pending → wake ksoftirqd
                      └─ Each vector: softirq_vec[i].action(h)
```

---

## 28.4 Locking Source Code

### Spinlock Fast Path

```
From kernel/locking/qspinlock.c:

queued_spin_lock_slowpath():
  1. Fast path: atomic_try_cmpxchg(&lock->val, 0, _Q_LOCKED_VAL)
     → If lock word was 0, set to LOCKED, return (done!)

  2. Pending path: If only locked (no queue), set pending bit
     → Spin on locked byte until cleared, then acquire

  3. Slow path: Queue via MCS protocol
     → Allocate per-CPU MCS node
     → Add to tail of queue
     → Spin on local node's locked field
     → When prev releases, acquire lock
```

### Mutex Path (kernel/locking/mutex.c)

```
__mutex_lock():
  1. Fast path: atomic_long_try_cmpxchg_acquire(&lock->owner, 0, current)
     → Set owner to current task, done!

  2. Optimistic spin (osq_lock):
     → Spin on MCS queue while owner is running on CPU
     → If owner sleeps or we're preempted → give up, go to slow path

  3. Slow path:
     → Add to lock->wait_list
     → set_current_state(TASK_UNINTERRUPTIBLE)
     → schedule()  (sleep)
     → On wake: try to acquire, if fail → back to sleep
```

---

## 28.5 How to Navigate the Source

### Finding Functions

```bash
# Use cscope (cross-reference):
cd /path/to/linux
make cscope
cscope -d   # Query mode

# Use grep/ripgrep:
rg "request_threaded_irq" --type c
rg "struct irqaction" include/

# Use ctags:
make tags
vim -t request_irq

# Use elixir.bootlin.com online:
# https://elixir.bootlin.com/linux/latest/source
```

### Key Files to Study

```
For interrupt understanding (read in this order):
  1. include/linux/interrupt.h          ← API overview
  2. include/linux/irqdesc.h            ← Core data structures
  3. kernel/irq/manage.c                ← request_irq implementation
  4. kernel/irq/handle.c                ← IRQ dispatch
  5. kernel/softirq.c                   ← Bottom half
  6. kernel/irq/chip.c                  ← Flow handlers

For locking understanding:
  1. include/linux/spinlock.h           ← API
  2. kernel/locking/qspinlock.c         ← Spinlock internals
  3. kernel/locking/mutex.c             ← Mutex internals
  4. kernel/locking/lockdep.c           ← Lock validator
  5. kernel/rcu/tree.c                  ← RCU internals

For architecture-specific:
  x86: arch/x86/entry/entry_64.S + arch/x86/kernel/irq.c
  ARM64: arch/arm64/kernel/entry.S + drivers/irqchip/irq-gic-v3.c
```

---

## 28.6 Cross-Referencing with elixir.bootlin.com

```
Online source browser:
  https://elixir.bootlin.com/linux/latest/source

Features:
  - Click any symbol to see definition + all references
  - Search by function/struct/macro name
  - Navigate between kernel versions
  - See callers and callees

Useful pages:
  /linux/latest/source/kernel/irq/manage.c
  /linux/latest/source/kernel/softirq.c
  /linux/latest/source/kernel/locking/mutex.c
  /linux/latest/ident/irq_desc
  /linux/latest/ident/request_irq
```

---

## 28.7 Building a Custom Kernel for Study

```bash
# Clone kernel source:
git clone --depth 1 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux

# Configure with debug options:
make defconfig
scripts/config --enable CONFIG_PROVE_LOCKING
scripts/config --enable CONFIG_LOCKDEP
scripts/config --enable CONFIG_DEBUG_SPINLOCK
scripts/config --enable CONFIG_DEBUG_MUTEXES
scripts/config --enable CONFIG_DEBUG_ATOMIC_SLEEP
scripts/config --enable CONFIG_KASAN
scripts/config --enable CONFIG_KCSAN
scripts/config --enable CONFIG_FTRACE
scripts/config --enable CONFIG_FUNCTION_GRAPH_TRACER
scripts/config --enable CONFIG_IRQSOFF_TRACER

# Build:
make -j$(nproc)

# Run in QEMU for testing:
qemu-system-x86_64 -kernel arch/x86/boot/bzImage \
  -initrd initramfs.cpio.gz -nographic -append "console=ttyS0"
```

---

## Kernel Source References

```
Source navigation tools:
  https://elixir.bootlin.com/           ← Online cross-reference
  https://lxr.misez.de/                 ← Alternative LXR
  scripts/tags.sh                       ← Kernel tag generation
  Documentation/dev-tools/              ← Development tools docs
```

---

## Interview Questions

1. **Where is request_irq() implemented? Walk through its call chain.**
2. **What is irq_desc? Where is it defined?**
3. **Trace the code path from hardware interrupt to driver handler on x86.**
4. **Where is the softirq execution code? What limits does __do_softirq apply?**
5. **How does the queued spinlock work? Where is the source?**
6. **What is the mutex fast path? Where in the source?**
7. **Where is lockdep implemented? What data structures does it use?**
8. **How do you navigate Linux kernel source efficiently?**
9. **Where are IRQ domain mappings defined?**
10. **What assembly entry code handles interrupts on ARM64?**

---

## Summary

- kernel/irq/: core IRQ subsystem (manage.c = request_irq, handle.c = dispatch)
- kernel/softirq.c: softirq execution (__do_softirq, ksoftirqd)
- kernel/locking/: all lock implementations (qspinlock, mutex, rtmutex, lockdep)
- kernel/rcu/: RCU implementation (tree.c is the main file)
- arch/x86/entry/ and arch/arm64/kernel/entry.S: low-level IRQ entry
- drivers/irqchip/: interrupt controller drivers (GIC, APIC)
- Use elixir.bootlin.com for online cross-referenced browsing
- Build a debug kernel with lockdep, KASAN, ftrace for study

---

*Next: [Chapter 29 — Performance Optimization](Chapter_29_Performance_Optimization.md)*
