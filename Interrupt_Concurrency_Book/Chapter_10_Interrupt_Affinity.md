# Chapter 10: Interrupt Affinity and CPU Distribution

## Learning Goals
- Understand how interrupts are distributed across CPUs in SMP systems
- Configure interrupt affinity via /proc and kernel API
- Know irqbalance and manual interrupt steering strategies
- Optimize interrupt distribution for throughput and latency

---

## 10.1 Interrupt Affinity Concept

```
On SMP systems, each IRQ can be directed to specific CPUs.
This is "interrupt affinity" — which CPU(s) handle which IRQ.

Default: Usually CPU 0 handles everything (not optimal!)

  Without affinity tuning:      With affinity tuning:
  ┌──────┐┌──────┐             ┌──────┐┌──────┐
  │ CPU 0││ CPU 1│             │ CPU 0││ CPU 1│
  │ IRQ 1││      │             │ IRQ 1││ IRQ 3│
  │ IRQ 2││      │             │ IRQ 2││ IRQ 4│
  │ IRQ 3││      │             │      ││      │
  │ IRQ 4││      │             │      ││      │
  │BUSY! ││idle  │             │balanced│balanced│
  └──────┘└──────┘             └──────┘└──────┘
```

### Why Affinity Matters

```
Performance:
  - Cache locality: Keep IRQ handler on same CPU as data consumer
  - NUMA: Handle interrupt on CPU close to device (no remote memory)
  - Throughput: Spread high-frequency IRQs across CPUs

Determinism (RT):
  - Isolate CPUs: Keep IRQs off CPUs running RT tasks
  - Bind specific device IRQs to non-RT CPUs
  - Prevent interrupt jitter on safety-critical CPUs
```

---

## 10.2 Distributing Interrupts Across CPUs

### Manual: /proc/irq/N/smp_affinity

```bash
# View current affinity (hex bitmask):
$ cat /proc/irq/33/smp_affinity
f       # 0xf = 1111 binary = CPUs 0,1,2,3

# Set to CPU 2 only:
$ echo 4 > /proc/irq/33/smp_affinity   # 0x4 = 0100 = CPU 2

# Set to CPUs 0 and 2:
$ echo 5 > /proc/irq/33/smp_affinity   # 0x5 = 0101 = CPU 0,2

# CPU list format (easier to read):
$ cat /proc/irq/33/smp_affinity_list
0-3
$ echo 2 > /proc/irq/33/smp_affinity_list    # CPU 2 only
$ echo 0,2 > /proc/irq/33/smp_affinity_list  # CPUs 0 and 2
```

### Kernel API: irq_set_affinity

```c
#include <linux/interrupt.h>

/* Set affinity to specific CPU: */
cpumask_t mask;
cpumask_clear(&mask);
cpumask_set_cpu(2, &mask);
irq_set_affinity(irq_num, &mask);

/* Set affinity hint (advisory, for irqbalance): */
irq_set_affinity_hint(irq_num, &mask);

/* In modern drivers (managed IRQs): */
/* Affinity is set automatically for MSI-X vectors */
struct irq_affinity affd = {
    .pre_vectors = 1,    /* admin queue */
    .post_vectors = 0,
};
pci_alloc_irq_vectors_affinity(pdev, min, max,
    PCI_IRQ_MSI_X | PCI_IRQ_AFFINITY, &affd);
```

---

## 10.3 Load Balancing Interrupts

### irqbalance Daemon

```
irqbalance is a user-space daemon that automatically
distributes interrupts across CPUs based on load.

  $ systemctl status irqbalance
  
  How it works:
    1. Reads /proc/interrupts periodically
    2. Calculates interrupt load per CPU
    3. Considers NUMA topology
    4. Writes to /proc/irq/*/smp_affinity
    5. Respects hints from irq_set_affinity_hint()

  Configuration:
    /etc/default/irqbalance
    IRQBALANCE_BANNED_CPUS="0000000c"  # Exclude CPUs 2,3
    IRQBALANCE_ONESHOT=0               # Continuous mode
```

### Manual Distribution Strategy

```
For high-performance systems, manual is often better:

NVMe SSD with MSI-X (4 queues, 4 CPUs):
  Queue 0 (IRQ 33) → CPU 0
  Queue 1 (IRQ 34) → CPU 1
  Queue 2 (IRQ 35) → CPU 2
  Queue 3 (IRQ 36) → CPU 3

  for i in 0 1 2 3; do
    echo $((1 << i)) > /proc/irq/$((33+i))/smp_affinity
  done

Network card (RSS + IRQ affinity):
  RX Queue 0 → IRQ → CPU 0 → App thread pinned to CPU 0
  RX Queue 1 → IRQ → CPU 1 → App thread pinned to CPU 1
  ...
  (Complete data locality: IRQ + processing on same CPU)
```

