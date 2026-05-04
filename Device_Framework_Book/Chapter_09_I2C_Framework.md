# Chapter 9: I2C Framework

## Learning Goals
- Understand I2C bus architecture and protocol fundamentals
- Know Linux I2C adapter, client, and driver model
- Write I2C device drivers using regmap or raw transfer
- Debug I2C issues with i2c-tools

---

## 9.1 I2C Protocol Basics

```
I2C Bus:

                    VDD
                     │
                    ┌┴┐
                    │R│ Pull-up        Pull-up
                    └┬┘ resistor       resistor
                     │                  │
     SDA ────────────┼──────────────────┼──────────── SDA
                     │                  │
     SCL ────────────┼──────────────────┼──────────── SCL
                     │                  │
               ┌─────┴─────┐      ┌────┴──────┐
               │  Master   │      │  Slave    │
               │ (SoC I2C  │      │ (Sensor,  │
               │ controller)│      │  PMIC,   │
               │            │      │  Codec)  │
               │ Addr: N/A  │      │ Addr:0x68│
               └────────────┘      └──────────┘

I2C Transaction (write):
  START → [Slave Addr (7-bit)] [W] → ACK → [Reg Addr] → ACK → [Data] → ACK → STOP
   _   _____ _____ _____ _____ _ _____ _
  | |_|A6|A5|A4|A3|A2|A1|A0|W| |R7..R0| |D7..D0| |
   S      Address      R/W  ACK  Reg   ACK Data  ACK STOP

I2C Transaction (read):
  START → [Addr] [W] → ACK → [Reg] → ACK →
  RESTART → [Addr] [R] → ACK → [Data] → NACK → STOP

Key properties:
  - 2 wires: SDA (data) + SCL (clock)
  - Multi-master, multi-slave
  - 7-bit or 10-bit addressing
  - Speeds: 100kHz (standard), 400kHz (fast), 1MHz (fast+), 3.4MHz (high)
  - Open-drain with pull-up resistors
```

---

## 9.2 Linux I2C Framework Architecture

```
I2C Framework:

User Space:
  ┌──────────┐  ┌──────────┐
  │ i2cdetect│  │ i2cget   │
  │ i2cdump  │  │ i2cset   │
  └─────┬────┘  └─────┬────┘
        │              │
        └──────┬───────┘
               │  /dev/i2c-N
Kernel:        │
  ┌────────────▼──────────────────────────────────┐
  │  I2C Core (drivers/i2c/i2c-core-*.c)          │
  │                                                │
  │  ┌──────────────────────────────────────────┐  │
  │  │  I2C Client Drivers                      │  │
  │  │  ├── hwmon (LM75, TMP102)                │  │
  │  │  ├── rtc (DS1307, PCF8563)               │  │
  │  │  ├── codec (WM8960, TLV320AIC)           │  │
  │  │  ├── touchscreen (Goodix GT911)          │  │
  │  │  ├── PMIC (PMIC regulators)              │  │
  │  │  └── EEPROM (AT24)                       │  │
  │  └──────────────────────────────────────────┘  │
  │                                                │
  │  ┌──────────────────────────────────────────┐  │
  │  │  I2C Adapter (Bus Controller) Drivers     │  │
  │  │  ├── i2c-qcom-geni (Qualcomm)            │  │
  │  │  ├── i2c-imx (NXP i.MX)                  │  │
  │  │  ├── i2c-rk3x (Rockchip)                 │  │
  │  │  ├── i2c-bcm2835 (Broadcom/RPi)          │  │
  │  │  └── i2c-designware (Synopsys)            │  │
  │  └──────────────────────────────────────────┘  │
  └────────────────────────────────────────────────┘

Terminology:
  Adapter = I2C bus controller (master hardware)
  Client  = I2C slave device instance (bound driver)
  Driver  = I2C client driver (handles a class of devices)
```

---

## 9.3 I2C Client Driver

