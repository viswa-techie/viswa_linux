# Chapter 18: Framework Interaction Architecture

## Learning Goals
- Understand how Linux device frameworks interact in real systems
- Know DMA-buf sharing between V4L2, DRM, and GPU
- Grasp cross-framework resource management patterns
- Debug complex multi-framework issues

---

## 18.1 Framework Ecosystem Map

```
Framework Interaction Map:

                    ┌─────────────┐
                    │ Device Tree │
                    │ (describes  │
                    │  all HW)    │
                    └──────┬──────┘
                           │ parsed by OF core
           ┌───────────────┼───────────────────┐
           │               │                   │
     ┌─────▼─────┐  ┌─────▼─────┐  ┌─────────▼──────┐
     │  Clock    │  │ Regulator │  │ Power Domain  │
     │ Framework │  │ Framework │  │ (genpd)       │
     └─────┬─────┘  └─────┬─────┘  └───────┬────────┘
           │               │               │
           │   ┌───────────┼───────────┐   │
           └──►│   Device Drivers      │◄──┘
               │                       │
               │  ┌───────┐ ┌───────┐  │
               │  │ V4L2  │ │  DRM  │  │
               │  │Camera │ │Display│  │
               │  └───┬───┘ └───┬───┘  │
               │      │  DMA-buf│      │
               │      └────┬───┘      │
               │           │           │
               │  ┌────────▼────────┐  │
               │  │  DMA Engine     │  │
               │  │  (data movers)  │  │
               │  └────────┬────────┘  │
               │           │           │
               │  ┌────────▼────────┐  │
               │  │   Bus Layer     │  │
               │  │  I2C/SPI/USB    │  │
               │  └─────────────────┘  │
               └───────────────────────┘
                           │
                    ┌──────▼──────┐
                    │  Thermal    │
                    │  Framework  │
                    │ (monitors   │
                    │  all above) │
                    └─────────────┘
```

---

## 18.2 DMA-buf: Zero-Copy Between Frameworks

```
DMA-buf — Zero-Copy Buffer Sharing:

Camera captures → GPU processes → Display shows

Without DMA-buf (3 copies):
  Camera DMA → Buffer A → memcpy → Buffer B (GPU) → memcpy → Buffer C (Display)
  Problem: 33MB × 3 × 60fps = 6 GB/s wasted bandwidth

With DMA-buf (0 copies):
  ┌──────────── Single physical buffer ────────────┐
  │                                                 │
  │  Camera DMA writes frame                        │
  │       ↓ (same memory)                           │
  │  GPU reads for processing                       │
  │       ↓ (same memory)                           │
  │  Display scans out to screen                    │
  │                                                 │
  └─────────────────────────────────────────────────┘

DMA-buf Flow:
  ┌─────────┐     fd     ┌─────────┐     fd     ┌─────────┐
  │  V4L2   │──export───►│ User    │──import───►│   DRM   │
  │ (Camera)│  DMA-buf   │ Space   │  DMA-buf   │(Display)│
  │         │    fd      │         │    fd      │         │
  └─────────┘            └─────────┘            └─────────┘
                              │
                         ┌────▼────┐
                         │  GPU    │  can also import
                         │         │  the same fd
                         └─────────┘

API:
  Exporter: dma_buf_export() → returns struct dma_buf
  File fd:  dma_buf_fd()     → returns file descriptor
  Importer: dma_buf_get()    → import fd to dma_buf
  Map:      dma_buf_map_attachment() → get sg_table for DMA
```

---

## 18.3 Automotive Display Pipeline Example

