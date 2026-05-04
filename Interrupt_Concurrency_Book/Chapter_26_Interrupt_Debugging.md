# Chapter 26: Interrupt Debugging

## Learning Goals
- Debug interrupt-related problems using kernel tools
- Use /proc/interrupts, /proc/softirqs, and /proc/stat
- Diagnose IRQ storms, missed interrupts, and latency issues
- Use lockdep, KASAN, and kernel debug options
- Debug concurrency bugs with deadlock detection

---

## 26.1 /proc/interrupts

The primary tool for interrupt monitoring:

```bash
$ cat /proc/interrupts
           CPU0       CPU1       CPU2       CPU3
  0:         30          0          0          0   IO-APIC   2-edge      timer
  1:          9          0          0          0   IO-APIC   1-edge      i8042
  8:          0          0          0          0   IO-APIC   8-edge      rtc0
  9:          0          0          0          0   IO-APIC   9-fasteoi   acpi
 16:      28947          0          0          0   IO-APIC  16-fasteoi   ehci_hcd
 24:          0          0          0          0  PCI-MSI  524288-edge   nvme0q0
 25:     194023          0          0          0  PCI-MSI  524289-edge   nvme0q1
 26:          0     183847          0          0  PCI-MSI  524290-edge   nvme0q2
NMI:        142        130        128        131   Non-maskable interrupts
LOC:     982341     975234     968123     971456   Local timer interrupts
RES:      34521      32456      31234      30876   Rescheduling interrupts
```

### Reading the Output

```
Column layout:
  IRQ#: CPU0_count CPU1_count ... Controller Trigger Device_name

Key observations:
  - Unbalanced counts → check affinity (smp_affinity)
  - Rapidly increasing → possible IRQ storm
  - Count stuck at 0 → IRQ not firing (wiring, enable, or driver bug)
  - NMI count → watchdog/profiling activity
  - LOC → local APIC timer (scheduler tick)
  - RES → IPI for rescheduling
```

### Monitoring Changes

```bash
# Watch interrupt rate (changes per second):
watch -n1 -d cat /proc/interrupts

# Count specific IRQ rate:
while true; do
    grep "nvme0q1" /proc/interrupts
    sleep 1
done

# Script to show IRQ rate:
awk 'NR==FNR{a[$1]=$2;next} {print $1, $2-a[$1]}' \
    <(cat /proc/interrupts) <(sleep 1; cat /proc/interrupts)
```

---

## 26.2 /proc/softirqs

```bash
$ cat /proc/softirqs
                    CPU0       CPU1       CPU2       CPU3
          HI:          2          0          0          0
       TIMER:     482631     479234     475123     477654
      NET_TX:       1234          0          0          0
      NET_RX:     198234          0          0     195432
       BLOCK:      34521      32456          0          0
    IRQ_POLL:          0          0          0          0
     TASKLET:       3456          0          0          0
       SCHED:     125432     123456     121234     122345
     HRTIMER:       8765       8432       8234       8123
         RCU:     234567     232345     230123     231890
```

### Diagnosing from Softirqs

```
NET_RX very high on one CPU → IRQ affinity issue, needs RSS/RPS
TIMER unbalanced → nohz_full active on some CPUs
TASKLET high → driver using tasklets heavily (consider workqueue)
RCU imbalanced → callback flood on specific CPU
SCHED → scheduler activity, correlates with load
```

---

## 26.3 Diagnosing Common Problems

### Problem 1: IRQ Storm

```
Symptom: System unresponsive, one CPU at 100% in IRQ context
         /proc/interrupts shows one IRQ count growing rapidly

Kernel message:
  "irq 16: nobody cared (try booting with the irqpoll option)"

Causes:
  1. Driver returns IRQ_HANDLED without actually clearing interrupt
  2. Hardware stuck asserting (faulty device)
  3. Wrong IRQ handler registered
  4. Level-triggered IRQ not ACKed properly

Debug steps:
  1. cat /proc/interrupts — identify the IRQ
  2. Check handler return value (must return IRQ_NONE if not ours)
  3. Verify hardware status register is being properly cleared
  4. Add debugging: pr_info in handler, check status register value
```

```bash
# Temporarily set IRQ affinity to isolate:
echo 2 > /proc/irq/16/smp_affinity  # Move to CPU 1 only

# Force-disable the interrupt:
echo 0 > /proc/irq/16/smp_affinity_list  # (may not work on all)
```

### Problem 2: Missed Interrupts

```
Symptom: Device operations timeout, but device seems functional
         IRQ count in /proc/interrupts not incrementing

Debug steps:
  1. Verify IRQ is registered: cat /proc/interrupts | grep mydev
  2. Check IRQ enable in hardware registers
  3. Check IRQ mask in interrupt controller
  4. Verify DT/ACPI interrupt description matches hardware
  5. Check for edge vs level trigger mismatch
  6. Try polling mode to confirm hardware works
```

