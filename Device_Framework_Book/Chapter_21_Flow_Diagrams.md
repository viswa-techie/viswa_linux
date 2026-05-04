# Chapter 21: End-to-End Flow Diagrams

## Learning Goals
- Trace complete data/control flows through multiple frameworks
- Understand boot-to-first-frame display pipeline
- Know audio playback end-to-end path
- Visualize camera capture pipeline with all framework interactions

---

## 21.1 Camera Capture — End to End

```
Camera Capture: Button Press to JPEG File

User presses "Capture" button in camera app:

1. App: open("/dev/video0")
   └─► kernel: v4l2_open() → driver: my_cam_open()

2. App: ioctl(VIDIOC_S_FMT, 1920x1080 YUYV)
   └─► kernel: v4l2_ioctl → vidioc_s_fmt_vid_cap()
       └─► driver: configure ISP format registers
           └─► v4l2_subdev: s_stream → sensor I2C config
               └─► I2C framework: i2c_transfer()
                   └─► I2C adapter: master_xfer()
                       └─► Hardware: I2C SCL/SDA to sensor

3. App: ioctl(VIDIOC_REQBUFS, count=4, MMAP)
   └─► kernel: vb2 queue_setup()
       └─► DMA: dma_alloc_coherent() × 4 buffers
           └─► CMA: allocate 4 × 4MB contiguous memory

4. App: mmap() each buffer
   └─► kernel: vb2_mmap() → map DMA memory to user VA

5. App: ioctl(VIDIOC_QBUF, index=0,1,2,3)
   └─► kernel: vb2 buf_queue() for each
       └─► driver: program DMA descriptor ring

6. App: ioctl(VIDIOC_STREAMON)
   └─► kernel: vb2 start_streaming()
       └─► driver:
           ├── Clock framework: clk_prepare_enable(csi_clk)
           ├── Regulator: regulator_enable(vdd)
           ├── GPIO: gpiod_set_value(reset, 0) (release reset)
           ├── I2C: sensor_write_regs() (start streaming)
           ├── DMA: dmaengine_prep_dma_cyclic() (start DMA)
           └── IRQ: request_irq() (frame complete)

7. Hardware pipeline running:
   Sensor → MIPI CSI-2 → ISP → DMA → Memory
   ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
   │ IMX219 │───►│ CSI-2  │───►│  ISP   │───►│  DMA   │
   │ sensor │MIPI│receiver│    │process │    │to RAM  │
   └────────┘    └────────┘    └────────┘    └───┬────┘
                                                  │
8. IRQ: frame_complete_isr()                      │
   └─► vb2_buffer_done(buf, VB2_BUF_STATE_DONE)──┘
       └─► wake_up(poll waitqueue)

9. App: poll(fd, POLLIN)  → returns ready
   App: ioctl(VIDIOC_DQBUF) → gets filled buffer index
   └─► kernel: dequeue buffer, return to user space

10. App: process frame data from mmap'd buffer
    └─► JPEG encode
    └─► write to file
    └─► ioctl(VIDIOC_QBUF, index) — re-queue buffer

11. App: ioctl(VIDIOC_STREAMOFF)
    └─► driver:
        ├── DMA: dmaengine_terminate_sync()
        ├── I2C: sensor stop streaming
        ├── Clock: clk_disable_unprepare()
        └── Runtime PM: pm_runtime_put()
```

---

## 21.2 Audio Playback — End to End

