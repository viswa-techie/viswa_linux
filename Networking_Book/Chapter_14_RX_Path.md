# Chapter 14: Packet Reception Path

## Learning Goals
- Trace every step from wire to recv() with kernel functions
- Understand GRO aggregation
- Know NAPI poll and softirq interaction in RX
- Understand packet type dispatch and protocol demultiplexing

---

## 14.1 Complete RX Path

```
┌───────────────────────────────────────────────────────────────┐
│ 1. HARDWARE                                                   │
│    PHY receives signal, demodulates to bits                   │
│    MAC verifies preamble, CRC — drops if bad                  │
│    MAC checks dst address filter (unicast/broadcast/multicast)│
│    DMA engine reads next RX descriptor (buffer address)        │
│    DMA engine writes packet data to buffer in host memory     │
│    DMA engine updates RX descriptor (length, status, csum)    │
│    Interrupt controller triggers IRQ (or coalesced)           │
└───────────────────────────────────────┬───────────────────────┘
                                        │
┌───────────────────────────────────────▼───────────────────────┐
│ 2. DRIVER — Interrupt Handler (hardirq context)               │
│    my_irq_handler(irq, dev_id)                                │
│    → Read interrupt cause register                            │
│    → Acknowledge interrupt in hardware                        │
│    → Disable NIC interrupts (write 0 to IRQ mask register)    │
│    → napi_schedule(&priv->napi)                               │
│      → Set NAPI_STATE_SCHED                                   │
│      → __raise_softirq_irqoff(NET_RX_SOFTIRQ)                │
│    → return IRQ_HANDLED                                       │
└───────────────────────────────────────┬───────────────────────┘
                                        │
┌───────────────────────────────────────▼───────────────────────┐
│ 3. SOFTIRQ — net_rx_action() (softirq context)               │
│    Scheduled by ksoftirqd or on return from hardirq           │
│    → Process NAPI poll list for this CPU                      │
│    → For each scheduled NAPI:                                 │
│      → napi_poll(napi, weight=64)                             │
│        → Call driver's poll function                          │
└───────────────────────────────────────┬───────────────────────┘
                                        │
┌───────────────────────────────────────▼───────────────────────┐
│ 4. DRIVER — NAPI Poll (softirq context)                       │
│    my_poll(napi, budget)                                      │
│    while (work_done < budget):                                │
│      → Read RX descriptor (check DD/done bit)                │
│      → dma_sync_single_for_cpu() (cache coherency)           │
│      → skb_put(skb, length)                                  │
│      → skb->protocol = eth_type_trans(skb, dev)              │
│        → parse dst MAC → set pkt_type                         │
│        → set skb->protocol = EtherType                       │
│        → skb_pull(ETH_HLEN)                                  │
│      → Set skb->ip_summed (checksum status from HW)          │
│      → napi_gro_receive(napi, skb) → GRO aggregation         │
│      → Refill RX descriptor with new buffer                  │
│    if (work_done < budget):                                   │
│      → napi_complete_done(napi, work_done)                   │
│      → Re-enable NIC interrupts                              │
│    return work_done                                           │
└───────────────────────────────────────┬───────────────────────┘
                                        │
┌───────────────────────────────────────▼───────────────────────┐
│ 5. GRO — Generic Receive Offload      net/core/gro.c         │
│    napi_gro_receive(napi, skb)                                │
│    → Try to merge skb with existing GRO flow                  │
│      → Same flow (src/dst IP+port): merge payload             │
│      → Create super-skb with combined payload                 │
│    → If cannot merge or flow complete:                        │
│      → napi_gro_complete() → netif_receive_skb()             │
│    → Benefit: 44 × 1460-byte packets → 1 × 64KB packet       │
│      → TCP processes once instead of 44 times                 │
└───────────────────────────────────────┬───────────────────────┘
                                        │
┌───────────────────────────────────────▼───────────────────────┐
│ 6. DEVICE LAYER                        net/core/dev.c         │
│    netif_receive_skb(skb)                                     │
│    → __netif_receive_skb()                                    │
│      → Packet tap: deliver to AF_PACKET sockets (tcpdump)    │
│      → XDP program (if attached): XDP_PASS/DROP/TX/REDIRECT  │
│      → Ingress TC qdisc (if configured)                      │
│      → Protocol type dispatch:                                │
│        → ptype_base hash lookup by skb->protocol              │
│          ETH_P_IP  (0x0800) → ip_rcv()                       │
│          ETH_P_ARP (0x0806) → arp_rcv()                      │
│          ETH_P_IPV6(0x86DD) → ipv6_rcv()                     │
└───────────────────────────────────────┬───────────────────────┘
                                        │
┌───────────────────────────────────────▼───────────────────────┐
│ 7. NETWORK LAYER                      net/ipv4/ip_input.c     │
│    ip_rcv(skb, dev, ptype) → ip_rcv_core()                    │
│    → Validate: version=4, IHL>=5, length, checksum            │
│    → NF_INET_PRE_ROUTING (netfilter hook)                     │
│    → ip_rcv_finish() → routing decision                       │
│      → dst matches local IP?                                  │
│        YES → ip_local_deliver()                               │
│               → ip_defrag() (reassemble fragments)            │
│               → NF_INET_LOCAL_IN                              │
│               → ip_local_deliver_finish()                     │
│                 → Protocol dispatch by IP protocol field:     │
│                   6  → tcp_v4_rcv()                           │
│                   17 → udp_rcv()                              │
│                   1  → icmp_rcv()                             │
│        NO  → ip_forward()                                     │
│               → TTL--, NF_INET_FORWARD, ip_output()           │
└───────────────────────────────────────┬───────────────────────┘
                                        │
┌───────────────────────────────────────▼───────────────────────┐
│ 8. TRANSPORT LAYER                    net/ipv4/tcp_ipv4.c     │
│    tcp_v4_rcv(skb)                                            │
│    → __inet_lookup_skb(): find socket by 5-tuple              │
│      → Established hash table (fast path)                     │
│      → Listen hash table (for SYN on new connections)         │
│    → tcp_v4_do_rcv(sk, skb)                                  │
│      → ESTABLISHED: tcp_rcv_established() (fast path)         │
│        → tcp_data_queue(sk, skb)                              │
│          → Enqueue to sk->sk_receive_queue                    │
│          → Update TCP window, send ACK if needed              │
│        → sk->sk_data_ready(sk)                                │
│          → Wake up sleeping reader                            │
│      → Other states: tcp_rcv_state_process()                  │
└───────────────────────────────────────┬───────────────────────┘
                                        │
┌───────────────────────────────────────▼───────────────────────┐
│ 9. SOCKET LAYER → USER SPACE                                  │
│    recv(fd, buf, len, 0)                                      │
│    → sys_recvfrom() → sock->ops->recvmsg()                   │
│    → tcp_recvmsg(sk, msg):                                    │
│      → Lock socket                                            │
│      → Dequeue skb from sk_receive_queue                      │
│      → skb_copy_datagram_msg(): copy data to user buffer      │
│      → Update sequence tracking                               │
│      → Free consumed sk_buff                                  │
│      → Return bytes received                                  │
└───────────────────────────────────────────────────────────────┘
```

