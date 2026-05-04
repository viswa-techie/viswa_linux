# Chapter 19: Routing and Forwarding

## Learning Goals
- Understand FIB, routing tables, and route lookup
- Know policy routing and multiple tables
- Understand NAT and connection tracking
- Know IP forwarding configuration and flow

---

## 19.1 Routing Architecture

```
┌─────────────────────── Routing Subsystem ──────────────────────┐
│                                                                 │
│  RPDB (Routing Policy Database):                               │
│    ip rule list:                                                │
│      0:    from all lookup local                                │
│      32766: from all lookup main                                │
│      32767: from all lookup default                             │
│                                                                 │
│  FIB Tables:                                                    │
│    local  (255): Local addresses, broadcasts                   │
│    main   (254): Default routing table                         │
│    default(253): Post-processing                               │
│    custom tables: policy routing (1-252)                        │
│                                                                 │
│  Data Structure: LC-Trie (Level-Compressed Trie)                │
│    O(log n) longest-prefix match                               │
│    net/ipv4/fib_trie.c                                         │
│                                                                 │
│  Route Cache: dst_entry attached to sk_buff                    │
│    Contains: output device, gateway, MTU, metrics              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 19.2 Routing Lookup Flow

```
ip_route_output_flow(net, flowi4)  [TX: find route for outgoing packet]
ip_route_input_slow(skb, ...)      [RX: determine local/forward/drop]
  │
  ▼
fib_lookup(net, fl4, &res)
  │
  ├── For each rule in RPDB (ordered by priority):
  │     if (rule matches packet):
  │       table = rule->table
  │       result = fib_table_lookup(table, fl4)
  │       if (result found): return result
  │
  ▼
Result: struct fib_result
  ├── type:      RTN_LOCAL (deliver locally)
  │              RTN_UNICAST (forward to gateway)
  │              RTN_BROADCAST (broadcast)
  │              RTN_BLACKHOLE (drop silently)
  │              RTN_UNREACHABLE (ICMP unreachable)
  │              RTN_PROHIBIT (ICMP admin prohibited)
  ├── fi:        fib_info (gateway, device, metrics)
  ├── nhsel:     next-hop selection (for ECMP)
  └── prefixlen: matched prefix length
```

---

## 19.3 IP Forwarding

```
Enable forwarding:
  sysctl net.ipv4.ip_forward = 1
  # Or per-interface:
  sysctl net.ipv4.conf.eth0.forwarding = 1

Forwarding path:
  ip_rcv() → ip_rcv_finish() → route lookup
    → type == RTN_UNICAST && !local
    → ip_forward(skb)
      ├── Check TTL > 1
      │   If TTL == 1 → send ICMP Time Exceeded, drop
      ├── TTL--
      ├── Check packet size vs output MTU
      │   If too large && DF set → ICMP Frag Needed, drop
      ├── NF_INET_FORWARD (netfilter hook)
      ├── ip_forward_finish()
      │   └── ip_output()
      │       └── NF_INET_POST_ROUTING
      │           └── ip_finish_output()
      │               └── neigh_output() → transmit
      └── Done

  Simple router:
    ┌──────┐     ┌──────────┐     ┌──────┐
    │Host A│────►│  Linux   │────►│Host B│
    │.1.10 │    ►│  Router  │    ►│.2.10 │
    └──────┘  eth0 │ fwd=1 │ eth1 └──────┘
            192.168.1.x    192.168.2.x
```

---

## 19.4 NAT (Network Address Translation)

```
NAT: Modify source or destination IP/port in transit.

SNAT (Source NAT / MASQUERADE):
  Internal host → Internet
  Change source IP to router's public IP

  Before:  src=192.168.1.10:5000 → dst=8.8.8.8:53
  After:   src=203.0.113.1:45678 → dst=8.8.8.8:53
  Reply:   src=8.8.8.8:53 → dst=203.0.113.1:45678
  De-NAT:  src=8.8.8.8:53 → dst=192.168.1.10:5000

  iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

DNAT (Destination NAT / Port Forwarding):
  Internet → Internal host
  Change destination IP/port to internal host

  Before:  src=1.2.3.4:9999 → dst=203.0.113.1:80
  After:   src=1.2.3.4:9999 → dst=192.168.1.10:8080

  iptables -t nat -A PREROUTING -p tcp --dport 80 \
           -j DNAT --to 192.168.1.10:8080
```

### Connection Tracking (conntrack)

```
NAT requires tracking connections to reverse-translate replies.