```
Audio Playback: MP3 File to Speaker

1. App: open("/dev/snd/pcmC0D0p")  (Card 0, Device 0, Playback)
   └─► ALSA: snd_pcm_open()
       └─► ASoC: soc_pcm_open()
           ├── CPU DAI: i2s_startup() — enable I2S clock
           ├── Codec: wm8960_startup()
           └── DAPM: trace path, power up widgets

2. App: ioctl(SNDRV_PCM_IOCTL_HW_PARAMS)
   └─► Rate=48000, Channels=2, Format=S16_LE
       └─► ASoC: soc_pcm_hw_params()
           ├── CPU DAI: i2s_hw_params()
           │   ├── Clock: clk_set_rate(mclk, 12288000)
           │   └── Configure I2S: 48kHz, 16-bit, stereo
           ├── Codec: wm8960_hw_params()
           │   ├── I2C: set codec clock dividers
           │   └── Configure DAC for 48kHz
           └── DMA: allocate ring buffer
               └── dma_alloc_coherent(period_size × periods)

3. App: mmap() the PCM ring buffer
   └─► User-space direct access to DMA buffer

4. App: write PCM samples to ring buffer
   └─► Advance appl_ptr

5. App: ioctl(SNDRV_PCM_IOCTL_START)
   └─► ASoC: soc_pcm_trigger(START)
       ├── DMA: dmaengine_prep_dma_cyclic()
       │   └─► DMA reads ring buffer → I2S FIFO
       ├── CPU DAI: i2s_trigger() → start I2S TX
       ├── Codec: through DAPM
       │   └─► Power up: DAC → Output Mixer → HP Amp
       └─► Data flow begins:
                                                  Analog
   [Ring Buffer] → DMA → [I2S FIFO] → I2S bus → [WM8960] → Speaker
    hw_ptr moves                        BCLK+     DAC→
    period IRQ                          LRCLK+    Mixer→
    fires every                         DATA      Amp→
    period                                        Output

6. DMA period elapsed:
   └─► period_elapsed_callback()
       └─► snd_pcm_period_elapsed()
           └─► wake up app to write more data

7. App: write more PCM samples → QBUF cycle continues

8. App: ioctl(SNDRV_PCM_IOCTL_STOP)
   └─► ASoC: soc_pcm_trigger(STOP)
       ├── DMA: dmaengine_terminate_sync()
       ├── CPU DAI: stop I2S
       └── DAPM: power down unused widgets
           └─► HP Amp OFF → Mixer OFF → DAC OFF
```

---

## 21.3 Display Pipeline — Boot to First Frame

```
Display: Boot to First Frame

1. Platform Device Creation (OF):
   └─► of_platform_populate() → create platform_device for display

2. Display Controller Probe:
   └─► my_display_probe()
       ├── Clock: devm_clk_get("pixel"), devm_clk_get("ahb")
       ├── Regulator: devm_regulator_get("vdd")
       ├── MMIO: devm_platform_ioremap_resource()
       ├── IRQ: platform_get_irq() — VBlank interrupt
       ├── DRM: drm_dev_alloc()
       │   ├── Create CRTC
       │   ├── Create primary plane
       │   ├── Create encoder
       │   └── Create connector
       └── drm_dev_register() → creates /dev/dri/card0

3. Panel Driver Probe (I2C or DSI):
   └─► Panel EDID read or DT-specified modes
       └─► Connector: drm_connector_attach_encoder()

4. User Space (compositor/SurfaceFlinger):
   a. Open /dev/dri/card0
   b. drmModeGetResources() — discover CRTCs/connectors
   c. drmModeGetConnector() — get available modes
   d. Create framebuffer:
      └─► drmIoctl(DRM_IOCTL_MODE_CREATE_DUMB) → GEM buffer
      └─► drmModeAddFB2() → create framebuffer from GEM

5. Atomic Commit:
   drmModeAtomicAlloc()
   drmModeAtomicAddProperty(plane, fb_id, ...)
   drmModeAtomicAddProperty(plane, crtc_id, ...)
   drmModeAtomicAddProperty(crtc, mode_id, ...)
   drmModeAtomicAddProperty(crtc, active, 1)
   drmModeAtomicCommit(ALLOW_MODESET)
   └─► Kernel:
       ├── atomic_check() — validate all changes
       │   ├── CRTC: check mode valid for display
       │   ├── Plane: check format, scaling, overlap
       │   └── Encoder: check mode fits PHY limits
       ├── atomic_commit() — apply changes
       │   ├── Clock: clk_set_rate(pixel_clk, mode_clock)
       │   ├── Encoder: configure DSI/LVDS/HDMI PHY
       │   ├── CRTC: program timing registers
       │   │   └─► HFP, HBP, HSW, VFP, VBP, VSW, active
       │   └── Plane: program scanout address
       │       └─► writel(gem->dma_addr, SCANOUT_ADDR)
       └─► First VBlank → pixels flowing to display!

   Memory → DMA scanout → CRTC timing → Encoder → Connector → Panel
   ┌─────┐   ┌─────┐      ┌─────┐     ┌──────┐   ┌──────┐  ┌─────┐
   │ GEM │──►│Plane│─────►│CRTC │────►│Encdr │──►│Conn  │─►│Panel│
   │ buf │   │     │      │     │     │DSI/  │  │      │  │LCD  │
   └─────┘   └─────┘      └─────┘     │HDMI  │  └──────┘  └─────┘
                                       └──────┘
```

