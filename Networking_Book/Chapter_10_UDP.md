# Chapter 10: Transport Layer — UDP and Other Protocols

## Learning Goals
- Understand UDP architecture and its simplicity
- Know when to use UDP vs TCP
- Understand SCTP, raw sockets, and AF_PACKET
- Know UDP performance characteristics and common patterns

---

## 10.1 UDP Architecture

```
UDP is a thin layer over IP:
  - No connection setup (no handshake)
  - No reliability (no ACKs, no retransmission)
  - No ordering guarantee
  - No flow control, no congestion control
  - Preserves message boundaries (datagram-oriented)
  - Very low overhead: 8-byte header

┌──────────────────────────────────────────────────────────┐
│ UDP Header (8 bytes)                                     │
│                                                          │
│  0      15 16     31                                     │
│  ┌────────┬────────┐                                     │
│  │Src Port│Dst Port│  (each 16 bits)                     │
│  ├────────┼────────┤                                     │
│  │ Length │Checksum│  Length = header + data               │
│  └────────┴────────┘                                     │
│                                                          │
│  vs TCP Header: 20+ bytes, complex state machine          │
│  UDP: 8 bytes, stateless                                  │
└──────────────────────────────────────────────────────────┘
```

---

## 10.2 UDP in the Linux Kernel

```
TX Path:
  sendto(fd, buf, len, 0, &dest_addr, addrlen)
    │
    ▼
  sys_sendto() → udp_sendmsg()       [net/ipv4/udp.c]
    │
    ├── ip_make_skb()                 Build sk_buff
    │   ├── Allocate skb
    │   ├── Copy user data
    │   └── Build IP header
    │
    ├── udp_send_skb()                Add UDP header
    │   ├── Fill udphdr (src port, dst port, length)
    │   ├── Compute checksum (or CHECKSUM_PARTIAL for HW)
    │   └── ip_send_skb()
    │       └── ip_output() → ... → driver
    │
    └── Return bytes sent

RX Path:
  ip_local_deliver() → protocol 17 → udp_rcv()
    │
    ▼
  __udp4_lib_rcv()                    [net/ipv4/udp.c]
    │
    ├── __udp4_lib_lookup_skb()       Find socket by (dst IP, dst port)
    │
    ├── udp_queue_rcv_skb()
    │   ├── Verify checksum
    │   ├── sock_queue_rcv_skb()      Queue to socket's receive buffer
    │   └── sk->sk_data_ready()       Wake up reader
    │
    └── Process complete

  recvfrom(fd, buf, len, 0, &src_addr, &addrlen)
    │
    ▼
  udp_recvmsg()
    ├── Dequeue skb from sk->sk_receive_queue
    ├── Copy data to user buffer
    ├── Fill src_addr with sender info
    └── Return bytes received
```

---

## 10.3 UDP vs TCP Comparison

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│ Feature          │ TCP                  │ UDP                  │
├──────────────────┼──────────────────────┼──────────────────────┤
│ Connection       │ Connection-oriented  │ Connectionless       │
│ Reliability      │ Guaranteed delivery  │ Best effort          │
│ Ordering         │ Ordered              │ Unordered            │
│ Flow control     │ Window-based         │ None                 │
│ Congestion ctrl  │ CUBIC/BBR/Reno       │ None                 │
│ Header size      │ 20-60 bytes          │ 8 bytes              │
│ Message boundary │ Byte stream          │ Preserved (datagram) │
│ Overhead         │ High (state, timers) │ Low (stateless)      │
│ Latency          │ Higher (handshake)   │ Lower (no setup)     │
│ Multicast        │ No                   │ Yes                  │
│ Broadcast        │ No                   │ Yes                  │
│ Use cases        │ Web, file transfer,  │ DNS, VoIP, video,    │
│                  │ email, SSH           │ gaming, IoT, NTP     │
└──────────────────┴──────────────────────┴──────────────────────┘
```

---

## 10.4 UDP Patterns and Use Cases

### DNS (Port 53)

```
Client: sendto(fd, dns_query, len, 0, &dns_server, ...)
  - Small query (< 512 bytes typically)
  - Response expected quickly
  - If no response, retry after timeout
  - Application handles retransmission

