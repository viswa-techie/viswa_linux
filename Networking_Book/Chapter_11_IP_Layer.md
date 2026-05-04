# Chapter 11: Internet Layer — IPv4 and IPv6

## Learning Goals
- Understand IPv4 header fields and their purpose
- Know IP routing internals: FIB, routing lookup
- Understand IP fragmentation and reassembly
- Know IPv6 differences and dual-stack operation
- Understand ICMP role and messages

---

## 11.1 IPv4 Header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |Type of Service|          Total Length         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |         Header Checksum       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (if IHL > 5)                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Key fields:
  Version     = 4 (IPv4)
  IHL         = Header length in 32-bit words (5 = 20 bytes, no options)
  Total Length = Entire packet including header (max 65535)
  TTL         = Decremented at each router, discarded at 0
  Protocol    = Transport: 6=TCP, 17=UDP, 1=ICMP
  Flags       = DF (Don't Fragment), MF (More Fragments)
  Frag Offset = Fragment position (in 8-byte units)
```

---

## 11.2 IP Processing in Linux Kernel

### TX Path

```
ip_queue_xmit(sk, skb)              [net/ipv4/ip_output.c]
  │
  ├── Route lookup: ip_route_output_flow()
  │   └── fib_lookup() → find output device, gateway
  │
  ├── Build IP header:
  │   iph->version  = 4
  │   iph->ihl      = 5
  │   iph->tos      = inet->tos
  │   iph->tot_len  = skb->len
  │   iph->id       = ip_idents[hash]++
  │   iph->ttl      = inet->ttl (or net.ipv4.ip_default_ttl = 64)
  │   iph->protocol = sk->sk_protocol (6=TCP, 17=UDP)
  │   iph->saddr    = source IP
  │   iph->daddr    = destination IP
  │
  ├── ip_output(sk, skb)
  │   └── NF_INET_POST_ROUTING hook
  │
  └── ip_finish_output(skb)
      ├── Check MTU: if skb->len > mtu → ip_fragment()
      └── neigh_output() → add L2 header → dev_queue_xmit()
```

### RX Path

```
netif_receive_skb() → ptype_dispatch → ETH_P_IP → ip_rcv()
  │                                               [net/ipv4/ip_input.c]
  ├── Validate:
  │   - version == 4
  │   - IHL >= 5
  │   - total_length >= IHL×4
  │   - Checksum valid
  │
  ├── NF_INET_PRE_ROUTING hook (netfilter)
  │
  ├── ip_rcv_finish() → routing decision
  │   │
  │   ├── Local delivery? (dst IP matches local)
  │   │   └── ip_local_deliver()
  │   │       ├── Reassemble fragments: ip_defrag()
  │   │       ├── NF_INET_LOCAL_IN hook
  │   │       └── Protocol dispatch:
  │   │           protocol 6  → tcp_v4_rcv()
  │   │           protocol 17 → udp_rcv()
  │   │           protocol 1  → icmp_rcv()
  │   │
  │   └── Forward? (ip_forwarding enabled)
  │       └── ip_forward()
  │           ├── TTL-- (discard if 0, send ICMP Time Exceeded)
  │           ├── NF_INET_FORWARD hook
  │           └── ip_output() → next hop
  │
  └── Neither? → drop (ICMP Destination Unreachable)
```

---

## 11.3 IP Routing

```
Routing Decision:
  For every packet, kernel asks: "Where do I send this?"
  Answer comes from the FIB (Forwarding Information Base).

  ┌────────────────────────────────────────────────────────┐
  │ FIB (Forwarding Information Base)                     │
  │                                                        │
  │ Destination        Gateway        Interface   Metric  │
  │ 192.168.1.0/24     0.0.0.0        eth0        100    │
  │ 10.0.0.0/8         192.168.1.1    eth0        200    │
  │ 0.0.0.0/0          192.168.1.1    eth0        600    │ ← default route
  └────────────────────────────────────────────────────────┘

  FIB uses LC-Trie (Level-Compressed Trie) for fast lookup.
  Source: net/ipv4/fib_trie.c
```

### Routing Lookup Flow

```
fib_lookup(net, flowi4, &result)
    │
    ├── Check routing rules (RPDB: ip rule list)
    │   Rule 0: from all lookup local     (local table)
    │   Rule 32766: from all lookup main  (main table)
    │   Rule 32767: from all lookup default
    │
    ├── For each matching rule:
    │   └── fib_table_lookup() on the specified table
    │       └── LC-Trie longest-prefix match
    │
    └── Result contains:
        - Output device (net_device)
        - Gateway address (next hop)
        - Type (RTN_LOCAL, RTN_UNICAST, RTN_BROADCAST)
        - MTU
```

### Routing Commands

```bash
# View routing table
ip route show
# default via 192.168.1.1 dev eth0
# 192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.100

# Add route
ip route add 10.0.0.0/8 via 192.168.1.1 dev eth0

# Add default gateway
ip route add default via 192.168.1.1

# Policy routing
ip rule add from 10.0.0.0/8 table 100
ip route add default via 10.0.0.1 table 100

# View routing cache / FIB
ip route get 8.8.8.8  # Show route for specific destination
```

---

## 11.4 IP Fragmentation and Reassembly

```
Fragmentation (ip_fragment):

  Original packet: 4000 bytes payload, MTU=1500

  Fragment 1:  IP hdr (20) + data (1480) = 1500 bytes
               MF=1, offset=0, id=12345

  Fragment 2:  IP hdr (20) + data (1480) = 1500 bytes
               MF=1, offset=185 (1480/8), id=12345

  Fragment 3:  IP hdr (20) + data (1040) = 1060 bytes
               MF=0, offset=370 (2960/8), id=12345

  offset is in 8-byte units → max offset = 8191 × 8 = 65528

Reassembly (ip_defrag):
  Fragments queued in ipq (IP queue) by (src, dst, id, protocol)
  When all fragments arrive: reconstruct original packet
  Timeout: 30 seconds (net.ipv4.ipfrag_time)
  If timeout expires: fragments discarded, ICMP Time Exceeded

Path MTU Discovery (PMTUD):
  Set DF (Don't Fragment) flag on all packets
  If router can't forward: drops + sends ICMP "Fragmentation Needed"
  Sender reduces packet size to reported MTU
  net.ipv4.ip_no_pmtu_disc = 0 (PMTUD enabled)
```

---

## 11.5 IPv6

```
IPv6 Header (40 bytes, fixed):
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version| Traffic Class |           Flow Label                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Payload Length        |  Next Header  |   Hop Limit  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                              |
+                         Source Address                       +
|                         (128 bits)                           |
+                                                              +
|                                                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                              |
+                      Destination Address                     +
|                         (128 bits)                           |
+                                                              +
|                                                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### IPv4 vs IPv6

| Feature | IPv4 | IPv6 |
|---------|------|------|
| Address size | 32 bits (4.3 billion) | 128 bits (3.4×10^38) |
| Header | Variable (20-60 bytes) | Fixed 40 bytes |
| Checksum | Header checksum | None (relies on L2/L4) |
| Fragmentation | Routers can fragment | Only source fragments |
| ARP | ARP protocol | NDP (ICMPv6) |
| Broadcast | Yes | No (multicast replaces) |
| Auto-config | DHCP | SLAAC (stateless) + DHCPv6 |
| IPsec | Optional | Mandatory (in spec) |
| NAT | Common | Not needed (enough addresses) |

### IPv6 in Linux Kernel

```
net/ipv6/                     ← IPv6 protocol implementation
  ip6_input.c                 ← IPv6 RX processing
  ip6_output.c                ← IPv6 TX processing
  route.c                     ← IPv6 routing
  ndisc.c                     ← Neighbor Discovery Protocol
  addrconf.c                  ← Address configuration (SLAAC)

Dual stack: Linux supports IPv4 and IPv6 simultaneously
  IPv6 socket can accept IPv4 connections (::ffff:1.2.3.4)
  net.ipv6.bindv6only = 0 (dual-stack by default)
```

---

## 11.6 ICMP

```
ICMP (Internet Control Message Protocol):
  - Error reporting and diagnostics
  - Carried inside IP packets (protocol = 1)
  - NOT a transport protocol — no ports

Common ICMP Types:
┌──────┬──────────────────────────┬──────────────────────────────┐
│ Type │ Name                     │ Use                          │
├──────┼──────────────────────────┼──────────────────────────────┤
│  0   │ Echo Reply               │ ping response                │
│  3   │ Destination Unreachable  │ No route, port closed, etc.  │
│  3/4 │ Frag Needed + DF Set     │ PMTUD (need smaller packets) │
│  5   │ Redirect                 │ Better route available       │
│  8   │ Echo Request             │ ping                         │
│  11  │ Time Exceeded            │ TTL=0 (traceroute)          │
└──────┴──────────────────────────┴──────────────────────────────┘

  Kernel: net/ipv4/icmp.c
  Rate limited: net.ipv4.icmp_ratelimit (default 1000ms)
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/ipv4/ip_input.c | ip_rcv(), ip_local_deliver() |
| net/ipv4/ip_output.c | ip_queue_xmit(), ip_output() |
| net/ipv4/ip_forward.c | ip_forward() |
| net/ipv4/ip_fragment.c | Fragmentation |
| net/ipv4/ip_options.c | IP options processing |
| net/ipv4/route.c | Routing cache |
| net/ipv4/fib_trie.c | FIB trie lookup |
| net/ipv4/fib_rules.c | Policy routing rules |
| net/ipv4/icmp.c | ICMP protocol |
| net/ipv6/ | Full IPv6 implementation |
| include/net/ip.h | IP structures |

---

## Interview Questions

**Q1: What happens to an IP packet when TTL reaches zero?**
A: The router that decrements TTL to zero drops the packet and sends an ICMP Time Exceeded (type 11, code 0) message back to the source. This is how traceroute works: it sends packets with incrementing TTL (1, 2, 3...) and collects ICMP responses to discover each hop. Linux default TTL is 64 (net.ipv4.ip_default_ttl).

**Q2: Explain IP routing lookup in the Linux kernel.**
A: ip_route_output_flow() calls fib_lookup() which consults the RPDB (policy routing rules). Each rule points to a FIB table. The table uses an LC-Trie (Level-Compressed Trie) for longest-prefix match on the destination address. The result provides: output net_device, gateway IP (next hop), route type (local/unicast/broadcast), and PMTU. This lookup happens for every locally-originated packet.

**Q3: What is Path MTU Discovery and why is it important?**
A: PMTUD discovers the smallest MTU along the path to avoid fragmentation. The sender sets the DF (Don't Fragment) flag. If a router can't forward because the packet exceeds its link MTU, it drops the packet and sends ICMP "Fragmentation Needed" with the MTU. The sender then reduces packet size. This avoids fragmentation overhead and the reliability problems of fragment loss (losing one fragment loses the entire datagram).

**Q4: What are the main differences between IPv4 and IPv6 from a kernel perspective?**
A: IPv6 has a fixed 40-byte header (no header checksum — reduces per-hop processing). No router fragmentation — only the source fragments, using Path MTU Discovery exclusively. ARP is replaced by NDP (Neighbor Discovery Protocol, ICMPv6). Neighbor Solicitation/Advertisement replaces ARP Request/Reply. Stateless Address Autoconfiguration (SLAAC) eliminates DHCP dependency. Separate source file tree in kernel: net/ipv6/.

**Q5: Why does Linux process ICMP in the kernel rather than user space?**
A: ICMP is critical for network operation: (1) PMTUD depends on ICMP "Fragmentation Needed" to update routing MTU. (2) ICMP error messages modify TCP/UDP behavior (e.g., port unreachable closes UDP sockets). (3) Low-latency ping response. (4) ICMP redirect updates routing table. These must be processed immediately by the kernel, not queued for a user-space process.

---

## Summary

- IPv4 header: 20 bytes minimum, contains TTL, protocol, addresses, checksum
- Routing uses FIB with LC-Trie for longest-prefix match
- Fragmentation splits large packets; PMTUD avoids it by discovering path MTU
- IPv6 simplifies: fixed header, no router fragmentation, NDP replaces ARP
- ICMP provides error reporting and diagnostics (essential for PMTUD, traceroute)
- Every packet triggers a routing lookup to determine output device and gateway

---

Next: [Chapter 12 — Link Layer: Ethernet, ARP, VLAN](Chapter_12_Link_Layer.md)
