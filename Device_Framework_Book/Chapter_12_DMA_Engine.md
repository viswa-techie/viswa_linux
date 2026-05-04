# Chapter 12: DMA Engine Framework

## Learning Goals
- Understand DMA concepts and why DMA eliminates CPU overhead
- Know the Linux DMA engine (dmaengine) consumer and provider APIs
- Grasp DMA transfer types: memcpy, slave, cyclic, scatter-gather
- Write DMA consumer code using the dmaengine API

---

## 12.1 DMA Architecture Overview

```
DMA — Direct Memory Access:

Without DMA (PIO mode):
  CPU reads byte from peripheral register
  CPU writes byte to memory
  CPU reads next byte...
  (CPU busy 100% during transfer)

With DMA:
  CPU programs DMA controller: source, dest, length
  CPU does other work (or sleeps)
  DMA controller moves data autonomously
  DMA controller signals completion via interrupt

                    ┌──────────────┐
                    │     CPU      │
                    │ (free to do  │
                    │  other work) │
                    └──────┬───────┘
                           │ programs DMA
                    ┌──────▼───────┐
                    │ DMA Controller│
                    │  (hardware)  │
                    └──┬───────┬───┘
                       │       │
              ┌────────▼───┐   │
              │  Memory    │   │  DMA transfer
              │  (RAM)     │◄──┘  (no CPU involvement)
              └────────────┘
                       ▲
                       │ DMA read
              ┌────────┴───┐
              │ Peripheral │
              │ FIFO (SPI, │
              │ I2S, UART) │
              └────────────┘

DMA Transfer Types:
  mem-to-mem:  RAM → RAM (memcpy acceleration)
  mem-to-dev:  RAM → Peripheral FIFO (TX: playback, SPI TX)
  dev-to-mem:  Peripheral FIFO → RAM (RX: capture, SPI RX)
  dev-to-dev:  Peripheral → Peripheral (rare)
```

---

## 12.2 DMA Engine Framework

```
DMA Engine Framework:

  ┌──────────────────────────────────────────────────┐
  │  DMA Consumers (client drivers)                   │
  │  ├── ALSA platform driver (audio DMA)             │
  │  ├── SPI controller (SPI data DMA)                │
  │  ├── UART driver (serial DMA)                     │
  │  ├── MMC/SD driver (storage DMA)                  │
  │  └── Custom peripheral drivers                    │
  ├──────────────────────────────────────────────────┤
  │  DMA Engine Core (drivers/dma/dmaengine.c)        │
  │  ├── dma_request_chan() — get DMA channel          │
  │  ├── dmaengine_prep_*() — prepare descriptor      │
  │  ├── dmaengine_submit() — submit to queue         │
  │  ├── dma_async_issue_pending() — start transfer   │
  │  └── dma_slave_config() — configure for slave     │
  ├──────────────────────────────────────────────────┤
  │  DMA Controller Drivers (providers)               │
  │  ├── pl330 (ARM PrimeCell DMA)                    │
  │  ├── dma-qcom-bam (Qualcomm BAM-DMA)             │
  │  ├── stm32-dma (STM32 DMA)                       │
  │  ├── edma (TI EDMA3)                              │
  │  └── dw-axi-dmac (DesignWare AXI DMA)            │
  └──────────────────────────────────────────────────┘
```

---

## 12.3 DMA Consumer API — Slave Transfer

