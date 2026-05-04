# Chapter 27: Network Debugging Tools and Techniques

## Learning Goals
- Master tcpdump, Wireshark, and packet capture techniques
- Know kernel tracing tools for network debugging (ftrace, perf, bpftrace)
- Understand ss, netstat, and socket diagnostics
- Know systematic approaches to debug network issues

---

## 27.1 Packet Capture and Analysis

### tcpdump

```bash
# Basic capture
tcpdump -i eth0                         # All traffic on eth0
tcpdump -i any                          # All interfaces
tcpdump -nn -i eth0                     # No name/port resolution (faster)
tcpdump -c 100 -i eth0                  # Capture 100 packets then stop
tcpdump -w capture.pcap -i eth0         # Write to file
tcpdump -r capture.pcap                 # Read from file

# Filtering (BPF filter syntax)
tcpdump -i eth0 host 10.0.0.1           # To/from specific host
tcpdump -i eth0 src 10.0.0.1            # From specific source
tcpdump -i eth0 dst port 80             # To port 80
tcpdump -i eth0 tcp port 443            # TCP port 443
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'  # SYN packets
tcpdump -i eth0 'tcp[tcpflags] == tcp-syn'       # Only SYN (no SYN-ACK)
tcpdump -i eth0 icmp                    # ICMP only
tcpdump -i eth0 arp                     # ARP only
tcpdump -i eth0 vlan                    # VLAN tagged traffic

# Complex filters
tcpdump -i eth0 'src 10.0.0.0/8 and dst port 80 and tcp'
tcpdump -i eth0 'not port 22'           # Exclude SSH
tcpdump -i eth0 'tcp[13] & 2 != 0'     # SYN flag set (offset 13, bit 1)

# Verbose output
tcpdump -v    # Verbose (TTL, TOS, ID)
tcpdump -vv   # More verbose (TCP options)
tcpdump -vvv  # Maximum verbosity
tcpdump -X    # Hex + ASCII payload
tcpdump -XX   # Include link headers
tcpdump -e    # Show Ethernet headers

# Performance
tcpdump -B 4096                         # Buffer size (KB)
tcpdump -s 96                           # Snap length (capture first N bytes)
tcpdump -s 0                            # Capture full packet

# BPF (Berkeley Packet Filter):
# Compiled to efficient bytecode, runs in kernel
# Same filter language used by: tcpdump, Wireshark, scapy
```

### Wireshark / tshark

```bash
# tshark: Command-line Wireshark
tshark -i eth0                          # Live capture
tshark -r capture.pcap                  # Read file
tshark -Y 'tcp.port == 80'             # Display filter
tshark -Y 'http.request.method == GET' # HTTP GET requests
tshark -Y 'tcp.analysis.retransmission' # Retransmissions
tshark -Y 'dns.qry.name contains "example"'  # DNS queries
tshark -z conv,tcp                      # TCP conversation stats
tshark -z io,stat,1                     # 1-second traffic stats
tshark -T fields -e ip.src -e ip.dst   # Extract specific fields

# Wireshark display filters (different from capture filters):
tcp.stream eq 5                         # Follow TCP stream #5
tcp.analysis.duplicate_ack              # Duplicate ACKs
tcp.analysis.zero_window                # Zero window events
tcp.analysis.window_update              # Window updates
frame.time_delta > 1                    # Gaps > 1 second
```

---

## 27.2 Socket and Connection Diagnostics

