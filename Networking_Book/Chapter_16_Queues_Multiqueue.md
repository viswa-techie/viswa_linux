# Chapter 16: Network Queues and Multiqueue

## Learning Goals
- Understand TX/RX queue architecture in Linux
- Know RSS, RPS, XPS, RFS and their roles
- Understand qdisc basics (traffic control at queue level)
- Know how multiqueue NICs scale across CPUs

---

## 16.1 Queue Architecture

```
                    TX Side                         RX Side
              ┌──────────────┐                ┌──────────────┐
              │  Application │                │  Application │
              │  send()      │                │  recv()       │
              └──────┬───────┘                └──────▲───────┘
                     │                               │
              ┌──────▼───────┐                ┌──────┴───────┐
              │ Socket Buffer│                │ Socket Buffer│
              │ sk_write_queue│               │sk_receive_queue│
              └──────┬───────┘                └──────▲───────┘
                     │                               │
              ┌──────▼───────┐                ┌──────┴───────┐
              │  TC (qdisc)  │                │  Protocol    │
              │  per-queue   │                │  Processing  │
              └──────┬───────┘                └──────▲───────┘
                     │                               │
        ┌────────────┼────────────┐      ┌───────────┼───────────┐
        ▼            ▼            ▼      ▼           ▼           ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
   │ TX Q 0  │ │ TX Q 1  │ │ TX Q 2  │ │ RX Q 0  │ │ RX Q 1  │ │ RX Q 2  │
   └────┬────┘ └────┬────┘ └────┬────┘ └────▲────┘ └────▲────┘ └────▲────┘
        │           │           │           │           │           │
   ┌────▼────┐ ┌────▼────┐ ┌────▼────┐ ┌────┴────┐ ┌────┴────┐ ┌────┴────┐
   │TX Ring 0│ │TX Ring 1│ │TX Ring 2│ │RX Ring 0│ │RX Ring 1│ │RX Ring 2│
   │ (DMA)   │ │ (DMA)   │ │ (DMA)   │ │ (DMA)   │ │ (DMA)   │ │ (DMA)   │
   └────┬────┘ └────┬────┘ └────┬────┘ └────▲────┘ └────▲────┘ └────▲────┘
        │           │           │           │           │           │
        └───────────┴─────┬─────┴───────────┴───────────┴───────────┘
                          │
                    ┌─────▼─────┐
                    │    NIC    │
                    └───────────┘
```

---

## 16.2 RSS — Receive Side Scaling

```
RSS: NIC distributes incoming packets across RX queues using a hash.

Flow:
  Packet arrives at NIC
    → NIC computes hash: hash(src_ip, dst_ip, src_port, dst_port)
    → NIC looks up indirection table: queue = table[hash % table_size]
    → Packet placed in selected RX queue
    → Queue's MSI-X interrupt fires on assigned CPU

  Result: different flows → different queues → different CPUs
          same flow → same queue → same CPU (ordering preserved)

  ┌───────────────────────────────────────────────────┐
  │ Indirection Table (e.g., 128 entries)            │
  │ [0]=Q0 [1]=Q1 [2]=Q2 [3]=Q3 [4]=Q0 [5]=Q1 ...  │
  │                                                   │
  │ Hash Function: Toeplitz hash with random key      │
  └───────────────────────────────────────────────────┘

Configuration:
  # View RSS settings
  ethtool -x eth0              # Show indirection table
  ethtool -l eth0              # Show queue count
  
  # Set number of queues
  ethtool -L eth0 combined 4   # 4 combined RX/TX queues
  
  # Change indirection table (equal distribution to 4 queues)
  ethtool -X eth0 equal 4
  
  # Set RSS hash key
  ethtool -X eth0 hkey <key>
  
  # Set hash fields
  ethtool -N eth0 rx-flow-hash tcp4 sdfn  # src/dst IP + src/dst port
```

---

## 16.3 RPS — Receive Packet Steering

```
RPS: Software-based RSS for NICs without hardware RSS.

Problem: NIC has 1 RX queue → all packets on 1 CPU.
RPS: After NAPI poll, steer packet to target CPU via IPI.

Flow:
  CPU 0: NAPI poll → reads all packets from single RX queue
    → For each skb:
      → hash = skb_get_hash(skb)  (compute flow hash)
      → cpu = hash % rps_cpu_count
      → enqueue_to_backlog(skb, cpu)
        → If cpu != current: send IPI to target CPU
    → Target CPU: process_backlog() → netif_receive_skb()

  ┌──────────┐     IPI      ┌──────────┐
  │  CPU 0   │──────────────►│  CPU 1   │
  │ (NAPI    │               │ (process │
  │  poll)   │               │  packet) │
  └──────────┘               └──────────┘
       ↑
  1 RX Queue                  ← NIC has only 1 queue 
  (all packets)                  but RPS distributes

Configuration:
  # Enable RPS on queue 0 to CPUs 0-3 (bitmask f = 1111)
  echo f > /sys/class/net/eth0/queues/rx-0/rps_cpus
  
  # Set backlog size
  sysctl net.core.netdev_max_backlog = 1000
```

