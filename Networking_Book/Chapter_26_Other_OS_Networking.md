# Chapter 26: Networking in Other Operating Systems

## Learning Goals
- Compare Linux networking with Windows, BSD, and RTOS network stacks
- Understand BSD socket origins and macOS/FreeBSD networking
- Know Windows networking architecture (NDIS, WFP, Winsock)
- Understand RTOS networking (lwIP, Zephyr)

---

## 26.1 BSD Networking Stack

```
BSD: Origin of the socket API (4.2BSD, 1983).
FreeBSD, OpenBSD, NetBSD, macOS inherit this lineage.

BSD vs Linux networking:
  ┌──────────────────────────────────────────────────────────────┐
  │ Aspect              │ BSD/FreeBSD       │ Linux              │
  ├─────────────────────┼───────────────────┼────────────────────┤
  │ Buffer structure    │ mbuf (chain of    │ sk_buff (linear    │
  │                     │ small clusters)   │ + paged frags)     │
  │ Interface structure │ struct ifnet      │ struct net_device  │
  │ Packet filter       │ pf (OpenBSD)      │ netfilter/nftables │
  │                     │ ipfw (FreeBSD)    │                    │
  │ Forwarding          │ if_bridge         │ bridge module      │
  │ Queueing            │ ALTQ              │ TC/qdisc           │
  │ Routing             │ radix tree        │ LC-Trie (FIB)      │
  │ High performance    │ netmap            │ XDP, DPDK          │
  │ Socket API          │ Original BSD      │ BSD-compatible      │
  │ License             │ BSD license       │ GPL                │
  │ VPN                 │ if_wg (WireGuard) │ wireguard module   │
  └──────────────────────┴───────────────────┴────────────────────┘

mbuf vs sk_buff:
  mbuf:     Chain of small buffers (256B typically)
            Good for variable-size data, protocol processing
            Multiple mbufs linked via m_next, m_nextpkt
            Header data may span mbufs (pullup needed)

  sk_buff:  Single large linear buffer + optional paged fragments
            Efficient for DMA (contiguous data)
            Headers always in linear region
            Separate head/data/tail/end pointers

FreeBSD netmap:
  High-performance framework similar to DPDK
  Maps NIC rings directly to userspace
  Achieves 14.88 Mpps on 10GbE
  Integrated into FreeBSD kernel
```

---

## 26.2 Windows Networking

```
Windows networking architecture:

  ┌──────────────────────────────────────────────────┐
  │  Applications                                    │
  │    Winsock API (ws2_32.dll)                      │
  │    WinHTTP / WinINet                             │
  │                                                  │
  │  ┌──────────────────────────────────────────┐    │
  │  │ AFD (Ancillary Function Driver)          │    │
  │  │   afd.sys — Winsock kernel support       │    │
  │  └──────────────────────────┬───────────────┘    │
  │                             │                    │
  │  ┌──────────────────────────┴───────────────┐    │
  │  │ TDI → TDX (Transport Driver Interface)   │    │
  │  │   tcpip.sys — TCP/IP stack               │    │
  │  └──────────────────────────┬───────────────┘    │
  │                             │                    │
  │  ┌──────────────────────────┴───────────────┐    │
  │  │ WFP (Windows Filtering Platform)         │    │
  │  │   Replaces NDIS filters for firewalling  │    │
  │  │   Layers: Stream, Transport, Network,    │    │
  │  │           Forward, ALE (Application)     │    │
  │  └──────────────────────────┬───────────────┘    │
  │                             │                    │
  │  ┌──────────────────────────┴───────────────┐    │
  │  │ NDIS (Network Driver Interface Spec)     │    │
  │  │   Miniport drivers (NIC drivers)         │    │
  │  │   Protocol drivers (bind to adapters)    │    │
  │  │   Filter drivers (packet inspection)     │    │
  │  │   NDIS 6.x: supports RSS, chimney, VMQ  │    │
  │  └──────────────────────────┬───────────────┘    │
  │                             │                    │
  │                         [NIC Hardware]            │
  └──────────────────────────────────────────────────┘

Key differences from Linux:
  - NDIS: Standardized driver model (vs Linux's ad-hoc netdev_ops)
  - WFP: Layered filtering platform (vs netfilter hooks)
  - Winsock: BSD socket API with extensions (WSA*, IOCP)
  - IOCP: I/O Completion Ports (proactor pattern vs Linux epoll reactor)
  - RSS: Similar concept, different configuration mechanism
  - Chimney/RSC: TCP offload to NIC (largely abandoned)

Winsock vs POSIX sockets:
  WSAStartup() / WSACleanup()  — initialization required
  closesocket() vs close()
  SOCKET type vs int
  WSAGetLastError() vs errno
  IOCP vs epoll for async I/O
  WSARecv/WSASend with overlapped I/O
```

