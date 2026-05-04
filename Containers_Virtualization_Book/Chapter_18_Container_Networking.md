# Chapter 18: Container Networking Deep Dive

## Learning Goals
- Understand CNI (Container Network Interface) plugin model
- Learn advanced networking: VXLAN overlays, service mesh
- Master iptables/nftables NAT for containers
- Know network policies and multi-network setups

---

## 1. CNI Plugin Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  CNI: Container Network Interface                        │
  │  Spec for configuring container networking               │
  │  Used by: Kubernetes, Podman, CRI-O                     │
  │  NOT used by: Docker (has its own libnetwork)            │
  │                                                           │
  │  How CNI works:                                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. Runtime creates network namespace      │            │
  │  │ 2. Runtime calls CNI plugin binary:       │            │
  │  │    CNI_COMMAND=ADD                        │            │
  │  │    CNI_CONTAINERID=abc123                 │            │
  │  │    CNI_NETNS=/proc/1234/ns/net            │            │
  │  │    CNI_IFNAME=eth0                        │            │
  │  │    ← stdin: JSON config                   │            │
  │  │                                           │            │
  │  │ 3. Plugin sets up networking:             │            │
  │  │    - Creates veth pair                    │            │
  │  │    - Moves one end into container NS      │            │
  │  │    - Assigns IP address                   │            │
  │  │    - Sets up routes                       │            │
  │  │                                           │            │
  │  │ 4. Plugin returns result (JSON):          │            │
  │  │    { "ips": [{"address":"10.244.1.5/24"}],│            │
  │  │      "routes": [{"dst":"0.0.0.0/0"}] }   │            │
  │  │                                           │            │
  │  │ 5. On container deletion:                 │            │
  │  │    CNI_COMMAND=DEL (cleanup)              │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  CNI plugin types:                                       │
  │  ┌────────────┬──────────────────────────────────┐      │
  │  │ bridge     │ Linux bridge + veth (default)    │      │
  │  │ macvlan    │ Direct L2 on physical NIC        │      │
  │  │ ipvlan     │ Shared MAC, L3 routing           │      │
  │  │ ptp        │ Point-to-point veth (no bridge)  │      │
  │  │ host-local │ IPAM: allocate IPs from local pool│     │
  │  │ dhcp       │ IPAM: get IP from DHCP server    │      │
  │  │ flannel    │ Overlay: VXLAN tunnels           │      │
  │  │ calico     │ BGP routing + iptables policy    │      │
  │  │ cilium     │ eBPF-based networking + policy   │      │
  │  │ weave      │ Mesh overlay network             │      │
  │  └────────────┴──────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. VXLAN Overlay Networking

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  VXLAN: Virtual Extensible LAN                           │
  │  Encapsulates L2 frames in L4 (UDP port 4789)           │
  │  Enables cross-host container networking                 │
  │                                                           │
  │  Node A (10.0.0.1)              Node B (10.0.0.2)       │
  │  ┌───────────────────┐         ┌───────────────────┐    │
  │  │ Pod 1 (10.244.0.5)│         │ Pod 3 (10.244.1.8)│    │
  │  │ Pod 2 (10.244.0.6)│         │ Pod 4 (10.244.1.9)│    │
  │  │       │            │         │       │            │    │
  │  │    cni0 bridge     │         │    cni0 bridge     │    │
  │  │       │            │         │       │            │    │
  │  │    flannel.1       │         │    flannel.1       │    │
  │  │   (VXLAN VNI=1)   │         │   (VXLAN VNI=1)   │    │
  │  │       │            │         │       │            │    │
  │  │    eth0 (10.0.0.1) │         │    eth0 (10.0.0.2) │    │
  │  └───────┼────────────┘         └───────┼────────────┘    │
  │          │                              │                 │
  │          └──────────UDP:4789────────────┘                 │
  │                  VXLAN tunnel                             │
  │                                                           │
  │  Packet flow (Pod 1 → Pod 3):                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ 1. Pod 1 sends: src=10.244.0.5           │            │
  │  │                 dst=10.244.1.8            │            │
  │  │ 2. Route table: 10.244.1.0/24 → flannel.1│            │
  │  │ 3. flannel.1 VXLAN device:                │            │
  │  │    - Lookup FDB: 10.244.1.8 → Node B     │            │
  │  │    - Encapsulate in UDP:4789              │            │
  │  │    - Outer: src=10.0.0.1, dst=10.0.0.2   │            │
  │  │ 4. Physical network delivers UDP packet   │            │
  │  │ 5. Node B flannel.1: decapsulate          │            │
  │  │ 6. Route to local Pod 3 via cni0 bridge   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Encapsulated packet format:                             │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Outer Ethernet │ Outer IP │ UDP:4789 │   │            │
  │  │ Outer MAC      │ 10.0.0.1 │          │   │            │
  │  │                │→10.0.0.2 │          │   │            │
  │  ├──────────────────────────────────────────┤            │
  │  │ VXLAN Header │ Inner Ethernet │ Inner IP │            │
  │  │ VNI=1        │ Inner MAC      │10.244.0.5│            │
  │  │              │                │→10.244.1.8│            │
  │  ├──────────────────────────────────────────┤            │
  │  │            Payload (TCP/HTTP/etc.)        │            │
  │  └──────────────────────────────────────────┘            │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. iptables NAT for Containers

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Docker/container iptables rules:                        │
  │                                                           │
  │  MASQUERADE (outgoing traffic):                          │
  │  ┌──────────────────────────────────────────┐            │
  │  │ -t nat -A POSTROUTING                    │            │
  │  │   -s 172.17.0.0/16 ! -o docker0          │            │
  │  │   -j MASQUERADE                          │            │
  │  │                                          │            │
  │  │ Container (172.17.0.2) → Internet:       │            │
  │  │   src 172.17.0.2 → MASQUERADE to host IP│            │
  │  │   External sees host IP, not container   │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  DNAT (port forwarding, -p 8080:80):                     │
  │  ┌──────────────────────────────────────────┐            │
  │  │ -t nat -A PREROUTING                     │            │
  │  │   -p tcp --dport 8080                    │            │
  │  │   -j DNAT --to-destination 172.17.0.2:80 │            │
  │  │                                          │            │
  │  │ External → host:8080                     │            │
  │  │   → DNAT to 172.17.0.2:80               │            │
  │  │   → response: SNAT back through host    │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Inter-container forwarding:                             │
  │  ┌──────────────────────────────────────────┐            │
  │  │ -A FORWARD -i docker0 -o docker0         │            │
  │  │   -j ACCEPT                              │            │
  │  │                                          │            │
  │  │ Container A → Container B (same bridge): │            │
  │  │   Bridged directly (L2 forwarding)       │            │
  │  │   No NAT needed                          │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Kubernetes uses kube-proxy for Service → Pod mapping:  │
  │  - iptables mode: DNAT rules per Service endpoint      │
  │  - IPVS mode: kernel IPVS load balancer (faster)       │
  │  - eBPF mode (Cilium): bypass iptables entirely        │
  └──────────────────────────────────────────────────────────┘