```bash
# === ss (socket statistics, replaces netstat) ===
ss -tlnp          # TCP listening sockets with process names
ss -tunap         # All TCP/UDP with process info
ss -s             # Socket summary statistics
ss -i             # TCP internal info (cwnd, RTT, retransmits)
ss -m             # Memory usage per socket
ss -e             # Extended info (uid, inode, timers)

# Filtering
ss state established '( sport == 80 )'
ss dst 10.0.0.1
ss -o state time-wait    # Show TIME_WAIT sockets with timers

# Example output (ss -ti):
#   State  Recv-Q  Send-Q  Local:Port  Peer:Port
#   ESTAB  0       0       10.0.0.1:22  10.0.0.2:54321
#       cubic wscale:7,7 rto:204 rtt:0.5/0.25 ato:40
#       cwnd:10 ssthresh:65535 send 234.5Mbps rcvmss:1448

# === netstat (legacy) ===
netstat -tlnp         # TCP listening
netstat -s            # Protocol statistics
netstat -r            # Routing table (same as route -n)
netstat -i            # Interface statistics

# === ip command diagnostics ===
ip -s link show eth0  # Interface stats (TX/RX bytes, errors, drops)
ip -s -s link show    # Detailed error counters
ip neigh show         # ARP/neighbor table
ip route get 8.8.8.8  # Show route to destination

# === /proc/net ===
cat /proc/net/tcp     # All TCP connections (hex addresses)
cat /proc/net/udp     # All UDP sockets
cat /proc/net/dev     # Interface statistics
cat /proc/net/snmp    # SNMP MIB counters
cat /proc/net/netstat # Extended network statistics
cat /proc/net/sockstat # Socket memory usage
```

---

## 27.3 Network Performance Diagnostics

```bash
# === ping / traceroute ===
ping -c 10 -i 0.2 10.0.0.1      # 10 pings, 200ms interval
ping -f 10.0.0.1                  # Flood ping (needs root)
ping -s 1472 -M do 10.0.0.1     # MTU discovery (1472 + 28 = 1500)

traceroute 10.0.0.1              # UDP probes (default)
traceroute -T 10.0.0.1           # TCP SYN probes (bypasses firewalls)
traceroute -I 10.0.0.1           # ICMP probes
mtr 10.0.0.1                     # Continuous traceroute with stats

# === Throughput testing ===
# iperf3 server:
iperf3 -s

# iperf3 client (TCP):
iperf3 -c 10.0.0.1 -t 30 -P 4  # 30 sec, 4 parallel streams

# iperf3 client (UDP):
iperf3 -c 10.0.0.1 -u -b 1G    # UDP at 1 Gbps target

# === Latency profiling ===
hping3 -S -p 80 10.0.0.1        # TCP SYN latency
sockperf ping-pong --ip 10.0.0.1 # Microsecond-level latency

# === ethtool diagnostics ===
ethtool eth0                      # Link status, speed, duplex
ethtool -S eth0                   # NIC driver statistics
ethtool -S eth0 | grep -i drop   # Find NIC-level drops
ethtool -S eth0 | grep -i error  # Find NIC-level errors
ethtool -d eth0                   # Register dump
ethtool -i eth0                   # Driver info (name, version)
ethtool -g eth0                   # Ring buffer sizes
ethtool -c eth0                   # Interrupt coalescing settings
ethtool -k eth0                   # Offload settings

# === NIC queue statistics ===
cat /proc/net/softnet_stat
# Columns: processed, dropped, time_squeeze, (flow_limit), ...
# dropped > 0 → increase netdev_max_backlog
# time_squeeze > 0 → softirq budget exhausted → increase netdev_budget
```

---

## 27.4 Kernel Network Tracing

