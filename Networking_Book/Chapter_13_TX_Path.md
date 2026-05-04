# Chapter 13: Packet Transmission Path

## Learning Goals
- Trace every step from send() to wire with kernel functions
- Understand GSO/TSO segmentation
- Know qdisc interaction during transmission
- Understand scatter-gather DMA for TX

---

## 13.1 Complete TX Path

```
┌─────────────────────────────── USER SPACE ────────────────────────┐
│ send(fd, buf, len, 0)                                             │
│   → glibc: syscall(__NR_sendto, fd, buf, len, 0, NULL, 0)        │
└───────────────────────────────────┬───────────────────────────────┘
                                    │ syscall
┌───────────────────────────────────▼───────────────────────────────┐
│ 1. SYSCALL LAYER                  net/socket.c                    │
│    __sys_sendto()                                                 │
│    → import_single_range()        copy iov from user              │
│    → sockfd_lookup_light(fd)      get struct socket from fd       │
│    → sock_sendmsg(sock, msg)                                     │
│      → sock->ops->sendmsg()      dispatch to protocol family     │
│        → inet_sendmsg()                                          │
└───────────────────────────────────┬───────────────────────────────┘
                                    │
┌───────────────────────────────────▼───────────────────────────────┐
│ 2. TRANSPORT LAYER                net/ipv4/tcp.c (or udp.c)      │
│    tcp_sendmsg(sk, msg, size)                                     │
│    → Copy user data to sk_write_queue                             │
│    → Segment into MSS-sized sk_buffs                              │
│    → tcp_push() → tcp_write_xmit()                               │
│      → tcp_transmit_skb(sk, skb)                                 │
│        → Build TCP header (seq, ack, window, flags)               │
│        → Compute checksum or mark CHECKSUM_PARTIAL                │
│        → icsk->icsk_af_ops->queue_xmit(sk, skb)                  │
│          → ip_queue_xmit()                                        │
└───────────────────────────────────┬───────────────────────────────┘
                                    │
┌───────────────────────────────────▼───────────────────────────────┐
│ 3. NETWORK LAYER                  net/ipv4/ip_output.c            │
│    ip_queue_xmit(sk, skb)                                         │
│    → ip_route_output_flow()       routing table lookup             │
│    → skb_dst_set(skb, dst)        attach route to skb             │
│    → Build IP header (saddr, daddr, ttl, protocol, id)            │
│    → NF_INET_LOCAL_OUT            netfilter hook                   │
│    → dst_output(skb)              → ip_output()                   │
│    → NF_INET_POST_ROUTING         netfilter hook                   │
│    → ip_finish_output(skb)                                        │
│      → Check MTU: if too large → ip_fragment()                    │
│      → ip_finish_output2()                                        │
│        → neigh_output()           ARP resolution + L2 header      │
└───────────────────────────────────┬───────────────────────────────┘
                                    │
┌───────────────────────────────────▼───────────────────────────────┐
│ 4. NEIGHBOR / L2 LAYER            net/core/neighbour.c            │
│    neigh_output(neigh, skb)                                       │
│    → neigh_resolve_output() or neigh_connected_output()           │
│    → dev_queue_xmit(skb)          hand off to device layer        │
└───────────────────────────────────┬───────────────────────────────┘
                                    │
┌───────────────────────────────────▼───────────────────────────────┐
│ 5. TRAFFIC CONTROL                net/core/dev.c, net/sched/      │
│    dev_queue_xmit(skb)                                            │
│    → __dev_queue_xmit()                                           │
│      → GSO: validate_xmit_skb()  check for needed segmentation   │
│      → Select TX queue: netdev_pick_tx()                          │
│      → Enqueue to qdisc: q->enqueue(skb, q)                      │
│      → __qdisc_run(q)            dequeue and transmit             │
│        → sch_direct_xmit()                                        │
│          → dev_hard_start_xmit()                                  │
└───────────────────────────────────┬───────────────────────────────┘
                                    │
┌───────────────────────────────────▼───────────────────────────────┐
│ 6. DRIVER LAYER                   drivers/net/ethernet/            │
│    dev_hard_start_xmit(skb, dev)                                  │
│    → dev->netdev_ops->ndo_start_xmit(skb, dev)                   │
│      → DMA map skb data and fragments                             │
│      → Fill TX descriptor(s)                                      │
│      → Memory barrier (wmb)                                       │
│      → Write tail pointer (ring doorbell)                         │
└───────────────────────────────────┬───────────────────────────────┘
                                    │
┌───────────────────────────────────▼───────────────────────────────┐
│ 7. HARDWARE                                                       │
│    NIC reads TX descriptor via DMA                                │
│    NIC fetches packet data via DMA                                │
│    NIC computes CRC if offloaded                                  │
│    NIC inserts VLAN tag if offloaded                              │
│    MAC transmits frame via PHY                                    │
│    PHY modulates signal on wire                                   │
│    NIC sets descriptor done bit                                   │
│    NIC generates TX completion interrupt                          │
└───────────────────────────────────────────────────────────────────┘
```

---

## 13.2 GSO/TSO — Segmentation Offload

