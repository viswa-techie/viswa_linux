# Chapter 9: Interrupt Threading

## Learning Goals
- Understand threaded interrupts and why they exist
- Master request_threaded_irq() API and usage patterns
- Know how PREEMPT_RT transforms interrupt handling
- Design real-time-friendly interrupt handlers
- Compare threaded vs traditional interrupt models

---

## 9.1 Threaded Interrupts Concept

### The Problem with Traditional Handling

```
Traditional model:
  HardIRQ handler runs with:
    - Interrupts disabled (same line)
    - Preemption disabled
    - Cannot sleep
    - Unpredictable latency for other tasks

  For RT systems, this is unacceptable:
    A long-running network IRQ handler can delay
    a safety-critical motor control task.
```

### The Threaded IRQ Solution

```
Threaded model:
  Minimal top half → wake dedicated kernel thread → thread processes

  ┌────────────────────┐     ┌────────────────────────┐
  │  TOP HALF (hardirq)│     │  THREADED HANDLER       │
  │  ~0.5-2 µs         │     │  (kernel thread)        │
  │  Ack hardware      │────→│  Runs in process context│
  │  Return IRQ_WAKE_  │     │  CAN sleep              │
  │       THREAD       │     │  CAN use mutexes        │
  │                    │     │  CAN be preempted       │
  │                    │     │  HAS RT priority         │
  └────────────────────┘     └────────────────────────┘

Benefits:
  1. Predictable: IRQ processing has a priority, scheduler decides
  2. Preemptible: Higher-priority RT task can preempt IRQ thread
  3. Sleepable: Can use mutex, GFP_KERNEL, I2C/SPI xfers
  4. Debuggable: Shows up in ps, can trace with perf/ftrace
```

### API: request_threaded_irq()

```c
int request_threaded_irq(
    unsigned int irq,            /* IRQ number */
    irq_handler_t handler,       /* top half (hardirq context) */
    irq_handler_t thread_fn,     /* threaded handler (process context) */
    unsigned long irqflags,      /* IRQF_ flags */
    const char *devname,         /* name for /proc/interrupts */
    void *dev_id                 /* device identifier */
);

/* handler (top half) can be:
   - A real function: ack hardware, return IRQ_WAKE_THREAD
   - NULL: use default irq_default_primary_handler
     (just returns IRQ_WAKE_THREAD)

   thread_fn: the main handler, runs in dedicated kthread
*/
```

### Complete Threaded IRQ Example

```c
#include <linux/interrupt.h>
#include <linux/i2c.h>

struct sensor_dev {
    struct i2c_client *client;
    int irq;
    struct mutex lock;
    int last_reading;
};

/* Top half: minimal — ack and wake thread */
static irqreturn_t sensor_top_half(int irq, void *dev_id)
{
    struct sensor_dev *dev = dev_id;
    
    /* Optional: read status register to confirm our IRQ */
    /* For GPIO-triggered sensors, often nothing to ack */
    
    return IRQ_WAKE_THREAD;  /* Wake the threaded handler */
}

/* Threaded handler: full processing in process context */
static irqreturn_t sensor_thread_fn(int irq, void *dev_id)
{
    struct sensor_dev *dev = dev_id;
    int value;
    
    /* CAN SLEEP! Do I2C transaction */
    mutex_lock(&dev->lock);
    
    /* i2c_smbus_read_word_data sleeps (sends I2C transaction) */
    value = i2c_smbus_read_word_data(dev->client, SENSOR_DATA_REG);
    if (value >= 0)
        dev->last_reading = value;
    
    mutex_unlock(&dev->lock);
    
    /* Notify user space */
    sysfs_notify(&dev->client->dev.kobj, NULL, "reading");
    
    return IRQ_HANDLED;
}

/* Registration */
static int sensor_probe(struct i2c_client *client)
{
    struct sensor_dev *dev;
    int ret;
    
    dev = devm_kzalloc(&client->dev, sizeof(*dev), GFP_KERNEL);
    dev->client = client;
    dev->irq = client->irq;
    mutex_init(&dev->lock);
    
    ret = devm_request_threaded_irq(&client->dev, dev->irq,
            sensor_top_half,     /* top half (or NULL) */
            sensor_thread_fn,    /* threaded handler */
            IRQF_TRIGGER_FALLING | IRQF_ONESHOT,
            "my-sensor", dev);
    
    /* IRQF_ONESHOT: keep IRQ masked until threaded handler
       completes. Essential for level-triggered + threaded! */
    
    return ret;
}
```

### IRQF_ONESHOT — Critical for Threaded IRQs