---

## 16.4 RFS — Receive Flow Steering

```
RFS: Steers packets to the CPU where the application is running.

Problem: RSS/RPS sends packet to CPU N, but application runs on CPU M.
  → Cache miss: packet data in CPU N's cache, processed on CPU M.

RFS: Tracks which CPU each flow's socket is using.
  When application calls recv():
    → rps_sock_flow_table[hash] = current CPU
  When packet arrives:
    → desired_cpu = rps_sock_flow_table[hash]
    → Steer to desired_cpu

  Result: packet and application on SAME CPU → cache hits

Configuration:
  # Enable RFS
  echo 32768 > /proc/sys/net/core/rps_sock_flow_entries
  echo 2048 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
```

---

## 16.5 XPS — Transmit Packet Steering

```
XPS: Maps CPUs to TX queues.

Without XPS: Any CPU can submit to any TX queue → cache contention.
With XPS: CPU N sends to TX Queue N → no cross-CPU contention.

  ┌───────────┐   ┌───────────┐   ┌───────────┐
  │   CPU 0   │   │   CPU 1   │   │   CPU 2   │
  │ (app thd) │   │ (app thd) │   │ (app thd) │
  └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
        │               │               │
        ▼               ▼               ▼
   ┌─────────┐    ┌─────────┐    ┌─────────┐
   │ TX Q 0  │    │ TX Q 1  │    │ TX Q 2  │
   └─────────┘    └─────────┘    └─────────┘

Configuration:
  echo 1 > /sys/class/net/eth0/queues/tx-0/xps_cpus  # CPU 0 → Q0
  echo 2 > /sys/class/net/eth0/queues/tx-1/xps_cpus  # CPU 1 → Q1
  echo 4 > /sys/class/net/eth0/queues/tx-2/xps_cpus  # CPU 2 → Q2
```

---

## 16.6 Scaling Summary

```
┌─────────┬───────────┬──────────────────────────────────────┐
│ Feature │ Direction │ Purpose                              │
├─────────┼───────────┼──────────────────────────────────────┤
│ RSS     │ RX (HW)   │ NIC distributes to RX queues by hash│
│ RPS     │ RX (SW)   │ Software RSS for single-queue NICs  │
│ RFS     │ RX (SW)   │ Steer to CPU running the application│
│ XPS     │ TX (SW)   │ Map CPUs to TX queues              │
│ aRFS    │ RX (HW)   │ HW-accelerated RFS (flow steering) │
└─────────┴───────────┴──────────────────────────────────────┘

Ideal combination for multi-core:
  RSS  → distribute RX across CPUs at NIC level
  RFS  → ensure packet → application CPU affinity
  XPS  → pin TX queues to CPUs
  IRQ affinity → pin queue interrupts to specific CPUs

  Result: packet arrives → processed → delivered to app → all on same CPU
```

---

## 16.7 Qdisc Basics (TX Scheduling)

```
Every net_device has a qdisc (queueing discipline) per TX queue.
Qdisc decides: which packet to send next.

Default qdisc: pfifo_fast or fq_codel (since ~Linux 4.x)

Qdisc types:
  pfifo_fast: 3-band priority FIFO (legacy default)
  fq_codel:   Fair Queueing + Controlled Delay (modern default)
  htb:        Hierarchical Token Bucket (bandwidth shaping)
  tbf:        Token Bucket Filter (simple rate limiting)
  prio:       Priority scheduling
  noqueue:    No qdisc (virtual devices, direct xmit)
  mq:         Multi-queue wrapper (one child qdisc per HW queue)

  ┌──────────────────────────────────────┐
  │ Qdisc per TX Queue                  │
  │                                      │
  │ dev_queue_xmit(skb)                 │
  │   → q->enqueue(skb)    add to qdisc │
  │   → __qdisc_run(q)     try to send  │
  │     → q->dequeue()     get next skb │
  │     → sch_direct_xmit()             │
  │       → dev_hard_start_xmit()       │
  │         → ndo_start_xmit()          │
  └──────────────────────────────────────┘
```