```c
/* I2C client driver (e.g., temperature sensor) */

static const struct i2c_device_id tmp102_ids[] = {
    { "tmp102", 0 },
    { }
};
MODULE_DEVICE_TABLE(i2c, tmp102_ids);

static const struct of_device_id tmp102_of_match[] = {
    { .compatible = "ti,tmp102" },
    { }
};

static int tmp102_probe(struct i2c_client *client)
{
    struct device *dev = &client->dev;
    u8 buf[2];
    int ret;

    /* Read temperature register (0x00) */
    ret = i2c_smbus_read_word_data(client, 0x00);
    if (ret < 0)
        return ret;

    dev_info(dev, "Temperature: %d.%d°C\n",
             (ret >> 4) / 16, ((ret >> 4) % 16) * 625);

    /* --- Or use raw I2C transfer --- */
    struct i2c_msg msgs[2];
    u8 reg = 0x00;

    /* Write: register address */
    msgs[0].addr  = client->addr;
    msgs[0].flags = 0;            /* write */
    msgs[0].len   = 1;
    msgs[0].buf   = &reg;

    /* Read: 2 bytes of data */
    msgs[1].addr  = client->addr;
    msgs[1].flags = I2C_M_RD;    /* read */
    msgs[1].len   = 2;
    msgs[1].buf   = buf;

    ret = i2c_transfer(client->adapter, msgs, 2);

    return 0;
}

static struct i2c_driver tmp102_driver = {
    .driver = {
        .name = "tmp102",
        .of_match_table = tmp102_of_match,
    },
    .probe = tmp102_probe,
    .id_table = tmp102_ids,
};
module_i2c_driver(tmp102_driver);
```

---

## 9.4 I2C with regmap

```c
/* Using regmap for I2C register access — preferred approach */

#include <linux/regmap.h>

static const struct regmap_config my_regmap_config = {
    .reg_bits   = 8,       /* register address is 8 bits */
    .val_bits   = 8,       /* register value is 8 bits */
    .max_register = 0xFF,
    .cache_type = REGCACHE_RBTREE,  /* cache registers */
};

static int my_sensor_probe(struct i2c_client *client)
{
    struct regmap *regmap;
    unsigned int val;

    /* Create regmap for I2C device */
    regmap = devm_regmap_init_i2c(client, &my_regmap_config);
    if (IS_ERR(regmap))
        return PTR_ERR(regmap);

    /* Read register */
    ret = regmap_read(regmap, REG_WHO_AM_I, &val);

    /* Write register */
    ret = regmap_write(regmap, REG_CTRL, 0x80);

    /* Read-modify-write (set bits 3:2 to 0b10) */
    ret = regmap_update_bits(regmap, REG_CTRL,
                             GENMASK(3, 2),   /* mask */
                             BIT(3));          /* value */

    /* Bulk read */
    u8 data[6];
    ret = regmap_bulk_read(regmap, REG_DATA_START, data, 6);

    return 0;
}

/*
 * regmap advantages:
 * - Same API for I2C, SPI, MMIO
 * - Register caching (avoid redundant reads)
 * - Read-modify-write with locking
 * - Debugfs register dump
 * - Endian handling
 */
```

---

## 9.5 I2C Adapter (Controller) Driver

```c
/* I2C adapter driver — manages the I2C bus controller hardware */

static int my_i2c_xfer(struct i2c_adapter *adap,
                        struct i2c_msg *msgs, int num)
{
    struct my_i2c *i2c = i2c_get_adapdata(adap);

    for (int i = 0; i < num; i++) {
        struct i2c_msg *msg = &msgs[i];

        /* Program hardware: slave address */
        writel(msg->addr << 1, i2c->regs + I2C_ADDR_REG);

        if (msg->flags & I2C_M_RD) {
            /* Read transfer */
            writel(msg->len, i2c->regs + I2C_RX_LEN);
            writel(I2C_START_READ, i2c->regs + I2C_CTRL);
            wait_for_completion(&i2c->done);
            memcpy(msg->buf, i2c->rx_buf, msg->len);
        } else {
            /* Write transfer */
            memcpy(i2c->tx_buf, msg->buf, msg->len);
            writel(msg->len, i2c->regs + I2C_TX_LEN);
            writel(I2C_START_WRITE, i2c->regs + I2C_CTRL);
            wait_for_completion(&i2c->done);
        }
    }
    return num;  /* Return number of messages transferred */
}

static const struct i2c_algorithm my_i2c_algo = {
    .master_xfer   = my_i2c_xfer,
    .functionality = my_i2c_func,
};

static int my_i2c_probe(struct platform_device *pdev)
{
    struct i2c_adapter *adap = &i2c->adap;

    adap->owner = THIS_MODULE;
    adap->algo = &my_i2c_algo;
    adap->dev.parent = &pdev->dev;
    adap->dev.of_node = pdev->dev.of_node;
    strlcpy(adap->name, "my-i2c", sizeof(adap->name));
    i2c_set_adapdata(adap, i2c);

    return i2c_add_adapter(adap);  /* creates /dev/i2c-N */
}
```

