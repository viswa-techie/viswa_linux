# Chapter 29: Kernel Source Code Map, Glossary, and References

## Learning Goals
- Navigate the Linux kernel networking source tree efficiently
- Know key data structure definitions and where to find them
- Have a comprehensive glossary of networking terms
- Know authoritative references and RFCs

---

## 29.1 Kernel Source Tree Map

```
net/                              ← Network subsystem root
├── core/                         ← Core networking
│   ├── dev.c                     ← net_device operations, __netif_receive_skb
│   ├── skbuff.c                  ← sk_buff allocation, manipulation
│   ├── sock.c                    ← Generic socket operations
│   ├── datagram.c                ← Datagram socket helpers
│   ├── stream.c                  ← Stream socket helpers
│   ├── filter.c                  ← Socket filter (BPF)
│   ├── flow_dissector.c          ← Packet flow key extraction
│   ├── neighbour.c               ← Neighbor subsystem (ARP, NDP)
│   ├── rtnetlink.c               ← Route netlink interface
│   ├── net-sysfs.c               ← /sys/class/net/
│   ├── xdp.c                     ← XDP framework core
│   └── drop_monitor.c            ← Packet drop monitoring
│
├── socket.c                      ← Top-level socket syscalls
│
├── ipv4/                         ← IPv4 protocol stack
│   ├── af_inet.c                 ← AF_INET socket family
│   ├── tcp.c                     ← TCP socket operations
│   ├── tcp_input.c               ← TCP receive processing
│   ├── tcp_output.c              ← TCP transmit processing
│   ├── tcp_timer.c               ← TCP timers
│   ├── tcp_ipv4.c                ← TCP/IPv4 glue
│   ├── tcp_cong.c                ← Congestion control framework
│   ├── tcp_cubic.c               ← CUBIC congestion control
│   ├── tcp_bbr.c                 ← BBR congestion control
│   ├── udp.c                     ← UDP protocol
│   ├── raw.c                     ← Raw sockets
│   ├── ip_input.c                ← IP receive (ip_rcv)
│   ├── ip_output.c               ← IP transmit (ip_output)
│   ├── ip_forward.c              ← IP forwarding
│   ├── ip_fragment.c             ← IP fragmentation
│   ├── route.c                   ← Routing cache / lookup
│   ├── fib_trie.c                ← FIB LC-Trie data structure
│   ├── fib_rules.c               ← Policy routing rules
│   ├── fib_semantics.c           ← Route nexthops, metrics
│   ├── arp.c                     ← ARP protocol
│   ├── devinet.c                 ← IPv4 address management
│   ├── icmp.c                    ← ICMP protocol
│   ├── igmp.c                    ← IGMP multicast
│   ├── inet_connection_sock.c    ← Connection-oriented helpers
│   ├── inet_hashtables.c         ← Socket lookup hash tables
│   └── netfilter/                ← IPv4 netfilter modules
│       ├── iptable_filter.c      ← filter table
│       ├── iptable_nat.c         ← nat table
│       ├── iptable_mangle.c      ← mangle table
│       └── nf_reject_ipv4.c      ← REJECT target
│
├── ipv6/                         ← IPv6 protocol stack
│   ├── af_inet6.c                ← AF_INET6 socket family
│   ├── tcp_ipv6.c                ← TCP/IPv6
│   ├── udp.c                     ← UDPv6
│   ├── ip6_input.c               ← IPv6 receive
│   ├── ip6_output.c              ← IPv6 transmit
│   ├── route.c                   ← IPv6 routing
│   ├── ndisc.c                   ← Neighbor Discovery (IPv6 ARP)
│   └── addrconf.c                ← IPv6 address autoconfiguration
│
├── netfilter/                    ← Netfilter framework
│   ├── core.c                    ← Hook registration
│   ├── nf_conntrack_core.c       ← Connection tracking
│   ├── nf_nat_core.c             ← NAT core
│   ├── nf_tables_api.c           ← nftables API
│   └── nf_tables_core.c          ← nftables evaluation
│
├── sched/                        ← Traffic control (TC)
│   ├── sch_generic.c             ← Qdisc framework
│   ├── sch_htb.c                 ← HTB
│   ├── sch_fq_codel.c            ← fq_codel (default)
│   ├── sch_tbf.c                 ← Token Bucket Filter
│   ├── sch_prio.c                ← Priority scheduler
│   ├── sch_netem.c               ← Network emulator
│   ├── sch_taprio.c              ← TSN Time-Aware shaper
│   ├── cls_u32.c                 ← u32 classifier
│   ├── cls_flower.c              ← flower classifier
│   └── cls_bpf.c                 ← BPF classifier
│
├── bridge/                       ← Ethernet bridging
│   ├── br_device.c               ← Bridge net_device
│   ├── br_input.c                ← Bridge input processing
│   ├── br_forward.c              ← Bridge forwarding
│   └── br_fdb.c                  ← Forwarding database
│
├── can/                          ← CAN bus
│   ├── af_can.c                  ← AF_CAN socket family
│   ├── raw.c                     ← CAN_RAW protocol
│   └── isotp.c                   ← ISO-TP transport
│
├── xdp/                          ← XDP / AF_XDP
│   └── xdp_umem.c               ← UMEM for AF_XDP
│
├── packet/                       ← AF_PACKET (raw)
│   └── af_packet.c               ← Raw packet sockets
│
├── netlink/                      ← Netlink IPC
│   ├── af_netlink.c              ← AF_NETLINK family
│   └── genetlink.c               ← Generic netlink
│
├── 8021q/                        ← VLAN (802.1Q)
│   └── vlan_dev.c                ← VLAN net_device
│
└── ethernet/                     ← Ethernet protocol
    └── eth.c                     ← eth_type_trans, eth_header

include/                          ← Header files
├── linux/
│   ├── skbuff.h                  ← struct sk_buff
│   ├── netdevice.h               ← struct net_device
│   ├── socket.h                  ← struct socket
│   ├── net.h                     ← Core networking types
│   ├── tcp.h                     ← TCP structures
│   ├── ip.h                      ← IP header
│   ├── netfilter.h               ← Netfilter hooks
│   └── bpf.h                     ← BPF types
├── net/
│   ├── sock.h                    ← struct sock
│   ├── tcp_states.h              ← TCP state enum
│   ├── inet_sock.h               ← struct inet_sock
│   ├── dst.h                     ← struct dst_entry
│   ├── neighbour.h               ← struct neighbour
│   ├── xdp.h                     ← XDP definitions
│   └── genetlink.h               ← Generic netlink API
└── uapi/linux/
    ├── tcp.h                     ← TCP user API
    ├── ip.h                      ← IP user API
    ├── if.h                      ← Interface flags
    ├── rtnetlink.h               ← Route netlink constants
    └── can.h                     ← CAN frame structures

drivers/net/                      ← Network drivers
├── ethernet/
│   ├── intel/                    ← Intel NICs (e1000e, igb, ixgbe, ice)
│   ├── broadcom/                 ← Broadcom (bnxt, tg3)
│   ├── mellanox/                 ← Mellanox/NVIDIA (mlx5)
│   └── realtek/                  ← Realtek (r8169)
├── virtio_net.c                  ← Virtio network driver (VMs)
├── veth.c                        ← Virtual ethernet pair
├── tun.c                         ← TUN/TAP
├── bonding/                      ← Bond driver
├── macvlan.c                     ← MACVLAN
├── ipvlan/                       ← IPVLAN
└── vxlan/                        ← VXLAN overlay driver
```

