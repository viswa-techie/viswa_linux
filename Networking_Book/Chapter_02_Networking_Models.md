# Chapter 2: Networking Models — OSI and TCP/IP

## Learning Goals
- Understand the OSI 7-layer and TCP/IP 4-layer models in depth
- Map each layer to its Linux kernel implementation
- Know what data transformations happen at each layer
- Identify which kernel source files implement each layer

---

## 2.1 OSI Reference Model

The OSI (Open Systems Interconnection) model divides networking into 7 layers. Each layer provides services to the layer above and consumes services from the layer below.

```
┌─────────────────────────────────────────────────────────────────────┐
│ Layer 7: APPLICATION    │ HTTP, FTP, DNS, SSH, SMTP               │
│                         │ Data unit: Message/Data                  │
├─────────────────────────────────────────────────────────────────────┤
│ Layer 6: PRESENTATION   │ SSL/TLS, encoding, encryption           │
│                         │ Data unit: Data                          │
├─────────────────────────────────────────────────────────────────────┤
│ Layer 5: SESSION        │ Session management, RPC                  │
│                         │ Data unit: Data                          │
├─────────────────────────────────────────────────────────────────────┤
│ Layer 4: TRANSPORT      │ TCP, UDP, SCTP                          │
│                         │ Data unit: Segment (TCP) / Datagram (UDP)│
├─────────────────────────────────────────────────────────────────────┤
│ Layer 3: NETWORK        │ IP, ICMP, IGMP                          │
│                         │ Data unit: Packet                        │
├─────────────────────────────────────────────────────────────────────┤
│ Layer 2: DATA LINK      │ Ethernet, Wi-Fi, PPP                    │
│                         │ Data unit: Frame                         │
├─────────────────────────────────────────────────────────────────────┤
│ Layer 1: PHYSICAL       │ Cables, signals, bit encoding           │
│                         │ Data unit: Bit                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Layer-by-Layer Responsibilities

| Layer | Key Functions | Addressing |
|-------|--------------|------------|
| 7-Application | Application protocols, data semantics | URLs, hostnames |
| 6-Presentation | Data format conversion, encryption | N/A |
| 5-Session | Session setup/teardown, synchronization | Session IDs |
| 4-Transport | End-to-end delivery, flow/error control | Port numbers |
| 3-Network | Routing, logical addressing, fragmentation | IP addresses |
| 2-Data Link | Framing, MAC access, error detection | MAC addresses |
| 1-Physical | Bit transmission, signaling, modulation | N/A |

---

## 2.2 TCP/IP Model

The TCP/IP model (also called Internet model or DoD model) is the practical model used by the Internet and Linux. It has 4 layers:

```
┌──────────────────────────────────────────────────────┐
│ Layer 4: APPLICATION     │ HTTP, DNS, SSH, FTP       │
│ (maps to OSI 5-7)       │ User-space processes      │
├──────────────────────────────────────────────────────┤
│ Layer 3: TRANSPORT       │ TCP, UDP, SCTP            │
│ (maps to OSI 4)         │ End-to-end communication  │
├──────────────────────────────────────────────────────┤
│ Layer 2: INTERNET        │ IPv4, IPv6, ICMP, IGMP    │
│ (maps to OSI 3)         │ Routing and addressing    │
├──────────────────────────────────────────────────────┤
│ Layer 1: NETWORK ACCESS  │ Ethernet, Wi-Fi, PPP      │
│ (maps to OSI 1-2)       │ Physical + Data Link      │
└──────────────────────────────────────────────────────┘
```

---

## 2.3 OSI to TCP/IP Mapping

```
        OSI Model              TCP/IP Model           Linux Implementation
    ┌───────────────┐
    │ 7-Application │
    │ 6-Presentation│──────► APPLICATION ──────► User space (libc, apps)
    │ 5-Session     │
    ├───────────────┤
    │ 4-Transport   │──────► TRANSPORT   ──────► net/ipv4/tcp.c, udp.c
    ├───────────────┤
    │ 3-Network     │──────► INTERNET    ──────► net/ipv4/ip_input.c
    ├───────────────┤                              ip_output.c, route.c
    │ 2-Data Link   │──────► NETWORK     ──────► net/core/dev.c
    │ 1-Physical    │        ACCESS      ──────► drivers/net/ethernet/
    └───────────────┘
