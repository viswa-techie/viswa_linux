# Chapter 6: Network Device Drivers

## Learning Goals
- Write a complete skeleton network driver
- Understand TX and RX paths in a driver
- Know how DMA descriptor rings are managed
- Understand NAPI integration in a driver
- Handle error paths and cleanup correctly

---

## 6.1 Network Driver Architecture

```
┌──────────────────────── Driver ─────────────────────────┐
│                                                          │
│  probe()                                                 │
│  ├── PCI setup (enable, BAR map, DMA mask, bus master)  │
│  ├── alloc_etherdev_mqs()                               │
│  ├── Set netdev_ops, ethtool_ops                        │
│  ├── Allocate descriptor rings + buffers                │
│  ├── Setup MSI-X interrupts                             │
│  ├── netif_napi_add()                                   │
│  └── register_netdev()                                  │
│                                                          │
│  ndo_open()                                              │
│  ├── Allocate RX buffers                                │
│  ├── Configure hardware (MAC, filters)                  │
│  ├── Request IRQs                                       │
│  ├── napi_enable()                                      │
│  ├── Enable RX/TX in hardware                           │
│  └── netif_start_queue()                                │
│                                                          │
│  ndo_start_xmit(skb, dev)                               │
│  ├── DMA map skb data                                   │
│  ├── Fill TX descriptor                                 │
│  ├── Advance tail pointer                               │
│  └── Check ring full → netif_stop_queue()               │
│                                                          │
│  ISR: my_irq_handler()                                   │
│  ├── Check interrupt cause                              │
│  ├── Disable NIC interrupts                             │
│  └── napi_schedule()                                    │
│                                                          │
│  NAPI poll: my_poll(napi, budget)                        │
│  ├── Process TX completions (clean TX ring)             │
│  ├── Process RX packets (clean RX ring)                 │
│  │   ├── Read RX descriptor                            │
│  │   ├── Build sk_buff                                 │
│  │   ├── skb->protocol = eth_type_trans()              │
│  │   └── napi_gro_receive(napi, skb)                   │
│  ├── If done < budget → napi_complete_done()            │
│  │   └── Re-enable NIC interrupts                      │
│  └── Return work_done                                   │
│                                                          │
│  ndo_stop()                                              │
│  ├── netif_stop_queue()                                 │
│  ├── napi_disable()                                     │
│  ├── Disable hardware RX/TX                            │
│  ├── Free IRQs                                         │
│  ├── Free DMA buffers                                  │
│  └── Clean descriptor rings                            │
│                                                          │
│  remove()                                                │
│  ├── unregister_netdev()                                │
│  ├── Free descriptor rings                             │
│  ├── iounmap(), pci_release_regions()                   │
│  ├── free_netdev()                                      │
│  └── pci_disable_device()                               │
└──────────────────────────────────────────────────────────┘
```

---

## 6.2 Complete Skeleton Driver

