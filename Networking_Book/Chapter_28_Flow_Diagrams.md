# Chapter 28: End-to-End Flow Diagrams

## Learning Goals
- Trace complete packet paths through the kernel with exact functions
- Understand the full TX and RX data flows including all subsystems
- Know the complete TCP connection lifecycle
- Visualize packet processing with annotated diagrams

---

## 28.1 Complete TX Flow: send() to Wire

```
Application calls send(fd, buf, len, 0):

  ┌─────────────────────── USER SPACE ──────────────────────┐
  │                                                          │
  │  send(fd, buf, len, 0)                                   │
  │    │                                                     │
  │    └── SYSCALL ENTRY: __sys_sendto()                     │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌─────────────────────── SOCKET LAYER ────────────────────┐
  │                                                          │
  │  sock_sendmsg(sock, &msg)                                │
  │    │                                                     │
  │    └── inet_sendmsg(sock, msg, len)    [AF_INET dispatch]│
  │          │                                               │
  │          └── tcp_sendmsg(sk, msg, len)  [SOCK_STREAM]    │
  │                │                                         │
  │                ├── Copy user data → sk_buff               │
  │                │   (may coalesce with existing skb)       │
  │                │                                         │
  │                └── tcp_push(sk, flags, mss, ...)          │
  │                      │                                   │
  │                      └── __tcp_push_pending_frames()      │
  │                            │                             │
  │                            └── tcp_write_xmit(sk, mss, ..)│
  │                                  │                       │
  │                                  └── [per segment loop]  │
  │                                        tcp_transmit_skb()│
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌─────────────────────── TRANSPORT (TCP) ─────────────────┐
  │                                                          │
  │  tcp_transmit_skb(sk, skb, ...)                          │
  │    │                                                     │
  │    ├── Build TCP header (seq, ack, flags, window, csum)  │
  │    ├── TCP options (timestamps, SACK, window scale)      │
  │    ├── Set skb->ip_summed = CHECKSUM_PARTIAL             │
  │    │                                                     │
  │    └── icsk->icsk_af_ops->queue_xmit(sk, skb, &fl)      │
  │         = ip_queue_xmit()                                │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌─────────────────────── NETWORK (IP) ────────────────────┐
  │                                                          │
  │  ip_queue_xmit(sk, skb, fl)                              │
  │    │                                                     │
  │    ├── Route lookup: ip_route_output_flow()              │
  │    │   → dst_entry attached to skb->_skb_refdst          │
  │    │                                                     │
  │    ├── Build IP header (version, IHL, TOS, TTL, proto,   │
  │    │   src IP, dst IP, ID)                               │
  │    │                                                     │
  │    └── ip_local_out(net, sk, skb)                        │
  │          │                                               │
  │          ├── NF_INET_LOCAL_OUT  [netfilter: OUTPUT chain] │
  │          │                                               │
  │          └── dst_output(net, sk, skb)                    │
  │                = ip_output(net, sk, skb)                  │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌─────────────────────── OUTPUT PATH ─────────────────────┐
  │                                                          │
  │  ip_output(net, sk, skb)                                 │
  │    │                                                     │
  │    ├── NF_INET_POST_ROUTING  [netfilter: POSTROUTING]    │
  │    │   (SNAT happens here)                               │
  │    │                                                     │
  │    └── ip_finish_output(net, sk, skb)                    │
  │          │                                               │
  │          ├── Check MTU → fragment if needed               │
  │          │   ip_fragment() if skb->len > mtu && !GSO      │
  │          │                                               │
  │          └── ip_finish_output2(net, sk, skb)              │
  │                │                                         │
  │                ├── Neighbor lookup: __neigh_lookup()      │
  │                │   ARP resolution if needed               │
  │                │                                         │
  │                └── neigh_output(neigh, skb)               │
  │                      = neigh_hh_output() [cached L2 hdr]  │
  │                        or neigh_resolve_output() [ARP]    │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌─────────────────────── DEVICE/QDISC ────────────────────┐
  │                                                          │
  │  dev_queue_xmit(skb)                                     │
  │    │                                                     │
  │    ├── Validate: netif_running(), netif_carrier_ok()     │
  │    │                                                     │
  │    ├── TC: qdisc_enqueue(skb, qdisc)                    │
  │    │   (fq_codel, HTB, etc.)                             │
  │    │                                                     │
  │    ├── GSO: if needed, segment here                      │
  │    │   dev_hard_start_xmit() per segment                 │
  │    │                                                     │
  │    └── ndo_start_xmit(skb, dev)                          │
  │         [driver's transmit function]                     │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌─────────────────────── DRIVER ──────────────────────────┐
  │                                                          │
  │  driver_start_xmit(skb, netdev)                          │
  │    │                                                     │
  │    ├── Map skb data for DMA:                             │
  │    │   dma_map_single() or dma_map_page()                │
  │    │                                                     │
  │    ├── Fill TX descriptor:                                │
  │    │   desc->addr = dma_addr                             │
  │    │   desc->len  = skb->len                             │
  │    │   desc->cmd  = TX_CMD (TSO, csum offload flags)     │
  │    │                                                     │
  │    ├── wmb()  ← ensure descriptor visible before notify  │
  │    │                                                     │
  │    ├── Ring doorbell: writel(tail, hw_reg)                │
  │    │   [NIC DMA engine starts reading descriptors]       │
  │    │                                                     │
  │    └── TX completion (interrupt or polling):              │
  │        dma_unmap_*()                                     │
  │        dev_consume_skb_any(skb)   ← free skb             │
  │        netdev_tx_completed_queue() ← BQL update          │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌─────────────────────── HARDWARE ────────────────────────┐
  │                                                          │
  │  NIC processes TX descriptor:                            │
  │    1. DMA read packet data from host memory              │
  │    2. Compute L4 checksum (if offloaded)                 │
  │    3. TSO segmentation (if offloaded)                    │
  │    4. Prepend preamble, SFD                              │
  │    5. Transmit on wire                                   │
  │    6. Write completion status to descriptor              │
  │    7. Raise TX completion interrupt (if enabled)         │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

---

## 28.2 Complete RX Flow: Wire to recv()

```
  ┌─────────────────────── HARDWARE ────────────────────────┐
  │                                                          │
  │  NIC receives frame:                                     │
  │    1. Check FCS (CRC32) → drop if bad                   │
  │    2. Check destination MAC (unicast/multicast/promisc)  │
  │    3. RSS hash → select RX queue                         │
  │    4. DMA write packet to pre-posted buffer              │
  │    5. Write RX descriptor (length, status, checksum)     │
  │    6. Raise interrupt (or coalesce)                      │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌─────────────────────── INTERRUPT HANDLER ───────────────┐
  │                                                          │
  │  driver_irq_handler(irq, dev_id)                         │
  │    │                                                     │
  │    ├── Disable further interrupts for this queue          │
  │    │                                                     │
  │    └── napi_schedule(&adapter->napi)                     │
  │         ├── Set NAPI_STATE_SCHED                         │
  │         └── __raise_softirq_irqoff(NET_RX_SOFTIRQ)      │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌─────────────────────── SOFTIRQ / NAPI ──────────────────┐
  │                                                          │
  │  net_rx_action(softirq)                                  │
  │    │                                                     │
  │    └── napi_poll(napi, budget=64)                        │
  │          │                                               │
  │          └── driver_poll(napi, budget)                    │
  │                │                                         │
  │                ├── [For each completed RX descriptor]:    │
  │                │   dma_unmap_*()                          │
  │                │   skb = napi_alloc_skb(napi, len)        │
  │                │   memcpy or page_ref to skb              │
  │                │   skb->protocol = eth_type_trans(skb, dev)│
  │                │   napi_gro_receive(napi, skb)            │
  │                │                                         │
  │                ├── If budget exhausted: return budget     │
  │                │   (NAPI stays scheduled, poll again)     │
  │                │                                         │
  │                └── If ring empty: napi_complete_done()    │
  │                    Re-enable interrupts                   │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌─────────────────────── GRO ─────────────────────────────┐
  │                                                          │
  │  napi_gro_receive(napi, skb)                             │
  │    │                                                     │
  │    ├── Try to merge with existing GRO flow:              │
  │    │   Same {src MAC, protocol, flow hash}               │
  │    │   Sequential TCP data                               │
  │    │   → Merge: extend skb, update length                │
  │    │                                                     │
  │    └── If no merge or flush needed:                      │
  │        napi_skb_finish() → netif_receive_skb()           │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌─────────────────────── NETIF_RECEIVE_SKB ───────────────┐
  │                                                          │
  │  netif_receive_skb(skb)                                  │
  │    │                                                     │
  │    ├── RPS: If enabled, enqueue to target CPU             │
  │    │   enqueue_to_backlog() → IPI                        │
  │    │                                                     │
  │    └── __netif_receive_skb_core(skb)                     │
  │          │                                               │
  │          ├── ptype_all: deliver to packet sniffers        │
  │          │   (tcpdump, AF_PACKET)                         │
  │          │                                               │
  │          ├── TC ingress: if ingress qdisc attached        │
  │          │   sch_handle_ingress()                         │
  │          │                                               │
  │          ├── Bridge: if port is bridge member             │
  │          │   br_handle_frame()                            │
  │          │                                               │
  │          └── ptype_base: dispatch by skb->protocol        │
  │              e.g., ETH_P_IP → ip_rcv()                   │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌─────────────────────── NETWORK (IP) ────────────────────┐
  │                                                          │
  │  ip_rcv(skb, dev, ptype, orig_dev)                       │
  │    │                                                     │
  │    ├── Validate: version, IHL, total length, checksum    │
  │    │                                                     │
  │    ├── NF_INET_PRE_ROUTING  [netfilter: PREROUTING]      │
  │    │   (DNAT and conntrack happen here)                   │
  │    │                                                     │
  │    └── ip_rcv_finish(net, sk, skb)                       │
  │          │                                               │
  │          ├── Route lookup: ip_route_input_slow()          │
  │          │   Determine: local delivery or forward         │
  │          │                                               │
  │          ├── [If local]: ip_local_deliver(skb)            │
  │          │     NF_INET_LOCAL_IN [INPUT chain]             │
  │          │     ip_local_deliver_finish()                  │
  │          │     → protocol dispatch (TCP/UDP/ICMP)         │
  │          │                                               │
  │          └── [If forward]: ip_forward(skb)                │
  │                TTL--, NF_INET_FORWARD, ip_output()        │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
         │ (local delivery path)
         ▼
  ┌─────────────────────── TRANSPORT (TCP) ─────────────────┐
  │                                                          │
  │  tcp_v4_rcv(skb)                                         │
  │    │                                                     │
  │    ├── Lookup socket: __inet_lookup_skb()                │
  │    │   Hash on {src IP, src port, dst IP, dst port}      │
  │    │                                                     │
  │    ├── [SYN to listener]: tcp_v4_do_rcv()                │
  │    │   → tcp_rcv_state_process() → tcp_v4_conn_request() │
  │    │   → Create request_sock, send SYN-ACK               │
  │    │                                                     │
  │    ├── [Established]: tcp_v4_do_rcv()                    │
  │    │   → tcp_rcv_established() [fast path]               │
  │    │     ├── Validate: seq, ack, window                  │
  │    │     ├── Process ACK: tcp_ack()                      │
  │    │     │   Free acknowledged data, advance snd_una     │
  │    │     │   Congestion control update                   │
  │    │     ├── Queue data: tcp_queue_rcv()                 │
  │    │     │   Add to sk->sk_receive_queue                 │
  │    │     └── Wake process: sk->sk_data_ready()           │
  │    │         → sock_def_readable() → wake_up()           │
  │    │                                                     │
  │    └── [Other states]: tcp_rcv_state_process()           │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌─────────────────────── SOCKET LAYER ────────────────────┐
  │                                                          │
  │  Process woken up, calls recv(fd, buf, len, 0):          │
  │    │                                                     │
  │    └── tcp_recvmsg(sk, msg, len, flags)                  │
  │          │                                               │
  │          ├── skb_queue_walk(sk->sk_receive_queue, skb)   │
  │          │   skb_copy_datagram_msg(skb, offset, msg, len)│
  │          │   [Copy data from kernel skb to user buffer]  │
  │          │                                               │
  │          ├── Update TCP receive window                   │
  │          │   tcp_rcv_space_adjust()                      │
  │          │                                               │
  │          └── Free consumed skbs                          │
  │              __kfree_skb() or skb_unlink()               │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
         │
         ▼
  ┌─────────────────────── USER SPACE ──────────────────────┐
  │                                                          │
  │  recv() returns with data in user buffer                 │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

