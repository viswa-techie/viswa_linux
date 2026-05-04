# Chapter 30: Interview Preparation

## Learning Goals
- Consolidate all networking concepts into interview-ready answers
- Practice system design for networking scenarios
- Know common gotchas and deep-dive questions
- Prepare for whiteboard and coding questions

---

## 30.1 Concept Quick-Fire Questions

### Fundamentals

**Q: What happens when you type a URL in a browser?**
A:
1. DNS resolution: browser cache → OS cache → resolver → root → TLD → authoritative
2. TCP connection: 3-way handshake (SYN, SYN-ACK, ACK)
3. TLS handshake (if HTTPS): ClientHello, ServerHello, certificates, key exchange
4. HTTP request: GET /path HTTP/1.1 with headers
5. Server processes request, sends HTTP response
6. Browser parses HTML, requests CSS/JS/images (parallel connections)
7. Rendering: DOM tree + CSSOM → render tree → layout → paint

**Q: Explain the difference between TCP and UDP.**
A:
| Aspect | TCP | UDP |
|--------|-----|-----|
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Reliability | Guaranteed delivery (ACK, retransmit) | Best-effort (no ACK) |
| Ordering | In-order delivery (sequence numbers) | No ordering guarantee |
| Flow control | Sliding window (rwnd) | None |
| Congestion control | CUBIC/BBR (cwnd) | None |
| Header size | 20-60 bytes | 8 bytes |
| Use cases | HTTP, SSH, FTP, SMTP | DNS, VoIP, video, gaming |

**Q: What is the difference between a switch, router, and firewall?**
A: **Switch**: L2 device, forwards frames by MAC address, uses FDB (forwarding database). **Router**: L3 device, forwards packets by IP address, uses routing table (FIB). Decrements TTL, may fragment. **Firewall**: Inspects packets against rules (L3/L4/L7), allows/blocks based on policy. Can be stateful (tracks connections) or stateless.

### Linux Kernel Networking

**Q: What is NAPI and why was it created?**
A: NAPI (New API) is an interrupt mitigation technique. Under high packet rates, per-packet interrupts cause livelock (CPU spends all time handling interrupts, no time processing). NAPI solution: first packet triggers interrupt → driver disables interrupts and schedules NAPI → softirq polls NIC in a loop (up to budget=64 packets) → when ring empties, re-enable interrupts. Transitions between interrupt-driven (low load) and polling (high load) automatically.

**Q: Explain sk_buff and its key operations.**
A: sk_buff is the kernel's packet data structure. Key layout: headroom → data → tailroom. Network headers are tracked by mac_header, network_header, transport_header offsets. Operations: skb_reserve() adds headroom, skb_put() extends tail (add data), skb_push() extends head (prepend header), skb_pull() remove header (advance data pointer). Cloning (skb_clone) shares data, copying (skb_copy) duplicates everything. Linear data + paged fragments (frags[]) for large packets.

**Q: How does sk_buff avoid copies on receive?**
A: The driver pre-allocates buffers and posts them to the NIC's DMA ring. When a packet arrives, the NIC DMA-writes directly to the pre-posted buffer. The driver then wraps this buffer in an sk_buff (or assigns it as a page fragment). The skb travels up the stack without copying. Only when data is delivered to userspace (tcp_recvmsg → skb_copy_datagram_msg) is it copied to user buffers. With MSG_ZEROCOPY or AF_XDP, even this copy can be eliminated.

**Q: Describe the net_device registration flow.**
A: alloc_etherdev_mqs(sizeof_priv, txqs, rxqs) allocates net_device + private data. Driver fills in: netdev_ops, ethtool_ops, name, features (NETIF_F_*), MTU, MAC address. register_netdev(dev) calls register_netdevice() which: assigns ifindex, adds to global list and hash tables, calls dev->netdev_ops->ndo_init() if defined, creates /sys/class/net/<name>/, sends RTM_NEWLINK netlink notification. Device starts in DOWN state; "ip link set up" triggers ndo_open().

---

## 30.2 Deep-Dive Questions

**Q: Walk through the complete lifecycle of a TCP connection from SYN to FIN.**
A:
```
Connection setup (server: LISTEN, client: CLOSED):
  1. Client: SYN (seq=X) → state: SYN_SENT
  2. Server: receives SYN → create request_sock in SYN queue
     Send SYN-ACK (seq=Y, ack=X+1) → request stays in SYN queue
  3. Client: receives SYN-ACK → state: ESTABLISHED
     Send ACK (ack=Y+1)
  4. Server: receives ACK → promote to accept queue (full socket)
     state: ESTABLISHED → accept() returns fd

Data transfer:
  5. send()/recv() → tcp_sendmsg/tcp_recvmsg
     TCP manages: sequence numbers, ACKs, retransmission,
     congestion window (CUBIC/BBR), flow control (rwnd)

Connection teardown (active close by client):
  6. Client: close() → FIN (seq=M) → state: FIN_WAIT_1
  7. Server: receives FIN → ACK → state: CLOSE_WAIT
     Application read() returns 0 (EOF)
  8. Server: close() → FIN (seq=N) → state: LAST_ACK
  9. Client: receives FIN → ACK → state: TIME_WAIT (2*MSL=60s)
  10. Server: receives ACK → state: CLOSED
  11. Client: after 2*MSL → state: CLOSED
```

