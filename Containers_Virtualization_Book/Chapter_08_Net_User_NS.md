# Chapter 8: Network and User Namespaces Deep Dive

## Learning Goals
- Understand network namespace architecture and virtual networking
- Master user namespace UID/GID mapping and capability model
- Learn veth, bridge, macvlan, and ipvlan internals
- Know how user namespaces enable rootless containers

---

## 1. Network Namespace Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Each network namespace gets its own:                    │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ - Network interfaces (eth0, lo, veth, etc.)   │      │
  │  │ - IP addresses and routing tables              │      │
  │  │ - iptables/nftables rules                      │      │
  │  │ - Socket table (all TCP/UDP connections)       │      │
  │  │ - /proc/net (virtual, per-NS)                  │      │
  │  │ - /sys/class/net (per-NS)                      │      │
  │  │ - ARP table, neighbor cache                    │      │
  │  │ - Netfilter conntrack table                    │      │
  │  │ - UNIX domain socket bindings (abstract ns)    │      │
  │  └────────────────────────────────────────────────┘      │
  │                                                           │
  │  A new network namespace starts with:                    │
  │  - lo (loopback) interface only — DOWN by default!       │
  │  - No routes, no iptables rules                         │
  │  - Must explicitly set up connectivity                  │
  │                                                           │
  │  struct net (kernel/net/net_namespace.c):                │
  │  ┌────────────────────────────────────────────────┐      │
  │  │ struct list_head     list;       /* all net NS */    │
  │  │ struct net_device    *loopback_dev;              │    │
  │  │ struct netns_ipv4    ipv4;       /* IPv4 state */    │
  │  │ struct netns_ipv6    ipv6;       /* IPv6 state */    │
  │  │ struct netns_nf      nf;         /* netfilter  */    │
  │  │ struct sock          *rtnl;      /* rtnetlink  */    │
  │  │ struct net_generic   *gen;       /* per-ns data*/    │
  │  └────────────────────────────────────────────────┘      │
  └──────────────────────────────────────────────────────────┘
```

---

## 2. Container Networking Models

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  Model 1: veth pair + bridge (Docker default)            │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ Host namespace                                │       │
  │  │   bridge0 (172.17.0.1)                       │       │
  │  │     ├── veth-host-1 ←──► veth-ct-1 (eth0)   │       │
  │  │     │                    172.17.0.2           │       │
  │  │     └── veth-host-2 ←──► veth-ct-2 (eth0)   │       │
  │  │                          172.17.0.3           │       │
  │  │   NAT (iptables MASQUERADE) for egress       │       │
  │  │   port-forward (iptables DNAT) for ingress   │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                           │
  │  Model 2: macvlan (direct L2, no bridge)                 │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ Physical NIC (eth0)                           │       │
  │  │   ├── macvlan0 (own MAC) → container 1 NS   │       │
  │  │   └── macvlan1 (own MAC) → container 2 NS   │       │
  │  │                                               │       │
  │  │ Each gets its own MAC address on the wire     │       │
  │  │ No NAT needed — containers directly on LAN   │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                           │
  │  Model 3: ipvlan (shared MAC, L3 routing)                │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ Physical NIC (eth0) — single MAC             │       │
  │  │   ├── ipvlan0 (same MAC, own IP) → ct 1 NS  │       │
  │  │   └── ipvlan1 (same MAC, own IP) → ct 2 NS  │       │
  │  │                                               │       │
  │  │ Useful when switch limits MACs (e.g., cloud) │       │
  │  │ L2 mode: bridge-like | L3 mode: routing      │       │
  │  └──────────────────────────────────────────────┘       │
  │                                                           │
  │  Model 4: host network (--net=host)                     │
  │  ┌──────────────────────────────────────────────┐       │
  │  │ Container shares host network namespace       │       │
  │  │ No isolation — sees all host interfaces       │       │
  │  │ Best performance (no veth overhead)           │       │
  │  │ Use case: high-performance networking          │       │
  │  └──────────────────────────────────────────────┘       │
  └──────────────────────────────────────────────────────────┘
```

---

## 3. veth Pair Internals

