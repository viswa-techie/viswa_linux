# Chapter 1: Foundations and History of Computer Networking

## Learning Goals
- Understand what computer networking is and why it exists
- Know the fundamental communication models and topologies
- Trace the evolution from ARPANET to modern Linux networking
- Define key networking metrics and terminology

---

## 1.1 What Is Computer Networking

Computer networking is the practice of connecting two or more computing devices to exchange data. A network enables resource sharing, communication, and distributed computation.

```
┌──────────┐                         ┌──────────┐
│  Host A  │◄── Physical/Logical ──►│  Host B  │
│ (sender) │       Link              │(receiver)│
└──────────┘                         └──────────┘

Data flows as discrete units called PACKETS.
Each packet carries: source address, destination address, payload.
```

### Why Networking Exists in Operating Systems

An OS manages hardware resources. Networking extends this to remote resources:

1. **Resource sharing** — printers, storage, compute
2. **Communication** — email, messaging, RPC
3. **Distributed systems** — databases, clusters, cloud
4. **Fault tolerance** — redundancy across machines

---

## 1.2 Communication Models

### Client-Server

```
┌────────┐   request    ┌────────┐
│ Client │─────────────►│ Server │
│        │◄─────────────│        │
└────────┘   response   └────────┘

- Server listens on well-known port (e.g., 80, 443)
- Client initiates connection
- Server handles multiple clients concurrently
- Examples: HTTP, DNS, SSH, NFS
```

### Peer-to-Peer (P2P)

```
┌────────┐◄────────────►┌────────┐
│ Peer A │              │ Peer B │
└────────┘              └────────┘
     ▲                       ▲
     └───────────────────────┘
              Peer C

- Every node is both client and server
- No central authority
- Examples: BitTorrent, blockchain networks
- Scalability advantage: more peers = more capacity
```

---

## 1.3 Network Topologies

```
BUS:                    STAR:                   RING:
───┬───┬───┬───         ┌─┐                    ┌─┐
   │   │   │          ┌─┤H├─┐                ┌─┤A├─┐
  [A] [B] [C]        [A]└─┘[B]              [D]└─┘[B]
                       │   │                  └──┤C├──┘
                      [C] [D]                    └─┘

MESH (Full):            TREE:
┌─┐───┌─┐               [Root]
│A│   │B│               /     \
└─┘\X/└─┘           [Sw1]   [Sw2]
┌─┐/X\┌─┐           / \       \
│C│   │D│         [A] [B]     [C]
└─┘───└─┘
```

| Topology | Fault Tolerance | Cost | Use Case |
|----------|----------------|------|----------|
| Bus | Low — single cable failure breaks all | Low | Legacy Ethernet (10BASE2) |
| Star | Medium — hub/switch is single point | Medium | Modern LANs (switched Ethernet) |
| Ring | Medium — break isolates segment | Medium | Token Ring, FDDI, some industrial |
| Mesh | High — multiple paths | High | Data centers, backbone |
| Tree | Medium — hierarchical | Medium | Enterprise campus networks |

---

## 1.4 Network Performance Metrics

| Metric | Definition | Units | Key Factors |
|--------|-----------|-------|-------------|
| Bandwidth | Maximum data rate of a link | bits/sec (Mbps, Gbps) | Physical medium, encoding |
| Throughput | Actual data rate achieved | bits/sec | Congestion, protocol overhead |
| Latency | Time for packet to travel src→dst | milliseconds | Distance, hops, processing |
| Jitter | Variation in latency | milliseconds | Queue depth, load variation |
| Packet loss | Fraction of packets not delivered | percentage | Congestion, errors, drops |
| RTT | Round-trip time (send + reply) | milliseconds | 2× one-way latency + processing |

```
Bandwidth-Delay Product (BDP):
  BDP = Bandwidth × RTT

  Example: 1 Gbps link, 10ms RTT
  BDP = 1,000,000,000 × 0.010 = 10,000,000 bits = 1.25 MB

  Meaning: 1.25 MB of data can be "in flight" at any time.
  TCP window must be >= BDP for full utilization.
```

---

## 1.5 History and Evolution

### Timeline

```
1969  ARPANET — 4 nodes (UCLA, SRI, UCSB, Utah)
1973  TCP/IP development begins (Cerf, Kahn)
1974  First TCP specification (RFC 675)
1981  IPv4 specification (RFC 791)
1983  ARPANET switches to TCP/IP (Jan 1, "Flag Day")
1985  BSD 4.2 — first widely-used TCP/IP socket API
1991  Linux 0.01 — no networking
1992  Linux 0.96 — first networking code (NET-1 by Ross Biro)
1993  Linux 1.0 — TCP/IP stack (Alan Cox rewrites: NET-2/NET-3)
1994  Linux 1.2 — stable networking, IPX, AppleTalk
1996  Linux 2.0 — SMP support, netfilter precursors
1999  Linux 2.2 — netfilter/iptables (Rusty Russell)
2001  Linux 2.4 — NAPI (New API for interrupt mitigation)
2003  Linux 2.6 — epoll, full preemption, advanced routing
2005  Linux 2.6.14 — Generic Receive Offload (GRO)
2010  Linux 2.6.35 — Receive Packet Steering (RPS)
2011  Linux 3.0 — Multiqueue support improvements
2014  Linux 3.19 — eBPF networking hooks
2016  Linux 4.8 — XDP (eXpress Data Path)
2018  Linux 4.19 — AF_XDP, CAKE qdisc
2019  Linux 5.1 — io_uring (async I/O for networking)
2020+ — Continued eBPF/XDP expansion, TSN support
```