```c
/* DMA consumer: slave transfer (peripheral → memory) */
#include <linux/dmaengine.h>

static int my_driver_dma_setup(struct my_device *priv)
{
    struct dma_chan *chan;
    struct dma_slave_config cfg;

    /* 1. Request DMA channel (from device tree 'dmas' property) */
    chan = dma_request_chan(&pdev->dev, "rx");  /* "rx" channel */
    if (IS_ERR(chan))
        return PTR_ERR(chan);

    /* 2. Configure slave DMA */
    memset(&cfg, 0, sizeof(cfg));
    cfg.direction       = DMA_DEV_TO_MEM;
    cfg.src_addr        = priv->phys_base + DATA_REG;  /* peripheral reg */
    cfg.src_addr_width  = DMA_SLAVE_BUSWIDTH_4_BYTES;
    cfg.src_maxburst    = 8;                            /* burst length */
    dmaengine_slave_config(chan, &cfg);

    priv->dma_chan = chan;
    return 0;
}

/* Start a DMA transfer */
static int my_start_dma_transfer(struct my_device *priv)
{
    struct dma_async_tx_descriptor *desc;
    dma_cookie_t cookie;

    /* 3. Prepare the descriptor */
    desc = dmaengine_prep_slave_single(priv->dma_chan,
                                       priv->dma_addr,  /* dest buffer */
                                       priv->buf_size,
                                       DMA_DEV_TO_MEM,
                                       DMA_PREP_INTERRUPT);
    if (!desc)
        return -ENOMEM;

    /* 4. Set completion callback */
    desc->callback = my_dma_complete;
    desc->callback_param = priv;

    /* 5. Submit descriptor to DMA engine queue */
    cookie = dmaengine_submit(desc);

    /* 6. Start the DMA transfer */
    dma_async_issue_pending(priv->dma_chan);

    return 0;
}

/* Completion callback (called from tasklet/softirq) */
static void my_dma_complete(void *data)
{
    struct my_device *priv = data;
    /* Transfer complete — process received data at priv->buf */
    process_data(priv->buf, priv->buf_size);
}
```

---

## 12.4 Cyclic DMA (Audio/Streaming)

```c
/* Cyclic DMA — continuous ring buffer (audio PCM, ADC streaming) */

/*
 * Ring buffer with DMA:
 *
 * ┌──────────┬──────────┬──────────┬──────────┐
 * │ Period 0 │ Period 1 │ Period 2 │ Period 3 │
 * └──────────┴──────────┴──────────┴──────────┘
 *      ▲                                ▲
 *      └── DMA reads here              └── DMA wraps here
 *
 * DMA runs continuously, wrapping from end to start.
 * Interrupt fires after each period — driver processes data.
 */

static int setup_cyclic_dma(struct my_audio *priv)
{
    struct dma_async_tx_descriptor *desc;
    size_t period_size = 4096;
    size_t buf_size = period_size * 4;  /* 4 periods */

    desc = dmaengine_prep_dma_cyclic(priv->dma_chan,
                                      priv->dma_addr,
                                      buf_size,
                                      period_size,
                                      DMA_DEV_TO_MEM,
                                      DMA_PREP_INTERRUPT);

    desc->callback = period_elapsed_callback;
    desc->callback_param = priv;

    priv->cookie = dmaengine_submit(desc);
    dma_async_issue_pending(priv->dma_chan);

    return 0;
}

static void period_elapsed_callback(void *data)
{
    struct my_audio *priv = data;
    /* One period of audio data is ready */
    snd_pcm_period_elapsed(priv->substream);
}
```

---

## 12.5 Scatter-Gather DMA

```c
/* Scatter-gather DMA — transfer to/from non-contiguous buffers */

struct scatterlist sg[4];

sg_init_table(sg, 4);
sg_set_buf(&sg[0], buf0, len0);
sg_set_buf(&sg[1], buf1, len1);
sg_set_buf(&sg[2], buf2, len2);
sg_set_buf(&sg[3], buf3, len3);

/* Map for DMA */
int nents = dma_map_sg(dev, sg, 4, DMA_FROM_DEVICE);

/* Prepare scatter-gather DMA descriptor */
desc = dmaengine_prep_slave_sg(chan, sg, nents,
                                DMA_DEV_TO_MEM,
                                DMA_PREP_INTERRUPT);

desc->callback = sg_complete;
dmaengine_submit(desc);
dma_async_issue_pending(chan);

/* After completion, unmap */
dma_unmap_sg(dev, sg, 4, DMA_FROM_DEVICE);
```

---

## 12.6 DMA Mapping API