---

## 26.3 macOS Networking

```
macOS: Based on XNU kernel (Mach + BSD hybrid).
Network stack derived from FreeBSD.

  ┌──────────────────────────────────────────────┐
  │  User Space                                  │
  │    BSD Sockets / CFNetwork / Network.framework│
  │                                              │
  │  ┌──────────────────────────────────────┐    │
  │  │ BSD Layer (FreeBSD-derived)          │    │
  │  │   mbuf-based packet processing      │    │
  │  │   TCP/IP stack from FreeBSD         │    │
  │  │   pf packet filter                  │    │
  │  └──────────────┬─────────────────────┘    │
  │                  │                          │
  │  ┌──────────────┴─────────────────────┐    │
  │  │ Network Kernel Extensions (NKE)     │    │
  │  │   Socket filter NKE                 │    │
  │  │   IP filter NKE                     │    │
  │  │   Interface filter NKE              │    │
  │  │   (Deprecated → NetworkExtension)   │    │
  │  └──────────────┬─────────────────────┘    │
  │                  │                          │
  │  ┌──────────────┴─────────────────────┐    │
  │  │ IOKit Network Drivers               │    │
  │  │   IONetworkController               │    │
  │  │   IOEthernetController              │    │
  │  └────────────────────────────────────┘    │
  └──────────────────────────────────────────────┘

  macOS specific:
  - Network.framework: Modern Swift/Obj-C networking API
  - SystemExtension: Replaces kernel extensions (NKE)
  - Content Filter: App-level filtering
  - DriverKit: User-space driver framework
```

---

## 26.4 RTOS Networking

```
Real-Time Operating Systems: lightweight TCP/IP stacks.

lwIP (Lightweight IP):
  - Designed for embedded systems with limited RAM (10s of KB)
  - Full TCP/IP v4/v6 stack
  - Two APIs: raw (callback), sequential (blocking, needs threads)
  - No dynamic memory for core path (pbuf pools)
  - Widely used: ESP32, STM32, NXP, many MCUs
  - Single-threaded core with optional multi-thread wrapper

  lwIP vs Linux stack:
  ┌────────────────────────────────────────────────────┐
  │ Aspect         │ lwIP            │ Linux           │
  ├────────────────┼─────────────────┼─────────────────┤
  │ RAM required   │ 10-40 KB        │ 10+ MB          │
  │ Code size      │ 40-100 KB       │ Several MB      │
  │ TCP features   │ Basic           │ Full (SACK,etc.)│
  │ Throughput     │ 10-100 Mbps     │ 10+ Gbps        │
  │ Socket API     │ Compatible      │ Full POSIX      │
  │ Zero-copy      │ pbuf chains     │ sk_buff + SG    │
  │ Buffer struct  │ pbuf            │ sk_buff         │
  └────────────────┴─────────────────┴─────────────────┘

Zephyr RTOS networking:
  - Full network stack with BSD sockets API
  - Supports: IPv4/IPv6, TCP/UDP, MQTT, CoAP, LwM2M
  - net_buf: Zephyr's packet buffer (similar to mbuf chain)
  - Network interface abstraction (struct net_if)
  - Runs on: ARM Cortex-M, RISC-V, x86, ARC
  - IEEE 802.15.4, BLE, Ethernet, Wi-Fi support

FreeRTOS+TCP:
  - TCP/IP stack for FreeRTOS
  - Zero-copy option with reference-counted buffers
  - Integrated with FreeRTOS task scheduler
  - Used with AWS IoT libraries

QNX networking:
  - POSIX-compliant, microkernel-based
  - io-pkt: Network stack runs as user-space service
  - Based on NetBSD stack
  - Resource manager architecture
  - Used in automotive (BlackBerry QNX)
  - Deterministic, safety-certified (ISO 26262 ASIL D)
```

---

## 26.5 Cross-Platform Comparison

