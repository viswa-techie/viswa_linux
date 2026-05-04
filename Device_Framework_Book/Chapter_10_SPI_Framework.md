# Chapter 10: SPI Framework

## Learning Goals
- Understand SPI protocol and bus topology
- Know Linux SPI controller and device driver model
- Write SPI device drivers using spi_transfer and regmap
- Compare SPI vs I2C for embedded design choices

---

## 10.1 SPI Protocol Fundamentals

```
SPI Bus Topology:

             Master (SoC)
         ┌───────────────────┐
         │      SPI Ctrl     │
         │  MOSI ──────────► │────────────► Slave 0 (Flash)
         │  MISO ◄────────── │◄──────────── (W25Q128)
         │  SCLK ──────────► │────────────►
         │  CS0  ──────────► │────────────► (active low)
         │  CS1  ──────────► │───┐
         │  CS2  ──────────► │─┐ │
         └───────────────────┘ │ │
                               │ └────────► Slave 1 (ADC)
                               │            (MCP3208)
                               └──────────► Slave 2 (Display)
                                            (ILI9341)

SPI Transaction:
  ┌──────┐     ┌──────┐
  │Master│     │Slave │
  └──┬───┘     └──┬───┘
     │  CS LOW    │         ← Select slave
     │───────────►│
     │            │
     │  SCLK      │         ← Clock from master
     │═══════════>│
     │            │
     │  MOSI data │         ← Master sends
     │───────────►│
     │            │
     │  MISO data │         ← Slave responds (simultaneous)
     │◄───────────│
     │            │
     │  CS HIGH   │         ← Deselect
     │───────────►│

SPI Modes (CPOL / CPHA):
  ┌──────┬──────┬──────┬─────────────────────────┐
  │ Mode │ CPOL │ CPHA │ Description              │
  ├──────┼──────┼──────┼─────────────────────────┤
  │  0   │  0   │  0   │ Idle low, sample rising  │
  │  1   │  0   │  1   │ Idle low, sample falling │
  │  2   │  1   │  0   │ Idle high, sample falling│
  │  3   │  1   │  1   │ Idle high, sample rising │
  └──────┴──────┴──────┴─────────────────────────┘

Key properties:
  - 4 wires: MOSI, MISO, SCLK, CS (per slave)
  - Full duplex (simultaneous TX and RX)
  - No addressing — CS selects the slave
  - Speeds: up to 100+ MHz (vs 3.4 MHz I2C max)
  - No ACK mechanism — master must know slave behavior
```

---

## 10.2 Linux SPI Framework Architecture

```
SPI Framework:

Kernel:
  ┌──────────────────────────────────────────────────┐
  │  SPI Device Drivers (clients)                     │
  │  ├── spi-nor (NOR flash: W25Q, MX25)             │
  │  ├── mcp320x (ADC)                               │
  │  ├── fbtft (SPI displays)                         │
  │  ├── CAN controllers (MCP2515)                    │
  │  └── Custom sensor/actuator drivers               │
  ├──────────────────────────────────────────────────┤
  │  SPI Core (drivers/spi/spi.c)                     │
  │  ├── spi_message / spi_transfer management        │
  │  ├── Device matching (DT compatible / id_table)   │
  │  ├── Transfer queuing and scheduling              │
  │  └── DMA mapping for large transfers              │
  ├──────────────────────────────────────────────────┤
  │  SPI Controller (Master) Drivers                  │
  │  ├── spi-qcom-qspi (Qualcomm)                    │
  │  ├── spi-imx (NXP i.MX)                          │
  │  ├── spi-bcm2835 (Broadcom/RPi)                  │
  │  ├── spi-rockchip (Rockchip)                      │
  │  └── spi-dw (DesignWare/Synopsys)                 │
  └──────────────────────────────────────────────────┘
```

---

## 10.3 SPI Device Driver

