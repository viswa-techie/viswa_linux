# Chapter 20: Netfilter and Packet Filtering

## Learning Goals
- Understand netfilter hook points and their position in the packet path
- Know iptables tables, chains, targets, and rule evaluation
- Understand nftables and its advantages over iptables
- Know connection tracking states and usage

---

## 20.1 Netfilter Architecture

```
Netfilter: Framework of hooks in the kernel network stack.
Allows modules to inspect, modify, drop, or queue packets at defined points.

Hook Points (IPv4):
  ┌──────────────── Packet Flow with Netfilter Hooks ────────────────┐
  │                                                                   │
  │  [Incoming Packet]                                                │
  │       │                                                           │
  │       ▼                                                           │
  │  NF_INET_PRE_ROUTING  ─── conntrack, DNAT                       │
  │       │                                                           │
  │       ├── Routing Decision ──┐                                    │
  │       │                      │                                    │
  │       ▼                      ▼                                    │
  │  (local delivery)      (forwarding)                               │
  │       │                      │                                    │
  │       ▼                      ▼                                    │
  │  NF_INET_LOCAL_IN      NF_INET_FORWARD                           │
  │       │                      │                                    │
  │       ▼                      ▼                                    │
  │  [Local Process]        NF_INET_POST_ROUTING ── SNAT              │
  │       │                      │                                    │
  │       ▼                      ▼                                    │
  │  NF_INET_LOCAL_OUT     [Outgoing Packet]                          │
  │       │                                                           │
  │       ▼                                                           │
  │  Routing Decision                                                 │
  │       │                                                           │
  │       ▼                                                           │
  │  NF_INET_POST_ROUTING  ── SNAT                                   │
  │       │                                                           │
  │       ▼                                                           │
  │  [Outgoing Packet]                                                │
  └───────────────────────────────────────────────────────────────────┘

Hook return values:
  NF_ACCEPT  — Continue to next hook/processing
  NF_DROP    — Drop packet, free skb
  NF_STOLEN  — Hook took ownership of skb (module handles it)
  NF_QUEUE   — Pass to userspace via nfqueue
  NF_REPEAT  — Call this hook again
```

---

## 20.2 iptables Tables and Chains

```
iptables organizes rules into tables, each containing chains.

Tables (in order of evaluation):
  ┌──────────────────────────────────────────────────────────────────┐
  │ Table     │ Purpose                │ Chains                      │
  ├───────────┼────────────────────────┼─────────────────────────────┤
  │ raw       │ Skip conntrack         │ PREROUTING, OUTPUT          │
  │ mangle    │ Modify packets (TOS,   │ All 5 chains                │
  │           │ TTL, mark)             │                             │
  │ nat       │ Address translation    │ PREROUTING, INPUT, OUTPUT,  │
  │           │                        │ POSTROUTING                 │
  │ filter    │ Accept/Drop decision   │ INPUT, FORWARD, OUTPUT      │
  │ security  │ SELinux/mandatory      │ INPUT, FORWARD, OUTPUT      │
  └───────────┴────────────────────────┴─────────────────────────────┘

Rule evaluation:
  Table processing order per hook:
    PRE_ROUTING:   raw → mangle → nat (DNAT)
    LOCAL_IN:      mangle → filter → security → nat
    FORWARD:       mangle → filter → security
    LOCAL_OUT:     raw → mangle → nat (DNAT) → filter → security
    POST_ROUTING:  mangle → nat (SNAT)

  Within a chain: rules evaluated top-to-bottom until match.
  Default policy (ACCEPT/DROP) if no rule matches.
```

---

## 20.3 iptables Rule Syntax

```bash
# General syntax:
iptables -t <table> -A <chain> <match> -j <target>

# Common matches:
-p tcp/udp/icmp             # Protocol
-s 192.168.1.0/24            # Source IP/network
-d 10.0.0.1                  # Destination IP
-i eth0                      # Input interface
-o eth1                      # Output interface
--dport 80                   # Destination port
--sport 1024:65535           # Source port range
-m state --state ESTABLISHED # Connection state
-m multiport --dports 80,443 # Multiple ports
-m limit --limit 10/min     # Rate limit

# Targets:
-j ACCEPT      # Allow packet
-j DROP        # Drop silently
-j REJECT      # Drop + send ICMP error
-j LOG         # Log to kernel log
-j SNAT        # Source NAT
-j DNAT        # Destination NAT
-j MASQUERADE  # Dynamic SNAT
-j MARK        # Set packet mark

# Examples:
# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH from specific subnet
iptables -A INPUT -p tcp -s 10.0.0.0/8 --dport 22 -j ACCEPT

# Block all other incoming traffic
iptables -P INPUT DROP

# Port forwarding
iptables -t nat -A PREROUTING -p tcp --dport 8080 \
         -j DNAT --to-destination 192.168.1.10:80

# Rate limit ICMP
iptables -A INPUT -p icmp --icmp-type echo-request \
         -m limit --limit 1/s -j ACCEPT

# Log and drop
iptables -A INPUT -j LOG --log-prefix "DROPPED: "
iptables -A INPUT -j DROP
```

