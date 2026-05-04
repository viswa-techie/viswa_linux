# Chapter 28: Performance Optimization

## Chapter Overview

Driver performance directly impacts system throughput and latency. This chapter covers latency optimization, throughput optimization, interrupt mitigation, DMA optimization, and profiling techniques.

---

## 28.1 Performance Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| **Latency** | Time from request to response | Minimize (µs–ms) |
| **Throughput** | Data processed per second | Maximize (MB/s, Mpps) |
| **CPU usage** | Cycles consumed by driver | Minimize |
| **IRQ rate** | Interrupts per second | Keep manageable (<100K/s) |
| **Cache misses** | L1/L2/LLC misses in driver | Minimize |

---

## 28.2 Latency Optimization

### Minimize Lock Hold Times

```c
/* SLOW: holding spinlock across slow operation */
spin_lock(&priv->lock);
for (i = 0; i < 1000; i++)
    readl(priv->base + i * 4);  /* 1000 MMIO reads under lock */
spin_unlock(&priv->lock);

/* FAST: copy under lock, process outside */
spin_lock(&priv->lock);
memcpy(snapshot, priv->ring, sizeof(snapshot));
spin_unlock(&priv->lock);
process_data(snapshot);
```

### Use Threaded IRQs for Long Handlers

```c
/* Hard IRQ: acknowledge only, return fast */
static irqreturn_t my_hard_irq(int irq, void *data)
{
    u32 status = readl(priv->base + IRQ_STATUS);
    if (!status)
        return IRQ_NONE;
    writel(status, priv->base + IRQ_CLEAR);  /* Acknowledge */
    return IRQ_WAKE_THREAD;
}

/* Threaded handler: runs in process context, can sleep */
static irqreturn_t my_thread_fn(int irq, void *data)
{
    /* Long processing: DMA completion, data copy, etc. */
    process_received_data(priv);
    return IRQ_HANDLED;
}
```

### Polling vs Interrupts Decision

```
Latency-sensitive low-frequency events → Interrupts
High-frequency events (>100K/s)        → NAPI polling / busy-poll
Mixed                                  → Interrupt coalescing
```

---

## 28.3 Throughput Optimization

### DMA for Bulk Data Transfer

```c
/* SLOW: PIO (CPU copies each byte) */
for (i = 0; i < len; i++)
    buf[i] = readb(priv->base + FIFO_REG);   /* CPU bound */

/* FAST: DMA (hardware copies, CPU free) */
dma_addr = dma_map_single(dev, buf, len, DMA_FROM_DEVICE);
writel(dma_addr, priv->base + DMA_ADDR);
writel(len, priv->base + DMA_LEN);
writel(DMA_START, priv->base + DMA_CTRL);
/* CPU free to do other work while DMA runs */
```

### Scatter-Gather DMA

```c
/* Instead of copying to contiguous buffer, DMA directly from pages */
struct scatterlist sg[MAX_SG];
int nents;

sg_init_table(sg, MAX_SG);
for (i = 0; i < nents; i++)
    sg_set_page(&sg[i], pages[i], PAGE_SIZE, 0);

dma_map_sg(dev, sg, nents, DMA_TO_DEVICE);
/* Program hardware with SG descriptor list */
```

### Ring Buffer / Descriptor Ring

```c
/*
 * Pre-allocate descriptors in a ring — avoid per-transfer allocation
 *
 *    ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐
 *    │D0│→│D1│→│D2│→│D3│→│D4│→│D5│→ (wraps)
 *    └──┘ └──┘ └──┘ └──┘ └──┘ └──┘
 *     ↑              ↑
 *   tail(HW)       head(SW)
 */
struct ring_desc {
    dma_addr_t addr;
    u32 len;
    u32 flags;
} __packed;

/* Pre-allocate entire ring as coherent DMA */
ring = dma_alloc_coherent(dev, RING_SIZE * sizeof(*ring),
                          &ring_dma, GFP_KERNEL);
```

---

## 28.4 Interrupt Mitigation

### NAPI (Network Drivers)

```c
/* Instead of IRQ per packet: IRQ → switch to polling */
static irqreturn_t my_irq(int irq, void *data)
{
    disable_hw_irq(priv);                /* Stop interrupts */
    napi_schedule(&priv->napi);          /* Schedule polling */
    return IRQ_HANDLED;
}

static int my_napi_poll(struct napi_struct *napi, int budget)
{
    int done = 0;
    while (done < budget && has_packets(priv)) {
        process_one_packet(priv);
        done++;
    }
    if (done < budget) {
        napi_complete_done(napi, done);
        enable_hw_irq(priv);            /* Re-enable IRQ */
    }
    return done;
}
```

