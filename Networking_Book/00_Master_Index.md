# Linux Networking Subsystem — Complete Book

## Comprehensive Guide: Beginner to Kernel Developer

```
Application Layer        User Space
─────────────────────────────────────────
Socket Layer             ← Ch 8
    |
Transport Layer (TCP/UDP)← Ch 9-10
    |
Internet Layer (IP)      ← Ch 11
    |
Link Layer (Ethernet)    ← Ch 12
    |
Network Device (net_device) ← Ch 5-6
    |
NIC Driver + DMA         ← Ch 6-7
    |
Network Hardware         ← Ch 3
```

---

## Part I: Foundations

| Ch | Title | Topics |
|----|-------|--------|
| [01](Chapter_01_Foundations_History.md) | Foundations and History | Networking basics, client-server, peer-to-peer, history, TCP/IP evolution |
| [02](Chapter_02_Networking_Models.md) | Networking Models — OSI and TCP/IP | OSI 7-layer, TCP/IP 4-layer, layer mapping, Linux kernel layers |
| [03](Chapter_03_Network_Hardware.md) | Network Hardware Architecture | NIC internals, Ethernet PHY/MAC, DMA engines, interrupt coalescing, MMIO |

## Part II: Linux Networking Core

| Ch | Title | Topics |
|----|-------|--------|
| [04](Chapter_04_Linux_Networking_Architecture.md) | Linux Networking Architecture | Subsystem overview, kernel layers, packet pipeline, data flow |
| [05](Chapter_05_Net_Device.md) | Network Devices — struct net_device | net_device anatomy, registration, lifecycle, virtual devices, sysfs |
| [06](Chapter_06_Network_Drivers.md) | Network Device Drivers | Driver architecture, init, TX/RX paths, interrupt handling, DMA |
| [07](Chapter_07_sk_buff.md) | sk_buff and Buffer Management | sk_buff structure, allocation, push/pull/put, cloning, fragments |
| [08](Chapter_08_Socket_Layer.md) | Socket Layer | Socket abstraction, types, creation, VFS integration, proto_ops |

## Part III: Protocol Stack

| Ch | Title | Topics |
|----|-------|--------|
| [09](Chapter_09_TCP.md) | Transport Layer — TCP Deep Dive | TCP state machine, 3-way handshake, congestion control, reliability |
| [10](Chapter_10_UDP.md) | Transport Layer — UDP and Others | UDP architecture, SCTP, raw sockets, protocol selection |
| [11](Chapter_11_IP_Layer.md) | Internet Layer — IPv4 and IPv6 | IP addressing, routing lookup, fragmentation, IPv6 differences, ICMP |
| [12](Chapter_12_Link_Layer.md) | Link Layer — Ethernet, ARP, VLAN | Ethernet framing, ARP resolution, VLAN tagging, 802.1Q, MAC addressing |

## Part IV: Packet Journey

| Ch | Title | Topics |
|----|-------|--------|
| [13](Chapter_13_TX_Path.md) | Packet Transmission Path | send() to wire, socket → transport → IP → device → NIC, GSO/TSO |
| [14](Chapter_14_RX_Path.md) | Packet Reception Path | Wire to recv(), NIC → NAPI → IP → transport → socket, GRO/LRO |
| [15](Chapter_15_NAPI_Interrupts.md) | Interrupt Handling and NAPI | Hardirq, softirq, NAPI poll, interrupt coalescing, busy polling |
| [16](Chapter_16_Queues_Multiqueue.md) | Network Queues and Multiqueue | TX/RX queues, qdisc, multiqueue NIC, RSS, RPS, XPS, RFS |

## Part V: Advanced Networking