---

## 20.4 nftables (Modern Replacement)

```
nftables: Replaces iptables/ip6tables/arptables/ebtables.
Uses nf_tables kernel subsystem (net/netfilter/nf_tables*.c).

Advantages over iptables:
  1. Single framework for IPv4, IPv6, ARP, bridge
  2. Atomic rule replacement (batch updates)
  3. Better performance (set-based matching, maps)
  4. No module loading per match/target
  5. Cleaner syntax
  6. Support for sets, maps, concatenations

# nft command syntax:
nft add table inet filter
nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input tcp dport 22 accept
nft add rule inet filter input tcp dport { 80, 443 } accept
nft list ruleset

# Set-based matching (efficient):
nft add set inet filter allowed_ports { type inet_service \; }
nft add element inet filter allowed_ports { 22, 80, 443, 8080 }
nft add rule inet filter input tcp dport @allowed_ports accept

# Map-based decisions:
nft add map inet filter port_verdict { type inet_service : verdict \; }
nft add element inet filter port_verdict { 22 : accept, 80 : accept, 23 : drop }
nft add rule inet filter input tcp dport vmap @port_verdict
```

iptables → nftables translation:

```
iptables                          nftables
─────────────────────────────────────────────────────
-A INPUT                          add rule inet filter input
-p tcp --dport 80                 tcp dport 80
-s 192.168.1.0/24                 ip saddr 192.168.1.0/24
-j ACCEPT                        accept
-j DROP                           drop
-m state --state ESTABLISHED      ct state established
-m multiport --dports 80,443      tcp dport { 80, 443 }
iptables -L                       nft list ruleset
```

---

## 20.5 Connection Tracking Deep Dive

```
conntrack: Stateful packet inspection for netfilter.

State machine:
  ┌──────────────────────────────────────────────────┐
  │                                                  │
  │  [NEW] ──── first packet ───► [ESTABLISHED]      │
  │    │                              │               │
  │    │         related ────────► [RELATED]          │
  │    │         connection           │               │
  │    │                              │               │
  │    └── timeout/invalid ──► [INVALID]              │
  │                                                  │
  └──────────────────────────────────────────────────┘

  NEW:         First packet of connection
  ESTABLISHED: Reply seen in both directions
  RELATED:     New connection related to ESTABLISHED
               (e.g., ICMP error, FTP data)
  INVALID:     Doesn't match any known connection
  UNTRACKED:   Bypassed via raw table NOTRACK

Implementation:
  nf_conntrack_tuple:
    Source: {IP, port, L3proto}
    Destination: {IP, port, L3proto, L4proto}

  Hash table of nf_conn entries:
    - Tuple in both directions (original + reply)
    - Timeout, status bits, NAT info
    - Extension area (helpers, NAT, labels)

  conntrack helpers: Protocol-specific (FTP, SIP, TFTP)
    Track L7 protocol to create RELATED expectations
    e.g., FTP PORT command → expect data connection

Tuning:
  sysctl net.netfilter.nf_conntrack_max = 262144
  sysctl net.netfilter.nf_conntrack_tcp_timeout_established = 432000
  # Exhaustion → new connections dropped ("nf_conntrack: table full")
```

---

## 20.6 Netfilter Kernel Internals

```c
/* Registering a netfilter hook */
static unsigned int my_hook_fn(void *priv,
                                struct sk_buff *skb,
                                const struct nf_hook_state *state)
{
    struct iphdr *iph = ip_hdr(skb);

    if (iph->protocol == IPPROTO_ICMP) {
        pr_info("ICMP packet from %pI4\n", &iph->saddr);
        return NF_DROP;
    }
    return NF_ACCEPT;
}

static struct nf_hook_ops my_hook = {
    .hook     = my_hook_fn,
    .pf       = NFPROTO_IPV4,
    .hooknum  = NF_INET_PRE_ROUTING,
    .priority = NF_IP_PRI_FIRST,
};

/* Registration in module init */
nf_register_net_hook(&init_net, &my_hook);
```