---

## 29.2 Key Data Structure Quick Reference

```
struct sk_buff:        include/linux/skbuff.h     ~250 fields
struct net_device:     include/linux/netdevice.h   ~200 fields
struct net_device_ops: include/linux/netdevice.h   ~80 callbacks
struct socket:         include/linux/net.h         user-facing socket
struct sock:           include/net/sock.h          network-layer socket
struct inet_sock:      include/net/inet_sock.h     IPv4 socket
struct tcp_sock:       include/linux/tcp.h         TCP socket (extends inet_conn_sock)
struct dst_entry:      include/net/dst.h           route cache entry
struct fib_info:       include/net/ip_fib.h        route information
struct neighbour:      include/net/neighbour.h     ARP/NDP entry
struct net:            include/net/net_namespace.h  network namespace
struct napi_struct:    include/linux/netdevice.h   NAPI polling
struct nf_hook_ops:    include/linux/netfilter.h   netfilter hook
struct Qdisc:          include/net/sch_generic.h   queueing discipline
struct ethhdr:         include/uapi/linux/if_ether.h Ethernet header
struct iphdr:          include/uapi/linux/ip.h     IPv4 header
struct tcphdr:         include/uapi/linux/tcp.h    TCP header
struct udphdr:         include/uapi/linux/udp.h    UDP header
```

---

## 29.3 Glossary