---

## 9.6 Device Tree and Debug

```dts
/* I2C device tree */
&i2c1 {
    clock-frequency = <400000>;  /* 400kHz fast mode */
    status = "okay";

    tmp102@48 {
        compatible = "ti,tmp102";
        reg = <0x48>;           /* 7-bit slave address */
    };

    pmic@34 {
        compatible = "vendor,my-pmic";
        reg = <0x34>;
        interrupt-parent = <&gpio1>;
        interrupts = <5 IRQ_TYPE_LEVEL_LOW>;
    };
};
```

```bash
# I2C debugging with i2c-tools

# Detect devices on bus 1
$ i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- 34 -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- 48 -- -- -- -- -- -- --

# Read register
$ i2cget -y 1 0x48 0x00 w    # read word from reg 0x00

# Write register
$ i2cset -y 1 0x48 0x01 0x80  # write 0x80 to reg 0x01

# Dump all registers
$ i2cdump -y 1 0x48

# Kernel I2C debug
$ echo 0x1 > /sys/module/i2c_core/parameters/debug
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| I2C core | drivers/i2c/i2c-core-base.c | Bus, adapter, client |
| I2C SMBUS | drivers/i2c/i2c-core-smbus.c | SMBus protocol helpers |
| I2C OF | drivers/i2c/i2c-core-of.c | Device tree handling |
| Adapters | drivers/i2c/busses/ | Controller drivers |
| i2c.h | include/linux/i2c.h | Core data structures |
| regmap | drivers/base/regmap/ | Register abstraction |

---

## Interview Questions

**Q1: Compare I2C SMBUS vs raw I2C transfer.**
A: SMBus (System Management Bus) is a subset of I2C with standardized transfer protocols: `i2c_smbus_read_byte_data()` reads one byte from a register, `i2c_smbus_write_word_data()` writes a 16-bit value. These functions handle the write-then-read pattern internally. Raw I2C uses `i2c_transfer()` with an array of `i2c_msg` structures — each message specifies address, flags (read/write), buffer, and length. Raw transfer is needed for non-standard protocols, multi-byte sequences, or devices that don't follow SMBus conventions.

**Q2: What is regmap and why is it preferred for I2C drivers?**
A: regmap is an abstraction layer that provides a uniform API for register access regardless of the underlying bus (I2C, SPI, MMIO). Benefits: (1) Same API (`regmap_read/write/update_bits`) works across buses. (2) Register caching avoids redundant I2C reads. (3) `regmap_update_bits()` provides atomic read-modify-write with locking. (4) Automatic endian conversion. (5) debugfs integration for register dump. (6) Range checking against `max_register`. A driver using regmap can be ported from I2C to SPI by changing only the regmap initialization.

**Q3: How does the kernel match I2C devices to drivers?**
A: Three matching mechanisms: (1) **Device Tree** (preferred): Kernel parses I2C adapter's DT node, finds child nodes with `compatible` strings, creates `i2c_client` for each, matches against driver's `of_match_table`. (2) **ACPI**: Similar to DT but uses ACPI device IDs. (3) **Board files** (legacy): `i2c_board_info` structures registered at boot with address and name, matched against driver's `id_table`. In all cases, when a match occurs, the driver's `probe()` is called with the `i2c_client` representing the device.

---

## Summary

- I2C is a 2-wire (SDA/SCL) bus for low-speed peripherals (sensors, PMICs, codecs)
- Linux I2C framework: adapter (controller), client (device), driver (handler)
- regmap provides a bus-agnostic register access API with caching
- SMBus helpers simplify common read/write patterns
- Device tree declares I2C devices as children of the adapter node
- i2c-tools (i2cdetect, i2cget, i2cdump) are essential for debugging

---

*Next: [Chapter 10 — SPI Framework](Chapter_10_SPI_Framework.md)*
