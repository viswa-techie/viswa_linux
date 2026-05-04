# Chapter 12: Link Layer — Ethernet, ARP, VLAN

## Learning Goals
- Understand Ethernet frame structure and addressing
- Know ARP protocol operation and kernel implementation
- Understand VLAN tagging (802.1Q) and Linux VLAN support
- Know neighbor cache management and NDP

---

## 12.1 Ethernet Frame Structure

```
┌──────────────┬──────────────┬──────────┬───────────────────┬─────┐
│  Dst MAC     │  Src MAC     │EtherType │    Payload        │ FCS │
│  6 bytes     │  6 bytes     │ 2 bytes  │  46-1500 bytes    │4 b  │
└──────────────┴──────────────┴──────────┴───────────────────┴─────┘
│◄────────────── 64-1518 bytes total ──────────────────────►│

MAC Address: 48 bits (6 bytes)
  Format: aa:bb:cc:dd:ee:ff
  Bit 0 of first byte: 0=unicast, 1=multicast
  Bit 1 of first byte: 0=globally unique (OUI), 1=locally administered
  ff:ff:ff:ff:ff:ff = broadcast

EtherType values:
  0x0800 = IPv4
  0x0806 = ARP
  0x86DD = IPv6
  0x8100 = 802.1Q VLAN tag
  0x8847 = MPLS unicast
  0x88CC = LLDP

FCS: 32-bit CRC checked/stripped by hardware
```

### 802.1Q VLAN Tagged Frame

```
┌──────────┬──────────┬──────┬───────┬──────────┬──────────┬─────┐
│ Dst MAC  │ Src MAC  │0x8100│TCI    │EtherType │ Payload  │ FCS │
│ 6 bytes  │ 6 bytes  │2 b   │2 bytes│ 2 bytes  │46-1500 b │4 b  │
└──────────┴──────────┴──────┴───────┴──────────┴──────────┴─────┘
                       │ TPID │PCP│DEI│   VID    │
                              │3b │1b │  12 bits │
                              └────────┘
  TPID = Tag Protocol Identifier (0x8100)
  PCP  = Priority Code Point (0-7, for QoS)
  DEI  = Drop Eligible Indicator
  VID  = VLAN ID (0-4095, 12 bits)

  Max frame size: 1518 + 4 (tag) = 1522 bytes
```

---

## 12.2 Ethernet in Linux Kernel

```c
/* Ethernet header structure */
struct ethhdr {
    unsigned char h_dest[ETH_ALEN];    /* Destination MAC */
    unsigned char h_source[ETH_ALEN];  /* Source MAC */
    __be16        h_proto;              /* EtherType */
};

/* Adding Ethernet header (TX path) */
eth_header(skb, dev, type, daddr, saddr, len);

/* Parsing Ethernet header (RX path) */
__be16 proto = eth_type_trans(skb, dev);
/*  1. Sets skb->dev = dev
    2. Sets skb->protocol = EtherType
    3. Sets skb->pkt_type (HOST/BROADCAST/MULTICAST/OTHERHOST)
    4. Pulls ETH_HLEN bytes (skb_pull)
    5. Returns protocol for upper layer dispatch */
```

---

## 12.3 ARP (Address Resolution Protocol)

```
ARP resolves IPv4 addresses to MAC addresses.

Who has 192.168.1.1?  Tell 192.168.1.100
  (broadcast)

192.168.1.1 is at aa:bb:cc:dd:ee:ff
  (unicast reply)

ARP Packet Format:
┌───────────┬───────────┬────┬────┬─────┐
│ HW Type   │ Proto Type│HLEN│PLEN│Oper │
│ (2 bytes) │ (2 bytes) │1 b │1 b │2 b  │
├───────────┴───────────┴────┴────┴─────┤
│ Sender HW Addr (6 bytes)             │
│ Sender Proto Addr (4 bytes)          │
│ Target HW Addr (6 bytes)            │
│ Target Proto Addr (4 bytes)          │
└───────────────────────────────────────┘
  HW Type = 1 (Ethernet)
  Proto Type = 0x0800 (IPv4)
  Oper: 1=Request, 2=Reply
```

### ARP Flow in Kernel