| Ch | Title | Topics |
|----|-------|--------|
| [17](Chapter_17_Network_Namespaces.md) | Network Namespaces and Containers | Namespace isolation, veth pairs, container networking, CNI |
| [18](Chapter_18_Virtual_Networking.md) | Virtual Networking | TUN/TAP, bridges, bonding/teaming, macvlan, ipvlan, VXLAN |
| [19](Chapter_19_Routing_Forwarding.md) | Routing and Forwarding | FIB, routing tables, policy routing, NAT, conntrack |
| [20](Chapter_20_Netfilter.md) | Netfilter and Packet Filtering | Netfilter hooks, iptables chains, nftables, connection tracking |
| [21](Chapter_21_Traffic_Control_QoS.md) | Traffic Control and QoS | tc framework, qdiscs (HTB, CBQ, fq_codel), classes, filters, shaping |
| [22](Chapter_22_Performance_Optimization.md) | Network Performance Optimization | Zero-copy (sendfile, splice), checksum/TSO offload, GRO, XDP |

## Part VI: Cutting Edge and Specialization

| Ch | Title | Topics |
|----|-------|--------|
| [23](Chapter_23_High_Performance.md) | High-Performance Networking | Kernel bypass, DPDK, AF_XDP, RDMA, io_uring, eBPF networking |
| [24](Chapter_24_Kernel_APIs_Netlink.md) | Kernel Networking APIs and Netlink | Socket API internals, netdev API, netlink protocol, genetlink |
| [25](Chapter_25_Embedded_Automotive.md) | Embedded and Automotive Networking | CAN bus, Ethernet AVB/TSN, automotive protocols, IoT networking |
| [26](Chapter_26_Other_OS_Networking.md) | Networking in Other Operating Systems | Windows NDIS, macOS Network.framework, QNX io-pkt, RTOS lwIP |

## Part VII: Reference and Interview

| Ch | Title | Topics |
|----|-------|--------|
| [27](Chapter_27_Debugging.md) | Network Debugging Tools and Techniques | tcpdump, Wireshark, ss, ip, ftrace, perf, dropwatch, eBPF tracing |
| [28](Chapter_28_Flow_Diagrams.md) | End-to-End Flow Diagrams | TX flow, RX flow, TCP handshake, NAPI cycle, routing decision |
| [29](Chapter_29_Source_Glossary_References.md) | Kernel Source Map, Glossary, References | Key source files, definitions, books, RFCs, kernel docs |
| [30](Chapter_30_Interview_Preparation.md) | Interview Preparation | 50+ categorized Q&A, system design, debugging scenarios |

---

## Reading Paths

### Path 1: Network Driver Developer
Ch 1-3 → 4-7 → 13-16 → 22-23 → 27 → 30

### Path 2: Protocol Stack Understanding
Ch 1-2 → 4 → 7-12 → 13-14 → 28 → 30

### Path 3: System/DevOps Engineer
Ch 1-2 → 17-21 → 22 → 27 → 30

### Path 4: Embedded/Automotive
Ch 1-3 → 4-7 → 15-16 → 25 → 27 → 30

### Path 5: Interview Preparation (Fast Track)
Ch 2 → 5 → 7 → 9 → 13-14 → 15 → 28 → 30

---

## Framework Summary

```
┌─────────────────────────────────────────────────────────┐
│                    USER SPACE                           │
│  Application: send()/recv(), read()/write()             │
├─────────────────────────────────────────────────────────┤
│                    SOCKET LAYER                         │
│  struct socket → struct sock → protocol ops             │
├─────────────────────────────────────────────────────────┤
│                  TRANSPORT LAYER                        │
│  TCP (net/ipv4/tcp.c)  │  UDP (net/ipv4/udp.c)         │
├─────────────────────────────────────────────────────────┤
│                   NETWORK LAYER                         │
│  IPv4 (net/ipv4/)  │  IPv6 (net/ipv6/)  │  Routing      │
├─────────────────────────────────────────────────────────┤
│              NETFILTER / TRAFFIC CONTROL                │
│  iptables/nftables hooks  │  tc qdiscs                  │
├─────────────────────────────────────────────────────────┤
│                   DEVICE LAYER                          │
│  struct net_device  │  dev_queue_xmit()  │  netif_rx()  │
├─────────────────────────────────────────────────────────┤
│                  DRIVER LAYER                           │
│  ndo_start_xmit()  │  NAPI poll()  │  DMA              │
├─────────────────────────────────────────────────────────┤
│                   HARDWARE                              │
│  NIC  │  PHY  │  MAC  │  DMA Engine  │  Interrupts      │
└─────────────────────────────────────────────────────────┘
```
