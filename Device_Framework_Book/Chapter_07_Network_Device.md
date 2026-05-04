# Chapter 7: Network Device Framework

## Learning Goals
- Understand Linux network device (net_device) architecture
- Know NAPI polling, sk_buff, and packet transmit/receive paths
- Grasp ethtool, netlink, and network namespace concepts
- Write or analyze a network device driver

---

## 7.1 Network Subsystem Architecture

```
Network Stack Architecture:

User Space:
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Browser  │  │ curl     │  │  iperf3  │
  └─────┬────┘  └─────┬────┘  └─────┬────┘
        │  socket()     │              │
        └──────────────┼──────────────┘
                       │
Kernel:                │
  ┌────────────────────▼─────────────────────────────┐
  │              Socket Layer                         │
  │  AF_INET / AF_INET6 / AF_PACKET / AF_CAN         │
  ├──────────────────────────────────────────────────┤
  │              Transport Layer                      │
  │  TCP (stream) / UDP (datagram) / SCTP / RAW       │
  ├──────────────────────────────────────────────────┤
  │              Network Layer                        │
  │  IPv4 / IPv6 / Routing / Netfilter (iptables)     │
  ├──────────────────────────────────────────────────┤
  │              Network Device Layer                 │
  │  ┌──────────────────────────────────────────┐    │
  │  │  net_device (struct net_device)           │    │
  │  │  ├── net_device_ops (ndo_start_xmit, ...) │    │
  │  │  ├── NAPI (softirq-based RX polling)      │    │
  │  │  ├── ethtool_ops (stats, link settings)   │    │
  │  │  └── Traffic control (TC qdisc)           │    │
  │  └──────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────┘
                       │
Hardware:              ▼
  ┌──────────────────────────────────┐
  │ NIC: Ethernet MAC + PHY          │
  │ DMA rings: TX ring + RX ring     │
  │ Interrupt → NAPI poll            │
  └──────────────────────────────────┘
```

---

## 7.2 struct net_device and net_device_ops

```c
/* Network device driver — key structures */

static const struct net_device_ops my_netdev_ops = {
    .ndo_open        = my_open,        /* ifconfig up */
    .ndo_stop        = my_stop,        /* ifconfig down */
    .ndo_start_xmit  = my_start_xmit,  /* transmit packet */
    .ndo_set_mac_address = eth_mac_addr,
    .ndo_get_stats64 = my_get_stats64, /* TX/RX counters */
    .ndo_set_rx_mode = my_set_rx_mode, /* multicast/promisc */
    .ndo_do_ioctl    = my_do_ioctl,
    .ndo_tx_timeout  = my_tx_timeout,
};

static int my_probe(struct platform_device *pdev)
{
    struct net_device *ndev;

    /* Allocate net_device with private data */
    ndev = alloc_etherdev(sizeof(struct my_priv));
    SET_NETDEV_DEV(ndev, &pdev->dev);

    ndev->netdev_ops = &my_netdev_ops;
    ndev->ethtool_ops = &my_ethtool_ops;
    ndev->watchdog_timeo = 5 * HZ;

    /* Set MAC address (from DT or OTP) */
    eth_hw_addr_set(ndev, mac_addr);

    /* Register — creates network interface (eth0) */
    ret = register_netdev(ndev);

    return ret;
}
```

---

## 7.3 Packet Transmit Path

```
TX Path: Application → Wire

Application:
  send(sockfd, data, len, 0)
        │
        ▼
Socket Layer:
  sock_sendmsg()
        │
        ▼
Transport (TCP):
  tcp_sendmsg() → tcp_write_xmit()
  Segment data, add TCP header
        │
        ▼
Network (IP):
  ip_queue_xmit()
  Add IP header, route lookup
  Netfilter (OUTPUT chain)
        │
        ▼
Device Layer:
  dev_queue_xmit()
  TC qdisc (queuing discipline)
        │
        ▼
Driver:
  ndo_start_xmit(skb, ndev)
```

