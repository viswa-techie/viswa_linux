# Chapter 18: Virtual Networking

## Learning Goals
- Understand TUN/TAP devices and their use cases
- Know bridge, bonding, and teaming
- Understand MACVLAN, IPVLAN, and VXLAN
- Know overlay networking concepts

---

## 18.1 TUN/TAP

```
TUN: Layer 3 (IP packets) — used by VPNs
TAP: Layer 2 (Ethernet frames) — used by VMs, bridging

  ┌─────────────┐
  │  User Space  │ ← Application reads/writes /dev/net/tun
  │  (OpenVPN,   │
  │   QEMU)      │
  └──────┬───────┘
         │ read()/write()
  ┌──────▼───────┐
  │  TUN/TAP     │ ← Virtual network device
  │  (tun0/tap0) │    Packets go to/from user-space app
  └──────┬───────┘
         │
  ┌──────▼───────┐
  │ Kernel Stack │ ← IP routing, netfilter, etc.
  └──────────────┘

TUN example (VPN):
  - OpenVPN opens /dev/net/tun → creates tun0
  - Kernel routes traffic for 10.8.0.0/24 via tun0
  - Packets for 10.8.0.x appear on tun0
  - OpenVPN reads from tun0, encrypts, sends via UDP to remote
  - Remote OpenVPN decrypts, writes to its tun0
  - Kernel delivers to destination

TAP example (VM):
  - QEMU opens /dev/net/tun in TAP mode → creates tap0
  - tap0 attached to bridge br0
  - VM sees a normal Ethernet NIC
  - VM's Ethernet frames appear on tap0 → bridged with host network
```

```bash
# Create TUN device
ip tuntap add tun0 mode tun user myuser
ip addr add 10.0.0.1/24 dev tun0
ip link set tun0 up

# Create TAP device
ip tuntap add tap0 mode tap user myuser
ip link set tap0 up
ip link set tap0 master br0    # Add to bridge
```

---

## 18.2 Bridge (L2 Switch)

```
Linux bridge: software L2 switch.

  ┌──────────────── br0 (bridge) ────────────────┐
  │   192.168.1.1/24                              │
  │                                                │
  │  Port 1     Port 2     Port 3     Port 4      │
  │  (eth0)     (tap0)     (veth0)    (eth1)      │
  └────┬──────────┬──────────┬──────────┬─────────┘
       │          │          │          │
  Physical    VM (QEMU)  Container   Physical

  Bridge learns MAC → port mapping (FDB table).
  Frames forwarded based on dst MAC:
    Known unicast → forward to specific port
    Unknown/broadcast → flood all ports
    Multicast → flood or IGMP snooping
```

```bash
# Create bridge
ip link add br0 type bridge
ip link set br0 up

# Add ports
ip link set eth0 master br0
ip link set tap0 master br0

# Assign IP to bridge (not ports)
ip addr add 192.168.1.1/24 dev br0

# View FDB (forwarding database)
bridge fdb show

# STP (Spanning Tree Protocol)
ip link set br0 type bridge stp_state 1
```

---

## 18.3 Bonding and Teaming

```
Bonding: combine multiple NICs for redundancy or throughput.

  ┌──────── bond0 ────────┐
  │   192.168.1.1/24      │
  │                        │
  │  Slave 1    Slave 2   │
  │  (eth0)     (eth1)    │
  └────┬──────────┬───────┘
       │          │
    Switch Port  Switch Port

Bonding modes:
  Mode 0 (balance-rr):   Round-robin TX across slaves
  Mode 1 (active-backup): Only one active, failover on failure
  Mode 2 (balance-xor):  XOR hash for TX slave selection
  Mode 4 (802.3ad/LACP): Link Aggregation with switch support
  Mode 5 (balance-tlb):  Adaptive TX load balancing
  Mode 6 (balance-alb):  Adaptive TX+RX load balancing
```

```bash
# Create bond
ip link add bond0 type bond mode 802.3ad
ip link set eth0 master bond0
ip link set eth1 master bond0
ip addr add 192.168.1.1/24 dev bond0
ip link set bond0 up

# View bond status
cat /proc/net/bonding/bond0
```

---

## 18.4 MACVLAN and IPVLAN

```
MACVLAN: Multiple virtual interfaces with different MACs on one physical.

  ┌─ macvlan0 (MAC: aa:bb:cc:dd:ee:01) ─┐
  │  10.0.0.1                             │
  └───────────────────────────────────────┘
  ┌─ macvlan1 (MAC: aa:bb:cc:dd:ee:02) ─┐
  │  10.0.0.2                             │
  └───────────────────────────────────────┘
              │
        ┌─────▼─────┐
        │   eth0     │  ← Physical NIC
        │ (parent)   │     NIC in promiscuous mode
        └────────────┘

Modes: bridge (communicate between macvlans),
       vepa (external switch handles),
       private (isolated)

IPVLAN: Like MACVLAN but shares parent MAC.
  All virtual interfaces use same MAC address.
  Differentiation by IP address (Layer 3).
  Works where MAC-limiting switches block MACVLAN.
```