```
Automotive Camera-to-Display Pipeline:

┌──────────┐    ┌─────────┐    ┌─────────┐    ┌──────────┐
│  MIPI    │i2c │  V4L2   │    │ GPU     │    │  DRM     │
│  Camera  │────│  Driver  │    │ (OpenGL)│    │  Display │
│  Sensor  │    │ /dev/    │    │         │    │ /dev/dri/│
│          │    │ video0   │    │         │    │ card0    │
└──────────┘    └────┬─────┘    └────┬────┘    └────┬─────┘
                     │               │              │
    Resources used:  │               │              │
    ┌────────────────┤               │              │
    │                │               │              │
  ┌─▼──────────┐  ┌─▼──────────┐  ┌▼──────────┐  ┌▼──────────┐
  │ Clock      │  │ DMA Engine │  │ DMA-buf   │  │ Regulator │
  │ Framework  │  │ (video DMA)│  │ (shared   │  │ Framework │
  │            │  │            │  │  buffers) │  │ (display  │
  │ CSI clock  │  │ Frame DMA  │  │ camera→   │  │  power)   │
  │ ISP clock  │  │ scatter-   │  │ GPU→      │  │           │
  │ Pixel clock│  │ gather     │  │ display   │  │ AVDD,DVDD│
  └────────────┘  └────────────┘  └───────────┘  └───────────┘
        │               │              │              │
  ┌─────▼───────────────▼──────────────▼──────────────▼──────┐
  │                  GPIO Framework                          │
  │  Camera enable, display backlight, reset pins            │
  ├──────────────────────────────────────────────────────────┤
  │                  Runtime PM                              │
  │  Camera PM domain, display PM domain, GPU PM domain      │
  ├──────────────────────────────────────────────────────────┤
  │                  Thermal Framework                       │
  │  GPU thermal throttling, SoC temperature monitoring      │
  └──────────────────────────────────────────────────────────┘
```

---

## 18.4 Camera Driver Interactions

```c
/* Camera driver — interacts with multiple frameworks */

static int camera_probe(struct platform_device *pdev)
{
    /* Clock framework */
    priv->mclk = devm_clk_get(&pdev->dev, "mclk");
    clk_set_rate(priv->mclk, 24000000);  /* 24 MHz */

    /* Regulator framework */
    priv->avdd = devm_regulator_get(&pdev->dev, "avdd");
    priv->dvdd = devm_regulator_get(&pdev->dev, "dvdd");

    /* GPIO framework */
    priv->reset = devm_gpiod_get(&pdev->dev, "reset", GPIOD_OUT_HIGH);
    priv->enable = devm_gpiod_get(&pdev->dev, "enable", GPIOD_OUT_LOW);

    /* DMA framework */
    priv->dma_chan = dma_request_chan(&pdev->dev, "rx");

    /* Power-up sequence (order matters!) */
    regulator_enable(priv->avdd);      /* analog power first */
    usleep_range(1000, 2000);
    regulator_enable(priv->dvdd);      /* digital power */
    usleep_range(1000, 2000);
    clk_prepare_enable(priv->mclk);    /* master clock */
    usleep_range(1000, 2000);
    gpiod_set_value(priv->reset, 0);  /* release reset */
    msleep(10);

    /* V4L2 framework registration */
    v4l2_device_register(&pdev->dev, &priv->v4l2_dev);
    /* ... register video_device, subdev, media_device ... */

    /* Runtime PM */
    pm_runtime_enable(&pdev->dev);

    return 0;
}
```

---

## 18.5 Framework Initialization Order

```
Boot-time Framework Initialization Order:

Order matters — consumers need providers registered first:

1. Core subsystems (early)
   ├── IRQ domain / interrupt controller
   ├── Clock framework (clk_init)
   └── GPIO controller

2. Bus controllers
   ├── I2C adapter
   ├── SPI controller
   └── USB host controller

3. Power infrastructure
   ├── PMIC (over I2C/SPI) → registers regulators
   ├── Power domains (genpd)
   └── Runtime PM enabled

4. Framework providers
   ├── DMA controller
   ├── Pinctrl
   └── Reset controller

5. Device drivers (consumers)
   ├── Camera sensor (I2C + clk + regulator + GPIO + V4L2)
   ├── Display controller (DRM + clk + regulator)
   ├── Audio codec (ASoC + I2C + clk + regulator)
   └── Network (netdev + DMA + clk)

Deferred probing handles ordering:
  - If sensor probes before PMIC → regulator_get() returns -EPROBE_DEFER
  - Kernel retries probe later after PMIC registers regulators
  - This automatically resolves dependency ordering
```