```
┌──────────────────────────────────────────────────────────────────────┐
│ Feature         │ Linux    │ FreeBSD  │ Windows  │ QNX      │ Zephyr │
├─────────────────┼──────────┼──────────┼──────────┼──────────┼────────┤
│ Socket API      │ POSIX    │ BSD orig.│ Winsock  │ POSIX    │ POSIX  │
│ Packet buffer   │ sk_buff  │ mbuf     │ NET_BUFF │ mbuf     │ net_buf│
│ Driver model    │ netdev   │ ifnet    │ NDIS     │ io-pkt   │ net_if │
│ Firewall        │ nftables │ pf       │ WFP      │ pf       │ N/A    │
│ High-perf       │ XDP/DPDK │ netmap   │ DPDK     │ N/A      │ N/A    │
│ Async I/O       │ epoll    │ kqueue   │ IOCP     │ ionotify │ poll   │
│ Namespaces      │ Yes      │ Jails    │ Containers│ N/A     │ N/A    │
│ Routing         │ FIB trie │ radix    │ RIB      │ radix    │ basic  │
│ CAN support     │ SocketCAN│ N/A      │ 3rd party│ devnp-can│ can    │
│ TSN support     │ taprio   │ Limited  │ Limited  │ Yes      │ Limited│
│ License         │ GPL      │ BSD      │ Propriet.│ Propriet.│ Apache │
└─────────────────┴──────────┴──────────┴──────────┴──────────┴────────┘

Async I/O model comparison:
  Linux epoll:    Reactor pattern — "ready" notification, app reads
  BSD kqueue:     Reactor pattern — unified event notification
  Windows IOCP:   Proactor pattern — OS completes I/O, notifies app
  Linux io_uring: Proactor-like — submission/completion ring buffers

  kqueue advantages over epoll:
  - Unified: files, sockets, signals, processes, timers
  - Edge+Level combined
  - Batch registration (changelist)

  io_uring advantages:
  - Zero-syscall submission via shared ring
  - Supports file I/O + networking
  - Registered buffers for zero-copy
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/socket.c | Linux BSD socket layer |
| net/ipv4/tcp.c | Linux TCP implementation |
| net/can/af_can.c | Linux SocketCAN |
| — | FreeBSD: sys/netinet/, sys/net/ |
| — | lwIP: src/core/, src/netif/ |
| — | Zephyr: subsys/net/ |

---

## Interview Questions

**Q1: Compare Linux sk_buff with BSD mbuf.**
A: **sk_buff**: Single large buffer with head/data/tail/end pointers and optional paged fragments (skb_shared_info). Headers are always in the linear region. Efficient for DMA and hardware offloads. One allocation per packet. **mbuf**: Chain of small clusters (~256B each) linked via m_next. Headers may span multiple mbufs (m_pullup needed to linearize). More flexible for protocol processing but more overhead for DMA. sk_buff is optimized for modern NIC DMA while mbuf is more memory-efficient for variable-size data.

**Q2: How does Windows networking architecture differ from Linux?**
A: Windows uses a layered architecture: Winsock (user API) → AFD.sys (kernel socket support) → tcpip.sys (TCP/IP stack) → NDIS (driver framework). Key differences: (1) NDIS provides a standardized, versioned driver model (vs Linux's evolving netdev_ops). (2) WFP replaces netfilter — layered filtering at multiple levels including application identity. (3) IOCP is a proactor model (OS completes I/O) vs Linux epoll's reactor model (OS notifies readiness). (4) No network namespaces equivalent until recent container support.

**Q3: When would you use lwIP vs the Linux kernel stack?**
A: lwIP for microcontrollers with 10-256KB RAM that can't run Linux. It provides basic TCP/IP (no SACK, limited window sizes) in ~40-100KB code. Linux kernel stack for systems with >32MB RAM needing full-featured networking (GRO, TSO, netfilter, namespaces, advanced TCP). If the device can run Linux, the Linux stack is always preferred for its maturity, security patches, and feature completeness. lwIP targets bare-metal or RTOS environments (ESP32, STM32, etc.).

---

## Summary

- BSD: Origin of socket API, uses mbuf chains, pf firewall, kqueue async I/O
- Windows: NDIS driver model, WFP firewall, IOCP proactor I/O, Winsock API
- macOS: XNU kernel with FreeBSD-derived stack, moving to user-space extensions
- RTOS: lwIP (minimal RAM), Zephyr (POSIX sockets), FreeRTOS+TCP, QNX (POSIX, safety)
- Linux strengths: XDP/eBPF performance, network namespaces, rich offload support
- Socket API is universal — BSD origin, adopted by POSIX, ported to all OS

---

Next: [Chapter 27 — Network Debugging Tools and Techniques](Chapter_27_Debugging.md)