---

## 28.3 TCP Three-Way Handshake Flow

```
  Client                              Server
  ──────                              ──────
  connect(fd, addr, len)              listen(fd, backlog)
    │                                 accept(fd, addr, len) [blocks]
    ▼
  tcp_v4_connect()
    inet_hash_connect() → ephemeral port
    tcp_connect() → build SYN skb
    tcp_transmit_skb() → send SYN
  sk state: SYN_SENT
                         ─── SYN (seq=X) ───►
                                               tcp_v4_rcv()
                                               tcp_v4_do_rcv()
                                               tcp_rcv_state_process()
                                               tcp_v4_conn_request()
                                                 Create request_sock
                                                 Add to SYN queue
                                                 tcp_v4_send_synack()
                         ◄── SYN-ACK (seq=Y, ack=X+1) ──
  tcp_rcv_state_process()
  tcp_ack() → mark SYN acked
  tcp_finish_connect()
  sk state: ESTABLISHED
  tcp_send_ack() → send ACK
                         ─── ACK (ack=Y+1) ──►
                                               tcp_check_req()
                                               inet_csk_complete_hashdance()
                                                 Create child socket
                                                 Move to accept queue
                                               sk state: ESTABLISHED
                                               accept() returns new fd

  SYN Queue (half-open):   request_sock entries, limited by tcp_max_syn_backlog
  Accept Queue (complete):  Full sockets waiting for accept(), limited by somaxconn
  SYN flood protection:     SYN cookies (tcp_syncookies=1)
```

