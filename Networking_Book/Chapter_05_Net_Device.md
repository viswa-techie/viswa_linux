# Chapter 5: Network Devices — struct net_device

## Learning Goals
- Understand struct net_device in depth — every important field
- Know the network device registration and lifecycle
- Understand net_device_ops and their role
- Know virtual vs physical network devices
- Understand sysfs and /proc representations

---

## 5.1 struct net_device Overview

`struct net_device` is the kernel's representation of a network interface. It is one of the largest structures in the kernel (~2000 bytes).

```
struct net_device
┌──────────────────────────────────────────────────────────┐
│ Identity                                                  │
│   name[IFNAMSIZ]     → "eth0", "wlan0", "lo"             │
│   ifindex            → unique interface index             │
│   dev_addr[ETH_ALEN] → hardware MAC address               │
│                                                           │
│ Operations                                                │
│   netdev_ops         → ndo_start_xmit, ndo_open, etc.    │
│   ethtool_ops        → ethtool operations                 │
│   header_ops         → L2 header operations               │
│                                                           │
│ Configuration                                             │
│   mtu                → maximum transmission unit           │
│   type               → ARPHRD_ETHER, ARPHRD_LOOPBACK     │
│   flags              → IFF_UP, IFF_RUNNING, IFF_PROMISC   │
│   features           → NETIF_F_TSO, NETIF_F_GRO, etc.    │
│                                                           │
│ Queues                                                    │
│   _tx[num_tx_queues] → TX queue array                     │
│   _rx[num_rx_queues] → RX queue array                     │
│   num_tx_queues      → number of TX queues                │
│   real_num_tx_queues → active TX queues                   │
│                                                           │
│ Scheduling                                                │
│   qdisc              → default queueing discipline        │
│   tx_queue_len       → TX queue length (default 1000)     │
│                                                           │
│ Statistics                                                │
│   stats              → rx/tx packets, bytes, errors       │
│                                                           │
│ Network Namespace                                         │
│   nd_net             → network namespace this device is in│
│                                                           │
│ Private Data                                              │
│   priv               → driver-specific data               │
└──────────────────────────────────────────────────────────┘
```

---

## 5.2 net_device_ops — Driver Operations

```c
struct net_device_ops {
    /* Device lifecycle */
    int  (*ndo_open)(struct net_device *dev);
    int  (*ndo_stop)(struct net_device *dev);

    /* Packet transmission */
    netdev_tx_t (*ndo_start_xmit)(struct sk_buff *skb,
                                  struct net_device *dev);

    /* Multicast/promiscuous */
    void (*ndo_set_rx_mode)(struct net_device *dev);

    /* MAC address */
    int  (*ndo_set_mac_address)(struct net_device *dev, void *addr);

    /* MTU */
    int  (*ndo_change_mtu)(struct net_device *dev, int new_mtu);

    /* Statistics */
    void (*ndo_get_stats64)(struct net_device *dev,
                            struct rtnl_link_stats64 *stats);

    /* VLAN */
    int  (*ndo_vlan_rx_add_vid)(struct net_device *dev,
                                __be16 proto, u16 vid);

    /* ioctl */
    int  (*ndo_eth_ioctl)(struct net_device *dev,
                          struct ifreq *ifr, int cmd);

    /* TX timeout */
    void (*ndo_tx_timeout)(struct net_device *dev,
                           unsigned int txqueue);

    /* Queue selection */
    u16  (*ndo_select_queue)(struct net_device *dev,
                             struct sk_buff *skb,
                             struct net_device *sb_dev);

    /* XDP */
    int  (*ndo_bpf)(struct net_device *dev,
                    struct netdev_bpf *bpf);
    int  (*ndo_xdp_xmit)(struct net_device *dev, int n,
                          struct xdp_frame **frames,
                          u32 flags);
};
```

### Callback Flow

```
ip link set eth0 up    →  ndo_open()
                          - Enable hardware
                          - Allocate DMA rings
                          - Request IRQs
                          - Start NAPI
                          - netif_start_queue()

Packet TX              →  ndo_start_xmit(skb, dev)
                          - DMA map skb data
                          - Fill TX descriptor
                          - Ring doorbell

ip link set eth0 down  →  ndo_stop()
                          - netif_stop_queue()
                          - Disable NAPI
                          - Free IRQs
                          - Free DMA rings
                          - Disable hardware

ip link set eth0 mtu 9000 → ndo_change_mtu()
                          - Validate new MTU
                          - Resize buffers if needed
                          - Update dev->mtu
```