```

---

## 2.4 Linux Kernel Networking Layers

Linux implements the TCP/IP model with additional internal layering:

```
┌─────────────────────────────────────────────────────────────┐
│ USER SPACE                                                  │
│  send(fd, buf, len, 0)  /  recv(fd, buf, len, 0)           │
│  Application uses POSIX socket API via glibc                │
├─────────────────────────────────────────────────────────────┤
│ SYSTEM CALL BOUNDARY (syscall)                              │
│  sys_sendto() → __sys_sendto()   net/socket.c              │
├─────────────────────────────────────────────────────────────┤
│ SOCKET LAYER                                                │
│  struct socket + struct sock                                │
│  VFS integration: socket is a file descriptor               │
│  Dispatch to protocol family (AF_INET, AF_INET6, AF_UNIX)  │
│  Source: net/socket.c                                       │
├─────────────────────────────────────────────────────────────┤
│ TRANSPORT LAYER                                             │
│  TCP: net/ipv4/tcp.c, tcp_output.c, tcp_input.c            │
│  UDP: net/ipv4/udp.c                                       │
│  Segmentation, reliability, port multiplexing               │
├─────────────────────────────────────────────────────────────┤
│ NETWORK LAYER                                               │
│  IPv4: net/ipv4/ip_output.c, ip_input.c                    │
│  IPv6: net/ipv6/ip6_output.c                               │
│  Routing: net/ipv4/route.c, fib_trie.c                     │
│  Fragmentation, TTL, checksum                               │
├─────────────────────────────────────────────────────────────┤
│ NETFILTER LAYER (hooks at multiple points)                  │
│  NF_INET_PRE_ROUTING, NF_INET_LOCAL_IN, etc.               │
│  Source: net/netfilter/                                     │
├─────────────────────────────────────────────────────────────┤
│ TRAFFIC CONTROL (tc) LAYER                                  │
│  Queueing disciplines (qdiscs), scheduling                  │
│  Source: net/sched/                                         │
├─────────────────────────────────────────────────────────────┤
│ DEVICE LAYER                                                │
│  struct net_device                                          │
│  dev_queue_xmit() for TX, netif_receive_skb() for RX       │
│  Source: net/core/dev.c                                     │
├─────────────────────────────────────────────────────────────┤
│ DRIVER LAYER                                                │
│  ndo_start_xmit() for TX, NAPI poll() for RX               │
│  DMA mapping, descriptor rings                              │
│  Source: drivers/net/ethernet/                              │
├─────────────────────────────────────────────────────────────┤
│ HARDWARE                                                    │
│  NIC, PHY, MAC, DMA engine, interrupt controller            │
└─────────────────────────────────────────────────────────────┘
```

---

## 2.5 Data Transformation at Each Layer

### Transmission (Top-Down)

```
Application:  "Hello"
                ↓
Transport:   [TCP HDR][Hello]           ← adds src/dst port, seq#, checksum
                ↓
Network:     [IP HDR][TCP HDR][Hello]   ← adds src/dst IP, TTL, protocol
                ↓
Link:        [ETH HDR][IP HDR][TCP HDR][Hello][FCS]  ← adds MAC addrs, type
                ↓
Physical:    101001001110010101001...   ← modulated signal on wire
```

### Reception (Bottom-Up)

```
Physical:    101001001110010101001...
                ↓
Link:        [ETH HDR][IP HDR][TCP HDR][Hello][FCS]  ← verify FCS, strip ETH
                ↓
Network:     [IP HDR][TCP HDR][Hello]   ← verify checksum, route decision
                ↓
Transport:   [TCP HDR][Hello]           ← reassemble, deliver to socket
                ↓
Application:  "Hello"
```

---

## 2.6 Protocol Numbers at Each Layer

```
Layer 2 (Ethernet):
  EtherType field identifies Layer 3 protocol:
    0x0800 = IPv4
    0x0806 = ARP
    0x86DD = IPv6
    0x8100 = VLAN (802.1Q)

Layer 3 (IP):
  Protocol field identifies Layer 4 protocol:
    1  = ICMP
    6  = TCP
    17 = UDP
    132 = SCTP

Layer 4 (TCP/UDP):
  Port number identifies application:
    22   = SSH
    53   = DNS
    80   = HTTP
    443  = HTTPS
    5060 = SIP
```

### Demultiplexing Flow

```
Frame arrives at NIC
    │
    ▼
Check EtherType (0x0800?)
    │ yes
    ▼
Pass to IP layer
    │
    ▼