---

## 21.4 Network Packet — End to End

```
Network: HTTP GET Request and Response

=== TX Path (sending request) ===

1. App: send(sock, "GET / HTTP/1.1\r\n...", len)
   └─► Socket: sock_sendmsg()
       └─► TCP: tcp_sendmsg()
           ├── Allocate sk_buff
           ├── Copy user data → skb
           ├── Add TCP header (src/dst port, seq, ack)
           └─► tcp_write_xmit()
               └─► IP: ip_queue_xmit()
                   ├── Route lookup → output device
                   ├── Add IP header (src/dst addr, TTL)
                   ├── Netfilter: NF_INET_LOCAL_OUT
                   └─► Neighbor: resolve MAC (ARP cache)
                       ├── Add Ethernet header
                       └─► dev_queue_xmit()
                           ├── TC qdisc (scheduling)
                           └─► Driver: ndo_start_xmit(skb)
                               ├── dma_map_single(skb->data)
                               ├── Fill TX descriptor
                               ├── Trigger DMA
                               └─► NIC sends packet

=== RX Path (receiving response) ===

2. NIC receives packet via DMA → RX ring buffer
   └─► IRQ: nic_isr()
       └─► napi_schedule()
           └─► NAPI poll:
               ├── Read RX descriptor
               ├── Allocate sk_buff, copy data
               ├── eth_type_trans() → set protocol
               └─► napi_gro_receive(skb)
                   └─► Netfilter: NF_INET_PRE_ROUTING
                       └─► IP: ip_rcv()
                           ├── Route: local delivery
                           └─► TCP: tcp_v4_rcv()
                               ├── Find socket by 4-tuple
                               ├── Process TCP state machine
                               ├── ACK handling
                               └─► sk_data_ready()
                                   └─► wake_up(socket wait)

3. App: recv(sock, buf, len)
   └─► Returns HTTP response data
```

---

## 21.5 I2C Transaction — Software to Wire

```
I2C Read: regmap_read() to Wire Signal

1. Driver: regmap_read(regmap, 0x0F, &val)
   └─► regmap: check cache → miss
       └─► regmap_raw_read()
           └─► regmap_i2c_read()
               └─► i2c_transfer(adapter, msgs, 2)

2. I2C Core:
   └─► __i2c_transfer(adapter, msgs, 2)
       ├── Lock adapter mutex
       └─► adapter->algo->master_xfer(adapter, msgs, 2)

3. I2C Adapter Driver (e.g., i2c-qcom-geni):
   Message 0 (write register address):
   ├── Program GENI: addr=0x48, write, len=1
   ├── Write 0x0F to TX FIFO
   ├── Trigger I2C START
   ├── wait_for_completion(&done)
   └── Hardware signals:
       START → [1001000][0] → ACK → [00001111] → ACK

   Message 1 (read register value):
   ├── Program GENI: addr=0x48, read, len=1
   ├── Trigger I2C RESTART
   ├── wait_for_completion(&done)
   ├── Read RX FIFO → data byte
   └── Hardware signals:
       RESTART → [1001000][1] → ACK → [data byte] → NACK → STOP

4. Return path:
   └─► i2c_transfer returns 2 (success)
       └─► regmap: update cache entry for reg 0x0F
           └─► regmap_read returns val
               └─► Driver has register value
```

---

## 21.6 Runtime PM Flow

