# Chapter 23: High-Performance Networking — DPDK, eBPF, RDMA

## Learning Goals
- Understand DPDK architecture and kernel bypass
- Know eBPF framework beyond XDP (TC, socket, tracing)
- Understand RDMA and kernel bypass for HPC
- Compare approaches: XDP vs DPDK vs kernel stack

---

## 23.1 DPDK (Data Plane Development Kit)

```
DPDK: User-space packet processing framework.
Bypasses the kernel network stack entirely.

Architecture:
  ┌─────────────────────────────────────────────────────────┐
  │  Traditional Stack         │  DPDK                      │
  │                            │                            │
  │  User App                  │  DPDK App (user space)     │
  │    │ syscall               │    │ direct                │
  │    ▼                       │    ▼                       │
  │  Socket Layer              │  rte_eth_rx_burst()        │
  │    │                       │    │                       │
  │  TCP/IP Stack              │  (no kernel at all)        │
  │    │                       │    │                       │
  │  Netfilter                 │  Poll-mode driver (PMD)    │
  │    │                       │    │                       │
  │  Device Driver             │  UIO / VFIO (DMA mapping)  │
  │    │ interrupt             │    │ poll                  │
  │    ▼                       │    ▼                       │
  │  [NIC]                     │  [NIC]                     │
  └─────────────────────────────────────────────────────────┘

Key concepts:
  PMD (Poll Mode Driver): User-space driver, busy-polls NIC
    - No interrupts → no context switch overhead
    - Dedicated CPU cores spinning in poll loop
    - Reaches 80+ Mpps on modern hardware

  Hugepages: 2MB/1GB pages reduce TLB misses
    echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
    mount -t hugetlbfs nodev /mnt/huge

  Memory pools (mbufs): Pre-allocated packet buffers
    rte_pktmbuf_pool_create("pool", 8192, 256, 0, RTE_MBUF_DEFAULT_BUF_SIZE, socket_id)

  Multi-queue: Each core handles separate queue
  NUMA awareness: Allocate memory on same NUMA node as NIC

DPDK application flow:
  1. rte_eal_init()              — Initialize environment
  2. rte_eth_dev_configure()     — Configure NIC (queues, RSS)
  3. rte_eth_rx_queue_setup()    — Setup RX queues
  4. rte_eth_tx_queue_setup()    — Setup TX queues
  5. rte_eth_dev_start()         — Start the device
  6. Loop:
     rte_eth_rx_burst() → process → rte_eth_tx_burst()
```

### DPDK Trade-offs

```
Advantages:
  + Highest throughput (80+ Mpps)
  + Lowest latency (< 1 microsecond)
  + Predictable performance (no kernel jitter)

Disadvantages:
  - Dedicated CPU cores (100% utilization even idle)
  - No kernel networking features (firewall, routing, etc.)
  - Must implement TCP/IP in user space or use DPDK TCP stacks
  - No kernel security features
  - Application must handle everything
  - Complex deployment
```

---

## 23.2 eBPF Framework

```
eBPF: In-kernel virtual machine for programmable packet processing,
      tracing, and security.

eBPF program types (networking-related):
  ┌──────────────────────────────────────────────────────────┐
  │ Type                    │ Hook Point      │ Use Case      │
  ├─────────────────────────┼─────────────────┼───────────────┤
  │ BPF_PROG_TYPE_XDP       │ Driver RX       │ Fast filtering│
  │ BPF_PROG_TYPE_SCHED_CLS │ TC ingress/     │ Classification│
  │                         │ egress          │ Redirect      │
  │ BPF_PROG_TYPE_SOCK_OPS  │ Socket events   │ TCP tuning    │
  │ BPF_PROG_TYPE_SK_SKB    │ Socket buffer   │ Proxy, splice │
  │ BPF_PROG_TYPE_CGROUP_SKB│ cgroup          │ Pod-level     │
  │ BPF_PROG_TYPE_LWT_*     │ Lightweight     │ Encap/decap   │
  │                         │ tunnel          │               │
  └──────────────────────────────────────────────────────────┘

eBPF architecture:
  ┌──────────┐    compile    ┌──────────┐   load    ┌────────┐
  │ C source │───────────►  │ BPF      │────────►  │ Kernel │
  │ (.c)     │   clang/llc  │ bytecode │  bpf()    │ verify │
  └──────────┘              │ (.o)     │  syscall  │ + JIT  │
                            └──────────┘           └────────┘
                                                       │
                                                       ▼
                                               [Hook point runs
                                                native machine code]

eBPF Maps: Key-value stores shared between BPF programs and userspace
  BPF_MAP_TYPE_HASH          — General hash table
  BPF_MAP_TYPE_ARRAY         — Fixed-size array
  BPF_MAP_TYPE_LRU_HASH      — LRU eviction
  BPF_MAP_TYPE_PERF_EVENT_ARRAY — Event streaming
  BPF_MAP_TYPE_RINGBUF       — Ring buffer (efficient)
  BPF_MAP_TYPE_LPM_TRIE      — Longest prefix match

  Used for:
  - Statistics counters
  - Connection tracking tables
  - Configuration (policy maps)
  - Communication between XDP and TC programs
```

