# Chapter 7: sk_buff and Buffer Management

## Learning Goals
- Master the sk_buff structure — the most important networking data structure
- Understand push, pull, put, and reserve operations
- Know how sk_buff traverses all layers without copying
- Understand cloning, copying, and linear vs paged data
- Know scatter-gather and fragment management

---

## 7.1 sk_buff Overview

`struct sk_buff` (socket buffer) is THE central data structure in Linux networking. Every packet at every layer is an sk_buff.

```
struct sk_buff
┌──────────────────────────────────────────────────────────────┐
│ Packet Pointers                                              │
│   head     → Start of allocated buffer                       │
│   data     → Start of current data (moves with push/pull)   │
│   tail     → End of current data (moves with put)            │
│   end      → End of allocated buffer                         │
│                                                              │
│ Layer Information                                            │
│   mac_header      → Offset to L2 (Ethernet) header          │
│   network_header  → Offset to L3 (IP) header                │
│   transport_header→ Offset to L4 (TCP/UDP) header            │
│                                                              │
│ Metadata                                                     │
│   dev            → net_device this packet is associated with │
│   protocol       → EtherType (ETH_P_IP, ETH_P_ARP)          │
│   pkt_type       → PACKET_HOST, PACKET_BROADCAST, etc.      │
│   ip_summed      → Checksum status                          │
│   len            → Total data length (linear + paged)        │
│   data_len       → Length of paged data only                 │
│   cb[48]         → Control buffer (per-layer private data)   │
│   tstamp         → Timestamp                                 │
│   sk             → Owning socket                             │
│   dst            → Routing entry                             │
│   mark           → Netfilter mark                            │
│   priority       → QoS priority (TC)                         │
│   queue_mapping  → TX queue index                            │
└──────────────────────────────────────────────────────────────┘
```

---

## 7.2 Buffer Layout

```
After allocation (netdev_alloc_skb_ip_align):

head                                                    end
  ↓                                                      ↓
  ┌───────────────────────────────────────────────────────┐
  │  headroom  │         (empty space)        │ tailroom  │
  └───────────────────────────────────────────────────────┘
               ↑                              ↑
              data                           tail

  headroom = reserved space for headers (NET_SKB_PAD + alignment)
  data == tail (no data yet)

After driver fills RX data (skb_put):

head                                                    end
  ↓                                                      ↓
  ┌───────────────────────────────────────────────────────┐
  │ headroom │  ETH│IP│TCP│ Payload data      │ tailroom │
  └───────────────────────────────────────────────────────┘
             ↑                                ↑
            data                             tail
             └──────── len = tail - data ─────┘
```

---

## 7.3 Buffer Manipulation Operations

### skb_put — Extend Data at Tail

```
Before skb_put(skb, 100):
  data              tail                        end
   ↓                 ↓                           ↓
   ├─── existing ────┤──── available space ──────┤

After skb_put(skb, 100):
  data                              tail        end
   ↓                                 ↓           ↓
   ├───── existing ──┤── 100 bytes ──┤── space ──┤

Used by: RX driver to set received data length
  skb_put(skb, received_length);
```

### skb_push — Prepend Data at Head

```
Before skb_push(skb, 14):
             data                    tail
              ↓                       ↓
  ┌─ free ───┤──── data ─────────────┤──┐
  headroom                               end

After skb_push(skb, 14):
       data                          tail
        ↓                             ↓
  ┌───┤──14──┤──── data ─────────────┤──┐
      └ Ethernet header written here

Used by: eth_header() to add Ethernet header before TX
  skb_push(skb, ETH_HLEN);
```

### skb_pull — Strip Data from Head

```
Before skb_pull(skb, 14):
  data                               tail
   ↓                                  ↓
   ├──ETH──┤──IP──┤──TCP──┤──Payload─┤

After skb_pull(skb, 14):
            data                      tail
             ↓                         ↓
   ├──ETH──┤──IP──┤──TCP──┤──Payload──┤
   (ETH header still in buffer but "invisible")

Used by: eth_type_trans() to skip Ethernet header on RX
  skb_pull(skb, ETH_HLEN);
```

### skb_reserve — Create Headroom