```

---

## 4. eBPF-Based Container Networking (Cilium)

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Traditional: iptables rules (sequential, O(n))          │
  │  Modern: eBPF programs (hash-based, O(1))                │
  │                                                           │
  │  ┌──────────────────────────────────────────┐            │
  │  │ Cilium with eBPF:                        │            │
  │  │                                          │            │
  │  │ Container → veth → TC hook:              │            │
  │  │   eBPF program runs:                     │            │
  │  │   1. Lookup destination in BPF map       │            │
  │  │   2. If same node: redirect to dst veth  │            │
  │  │      (skip bridge, iptables, routing!)   │            │
  │  │   3. If different node: VXLAN/Geneve     │            │
  │  │      encap, send to remote node          │            │
  │  │   4. Apply network policy (BPF map)      │            │
  │  │                                          │            │
  │  │ Advantages:                              │            │
  │  │ - O(1) lookup instead of O(n) iptables   │            │
  │  │ - No bridge needed for same-node traffic │            │
  │  │ - Inline network policy enforcement      │            │
  │  │ - L7 visibility (HTTP, gRPC, DNS)       │            │
  │  │ - No kube-proxy needed                   │            │
  │  │ - BPF maps updated atomically            │            │
  │  └──────────────────────────────────────────┘            │
  │                                                           │
  │  Performance impact:                                     │
  │  1000 services:                                          │
  │  iptables: ~1000 rules evaluated per packet (O(n))      │
  │  eBPF: hash lookup in BPF map (O(1))                    │
  │  At scale, eBPF is 10-100x faster for routing decisions │
  └──────────────────────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: How does CNI work, and what happens when Kubernetes creates a pod?**
**A:** CNI is a specification where the container runtime calls an external binary (the CNI plugin) to set up networking. When kubelet creates a pod: (1) The CRI runtime (containerd) creates a network namespace for the pod sandbox (pause container). (2) The CRI runtime calls the CNI plugin binary with environment variables: `CNI_COMMAND=ADD`, `CNI_NETNS=/proc/PID/ns/net`, `CNI_IFNAME=eth0`, and passes the CNI config JSON on stdin. (3) The CNI plugin (e.g., Calico, Cilium, Flannel) performs its specific networking setup: creates a veth pair, moves one end into the pod's namespace, assigns an IP (via IPAM plugin like host-local or Calico IPAM), sets up routes, installs iptables/eBPF rules. (4) The plugin returns a JSON result with the assigned IP and routes. The runtime stores this. (5) On pod deletion: `CNI_COMMAND=DEL` — plugin cleans up (deletes veth, releases IP back to pool, removes rules). CNI plugins are chained: multiple plugins run sequentially (e.g., bridge plugin creates veth, then bandwidth plugin applies traffic shaping). Kubernetes supports multiple network interfaces via Multus CNI (meta-plugin that delegates to multiple CNI plugins).

**Q2: Compare iptables-based and eBPF-based container networking.**
**A:** **iptables**: Traditional approach. Each Kubernetes Service creates DNAT rules mapping ClusterIP to pod endpoints. Rules are evaluated sequentially — for a packet, the kernel walks through ALL rules until a match. At 1000+ services, this means 5000+ rules evaluated per packet, causing measurable latency. Rule updates (endpoint changes) require rewriting the entire chain (atomic replacement), which is slow and can cause brief drops. `kube-proxy` manages these rules. **eBPF** (Cilium): Programs attached to TC (traffic control) hooks on veth interfaces. Uses BPF hash maps for O(1) lookups instead of sequential rule evaluation. Same-node pod-to-pod traffic can skip the bridge and iptables entirely — eBPF redirects packets directly between veth pairs (`bpf_redirect_peer()`). Network policies are compiled into BPF programs, enforced inline at the veth level. Map updates are atomic and don't affect existing connections. Additional capabilities: L7 protocol visibility (parse HTTP/gRPC headers in eBPF), DNS-aware network policies, transparent encryption (WireGuard integration). Trade-offs: eBPF requires kernel 4.19+ (5.10+ recommended), more complex debugging (BPF tools needed), and Cilium itself is a complex system. For < 100 services, iptables is fine; at scale (1000+ services, 10K+ pods), eBPF is significantly faster.

---

## Summary

- CNI: plugin-based networking for containers; runtime calls binary with ADD/DEL commands
- CNI plugins: bridge (default), Calico (BGP), Cilium (eBPF), Flannel (VXLAN), Weave
- VXLAN overlay: encapsulates L2 in UDP:4789 for cross-host pod communication
- iptables: MASQUERADE for outgoing NAT, DNAT for port forwarding, sequential O(n) evaluation
- eBPF (Cilium): O(1) BPF map lookups, skip bridge for same-node traffic, inline policy
- kube-proxy modes: iptables (default), IPVS (faster), or eBPF replacement (Cilium)
- At scale: eBPF 10-100x faster than iptables for routing decisions

---

[Previous: containerd and CRI-O ←](Chapter_17_Containerd_CRI.md) | [Next: Hardware Virtualization Foundations →](Chapter_19_HW_Virtualization.md)