```bash
# === ftrace: function tracer ===
# Trace TCP transmit path:
cd /sys/kernel/tracing
echo function > current_tracer
echo 'tcp_sendmsg tcp_write_xmit ip_queue_xmit' > set_ftrace_filter
echo 1 > tracing_on
# ... generate traffic ...
echo 0 > tracing_on
cat trace

# === ftrace: function_graph ===
echo function_graph > current_tracer
echo tcp_sendmsg > set_graph_function
echo 1 > tracing_on
cat trace_pipe
# Shows call tree with timings

# === Network tracepoints ===
# List available:
perf list 'net:*' 'skb:*' 'tcp:*' 'sock:*'

# Key tracepoints:
#   net:netif_receive_skb  — packet received by stack
#   net:net_dev_xmit       — packet transmitted
#   net:napi_gro_receive_entry — GRO merging
#   skb:skb_copy_datagram_iovec — copy to userspace
#   tcp:tcp_probe          — TCP state/cwnd/ssthresh
#   tcp:tcp_retransmit_skb — TCP retransmission
#   sock:inet_sock_set_state — TCP state changes

# === perf events ===
perf stat -e 'net:*' -a -- sleep 5    # Count network events
perf record -e 'net:netif_receive_skb' -a -- sleep 5
perf script

# === bpftrace ===
# Count packets by interface:
bpftrace -e 'tracepoint:net:netif_receive_skb { @[str(args->name)] = count(); }'

# Histogram of TCP send sizes:
bpftrace -e 'kprobe:tcp_sendmsg { @size = hist(arg2); }'

# Track TCP retransmissions:
bpftrace -e 'tracepoint:tcp:tcp_retransmit_skb {
    printf("retransmit: %s:%d -> %s:%d\n",
           ntop(args->saddr), args->sport,
           ntop(args->daddr), args->dport);
}'

# TCP state transitions:
bpftrace -e 'tracepoint:sock:inet_sock_set_state {
    printf("%s -> %s: %s:%d\n",
           @state[args->oldstate], @state[args->newstate],
           ntop(args->daddr), args->dport);
}'

# === Packet drops ===
# dropwatch: traces skb_kfree / kfree_skb
dropwatch -l kas

# perf skb:kfree_skb:
perf record -e skb:kfree_skb -a -- sleep 10
perf script  # Shows where packets are dropped (function + stack)
```

---

## 27.5 Common Network Issues and Diagnosis

```
Issue: Connection timeouts
  Diagnosis:
    1. ping destination → reachable?
    2. traceroute → where does it stop?
    3. tcpdump → are SYN packets going out?
    4. tcpdump on remote → are SYN packets arriving?
    5. ss -tn → connection state (SYN_SENT stuck?)
    6. iptables -L -v -n → firewall blocking?
    7. conntrack -L → conntrack table full?

Issue: Slow throughput
  Diagnosis:
    1. iperf3 → measure actual bandwidth
    2. ethtool eth0 → check speed/duplex (100M half = problem)
    3. ethtool -S eth0 | grep drop → NIC drops?
    4. ss -ti → check cwnd, RTT, retransmits
    5. cat /proc/net/softnet_stat → softirq drops?
    6. tc -s qdisc show → qdisc drops/overlimits?
    7. dmesg → any NIC errors?
    8. ethtool -k eth0 → offloads enabled?

Issue: Packet loss
  Diagnosis:
    1. ping -c 1000 → loss percentage
    2. ethtool -S eth0 → rx_dropped, rx_errors
    3. /proc/net/softnet_stat col 2 → backlog drops
    4. netstat -s → IP/TCP/UDP errors
    5. dropwatch → where in kernel are packets dropped
    6. perf record -e skb:kfree_skb → precise drop location

Issue: High latency
  Diagnosis:
    1. ping → baseline RTT
    2. mtr → which hop adds latency
    3. ss -ti → RTT and variation
    4. tc -s qdisc → queueing delay (high queue depth)
    5. ethtool -c → interrupt coalescing too aggressive?
    6. /proc/interrupts → IRQ distribution (all on CPU 0?)

Debugging checklist:
  ┌────────────────────────────────────────────────────────┐
  │ Layer     │ Check                                      │
  ├───────────┼────────────────────────────────────────────┤
  │ Physical  │ ethtool (link, speed, errors), cable       │
  │ Link      │ arp -n, ip neigh, MAC table                │
  │ Network   │ ip route, ping, traceroute                 │
  │ Transport │ ss -ti, TCP retransmits, window size       │
  │ Firewall  │ iptables -L, nft list, conntrack           │
  │ QoS       │ tc -s qdisc/class show                     │
  │ NIC       │ ethtool -S, /proc/interrupts               │
  │ Kernel    │ /proc/net/softnet_stat, dmesg, dropwatch   │
  │ App       │ strace, gdb, application logs              │
  └───────────┴────────────────────────────────────────────┘
```

