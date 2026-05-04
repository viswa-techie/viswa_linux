# Chapter 21: Traffic Control and QoS

## Learning Goals
- Understand the Linux TC (Traffic Control) framework
- Know qdisc types: classful and classless
- Understand shaping, scheduling, policing, and dropping
- Know how to configure TC rules for QoS

---

## 21.1 TC Architecture

```
TC (Traffic Control): Controls how packets are queued and dequeued
on output interfaces. Located between IP output and the driver TX queue.

┌──────────────────── TC Pipeline ────────────────────┐
│                                                      │
│  ip_output()                                         │
│       │                                              │
│       ▼                                              │
│  __dev_queue_xmit()                                  │
│       │                                              │
│       ▼                                              │
│  ┌────────────────────────────────────┐               │
│  │  Root Qdisc                        │               │
│  │  ┌──────────────────────────────┐  │               │
│  │  │ Classifier (filter/match)    │  │               │
│  │  │   → class 1:1 (high prio)   │  │               │
│  │  │   → class 1:2 (normal)      │  │               │
│  │  │   → class 1:3 (bulk)        │  │               │
│  │  └──────────────────────────────┘  │               │
│  │                                    │               │
│  │  Each class has a child qdisc      │               │
│  │  ┌──────┐ ┌──────┐ ┌──────┐       │               │
│  │  │qdisc │ │qdisc │ │qdisc │       │               │
│  │  │(SFQ) │ │(SFQ) │ │(TBF) │       │               │
│  │  └──────┘ └──────┘ └──────┘       │               │
│  └────────────────────────────────────┘               │
│       │                                              │
│       ▼                                              │
│  Driver TX Queue (ring buffer)                       │
│       │                                              │
│       ▼                                              │
│  [Network]                                           │
└──────────────────────────────────────────────────────┘

TC components:
  qdisc:      Queueing discipline — how packets are queued/dequeued
  class:      Sub-division within a classful qdisc
  filter:     Classifier that directs packets to classes
  action:     Additional action (mark, mirror, redirect)
```

---

## 21.2 Classless Qdiscs

```
No internal class hierarchy. Simple queuing algorithms.

pfifo_fast (legacy default):
  3-band priority queue (0=highest, 2=lowest)
  TOS field determines band assignment
  Dequeue: serve band 0 first, then 1, then 2

fq_codel (modern default, since kernel 3.12):
  Fair Queuing + Controlled Delay
  ┌──────────────────────────────────────┐
  │  Incoming packets                     │
  │       │                               │
  │       ▼                               │
  │  Hash flow (5-tuple) → bucket         │
  │       │                               │
  │  ┌────┴────┐                          │
  │  │ Flow 1  │ Flow 2  │ Flow 3  │ ... │
  │  │ [queue] │ [queue] │ [queue] │      │
  │  └────┬────┘                          │
  │       │                               │
  │  Round-robin dequeue (DRR)            │
  │  Per-flow CoDel AQM (drops if        │
  │    sojourn time > target=5ms)         │
  └──────────────────────────────────────┘

  fq_codel advantages:
  - Isolates flows: one bulk transfer can't starve others
  - Controls bufferbloat via CoDel AQM
  - No configuration needed (good defaults)

TBF (Token Bucket Filter):
  Rate limiting to specific bandwidth
  ┌─────────────────────────┐
  │  Tokens arrive at rate  │
  │  Bucket size = burst    │
  │  Each packet needs tokens│
  │  No tokens → queue/drop │
  └─────────────────────────┘

  tc qdisc add dev eth0 root tbf rate 10mbit burst 32kbit latency 50ms

SFQ (Stochastic Fairness Queuing):
  Hash flows into buckets, round-robin dequeue
  Fair sharing without guarantees

RED (Random Early Detection):
  Drop packets probabilistically before queue is full
  Reduces tail-drop synchronization
```

---

## 21.3 Classful Qdiscs

