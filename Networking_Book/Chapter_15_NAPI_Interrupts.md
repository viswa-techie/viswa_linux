# Chapter 15: Interrupt Handling and NAPI

## Learning Goals
- Understand the interrupt → softirq → NAPI flow
- Know how NAPI prevents interrupt storms
- Understand busy polling for ultra-low latency
- Know interrupt affinity and CPU binding

---

## 15.1 The Interrupt Problem

```
Without NAPI (old model — pre-2.4):

  Packet → IRQ → ISR processes packet → done
  Packet → IRQ → ISR processes packet → done
  Packet → IRQ → ISR processes packet → done
  ...

  At 1M packets/sec → 1M interrupts/sec
  Each IRQ: save context, jump to ISR, acknowledge, restore
  CPU spends 100% on interrupt handling → "livelock"
  No time for actual processing → throughput = 0

With NAPI:
  First packet → IRQ → disable interrupts → schedule poll
  Poll processes packets 1, 2, 3, ... N (up to budget)
  If done → re-enable interrupts, wait for next batch
  If budget hit → stay in poll mode, process more next round

  At 1M packets/sec → ~15K poll invocations (budget=64)
  98.5% reduction in interrupt overhead
```

---

## 15.2 NAPI Architecture

```
┌────────────────────────────────────────────────────────────┐
│                     NAPI Flow                              │
│                                                            │
│  1. HARDWARE INTERRUPT (hardirq)                           │
│     my_irq_handler():                                      │
│       - Read interrupt cause register                      │
│       - ACK interrupt in hardware                          │
│       - Disable further interrupts (write 0 to IRQ mask)   │
│       - napi_schedule(&priv->napi)                         │
│         → Set NAPI_STATE_SCHED bit                         │
│         → Add napi to per-CPU poll list                    │
│         → __raise_softirq_irqoff(NET_RX_SOFTIRQ)          │
│       - Return IRQ_HANDLED                                 │
│                                                            │
│  2. SOFTIRQ (deferred context)                             │
│     net_rx_action():                                       │
│       - Time limit: 2 jiffies                              │
│       - Packet limit: netdev_budget (default 300)          │
│       - For each NAPI on this CPU's poll list:             │
│         → napi_poll(napi, weight=64)                       │
│           → Call driver's poll function                     │
│                                                            │
│  3. DRIVER POLL (softirq context)                          │
│     my_poll(napi, budget):                                 │
│       - Process TX completions                             │
│       - Process RX:                                        │
│         while (work_done < budget && descriptors ready):   │
│           → Build skb, pass to napi_gro_receive()          │
│           → Refill descriptor buffer                       │
│           → work_done++                                    │
│       - If work_done < budget:     ← All packets processed│
│         → napi_complete_done(napi, work_done)              │
│           → Clear NAPI_STATE_SCHED                         │
│         → Re-enable NIC interrupts                         │
│       - Return work_done                                   │
│                                                            │
│  4. BACK TO WAIT STATE                                     │
│     NIC interrupts enabled → waiting for next packet       │
│     When next packet arrives → goto step 1                 │
└────────────────────────────────────────────────────────────┘
```

### NAPI State Machine

```
          ┌─────────────┐
          │   IDLE       │  ← Interrupts enabled, waiting
          │   (no poll)  │
          └──────┬──────┘
                 │ IRQ fires → napi_schedule()
          ┌──────▼──────┐
          │  SCHEDULED   │  ← On per-CPU poll list
          │  (SCHED bit) │     Interrupts disabled
          └──────┬──────┘
                 │ NET_RX_SOFTIRQ runs
          ┌──────▼──────┐
          │  POLLING     │  ← Driver poll() running
          │              │     Processing packets
          └──────┬──────┘
                 │
        ┌────────┴────────┐
        │                 │
  work < budget     work == budget
  (all done)        (more to process)
        │                 │
        ▼                 ▼
  napi_complete_done()  Stay SCHEDULED
  Clear SCHED bit       → net_rx_action re-polls
  Re-enable IRQ         → or ksoftirqd picks up
  → Back to IDLE        → Back to POLLING
```

---

## 15.3 Interrupt Affinity

