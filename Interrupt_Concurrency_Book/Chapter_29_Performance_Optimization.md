# Chapter 29: Performance Optimization

## Learning Goals
- Optimize interrupt handling for throughput and latency
- Reduce lock contention in high-performance drivers
- Apply interrupt coalescing, NAPI, and adaptive techniques
- Tune IRQ affinity for NUMA-aware performance
- Profile and measure before optimizing

---

## 29.1 Interrupt Overhead Analysis

```
Cost breakdown per interrupt (approximate):

Component                         │ x86 (ns)    │ ARM64 (ns)
──────────────────────────────────┼─────────────┼─────────────
CPU context save (registers)      │ 50-100      │ 50-80
IDT/VBAR lookup + branch          │ 10-20       │ 10-20
Interrupt controller acknowledge  │ 20-50       │ 30-80
Kernel C entry code               │ 30-50       │ 30-50
Handler execution (driver ISR)    │ 100-10000   │ 100-10000
Context restore + iret/eret       │ 50-100      │ 50-80
──────────────────────────────────┼─────────────┼─────────────
Total overhead (excl. handler)    │ ~200-400    │ ~200-400
```

For high-rate devices (NVMe, 100GbE), interrupt overhead at 1M IRQs/sec:
```
1,000,000 × 400ns = 400ms per second = 40% of one CPU core!
→ Must reduce interrupt rate
```

---

## 29.2 Interrupt Coalescing

Batch multiple events into fewer interrupts:

```
Without coalescing:                With coalescing:
  Packet 1 → IRQ                    Packet 1 → (wait)
  Packet 2 → IRQ                    Packet 2 → (wait)
  Packet 3 → IRQ                    Packet 3 → (wait)
  Packet 4 → IRQ                    Timer expires → IRQ (handle all 4)
  4 interrupts                       1 interrupt

Trade-off: throughput ↑ latency ↑
```

### Hardware Coalescing (ethtool)

```bash
# View current settings:
ethtool -c eth0

# Set coalescing:
ethtool -C eth0 rx-usecs 50 rx-frames 64
# Wait up to 50µs or 64 frames, whichever comes first

# Adaptive coalescing:
ethtool -C eth0 adaptive-rx on adaptive-tx on
# Hardware adjusts based on load
```

### Driver-Level Coalescing

```c
/* Hardware coalescing register programming: */
static void set_coalescing(struct my_dev *dev, u32 usecs, u32 frames)
{
    writel(usecs, dev->regs + IRQ_COALESCE_TIMER);
    writel(frames, dev->regs + IRQ_COALESCE_FRAMES);
}

/* Support ethtool interface: */
static int my_set_coalesce(struct net_device *ndev,
                            struct ethtool_coalesce *ec,
                            struct kernel_ethtool_coalesce *kec,
                            struct netlink_ext_ack *extack)
{
    struct my_priv *priv = netdev_priv(ndev);
    set_coalescing(priv, ec->rx_coalesce_usecs, ec->rx_max_coalesced_frames);
    return 0;
}
```

---

## 29.3 NAPI Optimization

NAPI (New API) switches between interrupt and polling mode:

```
Low load: Interrupt-driven (low latency)
High load: Polling (high throughput, no interrupt overhead)

                  ┌─────────────┐
                  │  Idle       │
                  │  (IRQ mode) │
                  └──────┬──────┘
                         │ Packet arrives → IRQ
                         ▼
                  ┌──────────────┐
                  │ Disable IRQ  │
                  │ Schedule NAPI│
                  └──────┬───────┘
                         │
                  ┌──────▼───────┐
                  │ Poll loop    │←──── Process up to 'budget' packets
                  │ (softirq)   │      per poll invocation
                  └──────┬───────┘
                         │
              ┌──────────┴─────────┐
              │ done < budget?     │
              │ (all packets done) │
              ├──── Yes ──────┐    │
              │               ▼    │
              │    ┌───────────┐   │
              │    │ Re-enable │   │
              │    │ IRQ       │   │
              │    └───────────┘   │
              │                    │
              └──── No ────────────┘
                         │
                  Continue polling
                  (stay in NAPI)
```