Check IP Protocol field (6?)
    │ yes
    ▼
Pass to TCP layer
    │
    ▼
Check dst port (80?)
    │ yes
    ▼
Deliver to socket bound to port 80
```

---

## 2.7 Practical Layer Identification

### Using tcpdump to See Layers

```bash
# Capture with all headers visible
tcpdump -i eth0 -XX -c 1

# Output breakdown:
# Bytes 0-13:   Ethernet header (dst MAC, src MAC, EtherType)
# Bytes 14-33:  IP header (version, TTL, protocol, src/dst IP)
# Bytes 34-53:  TCP header (src/dst port, seq, ack, flags)
# Bytes 54+:    Payload
```

### Using /proc and /sys to See Kernel Implementation

```bash
# Registered protocol families
cat /proc/net/protocols

# Active sockets per protocol
cat /proc/net/tcp      # TCP sockets
cat /proc/net/udp      # UDP sockets

# Network devices (Layer 2)
ls /sys/class/net/

# Interface statistics
cat /sys/class/net/eth0/statistics/rx_packets
```

---

## Kernel Source References

| Layer | Key Source Files |
|-------|-----------------|
| Socket | net/socket.c |
| TCP | net/ipv4/tcp.c, tcp_input.c, tcp_output.c |
| UDP | net/ipv4/udp.c |
| IPv4 | net/ipv4/ip_input.c, ip_output.c, ip_forward.c |
| IPv6 | net/ipv6/ip6_input.c, ip6_output.c |
| Routing | net/ipv4/route.c, fib_trie.c |
| Device | net/core/dev.c |
| Netfilter | net/netfilter/core.c, nf_tables_core.c |
| Traffic Control | net/sched/sch_generic.c |
| Drivers | drivers/net/ethernet/ |
| sk_buff | include/linux/skbuff.h, net/core/skbuff.c |

---

## Interview Questions

**Q1: Explain the difference between OSI and TCP/IP models.**
A: OSI has 7 layers (theoretical reference model by ISO). TCP/IP has 4 layers (practical model used by the Internet). OSI separates Application/Presentation/Session; TCP/IP combines them into Application. OSI separates Physical/Data Link; TCP/IP combines them into Network Access. TCP/IP was developed alongside actual protocols (TCP, IP); OSI was defined after the fact. Linux implements TCP/IP, not OSI.

**Q2: Which Linux kernel layers does a packet traverse during reception?**
A: NIC hardware → driver (NAPI poll, DMA descriptor) → net/core/dev.c (`netif_receive_skb()`) → netfilter PRE_ROUTING hook → IP layer (`ip_rcv()`) → netfilter LOCAL_IN hook → TCP/UDP layer (`tcp_v4_rcv()` / `udp_rcv()`) → socket receive queue → user space `recv()`.

**Q3: How does the kernel know which protocol to deliver a packet to?**
A: Demultiplexing. Ethernet EtherType identifies L3 (0x0800=IPv4). IP protocol field identifies L4 (6=TCP, 17=UDP). L4 destination port identifies the socket. Each layer strips its header and uses the embedded protocol identifier to dispatch upward.

**Q4: What is the difference between a socket and a port?**
A: A port is a 16-bit number (0-65535) that identifies a service on a host. A socket is the full endpoint: (protocol, IP address, port). A unique connection is identified by a 5-tuple: (protocol, src IP, src port, dst IP, dst port). Multiple sockets can share a port (e.g., SO_REUSEPORT).

**Q5: Where in the Linux kernel is each layer implemented?**
A: Socket layer: net/socket.c. Transport: net/ipv4/tcp.c (TCP), net/ipv4/udp.c (UDP). Network: net/ipv4/ip_input.c, ip_output.c. Device: net/core/dev.c. Drivers: drivers/net/. Netfilter: net/netfilter/. Traffic control: net/sched/.

---

## Summary

- OSI is a 7-layer reference model; TCP/IP is a 4-layer practical model
- Linux implements TCP/IP with additional internal layers (netfilter, tc, device)
- Each layer adds/strips headers — this is encapsulation/decapsulation
- Protocol numbers at each layer enable demultiplexing: EtherType → IP protocol → port
- Every layer maps to specific kernel source files under net/ and drivers/net/
- Understanding layering is essential for debugging — you isolate problems to specific layers

---

Next: [Chapter 3 — Network Hardware Architecture](Chapter_03_Network_Hardware.md)