```c
/* SPI device driver (e.g., SPI ADC) */

static const struct spi_device_id mcp3208_ids[] = {
    { "mcp3208", 8 },
    { }
};

static const struct of_device_id mcp3208_of_match[] = {
    { .compatible = "microchip,mcp3208" },
    { }
};

static int mcp3208_probe(struct spi_device *spi)
{
    /* Configure SPI mode and speed */
    spi->mode = SPI_MODE_0;
    spi->bits_per_word = 8;
    spi->max_speed_hz = 1000000;  /* 1 MHz */
    spi_setup(spi);

    /* Perform a SPI transfer */
    u8 tx_buf[3] = { 0x06, 0x00, 0x00 };  /* Start, channel 0 */
    u8 rx_buf[3] = { 0 };
    struct spi_transfer xfer = {
        .tx_buf = tx_buf,
        .rx_buf = rx_buf,
        .len    = 3,
    };
    struct spi_message msg;
    spi_message_init(&msg);
    spi_message_add_tail(&xfer, &msg);

    int ret = spi_sync(spi, &msg);  /* blocking transfer */
    if (ret == 0) {
        int adc_value = ((rx_buf[1] & 0x0F) << 8) | rx_buf[2];
        dev_info(&spi->dev, "ADC channel 0 = %d\n", adc_value);
    }

    return ret;
}

static struct spi_driver mcp3208_driver = {
    .driver = {
        .name = "mcp3208",
        .of_match_table = mcp3208_of_match,
    },
    .probe    = mcp3208_probe,
    .id_table = mcp3208_ids,
};
module_spi_driver(mcp3208_driver);
```

---

## 10.4 SPI Transfer API

```c
/* SPI transfer mechanisms */

/* 1. Simple read/write helpers */
u8 cmd = 0x9F;  /* JEDEC ID command */
u8 id[3];
spi_write(spi, &cmd, 1);
spi_read(spi, id, 3);

/* 2. Write-then-read (most common) */
u8 cmd = 0x9F;
u8 id[3];
spi_write_then_read(spi, &cmd, 1, id, 3);

/* 3. Multi-segment transfer */
struct spi_transfer xfers[2];
struct spi_message msg;

spi_message_init(&msg);

/* Segment 1: send command (CS stays low) */
memset(&xfers[0], 0, sizeof(xfers[0]));
xfers[0].tx_buf = &cmd;
xfers[0].len = 1;
spi_message_add_tail(&xfers[0], &msg);

/* Segment 2: receive data (CS stays low, then goes high) */
memset(&xfers[1], 0, sizeof(xfers[1]));
xfers[1].rx_buf = data;
xfers[1].len = len;
spi_message_add_tail(&xfers[1], &msg);

ret = spi_sync(spi, &msg);  /* Execute all segments atomically */

/* 4. Async transfer (non-blocking) */
msg.complete = my_spi_complete;  /* callback */
msg.context  = priv;
ret = spi_async(spi, &msg);     /* returns immediately */
/* my_spi_complete() called when done */

/* 5. SPI with regmap */
struct regmap *regmap = devm_regmap_init_spi(spi, &my_regmap_config);
regmap_read(regmap, REG_STATUS, &val);
```

---

## 10.5 SPI Controller (Master) Driver

```c
/* SPI controller driver — manages SPI hardware */

static int my_spi_transfer_one(struct spi_controller *ctlr,
                                struct spi_device *spi,
                                struct spi_transfer *xfer)
{
    struct my_spi *priv = spi_controller_get_devdata(ctlr);

    /* Program clock speed */
    u32 div = priv->clk_rate / xfer->speed_hz;
    writel(div, priv->regs + SPI_CLK_DIV);

    /* Program bits per word */
    writel(xfer->bits_per_word, priv->regs + SPI_BPW);

    /* Transmit data */
    if (xfer->tx_buf) {
        for (int i = 0; i < xfer->len; i++)
            writel(((u8 *)xfer->tx_buf)[i], priv->regs + SPI_TX);
    }

    /* Receive data */
    if (xfer->rx_buf) {
        for (int i = 0; i < xfer->len; i++)
            ((u8 *)xfer->rx_buf)[i] = readl(priv->regs + SPI_RX);
    }

    return 0;  /* 0 = done, positive = still running */
}

static int my_spi_probe(struct platform_device *pdev)
{
    struct spi_controller *ctlr;

    ctlr = devm_spi_alloc_master(&pdev->dev, sizeof(struct my_spi));
    ctlr->bus_num = pdev->id;
    ctlr->num_chipselect = 4;
    ctlr->mode_bits = SPI_CPOL | SPI_CPHA | SPI_CS_HIGH;
    ctlr->bits_per_word_mask = SPI_BPW_MASK(8) | SPI_BPW_MASK(16);
    ctlr->transfer_one = my_spi_transfer_one;
    ctlr->set_cs = my_spi_set_cs;
    ctlr->dev.of_node = pdev->dev.of_node;

    return devm_spi_register_controller(&pdev->dev, ctlr);
}
```