### NAPI Tuning

```bash
# Adjust NAPI budget (default 64):
# (higher = more packets per poll, better throughput, worse latency)
sysctl -w net.core.netdev_budget=300

# Adjust busy poll (bypass NAPI for ultra-low latency):
sysctl -w net.core.busy_poll=50
sysctl -w net.core.busy_read=50
# Application spins for 50µs polling for data before sleeping
```

---

## 29.4 Lock Optimization Strategies

### Strategy 1: Reduce Critical Section Size

```c
/* BAD: Large critical section */
spin_lock_irqsave(&dev->lock, flags);
data = readl(dev->regs + DATA);
process_data(data);          /* Expensive computation under lock */
update_statistics(dev);
spin_unlock_irqrestore(&dev->lock, flags);

/* GOOD: Minimal critical section */
spin_lock_irqsave(&dev->lock, flags);
data = readl(dev->regs + DATA);
spin_unlock_irqrestore(&dev->lock, flags);

process_data(data);          /* No lock needed for local variable */

this_cpu_inc(dev->stats->packets);  /* Per-CPU, no lock */
```

### Strategy 2: Per-CPU Data Instead of Locks

```c
/* BAD: Atomic counter with bouncing cache line */
atomic64_inc(&dev->total_bytes);   /* All CPUs contend */

/* GOOD: Per-CPU counters */
this_cpu_ptr(dev->pcpu_stats)->bytes += len;  /* No contention */
```

### Strategy 3: RCU Instead of rwlock

```c
/* BAD: rwlock for read-mostly config */
read_lock(&dev->config_lock);
val = dev->config->threshold;
read_unlock(&dev->config_lock);

/* GOOD: RCU - zero-cost readers */
rcu_read_lock();
cfg = rcu_dereference(dev->config);
val = cfg->threshold;
rcu_read_unlock();
```

### Strategy 4: Lock Splitting

```c
/* BAD: Single lock for everything */
struct device {
    spinlock_t lock;       /* One lock for TX + RX + config */
};

/* GOOD: Separate locks per function */
struct device {
    spinlock_t tx_lock;    /* Only TX path */
    spinlock_t rx_lock;    /* Only RX path */
    struct mutex cfg_lock; /* Configuration changes */
};
```

### Strategy 5: Lock-Free Algorithms

```c
/* Producer-consumer ring buffer without locks: */
struct ring {
    void *data[RING_SIZE];
    atomic_t head;  /* Producer writes */
    atomic_t tail;  /* Consumer writes */
};

/* Single producer, single consumer → no lock needed with barriers */
void produce(struct ring *r, void *item)
{
    int head = atomic_read(&r->head);
    r->data[head % RING_SIZE] = item;
    smp_store_release(&r->head, head + 1);
}

void *consume(struct ring *r)
{
    int tail = atomic_read(&r->tail);
    int head = smp_load_acquire(&r->head);
    if (tail == head)
        return NULL;
    void *item = r->data[tail % RING_SIZE];
    smp_store_release(&r->tail, tail + 1);
    return item;
}
```

---

## 29.5 IRQ Affinity for NUMA

```
NUMA (Non-Uniform Memory Access):
  Memory access time depends on which CPU accesses which memory node.
  
  ┌────────────┐         ┌────────────┐
  │  Node 0    │         │  Node 1    │
  │ CPU 0-7    │←─ QPI ─→│ CPU 8-15   │
  │ Local RAM  │  (slow) │ Local RAM  │
  │ PCIe slot  │         │ PCIe slot  │
  └────────────┘         └────────────┘

Rule: IRQ → CPU on same NUMA node as device → local memory access
```