---

## 5.3 Network Device Allocation

```c
/* Allocate Ethernet device with private data */
struct net_device *netdev;
struct my_priv *priv;

netdev = alloc_etherdev_mqs(sizeof(struct my_priv),
                            num_tx_queues, num_rx_queues);
if (!netdev)
    return -ENOMEM;

/* Access private data */
priv = netdev_priv(netdev);

/* alloc_etherdev_mqs internally calls: */
/*   alloc_netdev_mqs(sizeof_priv, "eth%d", NET_NAME_UNKNOWN, */
/*                    ether_setup, txqs, rxqs)                  */

/* ether_setup() sets defaults: */
/*   dev->type = ARPHRD_ETHER                                  */
/*   dev->mtu = ETH_DATA_LEN (1500)                            */
/*   dev->addr_len = ETH_ALEN (6)                              */
/*   dev->tx_queue_len = DEFAULT_TX_QUEUE_LEN (1000)           */
/*   dev->flags = IFF_BROADCAST | IFF_MULTICAST                */
```

### Memory Layout

```
┌──────────────────────────────────────────────┐
│        struct net_device                     │
│  ┌──────────────────────────────────────┐    │
│  │  name, ifindex, dev_addr, ...        │    │
│  │  netdev_ops, ethtool_ops             │    │
│  │  mtu, flags, features                │    │
│  │  queues, stats, ...                  │    │
│  └──────────────────────────────────────┘    │
│  ┌──────────────────────────────────────┐    │
│  │  struct my_priv (private data)       │    │ ← netdev_priv(netdev)
│  │    hw_addr, ring pointers, ...       │    │
│  └──────────────────────────────────────┘    │
└──────────────────────────────────────────────┘
  Allocated as single contiguous block
```

---

## 5.4 Network Device Registration

```c
/* Complete registration flow */
static int my_nic_probe(struct pci_dev *pdev,
                        const struct pci_device_id *id)
{
    struct net_device *netdev;
    struct my_priv *priv;
    int err;

    /* 1. Allocate */
    netdev = alloc_etherdev_mqs(sizeof(*priv), TX_QUEUES, RX_QUEUES);

    /* 2. Set parent device */
    SET_NETDEV_DEV(netdev, &pdev->dev);

    /* 3. Configure operations */
    netdev->netdev_ops = &my_netdev_ops;
    netdev->ethtool_ops = &my_ethtool_ops;

    /* 4. Set features */
    netdev->features |= NETIF_F_HW_CSUM | NETIF_F_TSO | NETIF_F_GRO;
    netdev->hw_features |= NETIF_F_HW_CSUM | NETIF_F_TSO;

    /* 5. Set MAC address */
    eth_hw_addr_set(netdev, mac_from_hw);

    /* 6. Register (assigns name, creates sysfs entries) */
    err = register_netdev(netdev);
    if (err)
        goto err_free;

    /* Device now appears as eth0 (or whatever name) */
    /* /sys/class/net/eth0/ entries created */
    /* udev receives KOBJ_ADD event */

    return 0;

err_free:
    free_netdev(netdev);
    return err;
}
```

### Registration Internals

```
register_netdev(netdev)
    │
    ├── dev_get_valid_name()    → assign unique name (eth0, eth1, ...)
    │
    ├── dev_index_reserve()     → reserve ifindex
    │
    ├── netdev_register_kobject() → create sysfs entries
    │   └── /sys/class/net/<name>/
    │       ├── address          (MAC address)
    │       ├── mtu              (MTU value)
    │       ├── operstate        (up/down)
    │       ├── statistics/      (counters)
    │       └── queues/          (TX/RX queue info)
    │
    ├── call_netdevice_notifiers(NETDEV_REGISTER)
    │   └── Notify all registered listeners
    │
    └── list_netdevice()        → add to global net_device list
```

---

## 5.5 Network Device Lifecycle

