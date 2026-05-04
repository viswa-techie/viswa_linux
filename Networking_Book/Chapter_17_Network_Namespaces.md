# Chapter 17: Network Namespaces and Containers

## Learning Goals
- Understand network namespace isolation
- Know veth pairs and container networking
- Understand CNI and container networking models
- Know namespace management APIs

---

## 17.1 Network Namespace Concept

```
Default: All processes share one network namespace (init_net).
Network namespace: isolated copy of the entire network stack.

Each namespace has its OWN:
  - Network devices (eth0, lo, ...)
  - IP addresses and routing tables
  - Firewall rules (iptables/nftables)
  - Socket hash tables
  - /proc/net content
  - ARP cache, conntrack entries

  ┌───────────────── Host ──────────────────┐
  │                                          │
  │  ┌──── Namespace: init_net ────┐         │
  │  │  eth0: 192.168.1.100       │         │
  │  │  lo: 127.0.0.1             │         │
  │  │  Routing table, iptables   │         │
  │  └────────────────────────────┘         │
  │                                          │
  │  ┌──── Namespace: ns1 ────────┐         │
  │  │  veth1: 10.0.0.1           │         │
  │  │  lo: 127.0.0.1             │ Container│
  │  │  Own routing, own iptables │         │
  │  └────────────────────────────┘         │
  │                                          │
  │  ┌──── Namespace: ns2 ────────┐         │
  │  │  veth3: 10.0.1.1           │         │
  │  │  lo: 127.0.0.1             │ Container│
  │  │  Own routing, own iptables │         │
  │  └────────────────────────────┘         │
  └──────────────────────────────────────────┘
```

---

## 17.2 Creating and Managing Namespaces

```bash
# Create namespace
ip netns add ns1

# List namespaces
ip netns list

# Execute command in namespace
ip netns exec ns1 ip addr show
ip netns exec ns1 ping 10.0.0.1

# Create veth pair (virtual Ethernet, always paired)
ip link add veth0 type veth peer name veth1

# Move one end to namespace
ip link set veth1 netns ns1

# Configure in root namespace
ip addr add 10.0.0.1/24 dev veth0
ip link set veth0 up

# Configure in ns1
ip netns exec ns1 ip addr add 10.0.0.2/24 dev veth1
ip netns exec ns1 ip link set veth1 up
ip netns exec ns1 ip link set lo up

# Now: root can ping 10.0.0.2, ns1 can ping 10.0.0.1

# Delete namespace
ip netns del ns1
```

---

## 17.3 Container Networking Model

```
Docker/Container networking:

  ┌─────────────────── Host ─────────────────────┐
  │  ┌──────── docker0 (bridge) ────────┐        │
  │  │        172.17.0.1/16             │        │
  │  │                                   │        │
  │  │  ┌─veth_a─┐      ┌─veth_c─┐     │  ┌────┐│
  │  │  └───┬────┘      └───┬────┘     │  │eth0││
  │  └──────┼────────────────┼──────────┘  │    ││
  │         │                │             └─┬──┘│
  │  ┌──────┼──┐      ┌─────┼───┐           │   │
  │  │Container│      │Container│       NAT/IP   │
  │  │  veth_b │      │  veth_d │    forwarding  │
  │  │172.17.0.2│     │172.17.0.3│          │    │
  │  └─────────┘      └─────────┘          │    │
  └──────────────────────────────────────── ┘    │
                                          Internet

  Components:
  1. docker0 bridge: L2 switch connecting containers
  2. veth pairs: one end in container, other in bridge
  3. NAT (iptables MASQUERADE): containers reach internet
  4. Each container: own network namespace
```

### CNI (Container Network Interface)

```
CNI: Standard for container networking plugins.

Kubernetes networking model:
  - Every pod gets a unique IP
  - All pods can reach all other pods (no NAT)
  - CNI plugin implements this

Popular CNI plugins:
  - Calico: BGP-based, L3 routing, network policies
  - Flannel: VXLAN overlay, simple
  - Cilium: eBPF-based, high performance
  - Weave: mesh overlay

  CNI flow:
    kubelet → calls CNI plugin → plugin creates veth pair
    → assigns IP → sets up routing/overlay
```

---

## 17.4 Kernel Implementation

```c
/* Network namespace structure */
struct net {
    refcount_t count;          /* Reference count */
    struct list_head list;     /* All namespaces */
    struct list_head dev_base_head; /* Devices in this ns */
    struct hlist_head *dev_name_head; /* Device name hash */
    struct proc_dir_entry *proc_net; /* /proc/net */
    struct netns_ipv4 ipv4;    /* IPv4 state */
    struct netns_ipv6 ipv6;    /* IPv6 state */
    /* Subsystem-specific namespace state: */
    /* nf (netfilter), xt (xtables), ct (conntrack) */
    /* routing tables, socket tables, etc. */
};

/* Every net_device belongs to a namespace */
dev->nd_net  →  struct net *

/* Move device between namespaces */
dev_change_net_namespace(dev, new_net, new_name);

/* Get current task's network namespace */
struct net *net = current->nsproxy->net_ns;
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/core/net_namespace.c | Namespace creation, management |
| include/net/net_namespace.h | struct net definition |
| drivers/net/veth.c | Virtual Ethernet pair driver |
| net/bridge/ | Bridge device for container networking |

---

## Interview Questions

**Q1: What is a network namespace and how does it provide isolation?**
A: A network namespace is a complete copy of the network stack: its own devices, IP addresses, routing tables, firewall rules, socket tables, and /proc/net. Processes in different namespaces cannot see or interfere with each other's networking. This is the foundation of container networking — each container gets its own namespace. Created via clone(CLONE_NEWNET) or ip netns add.

**Q2: How do containers communicate with the host and external networks?**
A: Veth pairs connect containers to the host. One end is in the container's namespace, the other in the host namespace (typically attached to a bridge). The bridge provides L2 switching between containers. For external access, iptables NAT (MASQUERADE) translates container IPs to the host's IP. IP forwarding (net.ipv4.ip_forward=1) enables routing between namespaces and the physical network.

**Q3: What is a veth pair?**
A: Veth (virtual Ethernet) is a pair of connected virtual network interfaces. Packets sent on one end appear on the other — like a virtual cable. One end lives in one namespace, the other in a different namespace (or the host). Created atomically as a pair with `ip link add veth0 type veth peer name veth1`. Used to connect containers to bridges, move traffic between namespaces.

---

## Summary

- Network namespaces provide complete network stack isolation
- veth pairs connect namespaces (virtual cable)
- Container networking: namespace + veth + bridge + NAT
- CNI standardizes container network plugin interface
- struct net holds per-namespace state for all networking subsystems

---

Next: [Chapter 18 — Virtual Networking](Chapter_18_Virtual_Networking.md)
