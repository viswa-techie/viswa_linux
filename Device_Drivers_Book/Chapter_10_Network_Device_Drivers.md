# Chapter 10: Network Device Drivers

## Chapter Overview

Network drivers are fundamentally different from char/block — they use the socket interface, process packets (sk_buffs), and require high-performance techniques like NAPI polling.

---

## 10.1 Network Driver Architecture

```
┌─────────────────────────────────────────────────┐
│ User Space:  socket() → send()/recv()           │
├─────────────────────────────────────────────────┤
│ Socket Layer:    AF_INET, AF_PACKET             │
├─────────────────────────────────────────────────┤
│ Protocol Layer:  TCP/UDP/IP/ICMP                │
├─────────────────────────────────────────────────┤
│ Network Core:    dev_queue_xmit() / netif_rx()  │
├─────────────────────────────────────────────────┤
│ Driver Layer:    net_device_ops → HW access      │
├─────────────────────────────────────────────────┤
│ Hardware:        NIC (DMA rings, PHY, MAC)       │
└─────────────────────────────────────────────────┘
```

---

## 10.2 Network Stack Interaction

### struct net_device — The Network Device

```c
struct net_device {
    char name[IFNAMSIZ];          /* "eth0", "wlan0" */
    unsigned int mtu;             /* Max transmission unit */
    unsigned char dev_addr[MAX_ADDR_LEN]; /* MAC address */
    const struct net_device_ops *netdev_ops;
    const struct ethtool_ops *ethtool_ops;
    unsigned int irq;
    struct netdev_queue *_tx;     /* TX queues */
    unsigned int num_tx_queues;
    struct napi_struct *napi;
    /* ... many more */
};
```

### struct net_device_ops

```c
static const struct net_device_ops my_netdev_ops = {
    .ndo_open       = my_open,        /* ifconfig up */
    .ndo_stop       = my_stop,        /* ifconfig down */
    .ndo_start_xmit = my_xmit,       /* Transmit packet */
    .ndo_get_stats64 = my_get_stats,  /* Statistics */
    .ndo_set_rx_mode = my_set_rx_mode, /* Multicast/promisc */
    .ndo_set_mac_address = my_set_mac,
    .ndo_change_mtu = my_change_mtu,
    .ndo_tx_timeout = my_tx_timeout,
};
```

---

## 10.3 Packet Processing Flow

### Transmit Path (TX)

```
send(sockfd, data, len)
       │
       ▼
TCP/IP builds sk_buff with headers
       │
       ▼
dev_queue_xmit(skb)
       │
       ▼
Traffic control (qdisc) → scheduling
       │
       ▼
ndo_start_xmit(skb, ndev)     ← YOUR DRIVER
       │
       ├── Copy skb data to TX DMA ring
       ├── Write TX doorbell register
       └── return NETDEV_TX_OK
       │
       ▼
TX completion interrupt → free skb
```

### Receive Path (RX) — NAPI

```
Hardware receives packet → DMA to RX ring → IRQ
       │
       ▼
IRQ handler: napi_schedule(&priv->napi)
       │
       ▼
NAPI poll callback (softirq context):
       │
       ├── Read packets from RX DMA ring
       ├── Allocate sk_buff per packet
       ├── skb->protocol = eth_type_trans(skb, ndev)
       ├── napi_gro_receive(&priv->napi, skb)
       └── If done: napi_complete_done() + re-enable IRQ
       │
       ▼
Network stack processes: IP → TCP → socket → user recv()
```

---

## 10.4-10.5 Minimal Network Driver Skeleton

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>
#include <linux/netdevice.h>
#include <linux/etherdevice.h>
#include <linux/platform_device.h>

struct my_net_priv {
    struct napi_struct napi;
    void __iomem *regs;
    struct net_device *ndev;
};

static int my_net_open(struct net_device *ndev)
{
    struct my_net_priv *priv = netdev_priv(ndev);
    napi_enable(&priv->napi);
    netif_start_queue(ndev);
    /* Enable HW RX/TX, enable interrupts */
    return 0;
}

static int my_net_stop(struct net_device *ndev)
{
    struct my_net_priv *priv = netdev_priv(ndev);
    netif_stop_queue(ndev);
    napi_disable(&priv->napi);
    /* Disable HW, disable interrupts */
    return 0;
}

static netdev_tx_t my_net_xmit(struct sk_buff *skb, struct net_device *ndev)
{
    struct my_net_priv *priv = netdev_priv(ndev);
    /* Place skb data into TX DMA descriptor ring */
    /* Ring doorbell register */
    ndev->stats.tx_packets++;
    ndev->stats.tx_bytes += skb->len;
    dev_kfree_skb_any(skb);
    return NETDEV_TX_OK;
}

static int my_net_poll(struct napi_struct *napi, int budget)
{
    struct my_net_priv *priv = container_of(napi, struct my_net_priv, napi);
    struct net_device *ndev = priv->ndev;
    int work_done = 0;

    while (work_done < budget) {
        /* Read RX descriptor from DMA ring */
        /* If no more packets: break */
        struct sk_buff *skb = netdev_alloc_skb(ndev, 1536);
        if (!skb) break;
        /* Copy/DMA data into skb->data */
        skb_put(skb, /* packet_len */);
        skb->protocol = eth_type_trans(skb, ndev);
        napi_gro_receive(napi, skb);
        ndev->stats.rx_packets++;
        work_done++;
    }

    if (work_done < budget) {
        napi_complete_done(napi, work_done);
        /* Re-enable RX interrupts */
    }
    return work_done;
}

static irqreturn_t my_net_irq(int irq, void *dev_id)
{
    struct my_net_priv *priv = dev_id;
    /* Disable RX IRQ to avoid interrupt storm */
    napi_schedule_irqoff(&priv->napi);
    return IRQ_HANDLED;
}

static const struct net_device_ops my_netdev_ops = {
    .ndo_open       = my_net_open,
    .ndo_stop       = my_net_stop,
    .ndo_start_xmit = my_net_xmit,
};

static int my_net_probe(struct platform_device *pdev)
{
    struct net_device *ndev;
    struct my_net_priv *priv;

    ndev = devm_alloc_etherdev(&pdev->dev, sizeof(*priv));
    if (!ndev) return -ENOMEM;

    priv = netdev_priv(ndev);
    priv->ndev = ndev;
    ndev->netdev_ops = &my_netdev_ops;

    eth_hw_addr_random(ndev);  /* Random MAC for demo */
    netif_napi_add(ndev, &priv->napi, my_net_poll);

    return register_netdev(ndev);
}

static void my_net_remove(struct platform_device *pdev)
{
    struct net_device *ndev = platform_get_drvdata(pdev);
    unregister_netdev(ndev);
}
```

---

## Interview Questions

**Q1: What is NAPI and why is it important?**
A: New API — switches between interrupt-driven and polling mode. On first packet: IRQ → schedule NAPI poll. During poll: process multiple packets without interrupts (reduce IRQ overhead). When done: re-enable IRQs. Critical for high-speed NICs (10GbE+).

**Q2: What is an sk_buff?**
A: The fundamental packet data structure. Contains packet data, network headers (L2/L3/L4), metadata (protocol, device, timestamp), and chain pointers. Allocated per-packet, passed through the entire network stack.

**Q3: How does the TX path handle backpressure?**
A: When TX ring is full, driver calls `netif_stop_queue()` — kernel stops sending more packets. When TX completions free ring space, driver calls `netif_wake_queue()` to resume.

---

*Next: [Chapter 11 — Bus Architecture in Linux](Chapter_11_Bus_Architecture.md)*
