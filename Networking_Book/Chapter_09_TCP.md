# Chapter 9: Transport Layer — TCP Deep Dive

## Learning Goals
- Understand TCP state machine and all state transitions
- Know the 3-way handshake and 4-way teardown in kernel detail
- Master TCP congestion control algorithms
- Understand TCP reliability: retransmission, SACK, fast recovery
- Know TCP performance features: window scaling, timestamps, MSS

---

## 9.1 TCP Architecture in Linux

```
┌─────────────────────────────────────────────────────────────┐
│                    TCP Implementation                       │
│                                                             │
│  net/ipv4/tcp.c          ← Core: sendmsg, recvmsg          │
│  net/ipv4/tcp_output.c   ← Segment building, transmission  │
│  net/ipv4/tcp_input.c    ← Segment processing, ACK handling│
│  net/ipv4/tcp_ipv4.c     ← IPv4-specific: tcp_v4_rcv       │
│  net/ipv4/tcp_timer.c    ← Retransmission, keepalive timers│
│  net/ipv4/tcp_cong.c     ← Congestion control framework    │
│  net/ipv4/tcp_cubic.c    ← CUBIC congestion algorithm      │
│  net/ipv4/tcp_bbr.c      ← BBR congestion algorithm        │
│  net/ipv4/tcp_fastopen.c ← TCP Fast Open                   │
│                                                             │
│  Key structures:                                            │
│    struct tcp_sock        ← TCP-specific socket state       │
│    struct tcp_skb_cb      ← Per-skb TCP metadata (in cb[]) │
│    struct inet_connection_sock ← Connection state           │
└─────────────────────────────────────────────────────────────┘
```

---

## 9.2 TCP Header Format

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |       |C|E|U|A|P|R|S|F|                              |
| Offset| Rsrvd |W|C|R|C|S|S|Y|I|         Window Size          |
|       |       |R|E|G|K|H|T|N|N|                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (variable)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Flags:
  SYN = Synchronize (connection setup)
  ACK = Acknowledgment
  FIN = Finish (connection teardown)
  RST = Reset (abort connection)
  PSH = Push (deliver immediately)
  URG = Urgent data
  ECE/CWR = Explicit Congestion Notification
```

---

## 9.3 TCP State Machine

```
                              ┌──────────────┐
                              │    CLOSED     │
                              └──────┬───────┘
                   ┌─────────────────┼─────────────────┐
          (server) │                 │                  │ (client)
      listen()     │           socket()           connect()
                   ▼                                   ▼
            ┌──────────────┐                    ┌──────────────┐
            │    LISTEN     │                    │   SYN_SENT   │
            └──────┬───────┘                    └──────┬───────┘
                   │ recv SYN                          │ recv SYN+ACK
                   │ send SYN+ACK                      │ send ACK
                   ▼                                   ▼
            ┌──────────────┐                    ┌──────────────┐
            │   SYN_RCVD   │                    │ ESTABLISHED  │
            └──────┬───────┘                    └──────────────┘
                   │ recv ACK                          │
                   ▼                                   │
            ┌──────────────┐                           │
            │ ESTABLISHED  │◄──────────────────────────┘
            └──────┬───────┘
                   │
        ┌──────────┴──────────┐
   close() (active)       recv FIN (passive)
        │                     │
        ▼                     ▼
 ┌──────────────┐      ┌──────────────┐
 │   FIN_WAIT_1 │      │  CLOSE_WAIT  │
 └──────┬───────┘      └──────┬───────┘
        │ recv ACK             │ close()
        ▼                     ▼
 ┌──────────────┐      ┌──────────────┐
 │   FIN_WAIT_2 │      │   LAST_ACK   │
 └──────┬───────┘      └──────┬───────┘
        │ recv FIN             │ recv ACK
        │ send ACK             ▼
        ▼               ┌──────────────┐
 ┌──────────────┐       │    CLOSED     │
 │   TIME_WAIT  │       └──────────────┘
 │  (2×MSL wait)│
 └──────┬───────┘
        │ timeout (60s)
        ▼
 ┌──────────────┐
 │    CLOSED     │
 └──────────────┘