```
Before skb_reserve(skb, NET_SKB_PAD + NET_IP_ALIGN):
  head/data/tail                              end
       ↓                                      ↓
       ├──────────── empty buffer ─────────────┤

After skb_reserve(skb, 66):
  head                                        end
   ↓                                           ↓
   ├──── 66 bytes headroom ──┤                 │
                              ↑
                           data/tail

Used by: Driver at allocation time to reserve space for headers
  skb = netdev_alloc_skb(dev, BUF_SIZE);
  skb_reserve(skb, NET_SKB_PAD + NET_IP_ALIGN);
```

---

## 7.4 Layer Header Tracking

```c
/* On RX, as packet moves up the stack: */

/* Driver / eth_type_trans: */
skb_reset_mac_header(skb);     /* mac_header = data offset */
skb_pull(skb, ETH_HLEN);      /* Strip Ethernet header */

/* IP layer: */
skb_reset_network_header(skb); /* network_header = data offset */
                                /* data points to IP header */

/* TCP/UDP layer: */
skb_set_transport_header(skb, ip_hdr_len);
                                /* transport_header offset set */

/* Accessing headers: */
struct ethhdr  *eth = eth_hdr(skb);       /* → mac_header */
struct iphdr   *iph = ip_hdr(skb);        /* → network_header */
struct tcphdr  *th  = tcp_hdr(skb);       /* → transport_header */
struct udphdr  *uh  = udp_hdr(skb);       /* → transport_header */
```

```
Full packet in memory:

mac_header  network_header  transport_header
    ↓            ↓              ↓
    ┌──ETH──┬──IP───┬──TCP──┬──Payload──┐
    │14 bytes│20 bytes│20 bytes│  data    │
    └────────┴───────┴───────┴──────────┘
    ↑                                    ↑
   data (after eth_type_trans,          tail
         data points to IP header)
```

---

## 7.5 sk_buff Allocation

```c
/* For RX buffers (driver) */
skb = netdev_alloc_skb_ip_align(dev, RX_BUF_SIZE);
/*   alloc_skb(size + NET_SKB_PAD + NET_IP_ALIGN)
     skb_reserve(skb, NET_SKB_PAD + NET_IP_ALIGN)
     NET_IP_ALIGN = 2 bytes → aligns IP header to 4-byte boundary */

/* For protocol layer TX */
skb = alloc_skb(size, GFP_KERNEL);

/* With specific headroom */
skb = alloc_skb(total_size, GFP_ATOMIC);
skb_reserve(skb, header_room);

/* sk_buff from socket send path */
skb = sock_alloc_send_skb(sk, size, nonblock, &err);

/* Free */
kfree_skb(skb);          /* Drop (traced by kfree_skb tracepoint) */
consume_skb(skb);         /* Normal consumption (not traced as drop) */
dev_kfree_skb_any(skb);  /* For drivers (any context) */
```

### Allocation Sizes

```
Typical RX buffer layout:

┌──────────────────────────────────────────────────┐
│ NET_SKB_PAD │ NET_IP_ALIGN │    RX Buffer        │
│  (64 bytes) │  (2 bytes)   │   (1536+ bytes)     │
└──────────────────────────────────────────────────┘

NET_SKB_PAD   = space for adding headers later (XDP, encapsulation)
NET_IP_ALIGN  = 2 bytes to align IP header on 4-byte boundary
1536          = common RX buffer (1500 MTU + Ethernet + padding)
```

---

## 7.6 Cloning and Copying

### skb_clone — Share Data

```
Original skb:                Clone skb:
┌────────────┐              ┌────────────┐
│ sk_buff A  │              │ sk_buff B  │  ← new sk_buff metadata
│ (metadata) │              │ (metadata) │
│ data ──────┼──────┐       │ data ──────┼──┐
└────────────┘      │       └────────────┘  │
                    ▼                       ▼
               ┌──────────────────┐
               │  SHARED data     │  ← refcount incremented
               │  (packet bytes)  │     NOT copied
               └──────────────────┘

- Clone creates new sk_buff pointing to SAME data
- Data is shared (read-only)
- Metadata changes are independent
- Used by: multicast delivery, packet sniffing
- Fast: no data copy
```