---

## 14.2 GRO Deep Dive

```
Problem: At 10Gbps, ~800K packets/sec at 1500 bytes each.
Each packet: NAPI poll + netif_receive_skb + ip_rcv + tcp_v4_rcv
= millions of function calls per second.

GRO solution: Merge same-flow packets before protocol processing.

Before GRO (44 individual packets):
  [TCP seq=0,   len=1460] → ip_rcv → tcp_v4_rcv
  [TCP seq=1460,len=1460] → ip_rcv → tcp_v4_rcv
  [TCP seq=2920,len=1460] → ip_rcv → tcp_v4_rcv
  ... × 44 times

After GRO (1 merged packet):
  [TCP seq=0, len=64240]  → ip_rcv → tcp_v4_rcv  (once!)

GRO merge criteria:
  - Same source/destination IP
  - Same source/destination port
  - Same protocol flags (no FIN, RST, SYN)
  - Sequential TCP sequence numbers
  - Same MAC addresses
  - Checksum correct

Flush triggers:
  - Different flow arrives
  - Non-mergeable flags (PSH, FIN)
  - GRO timeout (held count)
  - NAPI poll completes
```

### GRO vs LRO

```
GRO (Generic Receive Offload):
  - Software-based, in NAPI poll
  - Protocol-aware: preserves all headers
  - Correct for all cases (forwarding, bridging)
  - Default and recommended

LRO (Large Receive Offload):
  - Hardware-based (NIC merges)
  - Modifies headers (recalculates)
  - Incorrect for forwarding (breaks checksum)
  - Deprecated — use GRO instead
```

---

## 14.3 Packet Type Dispatch

```
After netif_receive_skb():

ptype_all list (packet taps):
  → tcpdump (AF_PACKET) sees every packet
  → Network monitoring tools

ptype_base[hash] (protocol dispatch):
  Protocol handlers registered via dev_add_pack():

  struct packet_type ip_packet_type = {
      .type = cpu_to_be16(ETH_P_IP),
      .func = ip_rcv,
  };
  dev_add_pack(&ip_packet_type);  /* called in inet_init() */

  Dispatch:
    skb->protocol == ETH_P_IP   → ip_rcv()
    skb->protocol == ETH_P_ARP  → arp_rcv()
    skb->protocol == ETH_P_IPV6 → ipv6_rcv()
    skb->protocol == ETH_P_8021Q→ vlan_skb_recv()
```