```
Without IRQF_ONESHOT:
  HardIRQ → unmask immediately → IRQ fires again!
  Threaded handler hasn't run yet → interrupt storm!
  
  Time →
  IRQ ──┬──ack──unmask──┬──IRQ fires again!──┬──storm!
        │               │                    │
        └── thread scheduled but hasn't run yet

With IRQF_ONESHOT:
  HardIRQ → IRQ stays MASKED → thread runs → unmask
  
  Time →
  IRQ ──┬──ack──(masked)───────── thread runs ──unmask──→
        │                              │
        └── Safe: no re-triggering     └── Now ready for next IRQ

Rule: ALWAYS use IRQF_ONESHOT with request_threaded_irq()
      when handler is NULL (default primary) or for level-triggered.
```

---

## 9.2 PREEMPT_RT Interrupt Handling

### Force-Threaded IRQs

```
On PREEMPT_RT kernels, the kernel FORCES most hardirq
handlers to run as threads, even if the driver uses
plain request_irq().

  Standard kernel:
    request_irq(irq, handler, ...)
    → handler runs in hardirq context

  PREEMPT_RT kernel:
    request_irq(irq, handler, ...)
    → handler runs in IRQ THREAD context (forced threading!)
    → Only IRQF_NO_THREAD handlers remain in hardirq

Mechanism:
  __setup_irq() checks force_irqthreads (set on PREEMPT_RT)
  Creates irq_thread even for non-threaded request_irq()
  Original handler becomes thread_fn
  Default primary handler just returns IRQ_WAKE_THREAD
```

### Which IRQs Are NOT Threaded on PREEMPT_RT

```
IRQF_NO_THREAD prevents force-threading:
  - Timer interrupts (need hardirq precision)
  - IPI (inter-processor, must be fast)
  - NMI (by definition non-maskable)
  - Some arch-specific IRQs

  /* This handler will ALWAYS run in hardirq, even on RT: */
  request_irq(irq, handler, IRQF_NO_THREAD, "timer", dev);
```

### Softirqs on PREEMPT_RT

```
PREEMPT_RT also changes softirq handling:

Standard kernel:
  Softirqs run in softirq context (non-preemptible)
  
PREEMPT_RT:
  Softirqs run in dedicated ksoftirqd threads (preemptible!)
  This makes ALL deferred work preemptible → predictable latency

Impact:
  - Network processing (NET_RX_SOFTIRQ) → preemptible
  - Timer processing (TIMER_SOFTIRQ) → preemptible
  - Block completion (BLOCK_SOFTIRQ) → preemptible
  - Everything can be scheduled by the RT scheduler
```

---

## 9.3 Real-Time Interrupt Processing

### IRQ Thread Priorities

```
IRQ threads default to SCHED_FIFO priority 50.
You can change per-thread priority via chrt or kernel API:

  $ ps -eo pid,cls,rtprio,comm | grep irq/
  65  FF  50  irq/28-my-device
  66  FF  50  irq/33-eth0
  67  FF  50  irq/42-spi0

Changing priority:
  $ chrt -f -p 80 65    # Set irq/28-my-device to FIFO prio 80

Or in the driver:
  struct sched_param param = { .sched_priority = 80 };
  sched_setscheduler(action->thread, SCHED_FIFO, &param);
```

### Priority Assignment for Embedded RT

```
Priority Design (automotive example):

  Priority │ Thread              │ Latency Requirement
  ─────────┼─────────────────────┼────────────────────
  99       │ Safety watchdog     │ < 50 µs
  90       │ Motor control IRQ   │ < 100 µs
  80       │ Sensor data IRQ     │ < 500 µs
  70       │ CAN bus IRQ         │ < 1 ms
  50       │ Display IRQ         │ < 5 ms (default)
  40       │ Network IRQ         │ < 10 ms
  20       │ Logging workqueue   │ Best effort
  0        │ Background tasks    │ Best effort
```

### Measuring RT Interrupt Latency

```bash
# Install rt-tests
$ apt-get install rt-tests

# Measure worst-case latency:
$ cyclictest -t1 -p 80 -n -i 1000 -l 100000
# -t1: one thread
# -p 80: SCHED_FIFO priority 80
# -n: use clock_nanosleep
# -i 1000: 1ms interval
# -l 100000: 100K iterations

# Output:
T: 0 ( 1234) P:80 I:1000 C: 100000 Min:      1 Act:    3 Avg:    4 Max:   23
#                                     ^^^                          ^^^
#                                   Best case                  Worst case (µs)

# On good PREEMPT_RT: Max < 50 µs
# On standard kernel: Max can be > 1000 µs
```