### Linux Networking Milestones

| Version | Contribution | Impact |
|---------|-------------|--------|
| 0.96 | NET-1 stack | First networking in Linux |
| 1.0 | NET-3 (Alan Cox) | Production-quality TCP/IP |
| 2.2 | Netfilter | Replaced ipchains; modular filtering |
| 2.4 | NAPI | Solved interrupt storms at high packet rates |
| 2.6 | epoll, splice | Scalable event-driven servers |
| 3.x | Multiqueue NIC | Per-CPU TX/RX for multi-core scaling |
| 4.x | eBPF, XDP | Programmable dataplane at near-wire speed |
| 5.x | AF_XDP, io_uring | Zero-copy, async networking |

---

## 1.6 Networking Terminology

| Term | Definition |
|------|-----------|
| Packet | Unit of data at network layer (IP packet) |
| Frame | Unit of data at link layer (Ethernet frame) |
| Segment | Unit of data at transport layer (TCP segment) |
| Datagram | Connectionless unit (UDP datagram, IP datagram) |
| MTU | Maximum Transmission Unit — largest frame payload (typically 1500 bytes for Ethernet) |
| MSS | Maximum Segment Size — largest TCP payload = MTU - IP header - TCP header |
| TTL | Time To Live — hop counter, decremented at each router |
| Socket | Endpoint for communication identified by (IP, port, protocol) |
| Port | 16-bit number identifying a service (0-65535) |
| Protocol | Set of rules governing communication (TCP, UDP, IP, ARP) |
| Interface | Logical or physical network attachment point (eth0, wlan0, lo) |
| Gateway | Router that connects two networks |
| Subnet | Logical subdivision of an IP network |
| CIDR | Classless Inter-Domain Routing — notation: 192.168.1.0/24 |
| ARP | Address Resolution Protocol — maps IP to MAC |
| MAC | Media Access Control — 48-bit hardware address (aa:bb:cc:dd:ee:ff) |

---

## 1.7 Packet Encapsulation Concept

```
Layer 4 (Transport):
┌─────────────────────────────────────────┐
│ TCP Header │        Payload             │
└─────────────────────────────────────────┘
         ↓ encapsulate

Layer 3 (Network):
┌──────────────────────────────────────────────────┐
│ IP Header │ TCP Header │        Payload           │
└──────────────────────────────────────────────────┘
         ↓ encapsulate

Layer 2 (Link):
┌────────────────────────────────────────────────────────────┐
│ ETH Header │ IP Header │ TCP Header │ Payload │ ETH FCS   │
└────────────────────────────────────────────────────────────┘
  14 bytes     20 bytes    20 bytes     data       4 bytes

Total overhead for TCP/IP over Ethernet: 54 bytes minimum
For 1500 MTU: max payload = 1500 - 20(IP) - 20(TCP) = 1460 bytes (MSS)
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/ | Top-level networking directory |
| net/core/ | Core networking code |
| net/ipv4/ | IPv4 protocol implementation |
| net/ipv6/ | IPv6 protocol implementation |
| drivers/net/ | Network device drivers |
| include/net/ | Networking header files |
| include/linux/netdevice.h | struct net_device definition |
| include/linux/skbuff.h | struct sk_buff definition |

---

## Interview Questions

**Q1: What is the difference between a packet, frame, segment, and datagram?**
A: These are PDUs (Protocol Data Units) at different layers. Frame = Layer 2 (Ethernet), Packet = Layer 3 (IP), Segment = Layer 4 connection-oriented (TCP), Datagram = Layer 4 connectionless (UDP) or Layer 3 (IP datagram).

**Q2: What is MTU and how does it affect networking?**
A: MTU is the maximum payload a link-layer frame can carry. Standard Ethernet MTU is 1500 bytes. If an IP packet exceeds the path MTU, it must be fragmented (IPv4) or the sender is notified via ICMP "Packet Too Big" (IPv6, and IPv4 with DF bit set). Jumbo frames use MTU of 9000 bytes for high-throughput applications.

**Q3: What is bandwidth-delay product and why does it matter for TCP?**
A: BDP = bandwidth x RTT. It represents the maximum amount of data that can be in-flight. TCP's receive window must be at least BDP to fully utilize the link. If window < BDP, the sender stalls waiting for ACKs before the pipe is full.

**Q4: What was NAPI and why was it introduced?**
A: NAPI (New API) was introduced in Linux 2.4 to solve interrupt storms. At high packet rates, per-packet interrupts consume 100% CPU. NAPI switches to polling mode after the first interrupt: the ISR disables further interrupts and schedules a poll function that processes multiple packets per invocation. When the queue is empty, interrupts are re-enabled.

---

## Summary

- Networking connects machines to share resources, communicate, and distribute computation
- Client-server and peer-to-peer are fundamental communication models
- Key metrics: bandwidth, throughput, latency, jitter, packet loss, RTT
- Linux networking evolved from zero (0.01) to a world-class stack in 30 years
- Packet encapsulation adds headers at each layer; total overhead matters for performance
- Understanding the history helps explain why the code is structured as it is

---

Next: [Chapter 2 — Networking Models: OSI and TCP/IP](Chapter_02_Networking_Models.md)