### TC-BPF (Traffic Control eBPF)

```c
/* TC-BPF: runs on sk_buff (more features than XDP) */
SEC("tc")
int tc_prog(struct __sk_buff *skb)
{
    void *data = (void *)(long)skb->data;
    void *data_end = (void *)(long)skb->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return TC_ACT_OK;

    /* Redirect to another interface */
    if (eth->h_proto == bpf_htons(ETH_P_IP))
        return bpf_redirect(ifindex_target, 0);

    return TC_ACT_OK;
}

/* TC actions: TC_ACT_OK, TC_ACT_SHOT (drop),
   TC_ACT_REDIRECT, TC_ACT_PIPE */

/* Attach: */
/* tc filter add dev eth0 ingress bpf da obj prog.o sec tc */
```

### Cilium: eBPF-based Kubernetes Networking

```
Cilium uses eBPF for container networking:
  - XDP: DDoS mitigation at driver level
  - TC-BPF: L3/L4 load balancing (replaces kube-proxy)
  - Socket-level: L7 policy enforcement
  - No iptables rules needed
  - Transparent encryption (WireGuard/IPsec)

Performance: 3-10x faster than iptables-based kube-proxy
```

---

## 23.3 RDMA (Remote Direct Memory Access)

```
RDMA: Application reads/writes remote memory directly,
      bypassing both local and remote kernel.

Architecture:
  ┌──────────────────────────────────────────────────────┐
  │  Traditional                │  RDMA                  │
  │                             │                        │
  │  App → syscall → kernel     │  App → RDMA verb       │
  │  → TCP/IP → driver → NIC   │  → RNIC hardware       │
  │  → wire → NIC → driver     │  → wire → RNIC         │
  │  → kernel → syscall → App  │  → remote memory → App │
  │                             │                        │
  │  Copies: 2+ per side       │  Copies: 0             │
  │  Syscalls: multiple        │  Syscalls: 0 (data path)│
  │  CPU involvement: high     │  CPU involvement: zero  │
  └──────────────────────────────────────────────────────┘

RDMA operations:
  SEND/RECV:    Two-sided (like socket but zero-copy)
  READ:         One-sided read from remote memory (remote CPU uninvolved)
  WRITE:        One-sided write to remote memory
  ATOMIC:       Compare-and-swap, fetch-and-add on remote memory

RDMA transports:
  InfiniBand:   HPC fabric (dedicated network)
  RoCE:         RDMA over Converged Ethernet (v1: L2, v2: UDP/IP)
  iWARP:        RDMA over TCP (software-friendly, lower performance)

RDMA API (verbs):
  ibv_open_device()     — Open RDMA device
  ibv_alloc_pd()        — Allocate protection domain
  ibv_reg_mr()          — Register memory region
  ibv_create_qp()       — Create queue pair (send + recv queues)
  ibv_post_send()       — Post send request
  ibv_post_recv()       — Post receive buffer
  ibv_poll_cq()         — Poll completion queue

  Key concepts:
  - Memory Registration: Pin user pages, create HW mapping
  - Queue Pair (QP): Send queue + receive queue
  - Completion Queue (CQ): Hardware signals completion
  - Protection Domain (PD): Memory isolation

Use cases:
  - HPC: MPI over RDMA
  - Storage: NVMe over Fabrics, iSER
  - Databases: distributed transaction commit
  - Machine learning: GPU-Direct RDMA
```

---