Server: recvfrom(fd, buf, len, 0, &client, ...)
  - Process query, send response to client address
  - Stateless: no per-client state in kernel
```

### Multicast (e.g., video streaming, mDNS)

```c
/* Join multicast group */
struct ip_mreq mreq;
mreq.imr_multiaddr.s_addr = inet_addr("239.1.1.1");
mreq.imr_interface.s_addr = htonl(INADDR_ANY);
setsockopt(fd, IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, sizeof(mreq));

/* Send to multicast group */
sendto(fd, data, len, 0, &mcast_addr, sizeof(mcast_addr));

/* Receive multicast */
recvfrom(fd, buf, len, 0, &src, &srclen);
```

### UDP with Application-Level Reliability

```
QUIC (HTTP/3):
  - UDP transport with encryption (TLS 1.3)
  - Application-level reliability, ordering, flow control
  - Faster connection setup: 0-RTT or 1-RTT
  - Stream multiplexing without head-of-line blocking
  - Runs in user space (library, not kernel)

  Why UDP instead of new transport protocol?
  - Middleboxes (NATs, firewalls) block unknown protocols
  - UDP passes through everywhere
  - Faster iteration (user-space library vs kernel module)
```

---

## 10.5 UDP Performance

```
UDP Buffer Management:

  net.core.rmem_default = 212992     # Default RX buffer
  net.core.rmem_max = 26214400      # Max RX buffer

  # Important for high-rate UDP:
  net.core.netdev_max_backlog = 1000 # Max packets queued per CPU

  If packets arrive faster than application reads:
    sk->sk_receive_queue fills → sk_rcvbuf limit hit → packets DROPPED
    
  Diagnostic:
    cat /proc/net/udp
    # ... drops column shows per-socket drops
    
    ss -u -a -n
    # Shows UDP socket state including drops
```

### UDP and IP Fragmentation

```
UDP has no segmentation — sends entire datagram to IP layer.

If datagram > path MTU:
  IPv4: IP layer fragments into multiple packets
    Fragment 1: IP header + UDP header + data[0..offset]
    Fragment 2: IP header + data[offset+1..offset+n]  (no UDP header!)
    Fragment 3: ...
    
  Receiver reassembles all fragments before delivering to UDP.
  If ANY fragment is lost, ENTIRE datagram is lost.

  Recommendation: keep UDP datagrams ≤ 1472 bytes
    (1500 MTU - 20 IP - 8 UDP = 1472 payload)

  IPv6: no router fragmentation allowed
    Source must discover path MTU and segment
```

---

## 10.6 SCTP (Stream Control Transmission Protocol)

```
SCTP combines TCP reliability with UDP message boundaries:

Features:
  - Multi-homing: multiple IP addresses per endpoint
  - Multi-streaming: multiple independent streams per association
  - Message-oriented: preserves boundaries (like UDP)
  - Reliable: ordered delivery with SACK (like TCP)
  - 4-way handshake (INIT/INIT-ACK/COOKIE-ECHO/COOKIE-ACK)
  - No SYN flood vulnerability (cookie-based)

Use cases:
  - Telecom signaling (SS7 over IP)
  - Multi-homed servers
  - Applications needing message boundaries + reliability

  Linux: net/sctp/
  Socket: socket(AF_INET, SOCK_STREAM, IPPROTO_SCTP)
```

---

## 10.7 Raw Sockets

```c
/* Raw socket bypasses transport layer */
int fd = socket(AF_INET, SOCK_RAW, IPPROTO_ICMP);

/* Receives raw IP packets (including IP header) */
/* Used by: ping, traceroute, custom protocol implementations */

/* Requires CAP_NET_RAW capability (root or via capabilities) */