---

## 27.6 Network Namespaces Debugging

```bash
# Debug container/namespace networking:

# List namespaces
ip netns list
ls -la /var/run/netns/

# Execute in namespace
ip netns exec ns1 ip addr show
ip netns exec ns1 ping 10.0.0.1
ip netns exec ns1 ss -tlnp
ip netns exec ns1 tcpdump -i eth0

# nsenter (for container PIDs):
nsenter -t <pid> -n ip addr show
nsenter -t <pid> -n ss -tlnp

# Docker:
docker exec <container> ip addr show
docker exec <container> ss -tlnp

# Trace veth pairs:
ethtool -S veth123  # Shows peer_ifindex
ip link show        # Match ifindex to find peer

# Bridge inspection:
bridge fdb show     # Forwarding database
bridge link show    # Bridge port details
brctl showstp br0   # STP state (if using brctl)
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/core/drop_monitor.c | Packet drop monitoring |
| net/core/net-sysfs.c | /sys/class/net/ interface |
| net/ipv4/proc.c | /proc/net/tcp, /proc/net/snmp |
| net/core/sock_diag.c | Socket diagnostics (ss) |
| include/trace/events/net.h | Network tracepoints |
| include/trace/events/tcp.h | TCP tracepoints |
| include/trace/events/skb.h | SKB tracepoints |

---

## Interview Questions

**Q1: A server is dropping packets. How do you systematically diagnose?**
A: Follow the packet path from bottom up: (1) Physical: ethtool — check link, speed, errors. (2) NIC: ethtool -S — look at rx_dropped, rx_errors, rx_no_buffer. (3) Kernel ingress: /proc/net/softnet_stat — column 2 for backlog drops, column 3 for time_squeeze. (4) Netfilter: iptables -L -v -n — check DROP counters. (5) Socket: ss -m — check Recv-Q depth. For precise location, use `perf record -e skb:kfree_skb` or dropwatch — this shows the exact kernel function where each packet is freed.

**Q2: How do you debug slow TCP throughput?**
A: (1) Measure with iperf3 to isolate from application issues. (2) Check ss -ti for: cwnd (too small?), RTT (high?), retransmissions. (3) High retransmits → packet loss somewhere → check each hop with mtr. (4) Check ethtool for speed negotiation (1000 vs 100 vs 10). (5) Check offloads with ethtool -k (TSO, GRO enabled?). (6) Check sysctl: tcp_rmem/wmem limits, window scaling. (7) Use tcpdump to examine the TCP handshake, window sizes, and SACK behavior. (8) Profile with perf/bpftrace if CPU-bound.

**Q3: What information does `ss -ti` provide and how do you interpret it?**
A: `ss -ti` shows per-connection TCP internals: **rtt** (smoothed RTT and variance), **cwnd** (congestion window — controls sending rate), **ssthresh** (slow start threshold), **send** (calculated send bandwidth = cwnd * mss / rtt), **retrans** (unrecovered retransmissions), **bytes_acked/received**, **segs_out/in**. Interpretation: low cwnd with high ssthresh = loss recovery. High rtt variance = unstable path. High retrans = packet loss. cwnd growing = slow start phase.

---

## Summary

- tcpdump: Packet capture with BPF filters, essential for protocol debugging
- ss -ti: Socket diagnostics with TCP internals (cwnd, RTT, retransmits)
- ethtool -S: NIC-level statistics (drops, errors, ring buffer state)
- /proc/net/softnet_stat: Kernel-level packet processing stats
- ftrace/bpftrace: Kernel function and tracepoint tracing
- dropwatch / perf skb:kfree_skb: Locate exact packet drop points
- Systematic approach: physical → NIC → kernel → firewall → transport → app

---

Next: [Chapter 28 — End-to-End Flow Diagrams](Chapter_28_Flow_Diagrams.md)
