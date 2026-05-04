# Chapter 4: Linux Networking Architecture Overview

## Learning Goals
- Understand the Linux networking subsystem as a whole
- Trace the packet processing pipeline from NIC to application
- Know the key kernel subsystems involved in networking
- Understand the interaction between layers

---

## 4.1 Linux Networking Subsystem Overview

Linux networking is one of the most sophisticated subsystems in the kernel. It implements the full TCP/IP stack plus extensive additional features.

```
┌─────────────────────────────────────────────────────────────┐
│                      USER SPACE                             │
│                                                             │
│   Applications: nginx, curl, iperf3, tcpdump               │
│   Libraries:    glibc (socket(), send(), recv())            │
│   Tools:        ip, tc, iptables, ethtool, ss, netstat      │
├─────────────────────────────────────────────────────────────┤
│                   SYSTEM CALL INTERFACE                      │
│   sys_socket, sys_bind, sys_listen, sys_accept,             │
│   sys_connect, sys_sendto, sys_recvfrom, sys_setsockopt     │
├──────────────────────┬──────────────────────────────────────┤
│   SOCKET LAYER       │  net/socket.c                        │
│   struct socket      │  Protocol family dispatch            │
│   struct sock        │  AF_INET, AF_INET6, AF_UNIX, AF_PACKET│
├──────────────────────┴──────────────────────────────────────┤
│   TRANSPORT LAYER      net/ipv4/tcp.c, udp.c               │
│   TCP: connection, reliability, flow control, congestion    │
│   UDP: connectionless, best-effort                          │
├─────────────────────────────────────────────────────────────┤
│   NETWORK LAYER        net/ipv4/ip_input.c, ip_output.c    │
│   IPv4/IPv6: addressing, routing, fragmentation            │
│   ICMP: error reporting, diagnostics                        │
├─────────────────────────────────────────────────────────────┤
│   NETFILTER            net/netfilter/                       │
│   Hooks at 5 points: PRE_ROUTING, INPUT, FORWARD,          │
│   OUTPUT, POST_ROUTING                                      │
├─────────────────────────────────────────────────────────────┤
│   TRAFFIC CONTROL      net/sched/                           │
│   qdiscs: pfifo_fast, fq_codel, htb, tbf                   │
│   Packet scheduling and shaping                             │
├─────────────────────────────────────────────────────────────┤
│   NETWORK DEVICE LAYER net/core/dev.c                       │
│   struct net_device    dev_queue_xmit(), netif_receive_skb()│
│   Generic device operations                                 │
├─────────────────────────────────────────────────────────────┤
│   DRIVER LAYER         drivers/net/                         │
│   ndo_start_xmit(), napi_poll()                             │
│   Hardware-specific: DMA, descriptor rings, interrupts      │
├─────────────────────────────────────────────────────────────┤
│   HARDWARE             NIC, PHY, Switch, Cable              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4.2 Key Subsystem Interactions

```
                    SEND PATH                     RECEIVE PATH
                    ─────────                     ────────────
User:          send(fd, buf, len)            recv(fd, buf, len)
                    │                              ↑
                    ▼                              │
Socket:    sock->ops->sendmsg()          sock_queue_rcv_skb()
                    │                              ↑
                    ▼                              │
Transport: tcp_sendmsg()                 tcp_v4_rcv()
           tcp_write_xmit()                    │
                    │                              ↑
                    ▼                              │
Network:   ip_queue_xmit()              ip_rcv() → ip_local_deliver()
           ip_output()                         │
                    │                              ↑
                    ▼                              │
Netfilter: NF_INET_LOCAL_OUT            NF_INET_LOCAL_IN
           NF_INET_POST_ROUTING         NF_INET_PRE_ROUTING
                    │                              ↑
                    ▼                              │
TC:        qdisc_run()                  ingress qdisc
                    │                              ↑
                    ▼                              │
Device:    dev_queue_xmit()             netif_receive_skb()
           dev_hard_start_xmit()        GRO processing
                    │                              ↑
                    ▼                              │
Driver:    ndo_start_xmit()             napi_poll()
           DMA map + ring update         DMA unmap + sk_buff create
                    │                              ↑
                    ▼                              │