```
                    ┌─────────────┐
                    │  UNREGISTERED│
                    └──────┬──────┘
                           │ register_netdev()
                    ┌──────▼──────┐
                    │  REGISTERED  │  ← Device exists but is DOWN
                    │  (IFF_UP=0)  │     ip link show: state DOWN
                    └──────┬──────┘
                           │ ip link set dev up
                           │ → ndo_open()
                    ┌──────▼──────┐
                    │    UP        │  ← Device is UP
                    │  (IFF_UP=1)  │     ip link show: state UP
                    │  (RUNNING?)  │
                    └──────┬──────┘
                           │ Link detected?
                    ┌──────▼──────┐
                    │   RUNNING    │  ← Link is active, can TX/RX
                    │(IFF_RUNNING) │     netif_carrier_on()
                    └──────┬──────┘
                           │ Cable unplugged
                    ┌──────▼──────┐
                    │  NO CARRIER  │  ← Link lost
                    │(carrier off) │     netif_carrier_off()
                    └──────┬──────┘
                           │ ip link set dev down
                           │ → ndo_stop()
                    ┌──────▼──────┐
                    │    DOWN      │
                    └──────┬──────┘
                           │ unregister_netdev()
                    ┌──────▼──────┐
                    │ UNREGISTERED │  → free_netdev()
                    └─────────────┘
```

### Carrier State Management

```c
/* In driver: link detection */
static void my_link_check(struct work_struct *work)
{
    u32 status = readl(priv->hw_addr + REG_LINK_STATUS);

    if (status & LINK_UP) {
        if (!netif_carrier_ok(netdev)) {
            netif_carrier_on(netdev);
            netdev_info(netdev, "Link is up %d Mbps %s\n",
                        speed, duplex ? "Full" : "Half");
        }
    } else {
        if (netif_carrier_ok(netdev)) {
            netif_carrier_off(netdev);
            netdev_info(netdev, "Link is down\n");
        }
    }
}
```

---

## 5.6 Network Device Features

```c
/* Feature flags control hardware offloads */
netdev->features       /* Currently active features */
netdev->hw_features    /* Features HW supports (user can toggle) */
netdev->vlan_features  /* Features available inside VLANs */

/* Important feature flags */
NETIF_F_HW_CSUM        /* Hardware computes TX checksum */
NETIF_F_RXCSUM         /* Hardware verifies RX checksum */
NETIF_F_TSO            /* TCP Segmentation Offload */
NETIF_F_TSO6           /* TSO for IPv6 */
NETIF_F_GSO            /* Generic Segmentation Offload */
NETIF_F_GRO            /* Generic Receive Offload */
NETIF_F_LRO            /* Large Receive Offload */
NETIF_F_SG             /* Scatter-Gather I/O */
NETIF_F_HIGHDMA        /* Can DMA from high memory */
NETIF_F_HW_VLAN_CTAG_TX  /* VLAN tag insertion in HW */
NETIF_F_HW_VLAN_CTAG_RX  /* VLAN tag stripping in HW */
NETIF_F_NTUPLE         /* N-tuple flow steering */
NETIF_F_RXHASH         /* Receive hashing offload */
```

```bash
# View features
ethtool -k eth0

# Toggle features
ethtool -K eth0 tso on gro on rx-checksumming on
```

---

## 5.7 Virtual Network Devices

```
Physical:                     Virtual:
┌──────────┐                 ┌──────────┐
│   eth0   │ ←real NIC       │   lo     │ ← loopback (127.0.0.1)
│ (driver) │                 │          │    no hardware
└──────────┘                 └──────────┘

┌──────────┐                 ┌──────────┐
│  wlan0   │ ←WiFi NIC       │  veth0   │ ← virtual ethernet pair
│ (driver) │                 │  veth1   │    (container networking)
└──────────┘                 └──────────┘

┌──────────┐                 ┌──────────┐
│   can0   │ ←CAN bus        │   br0    │ ← bridge device
│ (driver) │                 │          │    (L2 switch)
└──────────┘                 └──────────┘

                             ┌──────────┐
                             │  bond0   │ ← bonding/teaming
                             │          │    (link aggregation)
                             └──────────┘

                             ┌──────────┐
                             │  tun0    │ ← TUN/TAP
                             │  tap0    │    (VPN, user-space drivers)
                             └──────────┘
```

### Creating Virtual Devices

```bash
# Loopback (always present)
ip link show lo

# VETH pair (for containers)
ip link add veth0 type veth peer name veth1

# Bridge
ip link add br0 type bridge
ip link set eth0 master br0

# VLAN
ip link add link eth0 name eth0.100 type vlan id 100

# Bonding
ip link add bond0 type bond mode 802.3ad

# TUN (layer 3) / TAP (layer 2)
ip tuntap add tun0 mode tun
ip tuntap add tap0 mode tap

# MACVLAN
ip link add macvlan0 link eth0 type macvlan mode bridge

# VXLAN
ip link add vxlan0 type vxlan id 42 remote 10.0.0.2 dstport 4789
```

