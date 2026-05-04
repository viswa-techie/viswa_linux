# Chapter 3: Network Hardware Architecture

## Learning Goals
- Understand NIC hardware internals: MAC, PHY, DMA engine
- Know how descriptor rings work for TX/RX
- Understand interrupt coalescing and its importance
- Know Ethernet physical layer signaling basics
- Understand MMIO, PCI BAR, and hardware register access

---

## 3.1 NIC Architecture Overview

```
┌──────────────────────── NIC Card ──────────────────────────┐
│                                                             │
│  ┌─────────┐    ┌─────────┐    ┌───────────┐    ┌──────┐  │
│  │   PHY   │◄──►│   MAC   │◄──►│ DMA Engine│◄──►│ PCIe │  │
│  │(Layer 1)│    │(Layer 2)│    │           │    │Bridge│  │
│  └────┬────┘    └─────────┘    └───────────┘    └──┬───┘  │
│       │                                             │      │
│  ┌────┴────┐                                   ┌────┴───┐  │
│  │  RJ-45  │                                   │PCIe Bus│  │
│  │Connector│                                   │to Host │  │
│  └─────────┘                                   └────────┘  │
│                                                             │
│  ┌─────────────────────────────┐                           │
│  │   Interrupt Controller      │                           │
│  │   (MSI-X, Legacy INTx)      │                           │
│  └─────────────────────────────┘                           │
│                                                             │
│  ┌─────────────────────────────┐                           │
│  │   TX/RX Descriptor Rings    │                           │
│  │   (in NIC SRAM or host RAM) │                           │
│  └─────────────────────────────┘                           │
└─────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Function |
|-----------|----------|
| PHY | Physical layer transceiver — converts digital to analog signals, auto-negotiation, link detection |
| MAC | Media Access Control — frame assembly/disassembly, CRC generation/check, flow control, address filtering |
| DMA Engine | Transfers packet data between NIC and host memory without CPU involvement |
| PCIe Bridge | Interfaces NIC to system bus; provides BAR for MMIO register access |
| Interrupt Controller | Generates interrupts to CPU on events (packet arrival, TX completion, errors) |
| Descriptor Rings | Circular buffers describing packet locations for DMA |

---

## 3.2 PHY Layer

```
┌────────────────── PHY Chip ──────────────────┐
│                                               │
│  ┌──────────┐    ┌──────────┐    ┌────────┐ │
│  │  PCS     │◄──►│  PMA     │◄──►│  PMD   │ │
│  │(Phys.    │    │(Phys.    │    │(Phys.  │ │
│  │ Coding   │    │ Medium   │    │ Medium │ │
│  │ Sublayer)│    │ Attach.) │    │ Depend)│ │
│  └──────────┘    └──────────┘    └────────┘ │
│       ↑                              ↓       │
│    from MAC                     to cable     │
└──────────────────────────────────────────────┘

PCS: 8b/10b or 64b/66b encoding
PMA: Serialization/deserialization (SerDes)
PMD: Electrical interface to medium (copper/fiber)
```

### Ethernet Speed Standards

| Standard | Speed | Encoding | Medium |
|----------|-------|----------|--------|
| 10BASE-T | 10 Mbps | Manchester | Cat 3 copper |
| 100BASE-TX | 100 Mbps | 4B/5B + MLT-3 | Cat 5 copper |
| 1000BASE-T | 1 Gbps | PAM-5, 4 pairs | Cat 5e copper |
| 10GBASE-T | 10 Gbps | PAM-16 | Cat 6a copper |
| 10GBASE-SR | 10 Gbps | 64b/66b | Multimode fiber |
| 25GBASE-CR | 25 Gbps | 64b/66b | Direct attach copper |
| 100GBASE-SR4 | 100 Gbps | 64b/66b, 4 lanes | Multimode fiber |

### PHY Management in Linux

```
PHY drivers in Linux: drivers/net/phy/

MDIO bus connects MAC to PHY:
  ┌─────┐  MDIO  ┌─────┐
  │ MAC │◄──────►│ PHY │
  └─────┘  MDC   └─────┘

  MDIO = Management Data I/O (bidirectional data)
  MDC  = Management Data Clock

  Standard register set (IEEE 802.3):
    Register 0:  Control (reset, speed, autoneg)
    Register 1:  Status (link up, autoneg complete)
    Register 4:  Autoneg Advertisement
    Register 5:  Link Partner Ability