```
HTB (Hierarchical Token Bucket):
  Most popular classful qdisc. Guarantees bandwidth
  and allows borrowing.

  Example:  100Mbit link, 3 classes
  ┌──────── HTB root (rate 100mbit) ────────┐
  │                                          │
  │  class 1:1 (rate 100mbit, ceil 100mbit)  │
  │    │                                     │
  │    ├── class 1:10 (rate 50mbit, ceil 100mbit) │ Guaranteed 50, can burst to 100
  │    │   └── qdisc: fq_codel               │
  │    │                                     │
  │    ├── class 1:20 (rate 30mbit, ceil 80mbit)  │ Guaranteed 30, max 80
  │    │   └── qdisc: fq_codel               │
  │    │                                     │
  │    └── class 1:30 (rate 20mbit, ceil 50mbit)  │ Guaranteed 20, max 50
  │        └── qdisc: fq_codel               │
  └──────────────────────────────────────────┘

  rate:  Guaranteed minimum bandwidth
  ceil:  Maximum bandwidth (borrowing from parent)
  burst: Burst size for tokens

  # Configuration:
  tc qdisc add dev eth0 root handle 1: htb default 30

  tc class add dev eth0 parent 1: classid 1:1 htb rate 100mbit ceil 100mbit
  tc class add dev eth0 parent 1:1 classid 1:10 htb rate 50mbit ceil 100mbit
  tc class add dev eth0 parent 1:1 classid 1:20 htb rate 30mbit ceil 80mbit
  tc class add dev eth0 parent 1:1 classid 1:30 htb rate 20mbit ceil 50mbit

  # Attach leaf qdiscs
  tc qdisc add dev eth0 parent 1:10 handle 10: fq_codel
  tc qdisc add dev eth0 parent 1:20 handle 20: fq_codel
  tc qdisc add dev eth0 parent 1:30 handle 30: fq_codel

CBQ (Class-Based Queueing):
  Older classful qdisc, complex configuration.
  Similar goals as HTB but harder to use.
  Generally HTB is preferred.

PRIO:
  Priority-based scheduling with fixed classes.
  tc qdisc add dev eth0 root handle 1: prio
  # Creates 3 bands (1:1, 1:2, 1:3), strict priority
```

---

## 21.4 TC Filters and Classifiers

```bash
# Filters match packets and direct them to classes

# Match by destination port → class 1:10
tc filter add dev eth0 parent 1: protocol ip prio 1 u32 \
   match ip dport 80 0xffff flowid 1:10

# Match by source IP → class 1:20
tc filter add dev eth0 parent 1: protocol ip prio 2 u32 \
   match ip src 10.0.0.0/8 flowid 1:20

# Match by TOS field
tc filter add dev eth0 parent 1: protocol ip prio 3 u32 \
   match ip tos 0x10 0xff flowid 1:10

# Match by firewall mark (set by iptables)
iptables -t mangle -A OUTPUT -p tcp --dport 443 -j MARK --set-mark 1
tc filter add dev eth0 parent 1: protocol ip prio 1 handle 1 fw flowid 1:10

# BPF classifier (programmable):
tc filter add dev eth0 parent 1: bpf obj filter.o flowid 1:10

# Flower classifier (hardware offload capable):
tc filter add dev eth0 parent ffff: protocol ip flower \
   ip_proto tcp dst_port 80 action mirred egress redirect dev eth1

Filter types:
  u32:     Match on arbitrary packet header fields
  fw:      Match on firewall mark (nfmark)
  flower:  Modern, hardware-offloadable
  bpf:     eBPF program classifier
  cgroup:  Match by cgroup
  route:   Match by route attributes
```

---

## 21.5 Ingress TC and Policing

```
TC normally operates on egress. Ingress qdisc allows
limited control on incoming packets.

Ingress policing: rate-limit incoming traffic
  tc qdisc add dev eth0 ingress
  tc filter add dev eth0 parent ffff: protocol ip u32 \
     match u32 0 0 \
     action police rate 10mbit burst 100k conform-exceed drop

Policing vs Shaping:
  ┌──────────────────────────────────────────────┐
  │ Shaping (egress):                             │
  │   Delays packets in queue to match rate       │
  │   Smooth output, uses memory for buffering    │
  │   Better for controlled bandwidth             │
  │                                               │
  │ Policing (ingress/egress):                    │
  │   Drops excess packets immediately            │
  │   No queuing, immediate enforcement           │
  │   Sender must retransmit (TCP) or accept loss │
  └──────────────────────────────────────────────┘

IFB (Intermediate Functional Block):
  Redirect ingress to IFB device for full shaping

  modprobe ifb
  ip link set ifb0 up
  tc qdisc add dev eth0 ingress
  tc filter add dev eth0 parent ffff: protocol ip u32 \
     match u32 0 0 action mirred egress redirect dev ifb0
  # Now apply HTB/fq_codel on ifb0 (full shaping for ingress)
  tc qdisc add dev ifb0 root handle 1: htb default 10
  tc class add dev ifb0 parent 1: classid 1:10 htb rate 50mbit
```

---

## 21.6 Practical QoS Scenarios