```c
/*
 * veth pair: virtual Ethernet cable
 * Packet entering one end exits the other
 *
 * Kernel source: drivers/net/veth.c
 */

/*
 * Creating a veth pair programmatically
 * (what container runtimes do via netlink)
 */
#include <linux/if_link.h>
#include <net/if.h>

/* Using ip command (equivalent) */
// ip link add veth-host type veth peer name veth-container
// ip link set veth-container netns <container-pid>
// ip addr add 172.17.0.1/16 dev veth-host
// ip link set veth-host up
// nsenter -t <pid> -n ip addr add 172.17.0.2/16 dev veth-container
// nsenter -t <pid> -n ip link set veth-container up
// nsenter -t <pid> -n ip route add default via 172.17.0.1

/*
 * veth transmit path (kernel):
 *
 * Container sends packet via eth0 (veth-container):
 *   veth_xmit()
 *     → skb is redirected to peer device (veth-host)
 *     → netif_rx(skb) on peer → enters host network stack
 *     → bridge processes frame (if attached to bridge)
 *     → or routing + NAT (iptables) for external traffic
 *
 * XDP optimization: veth supports XDP (eBPF) at both ends
 * for high-performance packet processing without going
 * through the full network stack
 */
```

---

## 4. User Namespace Deep Dive

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  User Namespace: maps UIDs/GIDs between namespaces      │
  │                                                           │
  │  Host                    Container (User NS)             │
  │  ┌──────────┐           ┌──────────────────┐            │
  │  │UID 100000│ ◄────────►│ UID 0 (root)     │            │
  │  │UID 100001│ ◄────────►│ UID 1            │            │
  │  │UID 100002│ ◄────────►│ UID 2            │            │
  │  │ ...      │           │ ...              │            │
  │  │UID 165535│ ◄────────►│ UID 65535        │            │
  │  └──────────┘           └──────────────────┘            │
  │                                                           │
  │  /proc/<pid>/uid_map format:                             │
  │  ┌──────────────────────────────────────────┐           │
  │  │ Inside-UID  Outside-UID  Count          │           │
  │  │ 0           100000       65536          │           │
  │  └──────────────────────────────────────────┘           │
  │  Meaning: UID 0-65535 inside → UID 100000-165535 on host│
  │                                                           │
  │  /etc/subuid (host config):                              │
  │  ┌──────────────────────────────────────────┐           │
  │  │ johndoe:100000:65536                     │           │
  │  └──────────────────────────────────────────┘           │
  │  Meaning: user johndoe can use UIDs 100000-165535       │
  │            for user-namespace mappings                   │
  └──────────────────────────────────────────────────────────┘
```

---

## 5. User Namespace and Capabilities

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                           │
  │  User namespace GRANTS capabilities within child NS:    │
  │                                                           │
  │  Initial user NS (host)                                  │
  │  ├── Process has NO capabilities (unprivileged user)    │
  │  │                                                       │
  │  └── Creates child user NS                              │
  │      ├── Process has ALL capabilities inside child NS   │
  │      │   (CAP_SYS_ADMIN, CAP_NET_ADMIN, etc.)          │
  │      │                                                   │
  │      ├── Can create other namespaces                    │
  │      │   (mnt, pid, net, ipc, uts, cgroup, time)        │
  │      │                                                   │
  │      └── BUT capabilities only valid within this NS     │
  │          Cannot affect host or parent NS resources       │
  │                                                           │
  │  Ownership model:                                        │
  │  - Every non-user namespace has an owning user namespace│
  │  - Capability checks for NS operations check the        │
  │    owning user namespace, not the initial one           │
  │  - This is how rootless containers work:                │
  │    unprivileged user creates user NS → gets caps →      │
  │    creates net/mnt/pid NS using those caps              │
  │                                                           │
  │  Security restrictions:                                  │
  │  - Cannot mount block devices (even with CAP_SYS_ADMIN)│
  │  - Cannot load kernel modules                           │
  │  - Cannot access raw hardware I/O                       │
  │  - Cannot modify host network interfaces                │
  │  - /proc and /sys mostly read-only                      │
  │  - sysctl restrictions (net.* only in own net NS)       │
  └──────────────────────────────────────────────────────────┘
```

---

## 6. Network Namespace with User Namespace