```c
/* Driver's transmit function */
static netdev_tx_t my_start_xmit(struct sk_buff *skb,
                                  struct net_device *ndev)
{
    struct my_priv *priv = netdev_priv(ndev);
    dma_addr_t dma_addr;

    /* Map skb data for DMA */
    dma_addr = dma_map_single(&ndev->dev, skb->data,
                              skb->len, DMA_TO_DEVICE);

    /* Fill TX descriptor ring */
    priv->tx_ring[priv->tx_head].addr = dma_addr;
    priv->tx_ring[priv->tx_head].len  = skb->len;
    priv->tx_ring[priv->tx_head].flags = TX_DESC_OWN;
    priv->tx_head = (priv->tx_head + 1) % TX_RING_SIZE;

    /* Tell hardware there's a new packet */
    writel(TX_START, priv->regs + TX_CTRL);

    /* If ring is full, stop the queue */
    if (tx_ring_full(priv))
        netif_stop_queue(ndev);

    return NETDEV_TX_OK;
}
```

---

## 7.4 Packet Receive Path and NAPI

```
RX Path: Wire → Application

Hardware:
  NIC DMA writes packet to RX ring buffer
  Raises interrupt
        │
        ▼
Driver ISR:
  napi_schedule() — schedule NAPI poll
  Disable HW interrupt (avoid interrupt storm)
        │
        ▼
NAPI Poll (softirq context):
  napi_poll() called by NET_RX_SOFTIRQ
  Process up to 'budget' packets
  For each packet:
    ├── Allocate sk_buff
    ├── Copy/map DMA data
    ├── Set skb->protocol (eth_type_trans)
    └── napi_gro_receive() or netif_receive_skb()
  If budget exhausted: return budget (stay scheduled)
  If all done: napi_complete_done(), re-enable IRQ
        │
        ▼
Network Stack:
  Netfilter → IP → TCP/UDP → Socket → Application
  Application: recv(sockfd, buf, len, 0)
```

```c
/* NAPI-based receive */

/* ISR — minimal work, schedule NAPI */
static irqreturn_t my_isr(int irq, void *data)
{
    struct my_priv *priv = data;

    /* Disable receive interrupt */
    writel(0, priv->regs + RX_IRQ_EN);

    /* Schedule NAPI poll */
    napi_schedule(&priv->napi);

    return IRQ_HANDLED;
}

/* NAPI poll — process received packets */
static int my_napi_poll(struct napi_struct *napi, int budget)
{
    struct my_priv *priv = container_of(napi, struct my_priv, napi);
    int processed = 0;

    while (processed < budget) {
        struct sk_buff *skb;
        struct rx_desc *desc = &priv->rx_ring[priv->rx_tail];

        if (!(desc->flags & RX_DESC_DONE))
            break;

        /* Allocate skb and copy data */
        skb = netdev_alloc_skb(priv->ndev, desc->len + NET_IP_ALIGN);
        skb_reserve(skb, NET_IP_ALIGN);
        memcpy(skb_put(skb, desc->len), desc->buf, desc->len);

        skb->protocol = eth_type_trans(skb, priv->ndev);
        napi_gro_receive(napi, skb);

        priv->rx_tail = (priv->rx_tail + 1) % RX_RING_SIZE;
        processed++;
    }

    if (processed < budget) {
        napi_complete_done(napi, processed);
        /* Re-enable receive interrupt */
        writel(1, priv->regs + RX_IRQ_EN);
    }

    return processed;
}

/* Setup NAPI in probe */
netif_napi_add(ndev, &priv->napi, my_napi_poll);
```

---

## 7.5 sk_buff — The Network Buffer

```
sk_buff (Socket Buffer):

┌──────────────────────────────────────────────┐
│  struct sk_buff                               │
│  ├── head      → start of allocated buffer    │
│  ├── data      → start of current data        │
│  ├── tail      → end of current data          │
│  ├── end       → end of allocated buffer      │
│  ├── len       → data length (tail - data)    │
│  ├── protocol  → ETH_P_IP, ETH_P_ARP, ...    │
│  ├── dev       → associated net_device        │
│  ├── cb[48]    → protocol-private scratch     │
│  └── destructor→ called when skb freed        │
│                                               │
│  Buffer layout (after receiving):             │
│  head ──► ┌────────────────────┐              │
│           │   headroom         │              │
│  data ──► ├────────────────────┤              │
│           │ Ethernet header    │ 14 bytes     │
│           ├────────────────────┤              │
│           │ IP header          │ 20 bytes     │
│           ├────────────────────┤              │
│           │ TCP header         │ 20 bytes     │
│           ├────────────────────┤              │
│           │ Payload            │ variable     │
│  tail ──► ├────────────────────┤              │
│           │   tailroom         │              │
│  end  ──► └────────────────────┘              │
└──────────────────────────────────────────────┘

Key operations:
  skb_put(skb, len)   — extend data area (tail += len)
  skb_push(skb, len)  — prepend header (data -= len)
  skb_pull(skb, len)  — strip header (data += len)
  skb_reserve(skb, len) — add headroom before data
```