```

---

## 3.3 MAC Layer Hardware

The MAC handles raw Ethernet frame operations:

```
TX Path (MAC):
  1. Receive frame data from DMA engine
  2. Prepend preamble (7 bytes) + SFD (1 byte)
  3. Compute and append CRC32 (FCS, 4 bytes)
  4. Enforce minimum frame size (64 bytes including FCS)
  5. Apply inter-frame gap (96 bit-times)
  6. Transmit via PHY

RX Path (MAC):
  1. Detect preamble, strip it
  2. Check destination MAC (unicast, broadcast, multicast)
  3. Verify CRC32
  4. If CRC good: pass frame to DMA engine
  5. If CRC bad: increment error counter, drop frame
```

### Ethernet Frame on the Wire

```
┌──────────┬─────┬────────────┬────────────┬──────┬──────────────┬─────┐
│ Preamble │ SFD │   Dst MAC  │  Src MAC   │ Type │   Payload    │ FCS │
│ 7 bytes  │1 b  │  6 bytes   │  6 bytes   │2 b   │ 46-1500 b    │4 b  │
└──────────┴─────┴────────────┴────────────┴──────┴──────────────┴─────┘
                  ◄──────── 64-1518 bytes (on wire) ────────────►
                  ◄──── This is what the kernel sees ──────────►

Note: Preamble and SFD are added/stripped by hardware.
      Kernel never sees them.
```

---

## 3.4 DMA Engine and Descriptor Rings

DMA (Direct Memory Access) allows the NIC to read/write host memory without CPU involvement.

```
                    HOST MEMORY                           NIC
              ┌─────────────────────┐
              │   TX Descriptor     │
              │   Ring (in RAM)     │──── DMA read ────►  NIC reads
              │  ┌───┬───┬───┬───┐ │                      descriptors
              │  │ 0 │ 1 │ 2 │...│ │                      then fetches
              │  └───┴───┴───┴───┘ │                      packet data
              │         ↓          │
              │   TX Packet        │
              │   Buffers (in RAM) │──── DMA read ────►  NIC reads
              │  [pkt0][pkt1]...   │                      packet data
              ├─────────────────────┤                      and transmits
              │   RX Descriptor    │
              │   Ring (in RAM)    │◄─── DMA write ────  NIC writes
              │  ┌───┬───┬───┬───┐ │                      descriptors
              │  │ 0 │ 1 │ 2 │...│ │                      after DMA
              │  └───┴───┴───┴───┘ │
              │         ↓          │
              │   RX Packet        │
              │   Buffers (in RAM) │◄─── DMA write ────  NIC writes
              │  [buf0][buf1]...   │                      received data
              └─────────────────────┘
```

### TX Descriptor Ring Operation

```
Step 1: Driver prepares descriptor
  ┌─────────────────────────────────────────┐
  │ TX Descriptor Entry                     │
  │  buf_addr  = DMA address of sk_buff data│
  │  length    = packet length              │
  │  cmd       = EOP | IFCS | RS            │
  │  status    = 0 (NIC sets DD when done)  │
  └─────────────────────────────────────────┘

Step 2: Driver advances tail pointer
  Ring pointers:
    HEAD (NIC reads, points to next descriptor to transmit)
    TAIL (driver writes, points past last valid descriptor)

    ┌───┬───┬───┬───┬───┬───┬───┬───┐
    │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │
    └───┴───┴───┴───┴───┴───┴───┴───┘
          ↑               ↑
         HEAD            TAIL
    NIC transmits 1,2,3,4 then advances HEAD to 5

Step 3: NIC reads descriptor, fetches packet data via DMA
Step 4: NIC sets DD (Descriptor Done) bit in status
Step 5: Driver cleans up: unmaps DMA, frees sk_buff
```

### RX Descriptor Ring Operation

```
Step 1: Driver pre-allocates buffers, fills descriptors
  ┌─────────────────────────────────────────┐
  │ RX Descriptor Entry                     │
  │  buf_addr  = DMA address of empty buffer│
  │  length    = 0 (NIC fills after write)  │
  │  status    = 0 (NIC sets DD when ready) │
  └─────────────────────────────────────────┘

Step 2: Packet arrives
  - NIC writes packet data to buffer via DMA
  - NIC updates descriptor: length, status (DD=1), checksum

Step 3: Interrupt fires (or NAPI polls)
  - Driver reads descriptor, checks DD bit
  - Creates sk_buff pointing to the DMA buffer
  - Passes sk_buff up the stack via napi_gro_receive()
  - Allocates new buffer, updates descriptor
  - Advances tail pointer