---

## 10.6 Device Tree and SPI vs I2C

```dts
/* SPI device tree */
&spi0 {
    status = "okay";
    #address-cells = <1>;
    #size-cells = <0>;

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;                      /* CS0 */
        spi-max-frequency = <50000000>; /* 50 MHz */
        spi-cpol;                       /* Mode 2 or 3 */
    };

    adc@1 {
        compatible = "microchip,mcp3208";
        reg = <1>;                      /* CS1 */
        spi-max-frequency = <1000000>;  /* 1 MHz */
    };
};
```

```
SPI vs I2C Comparison:

┌──────────────┬──────────────────┬──────────────────┐
│ Feature      │ SPI              │ I2C              │
├──────────────┼──────────────────┼──────────────────┤
│ Wires        │ 4 + 1 CS/slave   │ 2 (shared)       │
│ Speed        │ Up to 100+ MHz   │ Up to 3.4 MHz    │
│ Duplex       │ Full duplex      │ Half duplex      │
│ Addressing   │ CS pin (no addr) │ 7/10-bit address │
│ Multi-master │ Complex          │ Supported        │
│ ACK          │ None             │ Yes (per byte)   │
│ Distance     │ Short (PCB)      │ Longer possible  │
│ Use case     │ Flash, ADC, LCD  │ Sensor, PMIC,    │
│              │ High-speed data  │ EEPROM, RTC      │
└──────────────┴──────────────────┴──────────────────┘
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| SPI core | drivers/spi/spi.c | Bus, device, transfer |
| SPI headers | include/linux/spi/spi.h | Core structures |
| SPI controllers | drivers/spi/ (spi-*.c) | Controller drivers |
| SPI NOR | drivers/mtd/spi-nor/ | NOR flash over SPI |
| regmap SPI | drivers/base/regmap/regmap-spi.c | regmap for SPI |

---

## Interview Questions

**Q1: Explain the SPI transfer model in Linux.**
A: SPI uses `spi_message` containing one or more `spi_transfer` segments. Each transfer specifies `tx_buf`, `rx_buf`, `len`, `speed_hz`, and `bits_per_word`. The SPI core manages CS assertion: CS goes low before the first transfer in a message and high after the last. Between segments, CS stays low (unless `cs_change` is set). `spi_sync()` blocks until complete; `spi_async()` returns immediately and calls a completion callback. For simple operations, `spi_write_then_read()` handles the common write-command-then-read-data pattern.

**Q2: How does SPI device matching work in Linux with Device Tree?**
A: The SPI controller's DT node has child nodes for each slave device. Each child has `reg = <N>` (chip-select number), `compatible` (for matching), and optional properties like `spi-max-frequency`, `spi-cpol`, `spi-cpha`. The SPI core creates `spi_device` for each child node and matches against registered `spi_driver` using the compatible string (via `of_match_table`) or the device name (via `id_table`). When matched, the driver's `probe()` is called with the `spi_device`.

**Q3: When should you use SPI vs I2C?**
A: Use SPI when: (1) High data rates are needed (flash, ADC, display — up to 100+ MHz). (2) Full-duplex communication is required. (3) Low latency is critical. Use I2C when: (1) Many devices share a bus (sensors, PMICs — only 2 wires). (2) Devices have built-in addressing (no extra CS pins). (3) Moderate speeds suffice (PMICs, RTCs, EEPROMs). (4) PCB space is limited (2 wires vs 4+). In automotive, I2C is common for PMICs and sensors; SPI for flash storage and high-speed peripherals.

---

## Summary

- SPI uses 4 wires (MOSI, MISO, SCLK, CS) — full-duplex, no addressing
- Linux SPI framework: controller (master) driver + device (client) driver
- Transfers use `spi_message` with `spi_transfer` segments
- CS management is automatic per message (stays low across segments)
- regmap works with SPI — same API as I2C
- SPI is preferred over I2C for high-speed data (flash, ADC, displays)

---

*Next: [Chapter 11 — GPIO Subsystem](Chapter_11_GPIO_Subsystem.md)*
