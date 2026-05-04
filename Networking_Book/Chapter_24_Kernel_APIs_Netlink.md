# Chapter 24: Kernel Networking APIs and Netlink

## Learning Goals
- Understand in-kernel socket API for kernel networking
- Know Netlink protocol for kernel-userspace communication
- Understand Generic Netlink subsystem
- Know rtnetlink for route and link management

---

## 24.1 In-Kernel Socket API

```
Kernel modules can create and use sockets directly
without going through the syscall interface.

Functions:
  sock_create_kern(net, AF_INET, SOCK_STREAM, IPPROTO_TCP, &sock)
  kernel_bind(sock, (struct sockaddr *)&addr, sizeof(addr))
  kernel_listen(sock, backlog)
  kernel_accept(sock, &new_sock, flags)
  kernel_connect(sock, (struct sockaddr *)&addr, sizeof(addr), flags)
  kernel_sendmsg(sock, &msg, &kvec, 1, len)
  kernel_recvmsg(sock, &msg, &kvec, 1, len, flags)
  sock_release(sock)

Use cases:
  - NFS client/server
  - iSCSI target
  - Kernel-space TCP proxy
  - Cluster communication (OCFS2, DLM)
  - Kernel TLS (kTLS) for TLS offload
```

```c
/* Example: kernel TCP client */
#include <linux/net.h>
#include <linux/in.h>

struct socket *sock;
struct sockaddr_in addr;
struct kvec iov;
struct msghdr msg = {};

sock_create_kern(&init_net, AF_INET, SOCK_STREAM, IPPROTO_TCP, &sock);

addr.sin_family = AF_INET;
addr.sin_port = htons(8080);
addr.sin_addr.s_addr = in_aton("10.0.0.1");

kernel_connect(sock, (struct sockaddr *)&addr, sizeof(addr), 0);

iov.iov_base = "Hello";
iov.iov_len = 5;
kernel_sendmsg(sock, &msg, &iov, 1, 5);

sock_release(sock);
```

---

## 24.2 Netlink Protocol

```
Netlink: Socket-based IPC between kernel and userspace.
Used for networking configuration (routes, links, addresses, rules).

Why not ioctl?
  - ioctl: synchronous, limited data, no multicast
  - Netlink: async, large messages, multicast groups, dump support

Netlink families (AF_NETLINK):
  NETLINK_ROUTE     — Routing/link/address management (rtnetlink)
  NETLINK_FIREWALL  — Deprecated (use NFNETLINK)
  NETLINK_NETFILTER — Netfilter subsystem (conntrack, nftables)
  NETLINK_KOBJECT_UEVENT — udev events
  NETLINK_GENERIC   — Generic netlink (extensible)
  NETLINK_AUDIT     — Audit subsystem
  NETLINK_XFRM      — IPsec policy and state

Netlink message format:
  ┌─────────────────────────────────────────────────────┐
  │  struct nlmsghdr (16 bytes)                         │
  │  ┌─────────────┬──────────┬────────┬───────────┐    │
  │  │ nlmsg_len   │ nlmsg_type│nlmsg_  │ nlmsg_    │   │
  │  │ (4 bytes)   │ (2 bytes) │flags   │ seq/pid   │   │
  │  │ total length│ RTM_*, etc│(2 bytes)│(4+4 bytes)│   │
  │  └─────────────┴──────────┴────────┴───────────┘    │
  │                                                     │
  │  Payload (family-specific header)                    │
  │  ┌──────────────────────────────────────────┐       │
  │  │ e.g., struct ifinfomsg (link info)        │       │
  │  │      struct ifaddrmsg (address info)      │       │
  │  │      struct rtmsg (route info)            │       │
  │  └──────────────────────────────────────────┘       │
  │                                                     │
  │  Attributes (TLV: Type-Length-Value)                 │
  │  ┌────────┬────────┬────────┬────────┐              │
  │  │ nla_len│nla_type│ value  │ padding│ (repeated)    │
  │  └────────┴────────┴────────┴────────┘              │
  └─────────────────────────────────────────────────────┘

  Flags:
    NLM_F_REQUEST  — Request message
    NLM_F_MULTI    — Multipart message
    NLM_F_DUMP     — Dump all entries
    NLM_F_CREATE   — Create new entry
    NLM_F_EXCL     — Error if exists
    NLM_F_REPLACE  — Replace existing
    NLM_F_ACK      — Request acknowledgment
```

---

## 24.3 rtnetlink (Route Netlink)