---

## 10.4 Interrupt Steering Mechanisms

### RSS (Receive Side Scaling) — Network

```
RSS distributes network packets across RX queues by hash:
  Packet hash (src/dst IP+port) → queue selection → per-queue IRQ

  ┌────────┐    hash     ┌─────┐    IRQ     ┌──────┐
  │ NIC    │───────────→ │ Q0  │───────────→│ CPU 0│
  │        │             │ Q1  │───────────→│ CPU 1│
  │        │             │ Q2  │───────────→│ CPU 2│
  │        │             │ Q3  │───────────→│ CPU 3│
  └────────┘             └─────┘            └──────┘

  # Configure RSS:
  ethtool -L eth0 combined 4  # 4 queues
  # Indirection table: maps hash → queue
  ethtool -X eth0 equal 4      # Equal distribution
```

### RPS (Receive Packet Steering) — Software

```
RPS distributes processing across CPUs in software
(for NICs without hardware RSS):

  # Enable RPS on all 4 CPUs:
  echo f > /sys/class/net/eth0/queues/rx-0/rps_cpus

  Packet arrives on CPU 0 → hash → IPI to CPU 2 → process on CPU 2
```

### RT CPU Isolation

```bash
# Boot parameter: isolate CPUs from general IRQ handling
# Kernel command line:
isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3

# After boot, CPUs 2-3 receive NO interrupts by default
# Move all IRQs to CPUs 0-1:
for irq in /proc/irq/*/smp_affinity; do
    echo 3 > "$irq" 2>/dev/null  # 0x3 = CPUs 0,1
done

# Pin RT application to isolated CPUs:
taskset -c 2,3 chrt -f 80 ./my_rt_app
```

---

## Practical Example: Complete IRQ Affinity Script

```bash
#!/bin/bash
# optimize_irqs.sh — Production interrupt affinity setup

# Disable irqbalance (manual control)
systemctl stop irqbalance

# NVMe: per-queue affinity
NVME_BASE_IRQ=33
for i in $(seq 0 3); do
    echo $((1 << i)) > /proc/irq/$((NVME_BASE_IRQ + i))/smp_affinity
    echo "NVMe Q$i → CPU $i"
done

# Network: per-queue affinity
ETH_BASE_IRQ=37
for i in $(seq 0 3); do
    echo $((1 << i)) > /proc/irq/$((ETH_BASE_IRQ + i))/smp_affinity
    echo "eth0 Q$i → CPU $i"  
done

# Isolate CPUs 4-7 for RT (no interrupts)
for irq_dir in /proc/irq/*/; do
    echo f > "${irq_dir}smp_affinity" 2>/dev/null  # CPUs 0-3 only
done

# Verify
echo "=== IRQ Distribution ==="
cat /proc/interrupts | head -20
```

---

## Kernel Source References

```
Affinity management:
  kernel/irq/manage.c          ← irq_set_affinity, __irq_set_affinity
  kernel/irq/proc.c            ← /proc/irq/N/smp_affinity handlers
  include/linux/interrupt.h    ← irq_set_affinity_hint API

MSI affinity:
  kernel/irq/affinity.c        ← irq_create_affinity_masks
  drivers/pci/msi.c            ← pci_alloc_irq_vectors_affinity

Architecture:
  arch/x86/kernel/apic/io_apic.c  ← x86 APIC affinity
  drivers/irqchip/irq-gic-v3.c    ← ARM GIC affinity
```

---

## Interview Questions

1. **What is interrupt affinity and why does it matter for performance?**
2. **How do you set IRQ affinity using /proc?**
3. **What is irqbalance and when would you disable it?**
4. **Explain the kernel API irq_set_affinity_hint() vs irq_set_affinity().**
5. **How does MSI-X enable per-queue interrupt affinity?**
6. **What is RSS and how does it relate to interrupt distribution?**
7. **Design an IRQ affinity scheme for a server with 2 NICs and 4 NVMe SSDs.**
8. **How do you isolate CPUs from interrupts for RT tasks?**
9. **What boot parameters help with CPU isolation?**
10. **What is the difference between RPS and RSS?**
11. **How does NUMA topology affect optimal IRQ placement?**
12. **What happens if you set affinity to a CPU that is offline?**

---

## Summary

- Interrupt affinity controls which CPU(s) handle each IRQ — critical for performance and RT
- /proc/irq/N/smp_affinity and kernel APIs control affinity
- irqbalance automates distribution; disable for manual tuning in specialized systems
- MSI-X enables per-queue vectors for perfect data locality
- CPU isolation (isolcpus, nohz_full) keeps interrupts off RT CPUs
- Match IRQ affinity to data flow for best cache behavior

---

*Next: [Chapter 11 — Inter-Processor Interrupts (IPI)](Chapter_11_IPI.md)*