---

## 7.6 Automotive Networking: CAN Bus

```
Linux CAN (SocketCAN):

AF_CAN socket:
  ┌─────────────────────────────────┐
  │ socket(AF_CAN, SOCK_RAW,       │
  │        CAN_RAW)                 │
  │ bind(sock, &ifr, sizeof(ifr))  │
  │ write(sock, &frame, sizeof())  │
  │ read(sock, &frame, sizeof())   │
  └─────────────┬───────────────────┘
                │
  ┌─────────────▼───────────────────┐
  │ CAN Core (net/can/)              │
  │ ├── can_raw (raw protocol)       │
  │ ├── can_bcm (broadcast manager)  │
  │ └── isotp  (ISO 15765-2)        │
  └─────────────┬───────────────────┘
                │
  ┌─────────────▼───────────────────┐
  │ CAN Driver (net_device)          │
  │ ├── drivers/net/can/             │
  │ ├── struct can_priv              │
  │ ├── Bittiming configuration      │
  │ └── Error handling (bus-off)     │
  └─────────────────────────────────┘
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| Net core | net/core/dev.c | net_device management |
| sk_buff | net/core/skbuff.c | Buffer operations |
| Ethernet | net/ethernet/eth.c | Ethernet helpers |
| NAPI | net/core/dev.c (napi_*) | Polling framework |
| netdevice.h | include/linux/netdevice.h | net_device/ndo defs |
| CAN | net/can/ | CAN protocol family |
| Ethtool | net/ethtool/ | Ethtool interface |

---

## Interview Questions

**Q1: What is NAPI and why is it used?**
A: NAPI (New API) is a polling mechanism for network packet reception. Without NAPI, every packet generates a hardware interrupt — at high rates (millions of packets/sec), interrupt overhead dominates CPU time. NAPI works by: (1) First packet triggers hardware interrupt. (2) ISR disables further interrupts and schedules a poll function via softirq. (3) The poll function processes up to `budget` packets per invocation without interrupts. (4) When all packets are processed, the poll function re-enables interrupts. This converts from interrupt-driven to poll-driven under load, dramatically improving throughput.

**Q2: Explain the sk_buff structure and its pointer operations.**
A: sk_buff is the network buffer representing one packet. It has four pointers: `head` (buffer start), `data` (current data start), `tail` (data end), `end` (buffer end). `skb_put(len)` extends the data area backward (tail moves forward) — used when adding data. `skb_push(len)` prepends to data (data moves backward) — used when adding a protocol header. `skb_pull(len)` removes from the front (data moves forward) — used when stripping a header. `skb_reserve(len)` creates headroom before data is placed — commonly done with NET_IP_ALIGN for alignment.

**Q3: Walk through the lifecycle of a received network packet.**
A: (1) NIC DMA writes packet to a pre-allocated RX descriptor ring buffer. (2) NIC raises interrupt. (3) ISR calls `napi_schedule()` and disables interrupt. (4) NAPI softirq calls poll function. (5) Driver reads RX descriptor, allocates sk_buff, copies/maps data. (6) `eth_type_trans()` sets `skb->protocol`. (7) `napi_gro_receive()` passes to network stack with GRO merging. (8) IP layer processes routing and netfilter. (9) TCP/UDP delivers to socket. (10) `recv()` returns data to application.

---

## Summary

- `net_device` is the core structure for network interfaces with `net_device_ops`
- TX: application → socket → TCP/IP → `ndo_start_xmit()` → DMA → wire
- RX: wire → DMA → interrupt → NAPI poll → network stack → socket → application
- NAPI converts interrupt-driven to poll-driven reception for high throughput
- sk_buff represents packets with head/data/tail/end pointer management
- CAN bus uses AF_CAN sockets (SocketCAN) for automotive networking

---

*Next: [Chapter 8 — USB Framework](Chapter_08_USB_Framework.md)*