```

---

## 3.5 Interrupt Handling in Hardware

### Interrupt Types

```
Legacy INTx:
  - Shared interrupt line (multiple devices share one IRQ)
  - Level-triggered: must read NIC register to clear
  - Problems: interrupt sharing overhead, routing complexity

MSI (Message Signaled Interrupts):
  - PCI write to special address = interrupt
  - No shared lines — each device has unique vector
  - Edge-triggered: no need to clear

MSI-X (Extended MSI):
  - Multiple interrupt vectors per device (up to 2048)
  - Each TX/RX queue can have its own interrupt vector
  - Maps to specific CPU cores for locality
  - Essential for multiqueue NICs

  MSI-X Usage:
    Queue 0 (RX) → IRQ 45 → CPU 0
    Queue 1 (RX) → IRQ 46 → CPU 1
    Queue 2 (RX) → IRQ 47 → CPU 2
    Queue 3 (RX) → IRQ 48 → CPU 3
```

### Interrupt Coalescing

```
Without coalescing:          With coalescing:
  Pkt → IRQ                   Pkt
  Pkt → IRQ                   Pkt
  Pkt → IRQ          →        Pkt     → 1 IRQ (batch)
  Pkt → IRQ                   Pkt
  Pkt → IRQ                   ...

  100K pkt/s = 100K IRQ/s     100K pkt/s = 10K IRQ/s

  ethtool -C eth0 rx-usecs 50 rx-frames 64
    → Interrupt after 50 microseconds OR 64 packets (whichever first)
```

---

## 3.6 PCI/PCIe and Register Access

### PCI BAR (Base Address Register)

```
┌──────────────────────────────────────────────┐
│     PCI Configuration Space (256 bytes)      │
│  Offset 0x10: BAR0 → NIC registers (MMIO)   │
│  Offset 0x14: BAR1 → I/O ports (legacy)     │
│  Offset 0x30: Expansion ROM                  │
└──────────────────────────────────────────────┘

In Linux:
  hw_addr = pci_ioremap_bar(pdev, 0);   // Map BAR0
  val = readl(hw_addr + REG_STATUS);     // Read register
  writel(val, hw_addr + REG_CTRL);       // Write register
```

### Driver PCI Probe Sequence

```c
static int my_nic_probe(struct pci_dev *pdev,
                        const struct pci_device_id *id)
{
    int err;

    /* Enable PCI device */
    err = pci_enable_device_mem(pdev);

    /* Request MMIO regions */
    err = pci_request_regions(pdev, "my_nic");

    /* Set DMA mask (64-bit capable) */
    err = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));

    /* Map BAR0 for register access */
    hw_addr = pci_ioremap_bar(pdev, 0);

    /* Enable bus mastering (allows NIC to DMA) */
    pci_set_master(pdev);

    /* Allocate net_device */
    netdev = alloc_etherdev(sizeof(struct my_priv));

    /* Setup MSI-X interrupts */
    err = pci_alloc_irq_vectors(pdev, 1, num_queues,
                                PCI_IRQ_MSIX | PCI_IRQ_MSI);

    /* Allocate descriptor rings and DMA buffers */
    /* ... */

    /* Register net_device */
    err = register_netdev(netdev);

    return 0;
}
```

---

## 3.7 DMA Mapping Types

```
Coherent (Consistent) DMA:
  ring = dma_alloc_coherent(dev, ring_size, &ring_dma, GFP_KERNEL);
  - Always synchronized between CPU and device
  - Used for: descriptor rings, small control structures
  - Typically uncached — slower CPU access, no sync needed

Streaming DMA:
  dma_addr = dma_map_single(dev, buf, len, DMA_FROM_DEVICE);
  /* ... NIC writes data ... */
  dma_sync_single_for_cpu(dev, dma_addr, len, DMA_FROM_DEVICE);
  /* CPU reads data */
  dma_unmap_single(dev, dma_addr, len, DMA_FROM_DEVICE);
  - Cached — fast CPU access, but requires explicit sync
  - Used for: packet data buffers (large, high throughput)

Direction flags:
  DMA_TO_DEVICE     — CPU → NIC (TX)
  DMA_FROM_DEVICE   — NIC → CPU (RX)
  DMA_BIDIRECTIONAL — Both directions