---

## 28.4 TCP Connection Teardown Flow

```
  Active Close                         Passive Close
  ────────────                         ─────────────
  close(fd)
    tcp_close()
    tcp_send_fin()
  sk state: FIN_WAIT_1
                         ─── FIN (seq=M) ───►
                                               tcp_rcv_established()
                                               tcp_fin()
                                               sk state: CLOSE_WAIT
                                               Wake user (read returns 0)
                         ◄── ACK (ack=M+1) ──
  sk state: FIN_WAIT_2
                                               User calls close(fd)
                                               tcp_close()
                                               tcp_send_fin()
                         ◄── FIN (seq=N) ──
  tcp_fin()
  tcp_send_ack()
  sk state: TIME_WAIT
                         ─── ACK (ack=N+1) ──►
                                               sk state: CLOSED
  [Wait 2*MSL = 60s]
  sk state: CLOSED

  TIME_WAIT purpose:
    1. Ensure final ACK reaches peer (retransmit if lost)
    2. Prevent old segments from previous connection
       being accepted by new connection on same 4-tuple
  Duration: 2 * MSL (Maximum Segment Lifetime) = 60 seconds
  Optimization: tcp_tw_reuse=1 allows reuse for outgoing connections
```

---

## 28.5 ARP Resolution Flow