```c
/* Add debug polling to verify hardware: */
static void debug_poll(struct my_dev *dev)
{
    u32 status = readl(dev->regs + IRQ_STATUS);
    if (status) {
        dev_info(dev->dev, "IRQ status=0x%08x but no IRQ!\n", status);
        /* Hardware is asserting, interrupt delivery is broken */
    }
}
```

### Problem 3: High Interrupt Latency

```
Symptom: Slow device response, missed deadlines (RT systems)

Debug approach:
  1. Measure with ftrace (irqsoff tracer)
  2. Check for long IRQ-disabled sections
  3. Check softirq backlog (ksoftirqd running = overloaded)
  4. Use cyclictest for RT latency measurement

# Enable IRQ-off latency tracer:
echo irqsoff > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# Trigger workload
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

---

## 26.4 Lockdep — Deadlock Detection

### Enabling

```bash
# Kernel config:
CONFIG_PROVE_LOCKING=y
CONFIG_LOCKDEP=y
CONFIG_DEBUG_LOCK_ALLOC=y
```

### What Lockdep Catches

```
1. ABBA deadlock:
   CPU 0: lock A → lock B
   CPU 1: lock B → lock A
   → lockdep detects inconsistent ordering at first occurrence!

2. IRQ context inconsistency:
   Process: spin_lock(&x)
   IRQ:     spin_lock(&x)
   → lockdep warns: "inconsistent {HARDIRQ-ON-W} -> {IN-HARDIRQ-W} usage"

3. Recursive locking:
   spin_lock(&x)
   spin_lock(&x)  → immediate deadlock detected

4. Lock held across sleep:
   spin_lock(&x)
   kmalloc(GFP_KERNEL)  → "BUG: sleeping function called from invalid context"
```

### Reading Lockdep Output

```
=============================================
[ INFO: possible circular locking dependency detected ]
---------------------------------------------
taskA/1234 is trying to acquire lock:
 (&dev->mutex){+.+.}-{3:3}, at: my_write+0x42/0x100

but task is already holding lock:
 (&dev->spinlock){....}-{2:2}, at: my_read+0x28/0x80

which lock already depends on the new lock

→ Translation: my_read takes spinlock then mutex,
  but somewhere else mutex is taken before spinlock.
  ABBA deadlock possible.
```

---

## 26.5 KASAN — Memory Bug Detection

```bash
# Kernel config:
CONFIG_KASAN=y
CONFIG_KASAN_GENERIC=y   # or CONFIG_KASAN_SW_TAGS for ARM64
```

### What KASAN Catches in IRQ Context

```
1. Use-after-free in IRQ handler:
   - kfree(dev) in remove, but IRQ handler still accesses dev
   - KASAN report: "BUG: KASAN: use-after-free in my_isr+0x20"

2. Out-of-bounds in DMA callback:
   - Buffer overflow in IRQ-triggered DMA completion
   - KASAN report: "BUG: KASAN: slab-out-of-bounds"

3. Stack buffer overflow in ISR:
   - Large local arrays in IRQ handler exhausting IRQ stack
```

---

## 26.6 Other Debug Kernel Configs

```
CONFIG_DEBUG_SPINLOCK=y           # Detect spinlock bugs (double unlock, etc.)
CONFIG_DEBUG_MUTEXES=y            # Detect mutex misuse
CONFIG_DEBUG_ATOMIC_SLEEP=y       # Detect sleep in atomic context
CONFIG_DEBUG_PREEMPT=y            # Detect preempt count imbalance
CONFIG_DETECT_HUNG_TASK=y         # Detect tasks blocked too long
CONFIG_SOFTLOCKUP_DETECTOR=y      # Detect CPU stuck in kernel (no schedule)
CONFIG_HARDLOCKUP_DETECTOR=y      # Detect CPU stuck with IRQs off
CONFIG_RCU_CPU_STALL_TIMEOUT=21   # RCU stall timeout (seconds)
CONFIG_DEBUG_OBJECTS=y            # Track object lifecycle (timers, work, etc.)
CONFIG_KCSAN=y                    # Detect data races
CONFIG_DEBUG_IRQFLAGS=y           # Track IRQ enable/disable state
```

### Softlockup vs Hardlockup

```
Softlockup:
  CPU doesn't schedule for >20 seconds (watchdog thread starved)
  Message: "BUG: soft lockup - CPU#0 stuck for 23s!"
  Cause: Infinite loop in kernel without schedule()

Hardlockup:
  CPU doesn't handle NMI watchdog for >10 seconds (IRQs disabled)
  Message: "NMI watchdog: Watchdog detected hard LOCKUP on cpu 0"
  Cause: IRQs disabled too long, deadlock in IRQ-off context