```

---

## 3.8 Hardware Offloads

Modern NICs offload work from the CPU:

| Offload | Function | Linux Attribute |
|---------|----------|----------------|
| Checksum TX | NIC computes IP/TCP/UDP checksum | NETIF_F_HW_CSUM |
| Checksum RX | NIC verifies checksum on receive | NETIF_F_RXCSUM |
| TSO | TCP Segmentation Offload — NIC splits large TCP segments | NETIF_F_TSO |
| GSO | Generic Segmentation Offload — kernel defers segmentation | NETIF_F_GSO |
| GRO | Generic Receive Offload — kernel merges small packets | NETIF_F_GRO |
| LRO | Large Receive Offload — NIC merges packets (less flexible) | NETIF_F_LRO |
| RSS | Receive Side Scaling — NIC distributes RX across queues | ethtool -X |
| VLAN | VLAN tag insertion/stripping in hardware | NETIF_F_HW_VLAN_CTAG_TX |
| Flow steering | NIC classifies flows to queues by rules | ethtool -N |

```bash
# View offload status
ethtool -k eth0

# Enable/disable TSO
ethtool -K eth0 tso on
ethtool -K eth0 tso off

# View ring sizes
ethtool -g eth0

# Set ring sizes
ethtool -G eth0 rx 4096 tx 4096
```

---

## Kernel Source References

| File | Purpose |
|------|---------|
| drivers/net/ethernet/ | Ethernet NIC drivers |
| drivers/net/phy/ | PHY drivers and MDIO bus |
| include/linux/netdevice.h | net_device structure |
| include/linux/etherdevice.h | Ethernet helper functions |
| include/linux/pci.h | PCI driver interface |
| include/linux/dma-mapping.h | DMA API |
| drivers/pci/ | PCI core and MSI-X |

---

## Interview Questions

**Q1: Explain how a NIC receives a packet at the hardware level.**
A: (1) Signal arrives on wire, PHY demodulates to digital bits. (2) MAC checks preamble/SFD, extracts frame. (3) MAC verifies CRC — drops if bad. (4) MAC filters destination address (unicast/multicast/broadcast). (5) DMA engine reads the next RX descriptor from host memory to find buffer address. (6) DMA engine writes packet data to the host buffer. (7) DMA engine updates the descriptor (length, status, checksum). (8) NIC triggers interrupt (or defers if coalescing). (9) Driver reads descriptor, creates sk_buff, passes up stack.

**Q2: What is the difference between coherent and streaming DMA?**
A: Coherent DMA (dma_alloc_coherent) provides memory that's always consistent between CPU and device — typically uncached, no sync calls needed, used for descriptor rings. Streaming DMA (dma_map_single/sg) maps existing memory for DMA — cached for performance but requires explicit dma_sync calls before CPU reads data written by device or after CPU writes data for device to read.

**Q3: Why is MSI-X important for modern networking?**
A: MSI-X provides multiple interrupt vectors per device (up to 2048). Each NIC queue gets its own IRQ that can be pinned to a specific CPU core. This enables per-queue, per-CPU processing — packet RX on CPU 0 doesn't contend with CPU 1's packets. Without MSI-X, a single interrupt line serializes all packet processing.

**Q4: What is interrupt coalescing and when would you tune it?**
A: Interrupt coalescing batches multiple packet events into a single interrupt. Configured by time delay (rx-usecs) or packet count (rx-frames). Increase coalescing for high throughput (reduces CPU overhead from 100K IRQ/s). Decrease for low latency (sub-microsecond response). `ethtool -C eth0 rx-usecs 50` sets a 50-microsecond delay.

**Q5: Walk through the TX path at the hardware level.**
A: (1) Driver writes packet data to a DMA-mapped buffer. (2) Driver fills TX descriptor: buffer DMA address, length, flags (end-of-packet, insert checksum). (3) Driver advances ring tail pointer (MMIO write to NIC register). (4) NIC reads descriptor via DMA. (5) NIC fetches packet data via DMA. (6) NIC computes checksum if offloaded. (7) NIC passes frame to MAC. (8) MAC adds preamble/FCS, transmits via PHY. (9) NIC sets descriptor done bit. (10) Driver cleans up descriptor, unmaps DMA, frees sk_buff.

---

## Summary

- NIC comprises PHY (physical), MAC (framing), DMA engine, and PCIe bridge
- Descriptor rings are circular buffers in host memory that coordinate DMA transfers
- MSI-X provides per-queue interrupts enabling multi-core scaling
- Interrupt coalescing trades latency for throughput by batching interrupts
- Coherent DMA for descriptors; streaming DMA for packet data
- Hardware offloads (checksum, TSO, GRO, RSS) dramatically reduce CPU load
- PCI probe maps BARs, enables bus mastering, allocates IRQ vectors

---

Next: [Chapter 4 — Linux Networking Architecture Overview](Chapter_04_Linux_Networking_Architecture.md)