```c
struct sk_buff *clone = skb_clone(skb, GFP_ATOMIC);
/* clone and skb share the same data buffer */
/* Modify clone metadata independently */
/* CANNOT modify shared data — copy-on-write not automatic */
```

### skb_copy — Full Copy

```c
struct sk_buff *copy = skb_copy(skb, GFP_ATOMIC);
/* copy has its own data buffer — completely independent */
/* Used when you need to modify packet data */
```

### pskb_copy — Copy Header, Share Pages

```c
struct sk_buff *copy = pskb_copy(skb, GFP_ATOMIC);
/* Copies linear data (headers), shares paged data (payload) */
/* Used when only headers need modification */
```

---

## 7.7 Linear vs Paged Data (Fragments)

```
Linear sk_buff (simple case):
┌──────────────────────────────────────────┐
│ sk_buff metadata                         │
│ head → [headroom][ETH|IP|TCP|payload]    │
│ len = 1500, data_len = 0                 │
└──────────────────────────────────────────┘

Paged sk_buff (scatter-gather):
┌──────────────────────────────────────────┐
│ sk_buff metadata                         │
│ head → [headroom][ETH|IP|TCP|partial]    │  ← linear part
│ len = 65536, data_len = 64000            │
│                                          │
│ frags[0] → page A (4096 bytes)           │  ← page fragments
│ frags[1] → page B (4096 bytes)           │
│ frags[2] → page C (4096 bytes)           │
│ ...                                      │
│ frags[N] → page N                        │
└──────────────────────────────────────────┘

Linear data = len - data_len (headers + some payload)
Paged data  = data_len       (bulk payload in pages)

Used for: TSO (64KB TCP segments), sendfile()/splice()
```

### Fragment API

```c
/* Check if skb has fragments */
int has_frags = skb_is_nonlinear(skb);

/* Number of fragments */
int nr_frags = skb_shinfo(skb)->nr_frags;

/* Access a fragment */
skb_frag_t *frag = &skb_shinfo(skb)->frags[i];
struct page *page = skb_frag_page(frag);
unsigned int offset = skb_frag_off(frag);
unsigned int size = skb_frag_size(frag);

/* Linearize (copy all fragments into linear buffer) */
int err = skb_linearize(skb);
/* Expensive! Avoid if possible */

/* Add fragment */
skb_fill_page_desc(skb, frag_idx, page, offset, size);
```

---

## 7.8 sk_buff Control Buffer (cb)

```c
/* cb[] is a 48-byte per-skb scratch area for each layer */

/* TCP uses it for: */
struct tcp_skb_cb {
    __u32 seq;          /* Starting sequence number */
    __u32 end_seq;      /* SEQ + FIN + SYN + datalen */
    __u8  tcp_flags;    /* TCP flags */
    __u32 ack_seq;      /* ACK sequence number */
    /* ... */
};
#define TCP_SKB_CB(__skb) ((struct tcp_skb_cb *)&((__skb)->cb[0]))

/* IP uses it for: */
struct inet_skb_parm {
    int iif;            /* Input interface index */
    __u8 flags;         /* IP flags */
    /* ... */
};
#define IPCB(skb) ((struct inet_skb_parm *)((skb)->cb))

/* Each layer can reuse cb[] — it's not preserved across layers */
```

---

## 7.9 Checksum Handling

```
skb->ip_summed values:

CHECKSUM_NONE:
  No checksum computed. Software must verify.
  Used when: HW cannot verify, or checksum is invalid.

CHECKSUM_UNNECESSARY:
  Hardware verified checksum — it's correct.
  Software can skip verification.
  Used when: NIC reports good checksum (most modern NICs).

CHECKSUM_COMPLETE:
  Hardware provides raw checksum value in skb->csum.
  Software uses it to verify (pseudo-header math).
  Used when: NIC provides checksum but software must finalize.

CHECKSUM_PARTIAL:
  Software computed partial checksum; hardware must finish.
  skb->csum_start, skb->csum_offset tell HW where to write.
  Used for TX when using hardware checksum offload.
```

```c
/* RX: Driver sets checksum status */
if (hw_csum_valid) {
    skb->ip_summed = CHECKSUM_UNNECESSARY;
} else {
    skb->ip_summed = CHECKSUM_NONE;
}

/* TX: Stack marks for HW offload */
skb->ip_summed = CHECKSUM_PARTIAL;
skb->csum_start = skb_headroom(skb) + offset_to_transport;
skb->csum_offset = offsetof(struct tcphdr, check);
```