**Q: What causes packet drops in the Linux network stack and how do you find them?**
A:
```
Drop points and diagnostics:
  1. NIC ring buffer full:      ethtool -S eth0 | grep -i drop
  2. Driver allocation failure: ethtool -S eth0 | grep alloc_fail
  3. Backlog overflow:          /proc/net/softnet_stat column 2
  4. Netfilter DROP rule:       iptables -L -v -n (check counters)
  5. Conntrack table full:      dmesg | grep conntrack
  6. Socket receive buffer full: ss -m (Recv-Q > 0 sustained)
  7. TCP out-of-window:         netstat -s | grep 'out of window'
  8. Route not found:           ip route get <dst>
  9. ARP resolution failure:    ip neigh show (FAILED entries)

  Precise diagnosis:
    dropwatch -l kas                    # Real-time drop location
    perf record -e skb:kfree_skb -a     # Record with stack traces
    bpftrace -e 'tracepoint:skb:kfree_skb { @[kstack] = count(); }'
```

**Q: Explain how Linux handles 10 million concurrent connections.**
A:
```
Key subsystems:
  1. Socket hash table: inet_hashtables.c
     - Established: ehash, hashed by 4-tuple, per-chain spinlock
     - Listening: lhash2, hashed by port
     - Size: net.core.somaxconn, net.ipv4.tcp_max_tw_buckets

  2. Memory:
     - Each connection: ~3-5 KB (tcp_sock + buffers)
     - 10M × 5KB = 50 GB → need large RAM
     - Tune: tcp_rmem/tcp_wmem min values, tcp_mem limits
     - Minimize per-connection buffering

  3. File descriptors:
     - ulimit -n → raise to > 10M
     - /proc/sys/fs/file-max → system-wide limit

  4. Event notification:
     - epoll: O(1) per event, O(n) total monitoring
     - Level-triggered: simpler, edge-triggered: fewer syscalls
     - io_uring: fewer syscalls, shared ring buffers

  5. Kernel tuning:
     - net.ipv4.tcp_tw_reuse=1 (reuse TIME_WAIT)
     - net.ipv4.tcp_fin_timeout=15 (reduce FIN wait)
     - net.core.netdev_max_backlog=30000
     - net.netfilter.nf_conntrack_max=10000000 (if using NAT)
```

---

## 30.3 System Design Questions

**Q: Design a high-performance load balancer.**
A:
```
Architecture options:
  1. L4 Load Balancer (transport layer):
     - Linux: IPVS (ip_vs) in kernel — 10+ Mpps
     - Methods: NAT, Direct Return (DR), IP tunneling
     - Scheduling: Round-robin, weighted, least-connections, hash
     - Healthchecks: keepalived

  2. XDP/eBPF Load Balancer:
     - Katran (Facebook), Cilium
     - Process at driver level — 20+ Mpps
     - No conntrack overhead
     - Consistent hashing for backend selection

  3. L7 Load Balancer:
     - nginx, HAProxy, Envoy
     - HTTP-aware: route by URL, headers, cookies
     - TLS termination
     - Lower throughput (~1 Mpps) but richer features

  Design for 10 Gbps:
  ┌──────┐    ┌──────────────┐    ┌──────────────┐
  │Client│───►│ L4 LB (IPVS/ │───►│ Backend      │
  │      │    │ XDP/DSR)     │    │ Servers      │
  └──────┘    └──────────────┘    └──────────────┘

  Direct Server Return (DSR):
  - LB only handles inbound (source: client → LB → backend)
  - Backend replies directly to client (bypass LB)
  - LB handles only SYN packets (~1% of traffic)
  - 10x capacity improvement
```