Diagnosis:
  Check the stack trace in the lockup message → identify the function
```

---

## 26.7 Dynamic Debug for Interrupts

```bash
# Enable verbose IRQ debugging at runtime:
echo "file kernel/irq/manage.c +p" > /sys/kernel/debug/dynamic_debug/control

# Enable all debug prints in your driver:
echo "module my_driver +p" > /sys/kernel/debug/dynamic_debug/control

# Trace specific function:
echo "func request_irq +p" > /sys/kernel/debug/dynamic_debug/control
```

### dev_dbg in Drivers

```c
/* In driver code: */
dev_dbg(dev->dev, "IRQ status: 0x%08x\n", status);

/* Enable at runtime: */
echo "file drivers/my/driver.c +p" > /sys/kernel/debug/dynamic_debug/control

/* Or via module parameter: */
echo 8 > /proc/sys/kernel/printk   /* Enable all debug messages */
```

---

## 26.8 Debugging Concurrency Bugs

### Reproducing Race Conditions

```bash
# Stress test with multiple threads:
for i in $(seq 1 8); do
    dd if=/dev/mydevice of=/dev/null bs=4096 count=10000 &
done
wait

# Use stress-ng for general system stress:
stress-ng --io 4 --cpu 4 --timeout 60

# CPU hotplug stress (triggers many races):
for cpu in 1 2 3; do
    echo 0 > /sys/devices/system/cpu/cpu$cpu/online
    sleep 0.1
    echo 1 > /sys/devices/system/cpu/cpu$cpu/online
done
```

### KCSAN Output

```
==================================================================
BUG: KCSAN: data-race in my_read / my_irq_handler

write to 0xffff8881234567a0 of 4 bytes by interrupt on cpu 1:
 my_irq_handler+0x42/0x80

read to 0xffff8881234567a0 of 4 bytes by task 1234 on cpu 0:
 my_read+0x38/0x100

Reported by Kernel Concurrency Sanitizer on:
CPU: 0 PID: 1234 Comm: dd

Fix: Use READ_ONCE/WRITE_ONCE or proper locking
==================================================================
```

---

## 26.9 Debug Checklist

```
□ /proc/interrupts: IRQ registered and counting?
□ /proc/softirqs: softirq running? Balanced across CPUs?
□ dmesg: Any IRQ-related errors?
□ lockdep: Enabled? Any warnings?
□ KASAN: Use-after-free in IRQ handler?
□ IRQ handler: Returns IRQ_NONE for non-owned interrupts?
□ Hardware: Status register cleared properly?
□ Trigger type: Edge vs level matches hardware?
□ DT/ACPI: IRQ number and flags correct?
□ Affinity: Balanced across CPUs? (/proc/irq/N/smp_affinity)
□ Latency: ftrace irqsoff tracer shows reasonable times?
□ Stress: Race conditions under heavy concurrent load?
```

---

## Kernel Source References

```
Debug infrastructure:
  kernel/locking/lockdep.c              ← Lockdep
  mm/kasan/                             ← KASAN
  kernel/kcsan/                         ← KCSAN
  kernel/watchdog.c                     ← Soft/hard lockup
  kernel/irq/debug.h                    ← IRQ debug helpers
  lib/dynamic_debug.c                   ← Dynamic debug

Proc interfaces:
  kernel/irq/proc.c                     ← /proc/interrupts
  kernel/softirq.c                      ← /proc/softirqs
  fs/proc/stat.c                        ← /proc/stat (irq section)
```

---

## Interview Questions

1. **How do you debug a missing interrupt?**
2. **What is /proc/interrupts? What information does each column show?**
3. **What is an IRQ storm? How do you detect and fix it?**
4. **What is lockdep? What types of bugs does it detect?**
5. **Explain softlockup vs hardlockup detection.**
6. **How do you use ftrace to measure IRQ latency?**
7. **What is KCSAN? How does it detect data races?**
8. **How do you stress-test a driver for concurrency bugs?**
9. **What kernel config options would you enable for debugging a driver?**
10. **How do you read and interpret lockdep output?**

---

## Summary

- /proc/interrupts: first tool for IRQ debugging (counts, affinity, handler)
- /proc/softirqs: monitor deferred work distribution and overload
- lockdep: catches deadlocks and IRQ consistency violations at runtime
- KASAN: catches use-after-free and out-of-bounds in IRQ handlers
- KCSAN: catches data races (missing READ_ONCE/WRITE_ONCE, missing locks)
- IRQ storms: always check status register, always return IRQ_NONE if not yours
- Softlockup/hardlockup detectors: catch CPUs stuck in loops
- Enable CONFIG_PROVE_LOCKING + KASAN + KCSAN during development

---

*Next: [Chapter 27 — Kernel Tracing for Interrupts](Chapter_27_Kernel_Tracing.md)*