```c
#include <linux/module.h>
#include <linux/pci.h>
#include <linux/netdevice.h>
#include <linux/etherdevice.h>

#define RING_SIZE   256
#define RX_BUF_SIZE 2048

struct my_desc {
    __le64 addr;      /* DMA buffer address */
    __le32 length;    /* Packet length */
    __le32 status;    /* Status/command flags */
};

struct my_ring {
    struct my_desc *desc;       /* Descriptor array */
    dma_addr_t     dma;         /* Descriptor ring DMA addr */
    struct sk_buff **skbs;      /* sk_buff per descriptor */
    dma_addr_t     *skb_dma;   /* DMA addr per buffer */
    int            head;        /* Driver produces (TX) / NIC produces (RX) */
    int            tail;        /* NIC consumes (TX) / Driver fills (RX)   */
    int            count;       /* Ring size */
};

struct my_priv {
    void __iomem       *hw_addr;    /* MMIO base */
    struct pci_dev     *pdev;
    struct net_device  *netdev;
    struct napi_struct  napi;
    struct my_ring      tx_ring;
    struct my_ring      rx_ring;
};

/* ---------- TX Path ---------- */

static netdev_tx_t my_start_xmit(struct sk_buff *skb,
                                  struct net_device *dev)
{
    struct my_priv *priv = netdev_priv(dev);
    struct my_ring *ring = &priv->tx_ring;
    struct my_desc *desc;
    dma_addr_t dma;
    int idx;

    /* Check for space */
    if (ring->head - ring->tail >= ring->count) {
        netif_stop_queue(dev);
        return NETDEV_TX_BUSY;
    }

    idx = ring->head & (ring->count - 1);

    /* DMA map packet data */
    dma = dma_map_single(&priv->pdev->dev, skb->data,
                         skb->len, DMA_TO_DEVICE);
    if (dma_mapping_error(&priv->pdev->dev, dma)) {
        dev_kfree_skb_any(skb);
        dev->stats.tx_dropped++;
        return NETDEV_TX_OK;
    }

    /* Save for cleanup */
    ring->skbs[idx] = skb;
    ring->skb_dma[idx] = dma;

    /* Fill descriptor */
    desc = &ring->desc[idx];
    desc->addr = cpu_to_le64(dma);
    desc->length = cpu_to_le32(skb->len);
    desc->status = cpu_to_le32(DESC_CMD_EOP | DESC_CMD_RS);

    /* Memory barrier before advancing head */
    wmb();
    ring->head++;

    /* Ring doorbell (write tail to NIC register) */
    writel(ring->head & (ring->count - 1),
           priv->hw_addr + REG_TX_TAIL);

    return NETDEV_TX_OK;
}

/* ---------- TX Completion ---------- */

static void my_clean_tx(struct my_priv *priv)
{
    struct my_ring *ring = &priv->tx_ring;
    struct net_device *dev = priv->netdev;

    while (ring->tail != ring->head) {
        int idx = ring->tail & (ring->count - 1);
        struct my_desc *desc = &ring->desc[idx];

        /* Check if NIC is done with this descriptor */
        if (!(le32_to_cpu(desc->status) & DESC_STATUS_DONE))
            break;

        /* Unmap DMA */
        dma_unmap_single(&priv->pdev->dev,
                         ring->skb_dma[idx],
                         ring->skbs[idx]->len,
                         DMA_TO_DEVICE);

        /* Free sk_buff */
        dev_consume_skb_any(ring->skbs[idx]);
        ring->skbs[idx] = NULL;

        dev->stats.tx_packets++;
        dev->stats.tx_bytes += le32_to_cpu(desc->length);

        ring->tail++;
    }

    /* Wake queue if we freed space */
    if (netif_queue_stopped(dev) &&
        (ring->head - ring->tail < ring->count - 16))
        netif_wake_queue(dev);
}

/* ---------- RX Path (NAPI Poll) ---------- */

static int my_poll(struct napi_struct *napi, int budget)
{
    struct my_priv *priv = container_of(napi, struct my_priv, napi);
    struct my_ring *ring = &priv->rx_ring;
    struct net_device *dev = priv->netdev;
    int work_done = 0;

    /* Clean TX completions */
    my_clean_tx(priv);

    /* Process RX */
    while (work_done < budget) {
        int idx = ring->head & (ring->count - 1);
        struct my_desc *desc = &ring->desc[idx];
        struct sk_buff *skb;
        int length;

        /* Check if NIC wrote this descriptor */
        if (!(le32_to_cpu(desc->status) & DESC_STATUS_DONE))
            break;

        /* Read before accessing data */
        rmb();

        length = le32_to_cpu(desc->length);

        /* Sync DMA buffer for CPU */
        dma_sync_single_for_cpu(&priv->pdev->dev,
                                ring->skb_dma[idx],
                                length, DMA_FROM_DEVICE);

        /* Build sk_buff */
        skb = ring->skbs[idx];
        skb_put(skb, length);
        skb->protocol = eth_type_trans(skb, dev);

        /* Hardware checksum verification */
        if (le32_to_cpu(desc->status) & DESC_STATUS_CSUM_OK)
            skb->ip_summed = CHECKSUM_UNNECESSARY;

        /* Pass to network stack with GRO */
        napi_gro_receive(napi, skb);

        dev->stats.rx_packets++;
        dev->stats.rx_bytes += length;

        /* Allocate new buffer for this descriptor */
        skb = netdev_alloc_skb_ip_align(dev, RX_BUF_SIZE);
        ring->skbs[idx] = skb;
        ring->skb_dma[idx] = dma_map_single(&priv->pdev->dev,
                                             skb->data, RX_BUF_SIZE,
                                             DMA_FROM_DEVICE);
        desc->addr = cpu_to_le64(ring->skb_dma[idx]);
        desc->status = 0;

        ring->head++;
        work_done++;
    }

    if (work_done < budget) {
        napi_complete_done(napi, work_done);
        /* Re-enable interrupts */
        writel(IRQ_RX_ENABLE, priv->hw_addr + REG_IRQ_MASK);
    }

    return work_done;
}

/* ---------- Interrupt Handler ---------- */

static irqreturn_t my_irq_handler(int irq, void *data)
{
    struct my_priv *priv = data;
    u32 cause;

    cause = readl(priv->hw_addr + REG_IRQ_CAUSE);
    if (!cause)
        return IRQ_NONE;  /* Not our interrupt */

    /* Acknowledge interrupt */
    writel(cause, priv->hw_addr + REG_IRQ_CAUSE);

    if (cause & (IRQ_RX | IRQ_TX_DONE)) {
        /* Disable further interrupts, switch to polling */
        writel(0, priv->hw_addr + REG_IRQ_MASK);
        napi_schedule(&priv->napi);
    }

    if (cause & IRQ_LINK_CHANGE) {
        u32 status = readl(priv->hw_addr + REG_LINK_STATUS);
        if (status & LINK_UP)
            netif_carrier_on(priv->netdev);
        else
            netif_carrier_off(priv->netdev);
    }

    return IRQ_HANDLED;
}

/* ---------- Device Operations ---------- */

static int my_open(struct net_device *dev)
{
    struct my_priv *priv = netdev_priv(dev);
    int err;

    /* Allocate RX buffers and fill descriptors */
    err = my_alloc_rx_buffers(priv);
    if (err)
        return err;

    /* Request interrupt */
    err = request_irq(pci_irq_vector(priv->pdev, 0),
                      my_irq_handler, 0, dev->name, priv);
    if (err)
        goto err_free_rx;

    /* Enable NAPI */
    napi_enable(&priv->napi);

    /* Enable hardware RX/TX */
    writel(HW_RX_ENABLE | HW_TX_ENABLE,
           priv->hw_addr + REG_CTRL);

    /* Enable interrupts */
    writel(IRQ_RX_ENABLE | IRQ_TX_ENABLE | IRQ_LINK_ENABLE,
           priv->hw_addr + REG_IRQ_MASK);

    /* Allow kernel to queue packets */
    netif_start_queue(dev);

    return 0;

err_free_rx:
    my_free_rx_buffers(priv);
    return err;
}

static int my_stop(struct net_device *dev)
{
    struct my_priv *priv = netdev_priv(dev);

    /* Stop kernel from queuing packets */
    netif_stop_queue(dev);

    /* Disable interrupts */
    writel(0, priv->hw_addr + REG_IRQ_MASK);

    /* Disable NAPI */
    napi_disable(&priv->napi);

    /* Disable hardware */
    writel(0, priv->hw_addr + REG_CTRL);

    /* Free interrupt */
    free_irq(pci_irq_vector(priv->pdev, 0), priv);

    /* Clean up rings */
    my_clean_tx_ring(priv);
    my_free_rx_buffers(priv);

    return 0;
}

static const struct net_device_ops my_netdev_ops = {
    .ndo_open        = my_open,
    .ndo_stop        = my_stop,
    .ndo_start_xmit  = my_start_xmit,
    .ndo_set_rx_mode = my_set_rx_mode,
    .ndo_get_stats64 = my_get_stats64,
    .ndo_set_mac_address = eth_mac_addr,
    .ndo_validate_addr   = eth_validate_addr,
};
```