**Q: Design a container networking solution.**
A:
```
Requirements: isolation, connectivity, scalability, policy

Approach (Kubernetes CNI):
  ┌─────────────────────────────────────────────────────────┐
  │ Node                                                    │
  │  ┌────────┐  ┌────────┐                                │
  │  │ Pod A  │  │ Pod B  │  Each pod gets its own          │
  │  │ eth0   │  │ eth0   │  network namespace              │
  │  │10.244  │  │10.244  │                                │
  │  │ .1.2   │  │ .1.3   │                                │
  │  └──┬─────┘  └──┬─────┘                                │
  │     │ veth      │ veth                                   │
  │     └─────┬─────┘                                       │
  │           │                                              │
  │     ┌─────┴─────┐                                       │
  │     │ Bridge    │  OR eBPF (Cilium)                      │
  │     │ cni0      │  No bridge needed with eBPF            │
  │     └─────┬─────┘                                       │
  │           │                                              │
  │     ┌─────┴─────┐                                       │
  │     │ eth0      │  Node's physical interface             │
  │     │10.0.0.5   │                                       │
  │     └─────┬─────┘                                       │
  └───────────┼─────────────────────────────────────────────┘
              │ VXLAN / Geneve overlay (or BGP for underlay)
              │
  ┌───────────┼─────────────────────────────────────────────┐
  │ Other Node│                                              │
  │     ┌─────┴─────┐                                       │
  │     │ eth0      │                                       │
  │     └─────┬─────┘                                       │
  │           │                                              │
  │     ┌─────┴─────┐                                       │
  │     │ Pods ...  │                                       │
  │     └───────────┘                                       │
  └──────────────────────────────────────────────────────────┘

  Network policy: eBPF or iptables rules per pod
  Service discovery: kube-proxy (iptables/IPVS) or Cilium eBPF
  Overlay vs underlay: VXLAN/Geneve for simplicity, BGP for performance
```

---

## 30.4 Coding Questions

### Implement a simple TCP echo server

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>

int main(void)
{
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);

    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_addr.s_addr = INADDR_ANY,
        .sin_port = htons(8080),
    };

    bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(server_fd, 128);

    while (1) {
        int client_fd = accept(server_fd, NULL, NULL);
        char buf[4096];
        ssize_t n;

        while ((n = read(client_fd, buf, sizeof(buf))) > 0)
            write(client_fd, buf, n);

        close(client_fd);
    }
}
```

### Implement epoll-based non-blocking server

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <netinet/in.h>
#include <errno.h>

#define MAX_EVENTS 1024

static void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main(void)
{
    int server_fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_addr.s_addr = INADDR_ANY,
        .sin_port = htons(8080),
    };
    bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(server_fd, 128);

    int epfd = epoll_create1(0);
    struct epoll_event ev = { .events = EPOLLIN, .data.fd = server_fd };
    epoll_ctl(epfd, EPOLL_CTL_ADD, server_fd, &ev);

    struct epoll_event events[MAX_EVENTS];

    while (1) {
        int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        for (int i = 0; i < nfds; i++) {
            if (events[i].data.fd == server_fd) {
                int client_fd = accept(server_fd, NULL, NULL);
                if (client_fd < 0) continue;
                set_nonblocking(client_fd);
                ev.events = EPOLLIN | EPOLLET;
                ev.data.fd = client_fd;
                epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);
            } else {
                char buf[4096];
                ssize_t n = read(events[i].data.fd, buf, sizeof(buf));
                if (n <= 0) {
                    close(events[i].data.fd);
                } else {
                    write(events[i].data.fd, buf, n);
                }
            }
        }
    }
}
```

### Parse a raw Ethernet frame

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <net/ethernet.h>
#include <netinet/ip.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>

int main(void)
{
    int fd = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
    unsigned char buf[65536];

    while (1) {
        ssize_t len = recvfrom(fd, buf, sizeof(buf), 0, NULL, NULL);
        if (len < (ssize_t)sizeof(struct ether_header)) continue;

        struct ether_header *eth = (struct ether_header *)buf;
        printf("ETH: %02x:%02x:%02x:%02x:%02x:%02x → %02x:%02x:%02x:%02x:%02x:%02x type=0x%04x\n",
               eth->ether_shost[0], eth->ether_shost[1], eth->ether_shost[2],
               eth->ether_shost[3], eth->ether_shost[4], eth->ether_shost[5],
               eth->ether_dhost[0], eth->ether_dhost[1], eth->ether_dhost[2],
               eth->ether_dhost[3], eth->ether_dhost[4], eth->ether_dhost[5],
               ntohs(eth->ether_type));

        if (ntohs(eth->ether_type) == ETHERTYPE_IP) {
            struct iphdr *ip = (struct iphdr *)(buf + sizeof(struct ether_header));
            char src[INET_ADDRSTRLEN], dst[INET_ADDRSTRLEN];
            inet_ntop(AF_INET, &ip->saddr, src, sizeof(src));
            inet_ntop(AF_INET, &ip->daddr, dst, sizeof(dst));
            printf("  IP: %s → %s proto=%d ttl=%d\n",
                   src, dst, ip->protocol, ip->ttl);

            if (ip->protocol == IPPROTO_TCP) {
                struct tcphdr *tcp = (struct tcphdr *)((char *)ip + ip->ihl * 4);
                printf("  TCP: %d → %d seq=%u ack=%u flags=%s%s%s%s\n",
                       ntohs(tcp->source), ntohs(tcp->dest),
                       ntohl(tcp->seq), ntohl(tcp->ack_seq),
                       tcp->syn ? "S" : "", tcp->ack ? "A" : "",
                       tcp->fin ? "F" : "", tcp->rst ? "R" : "");
            }
        }
    }
}
```

---

## 30.5 Common Gotchas and Tricky Questions

```
Q: Why does TIME_WAIT exist and can you eliminate it?
A: Ensures final ACK reaches peer (retransmit if lost) and prevents
   old segments from being accepted by new connections on same 4-tuple.
   Cannot eliminate but can: net.ipv4.tcp_tw_reuse=1 (reuse for outgoing),
   SO_REUSEADDR (allow bind to TIME_WAIT port for listening).
   Do NOT use tcp_tw_recycle (removed in 4.12, breaks behind NAT).