```
ACK:        Acknowledgment. TCP flag confirming receipt of data.
ARP:        Address Resolution Protocol. Maps IP → MAC address.
ASIC:       Application-Specific Integrated Circuit. Hardware pipeline.
BBR:        Bottleneck Bandwidth and RTT. Google's congestion control.
BDP:        Bandwidth-Delay Product. Optimal amount of in-flight data.
BPF:        Berkeley Packet Filter. Packet filtering bytecode VM.
CBS:        Credit-Based Shaper. AVB/TSN traffic shaping (802.1Qav).
CoDel:      Controlled Delay. AQM algorithm targeting sojourn time.
Conntrack:  Connection Tracking. Stateful packet inspection for NAT/firewall.
CUBIC:      Default Linux TCP congestion control algorithm.
cwnd:       Congestion Window. Sender's allowed unacked data (segments).
DMA:        Direct Memory Access. Hardware copies data without CPU.
DNAT:       Destination NAT. Change destination IP/port.
DPDK:       Data Plane Development Kit. User-space packet processing.
dst_entry:  Destination cache entry. Routing result attached to skb.
eBPF:       Extended BPF. In-kernel virtual machine for programmable hooks.
ECMP:       Equal-Cost Multi-Path. Load balance across multiple routes.
ECN:        Explicit Congestion Notification. Routers mark instead of drop.
FCS:        Frame Check Sequence. Ethernet CRC32 integrity check.
FIB:        Forwarding Information Base. Kernel routing tables.
FQ:         Fair Queuing. Per-flow scheduling for fairness.
GRO:        Generic Receive Offload. Merge received packets.
GSO:        Generic Segmentation Offload. Software TSO fallback.
ICMP:       Internet Control Message Protocol. Error/diagnostic messages.
IOCP:       I/O Completion Ports. Windows async I/O model.
IPI:        Inter-Processor Interrupt. CPU-to-CPU notification.
LC-Trie:    Level-Compressed Trie. FIB lookup data structure.
LRO:        Large Receive Offload. Hardware RX merging (deprecated).
MAC:        Media Access Control. L2 address (6 bytes for Ethernet).
mbuf:       BSD packet buffer structure (chain of small clusters).
MRU:        Maximum Receive Unit. Maximum received frame size.
MSI/MSI-X:  Message Signaled Interrupts. PCI interrupt mechanism.
MSS:        Maximum Segment Size. Largest TCP payload per segment.
MTU:        Maximum Transmission Unit. Largest L3 packet (1500 for Ethernet).
NAPI:       New API. Interrupt mitigation via polling.
NAT:        Network Address Translation. Rewrite IP/port in transit.
NDP:        Neighbor Discovery Protocol. IPv6 equivalent of ARP.
NIC:        Network Interface Card. Hardware network adapter.
PMTUD:      Path MTU Discovery. Find smallest MTU on path.
PMD:        Poll Mode Driver. DPDK driver that busy-polls.
qdisc:      Queueing Discipline. TC packet scheduling algorithm.
RDMA:       Remote Direct Memory Access. Zero-copy remote memory.
RED:        Random Early Detection. Probabilistic queue management.
RFS:        Receive Flow Steering. Direct packets to application CPU.
RoCE:       RDMA over Converged Ethernet.
RPDB:       Routing Policy Database. Policy routing rule list.
RPS:        Receive Packet Steering. Software RSS.
RSS:        Receive Side Scaling. NIC distributes RX across queues.
RTT:        Round-Trip Time. Time for packet + acknowledgment.
rwnd:       Receive Window. Receiver's advertised buffer space.
SACK:       Selective Acknowledgment. Report non-contiguous received data.
SFQ:        Stochastic Fairness Queuing. Hash-based fair scheduling.
sk_buff:    Socket buffer. Linux kernel packet data structure.
SNAT:       Source NAT. Change source IP/port.
softirq:    Software Interrupt. Deferred interrupt processing.
SYN:        Synchronize. TCP connection initiation flag.
TAS:        Time-Aware Shaper. TSN scheduled gates (802.1Qbv).
TBF:        Token Bucket Filter. Rate limiting qdisc.
TC:         Traffic Control. Linux QoS framework.
TSN:        Time-Sensitive Networking. Deterministic Ethernet standards.
TSO:        TCP Segmentation Offload. NIC segments large TCP payloads.
TTL:        Time To Live. IP hop count limit.
VLAN:       Virtual LAN. 802.1Q tagged Ethernet.
VXLAN:      Virtual Extensible LAN. L2 overlay over UDP.
XDP:        eXpress Data Path. eBPF at driver level.
XPS:        Transmit Packet Steering. Map TX queues to CPUs.
```

---

## 29.4 Essential RFCs