```
MSI-X provides per-queue interrupts.
Each IRQ can be pinned to a specific CPU.

Set IRQ affinity:
  # Find IRQ number for queue
  cat /proc/interrupts | grep eth0

  # Pin IRQ 45 to CPU 0
  echo 1 > /proc/irq/45/smp_affinity      # Bitmask (CPU 0)
  echo 2 > /proc/irq/46/smp_affinity      # Bitmask (CPU 1)
  echo 4 > /proc/irq/47/smp_affinity      # Bitmask (CPU 2)

  # Or use irqbalance daemon
  systemctl start irqbalance

Ideal setup:
  ┌──────────────────────────────────────────────┐
  │ Queue 0 (RX/TX) → IRQ 45 → CPU 0            │
  │   RX packets processed on CPU 0              │
  │   TCP processing on CPU 0                    │
  │   Application recv() on CPU 0                │
  │   → Complete cache locality                  │
  │                                              │
  │ Queue 1 (RX/TX) → IRQ 46 → CPU 1            │
  │   Different flows, different CPU             │
  │   → No contention with CPU 0                │
  └──────────────────────────────────────────────┘
```

---

## 15.4 Busy Polling

```
Normal path:
  Packet arrives → IRQ → NAPI (softirq) → socket queue → wake process
  Latency: IRQ delay + softirq scheduling + context switch = ~10-50 us

Busy polling:
  Application spins in kernel, directly polling NIC for packets
  No interrupt, no softirq, no context switch
  Latency: < 1-5 us

  setsockopt(fd, SOL_SOCKET, SO_BUSY_POLL, &usecs, sizeof(usecs));
  /* usecs = time to busy-poll before blocking */

  /* Or system-wide: */
  sysctl net.core.busy_poll = 50        # busy poll for 50us
  sysctl net.core.busy_read = 50        # busy read for 50us

  In epoll_wait / poll / select:
    → Before sleeping, call napi_busy_loop()
    → Direct poll: napi->poll() from process context
    → Skip IRQ and softirq entirely

  Trade-off:
    + Ultra-low latency (financial trading, HPC)
    - Burns CPU cycles while polling (no power savings)
    - Only useful for latency-sensitive workloads
```

---

## 15.5 Softirq Details

```
NET_RX_SOFTIRQ handler: net_rx_action()

Constraints:
  - Time budget: 2 jiffies (2ms at HZ=1000)
  - Packet budget: netdev_budget (default 300)
  
  If either exceeded:
    → Stop processing
    → Re-raise softirq
    → ksoftirqd kernel thread runs remaining work
    → Prevents softirq from starving user-space processes

net_rx_action() flow:
  start_time = jiffies
  budget = netdev_budget  (300)
  
  while (poll_list not empty && budget > 0 && time < 2 jiffies):
    napi = first entry on poll_list
    work = napi_poll(napi, min(weight, budget))
    budget -= work
    
    if (work < weight):
      remove napi from poll_list  (done polling)
    else:
      move napi to end of poll_list  (give others a turn)
  
  if (poll_list not empty):
    __raise_softirq_irqoff(NET_RX_SOFTIRQ)  (re-raise for ksoftirqd)
```

---

## 15.6 Interrupt Coalescing Tuning

```bash
# View current coalescing settings
ethtool -c eth0
# Adaptive RX: on
# rx-usecs: 50
# rx-frames: 64
# tx-usecs: 50
# tx-frames: 64

# Set coalescing (trade latency for throughput)
# Low latency (more interrupts):
ethtool -C eth0 rx-usecs 10 rx-frames 8

# High throughput (fewer interrupts):
ethtool -C eth0 rx-usecs 100 rx-frames 256

# Adaptive coalescing (NIC auto-adjusts):
ethtool -C eth0 adaptive-rx on adaptive-tx on

# Adaptive: NIC monitors traffic rate
#   Low rate → low coalescing (quick interrupt)
#   High rate → high coalescing (batch many packets)
```

---

## 15.7 Threaded NAPI