```
  Happens when ip_finish_output2() needs L2 header but
  neighbor entry is not resolved.

  ┌─────────────────── ARP Resolution ──────────────────────┐
  │                                                          │
  │  ip_finish_output2()                                     │
  │    neigh_output(neigh, skb)                              │
  │      → neigh->state == NUD_NONE or NUD_INCOMPLETE        │
  │                                                          │
  │  neigh_resolve_output(neigh, skb)                        │
  │    ├── Queue skb in neigh->arp_queue (max 3 packets)     │
  │    └── neigh_event_send() → arp_solicit()                │
  │                                                          │
  │  arp_send_dst()                                          │
  │    Build ARP REQUEST:                                     │
  │    Who has 10.0.0.2? Tell 10.0.0.1                        │
  │    Broadcast to ff:ff:ff:ff:ff:ff                         │
  │                                                          │
  │  [Target responds with ARP REPLY]                         │
  │                                                          │
  │  arp_rcv() → arp_process()                               │
  │    neigh_update(neigh, mac_addr, NUD_REACHABLE)          │
  │    → Flush queued skbs through the now-resolved neighbor  │
  │    → neigh_hh_init() → cache L2 header for fast path     │
  │                                                          │
  │  Subsequent packets:                                      │
  │    neigh_hh_output() → prepend cached L2 header (fast)   │
  │                                                          │
  │  Neighbor states:                                         │
  │    NUD_NONE → NUD_INCOMPLETE → NUD_REACHABLE              │
  │    → NUD_STALE → NUD_DELAY → NUD_PROBE → NUD_REACHABLE   │
  │    (or → NUD_FAILED if no response)                       │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

---

## 28.6 DNS Resolution + HTTP Request (Full Stack)

```
Application: curl http://example.com/page