---

## 6.3 TX Flow Diagram

```
Application: send(fd, data, len)
        │
        ▼
Kernel: tcp_sendmsg() → ip_queue_xmit() → dev_queue_xmit()
        │
        ▼
tc:     qdisc_run() → dequeue skb
        │
        ▼
dev:    dev_hard_start_xmit()
        │
        ▼
Driver: ndo_start_xmit(skb, dev)
        │
        ├── dma_map_single(skb->data)
        ├── Fill TX descriptor [addr, len, cmd]
        ├── wmb()  (write barrier)
        ├── Advance ring->head
        ├── writel(tail, REG_TX_TAIL)  ← ring doorbell
        │
        ▼
NIC:    reads descriptor via DMA
        fetches packet data via DMA
        computes FCS, transmits
        sets DESC_STATUS_DONE
        generates TX completion interrupt
        │
        ▼
Driver: NAPI poll → my_clean_tx()
        dma_unmap_single()
        dev_consume_skb_any(skb)
        netif_wake_queue() if space freed
```

---

## 6.4 RX Flow Diagram

```
NIC:    receives frame
        DMA writes to pre-allocated buffer
        updates RX descriptor [addr, len, status]
        generates interrupt
        │
        ▼
ISR:    my_irq_handler()
        disable NIC interrupts
        napi_schedule()
        │
        ▼
Softirq: NET_RX_SOFTIRQ → net_rx_action()
        │
        ▼
NAPI:   my_poll(napi, budget)
        │
        ├── Read RX descriptor
        ├── dma_sync_single_for_cpu()
        ├── skb_put(skb, length)
        ├── eth_type_trans(skb, dev)  → set skb->protocol
        ├── Set skb->ip_summed if HW checksum OK
        ├── napi_gro_receive(napi, skb)
        ├── Allocate new buffer, refill descriptor
        │
        ├── if work_done < budget:
        │   └── napi_complete_done()
        │       └── Re-enable NIC interrupts
        │
        ▼
Stack:  netif_receive_skb() → ip_rcv() → tcp_v4_rcv()
        │
        ▼
Socket: Data queued to socket receive buffer
        │
        ▼
User:   recv(fd, buf, len) returns data
```