Hook priorities (evaluated low → high):

```
NF_IP_PRI_RAW_BEFORE_DEFRAG  = -450
NF_IP_PRI_CONNTRACK_DEFRAG   = -400
NF_IP_PRI_RAW                = -300
NF_IP_PRI_SELINUX_FIRST      = -225
NF_IP_PRI_CONNTRACK          = -200
NF_IP_PRI_MANGLE             = -150
NF_IP_PRI_NAT_DST            = -100  (DNAT)
NF_IP_PRI_FILTER             = 0
NF_IP_PRI_SECURITY           = 50
NF_IP_PRI_NAT_SRC            = 100   (SNAT)
NF_IP_PRI_SELINUX_LAST       = 225
NF_IP_PRI_CONNTRACK_HELPER   = 300
NF_IP_PRI_CONNTRACK_CONFIRM  = INT_MAX
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/netfilter/core.c | Netfilter hook registration |
| net/netfilter/nf_conntrack_core.c | Connection tracking |
| net/netfilter/nf_nat_core.c | NAT core |
| net/netfilter/nf_tables_api.c | nftables API |
| net/ipv4/netfilter/iptable_filter.c | iptables filter table |
| net/ipv4/netfilter/nf_reject_ipv4.c | REJECT target |
| include/linux/netfilter.h | Hook definitions |
| include/uapi/linux/netfilter.h | Hook numbers, verdicts |

---

## Interview Questions

**Q1: What are the five netfilter hooks and when is each invoked?**
A: (1) NF_INET_PRE_ROUTING — immediately after IP header validation, before routing decision; used for DNAT. (2) NF_INET_LOCAL_IN — after routing decides packet is for this host; used for INPUT filtering. (3) NF_INET_FORWARD — after routing decides packet should be forwarded; used for FORWARD filtering. (4) NF_INET_LOCAL_OUT — for locally generated packets before routing; used for OUTPUT filtering. (5) NF_INET_POST_ROUTING — just before packet leaves the host; used for SNAT.

**Q2: How does connection tracking enable stateful firewalling?**
A: Conntrack maintains a hash table of seen connections (tuples: src IP/port, dst IP/port, protocol). The first packet is NEW. When a reply is seen, state becomes ESTABLISHED. Related connections (like ICMP errors or FTP data channels) are RELATED. Firewall rules can then allow ESTABLISHED,RELATED traffic while blocking NEW incoming connections, implementing a stateful firewall that only allows replies to outgoing connections.

**Q3: What advantages does nftables have over iptables?**
A: (1) Single framework for L2-L4 (replaces iptables, ip6tables, arptables, ebtables). (2) Atomic rule updates — batch entire ruleset replacement. (3) Better performance with set-based matching and maps. (4) No kernel module per match/target — uses generic expressions. (5) Cleaner syntax with native support for sets and concatenated matches. (6) In-kernel bytecode VM for efficient rule evaluation.

**Q4: What is the "conntrack table full" problem and how do you solve it?**
A: When nf_conntrack_max is reached, new connections are dropped. Symptoms: connection failures, "nf_conntrack: table full, dropping packet" in dmesg. Solutions: (1) Increase nf_conntrack_max. (2) Reduce timeouts (nf_conntrack_tcp_timeout_established). (3) Use raw table NOTRACK for high-volume traffic that doesn't need tracking. (4) Increase hash table size (nf_conntrack_buckets = max/4).

---

## Summary

- Netfilter provides 5 hook points: PRE_ROUTING, LOCAL_IN, FORWARD, LOCAL_OUT, POST_ROUTING
- iptables organizes rules into tables (raw, mangle, nat, filter, security) and chains
- nftables replaces iptables with atomic updates, sets, maps, unified framework
- Connection tracking enables stateful firewalling with NEW/ESTABLISHED/RELATED states
- NAT relies on connection tracking to reverse-translate return traffic
- Hook priorities determine evaluation order at each hook point

---

Next: [Chapter 21 — Traffic Control and QoS](Chapter_21_Traffic_Control_QoS.md)