### Threaded IRQ Flow on PREEMPT_RT

```
  Device IRQ fires
      │
      ▼
  Minimal hardirq stub (or IRQF_NO_THREAD handler)
      │ Returns IRQ_WAKE_THREAD
      ▼
  Wake irq/<N>-<name> kernel thread
      │
      ▼
  Scheduler runs (RT scheduler picks highest priority)
      │
      ▼
  ┌─────────────────────────────────────────────┐
  │  If higher-priority RT task is running:     │
  │    → IRQ thread WAITS (deterministic)       │
  │  If IRQ thread is highest priority:          │
  │    → IRQ thread runs immediately            │
  └─────────────────────────────────────────────┘
      │
      ▼
  thread_fn() executes in process context
      │
      ▼
  irq_finalize_oneshot() → unmask IRQ
```

---

## Design Patterns

### Pattern 1: Top-Half-Only (Simple, Fast)

```c
/* Use when: Handler is very short (<5µs), no sleeping needed */
request_irq(irq, simple_handler, 0, "simple", dev);
```

### Pattern 2: Top Half + Tasklet (Legacy)

```c
/* Use when: Legacy code, need to defer non-sleeping work */
request_irq(irq, top_half, 0, "legacy", dev);
/* top_half calls tasklet_schedule() */
```

### Pattern 3: Threaded IRQ, No Top Half

```c
/* Use when: Need to sleep, no urgent hardware ack needed */
request_threaded_irq(irq, NULL, thread_fn,
                     IRQF_ONESHOT, "sensor", dev);
/* Default primary handler used automatically */
```

### Pattern 4: Threaded IRQ with Top Half

```c
/* Use when: Need to ack hardware immediately + sleep in BH */
request_threaded_irq(irq, ack_handler, thread_fn,
                     IRQF_ONESHOT, "complex", dev);
/* ack_handler returns IRQ_WAKE_THREAD */
```

### Pattern 5: Top Half + Workqueue

```c
/* Use when: Need to sleep, want explicit workqueue control */
request_irq(irq, top_half, 0, "dev", dev);
/* top_half calls schedule_work(&dev->work) */
```

---

## Kernel Source References

```
Threaded IRQ implementation:
  kernel/irq/manage.c
    ├── __setup_irq()         ← Creates IRQ thread if thread_fn
    ├── irq_thread()           ← Main loop of IRQ thread
    ├── irq_thread_fn()        ← Calls action->thread_fn
    ├── irq_default_primary_handler()  ← Returns IRQ_WAKE_THREAD
    └── irq_finalize_oneshot() ← Unmask after thread completes

Force-threading:
  kernel/irq/manage.c: setup_forced_threading()
  kernel/irq/settings.h: irq_settings_can_thread()
```

---

## Interview Questions

1. **What is request_threaded_irq() and how does it differ from request_irq()?**
2. **What is IRQF_ONESHOT and why is it essential for threaded IRQs?**
3. **What happens if you pass NULL as the top half to request_threaded_irq()?**
4. **How does PREEMPT_RT change the behavior of request_irq()?**
5. **What scheduling policy do IRQ threads use? How would you change priority?**
6. **Give a real-world example where threaded IRQs are necessary (hint: I2C sensor).**
7. **Can a threaded IRQ handler be preempted? By what?**
8. **What is force_irqthreads and how does it work?**
9. **Compare the latency of hardirq handlers vs threaded IRQ handlers.**
10. **When should you use IRQF_NO_THREAD?**
11. **How do you measure interrupt latency on a PREEMPT_RT system?**
12. **What is the default priority for IRQ threads? Is it a good default?**
13. **Design an interrupt priority scheme for an automotive system with safety, sensor, and infotainment IRQs.**
14. **What happens to softirqs on a PREEMPT_RT kernel?**
15. **Why is threaded IRQ the recommended approach for new drivers?**

---

## Summary

- Threaded IRQs move interrupt processing from hardirq context to a schedulable kernel thread
- request_threaded_irq() allows a minimal top half + full-featured threaded handler
- IRQF_ONESHOT prevents re-triggering while the thread runs (essential for level-triggered)
- PREEMPT_RT force-threads most IRQ handlers for deterministic scheduling
- IRQ threads use SCHED_FIFO priority 50 by default — adjustable per system requirements
- This approach gives the RT scheduler full control over interrupt processing priority
- Modern best practice for new drivers: use threaded IRQs

---

*Next: [Chapter 10 — Interrupt Affinity and CPU Distribution](Chapter_10_Interrupt_Affinity.md)*