---

## 6.5 DMA Ring Management

```
TX Ring State Tracking:

   tail (cleaned)        head (next to fill)
     ↓                     ↓
  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ C │ C │ P │ P │ P │ F │   │   │
  └───┴───┴───┴───┴───┴───┴───┴───┘
    C = Completed (NIC done, driver cleaned)
    P = Pending   (NIC processing)
    F = Filled    (driver wrote, NIC not started)
    ' ' = Empty   (available for driver)

  Ring is FULL when: head - tail == count
  Ring is EMPTY when: head == tail

  Available: count - (head - tail)
  Stop queue when available < threshold (e.g., 16)
  Wake queue when available > threshold

RX Ring State Tracking:
   head (next to check)   tail (refilled)
     ↓                      ↓
  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ R │ R │ R │ E │ E │ E │ E │ E │
  └───┴───┴───┴───┴───┴───┴───┴───┘
    R = Ready (NIC wrote data, driver reads)
    E = Empty (buffer ready for NIC to fill)
```

---

## 6.6 Memory Barriers in Drivers

```c
/* Write memory barrier: ensure descriptor data is written
   before updating the tail pointer */
wmb();
writel(tail, hw + REG_TX_TAIL);

/* Read memory barrier: ensure descriptor status is read
   before accessing packet data */
if (desc->status & DONE) {
    rmb();
    memcpy(dest, buf, desc->length);
}

/* DMA barrier: dma_wmb() / dma_rmb() for descriptor access */
desc->addr = cpu_to_le64(dma);
desc->length = cpu_to_le32(len);
dma_wmb();  /* Ensure NIC sees addr+length before status */
desc->status = cpu_to_le32(CMD_GO);
```

---

## 6.7 Error Handling Patterns