Q: What's the maximum number of TCP connections a server can handle?
A: Not limited by port numbers (65535). A connection is identified by
   4-tuple: (src_ip, src_port, dst_ip, dst_port). Server listens on
   one port but accepts millions of connections from different clients.
   Practical limit: memory (~5KB per connection), file descriptors,
   CPU for processing.

Q: Why might you see RX drops but no errors?
A: NIC ring buffer overflow — packets arrive faster than kernel
   processes them. Solution: increase ring buffer (ethtool -G),
   enable RSS/RPS, tune NAPI budget, increase netdev_max_backlog,
   or use XDP to drop unwanted traffic early.

Q: Difference between SO_REUSEADDR and SO_REUSEPORT?
A: SO_REUSEADDR: allows bind() to port in TIME_WAIT. Required for
   server restart without waiting 60s. SO_REUSEPORT: allows multiple
   sockets to bind to the same port. Kernel distributes connections
   across sockets (load balancing for multi-threaded servers).

Q: What is head-of-line blocking?
A: TCP: if segment N is lost, segments N+1,N+2,... cannot be delivered
   to application until N is retransmitted. Blocks all multiplexed
   streams (HTTP/2 problem). UDP: no HOL blocking (used by QUIC/HTTP/3).

Q: What's the difference between a half-open and half-closed connection?
A: Half-open: one side crashed without sending FIN — the other side
   doesn't know the connection is dead (detected by keepalive or timeout).
   Half-closed: one side sent FIN (close for writing) but can still
   receive data — valid TCP state (FIN_WAIT / CLOSE_WAIT).
```

---

## 30.6 Interview Preparation Checklist

```
Before the interview:
  [ ] Draw TX path from memory (send → wire, all kernel functions)
  [ ] Draw RX path from memory (wire → recv, all kernel functions)
  [ ] Draw TCP state machine (11 states, all transitions)
  [ ] Explain sk_buff layout and operations with diagram
  [ ] Explain NAPI flow (interrupt → schedule → poll → complete)
  [ ] Know 5 ways to find packet drops in Linux
  [ ] Know sysctls for TCP performance tuning (5 key ones)
  [ ] Explain XDP vs DPDK trade-offs
  [ ] Explain connection tracking and NAT
  [ ] Know 3 netfilter hooks and their purpose
  [ ] Explain fq_codel (why it's default, how it works)
  [ ] Write echo server from memory (blocking + epoll)
  [ ] Know the DNS → TCP → HTTP flow end-to-end
  [ ] Explain RSS, RPS, RFS, XPS one sentence each

During the interview:
  [ ] Ask clarifying questions before diving into answers
  [ ] Draw diagrams for complex explanations
  [ ] Mention kernel source files to show depth
  [ ] Start with high-level overview, then drill into specifics
  [ ] Relate concepts to practical experience
  [ ] Mention trade-offs for design decisions
```

---

## Summary

This chapter consolidates the 29 preceding chapters into interview-ready format:
- Quick-fire answers for common questions (TCP vs UDP, NAPI, sk_buff)
- Deep-dive answers with ASCII diagrams (full TCP lifecycle, drop diagnosis)
- System design (load balancer, container networking)
- Coding (echo server, epoll server, raw packet parser)
- Common gotchas (TIME_WAIT, max connections, SO_REUSEADDR vs SO_REUSEPORT)
- Preparation checklist for review before interviews

---

**End of Linux Networking Subsystem Book**

Back to: [Master Index](00_Master_Index.md)