Hardware:  NIC transmits frame           NIC receives frame + DMA
```

---

## 4.3 Packet Processing Pipeline

### TX Pipeline (Detailed)

```
1. Application calls send(fd, "Hello", 5, 0)
   │
2. Syscall → __sys_sendto() [net/socket.c]
   │
3. sock->ops->sendmsg() → inet_sendmsg()
   │
4. sk->sk_prot->sendmsg() → tcp_sendmsg()
   │  - Copy data to send buffer (sk_write_queue)
   │  - Segment into MSS-sized chunks
   │
5. tcp_write_xmit() → tcp_transmit_skb()
   │  - Build TCP header (seq, ack, window, flags)
   │  - Compute TCP checksum (or mark for HW offload)
   │
6. ip_queue_xmit() [net/ipv4/ip_output.c]
   │  - Route lookup (fib_lookup)
   │  - Build IP header (src/dst IP, TTL, protocol)
   │  - Set output device from route
   │
7. Netfilter NF_INET_LOCAL_OUT hook
   │  - iptables OUTPUT chain processing
   │
8. ip_output() → ip_finish_output()
   │  - Check MTU, fragment if needed (ip_fragment)
   │  - Netfilter NF_INET_POST_ROUTING hook
   │
9. Neighbor lookup (ARP) → neigh_output()
   │  - Resolve destination MAC address
   │  - Add Ethernet header
   │
10. dev_queue_xmit() [net/core/dev.c]
    │  - Enqueue to qdisc (traffic control)
    │  - qdisc_run() → dequeue and transmit
    │
11. dev_hard_start_xmit()
    │  - Call driver: ndo_start_xmit()
    │
12. Driver: fill TX descriptor, DMA map, ring doorbell
    │
13. NIC: DMA fetch, add FCS, transmit on wire
```

### RX Pipeline (Detailed)

```
1. NIC receives frame on wire
   │
2. NIC: DMA write to pre-allocated buffer
   │  - Update RX descriptor (length, status, checksum)
   │  - Trigger interrupt (or coalesced)
   │
3. Driver ISR: napi_schedule() → schedule NAPI poll
   │
4. NAPI poll function:
   │  - Read RX descriptors
   │  - Build sk_buff from DMA buffer
   │  - napi_gro_receive(napi, skb) → GRO aggregation
   │
5. netif_receive_skb() [net/core/dev.c]
   │  - Packet type dispatch (ETH_P_IP, ETH_P_ARP)
   │  - Ingress TC qdisc processing
   │  - XDP program execution (if attached)
   │
6. ip_rcv() [net/ipv4/ip_input.c]
   │  - Validate IP header (version, checksum, length)
   │  - Netfilter NF_INET_PRE_ROUTING hook
   │
7. ip_rcv_finish() → routing decision
   │  - Is this for us? → ip_local_deliver()
   │  - Forward? → ip_forward()
   │
8. ip_local_deliver()
   │  - Reassemble fragments if needed
   │  - Netfilter NF_INET_LOCAL_IN hook
   │
9. Transport dispatch (based on IP protocol field)
   │  - Protocol 6 → tcp_v4_rcv()
   │  - Protocol 17 → udp_rcv()
   │
10. tcp_v4_rcv() [net/ipv4/tcp_ipv4.c]
    │  - Find matching socket (5-tuple lookup)
    │  - Process TCP state machine
    │  - Queue data to socket receive buffer
    │
11. Wake up process blocked on recv()
    │
12. Application reads data via recv(fd, buf, len, 0)
```

---

## 4.4 Key Data Structures

```
struct sk_buff (skb):
  - THE central networking data structure
  - Represents one packet at all layers
  - Contains packet data + metadata (device, protocol, timestamps)
  - Manipulated at every layer: push/pull headers

struct net_device:
  - Represents a network interface (eth0, wlan0, lo)
  - Contains operations (ndo_start_xmit, etc.)
  - Contains queues, statistics, features

struct socket:
  - User-facing socket object (has file descriptor)
  - Points to struct sock (protocol-specific state)

struct sock (sk):
  - Protocol-level socket state
  - Contains send/receive buffers, TCP state, timers

struct dst_entry:
  - Routing cache entry
  - Contains output device, gateway, MTU
  - Attached to sk_buff for output path