```c
/* DMA mapping — getting bus addresses for DMA transfers */

/* 1. Coherent mapping (always consistent, no sync needed) */
void *buf;
dma_addr_t dma_handle;
buf = dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL);
/* buf = CPU virtual address, dma_handle = DMA/bus address */
/* Used for: descriptor rings, small control structures */
dma_free_coherent(dev, size, buf, dma_handle);

/* 2. Streaming mapping (requires sync) */
dma_addr_t addr = dma_map_single(dev, cpu_buf, size, DMA_TO_DEVICE);
/* Before DMA reads: dma_sync_single_for_device(dev, addr, size, dir) */
/* After DMA writes: dma_sync_single_for_cpu(dev, addr, size, dir) */
dma_unmap_single(dev, addr, size, DMA_TO_DEVICE);

/* DMA direction:
 * DMA_TO_DEVICE   — CPU writes, device reads (TX)
 * DMA_FROM_DEVICE — device writes, CPU reads (RX)
 * DMA_BIDIRECTIONAL — both directions
 */
```

---

## 12.7 Device Tree DMA Bindings

```dts
/* DMA consumer in device tree */
my_peripheral: spi@40013000 {
    compatible = "vendor,my-spi";
    reg = <0x40013000 0x400>;
    dmas = <&dma1 4 1>,   /* TX: DMA controller, channel 4, request 1 */
           <&dma1 3 1>;   /* RX: DMA controller, channel 3, request 1 */
    dma-names = "tx", "rx";
};

/* DMA controller (provider) */
dma1: dma-controller@40026000 {
    compatible = "vendor,my-dma";
    reg = <0x40026000 0x400>;
    interrupts = <GIC_SPI 11 IRQ_TYPE_LEVEL_HIGH>;
    #dma-cells = <2>;  /* <channel request_line> */
    dma-channels = <8>;
    dma-requests = <16>;
};
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| DMA engine | drivers/dma/dmaengine.c | Core API |
| DMA headers | include/linux/dmaengine.h | Consumer API |
| DMA mapping | kernel/dma/mapping.c | DMA address mapping |
| DMA API doc | Documentation/core-api/dma-api.rst | DMA API guide |
| DMA controllers | drivers/dma/ (pl330, etc.) | Controller drivers |

---

## Interview Questions

**Q1: Explain the DMA consumer API flow.**
A: (1) `dma_request_chan()` — request a DMA channel by name from device tree. (2) `dmaengine_slave_config()` — configure peripheral address, bus width, burst size, direction. (3) `dmaengine_prep_slave_single/sg/cyclic()` — prepare a transfer descriptor. (4) Set `desc->callback` for completion notification. (5) `dmaengine_submit()` — add descriptor to the channel's queue. (6) `dma_async_issue_pending()` — tell DMA engine to start processing queued descriptors. (7) Completion callback fires when transfer finishes. (8) `dma_release_channel()` to free.

**Q2: What is the difference between coherent and streaming DMA mappings?**
A: Coherent (`dma_alloc_coherent`) — allocates memory that is always consistent between CPU and device views, typically using uncached or write-combined mappings. No explicit sync needed. Used for small, frequently-accessed structures like descriptor rings. Performance cost: uncached reads are slow. Streaming (`dma_map_single/sg`) — maps existing memory for DMA. Requires explicit `dma_sync_*` calls to ensure consistency. Higher performance for large data buffers because the memory can be cached. Must specify direction (TO_DEVICE, FROM_DEVICE).

**Q3: What is cyclic DMA and where is it used?**
A: Cyclic DMA transfers data continuously in a ring buffer, wrapping from end to start without stopping. The DMA engine fires an interrupt after each "period" (a fixed-size chunk of the buffer). Used for: (1) Audio (ALSA PCM) — continuous audio streaming where underflows must be avoided. (2) ADC sampling — constant-rate data acquisition. (3) Video frame capture — continuous frame DMA. The callback processes each completed period while DMA continues filling the next period, enabling real-time streaming without gaps.

---

## Summary

- DMA moves data between memory/peripherals without CPU involvement
- DMA engine framework: consumer API + provider (controller) drivers
- Transfer types: slave (peripheral I/O), cyclic (streaming), scatter-gather
- Flow: request channel → configure → prepare → submit → issue → callback
- Coherent mapping for small control structures; streaming for large data buffers
- Device tree specifies DMA channels with `dmas` and `dma-names` properties

---

*Next: [Chapter 13 — Clock Framework](Chapter_13_Clock_Framework.md)*