## 23.4 Comparison Matrix

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Feature          │ Kernel Stack │ XDP      │ DPDK        │ RDMA        │
├──────────────────┼──────────────┼──────────┼─────────────┼─────────────┤
│ Throughput       │ 1-5 Mpps     │ 20-30    │ 80+ Mpps    │ 100+ Mpps   │
│                  │              │ Mpps     │             │             │
│ Latency          │ 10-100 us    │ 1-10 us  │ < 1 us      │ < 1 us      │
│ CPU overhead     │ High         │ Medium   │ Dedicated   │ Near zero   │
│ Features         │ Full stack   │ L2-L4    │ Implement   │ Send/Recv/  │
│                  │              │ filter   │ yourself    │ RDMA ops    │
│ Programming      │ Sockets API  │ eBPF (C) │ DPDK API    │ Verbs API   │
│ Kernel bypass    │ No           │ Partial  │ Full        │ Full        │
│ Security         │ Netfilter,   │ Verifier │ App's       │ Hardware    │
│                  │ conntrack    │          │ responsibility│ keys      │
│ Use case         │ General      │ Filtering│ Network     │ HPC,        │
│                  │ purpose      │ LB, DDoS │ functions   │ storage     │
│ Complexity       │ Low          │ Medium   │ High        │ High        │
│ Integration      │ Full OS      │ Kernel   │ None        │ Kernel      │
│                  │              │ hooks    │             │ support     │
└─────────────────────────────────────────────────────────────────────────┘

When to use what:
  Kernel stack: General applications, web servers, most software
  XDP: DDoS mitigation, load balancing, container networking
  DPDK: Firewalls, routers, 5G UPF, NFV appliances
  RDMA: HPC clusters, distributed databases, NVMe-oF storage
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/core/xdp.c | XDP framework |
| kernel/bpf/syscall.c | BPF syscall (loading progs) |
| kernel/bpf/verifier.c | BPF verifier |
| net/sched/cls_bpf.c | TC-BPF classifier |
| net/core/filter.c | Socket filter / BPF helpers |
| drivers/infiniband/ | RDMA/InfiniBand subsystem |
| drivers/net/ethernet/*/xdp* | Driver XDP implementations |

---

## Interview Questions

**Q1: Compare DPDK and XDP. When would you use each?**
A: **DPDK** runs entirely in user space with dedicated CPU cores polling the NIC. It bypasses the kernel completely, achieving 80+ Mpps but requiring reimplementation of all network features. Use for: dedicated network appliances, firewalls, 5G core. **XDP** runs eBPF programs in the kernel driver's NAPI poll, before sk_buff allocation. It integrates with the kernel stack (XDP_PASS), achieving 20-30 Mpps. Use for: DDoS mitigation, load balancing, container networking where you still need kernel features.

**Q2: How does eBPF ensure safety in the kernel?**
A: The BPF verifier (kernel/bpf/verifier.c) statically analyzes every program before loading: (1) DAG check — no loops (ensures termination). (2) Register state tracking — every register and memory access validated. (3) Bounds checking — all pointer accesses proved safe. (4) Allowed helper functions only. (5) Program size limit (1M instructions). (6) Stack size limit (512 bytes). After verification, the JIT compiler translates bytecode to native machine code. This guarantees no crashes, no infinite loops, and no out-of-bounds access.

**Q3: What is RDMA and how does it bypass the kernel?**
A: RDMA allows an application to directly read/write remote memory without involving the OS on either side. The NIC (RNIC) handles the protocol (InfiniBand, RoCE, iWARP), DMA, and reliability. The application registers memory regions, posts work requests to queue pairs, and polls completion queues — all via memory-mapped I/O, no syscalls on the data path. This achieves sub-microsecond latency and near-zero CPU overhead.

**Q4: What are eBPF maps and how are they used in networking?**
A: Maps are kernel-side key-value stores accessible from both BPF programs and userspace. Types include hash maps, arrays, LRU hashes, LPM tries, ring buffers. In networking: (1) Connection tracking tables (hash map of 5-tuples to states). (2) Statistics counters (per-CPU arrays). (3) Load balancer backends (hash map of VIP → backend list). (4) Policy rules (LPM trie for CIDR matching). (5) Event streaming (perf event array / ring buffer for logging). Maps enable stateful programs and user-kernel communication.

---

## Summary

- DPDK: Full kernel bypass, user-space polling, dedicated CPU, 80+ Mpps
- eBPF: In-kernel VM with verifier safety, JIT compiled, hooks at XDP/TC/socket
- RDMA: Zero-copy remote memory access, sub-microsecond latency
- XDP: 20-30 Mpps, integrates with kernel stack, good for filtering/LB
- Cilium: eBPF-based Kubernetes networking replacing iptables
- Choice depends on: throughput needs, complexity tolerance, kernel integration needs

---

Next: [Chapter 24 — Kernel Networking APIs and Netlink](Chapter_24_Kernel_APIs_Netlink.md)