---

## 7.10 sk_buff Lifecycle

```
TX Lifecycle:
  1. alloc_skb() or sock_alloc_send_skb()
  2. Application data copied into skb (tcp_sendmsg)
  3. TCP adds header: skb_push(skb, tcp_header_size)
  4. IP adds header: skb_push(skb, ip_header_size)
  5. Ethernet adds header: skb_push(skb, ETH_HLEN)
  6. dev_queue_xmit() → qdisc → driver
  7. DMA map skb data
  8. NIC transmits
  9. TX completion: dma_unmap, dev_consume_skb_any()

RX Lifecycle:
  1. netdev_alloc_skb_ip_align() + skb_reserve()
  2. NIC DMA writes packet data
  3. skb_put(skb, received_length)
  4. eth_type_trans(): set protocol, skb_pull ETH header
  5. napi_gro_receive() → netif_receive_skb()
  6. ip_rcv(): validate, strip to transport
  7. tcp_v4_rcv(): process, queue to socket
  8. User recv(): copy to user buffer
  9. kfree_skb() when consumed
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| include/linux/skbuff.h | sk_buff definition and inline ops |
| net/core/skbuff.c | sk_buff allocation, cloning, manipulation |
| include/linux/netdevice.h | netdev_alloc_skb and friends |
| net/core/datagram.c | Datagram helper functions |

---

## Interview Questions

**Q1: Explain the four pointers in sk_buff: head, data, tail, end.**
A: head points to the start of the allocated buffer (never changes). data points to the start of current data (moves with push/pull). tail points to the end of current data (moves with put). end points to the end of allocated buffer (never changes). headroom = data - head (space for prepending headers). tailroom = end - tail (space for appending data). len = tail - data.

**Q2: What is the difference between skb_push and skb_pull?**
A: skb_push(skb, len) moves data backward by len bytes, prepending space — used to add headers (Ethernet, IP) during TX. skb_pull(skb, len) moves data forward by len bytes, stripping data — used to remove headers during RX. Push decreases headroom; pull increases it.

**Q3: How does sk_buff avoid data copies as a packet traverses layers?**
A: sk_buff uses a single data buffer with movable pointers. Instead of copying data between layers, each layer adjusts the data pointer to add (push) or remove (pull) its header. The buffer is allocated with enough headroom for all headers. The data is written once and pointers are simply adjusted at each layer — zero-copy within the stack.

**Q4: What is the difference between skb_clone and skb_copy?**
A: skb_clone creates a new sk_buff metadata structure but shares the data buffer (refcount incremented). Metadata changes are independent but data modifications are not safe. Used for multicast where multiple paths need the same packet. skb_copy creates a completely independent copy — new sk_buff AND new data buffer. Used when data modification is needed.

**Q5: What is CHECKSUM_UNNECESSARY and when is it set?**
A: CHECKSUM_UNNECESSARY means the hardware (NIC) has verified the packet's checksum and confirmed it's correct. Software can skip checksum verification. Set by the driver when the NIC reports good checksum status for received packets. This offloads CPU work — important at high packet rates.

**Q6: Explain linear vs paged sk_buff data.**
A: Linear data lives in the contiguous buffer between head and end — accessed directly. Paged data lives in page fragments referenced by skb_shinfo(skb)->frags[]. Total length is skb->len; paged portion is skb->data_len; linear portion is skb->len - skb->data_len. Paged data enables TSO (64KB super-segments), sendfile() zero-copy, and scatter-gather DMA without linearizing.

---

## Summary

- sk_buff is the universal packet representation — every layer uses it
- Four pointers (head, data, tail, end) enable zero-copy header manipulation
- push/pull/put/reserve adjust pointers without moving data
- Layer headers tracked by mac_header, network_header, transport_header offsets
- Cloning shares data (fast); copying duplicates everything (safe)
- Paged fragments enable large packets without contiguous allocation
- Hardware checksum offload avoids software verification

---

Next: [Chapter 8 — Socket Layer](Chapter_08_Socket_Layer.md)