---

## 18.6 Deferred Probing

```c
/* Deferred probing — automatic dependency resolution */

static int my_sensor_probe(struct platform_device *pdev)
{
    struct regulator *vdd;

    /* If PMIC hasn't probed yet, this returns -EPROBE_DEFER */
    vdd = devm_regulator_get(&pdev->dev, "vdd");
    if (IS_ERR(vdd)) {
        if (PTR_ERR(vdd) == -EPROBE_DEFER)
            /* Kernel will retry this probe() later */
            return -EPROBE_DEFER;
        return PTR_ERR(vdd);
    }

    /* Similarly for clocks, GPIOs, etc. */
    struct clk *clk = devm_clk_get(&pdev->dev, "mclk");
    if (IS_ERR(clk))
        return PTR_ERR(clk);  /* may be -EPROBE_DEFER */

    /* All dependencies satisfied — continue with probe */
    return 0;
}

/*
 * Deferred probe order:
 * 1. my_sensor probes → vdd regulator not yet → -EPROBE_DEFER
 * 2. pmic probes → registers LDO1 regulator
 * 3. Kernel retries my_sensor probe → gets regulator → succeeds
 */
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| DMA-buf | drivers/dma-buf/dma-buf.c | Buffer sharing |
| Deferred probe | drivers/base/dd.c | Retry mechanism |
| PM domain | drivers/base/power/domain.c | Power domain links |
| Resource managed | drivers/base/devres.c | devm_ resource tracking |

---

## Interview Questions

**Q1: How does DMA-buf enable zero-copy between camera and display?**
A: DMA-buf represents a buffer as a file descriptor that can be shared between device drivers. The camera (V4L2) allocates a DMA-capable buffer and exports it as a dma-buf fd. User space passes this fd to the display (DRM) for import. Both drivers access the same physical memory via different `sg_table` mappings for their respective DMA engines. No CPU copy occurs. This saves massive bandwidth — a 4K RGBA frame at 60fps would require 2GB/s per copy. DMA-buf also handles synchronization fencing between producer and consumer.

**Q2: What is deferred probing and why is it needed?**
A: Deferred probing solves driver dependency ordering. When a device driver's `probe()` requests a resource (regulator, clock, GPIO) whose provider hasn't registered yet, the framework returns `-EPROBE_DEFER`. The kernel removes the device from the normal queue and retries it later. After all initial probes complete, the kernel processes the deferred list repeatedly until all devices probe successfully or no progress is made. This eliminates the need for explicit initialization ordering — drivers just request resources, and the kernel sorts out the order.

**Q3: In a camera system, which frameworks interact and in what order?**
A: Power-on sequence: (1) **Regulator** — enable analog/digital power supplies (AVDD, DVDD). (2) **Clock** — enable master clock (MCLK) to the sensor. (3) **GPIO** — release reset pin. (4) **I2C** — configure sensor registers. (5) **V4L2/Media** — register video devices and pipeline. (6) **DMA** — setup frame capture DMA. During capture: V4L2 uses DMA engine to transfer frames from CSI to memory. DMA-buf shares frames with DRM/GPU. Runtime PM gates clocks/regulators when camera is idle. Thermal framework may throttle ISP if SoC overheats.

---

## Summary

- Real drivers interact with 5-10 frameworks simultaneously (clk, regulator, GPIO, DMA, etc.)
- DMA-buf enables zero-copy buffer sharing between V4L2, DRM, and GPU
- Deferred probing automatically resolves driver dependency ordering
- Power-up sequences must follow specific order (power → clock → reset → register)
- Framework providers (PMIC, clock controller) must probe before consumers
- devm_ managed resources simplify cross-framework cleanup on error/remove

---

*Next: [Chapter 19 — Framework Data Structures Deep Dive](Chapter_19_Data_Structures.md)*