Step 1: DNS Resolution
  getaddrinfo("example.com", "80")
    → socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
    → sendto(dns_fd, query, len, 0, &dns_server, ...)
    [Full TX path: UDP → IP → ARP → NIC → wire]
    → recvfrom(dns_fd, response, ...) → returns IP: 93.184.216.34
    [Full RX path: wire → NIC → NAPI → IP → UDP → socket]

Step 2: TCP Connection
  socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)
  connect(tcp_fd, {93.184.216.34:80}, ...)
    → SYN → SYN-ACK → ACK [see Section 28.3]

Step 3: HTTP Request
  send(tcp_fd, "GET /page HTTP/1.1\r\nHost: example.com\r\n\r\n", ...)
    → tcp_sendmsg() → tcp_write_xmit() → ip_queue_xmit()
    → ip_output() → neigh_output() → dev_queue_xmit()
    → ndo_start_xmit() → [NIC DMA → wire]

Step 4: HTTP Response
  [wire → NIC DMA → interrupt → NAPI → GRO]
    → ip_rcv() → NF_PRE_ROUTING → ip_local_deliver()
    → tcp_v4_rcv() → tcp_rcv_established()
    → queue to sk_receive_queue → wake process
  recv(tcp_fd, buf, ...) → "HTTP/1.1 200 OK\r\n..."

Step 5: Connection Close
  close(tcp_fd)
    → FIN → ACK → FIN → ACK [see Section 28.4]

Total kernel functions touched for one HTTP request: ~150+
```

---

## Interview Questions

**Q1: Trace the path of a TCP packet from send() to the wire.**
A: send() → syscall → sock_sendmsg() → inet_sendmsg() → tcp_sendmsg() (copy to skb, push) → tcp_write_xmit() → tcp_transmit_skb() (build TCP header, checksum) → ip_queue_xmit() (route lookup, build IP header) → NF_LOCAL_OUT → ip_output() → NF_POST_ROUTING → ip_finish_output() (fragmentation if needed) → ip_finish_output2() (ARP/neighbor lookup, add L2 header) → dev_queue_xmit() → qdisc enqueue → GSO segmentation → ndo_start_xmit() → driver DMA maps skb, fills TX descriptor, rings doorbell → NIC DMA reads data, computes checksum, transmits.

**Q2: Trace the path of a received TCP packet from NIC to recv().**
A: NIC receives frame, checks CRC, DMA to pre-posted buffer, writes RX descriptor, raises interrupt → driver ISR disables interrupts, calls napi_schedule() → NET_RX_SOFTIRQ → net_rx_action() → driver's NAPI poll: DMA unmap, build skb, eth_type_trans(), napi_gro_receive() (merge with flow) → netif_receive_skb() → ptype_all (tcpdump), ptype_base dispatch → ip_rcv() (validate, PRE_ROUTING) → ip_local_deliver() (INPUT chain) → tcp_v4_rcv() → socket lookup → tcp_rcv_established() (validate seq/ack, process ACK, queue data) → wake process → recv() → tcp_recvmsg() copies from sk_receive_queue to user buffer.

**Q3: What happens during ARP resolution and how are packets queued?**
A: When ip_finish_output2() finds no resolved neighbor entry, neigh_resolve_output() queues the skb in neigh->arp_queue (max 3 packets) and triggers arp_solicit() to broadcast an ARP request. When the ARP reply arrives, arp_rcv() → arp_process() updates the neighbor entry to NUD_REACHABLE with the resolved MAC. The queued skbs are then flushed through the neighbor. A cached L2 header (hh_cache) is initialized for fast-path subsequent packets.

---

## Summary

- TX: send() → socket → TCP (segment, header) → IP (route, header) → netfilter → neighbor/ARP → qdisc → driver → NIC DMA
- RX: NIC DMA → IRQ → NAPI poll → GRO → netif_receive_skb → IP → netfilter → TCP → socket → recv()
- TCP handshake: SYN → SYN-ACK → ACK with SYN/accept queues
- TCP teardown: FIN → ACK → FIN → ACK with TIME_WAIT (2*MSL)
- ARP: neighbor queue → broadcast request → reply → flush queue → cache L2 header
- ~150+ kernel functions involved in a single HTTP request

---

Next: [Chapter 29 — Kernel Source Code Map, Glossary, and References](Chapter_29_Source_Glossary_References.md)