```
┌────────────────────────────────────────────────────────────────────┐
│ RFC    │ Title                              │ Relevance           │
├────────┼────────────────────────────────────┼─────────────────────┤
│ 791    │ Internet Protocol (IPv4)           │ IP header format    │
│ 792    │ ICMP                               │ Error messages      │
│ 793    │ Transmission Control Protocol      │ TCP specification   │
│ 826    │ Ethernet ARP                       │ Address resolution  │
│ 768    │ User Datagram Protocol             │ UDP specification   │
│ 1122   │ Requirements for Internet Hosts    │ Implementation guide│
│ 1323   │ TCP Extensions (High Performance)  │ Window scale, TS    │
│ 2018   │ TCP SACK                           │ Selective ACK       │
│ 2460   │ IPv6 Specification                 │ IPv6 header         │
│ 2581   │ TCP Congestion Control             │ Slow start, AIMD    │
│ 5681   │ TCP Congestion Control (update)    │ Current standard    │
│ 6298   │ TCP Retransmission Timer           │ RTO computation     │
│ 7323   │ TCP Extensions (obsoletes 1323)    │ Updated timestamps  │
│ 8200   │ IPv6 Specification (obsoletes 2460)│ Current IPv6        │
│ 8312   │ CUBIC Congestion Control           │ Linux default CC    │
│ 9000   │ BBR Congestion Control             │ Google's BBR        │
│ 9293   │ TCP (obsoletes 793)                │ Current TCP spec    │
└────────┴────────────────────────────────────┴─────────────────────┘
```

---

## 29.5 Recommended Reading

```
Books:
  - "Understanding Linux Network Internals" — Christian Benvenuti (O'Reilly)
  - "Linux Kernel Networking" — Rami Rosen (Apress)
  - "TCP/IP Illustrated, Vol 1 & 2" — W. Richard Stevens
  - "Computer Networking: A Top-Down Approach" — Kurose & Ross
  - "The Linux Programming Interface" — Michael Kerrisk (Ch. 56-61)
  - "Linux Device Drivers, 3rd ed." — Corbet, Rubini, Kroah-Hartman (Ch. 17)

Kernel documentation:
  Documentation/networking/           ← Networking docs in kernel tree
  Documentation/networking/scaling.txt ← RSS, RPS, RFS, XPS
  Documentation/networking/napi.txt   ← NAPI
  Documentation/networking/ip-sysctl.txt ← sysctl parameters
  Documentation/bpf/                  ← BPF/XDP documentation

Online resources:
  - kernel.org/doc/html/latest/networking/
  - blog.cloudflare.com (networking deep dives)
  - lwn.net (Linux Weekly News — kernel development)
  - lartc.org (Linux Advanced Routing & Traffic Control HOWTO)
  - netdevconf.info (Netdev conference talks)
```

---

## Interview Questions

**Q1: Where would you look in the kernel source to debug a TCP retransmission issue?**
A: Start with net/ipv4/tcp_output.c (tcp_retransmit_skb), net/ipv4/tcp_input.c (tcp_ack, tcp_fastretrans_alert for loss detection), net/ipv4/tcp_timer.c (tcp_retransmit_timer for RTO-based retransmits). Tracepoints: tcp:tcp_retransmit_skb, tcp:tcp_probe. The congestion control module (tcp_cubic.c or tcp_bbr.c) determines cwnd adjustment after loss. Key structures: struct tcp_sock in include/linux/tcp.h.

**Q2: How is the kernel networking source organized?**
A: net/ is the root: net/core/ has generic infrastructure (dev.c for net_device, skbuff.c for sk_buff, sock.c for sockets). Protocol families: net/ipv4/, net/ipv6/, net/can/. Subsystems: net/netfilter/ (firewall), net/sched/ (TC/QoS), net/bridge/ (L2 bridging), net/netlink/ (IPC). Headers split between include/linux/ (kernel internal), include/net/ (networking subsystem), include/uapi/linux/ (user API). Drivers in drivers/net/ethernet/<vendor>/.

---

## Summary

- Kernel networking source: net/ (protocols) + drivers/net/ (drivers) + include/ (headers)
- Key source files: dev.c (net_device), skbuff.c (sk_buff), tcp.c/tcp_input.c/tcp_output.c (TCP)
- Glossary covers 60+ essential networking terms
- RFC 793/9293 (TCP), RFC 791 (IP), RFC 826 (ARP) are foundational
- Best books: Stevens TCP/IP Illustrated, Benvenuti Linux Network Internals
- Kernel Documentation/networking/ for subsystem-specific docs

---

Next: [Chapter 30 — Interview Preparation](Chapter_30_Interview_Preparation.md)