```
TX Path (need to resolve destination MAC):

ip_finish_output()
  │
  └── neigh_output() → arp_solicit()
      │
      ├── Check neighbor cache (ARP table)
      │   Found? → Use cached MAC → send frame
      │   Not found? → Send ARP request, queue packet
      │
      ├── ARP request sent (broadcast)
      │   dst MAC = ff:ff:ff:ff:ff:ff
      │
      └── ARP reply received → arp_rcv() → arp_process()
          ├── Update neighbor cache
          └── Send queued packet with resolved MAC

Kernel ARP table:
  ip neigh show
  # 192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

### Neighbor Cache States

```
┌──────────────┐  timeout   ┌──────────────┐
│   REACHABLE  │──────────►│    STALE      │
│ (confirmed)  │           │ (not recently │
└──────────────┘           │  confirmed)   │
       ↑                   └──────┬───────┘
       │ ARP reply                │ traffic sent
       │ confirmation             ▼
┌──────────────┐           ┌──────────────┐
│   PROBE      │◄──────────│    DELAY     │
│ (ARP sent,   │  timeout  │ (waiting for │
│  waiting)    │           │  confirm)    │
└──────┬───────┘           └──────────────┘
       │ no reply (retries exhausted)
       ▼
┌──────────────┐
│   FAILED     │ → ICMP Host Unreachable
└──────────────┘

States: INCOMPLETE → REACHABLE → STALE → DELAY → PROBE → REACHABLE
                                                  └──→ FAILED
```

### Neighbor Management Commands

```bash
# View ARP/neighbor cache
ip neigh show
# 192.168.1.1 dev eth0 lladdr 00:11:22:33:44:55 REACHABLE

# Add static entry
ip neigh add 192.168.1.1 lladdr 00:11:22:33:44:55 dev eth0

# Delete entry
ip neigh del 192.168.1.1 dev eth0

# Flush cache
ip neigh flush dev eth0

# Tuning
sysctl net.ipv4.neigh.eth0.gc_stale_time      # When REACHABLE → STALE
sysctl net.ipv4.neigh.eth0.base_reachable_time # Base reachable time
```

---

## 12.4 VLAN Support in Linux

```
VLAN (Virtual LAN): logically partitions a physical LAN.

Physical setup:
  ┌──────────┐          ┌──────────┐
  │  Host A  │──────────│  Switch  │
  │ VLAN 10  │          │ port 1:  │
  └──────────┘          │  VLAN 10 │
  ┌──────────┐          │ port 2:  │
  │  Host B  │──────────│  VLAN 20 │
  │ VLAN 20  │          │ trunk:   │
  └──────────┘          │  all VLANs│
                        └──────────┘

Linux VLAN interface:
  eth0        → physical interface (trunk, carries all VLANs)
  eth0.10     → VLAN 10 virtual interface
  eth0.20     → VLAN 20 virtual interface

  TX: eth0.10 takes untagged frame, adds 802.1Q tag (VID=10), sends via eth0
  RX: eth0 receives tagged frame, strips tag, delivers to eth0.10
```

### VLAN Configuration

```bash
# Create VLAN interface
ip link add link eth0 name eth0.100 type vlan id 100

# Assign IP
ip addr add 192.168.100.1/24 dev eth0.100

# Bring up
ip link set eth0.100 up

# View VLAN info
cat /proc/net/vlan/eth0.100

# Delete
ip link del eth0.100

# Hardware VLAN offload (NIC adds/strips tags)
ethtool -k eth0 | grep vlan
#  rx-vlan-offload: on
#  tx-vlan-offload: on
```

### VLAN in Kernel

```
Kernel implementation: net/8021q/

RX with HW VLAN offload:
  NIC strips 802.1Q tag → stores VID in skb metadata
  __vlan_hwaccel_put_tag(skb, htons(ETH_P_8021Q), vid)
  netif_receive_skb() → vlan_do_receive()
    → Lookup VLAN device for VID
    → Pass skb to VLAN device's RX handler

RX without HW offload:
  Frame arrives with 802.1Q tag in data
  vlan_skb_recv() → parse tag → strip tag → deliver

TX with HW VLAN offload:
  VLAN device stores VID in skb: vlan_dev_hard_start_xmit()
  NIC inserts tag in hardware

TX without HW offload:
  VLAN device inserts 802.1Q tag into frame data
  Sends via parent device (eth0)