```
Traditional NAPI: runs in softirq context
  - Cannot sleep
  - Shares CPU with other softirqs
  - May be preempted by hardirq

Threaded NAPI (Linux 5.12+):
  - Each NAPI runs in its own kernel thread
  - Can be pinned to specific CPU via cgroup/cpuset
  - Better scheduling control
  - Can be used with RT scheduling for deterministic latency

  Enable: dev_set_threaded(dev, true)
    → Creates napi/<dev>-<queueid> threads
    → Each thread processes one NAPI instance

  ps aux | grep napi
  # napi/eth0-0, napi/eth0-1, napi/eth0-2, napi/eth0-3
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/core/dev.c | net_rx_action(), napi_schedule(), napi_poll() |
| include/linux/netdevice.h | struct napi_struct, NAPI macros |
| kernel/softirq.c | Softirq processing, ksoftirqd |
| net/core/dev.c | napi_busy_loop() for busy polling |
| net/core/dev.c | napi_threaded_poll() for threaded NAPI |

---

## Interview Questions

**Q1: Explain the NAPI mechanism step by step.**
A: (1) Packet arrives, NIC triggers interrupt. (2) ISR disables NIC interrupts and calls napi_schedule() which adds the NAPI to the per-CPU poll list and raises NET_RX_SOFTIRQ. (3) Softirq handler net_rx_action() calls the driver's poll function with a budget (default 64). (4) Poll reads RX descriptors, builds sk_buffs, calls napi_gro_receive(). (5) If all packets processed (work < budget): napi_complete_done() re-enables interrupts, NAPI goes idle. (6) If budget exhausted: stay scheduled, process more next round. This adaptively handles both low and high packet rates.

**Q2: What is the difference between hardirq, softirq, and ksoftirqd?**
A: Hardirq: hardware interrupt handler, runs immediately, preempts everything, must be fast (disable NIC IRQs + schedule NAPI). Softirq: deferred processing, runs after hardirq returns or at specific check points, softirq_action handlers (NET_RX, NET_TX). If softirqs take too long (>2 jiffies), remaining work is deferred to ksoftirqd. ksoftirqd: per-CPU kernel thread that processes softirqs, schedulable like any other thread, prevents softirq from monopolizing CPU.

**Q3: How does busy polling reduce latency?**
A: Busy polling skips the interrupt → softirq → context switch path entirely. When an application calls recv() with SO_BUSY_POLL enabled, the kernel directly calls the NAPI poll function from process context, spinning for up to the configured microseconds. Packets are processed immediately without waiting for interrupt delivery or softirq scheduling. Reduces latency from ~10-50us to <5us. The trade-off is CPU consumption — the core burns cycles while polling.

**Q4: How do you tune interrupt affinity for a multi-queue NIC?**
A: Each RX queue gets its own MSI-X interrupt. Pin each IRQ to a specific CPU via `/proc/irq/<num>/smp_affinity`. Configure RSS to distribute flows across queues. Ideally: Queue N → IRQ → CPU N, with the application thread pinned to the same CPU. This ensures complete cache locality: packet data stays in L1/L2 cache from driver through TCP to application. The irqbalance daemon can automate this, or use scripts for manual control.

**Q5: What is threaded NAPI and when would you use it?**
A: Threaded NAPI (Linux 5.12+) runs each NAPI poll in a dedicated kernel thread instead of softirq context. Benefits: better CPU scheduling control (can use RT priority), isolation between NAPI instances, ability to use cgroups for CPU pinning. Use for: real-time systems needing deterministic latency, workloads requiring per-queue CPU isolation, systems where softirq contention causes jitter.

---

## Summary

- NAPI solves interrupt storms by switching from interrupt-driven to polling mode
- Flow: hardirq → disable IRQ → schedule NAPI → softirq polls → re-enable IRQ
- Net_rx_action processes up to 300 packets or 2ms per softirq invocation
- Interrupt affinity + RSS pins queues to CPUs for cache locality
- Busy polling: ultra-low latency by bypassing interrupts entirely
- Interrupt coalescing: batch interrupts for throughput vs. latency trade-off
- Threaded NAPI provides per-queue kernel threads for better scheduling

---

Next: [Chapter 16 — Network Queues and Multiqueue](Chapter_16_Queues_Multiqueue.md)