conntrack table:
  ┌──────────────────────────────────────────────────────────────┐
  │ Proto │ Src IP:Port      │ Dst IP:Port     │ State    │ NAT │
  ├───────┼──────────────────┼─────────────────┼──────────┼─────┤
  │ TCP   │ 192.168.1.10:5000│ 8.8.8.8:53      │ ESTABLISH│ SNAT│
  │ UDP   │ 192.168.1.20:4000│ 1.1.1.1:443     │ ASSURED  │ SNAT│
  └───────┴──────────────────┴─────────────────┴──────────┴─────┘

  conntrack -L    # List entries
  conntrack -C    # Count active connections

  sysctl net.netfilter.nf_conntrack_max = 131072  # Max entries

  States: NEW, ESTABLISHED, RELATED, INVALID
  RELATED: new connection related to existing (e.g., FTP data channel)
```

---

## 19.5 Policy Routing

```bash
# Multiple routing tables for source-based routing
# Example: traffic from 10.0.0.0/8 uses alternate gateway

# Create custom table
echo "100 custom" >> /etc/iproute2/rt_tables

# Add rule: traffic from 10.0.0.0/8 → lookup table "custom"
ip rule add from 10.0.0.0/8 table custom

# Add route in custom table
ip route add default via 10.255.0.1 table custom

# Result: 10.x.x.x traffic follows custom routes
#         Other traffic follows main table

# Multi-homed server (two ISPs):
ip rule add from 1.2.3.4 table isp1    # ISP1 for traffic from ISP1 IP
ip rule add from 5.6.7.8 table isp2    # ISP2 for traffic from ISP2 IP
ip route add default via 1.2.3.1 table isp1
ip route add default via 5.6.7.1 table isp2
```

---

## 19.6 ECMP (Equal-Cost Multi-Path)

```bash
# Multiple next-hops for load balancing
ip route add 10.0.0.0/8 \
   nexthop via 192.168.1.1 weight 1 \
   nexthop via 192.168.1.2 weight 1

# Kernel distributes flows across next-hops by hash
# Same flow → same path (consistent hashing)
# Different flows → potentially different paths
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/ipv4/route.c | Route lookup, cache |
| net/ipv4/fib_trie.c | FIB trie data structure |
| net/ipv4/fib_rules.c | Policy routing rules (RPDB) |
| net/ipv4/fib_semantics.c | Route metrics, nexthops |
| net/ipv4/ip_forward.c | IP forwarding |
| net/netfilter/nf_nat_core.c | NAT implementation |
| net/netfilter/nf_conntrack_core.c | Connection tracking |

---

## Interview Questions

**Q1: How does Linux perform a route lookup?**
A: The RPDB (Routing Policy Database) is consulted — a list of rules checked in priority order. Each rule specifies conditions (source IP, fwmark, interface) and a table to search. For each matching rule, the FIB table (LC-Trie) performs a longest-prefix match on the destination IP. The first match provides the output device, gateway, and type (local, unicast, blackhole). Default rules check: local table (own addresses) → main table → default table.

**Q2: What is NAT and how does Linux implement it?**
A: NAT modifies IP addresses/ports in packets. SNAT (source NAT) changes the source address — used for internal hosts reaching the internet via a gateway. DNAT (destination NAT) changes the destination — used for port forwarding. Linux implements NAT via netfilter hooks in the nat table. Connection tracking (conntrack) stores the mapping so replies can be reverse-translated. iptables MASQUERADE is dynamic SNAT using the outgoing interface's IP.

**Q3: How does IP forwarding work in the kernel?**
A: When ip_forward is enabled and a routing lookup determines the packet is not local (RTN_UNICAST), ip_forward() is called. It: (1) checks TTL > 1 (drops with ICMP if not), (2) decrements TTL, (3) checks packet size vs output MTU (ICMP Frag Needed if DF and too large), (4) runs FORWARD netfilter chain, (5) calls ip_output() which runs POST_ROUTING chain and transmits via the output device.

---

## Summary

- Routing uses RPDB (policy rules) → FIB tables (LC-Trie longest prefix match)
- IP forwarding: TTL decrement, MTU check, netfilter FORWARD, output
- NAT: SNAT for outbound, DNAT for inbound, backed by connection tracking
- Policy routing enables source-based, mark-based, or interface-based routing
- ECMP distributes flows across multiple equal-cost paths

---

Next: [Chapter 20 — Netfilter and Packet Filtering](Chapter_20_Netfilter.md)