```

---

## 9.4 Three-Way Handshake

```
    Client                                    Server
      │                                         │
      │──────── SYN (seq=x) ────────────────►│ tcp_v4_rcv() → tcp_v4_do_rcv()
      │  tcp_v4_connect()                       │ → tcp_v4_conn_request()
      │  tcp_connect()                          │ → SYN cookie or SYN queue
      │  State: SYN_SENT                        │ State: SYN_RCVD
      │                                         │
      │◄─────── SYN+ACK (seq=y, ack=x+1) ──────│ tcp_v4_send_synack()
      │  tcp_rcv_synsent_state_process()        │
      │                                         │
      │──────── ACK (ack=y+1) ──────────────►│ tcp_check_req()
      │  tcp_send_ack()                         │ → promote to ESTABLISHED
      │  State: ESTABLISHED                     │ → move to accept queue
      │                                         │ State: ESTABLISHED
      │                                         │
      │          DATA TRANSFER                  │
```

### SYN Queue and Accept Queue

```
┌──────────────────────────────────────────────────────┐
│ Listening Socket                                     │
│                                                      │
│  SYN Queue (half-open connections):                  │
│  ┌─────┬─────┬─────┐                                │
│  │ req │ req │ req │  ← SYN received, SYN+ACK sent  │
│  └─────┴─────┴─────┘    waiting for final ACK        │
│  Size: net.ipv4.tcp_max_syn_backlog (default 256)    │
│                                                      │
│  Accept Queue (fully established):                   │
│  ┌─────┬─────┬─────┐                                │
│  │ sk  │ sk  │ sk  │  ← 3-way handshake complete     │
│  └─────┴─────┴─────┘    waiting for accept()         │
│  Size: listen(fd, backlog) argument                  │
│                                                      │
│  SYN Flood Protection:                               │
│    SYN cookies: encode state in SEQ number           │
│    No SYN queue entry needed                         │
│    net.ipv4.tcp_syncookies = 1                       │
└──────────────────────────────────────────────────────┘
```

---

## 9.5 TCP Congestion Control

```
TCP Congestion Control Variables:
  cwnd   = congestion window (sender limit, in segments)
  rwnd   = receiver window (advertised by receiver)
  ssthresh = slow start threshold

  Actual send window = min(cwnd, rwnd)

Phases:
  ┌─────────────────────────────────────────────────────┐
  │                                                     │
  │  Slow Start          │  Congestion Avoidance        │
  │  cwnd < ssthresh     │  cwnd >= ssthresh            │
  │                      │                              │
  │  For each ACK:       │  For each RTT:               │
  │    cwnd += 1 MSS     │    cwnd += 1 MSS             │
  │  (exponential)       │  (linear, additive increase) │
  │                      │                              │
  └──────────────────────┴──────────────────────────────┘

  On packet loss:
    ssthresh = cwnd / 2
    cwnd = 1 (Reno) or cwnd/2 (CUBIC)
```

### Congestion Control Algorithms in Linux

```
CUBIC (default since Linux 2.6.19):
  - Window growth as cubic function of time since last loss
  - Aggressive growth when far from previous cwnd
  - Gentle growth near previous cwnd
  - Good for high-BDP networks

BBR (Bottleneck Bandwidth and RTT, Google):
  - Model-based: estimates bottleneck bandwidth and min RTT
  - Does NOT use packet loss as congestion signal
  - Achieves higher throughput on lossy links
  - Phases: Startup, Drain, ProbeBW, ProbeRTT
  - sysctl net.ipv4.tcp_congestion_control = bbr

Reno:
  - Classic AIMD (Additive Increase, Multiplicative Decrease)
  - cwnd halved on loss
  - Slow recovery on high-BDP links

Available algorithms:
  cat /proc/sys/net/ipv4/tcp_available_congestion_control
  # cubic reno bbr

Per-socket override:
  setsockopt(fd, IPPROTO_TCP, TCP_CONGESTION, "bbr", 3);
```

```
CUBIC Growth Curve:

cwnd │                    ╭───────────
     │                 ╭──╯
     │              ╭──╯
     │           ╭──╯
     │        ╭──╯         ← Aggressive growth (far from Wmax)
     │      ╭─╯
     │     ╱               ← Gentle growth (near Wmax)
     │   ╱
     │  ╱ ← Fast recovery
     │╱    (cwnd = Wmax × beta)
     └──────────────────────────── time
         loss event
```

---

## 9.6 TCP Reliability Mechanisms

### Retransmission

```
Sender sends data:
  SEQ=1000, LEN=100   → Segment 1
  SEQ=1100, LEN=100   → Segment 2
  SEQ=1200, LEN=100   → Segment 3 (LOST!)
  SEQ=1300, LEN=100   → Segment 4