```c
/*
 * Creating an isolated network setup without root
 * Using user namespace for capabilities
 */
#define _GNU_SOURCE
#include <sched.h>
#include <unistd.h>
#include <sys/wait.h>
#include <net/if.h>
#include <sys/ioctl.h>

static int child_fn(void *arg) {
    /* We are in new user NS + net NS */
    /* We have CAP_NET_ADMIN in this net namespace */

    /* Bring up loopback */
    int sock = socket(AF_INET, SOCK_DGRAM, 0);
    struct ifreq ifr = {0};
    strncpy(ifr.ifr_name, "lo", IFNAMSIZ);
    ifr.ifr_flags |= IFF_UP;
    ioctl(sock, SIOCSIFFLAGS, &ifr);
    close(sock);

    /* Now we have our own network stack */
    /* Parent must create veth pair and move one end in */
    execl("/bin/sh", "sh", NULL);
    return 0;
}

int main(void) {
    char stack[65536];

    pid_t pid = clone(child_fn,
                      stack + sizeof(stack),
                      CLONE_NEWUSER | CLONE_NEWNET | SIGCHLD,
                      NULL);

    /* Write UID mapping: container root → our UID */
    char path[64], map[64];
    snprintf(path, sizeof(path), "/proc/%d/uid_map", pid);
    snprintf(map, sizeof(map), "0 %d 1\n", getuid());
    int fd = open(path, O_WRONLY);
    write(fd, map, strlen(map));
    close(fd);

    /* Deny setgroups (required before writing gid_map) */
    snprintf(path, sizeof(path), "/proc/%d/setgroups", pid);
    fd = open(path, O_WRONLY);
    write(fd, "deny\n", 5);
    close(fd);

    /* Write GID mapping */
    snprintf(path, sizeof(path), "/proc/%d/gid_map", pid);
    snprintf(map, sizeof(map), "0 %d 1\n", getgid());
    fd = open(path, O_WRONLY);
    write(fd, map, strlen(map));
    close(fd);

    waitpid(pid, NULL, 0);
    return 0;
}
```

---

## Interview Questions

**Q1: How does container networking work with veth pairs and bridges?**
**A:** Container networking uses three components: (1) **veth pair** — a virtual Ethernet cable with two ends. One end (`veth-host`) stays in the host network namespace, the other (`veth-ct`) is moved to the container's network namespace (where it becomes `eth0`). Packets entering one end exit the other (kernel `veth_xmit()` redirects the skb to the peer device). (2) **bridge** — a software L2 switch in the host namespace. All host-side veth ends are attached to it (e.g., `docker0` or `cni0`). The bridge forwards frames between containers on the same host. (3) **NAT/routing** — iptables MASQUERADE rule translates container source IPs to the host IP for outgoing traffic. DNAT rules handle port forwarding for incoming traffic. Alternatives: macvlan (each container gets its own MAC, directly on physical network), ipvlan (shared MAC, L3 routing), host networking (no isolation, best performance), and overlay networks (VXLAN encapsulation for cross-host container communication in Kubernetes/Swarm).

**Q2: How do user namespaces enable rootless containers, and what are the security restrictions?**
**A:** User namespaces remap UIDs: an unprivileged host user (e.g., UID 1000) creates a user namespace where they are mapped to UID 0 (root). Inside this namespace, they have ALL capabilities (CAP_SYS_ADMIN, CAP_NET_ADMIN, etc.), which lets them create other namespaces (mount, PID, network, etc.) — operations that normally require root. This is how rootless containers work: the container runtime runs as an unprivileged user, creates user NS → gains caps → creates other NS → runs container. However, the kernel imposes security restrictions: (1) Cannot mount block devices (even with CAP_SYS_ADMIN in user NS). (2) Cannot load/unload kernel modules. (3) Cannot access raw hardware I/O (iopl/ioperm). (4) Cannot modify host network interfaces (caps only valid in own net NS). (5) Many /proc and /sys writes are blocked. (6) The `uid_map`/`gid_map` files can only be written once, and `setgroups` must be denied before writing `gid_map`. These restrictions ensure that user namespace capabilities cannot be used to escalate privileges on the host.

---

## Summary

- Network namespace: isolated network stack (interfaces, routes, iptables, sockets)
- veth pair: virtual cable; one end in host NS (on bridge), other in container NS
- Container networking: bridge + veth + NAT (default), or macvlan/ipvlan/host
- User namespace: UID/GID remapping; container root → host unprivileged user
- User NS grants all capabilities inside child NS but with kernel restrictions
- Rootless containers: user NS → caps → create other NS without root
- /proc/PID/uid_map: "0 100000 65536" maps container UIDs to host UIDs

---

[Previous: PID and Mount Namespaces ←](Chapter_07_PID_Mount_NS.md) | [Next: UTS, IPC, Cgroup, Time Namespaces →](Chapter_09_Other_Namespaces.md)
