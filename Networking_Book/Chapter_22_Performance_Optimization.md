# Chapter 22: Network Performance Optimization

## Learning Goals
- Understand zero-copy networking (sendfile, splice, io_uring)
- Know hardware offloads (TSO, GRO, checksum, RSS)
- Understand XDP for high-performance packet processing
- Know kernel tuning parameters for network performance

---

## 22.1 Performance Bottlenecks

```
Packet Processing Costs (approximate):
  ┌──────────────────────────────────────────────────────┐
  │ Operation              │ Cycles   │ Impact            │
  ├────────────────────────┼──────────┼───────────────────┤
  │ Per-packet overhead    │ 1000-3000│ CPU bound at Mpps │
  │ Context switch         │ 2000-5000│ syscall overhead   │
  │ Cache miss (L3)        │ 100-200  │ Data structure     │
  │ TLB miss               │ 200-500  │ Large data buffers │
  │ Memory copy (1500B)    │ 500-1000 │ Data path copies   │
  │ Interrupt handling     │ 1000-2000│ Per-interrupt cost  │
  │ Lock contention        │ variable │ Serialization      │
  └────────────────────────┴──────────┴───────────────────┘

  Key bottlenecks:
  1. CPU: per-packet processing overhead
  2. Memory bandwidth: data copies
  3. Cache: poor locality in packet processing
  4. Interrupts: high rates cause livelock
  5. Locking: contention on shared structures
  6. Syscalls: kernel-user transitions
```

---

## 22.2 Zero-Copy Techniques

```
Goal: Eliminate unnecessary memory copies between kernel and userspace.

Traditional send (4 copies):
  User buffer → kernel buffer → socket buffer → NIC DMA
  (read from disk: disk → page cache → user → kernel → NIC)

sendfile(2) — 2 copies, no user-kernel crossing:
  sendfile(out_fd, in_fd, offset, count)

  Page cache → NIC DMA (with SG-DMA: truly zero-copy)
  ┌──────────┐     ┌───────────┐     ┌──────┐
  │ Disk/    │ DMA │ Page      │ DMA │ NIC  │
  │ Storage  │────►│ Cache     │────►│      │
  └──────────┘     └───────────┘     └──────┘
  No user-space copy needed.
  Used by: nginx, Apache for static files

splice(2) / tee(2):
  Move data between two file descriptors via pipe buffer
  splice(fd_in, off_in, fd_out, off_out, len, flags)
  No data copy — just moves buffer references

  Proxy pattern:
    splice(client_fd, NULL, pipe_write, NULL, len, SPLICE_F_MOVE);
    splice(pipe_read, NULL, server_fd, NULL, len, SPLICE_F_MOVE);

MSG_ZEROCOPY (SO_ZEROCOPY):
  setsockopt(fd, SOL_SOCKET, SO_ZEROCOPY, &one, sizeof(one));
  send(fd, buf, len, MSG_ZEROCOPY);
  // Kernel maps user pages directly into skb
  // Completion notification via error queue (SO_EE_ORIGIN_ZEROCOPY)

  Caveats:
  - Only saves copies for large messages (>10KB)
  - Notification overhead for small messages
  - Pages pinned until TX complete

io_uring:
  Async I/O framework with shared ring buffers
  Reduces syscall overhead via submission/completion queues
  Supports registered buffers for true zero-copy
```

---

## 22.3 Hardware Offloads

```
Modern NICs offload processing from CPU to hardware.

Checksum Offload:
  TX: NETIF_F_HW_CSUM / NETIF_F_IP_CSUM
    Kernel sets skb->ip_summed = CHECKSUM_PARTIAL
    NIC computes L4 checksum during DMA
  RX: CHECKSUM_UNNECESSARY / CHECKSUM_COMPLETE
    NIC verifies checksum, kernel skips

TSO (TCP Segmentation Offload):
  Kernel sends large skb (up to 64KB) to NIC
  NIC segments into MSS-sized packets with correct headers
  ┌────────────────────────────────────┐
  │  Without TSO:                      │
  │    CPU creates 44 packets (64KB)   │
  │    44 × header + segmentation cost │
  │                                    │
  │  With TSO:                         │
  │    CPU creates 1 super-packet      │
  │    NIC creates 44 packets in HW    │
  │    1 × header creation cost        │
  └────────────────────────────────────┘
  NETIF_F_TSO, NETIF_F_TSO6, NETIF_F_TSO_ECN

GSO (Generic Segmentation Offload):
  Software TSO — segments in kernel just before driver
  Fallback when hardware doesn't support TSO
  Still saves per-packet overhead in upper layers

GRO (Generic Receive Offload):
  Merge small received packets into larger skbs
  Reverse of TSO — reduces per-packet processing on RX
  Merge criteria: same flow, sequential data, same headers

LRO (Large Receive Offload):
  Hardware RX merging — breaks with forwarding/bridging
  GRO preferred (software, protocol-aware)

Scatter-Gather I/O:
  NETIF_F_SG: DMA from non-contiguous memory regions
  Avoids copying to contiguous buffer

ethtool control:
  ethtool -K eth0 tso on gro on tx on rx on sg on
  ethtool -k eth0    # Show current offload status
```