---

## 5.8 /proc and /sys Interface

```
/sys/class/net/<device>/
├── address           # MAC address
├── addr_len          # Address length
├── broadcast         # Broadcast address
├── carrier           # 1=link up, 0=link down
├── dev_id            # Device ID
├── dormant           # Dormant state
├── duplex            # Full/Half
├── flags             # Interface flags
├── ifindex           # Interface index
├── iflink            # Interface link index
├── mtu               # Current MTU
├── operstate         # Operational state (up/down/unknown)
├── speed             # Link speed in Mbps
├── tx_queue_len      # TX queue length
├── type              # Hardware type (1=Ethernet)
├── statistics/       # Packet and byte counters
│   ├── rx_packets
│   ├── tx_packets
│   ├── rx_bytes
│   ├── tx_bytes
│   ├── rx_errors
│   ├── tx_errors
│   ├── rx_dropped
│   └── tx_dropped
└── queues/           # Per-queue information
    ├── rx-0/
    └── tx-0/

/proc/net/dev          # All interface statistics (one line each)
/proc/net/if_inet6     # IPv6 addresses
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| include/linux/netdevice.h | struct net_device, struct net_device_ops |
| net/core/dev.c | Device registration, packet dispatch |
| net/core/dev_addr_lists.c | MAC address management |
| net/core/net-sysfs.c | /sys/class/net/ entries |
| include/linux/etherdevice.h | Ethernet helper functions |
| drivers/net/loopback.c | Loopback device |
| drivers/net/veth.c | Virtual Ethernet pair |
| drivers/net/bridge/ | Bridge device |
| drivers/net/bonding/ | Bonding driver |
| drivers/net/tun.c | TUN/TAP driver |

---

## Interview Questions

**Q1: What is struct net_device and what fields does it contain?**
A: struct net_device is the kernel's representation of a network interface. Key fields: name (interface name), ifindex (unique index), dev_addr (MAC), netdev_ops (driver callbacks), ethtool_ops, mtu, flags (IFF_UP, IFF_RUNNING), features (offload capabilities), tx/rx queues, statistics, qdisc (traffic control), and nd_net (network namespace). It's allocated with alloc_etherdev() which appends private data accessible via netdev_priv().

**Q2: What is the difference between IFF_UP and IFF_RUNNING?**
A: IFF_UP means the interface has been administratively enabled (ip link set up → ndo_open called). IFF_RUNNING means the physical link is detected (carrier present). A device can be UP but not RUNNING (cable unplugged). The driver calls netif_carrier_on()/off() to manage RUNNING state.

**Q3: How does the kernel assign interface names like eth0, eth1?**
A: When alloc_etherdev() is called with format "eth%d", the kernel assigns the next available number during register_netdev(). Modern systems use udev/systemd "Predictable Network Interface Names" (e.g., enp0s3 — en=ethernet, p0=PCI bus 0, s3=slot 3). This prevents name changes when hardware is added/removed.

**Q4: How does a driver stop and resume packet transmission?**
A: netif_stop_queue(dev) signals the kernel to stop calling ndo_start_xmit() — used when TX ring is full. netif_wake_queue(dev) resumes transmission — called from TX completion interrupt when ring space is freed. For multiqueue: netif_stop_subqueue(dev, queue_idx) and netif_wake_subqueue(dev, queue_idx).

**Q5: What are network device features and how does a user control them?**
A: Features are bit flags (NETIF_F_*) describing hardware capabilities: checksum offload, TSO, GRO, scatter-gather, VLAN offload. The driver sets hw_features (what hardware supports) and features (what's currently active). Users toggle features via `ethtool -K eth0 tso on`. The kernel validates changes and calls ndo_set_features() if the driver needs to reconfigure hardware.

---

## Summary

- struct net_device is the central abstraction for all network interfaces
- net_device_ops contains driver callbacks: open, stop, start_xmit, set_rx_mode
- Devices go through: unregistered → registered → up → running → down → unregistered
- Virtual devices (veth, bridge, bond, tun/tap, vlan) share the same net_device abstraction
- Features control hardware offloads; toggled via ethtool
- /sys/class/net/ and /proc/net/dev expose device state to user space

---

Next: [Chapter 6 — Network Device Drivers](Chapter_06_Network_Drivers.md)