```

---

## 12.5 MAC Address Management

```c
/* Set MAC address */
ip link set eth0 address 00:11:22:33:44:55

/* Check address type */
is_valid_ether_addr(addr);      /* Not multicast, not zero */
is_zero_ether_addr(addr);       /* All zeros? */
is_multicast_ether_addr(addr);  /* Bit 0 of first byte set? */
is_broadcast_ether_addr(addr);  /* All ff? */

/* Generate random MAC */
eth_random_addr(addr);          /* Random unicast locally-administered */

/* Compare MACs */
ether_addr_equal(addr1, addr2);
```

### Promiscuous Mode

```
Normal mode:
  NIC accepts only: unicast to its MAC + broadcast + joined multicast

Promiscuous mode:
  NIC accepts ALL frames on the wire
  Used by: tcpdump, Wireshark, bridges, network monitoring

  ip link set eth0 promisc on
  
  In kernel: dev_set_promiscuity(dev, 1)
    → ndo_set_rx_mode(dev) → configures NIC filter
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| net/ethernet/eth.c | Ethernet helpers |
| net/ipv4/arp.c | ARP protocol |
| net/core/neighbour.c | Neighbor cache management |
| net/8021q/ | 802.1Q VLAN implementation |
| include/linux/if_ether.h | Ethernet constants, ethhdr |
| include/linux/if_vlan.h | VLAN structures |
| include/net/neighbour.h | Neighbor cache structures |
| drivers/net/macvlan.c | MACVLAN driver |

---

## Interview Questions

**Q1: How does ARP work and what kernel structures are involved?**
A: When IP needs to send a packet, it calls neigh_output() which checks the neighbor cache. If no entry exists, an ARP request (broadcast, EtherType 0x0806) is sent. The target replies with its MAC. arp_rcv() → arp_process() updates the neighbor cache (struct neighbour). States cycle: INCOMPLETE → REACHABLE → STALE → DELAY → PROBE. Entries expire after gc_stale_time. The neighbor subsystem is shared between ARP (IPv4) and NDP (IPv6).

**Q2: What is a VLAN and how does Linux implement it?**
A: VLAN (802.1Q) logically partitions a physical LAN using a 12-bit VLAN ID in a 4-byte tag inserted in the Ethernet frame. Linux creates virtual interfaces (eth0.100) that add/strip tags. With HW VLAN offload, the NIC handles tagging. Without it, the kernel's 8021q module manipulates frame data. VLAN lets one physical interface participate in multiple isolated networks.

**Q3: What is eth_type_trans() and what does it do?**
A: eth_type_trans(skb, dev) is called by network drivers on received packets. It: (1) Sets skb->dev to the receiving device. (2) Reads the EtherType to set skb->protocol (ETH_P_IP, ETH_P_ARP). (3) Checks dst MAC against dev->dev_addr to set skb->pkt_type (PACKET_HOST, PACKET_BROADCAST, PACKET_MULTICAST, PACKET_OTHERHOST). (4) Calls skb_pull(ETH_HLEN) to skip the Ethernet header. This prepares the skb for upper-layer processing.

**Q4: What is the neighbor subsystem and how is it shared between IPv4 and IPv6?**
A: The neighbor subsystem (net/core/neighbour.c) implements a generic next-hop resolution cache. IPv4 uses it as the ARP cache; IPv6 uses it for NDP (Neighbor Discovery Protocol). Both use struct neighbour with the same state machine (INCOMPLETE, REACHABLE, STALE, DELAY, PROBE, FAILED). Each protocol registers a neigh_ops structure with protocol-specific resolution functions (arp_solicit for IPv4, ndisc_solicit for IPv6).

---

## Summary

- Ethernet frames: 14-byte header (MACs + EtherType), 46-1500 byte payload, 4-byte FCS
- ARP resolves IP → MAC via broadcast request/unicast reply cycle
- Neighbor cache tracks resolution state: REACHABLE → STALE → PROBE
- VLAN (802.1Q) adds 4-byte tag with 12-bit VID; Linux creates virtual VLAN interfaces
- eth_type_trans() dispatches received frames to the correct protocol handler
- Hardware offloads (VLAN, checksum) reduce CPU overhead for L2 operations

---

Next: [Chapter 13 — Packet Transmission Path](Chapter_13_TX_Path.md)