### fq_codel (Default Modern Qdisc)

```
fq_codel = Fair Queuing + CoDel (Controlled Delay)

FQ: Multiple internal flows (one per 5-tuple hash).
    Each flow gets fair share of bandwidth.
    Prevents single flow from starving others.

CoDel: Monitors packet sojourn time (time spent in queue).
    If packets queue > target (5ms for 100ms interval):
      → Start dropping packets gradually
      → Signal ECN if supported
    Automatically controls queue depth → controls latency.

    No manual tuning needed — works well out of the box.
    Called "no knobs" algorithm.

Check current qdisc:
  tc qdisc show dev eth0
  # qdisc fq_codel 0: root refcnt 2 limit 10240p ...
```

---

## 16.8 Queue Statistics

```bash
# View per-queue stats
ls /sys/class/net/eth0/queues/

# TX queue info
cat /sys/class/net/eth0/queues/tx-0/tx_timeout

# RX queue info
cat /sys/class/net/eth0/queues/rx-0/rps_cpus

# NIC queue statistics
ethtool -S eth0 | grep -i queue
# tx_queue_0_packets: 123456
# rx_queue_0_packets: 789012
# rx_queue_0_drops: 0

# Ring sizes
ethtool -g eth0
# RX: 4096   (maximum)
# RX: 256    (current)
# TX: 4096
# TX: 256

# Set ring size
ethtool -G eth0 rx 4096 tx 4096

# View queue count
ethtool -l eth0
# Combined: 4  (current)
# Maximum:  8
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/core/dev.c | dev_queue_xmit(), netdev_pick_tx() |
| net/core/dev.c | enqueue_to_backlog() (RPS), get_rps_cpu() |
| net/core/flow_dissector.c | skb_get_hash() for RSS/RPS |
| net/sched/sch_generic.c | qdisc_run(), sch_direct_xmit() |
| net/sched/sch_fq_codel.c | fq_codel qdisc |
| include/linux/netdevice.h | netdev_queue, NAPI definition |

---

## Interview Questions

**Q1: What is RSS and how does it improve performance?**
A: RSS (Receive Side Scaling) is a NIC hardware feature that distributes incoming packets across multiple RX queues using a hash of the flow's 4-tuple (src/dst IP and port). Each queue has its own MSI-X interrupt pinned to a specific CPU. Result: different flows processed on different CPUs in parallel, eliminating the single-CPU bottleneck. Same flow always goes to the same queue, preserving ordering.

**Q2: What is the difference between RSS, RPS, and RFS?**
A: RSS is hardware — NIC distributes RX packets to queues by hash. RPS is software RSS — for NICs without hardware RSS, the kernel distributes packets across CPUs via IPIs after NAPI poll. RFS extends RPS by steering packets to the CPU where the consuming application is running, improving cache locality. RSS is preferred (no CPU overhead); RPS is a fallback; RFS optimizes application affinity.

**Q3: What is fq_codel and why is it the default qdisc?**
A: fq_codel combines Fair Queuing (one queue per flow, preventing any single flow from starving others) with CoDel (Controlled Delay: drops packets that sit in the queue too long, controlling latency). It's self-tuning — no manual configuration needed. It fights "bufferbloat" (excessive queuing that adds latency) by monitoring packet sojourn time and dropping proactively. This provides both fairness and low latency for all flows.

**Q4: How do you optimize a multi-core server for high packet rates?**
A: (1) Set queue count equal to CPU count: `ethtool -L eth0 combined N`. (2) Pin each queue's IRQ to a specific CPU: `/proc/irq/<n>/smp_affinity`. (3) Enable RSS with good hash distribution: `ethtool -X eth0 equal N`. (4) Configure XPS to pin TX queues to CPUs. (5) Enable RFS for application CPU affinity. (6) Increase ring sizes: `ethtool -G eth0 rx 4096 tx 4096`. (7) Ensure application threads are pinned to corresponding CPUs. (8) Enable GRO (default on). (9) Consider busy polling for ultra-low latency.

---

## Summary

- Multiqueue NICs have independent TX/RX queues with per-queue IRQs
- RSS distributes RX packets in hardware by flow hash; RPS does it in software
- RFS steers packets to the CPU running the application for cache locality
- XPS maps CPUs to TX queues to reduce cross-CPU contention
- qdisc per TX queue controls scheduling (fq_codel is the modern default)
- fq_codel provides fair bandwidth sharing and automatic latency control

---

Next: [Chapter 17 — Network Namespaces and Containers](Chapter_17_Network_Namespaces.md)