```
Runtime PM: Idle → Suspend → Resume on Access

Timeline:
  ─────────────────────────────────────────────────────────►
       │         │              │         │        │
  last access  put_auto   timer fires   get_sync  access
       │       suspend        │         │        HW
       │         │            │         │        │
  [ACTIVE]  [timer starts] [SUSPEND]  [RESUME] [ACTIVE]
       │         │            │         │        │
    clocks ON  delay=200ms  runtime_   runtime_  clocks ON
    regs OK    counting...  suspend()  resume()  regs OK
                            clk_off    clk_on

Driver code flow:
  /* After I2C transaction completes */
  pm_runtime_mark_last_busy(dev);
  pm_runtime_put_autosuspend(dev);
    └─► start 200ms timer

  /* 200ms later, no new access */
  runtime_suspend(dev) called:
    └─► clk_disable_unprepare(priv->clk);
        /* Device consuming zero dynamic power */

  /* New I2C request arrives */
  pm_runtime_get_sync(dev);
    └─► runtime_resume(dev) called:
        └─► clk_prepare_enable(priv->clk);
            /* Device ready, proceed with I2C */
```

---

## Kernel Source References

| Component | Path | Purpose |
|-----------|------|---------|
| V4L2 ioctl flow | drivers/media/v4l2-core/v4l2-ioctl.c | ioctl dispatch |
| DRM atomic | drivers/gpu/drm/drm_atomic.c | Atomic commit |
| TCP transmit | net/ipv4/tcp_output.c | TX path |
| IP receive | net/ipv4/ip_input.c | RX path |
| ASoC PCM | sound/soc/soc-pcm.c | Audio streaming |

---

## Interview Questions

**Q1: Trace the complete path of a camera frame from sensor to user space.**
A: (1) Sensor generates pixel data on MIPI CSI-2 lanes. (2) CSI-2 receiver deserializes and writes to ISP. (3) ISP processes (debayer, denoise, white balance) and outputs to DMA. (4) DMA writes frame to pre-allocated vb2 buffer in DDR. (5) DMA complete interrupt fires. (6) ISR calls `vb2_buffer_done()`, marking buffer DONE. (7) User space `poll()` returns ready. (8) `VIDIOC_DQBUF` ioctl returns the buffer index. (9) User space reads frame data from mmap'd address. (10) User space re-queues buffer with `VIDIOC_QBUF`.

**Q2: How many kernel frameworks are involved in a single audio playback?**
A: At minimum 6: (1) **ALSA/ASoC** — PCM management and card/DAI/codec model. (2) **DMA engine** — cyclic DMA for ring buffer streaming. (3) **Clock framework** — I2S and codec master clocks. (4) **I2C** — codec register configuration. (5) **Regulator** — codec power supplies. (6) **GPIO** — amplifier enable, headphone detect. Plus **Runtime PM** for power management, **pinctrl** for I2S pin configuration, and **interrupt** framework for DMA period elapsed. Each framework has its own probe, error handling, and debug interfaces.

**Q3: What happens during an atomic display commit?**
A: (1) User space builds an atomic request with property changes for planes, CRTCs, connectors. (2) `drmModeAtomicCommit()` enters kernel. (3) `atomic_check()` validates: bandwidth, format support, plane overlap, encoder limits — returns error if any check fails. (4) If TEST_ONLY flag set, return result without applying. (5) `atomic_commit()` applies all changes: sets pixel clock rate, configures encoder PHY, programs CRTC timing registers, updates plane scanout address to new framebuffer. (6) All changes take effect at next VBlank — no tearing. (7) If PAGE_FLIP_EVENT requested, user space gets a completion event.

---

## Summary

- Camera capture involves V4L2, DMA, I2C, clock, regulator, GPIO, and interrupt frameworks
- Audio playback flows through ALSA/ASoC, DMA cyclic, clock, I2C, regulator, and DAPM
- Display pipeline: GEM buffer → plane → CRTC → encoder → connector → panel
- Network TX: socket → TCP → IP → netfilter → driver → DMA → NIC
- Every hardware access involves Runtime PM → clock enable → register access → clock disable
- Understanding end-to-end flows is critical for debugging multi-framework issues

---

*Next: [Chapter 22 — Embedded/Automotive Architecture](Chapter_22_Embedded_Automotive.md)*