---

## 14.4 Socket Lookup (5-Tuple Hash)

```
TCP socket lookup in tcp_v4_rcv():

__inet_lookup_skb(skb):
  1. Established hash table (inet_ehash_bucket):
     Key: (src IP, src port, dst IP, dst port)
     → O(1) hash lookup
     → Finds ESTABLISHED or TIME_WAIT sockets
     → Fast path for active connections

  2. Listen hash table (inet_listen_hashbucket):
     Key: (dst port)
     → Finds LISTEN sockets for new connections (SYN)
     → Checks for SO_REUSEPORT distribution

  Hash table sizes:
    /proc/sys/net/ipv4/tcp_max_tw_buckets  (TIME_WAIT entries)
    Established: auto-sized based on memory (tcp_hashinfo)
```

---

## 14.5 Packet Flow Control

```
If application reads slowly:

  sk->sk_receive_queue fills up
    → sk_rcvbuf limit reached
    → TCP advertises smaller window
    → Eventually: window = 0 (zero window)
    → Sender stops sending
    → Persist timer probes periodically
    → When application reads: window reopens

If packets arrive faster than kernel processes:

  NAPI poll budget exhausted (64 packets per poll)
    → Remaining packets stay in NIC ring
    → NIC ring fills
    → NIC drops incoming packets
    → RX overrun counter increments
    
  Fix: increase ring size (ethtool -G eth0 rx 4096)
       tune NAPI budget (net.core.netdev_budget = 300)
       use RSS to spread across CPUs
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/core/dev.c | netif_receive_skb(), __netif_receive_skb() |
| net/core/gro.c | GRO aggregation |
| net/ipv4/ip_input.c | ip_rcv(), ip_local_deliver() |
| net/ipv4/tcp_ipv4.c | tcp_v4_rcv() |
| net/ipv4/udp.c | udp_rcv() |
| net/core/flow_dissector.c | Flow classification for GRO/RSS |
| include/linux/netdevice.h | NAPI structures |

---

## Interview Questions

**Q1: Trace an incoming TCP packet from NIC to application.**
A: NIC DMA → interrupt → NAPI schedule → poll() → read descriptor → build skb → eth_type_trans() → napi_gro_receive() → GRO merge → netif_receive_skb() → ptype dispatch (ETH_P_IP) → ip_rcv() → validate → PRE_ROUTING hook → route lookup → ip_local_deliver() → LOCAL_IN hook → tcp_v4_rcv() → socket lookup (5-tuple hash) → tcp_rcv_established() → tcp_data_queue() → sk_receive_queue → sk_data_ready() → wake reader → recv() copies to user.

**Q2: What is GRO and why is it important?**
A: GRO (Generic Receive Offload) merges multiple same-flow packets into one large packet before protocol processing. At 10Gbps, this reduces per-packet processing from 800K packets/sec to ~12K super-packets/sec. GRO checks: same flow (IPs, ports), sequential TCP sequences, same flags. Unlike LRO (hardware), GRO is protocol-aware and safe for forwarding. Enabled by default; critical for high-throughput networking.

**Q3: How does the kernel find the socket for an incoming TCP packet?**
A: __inet_lookup_skb() performs a two-stage lookup. First: established hash table (inet_ehash_bucket) using (src IP, src port, dst IP, dst port) as key — O(1) hash lookup for active connections. Second: if not found, listen hash table using (dst port) to find a LISTEN socket for new connections (SYN packets). SO_REUSEPORT distributes across multiple listen sockets using a hash of the source 4-tuple.

**Q4: What happens when the RX ring fills up?**
A: If NAPI poll can't process packets fast enough (or budget exhausted), the NIC's RX descriptor ring fills. New frames have no descriptors to write to — the NIC drops them silently. Visible via: `ethtool -S eth0 | grep rx_dropped` or `rx_no_buffer_count`. Fix: increase ring size (`ethtool -G eth0 rx 4096`), enable RSS/RPS to distribute across CPUs, increase NAPI budget, or use XDP for early processing.

---

## Summary

- RX path: NIC → hardirq → NAPI softirq → GRO → netif_receive_skb → IP → TCP → socket
- GRO merges same-flow packets: 44 small packets become 1 large — major CPU savings
- Socket lookup uses O(1) hash on 5-tuple for established connections
- Back-pressure: socket buffer full → zero window → sender stops → no drops
- NIC ring overflow → hardware drops → tune ring size, RSS, NAPI budget

---

Next: [Chapter 15 — Interrupt Handling and NAPI](Chapter_15_NAPI_Interrupts.md)