```
rtnetlink: NETLINK_ROUTE family.
Used by iproute2 tools (ip link, ip addr, ip route).

Message types:
  Links:     RTM_NEWLINK, RTM_DELLINK, RTM_GETLINK
  Addresses: RTM_NEWADDR, RTM_DELADDR, RTM_GETADDR
  Routes:    RTM_NEWROUTE, RTM_DELROUTE, RTM_GETROUTE
  Neighbors: RTM_NEWNEIGH, RTM_DELNEIGH, RTM_GETNEIGH
  Rules:     RTM_NEWRULE, RTM_DELRULE, RTM_GETRULE

How "ip addr show" works:
  1. Open AF_NETLINK, SOCK_DGRAM, NETLINK_ROUTE socket
  2. Send RTM_GETADDR with NLM_F_DUMP flag
  3. Kernel iterates all addresses, sends RTM_NEWADDR messages
  4. Each message contains: ifaddrmsg + attributes (IFA_ADDRESS, IFA_LABEL, etc.)
  5. Multi-part response ends with NLMSG_DONE

How "ip route add" works:
  1. Open NETLINK_ROUTE socket
  2. Send RTM_NEWROUTE with NLM_F_CREATE
  3. Payload: struct rtmsg + attributes (RTA_DST, RTA_GATEWAY, RTA_OIF)
  4. Kernel adds route to FIB table
  5. Returns NLMSG_ERROR (error code 0 = success)
```

```c
/* Userspace: add a route using netlink */
#include <linux/netlink.h>
#include <linux/rtnetlink.h>

struct {
    struct nlmsghdr  nlh;
    struct rtmsg     rtm;
    char             attrbuf[512];
} req;

int sock = socket(AF_NETLINK, SOCK_DGRAM, NETLINK_ROUTE);

memset(&req, 0, sizeof(req));
req.nlh.nlmsg_len   = NLMSG_LENGTH(sizeof(struct rtmsg));
req.nlh.nlmsg_type  = RTM_NEWROUTE;
req.nlh.nlmsg_flags = NLM_F_REQUEST | NLM_F_CREATE | NLM_F_ACK;
req.rtm.rtm_family  = AF_INET;
req.rtm.rtm_dst_len = 24;             /* /24 prefix */
req.rtm.rtm_table   = RT_TABLE_MAIN;
req.rtm.rtm_protocol = RTPROT_STATIC;
req.rtm.rtm_scope   = RT_SCOPE_UNIVERSE;
req.rtm.rtm_type    = RTN_UNICAST;

/* Add RTA_DST attribute: 10.0.0.0 */
uint32_t dst = inet_addr("10.0.0.0");
rtattr_add(&req.nlh, sizeof(req), RTA_DST, &dst, 4);

/* Add RTA_GATEWAY attribute: 192.168.1.1 */
uint32_t gw = inet_addr("192.168.1.1");
rtattr_add(&req.nlh, sizeof(req), RTA_GATEWAY, &gw, 4);

send(sock, &req, req.nlh.nlmsg_len, 0);
```

---

## 24.4 Generic Netlink (genetlink)

```
Generic Netlink: Extensible netlink framework for new families.
Avoids allocating fixed NETLINK_* protocol numbers.

Architecture:
  ┌─────────────────────────────────────────────┐
  │  Userspace                                  │
  │    libnl / libmnl / raw socket              │
  │      │                                      │
  │  ┌───┴───────────────────────────┐          │
  │  │ NETLINK_GENERIC socket        │          │
  │  └───┬───────────────────────────┘          │
  │      │                                      │
  │  ┌───┴───────────────────────────┐          │
  │  │ Generic Netlink Controller    │          │
  │  │ (genl_ctrl, family_id=0x10)   │          │
  │  │   Resolves family name → ID   │          │
  │  └───┬───────────────────────────┘          │
  │      │                                      │
  │  ┌───┴──────┐  ┌───────────┐  ┌──────────┐ │
  │  │ Family A │  │ Family B  │  │ Family C │ │
  │  │(nl80211) │  │(taskstats)│  │(custom)  │ │
  │  └──────────┘  └───────────┘  └──────────┘ │
  └─────────────────────────────────────────────┘

  Steps:
  1. Kernel module registers genl_family with ops
  2. Userspace queries controller for family ID
  3. Userspace sends commands with family ID
  4. Kernel dispatches to registered handler
```

```c
/* Kernel-side: register generic netlink family */
#include <net/genetlink.h>

enum my_cmd {
    MY_CMD_SET = 1,
    MY_CMD_GET,
};

enum my_attr {
    MY_ATTR_UNSPEC,
    MY_ATTR_DATA,
    __MY_ATTR_MAX,
};

static const struct nla_policy my_policy[__MY_ATTR_MAX] = {
    [MY_ATTR_DATA] = { .type = NLA_U32 },
};

static int my_cmd_set(struct sk_buff *skb, struct genl_info *info)
{
    u32 val;
    if (!info->attrs[MY_ATTR_DATA])
        return -EINVAL;
    val = nla_get_u32(info->attrs[MY_ATTR_DATA]);
    pr_info("Received value: %u\n", val);
    return 0;
}

static const struct genl_small_ops my_ops[] = {
    {
        .cmd    = MY_CMD_SET,
        .doit   = my_cmd_set,
        .policy = my_policy,
    },
};

static struct genl_family my_family = {
    .name    = "MY_GENL",
    .version = 1,
    .maxattr = __MY_ATTR_MAX - 1,
    .ops     = my_ops,
    .n_ops   = ARRAY_SIZE(my_ops),
};

/* In module init: */
genl_register_family(&my_family);
```