```bash
# Check device NUMA node:
cat /sys/class/net/eth0/device/numa_node
# Output: 0

# Check which CPUs are on node 0:
cat /sys/devices/system/node/node0/cpulist
# Output: 0-7

# Set IRQ affinity to node-local CPUs:
echo 0-7 > /proc/irq/25/smp_affinity_list

# For multi-queue NIC: one queue per core, NUMA-aware:
for i in $(seq 0 7); do
    irq=$(cat /proc/interrupts | grep "eth0-q$i" | awk '{print $1}' | tr -d ':')
    echo $i > /proc/irq/$irq/smp_affinity_list
done
```

---

## 29.6 Measuring Before Optimizing

### Profiling Checklist

```bash
# 1. Overall system profile:
perf top -a                          # Live hotspot view

# 2. Lock contention:
perf lock contention -a -- sleep 10

# 3. CPU time in IRQ/softirq:
sar -I ALL 1 5                       # IRQ rates
mpstat -P ALL 1 5                    # Per-CPU: %irq, %soft

# 4. Cache misses (lock bouncing indicator):
perf stat -e cache-misses,cache-references,L1-dcache-load-misses -a sleep 5

# 5. Function hotspots in driver:
perf record -g -p $(pgrep my_driver_user) sleep 10
perf report

# 6. IRQ-disabled latency:
echo irqsoff > /sys/kernel/debug/tracing/current_tracer
cat /sys/kernel/debug/tracing/trace
```

---

## 29.7 Optimization Decision Matrix

```
Symptom                         │ Profile Tool       │ Optimization
────────────────────────────────┼────────────────────┼──────────────────
High CPU in hardirq             │ /proc/interrupts   │ Coalescing, NAPI
                                │ mpstat %irq        │
Softirq overload (ksoftirqd)   │ /proc/softirqs     │ More queues, RPS
                                │ top                │
Lock contention on one lock     │ perf lock          │ Lock splitting
                                │                    │ RCU for readers
High cache misses               │ perf stat          │ Per-CPU data
                                │                    │ NUMA affinity
Cross-NUMA IRQ handling         │ numa_node mismatch │ Set affinity
Latency spikes                  │ cyclictest, ftrace │ Threaded IRQ,
                                │                    │ isolcpus
```

---

## Kernel Source References

```
NAPI:
  net/core/dev.c                        ← napi_schedule, napi_complete
  include/linux/netdevice.h             ← NAPI structures

Coalescing:
  include/linux/ethtool.h               ← ethtool_coalesce
  net/ethtool/                          ← ethtool implementation

Performance tools:
  tools/perf/                           ← perf source
  kernel/trace/                         ← ftrace
```

---

## Interview Questions

1. **What is interrupt coalescing? What are the trade-offs?**
2. **How does NAPI reduce interrupt overhead?**
3. **Name 5 strategies to reduce lock contention.**
4. **Why is NUMA-aware IRQ affinity important?**
5. **How would you profile a driver to find lock contention?**
6. **When is per-CPU data better than atomic counters?**
7. **Explain busy polling. When is it useful?**
8. **How do you measure the actual cost of an interrupt?**
9. **What is the danger of premature optimization in kernel code?**
10. **Design an optimized data path for a high-speed network driver.**

---

## Summary

- Profile FIRST: use perf, ftrace, /proc/interrupts before optimizing
- Interrupt coalescing: batch events to reduce IRQ rate (trade latency for throughput)
- NAPI: adaptive interrupt/poll mode — essential for high-rate network I/O
- Lock contention: reduce critical section, split locks, use per-CPU or RCU
- NUMA affinity: bind IRQs to CPUs on the same NUMA node as the device
- Per-CPU data: eliminates cache bouncing for statistics and counters
- Lock-free algorithms: SPSC ring buffers, atomic CAS patterns
- Measure the impact of every optimization — not all help in every scenario

---

*Next: [Chapter 30 — Important Diagrams](Chapter_30_Diagrams.md)*