```c
/* TX errors */
static netdev_tx_t my_start_xmit(struct sk_buff *skb,
                                  struct net_device *dev)
{
    /* DMA mapping failure */
    if (dma_mapping_error(&pdev->dev, dma)) {
        dev_kfree_skb_any(skb);
        dev->stats.tx_dropped++;
        return NETDEV_TX_OK;  /* NOT NETDEV_TX_BUSY */
    }

    /* Ring full */
    if (no_space) {
        netif_stop_queue(dev);
        /* Race: space may have freed between check and stop */
        smp_mb();
        if (has_space_now) {
            netif_wake_queue(dev);
        }
        return NETDEV_TX_BUSY;
    }
}

/* TX timeout — NIC stuck */
static void my_tx_timeout(struct net_device *dev,
                           unsigned int txqueue)
{
    struct my_priv *priv = netdev_priv(dev);
    netdev_err(dev, "TX timeout on queue %u\n", txqueue);

    /* Reset hardware */
    schedule_work(&priv->reset_work);
}

/* Watchdog: kernel calls ndo_tx_timeout if no TX
   completion for dev->watchdog_timeo jiffies */
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| drivers/net/ethernet/intel/e1000e/ | Intel e1000e driver (good reference) |
| drivers/net/ethernet/intel/igb/ | Intel igb driver (multiqueue) |
| drivers/net/ethernet/realtek/r8169.c | Realtek 8169 driver (simple) |
| drivers/net/ethernet/stmicro/stmmac/ | STMicro GMAC (embedded/SoC) |
| include/linux/netdevice.h | net_device_ops definition |
| net/core/dev.c | dev_queue_xmit, netif_receive_skb |

---

## Interview Questions

**Q1: Walk through what happens when ndo_start_xmit() is called.**
A: (1) Check TX ring has space; if not, netif_stop_queue() and return BUSY. (2) DMA-map the sk_buff data. (3) Fill TX descriptor with DMA address, length, and command flags. (4) wmb() to ensure descriptor writes are visible. (5) Advance ring head pointer. (6) Write tail register to ring the NIC doorbell. (7) Return NETDEV_TX_OK. Later, in TX completion (NAPI poll or interrupt), DMA-unmap, free sk_buff, update stats, and call netif_wake_queue() if space is freed.

**Q2: Why does the driver disable interrupts in the ISR and call napi_schedule()?**
A: This is the NAPI pattern. At high packet rates, per-packet interrupts consume 100% CPU. By disabling interrupts and switching to polling (napi_schedule), the driver processes multiple packets per poll invocation without interrupt overhead. When the queue is empty (work_done < budget), interrupts are re-enabled. This adaptive approach handles both low-rate (interrupt-driven) and high-rate (poll-driven) traffic.

**Q3: What memory barriers are needed in a network driver?**
A: wmb() before writing the tail pointer to ensure NIC sees complete descriptors. rmb() after reading the status to ensure CPU reads fresh packet data. dma_wmb()/dma_rmb() specifically for DMA descriptor ordering. These prevent CPU and compiler from reordering MMIO writes or reading stale data from DMA-written descriptors.

**Q4: How does a driver handle the TX ring becoming full?**
A: Call netif_stop_queue(dev) to prevent the kernel from calling ndo_start_xmit(). In TX completion processing, when enough descriptors are freed, call netif_wake_queue(dev). There's a race condition: check for space, then stop queue — space might free between check and stop. Use smp_mb() after stop and re-check. Return NETDEV_TX_BUSY only when truly stuck.

**Q5: What is the difference between dev_kfree_skb and dev_consume_skb?**
A: Both free sk_buffs but differ in semantic intent. dev_kfree_skb (or dev_kfree_skb_any) is for dropping/error paths. dev_consume_skb_any is for successful TX completion — semantically, the packet was consumed (transmitted). This distinction helps tracing tools (like ftrace/kfree_skb tracepoint) differentiate drops from normal frees.

---

## Summary

- Network drivers implement net_device_ops: open, stop, start_xmit
- TX path: DMA map → fill descriptor → ring doorbell → NIC transmits → completion cleanup
- RX path: NIC DMA writes → interrupt → NAPI poll → build sk_buff → pass to stack
- NAPI pattern: ISR disables interrupts, schedules poll; poll re-enables when done
- Memory barriers (wmb/rmb) are critical for correct descriptor ring operation
- TX ring flow control: stop_queue when full, wake_queue in completion

---

Next: [Chapter 7 — sk_buff and Buffer Management](Chapter_07_sk_buff.md)