---

## 22.4 Scaling Techniques

```
Distribute packet processing across multiple CPUs.

RSS (Receive Side Scaling):
  NIC hashes incoming packets → distributes to RX queues
  Each queue → separate CPU → parallel processing
  Hash: Toeplitz on 5-tuple (src/dst IP, src/dst port, proto)
  ethtool -N eth0 rx-flow-hash tcp4 sdfn

RPS (Receive Packet Steering):
  Software RSS — hash in kernel softirq
  Useful when NIC has fewer queues than CPUs
  echo f > /sys/class/net/eth0/queues/rx-0/rps_cpus

RFS (Receive Flow Steering):
  Direct packets to CPU where application runs
  Reduces cache misses from cross-CPU processing
  echo 32768 > /proc/sys/net/core/rps_sock_flow_entries
  echo 2048 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt

XPS (Transmit Packet Steering):
  Map TX queues to CPUs
  Reduces lock contention on TX queues
  echo 1 > /sys/class/net/eth0/queues/tx-0/xps_cpus

aRFS (Accelerated RFS):
  Hardware flow steering to application CPU
  NIC programs flow rules directly
  ethtool -K eth0 ntuple on
```

---

## 22.5 Socket and Buffer Tuning

```bash
# === Socket buffer sizes ===
# Default and max socket buffer sizes
sysctl -w net.core.rmem_default=262144
sysctl -w net.core.wmem_default=262144
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216

# TCP auto-tuning: min, default, max (bytes)
sysctl -w net.ipv4.tcp_rmem="4096 131072 16777216"
sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"

# === Backlog and queue sizes ===
sysctl -w net.core.netdev_max_backlog=10000     # Per-CPU backlog
sysctl -w net.core.somaxconn=4096                # Listen backlog
sysctl -w net.ipv4.tcp_max_syn_backlog=8192     # SYN queue

# === TCP performance ===
sysctl -w net.ipv4.tcp_window_scaling=1
sysctl -w net.ipv4.tcp_timestamps=1
sysctl -w net.ipv4.tcp_sack=1
sysctl -w net.ipv4.tcp_fastopen=3               # Client + server
sysctl -w net.ipv4.tcp_congestion_control=bbr   # BBR algorithm

# === Interrupt coalescing ===
ethtool -C eth0 rx-usecs 50 rx-frames 64
# Coalesce: delay interrupt until N usecs or N frames
# Trade latency for throughput

# === Ring buffer sizes ===
ethtool -G eth0 rx 4096 tx 4096
# Larger rings → fewer drops under burst
# But more memory and higher latency
```

---

## 22.6 XDP (eXpress Data Path)

```
XDP: Programmable packet processing at the earliest point in RX.
Runs eBPF program on packet BEFORE sk_buff allocation.

Position in stack:
  [NIC driver DMA] → XDP program → normal stack (or drop/redirect)

  Without XDP:                    With XDP:
  NIC → alloc skb → protocol     NIC → XDP decision → (maybe alloc skb)
  stack → netfilter → socket     
                                  10-20x faster for simple decisions

XDP actions:
  XDP_PASS:      Continue to normal stack
  XDP_DROP:      Drop packet immediately (no skb allocation)
  XDP_TX:        Transmit back out same NIC
  XDP_REDIRECT:  Send to different NIC, CPU, or AF_XDP socket
  XDP_ABORTED:   Error, drop + trace

Example XDP program (drop all UDP):
```

```c
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int xdp_filter(struct xdp_md *ctx)
{
    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_PASS;

    if (eth->h_proto != __constant_htons(ETH_P_IP))
        return XDP_PASS;

    struct iphdr *iph = (void *)(eth + 1);
    if ((void *)(iph + 1) > data_end)
        return XDP_PASS;

    if (iph->protocol == IPPROTO_UDP)
        return XDP_DROP;

    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

```
Loading XDP program:
  ip link set dev eth0 xdp obj xdp_filter.o sec xdp

XDP modes:
  Native (driver):  XDP runs in driver NAPI poll (fastest)
  Generic (SKB):    XDP runs on skb (fallback, slower)
  Hardware (HW):    XDP offloaded to NIC (fastest, limited)