struct neighbour:
  - ARP/NDP cache entry
  - Maps IP address to MAC address
```

---

## 4.5 Networking Subsystem Initialization

```
Kernel boot sequence for networking:

1. core_initcall:
   - sock_init()           → /proc/net, socket filesystem
   - netdev_init()         → per-CPU packet queues

2. subsys_initcall:
   - net_dev_init()        → softirq registration (NET_TX, NET_RX)
   - inet_init()           → TCP, UDP, ICMP, IGMP protocol registration
   - arp_init()            → ARP protocol

3. fs_initcall:
   - Various /proc/net entries

4. device_initcall:
   - NIC driver probes (PCI enumeration)
   - lo (loopback) device registration

5. late_initcall:
   - Routing cache, netfilter modules

Order matters: protocols register before devices probe.
```

---

## 4.6 Networking and Softirqs

Network processing heavily uses softirqs (deferred interrupt processing):

```
Two networking softirqs:

NET_TX_SOFTIRQ:
  - Triggered when TX completions need processing
  - Runs: net_tx_action()
  - Frees transmitted sk_buffs
  - Restarts devices with pending TX

NET_RX_SOFTIRQ:
  - Triggered by NAPI schedule (from ISR)
  - Runs: net_rx_action()
  - Calls NAPI poll functions
  - Processes up to netdev_budget packets (default 300)
  - Processes for up to 2 jiffies

Softirq context:
  - Runs with interrupts enabled
  - Preempts normal process context
  - Cannot sleep (no memory allocation with GFP_KERNEL)
  - Can be re-raised while running
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/socket.c | Socket system calls, protocol family dispatch |
| net/core/dev.c | Network device layer, packet dispatch |
| net/core/skbuff.c | sk_buff operations |
| net/ipv4/af_inet.c | AF_INET socket operations |
| net/ipv4/ip_input.c | IP packet reception |
| net/ipv4/ip_output.c | IP packet transmission |
| net/ipv4/tcp.c | TCP protocol core |
| net/ipv4/udp.c | UDP protocol core |
| net/core/net_namespace.c | Network namespace management |
| net/core/flow_dissector.c | Packet classification |

---

## Interview Questions

**Q1: Trace a packet from send() to the wire — name every kernel function.**
A: send() → __sys_sendto() → sock_sendmsg() → inet_sendmsg() → tcp_sendmsg() → tcp_write_xmit() → tcp_transmit_skb() → ip_queue_xmit() → [netfilter LOCAL_OUT] → ip_output() → ip_finish_output() → [netfilter POST_ROUTING] → neigh_output() → dev_queue_xmit() → qdisc_run() → dev_hard_start_xmit() → ndo_start_xmit() → hardware DMA.

**Q2: What are the two networking softirqs?**
A: NET_TX_SOFTIRQ handles TX completions (freeing transmitted buffers, restarting queues). NET_RX_SOFTIRQ handles packet reception via NAPI (polling NIC for received packets). Both run in softirq context — interrupts enabled, cannot sleep.

**Q3: How does Linux decide whether to deliver or forward a packet?**
A: After ip_rcv() validates the IP header and the PRE_ROUTING netfilter hook runs, ip_rcv_finish() performs a routing lookup. If the destination IP matches a local address, ip_local_deliver() is called. If ip_forwarding is enabled and the destination is remote, ip_forward() handles forwarding. Otherwise, the packet is dropped.

**Q4: What is the relationship between struct socket and struct sock?**
A: struct socket is the user-facing abstraction (has a file descriptor, protocol operations). struct sock (sk) is the protocol-level state (TCP state machine, send/receive buffers, timers, congestion state). socket->sk points to the sock. Multiple layers reference the sock: transport layer for protocol state, network layer for routing cache, device layer for output device.

---

## Summary

- Linux networking is layered: socket → transport → network → netfilter → tc → device → driver
- Every packet is an sk_buff that traverses all layers
- TX path: application → kernel → protocol headers → routing → qdisc → driver → NIC
- RX path: NIC → DMA → NAPI → netfilter → IP → transport → socket → application
- Softirqs (NET_TX, NET_RX) handle deferred packet processing
- Initialization order: protocols first, then devices

---

Next: [Chapter 5 — Network Devices: struct net_device](Chapter_05_Net_Device.md)