```bash
# === Scenario 1: Simple bandwidth limit ===
tc qdisc add dev eth0 root tbf rate 50mbit burst 32kbit latency 50ms

# === Scenario 2: Prioritize SSH over bulk downloads ===
tc qdisc add dev eth0 root handle 1: prio
# Band 0 (highest): interactive (SSH)
# Band 1: normal
# Band 2: bulk
tc filter add dev eth0 parent 1: protocol ip prio 1 u32 \
   match ip dport 22 0xffff flowid 1:1
tc filter add dev eth0 parent 1: protocol ip prio 2 u32 \
   match ip dport 80 0xffff flowid 1:2

# === Scenario 3: Full HTB with guarantees ===
# See Section 21.3 for complete HTB setup

# === Scenario 4: Simulate network conditions (netem) ===
# Add 100ms delay with 10ms jitter
tc qdisc add dev eth0 root netem delay 100ms 10ms

# Add 1% packet loss
tc qdisc add dev eth0 root netem loss 1%

# Add both delay and loss
tc qdisc add dev eth0 root netem delay 50ms loss 0.5%

# Simulate bandwidth + delay (chain qdiscs)
tc qdisc add dev eth0 root handle 1: netem delay 100ms
tc qdisc add dev eth0 parent 1: handle 2: tbf rate 1mbit burst 32kbit latency 200ms

# === View current TC config ===
tc -s qdisc show dev eth0
tc -s class show dev eth0
tc filter show dev eth0

# === Remove all TC config ===
tc qdisc del dev eth0 root
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/sched/sch_generic.c | Core qdisc framework |
| net/sched/sch_htb.c | HTB qdisc |
| net/sched/sch_fq_codel.c | fq_codel qdisc |
| net/sched/sch_tbf.c | TBF qdisc |
| net/sched/sch_netem.c | Network emulator |
| net/sched/cls_u32.c | u32 classifier |
| net/sched/cls_flower.c | flower classifier |
| net/sched/cls_bpf.c | BPF classifier |
| net/sched/act_mirred.c | Mirror/redirect action |
| net/sched/act_police.c | Policing action |
| include/net/pkt_sched.h | Scheduler definitions |

---

## Interview Questions

**Q1: What is the difference between shaping, scheduling, policing, and dropping?**
A: **Shaping** delays packets in a queue to match a target rate — smooth output using token bucket. **Scheduling** determines the order packets are dequeued — priority-based (PRIO) or weighted (HTB). **Policing** drops or re-marks packets that exceed a rate — no queuing, immediate action. **Dropping** removes packets from a queue before it's full — RED drops probabilistically, CoDel drops based on sojourn time, tail-drop drops when queue is full.

**Q2: Explain fq_codel and why it's the default qdisc.**
A: fq_codel combines Fair Queuing (FQ) with Controlled Delay (CoDel). FQ hashes packets by flow (5-tuple) into separate buckets, dequeuing round-robin — this prevents one flow from starving others. CoDel monitors each flow's sojourn time (time packet spends in queue); if consistently > 5ms target, it starts dropping to signal congestion. This fights bufferbloat without manual tuning. It's default because it works well out-of-the-box for most workloads.

**Q3: How do you set up bandwidth guarantees with HTB?**
A: (1) Create root HTB qdisc. (2) Create a root class with the link's total bandwidth. (3) Create child classes with `rate` (guaranteed minimum) and `ceil` (maximum with borrowing). rates must sum to ≤ parent rate. (4) Attach leaf qdiscs (fq_codel) to each leaf class. (5) Add filters (u32, flower, fw) to classify packets into classes. (6) Set a default class for unmatched traffic. When a class doesn't use its guaranteed bandwidth, other classes can borrow up to their ceil.

**Q4: How does TC interact with netfilter?**
A: TC operates at L2/L3 on the output path, after netfilter POST_ROUTING. However, iptables mangle can set packet marks (`-j MARK --set-mark N`) that TC filters can match (`handle N fw`), allowing integrated classification. TC also has ingress qdiscs that run before netfilter on input. TC actions can redirect packets (mirred), effectively bypassing normal kernel forwarding.

---

## Summary

- TC framework: qdiscs (queuing algorithms), classes (subdivisions), filters (classifiers)
- Classless qdiscs: fq_codel (default, anti-bufferbloat), TBF (rate limit), SFQ (fairness)
- Classful qdiscs: HTB (bandwidth guarantees + borrowing), PRIO (strict priority)
- Filters: u32 (header match), fw (mark), flower (HW offload), BPF (programmable)
- Ingress: policing only (drop excess) — use IFB for full shaping
- netem: simulate delay, loss, jitter for testing
- Shaping = delay packets, Policing = drop packets, Scheduling = reorder packets

---

Next: [Chapter 22 — Network Performance Optimization](Chapter_22_Performance_Optimization.md)