/* Example: send custom ICMP */
struct icmphdr icmp;
icmp.type = ICMP_ECHO;
icmp.code = 0;
icmp.un.echo.id = getpid();
icmp.un.echo.sequence = seq++;
icmp.checksum = 0;
icmp.checksum = compute_checksum(&icmp, sizeof(icmp));
sendto(fd, &icmp, sizeof(icmp), 0, &dest, sizeof(dest));
```

---

## 10.8 AF_PACKET (Layer 2 Raw Access)

```c
/* AF_PACKET provides raw access to link layer */
int fd = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL));

/* Receives ALL frames on an interface (like tcpdump) */
/* Can inject raw frames (like packet generators) */

/* Used by: tcpdump, Wireshark, dhclient, ARP tools */

/* Bind to specific interface */
struct sockaddr_ll sll = {
    .sll_family = AF_PACKET,
    .sll_protocol = htons(ETH_P_ALL),
    .sll_ifindex = if_nametoindex("eth0"),
};
bind(fd, (struct sockaddr *)&sll, sizeof(sll));

/* High-performance variant: PACKET_MMAP */
/* Maps ring buffer into user space — avoids copy */
/* Used by tcpdump for high packet rates */
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/ipv4/udp.c | UDP protocol implementation |
| net/ipv4/udp_offload.c | UDP offload (GRO/GSO) |
| net/ipv4/raw.c | Raw sockets |
| net/packet/af_packet.c | AF_PACKET implementation |
| net/sctp/ | SCTP protocol |
| include/net/udp.h | UDP structures |
| include/uapi/linux/udp.h | User-visible UDP definitions |

---

## Interview Questions

**Q1: When would you choose UDP over TCP?**
A: Use UDP when: (1) Low latency is critical (gaming, VoIP, live video — retransmitting stale data is useless). (2) Multicast/broadcast needed (TCP is point-to-point only). (3) Simple request-response (DNS — one packet each way, application retries). (4) Application handles reliability itself (QUIC, custom protocols). (5) IoT/embedded with constrained resources.

**Q2: What happens if a UDP datagram exceeds the MTU?**
A: IPv4 IP layer fragments it into multiple IP packets. Each fragment has its own IP header; only the first has the UDP header. The receiver reassembles fragments before delivering to UDP. If any fragment is lost, the entire datagram is lost (no selective retransmission). IPv6 does not allow router fragmentation — the source must perform PMTUD.

**Q3: What is the difference between SOCK_RAW and AF_PACKET?**
A: SOCK_RAW (with AF_INET) operates at Layer 3 — you see/craft IP packets but skip the transport layer. AF_PACKET operates at Layer 2 — you see/craft complete Ethernet frames. AF_PACKET can capture all traffic on an interface (promiscuous mode). SOCK_RAW sees only packets matching the specified protocol. tcpdump uses AF_PACKET.

**Q4: How does UDP handle socket demultiplexing?**
A: When a UDP packet arrives, __udp4_lib_lookup() searches the UDP hash table using (dst IP, dst port) as the primary key. If SO_REUSEPORT is set, multiple sockets can bind to the same port — the kernel selects one using a hash of the source 4-tuple for consistent flow affinity. If no socket matches, an ICMP "Port Unreachable" is generated.

**Q5: Why does QUIC use UDP instead of a new transport protocol?**
A: Middleboxes (NATs, firewalls, load balancers) are designed to handle TCP and UDP. A new IP protocol number would be blocked by most middleboxes. UDP passes through universally. Additionally, QUIC runs in user space as a library, enabling faster iteration and deployment without kernel changes. The "transport" functionality (reliability, congestion control, encryption) is implemented in the QUIC library on top of UDP.

---

## Summary

- UDP is a minimal, stateless protocol: 8-byte header, no connection, no reliability
- UDP preserves message boundaries (datagram-oriented) unlike TCP's byte stream
- UDP supports multicast and broadcast; TCP does not
- IP fragmentation handles oversized UDP datagrams but is fragile — avoid it
- Raw sockets (SOCK_RAW) access Layer 3; AF_PACKET accesses Layer 2
- SCTP combines TCP reliability with UDP message boundaries (multi-streaming)
- QUIC uses UDP as substrate to bypass middlebox restrictions

---

Next: [Chapter 11 — Internet Layer: IPv4 and IPv6](Chapter_11_IP_Layer.md)