AF_XDP: Socket type for zero-copy packet I/O
  XDP_REDIRECT to AF_XDP socket
  UMEM: shared memory region between kernel and userspace
  Rings: RX, TX, FILL, COMPLETION
  Performance: 20-30 Mpps on modern hardware
```

---

## 22.7 Performance Monitoring

```bash
# === Packet counters ===
cat /proc/net/dev                    # Interface statistics
ethtool -S eth0                      # Driver statistics (drops, errors)
nstat -az                            # Kernel network statistics

# === Key metrics to watch ===
# Drops:
cat /proc/net/softnet_stat
# Column 1: packets processed
# Column 2: drops (backlog full) ← BAD if non-zero
# Column 3: time_squeeze (budget exhausted) ← OK if occasional

# TCP metrics:
ss -ti  # Per-connection TCP info (RTT, cwnd, retransmits)
nstat TcpRetransSegs  # Total retransmissions

# === Tracing tools ===
perf top -e cycles                   # CPU hotspots
perf record -g -a -- sleep 5         # Record profile
flamegraph                           # Visualize

# bpftrace one-liners:
bpftrace -e 'kprobe:tcp_sendmsg { @bytes = hist(arg2); }'
bpftrace -e 'tracepoint:net:netif_receive_skb { @[comm] = count(); }'
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/ipv4/tcp.c | Zero-copy TX (MSG_ZEROCOPY) |
| fs/splice.c | splice implementation |
| net/core/dev.c | NAPI, GRO, XDP generic |
| drivers/net/ethernet/*/xdp* | Driver XDP support |
| net/xdp/xdp_umem.c | AF_XDP UMEM |
| net/core/xdp.c | XDP core framework |
| kernel/bpf/verifier.c | BPF verifier |
| include/net/xdp.h | XDP definitions |

---

## Interview Questions

**Q1: What is zero-copy networking and what techniques does Linux provide?**
A: Zero-copy avoids copying data between kernel and user space. Techniques: (1) sendfile() — transfers file data directly from page cache to socket/NIC. (2) splice() — moves data between fds via pipe without copying. (3) MSG_ZEROCOPY — kernel maps user buffers directly into skbs. (4) AF_XDP — shared memory ring buffers between kernel and userspace. (5) io_uring with registered buffers. Trade-offs: zero-copy adds notification/management overhead, so it's only beneficial for large transfers.

**Q2: Explain TSO vs GSO vs GRO.**
A: **TSO** (TX): Kernel sends one large segment (up to 64KB) to the NIC; hardware splits into MSS-sized packets with correct TCP headers. Saves CPU. **GSO** (TX): Software fallback for TSO — kernel delays segmentation until just before the driver, still saving per-packet overhead in the stack. **GRO** (RX): Merges multiple received packets of the same flow into one large skb before passing up the stack — reverse of TSO, reduces per-packet processing cost.

**Q3: How does XDP achieve high performance?**
A: XDP runs an eBPF program at the earliest point in the receive path — inside the driver's NAPI poll, before sk_buff allocation. This means: (1) No sk_buff allocation overhead for dropped/redirected packets. (2) Packet data is still in the DMA buffer (cache-hot). (3) Simple decisions (drop, redirect, TX) avoid the entire protocol stack. (4) eBPF programs are JIT-compiled to native code. (5) Per-CPU processing, no locking. Result: 10-20x throughput for packet filtering, reaching 20+ Mpps per core.

**Q4: What sysctls would you tune for a high-throughput server?**
A: (1) Increase tcp_rmem/tcp_wmem max for large BDP connections. (2) Set tcp_congestion_control to BBR for better throughput. (3) Enable tcp_fastopen for reduced latency. (4) Increase netdev_max_backlog for burst absorption. (5) Increase somaxconn and tcp_max_syn_backlog for high connection rates. (6) Enable window_scaling, timestamps, sack for TCP performance. (7) Tune interrupt coalescing (ethtool -C) and ring buffer size (ethtool -G). (8) Enable offloads: TSO, GRO, checksum (ethtool -K).

---

## Summary

- Zero-copy: sendfile, splice, MSG_ZEROCOPY, AF_XDP eliminate data copies
- Hardware offloads: TSO/GSO (TX segmentation), GRO (RX aggregation), checksum
- Scaling: RSS (HW), RPS/RFS (SW), XPS (TX) distribute load across CPUs
- XDP: eBPF at driver level, before skb allocation — 10-20x throughput for filtering
- Socket tuning: buffer sizes, backlog, TCP options (BBR, fastopen, SACK)
- Monitor: /proc/net/softnet_stat for drops, ethtool -S for driver stats

---

Next: [Chapter 23 — High-Performance Networking](Chapter_23_High_Performance.md)