Receiver sends ACKs:
  ACK=1100  (got segment 1)
  ACK=1200  (got segment 2)
  ACK=1200  (duplicate ACK — segment 3 missing, got 4)
  ACK=1200  (duplicate ACK)
  ACK=1200  (duplicate ACK — 3rd dup ACK!)

Fast Retransmit:
  After 3 duplicate ACKs → retransmit segment 3 immediately
  (Don't wait for timeout)

RTO (Retransmission Timeout):
  Calculated from RTT measurements:
    SRTT = (1-α)×SRTT + α×RTT_sample    (α = 1/8)
    RTTVAR = (1-β)×RTTVAR + β×|SRTT-RTT| (β = 1/4)
    RTO = SRTT + max(G, 4×RTTVAR)
  
  Minimum RTO: 200ms (net.ipv4.tcp_rto_min, tunable via ip route)
  Maximum RTO: 120s
```

### SACK (Selective Acknowledgment)

```
Without SACK:
  ACK=1200 tells sender "I have everything before 1200"
  Sender doesn't know if 1300, 1400... were received

With SACK:
  ACK=1200, SACK=1300-1500
  Tells sender: "I'm missing 1200-1299, but I have 1300-1499"
  Sender retransmits only bytes 1200-1299

  SACK option in TCP header:
    Kind=5, Length=variable
    Left Edge, Right Edge (pairs of received ranges)

  net.ipv4.tcp_sack = 1 (enabled by default)
```

---

## 9.7 TCP Performance Features

### Window Scaling

```
Standard TCP window: 16-bit = max 65535 bytes
With 100ms RTT and 1Gbps link: BDP = 12.5 MB >> 64 KB

Window Scale option (RFC 7323):
  Negotiated in SYN/SYN+ACK
  Shift factor 0-14
  Effective window = window << shift
  Shift 7 → max window = 65535 × 128 = 8 MB

  net.ipv4.tcp_window_scaling = 1 (default)
```

### TCP Timestamps

```
TCP Timestamps option (RFC 7323):
  TSval (Timestamp Value from sender)
  TSecr (Timestamp Echo Reply)

  Uses:
  1. RTT measurement: TSecr in ACK - TSval when sent
  2. PAWS (Protection Against Wrapped Sequences):
     Rejects old segments on high-speed links where
     sequence numbers wrap within MSL
  
  net.ipv4.tcp_timestamps = 1 (default)
```

### Nagle Algorithm and TCP_NODELAY

```
Nagle Algorithm (default on):
  If there is unacknowledged data AND new data < MSS:
    Buffer new data until ACK arrives
  
  Problem: adds latency for small messages
  Solution: TCP_NODELAY disables Nagle
    setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &one, sizeof(one));

  TCP_CORK: cork sending until uncork or buffer full
    Useful for sendfile() header + body pattern
```

### TCP Fast Open (TFO)

```
Normal: 1 RTT for handshake + 1 RTT for first data = 2 RTT
TFO:    Client sends data in SYN → 1 RTT for first data

First connection:
  Client → SYN + TFO cookie request        → Server
  Client ← SYN+ACK + TFO cookie            ← Server

Subsequent connections:
  Client → SYN + data + TFO cookie          → Server
  Server validates cookie, processes data immediately
  Client ← SYN+ACK + response              ← Server

  net.ipv4.tcp_fastopen = 3 (enable client + server)
```

---

## 9.8 TCP in the Kernel

```c
/* Key TCP functions */

/* TX path */
tcp_sendmsg(sk, msg, size)
  → tcp_push(sk, flags)
    → tcp_write_xmit(sk, mss, nonagle)
      → tcp_transmit_skb(sk, skb, clone_it)
        → Build TCP header
        → ip_queue_xmit(sk, skb)

/* RX path */
tcp_v4_rcv(skb)
  → __inet_lookup_skb()     /* Find socket by 5-tuple */
  → tcp_v4_do_rcv(sk, skb)
    → tcp_rcv_established()  /* Fast path for ESTABLISHED */
      → tcp_event_data_recv() /* Update receive state */
      → tcp_data_queue(sk, skb) /* Queue to socket */
      → sk->sk_data_ready()    /* Wake up reader */
    OR
    → tcp_rcv_state_process()  /* Other states (SYN_SENT, etc.) */