```
Without TSO:
  Application sends 64KB:
    tcp_sendmsg → create 44 × 1460-byte sk_buffs
    Each goes through: IP header → qdisc → driver → NIC
    44 packets processed individually ← CPU expensive

With TSO:
  Application sends 64KB:
    tcp_sendmsg → create 1 × 64KB sk_buff (super segment)
    One pass through: IP → qdisc → driver
    NIC splits into 44 segments with correct TCP headers
    44 packets on wire ← NIC handles segmentation

With GSO (Generic Segmentation Offload):
  Like TSO but segmentation done in software just before driver
  Works even if NIC doesn't support TSO
  Delayed segmentation: one large skb through most of the stack
  validate_xmit_skb() → skb_gso_segment() if needed

Flow:
  tcp_sendmsg()
    → single large skb (up to 64KB)
    → ip_queue_xmit() (one pass)
    → netfilter (one pass for all 44 future segments!)
    → dev_queue_xmit()
      → validate_xmit_skb()
        → if NIC supports TSO: pass super-skb to driver
        → if NO TSO: skb_gso_segment() → create 44 skbs → send each
```

### Scatter-Gather for Large Segments

```
64KB TSO segment in memory:

sk_buff:
  head → [TCP header + first data page]     ← linear data
  frags[0] → page 1 (4096 bytes)
  frags[1] → page 2 (4096 bytes)
  frags[2] → page 3 (4096 bytes)
  ...
  frags[15] → page 16

Driver DMA maps:
  DMA map linear data → TX descriptor 0
  DMA map frag 0     → TX descriptor 1
  DMA map frag 1     → TX descriptor 2
  ...
  Last descriptor has EOP (End of Packet) flag

NIC processes all descriptors as one TSO operation:
  Creates 44 individual packets with proper TCP headers
```

---

## 13.3 TX Queue Selection (Multiqueue)

```
For multiqueue NICs with N TX queues:

select_queue(dev, skb):
  1. skb->sk → sk_tx_queue_mapping (cached socket→queue mapping)
  2. If not set: hash(src port, dst port, src IP, dst IP) % num_queues
  3. Or: driver's ndo_select_queue() callback
  4. XPS (Transmit Packet Steering): CPU→queue affinity map

  Goal: same flow always goes to same queue for ordering
        different flows go to different queues for parallelism

  ┌────────────────────────────────────────────────────┐
  │ CPU 0 ─────► TX Queue 0 ─── MSI-X IRQ 0           │
  │ CPU 1 ─────► TX Queue 1 ─── MSI-X IRQ 1           │
  │ CPU 2 ─────► TX Queue 2 ─── MSI-X IRQ 2           │
  │ CPU 3 ─────► TX Queue 3 ─── MSI-X IRQ 3           │
  └────────────────────────────────────────────────────┘

  Configure XPS:
    echo 1 > /sys/class/net/eth0/queues/tx-0/xps_cpus  # CPU 0
    echo 2 > /sys/class/net/eth0/queues/tx-1/xps_cpus  # CPU 1
```

---

## 13.4 TX Completion

```
After NIC transmits:
  NIC sets done bit in TX descriptor
  NIC generates interrupt (MSI-X for the queue's CPU)

  ISR → napi_schedule()
  NAPI poll:
    clean_tx_ring():
      while (desc->status & DONE):
        dma_unmap_single()           free DMA mapping
        dev_consume_skb_any(skb)     free sk_buff
        stats.tx_packets++
        stats.tx_bytes += length
        advance tail pointer
      
      if queue was stopped && space available:
        netif_wake_queue(dev)
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/socket.c | __sys_sendto() |
| net/ipv4/tcp.c | tcp_sendmsg() |
| net/ipv4/tcp_output.c | tcp_write_xmit(), tcp_transmit_skb() |
| net/ipv4/ip_output.c | ip_queue_xmit(), ip_output(), ip_fragment() |
| net/core/neighbour.c | neigh_output() |
| net/core/dev.c | dev_queue_xmit(), dev_hard_start_xmit() |
| net/sched/sch_generic.c | qdisc_run() |
| net/core/skbuff.c | skb_gso_segment() |

---

## Interview Questions

**Q1: Trace a TCP packet from send() to the wire.**
A: send() → sys_sendto() → sock_sendmsg() → inet_sendmsg() → tcp_sendmsg() (copy data, segment) → tcp_write_xmit() → tcp_transmit_skb() (build TCP header) → ip_queue_xmit() (route lookup, build IP header) → netfilter LOCAL_OUT → ip_output() → netfilter POST_ROUTING → ip_finish_output() → neigh_output() (ARP resolve, add ETH header) → dev_queue_xmit() → qdisc → dev_hard_start_xmit() → ndo_start_xmit() → DMA map → NIC transmits.

**Q2: What is the difference between TSO and GSO?**
A: TSO (TCP Segmentation Offload) lets the NIC hardware split a large TCP segment (up to 64KB) into MTU-sized packets. GSO (Generic Segmentation Offload) is the software equivalent: segmentation is delayed until just before the driver. If the NIC supports TSO, GSO passes the super-segment through. If not, GSO performs segmentation in software at the last moment. GSO's benefit: the large segment traverses the stack as one packet, reducing per-packet processing overhead.

**Q3: How does multiqueue TX work?**
A: Multiqueue NICs have N independent TX queues, each with its own descriptor ring and MSI-X interrupt. For each packet, netdev_pick_tx() selects a queue based on flow hash or XPS (CPU→queue) mapping. Same flow → same queue (preserves order). Different flows → different queues (parallelism). Each queue can be processed by a different CPU, eliminating lock contention. XPS pins TX queues to CPUs for cache locality.

---

## Summary

- TX path traverses 7 layers: syscall → transport → network → neighbor → TC → driver → HW
- GSO/TSO defer segmentation, reducing per-packet overhead by 40-60×
- Multiqueue TX with XPS enables per-CPU parallel transmission
- TX completion: driver cleans descriptors, frees DMA mappings, wakes stopped queues
- Every layer adds headers: TCP → IP → Ethernet → NIC adds FCS

---

Next: [Chapter 14 — Packet Reception Path](Chapter_14_RX_Path.md)