### Interrupt Coalescing

```c
/* Hardware setting: don't interrupt for every event */
/* Example: interrupt after 64 packets OR 50µs, whichever first */
writel(64, priv->base + IRQ_COALESCE_FRAMES);
writel(50, priv->base + IRQ_COALESCE_USECS);
```

---

## 28.5 Cache Optimization

### Structure Layout

```c
/* SLOW: cache-unfriendly (hot and cold data mixed) */
struct my_dev {
    char name[64];            /* Cold: rarely accessed */
    spinlock_t lock;          /* Hot: every operation */
    u32 stats[32];            /* Cold: periodic read */
    void __iomem *base;       /* Hot: every MMIO access */
    struct list_head list;    /* Hot: queue operations */
};

/* FAST: separate hot/cold, align to cache line */
struct my_dev {
    /* --- Hot path (one cache line) --- */
    spinlock_t lock;
    void __iomem *base;
    struct list_head list;
    u32 head, tail;
    /* --- Cold path --- */
    char name[64] ____cacheline_aligned;
    u32 stats[32];
};
```

### Avoid False Sharing

```c
/* Per-CPU counters to avoid cache line bouncing */
struct my_stats {
    u64 packets;
    u64 bytes;
} ____cacheline_aligned_in_smp;

DEFINE_PER_CPU(struct my_stats, my_pcpu_stats);

/* In hot path: */
this_cpu_inc(my_pcpu_stats.packets);
this_cpu_add(my_pcpu_stats.bytes, len);
```

### Prefetch

```c
/* Prefetch next descriptor while processing current */
for (i = 0; i < count; i++) {
    prefetch(&desc[i + 1]);           /* Bring next into cache */
    process_descriptor(&desc[i]);
}
```

---

## 28.6 Profiling Tools

### perf — Hardware Performance Counters

```bash
# Profile driver function CPU usage
perf top -g -K  # Live kernel profiling

# Record driver activity
perf record -g -a -- sleep 10
perf report

# Count cache misses in a specific function
perf stat -e cache-misses,cache-references -a -- sleep 5
```

### ftrace Latency Measurement

```bash
# Measure function execution time
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo my_irq_handler > /sys/kernel/debug/tracing/set_graph_function
# Output shows per-call latency in microseconds
```

### /proc/interrupts Analysis

```bash
# Watch interrupt rate
watch -n1 cat /proc/interrupts

# Check for interrupt storms (>100K/s on one line)
```

---

## 28.7 Optimization Checklist

```
□ Use DMA instead of PIO for bulk transfers
□ Use threaded IRQs for long interrupt handlers
□ Implement NAPI or coalescing for high-rate devices
□ Pre-allocate ring buffers (avoid per-transfer alloc)
□ Use per-CPU data for statistics
□ Align hot structures to cache lines
□ Use readl_relaxed() where ordering permits
□ Batch register writes (avoid read-modify-write when possible)
□ Use regmap caching for slow buses (I2C, SPI)
□ Profile with perf before optimizing
```

---

## Interview Questions

**Q1: When should you use polling instead of interrupts?**
A: When the interrupt rate is very high (>100K/s), polling is more efficient because it avoids interrupt overhead (context switch, acknowledge, re-enable). NAPI uses this hybrid approach: interrupt to start, then poll until done.

**Q2: What is interrupt coalescing?**
A: Hardware batches multiple events before raising one interrupt. Example: interrupt after 64 received packets OR 50µs timeout. Trades latency for throughput — fewer interrupts means less CPU overhead but slightly delayed processing.

**Q3: How do you avoid cache line bouncing in SMP drivers?**
A: 1) Use per-CPU data (`DEFINE_PER_CPU`) for counters. 2) Align structures to cache lines (`____cacheline_aligned`). 3) Separate read-mostly data from write-frequently data. 4) Use read-copy-update (RCU) for read-heavy shared data.

**Q4: What is the advantage of scatter-gather DMA?**
A: SG-DMA transfers data directly from/to non-contiguous physical pages without first copying to a contiguous buffer. This eliminates a memcpy, reduces CPU usage, and avoids the need for large contiguous allocations.

---

*Next: [Chapter 29 — Source Code Locations](Chapter_29_Source_Code.md)*