/* Connection management */
tcp_v4_connect(sk, uaddr, addr_len)  /* Client connect */
tcp_v4_conn_request(sk, skb)         /* Server receives SYN */
tcp_check_req(sk, skb, req)          /* Complete 3-way handshake */
```

---

## 9.9 TCP Timers

```
┌─────────────────────────────────────────────────────────────┐
│ Timer              │ Purpose                    │ Default   │
├─────────────────────────────────────────────────────────────┤
│ Retransmit timer   │ Retransmit unACKed data    │ RTO-based │
│ Delayed ACK timer  │ Delay ACK to piggyback     │ 40ms max  │
│ Persist timer      │ Probe zero-window receiver │ varies    │
│ Keepalive timer    │ Detect dead connections     │ 2 hours   │
│ FIN_WAIT_2 timer   │ Timeout orphaned FIN_WAIT_2│ 60s       │
│ TIME_WAIT timer    │ 2×MSL wait after close     │ 60s       │
│ SYN-ACK timer      │ Retransmit SYN+ACK         │ 1s, exp.  │
└─────────────────────────────────────────────────────────────┘
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/ipv4/tcp.c | Core TCP: sendmsg, recvmsg |
| net/ipv4/tcp_input.c | Input processing, ACK, SACK |
| net/ipv4/tcp_output.c | Output: segment building, xmit |
| net/ipv4/tcp_ipv4.c | IPv4-specific: tcp_v4_rcv |
| net/ipv4/tcp_timer.c | All TCP timers |
| net/ipv4/tcp_cong.c | Congestion control framework |
| net/ipv4/tcp_cubic.c | CUBIC algorithm |
| net/ipv4/tcp_bbr.c | BBR algorithm |
| include/net/tcp.h | TCP structures and macros |
| include/uapi/linux/tcp.h | User-visible TCP definitions |

---

## Interview Questions

**Q1: Explain the TCP 3-way handshake at the kernel level.**
A: Client calls connect() → tcp_v4_connect() sends SYN with initial seq, sets state to SYN_SENT. Server's tcp_v4_rcv() → tcp_v4_conn_request() adds to SYN queue, sends SYN+ACK. Client's tcp_rcv_synsent_state_process() receives SYN+ACK, sends ACK, moves to ESTABLISHED. Server's tcp_check_req() receives ACK, promotes from SYN queue to accept queue, state ESTABLISHED. accept() dequeues from accept queue.

**Q2: What is TIME_WAIT and why does it exist?**
A: TIME_WAIT is entered by the side that initiates close (active closer) after receiving the final FIN+ACK. Duration: 2×MSL (60 seconds). Two purposes: (1) Ensure the final ACK reaches the peer — if lost, peer retransmits FIN and gets a new ACK. (2) Prevent old segments from a previous connection being confused with a new connection on the same 5-tuple. SO_REUSEADDR allows binding to a port in TIME_WAIT.

**Q3: Compare CUBIC and BBR congestion control.**
A: CUBIC is loss-based: cwnd grows as a cubic function of time since last loss event; on loss, cwnd is reduced. It's conservative and fair. BBR is model-based: it estimates bottleneck bandwidth and minimum RTT, then paces packets to match. BBR doesn't use loss as congestion signal, achieving higher throughput on lossy links (e.g., wireless). BBR sends at the estimated rate continuously, while CUBIC probes by increasing until loss.

**Q4: What is the SYN flood attack and how does Linux defend against it?**
A: SYN flood: attacker sends millions of SYN packets with spoofed source IPs. Server creates SYN queue entries for each, exhausting memory before accept queue fills. Defense: SYN cookies (tcp_syncookies=1). Instead of storing state in the SYN queue, the server encodes essential connection info (MSS, timestamps) in the initial sequence number of the SYN+ACK. When the ACK arrives, the server reconstructs the connection state from the sequence number. No queue entry needed.

**Q5: What happens when the receiver's socket buffer is full?**
A: The receiver advertises a zero window (window=0 in ACK). The sender stops transmitting and starts the persist timer. The persist timer sends window probes (tiny segments) periodically to check if the window has reopened. When the receiver frees buffer space (application calls recv()), it sends an ACK with the updated window. The sender resumes. This is TCP flow control via sliding window.

---

## Summary

- TCP uses an 11-state machine; key transitions involve SYN, FIN, and timeouts
- 3-way handshake: SYN → SYN+ACK → ACK with SYN/accept queue management
- Congestion control (CUBIC default, BBR alternative) regulates sending rate
- Reliability: retransmission timer, fast retransmit (3 dup ACKs), SACK
- Performance: window scaling, timestamps, TCP Fast Open, Nagle/NODELAY
- Multiple timers manage different aspects: retransmit, keepalive, TIME_WAIT

---

Next: [Chapter 10 — Transport Layer: UDP and Other Protocols](Chapter_10_UDP.md)