---

## 24.5 Netlink Libraries

```
libnl (netlink library suite):
  libnl-3     — Core netlink functions
  libnl-route — rtnetlink (links, routes, addresses)
  libnl-genl  — Generic netlink
  libnl-nf    — Netfilter subsystem

libmnl (minimalistic netlink):
  Lighter alternative to libnl
  Provides message building/parsing helpers

pyroute2 (Python):
  from pyroute2 import IPRoute
  ip = IPRoute()
  ip.link('set', index=2, state='up')
  ip.addr('add', index=2, address='10.0.0.1', prefixlen=24)
  ip.route('add', dst='192.168.0.0/24', gateway='10.0.0.1')
  ip.close()

iproute2 (userspace tools):
  ip link show     → RTM_GETLINK
  ip addr add      → RTM_NEWADDR
  ip route add     → RTM_NEWROUTE
  ip neigh show    → RTM_GETNEIGH
  ip rule add      → RTM_NEWRULE
```

---

## 24.6 Notification and Multicast

```
Netlink multicast: kernel broadcasts events to subscribers.

Multicast groups (rtnetlink):
  RTNLGRP_LINK      — Link state changes
  RTNLGRP_IPV4_ADDR — IPv4 address changes
  RTNLGRP_IPV4_ROUTE— IPv4 route changes
  RTNLGRP_NEIGH     — Neighbor table changes

Subscribing:
  struct sockaddr_nl addr = {
      .nl_family = AF_NETLINK,
      .nl_groups = RTMGRP_LINK | RTMGRP_IPV4_IFADDR,
  };
  bind(sock, (struct sockaddr *)&addr, sizeof(addr));

  // Now recv() will get RTM_NEWLINK, RTM_DELLINK,
  // RTM_NEWADDR, RTM_DELADDR events

Use case: Network manager daemon monitoring link and address changes
  ip monitor link → prints link up/down events in real time
  ip monitor route → prints route changes in real time
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/netlink/af_netlink.c | Netlink socket family |
| net/netlink/genetlink.c | Generic netlink framework |
| net/core/rtnetlink.c | rtnetlink implementation |
| include/linux/netlink.h | Netlink message structures |
| include/uapi/linux/rtnetlink.h | rtnetlink constants |
| include/net/genetlink.h | Generic netlink API |
| lib/nlattr.c | Netlink attribute helpers |
| net/ipv4/fib_frontend.c | FIB netlink interface |
| net/ipv4/devinet.c | Address netlink interface |

---

## Interview Questions

**Q1: What is Netlink and why is it used instead of ioctl?**
A: Netlink is a socket-based IPC mechanism between kernel and userspace (AF_NETLINK). Advantages over ioctl: (1) Asynchronous — supports non-blocking with poll/epoll. (2) Large messages — no ioctl buffer size constraints. (3) Multicast — kernel can broadcast events (link up/down, route changes) to multiple listeners. (4) Structured format — TLV attributes for extensibility. (5) Dump support — enumerate all entries with NLM_F_DUMP. The iproute2 tools (ip link, ip route) all use rtnetlink.

**Q2: How does Generic Netlink work?**
A: Generic Netlink multiplexes multiple families over a single NETLINK_GENERIC protocol number. A kernel module registers a genl_family with a string name and operations. Userspace first queries the genl controller (family_id 0x10) to resolve the family name to a numeric ID. Then messages are sent with that ID. Each family defines commands (ops) with handlers, and attributes with a policy for validation. This avoids the need for new NETLINK_* constants.

**Q3: How would you monitor network events in userspace?**
A: Open a NETLINK_ROUTE socket and bind to multicast groups (RTMGRP_LINK, RTMGRP_IPV4_IFADDR, RTMGRP_IPV4_ROUTE). The kernel sends notifications when links change state (RTM_NEWLINK/DELLINK), addresses are added/removed (RTM_NEWADDR/DELADDR), or routes change (RTM_NEWROUTE/DELROUTE). Parse the nlmsghdr + payload + attributes. The `ip monitor` command does exactly this. Libraries like libnl or pyroute2 simplify parsing.

---

## Summary

- Kernel socket API: sock_create_kern, kernel_sendmsg for in-kernel networking
- Netlink: AF_NETLINK socket IPC, replaces ioctl for network configuration
- rtnetlink: NETLINK_ROUTE — manages links, addresses, routes, neighbors, rules
- Generic Netlink: extensible framework for new families (nl80211, taskstats)
- Netlink message format: nlmsghdr + family header + TLV attributes
- Multicast groups enable event notification (link state, route changes)
- iproute2 tools translate commands to netlink messages

---

Next: [Chapter 25 — Embedded and Automotive Networking](Chapter_25_Embedded_Automotive.md)