```bash
# MACVLAN
ip link add macvlan0 link eth0 type macvlan mode bridge
ip addr add 10.0.0.1/24 dev macvlan0
ip link set macvlan0 up

# IPVLAN
ip link add ipvlan0 link eth0 type ipvlan mode l3
ip addr add 10.0.0.1/24 dev ipvlan0
ip link set ipvlan0 up
```

---

## 18.5 VXLAN (Virtual Extensible LAN)

```
VXLAN: L2 overlay over L3 underlay. Encapsulates Ethernet frames in UDP.

  Host A (10.0.0.1)                    Host B (10.0.0.2)
  ┌─────────────┐                      ┌─────────────┐
  │ Container X │                      │ Container Y │
  │ 192.168.1.1 │                      │ 192.168.1.2 │
  └──────┬──────┘                      └──────┬──────┘
         │ veth                                │ veth
  ┌──────▼──────┐                      ┌──────▼──────┐
  │   br0       │                      │   br0       │
  │  VXLAN VNI  │                      │  VXLAN VNI  │
  │   42        │                      │   42        │
  └──────┬──────┘                      └──────┬──────┘
         │ vxlan0                              │ vxlan0
  ┌──────▼──────┐                      ┌──────▼──────┐
  │   eth0      │─── IP Network ──────│   eth0      │
  │ 10.0.0.1    │    (underlay)        │ 10.0.0.2   │
  └─────────────┘                      └─────────────┘

  Container X sends to 192.168.1.2:
    1. Frame: [ETH][IP 192.168.1.1→.2][payload]
    2. VXLAN encap: [ETH][IP 10.0.0.1→.2][UDP:4789][VXLAN VNI=42][original frame]
    3. Travels over physical network to Host B
    4. Host B decapsulates, delivers to Container Y

  24-bit VNI → 16 million virtual networks (vs VLAN's 4096)
```

```bash
# Create VXLAN
ip link add vxlan0 type vxlan id 42 \
   remote 10.0.0.2 local 10.0.0.1 \
   dstport 4789 dev eth0
ip link set vxlan0 up
ip link set vxlan0 master br0
```

---

## 18.6 Summary Table

```
┌────────────┬───────┬──────────────────────────────────────────┐
│ Type       │ Layer │ Use Case                                 │
├────────────┼───────┼──────────────────────────────────────────┤
│ veth       │  L2   │ Container ↔ host connectivity            │
│ bridge     │  L2   │ L2 switch (connect VMs, containers)      │
│ bond       │  L2   │ NIC redundancy / aggregation             │
│ VLAN       │  L2   │ Network segmentation on one link         │
│ TUN        │  L3   │ VPN, user-space routing                  │
│ TAP        │  L2   │ VM networking, user-space bridge         │
│ MACVLAN    │  L2   │ Multiple MACs on one NIC                 │
│ IPVLAN     │  L3   │ Multiple IPs sharing one MAC             │
│ VXLAN      │  L2/3 │ Overlay networking, multi-host L2        │
│ GRE        │  L3   │ Simple tunnel encapsulation              │
│ WireGuard  │  L3   │ Modern encrypted VPN                     │
└────────────┴───────┴──────────────────────────────────────────┘
```

---

## Interview Questions

**Q1: What is the difference between TUN and TAP?**
A: TUN operates at Layer 3 — user-space application reads/writes IP packets. TAP operates at Layer 2 — user-space application reads/writes Ethernet frames. TUN is used for VPNs (routing IP traffic through a tunnel). TAP is used for VMs (QEMU creates a virtual NIC that the VM sees as real hardware) and can be bridged.

**Q2: How does VXLAN enable multi-host container networking?**
A: VXLAN encapsulates Layer 2 (Ethernet) frames inside UDP packets. Containers on different physical hosts can share the same L2 domain — they appear to be on the same LAN even though they're separated by an IP network. The 24-bit VNI supports 16M virtual networks vs VLAN's 4096 limit. The physical (underlay) network just sees normal IP/UDP traffic.

**Q3: When would you use bonding mode 4 (802.3ad/LACP)?**
A: When you need both redundancy and increased bandwidth with switch support. LACP negotiates link aggregation with the switch — both sides must agree on bonding. The switch distributes incoming traffic across both links. TX uses hash-based distribution. If one link fails, traffic automatically moves to the remaining link. Requires switch configuration (LACP-capable ports).

---

## Summary

- TUN/TAP connect user-space to kernel network stack (VPN, VM)
- Bridges provide L2 switching between physical/virtual interfaces
- Bonding aggregates NICs for redundancy and bandwidth
- MACVLAN/IPVLAN create virtual interfaces on physical NICs
- VXLAN provides L2 overlay over L3, enabling multi-host L2 networks

---

Next: [Chapter 19 — Routing and Forwarding](Chapter_19_Routing_Forwarding.md)
